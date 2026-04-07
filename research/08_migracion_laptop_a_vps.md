# Guía Exhaustiva de Migración: Laptop Talos → Hetzner CX32 VPS

## De Entorno de Desarrollo Local a Producción

**Proyecto:** Serenamente — Clínica Digital de Salud Mental Global
**Versión:** 1.0 — Abril 2026
**Autor:** djca
**Relación con otros documentos:**
- [`plans/07_fase1_laptop_talos_desarrollo.md`](./07_fase1_laptop_talos_desarrollo.md) — Entorno de desarrollo en laptop (origen)
- [`plans/05_orden_implementacion_capas_exhaustivo.md`](./05_orden_implementacion_capas_exhaustivo.md) — Plan Fase 1 para VPS (destino)
- [`plans/06_talos_k8s_decision_y_cambios_en_cadena.md`](./06_talos_k8s_decision_y_cambios_en_cadena.md) — Decisión arquitectónica Talos+k8s

---

## Índice

1. [¿Cuándo migrar?](#1-cuándo-migrar)
2. [Visión General del Proceso](#2-visión-general-del-proceso)
3. [Mapa de Diferencias: Laptop-Dev vs Hetzner-Prod](#3-mapa-de-diferencias-laptop-dev-vs-hetzner-prod)
4. [Fase de Preparación (antes de tocar el VPS)](#4-fase-de-preparación-antes-de-tocar-el-vps)
5. [M-01 — Aprovisionamiento del VPS Hetzner CX32](#5-m-01--aprovisionamiento-del-vps-hetzner-cx32)
6. [M-02 — Bootstrap de Talos Linux en el CX32](#6-m-02--bootstrap-de-talos-linux-en-el-cx32)
7. [M-03 — Sealed Secrets: Nueva Clave para Producción](#7-m-03--sealed-secrets-nueva-clave-para-producción)
8. [M-04 — Re-sealed de Todos los Secretos con Clave Prod](#8-m-04--re-sealed-de-todos-los-secretos-con-clave-prod)
9. [M-05 — FluxCD Bootstrap en Producción (rama main)](#9-m-05--fluxcd-bootstrap-en-producción-rama-main)
10. [M-06 — Overlay Hetzner-Prod: Kustomize](#10-m-06--overlay-hetzner-prod-kustomize)
11. [M-07 — cert-manager con Let's Encrypt en Producción](#11-m-07--cert-manager-con-lets-encrypt-en-producción)
12. [M-08 — Traefik: de hostPort a LoadBalancer](#12-m-08--traefik-de-hostport-a-loadbalancer)
13. [M-09 — CloudNativePG con hcloud-volumes y Backblaze B2](#13-m-09--cloudnativepg-con-hcloud-volumes-y-backblaze-b2)
14. [M-10 — Ory Kratos: dominios reales y SendGrid](#14-m-10--ory-kratos-dominios-reales-y-sendgrid)
15. [M-11 — IAM Domain Service: imagen desde GHCR](#15-m-11--iam-domain-service-imagen-desde-ghcr)
16. [M-12 — DNS en Cloudflare: cutover](#16-m-12--dns-en-cloudflare-cutover)
17. [M-13 — Verificación y Smoke Testing Post-Migración](#17-m-13--verificación-y-smoke-testing-post-migración)
18. [Plan de Rollback](#18-plan-de-rollback)
19. [Post-Migración: CI/CD con GitHub Actions](#19-post-migración-cicd-con-github-actions)
20. [Descomisionado del Entorno Laptop (Opcional)](#20-descomisionado-del-entorno-laptop-opcional)
21. [Criterios de Aceptación Global de la Migración](#21-criterios-de-aceptación-global-de-la-migración)
22. [Checklist de Migración (Resumen Operativo)](#22-checklist-de-migración-resumen-operativo)

---

## 1. ¿Cuándo migrar?

La migración del entorno de desarrollo (laptop Talos) al VPS Hetzner es un evento deliberado, no urgente. No hay presión de hacerlo en ningún momento específico. Las señales que indican que es el momento adecuado son:

### 1.1 Señales técnicas de madurez

1. **Los 23 criterios de aceptación del `plans/07`** están todos cumplidos (cluster local funcional, TLS self-signed operativo, IAM completo, CloudNativePG con backups a MinIO).
2. **Al menos una funcionalidad de usuario está completa end-to-end**: registro → verificación de email → login → sesión → logout funciona en `*.serenamente.local` sin errores.
3. **Los manifests en `infra/overlays/hetzner-prod/`** están escritos y revisados (el overlay de producción existe aunque nunca se haya aplicado).
4. **El pipeline de CI en GitHub Actions** tiene al menos el job de `go test ./...` pasando en la rama `main`.
5. **Los secretos de producción** (SendGrid, Backblaze B2, GitHub PAT para FluxCD) están disponibles.

### 1.2 Señales de negocio / operativas

1. Se quiere mostrar el producto a usuarios reales (testers, inversores, primeros clientes).
2. El dominio `serenamente.com` está registrado y los nameservers apuntan a Cloudflare.
3. El equipo tiene disponibilidad para mantener el servicio (no irse de vacaciones justo después del deploy).

### 1.3 Prerrequisitos irrenunciables antes de empezar la migración

| Prerrequisito | Dónde verificar |
|---------------|-----------------|
| Cuenta Hetzner con billing activo | hetzner.com/cloud |
| Cuenta Cloudflare con el dominio gestionado | cloudflare.com dashboard |
| Token SendGrid con permiso Mail Send | sendgrid.com → API Keys |
| Bucket B2 creado en Backblaze + App Key con acceso al bucket | backblaze.com → Buckets |
| Personal Access Token GitHub con scope `repo` (para FluxCD) | github.com → Settings → Developer Settings |
| `kubeseal` CLI instalado en la máquina de gestión | `kubeseal --version` |
| `talosctl` CLI v1.10.x instalado | `talosctl version --client` |
| `flux` CLI v2.x instalado | `flux version --client` |
| Acceso de escritura al repositorio `serenamente-infra` (o monorepo) | `git push origin main` funciona |

---

## 2. Visión General del Proceso

La migración es un proceso de aproximadamente **4-8 horas** si todos los prerrequisitos están cumplidos. Es fundamentalmente diferente a una "migración de datos" tradicional: como el sistema está en fase de desarrollo sin usuarios reales, **no hay datos de producción que migrar**. La migración consiste en:

1. Aprovisionar la infraestructura cloud (VPS + Floating IP)
2. Instalar Talos Linux en el VPS (diferente a laptop: sin monitor, vía Hetzner rescue)
3. Generar un nuevo par de claves de Sealed Secrets para producción
4. Re-sellar todos los secretos con la clave de producción
5. Hacer FluxCD bootstrap apuntando a la rama `main`
6. FluxCD aplica automáticamente el overlay `hetzner-prod` (que ya existe en el repo)
7. cert-manager obtiene certificados Let's Encrypt reales
8. Cloudflare DNS cutover
9. Smoke testing

```
ESTADO ORIGEN (laptop-dev)          ESTADO DESTINO (hetzner-prod)
─────────────────────────────        ──────────────────────────────
Talos Linux en laptop físico  ──►   Talos Linux en Hetzner CX32
IP local: 192.168.1.100       ──►   IP pública Floating: X.X.X.X
*.serenamente.local           ──►   *.serenamente.com
ClusterIssuer: local-ca       ──►   ClusterIssuer: letsencrypt-prod
hostPort :80/:443             ──►   Service LoadBalancer :80/:443
local-path StorageClass       ──►   hcloud-volumes StorageClass
MinIO (B2 sustituto)          ──►   Backblaze B2 real
Mailpit (SMTP dev)            ──►   SendGrid (SMTP real)
registry.serenamente.local    ──►   ghcr.io/serenamente
FluxCD → rama dev             ──►   FluxCD → rama main
SealedSecret clave dev        ──►   SealedSecret clave prod (nueva)
```

**Principio guía:** El 95% de los manifests son idénticos entre dev y prod. La migración es un cambio de overlay de Kustomize, no una reescritura.

---

## 3. Mapa de Diferencias: Laptop-Dev vs Hetzner-Prod

Esta tabla es la referencia central de la migración. Cada fila representa un cambio específico que debe ocurrir.

| # | Componente | Laptop-Dev | Hetzner-Prod | Cómo se implementa |
|---|-----------|-----------|-------------|-------------------|
| D-01 | SO del nodo | Talos en laptop físico | Talos en Hetzner CX32 | Bootstrap vía Hetzner rescue + talosctl |
| D-02 | IP del nodo | 192.168.1.100 (LAN privada) | Floating IP Hetzner (pública) | Provisionar Floating IP en panel Hetzner |
| D-03 | IP estática en Talos | Patch local `machine.network` | Patch prod con IP pública Hetzner | Patch YAML diferente en overlay |
| D-04 | Schematic Talos | ISO base (sin extensiones) | Schematic Hetzner con `qemu-guest-agent` | factory.talos.dev → generar schematic |
| D-05 | CCM (Cloud Controller Manager) | No existe | hcloud-ccm Helm Chart | HelmRelease en overlay hetzner-prod |
| D-06 | StorageClass | `local-path` (Rancher) | `hcloud-volumes` (Hetzner CSI Driver) | Patch StorageClass en CloudNativePG CRD |
| D-07 | ClusterIssuer TLS | `local-ca` (self-signed, 10 años) | `letsencrypt-prod` (Let's Encrypt ACME) | ClusterIssuer diferente en overlay |
| D-08 | Traefik Service | DaemonSet con hostPort :80/:443 | Deployment con Service `LoadBalancer` | Patch Traefik values en overlay |
| D-09 | DNS | `/etc/hosts` manual en máquina dev | Cloudflare → registros A apuntando a Floating IP | Cutover DNS en Cloudflare dashboard |
| D-10 | Dominios | `*.serenamente.local` | `*.serenamente.com` | Patch de URLs en todos los servicios |
| D-11 | SMTP | Mailpit (interceptor local) | SendGrid API SMTP (`smtp.sendgrid.net:587`) | SealedSecret con credenciales SendGrid |
| D-12 | S3/Backup | MinIO pod local | Backblaze B2 bucket real | SealedSecret con B2 App Key |
| D-13 | Registry de imágenes | `registry.serenamente.local:5000` (local) | `ghcr.io/serenamente` (GitHub Container Registry) | Patch image refs en Deployments |
| D-14 | Sealed Secrets clave | Clave de desarrollo (generada en laptop bootstrap) | Clave de producción (nueva, generada en primer boot) | `kubeseal` con nueva clave |
| D-15 | FluxCD rama | `dev` | `main` | `--branch=main` en flux bootstrap |
| D-16 | Mailpit | Presente en namespace `serenamente-dev` | Ausente (no se deploya en prod) | Kustomize: no incluir en overlay prod |
| D-17 | MinIO pod | Presente en namespace `serenamente-data` | Ausente (B2 externo) | Kustomize: no incluir en overlay prod |
| D-18 | Registry local | Presente en namespace `serenamente-dev` | Ausente (GHCR externo) | Kustomize: no incluir en overlay prod |
| D-19 | Kratos config | `password: enabled: true`, verbose logging | `password: enabled: false` (solo OIDC/passcode), json logging | ConfigMap diferente en overlay prod |
| D-20 | `allowSchedulingOnControlPlanes` | `true` (nodo único) | `false` (opcional, pero mejor dejarlo `false` para CX32) | Patch en talosconfig |
| D-21 | Hetzner Cloud Token | N/A | Necesario para CCM + CSI Driver | SealedSecret con HCLOUD_TOKEN |

---

## 4. Fase de Preparación (antes de tocar el VPS)

Esta fase se ejecuta completamente en la máquina de desarrollo, sin afectar el entorno laptop existente ni el VPS (que no existe aún). Es reversible en su totalidad.

### 4.1 Verificar el estado del repositorio

```bash
# Estado del repo: debe estar limpio y en main
git checkout main
git pull origin main
git status  # debe mostrar "nothing to commit"

# Verificar que el overlay hetzner-prod existe
ls infra/overlays/hetzner-prod/
# Debe contener: kustomization.yaml, patches/ directory

# Verificar que los manifests base compilan correctamente
kubectl kustomize infra/overlays/hetzner-prod/ --dry-run
```

Si el overlay `hetzner-prod` no existe aún, la sección 10 (M-06) detalla cómo crearlo. Es mejor tenerlo listo antes de provisionar el VPS.

### 4.2 Inventario de secretos a re-sellar

Listar todos los `SealedSecret` en el repositorio:

```bash
find infra/ -name "*.yaml" -exec grep -l "kind: SealedSecret" {} \;
```

Resultado esperado (mínimo para Fase 1):

```
infra/base/sealed-secrets/postgres-credentials.yaml
infra/base/sealed-secrets/kratos-smtp.yaml
infra/base/sealed-secrets/kratos-secrets.yaml
infra/base/sealed-secrets/iam-db-credentials.yaml
infra/overlays/hetzner-prod/sealed-secrets/b2-backup-credentials.yaml
infra/overlays/hetzner-prod/sealed-secrets/sendgrid-credentials.yaml
infra/overlays/hetzner-prod/sealed-secrets/hcloud-token.yaml
```

Anotar cada secreto, su namespace de destino y las claves que contiene. Esta lista se usará en M-04.

### 4.3 Recopilar credenciales de producción

Antes de crear secretos de producción, tener todos los valores listos:

```bash
# Credenciales a recopilar:
# 1. PostgreSQL passwords (generar nuevas para prod)
POSTGRES_PASSWORD=$(openssl rand -base64 32)
KRATOS_DB_PASSWORD=$(openssl rand -base64 32)
IAM_DB_PASSWORD=$(openssl rand -base64 32)

# 2. Kratos secrets (generar nuevos para prod)
KRATOS_COOKIE_SECRET=$(openssl rand -hex 32)
KRATOS_CIPHER_SECRET=$(openssl rand -hex 32)

# 3. SendGrid (desde el panel de SendGrid)
SENDGRID_SMTP_PASSWORD="SG.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
SENDGRID_SMTP_USER="apikey"

# 4. Backblaze B2 (desde el panel de Backblaze)
B2_KEY_ID="xxxxxxxxxxxxxxxxxxxxxxxx"
B2_APPLICATION_KEY="xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
B2_BUCKET_NAME="serenamente-prod-backups"
B2_BUCKET_REGION="us-west-004"  # región donde creaste el bucket

# 5. Hetzner API Token (desde el panel de Hetzner, con permiso Read+Write)
HCLOUD_TOKEN="xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

# Guardar en archivo temporal FUERA del repo git
cat > /tmp/prod-secrets.env << EOF
POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
KRATOS_DB_PASSWORD=${KRATOS_DB_PASSWORD}
IAM_DB_PASSWORD=${IAM_DB_PASSWORD}
KRATOS_COOKIE_SECRET=${KRATOS_COOKIE_SECRET}
KRATOS_CIPHER_SECRET=${KRATOS_CIPHER_SECRET}
SENDGRID_SMTP_PASSWORD=${SENDGRID_SMTP_PASSWORD}
SENDGRID_SMTP_USER=${SENDGRID_SMTP_USER}
B2_KEY_ID=${B2_KEY_ID}
B2_APPLICATION_KEY=${B2_APPLICATION_KEY}
B2_BUCKET_NAME=${B2_BUCKET_NAME}
B2_BUCKET_REGION=${B2_BUCKET_REGION}
HCLOUD_TOKEN=${HCLOUD_TOKEN}
EOF
chmod 600 /tmp/prod-secrets.env
```

**Advertencia de seguridad:** `/tmp/prod-secrets.env` es un archivo temporal. Eliminarlo al final de la migración con `shred -vzu /tmp/prod-secrets.env`. Nunca commitearlo al repositorio.

### 4.4 Preparar el overlay hetzner-prod (si no existe)

Ver sección M-06 para la creación completa del overlay. Crearlo antes de provisionar el VPS permite validarlo con `kubectl kustomize --dry-run`.

### 4.5 Verificar resolución DNS pública del dominio

```bash
# Verificar que serenamente.com está en Cloudflare
curl -s "https://api.cloudflare.com/client/v4/zones?name=serenamente.com" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: application/json" | jq '.result[].id'

# Guardar el Zone ID para usarlo más tarde
CF_ZONE_ID="xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

---

## 5. M-01 — Aprovisionamiento del VPS Hetzner CX32

### 5.1 Crear el servidor en Hetzner

La instalación de Talos en Hetzner es diferente a un laptop físico. Hetzner no tiene pantalla ni BIOS accesible directamente — se usa el modo rescue y luego se instala el OS vía `talosctl`.

**Alternativa A — vía Hetzner Cloud Console (interfaz web):**

1. Ir a `console.hetzner.cloud` → Create Server
2. Location: Falkenstein (FSN1) o Nuremberg (NBG1) — elegir el más cercano geográficamente
3. OS Image: seleccionar **"Ubuntu 24.04"** (temporal, solo para el boot inicial; Talos lo reemplazará)
4. Type: **CX32** (4 vCPU, 8 GB RAM, 80 GB NVMe) → ~$7.50/mes
5. SSH Keys: agregar tu clave pública (necesaria solo para el rescue inicial)
6. Name: `serenamente-prod-01`
7. Click "Create & Buy"

**Alternativa B — vía Hetzner CLI (`hcloud`):**

```bash
# Instalar hcloud CLI
# macOS: brew install hcloud
# Linux: descarga desde https://github.com/hetznercloud/cli/releases

# Autenticar
hcloud context create serenamente-prod
# Pega el HCLOUD_TOKEN cuando lo pida

# Crear servidor
hcloud server create \
  --name serenamente-prod-01 \
  --type cx32 \
  --image ubuntu-24.04 \
  --location fsn1 \
  --ssh-key "$(cat ~/.ssh/id_ed25519.pub)" \
  --label project=serenamente \
  --label env=prod

# Ver la IP asignada
hcloud server describe serenamente-prod-01 | grep "Public Net" -A5
```

### 5.2 Crear y asignar Floating IP

Una Floating IP es esencial en producción. Permite:
- Reasignar la IP a otro servidor sin cambiar DNS (zero-downtime server replacement)
- Mantener la misma IP si el servidor se recrea (Talos re-install, por ejemplo)
- Desacoplar el ciclo de vida del servidor de la IP pública

```bash
# Crear Floating IP en la misma región que el servidor
hcloud floating-ip create \
  --type ipv4 \
  --home-location fsn1 \
  --name serenamente-prod-ip \
  --label project=serenamente

# Anotar la IP asignada
FLOATING_IP=$(hcloud floating-ip describe serenamente-prod-ip -o json | jq -r '.ip')
echo "Floating IP: ${FLOATING_IP}"

# Asignar la Floating IP al servidor
hcloud floating-ip assign serenamente-prod-ip serenamente-prod-01
```

**Nota:** La Floating IP tiene un costo adicional de ~$0.43/mes cuando está asignada. Cuando no está asignada a ningún servidor, también se cobra (para que no la liberes sin querer).

### 5.3 Verificar conectividad

```bash
# IP del servidor (puede ser diferente a la Floating IP)
SERVER_IP=$(hcloud server describe serenamente-prod-01 -o json | jq -r '.public_net.ipv4.ip')

# Verificar SSH (con la imagen Ubuntu temporal)
ssh root@${SERVER_IP} "uname -a"
# Debe mostrar Ubuntu 24.04

# La Floating IP no responde aún (no configurada en la interfaz de red del OS)
```

---

## 6. M-02 — Bootstrap de Talos Linux en el CX32

### 6.1 Generar el schematic Hetzner

A diferencia del laptop, el CX32 es una VM de Hetzner y requiere extensiones específicas para funcionar correctamente: `qemu-guest-agent` (para que Hetzner pueda gestionar el servidor) y la capa de virtio para el almacenamiento.

**Generar schematic en factory.talos.dev:**

```yaml
# schematic-hetzner.yaml
customization:
  systemExtensions:
    officialExtensions:
      - siderolabs/qemu-guest-agent
```

```bash
# Subir el schematic a Talos Image Factory
curl -s -X POST \
  --data-binary @schematic-hetzner.yaml \
  https://factory.talos.dev/schematics | jq -r '.id'

# Resultado: un ID de schematic como "376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4b"
SCHEMATIC_ID="376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4b"

# URL del disco de instalación (raw disk image para Hetzner):
TALOS_VERSION="v1.10.0"
DISK_IMAGE_URL="https://factory.talos.dev/image/${SCHEMATIC_ID}/${TALOS_VERSION}/hcloud-amd64.raw.xz"
```

### 6.2 Instalar Talos via Rescue Mode de Hetzner

Hetzner permite arrancar en modo rescue (un Linux minimal) para instalar el OS:

```bash
# 1. Activar rescue mode desde la CLI de Hetzner
hcloud server enable-rescue serenamente-prod-01 --type linux64 --ssh-key "$(cat ~/.ssh/id_ed25519.pub)"

# 2. Reiniciar el servidor
hcloud server reboot serenamente-prod-01

# 3. Esperar ~30 segundos y SSH al rescue system
ssh root@${SERVER_IP}

# 4. En el servidor rescue: descargar e instalar la imagen de Talos
# (estos comandos se ejecutan DENTRO del servidor Hetzner, no en tu máquina local)
wget -q "https://factory.talos.dev/image/${SCHEMATIC_ID}/v1.10.0/hcloud-amd64.raw.xz" -O /tmp/talos.raw.xz
xz -d /tmp/talos.raw.xz
dd if=/tmp/talos.raw of=/dev/sda bs=4M status=progress oflag=sync
```

**Nota:** En CX32 el disco NVMe puede aparecer como `/dev/sda` o `/dev/nvme0n1` — verificar con `lsblk` en el rescue system.

```bash
# Verificar el nombre del dispositivo de disco
lsblk
# Usar el dispositivo correcto (disco más grande, generalmente /dev/sda en VMs Hetzner)

# Una vez completado el dd, apagar el servidor
poweroff
```

### 6.3 Generar configuración de Talos para producción

De vuelta en tu **máquina de desarrollo local** (no en el servidor):

```bash
# Crear directorio para la configuración de producción
mkdir -p talos/prod
cd talos/prod

# Generar configuración base
talosctl gen config serenamente-prod https://${FLOATING_IP}:6443 \
  --output-dir . \
  --with-secrets secrets.yaml  # genera un archivo de secrets que se debe guardar de forma segura

# Esto crea:
#   controlplane.yaml  — config del nodo controlplane
#   worker.yaml        — config de workers (no usaremos aún)
#   talosconfig        — client config para talosctl
#   secrets.yaml       — CA y otros secretos del cluster (¡GUARDAR CON CUIDADO!)
```

**Advertencia crítica:** `secrets.yaml` contiene el CA del cluster Talos. Si se pierde, no se puede regenerar la configuración para el mismo cluster. Guardar en un gestor de contraseñas (1Password, Bitwarden) o en un lugar seguro fuera del repositorio.

### 6.4 Crear el patch de Talos para producción

```yaml
# talos/prod/patch-prod.yaml
machine:
  network:
    hostname: serenamente-prod-01
    interfaces:
      # Interfaz principal del servidor (eth0 en Hetzner)
      - interface: eth0
        dhcp: true  # La IP del servidor la gestiona Hetzner DHCP
        # La Floating IP se configura como alias en eth0
        addresses:
          - "${FLOATING_IP}/32"  # Floating IP como /32 (alias)
        routes:
          # Ruta específica para la Floating IP (gateway de Hetzner)
          - network: "0.0.0.0/0"
            gateway: "172.31.1.1"  # Gateway de Hetzner para Floating IPs
  install:
    disk: /dev/sda
    wipe: true
  time:
    servers:
      - "ntp1.hetzner.de"
      - "ntp2.hetzner.de"
  # NO se configura registries.mirrors aquí (usaremos GHCR, no registry local)

cluster:
  allowSchedulingOnControlPlanes: true  # CX32 es nodo único
  network:
    cni:
      name: flannel
  # El Hetzner CCM necesita el provider ID del servidor
  externalCloudProvider:
    enabled: true
    manifests:
      - https://raw.githubusercontent.com/hetznercloud/hcloud-cloud-controller-manager/main/deploy/ccm.yaml
```

**Nota sobre la Floating IP en Hetzner:** La interfaz principal (`eth0`) recibe la IP del servidor via DHCP de Hetzner. La Floating IP se debe agregar como un alias de red adicional con `/32` y apuntar al gateway `172.31.1.1`. Esta es la forma estándar de Hetzner.

### 6.5 Aplicar configuración Talos al servidor

```bash
# Arrancar el servidor (saliendo del rescue mode hacia Talos)
hcloud server poweron serenamente-prod-01

# Esperar ~60 segundos a que Talos arranque en modo "awaiting configuration"
sleep 60

# Verificar que el servidor está esperando configuración
talosctl --talosconfig talos/prod/talosconfig \
  get machinestatus \
  --nodes ${SERVER_IP}
# Estado esperado: "Initializing" o similar

# Aplicar la configuración (merge del controlplane base + patch prod)
talosctl apply-config \
  --talosconfig talos/prod/talosconfig \
  --nodes ${SERVER_IP} \
  --file talos/prod/controlplane.yaml \
  --config-patch @talos/prod/patch-prod.yaml \
  --insecure  # necesario en el primer boot antes de que TLS esté configurado

# Esperar a que el servidor reinicie con la nueva configuración (~2-3 min)
watch talosctl \
  --talosconfig talos/prod/talosconfig \
  --nodes ${SERVER_IP} \
  get machinestatus
```

### 6.6 Bootstrap del cluster etcd

```bash
# Una vez que el nodo está en estado "Ready" (sin etcd aún)
talosctl bootstrap \
  --talosconfig talos/prod/talosconfig \
  --nodes ${SERVER_IP}

# Este comando inicializa etcd y arranca el control plane de Kubernetes
# Solo se ejecuta UNA VEZ en la vida del cluster

# Esperar a que Kubernetes esté disponible (~3-5 minutos)
watch talosctl \
  --talosconfig talos/prod/talosconfig \
  --nodes ${SERVER_IP} \
  get member
# Debe mostrar el nodo con status "joined"
```

### 6.7 Obtener el kubeconfig de producción

```bash
# Obtener kubeconfig apuntando a la Floating IP
talosctl kubeconfig \
  --talosconfig talos/prod/talosconfig \
  --nodes ${SERVER_IP} \
  --endpoints ${FLOATING_IP} \
  talos/prod/kubeconfig

# Verificar acceso al cluster de producción
KUBECONFIG=talos/prod/kubeconfig kubectl get nodes
# Debe mostrar: serenamente-prod-01   Ready   control-plane   Xm   v1.33.x

# Verificar que todos los pods del system están Running
KUBECONFIG=talos/prod/kubeconfig kubectl get pods -A
```

**Importante:** Tener DOS kubeconfigs separados:
- `talos/dev/kubeconfig` → apunta al laptop (cluster de desarrollo)
- `talos/prod/kubeconfig` → apunta al CX32 (cluster de producción)

Nunca mezclarlos. Usar variables de entorno `KUBECONFIG` explícitas o contextos separados:

```bash
# Agregar al .bashrc / .zshrc
alias k-dev="KUBECONFIG=${HOME}/talos/dev/kubeconfig kubectl"
alias k-prod="KUBECONFIG=${HOME}/talos/prod/kubeconfig kubectl"
alias flux-dev="KUBECONFIG=${HOME}/talos/dev/kubeconfig flux"
alias flux-prod="KUBECONFIG=${HOME}/talos/prod/kubeconfig flux"
```

---

## 7. M-03 — Sealed Secrets: Nueva Clave para Producción

Este es uno de los pasos más críticos de la migración. Sealed Secrets usa criptografía asimétrica: el operador genera un par de claves RSA dentro del cluster. Los secretos se cifran con la clave pública y solo se pueden descifrar dentro del mismo cluster con la clave privada correspondiente.

**El cluster de producción tiene una clave diferente al cluster de desarrollo.** Los `SealedSecret` creados para el laptop no funcionarán en producción, y viceversa.

### 7.1 Instalar Sealed Secrets en el cluster de producción

```bash
# Instalar el controlador de Sealed Secrets en producción
KUBECONFIG=talos/prod/kubeconfig helm upgrade --install sealed-secrets \
  sealed-secrets/sealed-secrets \
  --namespace sealed-secrets \
  --create-namespace \
  --version ">=2.0.0" \
  --wait

# Verificar que el controlador está Running
KUBECONFIG=talos/prod/kubeconfig kubectl -n sealed-secrets get pods
```

### 7.2 Obtener la clave pública de producción

```bash
# Obtener el certificado público del cluster de producción
# Esta clave se usa para cifrar secretos que solo el cluster prod puede descifrar
kubeseal --fetch-cert \
  --controller-name=sealed-secrets \
  --controller-namespace=sealed-secrets \
  --kubeconfig talos/prod/kubeconfig \
  > pub-keys/serenamente-prod.pem

# Verificar la clave
openssl x509 -in pub-keys/serenamente-prod.pem -noout -text | head -30

# Guardar la clave pública en el repositorio (es seguro, es pública)
git add pub-keys/serenamente-prod.pem
git commit -m "chore: add prod Sealed Secrets public key"
git push origin main
```

La clave pública se puede commitear al repositorio sin ningún problema de seguridad — es pública por diseño.

---

## 8. M-04 — Re-sealed de Todos los Secretos con Clave Prod

Para cada secreto de la lista creada en la sección 4.2, es necesario crear una nueva versión sellada con la clave pública de producción.

### 8.1 Proceso por secreto

El proceso es el mismo para cada secreto:

```bash
# Cargar las credenciales de producción
source /tmp/prod-secrets.env

# Plantilla del proceso:
kubectl create secret generic NOMBRE-DEL-SECRETO \
  --namespace=NAMESPACE-DESTINO \
  --from-literal=CLAVE_1=VALOR_1 \
  --from-literal=CLAVE_2=VALOR_2 \
  --dry-run=client \
  -o yaml | \
kubeseal \
  --cert pub-keys/serenamente-prod.pem \
  --format yaml \
  > infra/overlays/hetzner-prod/sealed-secrets/NOMBRE-DEL-SECRETO.yaml
```

### 8.2 Crear cada secreto de producción

**Secreto 1: Credenciales de PostgreSQL (superuser)**

```bash
kubectl create secret generic postgres-superuser \
  --namespace=serenamente-data \
  --from-literal=username=postgres \
  --from-literal=password="${POSTGRES_PASSWORD}" \
  --dry-run=client -o yaml | \
kubeseal \
  --cert pub-keys/serenamente-prod.pem \
  --format yaml \
  > infra/overlays/hetzner-prod/sealed-secrets/postgres-superuser.yaml
```

**Secreto 2: Credenciales de base de datos para Kratos**

```bash
kubectl create secret generic kratos-db-credentials \
  --namespace=serenamente-core \
  --from-literal=username=kratos_user \
  --from-literal=password="${KRATOS_DB_PASSWORD}" \
  --from-literal=uri="postgres://kratos_user:${KRATOS_DB_PASSWORD}@serenamente-db-rw.serenamente-data.svc.cluster.local:5432/kratos_db?sslmode=require" \
  --dry-run=client -o yaml | \
kubeseal \
  --cert pub-keys/serenamente-prod.pem \
  --format yaml \
  > infra/overlays/hetzner-prod/sealed-secrets/kratos-db-credentials.yaml
```

**Secreto 3: Credenciales de base de datos para IAM Service**

```bash
kubectl create secret generic iam-db-credentials \
  --namespace=serenamente-core \
  --from-literal=username=iam_user \
  --from-literal=password="${IAM_DB_PASSWORD}" \
  --from-literal=dsn="postgres://iam_user:${IAM_DB_PASSWORD}@serenamente-db-rw.serenamente-data.svc.cluster.local:5432/iam_db?sslmode=require" \
  --dry-run=client -o yaml | \
kubeseal \
  --cert pub-keys/serenamente-prod.pem \
  --format yaml \
  > infra/overlays/hetzner-prod/sealed-secrets/iam-db-credentials.yaml
```

**Secreto 4: Secrets internos de Kratos (cookie y cipher)**

```bash
kubectl create secret generic kratos-internal-secrets \
  --namespace=serenamente-core \
  --from-literal=cookie-secret="${KRATOS_COOKIE_SECRET}" \
  --from-literal=cipher-secret="${KRATOS_CIPHER_SECRET}" \
  --dry-run=client -o yaml | \
kubeseal \
  --cert pub-keys/serenamente-prod.pem \
  --format yaml \
  > infra/overlays/hetzner-prod/sealed-secrets/kratos-internal-secrets.yaml
```

**Secreto 5: SendGrid SMTP**

```bash
kubectl create secret generic sendgrid-smtp \
  --namespace=serenamente-core \
  --from-literal=smtp-uri="smtps://apikey:${SENDGRID_SMTP_PASSWORD}@smtp.sendgrid.net:465/" \
  --dry-run=client -o yaml | \
kubeseal \
  --cert pub-keys/serenamente-prod.pem \
  --format yaml \
  > infra/overlays/hetzner-prod/sealed-secrets/sendgrid-smtp.yaml
```

**Secreto 6: Backblaze B2 para CloudNativePG barman-cloud**

```bash
kubectl create secret generic b2-backup-credentials \
  --namespace=serenamente-data \
  --from-literal=ACCESS_KEY_ID="${B2_KEY_ID}" \
  --from-literal=ACCESS_SECRET_KEY="${B2_APPLICATION_KEY}" \
  --dry-run=client -o yaml | \
kubeseal \
  --cert pub-keys/serenamente-prod.pem \
  --format yaml \
  > infra/overlays/hetzner-prod/sealed-secrets/b2-backup-credentials.yaml
```

**Secreto 7: Hetzner Cloud Token (para CCM + CSI Driver)**

```bash
kubectl create secret generic hcloud-credentials \
  --namespace=kube-system \
  --from-literal=token="${HCLOUD_TOKEN}" \
  --dry-run=client -o yaml | \
kubeseal \
  --cert pub-keys/serenamente-prod.pem \
  --format yaml \
  > infra/overlays/hetzner-prod/sealed-secrets/hcloud-credentials.yaml
```

### 8.3 Commitear todos los secretos sellados

```bash
# Verificar que todos los archivos de secretos están creados
ls infra/overlays/hetzner-prod/sealed-secrets/
# postgres-superuser.yaml
# kratos-db-credentials.yaml
# iam-db-credentials.yaml
# kratos-internal-secrets.yaml
# sendgrid-smtp.yaml
# b2-backup-credentials.yaml
# hcloud-credentials.yaml

# Commitear al repositorio (son seguros — están cifrados)
git add infra/overlays/hetzner-prod/sealed-secrets/
git commit -m "feat(prod): add sealed secrets for production cluster"
git push origin main

# Eliminar las credenciales en claro del sistema local
shred -vzu /tmp/prod-secrets.env
```

---

## 9. M-05 — FluxCD Bootstrap en Producción (rama main)

### 9.1 Prerrequisitos para FluxCD bootstrap

```bash
# Verificar que flux CLI puede acceder al cluster de producción
KUBECONFIG=talos/prod/kubeconfig flux check --pre

# Verificar que GitHub PAT tiene los permisos correctos
# El PAT necesita: repo (full), read:packages, write:packages
# Verificar: el PAT permite push a la rama main del repo de infra
```

### 9.2 Ejecutar bootstrap de FluxCD

```bash
# Variables del repositorio (ajustar según la estructura del monorepo)
GITHUB_USER="serenamente"          # o el org/user del repo
GITHUB_REPO="serenamente-infra"    # o el nombre del monorepo
GITHUB_BRANCH="main"               # rama de producción

# Bootstrap FluxCD en el cluster de producción
KUBECONFIG=talos/prod/kubeconfig flux bootstrap github \
  --owner="${GITHUB_USER}" \
  --repository="${GITHUB_REPO}" \
  --branch="${GITHUB_BRANCH}" \
  --path="infra/overlays/hetzner-prod" \
  --personal \
  --components-extra=image-reflector-controller,image-automation-controller \
  --token-auth

# El proceso:
# 1. FluxCD instala sus componentes en el namespace flux-system
# 2. Crea un Deployment Key en el repositorio GitHub (con acceso de lectura)
# 3. Crea una GitRepository CR apuntando al repo y rama configurados
# 4. Crea una Kustomization CR apuntando a infra/overlays/hetzner-prod
# 5. Flux empieza a reconciliar: aplica todos los manifests del overlay prod

# Verificar el progreso del bootstrap
KUBECONFIG=talos/prod/kubeconfig flux get all -A

# Esperar a que todos los recursos estén "Ready"
KUBECONFIG=talos/prod/kubeconfig watch flux get kustomizations -A
```

### 9.3 Diferencia clave con el cluster de desarrollo

En el cluster de desarrollo (laptop), FluxCD apuntaba a:
- Repositorio: el mismo
- Rama: `dev`
- Path: `infra/overlays/laptop-dev`

En producción, FluxCD apunta a:
- Repositorio: el mismo
- Rama: `main`
- Path: `infra/overlays/hetzner-prod`

Esta es la separación de entornos mediante GitOps: la rama `dev` es el entorno de desarrollo; la rama `main` es producción. Un PR de `dev` → `main` es un "deployment" a producción.

---

## 10. M-06 — Overlay Hetzner-Prod: Kustomize

Si el overlay `hetzner-prod` no está creado aún, esta sección documenta su estructura completa.

### 10.1 Estructura del overlay

```
infra/
├── base/
│   ├── kustomization.yaml         # Todos los recursos base
│   ├── namespaces.yaml
│   ├── cert-manager/
│   │   ├── helmrelease.yaml
│   │   └── clusterissuers.yaml    # Ambos issuers (selfsigned + CA)
│   ├── traefik/
│   │   ├── helmrelease.yaml
│   │   └── middlewares.yaml
│   ├── sealed-secrets/
│   │   └── helmrelease.yaml
│   ├── cloudnativepg/
│   │   ├── helmrelease.yaml
│   │   └── cluster.yaml
│   └── ory-kratos/
│       ├── helmrelease.yaml
│       └── configmap.yaml
│
├── overlays/
│   ├── laptop-dev/
│   │   ├── kustomization.yaml
│   │   ├── patches/
│   │   │   ├── traefik-hostport.yaml
│   │   │   ├── cnpg-local-path.yaml
│   │   │   └── kratos-dev-config.yaml
│   │   └── sealed-secrets/
│   │       └── *.yaml (dev secrets)
│   │
│   └── hetzner-prod/
│       ├── kustomization.yaml
│       ├── patches/
│       │   ├── traefik-loadbalancer.yaml
│       │   ├── cnpg-hcloud-volumes.yaml
│       │   ├── kratos-prod-config.yaml
│       │   └── image-registry-ghcr.yaml
│       ├── sealed-secrets/
│       │   └── *.yaml (prod secrets — creados en M-04)
│       └── components/
│           ├── hcloud-ccm.yaml
│           └── hcloud-csi-driver.yaml
```

### 10.2 `kustomization.yaml` del overlay prod

```yaml
# infra/overlays/hetzner-prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: serenamente-core  # namespace por defecto

resources:
  # Base compartida con laptop-dev
  - ../../base

  # Recursos específicos de producción
  - sealed-secrets/postgres-superuser.yaml
  - sealed-secrets/kratos-db-credentials.yaml
  - sealed-secrets/iam-db-credentials.yaml
  - sealed-secrets/kratos-internal-secrets.yaml
  - sealed-secrets/sendgrid-smtp.yaml
  - sealed-secrets/b2-backup-credentials.yaml
  - sealed-secrets/hcloud-credentials.yaml
  - components/hcloud-ccm.yaml
  - components/hcloud-csi-driver.yaml

# Patches que modifican recursos del base
patches:
  # Traefik: hostPort → LoadBalancer
  - path: patches/traefik-loadbalancer.yaml
    target:
      kind: HelmRelease
      name: traefik

  # CloudNativePG: local-path → hcloud-volumes
  - path: patches/cnpg-hcloud-volumes.yaml
    target:
      kind: Cluster
      name: serenamente-db

  # Kratos: dev config → prod config
  - path: patches/kratos-prod-config.yaml
    target:
      kind: ConfigMap
      name: kratos-config

  # ClusterIssuer: local-ca → letsencrypt-prod
  - path: patches/letsencrypt-prod-issuer.yaml
    target:
      kind: ClusterIssuer
      name: local-ca

  # Imágenes: registry local → GHCR
  - path: patches/image-registry-ghcr.yaml

# No incluir componentes de desarrollo
# (Mailpit, MinIO, registry local son EXCLUIDOS del base por defecto
#  y solo se incluyen en el overlay laptop-dev)
```

### 10.3 Patch: Traefik LoadBalancer

```yaml
# infra/overlays/hetzner-prod/patches/traefik-loadbalancer.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: traefik
  namespace: traefik
spec:
  values:
    # En producción: Deployment + Service LoadBalancer (no DaemonSet + hostPort)
    deployment:
      kind: Deployment
      replicas: 1

    service:
      enabled: true
      type: LoadBalancer
      # El Hetzner CCM asigna la Floating IP automáticamente
      annotations:
        load-balancer.hetzner.cloud/name: serenamente-prod-lb
        load-balancer.hetzner.cloud/location: fsn1

    # Sin hostNetwork ni hostPort en producción
    hostNetwork: false

    ports:
      web:
        port: 80
        exposedPort: 80
        hostPort: null  # deshabilitar hostPort
      websecure:
        port: 443
        exposedPort: 443
        hostPort: null  # deshabilitar hostPort
```

### 10.4 Patch: CloudNativePG con hcloud-volumes

```yaml
# infra/overlays/hetzner-prod/patches/cnpg-hcloud-volumes.yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: serenamente-db
  namespace: serenamente-data
spec:
  # Más instancias en prod (alta disponibilidad)
  instances: 1  # empezar con 1, escalar cuando haya más carga

  storage:
    # En producción: Hetzner CSI Driver (volúmenes en la nube)
    storageClass: hcloud-volumes
    size: 20Gi

  # PostgreSQL tuning para producción (8GB RAM disponible)
  postgresql:
    parameters:
      shared_buffers: "2GB"
      effective_cache_size: "6GB"
      maintenance_work_mem: "512MB"
      checkpoint_completion_target: "0.9"
      wal_buffers: "64MB"
      default_statistics_target: "100"
      random_page_cost: "1.1"  # SSD NVMe
      effective_io_concurrency: "200"
      work_mem: "32MB"
      min_wal_size: "1GB"
      max_wal_size: "4GB"
      max_worker_processes: "4"  # = vCPUs en CX32
      max_parallel_workers_per_gather: "2"
      max_parallel_workers: "4"
      max_parallel_maintenance_workers: "2"
      log_statement: "none"  # no loggear todos los queries en prod
      log_duration: "off"
      log_min_duration_statement: "1000"  # loggear queries > 1 segundo

  # Backup real en producción: Backblaze B2 via barman-cloud
  backup:
    barmanObjectStore:
      destinationPath: "s3://${B2_BUCKET_NAME}/cnpg-backup"
      endpointURL: "https://s3.${B2_BUCKET_REGION}.backblazeb2.com"
      s3Credentials:
        accessKeyId:
          name: b2-backup-credentials
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: b2-backup-credentials
          key: ACCESS_SECRET_KEY
      wal:
        compression: gzip
        maxParallel: 8
      data:
        compression: gzip
    retentionPolicy: "30d"

  # Monitoreo activado en producción
  monitoring:
    enablePodMonitor: true
```

### 10.5 Patch: ClusterIssuer Let's Encrypt

```yaml
# infra/overlays/hetzner-prod/patches/letsencrypt-prod-issuer.yaml
# Este patch REEMPLAZA el ClusterIssuer "local-ca" (self-signed) con Let's Encrypt
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: local-ca  # mismo nombre, diferente spec (patch lo reemplaza)
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ops@serenamente.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      # Usar HTTP-01 challenge (Traefik debe estar expuesto públicamente)
      - http01:
          ingress:
            class: traefik
```

**Alternativa con DNS-01 (si se usa Cloudflare):** DNS-01 permite obtener certificados wildcards (`*.serenamente.com`), lo cual es más conveniente. Requiere un Cloudflare API Token con permiso `Zone:DNS:Edit`.

```yaml
# Alternativa con DNS-01 para wildcards
spec:
  acme:
    solvers:
      - dns01:
          cloudflare:
            email: ops@serenamente.com
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
```

### 10.6 Hetzner CCM (Cloud Controller Manager)

El Hetzner CCM es necesario para que los Services de tipo `LoadBalancer` funcionen. Sin él, los Services de tipo `LoadBalancer` quedarán en estado `Pending` indefinidamente.

```yaml
# infra/overlays/hetzner-prod/components/hcloud-ccm.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: kube-system
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: hcloud-cloud-controller-manager
  namespace: kube-system
spec:
  interval: 10m
  chart:
    spec:
      chart: hcloud-cloud-controller-manager
      version: ">=1.20.0"
      sourceRef:
        kind: HelmRepository
        name: hcloud
        namespace: flux-system
  values:
    env:
      NODE_NAME:
        valueFrom:
          fieldRef:
            fieldPath: spec.nodeName
    networking:
      enabled: true
      clusterCIDR: "10.244.0.0/16"
    existingSecret:
      name: hcloud-credentials
      key: token
```

---

## 11. M-07 — cert-manager con Let's Encrypt en Producción

### 11.1 Verificar el ClusterIssuer Let's Encrypt

Después de que FluxCD aplique el overlay, verificar que cert-manager puede comunicarse con los servidores ACME de Let's Encrypt:

```bash
KUBECONFIG=talos/prod/kubeconfig kubectl describe clusterissuer local-ca
# Debe mostrar: Status: Ready, Message: ACME account registered

# Si hay errores, ver los logs de cert-manager
KUBECONFIG=talos/prod/kubeconfig kubectl -n cert-manager logs \
  -l app.kubernetes.io/name=cert-manager \
  --tail=50
```

### 11.2 Verificar emisión de certificados

```bash
# Listar certificados y su estado
KUBECONFIG=talos/prod/kubeconfig kubectl get certificates -A

# Ver el detalle de un certificado específico
KUBECONFIG=talos/prod/kubeconfig kubectl describe certificate \
  serenamente-tls -n serenamente-core

# Estados posibles:
# READY: True  → certificado emitido correctamente
# READY: False → ver Events y CertificateRequest para errores

# Ver el CertificateRequest para más detalles
KUBECONFIG=talos/prod/kubeconfig kubectl get certificaterequest -A
```

### 11.3 Problemas comunes con Let's Encrypt

**Problema 1: HTTP-01 challenge falla**

Let's Encrypt necesita acceder a `http://tudominio.com/.well-known/acme-challenge/TOKEN`. Si el dominio no resuelve aún a la IP del servidor, o si Traefik no está expuesto en el puerto 80, el challenge falla.

Verificar:
```bash
# ¿Traefik tiene una IP asignada en el Service LoadBalancer?
KUBECONFIG=talos/prod/kubeconfig kubectl -n traefik get svc traefik

# Si EXTERNAL-IP muestra <pending>: el Hetzner CCM no está funcionando
# Ver logs del CCM:
KUBECONFIG=talos/prod/kubeconfig kubectl -n kube-system logs \
  -l app.kubernetes.io/name=hcloud-cloud-controller-manager --tail=50

# ¿El DNS ya apunta a la IP correcta?
dig api.serenamente.com
```

**Problema 2: Rate limiting de Let's Encrypt**

Let's Encrypt tiene límites: 5 certificados por dominio por semana en el staging, 50 en producción. Si se hacen muchos intentos fallidos, se puede alcanzar el rate limit.

Para pruebas, usar el servidor staging de Let's Encrypt:
```yaml
server: https://acme-staging-v02.api.letsencrypt.org/directory
```
El certificado de staging es inválido para navegadores pero permite verificar que el proceso funciona.

---

## 12. M-08 — Traefik: de hostPort a LoadBalancer

### 12.1 Cómo funciona en producción

En el laptop de desarrollo, Traefik usaba `hostPort` para exponer los puertos 80 y 443 directamente en la IP del laptop. En producción, el mecanismo es diferente:

```
Internet → Floating IP (Hetzner) → Hetzner Load Balancer → Service LoadBalancer (k8s) → Traefik Pod → IngressRoute → Backend Service
```

El Hetzner CCM detecta automáticamente los Services de tipo `LoadBalancer` en Kubernetes y crea el Load Balancer de Hetzner correspondiente, asociándolo a la Floating IP.

### 12.2 Verificar que el Service LoadBalancer tiene IP asignada

```bash
KUBECONFIG=talos/prod/kubeconfig kubectl -n traefik get svc traefik
# Debe mostrar:
# NAME      TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)
# traefik   LoadBalancer   10.96.x.x       ${FLOATING_IP}   80:xxxxx/TCP,443:xxxxx/TCP

# Si EXTERNAL-IP está en <pending> durante más de 5 minutos, hay un problema con el CCM
```

### 12.3 Verificar IngressRoutes

```bash
# Listar todas las IngressRoutes
KUBECONFIG=talos/prod/kubeconfig kubectl get ingressroute -A

# Verificar una ruta específica
KUBECONFIG=talos/prod/kubeconfig kubectl describe ingressroute \
  api-serenamente-route -n serenamente-core

# Probar conectividad HTTP (antes del DNS cutover, usando la IP directamente)
curl -k https://${FLOATING_IP}/health \
  -H "Host: api.serenamente.com"
# Debe responder 200 OK (TLS inválido porque el cert es para api.serenamente.com pero conectamos por IP)
```

---

## 13. M-09 — CloudNativePG con hcloud-volumes y Backblaze B2

### 13.1 Verificar el Cluster CRD

```bash
KUBECONFIG=talos/prod/kubeconfig kubectl -n serenamente-data get cluster
# NAME              AGE   INSTANCES   READY   STATUS               PRIMARY
# serenamente-db    5m    1           1       Cluster in healthy state   serenamente-db-1

# Ver detalles
KUBECONFIG=talos/prod/kubeconfig kubectl -n serenamente-data describe cluster serenamente-db
```

### 13.2 Verificar el PVC con hcloud-volumes

```bash
KUBECONFIG=talos/prod/kubeconfig kubectl -n serenamente-data get pvc
# NAME                    STATUS   VOLUME              CAPACITY   ACCESS MODES   STORAGECLASS
# serenamente-db-1        Bound    pvc-xxxxxxxx        20Gi       RWO            hcloud-volumes

# El VOLUME debe estar en estado Bound (no Pending)
# Si está Pending: el Hetzner CSI Driver no está funcionando
KUBECONFIG=talos/prod/kubeconfig kubectl -n kube-system get pods | grep csi
```

### 13.3 Verificar backup inicial a Backblaze B2

```bash
# Crear el ScheduledBackup CRD
cat <<'EOF' | KUBECONFIG=talos/prod/kubeconfig kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: serenamente-db-daily
  namespace: serenamente-data
spec:
  schedule: "0 2 * * *"  # 2:00 AM UTC todos los días
  cluster:
    name: serenamente-db
  backupOwnerReference: self
  immediate: true  # forzar backup inmediato para verificar
EOF

# Esperar y verificar el resultado del backup
sleep 30
KUBECONFIG=talos/prod/kubeconfig kubectl -n serenamente-data get backup
# Debe mostrar un backup con STATUS: completed

# Verificar en Backblaze B2 que el backup llegó
# (verificar en el panel web de Backblaze o con la CLI b2)
b2 ls "${B2_BUCKET_NAME}/cnpg-backup/"
```

### 13.4 Ejecutar las migraciones de base de datos

Las bases de datos en producción se inicializan desde cero (no se migran datos del desarrollo — están en fase dev sin usuarios reales). El `postInitSQL` del Cluster CRD crea las bases de datos y extensiones automáticamente.

Si hay migraciones adicionales (scripts SQL de inicialización de esquema):

```bash
# Verificar que las bases de datos existen
KUBECONFIG=talos/prod/kubeconfig kubectl exec -n serenamente-data \
  serenamente-db-1 -- psql -U postgres -c "\l"

# Ejecutar migraciones de Kratos (si no se ejecutaron vía init container)
KUBECONFIG=talos/prod/kubeconfig kubectl exec -n serenamente-core \
  deployment/kratos -- \
  kratos migrate sql --yes \
  "postgres://kratos_user:${KRATOS_DB_PASSWORD}@serenamente-db-rw.serenamente-data.svc.cluster.local:5432/kratos_db?sslmode=require"
```

---

## 14. M-10 — Ory Kratos: dominios reales y SendGrid

### 14.1 Diferencias de configuración Kratos dev → prod

El patch de Kratos en producción cambia:

```yaml
# infra/overlays/hetzner-prod/patches/kratos-prod-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kratos-config
  namespace: serenamente-core
data:
  kratos.yaml: |
    version: v1.3.1

    dsn: "DSN_PLACEHOLDER"  # se inyecta desde SealedSecret kratos-db-credentials

    serve:
      public:
        base_url: https://kratos.serenamente.com
        cors:
          enabled: true
          allowed_origins:
            - https://app.serenamente.com
          allowed_methods: [POST, GET, PUT, PATCH, DELETE]
          allowed_headers: [Authorization, Cookie, Content-Type]
          exposed_headers: [Content-Type, Set-Cookie]
      admin:
        base_url: http://kratos-admin.serenamente-core.svc.cluster.local:4434

    selfservice:
      default_browser_return_url: https://app.serenamente.com/
      allowed_return_urls:
        - https://app.serenamente.com

      flows:
        error:
          ui_url: https://app.serenamente.com/error
        login:
          ui_url: https://app.serenamente.com/login
          lifespan: 10m
        logout:
          default_browser_return_url: https://app.serenamente.com/login
        registration:
          lifespan: 10m
          ui_url: https://app.serenamente.com/register
          after:
            password:
              hooks:
                - hook: session
        verification:
          enabled: true
          use: code
          ui_url: https://app.serenamente.com/verification
          lifespan: 15m

      methods:
        # En producción: password deshabilitado, solo OIDC y código de verificación
        password:
          enabled: false
        code:
          enabled: true
        link:
          enabled: false
        oidc:
          enabled: true  # Google, GitHub, etc. (si se configuran providers)

    courier:
      smtp:
        connection_uri: "SENDGRID_SMTP_URI_PLACEHOLDER"  # desde SealedSecret
        from_address: noreply@serenamente.com
        from_name: Serenamente

    secrets:
      cookie:
        - "COOKIE_SECRET_PLACEHOLDER"  # desde SealedSecret
      cipher:
        - "CIPHER_SECRET_PLACEHOLDER"  # desde SealedSecret

    hashers:
      argon2:
        parallelism: 1
        memory: 128MB
        iterations: 2
        salt_length: 16
        key_length: 32

    log:
      level: warn  # en prod: solo warnings y errores
      format: json
      leak_sensitive_values: false  # NUNCA en producción
```

### 14.2 Verificar que Kratos puede enviar emails via SendGrid

```bash
# Verificar que Kratos está Running
KUBECONFIG=talos/prod/kubeconfig kubectl -n serenamente-core get pods -l app=kratos

# Ver logs de Kratos en producción
KUBECONFIG=talos/prod/kubeconfig kubectl -n serenamente-core logs \
  -l app=kratos --tail=50

# Probar el endpoint de salud de Kratos
KUBECONFIG=talos/prod/kubeconfig kubectl port-forward \
  -n serenamente-core svc/kratos-public 4433:4433 &

curl https://localhost:4433/health/ready
# {"status":"ok"}

# Detener el port-forward
kill %1
```

---

## 15. M-11 — IAM Domain Service: imagen desde GHCR

### 15.1 Publicar la imagen en GitHub Container Registry

Antes del DNS cutover, la imagen del IAM Service debe estar en GHCR:

```bash
# En la máquina de desarrollo (no el servidor):
# 1. Autenticarse en GHCR
echo "${GITHUB_PAT}" | docker login ghcr.io -u "${GITHUB_USER}" --password-stdin

# 2. Construir y pushear la imagen
docker build -t ghcr.io/serenamente/iam-service:latest \
  -f services/iam-service/Dockerfile \
  services/iam-service/

docker push ghcr.io/serenamente/iam-service:latest

# 3. Alternativamente, usar el tag de versión semántica
VERSION=$(git describe --tags --always --dirty)
docker tag ghcr.io/serenamente/iam-service:latest \
  ghcr.io/serenamente/iam-service:${VERSION}
docker push ghcr.io/serenamente/iam-service:${VERSION}
```

### 15.2 Configurar acceso privado al registry GHCR

Si el repositorio es privado, k8s necesita credenciales para hacer pull de las imágenes:

```bash
# Crear el secret de acceso a GHCR
kubectl create secret docker-registry ghcr-credentials \
  --namespace=serenamente-core \
  --docker-server=ghcr.io \
  --docker-username="${GITHUB_USER}" \
  --docker-password="${GITHUB_PAT}" \
  --docker-email="ops@serenamente.com" \
  --dry-run=client -o yaml | \
kubeseal \
  --cert pub-keys/serenamente-prod.pem \
  --format yaml \
  > infra/overlays/hetzner-prod/sealed-secrets/ghcr-credentials.yaml

# Agregar imagePullSecrets al Deployment del IAM Service
# (en el patch de producción)
```

### 15.3 Patch de imagen del IAM Service

```yaml
# infra/overlays/hetzner-prod/patches/image-registry-ghcr.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iam-service
  namespace: serenamente-core
spec:
  template:
    spec:
      imagePullSecrets:
        - name: ghcr-credentials
      containers:
        - name: iam-service
          image: ghcr.io/serenamente/iam-service:latest
          imagePullPolicy: Always
          env:
            - name: LOG_FORMAT
              value: "json"     # JSON en producción
            - name: LOG_LEVEL
              value: "warn"     # solo warnings y errores en prod
            - name: ENV
              value: "production"
```

---

## 16. M-12 — DNS en Cloudflare: cutover

El DNS cutover es el momento en que el tráfico real empieza a llegar al servidor de producción. Antes de hacerlo, todos los servicios deben estar funcionando correctamente.

### 16.1 Registros DNS necesarios

| Tipo | Nombre | Valor | Proxied | TTL |
|------|--------|-------|---------|-----|
| A | `serenamente.com` | `${FLOATING_IP}` | No (DNS only) | 60s |
| A | `www.serenamente.com` | `${FLOATING_IP}` | No (DNS only) | 60s |
| A | `api.serenamente.com` | `${FLOATING_IP}` | No (DNS only) | 60s |
| A | `app.serenamente.com` | `${FLOATING_IP}` | Sí (Proxied) | Auto |
| A | `kratos.serenamente.com` | `${FLOATING_IP}` | No (DNS only) | 60s |

**Por qué algunos registros NO usan el proxy de Cloudflare:**
- `api.serenamente.com`, `kratos.serenamente.com`: Los certificados Let's Encrypt se obtienen vía HTTP-01 challenge. El proxy de Cloudflare interfiere con el ACME HTTP-01 challenge de cert-manager. Después de obtener el certificado, se puede activar el proxy si se quiere.
- `app.serenamente.com`: Puede usar el proxy de Cloudflare porque es una CF Page (Cloudflare gestiona el TLS).

### 16.2 Crear los registros DNS via API de Cloudflare

```bash
# Variables (obtener CF_ZONE_ID y CF_API_TOKEN desde el panel de Cloudflare)
CF_ZONE_ID="xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
CF_API_TOKEN="xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
FLOATING_IP="X.X.X.X"

# Función helper para crear registros DNS
create_dns_record() {
  local name=$1
  local proxied=$2
  curl -s -X POST \
    "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/dns_records" \
    -H "Authorization: Bearer ${CF_API_TOKEN}" \
    -H "Content-Type: application/json" \
    --data "{
      \"type\": \"A\",
      \"name\": \"${name}\",
      \"content\": \"${FLOATING_IP}\",
      \"proxied\": ${proxied},
      \"ttl\": 60
    }" | jq '.success'
}

# Crear los registros
create_dns_record "serenamente.com" "false"
create_dns_record "www.serenamente.com" "false"
create_dns_record "api.serenamente.com" "false"
create_dns_record "kratos.serenamente.com" "false"

# Verificar propagación (puede tardar 1-5 minutos con TTL 60s)
watch dig +short api.serenamente.com
# Debe devolver ${FLOATING_IP}
```

### 16.3 Verificar la obtención de certificados Let's Encrypt

Una vez que el DNS resuelve correctamente, cert-manager puede obtener los certificados:

```bash
KUBECONFIG=talos/prod/kubeconfig kubectl get certificates -A
# Esperar hasta que READY sea True para todos los certificados

# Si un certificado está en estado False, ver el CertificateRequest
KUBECONFIG=talos/prod/kubeconfig kubectl get certificaterequest -A
KUBECONFIG=talos/prod/kubeconfig kubectl describe certificaterequest \
  serenamente-tls-xxxxx -n serenamente-core
```

---

## 17. M-13 — Verificación y Smoke Testing Post-Migración

### 17.1 Criterios de aceptación técnica (infraestructura)

```bash
# 1. Todos los nodos del cluster están Ready
KUBECONFIG=talos/prod/kubeconfig kubectl get nodes
# serenamente-prod-01   Ready   control-plane   Xm   v1.33.x

# 2. Todos los pods están Running (sin CrashLoopBackOff)
KUBECONFIG=talos/prod/kubeconfig kubectl get pods -A | grep -v Running | grep -v Completed
# Solo debe aparecer el header si todo está bien

# 3. FluxCD está reconciliando correctamente
KUBECONFIG=talos/prod/kubeconfig flux get kustomizations -A
# Todos deben mostrar: READY=True, SUSPENDED=False

# 4. cert-manager ha emitido todos los certificados
KUBECONFIG=talos/prod/kubeconfig kubectl get certificates -A
# Todos deben mostrar: READY=True

# 5. CloudNativePG cluster está saludable
KUBECONFIG=talos/prod/kubeconfig kubectl -n serenamente-data get cluster
# STATUS: Cluster in healthy state

# 6. Traefik tiene IP pública asignada
KUBECONFIG=talos/prod/kubeconfig kubectl -n traefik get svc traefik
# EXTERNAL-IP debe mostrar ${FLOATING_IP}

# 7. Backup inicial a B2 completado
KUBECONFIG=talos/prod/kubeconfig kubectl -n serenamente-data get backup
# STATUS: completed

# 8. Sealed Secrets está descifrando correctamente
KUBECONFIG=talos/prod/kubeconfig kubectl -n serenamente-core get secrets
# Debe incluir los secretos descifrados (iam-db-credentials, kratos-internal-secrets, etc.)
```

### 17.2 Smoke testing funcional end-to-end

```bash
# Test 1: TLS válido en todos los dominios
for domain in api.serenamente.com kratos.serenamente.com; do
  echo -n "TLS ${domain}: "
  curl -s -o /dev/null -w "%{http_code}" "https://${domain}/health"
  echo ""
done

# Test 2: Kratos health check público
curl https://kratos.serenamente.com/health/ready
# {"status":"ok"}

# Test 3: IAM Service health check
curl https://api.serenamente.com/health
# {"status":"ok","version":"x.x.x"}

# Test 4: Registro de usuario nuevo
REGISTRATION_RESPONSE=$(curl -s -X POST \
  https://kratos.serenamente.com/self-service/registration/api \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "flow_id": "FLOW_ID",
    "method": "code",
    "traits": {
      "email": "test-migration@example.com"
    }
  }')
echo ${REGISTRATION_RESPONSE} | jq '.ui.messages'

# Test 5: Verificar que el email de verificación llega via SendGrid
# (verificar en el inbox de test-migration@example.com o en los logs de SendGrid)

# Test 6: Login con el código de verificación
# (completar el flujo de Kratos UI)

# Test 7: Endpoint protegido por ForwardAuth
curl -v https://api.serenamente.com/api/v1/protected-endpoint
# Debe devolver 401 (no autenticado) con redirect a Kratos, no 502/503
```

### 17.3 Verificar WAL streaming y backup continuo

```bash
# Verificar que el WAL se está enviando a B2 continuamente
# (hacer una operación en la DB y verificar que se archiva)
KUBECONFIG=talos/prod/kubeconfig kubectl exec -n serenamente-data \
  serenamente-db-1 -- psql -U postgres -c "SELECT pg_switch_wal();"

# Esperar ~30 segundos y verificar en los logs del barman-cloud-wal-archive
KUBECONFIG=talos/prod/kubeconfig kubectl -n serenamente-data logs \
  serenamente-db-1 -c barman-cloud-wal-archive --tail=20

# Verificar en B2 (via CLI b2)
b2 ls "${B2_BUCKET_NAME}/cnpg-backup/serenamente-db/" --recursive | head -20
```

---

## 18. Plan de Rollback

El rollback a una situación anterior es posible en cualquier punto de la migración porque el entorno de desarrollo en el laptop sigue funcionando intacto. La migración no toca el laptop.

### 18.1 Rollback antes del DNS cutover (trivial)

Si algo falla antes de crear los registros DNS en Cloudflare, el impacto es cero para usuarios finales (no había usuarios en producción). Se puede:

1. Detener el proceso de migración
2. Analizar el problema
3. Recomenzar cuando esté resuelto

El VPS puede dejarse corriendo (ya pagamos el mes) o destruirse (si la causa del fallo requiere empezar desde cero).

### 18.2 Rollback después del DNS cutover

Si el DNS ya apunta al nuevo servidor y hay problemas, el rollback implica restaurar los registros DNS anteriores. Como el VPS reemplazó un entorno de desarrollo (no había VPS anterior), el "rollback" de DNS sería:

```bash
# Eliminar los registros DNS de Cloudflare para que el servicio quede inaccesible
# (preferible a apuntar a un servidor roto)
for record_id in $(curl -s \
  "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/dns_records" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" | \
  jq -r '.result[] | select(.name | contains("serenamente.com")) | .id'); do
  curl -s -X DELETE \
    "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/dns_records/${record_id}" \
    -H "Authorization: Bearer ${CF_API_TOKEN}"
done
```

### 18.3 Recuperación de datos desde backup B2

Si el cluster de producción tiene problemas de datos o corrupción, CloudNativePG puede hacer point-in-time recovery desde el backup en B2:

```yaml
# Cluster de recuperación (nuevo cluster desde backup)
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: serenamente-db-recovery
  namespace: serenamente-data
spec:
  instances: 1
  storage:
    storageClass: hcloud-volumes
    size: 20Gi

  bootstrap:
    recovery:
      source: serenamente-db  # nombre del cluster fuente
      recoveryTarget:
        targetTime: "2026-04-04 03:00:00"  # punto en el tiempo a recuperar

  externalClusters:
    - name: serenamente-db
      barmanObjectStore:
        destinationPath: "s3://${B2_BUCKET_NAME}/cnpg-backup"
        endpointURL: "https://s3.${B2_BUCKET_REGION}.backblazeb2.com"
        s3Credentials:
          accessKeyId:
            name: b2-backup-credentials
            key: ACCESS_KEY_ID
          secretAccessKey:
            name: b2-backup-credentials
            key: ACCESS_SECRET_KEY
```

---

## 19. Post-Migración: CI/CD con GitHub Actions

Una vez que el cluster de producción está funcionando, el siguiente paso natural es automatizar el deployment. Cada push a `main` debe producir un deployment automático.

### 19.1 Workflow de CI/CD básico

```yaml
# .github/workflows/deploy.yaml
name: Deploy to Production

on:
  push:
    branches: [main]
    paths:
      - 'services/iam-service/**'
      - 'infra/overlays/hetzner-prod/**'

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract version
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/serenamente/iam-service
          tags: |
            type=sha,prefix=,suffix=,format=short
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push IAM Service
        uses: docker/build-push-action@v5
        with:
          context: services/iam-service
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Update image tag in kustomization
        run: |
          cd infra/overlays/hetzner-prod
          # Actualizar el tag de imagen con el SHA del commit
          IMAGE_TAG=$(echo ${{ github.sha }} | cut -c1-7)
          kustomize edit set image \
            ghcr.io/serenamente/iam-service=ghcr.io/serenamente/iam-service:${IMAGE_TAG}
          git config user.email "ci@serenamente.com"
          git config user.name "GitHub Actions"
          git add kustomization.yaml
          git commit -m "ci: update iam-service image to ${IMAGE_TAG}"
          git push origin main

      # FluxCD detecta el commit automáticamente y reconcilia el cluster
      # No necesitamos hacer kubectl apply desde el CI
```

### 19.2 Actualización manual de imagen con FluxCD Image Automation

Alternativamente, FluxCD puede detectar automáticamente nuevas imágenes en GHCR y actualizar el repositorio:

```yaml
# infra/overlays/hetzner-prod/flux/image-policy.yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: iam-service
  namespace: flux-system
spec:
  image: ghcr.io/serenamente/iam-service
  interval: 5m
  secretRef:
    name: ghcr-credentials
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: iam-service
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: iam-service
  policy:
    semver:
      range: ">=1.0.0"
---
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: iam-service
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  git:
    push:
      branch: main
    commit:
      author:
        name: FluxCD Bot
        email: flux@serenamente.com
      messageTemplate: "ci: update iam-service to {{range .Updated.Images}}{{.}}{{end}}"
  update:
    path: "./infra/overlays/hetzner-prod"
    strategy: Setters
```

---

## 20. Descomisionado del Entorno Laptop (Opcional)

El entorno de desarrollo en el laptop puede seguir funcionando indefinidamente en paralelo con producción. No hay ninguna razón para apagarlo. Sin embargo, si se desea liberar el laptop para otros usos:

### 20.1 Resetear Talos (volver el laptop a su estado original)

```bash
# ADVERTENCIA: Esto destruye TODOS los datos del cluster de desarrollo
# Asegurarse de que no hay datos importantes en el cluster local

# Reset completo de Talos (borra etcd, k8s, y los datos de las PVCs)
talosctl reset \
  --talosconfig talos/dev/talosconfig \
  --nodes 192.168.1.100 \
  --graceful=false \
  --reboot

# El laptop se reiniciará en modo "awaiting configuration" de Talos
# Para reinstalar otro OS, entrar en el BIOS y bootear desde USB
```

### 20.2 Archivar la configuración del cluster de desarrollo

```bash
# Guardar los secretos del cluster de desarrollo (en caso de necesitar
# recrear el entorno de desarrollo en el futuro)
mkdir -p archive/dev-cluster/
cp talos/dev/talosconfig archive/dev-cluster/
cp talos/dev/kubeconfig archive/dev-cluster/
cp talos/dev/secrets.yaml archive/dev-cluster/  # GUARDAR EN LUGAR SEGURO

# Anotar la clave pública de Sealed Secrets del cluster de desarrollo
# (para poder descifrar secretos de desarrollo archivados si fuera necesario)
kubeseal --fetch-cert \
  --controller-name=sealed-secrets \
  --controller-namespace=sealed-secrets \
  --kubeconfig talos/dev/kubeconfig \
  > archive/dev-cluster/sealed-secrets-pubkey.pem
```

---

## 21. Criterios de Aceptación Global de la Migración

La migración se considera exitosa cuando todos estos criterios están cumplidos:

| # | Criterio | Verificación |
|---|----------|-------------|
| MA-01 | El nodo CX32 está en estado `Ready` en Kubernetes | `kubectl get nodes` |
| MA-02 | FluxCD está reconciliando el overlay `hetzner-prod` sin errores | `flux get kustomizations -A` → READY=True |
| MA-03 | Todos los pods están en estado `Running` (sin CrashLoopBackOff) | `kubectl get pods -A` |
| MA-04 | cert-manager ha emitido certificados TLS válidos para todos los dominios | `kubectl get certificates -A` → READY=True |
| MA-05 | Los certificados TLS son reconocidos como válidos por el navegador | `curl https://api.serenamente.com` sin flag `-k` |
| MA-06 | Traefik tiene la Floating IP asignada como EXTERNAL-IP | `kubectl -n traefik get svc traefik` |
| MA-07 | CloudNativePG cluster está en estado `Cluster in healthy state` | `kubectl -n serenamente-data get cluster` |
| MA-08 | Al menos un backup completo a Backblaze B2 está en estado `completed` | `kubectl -n serenamente-data get backup` |
| MA-09 | Kratos responde `/health/ready` con `{"status":"ok"}` via HTTPS público | `curl https://kratos.serenamente.com/health/ready` |
| MA-10 | El IAM Service responde `/health` con `{"status":"ok"}` via HTTPS público | `curl https://api.serenamente.com/health` |
| MA-11 | El flujo completo de registro → email → verificación → login funciona | Prueba manual con email real |
| MA-12 | Los Sealed Secrets están siendo descifrados correctamente en producción | `kubectl -n serenamente-core get secrets` muestra los secretos |
| MA-13 | Los registros DNS de Cloudflare apuntan a la Floating IP correcta | `dig api.serenamente.com` devuelve `${FLOATING_IP}` |
| MA-14 | El WAL de PostgreSQL se está archivando continuamente en B2 | Logs de `barman-cloud-wal-archive` sin errores |
| MA-15 | Ningún pod tiene `imagePullBackOff` (todas las imágenes accesibles desde GHCR) | `kubectl get pods -A \| grep -v Running` |
| MA-16 | Los logs de Kratos no muestran `leak_sensitive_values: true` | `kubectl logs -l app=kratos \| grep leak` → sin resultados |
| MA-17 | El Hetzner CCM está gestionando los Load Balancers correctamente | `kubectl -n kube-system logs -l app=hcloud-cloud-controller-manager` |
| MA-18 | El Hetzner CSI Driver está provisionando PVCs sin errores | `kubectl -n kube-system get pods \| grep csi` → Running |
| MA-19 | FluxCD puede pushear al repositorio `main` (verificación Image Automation) | `flux get imageupdateautomations` → READY=True |
| MA-20 | La Floating IP sobrevive a un reinicio del servidor (test de resiliencia) | `hcloud server reboot serenamente-prod-01` + verificar que IP sigue respondiendo |

---

## 22. Checklist de Migración (Resumen Operativo)

Este checklist es una versión condensada para ejecutar el día de la migración.

### Fase 0: Preparación (días antes)
- [ ] Todos los criterios de aceptación de `plans/07` completados en el laptop
- [ ] Overlay `infra/overlays/hetzner-prod/` creado y validado con `kubectl kustomize --dry-run`
- [ ] Credenciales de producción recopiladas (SendGrid, B2, Hetzner Token, GitHub PAT)
- [ ] Dominio `serenamente.com` registrado con nameservers en Cloudflare
- [ ] Cuenta Hetzner con billing activo

### Fase 1: Infraestructura (Día de migración, ~2h)
- [ ] Crear servidor CX32 en Hetzner (M-01)
- [ ] Crear y asignar Floating IP (M-01)
- [ ] Instalar Talos vía rescue mode (M-02)
- [ ] Aplicar patch de configuración Talos para producción (M-02)
- [ ] Bootstrap etcd y obtener kubeconfig (M-02)
- [ ] Verificar que el nodo está `Ready`

### Fase 2: Plataforma (~2h)
- [ ] Instalar Sealed Secrets y obtener clave pública de producción (M-03)
- [ ] Crear y commitear todos los secretos sellados con clave prod (M-04)
- [ ] FluxCD bootstrap apuntando a `main` branch (M-05)
- [ ] Verificar que FluxCD reconcilia sin errores
- [ ] Hetzner CCM y CSI Driver operativos (M-06)

### Fase 3: Servicios (~1h)
- [ ] cert-manager con ClusterIssuer letsencrypt-prod operativo (M-07)
- [ ] Traefik con Service LoadBalancer y Floating IP asignada (M-08)
- [ ] CloudNativePG cluster healthy con hcloud-volumes (M-09)
- [ ] Backup inicial a B2 completado (M-09)
- [ ] Ory Kratos con dominios prod y SendGrid (M-10)
- [ ] IAM Service con imagen desde GHCR (M-11)

### Fase 4: DNS Cutover (~30 min)
- [ ] Verificar que todos los pods están Running antes del cutover
- [ ] Crear registros DNS en Cloudflare apuntando a Floating IP (M-12)
- [ ] Esperar propagación DNS (1-5 minutos con TTL 60s)
- [ ] Verificar emisión de certificados Let's Encrypt (M-12)

### Fase 5: Verificación (~1h)
- [ ] Ejecutar todos los criterios de aceptación MA-01 a MA-20 (M-13)
- [ ] Smoke testing funcional end-to-end (M-13)
- [ ] Verificar WAL archiving a B2 (M-13)
- [ ] Configurar monitoreo de alertas (opcional, Fase 3 del proyecto)

### Fase 6: Post-migración
- [ ] Eliminar credenciales en claro del sistema local (`shred /tmp/prod-secrets.env`)
- [ ] Configurar CI/CD en GitHub Actions (M-19)
- [ ] Documentar la IP de producción y credenciales en el gestor de contraseñas del equipo
- [ ] Decidir si mantener o descomisionar el laptop de desarrollo (M-20)

---

*Documento generado: Abril 2026 | Versión: 1.0*
*Para el entorno de desarrollo: ver [`plans/07_fase1_laptop_talos_desarrollo.md`](./07_fase1_laptop_talos_desarrollo.md)*
