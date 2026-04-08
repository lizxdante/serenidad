# EP-09 — CI/CD Pipeline

**Épica:** Como equipo de desarrollo, necesitamos pipelines de GitLab CI/CD configurados para build automático del IAM Service, deploy del BFF a CF Workers, y deploy de la SPA a CF Pages, activados por cambios en las rutas relevantes del monorepo, para que cada push a `main` desencadene el ciclo completo de test → build → deploy sin intervención manual.

**Origen:** Sección §23 (V-18) del documento de decisión.
**Prioridad:** Alta — Automatiza el flujo de entrega continua.
**Sprint:** S3
**Dependencias Entrantes:** EP-07 HU-07.2 (IAM Service implementado), EP-08 HU-08.1 (BFF implementado), EP-08 HU-08.2 (SPA implementada), EP-01 HU-01.4 (variables CI/CD en GitLab).
**Dependencias Salientes:** Ninguna directa (cierra el ciclo de entrega).

---

## HU-09.1A — Orquestador Raíz y Pipeline del IAM Service (V-18, Parte A)

**Como** ingeniero de plataforma,
**quiero** tener un orquestador raíz de CI/CD en GitLab y un pipeline específico para el IAM Service que ejecute tests, build de imagen Distroless, push al Container Registry, y actualización GitOps del manifiesto,
**para que** cada push a `main` que modifique `services/iam/` desencadene automáticamente el ciclo test → build → deploy sin intervención manual.

**Dependencias:** EP-01 HU-01.4 (variables CI/CD: `HCLOUD_TOKEN`, `CF_API_TOKEN`, `CF_ACCOUNT_ID`, etc.), EP-07 HU-07.2 (IAM Service code + Dockerfile).
**Archivo detallado:** `09_EP09_cicd_pipeline/HU-09.1A_Root_y_IAM_Pipeline.md`

## HU-09.1B — Pipelines del Edge: BFF y SPA Web (V-18, Parte B)

**Como** ingeniero de plataforma,
**quiero** tener pipelines CI/CD que desplieguen automáticamente el BFF a Cloudflare Workers y la SPA a Cloudflare Pages al hacer push a `main`,
**para que** los componentes de edge se actualicen en producción sin intervención manual, con verificación de tipos TypeScript previa al deploy.

**Dependencias:** EP-08 HU-08.1 (BFF code + wrangler.toml), EP-08 HU-08.2 (SPA code + build config), EP-01 HU-01.4 (variables CF_API_TOKEN, CF_ACCOUNT_ID).
**Archivo detallado:** `09_EP09_cicd_pipeline/HU-09.1B_Edge_Pipelines_Docs.md`

### Tareas y Subtareas

#### T-09.1.1 — Crear orquestador principal `.gitlab-ci.yml`

- **ST-09.1.1.1** — Crear archivo `.gitlab-ci.yml` en la raíz del monorepo.
  - **CA:** Contenido:
    - `include`: 3 archivos locales:
      - `.gitlab/ci/iam-service.yml`
      - `.gitlab/ci/deploy-bff.yml`
      - `.gitlab/ci/deploy-web.yml`
    - `stages`: `test`, `build`, `deploy`.
  - **CA:** Archivo YAML válido.
- **ST-09.1.1.2** — Crear directorio `.gitlab/ci/`.
  - **CA:** Directorio existe.
- **ST-09.1.1.3** — Commitear y pushear.
  - **CA:** GitLab detecta `.gitlab-ci.yml` y habilita pipelines.

#### T-09.1.2 — Crear pipeline del IAM Domain Service (Go)

- **ST-09.1.2.1** — Crear archivo `.gitlab/ci/iam-service.yml`.
  - **CA:** Variables: `GO_VERSION: "1.25"`.

  **Job `test-iam` (stage: test):**
  - **CA:** Image: `golang:${GO_VERSION}-alpine`.
  - **CA:** Rules: activado por cambios en `services/iam/**/*` o `.gitlab/ci/iam-service.yml`.
  - **CA:** También activado por merge request events con cambios en `services/iam/**/*`.
  - **CA:** Script:
    - `cd services/iam && go mod download`.
    - `go test -v -race -coverprofile=coverage.out ./...`.
    - Verificar cobertura ≥ 60%, falla si es inferior.
    - `go build ./...`.
    - `go vet ./...`.
  - **CA:** `coverage` regex configurado: `'/total:\s+\(statements\)\s+(\d+\.\d+)%/'`.
  - **CA:** Cache: `go-mod-${CI_COMMIT_REF_SLUG}` para `vendor/` y `$GOPATH/pkg/mod/`.

  **Job `build-push-iam` (stage: build):**
  - **CA:** Image: `docker:26` con service `docker:26-dind`.
  - **CA:** Rules: solo en `$CI_COMMIT_BRANCH == "main"` con cambios en `services/iam/**/*`.
  - **CA:** `needs: [test-iam]` (dependencia explícita).
  - **CA:** Script:
    - `docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY`.
    - Build con tags: `${CI_REGISTRY_IMAGE}/iam-service:${CI_COMMIT_SHORT_SHA}` y `:latest`.
    - Labels OCI: `org.opencontainers.image.revision`, `org.opencontainers.image.source`.
    - Push de ambos tags.
    - `sed` para actualizar image tag en `infra/apps/iam-service/deployment.yaml`.
    - Git commit y push del manifiesto actualizado (FluxCD detecta ~1 min).
  - **CA:** Variables: `DOCKER_TLS_CERTDIR: "/certs"`.

- **ST-09.1.2.2** — Commitear y pushear.
  - **CA:** Pipeline visible en GitLab CI/CD → Pipelines.

#### T-09.1.3 — Crear pipeline de deploy del BFF a CF Workers

- **ST-09.1.3.1** — Crear archivo `.gitlab/ci/deploy-bff.yml`.

  **Job `deploy-bff` (stage: deploy):**
  - **CA:** Image: `node:22-alpine`.
  - **CA:** Rules: solo en `$CI_COMMIT_BRANCH == "main"` con cambios en `apps/bff/**/*` o `.gitlab/ci/deploy-bff.yml`.
  - **CA:** before_script:
    - `npm install -g bun wrangler --silent`.
    - `cd apps/bff && bun install --frozen-lockfile`.
  - **CA:** Script:
    - `bun run tsc --noEmit` (type check).
    - `wrangler deploy`.
  - **CA:** Variables: `CLOUDFLARE_API_TOKEN: $CF_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID: $CF_ACCOUNT_ID`.

- **ST-09.1.3.2** — Commitear y pushear.
  - **CA:** Pipeline visible.

#### T-09.1.4 — Crear pipeline de deploy de la SPA a CF Pages

- **ST-09.1.4.1** — Crear archivo `.gitlab/ci/deploy-web.yml`.

  **Job `deploy-web` (stage: deploy):**
  - **CA:** Image: `node:22-alpine`.
  - **CA:** Rules: solo en `$CI_COMMIT_BRANCH == "main"` con cambios en `apps/web/**/*` o `.gitlab/ci/deploy-web.yml`.
  - **CA:** before_script:
    - `npm install -g bun wrangler --silent`.
    - `cd apps/web && bun install --frozen-lockfile`.
  - **CA:** Script:
    - `bun run build`.
    - `wrangler pages deploy dist/ --project-name=serenidad-web --branch=main`.
  - **CA:** Variables:
    - `VITE_API_BASE_URL: https://api.sereni.dad/bff`.
    - `VITE_KRATOS_URL: https://api.sereni.dad/auth`.
    - `CLOUDFLARE_API_TOKEN: $CF_API_TOKEN`.
    - `CLOUDFLARE_ACCOUNT_ID: $CF_ACCOUNT_ID`.

- **ST-09.1.4.2** — Commitear y pushear.
  - **CA:** Pipeline visible.

#### T-09.1.5 — Verificar pipeline completo del IAM Service

- **ST-09.1.5.1** — Hacer un cambio trivial en `services/iam/` y pushear a `main`.
  - **CA:** Pipeline se activa automáticamente.
- **ST-09.1.5.2** — Verificar stage `test`.
  - **CA:** Job `test-iam` pasa: tests OK, cobertura ≥ 60%, build OK, vet OK.
- **ST-09.1.5.3** — Verificar stage `build`.
  - **CA:** Job `build-push-iam` pasa: imagen pusheada a `registry.gitlab.com`, manifiesto actualizado.
- **ST-09.1.5.4** — Verificar deploy automático via FluxCD.
  - **CA:** `flux get all` → kustomization aplicada con nueva revisión.
  - **CA:** `kubectl rollout status deployment/iam-service -n serenidad-core` → rolled out.
- **ST-09.1.5.5** — Verificar imagen en el registry.
  - **CA:** `docker pull ${CI_REGISTRY_IMAGE}/iam-service:${CI_COMMIT_SHORT_SHA}` → exitoso.
  - **CA:** Imagen tagged con SHA corto y `latest`.

#### T-09.1.6 — Verificar pipeline del BFF

- **ST-09.1.6.1** — Hacer un cambio trivial en `apps/bff/` y pushear a `main`.
  - **CA:** Pipeline se activa.
- **ST-09.1.6.2** — Verificar deploy.
  - **CA:** Job `deploy-bff` pasa: type check OK, wrangler deploy OK.
  - **CA:** `curl https://api.sereni.dad/bff/health` → nueva versión activa.

#### T-09.1.7 — Verificar pipeline de la SPA

- **ST-09.1.7.1** — Hacer un cambio trivial en `apps/web/` y pushear a `main`.
  - **CA:** Pipeline se activa.
- **ST-09.1.7.2** — Verificar deploy.
  - **CA:** Job `deploy-web` pasa: build OK, wrangler pages deploy OK.
  - **CA:** `curl https://app.sereni.dad` → nueva versión activa.

#### T-09.1.8 — Verificar que pipelines NO se activan por cambios irrelevantes

- **ST-09.1.8.1** — Hacer un cambio en `docs/` o `plans/` y pushear.
  - **CA:** Ningún pipeline de `iam-service`, `bff`, o `web` se activa.
  - **CA:** Solo se activa pipeline si hay coincidencia en las `rules.changes`.

#### T-09.1.9 — Documentar variables CI/CD automáticas de GitLab

- **ST-09.1.9.1** — Documentar las variables automáticas de GitLab CI relevantes.
  - **CA:** Documentación incluye:
    - `$CI_REGISTRY` → `registry.gitlab.com`.
    - `$CI_REGISTRY_USER` / `$CI_REGISTRY_PASSWORD` → credenciales auto-generadas por job.
    - `$CI_REGISTRY_IMAGE` → `registry.gitlab.com/<grupo>/<repo>`.
    - `$CI_COMMIT_SHORT_SHA` → SHA corto del commit.
    - `$CI_JOB_TOKEN` → token de corta duración para operaciones Git.
    - `$CI_SERVER_HOST` → `gitlab.com`.
    - `$CI_PROJECT_PATH` → `serenidad/serenidad-platform`.

### Criterios de Aceptación de la HU-09.1A (Orquestador + IAM)

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-09.1A-1 | Orquestador raíz incluye 3 sub-pipelines | `.gitlab-ci.yml` con `include` de los 3 archivos |
| CA-09.1A-2 | Pipeline IAM: push a services/iam → test + build + deploy | Pipeline completo exitoso en GitLab |
| CA-09.1A-3 | Cobertura ≥ 60% enforced | Pipeline falla si cobertura < 60% |
| CA-09.1A-4 | Imagen con tags correctos | SHA corto + latest en GitLab Container Registry |
| CA-09.1A-5 | FluxCD detecta cambio de imagen | Deployment actualizado tras push de manifiesto |
| CA-09.1A-6 | No hay bucle infinito de CI | El commit `[skip ci]` del bot no dispara nuevo pipeline |

### Criterios de Aceptación de la HU-09.1B (Edge: BFF + SPA)

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-09.1B-1 | Pipeline BFF: push a apps/bff → deploy a CF Workers | BFF actualizado en CF Workers |
| CA-09.1B-2 | Pipeline SPA: push a apps/web → deploy a CF Pages | SPA actualizada en CF Pages |
| CA-09.1B-3 | Activación selectiva | Solo se activan pipelines para paths con cambios |
| CA-09.1B-4 | Type check TypeScript previo al deploy del BFF | `bun run tsc --noEmit` pasa antes de `wrangler deploy` |
| CA-09.1B-5 | Health check post-deploy integrado | Jobs verifican HTTP 200 tras despliegue |
| CA-09.1B-6 | Variables CI/CD de Cloudflare masked | `CF_API_TOKEN`, `CF_ACCOUNT_ID` protegidas |

### Definition of Done — HU-09.1A

- [ ] `.gitlab-ci.yml` orquestador creado en la raíz con `include` de los 3 sub-pipelines.
- [ ] `.gitlab/ci/iam-service.yml` con `test-iam` y `build-push-iam` completos.
- [ ] Pipeline IAM Service: test → build → push registry → actualizar manifiesto → FluxCD deploy.
- [ ] Activación selectiva por paths verificada para `services/iam/`.
- [ ] Cobertura mínima de 60% enforced.
- [ ] Variables CI/CD de GitLab documentadas.

### Definition of Done — HU-09.1B

- [ ] `.gitlab/ci/deploy-bff.yml` con verificación de tipos TypeScript previo al deploy.
- [ ] `.gitlab/ci/deploy-web.yml` con build de Qwik v2 y validación de `dist/_worker.js`.
- [ ] Pipeline BFF: type check → wrangler deploy → health check.
- [ ] Pipeline SPA: build → wrangler pages deploy → content-type check.
- [ ] Activación selectiva verificada: cambios en BFF no despliegan la SPA y viceversa.
- [ ] Variables `CF_API_TOKEN` y `CF_ACCOUNT_ID` configuradas como secretos protegidos.

---

## Resumen de Dependencias Internas EP-09

```
HU-09.1A (Orquestador + IAM Pipeline):
  T-09.1A.1 (orquestador .gitlab-ci.yml)
    └──► T-09.1A.2 (pipeline IAM) ──► T-09.1A.5 (commit + activación)
         └──► T-09.1A.6 (verificar despliegue E2E)

HU-09.1B (Pipelines Edge):
  T-09.1B.3 (pipeline BFF) ──► T-09.1B.5 (commit + activación)
  T-09.1B.4 (pipeline SPA)     └──► T-09.1B.6 (verificar activación selectiva)

T-09.1A.6, T-09.1B.6 ──► T-09.1.8 (verificar no-activación cruzada)

Dependencias externas:
  HU-09.1A requiere: EP-07 HU-07.2 (IAM Service code + Dockerfile).
  HU-09.1B requiere: EP-08 HU-08.1 (BFF code + wrangler.toml),
                      EP-08 HU-08.2 (SPA code + build config).
  HU-09.1A.6 requiere: EP-02 (cluster K8s) + EP-03 (FluxCD) para verify deploy.
  Todas las pipelines requieren: EP-01 HU-01.4 (variables CI/CD en GitLab).
```
