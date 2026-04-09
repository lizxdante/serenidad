# EP-CS06 — Deployment, CI/CD & Monitoring

**Épica:** Como DevOps, necesito la landing page desplegada en Cloudflare Pages con custom domain `sereni.dad`, CI/CD automatizado desde GitLab, Lighthouse CI gating, y monitoreo básico — para que cada push a main despliegue automáticamente con garantías de calidad.

**Prioridad:** Must Have — Producción
**Sprint:** Sprint 3
**Dependencias Entrantes:** EP-CS02 (landing UI), EP-CS05 (WhatsApp integration)
**Dependencias Salientes:** Ninguna (entrega final)

---

## HU-CS06.1 — Cloudflare Pages Deployment

**Como** DevOps,
**quiero** configurar Cloudflare Pages con custom domain `sereni.dad`, HTTPS automático, y D1 binding,
**para que** la landing page sea accesible públicamente con SSL y el formulario funcione.

### Tareas y Subtareas

#### T-CS06.1.1 — Configurar proyecto Cloudflare Pages

- **ST-CS06.1.1.1** — Crear proyecto via wrangler CLI.
  ```bash
  cd frontend/comingsoon
  npx wrangler pages project create serenidad-comingsoon --production-branch main
  ```
  - **CA:** Proyecto `serenidad-comingsoon` creado en CF Pages.

- **ST-CS06.1.1.2** — Primer deploy manual.
  ```bash
  npm run build
  npx wrangler pages deploy dist/ --project-name serenidad-comingsoon
  ```
  - **CA:** Deploy exitoso, URL temporal generada (e.g., `serenidad-comingsoon.pages.dev`).

#### T-CS06.1.2 — Configurar custom domain

- **ST-CS06.1.2.1** — Agregar custom domain en CF Pages.
  ```bash
  npx wrangler pages project set-deployment-config \
    --project-name serenidad-comingsoon \
    --domain sereni.dad
  ```
  O vía dashboard: Pages → serenidad-comingsoon → Custom domains → Add `sereni.dad`.

  - **CA:** Domain `sereni.dad` configurado.

- **ST-CS06.1.2.2** — Verificar DNS en Cloudflare.
  ```
  Type: CNAME
  Name: sereni.dad (or @)
  Target: serenidad-comingsoon.pages.dev
  Proxy: Enabled (orange cloud)
  ```
  - **CA:** DNS CNAME configurado con proxy habilitado.

- **ST-CS06.1.2.3** — Verificar HTTPS.
  ```bash
  curl -sI https://sereni.dad | head -5
  # HTTP/2 200
  # server: cloudflare
  # cf-ray: ...
  ```
  - **CA:** HTTPS funciona con certificado de Cloudflare.

#### T-CS06.1.3 — Configurar environment variables

- **ST-CS06.1.3.1** — Configurar variables de producción.
  ```bash
  # Via dashboard: Pages → serenidad-comingsoon → Settings → Environment variables

  # Production variables:
  WHATSAPP_NUMBER=5215512345678
  NODE_VERSION=22
  ```
  - **CA:** Variables configuradas en producción.

- **ST-CS06.1.3.2** — Configurar variables de preview.
  ```bash
  # Preview variables (for PR previews):
  WHATSAPP_NUMBER=5215512345678
  NODE_VERSION=22
  ```
  - **CA:** Variables configuradas para preview deployments.

#### T-CS06.1.4 — Configurar D1 binding en Pages

- **ST-CS06.1.4.1** — Vincular D1 al proyecto Pages.
  ```bash
  npx wrangler pages secret put DB --project-name serenidad-comingsoon
  ```
  O vía `wrangler.toml` (ya configurado en EP-CS01).

  - **CA:** D1 binding funciona en Pages Functions.

#### T-CS06.1.5 — Verificar deployment end-to-end

- **ST-CS06.1.5.1** — Verificar landing page.
  ```bash
  curl -s https://sereni.dad | grep "Serenidad"
  # Debe encontrar el título
  ```
  - **CA:** Landing page accesible en `https://sereni.dad`.

- **ST-CS06.1.5.2** — Verificar API endpoint.
  ```bash
  curl -X POST https://sereni.dad/api/lead \
    -H "Content-Type: application/json" \
    -d '{"name":"Test Deploy","email":"test@deploy.com","phone":"+5215512345678"}'
  # 201 Created
  ```
  - **CA:** API endpoint funciona en producción.

- **ST-CS06.1.5.3** — Verificar WhatsApp redirect.
  ```bash
  # El response debe incluir whatsapp_url
  curl -s -X POST https://sereni.dad/api/lead \
    -H "Content-Type: application/json" \
    -d '{"name":"Test WA","email":"testwa@deploy.com","phone":"+5215598765432"}' \
    | grep "wa.me"
  ```
  - **CA:** Response contiene URL de WhatsApp válida.

### Criterios de Aceptación — HU-CS06.1

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Landing page accesible | `curl https://sereni.dad` → 200 |
| 2 | HTTPS funciona | Certificado SSL válido |
| 3 | Custom domain activo | `sereni.dad` resuelve |
| 4 | API endpoint funciona | POST /api/lead → 201 |
| 5 | D1 binding funciona | Lead se guarda en base de datos |
| 6 | WhatsApp URL generada | Response contiene wa.me |
| 7 | Environment vars cargadas | WHATSAPP_NUMBER accesible |

### Definition of Done — HU-CS06.1

- [ ] Proyecto Cloudflare Pages creado
- [ ] Custom domain `sereni.dad` configurado con HTTPS
- [ ] Environment variables configuradas
- [ ] D1 binding funcional
- [ ] Landing page accesible públicamente
- [ ] API endpoint funcional en producción
- [ ] WhatsApp redirect funciona

---

## HU-CS06.2 — GitLab CI/CD Pipeline

**Como** desarrollador,
**quiero** un pipeline CI/CD que valide el build, ejecute Lighthouse CI, y despliegue automáticamente a Cloudflare Pages,
**para que** cada cambio pase por quality gates antes de llegar a producción.

### Tareas y Subtareas

#### T-CS06.2.1 — Crear pipeline de CI/CD

- **ST-CS06.2.1.1** — Crear `.gitlab-ci.yml` en la raíz del monorepo (o `frontend/comingsoon/.gitlab-ci.yml` si es scoped).

  ```yaml
  # GitLab CI/CD — Serenidad Coming Soon Landing Page
  # Triggers: changes in frontend/comingsoon/**

  stages:
    - validate
    - build
    - test
    - deploy

  variables:
    NODE_VERSION: "22"
    CF_PROJECT: "serenidad-comingsoon"

  # Only run when landing page files change
  .landing-rules:
    rules:
      - changes:
          - frontend/comingsoon/**/*
      - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      - if: '$CI_COMMIT_BRANCH == "main"'

  # ── Stage: Validate ────────────────────────────────────────────────────
  validate:types:
    stage: validate
    image: node:${NODE_VERSION}
    extends: .landing-rules
    script:
      - cd frontend/comingsoon
      - npm ci
      - npx tsc --noEmit
    cache:
      key: ${CI_COMMIT_REF_SLUG}-comingsoon
      paths:
        - frontend/comingsoon/node_modules/

  # ── Stage: Build ───────────────────────────────────────────────────────
  build:landing:
    stage: build
    image: node:${NODE_VERSION}
    extends: .landing-rules
    script:
      - cd frontend/comingsoon
      - npm ci
      - npm run build
    artifacts:
      paths:
        - frontend/comingsoon/dist/
      expire_in: 1 hour
    cache:
      key: ${CI_COMMIT_REF_SLUG}-comingsoon
      paths:
        - frontend/comingsoon/node_modules/

  # ── Stage: Test ────────────────────────────────────────────────────────
  test:lighthouse:
    stage: test
    image: node:${NODE_VERSION}
    extends: .landing-rules
    needs:
      - build:landing
    script:
      - cd frontend/comingsoon
      - npm ci
      - npx serve dist/ -l 4321 &
      - sleep 3
      - npx lighthouse http://localhost:4321
          --output=json
          --output-path=./lighthouse-results.json
          --chrome-flags="--headless --no-sandbox"
          --only-categories=performance,accessibility,seo
      - |
        # Parse and assert scores
        SCORES=$(node -e "
          const r = require('./lighthouse-results.json');
          const cats = r.categories;
          console.log(JSON.stringify({
            performance: cats.performance.score,
            accessibility: cats.accessibility.score,
            seo: cats.seo.score
          }));
        ")
        echo "Lighthouse Scores: $SCORES"
        node -e "
          const s = JSON.parse('$SCORES');
          let fail = false;
          if (s.performance < 0.90) { console.error('Performance:', s.performance, '< 0.90'); fail = true; }
          if (s.accessibility < 0.90) { console.error('Accessibility:', s.accessibility, '< 0.90'); fail = true; }
          if (s.seo < 0.90) { console.error('SEO:', s.seo, '< 0.90'); fail = true; }
          if (fail) process.exit(1);
          console.log('All Lighthouse checks passed!');
        "
    artifacts:
      paths:
        - frontend/comingsoon/lighthouse-results.json
      expire_in: 7 days
      when: always

  # ── Stage: Deploy ──────────────────────────────────────────────────────
  deploy:production:
    stage: deploy
    image: node:${NODE_VERSION}
    extends: .landing-rules
    needs:
      - build:landing
      - test:lighthouse
    rules:
      - if: '$CI_COMMIT_BRANCH == "main"'
        changes:
          - frontend/comingsoon/**/*
    environment:
      name: production
      url: https://sereni.dad
    script:
      - cd frontend/comingsoon
      - npm ci
      - npm run build
      - npx wrangler pages deploy dist/
          --project-name ${CF_PROJECT}
          --commit-message "${CI_COMMIT_SHORT_SHA} ${CI_COMMIT_TITLE}"
    variables:
      CLOUDFLARE_API_TOKEN: ${CF_API_TOKEN}
      CLOUDFLARE_ACCOUNT_ID: ${CF_ACCOUNT_ID}

  deploy:preview:
    stage: deploy
    image: node:${NODE_VERSION}
    extends: .landing-rules
    needs:
      - build:landing
    rules:
      - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
        changes:
          - frontend/comingsoon/**/*
    environment:
      name: preview/${CI_MERGE_REQUEST_IID}
      url: https://${CF_PROJECT}-preview-${CI_MERGE_REQUEST_IID}.pages.dev
    script:
      - cd frontend/comingsoon
      - npm ci
      - npm run build
      - npx wrangler pages deploy dist/
          --project-name ${CF_PROJECT}
          --branch preview-${CI_MERGE_REQUEST_IID}
          --commit-message "MR ${CI_MERGE_REQUEST_IID}: ${CI_COMMIT_TITLE}"
    variables:
      CLOUDFLARE_API_TOKEN: ${CF_API_TOKEN}
      CLOUDFLARE_ACCOUNT_ID: ${CF_ACCOUNT_ID}
  ```
  - **CA:** Pipeline CI/CD creado con 4 stages: validate, build, test, deploy.
  - **CA:** Lighthouse CI gating con thresholds (Performance ≥ 90, Accessibility ≥ 90, SEO ≥ 90).
  - **CA:** Preview deployments automáticos para merge requests.
  - **CA:** Production deploy solo desde `main` branch.

#### T-CS06.2.2 — Configurar CI/CD variables en GitLab

- **ST-CS06.2.2.1** — Configurar variables en GitLab CI/CD Settings.
  ```
  Settings → CI/CD → Variables:
  - CF_API_TOKEN: Cloudflare API token (masked, protected)
  - CF_ACCOUNT_ID: Cloudflare account ID (protected)
  ```
  - **CA:** Variables configuradas y protegidas.

- **ST-CS06.2.2.2** — Crear Cloudflare API token.
  ```bash
  # En Cloudflare dashboard:
  # My Profile → API Tokens → Create Token
  # Permissions: Cloudflare Pages - Edit
  # Resources: serenidad-comingsoon project
  ```
  - **CA:** Token creado con permisos mínimos.

#### T-CS06.2.3 — Configurar Lighthouse CI

- **ST-CS06.2.3.1** — Crear `frontend/comingsoon/lighthouserc.json`.
  ```json
  {
    "ci": {
      "collect": {
        "numberOfRuns": 3
      },
      "assert": {
        "assertions": {
          "categories:performance": ["error", { "minScore": 0.9 }],
          "categories:accessibility": ["error", { "minScore": 0.9 }],
          "categories:seo": ["error", { "minScore": 0.9 }],
          "categories:best-practices": ["warn", { "minScore": 0.9 }],
          "largest-contentful-paint": ["error", { "maxNumericValue": 2000 }],
          "cumulative-layout-shift": ["error", { "maxNumericValue": 0.1 }],
          "total-blocking-time": ["error", { "maxNumericValue": 200 }],
          "color-contrast": ["error"],
          "document-title": ["error"],
          "meta-description": ["error"],
          "viewport": ["error"]
        }
      }
    }
  }
  ```
  - **CA:** Lighthouse CI configurado con thresholds estrictos.

### Criterios de Aceptación — HU-CS06.2

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Pipeline ejecuta en MR | Merge request → pipeline corre |
| 2 | TypeScript validation pasa | `tsc --noEmit` sin errores |
| 3 | Build genera dist/ | Artifact dist/ generado |
| 4 | Lighthouse CI gating | Score < 90 falla el pipeline |
| 5 | Preview deploy automático | MR → preview URL generada |
| 6 | Production deploy desde main | Push a main → deploy a sereni.dad |
| 7 | Variables protegidas | CF_API_TOKEN no visible en logs |

### Definition of Done — HU-CS06.2

- [ ] `.gitlab-ci.yml` creado con 4 stages
- [ ] Lighthouse CI configurado con thresholds
- [ ] Variables CI/CD configuradas en GitLab
- [ ] Cloudflare API token creado con permisos mínimos
- [ ] Preview deployments automáticos
- [ ] Production deploy automatizado desde main
- [ ] Pipeline pasa end-to-end

---

## HU-CS06.3 — Monitoreo y Verificación Post-Deploy

**Como** administrador del sistema,
**quiero** verificación automática post-deploy y monitoreo básico del uptime,
**para que** pueda detectar problemas rápidamente y asegurar que la landing page siempre esté disponible.

### Tareas y Subtareas

#### T-CS06.3.1 — Health check endpoint

- **ST-CS06.3.1.1** — Crear `functions/api/health.ts`.
  ```typescript
  /**
   * GET /api/health — Health check endpoint
   * Returns system status for monitoring
   */
  interface Env {
    DB: D1Database;
  }

  export const onRequestGet: PagesFunction<Env> = async (context) => {
    const start = Date.now();

    // Check D1 connectivity
    let dbStatus = 'ok';
    try {
      await context.env.DB.prepare('SELECT 1').first();
    } catch {
      dbStatus = 'error';
    }

    const responseTime = Date.now() - start;

    return new Response(
      JSON.stringify({
        status: dbStatus === 'ok' ? 'healthy' : 'degraded',
        service: 'serenidad-comingsoon',
        version: '1.0.0',
        timestamp: new Date().toISOString(),
        response_ms: responseTime,
        checks: {
          database: dbStatus,
        },
      }),
      {
        status: dbStatus === 'ok' ? 200 : 503,
        headers: {
          'Content-Type': 'application/json',
          'Cache-Control': 'no-cache',
        },
      },
    );
  };
  ```
  - **CA:** GET /api/health retorna status JSON.

#### T-CS06.3.2 — Post-deploy verification script

- **ST-CS06.3.2.1** — Crear `scripts/verify-deploy.sh`.
  ```bash
  #!/bin/bash
  # Post-deploy verification script
  # Usage: ./scripts/verify-deploy.sh [URL]

  set -euo pipefail

  URL="${1:-https://sereni.dad}"
  ERRORS=0

  echo "🔍 Verifying deployment: ${URL}"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

  # 1. Landing page accessible
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" "${URL}")
  if [ "${STATUS}" = "200" ]; then
    echo "✅ Landing page: ${STATUS}"
  else
    echo "❌ Landing page: ${STATUS} (expected 200)"
    ERRORS=$((ERRORS + 1))
  fi

  # 2. Health check
  HEALTH=$(curl -s "${URL}/api/health")
  HEALTH_STATUS=$(echo "${HEALTH}" | grep -o '"status":"[^"]*"' | cut -d'"' -f4)
  if [ "${HEALTH_STATUS}" = "healthy" ]; then
    echo "✅ Health check: ${HEALTH_STATUS}"
  else
    echo "❌ Health check: ${HEALTH_STATUS} (expected healthy)"
    ERRORS=$((ERRORS + 1))
  fi

  # 3. API endpoint responds
  API_STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
    -X POST "${URL}/api/lead" \
    -H "Content-Type: application/json" \
    -d '{"name":"Verify Test","email":"verify@test.com","phone":"+5215500000001"}')
  if [ "${API_STATUS}" = "201" ] || [ "${API_STATUS}" = "200" ]; then
    echo "✅ API endpoint: ${API_STATUS}"
  else
    echo "❌ API endpoint: ${API_STATUS} (expected 201 or 200)"
    ERRORS=$((ERRORS + 1))
  fi

  # 4. Privacy page accessible
  PRIV_STATUS=$(curl -s -o /dev/null -w "%{http_code}" "${URL}/privacidad")
  if [ "${PRIV_STATUS}" = "200" ]; then
    echo "✅ Privacy page: ${PRIV_STATUS}"
  else
    echo "❌ Privacy page: ${PRIV_STATUS} (expected 200)"
    ERRORS=$((ERRORS + 1))
  fi

  # 5. HTTPS redirect
  HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" -L "http://${URL#https://}")
  if [ "${HTTP_STATUS}" = "200" ]; then
    echo "✅ HTTP→HTTPS redirect works"
  else
    echo "❌ HTTP→HTTPS redirect: ${HTTP_STATUS}"
    ERRORS=$((ERRORS + 1))
  fi

  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  if [ "${ERRORS}" = "0" ]; then
    echo "🎉 All checks passed!"
    exit 0
  else
    echo "⚠️  ${ERRORS} check(s) failed"
    exit 1
  fi
  ```
  - **CA:** Script de verificación post-deploy creado.

#### T-CS06.3.3 — Uptime monitoring con Cloudflare

- **ST-CS06.3.3.1** — Configurar Cloudflare Health Checks.
  ```bash
  # Via Cloudflare dashboard:
  # Traffic → Health Checks → Create
  # - Name: serenidad-comingsoon
  # - URL: https://sereni.dad/api/health
  # - Interval: 60 seconds
  # - Regions: All
  # - Alert email: admin@sereni.dad
  ```
  - **CA:** Health check configurado con alertas por email.

#### T-CS06.3.4 — Export automático de leads (cron)

- **ST-CS06.3.4.1** — Crear `scripts/cron-export-leads.sh`.
  ```bash
  #!/bin/bash
  # Weekly lead export — add to crontab or GitLab scheduled pipeline
  # Crontab: 0 2 * * 0 /path/to/scripts/cron-export-leads.sh

  set -euo pipefail

  cd "$(dirname "$0")/.."

  TIMESTAMP=$(date +%Y%m%d_%H%M%S)
  EXPORT_DIR="exports"
  mkdir -p "${EXPORT_DIR}"

  echo "[$(date)] Starting weekly lead export..."

  npx wrangler d1 execute serenidad-leads --remote --command="
    SELECT id, name, email, phone, message, source, created_at
    FROM leads
    ORDER BY created_at DESC;
  " --json > "${EXPORT_DIR}/leads_export_${TIMESTAMP}.json"

  TOTAL=$(npx wrangler d1 execute serenidad-leads --remote --command="SELECT COUNT(*) as total FROM leads;" 2>/dev/null | grep -o '"total":[0-9]*' | cut -d':' -f2)

  echo "[$(date)] Export complete. Total leads: ${TOTAL}"
  echo "[$(date)] File: ${EXPORT_DIR}/leads_export_${TIMESTAMP}.json"
  ```
  - **CA:** Script de export semanal creado.

### Criterios de Aceptación — HU-CS06.3

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Health endpoint funciona | GET /api/health → 200 JSON |
| 2 | Post-deploy script funciona | `./scripts/verify-deploy.sh` → all checks passed |
| 3 | Cloudflare Health Check activo | Dashboard muestra health check verde |
| 4 | Alertas configuradas | Email de alerta configurado |
| 5 | Export script funciona | `./scripts/cron-export-leads.sh` genera JSON |

### Definition of Done — HU-CS06.3

- [ ] Health check endpoint implementado
- [ ] Post-deploy verification script creado
- [ ] Cloudflare Health Checks configurado
- [ ] Alertas por email configuradas
- [ ] Script de export semanal creado

---

## Criterios de Aceptación Global — EP-CS06

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-EP06-1 | Landing page live en sereni.dad | curl → 200 |
| CA-EP06-2 | HTTPS con certificado válido | SSL Labs A rating |
| CA-EP06-3 | API endpoint funciona en producción | POST → 201 |
| CA-EP06-4 | CI/CD pipeline pasa end-to-end | MR → validate → build → test → preview |
| CA-EP06-5 | Lighthouse CI gating funciona | Score < 90 falla pipeline |
| CA-EP06-6 | Production deploy automatizado | Push main → deploy automático |
| CA-EP06-7 | Health check funciona | GET /api/health → healthy |
| CA-EP06-8 | Post-deploy verification pasa | All checks passed |

### Definition of Done — EP-CS06

- [ ] Cloudflare Pages project creado y configurado
- [ ] Custom domain `sereni.dad` con HTTPS
- [ ] GitLab CI/CD pipeline funcional
- [ ] Lighthouse CI gating con thresholds
- [ ] Preview deployments automáticos
- [ ] Production deploy automatizado
- [ ] Health check endpoint
- [ ] Post-deploy verification script
- [ ] Cloudflare Health Checks configurado
- [ ] Export semanal de leads

---

## Resumen de Entregables EP-CS06

**Deployment:**
- Cloudflare Pages project `serenidad-comingsoon`
- Custom domain `sereni.dad` con HTTPS automático
- D1 binding configurado
- Environment variables configuradas

**CI/CD:**
- `.gitlab-ci.yml` con 4 stages (validate, build, test, deploy)
- `lighthouserc.json` con thresholds estrictos
- Preview deployments automáticos
- Production deploy desde main

**Monitoreo:**
- `functions/api/health.ts` — Health check endpoint
- `scripts/verify-deploy.sh` — Post-deploy verification
- `scripts/cron-export-leads.sh` — Export semanal
- Cloudflare Health Checks con alertas

**Verificación final:**
```bash
# 1. Deploy
git push origin main
# → GitLab pipeline → deploy → sereni.dad live

# 2. Verify
./scripts/verify-deploy.sh https://sereni.dad

# 3. Lighthouse
npx lighthouse https://sereni.dad --view
# Targets: Performance ≥ 95, Accessibility ≥ 95, SEO ≥ 95

# 4. Test form
# Open https://sereni.dad in browser
# Click CTA → Fill form → Submit → WhatsApp opens

# 5. Verify data
npx wrangler d1 execute serenidad-leads --remote \
  --command="SELECT COUNT(*) as total FROM leads;"
```
