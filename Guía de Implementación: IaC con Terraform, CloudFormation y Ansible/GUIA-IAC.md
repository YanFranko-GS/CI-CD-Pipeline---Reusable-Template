# Guía de Implementación: IaC con Terraform, CloudFormation y Ansible

> **Versión:** 1.0.0 | **Módulo 7 de la Suite DevOps**

---

## Tabla de Contenidos

1. [Principios de IaC en Pipelines](#1-principios-de-iac-en-pipelines)
2. [Configuración de OIDC para Autenticación Cloud](#2-configuración-de-oidc-para-autenticación-cloud)
3. [Estado Remoto de Terraform](#3-estado-remoto-de-terraform)
4. [Integración con el Pipeline CI/CD Existente](#4-integración-con-el-pipeline-cicd-existente)
5. [GitHub Environments para Aislamiento](#5-github-environments-para-aislamiento)
6. [Personalización por Nube](#6-personalización-por-nube)
7. [Solución de Problemas Comunes](#7-solución-de-problemas-comunes)

---

## 1. Principios de IaC en Pipelines

### 1.1 Separación Plan / Apply

El principio más importante del IaC en CI/CD es **nunca aplicar cambios sin revisión**:

```
PR abierto
    ↓
terraform-plan.yml       ← Plan automático, comentado en el PR
    ↓
Revisión humana del plan  ← "¿Es esto lo que queremos cambiar?"
    ↓
PR aprobado y mergeado
    ↓
terraform-apply.yml      ← Apply usa el plan ya revisado (no re-planifica)
    ↓
Infraestructura actualizada
```

**Por qué no re-planificar en el apply:** Entre el plan y el apply pueden pasar minutos u horas. Si alguien hizo otro cambio en ese tiempo, un re-plan podría incluir cambios no revisados. Usar el plan pregenerado garantiza que aplicamos exactamente lo que fue aprobado.

### 1.2 Flujo de Revisión en PRs

```yaml
# El flujo completo de un cambio de infraestructura:

1. git checkout -b infra/add-rds-instance
2. # Modificar terraform/database.tf
3. git push origin infra/add-rds-instance
4. # Abrir PR → terraform-plan.yml se dispara automáticamente
5. # El plan se comenta en el PR con los cambios propuestos
6. # El reviewer lee: "Plan: 3 to add, 0 to change, 0 to destroy"
7. # Aprueba el PR y hace merge
8. # terraform-apply.yml aplica el plan pregenerado
```

### 1.3 Estructura de Directorios Recomendada

```
repository/
├── terraform/
│   ├── main.tf              ← Recursos principales
│   ├── variables.tf         ← Declaración de variables
│   ├── outputs.tf           ← Valores exportados
│   ├── versions.tf          ← Versiones de providers y Terraform
│   ├── backends/
│   │   ├── staging.tfbackend    ← Config de backend para staging
│   │   └── production.tfbackend ← Config de backend para prod
│   ├── envs/
│   │   ├── staging.tfvars       ← Variables de staging
│   │   └── production.tfvars    ← Variables de producción
│   └── modules/
│       ├── networking/          ← Módulos reutilizables
│       └── compute/
├── cloudformation/
│   ├── template.yaml            ← Plantilla principal
│   └── nested/                  ← Plantillas anidadas
├── ansible/
│   ├── playbooks/
│   │   ├── deploy-app.yml
│   │   └── configure-nginx.yml
│   ├── inventories/
│   │   ├── staging/
│   │   │   ├── hosts.yml
│   │   │   └── aws_ec2.yml      ← Inventario dinámico
│   │   └── production/
│   ├── roles/
│   │   └── app/
│   └── requirements.yml
└── .github/
    └── workflows/
        ├── terraform-plan.yml
        ├── terraform-apply.yml
        ├── cloudformation-deploy.yml
        └── ansible-playbook.yml
```

---

## 2. Configuración de OIDC para Autenticación Cloud

OIDC (OpenID Connect) permite que GitHub Actions obtenga credenciales temporales del cloud sin guardar access keys de larga duración como secrets. Es la práctica recomendada por AWS, Azure y GCP.

### 2.1 AWS — Configuración de OIDC

**Paso 1: Crear el Identity Provider en IAM**

```bash
# Via AWS CLI
aws iam create-open-id-connect-provider \
  --url "https://token.actions.githubusercontent.com" \
  --client-id-list "sts.amazonaws.com" \
  --thumbprint-list "6938fd4d98bab03faadb97b34396831e3780aea1"

# Verificar
aws iam list-open-id-connect-providers
```

**Paso 2: Crear el Rol IAM con política de confianza**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": [
            "repo:MI_ORG/MI_REPO:environment:staging",
            "repo:MI_ORG/MI_REPO:environment:production",
            "repo:MI_ORG/MI_REPO:ref:refs/heads/main"
          ]
        }
      }
    }
  ]
}
```

```bash
# Crear el rol
aws iam create-role \
  --role-name github-actions-terraform \
  --assume-role-policy-document file://trust-policy.json

# Adjuntar políticas de Terraform (principio de menor privilegio)
aws iam attach-role-policy \
  --role-name github-actions-terraform \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess  # Para plan

# Política personalizada para apply (solo los recursos que Terraform gestiona):
cat > terraform-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:*", "rds:*", "s3:*", "iam:*",
        "elasticloadbalancing:*", "autoscaling:*",
        "cloudwatch:*", "logs:*"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["dynamodb:*"],
      "Resource": "arn:aws:dynamodb:*:*:table/terraform-state-lock"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name github-actions-terraform \
  --policy-name terraform-permissions \
  --policy-document file://terraform-policy.json
```

**Paso 3: Guardar el ARN como secret en GitHub**

```
Settings → Environments → staging → Secrets → New secret
Name: AWS_ROLE_ARN
Value: arn:aws:iam::123456789012:role/github-actions-terraform
```

### 2.2 Azure — Workload Identity Federation

**Paso 1: Crear la App Registration**

```bash
# Crear la app registration para el workflow
APP_ID=$(az ad app create \
  --display-name "github-actions-terraform" \
  --query appId -o tsv)

# Crear el Service Principal
OBJECT_ID=$(az ad sp create \
  --id "$APP_ID" \
  --query id -o tsv)

# Asignar rol de Contributor en la suscripción
az role assignment create \
  --role "Contributor" \
  --assignee-object-id "$OBJECT_ID" \
  --scope "/subscriptions/SUBSCRIPTION_ID"
```

**Paso 2: Configurar Federated Credential**

```bash
# Para el environment "staging"
az ad app federated-credential create \
  --id "$APP_ID" \
  --parameters '{
    "name": "github-staging",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:MI_ORG/MI_REPO:environment:staging",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# Para la rama main
az ad app federated-credential create \
  --id "$APP_ID" \
  --parameters '{
    "name": "github-main",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:MI_ORG/MI_REPO:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

**Paso 3: Guardar como secrets en GitHub**

```
AZURE_CLIENT_ID      = (Application ID de la app registration)
AZURE_TENANT_ID      = (Tenant ID de Azure AD)
AZURE_SUBSCRIPTION_ID = (Subscription ID)
```

### 2.3 GCP — Workload Identity Federation

```bash
# Paso 1: Crear el Workload Identity Pool
gcloud iam workload-identity-pools create "github-pool" \
  --project="MI_PROJECT" \
  --location="global" \
  --display-name="GitHub Actions Pool"

# Paso 2: Crear el Provider dentro del pool
gcloud iam workload-identity-pools providers create-oidc "github-provider" \
  --project="MI_PROJECT" \
  --location="global" \
  --workload-identity-pool="github-pool" \
  --display-name="GitHub Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --issuer-uri="https://token.actions.githubusercontent.com"

# Paso 3: Crear Service Account
gcloud iam service-accounts create terraform-sa \
  --display-name="Terraform Service Account" \
  --project="MI_PROJECT"

# Paso 4: Conceder permiso al SA para ser impersonado por GitHub
gcloud iam service-accounts add-iam-policy-binding \
  "terraform-sa@MI_PROJECT.iam.gserviceaccount.com" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/attribute.repository/MI_ORG/MI_REPO"

# Paso 5: Asignar roles al SA
gcloud projects add-iam-policy-binding "MI_PROJECT" \
  --member="serviceAccount:terraform-sa@MI_PROJECT.iam.gserviceaccount.com" \
  --role="roles/editor"
```

**Guardar como secrets:**

```
GCP_WORKLOAD_IDENTITY_PROVIDER = projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/providers/github-provider
GCP_SERVICE_ACCOUNT = terraform-sa@MI_PROJECT.iam.gserviceaccount.com
```

---

## 3. Estado Remoto de Terraform

### 3.1 Backend S3 con DynamoDB para Bloqueo

**Crear los recursos de infraestructura para el estado:**

```bash
# Crear bucket S3 para el estado
aws s3 mb s3://mi-empresa-terraform-state --region us-east-1

# Habilitar versioning (permite rollback del estado)
aws s3api put-bucket-versioning \
  --bucket mi-empresa-terraform-state \
  --versioning-configuration Status=Enabled

# Bloquear acceso público (el estado contiene información sensible)
aws s3api put-public-access-block \
  --bucket mi-empresa-terraform-state \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# Habilitar cifrado server-side con KMS (recomendado)
aws s3api put-bucket-encryption \
  --bucket mi-empresa-terraform-state \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms"
      }
    }]
  }'

# Crear tabla DynamoDB para el bloqueo del estado
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

**Archivo de backend por entorno:**

```hcl
# terraform/backends/staging.tfbackend
bucket         = "mi-empresa-terraform-state"
key            = "staging/terraform.tfstate"
region         = "us-east-1"
dynamodb_table = "terraform-state-lock"
encrypt        = true
# Para usar KMS:
# kms_key_id = "arn:aws:kms:us-east-1:123456789:key/xxxxx"
```

```hcl
# terraform/backends/production.tfbackend
bucket         = "mi-empresa-terraform-state"
key            = "production/terraform.tfstate"
region         = "us-east-1"
dynamodb_table = "terraform-state-lock"
encrypt        = true
```

**`versions.tf` con backend configurado:**

```hcl
# terraform/versions.tf
terraform {
  required_version = ">= 1.8.0"

  # El backend se configura via backend-config en el init (no hardcodeado aquí)
  # Esto permite usar el mismo código para múltiples entornos
  backend "s3" {}

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

### 3.2 Terraform Cloud / HCP Terraform

```hcl
# terraform/versions.tf (para Terraform Cloud)
terraform {
  cloud {
    organization = "mi-organizacion"

    workspaces {
      # Un workspace por entorno
      tags = ["environment:${var.environment}"]
      # O especificar el nombre directamente:
      # name = "mi-app-staging"
    }
  }
}
```

```bash
# El token de Terraform Cloud se pasa via variable de entorno:
# TF_TOKEN_app_terraform_io = "tu-token-de-terraform-cloud"
# Guardar como secret en GitHub: TF_CLOUD_TOKEN
```

### 3.3 Variable de Entorno para No Hardcodear

```hcl
# terraform/variables.tf
variable "environment" {
  description = "Entorno (staging, production)"
  type        = string
  validation {
    condition     = contains(["staging", "production"], var.environment)
    error_message = "El entorno debe ser 'staging' o 'production'."
  }
}
```

```hcl
# terraform/envs/staging.tfvars
environment    = "staging"
instance_type  = "t3.micro"
min_instances  = 1
max_instances  = 3
```

```hcl
# terraform/envs/production.tfvars
environment    = "production"
instance_type  = "t3.large"
min_instances  = 3
max_instances  = 10
```

---

## 4. Integración con el Pipeline CI/CD Existente

### 4.1 Llamar terraform-plan desde ci-cd.yml

```yaml
# En .github/workflows/ci-cd.yml — añadir como quality gate en PRs:
jobs:
  terraform-plan:
    name: "📋 IaC Plan"
    uses: ./.github/workflows/terraform-plan.yml
    if: |
      github.event_name == 'pull_request' &&
      contains(github.event.pull_request.labels.*.name, 'infra')
    with:
      environment: staging
      working_directory: ./terraform
    secrets: inherit
```

### 4.2 Llamar terraform-apply desde release-on-demand.yml

```yaml
# En .github/workflows/release-on-demand.yml
jobs:
  release:
    # ... bump de versión, tag, etc.

  deploy-infrastructure:
    name: "🏗️ Deploy Infrastructure"
    needs: release
    uses: ./.github/workflows/terraform-apply.yml
    with:
      environment: ${{ github.event.inputs.environment || 'staging' }}
      working_directory: ./terraform
    secrets: inherit

  configure-servers:
    name: "📜 Configure Servers"
    needs: deploy-infrastructure
    if: needs.deploy-infrastructure.outputs.apply_status == 'success'
    uses: ./.github/workflows/ansible-playbook.yml
    with:
      playbook: ansible/playbooks/deploy-app.yml
      inventory: ansible/inventories/${{ github.event.inputs.environment }}/aws_ec2.yml
      environment: ${{ github.event.inputs.environment || 'staging' }}
      extra_vars: '{"app_version": "${{ needs.release.outputs.new_version }}"}'
    secrets: inherit
```

---

## 5. GitHub Environments para Aislamiento

### 5.1 Crear Environments con Protecciones

```
Settings → Environments → New environment

Entorno: staging
  Variables:
    AWS_REGION          = us-east-1
    TF_BACKEND_BUCKET   = mi-empresa-terraform-state
    TF_BACKEND_KEY      = terraform/staging.tfstate
    TF_BACKEND_LOCK_TABLE = terraform-state-lock
    ANSIBLE_REMOTE_USER = ubuntu
    ENVIRONMENT_URL     = https://staging.mi-app.com
    CLOUD_PROVIDER      = aws

  Secrets:
    AWS_ROLE_ARN = arn:aws:iam::123456789:role/github-tf-staging
    ANSIBLE_SSH_PRIVATE_KEY = (clave SSH para los servidores de staging)
    ANSIBLE_VAULT_PASSWORD  = (contraseña del vault de Ansible)

  Protection rules:
    (sin protección — staging puede desplegarse automáticamente)

Entorno: production
  Variables:
    AWS_REGION          = us-east-1
    TF_BACKEND_BUCKET   = mi-empresa-terraform-state
    TF_BACKEND_KEY      = terraform/production.tfstate
    ENVIRONMENT_URL     = https://mi-app.com
    CLOUD_PROVIDER      = aws

  Secrets:
    AWS_ROLE_ARN = arn:aws:iam::123456789:role/github-tf-production
    # Nota: el rol de producción tiene permisos más limitados

  Protection rules:
    ✅ Required reviewers: devops-lead, cto (al menos 1 de 2)
    ✅ Prevent self-review: activado
    ✅ Wait timer: 5 minutes
    ✅ Deployment branches: only main
```

### 5.2 Roles IAM Separados por Entorno (Principio de Menor Privilegio)

```json
// Rol de staging: puede crear/modificar/destruir
// Rol de producción: puede crear/modificar PERO con restricciones en destroy

// trust-policy-production.json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::ACCOUNT:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
        "token.actions.githubusercontent.com:sub": "repo:MI_ORG/MI_REPO:environment:production"
      }
    }
  }]
}
```

---

## 6. Personalización por Nube

### 6.1 AWS — Inventario Dinámico para Ansible

```yaml
# ansible/inventories/staging/aws_ec2.yml
plugin: aws_ec2
regions:
  - us-east-1
filters:
  instance-state-name: running
  tag:Environment: staging
  tag:ManagedBy: terraform
keyed_groups:
  - key: tags.Role
    prefix: role
  - key: tags.AppName
    separator: ""
compose:
  ansible_host: public_ip_address
  ansible_user: "'ubuntu'"
```

### 6.2 Terraform con Azure Backend

```hcl
# terraform/backends/staging.tfbackend (Azure)
resource_group_name  = "terraform-state-rg"
storage_account_name = "miempresatfstate"
container_name       = "terraform-state"
key                  = "staging.tfstate"
use_oidc             = true  # Usar OIDC en lugar de SAS token
```

### 6.3 Terraform con GCP Backend

```hcl
# terraform/versions.tf (GCP)
terraform {
  backend "gcs" {}
  # Configurar via backend-config:
  # bucket = "mi-empresa-terraform-state"
  # prefix = "terraform/staging"
}
```

---

## 7. Solución de Problemas Comunes

### 7.1 Error de Permisos en OIDC

```
Error: InvalidIdentityToken: Couldn't retrieve verification key from your identity provider
```

**Causas y soluciones:**

```bash
# Causa 1: El thumbprint del OIDC provider está incorrecto
# Solución: regenerar el thumbprint
THUMBPRINT=$(openssl s_client -servername token.actions.githubusercontent.com \
  -connect token.actions.githubusercontent.com:443 \
  2>/dev/null | openssl x509 -fingerprint -noout -sha1 | \
  cut -d'=' -f2 | tr -d ':' | tr '[:upper:]' '[:lower:]')
echo "Thumbprint: $THUMBPRINT"

# Actualizar el OIDC provider con el thumbprint correcto
aws iam update-open-id-connect-provider-thumbprint \
  --open-id-connect-provider-arn arn:aws:iam::ACCOUNT:oidc-provider/token.actions.githubusercontent.com \
  --thumbprint-list "$THUMBPRINT"
```

```
Error: AccessDenied: User is not authorized to perform sts:AssumeRoleWithWebIdentity
```

```json
// Causa 2: La condición del Subject en la política de confianza no coincide
// El subject de OIDC tiene el formato exacto:
// repo:ORG/REPO:environment:ENVIRONMENT_NAME
// repo:ORG/REPO:ref:refs/heads/BRANCH
// repo:ORG/REPO:pull_request

// Verificar qué subject está generando el workflow:
// En el workflow, añadir este step de diagnóstico:
// - run: |
//     echo "OIDC Subject: $ACTIONS_ID_TOKEN_REQUEST_URL"
//     curl -H "Authorization: Bearer $ACTIONS_RUNTIME_TOKEN" \
//       "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=sts.amazonaws.com" | \
//       jq -R 'split(".") | .[1] | @base64d | fromjson | .sub'
```

### 7.2 Bloqueo del Estado de Terraform

```
Error: Error acquiring the state lock:
  Error message: ConditionalCheckFailedException: The conditional request failed
  Lock Info: ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

```bash
# Diagnóstico: ver quién tiene el lock
aws dynamodb get-item \
  --table-name terraform-state-lock \
  --key '{"LockID": {"S": "mi-empresa-terraform-state/staging.tfstate"}}' \
  --region us-east-1

# Solución: si el lock es huérfano (el proceso original murió), liberarlo
# ⚠️ SOLO hacer esto si estás SEGURO de que no hay un apply en progreso
terraform force-unlock LOCK_ID -force

# Prevención: el workflow tiene cancel-in-progress: false para apply
# Pero si el runner muere, el lock puede quedar huérfano
```

### 7.3 Fallos de Conectividad SSH en Ansible

```
UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: ..."}
```

```yaml
# Diagnóstico 1: Verificar que la clave SSH es correcta
- name: "🔍 Diagnóstico SSH"
  run: |
    # Intentar conexión manual
    ssh -i /tmp/key.pem -o ConnectTimeout=10 -o StrictHostKeyChecking=no \
      ubuntu@$HOST "echo 'SSH OK'" 2>&1 || echo "SSH FALLO"

# Diagnóstico 2: Verificar conectividad de red
- name: "🔍 Diagnóstico de red"
  run: |
    HOST="$ANSIBLE_HOST"
    echo "Testing conectividad a $HOST..."
    nc -zv -w 10 $HOST 22 2>&1 || echo "Puerto 22 no accesible"
    # Si el host está en VPC privada, necesitarás un bastion o VPN
```

```yaml
# Solución: Configurar SSH a través de un bastion host
# En ansible.cfg:
# [defaults]
# remote_user = ubuntu
#
# [ssh_connection]
# ssh_args = -o ProxyJump=ubuntu@BASTION_IP -o StrictHostKeyChecking=no
```

### 7.4 Debug Avanzado con TF_LOG

```yaml
# En terraform-plan.yml o terraform-apply.yml, activar debug logging:
- name: "📋 Terraform Plan (DEBUG)"
  env:
    TF_LOG: DEBUG
    TF_LOG_PATH: /tmp/terraform-debug.log
    # Opciones de TF_LOG: TRACE, DEBUG, INFO, WARN, ERROR
  run: terraform plan ...

- name: "💾 Subir logs de debug"
  uses: actions/upload-artifact@v4
  if: always()
  with:
    name: terraform-debug-logs
    path: /tmp/terraform-debug.log
```

```bash
# Para Ansible, usar la opción -vvvv (máxima verbosidad):
ansible-playbook -i inventory playbook.yml -vvvv 2>&1 | tee /tmp/ansible-debug.log

# Variables de debug de Ansible:
export ANSIBLE_DEBUG=true
export ANSIBLE_LOG_PATH=/tmp/ansible.log
```

### 7.5 Rollback de Terraform

```bash
# Si un apply falla parcialmente, Terraform puede quedar en estado inconsistente
# Opciones de rollback:

# Opción 1: Revertir el commit que causó el cambio y hacer apply del estado anterior
git revert HEAD  # Revertir el commit problemático
# → El plan del nuevo commit será un "undo" del anterior

# Opción 2: Restaurar un estado anterior desde S3
aws s3 cp \
  s3://mi-empresa-terraform-state/staging.tfstate.backup \
  terraform.tfstate.backup

terraform state push terraform.tfstate.backup
# ⚠️ PELIGROSO: Restaurar un estado antiguo puede causar drift

# Opción 3: Importar recursos que Terraform perdió del estado
terraform import aws_instance.web i-1234567890abcdef0

# Opción 4: Marcar recursos como "tainted" para recrearlos
terraform taint aws_instance.web_corrupted
# En el siguiente apply, ese recurso se destruirá y recreará
```

---

## Apéndice: Tabla de Secretos por Workflow

| Secret | Workflow | Scope | Descripción |
|--------|---------|-------|-------------|
| `AWS_ROLE_ARN` | terraform-plan, terraform-apply, cfn-deploy | Environment | ARN del rol IAM para OIDC |
| `AZURE_CLIENT_ID` | terraform-plan, terraform-apply | Environment | App Registration ID |
| `AZURE_TENANT_ID` | terraform-plan, terraform-apply | Environment | Azure AD Tenant ID |
| `AZURE_SUBSCRIPTION_ID` | terraform-plan, terraform-apply | Environment | Subscription ID |
| `GCP_WORKLOAD_IDENTITY_PROVIDER` | terraform-plan, terraform-apply | Environment | WIF Provider URL |
| `GCP_SERVICE_ACCOUNT` | terraform-plan, terraform-apply | Environment | SA email |
| `ANSIBLE_SSH_PRIVATE_KEY` | ansible-playbook | Environment | Clave privada SSH |
| `ANSIBLE_VAULT_PASSWORD` | ansible-playbook | Environment | Contraseña Ansible Vault |
| `ANSIBLE_BECOME_PASS` | ansible-playbook | Environment | Contraseña sudo |
| `SLACK_WEBHOOK_URL` | Todos | Repository | Webhook de Slack |

---

*Guía generada para IaC con Terraform, CloudFormation y Ansible v1.0.0 — Módulo 7 de la Suite DevOps*

*Suite completa: ci-cd → dependabot → release-on-demand → iac-security → scheduled-security-scan → e2e-tests → terraform/ansible*
