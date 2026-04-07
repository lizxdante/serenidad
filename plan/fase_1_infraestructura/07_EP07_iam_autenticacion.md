# EP-07 — IAM: Autenticación y Autorización

**Épica:** Como equipo de desarrollo, necesitamos Ory Kratos v1.3.1 desplegado como motor de autenticación con Passkeys/WebAuthn y el IAM Domain Service en Go con JWT Ed25519, Outbox pattern, y migraciones de base de datos, para que los usuarios puedan registrarse, autenticarse, y obtener tokens JWT que autoricen el acceso a todos los endpoints protegidos del sistema.

**Origen:** Secciones §18 (V-13) y §19 (V-14) del documento de decisión.
**Prioridad:** Crítica — Componente central de identidad para todo el sistema.
**Sprint:** S3
**Dependencias Entrantes:**
- EP-04 HU-04.2 (Traefik para IngressRoutes).
- EP-04 HU-04.3 (DNS para dominio público).
- EP-05 HU-05.1 (PostgreSQL para `kratos_db` y `iam_db`).
- EP-06 HU-06.1 (Protobuf para tipos de eventos en Outbox).
- EP-03 HU-03.3 (SOPS para secrets de JWT keys y Kratos).
**Dependencias Salientes:** EP-08 (BFF proxy al IAM Service, Frontend usa Kratos flows).

---

## HU-07.1 — Ory Kratos v1.3.1 (V-13)

**Como** ingeniero de plataforma,
**quiero** tener Ory Kratos v1.3.1 desplegado en el cluster con Passkeys como método principal de autenticación, identity schemas para patient y doctor, y email de verificación via Resend,
**para que** los usuarios puedan registrarse y autenticarse de forma segura sin contraseñas.

**Dependencias:** HU-04.2 (Traefik), HU-04.3 (DNS), HU-05.1 (PostgreSQL `kratos_db`).

### Tareas y Subtareas

#### T-07.1.1 — Crear HelmRepository de Ory

- **ST-07.1.1.1** — Crear archivo `infra/infrastructure/kratos/helmrepository.yaml` (o reutilizar en `infra/apps/kratos/`).
  - **CA:** URL: `https://k8s.ory.sh/helm/charts`.
  - **CA:** Namespace: `flux-system`, intervalo: `1h`.
- **ST-07.1.1.2** — Commitear y pushear.
  - **CA:** `flux get source helm ory` → `True`.

#### T-07.1.2 — Crear Identity Schemas (patient y doctor)

- **ST-07.1.2.1** — Crear directorio `infra/apps/kratos/schemas/`.
  - **CA:** Directorio existe.
- **ST-07.1.2.2** — Crear archivo `infra/apps/kratos/schemas/patient.json`.
  - **CA:** Schema JSON válido con `$id: https://sereni.dad/schemas/identity/patient.json`.
  - **CA:** Traits:
    - `email` (string, format email, identifier para password/code/passkey, recovery y verification via email).
    - `name` (object: first, last).
    - `birth_date` (string, format date).
    - `phone` (string).
    - `preferred_language` (enum: es, en, pt, default es).
  - **CA:** Required: `["email"]`.
  - **CA:** `additionalProperties: false`.
- **ST-07.1.2.3** — Crear archivo `infra/apps/kratos/schemas/doctor.json`.
  - **CA:** Schema JSON válido con `$id: https://sereni.dad/schemas/identity/doctor.json`.
  - **CA:** Traits:
    - `email` (string, format email, identifier para code/passkey, recovery y verification via email).
    - `name` (object: first, last — required).
    - `medical_license` (string).
    - `specialty` (enum: psychiatry, psychology, general_medicine).
    - `country_code` (string, pattern `^[A-Z]{2}$`).
  - **CA:** Required: `["email", "name", "medical_license", "specialty", "country_code"]`.
  - **CA:** `additionalProperties: false`.
- **ST-07.1.2.4** — Validar schemas JSON con un validador JSON Schema.
  - **CA:** Ambos schemas son JSON Schema draft-07 válidos.

#### T-07.1.3 — Crear secrets de Kratos cifrados con SOPS

- **ST-07.1.3.1** — Generar `KRATOS_COOKIE_SECRET` con `openssl rand -hex 32`.
  - **CA:** Secret de 64 caracteres hex generado.
- **ST-07.1.3.2** — Generar `KRATOS_CIPHER_SECRET` con `openssl rand -hex 32`.
  - **CA:** Secret de 64 caracteres hex generado.
- **ST-07.1.3.3** — Cifrar secrets con el script `encrypt-secret.sh`.
  - **CA:** `./scripts/encrypt-secret.sh serenidad-core kratos-secrets KRATOS_COOKIE_SECRET=... KRATOS_CIPHER_SECRET=... RESEND_API_KEY=...`.
  - **CA:** Archivo `infra/secrets/kratos-secrets.yaml` generado y cifrado.
- **ST-07.1.3.4** — Backup de los secrets en password manager.
  - **CA:** Cookie secret, cipher secret, y Resend API key respaldados.
- **ST-07.1.3.5** — Commitear y pushear.
  - **CA:** `git log --oneline -1` → `feat: add kratos secrets encrypted with SOPS`.
  - **CA:** FluxCD descifra y aplica: `kubectl get secret kratos-secrets -n serenidad-core` → existe.

#### T-07.1.4 — Crear HelmRelease de Kratos

- **ST-07.1.4.1** — Crear archivo `infra/apps/kratos/helmrelease.yaml`.
  - **CA:** `dependsOn`: cloudnative-pg (namespace cnpg-system), traefik (namespace traefik).
  - **CA:** Chart: `kratos`, version: `0.50.x`.
  - **CA:** Configuración de Kratos:
    - `dsn`: conexión a `kratos_db` via `serenidad-pg-rw.serenidad-data.svc.cluster.local:5432/kratos_db?sslmode=require`.
    - `serve.public.base_url`: `https://api.sereni.dad/auth/`.
    - `serve.public.cors`: enabled, allowed_origins `https://app.sereni.dad`.
    - `serve.admin.base_url`: `http://kratos-admin.serenidad-core.svc.cluster.local:4434/`.
    - `selfservice.default_browser_return_url`: `https://app.sereni.dad/`.
    - Flows: login (ui_url `/login`, 1h), registration (`/register`, 30m), verification (enabled, code via email), recovery (enabled, code), settings (`/settings`, 15m privileged), logout.
    - Methods: `passkey` enabled (rp display_name "serenidad — Clínica Digital", id "sereni.dad", origins `https://app.sereni.dad`). `webauthn` disabled. `password` disabled. `code` enabled.
    - Identity: default_schema_id `patient`, schemas patient y doctor.
    - Courier SMTP: `smtps://resend:${RESEND_API_KEY}@smtp.resend.com:465/`, from `noreply@sereni.dad`.
    - Secrets: cookie y cipher desde secret cifrado.
  - **CA:** Resources: requests `memory: "128Mi", cpu: "50m"`, limits `memory: "256Mi", cpu: "500m"`.
- **ST-07.1.4.2** — Commitear y pushear.
  - **CA:** FluxCD reconcilia el HelmRelease de Kratos.

#### T-07.1.5 — Crear IngressRoute para Kratos

- **ST-07.1.5.1** — Crear archivo `infra/apps/kratos/ingressroute.yaml`.
  - **CA:** IngressRoute para `websecure` entrypoint.
  - **CA:** Match: `Host('api.sereni.dad') && PathPrefix('/auth/')`.
  - **CA:** Service: `kratos-public` port `4433`.
  - **CA:** Middleware: `security-headers` (namespace traefik).
  - **CA:** TLS: certResolver `letsencrypt`, domain `api.sereni.dad`.
- **ST-07.1.5.2** — Commitear y pushear.
  - **CA:** `kubectl get ingressroute kratos-public -n serenidad-core` → existe.

#### T-07.1.6 — Crear kustomization.yaml del componente Kratos

- **ST-07.1.6.1** — Crear `infra/apps/kratos/kustomization.yaml`.
  - **CA:** Resources: `helmrepository.yaml` (si local), `helmrelease.yaml`, `ingressroute.yaml`.
  - **CA:** Referencia a secret: `../../secrets/kratos-secrets.yaml`.
- **ST-07.1.6.2** — Commitear y verificar reconciliación.
  - **CA:** FluxCD reconcilia sin error.

#### T-07.1.7 — Verificar despliegue de Kratos

- **ST-07.1.7.1** — Verificar pod de Kratos.
  - **CA:** `kubectl get pods -n serenidad-core | grep kratos` → pod Running.
- **ST-07.1.7.2** — Verificar health de Kratos internamente.
  - **CA:** `kubectl exec -n serenidad-core deployment/kratos -- wget -qO- http://localhost:4434/health/ready` → `{"status":"ok"}`.
- **ST-07.1.7.3** — Verificar Kratos desde el exterior.
  - **CA:** `curl https://api.sereni.dad/auth/.well-known/ory/webauthn.js` → HTTP 200, retorna JavaScript de WebAuthn.
- **ST-07.1.7.4** — Verificar que la base de datos fue migrada.
  - **CA:** `kubectl exec -it serenidad-pg-1 -n serenidad-data -- psql -U serenidad_admin -d kratos_db -c "\dt"` → tablas de Kratos (identity, session, etc.).
- **ST-07.1.7.5** — Verificar que Passkeys está habilitado.
  - **CA:** `curl https://api.sereni.dad/auth/self-service/registration/browser` → flow con método passkey disponible.
- **ST-07.1.7.6** — Verificar configuración de email.
  - **CA:** Crear un flow de registro de prueba y verificar que el email de verificación se envía via Resend.

### Criterios de Aceptación de la HU-07.1

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-1 | Kratos v1.3.1 Running | Pod Running en serenidad-core |
| CA-2 | Health ready | `/health/ready` → `{"status":"ok"}` |
| CA-3 | WebAuthn JS accesible | `curl https://api.sereni.dad/auth/.well-known/ory/webauthn.js` → 200 |
| CA-4 | DB migrada | Tablas de Kratos en `kratos_db` |
| CA-5 | Passkeys habilitado | Flow de registro incluye método passkey |
| CA-6 | Password deshabilitado | Flow de registro NO incluye método password |
| CA-7 | Email via Resend funcional | Email de verificación enviado correctamente |
| CA-8 | 2 identity schemas | patient y doctor schemas cargados |
| CA-9 | CORS configurado | Requests desde `app.sereni.dad` permitidas |
| CA-10 | Secrets cifrados en Git | `infra/secrets/kratos-secrets.yaml` cifrado con SOPS |

### Definition of Done — HU-07.1

- [ ] Kratos v1.3.1 desplegado via FluxCD HelmRelease.
- [ ] Identity schemas patient y doctor validados y cargados.
- [ ] Passkeys como método principal de autenticación.
- [ ] Passwords deshabilitadas (cero contraseñas).
- [ ] IngressRoute funcional en `api.sereni.dad/auth/*`.
- [ ] Email de verificación via Resend operativo.
- [ ] Base de datos `kratos_db` migrada.
- [ ] Secrets cifrados con SOPS.

---

## HU-07.2 — IAM Domain Service Go (V-14)

**Como** desarrollador backend,
**quiero** tener el IAM Domain Service en Go desplegado con JWT Ed25519, ForwardAuth endpoint, migraciones de base de datos (user_profiles + outbox_events), y Outbox pattern,
**para que** el sistema pueda intercambiar sesiones de Kratos por JWT, validar tokens en cada request, y emitir eventos de dominio con garantía at-least-once.

**Dependencias:** HU-07.1 (Kratos desplegado), HU-05.1 (PostgreSQL `iam_db`), HU-06.1 (Protobuf schemas), HU-04.2 (Traefik para IngressRoute).

### Tareas y Subtareas

#### T-07.2.1 — Crear estructura del proyecto Go del IAM Service

- **ST-07.2.1.1** — Crear estructura de directorios en `services/iam/`.
  - **CA:** Directorios creados:
    - `cmd/server/` (main.go).
    - `internal/config/` (config.go).
    - `internal/domain/` (user.go, events.go).
    - `internal/handlers/` (token.go, validate.go, health.go, metrics.go).
    - `internal/repository/` (user_repo.go, outbox_repo.go).
    - `internal/services/` (jwt_service.go, kratos_client.go, outbox_worker.go).
    - `internal/db/migrations/`.
- **ST-07.2.1.2** — Inicializar módulo Go.
  - **CA:** `go mod init gitlab.com/serenidad/serenidad-platform/services/iam`.
  - **CA:** `go.mod` creado con go directive `go 1.25`.
- **ST-07.2.1.3** — Crear `Makefile` con targets: `build`, `test`, `lint`, `migrate`, `docker-build`.
  - **CA:** `make build` compila el binario.
  - **CA:** `make test` ejecuta tests.

#### T-07.2.2 — Crear migraciones SQL de la base de datos IAM

- **ST-07.2.2.1** — Crear `000001_create_users.up.sql`.
  - **CA:** Crea tabla `user_profiles` con columnas:
    - `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`.
    - `kratos_id UUID NOT NULL UNIQUE`.
    - `tenant_id UUID NOT NULL`.
    - `role TEXT NOT NULL CHECK (role IN ('patient', 'doctor', 'admin'))`.
    - `email_hash TEXT NOT NULL` (SHA-256).
    - `did TEXT NOT NULL UNIQUE` (DID web).
    - `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`.
    - `updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`.
  - **CA:** Índices: `idx_user_profiles_kratos_id`, `idx_user_profiles_tenant_id`, `idx_user_profiles_role_tenant`.
  - **CA:** Row Level Security habilitado: `ALTER TABLE user_profiles ENABLE ROW LEVEL SECURITY`.
  - **CA:** Política `user_own_profile` para SELECT (propio perfil + admins del tenant).
- **ST-07.2.2.2** — Crear `000001_create_users.down.sql`.
  - **CA:** `DROP TABLE IF EXISTS user_profiles CASCADE`.
- **ST-07.2.2.3** — Crear `000002_create_outbox.up.sql`.
  - **CA:** Crea tabla `outbox_events` con columnas:
    - `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`.
    - `aggregate_id UUID NOT NULL`.
    - `aggregate_type TEXT NOT NULL`.
    - `event_type TEXT NOT NULL` (UserRegistered|DoctorOnboarded).
    - `payload BYTEA NOT NULL` (Protobuf serializado).
    - `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`.
    - `published_at TIMESTAMPTZ` (NULL = pendiente).
    - `retry_count INT NOT NULL DEFAULT 0`.
    - `last_error TEXT`.
  - **CA:** Índice parcial: `idx_outbox_pending ON outbox_events(created_at) WHERE published_at IS NULL`.
  - **CA:** Sin RLS en outbox (tabla interna).
- **ST-07.2.2.4** — Crear `000002_create_outbox.down.sql`.
  - **CA:** `DROP TABLE IF EXISTS outbox_events CASCADE`.

#### T-07.2.3 — Implementar el IAM Domain Service

- **ST-07.2.3.1** — Implementar `cmd/server/main.go` (punto de entrada con comandos `serve` y `migrate`).
  - **CA:** Binario acepta subcomandos: `./iam-service serve` y `./iam-service migrate`.
- **ST-07.2.3.2** — Implementar `internal/config/config.go` (lectura de env vars con validación).
  - **CA:** Variables requeridas: `PORT`, `DATABASE_URL`, `KRATOS_ADMIN_URL`, `JWT_PRIVATE_KEY_ED25519`, `JWT_PUBLIC_KEY_ED25519`, `JWT_ISSUER`, `JWT_AUDIENCE`, `JWT_TTL_SECONDS`, `NATS_URL`, `ENVIRONMENT`.
  - **CA:** Validación estricta al iniciar: falla rápido si falta una variable requerida.
- **ST-07.2.3.3** — Implementar `internal/services/jwt_service.go` (firma y verificación JWT Ed25519).
  - **CA:** Firma JWT con `alg: EdDSA`.
  - **CA:** Claims: `sub` (user_id), `role`, `tenant_id`, `did`, `iat`, `exp`, `iss`, `aud`.
  - **CA:** Verificación acepta tokens firmados con la clave privada correspondiente.
  - **CA:** TTL configurable via env var `JWT_TTL_SECONDS`.
- **ST-07.2.3.4** — Implementar `internal/services/kratos_client.go` (cliente HTTP Kratos Admin API :4434).
  - **CA:** Método `ValidateSession(sessionToken string)` que llama a Kratos Admin API.
  - **CA:** Retorna identidad del usuario (traits, schema_id) o error.
- **ST-07.2.3.5** — Implementar `internal/handlers/token.go` (POST `/api/iam/token/exchange`).
  - **CA:** Recibe session token de Kratos.
  - **CA:** Valida sesión con Kratos Admin API.
  - **CA:** Crea perfil de usuario en `iam_db` si no existe (upsert por `kratos_id`).
  - **CA:** Inserta evento `UserRegistered` o `DoctorOnboarded` en `outbox_events` en la MISMA transacción.
  - **CA:** Firma y retorna JWT Ed25519 con claims completos.
- **ST-07.2.3.6** — Implementar `internal/handlers/validate.go` (GET `/internal/validate-token`).
  - **CA:** Endpoint usado por Traefik ForwardAuth.
  - **CA:** Verifica JWT del header `Authorization: Bearer <token>`.
  - **CA:** Si válido: retorna 200 con headers `X-User-ID`, `X-User-Role`, `X-Tenant-ID`, `X-User-DID`.
  - **CA:** Si inválido/ausente: retorna 401.
- **ST-07.2.3.7** — Implementar `internal/handlers/health.go` (GET `/health` y `/ready`).
  - **CA:** `/health` retorna `{"status":"ok","service":"iam-service","version":"0.1.0"}`.
  - **CA:** `/ready` verifica conectividad a DB y Kratos, retorna `{"status":"ok","db":"connected","kratos":"reachable"}` o 503.
- **ST-07.2.3.8** — Implementar `internal/repository/user_repo.go` (CRUD de user_profiles).
  - **CA:** Método `UpsertByKratosID` con INSERT ON CONFLICT (kratos_id) DO UPDATE.
  - **CA:** Método `GetByID`, `GetByKratosID`.
- **ST-07.2.3.9** — Implementar `internal/repository/outbox_repo.go` (Outbox pattern).
  - **CA:** Método `InsertEvent(tx, aggregateID, aggregateType, eventType, payload)`.
  - **CA:** Usa la misma transacción que el upsert del perfil (garantía atómica).
- **ST-07.2.3.10** — Implementar `internal/services/outbox_worker.go` (Worker que lee outbox).
  - **CA:** Goroutine que poll la tabla `outbox_events WHERE published_at IS NULL`.
  - **CA:** En Fase 1 (sin NATS): espera con backoff exponencial hasta que NATS esté disponible.
  - **CA:** En Fase 2: publica en NATS JetStream y marca `published_at`.

#### T-07.2.4 — Escribir tests unitarios e integración

- **ST-07.2.4.1** — Tests para `jwt_service.go`: firma y verificación, expiración, claims.
  - **CA:** Cobertura ≥ 80% del módulo jwt_service.
- **ST-07.2.4.2** — Tests para `validate.go`: 401 sin token, 401 token inválido, 200 token válido con headers.
  - **CA:** Los 3 escenarios probados.
- **ST-07.2.4.3** — Tests para `token.go`: exchange exitoso, sesión inválida.
  - **CA:** Mock de Kratos client. Verificación de JWT retornado.
- **ST-07.2.4.4** — Tests para `outbox_repo.go`: inserción atómica con perfil.
  - **CA:** Test de integración con DB in-memory o real (golang-migrate).
- **ST-07.2.4.5** — Ejecutar suite completa con cobertura.
  - **CA:** `go test -race -coverprofile=coverage.out ./...` → PASS.
  - **CA:** Cobertura global ≥ 60%.

#### T-07.2.5 — Crear Dockerfile del IAM Service

- **ST-07.2.5.1** — Crear `services/iam/Dockerfile` con build en dos etapas.
  - **CA:** Etapa builder: `golang:1.25-alpine`, `CGO_ENABLED=0`, `-trimpath`, `-ldflags="-w -s"`.
  - **CA:** Etapa runtime: `gcr.io/distroless/static-debian12:nonroot`.
  - **CA:** Copia binario y migraciones SQL.
  - **CA:** `EXPOSE 8080`, `USER nonroot:nonroot`.
- **ST-07.2.5.2** — Build local de la imagen Docker.
  - **CA:** `docker build -t iam-service:dev -f services/iam/Dockerfile services/iam/` → build exitoso.
- **ST-07.2.5.3** — Push de imagen a GitLab Container Registry.
  - **CA:** `docker push registry.gitlab.com/serenidad/serenidad-platform/iam-service:latest` → push exitoso.

#### T-07.2.6 — Crear secrets del IAM Service cifrados con SOPS

- **ST-07.2.6.1** — Generar par de claves Ed25519 para JWT.
  - **CA:** `openssl genpkey -algorithm ed25519 -out /tmp/jwt_private_key.pem`.
  - **CA:** `openssl pkey -in /tmp/jwt_private_key.pem -pubout -out /tmp/jwt_public_key.pem`.
- **ST-07.2.6.2** — Cifrar claves con SOPS+age.
  - **CA:** `./scripts/encrypt-secret.sh serenidad-core iam-service-secrets JWT_PRIVATE_KEY_ED25519=... JWT_PUBLIC_KEY_ED25519=...`.
  - **CA:** Archivo `infra/secrets/iam-service-secrets.yaml` cifrado.
- **ST-07.2.6.3** — Backup de claves JWT en password manager.
  - **CA:** Private key y public key (base64) respaldados.
- **ST-07.2.6.4** — Limpiar claves del filesystem local.
  - **CA:** `rm -f /tmp/jwt_private_key.pem /tmp/jwt_public_key.pem`.
- **ST-07.2.6.5** — Commitear y pushear.
  - **CA:** `git log --oneline -1` → `feat: add IAM service JWT keys encrypted with SOPS`.

#### T-07.2.7 — Crear imagePullSecret para GitLab Container Registry

- **ST-07.2.7.1** — Generar Secret docker-registry cifrado con SOPS.
  - **CA:** Tipo `kubernetes.io/dockerconfigjson` para `registry.gitlab.com`.
  - **CA:** Usa Deploy Token de GitLab (read_registry).
  - **CA:** Archivo `infra/secrets/gitlab-registry-secret.yaml` cifrado.
- **ST-07.2.7.2** — Commitear y pushear.
  - **CA:** FluxCD aplica: `kubectl get secret gitlab-registry-secret -n serenidad-core` → existe.

#### T-07.2.8 — Crear manifests de Deployment del IAM Service

- **ST-07.2.8.1** — Crear `infra/apps/iam-service/deployment.yaml`.
  - **CA:** Init container `migrate` con mismo imagen, command `["/app/iam-service", "migrate"]`.
  - **CA:** Container principal `iam-service` con:
    - Image: `registry.gitlab.com/serenidad/serenidad-platform/iam-service:latest`.
    - Puerto 8080.
    - Env vars: PORT, DATABASE_URL (secretKeyRef), KRATOS_ADMIN_URL, JWT_PRIVATE_KEY_ED25519 (secretKeyRef), JWT_PUBLIC_KEY_ED25519 (secretKeyRef), JWT_ISSUER, JWT_AUDIENCE, JWT_TTL_SECONDS, NATS_URL, ENVIRONMENT.
    - LivenessProbe: `/health`, initialDelay 10s, period 30s.
    - ReadinessProbe: `/ready`, initialDelay 5s, period 10s.
    - Resources: requests `memory: "40Mi", cpu: "20m"`, limits `memory: "128Mi", cpu: "500m"`.
    - SecurityContext: `runAsNonRoot: true`, `runAsUser: 65534`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`, drop ALL capabilities.
  - **CA:** `imagePullSecrets`: `gitlab-registry-secret`.
  - **CA:** Annotations de FluxCD para image auto-update.
  - **CA:** RollingUpdate: maxUnavailable 0, maxSurge 1.
- **ST-07.2.8.2** — Crear Service en el mismo archivo.
  - **CA:** Service `iam-service` tipo ClusterIP, port 8080.
- **ST-07.2.8.3** — Commitear y pushear.
  - **CA:** FluxCD reconcilia.

#### T-07.2.9 — Crear IngressRoute del IAM Service

- **ST-07.2.9.1** — Crear `infra/apps/iam-service/ingressroute.yaml`.
  - **CA:** IngressRoute público:
    - Match: `Host('api.sereni.dad') && PathPrefix('/api/iam/')`.
    - Service: `iam-service` port 8080.
    - Middleware: `security-headers` (namespace traefik).
    - TLS: certResolver `letsencrypt`.
  - **CA:** Endpoint `/internal/validate-token` NO tiene IngressRoute público (acceso solo interno via cluster DNS).
- **ST-07.2.9.2** — Commitear y pushear.
  - **CA:** `kubectl get ingressroute iam-service-public -n serenidad-core` → existe.

#### T-07.2.10 — Crear kustomization.yaml del componente IAM Service

- **ST-07.2.10.1** — Crear `infra/apps/iam-service/kustomization.yaml`.
  - **CA:** Resources: `deployment.yaml`, `ingressroute.yaml`.
  - **CA:** Referencias a secrets: `../../secrets/iam-service-secrets.yaml`, `../../secrets/gitlab-registry-secret.yaml`.
- **ST-07.2.10.2** — Commitear y verificar.
  - **CA:** FluxCD reconcilia sin error.

#### T-07.2.11 — Verificar despliegue del IAM Service

- **ST-07.2.11.1** — Verificar que la migración init container completó.
  - **CA:** `kubectl describe pod -n serenidad-core -l app=iam-service` → init container `migrate` completed.
  - **CA:** Tablas `user_profiles` y `outbox_events` existen en `iam_db`.
- **ST-07.2.11.2** — Verificar pod Running.
  - **CA:** `kubectl get pods -n serenidad-core | grep iam-service` → Running.
- **ST-07.2.11.3** — Verificar health endpoint.
  - **CA:** `kubectl exec -it deployment/iam-service -n serenidad-core -- wget -qO- http://localhost:8080/health` → `{"status":"ok"}`.
- **ST-07.2.11.4** — Verificar readiness.
  - **CA:** `/ready` → `{"status":"ok","db":"connected","kratos":"reachable"}`.
- **ST-07.2.11.5** — Verificar validate-token retorna 401 sin JWT.
  - **CA:** `curl https://api.sereni.dad/api/iam/whoami` → 401 Unauthorized.
- **ST-07.2.11.6** — Verificar RLS activo en tabla user_profiles.
  - **CA:** `psql -d iam_db -c "SELECT tablename, rowsecurity FROM pg_tables WHERE rowsecurity=true"` → `user_profiles`.
- **ST-07.2.11.7** — Verificar índice parcial de outbox.
  - **CA:** `psql -d iam_db -c "\di idx_outbox_pending"` → índice presente.

### Criterios de Aceptación de la HU-07.2

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-1 | IAM Service Running | Pod Running en serenidad-core |
| CA-2 | Health OK | `/health` → `{"status":"ok"}` |
| CA-3 | Readiness OK | `/ready` → DB connected, Kratos reachable |
| CA-4 | Validate-token 401 sin JWT | Request sin Authorization → 401 |
| CA-5 | Token exchange funcional | POST con session token → JWT Ed25519 válido |
| CA-6 | JWT contiene claims completos | `sub`, `role`, `tenant_id`, `did`, `iat`, `exp`, `iss`, `aud` |
| CA-7 | JWT alg EdDSA | Header del JWT: `alg: "EdDSA"` |
| CA-8 | Migraciones aplicadas | Tablas `user_profiles` y `outbox_events` en `iam_db` |
| CA-9 | RLS activo | `user_profiles` tiene Row Level Security |
| CA-10 | Outbox atómico | Perfil + evento insertados en la misma TX |
| CA-11 | Security context | Pod ejecuta como non-root (UID 65534), read-only filesystem |
| CA-12 | Tests ≥ 60% cobertura | `go test -cover` → ≥ 60% |

### Definition of Done — HU-07.2

- [ ] IAM Domain Service implementado en Go con todos los handlers.
- [ ] Migraciones SQL aplicadas (user_profiles + outbox_events).
- [ ] JWT Ed25519 firma y verificación funcional.
- [ ] ForwardAuth endpoint `/internal/validate-token` operativo.
- [ ] Token exchange endpoint funcional con Kratos.
- [ ] Outbox pattern implementado con inserción atómica.
- [ ] Tests unitarios e integración con cobertura ≥ 60%.
- [ ] Dockerfile multi-stage con distroless runtime.
- [ ] Imagen pusheada a GitLab Container Registry.
- [ ] Deployment con security context restrictivo.
- [ ] Secrets JWT y imagePullSecret cifrados con SOPS.
- [ ] IngressRoute funcional.

---

## Resumen de Dependencias Internas EP-07

```
HU-07.1 (Kratos)
  Requiere: EP-04 HU-04.2 (Traefik), EP-04 HU-04.3 (DNS), EP-05 HU-05.1 (PG kratos_db)

HU-07.2 (IAM Service)
  Requiere: HU-07.1 (Kratos Admin API), EP-05 HU-05.1 (PG iam_db), EP-06 HU-06.1 (Protobuf), EP-04 HU-04.2 (Traefik IngressRoute)

Dependencias internas:
  T-07.1.1 → T-07.1.2 → T-07.1.3 → T-07.1.4 → T-07.1.5 → T-07.1.6 → T-07.1.7
  T-07.2.1 → T-07.2.2 → T-07.2.3 → T-07.2.4 → T-07.2.5 → T-07.2.6 → T-07.2.7 → T-07.2.8 → T-07.2.9 → T-07.2.10 → T-07.2.11

Nota: T-07.2.1 a T-07.2.5 (código Go) pueden desarrollarse en paralelo con HU-07.1 (deploy de Kratos).
      T-07.2.6+ (deploy del IAM Service) requiere HU-07.1 completada.
```
