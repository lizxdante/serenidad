# Orden de Implementación — Detalle Completo (Parte B)
## Subcapas 1.E a 3.D + Grafo de Dependencias + Matriz + Definición de Hecho

> **Complemento de:** [`plans/05_orden_implementacion_capas_exhaustivo.md`](./05_orden_implementacion_capas_exhaustivo.md)
> Este documento contiene el detalle exhaustivo de todas las subcapas desde 1.E en adelante,
> el grafo global de dependencias, la matriz de componentes y la definición de hecho por capa.
>
> **⚠️ NOTA DE INFRAESTRUCTURA (Abril 2026):**
> Las secciones `docker-compose.yml` en este documento aplican **solo a entorno de desarrollo local**.
> En **producción** (Hetzner CX32 + Talos Linux + Kubernetes 1.33.x), cada servicio se despliega
> como `Deployment + Service` en el namespace `serenamente-core` via FluxCD GitOps.
> Los templates de `Deployment` k8s están en [`plans/06_talos_k8s_decision_y_cambios_en_cadena.md`](./06_talos_k8s_decision_y_cambios_en_cadena.md).
> Las referencias a "Caddy forward_auth" corresponden al equivalente Kubernetes: **Traefik ForwardAuth Middleware**.

---

## 4.5 Subcapa 1.E — IAM: Ory Kratos v1.3.1 (AuthN)

**Prerequisito:** 4.3 (PostgreSQL con `kratos_db` y usuario `kratos_user`).

**Por qué antes que el IAM Domain Service Go:** Kratos es el proveedor de autenticación. El IAM Domain Service llama a la Admin API de Kratos para verificar sesiones. Sin Kratos funcionando, el IAM Service no puede verificar su funcionamiento en tiempo real.

### 4.5.1 — Identity Schemas

**`infra/kratos/schemas/patient.json`:**
```json
{
  "$id": "https://api.serenamente.com/schemas/identity/patient.json",
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Patient",
  "type": "object",
  "properties": {
    "traits": {
      "type": "object",
      "properties": {
        "email": {
          "type": "string",
          "format": "email",
          "title": "E-Mail",
          "ory.sh/kratos": {
            "credentials": {
              "password": { "identifier": true },
              "webauthn": { "identifier": true }
            },
            "verification": { "via": "email" },
            "recovery": { "via": "email" }
          }
        },
        "name": {
          "type": "object",
          "properties": {
            "first": { "type": "string", "title": "First Name" },
            "last":  { "type": "string", "title": "Last Name" }
          }
        }
      },
      "required": ["email"]
    }
  }
}
```

**`infra/kratos/schemas/doctor.json`:** Igual al de `patient.json` pero añade traits opcionales:
- `specialty_snomed`: código SNOMED-CT de la especialidad médica
- `license_number`: número de cédula/licencia profesional
- `country_iso2`: país de ejercicio (`"PE"` | `"MX"` | `"CO"` etc.)

El IAM Domain Service enriquece estos datos al crear el perfil de dominio en `iam_db`.

### 4.5.2 — `infra/kratos/kratos.yml`

```yaml
version: v1.3
dsn: postgres://kratos_user:${KRATOS_DB_PASSWORD}@postgres:5432/kratos_db?sslmode=disable

serve:
  public:
    base_url: https://api.serenamente.com/
    cors:
      enabled: true
      allowed_origins: [https://app.serenamente.com]
  admin:
    base_url: http://ory-kratos:4434/
    host: 0.0.0.0   # Solo accesible desde la red Docker interna

selfservice:
  default_browser_return_url: https://app.serenamente.com/
  flows:
    login:
      ui_url: https://app.serenamente.com/auth/login
      lifespan: 10m
    registration:
      ui_url: https://app.serenamente.com/auth/register
      lifespan: 10m
    recovery:
      enabled: true
      ui_url: https://app.serenamente.com/auth/recovery
      use: code
    verification:
      enabled: true
      ui_url: https://app.serenamente.com/auth/verification
      use: code
  methods:
    passkey:
      enabled: true
      config:
        rp:
          display_name: Serenamente
          id: serenamente.com
          origins: [https://app.serenamente.com]
    password:
      enabled: false   # Cero contraseñas por diseño (R1: "Build It Right")
    totp:
      enabled: true    # MFA como segundo factor disponible

session:
  cookie:
    domain: .serenamente.com
    same_site: Lax
    persistent: true
  lifespan: 720h   # 30 días

identity:
  default_schema_id: patient
  schemas:
    - id: patient
      url: file:///etc/config/kratos/schemas/patient.json
    - id: doctor
      url: file:///etc/config/kratos/schemas/doctor.json

courier:
  smtp:
    connection_uri: smtps://${SMTP_USER}:${SMTP_PASS}@smtp.sendgrid.net:465
    from_name: Serenamente
    from_address: noreply@serenamente.com

log:
  level: info
  format: json
```

### 4.5.3 — docker-compose.yml: Servicio Ory Kratos

```yaml
  ory-kratos:
    image: oryd/kratos:v1.3.1
    container_name: serenamente-kratos
    restart: unless-stopped
    depends_on:
      postgres: { condition: service_healthy }
    command: serve --config /etc/config/kratos/kratos.yml
    volumes:
      - ./infra/kratos:/etc/config/kratos:ro
    ports:
      - "4433:4433"   # Public API → expuesta via Caddy en /auth/*
      - "4434:4434"   # Admin API → SOLO interna, NUNCA exponer al exterior
    networks: [serenamente_net]
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:4434/health/ready | grep -q 'ok'"]
      interval: 10s
      timeout: 5s
      retries: 10
```

Kratos aplica migraciones de su schema SQL automáticamente al arrancar contra `kratos_db` vacía. No se necesita ejecutar `migrate` manualmente.

**Artefactos resultantes:** `infra/kratos/kratos.yml`, `infra/kratos/schemas/patient.json`, `infra/kratos/schemas/doctor.json`, servicio `ory-kratos` en `docker-compose.yml`.

**Criterio de aceptación:**
- `curl http://localhost:4434/health/ready` → `{"status":"ok"}`.
- `curl https://api.serenamente.com/auth/registration/api` devuelve un Kratos registration flow JSON.
- Tabla `identities` existe en `kratos_db` (migración aplicada).
- El self-service flow de Passkey funciona desde `app.serenamente.com` (WebAuthn challenge resuelto).

---

## 4.6 Subcapa 1.F — IAM: IAM Domain Service en Go 1.25.x

**Prerequisito:** 4.3 (`iam_db`), 4.4 (schemas Proto generados), 4.5 (Kratos funcionando), 1.B (Caddy para forward_auth).

**Componente más importante de Fase 1.** Emite el JWT enriquecido Ed25519 que alimenta el RLS de toda la persistencia. Sin él, ningún endpoint de negocio puede ser autenticado ni autorizado.

### 4.6.1 — Estructura completa: `services/iam/`

```
services/iam/
├── cmd/
│   └── main.go                    # Chi router, DI manual, graceful shutdown
├── internal/
│   ├── config/
│   │   └── config.go              # Env vars: DB_URL, KRATOS_ADMIN_URL, JWT_KEY, etc.
│   ├── domain/
│   │   ├── user.go                # Entidad User (UUIDv7, role, tenant_id, DID, is_active)
│   │   └── outbox_event.go        # Entidad OutboxEvent (id, type, subject, payload, published_at)
│   ├── jwt/
│   │   └── signer.go              # Ed25519 sign/verify con golang-jwt/jwt v5.x
│   ├── kratos/
│   │   └── client.go              # Wrapper Kratos Admin API :4434 (session verify)
│   ├── outbox/
│   │   └── worker.go              # Goroutine: SELECT pending → NATS publish → UPDATE
│   ├── repository/
│   │   ├── user_repo.go           # pgxpool: INSERT/SELECT en tabla users
│   │   └── outbox_repo.go         # pgxpool: INSERT/SELECT/UPDATE en outbox_events
│   └── handler/
│       ├── token.go               # POST /api/iam/token/exchange
│       ├── validate.go            # GET /internal/validate-token (Caddy forward_auth)
│       ├── register.go            # POST /api/iam/users (crear perfil de dominio)
│       ├── users.go               # GET /api/iam/users/{id}
│       └── health.go              # GET /health, GET /metrics
├── migrations/
│   ├── 000001_create_users.up.sql
│   ├── 000001_create_users.down.sql
│   ├── 000002_create_outbox.up.sql
│   └── 000002_create_outbox.down.sql
├── Dockerfile                     # Multi-stage: golang:1.25-alpine → alpine:3.20
├── go.mod
└── go.sum
```

### 4.6.2 — Schema BD: `000001_create_users.up.sql`

```sql
CREATE TABLE users (
    id         UUID PRIMARY KEY,
    kratos_id  UUID UNIQUE NOT NULL,       -- ID de Ory Kratos
    email      TEXT UNIQUE NOT NULL,
    role       TEXT NOT NULL CHECK (role IN ('patient', 'doctor', 'admin')),
    tenant_id  UUID NOT NULL,              -- UUIDv7 de la organización
    did        TEXT UNIQUE,               -- did:web:serenamente.com:users:{id}
    is_active  BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_kratos_id ON users(kratos_id);
CREATE INDEX idx_users_email     ON users(email);
CREATE INDEX idx_users_role      ON users(role);
CREATE INDEX idx_users_tenant_id ON users(tenant_id);
```

### 4.6.3 — Schema Outbox: `000002_create_outbox.up.sql`

```sql
CREATE TABLE outbox_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type      TEXT NOT NULL,          -- "UserRegistered" | "DoctorOnboarded"
    nats_subject    TEXT NOT NULL,          -- "iam.users.registered.v1"
    payload         BYTEA NOT NULL,         -- Protobuf proto3 serializado
    idempotency_key TEXT UNIQUE,            -- Previene duplicados en reintentos
    published_at    TIMESTAMPTZ,            -- NULL = pendiente de publicar
    retry_count     INTEGER NOT NULL DEFAULT 0,
    last_error      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Índice crítico: el Outbox Worker solo consulta eventos pendientes
CREATE INDEX idx_outbox_pending ON outbox_events(created_at ASC)
    WHERE published_at IS NULL;
```

### 4.6.4 — Claims del JWT Enriquecido (Ed25519)

```go
type EnrichedClaims struct {
    Sub      string `json:"sub"`        // UUIDv7 del usuario (PK en users)
    Role     string `json:"role"`       // "patient" | "doctor" | "admin"
    TenantID string `json:"tenant_id"`  // UUIDv7 de la organización
    DID      string `json:"did"`        // did:web:serenamente.com:users:{sub}
    Email    string `json:"email"`
    jwt.RegisteredClaims               // iat, exp, iss, aud
}
// Librería: golang-jwt/jwt v5.x
// Clave privada Ed25519: cargada desde env var base64 — NUNCA hardcoded
// Duración: 3600s (1 hora) — refresh via /api/iam/token/exchange
```

### 4.6.5 — Endpoint `/internal/validate-token` (Caddy forward_auth)

Este endpoint es llamado por Caddy para **cada request** a `/api/*`. Debe ser ultrarrápido (P99 < 5ms):

- Lee `Authorization: Bearer <token>` del header.
- Verifica firma Ed25519 con clave pública en memoria (sin roundtrip a DB ni a Kratos).
- Si válido: responde `200` con headers `X-User-ID`, `X-User-Role`, `X-Tenant-ID`.
- Si inválido/expirado: responde `401` → Caddy rechaza la petición original con `403`.

La verificación es local y sin estado. La clave pública se carga una sola vez al iniciar el servicio.

### 4.6.6 — Outbox Worker: goroutine interna

```go
func (w *OutboxWorker) Run(ctx context.Context) {
    ticker := time.NewTicker(500 * time.Millisecond)
    defer ticker.Stop()
    for {
        select {
        case <-ctx.Done():
            return  // Graceful shutdown
        case <-ticker.C:
            w.publishPendingEvents(ctx)
        }
    }
}

func (w *OutboxWorker) publishPendingEvents(ctx context.Context) {
    events, err := w.repo.GetPending(ctx, 50)
    if err != nil || len(events) == 0 {
        return
    }
    for _, e := range events {
        if err := w.nc.Publish(e.NATSSubject, e.Payload); err != nil {
            w.repo.IncrementRetry(ctx, e.ID, err.Error())
            continue  // NATS no disponible: backoff implícito en el próximo tick
        }
        w.repo.MarkPublished(ctx, e.ID)
    }
}
```

Si NATS no está disponible (Fase 1 sin NATS aún), el worker falla silenciosamente en cada tick y reintenta. El servicio no crashea por la ausencia de NATS — esto es un escenario esperado en Fase 1.

### 4.6.7 — Dockerfile multi-stage

```dockerfile
FROM golang:1.25-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /iam-service ./cmd/main.go

FROM alpine:3.20
RUN apk add --no-cache ca-certificates tzdata
COPY --from=builder /iam-service /usr/local/bin/iam-service
EXPOSE 8080
ENTRYPOINT ["iam-service"]
# Binario final: ~10-15MB — sin runtime de Go en la imagen final
```

### 4.6.8 — docker-compose.yml: IAM Service

```yaml
  iam-service:
    image: serenamente/iam:latest
    build: { context: ./services/iam }
    container_name: serenamente-iam
    restart: unless-stopped
    depends_on:
      postgres:   { condition: service_healthy }
      ory-kratos: { condition: service_healthy }
    environment:
      IAM_DB_URL: postgres://iam_user:${IAM_DB_PASSWORD}@postgres:5432/iam_db?sslmode=disable
      KRATOS_ADMIN_URL: http://ory-kratos:4434
      NATS_URL: nats://nats:4222
      JWT_PRIVATE_KEY_ED25519: ${JWT_PRIVATE_KEY_ED25519}  # base64 de clave privada
      JWT_ISSUER: https://api.serenamente.com
      JWT_EXPIRY: 3600
      LOG_LEVEL: info
      PORT: 8080
    networks: [serenamente_net]
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:8080/health | grep -q 'ok'"]
      interval: 10s
      timeout: 5s
      retries: 5
```

**Artefactos resultantes:** `services/iam/` completo, Dockerfile, migraciones SQL, servicio en `docker-compose.yml`.

**Criterio de aceptación:**
- `docker compose build iam-service` compila sin errores.
- Flujo completo: Passkey en Kratos → `POST /api/iam/token/exchange` → JWT Ed25519 → `GET /internal/validate-token` → headers `X-User-ID/Role/Tenant-ID` en el request a Caddy.
- Request a `/api/*` con JWT inválido → `401` vía Caddy.
- Tablas `users` y `outbox_events` existen en `iam_db` (golang-migrate aplicó migraciones).
- `GET /metrics` expone métricas Prometheus del servicio.

---

## 4.7 Subcapa 1.G — BFF: Cloudflare Workers + Hono v4.x

**Prerequisito:** 4.6 (IAM emitiendo JWTs). Clave pública Ed25519 como secret del Worker.

### 4.7.1 — Estructura completa: `apps/bff/`

```
apps/bff/
├── src/
│   ├── index.ts                  # Entry point: Hono app, middleware, routes
│   ├── middleware/
│   │   ├── auth.ts               # JWT EdDSA verificación local (jose)
│   │   └── cors.ts               # CORS restrictivo a app.serenamente.com
│   ├── routes/
│   │   ├── iam.ts                # Proxy /api/iam/* → VPS
│   │   ├── scheduling.ts         # Proxy /api/scheduling/* → VPS
│   │   ├── clinical.ts           # Proxy /api/clinical/* → VPS
│   │   └── billing.ts            # Proxy /api/billing/* → VPS
│   └── types.ts                  # CloudflareBindings, JWTPayload, etc.
├── package.json
├── tsconfig.json                  # TypeScript 6.0, target: es2025
└── wrangler.toml
```

### 4.7.2 — `src/index.ts` (Hono v4.x BFF)

```typescript
import { Hono } from "hono";
import { cors } from "hono/cors";
import { verifyJwt } from "./middleware/auth";

const app = new Hono<{ Bindings: CloudflareBindings }>();

// CORS: solo el portal de la aplicación
app.use("*", cors({
  origin: ["https://app.serenamente.com"],
  allowMethods: ["GET", "POST", "PUT", "DELETE", "PATCH"],
  allowHeaders: ["Authorization", "Content-Type"],
  credentials: true,
}));

// JWT check en todas las rutas /api/* EXCEPTO el exchange de token
app.use("/api/*", async (c, next) => {
  const path = new URL(c.req.url).pathname;
  if (path === "/api/iam/token/exchange") return next();
  return verifyJwt(c, next);
});

// Proxy transparente al VPS, añadiendo headers de claims del JWT
const VPS_BASE = "https://api.serenamente.com";
app.all("/api/*", async (c) => {
  const url = new URL(c.req.url);
  const targetUrl = `${VPS_BASE}${url.pathname}${url.search}`;
  const response = await fetch(targetUrl, {
    method: c.req.method,
    headers: {
      ...Object.fromEntries(c.req.raw.headers),
      "X-User-ID":   c.get("userId")   ?? "",
      "X-User-Role": c.get("userRole") ?? "",
      "X-Tenant-ID": c.get("tenantId") ?? "",
    },
    body: ["GET", "HEAD"].includes(c.req.method) ? undefined : c.req.raw.body,
  });
  return new Response(response.body, { status: response.status, headers: response.headers });
});

export default app;
```

### 4.7.3 — `middleware/auth.ts` — Verificación JWT EdDSA

```typescript
import { importJWK, jwtVerify } from "jose";
import { HTTPException } from "hono/http-exception";

export async function verifyJwt(c: Context, next: Next) {
  const authHeader = c.req.header("Authorization");
  if (!authHeader?.startsWith("Bearer ")) {
    throw new HTTPException(401, { message: "Missing Bearer token" });
  }
  const token = authHeader.slice(7);
  try {
    // La clave pública es un JWK base64 en la variable de entorno del Worker
    // Se importa en cada request (V8 Isolates son stateless — usar cache KV si es necesario)
    const publicKey = await importJWK(
      JSON.parse(atob(c.env.JWT_PUBLIC_KEY_ED25519)),
      "EdDSA"
    );
    const { payload } = await jwtVerify(token, publicKey, {
      issuer: "https://api.serenamente.com",
      algorithms: ["EdDSA"],
    });
    // Enriquecer el contexto con los claims del JWT para usar en el proxy
    c.set("userId",   payload.sub as string);
    c.set("userRole", payload["role"] as string);
    c.set("tenantId", payload["tenant_id"] as string);
  } catch {
    throw new HTTPException(401, { message: "Invalid or expired token" });
  }
  return next();
}
```

### 4.7.4 — `wrangler.toml`

```toml
name = "serenamente-bff"
main = "src/index.ts"
compatibility_date = "2026-04-01"
compatibility_flags = ["nodejs_compat"]

[vars]
ENVIRONMENT = "production"

# Secrets — configurar via: wrangler secret put JWT_PUBLIC_KEY_ED25519
# JWT_PUBLIC_KEY_ED25519 = "<base64 del JWK de la clave pública Ed25519>"
```

**Criterio de aceptación:**
- `bun run deploy` despliega en CF Workers exitosamente.
- Token inválido → `401`. Token válido → proxy correcto con headers `X-User-*`.
- Cold start: 0ms (V8 Isolates). Capacidad: 100K req/día free tier.
- TypeScript 6.0 compila sin errores con `target: "ES2025"`.

---

## 4.8 Subcapa 1.H — Frontend SPA: Qwik v2.0 en Cloudflare Pages

**Prerequisito:** 4.7 (BFF operativo), 4.5 (Kratos self-service flows disponibles).

### 4.8.1 — Migración Qwik v1.x → v2.0 (breaking change)

```bash
# Eliminar paquetes v1.x
bun remove @builder.io/qwik @builder.io/qwik-city

# Instalar v2.0 con nuevo scope @qwik.dev
bun add @qwik.dev/qwik @qwik.dev/qwik-city @qwik.dev/qwik-city/middleware/cloudflare-pages

# Actualizar tsconfig.json para TypeScript 6.0
# "target": "ES2025", "lib": ["ES2025", "DOM"]
```

Todos los imports en `src/` deben actualizarse:
- `"@builder.io/qwik"` → `"@qwik.dev/qwik"`
- `"@builder.io/qwik-city"` → `"@qwik.dev/qwik-city"`

### 4.8.2 — Flujos de UI para Fase 1 (solo autenticación)

Los portales médico y paciente se construyen en Fase 2. En Fase 1 solo se implementan:

| Ruta | Descripción |
|------|-------------|
| `/auth/login` | Botón Passkey → Kratos self-service login flow. Sin contraseña. |
| `/auth/register` | Formulario mínimo → Kratos registration flow con Passkey. |
| `/auth/callback` | Recibe sesión Kratos → `POST /api/iam/token/exchange` → almacena JWT en cookie HttpOnly vía BFF. |
| `/dashboard` | Vista de bienvenida post-login. Muestra nombre, rol, y links al portal correspondiente. Ruta protegida. |

### 4.8.3 — Protección de rutas

```typescript
// src/routes/layout.tsx — wrapper para rutas protegidas
export const onRequest: RequestHandler = async ({ cookie, redirect }) => {
  const token = cookie.get("serenamente_jwt");
  if (!token?.value) {
    throw redirect(302, "/auth/login");
  }
  // El BFF ya validó el JWT — aquí solo verificamos que existe la cookie
};
```

**Criterio de aceptación:** Registro con Passkey funcional en producción. JWT en cookie HttpOnly. `/dashboard` redirige a `/auth/login` para usuarios no autenticados. TTI < 100ms (Qwik v2.0 resumability).

---

## 4.9 Subcapa 1.I — Backups: pg_dump + rclone + Backblaze B2

**Prerequisito:** 4.3 (PostgreSQL con datos de producción).

**Regla fundamental:** Operativos desde el **primer día** con datos reales. No es una tarea de "Fase 3".

### 4.9.1 — Cuenta Backblaze B2

1. Crear cuenta en `backblaze.com` (10GB gratis, luego $0.006/GB/mes).
2. Crear bucket privado: `serenamente-backups`.
3. Crear Application Key con permisos solo al bucket.
4. Guardar `B2_ACCOUNT_ID` y `B2_APPLICATION_KEY` en `.env` del VPS.

### 4.9.2 — Instalación y configuración de rclone

```bash
curl https://rclone.org/install.sh | sudo bash
rclone config create serenamente-b2 b2 \
    account "${B2_ACCOUNT_ID}" \
    key "${B2_APPLICATION_KEY}"
rclone mkdir serenamente-b2:serenamente-backups/postgres/
```

### 4.9.3 — `scripts/backup.sh`

```bash
#!/bin/bash
set -euo pipefail

DATE=$(date -u +%Y%m%d_%H%M%S)
BACKUP_DIR="/var/backups/serenamente/postgres"
CONTAINER="serenamente-postgres"
PG_USER="serenamente_admin"
DATABASES=("iam_db" "scheduling_db" "clinical_db" "billing_db" "kratos_db" "openfga_db")

mkdir -p "${BACKUP_DIR}"
echo "[${DATE}] Iniciando backup de ${#DATABASES[@]} databases..."

for DB in "${DATABASES[@]}"; do
    BACKUP_FILE="${BACKUP_DIR}/${DB}_${DATE}.sql.gz"
    docker exec "${CONTAINER}" pg_dump \
        -U "${PG_USER}" --format=custom --compress=9 "${DB}" \
        | gzip > "${BACKUP_FILE}"
    echo "  ✓ ${DB}: $(du -sh ${BACKUP_FILE} | cut -f1)"
done

echo "Subiendo a Backblaze B2..."
rclone copy "${BACKUP_DIR}" serenamente-b2:serenamente-backups/postgres/ \
    --min-age 1s --log-level INFO

# Retener solo últimos 30 días localmente
find "${BACKUP_DIR}" -name "*.sql.gz" -mtime +30 -delete

echo "[${DATE}] Backup completado. Costo estimado B2: ~0.006 USD/GB/mes"
```

### 4.9.4 — `scripts/verify-restore.sh`

```bash
#!/bin/bash
# Verificación semanal: garantizar que los backups son restaurables
set -euo pipefail

LATEST_IAM=$(ls -t /var/backups/serenamente/postgres/iam_db_*.sql.gz | head -1)
echo "Verificando backup: ${LATEST_IAM}"
pg_restore --list "${LATEST_IAM}" | head -20
echo "Backup verificado: restauración viable."
```

### 4.9.5 — Cron jobs

```bash
chmod +x /opt/serenamente/scripts/backup.sh
chmod +x /opt/serenamente/scripts/verify-restore.sh

# Backup diario 02:00 UTC
echo "0 2 * * * serenamente /opt/serenamente/scripts/backup.sh >> /var/log/serenamente-backup.log 2>&1" \
    | sudo tee /etc/cron.d/serenamente-backup

# Verificación semanal domingo 03:00 UTC
echo "0 3 * * 0 serenamente /opt/serenamente/scripts/verify-restore.sh >> /var/log/serenamente-verify.log 2>&1" \
    | sudo tee -a /etc/cron.d/serenamente-backup
```

**Criterio de aceptación:** 6 archivos `.sql.gz` en B2 después del primer run. `pg_restore --list` lista objetos restaurables. Cron ejecuta sin errores. Alertas si el cron falla (via Alertmanager en Fase 3).

---

## 5. Fase 2 — Motor Médico: Detalle Completo

### 5.1 Subcapa 2.A — Event Bus: NATS JetStream v2.11.x

> ⚠️ **BREAKING vs v2.10.x.** Consultar [docs.nats.io/running-a-nats-service/upgrading](https://docs.nats.io/running-a-nats-service/upgrading) antes de instalar.

**Prerequisito:** 4.1 (Docker en VPS). Primera tarea de Fase 2.

**Por qué primero:** Todos los microservicios de Fase 2 publican y consumen eventos vía NATS. Debe estar operativo con sus streams configurados antes de que cualquier servicio lo intente usar.

#### `infra/nats/nats.conf`

```hcl
port: 4222
http_port: 8222
server_name: serenamente-nats
jetstream: {
  store_dir: /data
  max_memory_store: 512MB
  max_file_store: 10GB
}
logtime: true
log_file: /var/log/nats/nats.log
max_payload: 10MB
```

#### docker-compose.yml: Servicio NATS

```yaml
  nats:
    image: nats:2.11-alpine
    container_name: serenamente-nats
    restart: unless-stopped
    command: ["--config=/etc/nats/nats.conf"]
    volumes:
      - ./infra/nats/nats.conf:/etc/nats/nats.conf:ro
      - nats-data:/data
      - /var/log/nats:/var/log/nats
    ports:
      - "4222:4222"   # Client port — solo red interna
      - "8222:8222"   # Monitoring — solo red interna
    networks: [serenamente_net]
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:8222/healthz | grep -q 'ok'"]
      interval: 10s
      timeout: 5s
      retries: 5
```

#### 4 Streams Permanentes

| Stream | Subject | Retención | Razón |
|--------|---------|-----------|-------|
| `SERENAMENTE_IAM` | `iam.>` | 365 días | Auditoría de identidad (1 año) |
| `SERENAMENTE_SCHED` | `scheduling.>` | 730 días | Historial de citas (2 años) |
| `SERENAMENTE_CLINICAL` | `clinical.>` | **Indefinida** | Datos médicos permanentes por ley |
| `SERENAMENTE_BILLING` | `billing.>` | 3650 días | Requisito fiscal LATAM (10 años) |

```bash
nats stream add SERENAMENTE_IAM \
    --subjects "iam.>" --storage file --retention limits --max-age 365d --replicas 1
nats stream add SERENAMENTE_SCHED \
    --subjects "scheduling.>" --storage file --retention limits --max-age 730d --replicas 1
nats stream add SERENAMENTE_CLINICAL \
    --subjects "clinical.>" --storage file --retention limits --replicas 1
nats stream add SERENAMENTE_BILLING \
    --subjects "billing.>" --storage file --retention limits --max-age 3650d --replicas 1
```

Todos los streams: `storage=file` (persiste en disco), consumer ACK policy `explicit`, `MaxDeliver=5` (auto-DLQ tras 5 fallos).

**Criterio de aceptación:** `nats stream ls` lista 4 streams. Mensaje publicado en `iam.test.1` aparece en `SERENAMENTE_IAM`. NATS sobrevive `docker restart serenamente-nats` sin perder mensajes almacenados.

---

### 5.2 Subcapa 2.B — Outbox Pattern: Librería Go Compartida

**Prerequisito:** 5.1 (NATS).

Librería Go en `packages/go/outbox/`. Importada por IAM, Scheduling, Clinical y Billing. Evita duplicar la lógica del worker en cada microservicio.

**Estructura:**
```
packages/go/outbox/
├── worker.go         # Worker genérico: pgxpool + nats.Conn → ciclo SELECT/publish/UPDATE
├── event.go          # Tipo OutboxEvent y interface Repository
├── schema.sql        # Template del schema de tabla outbox (cada servicio lo incluye en su migración)
└── go.mod
```

**Garantía:** at-least-once delivery. **Responsabilidad del consumidor:** idempotencia via `event_id` UUIDv7 como deduplication key. Los consumidores NATS deben guardar los `event_id` procesados y rechazar duplicados.

**Criterio de aceptación:** Tests unitarios con mocks de pgxpool y nats pasan. Evento pendiente publicado en < 1s. `idempotency_key` unique constraint previene doble INSERT.

---

### 5.3 Subcapa 2.C — Scheduling Service

**Prerequisito:** 4.3 (`scheduling_db` con extensión `btree_gist`), 4.4 (proto scheduling generado), 4.6 (IAM JWT), 5.1 (NATS), 5.2 (Outbox librería).

**Rol:** Propietario de la agenda médica. Reemplaza Cal.com. Produce `AppointmentBooked` y `AppointmentCancelled`. Consume `DoctorOffboarded`.

#### Schema BD principal: `000001_create_appointments.up.sql`

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE doctors_availability (
    id          UUID PRIMARY KEY,
    doctor_id   UUID NOT NULL,
    day_of_week SMALLINT NOT NULL CHECK (day_of_week BETWEEN 0 AND 6),
    start_time  TIME NOT NULL,
    end_time    TIME NOT NULL,
    timezone    TEXT NOT NULL,   -- IANA: "America/Lima" | "America/Mexico_City"
    is_active   BOOLEAN NOT NULL DEFAULT true,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE appointments (
    id             UUID PRIMARY KEY,       -- UUIDv7
    doctor_id      UUID NOT NULL,
    patient_id     UUID NOT NULL,
    time_slot      TSTZRANGE NOT NULL,     -- PostgreSQL range: [start, end)
    status         TEXT NOT NULL DEFAULT 'BOOKED'
        CHECK (status IN ('BOOKED', 'COMPLETED', 'CANCELLED', 'NO_SHOW')),
    service_snomed TEXT NOT NULL,          -- Código SNOMED-CT del servicio médico
    timezone_iana  TEXT NOT NULL,
    fhir_resource  JSONB,                  -- FHIR R4 Appointment JSON completo
    notes          TEXT,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- GARANTÍA POSTGRESQL: Un médico no puede tener dos citas solapadas
    -- La base de datos rechaza el conflicto antes de que llegue al código Go
    EXCLUDE USING GIST (
        doctor_id WITH =,
        time_slot WITH &&
    ) WHERE (status = 'BOOKED')
);

CREATE INDEX idx_appointments_doctor_id   ON appointments(doctor_id);
CREATE INDEX idx_appointments_patient_id  ON appointments(patient_id);
CREATE INDEX idx_appointments_time_slot   ON appointments USING GIST(time_slot);
CREATE INDEX idx_appointments_doctor_slot ON appointments USING GIST(doctor_id, time_slot);
```

#### FHIR R4 Appointment Output

```go
// internal/fhir/appointment_mapper.go
func ToFHIRR4(apt *domain.Appointment) FHIRAppointment {
    return FHIRAppointment{
        ResourceType: "Appointment",
        ID:           apt.ID.String(),
        Status:       fhirStatusFrom(apt.Status),
        ServiceType: []CodeableConcept{{
            Coding: []Coding{{
                System:  "http://snomed.info/sct",
                Code:    apt.ServiceSNOMED,
                Display: apt.ServiceDisplay,
            }},
        }},
        Start: apt.TimeSlot.Lower.Format(time.RFC3339),
        End:   apt.TimeSlot.Upper.Format(time.RFC3339),
        Participant: []Participant{
            {Actor: Reference{Reference: "Practitioner/" + apt.DoctorID.String()}},
            {Actor: Reference{Reference: "Patient/" + apt.PatientID.String()}},
        },
        Meta: Meta{
            Profile: []string{"http://hl7.org/fhir/StructureDefinition/Appointment"},
        },
    }
}
```

#### docker-compose.yml: Scheduling Service

```yaml
  scheduling-service:
    image: serenamente/scheduling:latest
    build: { context: ./services/scheduling }
    container_name: serenamente-scheduling
    restart: unless-stopped
    depends_on:
      postgres: { condition: service_healthy }
      nats:     { condition: service_healthy }
      iam-service: { condition: service_healthy }
    environment:
      SCHEDULING_DB_URL: postgres://scheduling_user:${SCHEDULING_DB_PASSWORD}@postgres:5432/scheduling_db?sslmode=disable
      NATS_URL: nats://nats:4222
      IAM_JWT_PUBLIC_KEY: ${JWT_PUBLIC_KEY_ED25519}
      PORT: 8081
    networks: [serenamente_net]
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:8081/health | grep -q 'ok'"]
      interval: 10s
      retries: 5
```

**Criterio de aceptación:**
- `EXCLUDE USING GIST` rechaza INSERT con slots solapados (error de constraint PG).
- `GET /api/scheduling/availability?doctor_id=...&date=...` devuelve slots disponibles.
- `POST /api/scheduling/appointments` crea cita y `AppointmentBooked` aparece en `SERENAMENTE_SCHED`.
- Respuesta incluye FHIR R4 Appointment JSON válido (validable con HAPI FHIR validator).
- Consumer `DoctorOffboarded` cancela citas futuras del médico dado de baja.

---

### 5.4 Subcapa 2.D — Clinical Record Service

**Prerequisito:** 4.3 (`clinical_db`), 4.4 (proto clinical), 4.6 (IAM JWT para RLS), 5.1 (NATS), 5.2 (Outbox), 5.3 (Scheduling produce `AppointmentBooked`).

**El componente más complejo de toda la arquitectura.**

#### Event Store Schema (Inmutable): `000001_create_event_store.up.sql`

```sql
CREATE TABLE ehrs (
    id         UUID PRIMARY KEY,
    patient_id UUID UNIQUE NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
    -- Sin columnas de estado: todo estado se deriva del replay de clinical_events
    -- El EHR es solo el contenedor identificador
);

-- LA TABLA MÁS IMPORTANTE: Registro legal de salud. NUNCA se modifica.
CREATE TABLE clinical_events (
    id              UUID PRIMARY KEY,        -- UUIDv7 (orden temporal, B-tree eficiente)
    aggregate_id    UUID NOT NULL,           -- ID del EHR del paciente
    aggregate_type  TEXT NOT NULL
        CHECK (aggregate_type IN ('EHR', 'Encounter', 'Diagnosis', 'Prescription', 'Note')),
    event_type      TEXT NOT NULL,           -- "EncounterStarted" | "DiagnosisRecorded" | etc.
    event_version   INTEGER NOT NULL DEFAULT 1,  -- Para evolución de schema Protobuf
    payload         BYTEA NOT NULL,          -- Protobuf proto3 serializado
                                             -- Contiene openEHR Canonical JSON dentro del proto
    metadata        JSONB,                   -- doctor_id, ip_hash, user_agent_hash, etc.
    causation_id    UUID,                    -- ID del evento que causó este (trazabilidad)
    correlation_id  UUID,                    -- ID del request HTTP original (trazabilidad)
    sequence_number BIGINT GENERATED ALWAYS AS IDENTITY,  -- Secuencia global para ordering
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
    -- NO hay updated_at: intencional — es inmutable
    -- NO hay deleted_at: intencional — NUNCA se borra
);

CREATE INDEX idx_clinical_events_aggregate ON clinical_events(aggregate_id, sequence_number);
CREATE INDEX idx_clinical_events_type      ON clinical_events(event_type);
CREATE INDEX idx_clinical_events_date      ON clinical_events(occurred_at DESC);
```

#### Read Models: `000002_create_projections.up.sql`

```sql
-- Proyecciones actualizadas sincrónicamente en la misma TX que el AppendEvent
-- Evitan replay completo para queries frecuentes
CREATE TABLE encounters_view (
    id             UUID PRIMARY KEY,
    ehr_id         UUID NOT NULL REFERENCES ehrs(id),
    patient_id     UUID NOT NULL,
    doctor_id      UUID NOT NULL,
    appointment_id UUID,
    status         TEXT NOT NULL CHECK (status IN ('DRAFT', 'IN_PROGRESS', 'COMPLETED')),
    started_at     TIMESTAMPTZ,
    completed_at   TIMESTAMPTZ,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE diagnoses_view (
    id            UUID PRIMARY KEY,   -- ID del evento DiagnosisRecorded
    ehr_id        UUID NOT NULL REFERENCES ehrs(id),
    encounter_id  UUID NOT NULL REFERENCES encounters_view(id),
    patient_id    UUID NOT NULL,
    doctor_id     UUID NOT NULL,
    icd11_code    TEXT NOT NULL,      -- Para filtros y estadísticas
    icd11_display TEXT NOT NULL,      -- Para UI sin deserializar Protobuf
    archetype_id  TEXT NOT NULL,      -- Archetype openEHR usado
    recorded_at   TIMESTAMPTZ NOT NULL
);

CREATE TABLE prescriptions_view (
    id              UUID PRIMARY KEY,
    ehr_id          UUID NOT NULL REFERENCES ehrs(id),
    encounter_id    UUID NOT NULL REFERENCES encounters_view(id),
    patient_id      UUID NOT NULL,
    doctor_id       UUID NOT NULL,
    medication_code TEXT NOT NULL,    -- Código ATC (Anatomical Therapeutic Chemical)
    medication_name TEXT NOT NULL,
    dosage          TEXT NOT NULL,
    frequency       TEXT NOT NULL,
    duration_days   INTEGER,
    prescribed_at   TIMESTAMPTZ NOT NULL
);
```

#### Row-Level Security: `000003_enable_rls.up.sql`

```sql
ALTER TABLE clinical_events    ENABLE ROW LEVEL SECURITY;
ALTER TABLE encounters_view    ENABLE ROW LEVEL SECURITY;
ALTER TABLE diagnoses_view     ENABLE ROW LEVEL SECURITY;
ALTER TABLE prescriptions_view ENABLE ROW LEVEL SECURITY;

-- Policy SELECT: paciente ve solo su EHR; médico ve los EHRs de sus pacientes
CREATE POLICY patient_reads_own ON clinical_events
    FOR SELECT USING (
        aggregate_id IN (
            SELECT id FROM ehrs
            WHERE patient_id = current_setting('app.user_id', TRUE)::uuid
        )
        OR current_setting('app.user_role', TRUE) IN ('doctor', 'admin')
    );

-- Inmutabilidad reforzada a nivel de base de datos
CREATE POLICY append_only ON clinical_events
    FOR INSERT WITH CHECK (true);    -- Cualquier usuario autenticado puede insertar

CREATE POLICY deny_update ON clinical_events
    FOR UPDATE USING (false);        -- NUNCA permitido — false = siempre rechazado

CREATE POLICY deny_delete ON clinical_events
    FOR DELETE USING (false);        -- NUNCA permitido
```

#### AppendEvent: núcleo del Event Store

```go
func (s *EventStore) AppendEvent(ctx context.Context, event *ClinicalEvent) error {
    payload, err := proto.Marshal(event.Proto)
    if err != nil {
        return fmt.Errorf("marshal event %s: %w", event.Type, err)
    }

    tx, err := s.pool.Begin(ctx)
    if err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }
    defer tx.Rollback(ctx)  // No-op si ya se hizo Commit

    // 1. Inyectar JWT claims para que RLS evalúe correctamente las policies
    tx.Exec(ctx, "SET LOCAL app.user_id   = $1", event.ActorID)
    tx.Exec(ctx, "SET LOCAL app.user_role = $1", event.ActorRole)

    // 2. INSERT del evento (INMUTABLE — NUNCA UPDATE ni DELETE)
    if _, err = tx.Exec(ctx, `
        INSERT INTO clinical_events
            (id, aggregate_id, aggregate_type, event_type, event_version, payload, metadata, causation_id, correlation_id, occurred_at)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10)`,
        event.ID, event.AggregateID, event.AggregateType, event.Type,
        event.Version, payload, event.Metadata, event.CausationID,
        event.CorrelationID, event.OccurredAt,
    ); err != nil {
        return fmt.Errorf("insert event: %w", err)
    }

    // 3. INSERT en outbox en la MISMA TX (at-least-once garantizado)
    if _, err = tx.Exec(ctx, `
        INSERT INTO clinical_outbox (event_type, nats_subject, payload, idempotency_key)
        VALUES ($1, $2, $3, $4)`,
        event.Type, "clinical."+event.NATSSubject, payload, event.ID.String(),
    ); err != nil {
        return fmt.Errorf("insert outbox: %w", err)
    }

    // 4. Actualizar proyección/read model en la MISMA TX
    if err = s.projections.Update(ctx, tx, event); err != nil {
        return fmt.Errorf("update projection: %w", err)
    }

    return tx.Commit(ctx)
    // Si cualquier paso falla → rollback completo: consistencia garantizada
}
```

#### Consumidor NATS: `AppointmentBooked` → Encounter DRAFT

```go
func (c *Consumer) HandleAppointmentBooked(ctx context.Context, msg jetstream.Msg) error {
    var event schedulingv1.AppointmentBookedEvent
    if err := proto.Unmarshal(msg.Data(), &event); err != nil {
        msg.Nak()
        return fmt.Errorf("unmarshal AppointmentBooked: %w", err)
    }

    // Obtener o crear el EHR del paciente (cada paciente tiene exactamente 1 EHR)
    ehrID, err := c.ehrRepo.GetOrCreate(ctx, event.PatientId)
    if err != nil {
        msg.NakWithDelay(5 * time.Second)
        return err
    }

    // Crear el Encounter en estado DRAFT automáticamente
    encounterID := newUUIDv7()
    clinicalEvent := &ClinicalEvent{
        ID:            newUUIDv7(),
        AggregateID:   ehrID,
        AggregateType: "Encounter",
        Type:          "EncounterStarted",
        NATSSubject:   "encounters.started.v1",
        ActorID:       event.DoctorId,
        ActorRole:     "doctor",
        Proto: &clinicalv1.EncounterStartedEvent{
            EncounterId:   encounterID,
            AppointmentId: event.AppointmentId,
            PatientId:     event.PatientId,
            DoctorId:      event.DoctorId,
        },
    }

    if err := c.store.AppendEvent(ctx, clinicalEvent); err != nil {
        msg.NakWithDelay(5 * time.Second)
        return err
    }

    msg.Ack()  // ACK explícito: JetStream no reintentará este mensaje
    return nil
}
```

#### docker-compose.yml: Clinical Service

```yaml
  clinical-service:
    image: serenamente/clinical:latest
    build: { context: ./services/clinical }
    container_name: serenamente-clinical
    restart: unless-stopped
    depends_on:
      postgres: { condition: service_healthy }
      nats:     { condition: service_healthy }
      iam-service: { condition: service_healthy }
    environment:
      CLINICAL_DB_URL: postgres://clinical_user:${CLINICAL_DB_PASSWORD}@postgres:5432/clinical_db?sslmode=disable
      NATS_URL: nats://nats:4222
      IAM_JWT_PUBLIC_KEY: ${JWT_PUBLIC_KEY_ED25519}
      PORT: 8082
    networks: [serenamente_net]
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:8082/health | grep -q 'ok'"]
      interval: 10s
      retries: 5
```

**Criterio de aceptación:**
- `UPDATE` y `DELETE` en `clinical_events` → error `ERROR: new row violates row-level security policy`.
- `AppointmentBooked` de NATS crea automáticamente `Encounter DRAFT` en `clinical_db`.
- `POST /api/clinical/diagnoses` guarda el evento, actualiza `diagnoses_view`, publica `DiagnosisRecorded` en `SERENAMENTE_CLINICAL`.
- Paciente A no puede acceder al EHR de paciente B (RLS verificable con dos JWT distintos).
- `GET /api/clinical/ehr/{patient_id}` devuelve el historial completo paginado.

---

### 5.5 Subcapa 2.E — Landing Page: Astro v6.x en Cloudflare Pages

**Prerequisito:** Solo DNS `serenamente.com` apuntando a CF Pages. Puede desarrollarse en paralelo con Fase 2.

#### `astro.config.mjs`

```javascript
import { defineConfig } from "astro/config";
import qwik from "@qwik.dev/astro";
import cloudflare from "@astrojs/cloudflare";

export default defineConfig({
  output: "static",         // Zero JS por defecto
  adapter: cloudflare(),
  integrations: [qwik()],   // Solo para Qwik islands (formulario de contacto)
  compressHTML: true,
  build: { inlineStylesheets: "always" },
});
```

**Páginas a implementar:**
- `index.astro` — Hero, propuesta de valor, especialidades, CTA.
- `especialidades/psicologia.astro` — Detalle de la especialidad.
- `especialidades/psiquiatria.astro` — Detalle de la especialidad.
- `contacto.astro` — Formulario de contacto (Qwik island).

**Criterio de aceptación:** Lighthouse Performance ≥ 99. Lighthouse Accessibility ≥ 95. Zero JS en páginas de contenido estático. Formulario de contacto funcional.

---

## 6. Fase 3 — Plataforma Completa: Detalle Completo

### 6.1 Subcapa 3.A — IAM: OpenFGA v1.x (AuthZ Zanzibar)

**Prerequisito:** 4.3 (`openfga_db`), 4.6 (IAM Domain Service con stub `authz/openfga.go`).

**Migración gradual:** En Fases 1-2, el IAM Service usa roles simples del JWT (`role=doctor`). OpenFGA introduce relaciones granulares. La activación es aditiva con fallback.

#### docker-compose.yml: Servicio OpenFGA

```yaml
  openfga:
    image: openfga/openfga:latest
    container_name: serenamente-openfga
    restart: unless-stopped
    depends_on:
      postgres: { condition: service_healthy }
    command: run
    environment:
      OPENFGA_DATASTORE_ENGINE: postgres
      OPENFGA_DATASTORE_URI: postgres://openfga_user:${OPENFGA_DB_PASSWORD}@postgres:5432/openfga_db?sslmode=disable
      OPENFGA_HTTP_ADDR: 0.0.0.0:8080
      OPENFGA_GRPC_ADDR: 0.0.0.0:8081
      OPENFGA_LOG_FORMAT: json
      OPENFGA_LOG_LEVEL: info
    ports:
      - "8080:8080"    # HTTP API — solo red interna Docker
      - "8081:8081"    # gRPC API — solo red interna Docker
    networks: [serenamente_net]
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:8080/healthz | grep -q 'SERVING'"]
      interval: 10s
      timeout: 5s
      retries: 5
```

#### Authorization Model DSL v1.1

Verificar en [play.fga.dev](https://play.fga.dev) antes del deploy:

```
model
  schema 1.1

type user

type patient
  relations
    define user: [user]

type doctor
  relations
    define user: [user]

type organization
  relations
    define admin:  [user]
    define member: [user] or admin

type clinical_record
  relations
    define patient:   [patient]
    define can_read:  (user from patient) or can_write
    define can_write: [doctor]

type appointment
  relations
    define doctor:    [doctor]
    define patient:   [patient]
    define can_view:  (user from doctor) or (user from patient)
    define can_cancel: (user from patient) or (user from doctor)
```

#### Integración en IAM Domain Service

```go
// services/iam/internal/authz/openfga.go

// Check: ¿puede el médico escribir en esta historia clínica?
func (a *AuthzService) CanDoctorWriteRecord(ctx context.Context, doctorUserID, recordID string) (bool, error) {
    resp, err := a.fga.Check(ctx).Body(client.ClientCheckRequest{
        User:     fmt.Sprintf("user:%s", doctorUserID),
        Relation: "can_write",
        Object:   fmt.Sprintf("clinical_record:%s", recordID),
    }).Execute()
    if err != nil {
        // Fallback: si OpenFGA no responde en 50ms, usar el rol del JWT
        return a.fallbackRoleCheck(ctx, doctorUserID, "doctor"), nil
    }
    return *resp.Allowed, nil
}

// Asignar médico a historia clínica (crear tuple)
func (a *AuthzService) AssignDoctorToRecord(ctx context.Context, doctorID, recordID string) error {
    _, err := a.fga.WriteTuples(ctx).Body(client.ClientWriteRequest{
        Writes: []openfga.TupleKey{{
            User:     fmt.Sprintf("doctor:%s", doctorID),
            Relation: "can_write",
            Object:   fmt.Sprintf("clinical_record:%s", recordID),
        }},
    }).Execute()
    return err
}
```

SDK Go: `github.com/openfga/go-sdk` (Apache 2.0).

**Criterio de aceptación:**
- Médico no asignado a una HC → `403` al intentar escribir.
- `