# EP-08 — Frontend y BFF

**Épica:** Como equipo de desarrollo, necesitamos el BFF (Backend for Frontend) desplegado en Cloudflare Workers con Hono v4.x y verificación de JWT Ed25519, y la SPA Qwik v2.0 desplegada en Cloudflare Pages, para que los usuarios tengan una interfaz web performante que se comunique de forma segura con el backend a través del edge de Cloudflare.

**Origen:** Secciones §20 (V-15) y §21 (V-16) del documento de decisión.
**Prioridad:** Crítica — Componente de cara al usuario.
**Sprint:** S3
**Dependencias Entrantes:** EP-07 HU-07.2 (IAM Service para token exchange y validate-token), EP-07 HU-07.1 (Kratos para flujos de auth), EP-04 HU-04.3 (DNS para dominio público).
**Dependencias Salientes:** EP-09 (CI/CD automatiza deploys de BFF y SPA).

---

## HU-08.1 — BFF: Cloudflare Workers + Hono v4.x (V-15)

**Como** desarrollador frontend,
**quiero** tener un BFF desplegado en Cloudflare Workers que verifique JWT Ed25519, proxy requests al backend, y gestione CORS,
**para que** las llamadas del frontend al backend sean seguras, verificadas en el edge, y con latencia mínima global.

**Dependencias:** HU-07.2 (IAM Service para validate-token y token exchange), HU-07.1 (Kratos para auth proxy).

### Tareas y Subtareas

#### T-08.1.1 — Inicializar proyecto BFF con Hono

- **ST-08.1.1.1** — Navegar a `apps/bff/` y crear proyecto con plantilla Cloudflare Workers.
  - **CA:** `bun create hono@latest . --template cloudflare-workers` ejecuta sin error.
- **ST-08.1.1.2** — Instalar dependencias: `hono@^4.0.0` y `jose` (JWT verification).
  - **CA:** `bun add hono@^4.0.0 jose` → `package.json` actualizado.
  - **CA:** `bun install` completa sin errores.
- **ST-08.1.1.3** — Verificar estructura generada.
  - **CA:** Archivos presentes: `src/index.ts`, `wrangler.toml`, `tsconfig.json`, `package.json`.

#### T-08.1.2 — Configurar wrangler.toml

- **ST-08.1.2.1** — Editar `apps/bff/wrangler.toml`.
  - **CA:** `name = "serenidad-bff"`.
  - **CA:** `main = "src/index.ts"`.
  - **CA:** `compatibility_date = "2026-04-01"`.
  - **CA:** `compatibility_flags = ["nodejs_compat"]`.
  - **CA:** `[vars]`: `BACKEND_URL`, `ENVIRONMENT`, `JWT_ISSUER`, `JWT_AUDIENCE`.
  - **CA:** Route: `pattern = "api.sereni.dad/bff/*"`, `zone_name = "sereni.dad"`.
  - **CA:** Nota: `JWT_PUBLIC_KEY_ED25519` se agrega como secret (NO en wrangler.toml).

#### T-08.1.3 — Implementar middleware de autenticación JWT

- **ST-08.1.3.1** — Crear `apps/bff/src/middleware/auth.ts`.
  - **CA:** Función `extractBearerToken(header: string)`: extrae token del header `Authorization: Bearer <token>`.
  - **CA:** Función `verifyJWT(token, publicKey, options)`: verifica JWT Ed25519 con librería `jose`.
  - **CA:** Options: `issuer`, `audience`.
  - **CA:** Retorna payload del JWT o lanza error.
- **ST-08.1.3.2** — Crear `apps/bff/src/middleware/cors.ts` (si no se usa el built-in de Hono).
  - **CA:** CORS configurado: origin `https://app.sereni.dad`, methods, headers, credentials.

#### T-08.1.4 — Implementar router principal del BFF

- **ST-08.1.4.1** — Implementar `apps/bff/src/index.ts`.
  - **CA:** Tipos Bindings: `BACKEND_URL`, `ENVIRONMENT`, `JWT_PUBLIC_KEY_ED25519`, `JWT_ISSUER`, `JWT_AUDIENCE`.
  - **CA:** Middlewares globales: `secureHeaders()`, `cors()`.
  - **CA:** `GET /health` → `{"status":"ok","service":"bff"}` (sin auth).
  - **CA:** `ALL /auth/*` → proxy directo a Kratos (sin JWT, maneja sesión).
  - **CA:** `POST /api/iam/token/exchange` → proxy directo (sin JWT, es el endpoint que genera el JWT).
  - **CA:** `USE /api/*` → middleware JWT (verifica token, inyecta claims en context).
  - **CA:** `ALL /api/*` → proxy al backend con headers `X-User-ID`, `X-User-Role`, `X-Tenant-ID`.
  - **CA:** Error handler global: HTTPException → JSON error, otros → 500.
- **ST-08.1.4.2** — Verificar TypeScript compila sin errores.
  - **CA:** `bun run tsc --noEmit` → sin errores.

#### T-08.1.5 — Configurar secret de JWT public key en CF Workers

- **ST-08.1.5.1** — Ejecutar `wrangler secret put JWT_PUBLIC_KEY_ED25519`.
  - **CA:** Secret configurado en Cloudflare Workers (no en wrangler.toml).
  - **CA:** La clave pública Ed25519 en PEM se ingresa como valor.

#### T-08.1.6 — Deploy del BFF a Cloudflare Workers

- **ST-08.1.6.1** — Build de producción.
  - **CA:** `bun run build` o `wrangler deploy --dry-run` completa sin error.
- **ST-08.1.6.2** — Deploy a producción.
  - **CA:** `wrangler deploy` → deployed exitosamente.
  - **CA:** Output muestra URL y tamaño del bundle.
- **ST-08.1.6.3** — Verificar deploy.
  - **CA:** `curl https://api.sereni.dad/bff/health` → `{"status":"ok","service":"bff"}`.

#### T-08.1.7 — Verificar funcionalidad del BFF

- **ST-08.1.7.1** — Verificar health endpoint.
  - **CA:** `curl https://api.sereni.dad/bff/health` → 200 OK.
- **ST-08.1.7.2** — Verificar proxy de auth (Kratos).
  - **CA:** `curl https://api.sereni.dad/auth/.well-known/ory/webauthn.js` → 200 (proxied a Kratos).
- **ST-08.1.7.3** — Verificar rechazo de request sin JWT a endpoint protegido.
  - **CA:** `curl https://api.sereni.dad/bff/api/iam/whoami` → 401 `{"error":"Authorization header required"}`.
- **ST-08.1.7.4** — Verificar request con JWT válido (test end-to-end).
  - **CA:** Request con `Authorization: Bearer <jwt_valido>` → 200 con datos del usuario.
  - **CA:** Headers `X-User-ID`, `X-User-Role`, `X-Tenant-ID` propagados al backend.
- **ST-08.1.7.5** — Verificar CORS.
  - **CA:** Preflight request `OPTIONS` desde `https://app.sereni.dad` → `Access-Control-Allow-Origin: https://app.sereni.dad`.
  - **CA:** Request desde otro origen → sin header CORS.

### Criterios de Aceptación de la HU-08.1

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-1 | BFF desplegado en CF Workers | `curl .../bff/health` → `{"status":"ok","service":"bff"}` |
| CA-2 | JWT verification funcional | Request sin JWT → 401, con JWT válido → 200 |
| CA-3 | Proxy auth funcional | `/auth/*` proxied a Kratos correctamente |
| CA-4 | Token exchange sin JWT | `POST /api/iam/token/exchange` accesible sin JWT |
| CA-5 | Headers propagados | `X-User-ID`, `X-User-Role`, `X-Tenant-ID` en requests al backend |
| CA-6 | CORS correcto | Solo `app.sereni.dad` permitido |
| CA-7 | TypeScript compila | `tsc --noEmit` → sin errores |
| CA-8 | Secret JWT configurado | `JWT_PUBLIC_KEY_ED25519` como secret en CF Workers |

### Definition of Done — HU-08.1

- [ ] BFF implementado con Hono v4.x en TypeScript.
- [ ] JWT Ed25519 verification funcional en el edge.
- [ ] Proxy de auth y API funcional.
- [ ] CORS configurado para `app.sereni.dad`.
- [ ] Desplegado en Cloudflare Workers.
- [ ] Health, auth proxy, y JWT verification verificados.
- [ ] Secret de JWT public key configurado.

---

## HU-08.2 — Qwik v2.0 SPA en Cloudflare Pages (V-16)

**Como** usuario de serenidad,
**quiero** acceder a una aplicación web rápida y responsive en `app.sereni.dad` que permita registrarse con Passkey, iniciar sesión, y ver mi dashboard,
**para que** pueda interactuar con la plataforma de salud mental de forma segura y fluida desde cualquier dispositivo.

**Dependencias:** HU-08.1 (BFF para llamadas al backend), HU-07.1 (Kratos para flujos de auth).

### Tareas y Subtareas

#### T-08.2.1 — Inicializar proyecto Qwik v2.0

- **ST-08.2.1.1** — Navegar a `apps/web/` y crear proyecto con plantilla Cloudflare Pages.
  - **CA:** `bun create qwik@latest . --template cloudflare-pages` ejecuta sin error.
- **ST-08.2.1.2** — Instalar TypeScript 6.0.
  - **CA:** `bun add -D typescript@^6.0.0`.
  - **CA:** `package.json` muestra `typescript: ^6.0.0`.
- **ST-08.2.1.3** — Verificar que usa `@qwik.dev/qwik` (no `@builder.io/qwik`).
  - **CA:** `grep "@qwik.dev" package.json` → `@qwik.dev/qwik: ^2.0.0`, `@qwik.dev/router: ^2.0.0`.
- **ST-08.2.1.4** — Verificar build local.
  - **CA:** `bun run build` → genera directorio `dist/` sin errores.

#### T-08.2.2 — Configurar Cloudflare Pages

- **ST-08.2.2.1** — Crear proyecto en CF Pages via CLI.
  - **CA:** `wrangler pages project create serenidad-web --production-branch main` → proyecto creado.
- **ST-08.2.2.2** — Configurar variable de entorno `VITE_API_BASE_URL`.
  - **CA:** `wrangler pages secret put VITE_API_BASE_URL --project-name serenidad-web` → valor `https://api.sereni.dad/bff`.
- **ST-08.2.2.3** — Configurar variable de entorno `VITE_KRATOS_URL`.
  - **CA:** `wrangler pages secret put VITE_KRATOS_URL --project-name serenidad-web` → valor `https://api.sereni.dad/auth`.

#### T-08.2.3 — Implementar páginas base de la SPA

- **ST-08.2.3.1** — Implementar página de inicio (`/`).
  - **CA:** Landing page con navegación a login y registro.
  - **CA:** Diseño responsive y accesible.
- **ST-08.2.3.2** — Implementar página de registro (`/register`).
  - **CA:** Integración con Kratos registration flow.
  - **CA:** UI para registro de Passkey (WebAuthn).
  - **CA:** Manejo de errores de registro.
- **ST-08.2.3.3** — Implementar página de login (`/login`).
  - **CA:** Integración con Kratos login flow.
  - **CA:** UI para autenticación con Passkey.
  - **CA:** Manejo de errores de login.
- **ST-08.2.3.4** — Implementar página de verificación (`/verification`).
  - **CA:** Input para código de 6 dígitos recibido por email.
  - **CA:** Integración con Kratos verification flow.
- **ST-08.2.3.5** — Implementar página de recovery (`/recovery`).
  - **CA:** Input para código de recovery via email.
  - **CA:** Integración con Kratos recovery flow.
- **ST-08.2.3.6** — Implementar página de settings (`/settings`).
  - **CA:** Gestión de Passkeys del usuario.
  - **CA:** Integración con Kratos settings flow.
- **ST-08.2.3.7** — Implementar página de dashboard (`/dashboard`).
  - **CA:** Muestra información básica del usuario (nombre, rol).
  - **CA:** Requiere JWT válido para cargar datos.
  - **CA:** Llama a `GET /bff/api/iam/whoami` con JWT en header.

#### T-08.2.4 — Implementar lógica de autenticación en el frontend

- **ST-08.2.4.1** — Implementar servicio de auth (token exchange).
  - **CA:** Tras login exitoso en Kratos, llama a `POST /bff/api/iam/token/exchange` con session token.
  - **CA:** Almacena JWT recibido (sessionStorage o cookie httpOnly).
- **ST-08.2.4.2** — Implementar guard de rutas protegidas.
  - **CA:** Rutas `/dashboard`, `/settings` requieren JWT válido.
  - **CA:** Redirige a `/login` si no hay JWT o está expirado.
- **ST-08.2.4.3** — Implementar interceptor de requests HTTP.
  - **CA:** Agrega `Authorization: Bearer <jwt>` a todas las requests al BFF.
  - **CA:** Maneja 401 redirigiendo a `/login`.
- **ST-08.2.4.4** — Implementar logout.
  - **CA:** Llama a Kratos logout flow.
  - **CA:** Elimina JWT almacenado.
  - **CA:** Redirige a `/`.

#### T-08.2.5 — Deploy manual a Cloudflare Pages

- **ST-08.2.5.1** — Build de producción.
  - **CA:** `bun run build` → directorio `dist/` generado sin errores.
- **ST-08.2.5.2** — Deploy a CF Pages.
  - **CA:** `wrangler pages deploy dist/ --project-name serenidad-web --branch main` → deploy exitoso.
- **ST-08.2.5.3** — Verificar deploy.
  - **CA:** `curl https://app.sereni.dad` → HTML de la SPA, status 200.

#### T-08.2.6 — Verificar rendimiento y funcionalidad

- **ST-08.2.6.1** — Verificar accesibilidad de la SPA.
  - **CA:** `curl https://app.sereni.dad` → 200, HTML de Qwik.
- **ST-08.2.6.2** — Medir rendimiento con Lighthouse.
  - **CA:** `npx lighthouse https://app.sereni.dad --only-categories=performance` → Score ≥ 95.
  - **CA:** TTI < 100ms (Qwik resumability).
- **ST-08.2.6.3** — Verificar responsive design.
  - **CA:** SPA funcional en móvil, tablet, y desktop.
- **ST-08.2.6.4** — Verificar flujo completo de registro.
  - **CA:** Usuario puede registrar Passkey → recibe email → verifica → login → dashboard.

### Criterios de Aceptación de la HU-08.2

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-1 | SPA accesible en CF Pages | `curl https://app.sereni.dad` → 200 HTML |
| CA-2 | TTI < 100ms | Lighthouse Performance ≥ 95 |
| CA-3 | Registro con Passkey funcional | Flujo completo desde `/register` hasta Passkey registrada |
| CA-4 | Login con Passkey funcional | Flujo desde `/login` hasta JWT obtenido |
| CA-5 | Token exchange funcional | Session token → JWT en sessionStorage/cookie |
| CA-6 | Dashboard con datos del usuario | `/dashboard` muestra nombre y rol desde IAM API |
| CA-7 | Rutas protegidas | `/dashboard` redirige a `/login` sin JWT |
| CA-8 | Logout funcional | Elimina sesión Kratos y JWT, redirige a `/` |
| CA-9 | Responsive | Funcional en móvil, tablet, desktop |
| CA-10 | Qwik v2.0 con @qwik.dev | No usa @builder.io/qwik legacy |

### Definition of Done — HU-08.2

- [ ] SPA Qwik v2.0 creada con TypeScript 6.0.
- [ ] 7 páginas implementadas: /, /register, /login, /verification, /recovery, /settings, /dashboard.
- [ ] Integración con Kratos flows (registro, login, verificación, recovery, settings, logout).
- [ ] Token exchange y almacenamiento de JWT.
- [ ] Guard de rutas protegidas.
- [ ] Desplegada en Cloudflare Pages.
- [ ] Lighthouse Performance ≥ 95, TTI < 100ms.
- [ ] Responsive design verificado.

---

## Resumen de Dependencias Internas EP-08

```
HU-08.1 (BFF Hono CF Workers)
  Requiere: EP-07 HU-07.1 (Kratos), EP-07 HU-07.2 (IAM Service), EP-04 HU-04.3 (DNS)

HU-08.2 (Qwik SPA CF Pages)
  Requiere: HU-08.1 (BFF), EP-07 HU-07.1 (Kratos flows)

Desarrollo paralelo parcial:
  T-08.1.1 a T-08.1.4 (código BFF) puede desarrollarse en paralelo con EP-07.
  T-08.2.1 a T-08.2.3 (scaffold SPA + páginas) puede desarrollarse en paralelo con HU-08.1.
  T-08.1.5+ (deploy BFF) requiere EP-07 completada.
  T-08.2.4+ (integración auth) requiere HU-08.1 desplegado.
```
