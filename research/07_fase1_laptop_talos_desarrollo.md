# Fase 1 — Entorno de Desarrollo Local: Laptop con Talos Linux

## Guía Exhaustiva de Implementación para Desarrollo

**Proyecto:** Serenamente — Clínica Digital de Salud Mental Global
**Versión:** 1.0 — Abril 2026
**Autor:** djca
**Relación con otros documentos:**
- [`plans/05_orden_implementacion_capas_exhaustivo.md`](./05_orden_implementacion_capas_exhaustivo.md) — Plan Fase 1 para producción (VPS Hetzner)
- [`plans/06_talos_k8s_decision_y_cambios_en_cadena.md`](./06_talos_k8s_decision_y_cambios_en_cadena.md) — Decisión arquitectónica Talos+k8s
- [`plans/08_migracion_laptop_a_vps.md`](./08_migracion_laptop_a_vps.md) — Migración del laptop al VPS cuando sea el momento

---

## Índice

1. [¿Por qué usar el laptop con Talos para desarrollo?](#1-por-qué-usar-el-laptop-con-talos-para-desarrollo)
2. [Arquitectura del Entorno Local](#2-arquitectura-del-entorno-local)
3. [Diferencias entre Entorno Local y Producción (VPS)](#3-diferencias-entre-entorno-local-y-producción-vps)
4. [Prerrequisitos](#4-prerrequisitos)
5. [Tarea L-01 — Instalar Talos Linux en el Laptop](#5-tarea-l-01--instalar-talos-linux-en-el-laptop)
6. [Tarea L-02 — Bootstrap de Talos + Kubernetes](#6-tarea-l-02--bootstrap-de-talos--kubernetes)
7. [Tarea L-03 — Registro Local de Imágenes de Contenedor](#7-tarea-l-03--registro-local-de-imágenes-de-contenedor)
8. [Tarea L-04 — FluxCD (GitOps Local)](#8-tarea-l-04--fluxcd-gitops-local)
9. [Tarea L-05 — DNS Local y Resolución de Nombres](#9-tarea-l-05--dns-local-y-resolución-de-nombres)
10. [Tarea L-06 — cert-manager con ClusterIssuer Self-Signed](#10-tarea-l-06--cert-manager-con-clusterissuer-self-signed)
11. [Tarea L-07 — Traefik v3.x (acceso local vía hostPort)](#11-tarea-l-07--traefik-v3x-acceso-local-vía-hostport)
12. [Tarea L-08 — Sealed Secrets (clave de desarrollo)](#12-tarea-l-08--sealed-secrets-clave-de-desarrollo)
13. [Tarea L-09 — CloudNativePG v1.x + PostgreSQL 17.4](#13-tarea-l-09--cloudnativepg-v1x--postgresql-174)
14. [Tarea L-10 — Mailpit (sustituto SMTP para desarrollo)](#14-tarea-l-10--mailpit-sustituto-smtp-para-desarrollo)
15. [Tarea L-11 — Ory Kratos v1.3.1 (adaptado para local)](#15-tarea-l-11--ory-kratos-v131-adaptado-para-local)
16. [Tarea L-12 — IAM Domain Service Go (con hot-reload)](#16-tarea-l-12--iam-domain-service-go-con-hot-reload)
17. [Tarea L-13 — BFF Hono v4.x en local con Wrangler](#17-tarea-l-13--bff-hono-v4x-en-local-con-wrangler)
18. [Tarea L-14 — Qwik SPA en local](#18-tarea-l-14--qwik-spa-en-local)
19. [Tarea L-15 — MinIO como sustituto de Backblaze B2](#19-tarea-l-15--minio-como-sustituto-de-backblaze-b2)
20. [Flujo de Trabajo Diario](#20-flujo-de-trabajo-diario)
21. [Herramientas de Diagnóstico y Observabilidad Ligera](#21-herramientas-de-diagnóstico-y-observabilidad-ligera)
22. [Troubleshooting Frecuente](#22-troubleshooting-frecuente)
23. [Criterios de Aceptación Global del Entorno Local](#23-criterios-de-aceptación-global-del-entorno-local)

---

## 1. ¿Por qué usar el laptop con Talos para desarrollo?

### 1.1 Principio de paridad de entornos

El mayor riesgo en cualquier sistema de software es la divergencia entre entornos de desarrollo y producción. Con Docker Compose en desarrollo y Kubernetes en producción se introduce una brecha arquitectónica fundamental: los manifests, las políticas de red, los mecanismos de healthcheck, el descubrimiento de servicios y la gestión de secretos funcionan de manera radicalmente diferente. Esto viola R1 (Build It Right The First Time) porque el developer está efectivamente construyendo para un entorno que nunca verá en producción.

Tener Talos Linux en el laptop como nodo Kubernetes elimina esta brecha. El desarrollo ocurre exactamente sobre el mismo stack de orquestación que producción:

- Los manifests YAML son idénticos (salvo las diferencias documentadas en la sección 3)
- Los `ClusterIssuer`, `IngressRoute`, `HelmRelease`, `Cluster` CRDs son los mismos objetos
- Los microservicios Go, las migraciones SQL, los Outbox Workers funcionan igual
- Los errores de configuración se descubren en desarrollo, no en producción

### 1.2 Ventajas específicas sobre Docker Compose

| Aspecto | Docker Compose en dev | Talos+k8s en dev |
|---------|----------------------|------------------|
| Modelo de red | Bridge network, hostname resolution básico | DNS en cluster (CoreDNS), `svc.cluster.local` igual que prod |
| Secrets | `.env` files o `environment:` en YAML | Sealed Secrets, mismo mecanismo que prod |
| Health checks | `healthcheck:` Docker simple | `livenessProbe` + `readinessProbe` k8s, igual que prod |
| Service discovery | Nombre del servicio en compose | `svc.namespace.svc.cluster.local`, igual que prod |
| TLS interno | Sin TLS o auto-firmado manual | cert-manager, mismo operador que prod |
| Ingress | Port mapping directo | Traefik IngressRoute, mismo que prod |
| Storage | Bind mounts o volumes locales | PVC con local-path provisioner |
| Migraciones DB | Init scripts o entrypoint | Init containers en Deployment, igual que prod |
| GitOps | `git pull && docker compose up` | FluxCD reconciliation, igual que prod |

### 1.3 Cuándo este entorno tiene sentido

Este entorno es adecuado cuando:
- El laptop tiene **mínimo 8 GB RAM** (16 GB recomendado) y **mínimo 4 cores**
- Se dispone de una **máquina de desarrollo separada** (o se usa el patrón de desarrollo remoto descrito aquí)
- Se quiere aprender y dominar el stack de producción completo desde el primer día
- Se quiere que el proceso de migración a VPS sea trivial (solo cambiar IPs y secretos)

### 1.4 Topología de la estación de trabajo

```
┌─────────────────────────────────────────────────────────────────┐
│ RED LOCAL (ej: 192.168.1.0/24)                                  │
│                                                                  │
│  ┌───────────────────────────────┐    ┌────────────────────────┐│
│  │ LAPTOP DE DESARROLLO          │    │ LAPTOP TALOS (k8s node)││
│  │ (macOS / Linux / Windows)     │    │ Talos Linux v1.10.x    ││
│  │                               │    │ Kubernetes 1.33.x      ││
│  │ herramientas:                 │    │                        ││
│  │  - talosctl                   │    │ IP: 192.168.1.100      ││
│  │  - kubectl                    │    │ (o IP que asigne DHCP) ││
│  │  - helm                       │    │                        ││
│  │  - flux CLI                   │    │ Ports expuestos:       ││
│  │  - go 1.25.x                  │    │  :50000 (talosctl API) ││
│  │  - node / bun                 │    │  :6443  (k8s API)      ││
│  │  - IDE (VS Code / GoLand)     │    │  :80    (HTTP/Traefik) ││
│  │  - docker (para builds)       │    │  :443   (HTTPS/Traefik)││
│  │                               │    │  :5000  (registry local││
│  │ IP: 192.168.1.50              │    │                        ││
│  └───────────────────────────────┘    └────────────────────────┘│
│                                                                  │
│  Acceso a servicios desde laptop de desarrollo:                  │
│   https://api.serenamente.local → 192.168.1.100:443             │
│   https://app.serenamente.local → 192.168.1.100:443             │
│   https://kratos.serenamente.local → 192.168.1.100:443          │
└─────────────────────────────────────────────────────────────────┘
```

**Nota importante — máquina única:** Si solo tienes el laptop con Talos (sin otra máquina de desarrollo), las secciones [20.3 Desarrollo en Máquina Única](#203-desarrollo-en-máquina-única) describen cómo usar un pod de desarrollo dentro del propio cluster para correr el IDE y las herramientas.

---

## 2. Arquitectura del Entorno Local

### 2.1 Stack completo adaptado para desarrollo

```
CLOUDFLARE EDGE (simulado en local)
  CF Pages (Qwik SPA)    → localhost:5173  (vite dev server)
  CF Workers (BFF Hono)  → localhost:8787  (wrangler dev)

LAPTOP TALOS — Kubernetes 1.33.x (192.168.1.100)
  Namespace: traefik
    Traefik v3.x (DaemonSet, hostPort :80/:443)
      TLS: cert-manager con CA self-signed local
      ForwardAuth → iam-service

  Namespace: cert-manager
    cert-manager v1.x
    ClusterIssuer: local-ca (self-signed, NO Let's Encrypt)

  Namespace: sealed-secrets
    Sealed Secrets v0.27+ (clave de desarrollo, distinta a prod)

  Namespace: serenamente-data
    CloudNativePG v1.x (PostgreSQL 17.4)
      Storage: local-path PVC (disco del laptop)
      Backup: MinIO local (sustituto de B2)
    MinIO (sustituto de Backblaze B2)

  Namespace: serenamente-core
    Ory Kratos v1.3.1
      SMTP: Mailpit (no SendGrid)
      URLs: *.serenamente.local
    IAM Domain Service Go
      Imagen: registry.serenamente.local/iam-service:dev

  Namespace: serenamente-dev
    Mailpit (servidor SMTP + webUI de emails)
    Registry (registro local de imágenes Docker)

  Namespace: flux-system
    FluxCD v2.x (sincronizando rama dev del repo)
```

### 2.2 Dominios locales usados

Todos los servicios usan el dominio `.serenamente.local`. Este dominio se resuelve únicamente en tu red local mediante entradas en `/etc/hosts` del laptop de desarrollo.

| Dominio | Servicio | Puerto | Notas |
|---------|----------|--------|-------|
| `api.serenamente.local` | Traefik → todos los microservicios | 443 | Punto de entrada principal |
| `app.serenamente.local` | Qwik SPA dev server | 5173 | Directo, sin Traefik |
| `kratos.serenamente.local` | Ory Kratos UI flows | 443 | Via Traefik |
| `mail.serenamente.local` | Mailpit webUI | 443 | Via Traefik |
| `registry.serenamente.local` | Registry local | 5000 | Sin TLS (inseguro, ok para dev) |
| `minio.serenamente.local` | MinIO console | 443 | Via Traefik |

---

## 3. Diferencias entre Entorno Local y Producción (VPS)

Esta tabla es la referencia maestra para saber **exactamente qué cambia** y por qué. Cada diferencia tiene una justificación y las secciones correspondientes explican la implementación.

| Componente | Producción (Hetzner CX32) | Desarrollo (Laptop Talos) | Justificación |
|------------|--------------------------|--------------------------|---------------|
| **Hardware** | Hetzner CX32 — 4 vCPU, 8GB RAM, 80GB SSD, Ubuntu off | Laptop físico — variable | Dev: hardware existente |
| **IP del nodo** | IP pública fija (Floating IP) | IP privada LAN (ej: 192.168.1.100) | Dev: sin exposición a internet |
| **Acceso externo** | DNS público, CF Proxy, internet | `/etc/hosts` en dev machine, LAN only | Dev: sin exposición a internet |
| **Hetzner CCM** | Sí (Hetzner Cloud Controller Manager) | No (no hay Hetzner cloud) | Dev: irrelevante |
| **cert-manager ClusterIssuer** | `letsencrypt-prod` (ACME HTTP-01) | `local-ca` (self-signed, CA local) | Dev: sin dominio público |
| **Certificados TLS** | Let's Encrypt (confianza global) | CA auto-firmada (instalar en OS/browsers) | Dev: requiere confiar en CA local |
| **Traefik Service type** | `LoadBalancer` (Hetzner LB) | `hostPort :80/:443` o `NodePort` | Dev: sin cloud LB |
| **Storage class** | `hcloud-volumes` (CSI de Hetzner) | `local-path` (Rancher, incluido en Talos) | Dev: disco local del laptop |
| **FluxCD branch** | `main` | `dev` o rama feature | Dev: aislamiento del código |
| **Backups B2** | barman-cloud → Backblaze B2 real | barman-cloud → MinIO local | Dev: sin cuenta B2 necesaria |
| **SMTP / email** | SendGrid (real) | Mailpit (intercepta y muestra en webUI) | Dev: no mandar emails reales |
| **Kratos URLs base** | `https://api.serenamente.com/auth/` | `https://api.serenamente.local/auth/` | Dev: dominio local |
| **Qwik SPA** | CF Pages (deploy automático) | `vite dev` en localhost:5173 | Dev: hot-reload inmediato |
| **BFF Hono** | CF Workers (edge global) | `wrangler dev` en localhost:8787 | Dev: hot-reload inmediato |
| **Sealed Secrets** | Clave pública de producción | Clave pública de desarrollo (diferente) | Seguridad: claves separadas por entorno |
| **Registry de imágenes** | ghcr.io (GHCR público/privado) | Registry local en cluster | Dev: sin push a internet |
| **Postgres datos** | Datos reales (vacío en inicio) | Datos de prueba (fixtures) | Dev: datos controlados |
| **JWT secret** | Clave Ed25519 de producción (Sealed) | Clave Ed25519 de desarrollo (Sealed dev) | Seguridad: claves separadas |
| **Logs** | JSON a stdout, Loki (Fase 3) | JSON a stdout, kubectl logs | Dev: sin PLG stack pesado |
| **Talos schematic** | Con qemu-guest-agent (Hetzner) | Sin qemu-guest-agent (bare metal laptop) | Dev: hardware diferente |

---

## 4. Prerrequisitos

### 4.1 Hardware del laptop Talos (mínimos)

| Recurso | Mínimo | Recomendado | Notas |
|---------|--------|-------------|-------|
| **RAM** | **8 GB** | 16 GB | K8s control plane: ~850MB-1.1GB; Fase 1 stack completo: ~3.5GB |
| **CPU** | 4 cores (x86_64) | 6+ cores | Talos requiere x86_64 o arm64. AMD/Intel ambos válidos |
| **Disco** | 80 GB SSD | 120 GB NVMe | PostgreSQL + imágenes de contenedor consumen ~15GB |
| **Red** | 100 Mbps LAN | Gigabit | Para pull de imágenes desde internet en setup inicial |
| **BIOS** | VT-x / AMD-V habilitado | UEFI con Secure Boot desactivado | Talos requiere virtualización por hardware |
| **Arquitectura** | x86_64 (amd64) o arm64 | x86_64 | arm64 tiene soporte experimental en Talos 1.10 |

> **Sobre los 8 GB mínimos:** Con exactamente 8 GB el sistema funciona pero hay poca holgura. Si abres muchos pods simultáneamente (Kratos + IAM + PG + cert-manager + Traefik + FluxCD + registry) podrías llegar al 85% de memoria. Con 16 GB tienes comodidad total para toda la Fase 1 más herramientas de debugging.

### 4.2 Software en la máquina de desarrollo (NO en el laptop Talos)

El laptop con Talos no tiene shell, no tiene apt, no tiene nada que instalar. Todo el tooling va en tu máquina de desarrollo (el otro equipo con el que controlas el cluster).

```bash
# --- talosctl (gestión del OS Talos) ---
# macOS:
brew install siderolabs/tap/talosctl
# Linux (curl):
curl -sL https://talos.dev/install | sh
# Verificar:
talosctl version --client

# --- kubectl (gestión de Kubernetes) ---
# macOS:
brew install kubectl
# Linux:
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
# Verificar:
kubectl version --client

# --- Helm v3 (gestor de charts) ---
# macOS:
brew install helm
# Linux:
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
# Verificar:
helm version

# --- Flux CLI (GitOps controller CLI) ---
# macOS:
brew install fluxcd/tap/flux
# Linux:
curl -s https://fluxcd.io/install.sh | sudo bash
# Verificar:
flux version

# --- kubeseal (Sealed Secrets CLI) ---
# macOS:
brew install kubeseal
# Linux:
KUBESEAL_VERSION=$(curl -s https://api.github.com/repos/bitnami-labs/sealed-secrets/tags | jq -r '.[0].name[1:]')
curl -OL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz"
tar -xzf kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz kubeseal
sudo mv kubeseal /usr/local/bin/

# --- mkcert (para instalar CA local en el OS) ---
# macOS:
brew install mkcert
# Linux:
curl -JLO "https://dl.filippo.io/mkcert/latest?for=linux/amd64"
chmod +x mkcert-v*-linux-amd64 && sudo mv mkcert-v*-linux-amd64 /usr/local/bin/mkcert

# --- k9s (TUI para explorar el cluster) ---
# macOS:
brew install k9s
# Linux:
curl -sS https://webinstall.dev/k9s | bash

# --- Go 1.25.x ---
# https://go.dev/dl/ - descargar tarball correspondiente al OS/arch

# --- buf CLI (para Protobuf) ---
brew install bufbuild/buf/buf  # macOS
# Linux: https://buf.build/docs/installation

# --- wrangler (para CF Workers local) ---
npm install -g wrangler

# --- docker (para builds de imágenes) ---
# Instalar Docker Desktop o Docker Engine según el OS de desarrollo
```

### 4.3 Repositorio Git

El monorepo debe existir en GitHub (o similar) antes de proceder. FluxCD necesita acceso al repositorio para sincronizar.

```bash
# Estructura esperada del monorepo:
serenidad-platform/
├── apps/
│   ├── web/                  → Qwik v2.0 SPA
│   └── bff/                  → Hono v4.x CF Workers
├── services/
│   ├── iam/                  → Go IAM Domain Service
│   ├── scheduling/           → (Fase 2)
│   ├── clinical/             → (Fase 2)
│   └── billing/              → (Fase 3)
├── packages/
│   └── events/               → .proto schemas
└── infra/
    ├── clusters/
    │   ├── hetzner-prod/     → Configuración VPS (Fase futura)
    │   └── laptop-dev/       → Configuración laptop (este documento)
    ├── infrastructure/       → HelmReleases de infraestructura
    ├── apps/                 → Deployments de aplicaciones
    └── secrets/              → SealedSecrets (cifrados, seguros en Git)
```

---

## 5. Tarea L-01 — Instalar Talos Linux en el Laptop

### 5.1 Descargar el ISO de Talos

Talos no usa el schematic con `qemu-guest-agent` (ese es solo para Hetzner). Para bare metal normal se usa el ISO estándar o uno personalizado con los drivers de tu hardware.

```bash
# Obtener la última versión de Talos
TALOS_VERSION=$(curl -s https://api.github.com/repos/siderolabs/talos/releases/latest | jq -r .tag_name)
echo "Última versión: $TALOS_VERSION"

# Opción A: ISO estándar (funciona en la mayoría de laptops modernos)
curl -LO "https://github.com/siderolabs/talos/releases/download/${TALOS_VERSION}/metal-amd64.iso"

# Opción B: Imagen con controladores adicionales (WiFi, NIC específica)
# Usar el Talos Image Factory: https://factory.talos.dev/
# Seleccionar: bare-metal, amd64, versión 1.10.x
# Añadir extensiones: intel-ucode, amd-ucode, i915-ucode (Intel GPU), etc.
# Descarga ISO personalizado con los drivers necesarios
```

> **Importante sobre WiFi en laptops:** Talos NO soporta WiFi para el tráfico de datos del cluster (requiere drivers propietarios que no incluye). El laptop Talos **debe conectarse por cable Ethernet** a tu red local. Alternativamente, usa un adaptador USB-Ethernet.

### 5.2 Crear USB booteable

```bash
# macOS:
# Encontrar el device del USB:
diskutil list
# Grabar (reemplazar /dev/disk2 con tu device):
sudo dd if=metal-amd64.iso of=/dev/rdisk2 bs=4m status=progress

# Linux:
# Encontrar device:
lsblk
# Grabar:
sudo dd if=metal-amd64.iso of=/dev/sdb bs=4M status=progress oflag=sync

# Windows: usar Rufus o Balena Etcher
```

### 5.3 Configurar BIOS/UEFI del laptop

Antes de bootear desde el USB, configurar en el BIOS:

1. **Desactivar Secure Boot** — Talos funciona con Secure Boot pero requiere enrollar su propio certificado; para desarrollo es más simple desactivarlo.
2. **Habilitar VT-x / AMD-V** — Virtualización por hardware. Generalmente ya está habilitado.
3. **Boot order:** USB primero.
4. **CSM (Legacy BIOS):** Desactivar — usar UEFI puro.
5. **Fast Boot:** Desactivar — puede interferir con el boot desde USB.

### 5.4 Boot y modo de instalación

Al bootear desde el USB de Talos, el sistema entra en **modo mantenimiento (maintenance mode)**. En este estado:
- No hay UI, no hay shell interactivo
- El sistema escucha en el puerto `:50000` esperando que `talosctl apply-config` le envíe la configuración
- La IP se asigna via DHCP (verificar en tu router qué IP asignó)

```bash
# Desde tu máquina de desarrollo, en la misma red:
# Descubrir la IP del laptop Talos (buscar dispositivo con hostname "talos-*" en el router)
# O usar nmap para encontrar el puerto 50000:
nmap -p 50000 192.168.1.0/24
# El host que responda en :50000 es el laptop Talos en maintenance mode
```

### 5.5 Instalar Talos en el disco del laptop

Una vez identificada la IP del laptop en maintenance mode, se aplica la configuración. **Esto borrará todo el disco del laptop** — Talos se instala desde la configuración, no desde el ISO. El ISO solo sirve para bootstrap.

```bash
# Identificar el disco donde instalar (generalmente /dev/sda o /dev/nvme0n1)
# Verificar con:
talosctl disks --insecure --nodes <LAPTOP_IP>
# Ejemplo output:
# DEV             MODEL              SIZE     TYPE  ROTATIONAL
# /dev/nvme0n1    Samsung 970 EVO    512 GB   nvme  false

# La configuración del nodo especificará el disco.
# Esto se hace en la siguiente tarea (L-02).
```

---

## 6. Tarea L-02 — Bootstrap de Talos + Kubernetes

### 6.1 Crear la estructura de directorios para el entorno local

```bash
mkdir -p infra/clusters/laptop-dev/talos/patches
mkdir -p infra/clusters/laptop-dev/flux-system
```

### 6.2 Patch de configuración para laptop (DIFERENTE al de producción)

```yaml
# infra/clusters/laptop-dev/talos/patches/laptop-dev.yaml
machine:
  # Configuración de red estática (recomendado para estabilidad de desarrollo)
  network:
    hostname: serenamente-dev-01
    interfaces:
      - interface: eth0   # Ajustar al nombre de interfaz del laptop (ip link show)
        addresses:
          - 192.168.1.100/24   # IP estática deseada — ajustar al rango de tu red
        routes:
          - network: 0.0.0.0/0
            gateway: 192.168.1.1   # Gateway de tu router
        dhcp: false
    nameservers:
      - 1.1.1.1
      - 8.8.8.8

  # Especificar disco de instalación
  install:
    disk: /dev/nvme0n1   # Ajustar al disco del laptop (usar talosctl disks para verificar)
    bootloader: true
    wipe: true   # ¡BORRARÁ EL DISCO! Solo para instalación inicial

  # Configuración de registry local (definido en Tarea L-03)
  registries:
    mirrors:
      registry.serenamente.local:
        endpoints:
          - "http://registry.serenamente.local:5000"
    config:
      registry.serenamente.local:
        tls:
          insecureSkipVerify: true   # Registry local sin TLS

  # Tiempo (importante para certificados TLS)
  time:
    servers:
      - pool.ntp.org
      - time.cloudflare.com

cluster:
  # Single-node: el control plane también ejecuta pods de trabajo
  allowSchedulingOnControlPlanes: true

  # Sin Hetzner CCM (no es cloud Hetzner)
  # Sin extraManifests para cloud providers

  # Configuración de la red de pods
  network:
    podSubnets:
      - 10.244.0.0/16
    serviceSubnets:
      - 10.96.0.0/12
    cni:
      name: flannel   # Ligero, suficiente para single-node dev
```

> **Sobre la IP estática:** Es crítico que el laptop Talos tenga una IP fija en tu red. Si tiene DHCP dinámico, cada vez que reinicie podría cambiar de IP y romperá el kubeconfig, el talosconfig, y las entradas en `/etc/hosts`. Configura una reserva DHCP en tu router para la MAC del laptop, O usa IP estática en el patch.

### 6.3 Generar configuración del cluster

```bash
# Desde la raíz del monorepo, en tu máquina de desarrollo:
LAPTOP_IP="192.168.1.100"   # IP estática configurada en el patch

# Generar secrets únicos para el cluster dev (SOLO UNA VEZ)
talosctl gen secrets \
  --output-file infra/clusters/laptop-dev/talos/secrets.yaml

# IMPORTANTE: secrets.yaml contiene claves privadas del cluster.
# Añadir a .gitignore O cifrar con SOPS antes de commitear.
echo "infra/clusters/laptop-dev/talos/secrets.yaml" >> .gitignore

# Generar configuración del controlplane
talosctl gen config serenamente-dev "https://${LAPTOP_IP}:6443" \
  --with-secrets infra/clusters/laptop-dev/talos/secrets.yaml \
  --config-patch @infra/clusters/laptop-dev/talos/patches/laptop-dev.yaml \
  --output-dir infra/clusters/laptop-dev/talos/ \
  --force

# Genera:
#   infra/clusters/laptop-dev/talos/controlplane.yaml  ← configuración del nodo
#   infra/clusters/laptop-dev/talos/worker.yaml        ← para futuros workers (ignorar por ahora)
#   infra/clusters/laptop-dev/talos/talosconfig        ← credenciales talosctl
```

### 6.4 Aplicar configuración e instalar Talos

```bash
LAPTOP_IP="192.168.1.100"
TALOSCONFIG="infra/clusters/laptop-dev/talos/talosconfig"

# El laptop debe estar en maintenance mode (booteado desde USB)
# Aplicar configuración (esto instala Talos en el disco y reinicia)
talosctl apply-config \
  --insecure \
  --nodes $LAPTOP_IP \
  --file infra/clusters/laptop-dev/talos/controlplane.yaml

# El laptop se reiniciará automáticamente (~2-3 min)
# Talos se instala en el disco y arranca desde el disco (no más USB)

# Esperar a que el nodo esté en estado "ready" (~3-5 min después del reboot)
talosctl --talosconfig $TALOSCONFIG --nodes $LAPTOP_IP version
# Cuando responda, continuar:

# Bootstrap etcd (SOLO una vez, primer arranque del cluster)
talosctl bootstrap \
  --talosconfig $TALOSCONFIG \
  --nodes $LAPTOP_IP

# Esperar ~1-2 min para que etcd arranque y el API server esté disponible
```

### 6.5 Obtener kubeconfig

```bash
LAPTOP_IP="192.168.1.100"
TALOSCONFIG="infra/clusters/laptop-dev/talos/talosconfig"

# Obtener kubeconfig
talosctl kubeconfig \
  --talosconfig $TALOSCONFIG \
  --nodes $LAPTOP_IP \
  --output infra/clusters/laptop-dev/kubeconfig \
  --force

# Verificar que el cluster está operativo
kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig get nodes
# Output esperado:
# NAME                  STATUS   ROLES           AGE   VERSION
# serenamente-dev-01    Ready    control-plane   5m    v1.33.x

# Verificar pods del sistema
kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig get pods -A
# Todos los pods de kube-system deben estar Running

# Alias útil para desarrollo (añadir a ~/.bashrc o ~/.zshrc):
alias kdev='kubectl --kubeconfig ~/serenidad-platform/infra/clusters/laptop-dev/kubeconfig'
alias tdev='talosctl --talosconfig ~/serenidad-platform/infra/clusters/laptop-dev/talos/talosconfig --nodes 192.168.1.100'
```

### 6.6 Crear namespaces

```bash
KUBECONFIG="infra/clusters/laptop-dev/kubeconfig"

# Crear todos los namespaces necesarios
kubectl --kubeconfig $KUBECONFIG apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: traefik
---
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager
---
apiVersion: v1
kind: Namespace
metadata:
  name: sealed-secrets
---
apiVersion: v1
kind: Namespace
metadata:
  name: cnpg-system
---
apiVersion: v1
kind: Namespace
metadata:
  name: serenamente-data
---
apiVersion: v1
kind: Namespace
metadata:
  name: serenamente-core
---
apiVersion: v1
kind: Namespace
metadata:
  name: serenamente-dev
EOF
```

### 6.7 Verificar el estado del laptop Talos

```bash
# Estado de salud general
talosctl health \
  --talosconfig infra/clusters/laptop-dev/talos/talosconfig \
  --nodes 192.168.1.100

# Información del nodo
talosctl get nodestatus \
  --talosconfig infra/clusters/laptop-dev/talos/talosconfig \
  --nodes 192.168.1.100

# Ver logs del kernel (útil para diagnosticar problemas de drivers)
talosctl dmesg \
  --talosconfig infra/clusters/laptop-dev/talos/talosconfig \
  --nodes 192.168.1.100 | tail -50

# Temperatura y recursos del hardware
talosctl get cpufreqscalingcontrollers \
  --talosconfig infra/clusters/laptop-dev/talos/talosconfig \
  --nodes 192.168.1.100
```

**Criterio de aceptación L-01 + L-02:**
- `kubectl get nodes` → `serenamente-dev-01 Ready control-plane`
- `talosctl health` sin errores
- La IP del laptop es estable (no cambia entre reinicios)
- Todos los pods de `kube-system` en estado `Running`

---

## 7. Tarea L-03 — Registro Local de Imágenes de Contenedor

Para desarrollo necesitas poder construir imágenes de tus microservicios Go y desplegarlas en el cluster sin pushear a internet en cada cambio. La solución es un registry privado corriendo dentro del propio cluster.

### 7.1 ¿Por qué un registry local?

- **Velocidad:** Push/pull de imágenes en la red local (Gbps) vs internet (limitado por uplink)
- **Privacidad:** Las imágenes de desarrollo nunca salen de tu red
- **Funciona offline:** Desarrollo sin internet
- **Idéntico al flujo prod:** Prod usa ghcr.io; dev usa registry.serenamente.local:5000. Los Deployments solo cambian la URL del registry.

### 7.2 Desplegar el registry

```yaml
# infra/clusters/laptop-dev/registry/registry-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: local-registry
  namespace: serenamente-dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: local-registry
  template:
    metadata:
      labels:
        app: local-registry
    spec:
      containers:
      - name: registry
        image: registry:2.8
        ports:
        - containerPort: 5000
          hostPort: 5000   # Exponer directamente en el laptop IP:5000
        env:
        - name: REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY
          value: /var/lib/registry
        - name: REGISTRY_HTTP_ADDR
          value: ":5000"
        volumeMounts:
        - name: registry-data
          mountPath: /var/lib/registry
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
      volumes:
      - name: registry-data
        persistentVolumeClaim:
          claimName: registry-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: registry-pvc
  namespace: serenamente-dev
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: local-registry
  namespace: serenamente-dev
spec:
  selector:
    app: local-registry
  ports:
  - port: 5000
    targetPort: 5000
    nodePort: 30500
  type: NodePort
```

```bash
kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  apply -f infra/clusters/laptop-dev/registry/registry-deployment.yaml
```

### 7.3 Configurar Docker en la máquina de desarrollo para usar el registry inseguro

El registry local no tiene TLS. Docker necesita saber que está bien usarlo.

```json
// Añadir a /etc/docker/daemon.json (Linux) o Docker Desktop settings:
{
  "insecure-registries": ["192.168.1.100:5000", "registry.serenamente.local:5000"]
}
```

```bash
# Reiniciar Docker Desktop o:
sudo systemctl restart docker

# Añadir entrada en /etc/hosts en la máquina de DESARROLLO:
echo "192.168.1.100  registry.serenamente.local" | sudo tee -a /etc/hosts

# Test: push una imagen de prueba
docker pull hello-world
docker tag hello-world registry.serenamente.local:5000/hello-world:test
docker push registry.serenamente.local:5000/hello-world:test

# Verificar que el registry la almacenó:
curl http://registry.serenamente.local:5000/v2/hello-world/tags/list
# → {"name":"hello-world","tags":["test"]}
```

### 7.4 Script de build y push para desarrollo

Crear un script de conveniencia para el flujo build → push → rollout:

```bash
#!/bin/bash
# scripts/dev-push.sh
# Uso: ./scripts/dev-push.sh <servicio> [tag]
# Ejemplo: ./scripts/dev-push.sh iam latest

SERVICE=$1
TAG=${2:-dev}
REGISTRY="registry.serenamente.local:5000"
KUBECONFIG="infra/clusters/laptop-dev/kubeconfig"

set -euo pipefail

echo "→ Building ${SERVICE}..."
docker build \
  -t "${REGISTRY}/${SERVICE}:${TAG}" \
  -f "services/${SERVICE}/Dockerfile" \
  "services/${SERVICE}/"

echo "→ Pushing to local registry..."
docker push "${REGISTRY}/${SERVICE}:${TAG}"

echo "→ Rolling out in cluster..."
kubectl --kubeconfig "${KUBECONFIG}" rollout restart \
  deployment/${SERVICE} \
  -n serenamente-core

echo "→ Waiting for rollout..."
kubectl --kubeconfig "${KUBECONFIG}" rollout status \
  deployment/${SERVICE} \
  -n serenamente-core \
  --timeout=120s

echo "✓ ${SERVICE}:${TAG} deployed successfully"
```

```bash
chmod +x scripts/dev-push.sh
```

---

## 8. Tarea L-04 — FluxCD (GitOps Local)

FluxCD para desarrollo apunta a una rama `dev` (o `laptop-dev`) del monorepo. Esto permite commitear cambios de infra en esa rama sin afectar `main` (que usa producción).

### 8.1 Bootstrap FluxCD para el cluster dev

```bash
# Prerrequisito: GitHub token con permisos de repo:
export GITHUB_TOKEN=<tu_github_token>
export GITHUB_USER=<tu_usuario_github>

# Bootstrap FluxCD apuntando a la rama dev
flux bootstrap github \
  --owner="${GITHUB_USER}" \
  --repository=serenidad-platform \
  --branch=dev \
  --path=./infra/clusters/laptop-dev \
  --personal \
  --kubeconfig infra/clusters/laptop-dev/kubeconfig

# FluxCD creará:
#   - Namespace flux-system
#   - Controllers: source-controller, kustomize-controller, helm-controller
#   - GitRepository apuntando a la rama dev
#   - Kustomization que aplica todo en infra/clusters/laptop-dev/
```

### 8.2 Estructura de kustomization para laptop-dev

```yaml
# infra/clusters/laptop-dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - flux-system/
  - ../../infrastructure/cert-manager/
  - ../../infrastructure/traefik/
  - ../../infrastructure/cnpg/
  - ../../infrastructure/sealed-secrets/
  - ../../infrastructure/nats/      # Fase 2
  - ../../apps/iam-service/
  - registry/
  - dev-tools/                      # Mailpit, etc.
patches:
  # Patch para usar ClusterIssuer local en vez de letsencrypt
  - patch: |
      - op: replace
        path: /spec/issuerRef/name
        value: local-ca
    target:
      kind: Certificate
  # Patch para usar imágenes del registry local
  - patch: |
      - op: replace
        path: /spec/template/spec/containers/0/image
        value: registry.serenamente.local:5000/iam-service:dev
    target:
      kind: Deployment
      name: iam-service
```

> **Alternativa sin FluxCD:** Para máxima simplicidad durante el desarrollo temprano, puedes omitir FluxCD y aplicar manifests directamente con `kubectl apply -f`. FluxCD es valioso porque replica exactamente el flujo de producción (cambio en Git → auto-deploy), pero no es bloqueante para empezar.

---

## 9. Tarea L-05 — DNS Local y Resolución de Nombres

### 9.1 Opción A: `/etc/hosts` (más simple, suficiente para equipo individual)

```bash
# Añadir en la máquina de DESARROLLO (no en el laptop Talos):
LAPTOP_IP="192.168.1.100"

sudo tee -a /etc/hosts <<EOF

# Serenamente — entorno de desarrollo local
${LAPTOP_IP}  api.serenamente.local
${LAPTOP_IP}  app.serenamente.local
${LAPTOP_IP}  kratos.serenamente.local
${LAPTOP_IP}  mail.serenamente.local
${LAPTOP_IP}  minio.serenamente.local
${LAPTOP_IP}  registry.serenamente.local
EOF

# Verificar:
ping api.serenamente.local
# → PING api.serenamente.local (192.168.1.100)
```

### 9.2 Opción B: dnsmasq (si múltiples máquinas necesitan acceso)

Si tienes un equipo de desarrollo con múltiples máquinas que necesitan resolver `*.serenamente.local`:

```bash
# En macOS con Homebrew:
brew install dnsmasq

# Configurar:
echo "address=/.serenamente.local/192.168.1.100" >> /usr/local/etc/dnsmasq.conf

# Crear directorio de resolvers:
sudo mkdir -p /etc/resolver
echo "nameserver 127.0.0.1" | sudo tee /etc/resolver/serenamente.local

# Iniciar dnsmasq:
sudo brew services start dnsmasq

# Verificar:
dig api.serenamente.local @127.0.0.1
# → ;; ANSWER SECTION:
# → api.serenamente.local. 0 IN A 192.168.1.100
```

---

## 10. Tarea L-06 — cert-manager con ClusterIssuer Self-Signed

Esta es **la diferencia más importante** entre el entorno local y producción. En producción se usa Let's Encrypt (confianza global). En desarrollo se usa una CA auto-firmada que debes instalar en tu sistema operativo/browsers para que confíen en ella.

### 10.1 Instalar cert-manager (idéntico a producción)

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
      version: ">=1.14.0 <2.0.0"
      sourceRef:
        kind: HelmRepository
        name: jetstack
        namespace: flux-system
  values:
    installCRDs: true
    prometheus:
      enabled: true
      servicemonitor:
        enabled: false   # No PLG stack en dev
```

### 10.2 Crear el ClusterIssuer self-signed + CA local

```yaml
# infra/clusters/laptop-dev/cert-manager/local-ca-issuer.yaml
---
# 1. Issuer auto-firmado temporal (para crear la CA raíz)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
# 2. Certificado CA raíz de desarrollo (válido 10 años)
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: serenamente-dev-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: "Serenamente Dev CA"
  secretName: serenamente-dev-ca-secret
  duration: 87600h   # 10 años
  renewBefore: 720h  # renovar 30 días antes
  subject:
    organizations:
      - "Serenamente Dev"
    countries:
      - "PE"
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    kind: ClusterIssuer
    name: selfsigned-issuer
---
# 3. ClusterIssuer que usa la CA raíz para firmar todos los certificados
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: local-ca
spec:
  ca:
    secretName: serenamente-dev-ca-secret
```

```bash
kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  apply -f infra/clusters/laptop-dev/cert-manager/local-ca-issuer.yaml

# Verificar que el ClusterIssuer está listo:
kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  get clusterissuer local-ca
# → NAME       READY   AGE
# → local-ca   True    30s
```

### 10.3 Extraer e instalar la CA raíz en tu sistema

Para que Chrome, Firefox, curl, Go, y demás confíen en los certificados emitidos por la CA local:

```bash
KUBECONFIG="infra/clusters/laptop-dev/kubeconfig"

# Extraer el certificado CA
kubectl --kubeconfig $KUBECONFIG \
  get secret serenamente-dev-ca-secret \
  -n cert-manager \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > /tmp/serenamente-dev-ca.crt

# --- macOS: instalar en el keychain del sistema ---
sudo security add-trusted-cert \
  -d -r trustRoot \
  -k /Library/Keychains/System.keychain \
  /tmp/serenamente-dev-ca.crt
# Reiniciar Chrome/Safari para que tome efecto

# --- Ubuntu/Debian: ---
sudo cp /tmp/serenamente-dev-ca.crt /usr/local/share/ca-certificates/serenamente-dev-ca.crt
sudo update-ca-certificates
# Para Chrome: chrome://settings/certificates → Authorities → Import

# --- Windows: ---
certutil -addstore -f "ROOT" /tmp/serenamente-dev-ca.crt

# Verificar con curl:
curl https://api.serenamente.local/health
# → OK (sin error de certificado)
```

### 10.4 Certificate para api.serenamente.local

```yaml
# infra/clusters/laptop-dev/cert-manager/api-certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: api-serenamente-local-tls
  namespace: serenamente-core
spec:
  secretName: api-serenamente-local-tls
  duration: 2160h     # 90 días
  renewBefore: 360h   # Renovar 15 días antes
  commonName: "api.serenamente.local"
  dnsNames:
    - api.serenamente.local
    - kratos.serenamente.local
    - mail.serenamente.local
    - minio.serenamente.local
  issuerRef:
    kind: ClusterIssuer
    name: local-ca
```

---

## 11. Tarea L-07 — Traefik v3.x (acceso local vía hostPort)

En producción Traefik usa `Service type: LoadBalancer` (el CCM de Hetzner asigna una IP pública). En el laptop, no hay cloud controller, así que usamos `hostPort` en el DaemonSet para que Traefik escuche directamente en los puertos 80 y 443 de la IP del laptop.

```yaml
# infra/clusters/laptop-dev/traefik/helmrelease-dev.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: traefik
  namespace: flux-system
spec:
  interval: 1h
  url: https://traefik.github.io/charts
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: traefik
  namespace: traefik
spec:
  interval: 1h
  chart:
    spec:
      chart: traefik
      version: ">=30.0.0 <31.0.0"
      sourceRef:
        kind: HelmRepository
        name: traefik
        namespace: flux-system
  values:
    deployment:
      kind: DaemonSet   # Un pod por nodo (aquí solo hay un nodo)

    # DIFERENCIA CLAVE vs producción:
    # En dev usamos hostPort en vez de LoadBalancer
    service:
      type: ClusterIP   # No necesitamos servicio externo; usamos hostPort

    ports:
      web:
        port: 80
        hostPort: 80       # Escucha directamente en :80 del laptop
        expose:
          default: true
      websecure:
        port: 443
        hostPort: 443      # Escucha directamente en :443 del laptop
        expose:
          default: true
        tls:
          enabled: true

    # Redirigir HTTP → HTTPS
    additionalArguments:
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
      - "--log.level=DEBUG"   # Verbose logging para desarrollo

    # Sin Let's Encrypt (cert-manager gestiona los certs)
    certResolvers: {}

    # Dashboard de Traefik accesible en desarrollo
    ingressRoute:
      dashboard:
        enabled: true
        entryPoints:
          - websecure

    logs:
      general:
        level: DEBUG
      access:
        enabled: true

    metrics:
      prometheus:
        enabled: true

    # Tolerar que corra en el control plane (nodo único)
    tolerations:
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
```

### 11.1 IngressRoute adaptado para dominios locales

```yaml
# infra/clusters/laptop-dev/traefik/ingressroute-dev.yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: api-serenamente-local
  namespace: serenamente-core
spec:
  entryPoints:
    - websecure
  routes:
    - match: "Host(`api.serenamente.local`) && PathPrefix(`/auth`)"
      kind: Rule
      services:
        - name: ory-kratos-public
          port: 4433

    - match: "Host(`api.serenamente.local`) && PathPrefix(`/api/iam`)"
      kind: Rule
      middlewares:
        - name: iam-forward-auth
      services:
        - name: iam-service
          port: 8080

    - match: "Host(`api.serenamente.local`) && PathPrefix(`/api/scheduling`)"
      kind: Rule
      middlewares:
        - name: iam-forward-auth
      services:
        - name: scheduling-service
          port: 8081

    - match: "Host(`api.serenamente.local`) && PathPrefix(`/api/clinical`)"
      kind: Rule
      middlewares:
        - name: iam-forward-auth
      services:
        - name: clinical-service
          port: 8082

    - match: "Host(`api.serenamente.local`) && PathPrefix(`/api/billing`)"
      kind: Rule
      middlewares:
        - name: iam-forward-auth
      services:
        - name: billing-service
          port: 8083

    # Health check sin auth (para verificar que Traefik funciona)
    - match: "Host(`api.serenamente.local`) && Path(`/health`)"
      kind: Rule
      services:
        - name: iam-service
          port: 8080

  tls:
    secretName: api-serenamente-local-tls   # Creado por cert-manager con local-ca
---
# IngressRoute para Mailpit
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: mailpit-local
  namespace: serenamente-dev
spec:
  entryPoints:
    - websecure
  routes:
    - match: "Host(`mail.serenamente.local`)"
      kind: Rule
      services:
        - name: mailpit
          port: 8025
  tls:
    secretName: api-serenamente-local-tls
---
# IngressRoute para MinIO console
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: minio-local
  namespace: serenamente-dev
spec:
  entryPoints:
    - websecure
  routes:
    - match: "Host(`minio.serenamente.local`)"
      kind: Rule
      services:
        - name: minio
          port: 9001
  tls:
    secretName: api-serenamente-local-tls
```

---

## 12. Tarea L-08 — Sealed Secrets (clave de desarrollo)

Sealed Secrets tiene una clave pública/privada diferente en cada cluster. Los secretos sellados para el cluster dev **no se pueden abrir** en el cluster de producción, y viceversa. Esto es correcto por diseño.

```bash
# Instalar el controller de Sealed Secrets:
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update

kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  create namespace sealed-secrets || true

helm --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  install sealed-secrets \
  sealed-secrets/sealed-secrets \
  --namespace sealed-secrets \
  --version ">=0.27.0 <1.0.0"

# Esperar a que el controller esté listo:
kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  wait --for=condition=available \
  deployment/sealed-secrets \
  -n sealed-secrets \
  --timeout=120s

# Obtener la clave pública DEL CLUSTER DEV para sellar secrets:
kubeseal --fetch-cert \
  --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  --controller-name=sealed-secrets \
  --controller-namespace=sealed-secrets \
  > infra/clusters/laptop-dev/sealed-secrets-public-key-DEV.pem

# IMPORTANTE: Esta clave pública es SOLO para el cluster dev.
# Commitear este archivo al repo (es seguro — es pública).
# La clave privada NUNCA sale del cluster.
```

### 12.1 Cómo sellar un secret para desarrollo

```bash
# Ejemplo: sellar el JWT private key para el IAM Service en dev
# Generar una clave Ed25519 de DESARROLLO:
openssl genpkey -algorithm ed25519 -out /tmp/iam-dev-private.pem
openssl pkey -in /tmp/iam-dev-private.pem -pubout -out /tmp/iam-dev-public.pem
BASE64_KEY=$(base64 -w0 /tmp/iam-dev-private.pem)

# Crear el secret y sellarlo:
kubectl create secret generic iam-service-secrets \
  --dry-run=client \
  --namespace serenamente-core \
  --from-literal=jwt_private_key_b64="${BASE64_KEY}" \
  -o yaml | \
kubeseal \
  --cert infra/clusters/laptop-dev/sealed-secrets-public-key-DEV.pem \
  --format yaml \
  > infra/secrets/dev/iam-service-secrets.sealed.yaml

# Commitear el SealedSecret al repo (es seguro)
# El cluster dev lo descifrará automáticamente con su clave privada

# Limpiar claves en claro:
rm /tmp/iam-dev-private.pem /tmp/iam-dev-public.pem
```

---

## 13. Tarea L-09 — CloudNativePG v1.x + PostgreSQL 17.4

Esta tarea es **casi idéntica a producción**. La única diferencia es que usamos `local-path` como storage class en vez del CSI de Hetzner.

### 13.1 Instalar el operador CloudNativePG

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
  name: cnpg
  namespace: cnpg-system
spec:
  interval: 1h
  chart:
    spec:
      chart: cloudnative-pg
      version: ">=0.21.0 <1.0.0"
      sourceRef:
        kind: HelmRepository
        name: cnpg
        namespace: flux-system
  values:
    monitoring:
      podMonitorEnabled: false   # Sin Prometheus en dev (Fase 3)
```

### 13.2 Cluster CRD para desarrollo

```yaml
# infra/clusters/laptop-dev/cnpg/cluster-dev.yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: serenamente-pg
  namespace: serenamente-data
spec:
  instances: 1
  imageName: ghcr.io/cloudnative-pg/postgresql:17.4

  postgresql:
    parameters:
      # Parámetros reducidos para dev (laptop tiene menos recursos que CX32)
      max_connections: "100"
      shared_buffers: "128MB"
      effective_cache_size: "512MB"
      maintenance_work_mem: "32MB"
      work_mem: "4MB"
      wal_buffers: "8MB"
      log_statement: "all"          # Loggear TODAS las queries en dev
      log_duration: "on"
      log_min_duration_statement: "0"  # Loggear todas, sin umbral
      log_line_prefix: "%t [%p]: user=%u,db=%d "
      jit: "off"

  bootstrap:
    initdb:
      database: postgres
      owner: postgres
      postInitSQL:
        # Crear las 6 databases + extensiones
        - "CREATE DATABASE iam_db ENCODING 'UTF8' LC_COLLATE='en_US.UTF-8' LC_CTYPE='en_US.UTF-8' TEMPLATE template0;"
        - "CREATE DATABASE scheduling_db ENCODING 'UTF8' LC_COLLATE='en_US.UTF-8' LC_CTYPE='en_US.UTF-8' TEMPLATE template0;"
        - "CREATE DATABASE clinical_db ENCODING 'UTF8' LC_COLLATE='en_US.UTF-8' LC_CTYPE='en_US.UTF-8' TEMPLATE template0;"
        - "CREATE DATABASE billing_db ENCODING 'UTF8' LC_COLLATE='en_US.UTF-8' LC_CTYPE='en_US.UTF-8' TEMPLATE template0;"
        - "CREATE DATABASE kratos_db ENCODING 'UTF8' LC_COLLATE='en_US.UTF-8' LC_CTYPE='en_US.UTF-8' TEMPLATE template0;"
        - "CREATE DATABASE openfga_db ENCODING 'UTF8' LC_COLLATE='en_US.UTF-8' LC_CTYPE='en_US.UTF-8' TEMPLATE template0;"
        - "\\c iam_db; CREATE EXTENSION IF NOT EXISTS pg_uuidv7; CREATE EXTENSION IF NOT EXISTS hstore; REVOKE ALL ON SCHEMA public FROM PUBLIC;"
        - "\\c scheduling_db; CREATE EXTENSION IF NOT EXISTS pg_uuidv7; CREATE EXTENSION IF NOT EXISTS btree_gist; REVOKE ALL ON SCHEMA public FROM PUBLIC;"
        - "\\c clinical_db; CREATE EXTENSION IF NOT EXISTS pg_uuidv7; REVOKE ALL ON SCHEMA public FROM PUBLIC;"
        - "\\c billing_db; CREATE EXTENSION IF NOT EXISTS pg_uuidv7; REVOKE ALL ON SCHEMA public FROM PUBLIC;"

  # DIFERENCIA vs prod: usar local-path en vez de hcloud-volumes
  storage:
    size: 10Gi
    storageClass: local-path   # Provisioner incluido en Talos

  # Sin backup a B2 en dev básico (ver Tarea L-15 para MinIO opcional)
  # backup:
  #   retentionPolicy: "7d"
  #   barmanObjectStore:
  #     destinationPath: "s3://serenamente-pg-backups/wal"
  #     endpointURL: "http://minio.serenamente-dev.svc.cluster.local:9000"
  #     ... (ver L-15 para configuración completa con MinIO)

  monitoring:
    enablePodMonitor: false   # Sin Prometheus en Fase 1 dev
```

### 13.3 Herramientas para inspeccionar PostgreSQL desde la máquina de desarrollo

```bash
KUBECONFIG="infra/clusters/laptop-dev/kubeconfig"
PG_POD=$(kubectl --kubeconfig $KUBECONFIG get pod \
  -n serenamente-data \
  -l cnpg.io/cluster=serenamente-pg,role=primary \
  -o name)

# Verificar que el cluster está listo:
kubectl --kubeconfig $KUBECONFIG \
  get cluster serenamente-pg -n serenamente-data
# → STATUS=Cluster in healthy state

# Listar databases:
kubectl --kubeconfig $KUBECONFIG exec \
  -n serenamente-data $PG_POD \
  -- psql -U postgres -c "\l"

# Verificar extensiones en iam_db:
kubectl --kubeconfig $KUBECONFIG exec \
  -n serenamente-data $PG_POD \
  -- psql -U postgres -d iam_db -c "\dx"

# Port-forward para usar herramientas GUI (TablePlus, DBeaver, etc.):
kubectl --kubeconfig $KUBECONFIG port-forward \
  -n serenamente-data $PG_POD 5432:5432

# Desde TablePlus o DBeaver:
#   Host: localhost:5432
#   User: postgres
#   Password: (obtener del secret)
#   Database: iam_db (o cualquiera de las 6)

# Obtener la contraseña del usuario postgres:
kubectl --kubeconfig $KUBECONFIG \
  get secret serenamente-pg-superuser \
  -n serenamente-data \
  -o jsonpath='{.data.password}' | base64 -d
```

---

## 14. Tarea L-10 — Mailpit (sustituto SMTP para desarrollo)

Ory Kratos necesita enviar emails (verificación, recuperación). En desarrollo NO queremos mandar emails reales. Mailpit intercepta todos los emails y los muestra en una WebUI.

```yaml
# infra/clusters/laptop-dev/dev-tools/mailpit.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mailpit
  namespace: serenamente-dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mailpit
  template:
    metadata:
      labels:
        app: mailpit
    spec:
      containers:
      - name: mailpit
        image: axllent/mailpit:latest
        ports:
        - name: smtp
          containerPort: 1025   # Puerto SMTP (Kratos enviará aquí)
        - name: ui
          containerPort: 8025   # WebUI para ver los emails interceptados
        env:
        - name: MP_MAX_MESSAGES
          value: "200"   # Guardar los últimos 200 emails
        - name: MP_UI_AUTH
          value: ""       # Sin auth en dev
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: mailpit
  namespace: serenamente-dev
spec:
  selector:
    app: mailpit
  ports:
  - name: smtp
    port: 1025
    targetPort: 1025
  - name: ui
    port: 8025
    targetPort: 8025
```

```bash
kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  apply -f infra/clusters/laptop-dev/dev-tools/mailpit.yaml

# Acceder a la WebUI (después de configurar DNS local e IngressRoute):
# https://mail.serenamente.local

# O directamente con port-forward:
kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  port-forward -n serenamente-dev \
  svc/mailpit 8025:8025 &
# → Abrir http://localhost:8025
```

---

## 15. Tarea L-11 — Ory Kratos v1.3.1 (adaptado para local)

La mayor diferencia con producción es:
1. URLs apuntan a `*.serenamente.local` en vez de `*.serenamente.com`
2. SMTP apunta a Mailpit en vez de SendGrid
3. La cookie de sesión usa `.serenamente.local` en vez de `.serenamente.com`

### 15.1 Sealed Secret para kratos-smtp-dev

```bash
# Sellar credenciales SMTP para Mailpit (no se necesita password real):
kubectl create secret generic kratos-smtp-dev \
  --dry-run=client \
  --namespace serenamente-core \
  --from-literal=smtp_uri="smtp://mailpit.serenamente-dev.svc.cluster.local:1025/?skip_ssl_verify=true&legacy_ssl=false" \
  -o yaml | \
kubeseal \
  --cert infra/clusters/laptop-dev/sealed-secrets-public-key-DEV.pem \
  --format yaml \
  > infra/secrets/dev/kratos-smtp-dev.sealed.yaml
```

### 15.2 HelmRelease de Kratos adaptado para dev

```yaml
# infra/clusters/laptop-dev/apps/kratos-dev.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: ory-kratos
  namespace: serenamente-core
spec:
  interval: 1h
  chart:
    spec:
      chart: kratos
      version: ">=0.42.0"
      sourceRef:
        kind: HelmRepository
        name: ory
        namespace: flux-system
  values:
    image:
      tag: "v1.3.1"

    kratos:
      config:
        version: v0.13.0

        identity:
          default_schema_id: patient
          schemas:
            - id: patient
              url: file:///etc/config/kratos/schemas/patient.json
            - id: doctor
              url: file:///etc/config/kratos/schemas/doctor.json

        selfservice:
          default_browser_return_url: "https://app.serenamente.local/"
          allowed_return_urls:
            - "https://app.serenamente.local/"
            - "http://localhost:5173/"   # Qwik dev server

          flows:
            login:
              ui_url: "http://localhost:5173/auth/login"
              lifespan: 1h   # Más largo en dev para no reloguear continuamente
            registration:
              ui_url: "http://localhost:5173/auth/registration"
              lifespan: 1h
            recovery:
              enabled: true
              ui_url: "http://localhost:5173/auth/recovery"
              use: code
            verification:
              enabled: true
              ui_url: "http://localhost:5173/auth/verification"
              use: code

          methods:
            passkey:
              enabled: true
              config:
                rp:
                  display_name: "Serenamente Dev"
                  id: "serenamente.local"
                  origins:
                    - "https://app.serenamente.local"
                    - "http://localhost:5173"   # Para desarrollo sin TLS en el SPA
            password:
              enabled: true   # Activar passwords en dev para facilitar testing
              # En producción: false

        session:
          cookie:
            domain: ".serenamente.local"
            same_site: Lax
            secure: true
          lifespan: 168h   # 7 días en dev (más cómodo)

        courier:
          smtp:
            connection_uri: "smtp://mailpit.serenamente-dev.svc.cluster.local:1025/"
            from_name: "Serenamente Dev"
            from_address: "dev@serenamente.local"

        log:
          level: debug   # Verbose logging en dev
          format: json
          leak_sensitive_values: true   # Mostrar valores sensibles en logs (SOLO DEV)

        serve:
          public:
            base_url: "https://api.serenamente.local/auth/"
            port: 4433
          admin:
            port: 4434

      automigration:
        enabled: true   # Ejecutar migraciones automáticamente al arrancar

    postgresql:
      enabled: false

    externalPostgresql:
      host: "serenamente-pg-rw.serenamente-data.svc.cluster.local"
      port: 5432
      database: "kratos_db"
      username: "postgres"
      existingSecret: "serenamente-pg-superuser"
      existingSecretKey: "password"

    # Schemas montados como ConfigMap
    extraVolumes:
      - name: kratos-schemas
        configMap:
          name: kratos-identity-schemas

    extraVolumeMounts:
      - name: kratos-schemas
        mountPath: /etc/config/kratos/schemas
        readOnly: true
```

### 15.3 ConfigMap con los identity schemas

```yaml
# infra/apps/kratos/schemas-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kratos-identity-schemas
  namespace: serenamente-core
data:
  patient.json: |
    {
      "$id": "https://api.serenamente.local/schemas/identity/patient.json",
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
            }
          },
          "required": ["email"]
        }
      }
    }
  doctor.json: |
    {
      "$id": "https://api.serenamente.local/schemas/identity/doctor.json",
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
              "title": "E-Mail",
              "ory.sh/kratos": {
                "credentials": {
                  "password": { "identifier": false },
                  "webauthn": { "identifier": true }
                },
                "verification": { "via": "email" },
                "recovery": { "via": "email" }
              }
            },
            "license_number": {
              "type": "string",
              "title": "Medical License Number"
            }
          },
          "required": ["email", "license_number"]
        }
      }
    }
```

---

## 16. Tarea L-12 — IAM Domain Service Go (con hot-reload)

### 16.1 Deployment adaptado para desarrollo

```yaml
# infra/clusters/laptop-dev/apps/iam-service-dev.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iam-service
  namespace: serenamente-core
  annotations:
    # Forzar re-deploy con cada push a la rama dev
    fluxcd.io/automated: "true"
    fluxcd.io/tag.iam-service: "glob:dev-*"
spec:
  replicas: 1   # Solo 1 replica en dev (ahorra RAM)
  selector:
    matchLabels:
      app: iam-service
  template:
    metadata:
      labels:
        app: iam-service
    spec:
      initContainers:
        # Init container: ejecutar migraciones antes de arrancar el servicio
        - name: migrate
          image: registry.serenamente.local:5000/iam-service:dev
          command: ["/app/iam-service", "migrate", "up"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: serenamente-pg-app
                  key: uri
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
      containers:
        - name: iam-service
          image: registry.serenamente.local:5000/iam-service:dev
          imagePullPolicy: Always   # Siempre pull en dev para tener la última versión
          ports:
            - name: http
              containerPort: 8080
          env:
            - name: ENV
              value: "development"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: serenamente-pg-app
                  key: uri
            - name: KRATOS_ADMIN_URL
              value: "http://ory-kratos-admin.serenamente-core.svc.cluster.local:4434"
            - name: KRATOS_PUBLIC_URL
              value: "http://ory-kratos-public.serenamente-core.svc.cluster.local:4433"
            - name: OPENFGA_URL
              value: "http://openfga.serenamente-core.svc.cluster.local:8080"   # Fase 3
            - name: NATS_URL
              value: "nats://nats.serenamente-data.svc.cluster.local:4222"   # Fase 2
            - name: JWT_PRIVATE_KEY_B64
              valueFrom:
                secretKeyRef:
                  name: iam-service-secrets
                  key: jwt_private_key_b64
            - name: LOG_LEVEL
              value: "debug"   # Verbose en dev
            - name: LOG_FORMAT
              value: "text"    # Legible en dev (vs json en prod)
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 3
            periodSeconds: 5
          resources:
            # Más permisivo en dev
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "256Mi"
              cpu: "500m"
      # Tolerar que corra en control-plane (nodo único)
      tolerations:
        - key: "node-role.kubernetes.io/control-plane"
          operator: "Exists"
          effect: "NoSchedule"
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
      targetPort: 8080
  type: ClusterIP
```

### 16.2 Ciclo de desarrollo del IAM Service

```bash
# Workflow diario para cambios en el IAM Service:

# 1. Editar código en tu IDE (en la máquina de desarrollo)
#    services/iam/...

# 2. Compilar y verificar que compila:
cd services/iam
go build ./cmd/...
go test ./...

# 3. Build imagen y push al registry local:
./scripts/dev-push.sh iam dev

# 4. Seguir los logs del pod actualizado:
kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  logs -n serenamente-core \
  -l app=iam-service \
  -f --since=5m

# 5. Port-forward para debugging directo (sin pasar por Traefik):
kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  port-forward -n serenamente-core \
  svc/iam-service 8080:8080 &

# 6. Testing directo:
curl -X POST http://localhost:8080/api/iam/token/exchange \
  -H "Content-Type: application/json" \
  -d '{"kratos_session_token":"test"}'

# 7. Ver eventos del namespace para debugging:
kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  get events -n serenamente-core \
  --sort-by='.lastTimestamp' | tail -20
```

### 16.3 Usar Skaffold para hot-reload automático (opcional pero muy recomendado)

Skaffold detecta cambios en el código, hace build automático y re-despliega en el cluster. Elimina el paso manual de `dev-push.sh`.

```yaml
# skaffold.yaml (en la raíz del monorepo)
apiVersion: skaffold/v4beta11
kind: Config
metadata:
  name: serenamente-dev

build:
  local:
    push: true
  artifacts:
    - image: registry.serenamente.local:5000/iam-service
      context: services/iam
      docker:
        dockerfile: Dockerfile
      sync:
        # Hot-reload de archivos de configuración sin rebuild completo
        manual:
          - src: "config/*.yaml"
            dest: /app/config/

deploy:
  kubectl:
    manifests:
      - infra/clusters/laptop-dev/apps/iam-service-dev.yaml
  kubeContext: "admin@serenamente-dev"   # Context del kubeconfig dev

portForward:
  - resourceType: service
    resourceName: iam-service
    namespace: serenamente-core
    port: 8080
    localPort: 8080
```

```bash
# Instalar Skaffold:
brew install skaffold  # macOS
# Linux: https://skaffold.dev/docs/install/

# Ejecutar en modo desarrollo:
# (detecta cambios automáticamente y re-despliega)
skaffold dev \
  --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  --port-forward

# Cada vez que guardes un archivo .go, Skaffold:
# 1. Detecta el cambio
# 2. Ejecuta go build
# 3. Hace docker build
# 4. Push al registry local
# 5. kubectl rollout restart
# Todo en ~15-30 segundos
```

---

## 17. Tarea L-13 — BFF Hono v4.x en local con Wrangler

El BFF de Cloudflare Workers no puede correr como un pod Kubernetes real en local (requiere el runtime V8 de Cloudflare). Usar `wrangler dev` que simula el entorno de CF Workers localmente.

```bash
# En la máquina de desarrollo:
cd apps/bff

# Instalar dependencias:
npm install

# Configurar para apuntar al cluster local:
# En wrangler.toml añadir:
```

```toml
# apps/bff/wrangler.toml
name = "serenamente-bff"
main = "src/index.ts"
compatibility_date = "2026-04-01"

[vars]
# En desarrollo, apuntar al API del cluster local:
API_BASE_URL = "https://api.serenamente.local"
ENVIRONMENT = "development"

# En producción (rama main):
# API_BASE_URL = "https://api.serenamente.com"
```

```bash
# Ejecutar BFF localmente:
cd apps/bff
wrangler dev --local

# wrangler dev escucha en localhost:8787
# El Qwik SPA en localhost:5173 hace fetch() a localhost:8787
```

---

## 18. Tarea L-14 — Qwik SPA en local

```bash
cd apps/web

# Instalar dependencias:
npm install

# Configurar el endpoint del BFF:
# En apps/web/.env.local:
echo "VITE_API_URL=http://localhost:8787" > .env.local
echo "VITE_KRATOS_URL=https://api.serenamente.local/auth" >> .env.local

# Ejecutar dev server:
npm run dev
# → Escucha en http://localhost:5173
# → Hot Module Replacement activo
# → Cambios en componentes Qwik aparecen al instante
```

> **Nota:** La SPA en dev no usa Cloudflare Pages. Se ejecuta directamente con Vite en tu máquina de desarrollo. El deploy a CF Pages solo ocurre cuando haces push a `main`.

---

## 19. Tarea L-15 — MinIO como sustituto de Backblaze B2

Esta tarea es **opcional** para desarrollo básico. Solo necesaria si quieres probar el backup y restore de CloudNativePG completo.

```yaml
# infra/clusters/laptop-dev/dev-tools/minio.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: serenamente-dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        image: minio/minio:latest
        command:
          - minio
          - server
          - /data
          - "--console-address"
          - ":9001"
        ports:
        - containerPort: 9000   # S3 API
        - containerPort: 9001   # WebUI console
        env:
        - name: MINIO_ROOT_USER
          value: "minioadmin"
        - name: MINIO_ROOT_PASSWORD
          value: "minioadmin123"  # Solo dev, sin secretos reales
        - name: MINIO_DEFAULT_BUCKETS
          value: "serenamente-pg-backups"
        volumeMounts:
        - name: minio-data
          mountPath: /data
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      volumes:
      - name: minio-data
        persistentVolumeClaim:
          claimName: minio-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-pvc
  namespace: serenamente-dev
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: serenamente-dev
spec:
  selector:
    app: minio
  ports:
  - name: s3api
    port: 9000
    targetPort: 9000
  - name: console
    port: 9001
    targetPort: 9001
```

### 19.1 Configurar CloudNativePG para usar MinIO en dev

Una vez MinIO está corriendo, actualizar el Cluster CRD para activar backup:

```yaml
# Añadir a infra/clusters/laptop-dev/cnpg/cluster-dev.yaml:
# (descomentar la sección backup)
  backup:
    retentionPolicy: "3d"   # Solo 3 días en dev (ahorra disco)
    barmanObjectStore:
      destinationPath: "s3://serenamente-pg-backups/wal"
      endpointURL: "http://minio.serenamente-dev.svc.cluster.local:9000"
      s3Credentials:
        accessKeyId:
          name: minio-credentials
          key: access_key_id
        secretAccessKey:
          name: minio-credentials
          key: secret_access_key
```

```bash
# Crear el Secret con las credenciales de MinIO (para dev, sin sellar):
kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  create secret generic minio-credentials \
  --namespace serenamente-data \
  --from-literal=access_key_id=minioadmin \
  --from-literal=secret_access_key=minioadmin123

# Forzar un backup manual para verificar:
kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  cnpg backup serenamente-pg \
  -n serenamente-data

# Ver el estado del backup:
kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  get backup -n serenamente-data

# Ver en la consola de MinIO:
# https://minio.serenamente.local → console
```

---

## 20. Flujo de Trabajo Diario

### 20.1 Inicio del día de desarrollo

```bash
# 1. Verificar que el laptop Talos está up y el cluster saludable:
kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig get nodes
# → serenamente-dev-01   Ready

# 2. Verificar que todos los pods están corriendo:
kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig get pods -A
# Buscar pods en estado distinto a Running/Completed

# 3. Si FluxCD está activo, ver estado de sincronización:
flux --kubeconfig infra/clusters/laptop-dev/kubeconfig get all

# 4. Arrancar herramientas de desarrollo:
# Terminal 1 — BFF:
cd apps/bff && wrangler dev --local

# Terminal 2 — SPA:
cd apps/web && npm run dev

# Terminal 3 — Port-forward para acceso directo a PostgreSQL (si necesario):
kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  port-forward -n serenamente-data \
  $(kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig get pod \
    -n serenamente-data -l cnpg.io/cluster=serenamente-pg,role=primary \
    -o name) 5432:5432 &

# Terminal 4 — k9s (monitor del cluster):
k9s --kubeconfig infra/clusters/laptop-dev/kubeconfig
```

### 20.2 Ciclo de desarrollo de un microservicio Go

```bash
# Fase 1: Escribir código
vim services/iam/internal/domain/user.go

# Fase 2: Ejecutar tests unitarios (RÁPIDO — no requiere cluster):
cd services/iam
go test ./internal/... -v

# Fase 3: Build imagen y desplegar en cluster local:
./scripts/dev-push.sh iam dev

# Fase 4: Ver logs en tiempo real:
kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  logs -n serenamente-core \
  -l app=iam-service \
  -f --tail=100

# Fase 5: Testing de integración via API:
# (Con Traefik corriendo)
TOKEN=$(curl -s -X POST https://api.serenamente.local/api/iam/token/exchange \
  -H "Content-Type: application/json" \
  -d '{"kratos_session_token":"<kratos-session>"}' | jq -r .token)

curl -H "Authorization: Bearer $TOKEN" \
  https://api.serenamente.local/api/iam/users/me

# Fase 6: Si hay errores, inspeccionar directamente:
kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  exec -n serenamente-core \
  -it $(kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig \
    get pod -n serenamente-core -l app=iam-service -o name | head -1) \
  -- /bin/sh
# Nota: el contenedor debe tener un shell (Alpine base image)
```

### 20.3 Desarrollo en Máquina Única (el laptop Talos ES tu única máquina)

Si solo tienes el laptop con Talos y necesitas desarrollar desde él mismo, existen dos estrategias:

**Estrategia A: Pod de desarrollo dentro del cluster**

```yaml
# infra/clusters/laptop-dev/dev-tools/dev-pod.yaml
# Un pod con todas las herramientas de desarrollo:
apiVersion: v1
kind: Pod
metadata:
  name: dev-workspace
  namespace: serenamente-dev
spec:
  containers:
  - name: dev
    image: golang:1.25-bookworm
    command: ["sleep", "infinity"]
    workingDir: /workspace
    volumeMounts:
    - name: workspace
      mountPath: /workspace
    - name: kubeconfig
      mountPath: /root/.kube
      readOnly: true
    resources:
      requests:
        memory: "1Gi"
        cpu: "500m"
      limits:
        memory: "4Gi"
        cpu: "2000m"
  volumes:
  - name: workspace
    persistentVolumeClaim:
      claimName: dev-workspace-pvc
  - name: kubeconfig
    secret:
      secretName: dev-kubeconfig
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dev-workspace-pvc
  namespace: serenamente-dev
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: local-path
  resources:
    requests:
      storage: 20Gi
```

```bash
# Aplicar el pod de desarrollo:
kubectl apply -f infra/clusters/laptop-dev/dev-tools/dev-pod.yaml

# Entrar al pod:
kubectl exec -it -n serenamente-dev dev-workspace -- bash

# Dentro del pod: instalar herramientas y clonar repo:
apt-get update && apt-get install -y git curl nodejs npm
curl -sL https://talos.dev/install | sh
# Clonar el monorepo:
git clone https://github.com/serenamente/serenidad-platform /workspace

# Desde dentro del pod puedes hacer kubectl, go build, etc.
```

**Estrategia B: VS Code Remote + talosctl machineconfig con SSH habilitado (no recomendado para producción)**

> Esta estrategia compromete el principio de zero-SSH de Talos. Solo usar en dev, nunca en producción.

---

## 21. Herramientas de Diagnóstico y Observabilidad Ligera

En desarrollo no se despliega el stack completo de observabilidad (Prometheus + Loki + Tempo + Grafana). Estas herramientas ligeras son suficientes:

### 21.1 k9s — TUI para el cluster

```bash
# Instalar:
brew install k9s  # macOS
# Linux: https://k9scli.io/topics/install/

# Usar:
k9s --kubeconfig infra/clusters/laptop-dev/kubeconfig

# Comandos útiles dentro de k9s:
# :pods → listar todos los pods
# :svc → listar todos los services
# :pvc → listar persistent volume claims
# :events → ver events del cluster
# d → describe el recurso seleccionado
# l → ver logs del pod seleccionado
# e → editar el recurso seleccionado
# ctrl+d → delete recurso
# / → filtrar por nombre
```

### 21.2 Port-forwards útiles para debugging

```bash
#!/bin/bash
# scripts/port-forwards.sh
# Arrancar todos los port-forwards de desarrollo de una vez

KUBECONFIG="infra/clusters/laptop-dev/kubeconfig"
SERENAMENTE_PG_POD=$(kubectl --kubeconfig $KUBECONFIG get pod \
  -n serenamente-data \
  -l cnpg.io/cluster=serenamente-pg,role=primary \
  -o name 2>/dev/null | head -1)

# PostgreSQL (para TablePlus/DBeaver/psql directo)
kubectl --kubeconfig $KUBECONFIG port-forward \
  -n serenamente-data \
  $SERENAMENTE_PG_POD 5432:5432 &
echo "PostgreSQL disponible en localhost:5432"

# Kratos Admin API (para inspeccionar identidades)
kubectl --kubeconfig $KUBECONFIG port-forward \
  -n serenamente-core \
  svc/ory-kratos-admin 4434:4434 &
echo "Kratos Admin API en localhost:4434"

# IAM Service directo (sin Traefik)
kubectl --kubeconfig $KUBECONFIG port-forward \
  -n serenamente-core \
  svc/iam-service 8080:8080 &
echo "IAM Service en localhost:8080"

# Mailpit UI
kubectl --kubeconfig $KUBECONFIG port-forward \
  -n serenamente-dev \
  svc/mailpit 8025:8025 &
echo "Mailpit UI en http://localhost:8025"

echo ""
echo "Todos los port-forwards activos. Ctrl+C para detener."
wait
```

### 21.3 Ver logs agregados de todos los servicios

```bash
# Ver logs de todos los servicios en serenamente-core simultáneamente:
# Instalar stern (log aggregator multi-pod):
brew install stern  # macOS

# Ver todos los logs del namespace:
stern --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  -n serenamente-core \
  --all-containers \
  --color always \
  ".*"

# Solo logs del IAM service:
stern --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  -n serenamente-core \
  iam-service
```

---

## 22. Troubleshooting Frecuente

### 22.1 El laptop Talos no responde en la red

```bash
# Verificar que el laptop está encendido y en la IP correcta:
ping 192.168.1.100

# Si no responde: verificar en tu router que la IP está asignada
# Si sí responde pero talosctl no:
talosctl version --nodes 192.168.1.100 --insecure
# Si falla: el nodo puede estar en maintenance mode (requiere apply-config de nuevo)
# O el firewall del router está bloqueando el puerto 50000
```

### 22.2 Pods en estado Pending

```bash
KUBECONFIG="infra/clusters/laptop-dev/kubeconfig"

# Ver por qué está pending:
kubectl --kubeconfig $KUBECONFIG describe pod <nombre-pod> -n <namespace>
# Buscar la sección "Events" al final

# Causas comunes:
# "0/1 nodes are available: 1 node(s) had untolerated taint"
# → El pod no tiene toleración para el control-plane
# → Añadir al spec del pod:
#   tolerations:
#   - key: "node-role.kubernetes.io/control-plane"
#     operator: "Exists"
#     effect: "NoSchedule"

# "PVC not bound" o "no persistent volumes available"
# → El local-path provisioner no está instalado
# Verificar:
kubectl --kubeconfig $KUBECONFIG get storageclass
# → debe mostrar "local-path" como DEFAULT
# Si no está, instalar:
kubectl --kubeconfig $KUBECONFIG apply -f \
  https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

### 22.3 Certificados TLS no válidos en el browser

```bash
# Verificar que el certificado fue emitido:
kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  get certificate -n serenamente-core

# Si el estado no es READY=True:
kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  describe certificate api-serenamente-local-tls -n serenamente-core
# Buscar la sección "Events" para ver el error de cert-manager

# Si el certificado está emitido pero el browser no confía:
# → No instalaste la CA raíz en tu sistema
# → Repetir Tarea L-06 sección 10.3
```

### 22.4 CloudNativePG no arranca

```bash
KUBECONFIG="infra/clusters/laptop-dev/kubeconfig"

# Ver estado del cluster:
kubectl --kubeconfig $KUBECONFIG \
  describe cluster serenamente-pg -n serenamente-data

# Ver logs del pod PostgreSQL:
kubectl --kubeconfig $KUBECONFIG \
  logs -n serenamente-data \
  -l cnpg.io/cluster=serenamente-pg \
  --all-containers

# Problema frecuente: "local-path" storage class no existe
kubectl --kubeconfig $KUBECONFIG get storageclass
# Si no aparece local-path, instalarlo:
kubectl --kubeconfig $KUBECONFIG apply -f \
  https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

### 22.5 Imágenes no se pull desde el registry local

```bash
# Verificar que el registry está corriendo:
curl http://192.168.1.100:5000/v2/
# → {"repositories":[...]}

# Verificar que Talos tiene la configuración del registry inseguro:
talosctl get registries \
  --talosconfig infra/clusters/laptop-dev/talos/talosconfig \
  --nodes 192.168.1.100

# Si no aparece el mirror, actualizar la config de Talos:
talosctl apply-config \
  --talosconfig infra/clusters/laptop-dev/talos/talosconfig \
  --nodes 192.168.1.100 \
  --file infra/clusters/laptop-dev/talos/controlplane.yaml
# (Asegurarse de que el patch laptop-dev.yaml tiene la sección registries)
```

### 22.6 El laptop Talos se queda sin memoria

```bash
# Ver uso de memoria actual:
talosctl memory \
  --talosconfig infra/clusters/laptop-dev/talos/talosconfig \
  --nodes 192.168.1.100

# Ver los pods que más consumen:
kubectl --kubeconfig infra/clusters/laptop-dev/kubeconfig \
  top pods -A --sort-by=memory

# Si supera el 85%:
# 1. Reducir replicas en desarrollo (de 2 a 1 en cada Deployment)
# 2. Reducir memory limits en los Deployments
# 3. Desinstalar componentes que no necesitas en este momento
# 4. Considerar upgrade del laptop a 16GB RAM
```

---

## 23. Criterios de Aceptación Global del Entorno Local

El entorno local está completamente configurado cuando todos estos checks pasan:

```bash
KUBECONFIG="infra/clusters/laptop-dev/kubeconfig"

echo "=== 1. CLUSTER KUBERNETES ==="
kubectl --kubeconfig $KUBECONFIG get nodes
# ✓ serenamente-dev-01 Ready

echo "=== 2. CERT-MANAGER ==="
kubectl --kubeconfig $KUBECONFIG get clusterissuer local-ca
# ✓ READY=True

echo "=== 3. TRAEFIK ==="
kubectl --kubeconfig $KUBECONFIG get pods -n traefik
# ✓ traefik-* Running

echo "=== 4. TLS LOCAL ==="
curl -s -o /dev/null -w "%{http_code}" https://api.serenamente.local/health
# ✓ 200 (o 404 — lo que importa es que TLS es válido, no error de certificado)

echo "=== 5. CLOUDNATIVEPG ==="
kubectl --kubeconfig $KUBECONFIG get cluster serenamente-pg -n serenamente-data
# ✓ STATUS=Cluster in healthy state

echo "=== 6. BASES DE DATOS ==="
kubectl --kubeconfig $KUBECONFIG exec \
  -n serenamente-data \
  $(kubectl --kubeconfig $KUBECONFIG get pod -n serenamente-data \
    -l cnpg.io/cluster=serenamente-pg,role=primary -o name) \
  -- psql -U postgres -c "SELECT datname FROM pg_database WHERE datname IN ('iam_db','scheduling_db','clinical_db','billing_db','kratos_db','openfga_db');"
# ✓ Las 6 databases listadas

echo "=== 7. SEALED SECRETS ==="
kubectl --kubeconfig $KUBECONFIG get pods -n sealed-secrets
# ✓ sealed-secrets-* Running

echo "=== 8. MAILPIT ==="
curl -s http://localhost:8025/api/v1/messages | jq .total
# ✓ Número (0 si no hay emails aún) — sin error de conexión

echo "=== 9. ORY KRATOS ==="
kubectl --kubeconfig $KUBECONFIG get pods -n serenamente-core -l app.kubernetes.io/name=kratos
# ✓ kratos-* Running

echo "=== 10. IAM SERVICE ==="
kubectl --kubeconfig $KUBECONFIG get pods -n serenamente-core -l app=iam-service
# ✓ iam-service-* Running

echo "=== 11. REGISTRY LOCAL ==="
curl -s http://registry.serenamente.local:5000/v2/ | jq .
# ✓ {} (registry vacío pero respondiendo)

echo "=== 12. FLUJO COMPLETO ==="
# Registro de un paciente test:
curl -X POST https://api.serenamente.local/auth/self-service/registration/api
# ✓ Retorna un flow de Kratos (JSON con action, ui, etc.)
# Verificar en Mailpit que el email de verificación llegó:
# https://mail.serenamente.local (o localhost:8025)
```

---

## Apéndice: Mapeo de diferencias entre ramas Git

Para mantener los entornos separados sin duplicar código, se recomienda usar **Kustomize overlays**:

```
infra/
├── base/                    ← Configuración base (igual en dev y prod)
│   ├── cert-manager/
│   ├── traefik/
│   ├── cnpg/
│   └── apps/
├── overlays/
│   ├── laptop-dev/          ← Parches específicos de dev
│   │   ├── kustomization.yaml
│   │   ├── traefik-service-patch.yaml    (ClusterIP en vez de LoadBalancer)
│   │   ├── clusterissuer-local-ca.yaml   (Self-signed en vez de ACME)
│   │   ├── cnpg-local-storage-patch.yaml (local-path en vez de hcloud-volumes)
│   │   ├── kratos-urls-patch.yaml        (*.local en vez de *.com)
│   │   └── image-registry-patch.yaml     (registry.serenamente.local en vez de ghcr.io)
│   └── hetzner-prod/        ← Parches específicos de producción
│       ├── kustomization.yaml
│       ├── traefik-loadbalancer.yaml
│       ├── clusterissuer-letsencrypt.yaml
│       └── cnpg-hcloud-storage.yaml
```

Con esta estructura, cada entorno aplica solo sus patches sobre la base común. Los manifests de los microservicios son idénticos; solo cambian las referencias a dominios, storage classes, y ClusterIssuers.

```yaml
# infra/overlays/laptop-dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base/cert-manager
  - ../../base/traefik
  - ../../base/cnpg
  - ../../base/apps/iam-service
patches:
  - path: traefik-service-patch.yaml
  - path: clusterissuer-local-ca.yaml
  - path: cnpg-local-storage-patch.yaml
  - path: kratos-urls-patch.yaml
  - path: image-registry-patch.yaml
```

---

*Documento v1.0 — Abril 2026. Para la migración del laptop al VPS de producción, ver [`plans/08_migracion_laptop_a_vps.md`](./08_migracion_laptop_a_vps.md).*
