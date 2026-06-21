# Guía de Implementación: Gestión Automática de Dependencias con Dependabot

> **Versión:** 1.0.0 | **Actualizado:** 2025

[![Dependabot Updates](https://github.com/<ORG>/<REPO>/actions/workflows/dependabot/badge.svg)](https://github.com/<ORG>/<REPO>/network/updates)

---

## Tabla de Contenidos

1. [Configuración Inicial del Repositorio](#1-configuración-inicial-del-repositorio)
2. [Habilitación de Dependabot en GitHub](#2-habilitación-de-dependabot-en-github)
3. [Configuración del Token y Permisos para Auto-merge](#3-configuración-del-token-y-permisos-para-auto-merge)
4. [Configuración de Protección de Rama](#4-configuración-de-protección-de-rama)
5. [Personalización de las Reglas de Auto-merge](#5-personalización-de-las-reglas-de-auto-merge)
6. [Renovate como Alternativa a Dependabot](#6-renovate-como-alternativa-a-dependabot)
7. [Monitoreo y Solución de Problemas](#7-monitoreo-y-solución-de-problemas)
8. [Integración con Workflows de CI/CD Existentes](#8-integración-con-workflows-de-cicd-existentes)

---

## 1. Configuración Inicial del Repositorio

### 1.1 Estructura de Archivos

Copia los archivos generados a tu repositorio con la siguiente estructura:

```
mi-proyecto/
├── .github/
│   ├── dependabot.yml                        ← Configuración de Dependabot
│   └── workflows/
│       └── auto-merge-dependabot.yml         ← Workflow de auto-merge
├── renovate.json                             ← (Opcional) Config de Renovate
└── README.md                                 ← Añade el badge aquí
```

```bash
# Crear la estructura desde cero
mkdir -p .github/workflows

# Copiar los archivos generados
cp dependabot.yml .github/dependabot.yml
cp auto-merge-dependabot.yml .github/workflows/auto-merge-dependabot.yml
cp renovate.json renovate.json  # Opcional

# Confirmar y subir
git add .github/ renovate.json
git commit -m "chore(ci): add Dependabot config and auto-merge workflow"
git push origin main
```

### 1.2 Personalizar `dependabot.yml` para tu Proyecto

Antes de subir, edita el archivo para **habilitar solo los ecosistemas que usas** y **comentar los que no aplican**. Mantener ecosistemas innecesarios activos genera ruido de PRs vacías.

**Checklist de personalización:**

```yaml
# ✅ Ecosistemas a activar según tu proyecto:
#   Node.js/npm          → package-ecosystem: "npm"
#   Python/pip           → package-ecosystem: "pip"
#   Java/Maven           → package-ecosystem: "maven"
#   Java/Gradle          → package-ecosystem: "gradle"
#   Docker               → package-ecosystem: "docker"
#   Go                   → package-ecosystem: "gomod"
#   GitHub Actions       → package-ecosystem: "github-actions"  ← Siempre activar
```

**Ajustes mínimos obligatorios:**

| Campo | Valor a Cambiar | Por Qué |
|-------|----------------|---------|
| `timezone` | `"America/Lima"` → tu zona horaria | Los checks corren en el horario correcto |
| `target-branch` | `"main"` → tu rama principal | Si usas `master` u otra rama |
| `labels` | Añadir etiquetas de tu proyecto | Para filtrar PRs en el tablero |
| `directory` | `"/"` → ruta de tu manifiesto | Para monorepos con múltiples directorios |

### 1.3 Badge para README

Añade este badge al inicio de tu `README.md`:

```markdown
[![Dependabot Updates](https://github.com/<ORG>/<REPO>/actions/workflows/dependabot/badge.svg)](https://github.com/<ORG>/<REPO>/network/updates)
```

Reemplaza `<ORG>` y `<REPO>` con los valores de tu repositorio.

---

## 2. Habilitación de Dependabot en GitHub

### 2.1 Activación Automática

Dependabot se activa **automáticamente** al detectar el archivo `.github/dependabot.yml` en la rama por defecto del repositorio. No requiere configuración adicional en la UI para empezar.

### 2.2 Verificación en la UI de GitHub

Para confirmar que Dependabot está activo:

```
Tu repo → Insights → Dependency graph → Dependabot
```

Verás una lista de todos los ecosistemas configurados con:
- **Estado**: Activo / Deshabilitado / Error
- **Última revisión**: Cuándo Dependabot revisó por última vez
- **Próxima revisión**: Cuándo volverá a revisar según el schedule
- **PRs abiertas**: Cuántas PRs tiene Dependabot abiertas actualmente

### 2.3 Habilitar Alertas de Seguridad (Dependabot Security Updates)

Además de las actualizaciones de versión configuradas en `dependabot.yml`, Dependabot puede abrir PRs automáticamente cuando detecta vulnerabilidades conocidas (CVEs) en tus dependencias:

```
Settings → Code security → Dependabot
```

Habilitar:
- ✅ **Dependabot alerts** — Notificaciones de vulnerabilidades
- ✅ **Dependabot security updates** — PRs automáticas para cerrar vulnerabilidades
- ✅ **Dependabot version updates** — Las PRs de versión configuradas en dependabot.yml

Las **Dependabot security updates** tienen prioridad y no cuentan contra el límite de `open-pull-requests-limit`. Una razón más para que el auto-merge esté activo: las correcciones de seguridad deben llegar a producción lo antes posible.

### 2.4 Forzar una Revisión Inmediata

Si acabas de configurar Dependabot y no quieres esperar al schedule:

```
Insights → Dependency graph → Dependabot → [ecosistema] → "Check for updates"
```

O desde la API de GitHub (requiere PAT con scope `repo`):

```bash
# Disparar revisión manual de Dependabot via API
curl -X POST \
  -H "Authorization: Bearer $GITHUB_PAT" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/<ORG>/<REPO>/actions/workflows/dependabot/dispatches"
```

---

## 3. Configuración del Token y Permisos para Auto-merge

Este es el punto más complejo de la implementación. Existen dos enfoques.

### 3.1 Opción A: Solo GITHUB_TOKEN (Más Simple)

Funciona si tu protección de rama **no requiere aprobaciones de revisores** (`Required approving reviews: 0`).

**Configurar en la UI de GitHub:**

```
Settings → Actions → General → Workflow permissions
→ ● Read and write permissions  ← Seleccionar esto
→ ☑ Allow GitHub Actions to create and approve pull requests  ← Marcar esto
```

**Ventaja:** Cero configuración adicional de secrets.
**Limitación:** No puede aprobar PRs si la rama requiere aprobaciones humanas.

### 3.2 Opción B: PAT de Cuenta de Servicio (Recomendado para Producción)

Necesario cuando la protección de rama requiere al menos una aprobación de revisor.

**Paso 1: Crear una cuenta de servicio en GitHub**

Crea una cuenta separada (ej. `miorg-bot`) que actuará como el "aprobador" del bot. Esta cuenta debe:
- Ser miembro del repositorio con rol de **Collaborator** o **Write**
- No ser la misma que abrió la PR (GitHub no permite auto-aprobaciones)

**Paso 2: Generar un Personal Access Token (PAT) Clásico**

```
Cuenta del bot → Settings → Developer settings → Personal access tokens → Tokens (classic)
→ Generate new token (classic)
```

Configuración del token:
- **Note:** `Dependabot Auto-merge [nombre-repo]`
- **Expiration:** 90 days (o No expiration para cuentas de servicio dedicadas)
- **Scopes requeridos:**
  - ✅ `repo` → Acceso completo al repositorio (necesario para merge)
  - ✅ `workflow` → Si también necesitas disparar workflows

**Paso 3: Añadir el PAT como Secret en el Repositorio**

```
Settings → Secrets and variables → Actions → New repository secret
```

- **Name:** `AUTOMERGE_TOKEN`
- **Secret:** Pegar el PAT generado

**Paso 4: Actualizar el workflow para usar el PAT**

En `.github/workflows/auto-merge-dependabot.yml`, cambia los steps de aprobación y merge:

```yaml
# PASO 8: Aprobar PR
- name: "✅ Aprobar PR de Dependabot"
  run: |
    gh pr review "${{ github.event.pull_request.number }}" --approve
  env:
    GH_TOKEN: ${{ secrets.AUTOMERGE_TOKEN }}  # ← Usar PAT, no GITHUB_TOKEN

# PASO 9: Habilitar auto-merge
- name: "🔀 Habilitar auto-merge"
  run: |
    gh pr merge "${{ github.event.pull_request.number }}" --auto --squash
  env:
    GH_TOKEN: ${{ secrets.AUTOMERGE_TOKEN }}  # ← Usar PAT, no GITHUB_TOKEN
```

**Importante de seguridad:** El PAT tiene acceso completo al repo. Nunca lo imprimas en los logs ni lo uses en run: echo. El CLI de `gh` lo usa internamente sin exponerlo.

### 3.3 Opción C: GitHub App (Enterprise / Máxima Seguridad)

Para organizaciones con requisitos de seguridad más estrictos, crear una GitHub App propia es la opción más segura porque:
- Los tokens se rotan automáticamente cada hora
- Los permisos son más granulares (solo write en PRs, no en todo el repo)
- Se puede auditar cada acción de la App en el log de auditoría de la organización

```yaml
# En el workflow, usar la action de generación de tokens de App:
- name: "🔑 Obtener token de GitHub App"
  id: app-token
  uses: actions/create-github-app-token@v1
  with:
    app-id: ${{ secrets.BOT_APP_ID }}
    private-key: ${{ secrets.BOT_PRIVATE_KEY }}

- name: "✅ Aprobar PR"
  run: gh pr review "$PR_NUM" --approve
  env:
    GH_TOKEN: ${{ steps.app-token.outputs.token }}
    PR_NUM: ${{ github.event.pull_request.number }}
```

---

## 4. Configuración de Protección de Rama

La protección de rama es lo que garantiza que el auto-merge no salte los controles de calidad.

### 4.1 Crear Regla de Protección para `main`

```
Settings → Branches → Add branch protection rule
```

**Branch name pattern:** `main`

**Configuración recomendada:**

| Opción | Valor | Por Qué |
|--------|-------|---------|
| Require a pull request before merging | ✅ | Nadie pushea directo a main |
| Required approving reviews | `1` | Al menos una aprobación (del bot o humano) |
| Dismiss stale reviews | ✅ | Si hay nuevos commits, re-aprobar |
| Require review from Code Owners | Opcional | Para proyectos grandes |
| Require status checks to pass | ✅ | **CRÍTICO**: CI debe pasar antes del merge |
| Status checks que deben pasar | `lint`, `test` | Exactamente los nombres de tus jobs |
| Require branches to be up to date | ✅ | Evita merges con código desactualizado |
| **Allow auto-merge** | ✅ | **CRÍTICO**: Habilita el `gh pr merge --auto` |
| Do not allow bypassing the above settings | ✅ | Ni los admins pueden saltarse los checks |

### 4.2 Por Qué "Allow auto-merge" es Clave

El comando `gh pr merge --auto` que ejecuta el workflow NO fusiona inmediatamente. En cambio, **registra la intención de merge** en GitHub. GitHub esperará a que:
1. Todos los status checks requeridos pasen ✅
2. La PR tenga las aprobaciones necesarias ✅
3. La rama esté actualizada con `main` ✅

Solo cuando se cumplen todas las condiciones, GitHub ejecuta el merge automáticamente. Esto garantiza que el código que llega a `main` siempre pasó los controles de calidad.

---

## 5. Personalización de las Reglas de Auto-merge

### 5.1 Ajustar Qué Tipos de Actualización se Auto-mergean

En el workflow `auto-merge-dependabot.yml`, las condiciones están en los campos `if:` de cada step:

```yaml
# Configuración actual: solo patch y minor
if: |
  steps.dependabot-metadata.outputs.update-type == 'version-update:semver-patch' ||
  steps.dependabot-metadata.outputs.update-type == 'version-update:semver-minor'

# Para agregar major (NO recomendado sin revisión):
if: |
  steps.dependabot-metadata.outputs.update-type == 'version-update:semver-patch' ||
  steps.dependabot-metadata.outputs.update-type == 'version-update:semver-minor' ||
  (steps.dependabot-metadata.outputs.update-type == 'version-update:semver-major' &&
   env.AUTO_MERGE_MAJOR == 'true')
```

Cambia la variable de entorno `AUTO_MERGE_MAJOR` a `"true"` en el bloque `env:` global del workflow para habilitar el auto-merge de major (con precaución).

### 5.2 Excluir Dependencias Específicas del Auto-merge

Hay dependencias que, incluso siendo actualizaciones patch, requieren revisión manual (drivers de DB, librerías de autenticación, etc.):

```yaml
# En el workflow, añadir un step de verificación de lista negra ANTES del merge:
- name: "🚫 Verificar lista negra de dependencias"
  id: check-blocklist
  run: |
    DEPS="${{ steps.dependabot-metadata.outputs.dependency-names }}"
    # Lista de dependencias que NUNCA se auto-mergean
    BLOCKLIST=(
      "pg"
      "mongoose"
      "passport"
      "jsonwebtoken"
      "aws-sdk"
    )
    for blocked in "${BLOCKLIST[@]}"; do
      if echo "$DEPS" | grep -q "$blocked"; then
        echo "blocked=true" >> "$GITHUB_OUTPUT"
        echo "🚫 Dependencia bloqueada: $blocked (en lista negra de auto-merge)"
        exit 0
      fi
    done
    echo "blocked=false" >> "$GITHUB_OUTPUT"
    echo "✅ Ninguna dependencia en lista negra"

# Luego, en los steps de aprobación y merge, añadir esta condición:
- name: "✅ Aprobar PR"
  if: steps.check-blocklist.outputs.blocked != 'true'
  # ... resto del step
```

### 5.3 Auto-merge Solo para Ecosistemas Específicos

Si solo quieres auto-merge para GitHub Actions y npm, pero no para Docker:

```yaml
- name: "🔀 Habilitar auto-merge"
  if: |
    (steps.dependabot-metadata.outputs.package-ecosystem == 'npm' ||
     steps.dependabot-metadata.outputs.package-ecosystem == 'github_actions') &&
    (steps.dependabot-metadata.outputs.update-type == 'version-update:semver-patch' ||
     steps.dependabot-metadata.outputs.update-type == 'version-update:semver-minor')
  run: gh pr merge "${{ github.event.pull_request.number }}" --auto --squash
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 5.4 Auto-merge Solo para Actualizaciones de Seguridad

Maximizar la velocidad de respuesta a CVEs:

```yaml
- name: "🔒 Auto-merge de actualización de seguridad"
  # Se auto-mergea si hay una alerta de seguridad, independientemente del tipo de update
  if: steps.dependabot-metadata.outputs.alert-state == 'OPEN'
  run: |
    echo "🔒 Actualizacion de seguridad detectada (CVSS: ${{ steps.dependabot-metadata.outputs.cvss }})"
    gh pr merge "${{ github.event.pull_request.number }}" --auto --squash
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## 6. Renovate como Alternativa a Dependabot

### 6.1 Comparativa Dependabot vs Renovate

| Característica | Dependabot | Renovate |
|----------------|------------|----------|
| Integración nativa con GitHub | ✅ Nativa | ⚙️ Requiere App/PAT |
| Configuración | YAML simple | JSON flexible y potente |
| Agrupación de PRs | ✅ Básica (groups) | ✅ Avanzada y flexible |
| Ecosistemas soportados | ~20 | ~60+ |
| Programación | Weekly/Daily/Monthly | Cron completo |
| Dependency Dashboard | ❌ | ✅ Tabla centralizada en un issue |
| Soporte monorepos | ⚠️ Limitado | ✅ Excelente |
| Auto-merge nativo | ⚠️ Requiere workflow | ✅ Configuración directa |
| Costo | Gratuito | Gratuito (self-hosted o App) |

### 6.2 Instalar la GitHub App de Renovate

```
1. Ir a: https://github.com/apps/renovate
2. Click "Install"
3. Seleccionar la organización o cuenta
4. Elegir "Only select repositories" → seleccionar tu repo
5. Completar la instalación
```

Renovate abrirá automáticamente una PR con una configuración inicial de `renovate.json`. Revísala y mergéala.

### 6.3 Activar Auto-merge en Renovate

El `renovate.json` incluido ya tiene la configuración base. Los campos clave:

```json
{
  "packageRules": [
    {
      "matchUpdateTypes": ["patch"],
      "automerge": true,              // ← Activar auto-merge
      "automergeType": "pr",          // ← Merge vía PR (más seguro que branch)
      "platformAutomerge": true       // ← Usar el auto-merge nativo de GitHub
    }
  ]
}
```

Con `platformAutomerge: true`, Renovate usa la función de auto-merge de GitHub (igual que el workflow de Dependabot), lo que significa que los checks de CI deben pasar antes del merge.

### 6.4 Self-Hosted Renovate con GitHub Actions

Si prefieres control total sin instalar la App:

```yaml
# .github/workflows/renovate.yml
name: "Renovate"
on:
  schedule:
    - cron: "0 9 * * 1"  # Cada lunes a las 9:00 UTC
  workflow_dispatch:

jobs:
  renovate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: renovatebot/github-action@v40
        with:
          configurationFile: renovate.json
          token: ${{ secrets.RENOVATE_TOKEN }}
          # RENOVATE_TOKEN: PAT con scopes: repo, workflow, write:packages
```

---

## 7. Monitoreo y Solución de Problemas

### 7.1 Dónde Ver la Actividad de Dependabot

**Ver PRs abiertas por Dependabot:**
```
Pull requests → Filters: author:app/dependabot
```

**Ver el estado de cada ecosistema:**
```
Insights → Dependency graph → Dependabot → [click en cada ecosistema]
```

Aquí verás el log de la última ejecución de Dependabot, incluyendo:
- Qué dependencias revisó
- Por qué no abrió PR para alguna (versión ya actualizada, ignorada, etc.)
- Errores de autenticación o de lectura de manifiestos

**Ver los logs del workflow de auto-merge:**
```
Actions → "Auto-merge Dependabot PRs" → [click en la ejecución]
```

### 7.2 Errores Comunes y Soluciones

#### Error: `Resource not accessible by integration`

```
Error: Resource not accessible by integration
gh: pull request #XX is not mergeable: the base branch policy prohibits the merge
```

**Causa:** El `GITHUB_TOKEN` no tiene permisos suficientes para aprobar o hacer merge.

**Solución:**
```
Settings → Actions → General → Workflow permissions
→ ● Read and write permissions
→ ☑ Allow GitHub Actions to create and approve pull requests
```
O bien, cambiar a `AUTOMERGE_TOKEN` (PAT) en los steps relevantes.

---

#### Error: `GraphQL: Required status check "test" is expected`

```
gh: Cannot enable auto merge: Pull request is not mergeable.
Required status checks have not passed.
```

**Causa:** Los checks de CI todavía están corriendo cuando el workflow de auto-merge intentó habilitar el merge.

**Solución:** El step "Esperar checks de CI" (`lewagon/wait-on-check-action`) debería manejar esto. Verificar que el `check-name` en el workflow coincida exactamente con el nombre del job en tu CI:

```yaml
# En tu workflow de CI (ci-cd.yml):
jobs:
  test:
    name: "🧪 Tests"  # ← Este es el nombre que debes usar

# En auto-merge-dependabot.yml:
- uses: lewagon/wait-on-check-action@v1.3.4
  with:
    check-name: "🧪 Tests"  # ← Debe coincidir exactamente (emoji incluido)
```

---

#### Error: `Pull request is not mergeable: There are merge conflicts`

**Causa:** La rama de Dependabot tiene conflictos con `main` (alguien mergeó algo incompatible mientras tanto).

**Solución:** Dependabot puede actualizar la rama con un comentario:

```bash
# Comentar en la PR de Dependabot:
@dependabot rebase

# Otros comandos útiles de Dependabot vía comentarios:
@dependabot recreate          # Recrear la PR desde cero
@dependabot merge             # Fusionar manualmente (si tienes permisos)
@dependabot squash and merge  # Squash merge manual
@dependabot ignore this major version
@dependabot ignore this minor version
@dependabot ignore this dependency
```

---

#### Error: `dependabot/fetch-metadata: Error: Not a Dependabot PR`

**Causa:** El workflow se disparó con `pull_request_target` por un PR que NO es de Dependabot (quizás alguien abrió un PR manual y disparó el workflow de alguna forma).

**Solución:** La condición `if: github.actor == 'dependabot[bot]'` en el job debería prevenir esto. Si el error persiste, verificar que el trigger `pull_request_target` tenga el filtro de autor correcto (en GitHub no se puede filtrar por autor en los triggers, solo con la condición `if:`).

---

#### Error: Token expirado

```
Error: Authentication token is expired
```

**Causa:** El PAT almacenado en `secrets.AUTOMERGE_TOKEN` expiró.

**Solución:**
```
1. Ir a la cuenta del bot → Settings → Developer settings → Personal access tokens
2. Renovar el token (o crear uno nuevo sin expiración para cuentas de servicio)
3. Actualizar el secret en: Settings → Secrets → AUTOMERGE_TOKEN → Update
```

Para evitar esto, configura alertas de expiración de token en GitHub (`Settings → Notifications → Email → Token expiry`).

---

#### Dependabot no abre PRs después de configurarlo

**Checklist:**

```
□ El archivo .github/dependabot.yml está en la rama PRINCIPAL (main/master)
□ La sintaxis YAML es correcta (validar en https://yaml.lint.com/)
□ El package-ecosystem es correcto para tu gestor de paquetes
□ El directory apunta al directorio correcto (donde está el manifiesto)
□ No se alcanzó el límite de open-pull-requests-limit
□ Las dependencias no están en la lista de ignore
□ Dependabot está habilitado en Settings → Code security
```

---

### 7.3 Interpretar la Salida de `dependabot/fetch-metadata`

Los outputs más importantes del step de metadatos:

| Output | Valores Posibles | Uso |
|--------|-----------------|-----|
| `update-type` | `version-update:semver-major`, `semver-minor`, `semver-patch` | Decidir si auto-merge |
| `dependency-type` | `direct:production`, `direct:development`, `indirect` | Auto-merge más agresivo para dev |
| `package-ecosystem` | `npm`, `pip`, `docker`, `github_actions`, etc. | Filtrar por ecosistema |
| `compatible` | `true`, `false`, `unknown` | Solo auto-merge si es compatible |
| `alert-state` | `OPEN`, `DISMISSED`, `FIXED`, `""` | Priorizar updates de seguridad |
| `cvss` | `0.0` - `10.0` | Urgencia de la actualización |

---

## 8. Integración con Workflows de CI/CD Existentes

### 8.1 Por Qué `pull_request_target` en Auto-merge vs `pull_request` en CI

Esta distinción es crucial para la seguridad:

| Evento | Contexto de Ejecución | Acceso a Secrets |
|--------|----------------------|-----------------|
| `pull_request` | Rama del PR (código del PR) | ❌ No (en PRs de forks) |
| `pull_request_target` | Rama BASE (main) | ✅ Sí (siempre) |

El workflow de CI (lint, tests) debe usar `pull_request` — así corre el código del PR y detecta si los tests pasan con los cambios propuestos.

El workflow de auto-merge usa `pull_request_target` — necesita acceso a los secrets para poder hacer merge.

### 8.2 Asegurar que el CI se Dispara en PRs de Dependabot

Los workflows de CI con `on: pull_request` se disparan automáticamente en las PRs de Dependabot. No necesitas cambios.

Sin embargo, si tu CI tiene condiciones `if:` que pueden excluir a Dependabot:

```yaml
# ❌ Esto excluiría a Dependabot:
jobs:
  test:
    if: github.actor != 'dependabot[bot]'

# ✅ Asegurarse de incluir a Dependabot explícitamente:
jobs:
  test:
    if: |
      github.event_name == 'push' ||
      github.event_name == 'pull_request'
      # Sin excluir a dependabot[bot]
```

### 8.3 Saltar Steps Costosos en PRs de Dependabot

Algunos steps del CI son innecesarios para PRs de dependencias (como el deploy a staging):

```yaml
# En tu workflow ci-cd.yml existente:
jobs:
  deploy-staging:
    # No desplegar a staging para PRs de Dependabot
    # (sus cambios son solo de versiones de deps, no de lógica de negocio)
    if: |
      github.event_name == 'push' ||
      (github.event_name == 'pull_request' && github.actor != 'dependabot[bot]')

  # Pero SÍ ejecutar los tests completos:
  test:
    # Sin condición de exclusión para Dependabot
    if: github.event_name == 'push' || github.event_name == 'pull_request'
```

### 8.4 Orden de Ejecución Completo

El flujo completo cuando Dependabot abre una PR:

```
1. Dependabot abre PR (rama: dependabot/npm_and_yarn/lodash-4.17.22)
         │
         ▼
2. Se dispara: on: pull_request (workflow CI: lint + test + security)
         │
         ├── lint ────────────────────────────────── ✅
         ├── test (matrix: ubuntu, node 18/20/22) ── ✅
         └── security ────────────────────────────── ✅
         │
         ▼
3. Se dispara: on: pull_request_target (auto-merge-dependabot.yml)
         │
         ├── Obtener metadatos → update-type: semver-patch ✅
         ├── Verificar actor → dependabot[bot] ✅
         ├── Esperar CI (wait-on-check-action) → CI pasó ✅
         ├── Aprobar PR → gh pr review --approve ✅
         └── Habilitar auto-merge → gh pr merge --auto --squash ✅
         │
         ▼
4. GitHub detecta: todos los checks requeridos ✅ + aprobación ✅
         │
         ▼
5. GitHub ejecuta el merge automáticamente
         │
         ▼
6. Se dispara: on: push (rama main) → CI/CD completo de producción
```

### 8.5 Notificaciones Adicionales para el Equipo

Para que el equipo sepa qué se mergeó automáticamente, añade una notificación Slack al finalizar el auto-merge:

```yaml
# Al final del job auto-merge, añadir:
- name: "📢 Notificar merge a Slack"
  if: success()
  uses: slackapi/slack-github-action@v1.27.0
  with:
    payload: |
      {
        "text": "🤖 Dependabot auto-mergeó: `${{ steps.dependabot-metadata.outputs.dependency-names }}` (${{ steps.dependabot-metadata.outputs.update-type }}) en `${{ github.repository }}`",
        "blocks": [{
          "type": "section",
          "text": {
            "type": "mrkdwn",
            "text": "✅ *Auto-merge completado*\n• Repo: `${{ github.repository }}`\n• Dep: `${{ steps.dependabot-metadata.outputs.dependency-names }}`\n• Tipo: `${{ steps.dependabot-metadata.outputs.update-type }}`\n• <${{ github.event.pull_request.html_url }}|Ver PR>"
          }
        }]
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
    SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK
```

---

*Guía generada para la configuración de Dependabot + Auto-merge v1.0.0*
*Para reportar problemas, abre un issue con los logs del workflow y el contenido de tu dependabot.yml.*
