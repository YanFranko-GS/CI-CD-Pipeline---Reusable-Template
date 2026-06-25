# Guía de Implementación: Generación y Publicación Automática de Documentación

> **Versión:** 1.0.0 | **Módulo 8 de la Suite DevOps**

---

## Tabla de Contenidos

1. [Elección de la Herramienta de Documentación](#1-elección-de-la-herramienta-de-documentación)
2. [Estructura Recomendada del Repositorio](#2-estructura-recomendada-del-repositorio)
3. [Configuración de GitHub Pages](#3-configuración-de-github-pages)
4. [Configuración de Netlify y Vercel](#4-configuración-de-netlify-y-vercel)
5. [Integración con el Pipeline CI/CD Existente](#5-integración-con-el-pipeline-cicd-existente)
6. [Generación de API Docs desde el Código](#6-generación-de-api-docs-desde-el-código)
7. [Pruebas Locales y Validación](#7-pruebas-locales-y-validación)
8. [Solución de Problemas Comunes](#8-solución-de-problemas-comunes)

---

## 1. Elección de la Herramienta de Documentación

### 1.1 Comparativa de Herramientas

| Herramienta | Lenguaje | Formato fuente | Ideal para | Curva de aprendizaje |
|-------------|---------|----------------|-----------|---------------------|
| **Docusaurus** | Node.js | MDX (Markdown + JSX) | Proyectos de software con blog, versionado | Media |
| **MkDocs + Material** | Python | Markdown puro | Documentación técnica elegante, rápida de escribir | Baja |
| **Sphinx** | Python | reStructuredText / MyST | Proyectos Python, documentación de APIs Python | Alta |
| **VitePress** | Node.js | Markdown + Vue | Proyectos Vue.js, documentación minimalista | Baja |
| **Astro Starlight** | Node.js | MDX | Documentación moderna con rendimiento excepcional | Media |
| **Javadoc** | Java | Comentarios Javadoc | Librerías Java, APIs JVM | Baja (ya en el código) |

**Criterios de selección:**

- **Usa MkDocs Material** si: tu equipo escribe en Markdown, quieres un resultado visualmente profesional con el mínimo esfuerzo, y el proyecto es de cualquier lenguaje.
- **Usa Docusaurus** si: necesitas versionado de documentación, un blog integrado, internacionalización, o la documentación tiene componentes interactivos (MDX).
- **Usa Sphinx** si: el proyecto es en Python y quieres documentación generada desde docstrings (`sphinx-autodoc`).
- **Usa VitePress** si: el proyecto es Vue.js o el equipo ya trabaja con Vite.
- **Usa Astro Starlight** si: priorizas rendimiento (Lighthouse 100) y tu equipo conoce Astro.
- **Usa Javadoc** si: el proyecto es Java/Kotlin y solo necesitas documentar la API pública.

### 1.2 Configuración Inicial por Herramienta

#### MkDocs + Material (recomendado para la mayoría de proyectos)

```bash
# Instalar localmente
pip install mkdocs mkdocs-material

# Inicializar el proyecto
mkdocs new .
# Crea: mkdocs.yml y docs/index.md
```

**`mkdocs.yml` completo y recomendado:**

```yaml
site_name: "Mi Proyecto — Documentación"
site_url: https://mi-org.github.io/mi-repo/
repo_url: https://github.com/mi-org/mi-repo
repo_name: mi-org/mi-repo
edit_uri: edit/main/docs/

theme:
  name: material
  language: es
  palette:
    - scheme: default
      primary: indigo
      accent: indigo
      toggle:
        icon: material/brightness-7
        name: Cambiar a modo oscuro
    - scheme: slate
      primary: indigo
      accent: indigo
      toggle:
        icon: material/brightness-4
        name: Cambiar a modo claro
  features:
    - navigation.tabs
    - navigation.sections
    - navigation.top
    - search.suggest
    - search.highlight
    - content.tabs.link
    - content.code.copy

plugins:
  - search:
      lang: es
  - git-revision-date-localized:
      enable_creation_date: true
  - minify:
      minify_html: true

markdown_extensions:
  - pymdownx.highlight:
      anchor_linenums: true
  - pymdownx.inlinehilite
  - pymdownx.snippets
  - pymdownx.superfences
  - admonition
  - footnotes
  - attr_list

nav:
  - Inicio: index.md
  - Guía de inicio rápido: getting-started.md
  - Referencia de API: api-reference.md
  - Contribuir: contributing.md

# Para publicar en subdirectorio de GitHub Pages:
# use_directory_urls: true
```

**Instalar dependencias en un archivo específico:**

```text
# docs/requirements.txt
mkdocs==1.6.0
mkdocs-material==9.5.0
mkdocs-material-extensions==1.3.1
pymdown-extensions==10.7
mkdocs-git-revision-date-localized-plugin==1.2.4
mkdocs-minify-plugin==0.8.0
mkdocs-redirects==1.2.1
```

#### Docusaurus 3.x

```bash
# Inicializar Docusaurus en la carpeta docs/
npx create-docusaurus@latest docs classic --typescript

# Estructura creada:
# docs/
#   docusaurus.config.ts
#   docs/intro.md
#   src/pages/index.tsx
#   static/
#   package.json
```

**`docusaurus.config.ts` básico:**

```typescript
import type {Config} from '@docusaurus/types';
import type * as Preset from '@docusaurus/preset-classic';

const config: Config = {
  title: 'Mi Proyecto',
  tagline: 'Documentación oficial',
  url: 'https://mi-org.github.io',
  baseUrl: '/mi-repo/',  // ← Importante para GitHub Pages en subdirectorio
  organizationName: 'mi-org',
  projectName: 'mi-repo',

  onBrokenLinks: 'throw',  // Fallar si hay links rotos
  onBrokenMarkdownLinks: 'warn',

  i18n: {
    defaultLocale: 'es',
    locales: ['es', 'en'],
  },

  presets: [
    ['classic', {
      docs: {
        sidebarPath: './sidebars.ts',
        editUrl: 'https://github.com/mi-org/mi-repo/tree/main/docs/',
      },
      blog: false,  // Desactivar blog si no se necesita
      theme: {
        customCss: './src/css/custom.css',
      },
    } satisfies Preset.Options],
  ],
};

export default config;
```

#### Sphinx (Python)

```bash
# Instalar
pip install sphinx sphinx-rtd-theme myst-parser

# Inicializar (desde la carpeta docs/)
mkdir docs && cd docs
sphinx-quickstart --quiet --project="Mi Proyecto" --author="Mi Equipo" --language=es
```

**`docs/conf.py` recomendado:**

```python
project = 'Mi Proyecto'
copyright = '2025, Mi Equipo'
author = 'Mi Equipo'
release = '1.0.0'

extensions = [
    'sphinx.ext.autodoc',         # Documentar desde docstrings
    'sphinx.ext.viewcode',        # Links al código fuente
    'sphinx.ext.napoleon',        # Soporte para Google/NumPy docstrings
    'sphinx.ext.intersphinx',     # Links a docs de otras librerías
    'myst_parser',                # Soporte para Markdown
]

templates_path = ['_templates']
exclude_patterns = ['_build', 'Thumbs.db', '.DS_Store']

html_theme = 'sphinx_rtd_theme'
html_static_path = ['_static']

# Para autodoc
autodoc_default_options = {
    'members': True,
    'undoc-members': False,
    'show-inheritance': True,
}
```

---

## 2. Estructura Recomendada del Repositorio

```
mi-proyecto/
├── docs/                       ← Fuentes de documentación
│   ├── index.md               ← Página principal
│   ├── getting-started.md
│   ├── api/
│   │   └── reference.md
│   ├── guides/
│   ├── requirements.txt       ← Deps de MkDocs/Sphinx
│   └── conf.py                ← Config de Sphinx (si aplica)
├── src/                       ← Código fuente
│   └── (código con JSDoc/docstrings/Javadoc)
├── api/                       ← Especificaciones de API
│   └── openapi.yaml
├── mkdocs.yml                 ← Configuración MkDocs (raíz)
├── docusaurus.config.ts       ← Configuración Docusaurus (raíz o en docs/)
├── package.json               ← Para Docusaurus/VitePress
├── .github/
│   └── workflows/
│       ├── deploy-docs.yml    ← Este workflow
│       └── api-docs.yml       ← Este workflow
└── README.md
```

**Variables de repositorio a configurar:**

```
Settings → Secrets and variables → Actions → Variables

DOCS_TOOL         = mkdocs          (o docusaurus, sphinx, javadoc)
DOCS_DEPLOY_TARGET = github_pages   (o netlify, vercel)
DOCS_DIR          = docs             (directorio fuente)
DOCS_BUILD_DIR    = site             (directorio output, si no es el default)
DOCS_BASE_URL     = /mi-repo/        (para subdirectorios en GitHub Pages)
API_TITLE         = Mi API           (título del portal de API)
SLACK_NOTIFY_DOCS = true             (activar notificaciones)
```

---

## 3. Configuración de GitHub Pages

### 3.1 Habilitar GitHub Pages con Fuente "GitHub Actions"

Este paso es **obligatorio** antes de que el workflow pueda publicar:

```
1. Ir al repositorio en GitHub
2. Settings → Pages
3. Source: cambiar de "Deploy from a branch" a "GitHub Actions"
4. Guardar
```

Una vez configurado, GitHub Pages usará el artefacto subido por `actions/upload-pages-artifact` en lugar de una rama específica.

### 3.2 Entender el Flujo de GitHub Pages

```
upload-pages-artifact@v3    ← Empaqueta el directorio en un artefacto especial
         ↓
deploy-pages@v4             ← Despliega el artefacto en Pages (requiere id-token: write)
         ↓
GitHub CDN                  ← Sirve el sitio en https://<org>.github.io/<repo>/
```

### 3.3 Configurar el `baseUrl` para Subdirectorios

Si el sitio se publica en `https://mi-org.github.io/mi-repo/` (no en la raíz), necesitas configurar el baseUrl en cada herramienta:

**MkDocs:**
```yaml
# mkdocs.yml
site_url: https://mi-org.github.io/mi-repo/
```

**Docusaurus:**
```typescript
// docusaurus.config.ts
baseUrl: '/mi-repo/',
```

**VitePress:**
```typescript
// docs/.vitepress/config.ts
export default {
  base: '/mi-repo/',
}
```

**Sphinx:**
```python
# docs/conf.py
# Sphinx no necesita baseUrl; GitHub Pages maneja las rutas
```

### 3.4 Dominio Personalizado

Para usar `docs.mi-empresa.com` en lugar de `mi-org.github.io/mi-repo/`:

```
1. Settings → Pages → Custom domain: docs.mi-empresa.com
2. En tu proveedor DNS, añadir:
   - CNAME: docs → mi-org.github.io
   - O registros A/AAAA apuntando a las IPs de GitHub Pages

3. Esperar a que GitHub verifique el dominio (15-60 min)
4. Marcar "Enforce HTTPS"
```

Para que el workflow respete el dominio personalizado, crear el archivo `CNAME` en el directorio de build:

```yaml
# En el workflow, añadir después del build:
- name: "🌐 Configurar dominio personalizado"
  if: vars.CUSTOM_DOMAIN != ''
  run: |
    echo "${{ vars.CUSTOM_DOMAIN }}" > ${{ steps.detect.outputs.build_dir }}/CNAME
```

---

## 4. Configuración de Netlify y Vercel

### 4.1 Netlify

**Obtener los tokens:**

```
1. netlify.com → User Settings → Applications → New access token
   → Nombre: "GitHub Actions docs deploy"
   → Copiar: netlify_pat_xxxxxxxxxxxxxxxxxxxxxxxx
   → Guardar como secret: NETLIFY_AUTH_TOKEN

2. netlify.com → Sites → tu sitio → Site Settings → General → Site information
   → Copiar: Site ID (xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx)
   → Guardar como secret: NETLIFY_SITE_ID
```

**Ventajas de Netlify sobre GitHub Pages:**
- **Deploy previews**: cada PR tiene su propia URL de preview automáticamente
- **Rollbacks**: con un clic en la UI de Netlify
- **Edge functions, formularios, redirects**: sin configuración adicional
- **Analytics**: integradas sin JavaScript

**`netlify.toml` opcional para configuración avanzada:**

```toml
[build]
  publish = "build"
  command = "mkdocs build"

[[redirects]]
  from = "/docs/*"
  to = "/:splat"
  status = 301

[[headers]]
  for = "/*"
  [headers.values]
    X-Frame-Options = "DENY"
    X-XSS-Protection = "1; mode=block"
    Cache-Control = "public, max-age=3600"

[[headers]]
  for = "*.js"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"
```

### 4.2 Vercel

```bash
# Opción A: Crear el proyecto via CLI
npm install -g vercel
vercel login
vercel --cwd build  # Apuntar al directorio de build de la documentación

# Opción B: Crear en la UI
# vercel.com → New Project → Import Git Repository
```

**Obtener los tokens:**
```
VERCEL_TOKEN:
  vercel.com → Settings → Tokens → Create Token
  → Nombre: "GitHub Actions Docs"
  → Scope: Full Account
  → Copiar el token generado

VERCEL_ORG_ID:
  vercel.com → Settings → General → Team ID (o your Personal Account ID)

VERCEL_PROJECT_ID:
  vercel.com → [tu proyecto] → Settings → General → Project ID
```

---

## 5. Integración con el Pipeline CI/CD Existente

### 5.1 Invocar desde release-on-demand.yml

```yaml
# En release-on-demand.yml, añadir después del deploy de producción:
jobs:
  deploy-production:
    # ... (deploy existente)

  update-docs:
    name: "📚 Actualizar Documentación"
    needs: deploy-production
    if: needs.deploy-production.result == 'success'
    uses: ./.github/workflows/deploy-docs.yml
    with:
      doc_tool: "auto-detect"
      deploy_target: "github_pages"
    secrets: inherit
```

### 5.2 Separar Documentación del Build Principal

Para no ralentizar el CI en cada push, el workflow `deploy-docs.yml` ya tiene `paths:` configurado para activarse solo cuando cambian archivos de documentación. Pero si también quieres actualizarla en cada release:

```yaml
# En ci-cd.yml, añadir un trigger adicional al deploy-docs.yml
# Para activarlo via repository dispatch al finalizar un release:
- name: "📚 Disparar actualización de docs"
  if: success() && github.ref == 'refs/heads/main'
  uses: actions/github-script@v7
  with:
    script: |
      await github.rest.repos.createDispatchEvent({
        owner: context.repo.owner,
        repo: context.repo.repo,
        event_type: 'docs-update',
        client_payload: {
          version: '${{ needs.release.outputs.new_version }}'
        }
      });
```

```yaml
# En deploy-docs.yml, añadir el trigger:
on:
  repository_dispatch:
    types: [docs-update]
  # ... resto de triggers
```

---

## 6. Generación de API Docs desde el Código

### 6.1 FastAPI → OpenAPI

FastAPI genera el spec automáticamente. Para extraerlo en CI:

```python
# extract_openapi.py (en la raíz del proyecto)
"""Script para exportar el spec OpenAPI de FastAPI a un archivo."""
import json
import sys
import yaml

# Importar tu app FastAPI
from main import app  # Ajustar según la estructura de tu proyecto

def extract(output_format: str = "yaml"):
    spec = app.openapi()
    output_file = f"api-spec.{output_format}"

    if output_format == "json":
        with open(output_file, "w", encoding="utf-8") as f:
            json.dump(spec, f, indent=2, ensure_ascii=False)
    else:
        with open(output_file, "w", encoding="utf-8") as f:
            yaml.dump(spec, f, default_flow_style=False, allow_unicode=True)

    print(f"✅ OpenAPI spec exportado: {output_file}")
    return output_file

if __name__ == "__main__":
    fmt = sys.argv[1] if len(sys.argv) > 1 else "yaml"
    extract(fmt)
```

### 6.2 Express + swagger-jsdoc → OpenAPI

```javascript
// scripts/generate-openapi.js
const swaggerJsdoc = require('swagger-jsdoc');
const fs = require('fs');
const yaml = require('js-yaml');
const path = require('path');

const options = {
  definition: {
    openapi: '3.0.0',
    info: {
      title: require('../package.json').name,
      version: require('../package.json').version,
      description: require('../package.json').description,
      contact: {
        name: 'API Support',
        email: 'api@mi-empresa.com',
      },
    },
    servers: [
      { url: process.env.API_URL || 'http://localhost:3000', description: 'Development' },
      { url: 'https://api.mi-empresa.com', description: 'Production' },
    ],
  },
  // Escanear todos los archivos de rutas para encontrar anotaciones JSDoc
  apis: [
    './src/routes/**/*.js',
    './src/routes/**/*.ts',
    './src/controllers/**/*.js',
    './src/controllers/**/*.ts',
  ],
};

const spec = swaggerJsdoc(options);
const format = process.argv[2] || 'yaml';

if (format === 'json') {
  fs.writeFileSync('api-spec.json', JSON.stringify(spec, null, 2));
} else {
  fs.writeFileSync('api-spec.yaml', yaml.dump(spec));
}

console.log(`✅ OpenAPI spec generado: api-spec.${format}`);
```

**Añadir el script a package.json:**

```json
{
  "scripts": {
    "generate-spec": "node scripts/generate-openapi.js yaml",
    "generate-spec:json": "node scripts/generate-openapi.js json"
  }
}
```

**Ejemplo de anotación JSDoc en Express:**

```javascript
/**
 * @swagger
 * /api/users/{id}:
 *   get:
 *     summary: Obtener un usuario por ID
 *     tags: [Users]
 *     parameters:
 *       - in: path
 *         name: id
 *         required: true
 *         schema:
 *           type: string
 *         description: ID del usuario
 *     responses:
 *       200:
 *         description: Usuario encontrado
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/User'
 *       404:
 *         description: Usuario no encontrado
 */
router.get('/users/:id', async (req, res) => { ... });
```

### 6.3 Spring Boot + SpringDoc → OpenAPI

**`pom.xml` (añadir dependencia):**

```xml
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>2.5.0</version>
</dependency>
```

**`application-apidocs.properties` (perfil para generar el spec):**

```properties
springdoc.api-docs.path=/v3/api-docs
springdoc.swagger-ui.path=/swagger-ui.html
springdoc.swagger-ui.enabled=true
springdoc.packages-to-scan=com.mi.empresa.controllers
```

**Script de extracción para CI (sin arrancar el servidor completo):**

```xml
<!-- pom.xml: añadir el plugin de SpringDoc para generación offline -->
<plugin>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-maven-plugin</artifactId>
  <version>1.4</version>
  <executions>
    <execution>
      <id>integration-test</id>
      <goals><goal>generate</goal></goals>
    </execution>
  </executions>
  <configuration>
    <apiDocsUrl>http://localhost:8080/v3/api-docs</apiDocsUrl>
    <outputFileName>api-spec.json</outputFileName>
    <outputDir>${project.build.directory}</outputDir>
  </configuration>
</plugin>
```

```bash
# Generar el spec con el plugin (arranca la app temporalmente)
mvn verify -Dspring.profiles.active=apidocs springdoc-openapi:generate
```

### 6.4 Mantener el Spec Sincronizado

Una buena práctica es **commitear el spec generado** al repositorio para:
1. Revisión en PRs ("esta PR cambia la API de esta forma")
2. Detección de cambios breaking (comparar el spec antes/después)

```yaml
# Añadir al workflow api-docs.yml para commitear el spec si cambió:
- name: "💾 Commitear spec si cambió"
  run: |
    SPEC_FILE="${{ steps.generate.outputs.spec_file }}"

    # Copiar el spec al directorio de la API
    mkdir -p api/
    cp "$SPEC_FILE" "api/openapi.yaml"

    # Verificar si hay cambios
    if git diff --quiet api/openapi.yaml 2>/dev/null; then
      echo "ℹ️ El spec no cambió — no se hace commit"
    else
      git config user.name  "github-actions[bot]"
      git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
      git add api/openapi.yaml
      git commit -m "chore(api): update OpenAPI spec [skip ci]"
      git push origin HEAD:${{ github.ref_name }}
      echo "✅ Spec commiteado a api/openapi.yaml"
    fi
```

---

## 7. Pruebas Locales y Validación

### 7.1 Probar la Generación Localmente

```bash
# MkDocs: servidor con hot-reload
mkdocs serve
# Abre: http://127.0.0.1:8000/

# Docusaurus: servidor de desarrollo
cd docs && npm start
# Abre: http://localhost:3000/

# Sphinx: build y servidor
cd docs && make html && python -m http.server 8080 --directory _build/html
# Abre: http://localhost:8080/

# VitePress:
npm run docs:dev
```

### 7.2 Simular el Workflow con Docker

```bash
# Simular el deploy de MkDocs localmente con el mismo entorno que GitHub Actions
docker run --rm \
  -v "$(pwd):/docs" \
  -p 8000:8000 \
  squidfunk/mkdocs-material \
  serve --dev-addr=0.0.0.0:8000
```

### 7.3 Workflow de Validación de Calidad de Documentación

```yaml
# .github/workflows/lint-docs.yml
name: "🔍 Lint Documentation"
on:
  pull_request:
    paths: ["docs/**", "*.md"]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Verificar links rotos en Markdown
      - name: "🔗 Check broken links"
        uses: gaurav-nelson/github-action-markdown-link-check@v1
        with:
          use-quiet-mode: 'no'
          config-file: '.mlc_config.json'

      # Linting de Markdown con markdownlint
      - name: "📝 markdownlint"
        uses: DavidAnson/markdownlint-cli2-action@v16
        with:
          globs: "docs/**/*.md"

      # Vale: linter de prosa (estilo de escritura)
      - name: "✍️ Vale (estilo de escritura)"
        uses: errata-ai/vale-action@reviewdog
        with:
          files: docs/
          reporter: github-pr-review
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**`.mlc_config.json`** (configuración de link checker):

```json
{
  "ignorePatterns": [
    { "pattern": "^https://localhost" },
    { "pattern": "^http://localhost" }
  ],
  "replacementPatterns": [
    {
      "pattern": "^/",
      "replacement": "https://github.com/mi-org/mi-repo/blob/main/"
    }
  ],
  "retryOn429": true,
  "retryCount": 3,
  "httpHeaders": [
    {
      "urls": ["https://github.com"],
      "headers": {
        "Authorization": "Bearer ${GITHUB_TOKEN}"
      }
    }
  ]
}
```

---

## 8. Solución de Problemas Comunes

### 8.1 Error: "No such file or directory: build/"

**Causa:** El comando de build no encontró los archivos de configuración o falló silenciosamente.

```bash
# Diagnóstico: ejecutar el build localmente
mkdocs build --verbose
npm run build 2>&1 | tail -30

# En el workflow, añadir diagnóstico antes del deploy:
- name: "🔍 Diagnóstico de build"
  if: failure()
  run: |
    echo "Estructura de directorios:"
    find . -type f -name "*.html" | head -20
    echo ""
    echo "Contenido del directorio de build esperado:"
    ls -la "${{ steps.detect.outputs.build_dir }}" 2>/dev/null || echo "No existe"
    echo ""
    echo "Herramienta detectada: ${{ steps.detect.outputs.doc_tool }}"
```

**Soluciones comunes:**

```bash
# MkDocs: verificar que mkdocs.yml está en la raíz
ls mkdocs.yml  # Debe existir

# Docusaurus: verificar que el script build existe en package.json
cat package.json | jq '.scripts.build'
# Debe ser: "docusaurus build"

# Sphinx: verificar que conf.py existe
ls docs/conf.py
```

### 8.2 Error al Desplegar en GitHub Pages: "Resource not accessible by integration"

**Causa:** La fuente de GitHub Pages no está configurada como "GitHub Actions".

```
Settings → Pages → Source → GitHub Actions  ← CAMBIAR ESTO
```

También puede ser un problema de permisos del workflow:

```yaml
# Verificar que el workflow tiene estos permisos:
permissions:
  pages: write
  id-token: write
  contents: read
```

### 8.3 Token de Netlify/Vercel Incorrecto o Caducado

```bash
# Verificar el token de Netlify manualmente:
curl -H "Authorization: Bearer $NETLIFY_AUTH_TOKEN" \
  "https://api.netlify.com/api/v1/user" | jq '.email'
# Si retorna null o error: el token es inválido

# Verificar el Site ID:
curl -H "Authorization: Bearer $NETLIFY_AUTH_TOKEN" \
  "https://api.netlify.com/api/v1/sites/$NETLIFY_SITE_ID" | jq '.name'
# Debe retornar el nombre del sitio
```

**Renovar el token:**
```
netlify.com → User Settings → Applications → [token expirado] → Revoke
                                                               → New access token
```

### 8.4 Limpiar la Caché de GitHub Pages

Si la documentación no se actualiza aunque el deploy fue exitoso:

```bash
# Opción 1: Forzar re-deploy desde la UI
# Actions → [workflow run] → Re-run all jobs

# Opción 2: Hacer un cambio trivial para forzar invalidación
echo "# $(date)" >> docs/index.md
git add docs/index.md
git commit -m "docs: force cache invalidation [skip ci]"
git push

# Opción 3: Verificar en el Settings → Pages que el deploy fue reciente
```

### 8.5 Docusaurus falla con "baseUrl not found"

```typescript
// Si el sitio está en https://org.github.io/repo/
// la URL base debe incluir el nombre del repo:
baseUrl: '/repo/',  // ← Con slashes inicial y final

// El error "broken anchor" o "broken link" en Docusaurus con --strict:
// Desactivar temporalmente para diagnosticar:
// onBrokenLinks: 'warn',  // en lugar de 'throw'
```

### 8.6 auto-detect no detecta la herramienta correcta

Si tienes múltiples herramientas (ej. `package.json` Y `mkdocs.yml`):

```bash
# Especificar explícitamente via variable de repositorio:
# Settings → Variables → DOCS_TOOL = mkdocs

# O via input en workflow_dispatch:
# doc_tool: mkdocs
```

---

## Apéndice: Tabla de Variables y Secrets

### Variables del repositorio (no secretas)

| Variable | Descripción | Valor ejemplo |
|----------|-------------|---------------|
| `DOCS_TOOL` | Herramienta de documentación | `mkdocs` |
| `DOCS_DEPLOY_TARGET` | Destino de publicación | `github_pages` |
| `DOCS_DIR` | Directorio fuente | `docs` |
| `DOCS_BUILD_DIR` | Directorio de output | `site` |
| `DOCS_BASE_URL` | URL base (subdirectorio) | `/mi-repo/` |
| `DOCS_SITE_URL` | URL completa del sitio | `https://mi-org.github.io/mi-repo/` |
| `CUSTOM_DOMAIN` | Dominio personalizado | `docs.mi-empresa.com` |
| `API_TITLE` | Título del portal de API | `Mi Proyecto API` |
| `API_TYPE` | Tipo de API | `openapi` |
| `API_FRAMEWORK` | Framework del servidor | `fastapi` |
| `API_PORTAL_TYPE` | Tipo de portal interactivo | `redoc` |
| `API_OUTPUT_FORMAT` | Formato del spec | `yaml` |
| `SLACK_NOTIFY_DOCS` | Notificar en Slack | `true` |

### Secrets requeridos por destino

| Secret | Requerido para | Cómo obtenerlo |
|--------|---------------|----------------|
| `NETLIFY_AUTH_TOKEN` | Netlify | netlify.com → User Settings → Applications |
| `NETLIFY_SITE_ID` | Netlify | Site Settings → General → Site ID |
| `VERCEL_TOKEN` | Vercel | vercel.com → Settings → Tokens |
| `VERCEL_ORG_ID` | Vercel | vercel.com → Settings → General |
| `VERCEL_PROJECT_ID` | Vercel | Project → Settings → General |
| `SLACK_WEBHOOK_URL` | Notificaciones | Slack API → Incoming Webhooks |

---

*Guía generada para Deploy Documentation & API Docs v1.0.0 — Módulo 8 de la Suite DevOps*

*Suite completa: ci-cd → dependabot → release-on-demand → iac-security → scheduled-security-scan → e2e-tests → terraform/ansible → deploy-docs/api-docs*
