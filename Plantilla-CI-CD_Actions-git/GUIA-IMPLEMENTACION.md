# Guía de Implementación: CI/CD Pipeline Reutilizable con GitHub Actions

> **Versión:** 2.0.0 | **Actualizado:** 2025 | **Nivel:** Intermedio-Avanzado

---

## Tabla de Contenidos

1. [Preparación del Repositorio](#1-preparación-del-repositorio)
2. [Configuración de Secrets y Variables](#2-configuración-de-secrets-y-variables)
3. [Configuración de Entornos (Environments)](#3-configuración-de-entornos-environments)
4. [Personalización por Stack Tecnológico](#4-personalización-por-stack-tecnológico)
5. [Integración con Herramientas de Calidad](#5-integración-con-herramientas-de-calidad)
6. [Estrategia de Versionado y Releases](#6-estrategia-de-versionado-y-releases)
7. [Solución de Problemas Comunes](#7-solución-de-problemas-comunes)

---

## 1. Preparación del Repositorio

### 1.1 Estructura de Carpetas Recomendada

```
mi-proyecto/
├── .github/
│   ├── workflows/
│   │   ├── ci-cd.yml              ← Plantilla principal (este archivo)
│   │   └── dependency-update.yml  ← Workflow auxiliar (Dependabot, etc.)
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug_report.md
│   │   └── feature_request.md
│   └── PULL_REQUEST_TEMPLATE.md
├── src/                           ← Código fuente
├── tests/                         ← Tests unitarios e integración
│   ├── unit/
│   └── integration/
├── docs/                          ← Documentación
├── Dockerfile                     ← Solo si usas contenedores
├── .dockerignore
├── sonar-project.properties       ← Config de SonarCloud
├── .releaserc.json                ← Config de semantic-release
├── .eslintrc.js / .eslintrc.json  ← Config ESLint (Node.js)
├── .pylintrc / pyproject.toml     ← Config Pylint/Black (Python)
├── codecov.yml                    ← Config de Codecov
└── README.md                      ← Con badge del workflow
```

### 1.2 Crear el Archivo de Workflow

```bash
# Desde la raíz de tu repositorio
mkdir -p .github/workflows
cp ci-cd.yml .github/workflows/ci-cd.yml
git add .github/workflows/ci-cd.yml
git commit -m "ci: add CI/CD pipeline template"
git push origin main
```

### 1.3 Badge de Estado para README.md

Agrega esto al inicio de tu `README.md`, sustituyendo `<ORG>` y `<REPO>`:

```markdown
![CI/CD Pipeline](https://github.com/<ORG>/<REPO>/actions/workflows/ci-cd.yml/badge.svg)
[![Coverage](https://codecov.io/gh/<ORG>/<REPO>/branch/main/graph/badge.svg)](https://codecov.io/gh/<ORG>/<REPO>)
[![Security Rating](https://sonarcloud.io/api/project_badges/measure?project=<ORG>_<REPO>&metric=security_rating)](https://sonarcloud.io/summary/new_code?id=<ORG>_<REPO>)
```

### 1.4 Archivos de Configuración de Linters

**`.eslintrc.json` (Node.js):**
```json
{
  "env": { "node": true, "es2022": true },
  "extends": ["eslint:recommended"],
  "parserOptions": { "ecmaVersion": 2022, "sourceType": "module" },
  "rules": {
    "no-console": "warn",
    "no-unused-vars": "error"
  }
}
```

**`pyproject.toml` (Python con Black y Flake8):**
```toml
[tool.black]
line-length = 88
target-version = ["py311", "py312"]

[tool.isort]
profile = "black"
line_length = 88

[tool.pylint.messages_control]
disable = ["C0114", "C0115", "C0116"]
```

**`.golangci.yml` (Go):**
```yaml
linters:
  enable:
    - errcheck
    - gosimple
    - govet
    - ineffassign
    - staticcheck
    - unused
run:
  timeout: 5m
```

### 1.5 Dockerfile Recomendado (Multi-stage)

Un Dockerfile seguro y optimizado que el workflow puede construir:

```dockerfile
# =====================================================================
# Stage 1: Build (imagen completa con herramientas de desarrollo)
# =====================================================================
FROM node:20-alpine AS builder
WORKDIR /app

# Copiar solo los archivos de dependencias primero (mejor caché)
COPY package*.json ./
RUN npm ci --only=production

# Copiar código fuente y compilar
COPY . .
RUN npm run build

# =====================================================================
# Stage 2: Production (imagen mínima y segura)
# =====================================================================
FROM node:20-alpine AS production

# Seguridad: no ejecutar como root
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

WORKDIR /app

# Copiar solo lo necesario del stage de build
COPY --from=builder --chown=nextjs:nodejs /app/dist ./dist
COPY --from=builder --chown=nextjs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nextjs:nodejs /app/package.json ./package.json

USER nextjs

EXPOSE 3000
ENV NODE_ENV=production

# Health check integrado en la imagen
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

CMD ["node", "dist/index.js"]
```

---

## 2. Configuración de Secrets y Variables

### 2.1 ¿Qué son Secrets vs Variables?

| Tipo | Visibilidad | Uso típico |
|------|-------------|------------|
| **Secret** | Cifrado, nunca visible en logs | Tokens, contraseñas, claves API |
| **Variable** | Visible en la UI de GitHub | URLs, nombres de servicios, configuración |

### 2.2 Dónde Crear Secrets en GitHub

Navega a tu repositorio → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**.

### 2.3 Lista Completa de Secrets Requeridos

#### Secrets Universales (todos los proyectos)

| Secret | Cómo obtenerlo | Descripción |
|--------|---------------|-------------|
| `SLACK_WEBHOOK_URL` | [Slack API](https://api.slack.com/apps) → Create App → Incoming Webhooks | Notificaciones Slack |
| `DISCORD_WEBHOOK_URL` | Discord → Configuración del servidor → Webhooks | Notificaciones Discord |
| `TEAMS_WEBHOOK_URL` | Teams → Canal → Conectores → Incoming Webhook | Notificaciones Teams |
| `SONAR_TOKEN` | [SonarCloud](https://sonarcloud.io) → My Account → Security | Análisis de calidad |
| `CODECOV_TOKEN` | [Codecov](https://app.codecov.io) → Settings → Repository token | Cobertura de tests |
| `SNYK_TOKEN` | [Snyk](https://app.snyk.io) → Account Settings → API token | Escaneo de seguridad |

#### Secrets para Docker Hub

```bash
# 1. Ir a hub.docker.com → Account Settings → Security → New Access Token
# 2. Copiar el token generado (solo se muestra una vez)
# 3. Crear los siguientes secrets en GitHub:
DOCKER_HUB_USERNAME=tu_usuario_dockerhub
DOCKER_HUB_TOKEN=dckr_pat_xxxxxxxxxxxxxxxxxxxx
```

#### Secrets para AWS

```bash
# Opción A: Access Keys (más simple, menos seguro)
# IAM → Usuarios → tu_usuario → Credenciales de seguridad → Crear clave de acceso
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

# Opción B: OIDC (recomendado para producción - sin access keys)
# Configurar un Identity Provider en IAM → Identity Providers → OpenID Connect
# Provider URL: https://token.actions.githubusercontent.com
# Audience: sts.amazonaws.com
# Luego crear un rol IAM con la política necesaria y añadir:
AWS_ROLE_ARN=arn:aws:iam::123456789012:role/GitHubActionsRole
```

#### Secrets para Vercel

```bash
# 1. Ir a vercel.com → Settings → Tokens → Create Token
VERCEL_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# 2. En tu proyecto de Vercel → Settings → General → Project ID y Team ID
VERCEL_PROJECT_ID=prj_xxxxxxxxxxxxxxxxxxxxxxxx
VERCEL_ORG_ID=team_xxxxxxxxxxxxxxxxxxxxxxxx
```

#### Secrets para Netlify

```bash
# 1. app.netlify.com → User Settings → Applications → New access token
NETLIFY_AUTH_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# 2. En tu sitio de Netlify → Site Settings → General → Site ID
NETLIFY_SITE_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

#### Secrets para Kubernetes

```bash
# Generar kubeconfig con acceso mínimo (solo namespace staging/production)
kubectl config view --minify --flatten | base64 -w 0
# Copiar el output como secret:
KUBECONFIG_STAGING=<base64_del_kubeconfig_staging>
KUBECONFIG_PROD=<base64_del_kubeconfig_produccion>
```

### 2.4 Variables de Repositorio (No Secretas)

Navega a **Settings** → **Secrets and variables** → **Actions** → **Variables** → **New repository variable**.

| Variable | Valor Ejemplo | Descripción |
|----------|--------------|-------------|
| `PROJECT_TYPE` | `node` | Tipo de proyecto: node, python, java, go, docker |
| `DEPLOYMENT_TARGET` | `docker_hub` | Plataforma de despliegue |
| `IMAGE_NAME` | `mi-org/mi-app` | Nombre de imagen Docker |
| `WORKING_DIR` | `.` | Directorio de trabajo (monorepos) |
| `STAGING_URL` | `https://staging.miapp.com` | URL del entorno de staging |
| `PRODUCTION_URL` | `https://miapp.com` | URL de producción |
| `AWS_REGION` | `us-east-1` | Región de AWS |
| `ECS_CLUSTER_STAGING` | `mi-cluster-staging` | Nombre del cluster ECS staging |
| `ECS_SERVICE_STAGING` | `mi-servicio-staging` | Nombre del servicio ECS staging |
| `ECS_CLUSTER_PROD` | `mi-cluster-prod` | Nombre del cluster ECS producción |
| `ECS_SERVICE_PROD` | `mi-servicio-prod` | Nombre del servicio ECS producción |
| `DOMAIN` | `miapp.com` | Dominio principal |
| `NOTIFY_SLACK` | `true` | Activar notificaciones Slack |
| `NOTIFY_DISCORD` | `false` | Activar notificaciones Discord |
| `NOTIFY_TEAMS` | `false` | Activar notificaciones Teams |

---

## 3. Configuración de Entornos (Environments)

Los **GitHub Environments** son una capa de protección y control sobre los despliegues. Permiten requerir aprobación manual antes de desplegar a producción.

### 3.1 Crear el Entorno "staging"

1. Ve a **Settings** → **Environments** → **New environment**
2. Nombre: `staging`
3. Configura las siguientes protecciones:

```
☑ Required reviewers: (dejar vacío para staging — despliegue automático)
☑ Wait timer: 0 minutes
☑ Deployment branches and tags:
   → Selected branches: develop, main
```

4. Añade variables específicas del entorno:
   - `STAGING_URL` = `https://staging.tudominio.com`

### 3.2 Crear el Entorno "production"

1. **Settings** → **Environments** → **New environment**
2. Nombre: `production`
3. Configura protecciones estrictas:

```
☑ Required reviewers:
   → Añadir: @devops-lead, @tech-lead (mínimo 1 aprobador)
☑ Wait timer: 5 minutes  ← Tiempo para cancelar si algo va mal
☑ Prevent self-review: activado
☑ Deployment branches and tags:
   → Selected branches: main (SOLO main puede desplegar a producción)
```

4. Añade secrets específicos de producción (sobreescriben los del repositorio):
   - `DOCKER_HUB_TOKEN` (puede ser un token con más permisos)
   - `AWS_SECRET_ACCESS_KEY` (credenciales del entorno prod)

### 3.3 Flujo de Aprobación en Producción

Cuando el workflow llega al job `deploy-production`:

```
1. GitHub pausa la ejecución
2. Envía email/notificación a los reviewers configurados
3. El reviewer ve el resumen del deployment en la UI
4. Aprueba o rechaza con comentario
5. El workflow continúa (o termina si es rechazado)
```

El reviewer puede aprobar desde:
- La UI de GitHub Actions (pestaña Actions del repo)
- El email de notificación
- La app móvil de GitHub

---

## 4. Personalización por Stack Tecnológico

### 4.1 Proyecto Node.js (Express / Next.js / NestJS)

Ajustes específicos en el YAML:

```yaml
# En la sección env: global
env:
  PROJECT_TYPE: node
  DEPLOYMENT_TARGET: vercel  # o docker_hub, netlify

# En el job test, ajusta la matriz de versiones:
strategy:
  matrix:
    os: [ubuntu-latest]
    version: ["18", "20", "22"]  # LTS versions de Node
```

**`package.json` mínimo requerido:**

```json
{
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "test": "jest --coverage",
    "lint": "eslint src --ext .ts"
  },
  "jest": {
    "testEnvironment": "node",
    "coverageReporters": ["lcov", "text"],
    "coverageDirectory": "coverage"
  }
}
```

**`.releaserc.json` para semantic-release:**

```json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    ["@semantic-release/changelog", {
      "changelogFile": "CHANGELOG.md"
    }],
    "@semantic-release/npm",
    ["@semantic-release/github", {
      "assets": ["dist/*.js", "dist/*.js.map"]
    }],
    ["@semantic-release/git", {
      "assets": ["CHANGELOG.md", "package.json"],
      "message": "chore(release): ${nextRelease.version} [skip ci]"
    }]
  ]
}
```

### 4.2 Proyecto Python (Flask / FastAPI / Django)

Ajustes en el YAML:

```yaml
env:
  PROJECT_TYPE: python
  DEPLOYMENT_TARGET: docker_hub

# Cambiar la matriz de versiones en el job test:
strategy:
  matrix:
    os: [ubuntu-latest]
    version: ["3.10", "3.11", "3.12"]
```

**`requirements.txt` mínimo:**

```
flask==3.0.0          # O fastapi, django, etc.
pytest==8.0.0
pytest-cov==5.0.0
pytest-asyncio==0.23.0
black==24.0.0
flake8==7.0.0
isort==5.13.0
```

**`pyproject.toml` completo:**

```toml
[build-system]
requires = ["setuptools>=68.0", "wheel"]
build-backend = "setuptools.backends.legacy:build"

[project]
name = "mi-app"
version = "1.0.0"
requires-python = ">=3.10"

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--cov=src --cov-report=xml --cov-report=lcov"

[tool.black]
line-length = 88

[tool.isort]
profile = "black"
```

**`Dockerfile` para Python/Flask:**

```dockerfile
FROM python:3.12-slim AS production

WORKDIR /app
RUN adduser --disabled-password --gecos '' appuser

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY --chown=appuser:appuser . .
USER appuser

EXPOSE 5000
HEALTHCHECK CMD curl -f http://localhost:5000/health || exit 1
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "src.app:app"]
```

### 4.3 Contenedor Docker Puro

Para proyectos que solo necesitan construir y publicar una imagen:

```yaml
env:
  PROJECT_TYPE: docker
  DEPLOYMENT_TARGET: docker_hub

# En el job lint: usar hadolint para lintear el Dockerfile
- name: "🔍 Lint Dockerfile"
  uses: hadolint/hadolint-action@v3.1.0
  with:
    dockerfile: Dockerfile
    failure-threshold: warning

# En el job test: puedes hacer tests de la imagen con container-structure-test
- name: "🧪 Container Structure Test"
  run: |
    curl -LO https://storage.googleapis.com/container-structure-test/latest/container-structure-test-linux-amd64
    chmod +x container-structure-test-linux-amd64
    ./container-structure-test-linux-amd64 test \
      --image mi-org/mi-imagen:latest \
      --config tests/container-structure-test.yaml
```

**`tests/container-structure-test.yaml`:**

```yaml
schemaVersion: 2.0.0

commandTests:
  - name: "Node.js instalado"
    command: "node"
    args: ["--version"]
    expectedOutput: ["v20.*"]

  - name: "Usuario no es root"
    command: "whoami"
    expectedOutput: ["nodeuser"]

fileExistenceTests:
  - name: "dist/ existe"
    path: "/app/dist"
    shouldExist: true

metadataTest:
  exposedPorts: ["3000"]
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
```

---

## 5. Integración con Herramientas de Calidad

### 5.1 SonarCloud

**Paso 1:** Crear cuenta en [sonarcloud.io](https://sonarcloud.io) con tu cuenta de GitHub.

**Paso 2:** Importar el repositorio (Organization → Import repository).

**Paso 3:** Obtener el token:
- My Account → Security → Generate Tokens
- Añadir como secret `SONAR_TOKEN` en GitHub

**Paso 4:** Crear `sonar-project.properties` en la raíz del proyecto:

```properties
# Identificadores (obligatorios)
sonar.projectKey=mi-org_mi-proyecto
sonar.organization=mi-org

# Fuentes y tests
sonar.sources=src
sonar.tests=tests
sonar.exclusions=**/*.test.js,**/node_modules/**,**/dist/**

# Cobertura
sonar.javascript.lcov.reportPaths=coverage/lcov.info
sonar.python.coverage.reportPaths=coverage.xml

# Encoding
sonar.sourceEncoding=UTF-8
```

### 5.2 Codecov

**Paso 1:** Ir a [app.codecov.io](https://app.codecov.io), autenticarse con GitHub.

**Paso 2:** Añadir el repositorio y copiar el token del repo (para repos privados).

**Paso 3:** Crear `codecov.yml` en la raíz:

```yaml
coverage:
  status:
    project:
      default:
        target: 80%        # Mínimo 80% de cobertura total
        threshold: 1%      # Permitir caída de hasta 1% sin fallar
    patch:
      default:
        target: 70%        # 70% en código nuevo del PR

comment:
  layout: "reach, diff, flags, files"
  behavior: default
  require_changes: true    # Solo comentar si la cobertura cambia
```

### 5.3 Snyk

**Paso 1:** Crear cuenta en [snyk.io](https://snyk.io), conectar con GitHub.

**Paso 2:** Obtener el token en Account Settings → API token.

**Paso 3:** El workflow ya tiene integrado el step de Snyk. El escaneo también puede ejecutarse localmente:

```bash
# Instalar CLI de Snyk
npm install -g snyk

# Autenticar
snyk auth

# Escanear dependencias
snyk test

# Monitorear continuamente (enviar resultados a la UI de Snyk)
snyk monitor
```

### 5.4 GitLeaks (Configuración Avanzada)

Crea `.gitleaks.toml` para personalizar las reglas:

```toml
[extend]
useDefault = true  # Usar todas las reglas por defecto

[[rules]]
description = "Custom: API Keys internas"
id = "internal-api-key"
regex = '''MYAPP_API_KEY[_-]?[0-9A-Za-z]{32,}'''
tags = ["api-key", "internal"]

[allowlist]
description = "Archivos ignorados"
paths = [
  '''\.gitleaks\.toml''',
  '''tests/fixtures/''',
  '''docs/examples/'''
]
regexes = [
  # Ignorar tokens de ejemplo en la documentación
  '''EXAMPLE_TOKEN_[A-Z]+''',
]
```

---

## 6. Estrategia de Versionado y Releases

### 6.1 Convención de Commits (Conventional Commits)

El versionado semántico automático depende de que los commits sigan la convención:

```
<tipo>[scope opcional]: <descripción>

Tipos que incrementan versión:
  feat:     Nueva funcionalidad        → MINOR (1.0.0 → 1.1.0)
  fix:      Corrección de bug          → PATCH (1.0.0 → 1.0.1)
  BREAKING CHANGE: en el footer       → MAJOR (1.0.0 → 2.0.0)

Tipos que NO incrementan versión:
  docs:     Documentación
  style:    Formato de código
  refactor: Refactorización
  test:     Tests
  chore:    Mantenimiento
  ci:       Cambios en CI/CD
  perf:     Mejoras de rendimiento

Ejemplos:
  feat(auth): add OAuth2 login with Google
  fix(api): resolve null pointer in user endpoint
  feat!: remove support for Node.js 16  ← BREAKING CHANGE
```

### 6.2 Instalar Commitlint para Forzar la Convención

```bash
# Instalar dependencias
npm install --save-dev @commitlint/cli @commitlint/config-conventional husky

# Inicializar Husky (hooks de Git)
npx husky init
echo "npx --no -- commitlint --edit \$1" > .husky/commit-msg
```

**`commitlint.config.js`:**

```javascript
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [2, 'always', [
      'feat', 'fix', 'docs', 'style', 'refactor',
      'test', 'chore', 'ci', 'perf', 'revert', 'build'
    ]],
    'subject-max-length': [2, 'always', 100],
    'body-max-line-length': [2, 'always', 200],
  }
};
```

### 6.3 Añadir Commitlint al Pipeline de CI

```yaml
# Añadir en el job lint
- name: "📝 Validar mensajes de commit"
  if: github.event_name == 'pull_request'
  uses: wagoid/commitlint-github-action@v5
  with:
    configFile: commitlint.config.js
```

### 6.4 Configuración Completa de semantic-release

```bash
# Instalar paquetes necesarios
npm install --save-dev \
  semantic-release \
  @semantic-release/changelog \
  @semantic-release/commit-analyzer \
  @semantic-release/exec \
  @semantic-release/git \
  @semantic-release/github \
  @semantic-release/npm \
  @semantic-release/release-notes-generator
```

**`.releaserc.json` con todas las opciones:**

```json
{
  "branches": [
    "main",
    { "name": "develop", "prerelease": "beta" },
    { "name": "feature/*", "prerelease": "alpha" }
  ],
  "plugins": [
    ["@semantic-release/commit-analyzer", {
      "preset": "conventionalcommits",
      "releaseRules": [
        { "type": "feat",     "release": "minor" },
        { "type": "fix",      "release": "patch" },
        { "type": "perf",     "release": "patch" },
        { "type": "revert",   "release": "patch" },
        { "breaking": true,   "release": "major" }
      ]
    }],
    "@semantic-release/release-notes-generator",
    ["@semantic-release/changelog", {
      "changelogFile": "CHANGELOG.md"
    }],
    ["@semantic-release/exec", {
      "prepareCmd": "echo ${nextRelease.version} > VERSION"
    }],
    "@semantic-release/github",
    ["@semantic-release/git", {
      "assets": ["CHANGELOG.md", "package.json", "VERSION"],
      "message": "chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}"
    }]
  ]
}
```

### 6.5 Flujo Completo de Versionado

```
develop   ←── feature/nueva-funcionalidad
    │           (commits: feat, fix)
    │
    ▼
  PR a develop ──► CI Pipeline (lint + test + security)
    │
    ├── semantic-release publica: 1.1.0-beta.1
    │
  PR a main ──► CI Pipeline completo + Aprobación manual
    │
    ▼
  main ──► Deploy staging ──► Aprobación ──► Deploy production
    │
    └── semantic-release publica: v1.1.0
        - Crea GitHub Release
        - Actualiza CHANGELOG.md
        - Publica en npm (si aplica)
```

---

## 7. Solución de Problemas Comunes

### 7.1 Errores de Autenticación

**Error: `Error: Unable to authenticate to Docker Hub`**

```bash
# Verificar que el secret DOCKER_HUB_TOKEN existe
# (NO usar la contraseña de Docker Hub, usar Access Token)

# Ir a: hub.docker.com → Account Settings → Security → New Access Token
# Permisos mínimos: Read & Write (no necesitas Delete)

# Verificar en la UI de GitHub que el secret está configurado:
# Settings → Secrets → Actions → buscar DOCKER_HUB_TOKEN
```

**Error: `Error: credentials not found` en AWS**

```bash
# Verificar que las variables de entorno están en el step correcto
# Las credenciales deben estar en el bloque env: del step, NO en env: global

# ✅ Correcto:
- name: "Deploy ECS"
  run: aws ecs update-service ...
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

# ❌ Incorrecto (el step no hereda env: global de forma implícita en AWS CLI):
env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}  # Nivel global
```

**Error en OIDC con AWS: `Could not assume role`**

```json
// La política de confianza del rol IAM debe incluir:
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
        "token.actions.githubusercontent.com:sub": "repo:<ORG>/<REPO>:ref:refs/heads/main"
      }
    }
  }]
}
```

### 7.2 Fallos de Caché

**El caché de npm no funciona (node_modules incompletos):**

```yaml
# Problema: el hash del lock file cambió pero la clave es la misma
# Solución: invalidar la caché manualmente

# Opción A: Cambiar el prefijo de la clave
key: v2-${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
#    ^^^ Incrementar la versión fuerza nuevo caché

# Opción B: Borrar caché desde la UI
# Actions → Caches (en el menú lateral del repo) → borrar entradas
```

**El caché de Docker no reduce el tiempo de build:**

```bash
# Problema: el Dockerfile no está optimizado para caché por capas
# Regla: lo que cambia menos frecuente va primero

# ❌ Mal orden (invalida caché con cada cambio de código):
COPY . .
RUN npm install

# ✅ Orden correcto (npm install solo se re-ejecuta si cambia package.json):
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
```

### 7.3 Timeouts en Tests

**El job de tests excede el timeout:**

```yaml
# Aumentar el timeout del job específico:
jobs:
  test:
    timeout-minutes: 60  # Aumentar según necesidad (default: 6 horas)

# O agregar timeout a pasos individuales:
- name: "Tests de integración"
  timeout-minutes: 20
  run: npm run test:integration
```

**Tests de integración que esperan conexiones externas:**

```yaml
# Usar servicios Docker de GitHub Actions:
jobs:
  test:
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 3s
          --health-retries 10
          --health-start-period 10s  # Tiempo inicial para arrancar
```

### 7.4 Cómo Leer los Logs Correctamente

**Navegar a los logs de un workflow:**

```
1. GitHub → tu repo → pestaña "Actions"
2. Click en el workflow run fallido
3. Click en el job fallido (aparece con ❌)
4. Expandir el step fallido (click en el nombre)
5. Los logs se muestran con timestamp y número de línea
```

**Buscar el error real (a veces está oculto):**

```bash
# En los logs busca:
##[error]     ← Errores de GitHub Actions
Error:        ← Errores de Node.js/npm
FAILED        ← Errores de pytest
exit code:    ← Código de salida del proceso

# Los mensajes de "Process completed with exit code N" al final
# del step indican si tuvo éxito (0) o fallo (1+)
```

### 7.5 Activar el Modo Debug

**Debug de GitHub Actions (máxima verbosidad):**

```bash
# Método 1: Secrets especiales (recomendado para un run puntual)
# Crear temporalmente estos secrets en tu repo:
ACTIONS_RUNNER_DEBUG = true    # Logs del runner
ACTIONS_STEP_DEBUG   = true    # Logs de cada step

# Método 2: Re-ejecutar con debug activado
# Actions → seleccionar run fallido → "Re-run jobs" → "Enable debug logging"

# Método 3: SSH al runner (action-tmate para inspección interactiva)
- name: "🐛 Debug interactivo con tmate"
  if: failure()  # Solo si algo falla
  uses: mxschmitt/action-tmate@v3
  with:
    limit-access-to-actor: true  # Solo tú puedes conectarte
  timeout-minutes: 30
# Esto imprime en los logs: ssh xxxx@nyc1.tmate.io
```

**Imprimir contexto completo de GitHub Actions:**

```yaml
- name: "🔍 Dump GitHub context"
  if: runner.debug == '1'
  env:
    GITHUB_CONTEXT: ${{ toJson(github) }}
    RUNNER_CONTEXT: ${{ toJson(runner) }}
    ENV_CONTEXT: ${{ toJson(env) }}
  run: |
    echo "=== GITHUB CONTEXT ==="
    echo "$GITHUB_CONTEXT"
    echo "=== RUNNER CONTEXT ==="
    echo "$RUNNER_CONTEXT"
    # NUNCA imprimir secrets, solo env vars no sensibles
```

### 7.6 Problemas Comunes Adicionales

**`if:` condicional no funciona como esperas:**

```yaml
# ❌ Problema: comparar outputs de otros jobs puede fallar silenciosamente
if: needs.build.outputs.version == 'v1.0.0'

# ✅ Solución: usar fromJson() para valores que vienen de steps
if: ${{ fromJson(needs.build.outputs.deploy_needed) == true }}

# Para condicionales de secrets (secreto existe o no):
if: ${{ secrets.SNYK_TOKEN != '' }}  # ✅ Funciona
if: ${{ secrets.SNYK_TOKEN }}        # ❌ Siempre evalúa como true (vacío o no)
```

**Workflow no se dispara en `feature/**`:**

```yaml
# Verificar que el patrón usa la sintaxis correcta de glob:
on:
  push:
    branches:
      - "feature/**"   # ✅ Con comillas (YAML puede interpretar ** de otra forma)
      - feature/**     # ⚠️  Puede funcionar pero es ambiguo
```

**Concurrencia cancela despliegues que no debería:**

```yaml
# ❌ Problema: el grupo de concurrencia es demasiado genérico
concurrency:
  group: ${{ github.workflow }}  # Cancela TODOS los runs del workflow

# ✅ Solución: incluir la rama para separar concurrencia por rama
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true  # OK para CI (lint/test)

# Para despliegues: NUNCA cancelar in-progress
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false  # ← crítico para deploys
```

---

## Apéndice: Uso como Reusable Workflow

Para llamar este workflow desde otro repositorio:

```yaml
# En .github/workflows/deploy.yml del OTRO repositorio:
name: "Deploy via shared pipeline"
on:
  push:
    branches: [main]

jobs:
  deploy:
    uses: mi-org/devops-templates/.github/workflows/ci-cd.yml@main
    with:
      project_type: "node"
      deployment_target: "vercel"
      skip_tests: false
    secrets:
      DOCKER_HUB_USERNAME: ${{ secrets.DOCKER_HUB_USERNAME }}
      DOCKER_HUB_TOKEN: ${{ secrets.DOCKER_HUB_TOKEN }}
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
      # Alternativa: heredar TODOS los secrets del repo llamante
      # secrets: inherit
```

**Restricciones de Reusable Workflows:**
- Solo pueden ser llamados desde el mismo owner/organización (por defecto)
- Para compartir entre organizaciones: Settings → Actions → General → "Allow all actions and reusable workflows"
- Los `env:` globales del workflow llamante NO se pasan automáticamente
- Los outputs del reusable workflow deben declararse explícitamente en `on.workflow_call.outputs`

---

*Documentación generada para el CI/CD Pipeline Reusable Template v2.0.0*
*Para reportar problemas o sugerir mejoras, abre un issue en el repositorio.*
