# Guía de Implementación: Escaneos de Seguridad Programados y Verificación de Secretos

> **Versión:** 1.0.0 | **Módulo 5 de la Suite DevOps**

---

## Tabla de Contenidos

1. [Configuración de Escaneos Programados](#1-configuración-de-escaneos-programados)
2. [Tokens y Credenciales para Herramientas de Seguridad](#2-tokens-y-credenciales-para-herramientas-de-seguridad)
3. [Configuración de Secrets y Variables en GitHub](#3-configuración-de-secrets-y-variables-en-github)
4. [Integración con el Pipeline CI/CD Principal](#4-integración-con-el-pipeline-cicd-principal)
5. [Interpretación de Resultados y Gestión de Vulnerabilidades](#5-interpretación-de-resultados-y-gestión-de-vulnerabilidades)
6. [Implementación de la Verificación de Secretos](#6-implementación-de-la-verificación-de-secretos)
7. [Solución de Problemas Comunes](#7-solución-de-problemas-comunes)

---

## 1. Configuración de Escaneos Programados

### 1.1 Elegir la Frecuencia Adecuada

La frecuencia del escaneo depende del perfil de riesgo del proyecto:

| Tipo de proyecto | Frecuencia recomendada | Cron |
|-----------------|----------------------|------|
| App pública con datos sensibles | Diario | `0 6 * * *` |
| API interna con autenticación | 2 veces/semana | `0 6 * * 1,4` |
| Librería open source | Semanal | `0 6 * * 1` |
| Proyecto interno sin datos críticos | Mensual | `0 6 1 * *` |
| Infraestructura crítica (fintech, salud) | Cada 12h | `0 */12 * * *` |

**Consideraciones importantes:**

- Los crons de GitHub Actions usan **UTC estrictamente**. Lima (Perú) está en UTC-5 (UTC-4 en verano). Un cron `0 6 * * 1` ejecuta a las 01:00 hora Lima.
- GitHub puede retrasar hasta **15-30 minutos** la ejecución de workflows por schedule en momentos de alta carga.
- Los workflows con schedule **no se ejecutan si no hay commits** en los últimos 60 días (GitHub deshabilita el schedule en repos inactivos).

**Verificar que el schedule está activo:**

```bash
# Via GitHub CLI
gh workflow list --repo <ORG>/<REPO>
gh workflow view "scheduled-security-scan.yml" --repo <ORG>/<REPO>
```

### 1.2 Estructura del Cron en GitHub Actions

```yaml
# Formato: minuto hora día-del-mes mes día-de-la-semana
# Rangos:   0-59  0-23  1-31        1-12 0-7 (0 y 7 = domingo)

# Cada día laborable a las 09:00 UTC:
- cron: "0 9 * * 1-5"

# Cada hora entre las 08:00 y 18:00 UTC los días laborables:
- cron: "0 8-18 * * 1-5"

# Dos veces al día: 06:00 y 18:00 UTC:
- cron: "0 6,18 * * *"

# Cada primer lunes del mes:
- cron: "0 6 1-7 * 1"
```

### 1.3 Activar el Escaneo Manualmente

Para ejecutar el escaneo bajo demanda sin esperar al schedule:

```bash
# Via GitHub CLI
gh workflow run scheduled-security-scan.yml \
  --repo <ORG>/<REPO> \
  --field scan_branch=main \
  --field severity_filter=CRITICAL,HIGH,MEDIUM \
  --field run_codeql=true

# Ver el resultado
gh run list --workflow=scheduled-security-scan.yml --repo <ORG>/<REPO> --limit 5
gh run view --log <RUN_ID>
```

También desde la UI: **Actions → "🔍 Scheduled Security Scan" → "Run workflow"**.

---

## 2. Tokens y Credenciales para Herramientas de Seguridad

### 2.1 Snyk

**Crear cuenta y obtener token:**

```
1. Ir a app.snyk.io → Register with GitHub
2. Account Settings → General → Auth Token → Copy
3. En GitHub: Settings → Secrets → SNYK_TOKEN = snyk_xxxxxxxxxxxx
```

**Obtener el Organization slug:**

```
1. app.snyk.io → seleccionar la organización
2. La URL será: app.snyk.io/org/MI-ORG-SLUG
3. En GitHub: Settings → Secrets → SNYK_ORG = mi-org-slug
```

**Verificar que el token funciona:**

```bash
# Instalar Snyk CLI
npm install -g snyk

# Autenticar
snyk auth TU_TOKEN

# Verificar identidad
snyk whoami

# Test de escaneo en el proyecto
snyk test --severity-threshold=high
```

**Niveles de plan de Snyk:**

| Plan | Proyectos | Historial | Integrations |
|------|-----------|-----------|-------------|
| Free | 200 tests/mes | 7 días | GitHub, GitLab |
| Team | Ilimitado | 90 días | Jira, Slack, etc. |
| Business | Ilimitado | 1 año | SIEM, SSO |

Para la mayoría de proyectos open source, el **plan gratuito** es suficiente.

### 2.2 Trivy

Trivy **no requiere token** para escanear imágenes públicas o el filesystem. Para imágenes privadas en Docker Hub:

```bash
# El token de Docker Hub ya configurado para el pipeline CI/CD funciona aquí
# Trivy usa las credenciales de Docker configuradas en el sistema
docker login -u $DOCKER_HUB_USERNAME -p $DOCKER_HUB_TOKEN

# Trivy las detecta automáticamente después del login
trivy image mi-org/mi-imagen:latest
```

**Base de datos de vulnerabilidades de Trivy:**

Trivy descarga automáticamente su base de datos de CVEs al ejecutarse. En entornos con acceso restringido a internet, puede pre-descargarse:

```bash
# Pre-descargar la base de datos (útil para entornos offline)
trivy image --download-db-only

# Usar base de datos local (sin acceso a internet durante el escaneo)
trivy image --skip-db-update mi-imagen:latest
```

**Archivo `.trivyignore` para suprimir falsos positivos:**

```
# .trivyignore
# Formato: CVE-ID o ID de misconfiguration
# Un CVE por línea; se pueden añadir comentarios con #

# Ejemplo: ignorar una CVE sin parche disponible en la imagen base
# (documentar siempre el motivo y la fecha de revisión)
CVE-2023-45853   # 2024-03-15: zlib sin parche en Alpine 3.18, monitorear próxima versión
CVE-2023-52426   # 2024-03-15: libexpat, no exploitable en nuestro contexto

# Ignorar misconfigurations de Dockerfile conocidas y justificadas
DS001            # Imagen base sin tag específico: usamos digest sha256 en producción
```

### 2.3 CodeQL

CodeQL está **integrado en GitHub** y no requiere tokens adicionales. Solo necesitas:

1. El repositorio debe ser **público** O tener **GitHub Advanced Security** (plan Team/Enterprise para repos privados).
2. El permiso `security-events: write` en el workflow (ya incluido).
3. En repos privados con plan Free: CodeQL no está disponible. Usar tfsec/trivy como alternativas.

**Verificar que CodeQL está disponible:**

```
Settings → Code security → Code scanning → Set up → Advanced
```

Si ves el botón "Set up", CodeQL está disponible. Si no aparece, el plan no lo soporta.

### 2.4 FOSSA (Licencias)

```bash
# Obtener API key: app.fossa.com → Settings → Integrations → API

# Instalar CLI
curl -H 'Cache-Control: no-cache' \
  https://raw.githubusercontent.com/fossas/fossa-cli/master/install-latest.sh | bash

# Verificar proyecto
fossa analyze
fossa test
```

---

## 3. Configuración de Secrets y Variables en GitHub

### 3.1 Secrets para `scheduled-security-scan.yml`

```
Settings → Secrets and variables → Actions → New repository secret
```

| Secret | Requerido | Descripción |
|--------|----------|-------------|
| `SNYK_TOKEN` | Opcional | Token de API de Snyk |
| `SNYK_ORG` | Opcional | Slug de organización de Snyk |
| `DOCKER_HUB_USERNAME` | Opcional* | Usuario de Docker Hub |
| `DOCKER_HUB_TOKEN` | Opcional* | Token de Docker Hub |
| `SLACK_WEBHOOK_URL` | Recomendado | URL del webhook de Slack |
| `FOSSA_API_KEY` | Opcional | API key de FOSSA para licencias |

*Requerido solo si `IMAGE_NAME` apunta a una imagen privada.

### 3.2 Variables para `scheduled-security-scan.yml`

```
Settings → Secrets and variables → Actions → Variables → New repository variable
```

| Variable | Valor por defecto | Descripción |
|----------|------------------|-------------|
| `IMAGE_NAME` | *(vacío)* | Nombre de la imagen Docker a escanear (ej: `mi-org/mi-app`) |
| `IMAGE_REGISTRY` | `docker.io` | Registry de la imagen |
| `TRIVY_SEVERITY` | `CRITICAL,HIGH` | Severidades a reportar |
| `TRIVY_IGNORE_UNFIXED` | `true` | Ignorar CVEs sin parche |
| `SCAN_BRANCH` | `main` | Rama a escanear |
| `NOTIFY_SLACK` | `true` | Activar notificaciones Slack |
| `VULN_THRESHOLD_CRITICAL` | `0` | Nº de CVEs críticos que disparan alerta |
| `VULN_THRESHOLD_HIGH` | `5` | Nº de CVEs high que disparan alerta |

### 3.3 Secrets para `secret-rotation-check.yml`

| Secret | Propósito | Cómo obtenerlo |
|--------|----------|----------------|
| `GITHUB_CHECK_PAT` | Verificar validez de PATs | Cuenta de servicio → Settings → Tokens |
| `RELEASE_TOKEN` | Verificar token de releases | El PAT ya configurado para releases |
| `AUTOMERGE_TOKEN` | Verificar token de auto-merge | El PAT ya configurado para Dependabot |
| `AWS_ACCESS_KEY_ID` | Verificar expiración de AWS keys | Ya configurado en el pipeline |
| `AWS_SECRET_ACCESS_KEY` | Verificar expiración de AWS keys | Ya configurado en el pipeline |
| `GCP_SA_KEY` | Verificar claves de GCP | GCP Console → IAM → Service Accounts → Keys |
| `AZURE_CLIENT_ID` | Verificar Azure SP | Azure Portal → App Registrations |
| `AZURE_CLIENT_SECRET` | Verificar Azure SP | Azure Portal → App Registrations → Secrets |
| `AZURE_TENANT_ID` | Verificar Azure SP | Azure Portal → Properties |
| `DOCKER_HUB_TOKEN` | Verificar token Docker Hub | hub.docker.com → Security |
| `NPM_TOKEN` | Verificar token npm | npmjs.com → Access Tokens |
| `PYPI_API_TOKEN` | Verificar token PyPI | pypi.org → API tokens |
| `SONATYPE_USERNAME` | Verificar Sonatype | oss.sonatype.org → User Token |
| `SONATYPE_PASSWORD` | Verificar Sonatype | oss.sonatype.org → User Token |
| `INFRACOST_API_KEY` | Verificar Infracost | infracost.io → API Keys |
| `SLACK_WEBHOOK_URL` | Enviar alertas | Slack API → Webhooks |

---

## 4. Integración con el Pipeline CI/CD Principal

### 4.1 Diferencias con los Escaneos en ci-cd.yml

| Característica | ci-cd.yml (en push) | scheduled-security-scan.yml |
|---------------|--------------------|-----------------------------|
| Cuándo ejecuta | En cada push/PR | Periódicamente (cron) |
| Qué escanea | El código del commit | La rama main siempre |
| Imagen Docker | Dockerfile (no publicado) | Imagen publicada en registry |
| Snyk modo | `test` (solo reporta) | `monitor` (sube a plataforma) |
| CodeQL | Quick scan | `security-and-quality` (profundo) |
| Objetivo | Bloquear código malo | Detectar CVEs post-publicación |

### 4.2 Unificar Resultados en el Security Tab

Todos los workflows que suben SARIF con `github/codeql-action/upload-sarif@v3` aparecen en el mismo Security tab, separados por `category`. Esto permite ver un inventario unificado:

```
Security → Code scanning → All alerts
→ Filtrar por: Tool = "Trivy" (o "Snyk", "CodeQL")
→ Filtrar por: Branch = "main"
→ Filtrar por: Severity = "Critical"
```

Para diferenciar las alertas del scan programado de las del CI:

```yaml
# En ci-cd.yml:
category: "trivy-fs-ci"           # Escaneo en cada push

# En scheduled-security-scan.yml:
category: "trivy-fs-scheduled"    # Escaneo programado
```

Esto crea dos "categorías" distintas en el Security tab, facilitando el filtrado.

### 4.3 Evitar Duplicación de Alertas

Si el mismo CVE aparece tanto en el escaneo de CI como en el programado, GitHub consolida automáticamente las alertas por CVE ID — no crea duplicados. Las dos categorías simplemente "confirman" la misma alerta desde dos fuentes.

### 4.4 Integrar Hallazgos del Scheduled Scan en el Dashboard de Métricas

Modifica `pipeline-metrics.yml` para incluir el conteo de vulnerabilidades activas:

```yaml
# Añadir al job collect-metrics en pipeline-metrics.yml:
- name: "📊 Métricas de seguridad"
  run: |
    # Contar alertas de Code Scanning abiertas
    OPEN_ALERTS=$(gh api \
      "repos/${{ github.repository }}/code-scanning/alerts" \
      --jq '[.[] | select(.state == "open")] | length' \
      2>/dev/null || echo "0")

    CRITICAL_ALERTS=$(gh api \
      "repos/${{ github.repository }}/code-scanning/alerts" \
      --jq '[.[] | select(.state == "open" and .rule.severity == "critical")] | length' \
      2>/dev/null || echo "0")

    echo "Alertas de seguridad abiertas: $OPEN_ALERTS"
    echo "Alertas críticas: $CRITICAL_ALERTS"
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## 5. Interpretación de Resultados y Gestión de Vulnerabilidades

### 5.1 Leer el Security Tab de GitHub

Navegar a: **Security → Code scanning alerts**

Cada alerta muestra:
- **Regla**: ID del CVE o nombre de la misconfiguration
- **Severidad**: Critical, High, Medium, Low
- **Herramienta**: Trivy, Snyk, CodeQL, etc.
- **Archivo y línea**: Dónde en el código
- **Estado**: Open, Dismissed, Fixed
- **Primera vez detectada**: Cuándo apareció
- **Ramas afectadas**: En qué ramas existe

**Flujo recomendado de triaje:**

```
1. Revisar alertas de severidad CRITICAL primero
2. Para cada alerta:
   a. ¿Es real? → Verificar en el NVD (nvd.nist.gov/vuln)
   b. ¿Está en el camino de ejecución de mi app?
   c. ¿Hay una actualización disponible?
3. Acciones posibles:
   - Fix: actualizar la dependencia
   - Dismiss: si es falso positivo (documentar el motivo)
   - Accept risk: si no hay parche y el riesgo es aceptado
```

### 5.2 Descartar Alertas (Dismiss) con Justificación

```bash
# Via GitHub CLI - marcar una alerta como "won't fix"
gh api \
  -X PATCH \
  "repos/<ORG>/<REPO>/code-scanning/alerts/<ALERT_NUMBER>" \
  --field state=dismissed \
  --field dismissed_reason=used_in_tests \
  --field dismissed_comment="Esta dependencia solo existe en tests, no en producción"

# Razones válidas:
# won't_fix         → El fix no está disponible o el riesgo es aceptable
# used_in_tests     → Solo presente en entorno de tests
# false_positive    → No es una vulnerabilidad real en este contexto
```

### 5.3 Configurar Alertas Solo para Vulnerabilidades Nuevas

El workflow de notificación ya filtra por umbral de severidad, pero para recibir alertas **solo cuando aparecen CVEs nuevos** (no reportar los mismos cada semana):

```yaml
# En el job notify de scheduled-security-scan.yml,
# añadir comparación con el run anterior:
- name: "🆕 Detectar CVEs nuevos vs semana anterior"
  run: |
    # Obtener alertas abiertas actuales
    CURRENT_ALERTS=$(gh api \
      "repos/${{ github.repository }}/code-scanning/alerts" \
      --jq '[.[] | select(.state == "open") | .rule.id]' 2>/dev/null)

    # Comparar con la semana anterior (artefacto del run previo)
    # Si tienes artefactos previos, descargarlos y comparar
    # Para una implementación simple: solo alertar si hay NUEVAS alertas
    # que no existían en el último run
    echo "Alertas actuales: $(echo "$CURRENT_ALERTS" | jq 'length')"
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 5.4 Crear Issues de GitHub Automáticamente para CVEs Críticos

```yaml
# Añadir al final del job notify:
- name: "🐛 Crear issue para CVEs críticos"
  if: steps.consolidate.outputs.total_critical > '0'
  uses: actions/github-script@v7
  with:
    script: |
      const critical = '${{ steps.consolidate.outputs.total_critical }}';
      const runUrl = '${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}';

      // Verificar si ya existe un issue abierto para este escaneo
      const existingIssues = await github.rest.issues.listForRepo({
        owner: context.repo.owner,
        repo: context.repo.repo,
        labels: 'security,critical-vulnerability',
        state: 'open'
      });

      if (existingIssues.data.length === 0) {
        await github.rest.issues.create({
          owner: context.repo.owner,
          repo: context.repo.repo,
          title: `🔴 Security: ${critical} CVE(s) crítico(s) detectado(s) — ${new Date().toISOString().split('T')[0]}`,
          body: `## Vulnerabilidades Críticas Detectadas\n\n` +
                `El escaneo programado detectó **${critical}** CVE(s) de severidad CRITICAL.\n\n` +
                `**Acción requerida:** Revisar y corregir antes del próximo release.\n\n` +
                `🔗 [Ver detalles en el Security Tab](/${context.repo.owner}/${context.repo.repo}/security/code-scanning)\n` +
                `🔗 [Logs del escaneo](${runUrl})`,
          labels: ['security', 'critical-vulnerability', 'priority-high']
        });
      }
```

---

## 6. Implementación de la Verificación de Secretos

### 6.1 Arquitectura de la Verificación

El workflow `secret-rotation-check.yml` usa un enfoque pragmático:

```
┌─────────────────────────────────────────────────────────┐
│ Principio fundamental:                                   │
│ No podemos LEER el valor de un secret de GitHub.        │
│ Solo podemos USAR el secret en una llamada de API       │
│ y verificar si la respuesta indica validez.             │
└─────────────────────────────────────────────────────────┘

Para cada secret/credencial:
  1. Hacer una llamada de SOLO LECTURA a la API correspondiente
  2. Verificar el código HTTP de respuesta:
     200/201 → válido
     401/403 → inválido o expirado
     otros   → incertidumbre (reportar como advertencia)
  3. Si la API expone fecha de expiración → comparar con umbral
  4. Si no → reportar solo validez/invalidez
```

### 6.2 Configurar Cuenta de Servicio para Verificación

Para que el workflow pueda verificar los PATs de GitHub sin comprometer la seguridad:

**Paso 1:** Crear una cuenta de servicio en GitHub (ej: `mi-org-bot`).

**Paso 2:** Generar un PAT de solo lectura en esa cuenta:

```
Cuenta bot → Settings → Developer settings → Fine-grained tokens → Generate new token

Permissions requeridos (mínimos):
- Account permissions → Personal access tokens → Read-only
- Repository permissions → Metadata → Read-only
```

**Paso 3:** Guardar como secret `GITHUB_CHECK_PAT`.

Este PAT solo se usa para llamadas de verificación; no tiene permisos de escritura, así que si se comprometiera el daño sería mínimo.

### 6.3 Política de Rotación Recomendada por Tipo de Credencial

| Credencial | Vida útil máxima | Herramienta de rotación |
|-----------|-----------------|------------------------|
| GitHub PATs (clásicos) | 90 días | Manual |
| GitHub Fine-grained tokens | 1 año máx. | Manual |
| AWS Access Keys | 90 días (CIS Benchmark) | `aws-rotate-key` |
| GCP SA Keys | 90 días | `gcloud iam service-accounts keys create` |
| Azure SP Secrets | 1 año | `az ad app credential reset` |
| Docker Hub tokens | Sin expiración* | Manual (recomendado: 1 año) |
| npm tokens (Automation) | Sin expiración* | Manual (recomendado: 1 año) |
| Snyk tokens | Sin expiración* | Manual |

*Sin expiración automática: definir política interna de rotación.

### 6.4 Automatizar la Rotación de AWS Keys (Avanzado)

Si quieres ir más allá de la detección y automatizar la rotación de AWS keys:

```bash
#!/bin/bash
# script: rotate-aws-key.sh
# EJECUTAR MANUALMENTE, no en CI directamente (riesgo de lockout)

USER_NAME="mi-cuenta-de-servicio"
REGION="us-east-1"

# 1. Crear nueva key
NEW_KEY=$(aws iam create-access-key --user-name "$USER_NAME" --output json)
NEW_KEY_ID=$(echo "$NEW_KEY" | jq -r '.AccessKey.AccessKeyId')
NEW_SECRET=$(echo "$NEW_KEY" | jq -r '.AccessKey.SecretAccessKey')

echo "Nueva key creada: $NEW_KEY_ID"

# 2. Actualizar el secret en GitHub (requiere gh auth con permisos de admin)
gh secret set AWS_ACCESS_KEY_ID \
  --body "$NEW_KEY_ID" \
  --repo MI_ORG/MI_REPO

gh secret set AWS_SECRET_ACCESS_KEY \
  --body "$NEW_SECRET" \
  --repo MI_ORG/MI_REPO

echo "Secrets actualizados en GitHub"

# 3. Verificar que la nueva key funciona
AWS_ACCESS_KEY_ID="$NEW_KEY_ID" \
AWS_SECRET_ACCESS_KEY="$NEW_SECRET" \
aws sts get-caller-identity --region "$REGION"

# 4. Desactivar la key anterior
OLD_KEY_ID="LA_KEY_ANTERIOR"  # Obtener del estado anterior
aws iam update-access-key \
  --access-key-id "$OLD_KEY_ID" \
  --status Inactive \
  --user-name "$USER_NAME"

echo "Key anterior desactivada: $OLD_KEY_ID"

# 5. Después de verificar que todo funciona, eliminar la key anterior
# aws iam delete-access-key --access-key-id "$OLD_KEY_ID" --user-name "$USER_NAME"
echo "MANUAL: eliminar la key anterior una vez verificado: $OLD_KEY_ID"
```

### 6.5 Dashboard Mínimo de Estado de Secretos

Para tener visibilidad del estado de todos los secretos, crear un issue o wiki page que se actualice automáticamente:

```yaml
# Añadir al job notify de secret-rotation-check.yml:
- name: "📋 Actualizar wiki con estado de secretos"
  if: always()
  run: |
    STATUS_TABLE="# Estado de Secretos — $(date '+%Y-%m-%d')

    | Secret | Estado | Última verificación |
    |--------|--------|-------------------|
    | GITHUB_CHECK_PAT | ${{ needs.check-github-tokens.outputs.status }} | $(date '+%Y-%m-%d') |
    | AWS credentials | ${{ needs.check-cloud-credentials.outputs.status }} | $(date '+%Y-%m-%d') |
    | DOCKER_HUB_TOKEN | ${{ needs.check-docker-tokens.outputs.status }} | $(date '+%Y-%m-%d') |
    | SNYK_TOKEN | ${{ needs.check-api-keys.outputs.status }} | $(date '+%Y-%m-%d') |
    | NPM_TOKEN | ${{ needs.check-api-keys.outputs.status }} | $(date '+%Y-%m-%d') |

    > Actualizado automáticamente por workflow secret-rotation-check.yml
    "

    # Guardar como artefacto para consulta
    echo "$STATUS_TABLE" > /tmp/secrets-status.md

- name: "💾 Subir reporte de estado"
  uses: actions/upload-artifact@v4
  with:
    name: secrets-status-${{ github.run_id }}
    path: /tmp/secrets-status.md
    retention-days: 30
```

---

## 7. Solución de Problemas Comunes

### 7.1 Error de Autenticación en Trivy al Escanear Imágenes Privadas

**Síntoma:**
```
FATAL   image scan error: unable to initialize a image scanner: ...
unauthorized: incorrect username or password
```

**Causas y soluciones:**

```yaml
# Causa 1: El login de Docker no se hizo ANTES del step de Trivy
# Solución: Mover el step de docker login ANTES de trivy-action

- name: "🔐 Login Docker Hub"  # ← Este step PRIMERO
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKER_HUB_USERNAME }}
    password: ${{ secrets.DOCKER_HUB_TOKEN }}

- name: "🔬 Trivy scan"        # ← Trivy DESPUÉS del login
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: image
    image-ref: "docker.io/mi-org/mi-app:latest"
```

```bash
# Causa 2: El token de Docker Hub expiró o fue revocado
# Diagnóstico:
curl -sf \
  -H "Content-Type: application/json" \
  -d '{"username":"MI_USER","password":"MI_TOKEN"}' \
  "https://hub.docker.com/v2/users/login" | jq '.token'
# Si retorna null o error: rotar el token en hub.docker.com
```

```yaml
# Causa 3: Imagen en GHCR (no en Docker Hub)
# Solución: usar el token específico de GHCR
- name: "🔐 Login GHCR"
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}  # Token automático de Actions

- name: "🔬 Trivy scan GHCR"
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: "ghcr.io/mi-org/mi-app:latest"
```

### 7.2 Falsos Positivos en Trivy: Cómo Suprimirlos

**Identificar un falso positivo:**

```bash
# Verificar la CVE en el NVD (National Vulnerability Database)
open "https://nvd.nist.gov/vuln/detail/CVE-2023-XXXXX"

# Verificar si la CVE afecta realmente a tu uso de la librería:
# - ¿Usas la función/módulo vulnerable?
# - ¿El vector de ataque aplica a tu contexto (red, local, físico)?
# - ¿Hay una versión sin parche disponible?
```

**Suprimir con `.trivyignore`:**

```
# .trivyignore
# =========================================================
# Falsos positivos documentados
# =========================================================
# Formato: CVE-ID  # Fecha: YYYY-MM-DD | Motivo | Revisar: YYYY-MM-DD
# =========================================================

CVE-2023-45853
# Fecha: 2024-01-15
# Motivo: zlib en imagen Alpine 3.18 — no hay parche disponible. El vector de
# ataque (parsing de PNG malicioso) no aplica: la app no procesa imágenes externas.
# Revisar: 2024-06-15 (cuando salga Alpine 3.19)

CVE-2023-52426
# Fecha: 2024-01-15
# Motivo: libexpat 2.5.0 — CVE requiere XML con entidades externas. La app no
# procesa XML de fuentes no confiables.
# Revisar: 2024-07-15
```

**Suprimir inline en el código (para misconfigurations de Trivy en IaC):**

```hcl
# terraform/main.tf
resource "aws_s3_bucket" "logs" {
  bucket = "mi-bucket-de-logs"

  # trivy:ignore:AVD-AWS-0089
  # Justificación: El bucket de logs no necesita versioning ya que
  # los logs son inmutables por política de retención.
}
```

### 7.3 Limitaciones de la API de GitHub para Expiración de PATs

**Limitación fundamental:** GitHub no expone la fecha de expiración de un PAT en ningún endpoint público de la API. La API solo permite verificar **si el token es válido**, no cuándo expira.

**Alternativas para rastrear expiración:**

```yaml
# Opción 1: Almacenar la fecha de creación como variable (no secret)
# Al crear un PAT, registrar la fecha en una variable de repositorio:
gh variable set GITHUB_CHECK_PAT_CREATED \
  --body "2024-01-15" \
  --repo MI_ORG/MI_REPO

# El workflow compara con la política de rotación:
- name: "Verificar antigüedad del PAT"
  run: |
    CREATED="${{ vars.GITHUB_CHECK_PAT_CREATED }}"
    CREATED_EPOCH=$(date -d "$CREATED" +%s)
    TODAY=$(date +%s)
    AGE_DAYS=$(( (TODAY - CREATED_EPOCH) / 86400 ))
    MAX_AGE=90  # Política: rotar cada 90 días

    echo "PAT creado: $CREATED ($AGE_DAYS días)"
    if [ "$AGE_DAYS" -gt "$MAX_AGE" ]; then
      echo "⚠️ PAT supera los $MAX_AGE días — rotar"
    fi
```

```yaml
# Opción 2: Usar Fine-grained tokens con fecha de expiración explícita
# Los fine-grained PATs SÍ tienen fecha de expiración configurable (max 1 año).
# GitHub envía un email de aviso 7 días antes de la expiración.
# El workflow solo necesita verificar la validez.
```

```bash
# Opción 3: GitHub Apps en lugar de PATs
# Las GitHub Apps tienen tokens de corta duración (1 hora) que se renuevan
# automáticamente. No necesitan rotación manual.
# Documentación: docs.github.com/en/apps
```

### 7.4 El Workflow Scheduled No Se Ejecuta

**Causa más común:** GitHub deshabilita los workflows con schedule después de **60 días sin actividad** (sin commits) en el repositorio.

```bash
# Verificar si el workflow está habilitado
gh workflow list --repo MI_ORG/MI_REPO

# Si aparece como "disabled (inactivity)":
gh workflow enable scheduled-security-scan.yml --repo MI_ORG/MI_REPO

# O desde la UI: Actions → workflow → Enable workflow
```

**Otras causas:**

```yaml
# La rama por defecto del repo debe coincidir con donde está el workflow
# Si tu rama principal es 'master' en lugar de 'main', el cron no se ejecuta
# desde 'main'. Cambiar en GitHub Settings → Branches → Default branch.

# Los workflows con schedule solo leen la versión del workflow
# en la RAMA POR DEFECTO del repositorio.
```

### 7.5 CodeQL Falla con "No code found to analyze"

```yaml
# Causa: El lenguaje especificado no coincide con el código del repo
# Solución: Usar detección automática de lenguajes

- name: "🔭 CodeQL — Initialize (auto-detect)"
  uses: github/codeql-action/init@v3
  with:
    # Dejar en blanco para auto-detectar
    # languages: javascript-typescript  ← COMENTAR ESTO
    config-file: .github/codeql-config.yml  # Usar config externa

# .github/codeql-config.yml:
# name: "Custom CodeQL Config"
# languages:
#   - javascript
# paths-ignore:
#   - "node_modules"
#   - "dist"
#   - "build"
#   - "**/*.test.js"
#   - "**/*.spec.js"
```

### 7.6 Snyk Falla con "Could not find lockfile"

```bash
# Causa: Snyk necesita los lockfiles para analizar el árbol de dependencias
# Solución: Asegurarse de que el lockfile está commiteado en el repo

# Para npm:
npm install  # genera package-lock.json
git add package-lock.json
git commit -m "chore: add lockfile for Snyk analysis"

# Para Python:
pip install pip-tools
pip-compile requirements.in -o requirements.txt  # genera el lockfile
git add requirements.txt

# Si el proyecto no usa lockfiles, usar --skip-unresolved:
snyk test --skip-unresolved --all-projects
```

---

## Apéndice: Tabla de Herramientas del Módulo 5

| Herramienta | Workflow | Licencia | Docs |
|-------------|---------|---------|------|
| Trivy (FS) | scheduled-security-scan | Apache 2.0 | aquasecurity.github.io/trivy |
| Trivy (Image) | scheduled-security-scan | Apache 2.0 | aquasecurity.github.io/trivy |
| Snyk | scheduled-security-scan | Propietario (Free tier) | docs.snyk.io |
| CodeQL | scheduled-security-scan | Propietario (gratuito) | codeql.github.com |
| OSSAR | scheduled-security-scan | MIT | github.com/github/ossar-action |
| FOSSA | scheduled-security-scan | Propietario (Free tier) | docs.fossa.com |
| GitHub API | secret-rotation-check | N/A | docs.github.com/rest |
| AWS CLI | secret-rotation-check | Apache 2.0 | docs.aws.amazon.com/cli |
| gcloud CLI | secret-rotation-check | Apache 2.0 | cloud.google.com/sdk/docs |
| Azure CLI | secret-rotation-check | MIT | docs.microsoft.com/cli/azure |

---

*Guía generada para Scheduled Security Scans & Secret Rotation Check v1.0.0 — Módulo 5 de la Suite DevOps*

*Suite completa: ci-cd.yml → dependabot.yml → release-on-demand.yml → iac-security.yml → scheduled-security-scan.yml*
