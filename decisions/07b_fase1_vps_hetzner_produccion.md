# Fase 1 — Entorno de Producción Directo: VPS Hetzner CX32 con Talos Linux

## Guía Exhaustiva de Implementación — Ruta Directa a Producción

**Proyecto:** Serenamente — Clínica Digital de Salud Mental Global
**Versión:** 1.0 — Abril 2026
**Autor:** djca
**Documento alternativo a:** [`plans/07_fase1_laptop_talos_desarrollo.md`](./07_fase1_laptop_talos_desarrollo.md) — variante laptop local
**Relación con otros documentos:**
- [`plans/05_orden_implementacion_capas_exhaustivo.md`](./05_orden_implementacion_capas_exhaustivo.md) — Plan Fase 1 canónico
- [`plans/05b_orden_implementacion_capas_detalle.md`](./05b_orden_implementacion_capas_detalle.md) — Subcapas 1.E–3.D con criterios de aceptación
- [`plans/06_talos_k8s_decision_y_cambios_en_cadena.md`](./06_talos_k8s_decision_y_cambios_en_cadena.md) — ADR-010–014: Talos+k8s, cert-manager, Traefik, CloudNativePG, FluxCD
- [`target_arch/comp_infra_operaciones.d2`](../target_arch/comp_infra_operaciones.d2) — Diagrama de topología Kubernetes

---

## Índice

1. [¿Por qué ir directamente al VPS?](#1-por-qué-ir-directamente-al-vps)
2. [Arquitectura del entorno de producción](#2-arquitectura-del-entorno-de-producción)
3. [Presupuesto de recursos y costos](#3-presupuesto-de-recursos-y-costos)
4. [Prerrequisitos y cuentas externas](#4-prerrequisitos-y-cuentas-externas)
5. [Tarea V-00 — Preparación del monorepo](#5-tarea-v-00--preparación-del-monorepo)
6. [Tarea V-01 — Aprovisionamiento del VPS Hetzner](#6-tarea-v-01--aprovisionamiento-del-vps-hetzner)
7. [Tarea V-02 — Instalación de Talos Linux en Hetzner CX32](#7-tarea-v-02--instalación-de-talos-linux-en-hetzner-cx32)
8. [Tarea V-03 — Bootstrap de Talos + Kubernetes 1.33.x](#8-tarea-v-03--bootstrap-de-talos--kubernetes-133x)
9. [Tarea V-04 — Hetzner Cloud Controller Manager (CCM)](#9-tarea-v-04--hetzner-cloud-controller-manager-ccm)
10. [Tarea V-05 — FluxCD v2.x (GitOps desde el primer día)](#10-tarea-v-05--fluxcd-v2x-gitops-desde-el-primer-día)
11. [Tarea V-06 — Estructura GitOps del repositorio](#11-tarea-v-06--estructura-gitops-del-repositorio)
12. [Tarea V-07 — cert-manager v1.x + Let's Encrypt](#12-tarea-v-07--cert-manager-v1x--lets-encrypt)
13. [Tarea V-08 — Traefik v3.x (Ingress Controller)](#13-tarea-v-08--traefik-v3x-ingress-controller)
14. [Tarea V-09 — SOPS + age (cifrado de secretos GitOps)](#14-tarea-v-09--sops--age-cifrado-de-secretos-gitops)
15. [Tarea V-10 — CloudNativePG v1.x + PostgreSQL 17.4](#15-tarea-v-10--cloudnativepg-v1x--postgresql-174)
16. [Tarea V-11 — DNS y Cloudflare (dominios públicos)](#16-tarea-v-11--dns-y-cloudflare-dominios-públicos)
17. [Tarea V-12 — Schemas Protobuf + buf.build CLI](#17-tarea-v-12--schemas-protobuf--bufbuild-cli)
18. [Tarea V-13 — Ory Kratos v1.3.1 (AuthN)](#18-tarea-v-13--ory-kratos-v131-authn)
19. [Tarea V-14 — IAM Domain Service Go](#19-tarea-v-14--iam-domain-service-go)
20. [Tarea V-15 — BFF: Cloudflare Workers + Hono v4.x](#20-tarea-v-15--bff-cloudflare-workers--hono-v4x)
21. [Tarea V-16 — Qwik v2.0 SPA en Cloudflare Pages](#21-tarea-v-16--qwik-v20-spa-en-cloudflare-pages)
22. [Tarea V-17 — CloudNativePG ScheduledBackup → Backblaze B2](#22-tarea-v-17--cloudnativepg-scheduledbackup--backblaze-b2)
23. [Tarea V-18 — GitLab CI/CD](#23-tarea-v-18--gitlab-cicd)
24. [Tarea V-19 — Firewall y hardening de red](#24-tarea-v-19--firewall-y-hardening-de-red)
25. [Flujo de trabajo diario en producción directa](#25-flujo-de-trabajo-diario-en-producción-directa)
26. [Observabilidad mínima viable (Fase 1)](#26-observabilidad-mínima-viable-fase-1)
27. [Troubleshooting exhaustivo](#27-troubleshooting-exhaustivo)
28. [Criterios de aceptación global de la Fase 1](#28-criterios-de-aceptación-global-de-la-fase-1)

---

## 1. ¿Por qué ir directamente al VPS?

### 1.1 El argumento central: eliminar la migración como clase de riesgo

El plan alternativo de desarrollo en laptop (documento `07_fase1_laptop_talos_desarrollo.md`) es válido cuando se dispone de hardware dedicado y un segundo equipo para trabajar. Sin embargo, la ruta directa al VPS elimina de raíz una categoría entera de riesgo: **la divergencia de entornos entre desarrollo y producción**.

Cuando el desarrollo ocurre en hardware local y producción en un VPS, siempre existen diferencias que introducen fricciones:

- El `talosconfig` del laptop referencia una IP privada; producción tiene una IP pública.
- Los secretos cifrados con SOPS+age usan la misma clave `age` independientemente del cluster — no hay que re-cifrar al cambiar de entorno.
- `ClusterIssuer` usa CA auto-firmada en local y Let's Encrypt en producción — los certificados tienen comportamientos diferentes.
- El `Hetzner CCM` no existe en laptop; en producción gestiona el ciclo de vida de nodos y load balancers.
- Las latencias de red son radicalmente distintas: LAN vs internet.
- Las variables de entorno de los Workers de Cloudflare apuntan a `localhost` en dev y a `api.serenamente.com` en producción.

Cada una de estas diferencias es un vector de error que no se descubre hasta el momento de la migración. Con una inversión inicial de aproximadamente €6.80/mes (VPS CX32), este vector desaparece por completo. El código que funciona en el VPS es el código de producción.

### 1.2 Casos donde esta ruta es la correcta

Esta guía es la opción preferida cuando:

- **No hay hardware dedicado disponible** para ser el "laptop Talos" de desarrollo permanente.
- **El desarrollador trabaja solo o en equipo pequeño** y no existe diferenciación entre entorno de desarrollo y de producción en las primeras fases.
- **R1 (Build It Right The First Time)** tiene máxima prioridad y no se quiere introducir una deuda de migración, por pequeña que sea.
- **Se quiere aprender Hetzner + Talos + Kubernetes en contexto real** desde el primer commit.
- **El costo de €6.80/mes es aceptable** durante el desarrollo (inferior al costo de tiempo de cualquier migración posterior).

### 1.3 Implicaciones operacionales

Trabajar directamente en el VPS tiene implicaciones que deben entenderse y aceptarse conscientemente:

| Aspecto | Laptop local | VPS directo |
|---------|-------------|-------------|
| **Latencia de deploy** | Segundos (LAN) | 30-90 seg (build + push a registry.gitlab.com + pull en cluster) |
| **Hot-reload de código** | No nativo (build + push) | No nativo (misma razón) |
| **Costo siempre activo** | €0 (hardware propio) | ~€6.80/mes |
| **Acceso desde cualquier lugar** | Solo en la red local | Sí, internet |
| **Datos reales vs fixtures** | Fixtures | Datos reales desde el inicio |
| **Interrupción por update de Talos** | Solo afecta al dev | Afecta al servicio si hay usuarios |
| **Certificados TLS** | Self-signed (instalar CA en browser) | Let's Encrypt (confianza global) |
| **Flujo CI/CD** | Opcional (aplicar manifests directos) | Obligatorio desde el primer día (FluxCD + GitLab CI) |

> **Decisión arquitectónica clave:** Esta ruta acepta el overhead de ciclos de deploy más largos a cambio de cero deuda de migración y paridad perfecta entre el entorno de desarrollo y el entorno que verán los usuarios. Para desarrollo de lógica de negocio pura en Go, el código se puede testar localmente con `go test` sin necesidad de desplegar; solo los cambios de infraestructura o de contrato de API requieren un ciclo completo de deploy.

### 1.4 Patrón de trabajo recomendado

Para minimizar la latencia del ciclo de desarrollo en el VPS directo se recomienda el siguiente patrón:

```
Ciclo de desarrollo Go (lógica de negocio):
  editar código → go test ./... (local, milisegundos) → commit

Ciclo de integración (deploy al VPS):
  commit → git push → GitLab CI build+push → FluxCD detecta → deploy

Ciclo de emergencia (patch rápido sin CI/CD):
  docker build → docker push registry.gitlab.com → kubectl rollout restart deployment/iam-service -n serenamente-core
```

---

## 2. Arquitectura del entorno de producción

### 2.1 Topología completa de Fase 1

```
INTERNET
    │
    ▼
╔══════════════════════════════════════════════════════════════════╗
║ CLOUDFLARE EDGE — $0/mes                                         ║
║                                                                  ║
║  ┌──────────────────────────┐  ┌────────────────────────────────┐║
║  │ CF Pages                 │  │ CF Workers                     │║
║  │ Qwik v2.0 SPA            │  │ Hono v4.x BFF                  │║
║  │ @qwik.dev/qwik TS 6.0    │  │ JWT verify Ed25519 TS 6.0      │║
║  │ app.serenamente.com      │  │ 100K req/día gratis            │║
║  │ TTI ~50ms (resumable)    │  │ fetch() → api.serenamente.com  │║
║  └──────────────────────────┘  └────────────────────────────────┘║
╚══════════════════════════════════════════════════════════════════╝
    │
    │ HTTPS api.serenamente.com :443 TLS 1.3
    │ (Cloudflare Proxy activo — DDoS protection gratis)
    │
    ▼
╔══════════════════════════════════════════════════════════════════╗
║ HETZNER CX32 — ~€6.80/mes                                        ║
║ 4 vCPU AMD EPYC | 8 GB RAM | 80 GB NVMe SSD                     ║
║ Talos Linux v1.10.x | Kubernetes 1.33.x                          ║
║ IP pública: <HETZNER_IP> (Floating IP recomendado)               ║
║                                                                  ║
║ ┌─────────────────────────────────────────────────────────────┐  ║
║ │ ns: flux-system                                             │  ║
║ │   FluxCD v2.x — GitOps controller                          │  ║
║ │   Sincroniza: gitlab.com/serenamente/serenidad-platform      │  ║
║ │   Branch: main — Reconcilia cada 1 minuto                  │  ║
║ └─────────────────────────────────────────────────────────────┘  ║
║                                                                  ║
║ ┌─────────────────────────────────────────────────────────────┐  ║
║ │ ns: cert-manager                                            │  ║
║ │   cert-manager v1.x — ClusterIssuer: letsencrypt-prod      │  ║
║ │   Renovación automática — ACME HTTP-01                     │  ║
║ └─────────────────────────────────────────────────────────────┘  ║
║                                                                  ║
║ ┌─────────────────────────────────────────────────────────────┐  ║
║ │ ns: traefik                                                 │  ║
║ │   Traefik v3.x — DaemonSet hostPort :80/:443               │  ║
║ │   ForwardAuth Middleware → IAM /internal/validate-token     │  ║
║ │   IngressRoutes por namespace                               │  ║
║ └─────────────────────────────────────────────────────────────┘  ║
║                                                                  ║
║ ┌─────────────────────────────────────────────────────────────┐  ║
║ │ Secrets: SOPS + age — cifrado client-side                   │  ║
║ │   Clave age en Secret kubectl + backup en password manager  │  ║
║ │   FluxCD descifra: spec.decryption.provider: sops           │  ║
║ └─────────────────────────────────────────────────────────────┘  ║
║                                                                  ║
║ ┌─────────────────────────────────────────────────────────────┐  ║
║ │ ns: serenamente-data                                        │  ║
║ │   CloudNativePG v1.x — Cluster CRD — PostgreSQL 17.4       │  ║
║ │   6 databases: iam_db, scheduling_db, clinical_db,         │  ║
║ │                billing_db, kratos_db, openfga_db           │  ║
║ │   WAL archiving continuo → Backblaze B2                    │  ║
║ └─────────────────────────────────────────────────────────────┘  ║
║                                                                  ║
║ ┌─────────────────────────────────────────────────────────────┐  ║
║ │ ns: serenamente-core                                        │  ║
║ │   Ory Kratos v1.3.1 — AuthN Passkeys/WebAuthn              │  ║
║ │   IAM Domain Service Go — JWT Ed25519 + Outbox             │  ║
║ └─────────────────────────────────────────────────────────────┘  ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
    │
    ▼
╔══════════════════════════════════════════════════════════════════╗
║ BACKBLAZE B2 — ~$0.05/mes                                        ║
║   WAL continuo (barman-cloud) + daily base backup               ║
║   Retención: 30 días base, 90 días WAL                          ║
║   PITR disponible a cualquier punto en el tiempo               ║
╚══════════════════════════════════════════════════════════════════╝
```

### 2.2 Inventario de URLs públicas (Fase 1)

| URL | Servicio | Namespace | Ruta Traefik |
|-----|----------|-----------|--------------|
| `https://app.serenamente.com` | Qwik v2.0 SPA | CF Pages (externo) | — |
| `https://api.serenamente.com/auth/*` | Ory Kratos (public :4433) | serenamente-core | /auth/* |
| `https://api.serenamente.com/api/iam/*` | IAM Domain Service | serenamente-core | /api/iam/* |
| `https://api.serenamente.com/api/scheduling/*` | Scheduling (Fase 2) | serenamente-core | /api/scheduling/* |
| `https://api.serenamente.com/api/clinical/*` | Clinical Record (Fase 2) | serenamente-core | /api/clinical/* |
| `https://api.serenamente.com/api/billing/*` | Billing (Fase 3) | serenamente-core | /api/billing/* |

> **Nota sobre subdominios vs rutas:** Se usa un único dominio `api.serenamente.com` con rutas de path prefix en lugar de subdominios por microservicio. Esto simplifica la gestión de certificados (un solo certificado wildcard o SAN) y el routing en Traefik.

---

## 3. Presupuesto de recursos y costos

### 3.1 Recursos del CX32 vs demanda de Fase 1

| Componente | RAM estimada | CPU (idle) | CPU (peak) | Disco |
|-----------|-------------|-----------|-----------|-------|
| Talos OS + containerd | ~150 MB | <1% | <5% | ~500 MB |
| Kubernetes control plane (etcd + apiserver + cm + scheduler) | ~850 MB-1.1 GB | 2-5% | 15-30% | ~2 GB |
| FluxCD v2.x (4 controllers) | ~200 MB | <1% | <3% | ~100 MB |
| cert-manager | ~80 MB | <1% | <2% | ~50 MB |
| Traefik v3.x | ~80 MB | <1% | 5-10% | ~50 MB |
| SOPS (en kustomize-controller) | 0 MB adicional | 0% | 0% | 0 MB |
| CloudNativePG operator | ~100 MB | <1% | <2% | ~100 MB |
| PostgreSQL 17.4 (6 DBs vacías) | ~300-400 MB | 2-5% | 20-40% | ~2-5 GB |
| Ory Kratos v1.3.1 | ~150 MB | <1% | 5-10% | ~100 MB |
| IAM Domain Service Go | ~40-80 MB | <1% | 5-15% | ~50 MB |
| **TOTAL Fase 1** | **~2.0-2.4 GB** | **~10%** | **~50-60%** | **~5-8 GB** |
| **Disponible (8 GB total)** | **~5.6-6 GB libres** | — | — | **~72 GB libres** |

> La Fase 1 completa consume aproximadamente el 30% de la RAM del CX32. Hay holgura cómoda para agregar los servicios de Fase 2 (NATS: ~50 MB, Scheduling: ~60 MB, Clinical: ~60 MB) sin cambiar el plan de hardware.

### 3.2 Costo mensual consolidado

| Recurso | Costo/mes | Notas |
|---------|----------|-------|
| Hetzner CX32 (4 vCPU, 8 GB RAM) | ~€6.80 | Facturación por hora. Precio en EUR en Hetzner EU |
| Cloudflare Pages | $0 | Tier gratuito: 500 builds/mes, bandwidth ilimitado |
| Cloudflare Workers | $0 | Tier gratuito: 100K requests/día |
| Backblaze B2 | ~$0.01-0.05 | Almacenamiento WAL (~100-500 MB) + egress mínimo |
| GitLab.com (monorepo + CI/CD) | $0 | Tier gratuito: 400 min CI/mes con runners compartidos (suficiente para Fase 1); runners auto-escalados |  
| GitLab Container Registry | $0 | Incluido en GitLab.com gratuito; 10 GB storage por proyecto; `registry.gitlab.com` |
| Dominio `serenamente.com` | ~$10-15/año | Cloudflare Registrar (precio de costo, sin markup) |
| Let's Encrypt certificados | $0 | Open Source, siempre gratuito |
| **TOTAL FASE 1** | **~€7-8/mes** | Más dominio ($1.20/mes amortizado) |

---

## 4. Prerrequisitos y cuentas externas

### 4.1 Cuentas a crear (en este orden)

Antes de ejecutar cualquier tarea técnica, se necesitan las siguientes cuentas:

**4.1.1 GitLab**
- Cuenta personal o de grupo: `gitlab.com`
- Crear el monorepo: `serenamente/serenidad-platform` (privado o público según preferencia)
- GitLab CI/CD viene habilitado por defecto; verificar en Settings → CI/CD → Runners que los runners compartidos están activos
- Crear un Personal Access Token (PAT) con scopes: `api`, `read_repository`, `write_repository`, `read_registry`, `write_registry`
  - Avatar → Edit Profile → Access Tokens → Add new token
  - Guardar el token — solo se muestra una vez
- Crear un Deploy Token para que FluxCD pueda leer el repositorio y el registry:
  - Settings → Repository → Deploy tokens → Add token
  - Scopes: `read_repository`, `read_registry`
  - Guardar el nombre de usuario y el token del deploy token

**4.1.2 Hetzner Cloud**
- Crear cuenta en `https://console.hetzner.cloud`
- Crear un proyecto: `serenamente-production`
- Generar API Token del proyecto: Project → Security → API Tokens → Generate API Token (Read & Write)
- Guardar el token de API de Hetzner

**4.1.3 Cloudflare**
- Crear cuenta en `cloudflare.com`
- Agregar el dominio `serenamente.com` (o el dominio elegido)
  - Opción A: Transferir dominio a Cloudflare Registrar (recomendado — precio de costo)
  - Opción B: Cambiar nameservers del dominio actual a los de Cloudflare
- Verificar que el dominio está en estado `Active` en Cloudflare
- Obtener Zone ID del dominio: Dashboard → dominio → Overview → Zone ID (lateral derecho)
- Crear API Token de Cloudflare: Profile → API Tokens → Create Token
  - Template: "Edit zone DNS" + añadir permiso "Cloudflare Pages: Edit" + "Workers Scripts: Edit"
  - Guardar el token de API de Cloudflare

**4.1.4 Backblaze B2**
- Crear cuenta en `backblaze.com`
- Crear un bucket: `serenamente-pg-backups`
  - Tipo: Private
  - Región: elegir la más cercana geográficamente (Europe: eu-central)
- Crear Application Key: Account → App Keys → Add a New Application Key
  - Name: `cnpg-barman-key`
  - Bucket access: `serenamente-pg-backups` (solo este bucket)
  - Type of access: Read and Write
  - Guardar: `keyID` y `applicationKey`

**4.1.5 Resend (SMTP para emails transaccionales — Tier gratuito)**
- Crear cuenta en `resend.com`
- Añadir y verificar el dominio `serenamente.com` en Resend
- Crear API Key: Resend Dashboard → API Keys → Create API Key
- Tier gratuito: 3,000 emails/mes, 100/día — suficiente para Fase 1

### 4.2 Software en la máquina de trabajo (tu computadora)

El VPS con Talos no tiene shell, no tiene apt, no tiene herramientas instaladas. Todo el tooling para gestionar el cluster debe estar en tu máquina de trabajo local (la que uses para desarrollar y administrar).

```bash
# =============================================================
# talosctl — CLI de gestión del OS Talos
# =============================================================
# macOS:
brew install siderolabs/tap/talosctl

# Linux (script oficial):
curl -sL https://talos.dev/install | sh

# Windows (PowerShell como administrador):
# Descargar desde: https://github.com/siderolabs/talos/releases/latest
# Buscar: talosctl-windows-amd64.exe

# Verificar versión (debe coincidir con Talos v1.10.x del VPS):
talosctl version --client
# → Client: v1.10.x

# =============================================================
# kubectl — CLI de Kubernetes
# =============================================================
# macOS:
brew install kubectl

# Linux:
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Windows: https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/

# Verificar:
kubectl version --client
# → Client Version: v1.33.x

# =============================================================
# Helm v3.x — gestor de charts de Kubernetes
# =============================================================
# macOS:
brew install helm

# Linux:
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verificar:
helm version
# → version.BuildInfo{Version:"v3.x.x"}

# =============================================================
# flux CLI — CLI de FluxCD
# =============================================================
# macOS:
brew install fluxcd/tap/flux

# Linux:
curl -s https://fluxcd.io/install.sh | sudo bash

# Verificar:
flux version
# → flux: v2.x.x (client only — OK antes del bootstrap)

# =============================================================
# sops + age — Cifrado de secretos para GitOps (nativo en FluxCD)
# =============================================================
# macOS:
brew install sops age

# Linux:
# sops:
SOPS_VERSION=$(curl -s https://api.github.com/repos/getsops/sops/releases/latest | jq -r .tag_name)
curl -OL "https://github.com/getsops/sops/releases/download/${SOPS_VERSION}/sops-${SOPS_VERSION}.linux.amd64"
sudo install -m 755 "sops-${SOPS_VERSION}.linux.amd64" /usr/local/bin/sops

# age:
AGE_VERSION=$(curl -s https://api.github.com/repos/FiloSottile/age/releases/latest | jq -r .tag_name)
curl -OL "https://github.com/FiloSottile/age/releases/download/${AGE_VERSION}/age-${AGE_VERSION}-linux-amd64.tar.gz"
tar -xzf "age-${AGE_VERSION}-linux-amd64.tar.gz"
sudo install -m 755 age/age age/age-keygen /usr/local/bin/

# Verificar:
sops --version
age --version

# =============================================================
# hcloud CLI — CLI de Hetzner Cloud
# =============================================================
# macOS:
brew install hcloud

# Linux:
curl -LO https://github.com/hetznercloud/cli/releases/latest/download/hcloud-linux-amd64.tar.gz
tar -xzf hcloud-linux-amd64.tar.gz
sudo mv hcloud /usr/local/bin/

# Configurar (usar el API Token generado en 4.1.2):
hcloud context create serenamente-production
# → Enter API token: <pegar token>
hcloud context use serenamente-production

# Verificar:
hcloud server list

# =============================================================
# k9s — TUI para explorar el cluster (muy recomendado)
# =============================================================
# macOS:
brew install k9s

# Linux:
curl -sS https://webinstall.dev/k9s | bash

# =============================================================
# Go 1.25.x — para desarrollo local de microservicios
# =============================================================
# Descargar desde: https://go.dev/dl/
# macOS ARM:  go1.25.x.darwin-arm64.tar.gz
# macOS x86:  go1.25.x.darwin-amd64.tar.gz
# Linux:      go1.25.x.linux-amd64.tar.gz

# Linux (ejemplo):
curl -LO https://go.dev/dl/go1.25.0.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.25.0.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc

# Verificar:
go version
# → go version go1.25.x linux/amd64

# =============================================================
# buf CLI — toolchain de Protobuf
# =============================================================
# macOS:
brew install bufbuild/buf/buf

# Linux:
BUF_VERSION=$(curl -s https://api.github.com/repos/bufbuild/buf/releases/latest | jq -r .tag_name)
curl -OL "https://github.com/bufbuild/buf/releases/download/${BUF_VERSION}/buf-Linux-x86_64"
chmod +x buf-Linux-x86_64
sudo mv buf-Linux-x86_64 /usr/local/bin/buf

# Verificar:
buf --version

# =============================================================
# Node.js + Bun — para BFF Hono y Qwik SPA en local
# =============================================================
# Node.js LTS (para npm/npx):
# https://nodejs.org/en/download/

# Bun 1.3.x (runtime rápido para dev local):
curl -fsSL https://bun.sh/install | bash

# wrangler (Cloudflare Workers CLI):
npm install -g wrangler
# Verificar:
wrangler --version

# =============================================================
# Docker — para builds de imágenes de contenedor
# =============================================================
# Instalar Docker Desktop (Mac/Windows) o Docker Engine (Linux)
# https://docs.docker.com/get-docker/

# Verificar:
docker version

# =============================================================
# jq — procesador JSON (usado en scripts)
# =============================================================
# macOS: brew install jq
# Linux: sudo apt install jq / sudo yum install jq

# =============================================================
# git — control de versiones
# =============================================================
# macOS: brew install git
# Linux: sudo apt install git
# Verificar versión ≥ 2.30 (para soporte de signed commits):
git --version
```

### 4.3 Variables de entorno de trabajo

Crear un archivo `.envrc` en la raíz del monorepo (NO commitear — añadir a `.gitignore`):

```bash
# .envrc — Variables de entorno de trabajo (NO commitear)
# Cargar con: source .envrc  o  usar direnv (https://direnv.net/)

# Hetzner
export HCLOUD_TOKEN="<token-api-hetzner>"
export HETZNER_PROJECT="serenamente-production"
export VPS_IP=""  # Se completa en Tarea V-01

# GitLab
export GITLAB_TOKEN="<token-gitlab-pat>"
export GITLAB_USER="serenamente"
export GITLAB_REPO="serenidad-platform"
export GITLAB_DEPLOY_TOKEN_USER="<deploy-token-username>"
export GITLAB_DEPLOY_TOKEN="<deploy-token-value>"

# Cloudflare
export CF_API_TOKEN="<token-api-cloudflare>"
export CF_ZONE_ID="<zone-id-cloudflare>"
export CF_ACCOUNT_ID="<account-id-cloudflare>"
export CF_DOMAIN="serenamente.com"

# Backblaze B2
export B2_KEY_ID="<keyID-backblaze>"
export B2_APPLICATION_KEY="<applicationKey-backblaze>"
export B2_BUCKET="serenamente-pg-backups"
export B2_ENDPOINT="s3.us-west-004.backblazeb2.com"  # Ajustar por región

# Resend (SMTP)
export RESEND_API_KEY="<api-key-resend>"

# kubeconfig y talosconfig (se generan en V-03)
export KUBECONFIG="$(pwd)/infra/clusters/hetzner-prod/kubeconfig"
export TALOSCONFIG="$(pwd)/infra/clusters/hetzner-prod/talos/talosconfig"
```

### 4.4 Estructura del monorepo (a crear antes de empezar)

```bash
# Clonar el repo (ya debe existir en GitLab):
git clone git@gitlab.com:${GITLAB_USER}/${GITLAB_REPO}.git
cd ${GITLAB_REPO}

# Crear estructura de directorios completa:
mkdir -p infra/clusters/hetzner-prod/talos/patches
mkdir -p infra/clusters/hetzner-prod/flux-system
mkdir -p infra/infrastructure/cert-manager
mkdir -p infra/infrastructure/traefik
mkdir -p infra/infrastructure/cnpg
mkdir -p infra/infrastructure/nats           # Fase 2
mkdir -p infra/apps/iam-service
mkdir -p infra/apps/kratos
mkdir -p infra/apps/scheduling-service       # Fase 2
mkdir -p infra/apps/clinical-service         # Fase 2
mkdir -p infra/apps/billing-service          # Fase 3
mkdir -p infra/apps/openfga                  # Fase 3
mkdir -p infra/secrets
mkdir -p services/iam
mkdir -p services/scheduling                 # Fase 2
mkdir -p services/clinical                   # Fase 2
mkdir -p services/billing                    # Fase 3
mkdir -p packages/events/proto               # Schemas Protobuf
mkdir -p apps/web                            # Qwik SPA
mkdir -p apps/bff                            # Hono CF Workers
mkdir -p scripts

# Archivo .gitignore inicial:
cat > .gitignore <<'EOF'
# Secrets locales (NUNCA commitear)
.envrc
*.talosconfig
infra/clusters/*/talos/secrets.yaml
infra/clusters/*/kubeconfig
*.env
*.env.local
.env.*

# Herramientas de build
node_modules/
dist/
.wrangler/
*.wasm

# Go
bin/
vendor/

# OS
.DS_Store
Thumbs.db
EOF

git add .gitignore
git commit -m "chore: initial monorepo structure and gitignore"
git push origin main
```

---

## 5. Tarea V-00 — Preparación del monorepo

### 5.1 Configurar SSH Key para Hetzner (acceso rescue)

Aunque Talos Linux no usa SSH en producción, el modo rescue de Hetzner sí lo usa para recuperación de emergencia. También se necesita la SSH key para el bootstrap inicial mediante `talosctl` si se usa la imagen de instalación vía red.

```bash
# Generar SSH key dedicada para Hetzner (si no tienes una):
ssh-keygen -t ed25519 -C "hetzner-serenamente-$(date +%Y%m%d)" \
  -f ~/.ssh/hetzner_serenamente_ed25519

# Mostrar la clave pública para agregar a Hetzner:
cat ~/.ssh/hetzner_serenamente_ed25519.pub

# Agregar al proyecto Hetzner via CLI:
hcloud ssh-key create \
  --name "serenamente-admin-key" \
  --public-key-from-file ~/.ssh/hetzner_serenamente_ed25519.pub

# Verificar:
hcloud ssh-key list
# → ID    NAME                    FINGERPRINT
# → 12345 serenamente-admin-key   xx:xx:xx:...
```

### 5.2 Configurar variables CI/CD de GitLab

Antes de implementar el pipeline CI/CD, precargar las variables en GitLab para que los jobs puedan usarlas:

```bash
# Instalar GitLab CLI si no está disponible:
# macOS: brew install glab
# Linux/Windows: https://gitlab.com/gitlab-org/cli/-/releases

# Autenticar GitLab CLI:
glab auth login

# Agregar variables CI/CD al repositorio (--masked las oculta en los logs de pipeline):
glab variable set HCLOUD_TOKEN --value "${HCLOUD_TOKEN}" --masked
glab variable set CF_API_TOKEN --value "${CF_API_TOKEN}" --masked
glab variable set CF_ACCOUNT_ID --value "${CF_ACCOUNT_ID}" --masked
glab variable set B2_KEY_ID --value "${B2_KEY_ID}" --masked
glab variable set B2_APPLICATION_KEY --value "${B2_APPLICATION_KEY}" --masked
glab variable set RESEND_API_KEY --value "${RESEND_API_KEY}" --masked

# Verificar:
glab variable list

# Alternativa via web UI: Settings → CI/CD → Variables → Add variable
# Marcar "Masked" para secrets y "Protected" para que solo corran en ramas protegidas (main)
```

---

## 6. Tarea V-01 — Aprovisionamiento del VPS Hetzner

### 6.1 Crear el servidor CX32

La creación del servidor se hace a través de la CLI `hcloud` para reproducibilidad. El datacenter recomendado es `nbg1` (Nuremberg, Alemania) o `hel1` (Helsinki, Finlandia) por latencia hacia Europa y América Latina.

```bash
# Verificar tipos de servidor disponibles:
hcloud server-type list | grep CX32
# → CX32  4 cores  8 GB RAM  80 GB SSD

# Verificar datacenters disponibles:
hcloud datacenter list
# Elegir: nbg1-dc3 (Nuremberg) o hel1-dc2 (Helsinki)

# Crear el servidor CX32
# IMPORTANTE: La imagen "ubuntu-24.04" es temporal — Talos reemplazará el OS.
# El servidor se crea con Ubuntu para poder usar el modo rescate de Hetzner.
hcloud server create \
  --name serenamente-prod-01 \
  --type cx32 \
  --image ubuntu-24.04 \
  --datacenter nbg1-dc3 \
  --ssh-key serenamente-admin-key \
  --location nbg1

# Guardar la IP asignada:
VPS_IP=$(hcloud server describe serenamente-prod-01 -o json | jq -r '.public_net.ipv4.ip')
echo "VPS_IP=${VPS_IP}" >> .envrc
echo "IP del servidor: ${VPS_IP}"
```

### 6.2 Crear y asignar una Floating IP (IP estática)

Una Floating IP permanece aunque el servidor sea reconstruido. Esto es crítico: si Talos necesita ser reinstalado o el servidor migrado, la IP pública no cambiará, preservando los registros DNS y la configuración de Cloudflare.

```bash
# Crear Floating IP en la misma región que el servidor:
hcloud floating-ip create \
  --type ipv4 \
  --home-location nbg1 \
  --name serenamente-prod-fip \
  --description "IP flotante produccion Serenamente"

# Obtener la Floating IP asignada:
FLOATING_IP=$(hcloud floating-ip describe serenamente-prod-fip -o json | jq -r '.ip')
echo "Floating IP: ${FLOATING_IP}"
echo "FLOATING_IP=${FLOATING_IP}" >> .envrc

# Asignar la Floating IP al servidor:
hcloud floating-ip assign serenamente-prod-fip serenamente-prod-01

# Verificar asignación:
hcloud floating-ip describe serenamente-prod-fip
```

> **Nota sobre Floating IP vs IP del servidor:** El servidor tiene su propia IP pública asignada automáticamente por Hetzner. La Floating IP es una IP adicional que puedes reasignar entre servidores. Para Serenamente se usará la Floating IP como la IP pública canónica (la que configurarás en DNS). La IP directa del servidor queda como acceso de emergencia.

### 6.3 Configurar Firewall de Hetzner

El Firewall de Hetzner opera a nivel de infraestructura, antes de que el tráfico llegue al servidor. Es la primera línea de defensa.

```bash
# Crear el firewall:
hcloud firewall create --name serenamente-prod-fw

# Reglas de entrada (ingress):
# HTTP — necesario para ACME HTTP-01 challenge de Let's Encrypt
hcloud firewall add-rule serenamente-prod-fw \
  --direction in \
  --protocol tcp \
  --port 80 \
  --source-ips 0.0.0.0/0 \
  --source-ips ::/0 \
  --description "HTTP Let's Encrypt ACME"

# HTTPS — tráfico principal de la aplicación
hcloud firewall add-rule serenamente-prod-fw \
  --direction in \
  --protocol tcp \
  --port 443 \
  --source-ips 0.0.0.0/0 \
  --source-ips ::/0 \
  --description "HTTPS aplicacion"

# Talos API — RESTRINGIR a tu IP de trabajo solamente
MY_IP=$(curl -s https://ifconfig.me)
hcloud firewall add-rule serenamente-prod-fw \
  --direction in \
  --protocol tcp \
  --port 50000 \
  --source-ips "${MY_IP}/32" \
  --description "Talos API (solo IP de trabajo)"

# Kubernetes API — RESTRINGIR a tu IP de trabajo solamente
hcloud firewall add-rule serenamente-prod-fw \
  --direction in \
  --protocol tcp \
  --port 6443 \
  --source-ips "${MY_IP}/32" \
  --description "Kubernetes API (solo IP de trabajo)"

# Aplicar firewall al servidor:
hcloud firewall apply-to-resource serenamente-prod-fw \
  --type server \
  --server serenamente-prod-01

# Verificar reglas:
hcloud firewall describe serenamente-prod-fw
```

> **Advertencia de seguridad:** Los puertos 50000 (Talos API) y 6443 (Kubernetes API) solo deben ser accesibles desde tu IP de trabajo. Si tu IP cambia (IP dinámica de ISP), deberás actualizar estas reglas. Considera usar una VPN con IP fija para el acceso administrativo.

### 6.4 Verificar acceso SSH al servidor (Ubuntu temporal)

```bash
# El servidor tiene Ubuntu temporalmente — verificar acceso antes de instalar Talos:
ssh -i ~/.ssh/hetzner_serenamente_ed25519 root@${VPS_IP}
# Debe responder con el prompt de Ubuntu

# Dentro del servidor, verificar hardware:
nproc                    # → 4 (vCPUs)
free -h                  # → 7.7 GB RAM aprox
df -h /                  # → 80 GB disco
lsblk                    # → Identificar el disco principal (generalmente /dev/sda en CX32)

# Anotar el nombre del disco para usar en V-02:
# En CX32 típicamente es /dev/sda
exit
```

---

## 7. Tarea V-02 — Instalación de Talos Linux en Hetzner CX32

### 7.1 Estrategia de instalación: imagen Talos vía modo rescate de Hetzner

Hetzner no tiene una imagen nativa de Talos Linux en su catálogo (a diferencia de Ubuntu/Debian/etc.). La forma recomendada para instalar Talos en un servidor Hetzner es usando el **modo rescate de Hetzner** para descargar y flashear directamente la imagen de Talos en el disco del servidor.

```bash
# Paso 1: Obtener la versión actual de Talos:
TALOS_VERSION=$(curl -s https://api.github.com/repos/siderolabs/talos/releases/latest | jq -r .tag_name)
echo "Versión Talos a instalar: ${TALOS_VERSION}"
# → v1.10.x

# Paso 2: Poner el servidor en modo rescate (Hetzner Rescue System):
hcloud server enable-rescue serenamente-prod-01 \
  --type linux64 \
  --ssh-key serenamente-admin-key
# → Rescue activado. El servidor DEBE ser reiniciado para entrar en rescue.

# Paso 3: Reiniciar el servidor para que entre en modo rescate:
hcloud server reset serenamente-prod-01

# Esperar ~30-60 segundos para que el rescue system arranque:
sleep 60

# Paso 4: Conectar al rescue system:
ssh -i ~/.ssh/hetzner_serenamente_ed25519 root@${VPS_IP}
# El prompt será del Hetzner Rescue Linux, no de Ubuntu
```

### 7.2 Generar imagen de Talos con extensión qemu-guest-agent

El CX32 es una VM QEMU en la infraestructura de Hetzner. Para que Hetzner pueda comunicarse con el sistema operativo (métricas de la VM, shutdown ordenado, etc.), Talos necesita la extensión `qemu-guest-agent`. Esta extensión se incluye en el schematic de Talos Image Factory.

```bash
# Desde tu MÁQUINA DE TRABAJO (no desde el rescue SSH):
# Construir el schematic con la extensión qemu-guest-agent:

# Crear el archivo de schematic:
cat > /tmp/talos-hetzner-schematic.yaml <<'EOF'
customization:
  systemExtensions:
    officialExtensions:
      - siderolabs/qemu-guest-agent
EOF

# Subir el schematic al Talos Image Factory y obtener el ID:
SCHEMATIC_ID=$(curl -s -X POST \
  "https://factory.talos.dev/schematics" \
  -H "Content-Type: application/yaml" \
  --data-binary @/tmp/talos-hetzner-schematic.yaml \
  | jq -r '.id')

echo "Schematic ID: ${SCHEMATIC_ID}"
# → Ejemplo: 376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4b

# La URL de la imagen disk.raw para Hetzner (x86_64):
TALOS_VERSION="v1.10.0"  # Ajustar a la versión exacta actual
IMAGE_URL="https://factory.talos.dev/image/${SCHEMATIC_ID}/${TALOS_VERSION}/hcloud-amd64.raw.xz"
echo "URL de imagen: ${IMAGE_URL}"
```

### 7.3 Flashear Talos en el disco del servidor (desde rescue)

```bash
# Desde la sesión SSH del RESCUE SYSTEM de Hetzner:
# (la sesión SSH abierta en el paso 7.1)

TALOS_VERSION="v1.10.0"
SCHEMATIC_ID="376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4b"
IMAGE_URL="https://factory.talos.dev/image/${SCHEMATIC_ID}/${TALOS_VERSION}/hcloud-amd64.raw.xz"

# Verificar el disco principal del servidor:
lsblk
# → NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
# → sda    8:0     0   80G  0  disk

# Descargar y flashear la imagen Talos directamente al disco:
# Este comando descarga la imagen comprimida (.xz) y la descomprime en /dev/sda
# En un CX32 esto tarda aproximadamente 2-4 minutos
curl -L "${IMAGE_URL}" \
  | xz --decompress \
  | dd of=/dev/sda bs=4M status=progress

# Sincronizar buffers de disco antes de reiniciar:
sync

# Reiniciar el servidor para que arranque desde la imagen de Talos:
reboot
```

### 7.4 Verificar que Talos está en modo mantenimiento

```bash
# Desde tu MÁQUINA DE TRABAJO:
# Esperar ~60-90 segundos para que el servidor reinicie y Talos entre en modo mantenimiento:
sleep 90

# Verificar que Talos responde en el puerto 50000 (modo mantenimiento):
talosctl version --insecure --nodes ${VPS_IP}
# Output esperado:
# Client:
#   Tag: v1.10.x
#   ...
# Server:
#   Tag: v1.10.x  (esto confirma que Talos está en maintenance mode)
#   ...

# Si no responde: verificar que el firewall permite el puerto 50000 desde tu IP.
# Si el puerto no responde después de 3 minutos: verificar en consola Hetzner que el servidor está corriendo.
```

---

## 8. Tarea V-03 — Bootstrap de Talos + Kubernetes 1.33.x

### 8.1 Patch de configuración específico para Hetzner CX32

```yaml
# infra/clusters/hetzner-prod/talos/patches/hetzner-cx32.yaml
# Patch de configuración específico para Hetzner CX32 con Talos Linux
machine:
  # Configuración de red con Floating IP
  network:
    hostname: serenamente-prod-01
    # Hetzner asigna la IP principal via DHCP — dejar DHCP para la IP principal
    # La Floating IP se configura como alias adicional
    interfaces:
      - interface: eth0
        dhcp: true
        # Alias de Floating IP (Hetzner no la configura vía DHCP — se agrega manualmente)
        addresses:
          - "${FLOATING_IP}/32"   # Reemplazar con la Floating IP real

  # Disco de instalación (en CX32 siempre es /dev/sda)
  install:
    disk: /dev/sda
    # La imagen ya está flasheada — bootloader está en su lugar
    bootloader: true
    wipe: false   # NO wipe — la imagen ya está instalada

  # Configuración de tiempo (crítico para certificados TLS)
  time:
    servers:
      - 0.de.pool.ntp.org
      - 1.de.pool.ntp.org
      - time.cloudflare.com

  # kubelet extra config para Hetzner
  kubelet:
    extraArgs:
      # Hetzner CCM requiere este provider-id
      provider-id: "hcloud://$(cat /sys/class/dmi/id/product_serial)"
    nodeIP:
      validSubnets:
        - 10.0.0.0/8    # Red privada de Hetzner si se configura
        - 0.0.0.0/0     # IP pública como fallback

  # Configuración del registry (registry.gitlab.com — producción, requiere imagePullSecret)
  # La autenticación al GitLab Container Registry se gestiona con imagePullSecret
  # en cada Deployment (ver Tarea V-14). Imágenes de terceros (ghcr.io, gcr.io) no necesitan config.
  registries: {}

  # Sysctls recomendados para producción
  sysctls:
    net.ipv4.ip_forward: "1"
    net.bridge.bridge-nf-call-iptables: "1"
    net.bridge.bridge-nf-call-ip6tables: "1"
    vm.max_map_count: "262144"   # Requerido por ElasticSearch si se agrega en Fase 4+
    fs.inotify.max_user_instances: "8192"
    fs.inotify.max_user_watches: "524288"
    kernel.pid_max: "4194304"

cluster:
  # Single-node: el control plane también ejecuta pods de trabajo
  allowSchedulingOnControlPlanes: true

  # Configuración del API server
  apiServer:
    certSANs:
      - "${VPS_IP}"        # IP directa del servidor
      - "${FLOATING_IP}"   # Floating IP (usar esta en kubeconfig)
      - "127.0.0.1"

  # Hetzner Cloud Controller Manager necesita acceso al API de Hetzner
  # Se configura como DaemonSet en V-04, no aquí

  # Configuración de red del cluster
  network:
    podSubnets:
      - 10.244.0.0/16
    serviceSubnets:
      - 10.96.0.0/12
    cni:
      name: flannel   # Flannel es suficiente para single-node

  # etcd configuración (por defecto es correcta, documentar por claridad)
  etcd:
    advertisedSubnets:
      - 10.0.0.0/8
      - 0.0.0.0/0
```

> **Nota sobre la Floating IP en el patch:** Talos necesita saber sobre la Floating IP para incluirla en el SANs del certificado del API server. El CX32 recibe su IP principal via DHCP de Hetzner. La Floating IP debe añadirse como alias de interfaz para que el tráfico llegue correctamente.

### 8.2 Generar secrets y configuración del cluster

```bash
# Desde la raíz del monorepo, en tu máquina de trabajo:
source .envrc

# Crear directorio de talos:
mkdir -p infra/clusters/hetzner-prod/talos/patches

# Sustituir variables en el patch (crear versión con IPs reales):
envsubst < infra/clusters/hetzner-prod/talos/patches/hetzner-cx32.yaml.template \
  > infra/clusters/hetzner-prod/talos/patches/hetzner-cx32.yaml

# Generar secrets del cluster (SOLO UNA VEZ — regenerar invalida el cluster):
talosctl gen secrets \
  --output-file infra/clusters/hetzner-prod/talos/secrets.yaml
# IMPORTANTE: Este archivo contiene claves privadas del cluster.
# Está en .gitignore — NO commitear.
# Hacer backup seguro: password manager o vault cifrado.

echo "BACKUP CRÍTICO: infra/clusters/hetzner-prod/talos/secrets.yaml"
echo "Si pierdes este archivo, no podrás regenerar las credenciales del cluster."

# Generar configuración del cluster:
talosctl gen config serenamente-production \
  "https://${FLOATING_IP}:6443" \
  --with-secrets infra/clusters/hetzner-prod/talos/secrets.yaml \
  --config-patch @infra/clusters/hetzner-prod/talos/patches/hetzner-cx32.yaml \
  --output-dir infra/clusters/hetzner-prod/talos/ \
  --force

# Genera:
# infra/clusters/hetzner-prod/talos/controlplane.yaml  ← config del nodo (NO commitear raw)
# infra/clusters/hetzner-prod/talos/worker.yaml        ← para futuros workers en Fase 4+
# infra/clusters/hetzner-prod/talos/talosconfig        ← credenciales talosctl (NO commitear)

# Agregar a .gitignore para seguridad:
cat >> .gitignore <<'EOF'
infra/clusters/hetzner-prod/talos/controlplane.yaml
infra/clusters/hetzner-prod/talos/worker.yaml
infra/clusters/hetzner-prod/talos/talosconfig
infra/clusters/hetzner-prod/talos/secrets.yaml
infra/clusters/hetzner-prod/kubeconfig
EOF
```

### 8.3 Aplicar configuración al servidor Talos

```bash
source .envrc

TALOSCONFIG="infra/clusters/hetzner-prod/talos/talosconfig"

# El servidor debe estar en maintenance mode (verificado en 7.4)
# Aplicar la configuración del control plane:
talosctl apply-config \
  --insecure \
  --nodes ${VPS_IP} \
  --file infra/clusters/hetzner-prod/talos/controlplane.yaml

# El servidor se reconfigurará y puede reiniciarse (~1-3 min)
# Esperar a que Talos esté en estado "ready" después del reinicio:
echo "Esperando que Talos esté disponible..."
sleep 90

# Verificar que Talos responde con las nuevas credenciales:
talosctl --talosconfig ${TALOSCONFIG} version --nodes ${FLOATING_IP}
# El servidor ahora responderá en la Floating IP

# Bootstrap etcd (SOLO una vez, primer arranque del cluster):
# Este comando inicia el cluster de Kubernetes. Solo se ejecuta una vez en la vida del cluster.
talosctl bootstrap \
  --talosconfig ${TALOSCONFIG} \
  --nodes ${FLOATING_IP}

echo "Bootstrap etcd ejecutado. Esperar ~2-3 minutos para que el API server esté disponible..."
sleep 180
```

### 8.4 Obtener kubeconfig y verificar el cluster

```bash
source .envrc
TALOSCONFIG="infra/clusters/hetzner-prod/talos/talosconfig"

# Obtener el kubeconfig:
talosctl kubeconfig \
  --talosconfig ${TALOSCONFIG} \
  --nodes ${FLOATING_IP} \
  --output infra/clusters/hetzner-prod/kubeconfig \
  --force

# El kubeconfig apunta a la Floating IP
export KUBECONFIG="$(pwd)/infra/clusters/hetzner-prod/kubeconfig"

# Verificar el nodo:
kubectl get nodes -o wide
# Output esperado:
# NAME                    STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP
# serenamente-prod-01     Ready    control-plane   5m    v1.33.x   <FLOATING_IP> <none>

# Verificar todos los pods del sistema:
kubectl get pods -A
# Todos los pods de kube-system deben estar Running o Completed.
# flannel, coredns, kube-apiserver, kube-controller-manager, kube-scheduler, etcd

# Verificar salud general del cluster:
talosctl health \
  --talosconfig ${TALOSCONFIG} \
  --nodes ${FLOATING_IP}
# → [OK] etcd is healthy
# → [OK] control plane is healthy
# → [OK] all nodes are ready

# Crear alias para el trabajo diario:
echo "alias kp='kubectl --kubeconfig $(pwd)/infra/clusters/hetzner-prod/kubeconfig'" >> ~/.bashrc
echo "alias tp='talosctl --talosconfig $(pwd)/infra/clusters/hetzner-prod/talos/talosconfig --nodes ${FLOATING_IP}'" >> ~/.bashrc
source ~/.bashrc
```

### 8.5 Crear namespaces del cluster

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: traefik
  labels:
    app.kubernetes.io/managed-by: flux
---
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager
  labels:
    app.kubernetes.io/managed-by: flux
---
apiVersion: v1
kind: Namespace
metadata:
  name: serenamente-secrets
  labels:
    app.kubernetes.io/managed-by: flux
---
apiVersion: v1
kind: Namespace
metadata:
  name: cnpg-system
  labels:
    app.kubernetes.io/managed-by: flux
---
apiVersion: v1
kind: Namespace
metadata:
  name: serenamente-data
  labels:
    environment: production
    app.kubernetes.io/managed-by: flux
---
apiVersion: v1
kind: Namespace
metadata:
  name: serenamente-core
  labels:
    environment: production
    app.kubernetes.io/managed-by: flux
---
apiVersion: v1
kind: Namespace
metadata:
  name: serenamente-ops
  labels:
    environment: production
    app.kubernetes.io/managed-by: flux
EOF

# Verificar:
kubectl get namespaces
```

---

## 9. Tarea V-04 — Hetzner Cloud Controller Manager (CCM)

### 9.1 ¿Por qué es necesario el CCM?

El Hetzner Cloud Controller Manager es un componente que conecta Kubernetes con la API de Hetzner Cloud. Sin él, Kubernetes no sabe que está corriendo en Hetzner y varias funcionalidades fallan:

- Los nodos quedan en estado `NotReady` con el taint `node.cloudprovider.kubernetes.io/uninitialized`
- Los `LoadBalancer` Services no crean load balancers en Hetzner (necesario para Traefik en modo Service type LoadBalancer)
- La metadata del nodo (zona, región) no está disponible para el scheduler

Para Fase 1 con un solo nodo, el CCM es especialmente importante para quitar el taint `uninitialized` y permitir que Traefik use `hostPort` correctamente.

### 9.2 Crear el Secret con el API token de Hetzner

```bash
# El CCM necesita el API token de Hetzner para comunicarse con la API de Hetzner Cloud.
# Este Secret se crea ANTES de FluxCD porque el CCM es prerequisito del cluster.

kubectl create namespace hcloud-system --dry-run=client -o yaml | kubectl apply -f -

# Crear el Secret con el token de Hetzner:
kubectl create secret generic hcloud \
  --namespace hcloud-system \
  --from-literal=token="${HCLOUD_TOKEN}" \
  --from-literal=network="" \
  --dry-run=client -o yaml | kubectl apply -f -

# Verificar (el token debe aparecer ofuscado):
kubectl get secret hcloud -n hcloud-system -o yaml
```

### 9.3 Desplegar el CCM como manifiesto directo

Para el bootstrap inicial, el CCM se despliega directamente antes de que FluxCD esté disponible. Luego FluxCD lo tomará bajo su gestión.

```yaml
# infra/infrastructure/hcloud-ccm/hcloud-ccm-deployment.yaml
# Ref: https://github.com/hetznercloud/hcloud-cloud-controller-manager
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hcloud-cloud-controller-manager
  namespace: hcloud-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hcloud-cloud-controller-manager
  template:
    metadata:
      labels:
        app: hcloud-cloud-controller-manager
    spec:
      serviceAccountName: hcloud-cloud-controller-manager
      tolerations:
        # El CCM debe tolerar nodos no inicializados para poder inicializarlos
        - key: node.cloudprovider.kubernetes.io/uninitialized
          value: "true"
          effect: NoSchedule
        - key: CriticalAddonsOnly
          operator: Exists
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
          operator: Exists
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
          operator: Exists
        - key: node.kubernetes.io/not-ready
          effect: NoSchedule
      containers:
        - name: hcloud-cloud-controller-manager
          image: hetznercloud/hcloud-cloud-controller-manager:latest
          command:
            - "/bin/hcloud-cloud-controller-manager"
            - "--cloud-provider=hcloud"
            - "--leader-elect=false"
            - "--allow-untagged-cloud"
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: HCLOUD_TOKEN
              valueFrom:
                secretKeyRef:
                  name: hcloud
                  key: token
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "200m"
      priorityClassName: system-cluster-critical
```

```bash
# Aplicar los manifests del CCM (RBAC + Deployment):
# Usar el manifest oficial del repositorio hcloud:
kubectl apply -f https://github.com/hetznercloud/hcloud-cloud-controller-manager/releases/latest/download/ccm.yaml

# Verificar que el CCM está corriendo:
kubectl get pods -n hcloud-system
# → hcloud-cloud-controller-manager-xxx   Running

# Verificar que el nodo ya no tiene el taint uninitialized:
kubectl describe node serenamente-prod-01 | grep Taint
# → No taints after CCM initialization

# Verificar que el nodo tiene las labels de Hetzner:
kubectl get node serenamente-prod-01 -o yaml | grep -A5 "topology.kubernetes.io"
# → topology.kubernetes.io/region: eu-central
# → topology.kubernetes.io/zone: nbg1
```

---

## 10. Tarea V-05 — FluxCD v2.x (GitOps desde el primer día)

FluxCD es el núcleo del principio de operaciones de Serenamente: **Git es la única fuente de verdad**. Todo cambio a producción debe pasar por un commit en el repositorio. Esta tarea bootstrap FluxCD en el cluster y conecta el repositorio de GitLab.

### 10.1 Pre-checks antes del bootstrap

```bash
# Verificar que el cluster está listo para FluxCD:
flux check --pre
# → ✓ Kubernetes 1.33.x >= 1.28.0
# → ✓ prerequisites checks passed

# Verificar que el GitLab token tiene los permisos correctos:
# (GITLAB_TOKEN debe estar exportado en el entorno)
export GITLAB_TOKEN="${GITLAB_TOKEN}"
flux check --pre
```

### 10.2 Bootstrap FluxCD apuntando al monorepo

```bash
source .envrc

# Bootstrap de FluxCD — apunta a main y al path de producción:
export GITLAB_TOKEN="${GITLAB_TOKEN}"
flux bootstrap gitlab \
  --owner="${GITLAB_USER}" \
  --repository="${GITLAB_REPO}" \
  --branch=main \
  --path="./infra/clusters/hetzner-prod" \
  --personal \
  --components-extra=image-reflector-controller,image-automation-controller

# Este comando:
# 1. Instala los controllers de FluxCD en el namespace flux-system
# 2. Crea un deploy key en el repo de GitLab para que FluxCD pueda leer el código
# 3. Commite los manifests de FluxCD en infra/clusters/hetzner-prod/flux-system/
# 4. Crea un GitRepository que apunta a gitlab.com/{GITLAB_USER}/{GITLAB_REPO}
# 5. Crea una Kustomization que aplica todo lo que está en infra/clusters/hetzner-prod/

# Verificar que FluxCD está sincronizando:
flux get all
# → NAME             READY   MESSAGE
# → gitrepository    True    stored artifact for revision 'main@sha1:...'
# → kustomization    True    Applied revision: main@sha1:...

# Verificar pods de FluxCD:
kubectl get pods -n flux-system

# → source-controller-xxx             Running
# → kustomize-controller-xxx          Running
# → helm-controller-xxx               Running
# → notification-controller-xxx       Running
# → image-reflector-controller-xxx    Running  (para auto-update de imágenes)
# → image-automation-controller-xxx   Running  (para auto-update de imágenes)
```

### 10.3 Configurar SOPS+age para decryption en FluxCD

Este paso es crítico y debe ejecutarse **inmediatamente después** del bootstrap. FluxCD necesita la clave `age` para descifrar los secrets en Git antes de aplicarlos al cluster.

```bash
source .envrc

# 1. Generar la clave age (si no existe ya):
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt 2>&1 | tee /tmp/age-keygen-output.txt
# Salida: Public key: age1xxxx... (guardar este valor en .envrc)

# Exportar la clave pública (la clave PRIVADA permanece en ~/.config/sops/age/keys.txt):
export AGE_PUBLIC_KEY=$(grep '^# public key:' ~/.config/sops/age/keys.txt | awk '{print $4}')
echo "AGE_PUBLIC_KEY=${AGE_PUBLIC_KEY}"

# BACKUP CRÍTICO: guardar en password manager la clave PRIVADA
echo "=== GUARDAR EN PASSWORD MANAGER ==="
cat ~/.config/sops/age/keys.txt
echo "====================================="

# 2. Crear el Secret de Kubernetes con la clave privada age:
# FluxCD kustomize-controller lo usa para descifrar los secrets en Git
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=~/.config/sops/age/keys.txt

# Verificar:
kubectl get secret sops-age -n flux-system

# 3. Habilitar el decryption en la Kustomization principal de FluxCD:
# Parchear la Kustomization que FluxCD creó en el bootstrap:
kubectl patch kustomization flux-system \
  -n flux-system \
  --type=merge \
  -p '{"spec":{"decryption":{"provider":"sops","secretRef":{"name":"sops-age"}}}}'

# Verificar que la Kustomization tiene el decryption configurado:
kubectl get kustomization flux-system -n flux-system -o yaml | grep -A5 decryption
# → decryption:
# →   provider: sops
# →   secretRef:
# →     name: sops-age

# 4. Crear el archivo .sops.yaml en la raiz del repo para definir las reglas de cifrado:
cat > infra/secrets/.sops.yaml <<EOF
creation_rules:
  - path_regex: infra/secrets/.*\.yaml$
    encrypted_regex: ^(data|stringData)$
    age: ${AGE_PUBLIC_KEY}
EOF

git add infra/secrets/.sops.yaml
git commit -m "feat: add SOPS age encryption rules for infra/secrets/"
git push

echo "✓ SOPS+age configurado. FluxCD descifrara automaticamente todos los secrets en infra/secrets/"
```

### 10.4 Estructura del directorio de producción para FluxCD

```yaml
# infra/clusters/hetzner-prod/kustomization.yaml
# Este es el punto de entrada principal de FluxCD para el cluster de producción.
# FluxCD lee este archivo y aplica todo lo que referencia.
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # Bootstrap de FluxCD (auto-generado, no modificar manualmente)
  - flux-system/

  # Infraestructura base (en orden de dependencia)
  - ../../infrastructure/hcloud-ccm/          # CCM ya desplegado, FluxCD lo adopta
  - ../../infrastructure/cert-manager/        # V-07
  - ../../infrastructure/traefik/             # V-08
  - ../../infrastructure/cnpg/               # V-10

  # Aplicaciones de Fase 1
  - ../../apps/kratos/                        # V-13
  - ../../apps/iam-service/                   # V-14

  # Fase 2 (descommentar cuando llegue el momento)
  # - ../../infrastructure/nats/
  # - ../../apps/scheduling-service/
  # - ../../apps/clinical-service/

  # Fase 3 (descommentar cuando llegue el momento)
  # - ../../apps/openfga/
  # - ../../apps/billing-service/
  # - ../../infrastructure/kube-prometheus-stack/
```

---

## 11. Tarea V-06 — Estructura GitOps del repositorio

### 11.1 Organización de la infraestructura compartida

La infraestructura compartida (cert-manager, Traefik, etc.) se organiza de forma que pueda ser referenciada desde cualquier cluster (hetzner-prod, o futuros clusters). Los patches específicos del cluster van en `infra/clusters/hetzner-prod/`.

```bash
# Estructura completa de la capa infra/:
infra/
├── clusters/
│   └── hetzner-prod/
│       ├── kustomization.yaml          ← Punto de entrada de FluxCD (V-05)
│       ├── flux-system/                ← Auto-generado por flux bootstrap
│       │   ├── gotk-components.yaml
│       │   ├── gotk-sync.yaml
│       │   └── kustomization.yaml
│       └── talos/
│           ├── patches/
│           │   └── hetzner-cx32.yaml   ← Patch de configuración Talos (V-03)
│           ├── secrets.yaml            ← .gitignore — NO commitear
│           ├── talosconfig             ← .gitignore — NO commitear
│           ├── controlplane.yaml       ← .gitignore — NO commitear
│           └── worker.yaml             ← .gitignore — NO commitear
│
├── infrastructure/
│   ├── cert-manager/
│   │   ├── namespace.yaml
│   │   ├── helmrepository.yaml         ← HelmRepository de jetstack
│   │   ├── helmrelease.yaml            ← HelmRelease cert-manager
│   │   └── clusterissuers.yaml         ← ClusterIssuers letsencrypt-prod + staging
│   │
│   ├── traefik/
│   │   ├── namespace.yaml
│   │   ├── helmrepository.yaml
│   │   ├── helmrelease.yaml
│   │   └── middlewares.yaml            ← ForwardAuth middleware para IAM
│   │
│   ├── cnpg/
│   │   ├── namespace.yaml
│   │   ├── helmrepository.yaml
│   │   ├── helmrelease.yaml            ← CloudNativePG operator
│   │   └── cluster.yaml                ← PostgreSQL 17.4 Cluster CRD (6 databases)
│   │
│   └── nats/                           ← Fase 2
│       ├── namespace.yaml
│       ├── helmrepository.yaml
│       ├── helmrelease.yaml
│       └── jetstreams.yaml             ← NACK CRDs para streams y consumers
│
├── apps/
│   ├── kratos/
│   │   ├── helmrepository.yaml
│   │   ├── helmrelease.yaml
│   │   └── configmap-kratos.yaml
│   │
│   ├── iam-service/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── ingress-route.yaml          ← IngressRoute Traefik
│   │   └── migrations-job.yaml         ← Job de migración DB init
│   │
│   └── (otros servicios Fase 2/3...)
│
└── secrets/
    ├── .sops.yaml                      ← Reglas SOPS: qué cifrar y con qué clave age
    ├── iam-service-secrets.yaml        ← Secret cifrado con SOPS+age (seguro en Git)
    ├── kratos-secrets.yaml             ← Secret cifrado con SOPS+age
    ├── cnpg-serenamente-admin-creds.yaml ← Secret cifrado con SOPS+age
    ├── cnpg-b2-credentials.yaml        ← Secret cifrado con SOPS+age
    └── gitlab-registry-secret.yaml     ← dockerconfigjson cifrado con SOPS+age
```

---

## 12. Tarea V-07 — cert-manager v1.x + Let's Encrypt

### 12.1 HelmRepository y HelmRelease de cert-manager

```yaml
# infra/infrastructure/cert-manager/helmrepository.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: jetstack
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.jetstack.io
---
# infra/infrastructure/cert-manager/helmrelease.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: cert-manager
  namespace: cert-manager
spec:
  interval: 1h
  chart:
    spec:
      chart: cert-manager
      version: ">=1.15.0 <2.0.0"   # Versión estable v1.x
      sourceRef:
        kind: HelmRepository
        name: jetstack
        namespace: flux-system
  values:
    # Instalar los CRDs como parte del Helm chart (recomendado para FluxCD)
    crds:
      enabled: true
    # Habilitar el webhook de validación
    webhook:
      enabled: true
    # Configurar recursos
    resources:
      requests:
        cpu: 20m
        memory: 64Mi
      limits:
        cpu: 200m
        memory: 128Mi
    # Prometheus metrics
    prometheus:
      enabled: true
      servicemonitor:
        enabled: false   # Habilitar en Fase 3 cuando exista el PLG stack
```

### 12.2 ClusterIssuers: staging y producción

**Es crítico empezar con el issuer de staging** para verificar que todo funciona antes de usar el issuer de producción. Let's Encrypt tiene rate limits estrictos en producción: 50 certificados por dominio registrado por semana. Un error de configuración repetido puede agotar este límite.

```yaml
# infra/infrastructure/cert-manager/clusterissuers.yaml
---
# STAGING — usar primero para verificar configuración sin consumir rate limits
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: admin@serenamente.com   # Ajustar con email real
    privateKeySecretRef:
      name: letsencrypt-staging-key
    solvers:
      - http01:
          ingress:
            class: traefik
---
# PRODUCCIÓN — usar solo después de verificar con staging
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@serenamente.com   # Ajustar con email real
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: traefik
```

### 12.3 Verificar cert-manager después del deploy

```bash
# Verificar pods:
kubectl get pods -n cert-manager
# → cert-manager-xxx                   Running
# → cert-manager-cainjector-xxx        Running
# → cert-manager-webhook-xxx           Running

# Verificar ClusterIssuers:
kubectl get clusterissuers
# → NAME                  READY   AGE
# → letsencrypt-staging   True    2m
# → letsencrypt-prod      True    2m

# Test completo — crear un certificado de prueba con staging:
kubectl apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: test-certificate
  namespace: serenamente-core
spec:
  secretName: test-tls-secret
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  dnsNames:
    - api.serenamente.com
EOF

# Monitorear el estado (puede tardar 1-3 minutos):
kubectl describe certificate test-certificate -n serenamente-core
# Buscar: Status: True, Reason: Ready

# Cuando esté Ready, eliminar el test y cambiar a producción:
kubectl delete certificate test-certificate -n serenamente-core
```

---

## 13. Tarea V-08 — Traefik v3.x (Ingress Controller)

### 13.1 HelmRelease de Traefik

```yaml
# infra/infrastructure/traefik/helmrepository.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: traefik
  namespace: flux-system
spec:
  interval: 1h
  url: https://helm.traefik.io/traefik
---
# infra/infrastructure/traefik/helmrelease.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: traefik
  namespace: traefik
spec:
  interval: 1h
  dependsOn:
    - name: cert-manager
      namespace: cert-manager
  chart:
    spec:
      chart: traefik
      version: ">=30.0.0 <31.0.0"   # Traefik v3.x chart
      sourceRef:
        kind: HelmRepository
        name: traefik
        namespace: flux-system
  values:
    # Deployment como DaemonSet en single-node — garantiza que usa hostPort
    deployment:
      kind: DaemonSet

    # Configuración de puertos
    ports:
      web:
        port: 8000
        expose:
          default: true
        exposedPort: 80
        hostPort: 80    # Exponer directamente en el host (VPS) en :80
        protocol: TCP
        # Redirigir HTTP a HTTPS automáticamente
        redirectTo:
          port: websecure
      websecure:
        port: 8443
        expose:
          default: true
        exposedPort: 443
        hostPort: 443   # Exponer directamente en el host (VPS) en :443
        protocol: TCP
        tls:
          enabled: true

    # Service type: ClusterIP (el tráfico llega via hostPort del DaemonSet)
    service:
      type: ClusterIP

    # No usar LoadBalancer — el Hetzner CCM lo crearía innecesariamente (costo extra)
    # En single-node con DaemonSet + hostPort, el LoadBalancer no es necesario.

    # IngressClass — debe ser el default para que cert-manager ACME HTTP-01 funcione
    ingressClass:
      enabled: true
      isDefaultClass: true

    # Configuración de providers
    providers:
      kubernetesCRD:
        enabled: true
        allowCrossNamespace: true    # Permite IngressRoutes en cualquier namespace
        allowExternalNameServices: true
      kubernetesIngress:
        enabled: true
        allowExternalNameServices: true

    # Logs
    logs:
      general:
        level: INFO
      access:
        enabled: true

    # Métricas para Prometheus (Fase 3)
    metrics:
      prometheus:
        entryPoint: metrics

    # Habilitar API/Dashboard (acceso interno solamente)
    api:
      dashboard: true
      insecure: false  # Solo via IngressRoute con auth

    # Configuración adicional de seguridad
    additionalArguments:
      - "--entrypoints.websecure.http.tls.certResolver=letsencrypt"
      - "--certificatesresolvers.letsencrypt.acme.tlschallenge=false"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.letsencrypt.acme.email=admin@serenamente.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/data/acme.json"

    # Persistencia para almacenar los certificados ACME
    persistence:
      enabled: true
      name: data
      storageClass: local-path
      size: 128Mi

    # Recursos
    resources:
      requests:
        cpu: 50m
        memory: 64Mi
      limits:
        cpu: 300m
        memory: 256Mi
```

### 13.2 ForwardAuth Middleware para IAM

```yaml
# infra/infrastructure/traefik/middlewares.yaml
---
# Middleware ForwardAuth — delega autenticación al IAM Domain Service
# TODOS los endpoints protegidos deben referenciar este middleware en su IngressRoute
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: iam-forward-auth
  namespace: traefik
spec:
  forwardAuth:
    address: "http://iam-service.serenamente-core.svc.cluster.local:8080/internal/validate-token"
    trustForwardHeader: true
    authResponseHeaders:
      - "X-User-ID"
      - "X-User-Role"
      - "X-Tenant-ID"
      - "X-User-DID"
    # Timeout de validación — debe ser inferior al timeout de la request del usuario
    authRequestHeaders:
      - "Authorization"
      - "Cookie"
---
# Middleware de redireccionamiento HTTP → HTTPS
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect-to-https
  namespace: traefik
spec:
  redirectScheme:
    scheme: https
    permanent: true
---
# Middleware de headers de seguridad (HSTS, etc.)
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: security-headers
  namespace: traefik
spec:
  headers:
    frameDeny: true
    contentTypeNosniff: true
    browserXssFilter: true
    referrerPolicy: "strict-origin-when-cross-origin"
    customResponseHeaders:
      X-Powered-By: ""   # Ocultar información del servidor
    # HSTS — forzar HTTPS durante 1 año
    stsSeconds: 31536000
    stsIncludeSubdomains: true
    stsPreload: true
```

### 13.3 Verificar Traefik

```bash
# Verificar pod de Traefik:
kubectl get pods -n traefik
# → traefik-xxxx   Running

# Verificar que los puertos hostPort están escuchando en el VPS:
# (ejecutar desde tu máquina de trabajo)
curl -I http://${FLOATING_IP}
# → HTTP/1.1 308 Permanent Redirect   ← redirige HTTP a HTTPS ✓

curl -k -I https://${FLOATING_IP}
# → HTTP/1.1 404 Not Found   ← Traefik responde (404 porque no hay IngressRoutes aún) ✓

# Verificar IngressClass:
kubectl get ingressclass
# → NAME      CONTROLLER              PARAMETERS   AGE
# → traefik   traefik.io/ingress-lb   <none>       5m

---

## 14. Tarea V-09 — SOPS + age (cifrado de secretos GitOps)

SOPS (Secrets OPerationS) cifra los valores de los Kubernetes Secrets directamente en Git usando una clave `age` gestionada por el operador. FluxCD descifra los secrets en tiempo de reconciliación vía `spec.decryption.provider: sops` — sin ningún controller adicional en el cluster.

**Ventajas sobre Sealed Secrets:**
- Zero controllers adicionales (el descifrado ocurre dentro del kustomize-controller existente).
- La misma clave `age` funciona en cualquier cluster — no hay que re-cifrar al cambiar de entorno.
- El cifrado es client-side: los secrets nunca viajan en texto plano al cluster durante la creación.
- Rotación de clave explícita y controlada (no automática e inesperada).

> **Prerrequisito:** Los pasos 1-4 de la sección 10.3 deben estar completados antes de continuar.

### 14.1 Script auxiliar para cifrar secrets con SOPS

```bash
#!/bin/bash
# scripts/encrypt-secret.sh
# Uso: ./scripts/encrypt-secret.sh <namespace> <secret-name> <key>=<value> [<key>=<value> ...]
# Ejemplo: ./scripts/encrypt-secret.sh serenamente-core iam-jwt-secret JWT_PRIVATE_KEY="$(cat key.pem)"
#
# Genera un Kubernetes Secret cifrado con SOPS+age listo para commitear en Git.
# FluxCD lo descifrará automáticamente durante la reconciliación.

NAMESPACE=$1
SECRET_NAME=$2
shift 2

OUTPUT_FILE="infra/secrets/${SECRET_NAME}.yaml"
SOPS_CONFIG="infra/secrets/.sops.yaml"

if [ ! -f "${SOPS_CONFIG}" ]; then
  echo "ERROR: Reglas SOPS no encontradas: ${SOPS_CONFIG}"
  echo "Ejecutar primero la sección 10.3 para configurar SOPS+age."
  exit 1
fi

# Construir el Secret de Kubernetes en memoria y cifrarlo con SOPS:
kubectl create secret generic "${SECRET_NAME}" \
  --namespace="${NAMESPACE}" \
  $(printf -- "--from-literal=%s " "$@") \
  --dry-run=client \
  -o yaml \
| sops \
  --config "${SOPS_CONFIG}" \
  --encrypt \
  --input-type yaml \
  --output-type yaml \
  /dev/stdin \
  > "${OUTPUT_FILE}"

echo "✓ Secret cifrado creado: ${OUTPUT_FILE}"
echo "→ Agregar al repo: git add ${OUTPUT_FILE} && git commit && git push"
```

```bash
chmod +x scripts/encrypt-secret.sh
```

### 14.2 Kustomization de FluxCD para secrets — incluir el directorio

Para que FluxCD aplique los secrets cifrados, el directorio `infra/secrets/` debe estar referenciado en la Kustomization del cluster o en cada componente de infraestructura/app que los necesite. El patrón recomendado es incluirlos desde cada componente:

```yaml
# Ejemplo: infra/infrastructure/cnpg/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - helmrepository.yaml
  - helmrelease.yaml
  - cluster.yaml
  - ../../../secrets/cnpg-serenamente-admin-creds.yaml  # Secret cifrado SOPS
  - ../../../secrets/cnpg-b2-credentials.yaml            # Secret cifrado SOPS
```

```yaml
# Ejemplo: infra/apps/iam-service/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - ingressroute.yaml
  - ../../secrets/iam-service-secrets.yaml      # JWT keys cifradas con SOPS
  - ../../secrets/gitlab-registry-secret.yaml   # imagePullSecret cifrado con SOPS
```

### 14.3 Verificar que FluxCD descifra correctamente

```bash
# Forzar reconciliación y verificar que los secrets se aplican:
flux reconcile kustomization flux-system --with-source
flux get kustomizations -A

# Verificar que los secrets están en el cluster:
kubectl get secrets -n serenamente-core
# → NAME                     TYPE                             DATA
# → iam-service-secrets       Opaque                           2
# → gitlab-registry-secret    kubernetes.io/dockerconfigjson   1

kubectl get secrets -n serenamente-data
# → NAME                              TYPE     DATA
# → cnpg-serenamente-admin-creds      Opaque   2
# → cnpg-b2-credentials               Opaque   2

# Si hay errores de decryption en FluxCD:
kubectl logs deployment/kustomize-controller -n flux-system | grep -i "sops\|decrypt\|age"
# → Buscar: "decryption failed" para diagnóstico
# → Verificar que el secret sops-age existe: kubectl get secret sops-age -n flux-system
```

---

## 15. Tarea V-10 — CloudNativePG v1.x + PostgreSQL 17.4

### 15.1 HelmRelease del operador CloudNativePG

```yaml
# infra/infrastructure/cnpg/helmrepository.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: cnpg
  namespace: flux-system
spec:
  interval: 1h
  url: https://cloudnative-pg.github.io/charts
---
# infra/infrastructure/cnpg/helmrelease.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: cloudnative-pg
  namespace: cnpg-system
spec:
  interval: 1h
  chart:
    spec:
      chart: cloudnative-pg
      version: ">=0.22.0"   # Versión del chart que instala el operator v1.x
      sourceRef:
        kind: HelmRepository
        name: cnpg
        namespace: flux-system
  values:
    # Habilitar métricas para Prometheus (Fase 3)
    monitoring:
      podMonitorEnabled: false   # Habilitar en Fase 3
    resources:
      requests:
        cpu: 20m
        memory: 64Mi
      limits:
        cpu: 200m
        memory: 128Mi
```

### 15.2 Cluster CRD — PostgreSQL 17.4 con 6 databases

```yaml
# infra/infrastructure/cnpg/cluster.yaml
# Un solo Cluster CRD gestiona la instancia de PostgreSQL 17.4 con 6 databases aisladas.
# Ref: ADR-013, plans/06_talos_k8s_decision_y_cambios_en_cadena.md
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: serenamente-pg
  namespace: serenamente-data
spec:
  # PostgreSQL 17.4 — imagen oficial de CloudNativePG
  imageName: ghcr.io/cloudnative-pg/postgresql:17.4

  # Single-node en Fase 1 — 1 instancia primaria, sin réplicas
  # En Fase 4+: aumentar a 2 (primaria + 1 réplica standby) sin downtime
  instances: 1

  # Configuración de PostgreSQL 17.4
  postgresql:
    parameters:
      # Memoria — calibrado para 8 GB RAM del CX32
      shared_buffers: "512MB"           # 25% de la RAM disponible para PG
      effective_cache_size: "1536MB"    # ~75% de RAM para estimación del planner
      maintenance_work_mem: "128MB"     # Para VACUUM, CREATE INDEX
      work_mem: "16MB"                  # Por operación de sort/hash (cuidado: por conexión)
      max_connections: "100"            # Suficiente para Fase 1-3 via pgxpool
      wal_buffers: "16MB"
      checkpoint_completion_target: "0.9"
      random_page_cost: "1.1"           # SSD NVMe — valor bajo para preferir index scans
      effective_io_concurrency: "200"   # SSD NVMe — lecturas paralelas
      min_wal_size: "256MB"
      max_wal_size: "2GB"

      # Configuración de logging
      log_destination: "stderr"
      log_line_prefix: "%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h "
      log_min_duration_statement: "1000"   # Log queries >1s (diagnóstico)
      log_checkpoints: "on"
      log_connections: "off"
      log_disconnections: "off"
      log_lock_waits: "on"
      log_temp_files: "0"

      # SSL
      ssl: "on"
      ssl_cert_file: "/var/run/secrets/postgres-cert/tls.crt"
      ssl_key_file: "/var/run/secrets/postgres-cert/tls.key"

      # UUIDv7 — disponible nativamente en PG17
      # No se necesita extensión adicional: uuid_generate_v7() está disponible

    # Extensiones a instalar en todas las databases
    shared_preload_libraries:
      - "pg_stat_statements"   # Análisis de performance de queries

  # Bootstrap — crear las 6 databases iniciales
  bootstrap:
    initdb:
      database: serenamente_init   # DB temporal para el bootstrap
      owner: serenamente_admin
      secret:
        name: cnpg-serenamente-admin-creds
      # Las databases reales se crean via golang-migrate en los init containers
      postInitSQL:
        # IAM databases
        - "CREATE DATABASE iam_db WITH OWNER serenamente_admin ENCODING 'UTF8' LC_COLLATE 'C' LC_CTYPE 'C' TEMPLATE template0;"
        - "CREATE DATABASE kratos_db WITH OWNER serenamente_admin ENCODING 'UTF8' LC_COLLATE 'C' LC_CTYPE 'C' TEMPLATE template0;"
        - "CREATE DATABASE openfga_db WITH OWNER serenamente_admin ENCODING 'UTF8' LC_COLLATE 'C' LC_CTYPE 'C' TEMPLATE template0;"
        # Domain databases
        - "CREATE DATABASE scheduling_db WITH OWNER serenamente_admin ENCODING 'UTF8' LC_COLLATE 'C' LC_CTYPE 'C' TEMPLATE template0;"
        - "CREATE DATABASE clinical_db WITH OWNER serenamente_admin ENCODING 'UTF8' LC_COLLATE 'C' LC_CTYPE 'C' TEMPLATE template0;"
        - "CREATE DATABASE billing_db WITH OWNER serenamente_admin ENCODING 'UTF8' LC_COLLATE 'C' LC_CTYPE 'C' TEMPLATE template0;"
        # Extensiones en las databases relevantes
        - "\\c iam_db; CREATE EXTENSION IF NOT EXISTS pg_stat_statements; CREATE EXTENSION IF NOT EXISTS btree_gist;"
        - "\\c scheduling_db; CREATE EXTENSION IF NOT EXISTS pg_stat_statements; CREATE EXTENSION IF NOT EXISTS btree_gist;"
        - "\\c clinical_db; CREATE EXTENSION IF NOT EXISTS pg_stat_statements;"
        - "\\c billing_db; CREATE EXTENSION IF NOT EXISTS pg_stat_statements;"

  # Recursos del pod de PostgreSQL
  resources:
    requests:
      memory: "256Mi"
      cpu: "100m"
    limits:
      memory: "512Mi"
      cpu: "1000m"

  # Storage — PVC local en el NVMe del CX32
  storage:
    size: 20Gi
    storageClass: local-path   # StorageClass incluida en Talos (Rancher local-path-provisioner)

  # WAL archiving — backup continuo a Backblaze B2
  # Se configura en Tarea V-17 — backup habilitado solo después de crear los secrets B2
  backup:
    barmanObjectStore:
      destinationPath: "s3://serenamente-pg-backups/wal"
      endpointURL: "https://s3.us-west-004.backblazeb2.com"   # Ajustar por región B2
      s3Credentials:
        accessKeyId:
          name: cnpg-b2-credentials
          key: B2_KEY_ID
        secretAccessKey:
          name: cnpg-b2-credentials
          key: B2_APPLICATION_KEY
      wal:
        compression: gzip
        maxParallel: 2
      data:
        compression: gzip
        immediateCheckpoint: false
        jobs: 2
    retentionPolicy: "30d"

  # Monitoreo (Fase 3)
  monitoring:
    enablePodMonitor: false   # Habilitar en Fase 3
```

### 15.3 Crear el Secret de credenciales admin de PostgreSQL

```bash
# Crear el Secret para las credenciales del admin de PostgreSQL.
# CloudNativePG crea secretos adicionales con credenciales para cada app.

# Generar una contraseña segura:
PG_ADMIN_PASSWORD=$(openssl rand -base64 32)

# Cifrar y guardar con SOPS+age:
./scripts/encrypt-secret.sh serenamente-data cnpg-serenamente-admin-creds \
  "username=serenamente_admin" \
  "password=${PG_ADMIN_PASSWORD}"

# BACKUP CRÍTICO: guardar la contraseña en el password manager antes de continuar:
echo "PG_ADMIN_PASSWORD=${PG_ADMIN_PASSWORD}"

git add infra/secrets/cnpg-serenamente-admin-creds.yaml
git commit -m "feat: add cnpg admin credentials encrypted with SOPS"
git push
```

### 15.4 Kustomization del componente CNPG

```yaml
# infra/infrastructure/cnpg/kustomization.yaml (crear si no existe):
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - helmrepository.yaml
  - helmrelease.yaml
  - cluster.yaml
  - ../../../secrets/cnpg-serenamente-admin-creds.yaml  # SOPS-encrypted
  - ../../../secrets/cnpg-b2-credentials.yaml            # SOPS-encrypted (se crea en V-17)
```

### 15.5 Verificar el cluster de PostgreSQL

```bash
# Esperar a que el Cluster CRD esté Ready (~3-5 minutos en CX32):
kubectl wait cluster serenamente-pg \
  --for=condition=Ready \
  -n serenamente-data \
  --timeout=300s

# Verificar estado del cluster:
kubectl get cluster serenamente-pg -n serenamente-data
# → NAME              AGE   INSTANCES   READY   STATUS         PRIMARY
# → serenamente-pg    5m    1           1       Cluster in healthy state   serenamente-pg-1

# Ver los secrets generados por CloudNativePG (uno por database):
kubectl get secrets -n serenamente-data | grep cnpg
# → serenamente-pg-app                kubernetes.io/basic-auth  ...
# → serenamente-pg-superuser          kubernetes.io/basic-auth  ...
# → serenamente-pg-ca                 Opaque                    ...
# → serenamente-pg-replication        kubernetes.io/basic-auth  ...

# Verificar que las 6 databases existen:
kubectl exec -it serenamente-pg-1 -n serenamente-data -- \
  psql -U serenamente_admin -c "\l"
# → iam_db, kratos_db, openfga_db, scheduling_db, clinical_db, billing_db

# Verificar extensiones:
kubectl exec -it serenamente-pg-1 -n serenamente-data -- \
  psql -U serenamente_admin -d iam_db -c "\dx"
# → btree_gist, pg_stat_statements
```

---

## 16. Tarea V-11 — DNS y Cloudflare (dominios públicos)

### 16.1 Configurar registros DNS en Cloudflare

```bash
source .envrc

# Registros A para el dominio principal:
# api.serenamente.com → Floating IP del VPS
# El proxy de Cloudflare (naranja ☁) está habilitado para protección DDoS.

# Crear registro A para api.serenamente.com:
curl -s -X POST "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/dns_records" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "A",
    "name": "api",
    "content": "'"${FLOATING_IP}"'",
    "ttl": 1,
    "proxied": true,
    "comment": "Serenamente API — Traefik VPS"
  }' | jq '.result.id'

# Verificar que el DNS propaga (puede tardar 1-5 minutos con Cloudflare Proxy):
dig api.serenamente.com @1.1.1.1
# → api.serenamente.com.  300  IN  A  104.x.x.x  (IP de Cloudflare, no la del VPS — ✓ proxy activo)
```

> **Sobre Cloudflare Proxy vs DNS-only:** Con el proxy activo (ícono naranja ☁), la IP real del VPS está oculta detrás de los servidores de Cloudflare. Esto proporciona protección DDoS básica y mejora la distribución geográfica. El tráfico llega al VPS desde las IPs de Cloudflare, no desde las IPs de los clientes. **Esto NO interfiere con Let's Encrypt ACME HTTP-01** porque cert-manager hace el challenge desde dentro del cluster.

### 16.2 Habilitar HSTS en Cloudflare

```bash
# Configurar SSL/TLS en Cloudflare para modo "Full (strict)":
# Esto garantiza que Cloudflare verifique el certificado del VPS (Let's Encrypt)

curl -s -X PATCH \
  "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/settings/ssl" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: application/json" \
  --data '{"value":"strict"}' | jq '.result'

# Habilitar HSTS en Cloudflare (capa adicional de seguridad):
curl -s -X PATCH \
  "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/settings/security_header" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: application/json" \
  --data '{
    "value": {
      "strict_transport_security": {
        "enabled": true,
        "max_age": 31536000,
        "include_subdomains": true,
        "preload": true,
        "nosniff": true
      }
    }
  }' | jq '.result'
```

---

## 17. Tarea V-12 — Schemas Protobuf + buf.build CLI

### 17.1 Inicializar el workspace de buf

Los schemas Protobuf son el contrato entre microservicios. Deben estar en el monorepo y ser la fuente de verdad de todos los eventos que circulan por NATS JetStream.

```bash
# Crear el workspace de buf:
cd packages/events

# Inicializar buf.yaml:
buf config init
# Crea: buf.yaml

# Editar buf.yaml:
cat > buf.yaml <<'EOF'
version: v2
modules:
  - path: proto
    name: buf.build/serenamente/events
lint:
  use:
    - STANDARD
breaking:
  use:
    - FILE
EOF

# Crear buf.gen.yaml para generación de código Go:
cat > buf.gen.yaml <<'EOF'
version: v2
managed:
  enabled: true
  override:
    - file_option: go_package_prefix
      value: gitlab.com/serenamente/serenidad-platform/packages/events/gen
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen/go
    opt:
      - paths=source_relative
  - remote: buf.build/grpc/go
    out: gen/go
    opt:
      - paths=source_relative
      - require_unimplemented_servers=false
EOF
```

### 17.2 Definir los schemas de eventos de Fase 1

```protobuf
// packages/events/proto/iam/v1/events.proto
syntax = "proto3";
package iam.v1;
option go_package = "gitlab.com/serenamente/serenidad-platform/packages/events/gen/iam/v1;iamv1";

import "google/protobuf/timestamp.proto";

// UserRegistered — emitido cuando un nuevo usuario completa el registro
message UserRegistered {
  string event_id = 1;          // UUIDv7 — identificador único del evento
  string user_id = 2;           // UUIDv7 — identificador del usuario
  string tenant_id = 3;         // UUIDv7 — organización del usuario
  string email = 4;             // Email del usuario (hash SHA-256 para privacidad en logs)
  string role = 5;              // "patient" | "doctor" | "admin"
  string did = 6;               // Decentralized Identifier: did:web:serenamente.com:users:UUID
  google.protobuf.Timestamp occurred_at = 7;   // Timestamp UTC RFC 3339
  google.protobuf.Timestamp emitted_at = 8;    // Timestamp de emisión por el Outbox Worker
}

// DoctorOnboarded — emitido cuando un médico completa su perfil profesional
message DoctorOnboarded {
  string event_id = 1;
  string doctor_id = 2;         // UUIDv7
  string tenant_id = 3;         // UUIDv7
  string specialty = 4;         // "psychiatry" | "psychology" | "general"
  string license_number = 5;    // Número de licencia médica
  string country_code = 6;      // ISO 3166-1 alpha-2
  google.protobuf.Timestamp occurred_at = 7;
  google.protobuf.Timestamp emitted_at = 8;
}
```

```protobuf
// packages/events/proto/scheduling/v1/events.proto
syntax = "proto3";
package scheduling.v1;
option go_package = "gitlab.com/serenamente/serenidad-platform/packages/events/gen/scheduling/v1;schedulingv1";

import "google/protobuf/timestamp.proto";

// AppointmentBooked — emitido cuando una cita médica es reservada exitosamente
message AppointmentBooked {
  string event_id = 1;
  string appointment_id = 2;    // UUIDv7
  string patient_id = 3;        // UUIDv7
  string doctor_id = 4;         // UUIDv7
  string tenant_id = 5;         // UUIDv7
  google.protobuf.Timestamp starts_at = 6;    // Inicio de la cita (UTC)
  google.protobuf.Timestamp ends_at = 7;      // Fin de la cita (UTC)
  string modality = 8;          // "video" | "phone" | "in_person"
  string fhir_resource_json = 9; // FHIR R4 Appointment resource serializado
  google.protobuf.Timestamp occurred_at = 10;
  google.protobuf.Timestamp emitted_at = 11;
}

// AppointmentCancelled — emitido cuando una cita es cancelada
message AppointmentCancelled {
  string event_id = 1;
  string appointment_id = 2;
  string cancelled_by_id = 3;   // patient_id o doctor_id o admin_id
  string reason = 4;            // Razón de cancelación (libre)
  google.protobuf.Timestamp occurred_at = 5;
  google.protobuf.Timestamp emitted_at = 6;
}
```

```protobuf
// packages/events/proto/clinical/v1/events.proto
syntax = "proto3";
package clinical.v1;
option go_package = "gitlab.com/serenamente/serenidad-platform/packages/events/gen/clinical/v1;clinicalv1";

import "google/protobuf/timestamp.proto";

// ConsultationFinished — emitido cuando una consulta médica finaliza
// Desencadena la creación de la factura en Billing Service
message ConsultationFinished {
  string event_id = 1;
  string consultation_id = 2;   // UUIDv7
  string appointment_id = 3;    // UUIDv7 — referencia a la cita
  string patient_id = 4;        // UUIDv7
  string doctor_id = 5;         // UUIDv7
  string tenant_id = 6;         // UUIDv7
  string openehr_json = 7;      // openEHR Canonical JSON de la consulta
  int32 duration_minutes = 8;   // Duración real de la consulta
  google.protobuf.Timestamp occurred_at = 9;
  google.protobuf.Timestamp emitted_at = 10;
}

// DiagnosisRecorded — emitido cuando un diagnóstico es agregado al EHR
message DiagnosisRecorded {
  string event_id = 1;
  string diagnosis_id = 2;      // UUIDv7
  string clinical_record_id = 3; // UUIDv7
  string patient_id = 4;
  string doctor_id = 5;
  string tenant_id = 6;
  string icd10_code = 7;        // Código ICD-10 del diagnóstico
  string description = 8;       // Descripción clínica
  google.protobuf.Timestamp occurred_at = 9;
  google.protobuf.Timestamp emitted_at = 10;
}
```

### 17.3 Generar código Go desde los schemas

```bash
cd packages/events

# Lint para verificar la calidad de los schemas:
buf lint
# → (sin salida = sin errores)

# Generar código Go:
buf generate
# → Genera: gen/go/iam/v1/*.pb.go
# → Genera: gen/go/scheduling/v1/*.pb.go
# → Genera: gen/go/clinical/v1/*.pb.go

# Verificar que el código generado compila:
cd gen/go
go mod init gitlab.com/serenamente/serenidad-platform/packages/events/gen
go mod tidy
go build ./...

# Verificar breaking changes (útil en CI para detectar incompatibilidades):
buf breaking --against "https://gitlab.com/serenamente/serenidad-platform.git#branch=main,subdir=packages/events/proto"
# En el primer commit esto no ejecuta nada (sin historia de comparación)

# Commitear los schemas y el código generado:
cd ../../..  # Volver a la raíz del monorepo
git add packages/events/
git commit -m "feat: add protobuf event schemas for IAM, Scheduling, and Clinical domains (Phase 1)"
git push
```

---

## 18. Tarea V-13 — Ory Kratos v1.3.1 (AuthN)

### 18.1 HelmRepository y HelmRelease de Kratos

```yaml
# infra/infrastructure/kratos/helmrepository.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: ory
  namespace: flux-system
spec:
  interval: 1h
  url: https://k8s.ory.sh/helm/charts
---
# infra/apps/kratos/helmrelease.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: kratos
  namespace: serenamente-core
spec:
  interval: 1h
  dependsOn:
    - name: cloudnative-pg
      namespace: cnpg-system
    - name: traefik
      namespace: traefik
  chart:
    spec:
      chart: kratos
      version: "0.50.x"   # Versión del chart para Kratos v1.3.1
      sourceRef:
        kind: HelmRepository
        name: ory
        namespace: flux-system
  values:
    kratos:
      config:
        dsn: "postgres://serenamente_admin:${PG_ADMIN_PASSWORD}@serenamente-pg-rw.serenamente-data.svc.cluster.local:5432/kratos_db?sslmode=require"

        serve:
          public:
            base_url: "https://api.serenamente.com/auth/"
            cors:
              enabled: true
              allowed_origins:
                - "https://app.serenamente.com"
              allowed_methods:
                - GET
                - POST
                - PUT
                - PATCH
                - DELETE
              allowed_headers:
                - Authorization
                - Content-Type
                - X-Session-Token
              exposed_headers:
                - Content-Type
                - Set-Cookie
          admin:
            base_url: "http://kratos-admin.serenamente-core.svc.cluster.local:4434/"

        selfservice:
          default_browser_return_url: "https://app.serenamente.com/"
          allowed_return_urls:
            - "https://app.serenamente.com"
            - "https://api.serenamente.com"

          flows:
            login:
              ui_url: "https://app.serenamente.com/login"
              lifespan: "1h"
            registration:
              ui_url: "https://app.serenamente.com/register"
              lifespan: "30m"
            verification:
              enabled: true
              ui_url: "https://app.serenamente.com/verification"
              use: "code"  # Código de 6 dígitos via email
              after:
                default_browser_return_url: "https://app.serenamente.com/dashboard"
            recovery:
              enabled: true
              ui_url: "https://app.serenamente.com/recovery"
              use: "code"
            settings:
              ui_url: "https://app.serenamente.com/settings"
              privileged_session_max_age: "15m"
              after:
                default_browser_return_url: "https://app.serenamente.com/dashboard"
            logout:
              after:
                default_browser_return_url: "https://app.serenamente.com/"

          methods:
            passkey:
              enabled: true
              config:
                rp:
                  display_name: "Serenamente — Clínica Digital"
                  id: "serenamente.com"
                  origins:
                    - "https://app.serenamente.com"
            webauthn:
              enabled: false   # Redundante si Passkeys está habilitado
            password:
              enabled: false   # CERO contraseñas — solo Passkeys
            code:
              enabled: true    # Para recovery y verification

        identity:
          default_schema_id: patient
          schemas:
            - id: patient
              url: "base64://$(cat infra/apps/kratos/schemas/patient.json | base64 -w 0)"
            - id: doctor
              url: "base64://$(cat infra/apps/kratos/schemas/doctor.json | base64 -w 0)"

        courier:
          smtp:
            connection_uri: "smtps://resend:${RESEND_API_KEY}@smtp.resend.com:465/"
            from_address: "noreply@serenamente.com"
            from_name: "Serenamente"

        secrets:
          cookie:
            - "${KRATOS_COOKIE_SECRET}"
          cipher:
            - "${KRATOS_CIPHER_SECRET}"

      identitySchemas:
        # Los schemas se montan como ConfigMap — definidos en configmap-kratos.yaml

    deployment:
      resources:
        requests:
          memory: "128Mi"
          cpu: "50m"
        limits:
          memory: "256Mi"
          cpu: "500m"
```

### 18.2 Identity Schemas (patient y doctor)

```json
// infra/apps/kratos/schemas/patient.json
{
  "$id": "https://serenamente.com/schemas/identity/patient.json",
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
          "title": "Email",
          "minLength": 3,
          "ory.sh/kratos": {
            "credentials": {
              "password": {"identifier": true},
              "code": {"identifier": true, "via": "email"},
              "passkey": {"display_name": "email"}
            },
            "recovery": {"via": "email"},
            "verification": {"via": "email"}
          }
        },
        "name": {
          "type": "object",
          "properties": {
            "first": {"type": "string", "title": "Nombre"},
            "last": {"type": "string", "title": "Apellido"}
          }
        },
        "birth_date": {
          "type": "string",
          "format": "date",
          "title": "Fecha de Nacimiento"
        },
        "phone": {
          "type": "string",
          "title": "Teléfono"
        },
        "preferred_language": {
          "type": "string",
          "enum": ["es", "en", "pt"],
          "default": "es"
        }
      },
      "required": ["email"],
      "additionalProperties": false
    }
  }
}
```

```json
// infra/apps/kratos/schemas/doctor.json
{
  "$id": "https://serenamente.com/schemas/identity/doctor.json",
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Doctor",
  "type": "object",
  "properties": {
    "traits": {
      "type": "object",
      "properties": {
        "email": {
          "type": "string",
          "format": "email",
          "title": "Email Profesional",
          "minLength": 3,
          "ory.sh/kratos": {
            "credentials": {
              "code": {"identifier": true, "via": "email"},
              "passkey": {"display_name": "email"}
            },
            "recovery": {"via": "email"},
            "verification": {"via": "email"}
          }
        },
        "name": {
          "type": "object",
          "properties": {
            "first": {"type": "string", "title": "Nombre"},
            "last": {"type": "string", "title": "Apellido"}
          },
          "required": ["first", "last"]
        },
        "medical_license": {
          "type": "string",
          "title": "Número de Licencia Médica"
        },
        "specialty": {
          "type": "string",
          "enum": ["psychiatry", "psychology", "general_medicine"],
          "title": "Especialidad"
        },
        "country_code": {
          "type": "string",
          "pattern": "^[A-Z]{2}$",
          "title": "País de ejercicio (ISO 3166-1 alpha-2)"
        }
      },
      "required": ["email", "name", "medical_license", "specialty", "country_code"],
      "additionalProperties": false
    }
  }
}
```

### 18.3 Crear los secrets de Kratos cifrados con SOPS

```bash
# Generar secrets de Kratos:
KRATOS_COOKIE_SECRET=$(openssl rand -hex 32)
KRATOS_CIPHER_SECRET=$(openssl rand -hex 32)

./scripts/encrypt-secret.sh serenamente-core kratos-secrets \
  "KRATOS_COOKIE_SECRET=${KRATOS_COOKIE_SECRET}" \
  "KRATOS_CIPHER_SECRET=${KRATOS_CIPHER_SECRET}" \
  "RESEND_API_KEY=${RESEND_API_KEY}"

git add infra/secrets/kratos-secrets.yaml
git commit -m "feat: add kratos secrets encrypted with SOPS"
git push
```

### 18.4 IngressRoute para Kratos

```yaml
# infra/apps/kratos/ingressroute.yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: kratos-public
  namespace: serenamente-core
spec:
  entryPoints:
    - websecure
  routes:
    - match: "Host(`api.serenamente.com`) && PathPrefix(`/auth/`)"
      kind: Rule
      services:
        - name: kratos-public
          port: 4433
      middlewares:
        - name: security-headers
          namespace: traefik
  tls:
    certResolver: letsencrypt
    domains:
      - main: api.serenamente.com
```

### 18.5 Verificar Kratos

```bash
# Verificar pods:
kubectl get pods -n serenamente-core | grep kratos
# → kratos-xxx   Running

# Verificar health de Kratos:
kubectl exec -n serenamente-core deployment/kratos -- \
  wget -qO- http://localhost:4434/health/ready
# → {"status":"ok"}

# Verificar desde el exterior:
curl https://api.serenamente.com/auth/.well-known/ory/webauthn.js
# → Debe retornar el JavaScript de WebAuthn de Kratos

# Verificar que la base de datos fue migrada:
kubectl exec -it serenamente-pg-1 -n serenamente-data -- \
  psql -U serenamente_admin -d kratos_db -c "\dt"
# → Debe listar las tablas de Kratos (identity, session, etc.)
```

---

## 19. Tarea V-14 — IAM Domain Service Go

### 19.1 Estructura del servicio IAM en Go

```bash
# Estructura del microservicio IAM:
services/iam/
├── cmd/
│   └── server/
│       └── main.go              ← Punto de entrada
├── internal/
│   ├── config/
│   │   └── config.go            ← Lectura de env vars con validación
│   ├── domain/
│   │   ├── user.go              ← Entidades del dominio
│   │   └── events.go            ← Tipos de eventos de dominio
│   ├── handlers/
│   │   ├── token.go             ← POST /api/iam/token/exchange
│   │   ├── validate.go          ← GET /internal/validate-token (ForwardAuth)
│   │   ├── health.go            ← GET /health (liveness) + /ready (readiness)
│   │   └── metrics.go           ← GET /metrics (Prometheus)
│   ├── repository/
│   │   ├── user_repo.go         ← CRUD de perfiles en iam_db
│   │   └── outbox_repo.go       ← Outbox pattern
│   ├── services/
│   │   ├── jwt_service.go       ← Firma y verificación JWT Ed25519
│   │   ├── kratos_client.go     ← Cliente HTTP Kratos Admin API :4434
│   │   └── outbox_worker.go     ← Worker que lee outbox y publica en NATS
│   └── db/
│       └── migrations/          ← SQL migrations (golang-migrate)
│           ├── 000001_create_users.up.sql
│           ├── 000001_create_users.down.sql
│           ├── 000002_create_outbox.up.sql
│           └── 000002_create_outbox.down.sql
├── go.mod
├── go.sum
├── Dockerfile
└── Makefile
```

### 19.2 Migración inicial de la base de datos IAM

```sql
-- services/iam/internal/db/migrations/000001_create_users.up.sql
-- Tabla de perfiles de usuario del dominio IAM
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE IF NOT EXISTS user_profiles (
  id            UUID        PRIMARY KEY DEFAULT gen_random_uuid(),   -- UUIDv7 en PG17
  kratos_id     UUID        NOT NULL UNIQUE,     -- ID de Ory Kratos (vincula sesión ↔ dominio)
  tenant_id     UUID        NOT NULL,            -- Organización del usuario
  role          TEXT        NOT NULL CHECK (role IN ('patient', 'doctor', 'admin')),
  email_hash    TEXT        NOT NULL,            -- SHA-256 del email (privacidad en logs)
  did           TEXT        NOT NULL UNIQUE,     -- did:web:serenamente.com:users:{uuid}
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_user_profiles_kratos_id ON user_profiles(kratos_id);
CREATE INDEX idx_user_profiles_tenant_id ON user_profiles(tenant_id);
CREATE INDEX idx_user_profiles_role_tenant ON user_profiles(role, tenant_id);

-- Row Level Security
ALTER TABLE user_profiles ENABLE ROW LEVEL SECURITY;

-- Política: cada usuario solo ve su propio perfil (excepto admins de su tenant)
CREATE POLICY user_own_profile ON user_profiles
  FOR SELECT
  USING (
    id = current_setting('app.user_id', true)::UUID
    OR (
      role = 'admin'
      AND tenant_id = current_setting('app.tenant_id', true)::UUID
    )
  );
```

```sql
-- services/iam/internal/db/migrations/000002_create_outbox.up.sql
-- Tabla de Outbox para garantía at-least-once con NATS JetStream
CREATE TABLE IF NOT EXISTS outbox_events (
  id            UUID        PRIMARY KEY DEFAULT gen_random_uuid(),   -- UUIDv7
  aggregate_id  UUID        NOT NULL,      -- ID del agregado que generó el evento
  aggregate_type TEXT       NOT NULL,      -- "user_profile"
  event_type    TEXT        NOT NULL,      -- "UserRegistered" | "DoctorOnboarded"
  payload       BYTEA       NOT NULL,      -- Protobuf serializado
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  published_at  TIMESTAMPTZ,              -- NULL = pendiente, NOT NULL = publicado
  retry_count   INT         NOT NULL DEFAULT 0,
  last_error    TEXT                       -- Último error de publicación
);

-- Índice para el worker de outbox (solo lee los pendientes):
CREATE INDEX idx_outbox_pending ON outbox_events(created_at)
  WHERE published_at IS NULL;

-- El outbox NO tiene RLS — es una tabla interna del servicio, no expuesta a clientes.
```

### 19.3 Deployment del IAM Domain Service

```yaml
# infra/apps/iam-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iam-service
  namespace: serenamente-core
  annotations:
    # FluxCD auto-update: actualiza la imagen cuando GitLab CI pushea nueva versión a registry.gitlab.com
    image.fluxcd.io/update-policy: semver
    image.fluxcd.io/update-pattern: "0.x.x"
spec:
  replicas: 1   # Single-node, single-pod en Fase 1
  selector:
    matchLabels:
      app: iam-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: iam-service
        version: "0.1.0"
    spec:
      # Init container: ejecutar migraciones ANTES de que el servicio arranque
      initContainers:
        - name: migrate
          image: registry.gitlab.com/serenamente/serenidad-platform/iam-service:latest
          command: ["/app/iam-service", "migrate"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: cnpg-iam-db-credentials
                  key: uri
          resources:
            requests:
              memory: "32Mi"
              cpu: "50m"
            limits:
              memory: "64Mi"
              cpu: "200m"

      containers:
        - name: iam-service
          image: registry.gitlab.com/serenamente/serenidad-platform/iam-service:latest   # {flux-image}
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          env:
            - name: PORT
              value: "8080"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: cnpg-iam-db-credentials
                  key: uri
            - name: KRATOS_ADMIN_URL
              value: "http://kratos-admin.serenamente-core.svc.cluster.local:4434"
            - name: JWT_PRIVATE_KEY_ED25519
              valueFrom:
                secretKeyRef:
                  name: iam-service-secrets
                  key: JWT_PRIVATE_KEY_ED25519
            - name: JWT_PUBLIC_KEY_ED25519
              valueFrom:
                secretKeyRef:
                  name: iam-service-secrets
                  key: JWT_PUBLIC_KEY_ED25519
            - name: JWT_ISSUER
              value: "https://api.serenamente.com"
            - name: JWT_AUDIENCE
              value: "serenamente-api"
            - name: JWT_TTL_SECONDS
              value: "3600"   # 1 hora
            # NATS — solo en Fase 2. En Fase 1 el Outbox Worker espera a NATS con backoff.
            - name: NATS_URL
              value: "nats://nats.serenamente-data.svc.cluster.local:4222"
            - name: ENVIRONMENT
              value: "production"

          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 10
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3

          resources:
            requests:
              memory: "40Mi"
              cpu: "20m"
            limits:
              memory: "128Mi"
              cpu: "500m"

          # Security context — ejecutar como usuario no-root
          securityContext:
            runAsNonRoot: true
            runAsUser: 65534     # nobody
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL

      # imagePullSecret para autenticarse al GitLab Container Registry (registry.gitlab.com)
      # El secret se crea en la Tarea V-14 usando el Deploy Token de GitLab
      imagePullSecrets:
        - name: gitlab-registry-secret

      # Garantizar que el pod NO se programa en el mismo nodo que PostgreSQL
      # (en single-node esto no aplica, pero es buena práctica para Fase 4+)
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 50
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - serenamente-pg
                topologyKey: kubernetes.io/hostname
---
apiVersion: v1
kind: Service
metadata:
  name: iam-service
  namespace: serenamente-core
spec:
  selector:
    app: iam-service
  ports:
    - name: http
      port: 8080
      targetPort: http
      protocol: TCP
  type: ClusterIP
```

### 19.4 IngressRoute del IAM Domain Service

```yaml
# infra/apps/iam-service/ingressroute.yaml
---
# Endpoint público: intercambio de token (sesión Kratos → JWT Ed25519)
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: iam-service-public
  namespace: serenamente-core
spec:
  entryPoints:
    - websecure
  routes:
    - match: "Host(`api.serenamente.com`) && PathPrefix(`/api/iam/`)"
      kind: Rule
      services:
        - name: iam-service
          port: 8080
      middlewares:
        - name: security-headers
          namespace: traefik
  tls:
    certResolver: letsencrypt
---
# Endpoint interno: validate-token (llamado por Traefik ForwardAuth)
# Este endpoint NO debe ser accesible desde el exterior — solo desde Traefik internamente.
# Se omite IngressRoute público. Traefik lo llama via cluster DNS directamente.
```

### 19.5 Crear los secrets del IAM Service cifrados con SOPS

```bash
# Generar par de claves Ed25519 para firma de JWT:
# La clave privada firma los JWT; la clave pública la usan todos los servicios para verificar.

# Generar clave privada:
openssl genpkey -algorithm ed25519 -out /tmp/jwt_private_key.pem
# Generar clave pública derivada:
openssl pkey -in /tmp/jwt_private_key.pem -pubout -out /tmp/jwt_public_key.pem

# Mostrar en base64 para el secret:
JWT_PRIVATE_KEY=$(cat /tmp/jwt_private_key.pem | base64 -w 0)
JWT_PUBLIC_KEY=$(cat /tmp/jwt_public_key.pem | base64 -w 0)

# BACKUP CRÍTICO — guardar en password manager:
echo "JWT Private Key (base64):"
echo "${JWT_PRIVATE_KEY}"
echo ""
echo "JWT Public Key (base64):"
echo "${JWT_PUBLIC_KEY}"

# Cifrar y guardar con SOPS+age:
./scripts/encrypt-secret.sh serenamente-core iam-service-secrets \
  "JWT_PRIVATE_KEY_ED25519=${JWT_PRIVATE_KEY}" \
  "JWT_PUBLIC_KEY_ED25519=${JWT_PUBLIC_KEY}"

git add infra/secrets/iam-service-secrets.yaml
git commit -m "feat: add IAM service JWT keys encrypted with SOPS"
git push

# Limpiar claves del filesystem local:
rm -f /tmp/jwt_private_key.pem /tmp/jwt_public_key.pem
```

### 19.6 Crear el imagePullSecret para GitLab Container Registry

El cluster Kubernetes necesita credenciales para descargar imágenes desde `registry.gitlab.com`. Se usa el **Deploy Token** creado en 4.1.1 (con scope `read_registry`).

```bash
# Generar el Secret de tipo docker-registry en memoria y cifrarlo con SOPS:
kubectl create secret docker-registry gitlab-registry-secret \
  --namespace serenamente-core \
  --docker-server=registry.gitlab.com \
  --docker-username="${GITLAB_DEPLOY_TOKEN_USER}" \
  --docker-password="${GITLAB_DEPLOY_TOKEN}" \
  --docker-email="ops@serenamente.com" \
  --dry-run=client -o yaml \
| sops \
  --config infra/secrets/.sops.yaml \
  --encrypt \
  --input-type yaml \
  --output-type yaml \
  /dev/stdin \
  > infra/secrets/gitlab-registry-secret.yaml

git add infra/secrets/gitlab-registry-secret.yaml
git commit -m "feat: add gitlab registry pull secret encrypted with SOPS"
git push

# IMPORTANTE: Si el Deploy Token expira, regenerar el token en GitLab
# (Settings → Repository → Deploy tokens) y repetir este bloque.
# FluxCD aplicará el nuevo secret automáticamente tras el push.
```

### 19.7 Dockerfile del IAM Domain Service

```dockerfile
# services/iam/Dockerfile
# Build en dos etapas: builder + runtime mínimo
FROM golang:1.25-alpine AS builder

WORKDIR /build

# Descargar dependencias (cache layer separado del código)
COPY go.mod go.sum ./
RUN go mod download

# Copiar código fuente:
COPY . .

# Compilar con flags de seguridad y optimización:
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build \
    -trimpath \
    -ldflags="-w -s -X main.version=$(git describe --tags --always 2>/dev/null || echo 'dev')" \
    -o /app/iam-service \
    ./cmd/server/

# Runtime mínimo: scratch o distroless
FROM gcr.io/distroless/static-debian12:nonroot

COPY --from=builder /app/iam-service /app/iam-service

# Copiar migraciones SQL:
COPY --from=builder /build/internal/db/migrations /app/migrations

# Puerto del servicio
EXPOSE 8080

# Usuario no-root (nonroot es UID 65532 en distroless)
USER nonroot:nonroot

ENTRYPOINT ["/app/iam-service"]
CMD ["serve"]
```

---

## 20. Tarea V-15 — BFF: Cloudflare Workers + Hono v4.x

### 20.1 Inicializar el proyecto del BFF

```bash
# Crear el proyecto del BFF con Hono + Wrangler:
cd apps/bff

# Inicializar con la plantilla de Cloudflare Workers:
bun create hono@latest . --template cloudflare-workers

# Instalar dependencias adicionales:
bun add hono@^4.0.0
bun add jose   # Para verificación de JWT Ed25519 en el edge

# Estructura del proyecto:
# apps/bff/
# ├── src/
# │   ├── index.ts          ← Router principal
# │   ├── middleware/
# │   │   ├── auth.ts       ← JWT verification middleware
# │   │   └── cors.ts       ← CORS configuration
# │   └── routes/
# │       ├── iam.ts        ← /api/iam/* proxy
# │       ├── scheduling.ts ← /api/scheduling/* proxy (Fase 2)
# │       └── clinical.ts   ← /api/clinical/* proxy (Fase 2)
# ├── wrangler.toml          ← Configuración de CF Workers
# ├── tsconfig.json
# └── package.json
```

### 20.2 Configuración de Wrangler

```toml
# apps/bff/wrangler.toml
name = "serenamente-bff"
main = "src/index.ts"
compatibility_date = "2026-04-01"
compatibility_flags = ["nodejs_compat"]

[vars]
BACKEND_URL = "https://api.serenamente.com"
ENVIRONMENT = "production"
JWT_ISSUER = "https://api.serenamente.com"
JWT_AUDIENCE = "serenamente-api"

# Los secrets (JWT_PUBLIC_KEY) se agregan via wrangler secret put — NO en wrangler.toml
# wrangler secret put JWT_PUBLIC_KEY_ED25519

[[routes]]
pattern = "api.serenamente.com/bff/*"
zone_name = "serenamente.com"
```

### 20.3 BFF principal con verificación de JWT

```typescript
// apps/bff/src/index.ts
import { Hono } from 'hono'
import { cors } from 'hono/cors'
import { secureHeaders } from 'hono/secure-headers'
import { HTTPException } from 'hono/http-exception'
import { verifyJWT, extractBearerToken } from './middleware/auth'

// Bindings de Cloudflare Workers (env vars y secrets)
type Bindings = {
  BACKEND_URL: string
  ENVIRONMENT: string
  JWT_PUBLIC_KEY_ED25519: string   // Secret — clave pública Ed25519 en PEM
  JWT_ISSUER: string
  JWT_AUDIENCE: string
}

const app = new Hono<{ Bindings: Bindings }>()

// ─── Middlewares globales ───────────────────────────────────────
app.use('*', secureHeaders())
app.use('*', cors({
  origin: ['https://app.serenamente.com'],
  allowMethods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
  allowHeaders: ['Authorization', 'Content-Type', 'X-Session-Token'],
  exposeHeaders: ['Content-Type'],
  credentials: true,
  maxAge: 86400,
}))

// ─── Health check del BFF (no requiere auth) ────────────────────
app.get('/health', (c) => c.json({ status: 'ok', service: 'bff' }))

// ─── Proxy de autenticación (Kratos — sin JWT, maneja la sesión) ─
app.all('/auth/*', async (c) => {
  const url = new URL(c.req.url)
  const targetUrl = `${c.env.BACKEND_URL}${url.pathname}${url.search}`

  const response = await fetch(targetUrl, {
    method: c.req.method,
    headers: c.req.raw.headers,
    body: c.req.method !== 'GET' && c.req.method !== 'HEAD'
      ? await c.req.raw.blob()
      : undefined,
  })

  return new Response(response.body, {
    status: response.status,
    headers: response.headers,
  })
})

// ─── Middleware de verificación JWT para endpoints protegidos ────
// Todos los /api/* (excepto /api/iam/token/exchange) requieren JWT válido
const jwtMiddleware = async (c: any, next: any) => {
  const token = extractBearerToken(c.req.header('Authorization'))
  if (!token) {
    throw new HTTPException(401, { message: 'Authorization header required' })
  }

  const payload = await verifyJWT(token, c.env.JWT_PUBLIC_KEY_ED25519, {
    issuer: c.env.JWT_ISSUER,
    audience: c.env.JWT_AUDIENCE,
  })

  // Inyectar claims del JWT como headers para el backend
  c.set('jwtPayload', payload)
  c.set('userId', payload.sub)
  c.set('userRole', payload.role)
  c.set('tenantId', payload.tenant_id)

  await next()
}

// ─── Token exchange (sesión Kratos → JWT Ed25519) — sin JWT ─────
// Este endpoint es el único de /api/iam que no requiere JWT previo
app.post('/api/iam/token/exchange', async (c) => {
  const targetUrl = `${c.env.BACKEND_URL}/api/iam/token/exchange`
  const response = await fetch(targetUrl, {
    method: 'POST',
    headers: c.req.raw.headers,
    body: await c.req.raw.blob(),
  })
  return new Response(response.body, {
    status: response.status,
    headers: response.headers,
  })
})

// ─── Endpoints protegidos (requieren JWT válido) ─────────────────
app.use('/api/*', jwtMiddleware)

app.all('/api/*', async (c) => {
  const url = new URL(c.req.url)
  const targetUrl = `${c.env.BACKEND_URL}${url.pathname}${url.search}`

  // Propagar los claims del JWT como headers al backend
  // El backend (IAM Service, etc.) confía en estos headers porque
  // el BFF ya verificó el JWT con la clave pública Ed25519.
  const headers = new Headers(c.req.raw.headers)
  headers.set('X-User-ID', c.get('userId') || '')
  headers.set('X-User-Role', c.get('userRole') || '')
  headers.set('X-Tenant-ID', c.get('tenantId') || '')

  const response = await fetch(targetUrl, {
    method: c.req.method,
    headers,
    body: c.req.method !== 'GET' && c.req.method !== 'HEAD'
      ? await c.req.raw.blob()
      : undefined,
  })

  return new Response(response.body, {
    status: response.status,
    headers: response.headers,
  })
})

// ─── Manejo de errores global ────────────────────────────────────
app.onError((err, c) => {
  if (err instanceof HTTPException) {
    return c.json({ error: err.message }, err.status)
  }
  console.error('BFF Error:', err)
  return c.json({ error: 'Internal server error' }, 500)
})

export default app
```

### 20.4 Deploy del BFF a Cloudflare Workers

```bash
cd apps/bff

# Configurar la clave pública JWT como secret de CF Workers:
# (NO incluir en wrangler.toml — es un secret)
wrangler secret put JWT_PUBLIC_KEY_ED25519
# → Enter a secret value: <pegar la clave pública Ed25519 en PEM>

# Build de producción:
bun run build  # o: wrangler deploy --dry-run para verificar sin desplegar

# Deploy a producción:
wrangler deploy
# → Uploading...
# → Deployed serenamente-bff (1.23 kB) to serenamente.com

# Verificar el deploy:
curl https://api.serenamente.com/bff/health
# → {"status":"ok","service":"bff"}
```

---

## 21. Tarea V-16 — Qwik v2.0 SPA en Cloudflare Pages

### 21.1 Inicializar el proyecto Qwik v2.0

```bash
cd apps/web

# Crear proyecto Qwik v2 con CF Pages:
bun create qwik@latest . --template cloudflare-pages

# Instalar TypeScript 6.0:
bun add -D typescript@^6.0.0

# Verificar que usa @qwik.dev/qwik (no @builder.io/qwik):
grep "@qwik.dev" package.json
# → "@qwik.dev/qwik": "^2.0.0"
# → "@qwik.dev/router": "^2.0.0"
```

### 21.2 Configuración de Cloudflare Pages

```bash
# Crear el proyecto en CF Pages via CLI:
wrangler pages project create serenamente-web \
  --production-branch main

# Configurar variables de entorno en CF Pages:
wrangler pages secret put VITE_API_BASE_URL --project-name serenamente-web
# → Enter a secret value: https://api.serenamente.com/bff

wrangler pages secret put VITE_KRATOS_URL --project-name serenamente-web
# → Enter a secret value: https://api.serenamente.com/auth

# El pipeline de CI/CD (V-18) hace el deploy automático en cada push a main.
# Para deploy manual:
bun run build
wrangler pages deploy dist/ --project-name serenamente-web --branch main
```

---

## 22. Tarea V-17 — CloudNativePG ScheduledBackup → Backblaze B2

### 22.1 Crear el secret con las credenciales de B2 cifrado con SOPS

```bash
# Cifrar las credenciales de Backblaze B2 con SOPS+age:
./scripts/encrypt-secret.sh serenamente-data cnpg-b2-credentials \
  "B2_KEY_ID=${B2_KEY_ID}" \
  "B2_APPLICATION_KEY=${B2_APPLICATION_KEY}"

git add infra/secrets/cnpg-b2-credentials.yaml
git commit -m "feat: add backblaze B2 credentials encrypted with SOPS for WAL archiving"
git push
```

### 22.2 ScheduledBackup CRD

```yaml
# infra/infrastructure/cnpg/scheduled-backup.yaml
# Backup base diario a las 02:00 UTC — adicional al WAL continuo
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: serenamente-pg-daily-backup
  namespace: serenamente-data
spec:
  schedule: "0 2 * * *"   # 02:00 UTC todos los días
  backupOwnerReference: self
  cluster:
    name: serenamente-pg
  immediate: true   # Ejecutar un backup inmediatamente al crear este CRD
```

### 22.3 Verificar el backup

```bash
# Verificar que el backup fue creado:
kubectl get backup -n serenamente-data
# → NAME                                    AGE   STATUS      CLUSTER
# → serenamente-pg-daily-backup-xxxxxxxxxx  5m    Completed   serenamente-pg

# Ver detalles del backup:
kubectl describe backup serenamente-pg-daily-backup-xxx -n serenamente-data
# → Destination: s3://serenamente-pg-backups/wal/base/...

# Verificar WAL archiving:
kubectl logs serenamente-pg-1 -n serenamente-data | grep -i "wal\|archive\|barman"
# → Archiving WAL segment: 000000010000000000000001
# → WAL file archived successfully

# Verificar en Backblaze B2 (desde tu máquina local):
# Instalar b2 CLI: https://www.backblaze.com/docs/cloud-storage-command-line-tools
b2 ls serenamente-pg-backups
# → wal/
# → wal/base/
# → wal/wals/
```

---

## 23. Tarea V-18 — GitLab CI/CD

GitLab CI/CD usa un único fichero `.gitlab-ci.yml` en la raíz del monorepo como punto de entrada, con pipelines específicos organizados en `.gitlab/ci/`. Las variables CI/CD definidas en 5.2 (`HCLOUD_TOKEN`, `CF_API_TOKEN`, etc.) son automáticamente inyectadas en todos los jobs.

**Variables automáticas de GitLab CI relevantes:**
- `$CI_REGISTRY` → `registry.gitlab.com`
- `$CI_REGISTRY_USER` / `$CI_REGISTRY_PASSWORD` → credenciales auto-generadas por job
- `$CI_REGISTRY_IMAGE` → `registry.gitlab.com/<grupo>/<repo>`
- `$CI_COMMIT_SHORT_SHA` → SHA corto del commit (equivalente a `GITHUB_SHA` en GitHub Actions)
- `$CI_JOB_TOKEN` → token de corta duración para operaciones Git en el mismo proyecto
- `$CI_SERVER_HOST` → `gitlab.com`
- `$CI_PROJECT_PATH` → `serenamente/serenidad-platform`

### 23.1 Orquestador principal `.gitlab-ci.yml`

```yaml
# .gitlab-ci.yml — Punto de entrada del CI/CD (raíz del monorepo)
include:
  - local: '.gitlab/ci/iam-service.yml'   # Build + test del IAM Domain Service
  - local: '.gitlab/ci/deploy-bff.yml'    # Deploy del BFF a CF Workers
  - local: '.gitlab/ci/deploy-web.yml'    # Deploy del SPA a CF Pages

stages:
  - test
  - build
  - deploy
```

### 23.2 Pipeline de build y push del IAM Domain Service

```yaml
# .gitlab/ci/iam-service.yml
# Pipeline CI/CD para el IAM Domain Service (Go)
# Activado solo cuando cambian archivos bajo services/iam/

variables:
  GO_VERSION: "1.25"

# ─── Stage: test ──────────────────────────────────────────────────
test-iam:
  stage: test
  image: golang:${GO_VERSION}-alpine
  rules:
    - changes:
        - services/iam/**/*
        - .gitlab/ci/iam-service.yml
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        - services/iam/**/*
  before_script:
    - cd services/iam
    - go mod download
  script:
    - go test -v -race -coverprofile=coverage.out ./...
    - |
      COVERAGE=$(go tool cover -func coverage.out | tail -1 | awk '{print $3}' | tr -d '%')
      echo "Coverage: ${COVERAGE}%"
      if (( $(echo "${COVERAGE} < 60" | bc -l) )); then
        echo "ERROR: Coverage ${COVERAGE}% es inferior al umbral del 60%"
        exit 1
      fi
    - go build ./...
    - go vet ./...
  coverage: '/total:\s+\(statements\)\s+(\d+\.\d+)%/'
  cache:
    key: go-mod-${CI_COMMIT_REF_SLUG}
    paths:
      - services/iam/vendor/
      - $GOPATH/pkg/mod/

# ─── Stage: build ─────────────────────────────────────────────────
build-push-iam:
  stage: build
  image: docker:26
  services:
    - docker:26-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      changes:
        - services/iam/**/*
  needs:
    - job: test-iam
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - |
      IMAGE_BASE="${CI_REGISTRY_IMAGE}/iam-service"
      IMAGE_TAG="${CI_COMMIT_SHORT_SHA}"
      # Build de la imagen con dos tags: SHA corto + latest
      docker build \
        --label "org.opencontainers.image.revision=${CI_COMMIT_SHA}" \
        --label "org.opencontainers.image.source=${CI_PROJECT_URL}" \
        -t "${IMAGE_BASE}:${IMAGE_TAG}" \
        -t "${IMAGE_BASE}:latest" \
        -f services/iam/Dockerfile services/iam/
      docker push "${IMAGE_BASE}:${IMAGE_TAG}"
      docker push "${IMAGE_BASE}:latest"
    - |
      # Actualizar el tag en el manifest de FluxCD y pushear al repositorio
      # FluxCD detecta el cambio y despliega automáticamente (~1 min)
      sed -i "s|${CI_REGISTRY_IMAGE}/iam-service:.*|${CI_REGISTRY_IMAGE}/iam-service:${CI_COMMIT_SHORT_SHA}|g" \
        infra/apps/iam-service/deployment.yaml
      git config user.name "GitLab CI"
      git config user.email "ci@serenamente.com"
      git remote set-url origin "https://oauth2:${CI_JOB_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git"
      git add infra/apps/iam-service/deployment.yaml
      git commit -m "chore: update iam-service image to sha-${CI_COMMIT_SHORT_SHA}" || true
      git push origin HEAD:main
```

### 23.3 Pipeline de deploy del BFF a CF Workers

```yaml
# .gitlab/ci/deploy-bff.yml
# Pipeline de deploy del BFF Hono a Cloudflare Workers

deploy-bff:
  stage: deploy
  image: node:22-alpine
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      changes:
        - apps/bff/**/*
        - .gitlab/ci/deploy-bff.yml
  before_script:
    - npm install -g bun wrangler --silent
    - cd apps/bff
    - bun install --frozen-lockfile
  script:
    - bun run tsc --noEmit
    - wrangler deploy
  variables:
    CLOUDFLARE_API_TOKEN: $CF_API_TOKEN
    CLOUDFLARE_ACCOUNT_ID: $CF_ACCOUNT_ID
```

### 23.4 Pipeline de deploy del SPA a CF Pages

```yaml
# .gitlab/ci/deploy-web.yml
# Pipeline de deploy del Qwik SPA a Cloudflare Pages

deploy-web:
  stage: deploy
  image: node:22-alpine
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      changes:
        - apps/web/**/*
        - .gitlab/ci/deploy-web.yml
  before_script:
    - npm install -g bun wrangler --silent
    - cd apps/web
    - bun install --frozen-lockfile
  script:
    - bun run build
    - wrangler pages deploy dist/ --project-name=serenamente-web --branch=main
  variables:
    VITE_API_BASE_URL: https://api.serenamente.com/bff
    VITE_KRATOS_URL: https://api.serenamente.com/auth
    CLOUDFLARE_API_TOKEN: $CF_API_TOKEN
    CLOUDFLARE_ACCOUNT_ID: $CF_ACCOUNT_ID
```

---

## 24. Tarea V-19 — Firewall y hardening de red

### 24.1 Firewall de Kubernetes con NetworkPolicies

Talos + Kubernetes con flannel permite usar NetworkPolicies para limitar el tráfico entre pods. Esta es la segunda línea de defensa (el Firewall de Hetzner es la primera).

```yaml
# infra/infrastructure/network-policies/deny-all-default.yaml
# Política base: denegar todo el tráfico por defecto en serenamente-core y serenamente-data.
# Los pods solo pueden comunicarse si hay una NetworkPolicy explícita que lo permita.
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: serenamente-core
spec:
  podSelector: {}   # Aplica a todos los pods del namespace
  policyTypes:
    - Ingress
    - Egress
  # Sin rules = denegar todo
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: serenamente-data
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
# Permitir tráfico de Traefik → IAM Service
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-traefik-to-iam
  namespace: serenamente-core
spec:
  podSelector:
    matchLabels:
      app: iam-service
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: traefik
      ports:
        - protocol: TCP
          port: 8080
---
# Permitir tráfico IAM → PostgreSQL
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-iam-to-postgres
  namespace: serenamente-data
spec:
  podSelector:
    matchLabels:
      cnpg.io/cluster: serenamente-pg
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: serenamente-core
      ports:
        - protocol: TCP
          port: 5432
---
# Permitir egress DNS (CoreDNS) para todos los pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: serenamente-core
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

### 24.2 Actualizar reglas del Firewall de Hetzner para IPs dinámicas

Si tu IP de trabajo cambia, actualizar el firewall de Hetzner:

```bash
# Script para actualizar el firewall con tu IP actual:
# scripts/update-firewall.sh
#!/bin/bash
set -euo pipefail

source .envrc

MY_IP=$(curl -s https://ifconfig.me)
echo "Tu IP actual: ${MY_IP}"

# Obtener el ID del firewall:
FIREWALL_ID=$(hcloud firewall list -o json | jq -r '.[] | select(.name=="serenamente-prod-fw") | .id')

# Actualizar regla del puerto Talos (50000):
hcloud firewall replace-rule ${FIREWALL_ID} \
  --direction in \
  --protocol tcp \
  --port 50000 \
  --source-ips "${MY_IP}/32"

# Actualizar regla del puerto K8s API (6443):
hcloud firewall replace-rule ${FIREWALL_ID} \
  --direction in \
  --protocol tcp \
  --port 6443 \
  --source-ips "${MY_IP}/32"

echo "✓ Firewall actualizado para la IP: ${MY_IP}"
```

```bash
chmod +x scripts/update-firewall.sh
```

---

## 25. Flujo de trabajo diario en producción directa

### 25.1 Ciclo de desarrollo de código Go (lógica de negocio)

El desarrollo de lógica de negocio en Go es completamente local. Solo el deploy requiere interacción con el VPS.

```bash
# 1. Código y tests locales (milisegundos de feedback):
cd services/iam
go test ./... -v -race
# → PASS (local, instantáneo)

# 2. Commit y push desencadena el pipeline CI:
git add .
git commit -m "feat(iam): add doctor specialty validation in onboarding flow"
git push origin main
# → GitLab CI: test → build → push a registry.gitlab.com → actualiza image tag en deployment.yaml

# 3. FluxCD detecta el cambio en deployment.yaml y despliega (~1-3 min):
flux get all
# → NAME              READY   MESSAGE
# → kustomization     True    Applied revision: main@sha1:...

# 4. Verificar el rollout:
kubectl rollout status deployment/iam-service -n serenamente-core
# → deployment "iam-service" successfully rolled out

# 5. Verificar logs en tiempo real:
kubectl logs -f deployment/iam-service -n serenamente-core
```

### 25.2 Ciclo de deploy de emergencia (bypass CI para hotfix crítico)

Solo para bugs críticos en producción que no pueden esperar el pipeline CI.

```bash
# SOLO PARA EMERGENCIAS — preferir siempre el pipeline CI normal

# 1. Hacer cambio mínimo en el código
# 2. Build local de la imagen:
cd services/iam
docker build -t registry.gitlab.com/serenamente/serenidad-platform/iam-service:hotfix-$(date +%Y%m%d%H%M) .

# 3. Login y push directo a GitLab Container Registry:
docker login registry.gitlab.com -u ${GITLAB_USER} -p ${GITLAB_TOKEN}
docker push registry.gitlab.com/serenamente/serenidad-platform/iam-service:hotfix-$(date +%Y%m%d%H%M)

# 4. Actualizar la imagen directamente en el cluster (bypass FluxCD temporalmente):
kubectl set image deployment/iam-service \
  iam-service=registry.gitlab.com/serenamente/serenidad-platform/iam-service:hotfix-$(date +%Y%m%d%H%M) \
  -n serenamente-core

# 5. Después del fix: hacer commit, push, y dejar que FluxCD re-sincronice a la versión normal.
# FluxCD revertirá el hotfix manual a la versión en Git — eso es correcto.
```

### 25.3 Gestión de secretos nuevos

```bash
# Para agregar un nuevo secret a producción:
# 1. Cifrar el secret con SOPS+age:
./scripts/encrypt-secret.sh <namespace> <nombre-secret> KEY=valor KEY2=valor2

# 2. Commitear y pushear:
git add infra/secrets/<nombre-secret>.yaml
git commit -m "feat: add <nombre-secret> encrypted with SOPS"
git push

# 3. FluxCD aplica automáticamente el nuevo secret al cluster (~1 min)
# 4. kustomize-controller descifra con la clave age del secret sops-age

# Para rotar la clave age (e.g., ante compromiso de la clave privada):
# 1. Generar nueva clave age:
age-keygen -o ~/.config/sops/age/keys-new.txt
NEW_PUBLIC_KEY=$(grep '^# public key:' ~/.config/sops/age/keys-new.txt | awk '{print $4}')

# 2. Actualizar .sops.yaml con la nueva clave:
# Editar infra/secrets/.sops.yaml: reemplazar age: <old-key> por age: <new-key>

# 3. Re-cifrar todos los secrets con la nueva clave:
for f in infra/secrets/*.yaml; do
  sops --config infra/secrets/.sops.yaml --rotate --in-place "$f"
done

# 4. Actualizar el secret en el cluster:
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=~/.config/sops/age/keys-new.txt \
  --dry-run=client -o yaml | kubectl apply -f -

# 5. Commitear y pushear los secrets re-cifrados
git add infra/secrets/
git commit -m "chore: rotate SOPS age encryption key"
git push
```

### 25.4 Actualización de Talos Linux

```bash
# Actualizar Talos OS (actualiza el OS y Kubernetes en un solo paso):
# Verificar versión actual:
talosctl version --nodes ${FLOATING_IP}

# Actualizar a nueva versión (reemplazar <NEW_VERSION>):
talosctl upgrade \
  --nodes ${FLOATING_IP} \
  --image ghcr.io/siderolabs/installer:<NEW_VERSION>
# Nota: ghcr.io/siderolabs/installer es la imagen oficial de Talos Linux
# distribuida por Siderolabs en GitHub Container Registry (no es nuestro registry).
# El nodo se reinicia automáticamente con la nueva versión

# Verificar versión después del upgrade:
talosctl version --nodes ${FLOATING_IP}
kubectl get nodes
# → El nodo debe volver a Ready con la nueva versión de Kubernetes
```

### 25.5 Recuperación ante fallo del nodo

```bash
# Si el VPS no responde — verificar en Hetzner:
hcloud server describe serenamente-prod-01

# Si el servidor está parado, iniciarlo:
hcloud server poweron serenamente-prod-01

# Si el servidor está corrupto, reconstruirlo:
# TODOS LOS DATOS SE PIERDEN EXCEPTO:
# - Los backups de PostgreSQL en B2 (recuperables via PITR)
# - Los secrets cifrados con SOPS en Git (se re-aplican automáticamente por FluxCD)
# - PERO: el secret sops-age del cluster se pierde al destruir el nodo
# - POST-RECOVERY: recrear el secret sops-age manualmente (paso 2 de la sección 10.3)
# - Las imágenes en registry.gitlab.com

# 1. Poner el servidor en rescue mode:
hcloud server enable-rescue serenamente-prod-01 \
  --type linux64 --ssh-key serenamente-admin-key
hcloud server reset serenamente-prod-01

# 2. Reinstalar Talos desde rescue (ver Tarea V-02):
# ... (repetir proceso de flasheo)

# 3. Aplicar la misma configuración de Talos (los secrets del cluster aún son válidos):
talosctl apply-config --insecure --nodes ${VPS_IP} \
  --file infra/clusters/hetzner-prod/talos/controlplane.yaml

# 4. Bootstrap Kubernetes:
talosctl bootstrap --nodes ${FLOATING_IP}

# 5. FluxCD se reinstala automáticamente (está en Git) después de obtener el kubeconfig.
# flux bootstrap gitlab ... (repetir Tarea V-05)

# 6. CloudNativePG restaura PostgreSQL desde B2:
kubectl cnpg restore serenamente-pg \
  --backup serenamente-pg-daily-backup-latest \
  -n serenamente-data
# O usar PITR para recuperar hasta un punto específico en el tiempo.

# Tiempo de recuperación estimado: 30-60 minutos para un servidor completamente nuevo.
# Con PITR: la pérdida máxima de datos es el WAL no archivado antes del fallo (typically <5 minutos).
```

---

## 26. Observabilidad mínima viable (Fase 1)

En Fase 1 no se despliega el PLG stack completo (Prometheus + Loki + Tempo + Grafana). Se usa una observabilidad mínima que no consume recursos adicionales significativos.

### 26.1 Health checks obligatorios en cada microservicio Go

Todo microservicio Go en Serenamente debe implementar estos tres endpoints como parte de su definición de hecho:

```go
// Liveness: el proceso está vivo (para que k8s haga restart si falla)
GET /health  → {"status": "ok", "service": "iam-service", "version": "0.1.0"}

// Readiness: el servicio puede atender requests (para que k8s no envíe tráfico si no está listo)
GET /ready   → {"status": "ok", "db": "connected", "kratos": "reachable"}

// Métricas: exponer métricas en formato Prometheus (para Fase 3)
GET /metrics → # HELP ... (formato OpenMetrics)
```

### 26.2 Consultas de estado rápidas

```bash
# Estado general del cluster:
kubectl get pods -A --sort-by='.status.startTime'
# → Ver todos los pods ordenados por tiempo de inicio

# Pods con problemas:
kubectl get pods -A | grep -v "Running\|Completed"
# → Solo muestra pods que NO están en estado normal

# Logs de un pod en tiempo real:
kubectl logs -f deployment/iam-service -n serenamente-core

# Logs de los últimos N minutos:
kubectl logs deployment/iam-service -n serenamente-core --since=10m

# Ver eventos del cluster (útil para diagnosticar problemas de scheduling):
kubectl get events -A --sort-by='.lastTimestamp' | tail -20

# Estado de FluxCD:
flux get all --status-selector ready=false
# → Solo muestra recursos de FluxCD que NO están sincronizados correctamente

# Estado de CloudNativePG:
kubectl get cluster,backup,scheduledbackup -n serenamente-data

# CPU y memoria de pods:
kubectl top pods -A --sort-by=memory
# → Requiere: kubectl top funcionando (metrics-server — incluido en Talos)

# Estado de los volúmenes:
kubectl get pvc -A
```

### 26.3 Alertas básicas via Hetzner Monitoring

Hetzner Cloud ofrece alertas básicas de infraestructura en el panel de control:
- CPU utilization > 90% por más de 5 minutos
- Network traffic (útil para detectar tráfico inusual)
- Server unavailability (ping check)

Configurar en: Hetzner Cloud Console → serenamente-prod-01 → Monitoring → Configure Alerts.

> El PLG stack completo (Prometheus + Loki + Tempo + Grafana + Alertmanager → Telegram) se despliega en **Tarea 3.C y 3.D de Fase 3**.

---

## 27. Troubleshooting exhaustivo

### 27.1 El nodo está en NotReady

```bash
# Ver la razón del NotReady:
kubectl describe node serenamente-prod-01 | grep -A20 "Conditions:"

# Verificar taints (el CCM puede haber puesto el taint uninitialized):
kubectl get node serenamente-prod-01 -o json | jq '.spec.taints'

# Si el taint es node.cloudprovider.kubernetes.io/uninitialized:
# → El CCM no está corriendo o tiene error. Verificar:
kubectl get pods -n hcloud-system
kubectl logs deployment/hcloud-cloud-controller-manager -n hcloud-system

# Verificar etcd:
talosctl etcd status --nodes ${FLOATING_IP}

# Reiniciar kubelet (no es necesario normalmente — Talos lo gestiona):
talosctl service kubelet restart --nodes ${FLOATING_IP}
```

### 27.2 FluxCD no sincroniza

```bash
# Ver qué recurso de FluxCD tiene error:
flux get all

# Ver logs del controller relevante:
kubectl logs deployment/source-controller -n flux-system | tail -50
kubectl logs deployment/kustomize-controller -n flux-system | tail -50
kubectl logs deployment/helm-controller -n flux-system | tail -50

# Forzar reconciliation manual:
flux reconcile source git flux-system
flux reconcile kustomization flux-system

# Si hay errores de autenticación con GitLab:
kubectl get secret flux-system -n flux-system -o yaml
# Verificar que el deploy key sigue activo en GitLab (Settings → Repository → Deploy keys)
```

### 27.3 cert-manager no emite certificados

```bash
# Ver el estado del Certificate:
kubectl describe certificate <nombre> -n <namespace>
# → Buscar: Status, Events

# Ver los challenges de ACME:
kubectl get challenges -A

# El error más común: el dominio no resuelve a la IP del VPS
dig api.serenamente.com
# Debe resolver a la IP de Cloudflare (si el proxy está activo) o al VPS (si es DNS-only)

# Verificar que el challenge HTTP-01 funciona:
# cert-manager crea un pod temporal en el namespace del Certificate que sirve el challenge
kubectl get pods -n serenamente-core | grep cm-acme

# Ver logs del cert-manager:
kubectl logs deployment/cert-manager -n cert-manager | grep -i "error\|challenge" | tail -20

# Solución común: asegurarse de que el puerto :80 está abierto en el Firewall de Hetzner
hcloud firewall describe serenamente-prod-fw | grep -A3 "80"
```

### 27.4 PostgreSQL no inicia

```bash
# Ver estado del Cluster CloudNativePG:
kubectl describe cluster serenamente-pg -n serenamente-data
# → Buscar: Status, Events

# Ver logs del pod de PostgreSQL:
kubectl logs serenamente-pg-1 -n serenamente-data | tail -50

# Error común: el PVC no se pudo crear (problema con StorageClass):
kubectl get pvc -n serenamente-data
kubectl describe pvc serenamente-pg-1 -n serenamente-data

# Verificar que local-path-provisioner está corriendo:
kubectl get pods -n kube-system | grep local-path

# Error de permisos en el PVC:
# Talos incluye local-path-provisioner. Verificar que el StorageClass existe:
kubectl get storageclass
# → local-path  rancher.io/local-path  Delete  WaitForFirstConsumer  false  10m
```

### 27.5 Kratos no puede conectar a la base de datos

```bash
# Ver logs de Kratos:
kubectl logs deployment/kratos -n serenamente-core | grep -i "error\|database\|dsn"

# Verificar que el Secret de credenciales existe:
kubectl get secret kratos-secrets -n serenamente-core

# Verificar conectividad a PostgreSQL desde el pod de Kratos:
kubectl exec -it deployment/kratos -n serenamente-core -- \
  wget -qO- http://serenamente-pg-rw.serenamente-data.svc.cluster.local:5432/ || true
# No debe retornar "connection refused"

# Verificar NetworkPolicy — puede estar bloqueando la conexión:
kubectl get networkpolicies -n serenamente-data
```

### 27.6 El IAM Service retorna 500 en /internal/validate-token

```bash
# Ver logs del IAM Service:
kubectl logs deployment/iam-service -n serenamente-core --since=5m

# Verificar que el Secret JWT existe:
kubectl get secret iam-service-secrets -n serenamente-core

# Verificar que Kratos Admin API está accesible:
kubectl exec -it deployment/iam-service -n serenamente-core -- \
  wget -qO- http://kratos-admin.serenamente-core.svc.cluster.local:4434/health/ready
# → {"status":"ok"}

# Verificar las variables de entorno del pod:
kubectl exec -it deployment/iam-service -n serenamente-core -- env | grep -v KEY | grep -v SECRET
```

### 27.7 Traefik no enruta correctamente

```bash
# Ver IngressRoutes configurados:
kubectl get ingressroutes -A

# Ver middlewares configurados:
kubectl get middlewares -A

# Ver logs de Traefik:
kubectl logs daemonset/traefik -n traefik | grep -i "error\|level=error" | tail -30

# Ver la configuración dinámica de Traefik (acceder al dashboard internamente):
kubectl port-forward -n traefik daemonset/traefik 9000:9000
# En otro terminal:
curl http://localhost:9000/api/rawdata | jq '.routers | keys'
# → Ver todas las rutas configuradas

# Test de acceso directo a la IP del VPS (bypass Cloudflare):
curl -k --resolve api.serenamente.com:443:${FLOATING_IP} \
  https://api.serenamente.com/auth/.well-known/ory/webauthn.js
```

---

## 28. Criterios de aceptación global de la Fase 1

Esta sección define con precisión cuándo la Fase 1 está **completa** y lista para que los primeros usuarios accedan al sistema. Todos los criterios deben cumplirse antes de considerar la Fase 1 terminada.

### 28.1 Infraestructura (Tareas V-01 a V-04)

| # | Criterio | Verificación |
|---|---------|-------------|
| I-01 | VPS Hetzner CX32 corriendo con Talos Linux v1.10.x | `talosctl version --nodes ${FLOATING_IP}` → v1.10.x |
| I-02 | Kubernetes 1.33.x funcionando single-node | `kubectl get nodes` → `Ready control-plane v1.33.x` |
| I-03 | Hetzner CCM corriendo, nodo sin taint uninitialized | `kubectl get node` → sin taints del CCM |
| I-04 | Floating IP asignada y estable | `hcloud floating-ip describe serenamente-prod-fip` → `Assigned` |
| I-05 | Firewall de Hetzner correctamente configurado | Puertos 80, 443 públicos; 50000, 6443 restringidos a IP de trabajo |

### 28.2 GitOps y CI/CD (Tareas V-05 a V-06, V-18)

| # | Criterio | Verificación |
|---|---------|-------------|
| G-01 | FluxCD v2.x sincronizando la rama main del monorepo | `flux get all` → todos Ready |
| G-02 | Push a `services/iam/` desencadena build + deploy automático | `git push` → GitLab CI build OK → FluxCD deploy OK |
| G-03 | Push a `apps/bff/` desencadena deploy a CF Workers | `wrangler deploy` en GitLab CI → BFF accesible |
| G-04 | Push a `apps/web/` desencadena deploy a CF Pages | Deploy OK → `https://app.serenamente.com` accesible |
| G-05 | Todos los secrets sensibles cifrados con SOPS+age en Git | Sin secrets en texto plano en ningún commit; `sops -d infra/secrets/*.yaml` funciona con la clave age |

### 28.3 Networking y TLS (Tareas V-07 a V-08, V-11)

| # | Criterio | Verificación |
|---|---------|-------------|
| N-01 | cert-manager emite certificado Let's Encrypt para `api.serenamente.com` | `kubectl get certificate -A` → Ready: True |
| N-02 | HTTPS funciona con certificado de confianza global | `curl https://api.serenamente.com/auth/.well-known/ory/webauthn.js` → HTTP 200, sin warnings TLS |
| N-03 | HTTP redirige a HTTPS automáticamente | `curl http://api.serenamente.com` → 308 Permanent Redirect |
| N-04 | Traefik sirve tráfico en :80 y :443 del VPS | `curl -I http://${FLOATING_IP}` → 308 |
| N-05 | ForwardAuth middleware configurado correctamente | Request sin JWT → 401. Request con JWT válido → 200 |

### 28.4 Base de datos (Tarea V-10)

| # | Criterio | Verificación |
|---|---------|-------------|
| D-01 | CloudNativePG cluster en estado Healthy | `kubectl get cluster serenamente-pg -n serenamente-data` → STATUS: Cluster in healthy state |
| D-02 | Las 6 databases existen | `psql -c "\l"` → iam_db, kratos_db, openfga_db, scheduling_db, clinical_db, billing_db |
| D-03 | RLS activo en todas las tablas de usuario | `psql -c "SELECT tablename, rowsecurity FROM pg_tables WHERE rowsecurity=true"` |
| D-04 | Extensiones instaladas | `psql -d iam_db -c "\dx"` → pg_stat_statements, btree_gist |
| D-05 | UUIDv7 funcional en PG17 | `psql -c "SELECT gen_random_uuid()::text"` → retorna UUID v7 (tiempo en los primeros bits) |

### 28.5 Backups (Tarea V-17)

| # | Criterio | Verificación |
|---|---------|-------------|
| B-01 | WAL archiving a Backblaze B2 funcionando | `kubectl logs serenamente-pg-1 -n serenamente-data \| grep "archived"` → sin errores |
| B-02 | Backup base ejecutado con éxito | `kubectl get backup -n serenamente-data` → STATUS: Completed |
| B-03 | PITR teóricamente disponible | Diferencia entre `created_at` del backup y `NOW()` < 10 minutos |

### 28.6 IAM (Tareas V-13 a V-14)

| # | Criterio | Verificación |
|---|---------|-------------|
| A-01 | Kratos v1.3.1 responde `/health/ready` | `curl https://api.serenamente.com/auth/.well-known/ory/webauthn.js` → HTTP 200 |
| A-02 | Kratos DB migrada correctamente | `psql -d kratos_db -c "\dt"` → tablas de Kratos presentes |
| A-03 | IAM Domain Service responde `/health` | `kubectl exec -it deployment/iam-service -n serenamente-core -- wget -qO- http://localhost:8080/health` → `{"status":"ok"}` |
| A-04 | `/internal/validate-token` retorna 401 para request sin token | `curl https://api.serenamente.com/api/iam/whoami` → 401 Unauthorized |
| A-05 | Flujo completo de registro + login con Passkey funcional | Test manual en `https://app.serenamente.com/register` → Passkey registrada, login exitoso |
| A-06 | POST `/api/iam/token/exchange` retorna JWT firmado con Ed25519 | Decodificar JWT con `jwt.io` → header alg: "EdDSA", claims: sub, role, tenant_id, did |
| A-07 | JWT verificable con la clave pública Ed25519 | `openssl dgst -verify jwt_public_key.pem -signature <sig> <data>` → Verification OK |
| A-08 | JWT contiene todos los claims necesarios | `sub`, `role`, `tenant_id`, `did`, `iat`, `exp` presentes en el payload |

### 28.7 BFF y Frontend (Tareas V-15 a V-16)

| # | Criterio | Verificación |
|---|---------|-------------|
| F-01 | BFF Hono desplegado en CF Workers | `curl https://api.serenamente.com/bff/health` → `{"status":"ok","service":"bff"}` |
| F-02 | BFF rechaza requests sin JWT a endpoints protegidos | `curl https://api.serenamente.com/bff/api/iam/whoami` → 401 |
| F-03 | Qwik SPA accesible en CF Pages | `curl https://app.serenamente.com` → HTML de la SPA (status 200) |
| F-04 | Qwik SPA tiene TTI < 100ms medido con Lighthouse | `npx lighthouse https://app.serenamente.com --only-categories=performance` → Score ≥ 95 |

### 28.8 Flujo end-to-end (verificación final)

El criterio definitivo de aceptación de la Fase 1 es el **flujo completo** funcional:

```
1. Usuario abre https://app.serenamente.com en Chrome/Safari/Firefox
2. Hace clic en "Registrarse"
3. Ory Kratos muestra el flow de registro
4. Usuario registra una Passkey (WebAuthn/FIDO2) con su dispositivo
5. Kratos envía email de verificación via Resend
6. Usuario verifica su email
7. Usuario inicia sesión con su Passkey
8. Kratos emite session token
9. Frontend llama a POST /bff/api/iam/token/exchange con el session token
10. IAM Domain Service verifica la sesión con Kratos Admin API
11. IAM Domain Service crea el perfil de usuario en iam_db (si no existe)
12. IAM Domain Service inserta evento UserRegistered en outbox_events en la MISMA TX
13. IAM Domain Service firma JWT Ed25519 con claims: sub, role, tenant_id, did
14. Frontend recibe el JWT y lo almacena (sessionStorage o cookie httpOnly)
15. Frontend hace GET /bff/api/iam/whoami con el JWT en Authorization header
16. BFF verifica el JWT con la clave pública Ed25519
17. BFF propaga X-User-ID, X-User-Role, X-Tenant-ID al backend
18. Traefik ejecuta ForwardAuth → IAM /internal/validate-token → 200 OK
19. IAM Domain Service retorna el perfil del usuario
20. Frontend muestra el dashboard del usuario

✓ Todos los pasos deben completarse en <3 segundos en condiciones normales.
```

---

## Apéndice A — Diagrama de dependencias de tareas

```
V-00 Preparar monorepo
  └─► V-01 Aprovisionamiento Hetzner CX32
       └─► V-02 Instalar Talos Linux (modo rescue)
            └─► V-03 Bootstrap Talos + Kubernetes 1.33.x
                 ├─► V-04 Hetzner CCM                        ← prerequisito del cluster funcional
                 ├─► V-05 FluxCD v2.x bootstrap
                 │    └─► V-06 Estructura GitOps del repo
                 │         ├─► V-07 cert-manager v1.x
                 │         │    └─► V-08 Traefik v3.x
                 │         │         └─► V-11 DNS Cloudflare ← prerequisito para TLS válido
                 │         ├─► V-09 SOPS + age (cifrado GitOps)
                 │         │    └─► V-12 Protobuf schemas
                 │         └─► V-10 CloudNativePG + PG17.4
                 │              └─► V-17 ScheduledBackup → B2
                 │                   └─► V-13 Ory Kratos v1.3.1
                 │                        └─► V-14 IAM Domain Service Go
                 │                             ├─► V-15 BFF Hono CF Workers
                 │                             │    └─► V-16 Qwik SPA CF Pages
                 │                             └─► V-18 GitLab CI/CD
                 └─► V-19 Firewall y hardening de red
```

## Apéndice B — Resumen de archivos críticos y su estado en Git

| Archivo | ¿En Git? | ¿Cifrado? | Notas |
|---------|----------|----------|-------|
| `infra/clusters/hetzner-prod/talos/secrets.yaml` | NO | — | `.gitignore`. Backup en password manager. |
| `infra/clusters/hetzner-prod/talos/talosconfig` | NO | — | `.gitignore`. Backup en password manager. |
| `infra/clusters/hetzner-prod/kubeconfig` | NO | — | `.gitignore`. |
| `infra/secrets/.sops.yaml` | **SÍ** | — | Reglas SOPS: path\_regex + clave age pública (segura en Git). |
| `infra/secrets/*.yaml` (SOPS-encrypted) | **SÍ** | SÍ (AES-256-GCM + age) | Valores cifrados client-side. `sops -d` para inspeccionar. |
| `~/.config/sops/age/keys.txt` (clave age privada) | NO | — | Fuera del repo. Backup **obligatorio** en password manager. |
| `.envrc` | NO | — | `.gitignore`. Variables locales de trabajo. |
| `infra/clusters/hetzner-prod/talos/patches/hetzner-cx32.yaml` | **SÍ** | — | Patch de configuración (sin secretos). |

## Apéndice C — Glosario de comandos de uso diario

```bash
# Alias recomendados (añadir a ~/.bashrc o ~/.zshrc):
alias kp='kubectl --kubeconfig ~/serenidad-platform/infra/clusters/hetzner-prod/kubeconfig'
alias tp='talosctl --talosconfig ~/serenidad-platform/infra/clusters/hetzner-prod/talos/talosconfig --nodes <FLOATING_IP>'
alias fp='flux --kubeconfig ~/serenidad-platform/infra/clusters/hetzner-prod/kubeconfig'

# Comandos más usados:
kp get pods -A                                          # Estado de todos los pods
kp logs -f deployment/iam-service -n serenamente-core  # Logs en tiempo real
kp top pods -A --sort-by=memory                        # Uso de recursos
kp get events -A --sort-by='.lastTimestamp' | tail -20 # Eventos recientes
fp get all                                              # Estado de FluxCD
fp reconcile source git flux-system                     # Forzar sincronización
tp health                                               # Salud de Talos
tp dmesg | tail -20                                     # Logs del kernel Talos
```
