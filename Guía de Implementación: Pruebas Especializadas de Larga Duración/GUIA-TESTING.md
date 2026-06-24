# Guía de Implementación: Pruebas Especializadas de Larga Duración

> **Versión:** 1.0.0 | **Módulo 6 de la Suite DevOps**

---

## Tabla de Contenidos

1. [Arquitectura de la Pirámide de Tests](#1-arquitectura-de-la-pirámide-de-tests)
2. [Elección y Configuración de Herramientas E2E](#2-elección-y-configuración-de-herramientas-e2e)
3. [Configuración de Entornos de Prueba](#3-configuración-de-entornos-de-prueba)
4. [Configuración de k6 para Pruebas de Carga](#4-configuración-de-k6-para-pruebas-de-carga)
5. [Chaos Engineering (Avanzado)](#5-chaos-engineering-avanzado)
6. [Programación de Pruebas Nocturnas](#6-programación-de-pruebas-nocturnas)
7. [Integración con CI/CD y Release](#7-integración-con-cicd-y-release)
8. [Solución de Problemas Comunes](#8-solución-de-problemas-comunes)

---

## 1. Arquitectura de la Pirámide de Tests

### 1.1 La Pirámide en el Contexto de Esta Suite

```
                    ╔══════════════╗
                    ║   CHAOS      ║  chaos-engineering.yml
                    ║  (manual,    ║  Solo staging, K8s
                    ║   semanal)   ║  ~10-30 min
                    ╠══════════════╣
                  ╔═╩══════════════╩═╗
                  ║   PERFORMANCE    ║  performance-tests.yml
                  ║   (semanal,      ║  k6, JMeter, Gatling
                  ║    pre-release)  ║  ~5-60 min
                  ╠══════════════════╣
                ╔═╩══════════════════╩═╗
                ║   END-TO-END (E2E)   ║  e2e-tests.yml
                ║   (nocturno, label,  ║  Playwright, Cypress
                ║    pre-release)      ║  ~15-120 min
                ╠══════════════════════╣
              ╔═╩══════════════════════╩═╗
              ║   INTEGRACIÓN / API      ║  ci-cd.yml (job test)
              ║   (en cada PR)           ║  supertest, pytest, JUnit
              ║                          ║  ~2-10 min
              ╠══════════════════════════╣
            ╔═╩══════════════════════════╩═╗
            ║     UNITARIOS                ║  ci-cd.yml (job test)
            ║   (en cada push)             ║  jest, pytest, JUnit
            ║                              ║  ~30s-3 min
            ╚══════════════════════════════╝
```

**Regla de oro:** Los tests más rápidos y baratos bloquean el pipeline; los más lentos y caros son puertas de calidad opcionales o programadas.

### 1.2 Cuándo Ejecutar Cada Capa

| Capa | Trigger | Bloquea merge | Herramienta |
|------|---------|--------------|-------------|
| Unitarios | Cada push | ✅ Sí | Jest, pytest, JUnit |
| Integración | Cada PR | ✅ Sí | supertest, pytest |
| E2E Smoke | Label `run-e2e` o `/e2e` | ⚠️ Opcional | Playwright |
| E2E Regression | Pre-release, nocturno | ✅ En release | Playwright |
| Performance | Pre-release, semanal | ✅ En release | k6 |
| Chaos | Manual, semanal | ❌ No | Chaos Mesh |

### 1.3 Integración sin Interferencias con el CI Principal

Los workflows de pruebas especializadas usan `workflow_call` o `gh workflow run`, lo que significa que no se mezclan con el pipeline de CI. El feedback rápido de tests unitarios sigue siendo de segundos.

```yaml
# En ci-cd.yml — los tests rápidos siempre corren:
jobs:
  test:
    name: "🧪 Tests"
    runs-on: ubuntu-latest
    # Timeout corto: si tarda más de 10 minutos, hay un problema
    timeout-minutes: 10
    steps:
      - run: npm test  # Solo tests unitarios e integración

# Los E2E y performance son workflows separados que NO bloquean el CI base
```

---

## 2. Elección y Configuración de Herramientas E2E

### 2.1 Comparativa: Playwright vs Cypress vs Selenium

| Característica | Playwright | Cypress | Selenium |
|---------------|-----------|---------|---------|
| Navegadores | Chromium, Firefox, WebKit | Chrome, Firefox, Edge | Todos (via WebDriver) |
| Lenguajes | JS/TS, Python, Java, .NET | JS/TS | Java, Python, C#, Ruby, JS |
| Velocidad | ⚡ Muy rápido | 🔶 Rápido | 🐌 Más lento |
| Paralelismo | ✅ Nativo | ✅ Con Dashboard | ⚠️ Con Grid |
| Setup en CI | ✅ Muy sencillo | ✅ Sencillo | ⚠️ Requiere Grid |
| Modo headless | ✅ Nativo | ✅ Nativo | ✅ Con args |
| Video/screenshots | ✅ Automáticos | ✅ Automáticos | ⚠️ Manual |
| Costo | Gratuito | Free + Dashboard de pago | Gratuito |
| **Recomendado para** | **Proyectos nuevos** | **Equipos con Cypress** | **Proyectos legacy** |

**Recomendación:** Para proyectos nuevos, **Playwright** es la opción superior. Cypress es excelente si el equipo ya lo usa. Selenium solo para proyectos Java legacy o cuando necesitas soporte de todos los navegadores incluyendo IE.

### 2.2 Configurar Playwright en GitHub Actions

**Paso 1: Instalar Playwright en el proyecto**

```bash
npm init playwright@latest

# Seleccionar:
# ✓ TypeScript
# ✓ tests/ como directorio de tests
# ✓ Add GitHub Actions workflow? → No (usamos el nuestro)
# ✓ Install Playwright browsers? → Yes
```

**Paso 2: Configurar `playwright.config.ts`**

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  // Directorio de tests
  testDir: './tests/e2e',

  // Timeout por test individual (30s es razonable para E2E)
  timeout: parseInt(process.env.E2E_TIMEOUT_MS || '30000'),

  // Reintentos en CI (mitiga flakiness sin ocultar bugs reales)
  retries: process.env.CI ? parseInt(process.env.E2E_RETRIES || '2') : 0,

  // Workers: en CI usar los definidos por el workflow
  workers: process.env.CI ? parseInt(process.env.E2E_WORKERS || '4') : undefined,

  // Reporteros: múltiples formatos simultáneos
  reporter: [
    ['html', { outputFolder: 'playwright-report' }],
    ['junit', { outputFile: 'test-results/junit.xml' }],
    process.env.CI ? ['github'] : ['list'],  // En CI: anotaciones en los logs
  ],

  // Configuración compartida de todos los tests
  use: {
    // URL base del entorno (se sobreescribe desde la variable de entorno)
    baseURL: process.env.BASE_URL || 'http://localhost:3000',

    // Capturar screenshots solo en fallos
    screenshot: 'only-on-failure',

    // Grabar video solo en fallos (los videos completos pesan mucho en CI)
    video: 'retain-on-failure',

    // Capturar trazas para debugging (solo en el primer reintento)
    trace: 'on-first-retry',

    // Ignorar errores de certificado SSL en entornos de staging
    ignoreHTTPSErrors: process.env.E2E_IGNORE_HTTPS === 'true',

    // Viewport estándar
    viewport: { width: 1280, height: 720 },
  },

  // Proyectos: un proyecto por navegador
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
    // Para tests móviles:
    // {
    //   name: 'Mobile Chrome',
    //   use: { ...devices['Pixel 5'] },
    // },
  ],

  // Directorio de salida para artefactos
  outputDir: 'test-results/',
});
```

**Paso 3: Estructura de tests con tags para filtrar**

```typescript
// tests/e2e/auth/login.spec.ts
import { test, expect } from '@playwright/test';

// Tag @smoke: se incluye en la suite "smoke"
test.describe('Login Flow @smoke', () => {
  test('should login with valid credentials', async ({ page }) => {
    await page.goto('/login');
    await page.fill('[data-testid="email"]', process.env.E2E_TEST_USER!);
    await page.fill('[data-testid="password"]', process.env.E2E_TEST_PASSWORD!);
    await page.click('[data-testid="login-button"]');

    // Verificar redirección al dashboard
    await expect(page).toHaveURL('/dashboard');
    await expect(page.locator('[data-testid="welcome-message"]')).toBeVisible();
  });

  test('should show error with invalid credentials @regression', async ({ page }) => {
    await page.goto('/login');
    await page.fill('[data-testid="email"]', 'invalid@test.com');
    await page.fill('[data-testid="password"]', 'wrongpassword');
    await page.click('[data-testid="login-button"]');

    await expect(page.locator('[data-testid="error-message"]')).toBeVisible();
    await expect(page).toHaveURL('/login');
  });
});
```

---

## 3. Configuración de Entornos de Prueba

### 3.1 Entorno Preview Efímero con Docker Compose

Crea `docker-compose.e2e.yml` en la raíz del repositorio:

```yaml
# docker-compose.e2e.yml
version: "3.9"
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: test
      DATABASE_URL: postgresql://test:test@db:5432/testdb
      REDIS_URL: redis://redis:6379
      JWT_SECRET: test-secret-for-e2e-only
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
      POSTGRES_DB: testdb
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test"]
      interval: 5s
      timeout: 3s
      retries: 10

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10

  # Seed de datos de prueba (se ejecuta y termina)
  seed:
    build: .
    command: npm run db:seed:e2e
    environment:
      DATABASE_URL: postgresql://test:test@db:5432/testdb
    depends_on:
      db:
        condition: service_healthy
    restart: "no"
```

### 3.2 Variables de GitHub Environments

Configura estos valores en cada Environment de GitHub:

**Environment `staging`** (Settings → Environments → staging):

```
Variables:
  E2E_BASE_URL = https://staging.mi-app.com

Secrets:
  E2E_TEST_USER = test.user@mi-app.com
  E2E_TEST_PASSWORD = [contraseña del usuario de test en staging]
  E2E_API_KEY = [API key de test con permisos mínimos]
```

**Environment `production`** (solo para smoke tests):

```
Variables:
  E2E_BASE_URL = https://mi-app.com

Secrets:
  E2E_TEST_USER = smoke.test@mi-app.com  # Cuenta dedicada, datos no sensibles
  E2E_TEST_PASSWORD = [contraseña]

Protección:
  ✅ Required reviewers: devops-lead
  ✅ Wait timer: 5 minutos
  ✅ Restrict deployments: solo rama main
```

---

## 4. Configuración de k6 para Pruebas de Carga

### 4.1 Estructura de Directorio para Scripts k6

```
k6-scripts/
├── smoke.js         ← Generado automáticamente por el workflow si no existe
├── load.js          ← Idem
├── stress.js        ← Idem
├── soak.js          ← Idem
├── spike.js         ← Idem
├── scenarios/
│   ├── checkout.js  ← Escenario específico de checkout
│   └── api.js       ← Tests de API
└── lib/
    ├── auth.js      ← Helper para autenticación
    └── helpers.js   ← Utilidades compartidas
```

### 4.2 Thresholds Realistas por Tipo de Aplicación

```javascript
// Thresholds para una API REST típica
export const options = {
  thresholds: {
    // Latencia (ajustar según SLAs del producto):
    'http_req_duration': [
      'p(50)<500',    // Mediana < 500ms
      'p(95)<2000',   // p95 < 2s (tolerable para usuarios)
      'p(99)<5000',   // p99 < 5s (límite de tolerancia)
    ],

    // Tasa de error (0% es el ideal, pero la realidad tiene ruido):
    'http_req_failed': ['rate<0.01'],   // < 1% de errores

    // Para endpoints específicos (más estrictos que el global):
    'http_req_duration{url:/api/products}': ['p(95)<1000'],
    'http_req_duration{url:/api/checkout}': ['p(95)<3000'],
  },
};

// Thresholds para una aplicación de e-commerce (más estrictos):
// p95 < 1500ms para páginas de producto (SEO y UX)
// p95 < 3000ms para el checkout (alta intención de compra)
// error_rate < 0.1% (10x más estricto que el default)

// Thresholds para un servicio interno (más laxos):
// p95 < 5000ms (no hay usuario final esperando)
// error_rate < 2%
```

### 4.3 Integración con Grafana Cloud (Opcional)

```bash
# 1. Crear cuenta gratuita en grafana.com/products/cloud/k6
# 2. Obtener API token: grafana.com → My Account → API Keys

# Añadir el secret en GitHub:
# GRAFANA_CLOUD_TOKEN = tu_token_aqui

# En el workflow, añadir el output de Grafana Cloud:
k6 run \
  --out cloud \
  --vus 10 --duration 5m \
  k6-scripts/load.js
```

```javascript
// En el script k6, configurar el proyecto de Grafana Cloud:
export const options = {
  ext: {
    loadimpact: {
      projectID: 123456,  // Tu project ID en Grafana Cloud k6
      name: "Load Test - API",
    },
  },
  thresholds: { ... }
};
```

---

## 5. Chaos Engineering (Avanzado)

### 5.1 Prerequisitos de Infraestructura

**Clúster Kubernetes de staging dedicado:**

```bash
# Verificar que Chaos Mesh está instalado
kubectl get pods -n chaos-mesh

# Si no está instalado:
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm install chaos-mesh chaos-mesh/chaos-mesh \
  --namespace chaos-mesh \
  --create-namespace \
  --version 2.6.3
```

### 5.2 Configurar el kubeconfig como Secret

```bash
# Generar un kubeconfig con acceso mínimo (solo el namespace de staging)
# Paso 1: Crear un ServiceAccount específico para CI
kubectl create serviceaccount chaos-ci-sa -n staging

# Paso 2: Crear Role con permisos mínimos necesarios
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: chaos-ci-role
  namespace: staging
rules:
- apiGroups: ["chaos-mesh.org"]
  resources: ["*"]
  verbs: ["get", "list", "create", "delete", "watch"]
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list"]
EOF

# Paso 3: Hacer el binding
kubectl create rolebinding chaos-ci-binding \
  --role=chaos-ci-role \
  --serviceaccount=staging:chaos-ci-sa \
  -n staging

# Paso 4: Exportar el kubeconfig para este SA y codificar en base64
kubectl config view --minify --flatten | base64 -w 0
# Guardar el output como secret KUBECONFIG_STAGING en GitHub
```

### 5.3 Ejemplo de Experimento Pod-Kill Manual

```bash
# Verificar que el experimento se puede aplicar
kubectl apply -f - <<EOF --dry-run=client
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: test-pod-kill
  namespace: staging
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces: [staging]
    labelSelectors:
      app: mi-api
  duration: "1m"
EOF

# Ver el estado del experimento
kubectl get podchaos -n staging

# Cancelar manualmente si algo va mal
kubectl delete podchaos test-pod-kill -n staging
```

---

## 6. Programación de Pruebas Nocturnas

### 6.1 Configuración del Schedule

```yaml
# En e2e-tests.yml — habilitar la ejecución nocturna:
on:
  schedule:
    - cron: "0 3 * * *"   # 03:00 UTC = 22:00 Lima (fuera del horario laboral)
```

### 6.2 Notificar Solo si Empeora (No Alertas Repetitivas)

```yaml
# Estrategia: comparar con el resultado del día anterior
# usando artefactos del último run exitoso

- name: "📊 Comparar con resultado anterior"
  run: |
    # Descargar el resumen del último run nocturno
    LAST_RUN=$(gh run list \
      --workflow=e2e-tests.yml \
      --event=schedule \
      --status=completed \
      --limit=1 \
      --json databaseId \
      --jq '.[0].databaseId' 2>/dev/null || echo "")

    if [ -z "$LAST_RUN" ]; then
      echo "No hay run anterior para comparar"
      echo "should_notify=true" >> "$GITHUB_ENV"
      exit 0
    fi

    # Para una comparación real, guardar el conteo de fallos como artefacto
    # y comparar con el conteo actual
    CURRENT_FAILURES=5  # Obtener del resultado real
    PREVIOUS_FAILURES=3 # Descargar del artefacto anterior

    if [ "$CURRENT_FAILURES" -gt "$PREVIOUS_FAILURES" ]; then
      echo "📈 Empeoramiento detectado: $PREVIOUS_FAILURES → $CURRENT_FAILURES fallos"
      echo "should_notify=true" >> "$GITHUB_ENV"
    else
      echo "✅ Sin empeoramiento: $PREVIOUS_FAILURES → $CURRENT_FAILURES fallos"
      echo "should_notify=false" >> "$GITHUB_ENV"
    fi
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## 7. Integración con CI/CD y Release

### 7.1 Llamar E2E desde release-on-demand.yml

```yaml
# En release-on-demand.yml — añadir como quality gate antes de producción:
jobs:
  release:
    # ... (jobs existentes de bump + tag)

  quality-gate-e2e:
    name: "🎭 Quality Gate — E2E"
    needs: [deploy-staging]
    uses: ./.github/workflows/e2e-tests.yml
    with:
      environment: staging
      test_suite: regression
      base_url: ""  # Usar la URL del Environment de staging
    secrets: inherit

  quality-gate-performance:
    name: "⚡ Quality Gate — Performance"
    needs: [deploy-staging]
    uses: ./.github/workflows/performance-tests.yml
    with:
      environment: staging
      test_type: smoke
      threshold_checks: true
    secrets: inherit

  deploy-production:
    name: "🚀 Deploy → Production"
    needs: [quality-gate-e2e, quality-gate-performance]
    if: |
      needs.quality-gate-e2e.result == 'success' &&
      needs.quality-gate-performance.result == 'success'
    # ... resto del job de deploy a producción
```

### 7.2 Ejecutar desde CLI con gh

```bash
# Disparar E2E manualmente y esperar el resultado
gh workflow run e2e-tests.yml \
  --repo MI_ORG/MI_REPO \
  --field environment=staging \
  --field test_suite=smoke \
  --field browser=chromium

# Esperar y ver los resultados
RUN_ID=$(gh run list --workflow=e2e-tests.yml --limit=1 --json databaseId -q '.[0].databaseId')
gh run watch "$RUN_ID"
gh run view "$RUN_ID" --log-failed
```

---

## 8. Solución de Problemas Comunes

### 8.1 Playwright: Navegadores No Encontrados

```bash
# Error típico: "Executable doesn't exist at..."
# Causa: la caché de navegadores es del runner anterior o no se instalaron

# Solución: forzar reinstalación limpiando la caché
- name: "🎭 Forzar instalación de navegadores"
  run: npx playwright install --with-deps --force
  if: steps.playwright-cache.outputs.cache-hit != 'true'
```

### 8.2 Videos E2E Demasiado Grandes

```typescript
// En playwright.config.ts — limitar la grabación:
use: {
  // Solo grabar en fallo (no todos los tests)
  video: {
    mode: 'retain-on-failure',
    size: { width: 1280, height: 720 },
  },
  // Reducir FPS para archivos más pequeños
  // (no hay opción directa en Playwright; usar una resolución menor)
}
```

```yaml
# En el workflow, comprimir los videos antes de subir:
- name: "🗜️ Comprimir artefactos de video"
  if: always()
  run: |
    find test-results/ -name "*.webm" -size +10M | \
      xargs -I{} ffmpeg -i {} -vf "scale=960:-1" -c:v libvpx-vp9 -b:v 0 -crf 33 {}.compressed.webm
```

### 8.3 k6: Memoria Insuficiente en el Runner

```bash
# Error: "signal: killed" o "cannot allocate memory"
# Causa: k6 con muchos VUs consume mucha RAM (cada VU ~1MB)
# 100 VUs + workers ≈ 500MB; el runner de GitHub tiene 7GB

# Diagnóstico: monitorear memoria durante el test
- name: "📊 Monitorear memoria"
  run: |
    while true; do
      echo "$(date): $(free -h | grep Mem)"
      sleep 5
    done &
    MONITOR_PID=$!

    k6 run --vus 100 --duration 5m k6-scripts/load.js

    kill $MONITOR_PID 2>/dev/null

# Solución para cargas muy altas: usar k6 Cloud
k6 run --out cloud k6-scripts/stress.js
# (requiere GRAFANA_CLOUD_TOKEN)
```

### 8.4 Timeouts en Tests E2E por Entorno Lento

```typescript
// En los tests individuales, aumentar el timeout para operaciones lentas:
test('checkout completo', async ({ page }) => {
  // Timeout específico para este test (sobreescribe el global)
  test.setTimeout(60000);  // 60 segundos

  await page.goto('/checkout');

  // Para esperas específicas en operaciones lentas:
  await page.waitForSelector('[data-testid="order-confirmed"]', {
    timeout: 30000  // 30 segundos para confirmación de orden
  });
});
```

### 8.5 Chaos: Error de Conexión al Clúster K8s

```bash
# Error: "Unable to connect to the server: dial tcp: i/o timeout"
# Causas posibles:
# 1. El kubeconfig en el secret es incorrecto
# 2. El endpoint del API server no es accesible desde GitHub Actions
# 3. El certificado del clúster expiró

# Diagnóstico:
- name: "🔍 Diagnóstico de conectividad K8s"
  run: |
    echo "=== kubeconfig ==="
    kubectl config view  # NO imprime credenciales, solo endpoints

    echo "=== Conectividad ==="
    kubectl cluster-info --request-timeout=10s || true

    echo "=== Namespaces ==="
    kubectl get namespaces --request-timeout=10s || true

# Solución 1: Si el API server no es público, usar un bastion/VPN
# Solución 2: Abrir el API server solo a los IPs de GitHub Actions
#   IPs de GitHub Actions: https://api.github.com/meta → "actions"
```

### 8.6 Debug con ACTIONS_RUNNER_DEBUG

```bash
# Activar debug completo del runner para un run específico:
# Actions → seleccionar run fallido → "Re-run jobs" → "Enable debug logging"

# O crear los secrets temporalmente:
# ACTIONS_RUNNER_DEBUG = true   → Logs del runner
# ACTIONS_STEP_DEBUG = true     → Logs de cada step

# En el workflow, imprimir variables de debug (nunca secrets):
- name: "🔍 Debug info"
  if: runner.debug == '1'
  run: |
    echo "=== Environment ==="
    env | grep -v -E "(TOKEN|SECRET|PASSWORD|KEY)" | sort

    echo "=== Node versions ==="
    node --version
    npm --version
    npx playwright --version 2>/dev/null || true

    echo "=== Disk space ==="
    df -h

    echo "=== Memory ==="
    free -h
```

---

## Apéndice: Variables de Entorno de los Workflows

### Variables para `e2e-tests.yml`

| Variable/Secret | Scope | Descripción |
|----------------|-------|-------------|
| `E2E_BASE_URL` | Variable (environment) | URL base del entorno |
| `E2E_TEST_USER` | Secret (environment) | Usuario de prueba |
| `E2E_TEST_PASSWORD` | Secret (environment) | Contraseña del usuario |
| `E2E_API_KEY` | Secret (environment) | API key para autenticación |
| `CYPRESS_RECORD_KEY` | Secret (repo) | Key del Cypress Dashboard |
| `SLACK_WEBHOOK_URL` | Secret (repo) | Webhook para notificaciones |
| `E2E_FRAMEWORK` | Variable (repo) | `playwright` \| `cypress` \| `selenium` |
| `E2E_RETRIES` | Variable (repo) | Número de reintentos (default: 2) |
| `E2E_TIMEOUT_MS` | Variable (repo) | Timeout por test en ms (default: 30000) |

### Variables para `performance-tests.yml`

| Variable/Secret | Scope | Descripción |
|----------------|-------|-------------|
| `PERF_TARGET_URL` | Variable (environment) | URL del servicio bajo prueba |
| `PERF_AUTH_TOKEN` | Secret (environment) | Token de autenticación para las peticiones |
| `K6_CLOUD_TOKEN` | Secret (repo) | Token de Grafana Cloud k6 |
| `SLACK_WEBHOOK_URL` | Secret (repo) | Webhook para notificaciones |

### Variables para `chaos-engineering.yml`

| Variable/Secret | Scope | Descripción |
|----------------|-------|-------------|
| `KUBECONFIG_STAGING` | Secret (repo) | kubeconfig del clúster de staging (base64) |
| `STAGING_URL` | Variable (environment staging) | URL del servicio para health checks |
| `SLACK_WEBHOOK_URL` | Secret (repo) | Webhook para notificaciones |

---

*Guía generada para Testing Especializado (E2E + Performance + Chaos) v1.0.0 — Módulo 6 de la Suite DevOps*

*Suite completa: ci-cd → dependabot → release-on-demand → iac-security → scheduled-security-scan → e2e-tests*
