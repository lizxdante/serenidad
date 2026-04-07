# EP-01 — Preparación y Cuentas Externas

**Épica:** Como equipo de infraestructura, necesitamos tener todas las cuentas externas creadas, tokens generados, tooling instalado y el monorepo preparado, para poder ejecutar las tareas de infraestructura sin bloqueos por dependencias de acceso.

**Origen:** Secciones §4 (Prerrequisitos y cuentas externas) y §5 (Tarea V-00) del documento de decisión.
**Prioridad:** Crítica — Bloqueante para todas las demás épicas.
**Sprint:** S1
**Dependencias Entrantes:** Ninguna (es el punto de partida).
**Dependencias Salientes:** EP-02, EP-03, EP-04, EP-05, EP-06, EP-07, EP-08, EP-09.

---

## HU-01.1 — Creación de Cuentas en Servicios Externos

**Como** ingeniero de plataforma,
**quiero** tener todas las cuentas de servicios externos creadas y configuradas con los tokens de API necesarios,
**para que** ninguna tarea de infraestructura posterior se bloquee por falta de acceso a un servicio.

### Tareas y Subtareas

#### T-01.1.1 — Crear y configurar cuenta de GitLab

- **ST-01.1.1.1** — Crear cuenta personal o de grupo en `gitlab.com`.
  - **CA:** La cuenta existe y es accesible via navegador.
- **ST-01.1.1.2** — Crear el monorepo `serenidad/serenidad-platform` (privado).
  - **CA:** El repositorio existe en GitLab y es accesible via `git clone`.
- **ST-01.1.1.3** — Verificar que GitLab CI/CD está habilitado con runners compartidos activos.
  - **CA:** En Settings → CI/CD → Runners se muestran runners compartidos disponibles.
- **ST-01.1.1.4** — Crear Personal Access Token (PAT) con scopes: `api`, `read_repository`, `write_repository`, `read_registry`, `write_registry`.
  - **CA:** El token se genera correctamente y se almacena en el password manager.
  - **CA:** El token permite operaciones `git push` y `docker push` al registry.
- **ST-01.1.1.5** — Crear Deploy Token con scopes: `read_repository`, `read_registry`.
  - **CA:** El deploy token se genera correctamente (username + token).
  - **CA:** El deploy token permite `git clone` y `docker pull` del registry.
  - **CA:** Username y token almacenados en el password manager.

#### T-01.1.2 — Crear y configurar cuenta de Hetzner Cloud

- **ST-01.1.2.1** — Crear cuenta en `console.hetzner.cloud`.
  - **CA:** La cuenta está activa y verificada.
- **ST-01.1.2.2** — Crear proyecto `serenidad-production`.
  - **CA:** El proyecto aparece en el dashboard de Hetzner Cloud.
- **ST-01.1.2.3** — Generar API Token del proyecto con permisos Read & Write.
  - **CA:** El token se genera y almacena en el password manager.
  - **CA:** `hcloud server list` responde sin error al usar el token.

#### T-01.1.3 — Crear y configurar cuenta de Cloudflare

- **ST-01.1.3.1** — Crear cuenta en `cloudflare.com`.
  - **CA:** La cuenta está activa y verificada.
- **ST-01.1.3.2** — Agregar el dominio `sereni.dad` a Cloudflare.
  - **CA:** El dominio está en estado `Active` en el dashboard de Cloudflare.
  - **CA:** Los nameservers del dominio apuntan a Cloudflare.
- **ST-01.1.3.3** — Obtener Zone ID del dominio.
  - **CA:** El Zone ID está documentado y almacenado en el password manager.
- **ST-01.1.3.4** — Crear API Token con permisos: "Edit zone DNS" + "Cloudflare Pages: Edit" + "Workers Scripts: Edit".
  - **CA:** El token se genera y almacena en el password manager.
  - **CA:** El token permite crear registros DNS via API.
- **ST-01.1.3.5** — Obtener Account ID de Cloudflare.
  - **CA:** El Account ID está documentado y almacenado.

#### T-01.1.4 — Crear y configurar cuenta de Backblaze B2

- **ST-01.1.4.1** — Crear cuenta en `backblaze.com`.
  - **CA:** La cuenta está activa y verificada.
- **ST-01.1.4.2** — Crear bucket `serenidad-pg-backups` (Private, región Europe: eu-central).
  - **CA:** El bucket existe y es de tipo Private.
  - **CA:** La región es geográficamente cercana al datacenter de Hetzner elegido.
- **ST-01.1.4.3** — Crear Application Key `cnpg-barman-key` con acceso Read/Write al bucket `serenidad-pg-backups`.
  - **CA:** `keyID` y `applicationKey` generados y almacenados en el password manager.
  - **CA:** La Application Key tiene acceso restringido al bucket único `serenidad-pg-backups`.

#### T-01.1.5 — Crear y configurar cuenta de Resend (SMTP)

- **ST-01.1.5.1** — Crear cuenta en `resend.com`.
  - **CA:** La cuenta está activa.
- **ST-01.1.5.2** — Añadir y verificar el dominio `sereni.dad` en Resend.
  - **CA:** El dominio está verificado (DNS records DKIM/SPF configurados).
- **ST-01.1.5.3** — Crear API Key en Resend.
  - **CA:** La API Key se genera y almacena en el password manager.
  - **CA:** Tier gratuito confirmado: 3,000 emails/mes, 100/día.

### Criterios de Aceptación de la HU-01.1

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | 5 cuentas externas creadas | Acceso verificado a GitLab, Hetzner, Cloudflare, Backblaze, Resend |
| 2 | Todos los tokens/API keys generados | Listado completo de tokens en password manager |
| 3 | Deploy Token de GitLab funcional | `git clone` y `docker pull` exitoso con el deploy token |
| 4 | Dominio activo en Cloudflare | `dig sereni.dad @1.1.1.1` resuelve correctamente |
| 5 | Bucket B2 creado y accesible | API call exitoso con la Application Key |
| 6 | Dominio verificado en Resend | Email de prueba enviado exitosamente desde `noreply@sereni.dad` |

### Definition of Done — HU-01.1

- [ ] Las 5 cuentas están creadas y activas.
- [ ] Todos los tokens y API keys están almacenados en el password manager.
- [ ] Ningún token está en texto plano en ningún archivo del repositorio.
- [ ] Cada token ha sido verificado con una operación básica (listar recursos, etc.).

---

## HU-01.2 — Instalación del Tooling en la Máquina de Trabajo

**Como** ingeniero de plataforma,
**quiero** tener todas las herramientas CLI instaladas y verificadas en mi máquina de trabajo,
**para que** pueda gestionar el cluster de Kubernetes, cifrar secretos, y desarrollar microservicios sin interrupciones.

### Tareas y Subtareas

#### T-01.2.1 — Instalar herramientas de gestión de cluster

- **ST-01.2.1.1** — Instalar `talosctl` (versión compatible con Talos v1.10.x).
  - **CA:** `talosctl version --client` → `v1.10.x`.
- **ST-01.2.1.2** — Instalar `kubectl` (versión compatible con Kubernetes 1.33.x).
  - **CA:** `kubectl version --client` → `v1.33.x`.
- **ST-01.2.1.3** — Instalar `helm` v3.x.
  - **CA:** `helm version` → `v3.x.x`.
- **ST-01.2.1.4** — Instalar `flux` CLI v2.x.
  - **CA:** `flux version` → `v2.x.x` (client only antes del bootstrap).
- **ST-01.2.1.5** — Instalar `hcloud` CLI y configurar contexto `serenidad-production`.
  - **CA:** `hcloud context use serenidad-production` → sin error.
  - **CA:** `hcloud server list` → responde (puede estar vacío).
- **ST-01.2.1.6** — Instalar `k9s` (TUI para exploración del cluster).
  - **CA:** `k9s version` → versión instalada.

#### T-01.2.2 — Instalar herramientas de cifrado y secretos

- **ST-01.2.2.1** — Instalar `sops`.
  - **CA:** `sops --version` → versión instalada.
- **ST-01.2.2.2** — Instalar `age` y `age-keygen`.
  - **CA:** `age --version` → versión instalada.
  - **CA:** `age-keygen --version` o `age-keygen` → funcional.

#### T-01.2.3 — Instalar herramientas de desarrollo

- **ST-01.2.3.1** — Instalar Go 1.25.x.
  - **CA:** `go version` → `go1.25.x`.
- **ST-01.2.3.2** — Instalar `buf` CLI (toolchain Protobuf).
  - **CA:** `buf --version` → versión instalada.
- **ST-01.2.3.3** — Instalar Node.js LTS y Bun 1.3.x.
  - **CA:** `node --version` → LTS.
  - **CA:** `bun --version` → `1.3.x`.
- **ST-01.2.3.4** — Instalar `wrangler` (Cloudflare Workers CLI).
  - **CA:** `wrangler --version` → versión instalada.
- **ST-01.2.3.5** — Instalar Docker (Docker Desktop o Engine).
  - **CA:** `docker version` → client y server funcionando.
- **ST-01.2.3.6** — Instalar `jq` (procesador JSON).
  - **CA:** `jq --version` → versión instalada.
- **ST-01.2.3.7** — Verificar `git` ≥ 2.30 (soporte signed commits).
  - **CA:** `git --version` → ≥ 2.30.

### Criterios de Aceptación de la HU-01.2

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | 15 herramientas instaladas | Cada una responde a `--version` sin error |
| 2 | Versiones compatibles | talosctl ↔ Talos v1.10.x, kubectl ↔ K8s 1.33.x |
| 3 | hcloud CLI autenticado | `hcloud server list` funciona sin pedir token |
| 4 | Docker funcional | `docker run hello-world` ejecuta correctamente |

### Definition of Done — HU-01.2

- [ ] Todas las herramientas instaladas y verificadas con `--version`.
- [ ] Compatibilidad de versiones confirmada (talosctl ↔ Talos, kubectl ↔ K8s).
- [ ] `hcloud` autenticado contra el proyecto `serenidad-production`.

---

## HU-01.3 — Configuración de Variables de Entorno de Trabajo

**Como** ingeniero de plataforma,
**quiero** tener un archivo `.envrc` centralizado con todas las variables de entorno necesarias,
**para que** cada sesión de trabajo tenga acceso consistente a tokens y configuraciones sin riesgo de exponer secretos.

### Tareas y Subtareas

#### T-01.3.1 — Crear archivo `.envrc` en la raíz del monorepo

- **ST-01.3.1.1** — Crear el archivo `.envrc` con todas las variables documentadas en §4.3.
  - **CA:** El archivo contiene las variables: `HCLOUD_TOKEN`, `HETZNER_PROJECT`, `VPS_IP`, `GITLAB_TOKEN`, `GITLAB_USER`, `GITLAB_REPO`, `GITLAB_DEPLOY_TOKEN_USER`, `GITLAB_DEPLOY_TOKEN`, `CF_API_TOKEN`, `CF_ZONE_ID`, `CF_ACCOUNT_ID`, `CF_DOMAIN`, `B2_KEY_ID`, `B2_APPLICATION_KEY`, `B2_BUCKET`, `B2_ENDPOINT`, `RESEND_API_KEY`, `KUBECONFIG`, `TALOSCONFIG`.
- **ST-01.3.1.2** — Verificar que `.envrc` está en `.gitignore`.
  - **CA:** `git status` no muestra `.envrc` como archivo no rastreado.
  - **CA:** `grep '.envrc' .gitignore` retorna coincidencia.
- **ST-01.3.1.3** — Verificar que `source .envrc` carga todas las variables.
  - **CA:** `echo $HCLOUD_TOKEN` retorna un valor no vacío después de `source .envrc`.
  - **CA:** Todas las variables exportadas están disponibles en subshells.

#### T-01.3.2 — (Opcional) Configurar `direnv` para auto-carga

- **ST-01.3.2.1** — Instalar `direnv` y configurar hook en el shell.
  - **CA:** Al entrar al directorio del monorepo, las variables se cargan automáticamente.

### Criterios de Aceptación de la HU-01.3

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | `.envrc` creado con todas las variables | `cat .envrc` muestra todas las variables de §4.3 |
| 2 | `.envrc` en `.gitignore` | `git check-ignore .envrc` → `.envrc` |
| 3 | Variables funcionales | `source .envrc && echo $HCLOUD_TOKEN` → valor no vacío |
| 4 | Sin secrets en el repositorio | `git log --all --oneline -- .envrc` → sin resultados |

### Definition of Done — HU-01.3

- [ ] Archivo `.envrc` creado con todas las variables requeridas.
- [ ] `.envrc` confirmado en `.gitignore`.
- [ ] Cada variable validada con una operación básica (echo, API call).

---

## HU-01.4 — Preparación del Monorepo y Bootstrap Inicial (V-00)

**Como** ingeniero de plataforma,
**quiero** tener el monorepo inicializado con la estructura de directorios completa, SSH key para Hetzner, y variables CI/CD configuradas en GitLab,
**para que** todas las tareas de infraestructura puedan ejecutarse sin bloqueos.

### Tareas y Subtareas

#### T-01.4.1 — Crear estructura de directorios del monorepo

- **ST-01.4.1.1** — Clonar el repositorio desde GitLab.
  - **CA:** `git clone` exitoso, directorio del monorepo creado.
- **ST-01.4.1.2** — Crear todos los directorios de la estructura definida en §4.4.
  - **CA:** Existen los directorios:
    - `infra/clusters/hetzner-prod/talos/patches`
    - `infra/clusters/hetzner-prod/flux-system`
    - `infra/infrastructure/cert-manager`
    - `infra/infrastructure/traefik`
    - `infra/infrastructure/cnpg`
    - `infra/infrastructure/nats` (Fase 2 — placeholder)
    - `infra/apps/iam-service`
    - `infra/apps/kratos`
    - `infra/apps/scheduling-service` (Fase 2 — placeholder)
    - `infra/apps/clinical-service` (Fase 2 — placeholder)
    - `infra/apps/billing-service` (Fase 3 — placeholder)
    - `infra/apps/openfga` (Fase 3 — placeholder)
    - `infra/secrets`
    - `services/iam`
    - `services/scheduling` (Fase 2 — placeholder)
    - `services/clinical` (Fase 2 — placeholder)
    - `services/billing` (Fase 3 — placeholder)
    - `packages/events/proto`
    - `apps/web`
    - `apps/bff`
    - `scripts`
- **ST-01.4.1.3** — Crear archivo `.gitignore` con las reglas definidas en §4.4.
  - **CA:** `.gitignore` contiene exclusiones para: `.envrc`, `*.talosconfig`, `secrets.yaml` de Talos, `kubeconfig`, `*.env`, `*.env.local`, `.env.*`, `node_modules/`, `dist/`, `.wrangler/`, `*.wasm`, `bin/`, `vendor/`, `.DS_Store`, `Thumbs.db`.
- **ST-01.4.1.4** — Commit inicial y push a `main`.
  - **CA:** `git log --oneline -1` → `chore: initial monorepo structure and gitignore`.
  - **CA:** El commit está en la rama `main` en GitLab.

#### T-01.4.2 — Configurar SSH Key para Hetzner (acceso rescue)

- **ST-01.4.2.1** — Generar SSH key Ed25519 dedicada para Hetzner.
  - **CA:** Archivo `~/.ssh/hetzner_serenidad_ed25519` y `.pub` existen.
  - **CA:** Key type es Ed25519 (verificar con `ssh-keygen -l -f`).
- **ST-01.4.2.2** — Agregar la SSH key pública al proyecto Hetzner via CLI.
  - **CA:** `hcloud ssh-key list` muestra `serenidad-admin-key` con fingerprint correcto.
- **ST-01.4.2.3** — Backup de la clave privada SSH en el password manager.
  - **CA:** Backup verificado en el password manager.

#### T-01.4.3 — Configurar variables CI/CD en GitLab

- **ST-01.4.3.1** — Instalar y autenticar GitLab CLI (`glab`).
  - **CA:** `glab auth status` → autenticado correctamente.
- **ST-01.4.3.2** — Agregar variable `HCLOUD_TOKEN` (masked).
  - **CA:** `glab variable list` muestra `HCLOUD_TOKEN` como masked.
- **ST-01.4.3.3** — Agregar variable `CF_API_TOKEN` (masked).
  - **CA:** `glab variable list` muestra `CF_API_TOKEN` como masked.
- **ST-01.4.3.4** — Agregar variable `CF_ACCOUNT_ID` (masked).
  - **CA:** `glab variable list` muestra `CF_ACCOUNT_ID` como masked.
- **ST-01.4.3.5** — Agregar variable `B2_KEY_ID` (masked).
  - **CA:** `glab variable list` muestra `B2_KEY_ID` como masked.
- **ST-01.4.3.6** — Agregar variable `B2_APPLICATION_KEY` (masked).
  - **CA:** `glab variable list` muestra `B2_APPLICATION_KEY` como masked.
- **ST-01.4.3.7** — Agregar variable `RESEND_API_KEY` (masked).
  - **CA:** `glab variable list` muestra `RESEND_API_KEY` como masked.
- **ST-01.4.3.8** — Verificar todas las variables via web UI o CLI.
  - **CA:** `glab variable list` muestra 6 variables masked.
  - **CA:** Verificar en Settings → CI/CD → Variables que todas aparecen como "Masked".

### Criterios de Aceptación de la HU-01.4

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Monorepo con estructura completa | `find infra services packages apps scripts -type d` → todos los directorios existen |
| 2 | `.gitignore` correcto | `cat .gitignore` → contiene todas las reglas de §4.4 |
| 3 | Commit inicial en `main` | `git log --oneline -1 origin/main` → commit de estructura |
| 4 | SSH key registrada en Hetzner | `hcloud ssh-key list` → `serenidad-admin-key` presente |
| 5 | 6 variables CI/CD en GitLab | `glab variable list` → 6 variables masked |
| 6 | SSH key respaldada | Backup verificado en password manager |

### Definition of Done — HU-01.4

- [ ] Estructura de directorios commiteada en `main`.
- [ ] SSH key generada, registrada en Hetzner, y respaldada.
- [ ] 6 variables CI/CD configuradas como masked en GitLab.
- [ ] Push exitoso a `main` desde la máquina de trabajo.

---

## Resumen de Dependencias Internas EP-01

```
T-01.1.1 (GitLab) ──► T-01.4.1 (Monorepo), T-01.4.3 (Variables CI/CD)
T-01.1.2 (Hetzner) ──► T-01.4.2 (SSH Key), T-01.3.1 (.envrc)
T-01.1.3 (Cloudflare) ──► T-01.3.1 (.envrc)
T-01.1.4 (Backblaze) ──► T-01.3.1 (.envrc)
T-01.1.5 (Resend) ──► T-01.3.1 (.envrc)
T-01.2.* (Tooling) ──► T-01.4.* (requiere herramientas instaladas)
T-01.3.1 (.envrc) ──► T-01.4.3 (Variables CI/CD usan mismos valores)
```
