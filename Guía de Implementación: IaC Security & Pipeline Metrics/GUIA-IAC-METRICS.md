# Guía de Implementación: IaC Security & Pipeline Metrics

> **Versión:** 1.0.0 | **Módulo 4 de la Suite DevOps**

---

## Tabla de Contenidos

1. [Estructura y Prerrequisitos](#1-estructura-y-prerrequisitos)
2. [Configuración de IaC Security](#2-configuración-de-iac-security)
3. [Configuración de Pipeline Metrics](#3-configuración-de-pipeline-metrics)
4. [Integración con la Suite CI/CD Completa](#4-integración-con-la-suite-cicd-completa)
5. [Personalización por Tipo de Infraestructura](#5-personalización-por-tipo-de-infraestructura)
6. [Solución de Problemas](#6-solución-de-problemas)

---

## 1. Estructura y Prerrequisitos

### 1.1 Archivos del Módulo

```
.github/
├── workflows/
│   ├── ci-cd.yml                   ← Módulo 1: CI/CD principal
│   ├── auto-merge-dependabot.yml   ← Módulo 2: Auto-merge
│   ├── dependabot.yml              ← Módulo 2: Dependabot config
│   ├── release-on-demand.yml       ← Módulo 3: Releases
│   ├── publish-package.yml         ← Módulo 3: Publicación
│   ├── iac-security.yml            ← Módulo 4: Este archivo
│   └── pipeline-metrics.yml        ← Módulo 4: Este archivo
├── pipeline-metrics.json           ← Histórico de métricas (auto-generado)
└── dependabot.yml
terraform/                          ← Código Terraform
kubernetes/                         ← Manifiestos K8s
helm/                               ← Charts de Helm
```

### 1.2 Secrets Necesarios

```
Settings → Secrets and variables → Actions → New repository secret
```

| Secret | Requerido para | Cómo obtenerlo |
|--------|---------------|----------------|
| `TF_API_TOKEN` | Terraform Cloud (opcional) | app.terraform.io → Tokens |
| `INFRACOST_API_KEY` | Estimación de costos | infracost.io → Org Settings → API Keys |
| `AWS_ACCESS_KEY_ID` | Terraform plan con AWS | IAM → Users → Security credentials |
| `AWS_SECRET_ACCESS_KEY` | Terraform plan con AWS | IAM → Users → Security credentials |
| `DATADOG_API_KEY` | Envío de métricas a Datadog | app.datadoghq.com → Organization Settings → API Keys |
| `SLACK_WEBHOOK_URL` | Alertas de degradación | Slack API → Incoming Webhooks |

### 1.3 Variables de Repositorio

```
Settings → Secrets and variables → Actions → Variables
```

| Variable | Valor Ejemplo | Descripción |
|----------|--------------|-------------|
| `TF_DIR` | `terraform` | Directorio de archivos Terraform |
| `TF_VERSION` | `1.9.0` | Versión de Terraform a usar |
| `K8S_DIR` | `kubernetes` | Directorio de manifiestos K8s |
| `HELM_DIR` | `helm` | Directorio de charts de Helm |
| `AWS_REGION` | `us-east-1` | Región de AWS por defecto |

---

## 2. Configuración de IaC Security

### 2.1 Habilitar el Security Tab de GitHub

Los workflows suben resultados en formato SARIF al Security tab de GitHub, que muestra un inventario centralizado de todas las vulnerabilidades encontradas:

```
Settings → Code security → Code scanning → Set up → GitHub Actions
```

Habilitar:
- ✅ **Code scanning alerts** — Para resultados SARIF de tfsec, checkov, trivy, etc.
- ✅ **Secret scanning** — Para detectar secretos commiteados
- ✅ **Dependabot alerts** — Para vulnerabilidades en dependencias

Una vez configurado, todos los resultados SARIF de los jobs aparecen en:

```
Security → Code scanning → [lista de alertas con severidad, herramienta y línea de código]
```

### 2.2 Configurar `.tfsec.yml` para Reglas Personalizadas

```yaml
# .tfsec.yml (en la raíz del directorio terraform/)
minimum_severity: MEDIUM

exclude:
  # Excluir reglas que no aplican a tu entorno
  - AWS013   # Task definition requires read-only root (válido en algunos casos)
  - AWS004   # CloudFront should have logging enabled (entorno dev)

custom_checks:
  - code: MYCHECK001
    description: "No usar la region us-east-1 para datos sensibles"
    impact: "Los datos sensibles deben estar en regiones específicas"
    resolution: "Mover a eu-west-1 o ap-southeast-1"
    required_types:
      - resource
    required_labels:
      - aws_s3_bucket
    blocks:
      - attribute: region
        equals: us-east-1
    severity: MEDIUM
```

### 2.3 Configurar Checkov con `.checkov.yaml`

```yaml
# .checkov.yaml (en la raíz del repositorio)
directory:
  - terraform
  - kubernetes
  - helm

framework:
  - terraform
  - kubernetes
  - dockerfile
  - helm

skip-check:
  # Justificaciones documentadas de reglas omitidas:
  - CKV_K8S_8   # Liveness probe: no aplica a jobs batch de corta duración
  - CKV_K8S_9   # Readiness probe: idem
  - CKV2_AWS_5  # Security group attached: se usa SG sin EC2 en algunos casos

soft-fail: true  # No fallar el build; solo reportar

compact: true
quiet: false

# Para repos privados con Prisma Cloud:
# bc-api-key: ${BC_API_KEY}
# repo-id: "org/repo"
```

### 2.4 Configurar hadolint con `.hadolint.yaml`

```yaml
# .hadolint.yaml (en la raíz del repositorio)
ignore:
  - DL3008  # Pin versions in apt-get install (aceptable en algunas bases)
  - DL3013  # Pin versions in pip install (gestionado por requirements.txt)

trustedRegistries:
  - docker.io
  - gcr.io
  - public.ecr.aws
  - ghcr.io

# Tratar estos warnings como errores (más estricto)
failure-threshold: warning

# Formato de output (sarif para GitHub Code Scanning)
format: sarif
```

### 2.5 Configurar `.secrets.baseline` para detect-secrets

```bash
# Generar el baseline inicial (excluye falsos positivos conocidos)
pip install detect-secrets
detect-secrets scan \
  --exclude-files "\.git/.*" \
  --exclude-files "node_modules/.*" \
  --exclude-files "vendor/.*" \
  --exclude-files ".*\.lock$" \
  --exclude-files "package-lock\.json" \
  > .secrets.baseline

# Revisar el baseline generado y marcar falsos positivos como auditados
detect-secrets audit .secrets.baseline

git add .secrets.baseline
git commit -m "chore: add secrets baseline for detect-secrets"
```

### 2.6 Configurar .trufflehog-include.txt

```
# .trufflehog-include.txt
# Solo escanear estos patrones de archivos (reduce ruido)
terraform/.*
kubernetes/.*
helm/.*
.*\.tf
.*\.yaml
.*\.yml
Dockerfile.*
docker-compose.*
.*\.env\.example
.*\.cfg
.*\.ini
.*\.conf
```

### 2.7 Ejemplo de Configuración Terraform para Pasar los Checks

```hcl
# terraform/main.tf — ejemplo que pasa los checks de seguridad

# ✅ Bucket S3 con cifrado, versionado y bloqueo de acceso público
resource "aws_s3_bucket" "app_data" {
  bucket = "${var.environment}-mi-app-data"

  tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

resource "aws_s3_bucket_versioning" "app_data" {
  bucket = aws_s3_bucket.app_data.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "app_data" {
  bucket = aws_s3_bucket.app_data.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "app_data" {
  bucket                  = aws_s3_bucket.app_data.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# ✅ Security Group con reglas específicas (sin 0.0.0.0/0 en ingress)
resource "aws_security_group" "app" {
  name        = "${var.environment}-app-sg"
  description = "Security group para la aplicación"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = [var.alb_security_group_id]  # Solo desde el ALB
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]  # Egress abierto es aceptable
  }
}
```

---

## 3. Configuración de Pipeline Metrics

### 3.1 Cómo Interpretar las Métricas DORA

Las métricas DORA (DevOps Research and Assessment) son el estándar de la industria para medir la efectividad de los equipos de ingeniería:

```
Estado de tu equipo según DORA:

⭐ ELITE  → Top 10% de la industria
           Deployment Frequency: >1 deploy/día
           Lead Time: <1 hora
           Change Failure Rate: <5%
           MTTR: <1 hora

🟢 HIGH   → Equipos de alto rendimiento
           Deployment Frequency: 1/semana a 1/día
           Lead Time: 1 día a 1 semana
           Change Failure Rate: 5-10%
           MTTR: 1h a 1 día

🟡 MEDIUM → Rendimiento típico de la industria
           Deployment Frequency: 1/mes a 1/semana
           Lead Time: 1 semana a 1 mes
           Change Failure Rate: 10-15%
           MTTR: 1 día a 1 semana

🔴 LOW    → Área de mejora prioritaria
           Deployment Frequency: <1/mes
           Lead Time: >1 mes
           Change Failure Rate: >15%
           MTTR: >1 semana
```

### 3.2 Acceder al Dashboard de Métricas

El dashboard se genera en el **Job Summary** de cada ejecución del workflow:

```
Actions → pipeline-metrics.yml → [click en una ejecución] → Summary
```

Para ver el histórico de artefactos con datos JSON y CSV:

```
Actions → pipeline-metrics.yml → [click en una ejecución] → Artifacts → pipeline-metrics-XXXXX
```

### 3.3 Configurar Alertas de Degradación

Por defecto, el workflow envía una alerta a Slack cuando:
- La tasa de éxito cae por debajo del **80%**
- El MTTR supera las **24 horas**

Para cambiar los umbrales, edita la condición en el job `generate-dashboard`:

```yaml
- name: "🔔 Alertar si hay degradación de métricas"
  if: |
    secrets.SLACK_WEBHOOK_URL != '' &&
    (needs.collect-metrics.outputs.success_rate < '90' ||   # ← cambiar umbral
     needs.collect-metrics.outputs.mttr_hours > '8')        # ← cambiar umbral
```

### 3.4 Integrar con Grafana + InfluxDB

Para un dashboard más visual, puedes enviar las métricas a InfluxDB:

```yaml
# Añadir al job generate-dashboard en pipeline-metrics.yml:
- name: "📡 Enviar métricas a InfluxDB"
  if: secrets.INFLUXDB_TOKEN != ''
  run: |
    curl -sf -X POST \
      "${{ secrets.INFLUXDB_URL }}/api/v2/write?org=${{ secrets.INFLUXDB_ORG }}&bucket=github-actions&precision=s" \
      -H "Authorization: Token ${{ secrets.INFLUXDB_TOKEN }}" \
      -H "Content-Type: text/plain; charset=utf-8" \
      --data-binary "
      pipeline_metrics,repo=${{ github.repository }},env=production \
        success_rate=${{ needs.collect-metrics.outputs.success_rate }},\
        avg_duration=${{ needs.collect-metrics.outputs.avg_duration }},\
        deploy_frequency=${{ needs.collect-metrics.outputs.deploy_frequency }},\
        mttr_hours=${{ needs.collect-metrics.outputs.mttr_hours }} \
        $(date +%s)"
```

Luego en Grafana, crear un dashboard con panels de serie temporal usando InfluxDB como datasource.

### 3.5 Badge Dinámico de Estado del Pipeline

Añade este badge al README para mostrar el estado en tiempo real:

```markdown
![Pipeline Health](https://img.shields.io/github/actions/workflow/status/<ORG>/<REPO>/ci-cd.yml?branch=main&label=Pipeline&style=for-the-badge)
![Deploy Frequency](https://img.shields.io/badge/deploys-weekly-blue?style=for-the-badge)
```

Para un badge con la tasa de éxito dinámica, usar [shields.io endpoint](https://shields.io/endpoint):

```json
// Crear endpoint JSON en tu repo (ej. .github/badges/success-rate.json):
{
  "schemaVersion": 1,
  "label": "Pipeline Success",
  "message": "97%",
  "color": "brightgreen"
}
```

```markdown
![Success Rate](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/<ORG>/<REPO>/main/.github/badges/success-rate.json)
```

---

## 4. Integración con la Suite CI/CD Completa

### 4.1 Mapa de Dependencias entre Módulos

```
PUSH a feature/*
    ↓
ci-cd.yml         → lint + test + security
    ↓ (PR merged)
ci-cd.yml         → build + deploy-staging
    ↓ (aprobación)
ci-cd.yml         → deploy-production
    ↓ (completado)
pipeline-metrics  → actualiza métricas DORA

MANUAL (Actions UI)
    ↓
release-on-demand → bump versión + GitHub Release
    ↓ (automático)
publish-package   → publica en npm/PyPI/Docker

PUSH a archivos .tf/*.yaml/Dockerfile
    ↓
iac-security      → tfsec + checkov + trivy + kubesec

DEPENDABOT abre PR
    ↓
ci-cd.yml         → lint + test (en la PR)
    ↓ (CI pasa)
auto-merge        → aprueba + habilita auto-merge

SCHEDULE (lunes 08:00)
    ↓
pipeline-metrics  → reporte DORA semanal → Slack
dependabot        → revisa nuevas versiones
iac-security      → escaneo completo de seguridad
```

### 4.2 Evitar Ejecuciones Redundantes

Con todos los módulos activos, algunos eventos disparan múltiples workflows. Para optimizar:

```yaml
# En ci-cd.yml, saltar escaneos de IaC (ya los hace iac-security.yml):
jobs:
  security:
    name: "🔒 Security Scan"
    # Saltar si los únicos cambios son en archivos de IaC
    if: |
      !contains(join(github.event.commits.*.modified, ','), 'terraform/') &&
      !contains(join(github.event.commits.*.modified, ','), 'kubernetes/')
```

```yaml
# En iac-security.yml, saltar si no hay cambios de IaC en el push:
on:
  push:
    paths:
      - "terraform/**"     # Solo si cambian estos archivos
      - "kubernetes/**"
      - "Dockerfile"
      # Sin paths: siempre corre (más seguro pero más costoso)
```

### 4.3 Conectar pipeline-metrics.yml a Todos los Workflows

El workflow de métricas ya tiene configurado `workflow_run` para escuchar a los workflows principales. Si añades nuevos workflows a la suite, agrégalos a la lista:

```yaml
# En pipeline-metrics.yml, sección on.workflow_run.workflows:
workflows:
  - "CI/CD Pipeline - Reusable Template"
  - "🚀 Release on Demand"
  - "🔒 IaC Security & Validation"
  - "🤖 Auto-merge Dependabot PRs"
  - "📦 Publish Package"        # ← Añadir nuevos workflows aquí
  - "Mi Workflow Personalizado"
```

---

## 5. Personalización por Tipo de Infraestructura

### 5.1 Solo Terraform (sin K8s ni Helm)

Simplifica el workflow de IaC deshabilitando los jobs no necesarios:

```yaml
# En iac-security.yml:
kubernetes-security:
  if: false  # Deshabilitar permanentemente

secrets-scan:
  # Mantener siempre activo
```

### 5.2 Solo Kubernetes (sin Terraform)

```yaml
terraform-security:
  if: false  # Deshabilitar permanentemente

docker-security:
  # Mantener: los containers son parte de K8s
```

### 5.3 Monorepo con Múltiples Entornos Terraform

Si tienes `terraform/staging/` y `terraform/production/`:

```yaml
# Repetir el job terraform-security para cada entorno:
terraform-staging:
  uses: ./.github/workflows/iac-security.yml
  with:
    TF_DIR: terraform/staging

terraform-production:
  uses: ./.github/workflows/iac-security.yml
  with:
    TF_DIR: terraform/production
  # Más estricto en producción:
  env:
    CHECKOV_SOFT_FAIL: "false"  # Fallar el build si hay issues en prod
```

### 5.4 Integrar con Terraform Cloud

```yaml
# En el step "terraform init" de iac-security.yml:
- name: "⚙️ terraform init (con Terraform Cloud)"
  working-directory: ${{ env.TF_DIR }}
  run: terraform init -input=false
  env:
    TF_CLI_CONFIG_FILE: ${{ runner.temp }}/.terraformrc
    TF_TOKEN_app_terraform_io: ${{ secrets.TF_API_TOKEN }}

# Crear el archivo de config antes del init:
- name: "🔧 Configurar Terraform Cloud"
  run: |
    cat > ${{ runner.temp }}/.terraformrc << EOF
    credentials "app.terraform.io" {
      token = "${{ secrets.TF_API_TOKEN }}"
    }
    EOF
```

---

## 6. Solución de Problemas

### 6.1 Los resultados SARIF no aparecen en el Security Tab

**Causa:** El Security tab de Code Scanning no está habilitado o el repositorio es privado sin GitHub Advanced Security.

```
# Verificar que Code Scanning está activo:
Settings → Code security → Code scanning

# Para repositorios privados sin GHAS:
# Los resultados SARIF solo están disponibles con GitHub Advanced Security
# Alternativa: usar los resultados en formato de texto en los logs del workflow
```

**Solución alternativa para repos privados sin GHAS:**

```yaml
# Cambiar el formato de output a texto legible en lugar de SARIF:
- name: "🛡️ tfsec (sin SARIF)"
  run: |
    docker run --rm \
      -v "$(pwd):/src" \
      aquasec/tfsec:latest \
      /src/${{ env.TF_DIR }} \
      --format text \
      --minimum-severity MEDIUM
```

### 6.2 `terraform init` falla en CI por falta de credenciales

**Causa:** El step de `init` intenta conectarse al backend de Terraform (S3, Terraform Cloud, etc.) pero no hay credenciales disponibles.

**Solución:** Usar `-backend=false` para validar sin conectarse al backend:

```yaml
- name: "⚙️ terraform init (sin backend)"
  run: terraform init -backend=false -input=false
  # Suficiente para fmt, validate, tfsec y checkov
  # Solo terraform plan y apply necesitan el backend real
```

### 6.3 `pipeline-metrics.yml` muestra métricas de 0 o vacías

**Causa:** La API de GitHub devuelve resultados paginados o el período es muy largo.

```bash
# Verificar que el GH_TOKEN tiene permisos de lectura de Actions:
# Settings → Actions → General → Workflow permissions → Read and write

# Verificar manualmente la respuesta de la API:
gh api "repos/$OWNER/$REPO/actions/runs" --jq '.workflow_runs | length'

# Si devuelve 0, puede ser que los runs sean muy antiguos:
gh api "repos/$OWNER/$REPO/actions/runs" --jq '.[0].created_at'
```

### 6.4 infracost falla con "No Terraform files found"

```bash
# infracost necesita el plan de Terraform, no solo los archivos .tf
# Generar el plan primero:
cd terraform/
terraform init
terraform plan -out=tfplan.binary
terraform show -json tfplan.binary > tfplan.json

# Luego ejecutar infracost con el plan:
infracost breakdown --path tfplan.json
```

### 6.5 kubesec devuelve score 0 para todos los manifiestos

**Causa:** kubesec puede devolver score 0 si el manifiesto no tiene un `kind` soportado (solo soporta Deployment, DaemonSet, StatefulSet, Pod, CronJob, Job).

```bash
# Verificar qué tipos de recursos tienes:
grep "^kind:" kubernetes/*.yaml

# kubesec solo analiza:
# Pod, Deployment, DaemonSet, StatefulSet, ReplicaSet, Job, CronJob
# Ignora: ConfigMap, Service, Ingress, PersistentVolumeClaim, etc.
```

---

## Apéndice: Resumen de Herramientas de IaC Security

| Herramienta | Tipo | Ecosistema | Output SARIF | Licencia |
|-------------|------|-----------|-------------|---------|
| **tfsec** | Seguridad | Terraform | ✅ | MIT |
| **checkov** | Multi-framework | TF, K8s, Docker, Helm | ✅ | Apache 2.0 |
| **hadolint** | Lint | Dockerfile | ✅ | GPL 3.0 |
| **trivy** | Vulnerabilidades + Config | Docker, FS, K8s, IaC | ✅ | Apache 2.0 |
| **kubeval** | Validación schema | Kubernetes | ❌ | Apache 2.0 |
| **kube-score** | Mejores prácticas | Kubernetes | ✅ (parcial) | MIT |
| **kubesec** | Seguridad | Kubernetes | ❌ (JSON) | Apache 2.0 |
| **helm lint** | Validación | Helm | ❌ | Apache 2.0 |
| **TruffleHog** | Secretos | Todo | ❌ | AGPL 3.0 |
| **detect-secrets** | Secretos | Todo | ❌ | Apache 2.0 |
| **syft** | SBOM | Docker, FS | SPDX/CycloneDX | Apache 2.0 |
| **grype** | Vulnerabilidades en SBOM | SBOM | ✅ | Apache 2.0 |
| **infracost** | Costos | Terraform | ❌ (JSON) | Apache 2.0 |

---

*Guía generada para IaC Security & Pipeline Metrics v1.0.0 — Módulo 4 de la Suite DevOps*
