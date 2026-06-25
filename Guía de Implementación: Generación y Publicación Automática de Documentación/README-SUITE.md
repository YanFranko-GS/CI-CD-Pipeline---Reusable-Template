# 🚀 Suite DevOps GitHub Actions — Referencia Maestra

> **20,120 líneas · 32 archivos · 8 módulos** · Producción-ready para cualquier proyecto

[![CI/CD Pipeline](https://img.shields.io/badge/CI%2FCD-GitHub_Actions-2088FF?logo=github-actions&logoColor=white)](https://github.com/features/actions)
[![IaC](https://img.shields.io/badge/IaC-Terraform_%7C_Ansible_%7C_CloudFormation-7B42BC?logo=terraform)](https://terraform.io)
[![Security](https://img.shields.io/badge/Security-Trivy_%7C_Snyk_%7C_CodeQL-EF4444?logo=snyk)](https://snyk.io)
[![Docs](https://img.shields.io/badge/Docs-Docusaurus_%7C_MkDocs_%7C_Sphinx-3ECC5F?logo=readthedocs)](https://readthedocs.org)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

---

## 📦 Inventario Completo

### Módulo 1 — CI/CD Principal
| Archivo | Líneas | Descripción |
|---------|--------|-------------|
| `ci-cd.yml` | 1,313 | Pipeline completo: lint → test → security → build → deploy staging/prod → notify |

**Herramientas:** ESLint, Pylint, Jest, pytest, JUnit, Trivy, Snyk, CodeQL, Docker Buildx, GitHub Environments, Slack

---

### Módulo 2 — Gestión de Dependencias
| Archivo | Líneas | Descripción |
|---------|--------|-------------|
| `dependabot.yml` | 459 | 6 ecosistemas activos + 8 comentados, grupos de PRs, versioning-strategy |
| `auto-merge-dependabot.yml` | 430 | Auto-merge seguro con doble verificación de actor, manejo de major |
| `renovate.json` | 107 | Config alternativa con automerge, lockFileMaintenance, vulnerabilityAlerts |

**Herramientas:** Dependabot, Renovate, `dependabot/fetch-metadata`, `lewagon/wait-on-check-action`

---

### Módulo 3 — Releases y Publicación
| Archivo | Líneas | Descripción |
|---------|--------|-------------|
| `release-on-demand.yml` | 839 | Bump semántico, changelog auto, tag, GitHub Release, detección de ecosistema |
| `publish-package.yml` | 703 | Publicación en npm, PyPI (OIDC), Maven Central (GPG), Docker Hub, Go modules |

**Herramientas:** `npx semver`, bump2version, `mvn versions:set`, twine, `pypa/gh-action-pypi-publish`, `softprops/action-gh-release`

---

### Módulo 4 — Seguridad de Infraestructura + Métricas
| Archivo | Líneas | Descripción |
|---------|--------|-------------|
| `iac-security.yml` | 890 | tfsec, checkov, hadolint, Trivy, kubeval, kube-score, kubesec, helm lint, Syft SBOM |
| `pipeline-metrics.yml` | 614 | Métricas DORA (4 métricas), alertas de degradación, exportación Datadog/InfluxDB |

**Herramientas:** tfsec, Checkov, hadolint, Trivy, kubesec, kube-score, Syft, Grype, Infracost, GitHub API

---

### Módulo 5 — Escaneos de Seguridad Periódicos
| Archivo | Líneas | Descripción |
|---------|--------|-------------|
| `scheduled-security-scan.yml` | 870 | Trivy FS + Image, Snyk monitor, CodeQL profundo, OSSAR, FOSSA, notificación |
| `secret-rotation-check.yml` | 994 | Verificación de validez: GitHub PATs, AWS IAM, GCP SA, Azure SP, Docker, npm, PyPI |

**Herramientas:** Trivy, Snyk, CodeQL, OSSAR, FOSSA, detect-secrets, TruffleHog, AWS CLI, gcloud, az CLI

---

### Módulo 6 — Testing Especializado
| Archivo | Líneas | Descripción |
|---------|--------|-------------|
| `e2e-tests.yml` | 720 | Playwright + Cypress + Selenium; smoke/regression/full; slash command /e2e; preview efímero |
| `performance-tests.yml` | 806 | k6 con 5 tipos de prueba; thresholds; reporte HTML; bloqueo de producción |
| `chaos-engineering.yml` | 696 | Chaos Mesh: pod-kill, network-delay, cpu-stress, http-abort; medición de disponibilidad |

**Herramientas:** Playwright, Cypress, Selenium, k6, Grafana Cloud, Chaos Mesh, Litmus, `dorny/test-reporter`

---

### Módulo 7 — Infraestructura como Código
| Archivo | Líneas | Descripción |
|---------|--------|-------------|
| `terraform-plan.yml` | 610 | Plan con OIDC (AWS/Azure/GCP), comentario en PR, artefacto del plan, parseo de cambios |
| `terraform-apply.yml` | 391 | Apply con plan pregenerado, bloqueo de concurrencia, outputs, rollback info |
| `cloudformation-deploy.yml` | 514 | Deploy/changeset/delete; validación; comentario en PR; outputs del stack |
| `ansible-playbook.yml` | 613 | Inventario dinámico, Vault, SSH agent efímero, lint previo, PLAY RECAP parsing |

**Herramientas:** Terraform, OpenTofu, AWS CloudFormation, SAM, Ansible, `hashicorp/setup-terraform`, `aws-actions/configure-aws-credentials` (OIDC)

---

### Módulo 8 — Documentación Automática
| Archivo | Líneas | Descripción |
|---------|--------|-------------|
| `deploy-docs.yml` | 722 | Docusaurus, MkDocs, Sphinx, Javadoc, VitePress, Astro; GitHub Pages/Netlify/Vercel |
| `api-docs.yml` | 810 | OpenAPI desde Express/FastAPI/Spring; GraphQL schema; Redoc/Swagger UI portal |

**Herramientas:** Docusaurus, MkDocs Material, Sphinx, Javadoc, VitePress, Astro Starlight, swagger-jsdoc, Redoc, Swagger UI, Spectral

---

### Archivos de Configuración Auxiliares
| Archivo | Descripción |
|---------|-------------|
| `codecov.yml` | Configuración de Codecov (coverage targets, flags) |
| `releaserc.json` | Config de semantic-release con todos los plugins |
| `renovate.json` | Config de Renovate (alternativa a Dependabot) |
| `dependency-update.yml` | Workflow auxiliar de actualización de dependencias |
| `caller-example.yml` | Ejemplo de cómo llamar workflows como reusable |

---

## 🗺️ Mapa de Dependencias entre Módulos

```
PUSH a feature/*
    └─► ci-cd.yml (lint → test → security → build)
              │
              └─► [PR] terraform-plan.yml (si cambian .tf)
              └─► [label run-e2e] e2e-tests.yml (smoke)

MERGE a main
    └─► ci-cd.yml (build → deploy-staging)
    └─► deploy-docs.yml (si cambian docs/)
    └─► api-docs.yml (si cambian src/ o openapi.*)
    └─► iac-security.yml (si cambian .tf, Dockerfile, k8s/)

MANUAL (workflow_dispatch)
    └─► release-on-demand.yml
              ├─► publish-package.yml
              ├─► e2e-tests.yml (quality gate)
              ├─► performance-tests.yml (quality gate)
              └─► terraform-apply.yml (si hay cambios de infra)
                        └─► ansible-playbook.yml (post-infra)

SCHEDULE (cron)
    └─► scheduled-security-scan.yml (lunes 06:00 UTC)
    └─► secret-rotation-check.yml (diario 08:00 UTC)
    └─► pipeline-metrics.yml (lunes 08:00 UTC — reporte DORA)
    └─► e2e-tests.yml (03:00 UTC — suite completa nocturna)

DEPENDABOT (PRs automáticas)
    └─► ci-cd.yml (lint + test en la PR)
    └─► auto-merge-dependabot.yml (aprobar + merge si patch/minor)
```

---

## 📋 Guía de Adopción Gradual

No tienes que implementar todo de golpe. Esta es la secuencia recomendada:

### Semana 1: Base (Módulo 1)
```bash
cp ci-cd.yml .github/workflows/
# Ajustar: PROJECT_TYPE, DEPLOYMENT_TARGET
# Configurar: GitHub Environments staging + production
# Añadir secrets: DOCKER_HUB_TOKEN, SLACK_WEBHOOK_URL
```
**Resultado:** CI/CD completo con lint, test, build, deploy staging y notificaciones.

### Semana 2: Dependencias (Módulo 2)
```bash
cp dependabot.yml .github/
cp auto-merge-dependabot.yml .github/workflows/
# Ajustar: habilitar solo los ecosistemas que usas
# Configurar: Allow auto-merge en la protección de rama
```
**Resultado:** Actualizaciones de dependencias automáticas y seguras.

### Semana 3: Releases (Módulo 3)
```bash
cp release-on-demand.yml .github/workflows/
cp publish-package.yml .github/workflows/
# Configurar: NPM_TOKEN o PYPI_API_TOKEN según el ecosistema
# Instalar: commitlint + husky para conventional commits
```
**Resultado:** Releases con un clic desde la UI de Actions.

### Semana 4-5: Seguridad (Módulos 4 y 5)
```bash
cp iac-security.yml .github/workflows/     # Si tienes IaC
cp scheduled-security-scan.yml .github/workflows/
cp secret-rotation-check.yml .github/workflows/
# Configurar: SNYK_TOKEN, habilitar Code Scanning en el repo
# Habilitar: Security tab → Code scanning
```
**Resultado:** CVEs detectados automáticamente, alertas de secretos expirados.

### Semana 6: Testing Especializado (Módulo 6)
```bash
cp e2e-tests.yml .github/workflows/
cp performance-tests.yml .github/workflows/
# Configurar: E2E_BASE_URL, E2E_TEST_USER/PASSWORD
# Instalar: Playwright en el proyecto (npx playwright install)
# Crear: k6-scripts/smoke.js (generado automáticamente por el workflow)
```
**Resultado:** E2E tests on-demand y pruebas de carga con k6.

### Semana 7-8: IaC (Módulo 7)
```bash
cp terraform-plan.yml .github/workflows/
cp terraform-apply.yml .github/workflows/
# Configurar: OIDC en AWS/Azure/GCP (ver Guía sección 2)
# Crear: terraform/backends/staging.tfbackend
# Habilitar: bucket S3 + DynamoDB para estado remoto
```
**Resultado:** Plan en PRs, apply con aprobación manual para producción.

### Semana 9: Documentación (Módulo 8)
```bash
cp deploy-docs.yml .github/workflows/
cp api-docs.yml .github/workflows/
# Settings → Pages → Source: GitHub Actions
# Configurar: mkdocs.yml o docusaurus.config.ts
```
**Resultado:** Documentación publicada automáticamente en cada push a main.

---

## ⚡ Quick Start: Implementación en 30 Minutos

Para un proyecto Node.js nuevo que quiera lo esencial:

```bash
# 1. Copiar los workflows básicos
mkdir -p .github/workflows
cp ci-cd.yml .github/workflows/
cp dependabot.yml .github/
cp auto-merge-dependabot.yml .github/workflows/
cp release-on-demand.yml .github/workflows/
cp publish-package.yml .github/workflows/

# 2. Configurar variables de repositorio
gh variable set PROJECT_TYPE --body "node"
gh variable set DEPLOYMENT_TARGET --body "docker_hub"
gh variable set IMAGE_NAME --body "mi-org/mi-app"
gh variable set NOTIFY_SLACK --body "true"

# 3. Configurar secrets
gh secret set DOCKER_HUB_USERNAME --body "mi-usuario"
gh secret set DOCKER_HUB_TOKEN --body "dckr_pat_xxx"
gh secret set SLACK_WEBHOOK_URL --body "https://hooks.slack.com/xxx"
gh secret set NPM_TOKEN --body "npm_xxx"

# 4. Crear entornos con protección
gh api repos/{owner}/{repo}/environments/staging -X PUT \
  --field wait_timer=0
gh api repos/{owner}/{repo}/environments/production -X PUT \
  --field wait_timer=300  # 5 min de espera

# 5. Habilitar auto-merge en la rama main
gh api repos/{owner}/{repo}/branches/main/protection -X PUT \
  --field required_status_checks='{"strict":true,"contexts":["🧪 Tests"]}' \
  --field allow_auto_merge=true

# 6. Verificar que todo está configurado
gh workflow list
```

---

## 🔑 Tabla Maestra de Secrets

| Secret | Módulo | Scope | Cómo obtenerlo |
|--------|--------|-------|---------------|
| `GITHUB_TOKEN` | Todos | Automático | Automático en Actions |
| `RELEASE_TOKEN` | 2, 3 | Repo | GitHub → Settings → Developer settings → PAT |
| `AUTOMERGE_TOKEN` | 2 | Repo | GitHub → Settings → Developer settings → PAT |
| `DOCKER_HUB_USERNAME` | 1, 3, 5 | Repo | hub.docker.com → Username |
| `DOCKER_HUB_TOKEN` | 1, 3, 5 | Repo | hub.docker.com → Security → Access Tokens |
| `NPM_TOKEN` | 3, 5 | Repo | npmjs.com → Access Tokens → Automation |
| `PYPI_API_TOKEN` | 3, 5 | Repo | pypi.org → Account Settings → API tokens |
| `SONATYPE_USERNAME` | 3, 5 | Repo | oss.sonatype.org → User Token |
| `SONATYPE_PASSWORD` | 3, 5 | Repo | oss.sonatype.org → User Token |
| `GPG_PRIVATE_KEY` | 3 | Repo | `gpg --armor --export-secret-keys \| base64` |
| `GPG_PASSPHRASE` | 3 | Repo | Passphrase de tu clave GPG |
| `SNYK_TOKEN` | 4, 5 | Repo | app.snyk.io → Account Settings → API Token |
| `SONAR_TOKEN` | 1 | Repo | sonarcloud.io → My Account → Security |
| `CODECOV_TOKEN` | 1 | Repo | app.codecov.io → Settings |
| `INFRACOST_API_KEY` | 4 | Repo | infracost.io → API Keys |
| `DATADOG_API_KEY` | 4 | Repo | app.datadoghq.com → Organization Settings |
| `SLACK_WEBHOOK_URL` | Todos | Repo | Slack API → Incoming Webhooks |
| `DISCORD_WEBHOOK_URL` | 1 | Repo | Discord → Configuración → Webhooks |
| `TEAMS_WEBHOOK_URL` | 1 | Repo | Teams → Canal → Conectores |
| `E2E_TEST_USER` | 6 | Environment | Usuario de prueba en el entorno |
| `E2E_TEST_PASSWORD` | 6 | Environment | Contraseña del usuario de prueba |
| `KUBECONFIG_STAGING` | 6 | Repo | `kubectl config view --minify --flatten \| base64` |
| `AWS_ROLE_ARN` | 4, 7 | Environment | IAM → Roles → ARN del rol OIDC |
| `AZURE_CLIENT_ID` | 7 | Environment | Azure AD → App Registrations |
| `AZURE_TENANT_ID` | 7 | Environment | Azure AD → Properties |
| `AZURE_SUBSCRIPTION_ID` | 7 | Environment | Azure Portal → Subscriptions |
| `GCP_WORKLOAD_IDENTITY_PROVIDER` | 7 | Environment | GCP → IAM → Workload Identity |
| `GCP_SERVICE_ACCOUNT` | 7 | Environment | GCP → IAM → Service Accounts |
| `ANSIBLE_SSH_PRIVATE_KEY` | 7 | Environment | Clave privada SSH para los servidores |
| `ANSIBLE_VAULT_PASSWORD` | 7 | Environment | Contraseña de Ansible Vault |
| `NETLIFY_AUTH_TOKEN` | 8 | Repo | netlify.com → User Settings → Applications |
| `NETLIFY_SITE_ID` | 8 | Repo | netlify.com → Site Settings → General |
| `VERCEL_TOKEN` | 8 | Repo | vercel.com → Settings → Tokens |
| `VERCEL_ORG_ID` | 8 | Repo | vercel.com → Settings → General |
| `VERCEL_PROJECT_ID` | 8 | Repo | Project → Settings → General |

---

## 📊 Resumen de la Suite

| Categoría | Archivos | Líneas | Herramientas cubiertas |
|-----------|---------|--------|----------------------|
| CI/CD Principal | 1 | 1,313 | Node, Python, Java, Go, Docker |
| Dependencias | 3 | 996 | Dependabot, Renovate |
| Releases | 2 | 1,542 | npm, PyPI, Maven, Docker Hub |
| Seguridad IaC + Métricas | 2 | 1,504 | tfsec, Checkov, Trivy, DORA |
| Security Scans | 2 | 1,864 | Snyk, CodeQL, OSSAR, AWS/GCP/Azure |
| Testing | 3 | 2,222 | Playwright, k6, Chaos Mesh |
| IaC Deploy | 4 | 2,128 | Terraform, CloudFormation, Ansible |
| Documentación | 2 | 1,532 | Docusaurus, MkDocs, OpenAPI |
| **TOTAL** | **32** | **20,120** | **40+ herramientas** |

---

*Suite DevOps GitHub Actions — Generada con Claude Sonnet 4.6*  
*Para reportar problemas o sugerir mejoras, abrir un issue.*
