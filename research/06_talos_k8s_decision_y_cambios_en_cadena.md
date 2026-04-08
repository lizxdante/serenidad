# ADR Maestro: Talos Linux + Kubernetes Upstream como Stack de Infraestructura Definitivo
## Análisis, Validación, Investigación y Cambios en Cadena

| Campo | Valor |
|-------|-------|
| **Proyecto** | Serenidad — Clínica Digital de Salud Mental |
| **Versión** | 1.0 |
| **Fecha** | Abril 2026 |
| **Autor** | djca |
| **Estado** | APROBADO — Reemplaza la configuración Ubuntu + Docker Compose |
| **Referencias** | `target_arch/descripcion.md` v3.1, `plans/05_orden_implementacion_capas_exhaustivo.md`, investigación Tavily abril 2026 |

---

## Tabla de Contenidos

1. [Propuesta Original y Corrección Técnica Crítica](#1-propuesta-original-y-corrección-técnica-crítica)
2. [Validación contra Restricciones de Diseño R1–R7](#2-validación-contra-restricciones-de-diseño-r1r7)
3. [Hallazgos de Investigación — Estado del Arte Abril 2026](#3-hallazgos-de-investigación--estado-del-arte-abril-2026)
4. [Decisión Final: Stack Definitivo de Infraestructura](#4-decisión-final-stack-definitivo-de-infraestructura)
5. [Tabla Completa de Cambios en Cadena](#5-tabla-completa-de-cambios-en-cadena)
6. [Cambio 1: OS — Ubuntu → Talos Linux v1.10.x](#6-cambio-1-os--ubuntu--talos-linux-v110x)
7. [Cambio 2: Orquestación — Docker Compose → Kubernetes 1.33.x](#7-cambio-2-orquestación--docker-compose--kubernetes-133x)
8. [Cambio 3: VPS — CX31 4 GB → CX32 8 GB](#8-cambio-3-vps--cx31-4-gb--cx32-8-gb)
9. [Cambio 4: Ingress + TLS — Caddy → cert-manager + Traefik v3.x](#9-cambio-4-ingress--tls--caddy--cert-manager--traefik-v3x)
10. [Cambio 5: PostgreSQL — Container Directo → CloudNativePG Operator](#10-cambio-5-postgresql--container-directo--cloudnativepg-operator)
11. [Cambio 6: Backups — pg_dump cron → CloudNativePG WAL + Barman B2](#11-cambio-6-backups--pg_dump-cron--cloudnativepg-wal--barman-b2)
12. [Cambio 7: Secretos — .env Files → Kubernetes Secrets + Sealed Secrets](#12-cambio-7-secretos--env-files--kubernetes-secrets--sealed-secrets)
13. [Cambio 8: Deployment — docker compose up → FluxCD GitOps + Helm](#13-cambio-8-deployment--docker-compose-up--fluxcd-gitops--helm)
14. [Cambio 9: Observabilidad — Docker Compose Services → kube-prometheus-stack](#14-cambio-9-observabilidad--docker-compose-services--kube-prometheus-stack)
15. [Cambio 10: Escalado Fase 3+ — Docker Swarm → Add k8s Worker Nodes](#15-cambio-10-escalado-fase-3--docker-swarm--add-k8s-worker-nodes)
16. [Nuevos ADRs (ADR-010 a ADR-014)](#16-nuevos-adrs-adr-010-a-adr-014)
17. [ADRs Actualizados (ADR-003, ADR-006)](#17-adrs-actualizados-adr-003-adr-006)
18. [Impacto en Fases: Mapa de Cambios por Fase](#18-impacto-en-fases-mapa-de-cambios-por-fase)
19. [Estructura de Namespaces Kubernetes](#19-estructura-de-namespaces-kubernetes)
20. [Estructura GitOps: Monorepo + FluxCD](#20-estructura-gitops-monorepo--fluxcd)
21. [Mapa de Helm Charts por Servicio](#21-mapa-de-helm-charts-por-servicio)
22. [Análisis de Costos: Antes vs Después](#22-análisis-de-costos-antes-vs-después)
23. [Riesgos y Mitigaciones del Cambio](#23-riesgos-y-mitigaciones-del-cambio)

---

## 1. Propuesta Original y Corrección Técnica Crítica

### 1.1 La Propuesta

Se propuso adoptar **Talos Linux + k3s** como OS y orquestador de Kubernetes desde Fase 1, reemplazando Ubuntu 24.04 LTS + Docker Compose.

### 1.2 Corrección Técnica Crítica: No Existe "Talos + k3s"

Esta distinción es fundamental y cambia el análisis completo:

**Talos Linux NO es un OS de propósito general sobre el cual se instala k3s.** Talos es un sistema operativo diseñado exclusivamente para Kubernetes que:

1. **Gestiona la instalación de Kubernetes directamente** — usando el mismo mecanismo que `kubeadm`, desplegando upstream Kubernetes (vanilla). No existe un paso separado de "instalar k3s".
2. **No tiene shell, no tiene SSH, no tiene package manager** — no hay un mecanismo para ejecutar `curl -sfL https://get.k3s.io | sh -` en un nodo Talos.
3. **Talos ES el control plane + OS atomic unit** — el OS y Kubernetes se configuran, se actualizan y se recuperan juntos como una sola unidad declarativa.

La comparación correcta es:
```
Talos Linux + Kubernetes upstream
        vs
Ubuntu/Flatcar + k3s (distribución ligera)
        vs
Ubuntu/Flatcar + RKE2 (distribución hardened)
```

k3s es una **distribución de Kubernetes** que se instala en sistemas operativos de propósito general. Talos no es un OS de propósito general; es un appliance de Kubernetes. Ambos resuelven el mismo problema de formas diametralmente distintas.

**Decisión resultante:** Se adopta **Talos Linux v1.10.x + Kubernetes 1.33.x (upstream)**, no "Talos + k3s".

---

## 2. Validación contra Restricciones de Diseño R1–R7

### R1 — No MVP — Build It Right The First Time ✅ ALINEADO (Refuerza)

> "Cero deuda técnica asumida. Estándares definitivos desde el día cero."

El cambio a Talos + Kubernetes **refuerza activamente R1** al eliminar la deuda técnica de orquestación que existía en el plan anterior:

- **Deuda eliminada:** El plan anterior usaba Docker Compose en Fase 1-2 y prometía migrar a Docker Swarm en Fase 3+. Esta migración es una **refactorización estructural de operaciones** — exactamente lo que R1 prohíbe.
- **Con Talos + k8s desde Fase 1:** El modelo operativo es definitivo. Escalar de 1 a N nodos es `talosctl gen config worker && talosctl apply-config` — no una migración arquitectónica.
- **Inmutabilidad del OS:** Talos ejecuta desde SquashFS firmado y versionado. El OS no puede derivar de su configuración declarada. Esto es Build It Right The First Time aplicado a la infraestructura misma.

### R2 — Inmutabilidad Arquitectónica ✅ ALINEADO (Refuerza Fuertemente)

> "La arquitectura debe escalar de 1 a 10,000 médicos sin refactorizaciones estructurales."

- **Docker Compose → Swarm → k8s** era una ruta con **dos** refactorizaciones estructurales de operaciones.
- **Talos + k8s desde Fase 1:** Escalar de 1 nodo a N nodos es idéntico en todos los niveles. Agregar un worker node no requiere cambiar ni un solo Helm values file, ni un solo Kubernetes manifest. La arquitectura es estructuralmente idéntica a 1 médico o a 10,000.

### R3 — Desacoplamiento Absoluto ✅ ALINEADO

> "Lógica de negocio aislada de infraestructura, base de datos y frameworks de UI."

Kubernetes **impone** desacoplamiento estructural:
- Los microservicios Go no conocen el nombre del host donde corren — solo conocen el nombre del Service de Kubernetes.
- La base de datos es un endpoint de Service, no un hostname específico.
- Los Deployments son declarativos y portables a cualquier cluster k8s conforme.

### R4 — Soberanía Tecnológica ✅ ALINEADO

> "Open source y self-hosted prioritariamente. Sin dependencia de terceros en el path crítico."

- **Talos Linux**: MPL 2.0. 100% open source. Mantenido por Sidero Labs.
- **Kubernetes**: Apache 2.0. CNCF graduated project. El proyecto open source más adoptado en infraestructura cloud-native.
- **cert-manager**: Apache 2.0. CNCF graduated project.
- **Traefik v3**: MIT. Open source.
- **FluxCD**: Apache 2.0. CNCF graduated project.
- **CloudNativePG**: Apache 2.0. CNCF sandbox project (en camino a incubación). Mantenido por EDB.

Ningún componente tiene dependencia de terceros en el path crítico. Todo es self-hosted.

### R5 — Costo Mínimo de Propiedad ✅ ALINEADO con ajuste de VPS

> "Software open source + self-hosted. TCO < $35/mes en fases iniciales."

El único cambio de costo es el VPS:
- **Antes:** Hetzner CX31 (4 GB RAM) ≈ $10/mes
- **Después:** Hetzner CX32 (8 GB RAM) ≈ €6.80/mes ≈ $7.50/mes

**La actualización de VPS es en realidad más barata**, porque la línea actual de Hetzner (CX22/CX32) ofrece mejor relación precio/rendimiento que la línea CX31 anterior:
- CX32: 4 vCPU AMD, 8 GB RAM, 80 GB SSD por €6.80/mes — más vCPU, más RAM, mismo storage, precio inferior al CX31 previo
- TCO total Fase 1 con CX32: ~€6.80 (VPS) + ~$0.05 (B2 backups) + $0 (Cloudflare free) ≈ **$7.60/mes** — bajo el threshold de $35.

### R6 — Ultravelocidad ✅ ALINEADO

> "TTI < 100ms en frontend. Latencia P99 < 50ms en APIs."

Kubernetes no introduce latencia observable en las rutas críticas:
- **Frontend:** Cloudflare Pages no cambia — Qwik v2.0 y Astro v6.x siguen en Cloudflare Edge. TTI no se ve afectado.
- **API Gateway:** BFF en Cloudflare Workers no cambia.
- **APIs internas:** La comunicación pod-a-pod dentro del mismo nodo pasa por el CNI (loopback en nodo único) con latencia ~1-5μs — equivalente al bridge Docker. La latencia P99 < 50ms es alcanzable.
- **PostgreSQL:** CloudNativePG en el mismo nodo tiene latencia idéntica a un container Docker directo.

Talos tiene específicamente **-49% de Disk I/O** vs el baseline kubeadm en benchmarks de Sidero Labs — beneficia a PostgreSQL y NATS directamente.

### R7 — No Reinventar la Rueda ✅ ALINEADO

> "Usar librerías y herramientas probadas de la industria."

Todos los componentes del nuevo stack son herramientas industry-standard con adopción masiva:
- Talos Linux: usado en producción en Fortune 500 y grandes clusters globales
- Kubernetes: el estándar de orquestación de facto
- cert-manager: ~25M+ instancias desplegadas globalmente
- Traefik v3: millones de instancias de producción
- CloudNativePG: operador PostgreSQL CNCF, respaldado por EDB
- FluxCD: CNCF graduated, ampliamente adoptado en GitOps enterprise

**Veredicto General: La propuesta está perfectamente alineada con todas las restricciones de diseño.** En cinco de siete restricciones, el cambio no solo se alinea sino que refuerza activamente el principio. En R5, el costo real desciende levemente. Solo R6 y R7 son neutrales (no hay degradación).

---

## 3. Hallazgos de Investigación — Estado del Arte Abril 2026

### 3.1 Talos Linux v1.10.x — Estado Actual

**Versión estable:** v1.10.0 (liberada abril 2026)
- Linux kernel 6.12.25
- Kubernetes 1.33.0
- containerd 2.0.5
- etcd 3.5.20

**Características clave confirmadas:**
- OS completo: **<80 MB** comprimido. Ejecuta desde SquashFS firmado.
- Superficie de ataque: **12 binarios únicos** en el PATH del sistema. Comparar: Ubuntu tiene miles.
- Sin SSH, sin shell, sin curl, sin apt. Toda administración via API mTLS con `talosctl`.
- Hetzner Cloud: **ISO público oficial disponible** desde el 23 de abril 2025 con esquema ID `ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515` incluyendo qemu-guest-agent.
- Hetzner CCM (Cloud Controller Manager): **totalmente compatible** con Talos, incluyendo LoadBalancer provisioning y node lifecycle management.
- Terraform modules: `hcloud-talos/terraform-hcloud-talos` — producción-ready.
- Upgrades atómicos: OS + Kubernetes se actualizan juntos en una sola operación (`talosctl upgrade`). Sin divergencia posible entre versión del OS y versión del cluster.
- SELinux enforcing mode: disponible desde v1.10.
- Single-node soportado: `allowSchedulingOnControlPlanes: true` — el scheduler puede asignar pods al nodo control plane.

**Overhead medido en benchmarks (Sidero Labs, comparativo vs kubeadm baseline):**

| Métrica | Talos Linux | k3s | RKE2 | MicroK8s |
|---------|------------|-----|------|----------|
| Memoria | **-7%** | +15% | +150% | +201% |
| CPU | +6% | -19% | +230% | — |
| Disk I/O | **-49%** | +50% | +260% | +533% |
| Footprint almacenamiento | **-47%** | -8% | +143% | — |

**Consumo de RAM estimado para nodo único con full stack:**

| Componente | RAM estimada |
|-----------|-------------|
| Talos OS | ~200 MB |
| etcd | ~200-300 MB |
| kube-apiserver | ~200-300 MB |
| kube-scheduler + controller-manager | ~100-150 MB |
| containerd | ~100 MB |
| CNI (Cilium o Flannel) | ~50-100 MB |
| **Subtotal control plane** | **~850 MB – 1.15 GB** |
| 4 microservicios Go (IAM, Scheduling, Clinical, Billing) × 100 MB | ~400 MB |
| CloudNativePG operator + 1 PG instance | ~300-500 MB |
| NATS JetStream | ~100-200 MB |
| Ory Kratos + OpenFGA | ~150-300 MB |
| Traefik + cert-manager | ~100-150 MB |
| FluxCD | ~50-100 MB |
| **Subtotal workloads Fase 1** | **~1.1-1.65 GB** |
| **TOTAL Fase 1 sin monitoreo** | **~2.0-2.8 GB** |
| kube-prometheus-stack completo | ~500 MB – 1 GB |
| **TOTAL Fase 3 completo** | **~2.5-3.8 GB** |

**Conclusión:** 8 GB (CX32) es el mínimo práctico para el full stack con margen de seguridad. 4 GB (CX22) es insuficiente para producción con observabilidad completa.

### 3.2 Comparativa de Alternativas

| Alternativa | Inmutabilidad | Seguridad | UX Nodo Único | Escala Multi-Nodo | Overhead | Complejidad |
|-------------|--------------|---------|--------------|------------------|---------|------------|
| **Talos + k8s upstream** ⭐ | Excelente | Excelente | Buena (diseño) | Excelente | Medio-Bajo | Alta |
| Flatcar + k3s | Buena (/usr) | Buena | Excelente | Buena | Bajo | Media |
| Ubuntu + k3s | Ninguna | Baseline | Excelente | Buena | Mínimo | Baja |
| Ubuntu + MicroK8s | Ninguna | Baseline | Excelente | Buena | Medio | Baja |
| Talos + RKE2 | Excelente | Excelente | Mala | Excelente | Muy Alto | Muy Alta |

**¿Por qué no Flatcar + k3s?**
Flatcar es el competidor más cercano a Talos. La diferencia clave: Flatcar tiene SSH disponible, ~2,300 binarios en PATH, y solo `/usr` es read-only (no inmutabilidad total del OS). Para una plataforma médica donde los datos clínicos son inmutables por diseño, la consistencia de aplicar ese mismo principio al OS mismo es coherente con R1.

**¿Por qué no Ubuntu + k3s?**
k3s en Ubuntu es la opción más pragmática y de menor curva de aprendizaje. Sin embargo, para el enfoque "Build It Right The First Time" de esta plataforma, Ubuntu introduce: posible drift de configuración, superficie de ataque grande, SSH como vector de ataque, y requerirá hardening manual continuo (fail2ban, ufw, auditd, etc.). La deuda operacional resultante contradice R1.

**¿Por qué no RKE2?**
150% más memoria y 230% más CPU vs baseline para la misma funcionalidad que Talos. No tiene sentido en 8 GB de RAM.

### 3.3 cert-manager + Traefik vs Caddy Ingress en Kubernetes

**Caddy Ingress Controller** (oficial `caddyserver/ingress`):
- Marcado como WIP en el repositorio oficial desde 2024
- La elegancia de Caddy (auto-TLS, Caddyfile sintaxis) se pierde en k8s — debe traducirse a IngressRoute YAML de todos modos
- El fork comunitario `brdelphus/ingress-caddy` añade funcionalidad (WAF, CrowdSec) pero no es el proyecto oficial
- **Veredicto: No recomendado para producción médica**

**cert-manager + Traefik v3.x**:
- cert-manager es CNCF graduated, el estándar absoluto de TLS en Kubernetes. Usado en ~25M+ instancias
- Traefik v3 incluye ForwardAuth Middleware — equivalente funcional del `forward_auth` de Caddy para el IAM Domain Service
- Traefik IngressRoute CRD: más potente que las Ingress nativas de k8s
- Gateway API: Traefik v3 soporta Gateway API, el estándar futuro
- Traefik Dashboard: observabilidad de rutas integrada
- **Veredicto: Elegido como reemplazo de Caddy en Kubernetes**

### 3.4 CloudNativePG vs Plain PostgreSQL Container

| Aspecto | CloudNativePG v1.x | postgres:17-alpine (directo) |
|---------|-------------------|-----------------------------|
| Lifecycle management | CRD declarativo | Manual YAML volumes |
| Backup WAL automático | Sí (barman-cloud → B2/S3) | Manual (cron + pg_dump) |
| Self-healing | Sí (operator restart) | No |
| Rolling upgrades | Sí (operator-managed) | Manual |
| Connection pooling | PgBouncer integrado vía Pooler CRD | Manual |
| TLS entre pods | Automático (operator gestiona certs) | Manual |
| Overhead del operator | ~100-150 MB | Cero |
| 6 databases en 1 cluster | Sí (misma lógica que antes) | Sí (mismo comportamiento) |
| **Veredicto para plataforma médica** | **Obligatorio** | Inaceptable para prod |

Para datos de historia clínica inmutables, backup WAL automático hacia B2 es crítico. `pg_dump` cron es propenso a fallos silenciosos; CloudNativePG usa Point-In-Time Recovery (PITR) con WAL continuo.

---

## 4. Decisión Final: Stack Definitivo de Infraestructura

### Stack Definitivo (Post-ADR)

| Componente | Stack Anterior | Stack Nuevo | Cambio |
|-----------|--------------|------------|--------|
| **OS** | Ubuntu 24.04 LTS | **Talos Linux v1.10.x** | ✅ Reemplazado |
| **Orquestación** | Docker Compose v2.x | **Kubernetes 1.33.x (via Talos)** | ✅ Reemplazado |
| **VPS** | Hetzner CX31 (4 GB) | **Hetzner CX32 (8 GB)** | ✅ Actualizado |
| **Ingress + TLS** | Caddy v2.9+ (standalone) | **cert-manager v1.x + Traefik v3.x** | ✅ Reemplazado |
| **Forward Auth (IAM)** | Caddy `forward_auth` directive | **Traefik ForwardAuth Middleware** | ✅ Equivalente |
| **PostgreSQL** | postgres:17-alpine (container) | **CloudNativePG v1.x + PostgreSQL 17.x** | ✅ Reemplazado |
| **Backup DB** | pg_dump + cron + rclone | **CloudNativePG WAL + barman-cloud → B2** | ✅ Reemplazado |
| **Secretos** | .env files / env vars | **Kubernetes Secrets + Sealed Secrets v0.27+** | ✅ Nuevo |
| **GitOps / Deploy** | SSH + docker compose pull | **FluxCD v2.x (CNCF graduated)** | ✅ Nuevo |
| **Helm** | No aplica (Docker Compose) | **Helm v3.x para todos los charts** | ✅ Nuevo |
| **Observabilidad** | Docker Compose services | **kube-prometheus-stack v82.x (Helm)** | ✅ Mismo stack, k8s-native |
| **Scaling Fase 3+** | Docker Swarm (futuro) | **Add k8s worker nodes (mismo cluster)** | ✅ Simplificado |
| **Gestión cluster** | docker compose CLI | **talosctl + kubectl** | ✅ Nuevo |
| **CI/CD imagen** | Build local | **GitHub Actions → ghcr.io (OCI)** | ✅ Nuevo |

### Componentes Sin Cambio

Los siguientes componentes **NO cambian** — solo el mecanismo de deployment (de Docker Compose a Helm/Kubernetes):

- Cloudflare Pages (Qwik v2.0, Astro v6.x) — sigue igual, no toca k8s
- Cloudflare Workers (Hono v4.x BFF) — sigue igual, no toca k8s
- NATS JetStream v2.11.x — mismo software, ahora via Helm chart oficial
- Ory Kratos v1.3.1 — mismo software, ahora via Helm chart oficial (`ory/kratos`)
- OpenFGA v1.x — mismo software, ahora via Helm chart oficial (`openfga`)
- IAM Domain Service (Go 1.25.x) — mismo código, ahora Kubernetes Deployment
- Scheduling Service (Go 1.25.x) — mismo código, ahora Kubernetes Deployment
- Clinical Record Service (Go 1.25.x) — mismo código, ahora Kubernetes Deployment
- Billing & Ops Service (Go 1.25.x) — mismo código, ahora Kubernetes Deployment
- Protobuf proto3 + buf.build — idéntico
- Backblaze B2 — mismo destino, diferente cliente (barman-cloud vs rclone)
- pgx/v5 pgxpool — driver Go idéntico, se conecta al Service de CloudNativePG
- golang-migrate v4.x — se ejecuta como init container o Job k8s
- UUIDv7, RLS, Outbox Pattern — patrones de código, no cambian

---

## 5. Tabla Completa de Cambios en Cadena

| # | Área | Antes | Después | Fase Impactada | Documentos a Actualizar |
|---|------|-------|---------|----------------|------------------------|
| C-01 | OS | Ubuntu 24.04 LTS | Talos Linux v1.10.x | Fase 1 (1.A) | descripcion.md, 05_, puml |
| C-02 | Orquestación | Docker Compose v2.x | Kubernetes 1.33.x | Fase 1 (1.A) | descripcion.md, 05_, puml |
| C-03 | VPS Spec | CX31 2vCPU 4GB | CX32 4vCPU 8GB | Fase 1 (1.A) | descripcion.md, 04_, 05_, puml |
| C-04 | VPS OS Bootstrap | apt + docker install | talosctl apply-config | Fase 1 (1.A) | 05_ |
| C-05 | Ingress | Caddy v2.9+ standalone | cert-manager + Traefik v3 | Fase 1 (1.B) | descripcion.md, 05_, puml, ADR-006 |
| C-06 | Forward Auth | Caddy `forward_auth` | Traefik ForwardAuth Middleware | Fase 1 (1.B) | 05_, ADR-006 |
| C-07 | TLS Provisioning | Caddy CertMagic | cert-manager + Let's Encrypt | Fase 1 (1.B) | 05_, nuevo ADR-012 |
| C-08 | PostgreSQL Deploy | postgres:17-alpine | CloudNativePG v1.x | Fase 1 (1.C) | descripcion.md, 05_, puml, ADR-003 |
| C-09 | DB Backup | pg_dump + rclone cron | CloudNativePG WAL + barman-cloud | Fase 1 (1.I) | descripcion.md, 05_ |
| C-10 | Secret Management | .env files | Kubernetes Secrets + Sealed Secrets | Fase 1 nueva | 05_, nuevo ADR-015 |
| C-11 | Deployment | docker compose up | FluxCD + Helm | Fase 1 nueva | 05_, nuevo ADR-014 |
| C-12 | Container Registry | Local build | ghcr.io (GitHub Container Registry) | Fase 1 nueva | 05_ |
| C-13 | DB Migrations | golang-migrate CLI | k8s Job / init container | Todas | 05_ |
| C-14 | Health checks | Docker HEALTHCHECK | k8s livenessProbe + readinessProbe | Todas | código Go |
| C-15 | Scaling Fase 3+ | Docker Swarm (futuro) | k8s worker node add | Fase 3 | descripcion.md, 04_ |
| C-16 | Observabilidad Deploy | docker compose services | kube-prometheus-stack Helm | Fase 3 (3.C) | 05_ |
| C-17 | Monorepo estructura | apps/ services/ infra/ | + infra/clusters/ infra/k8s/ | Fase 1 | 05_ |
| C-18 | ADRs | ADR-001 a ADR-009 | Añadir ADR-010 a ADR-014, actualizar ADR-003, ADR-006 | — | descripcion.md |

---

## 6. Cambio 1: OS — Ubuntu → Talos Linux v1.10.x

### Qué cambia

El servidor VPS en Hetzner ya no ejecuta Ubuntu 24.04 LTS con Docker Engine + Docker Compose. En su lugar, ejecuta Talos Linux v1.10.x como appliance de Kubernetes.

### Proceso de Bootstrap

**Paso 1: Obtener imagen Talos para Hetzner**

Usar el schematic ID oficial de Hetzner (incluye qemu-guest-agent requerido por Hetzner):
```
Schematic ID: ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515
```
O generar un schematic personalizado en [factory.talos.dev](https://factory.talos.dev) con extensiones adicionales si es necesario (ej. Cilium CNI, iSCSI para CSI).

**Paso 2: Generar Machine Config**

```bash
# Instalar talosctl en la máquina de desarrollo
brew install siderolabs/tap/talosctl  # macOS
# o: curl -sL https://talos.dev/install | sh

# Generar configuración del cluster
talosctl gen config serenidad-prod https://<VPS_IP>:6443 \
  --with-secrets secrets.yaml \
  --config-patch @patches/single-node.yaml \
  --output-dir ./infra/clusters/hetzner-prod/talos/

# El comando genera:
# - controlplane.yaml (configuración del nodo)
# - talosconfig (credenciales para talosctl)
```

**Patch para nodo único (`patches/single-node.yaml`):**
```yaml
machine:
  network:
    hostname: serenidad-prod-01
  kubelet:
    extraArgs:
      node-labels: "node-role.kubernetes.io/worker="
cluster:
  allowSchedulingOnControlPlanes: true
  network:
    cni:
      name: flannel  # O 'none' si se usa Cilium via Helm
  extraManifests:
    - https://github.com/hetznercloud/hcloud-cloud-controller-manager/releases/latest/download/ccm.yaml
```

**Paso 3: Aplicar configuración al nodo**

```bash
# Boot desde el ISO de Hetzner Talos
# Esperar a que el nodo entre en maintenance mode
talosctl apply-config --insecure \
  --nodes <VPS_IP> \
  --file ./infra/clusters/hetzner-prod/talos/controlplane.yaml

# Bootstrap etcd (solo necesario una vez)
talosctl bootstrap --nodes <VPS_IP>

# Obtener kubeconfig
talosctl kubeconfig --nodes <VPS_IP> --output ./infra/clusters/hetzner-prod/kubeconfig

# Verificar
kubectl --kubeconfig ./infra/clusters/hetzner-prod/kubeconfig get nodes
```

### Qué desaparece

- `apt-get`, `systemctl`, `bash`, SSH — no existen en Talos
- `/etc/` editable — no existe; toda config es via `talosctl apply-config`
- Docker Compose files — reemplazados por Helm charts y Kubernetes manifests
- `adduser serenidad`, `usermod -aG docker` — no existe modelo de usuario en Talos

### Qué herramientas se añaden

- `talosctl` — CLI para gestión del OS (equivalente a `ssh root@vps`)
- `kubectl` — CLI de Kubernetes (siempre fue parte del plan, ahora es el único punto de acceso)
- `helm` v3.x — gestor de charts
- `flux` CLI — GitOps controller management

---

## 7. Cambio 2: Orquestación — Docker Compose → Kubernetes 1.33.x

### Equivalencias de Conceptos

| Docker Compose | Kubernetes |
|---------------|-----------|
| `services:` | `Deployment` + `Service` |
| `image:` | `spec.containers[].image` |
| `ports:` | `Service.spec.ports` |
| `environment:` | `envFrom: - secretRef` |
| `volumes:` | `PersistentVolumeClaim` |
| `networks:` | `Namespace` + `NetworkPolicy` |
| `depends_on:` | `initContainers` + `readinessProbe` |
| `healthcheck:` | `livenessProbe` + `readinessProbe` |
| `restart: unless-stopped` | `restartPolicy: Always` (default) |
| `deploy.resources.limits:` | `resources.limits` |
| `docker compose logs` | `kubectl logs` |
| `docker compose exec` | `kubectl exec` |
| `docker compose ps` | `kubectl get pods` |

### Service Discovery

**Antes (Docker Compose):** Los servicios se comunican por nombre de container en la red Docker: `http://ory-kratos:4433/`

**Después (Kubernetes):** Los Services de Kubernetes mantienen el mismo patrón dentro del mismo namespace: `http://ory-kratos:4433/`. Entre namespaces: `http://ory-kratos.serenidad-core.svc.cluster.local:4433/`.

**El código de los microservicios Go no necesita cambios** para el service discovery interno — el nombre del Service de Kubernetes puede ser idéntico al nombre del container de Docker.

### Estructura de Manifests

Todos los docker-compose.yml se convierten en Helm values files. **No se escriben manifests k8s a mano** — todo se gestiona via Helm charts oficiales o charts propios para los microservicios Go.

---

## 8. Cambio 3: VPS — CX31 4 GB → CX32 8 GB

### Especificaciones

| Especificación | CX31 (anterior) | CX32 (nuevo) |
|---------------|----------------|-------------|
| vCPU | 2 AMD | **4 AMD** |
| RAM | 4 GB | **8 GB** |
| Storage | 80 GB SSD | **80 GB SSD** |
| Precio | ~$10/mes | **~€6.80/mes ≈ $7.50/mes** |
| SO | Ubuntu 24.04 LTS | **Talos Linux v1.10.x** |

El upgrade de 4 GB a 8 GB es **obligatorio** para el control plane de Kubernetes. Sin 8 GB, el nodo único con el full stack experimentará OOM kills en PostgreSQL o Prometheus durante picos de carga.

**Nota:** La línea CX22/CX32 de Hetzner (2026) reemplaza la anterior CX21/CX31. El CX32 ofrece mejor precio/rendimiento que el CX31 anterior.

---

## 9. Cambio 4: Ingress + TLS — Caddy → cert-manager + Traefik v3.x

### Por qué no Caddy Ingress en Kubernetes

El repositorio oficial `caddyserver/ingress` está marcado como WIP. La elegancia del Caddyfile no se traslada a Kubernetes — de todos modos se necesita YAML (IngressRoute). cert-manager + Traefik es la combinación estándar de producción en la industria k8s.

### Arquitectura de Ingress

```
Internet
  │
  ▼
Traefik v3 (DaemonSet, hostPort 80/443)
  │
  ├─► cert-manager ─► Let's Encrypt ACME ─► TLS Secret
  │
  ├─► ForwardAuth Middleware ──► IAM Domain Service :8080 /internal/validate-token
  │       │
  │       └─► (si 200) continúa al servicio destino
  │
  ├─► /api/iam/*      ──► IAM Service :8080
  ├─► /api/scheduling/* ──► Scheduling Service :8081
  ├─► /api/clinical/*  ──► Clinical Service :8082
  ├─► /api/billing/*   ──► Billing Service :8083
  └─► /auth/*          ──► Ory Kratos :4433
```

### Equivalencia Funcional Caddy → Traefik

**Caddyfile anterior:**
```caddy
api.sereni.dad {
  forward_auth iam-service:8080 {
    uri /internal/validate-token
    copy_headers X-User-ID X-User-Role X-Tenant-ID
  }
  handle /api/scheduling/* {
    reverse_proxy scheduling-service:8081
  }
}
```

**IngressRoute + Middleware Traefik equivalente:**
```yaml
# Middleware de ForwardAuth (equivale al forward_auth de Caddy)
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: iam-forward-auth
  namespace: serenidad-core
spec:
  forwardAuth:
    address: http://iam-service:8080/internal/validate-token
    authResponseHeaders:
      - X-User-ID
      - X-User-Role
      - X-Tenant-ID

---
# IngressRoute para los microservicios
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: api-routes
  namespace: serenidad-core
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`api.sereni.dad`) && PathPrefix(`/api/scheduling`)
      kind: Rule
      services:
        - name: scheduling-service
          port: 8081
      middlewares:
        - name: iam-forward-auth
    - match: Host(`api.sereni.dad`) && PathPrefix(`/auth`)
      kind: Rule
      services:
        - name: ory-kratos
          port: 4433
  tls:
    certResolver: letsencrypt
```

**ClusterIssuer (cert-manager) para TLS automático:**
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ops@sereni.dad
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: traefik
```

### Traefik Helm Values (fragmento)

```yaml
# infra/infrastructure/traefik/values.yaml
deployment:
  kind: DaemonSet

ports:
  web:
    port: 80
    hostPort: 80
  websecure:
    port: 443
    hostPort: 443

additionalArguments:
  - "--certificatesresolvers.letsencrypt.acme.email=ops@sereni.dad"
  - "--certificatesresolvers.letsencrypt.acme.storage=/data/acme.json"
  - "--certificatesresolvers.letsencrypt.acme.tlschallenge=true"
  - "--api.dashboard=true"
  - "--api.insecure=false"

providers:
  kubernetesIngress:
    enabled: true
  kubernetesCRD:
    enabled: true
```

---

## 10. Cambio 5: PostgreSQL — Container Directo → CloudNativePG Operator

### Estructura de Deployment

**Antes (docker-compose.yml):**
```yaml
postgres:
  image: postgres:17-alpine
  environment:
    POSTGRES_USER: serenidad_admin
    POSTGRES_PASSWORD: ${PG_ADMIN_PASSWORD}
  volumes:
    - postgres-data:/var/lib/postgresql/data
    - ./infra/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
```

**Después (CloudNativePG Cluster CRD):**
```yaml
# infra/infrastructure/cnpg/cluster.yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: serenidad-pg
  namespace: serenidad-data
spec:
  instances: 1  # Single-node; aumentar a 3 en Fase 3+ para HA
  imageName: ghcr.io/cloudnative-pg/postgresql:17.4
  
  postgresql:
    parameters:
      max_connections: "200"
      shared_buffers: "256MB"
      effective_cache_size: "1GB"
      work_mem: "4MB"
      maintenance_work_mem: "64MB"
      log_statement: "ddl"

  bootstrap:
    initdb:
      database: iam_db
      owner: iam_user
      postInitSQL:
        - CREATE DATABASE scheduling_db OWNER scheduling_user;
        - CREATE DATABASE clinical_db   OWNER clinical_user;
        - CREATE DATABASE billing_db    OWNER billing_user;
        - CREATE DATABASE kratos_db     OWNER kratos_user;
        - CREATE DATABASE openfga_db    OWNER openfga_user;
        - CREATE EXTENSION IF NOT EXISTS "pg_uuidv7" WITH SCHEMA public;
        - CREATE EXTENSION IF NOT EXISTS "btree_gist";
        - CREATE EXTENSION IF NOT EXISTS "hstore";

  storage:
    size: 50Gi
    storageClass: local-path  # k3s local-path o Hetzner CSI

  backup:
    barmanObjectStore:
      destinationPath: s3://serenidad-backups/postgres/
      endpointURL: https://s3.us-west-004.backblazeb2.com
      s3Credentials:
        accessKeyId:
          name: b2-credentials
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: b2-credentials
          key: SECRET_ACCESS_KEY
      wal:
        compression: gzip
        maxParallel: 2
      data:
        compression: gzip
    retentionPolicy: "30d"

  monitoring:
    enablePodMonitor: true  # Prometheus ServiceMonitor automático
```

### Migrations con golang-migrate como Kubernetes Job

```yaml
# Para cada microservicio, un Job de migración antes del deploy:
apiVersion: batch/v1
kind: Job
metadata:
  name: iam-db-migrate
  namespace: serenidad-core
spec:
  template:
    spec:
      initContainers:
        - name: wait-for-postgres
          image: busybox
          command: ['sh', '-c', 'until nc -z serenidad-pg-rw.serenidad-data.svc.cluster.local 5432; do sleep 2; done']
      containers:
        - name: migrate
          image: ghcr.io/serenidad/iam:latest
          command: ["/app/iam", "migrate"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: serenidad-pg-app
                  key: uri  # CloudNativePG genera este secret automáticamente
      restartPolicy: OnFailure
```

---

## 11. Cambio 6: Backups — pg_dump cron → CloudNativePG WAL + Barman B2

### Estrategia de Backup Mejorada

El pg_dump diario con cron es reemplazado por una estrategia de backup industrial:

**WAL Archiving Continuo:**
- CloudNativePG archiva automáticamente cada WAL segment (generalmente ~16 MB cada 1-5 minutos) a Backblaze B2
- Permite Point-In-Time Recovery (PITR) a cualquier momento, no solo al último dump diario
- Para datos clínicos médicos, PITR es una mejora significativa sobre pg_dump

**Scheduled Base Backups:**
```yaml
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: serenidad-daily-backup
  namespace: serenidad-data
spec:
  schedule: "0 2 * * *"  # 02:00 UTC daily
  backupOwnerReference: self
  cluster:
    name: serenidad-pg
```

**Restore desde B2:**
```bash
# PITR a un punto específico:
kubectl cnpg restore serenidad-pg \
  --backup-name serenidad-daily-backup-2026-04-01 \
  --target-time "2026-04-01T14:30:00Z"
```

**Retención:** CloudNativePG respeta `retentionPolicy: "30d"` para base backups. Los WAL en B2 se retienen 90 días (configurar lifecycle rule en B2 bucket).

---

## 12. Cambio 7: Secretos — .env Files → Kubernetes Secrets + Sealed Secrets

### Por qué Sealed Secrets

Con GitOps (FluxCD), todos los manifests están en Git. Los Kubernetes Secrets normales en base64 no pueden estar en Git (son decodificables). Sealed Secrets (Bitnami/Helm) resuelve esto:

- Un `SealedSecret` está cifrado con la clave pública del cluster
- Solo el controller de Sealed Secrets (corriendo en el cluster) puede descifrarlo
- Los `SealedSecret` YAML pueden commitearse en Git de forma segura

**Flujo:**
```bash
# Instalar kubeseal CLI
# Crear un secret y sellarlo:
kubectl create secret generic kratos-config \
  --from-literal=DSN="postgres://kratos_user:${KRATOS_DB_PASSWORD}@serenidad-pg-rw:5432/kratos_db" \
  --dry-run=client -o yaml | \
  kubeseal --format yaml > infra/k8s/secrets/kratos-config-sealed.yaml

# El SealedSecret se commitea en Git
# FluxCD lo aplica y Sealed Secrets controller lo descifra en el cluster
```

**Secretos que necesitan Sealed Secrets:**
- KRATOS_DB_PASSWORD, IAM_DB_PASSWORD, SCHEDULING_DB_PASSWORD, CLINICAL_DB_PASSWORD, BILLING_DB_PASSWORD, OPENFGA_DB_PASSWORD
- JWT_PRIVATE_KEY (Ed25519 para IAM Domain Service)
- NATS_CREDENTIALS
- B2_ACCESS_KEY_ID, B2_SECRET_ACCESS_KEY
- IZIPAY_SECRET_KEY, STRIPE_API_KEY, CONEKTA_API_KEY
- SMTP_USER, SMTP_PASS (para Kratos email delivery)
- TELEGRAM_BOT_TOKEN (para Alertmanager)

---

## 13. Cambio 8: Deployment — docker compose up → FluxCD GitOps + Helm

### Flujo GitOps con FluxCD v2.x

```
Developer pushes code to main branch
        │
        ▼
GitHub Actions CI:
  1. go test ./...
  2. docker build → ghcr.io/serenidad/iam:sha-<commit>
  3. Update infra/apps/iam-service/values.yaml: image.tag: sha-<commit>
  4. Commit + push values change
        │
        ▼
FluxCD GitRepository detects new commit
        │
        ▼
FluxCD Kustomization reconciles:
  - HelmRelease iam-service with new tag
        │
        ▼
Helm upgrade --install iam-service
        │
        ▼
Kubernetes rolling update:
  - Nuevo pod arranca con nueva imagen
  - Health checks pasan → tráfico migrado
  - Pod viejo termina (zero-downtime deploy)
```

### Estructura del Repositorio GitOps

```
infra/
├── clusters/
│   └── hetzner-prod/
│       ├── talos/
│       │   ├── controlplane.yaml     # Talos machine config (encrypted)
│       │   └── talosconfig           # Talos admin credentials
│       ├── flux-system/
│       │   ├── gotk-components.yaml  # FluxCD bootstrap (auto-generado)
│       │   └── gotk-sync.yaml        # Apunta al repositorio
│       ├── infrastructure.yaml       # Kustomization para infra/
│       └── apps.yaml                 # Kustomization para apps/
│
├── infrastructure/
│   ├── cert-manager/
│   │   ├── helmrelease.yaml          # cert-manager Helm chart
│   │   ├── clusterissuer.yaml        # Let's Encrypt ClusterIssuer
│   │   └── kustomization.yaml
│   ├── traefik/
│   │   ├── helmrelease.yaml          # Traefik v3 Helm chart
│   │   ├── values.yaml               # Traefik config
│   │   └── kustomization.yaml
│   ├── cnpg/
│   │   ├── helmrelease.yaml          # CloudNativePG operator
│   │   ├── cluster.yaml              # Cluster CRD
│   │   ├── scheduled-backup.yaml     # Scheduled backup
│   │   └── kustomization.yaml
│   ├── nats/
│   │   ├── helmrelease.yaml          # NATS Helm chart
│   │   ├── values.yaml               # JetStream config
│   │   └── kustomization.yaml
│   ├── sealed-secrets/
│   │   └── helmrelease.yaml
│   └── monitoring/                   # Fase 3
│       ├── helmrelease.yaml          # kube-prometheus-stack
│       └── values.yaml
│
├── apps/
│   ├── iam-service/
│   │   ├── helmrelease.yaml
│   │   ├── values.yaml               # image.tag: sha-<commit>
│   │   └── kustomization.yaml
│   ├── scheduling-service/
│   ├── clinical-service/
│   ├── billing-service/
│   ├── ory-kratos/
│   │   ├── helmrelease.yaml          # ory/kratos Helm chart
│   │   └── values.yaml
│   └── openfga/
│       ├── helmrelease.yaml          # openfga Helm chart
│       └── values.yaml
│
└── secrets/
    ├── kratos-config-sealed.yaml     # SealedSecret (safe for Git)
    ├── iam-jwt-key-sealed.yaml
    ├── b2-credentials-sealed.yaml
    └── ...
```

### Bootstrap de FluxCD en el Cluster

```bash
# Instalar FluxCD en el cluster Talos
flux bootstrap github \
  --owner=serenidad \
  --repository=serenidad-platform \
  --branch=main \
  --path=./infra/clusters/hetzner-prod \
  --personal
```

A partir de este momento, **todo cambio de infraestructura se hace via Git** — nunca directamente con `kubectl apply`. El cluster se auto-reconcilia con el estado deseado declarado en el repositorio.

---

## 14. Cambio 9: Observabilidad — Docker Compose Services → kube-prometheus-stack

### Helm Chart Consolidado

**Antes:** 5 containers separados en Docker Compose (Prometheus, Loki, Promtail, Tempo, Grafana).

**Después:** Un solo Helm release `kube-prometheus-stack` que incluye:
- Prometheus Operator + Prometheus
- Grafana
- Alertmanager
- node-exporter (métricas del nodo Talos)
- kube-state-metrics (métricas del cluster k8s)

Adicionalmente: Loki + Promtail y Tempo como releases separados.

### ServiceMonitor para auto-discovery

Con CloudNativePG, kube-prometheus-stack, y los Helm charts de los servicios configurados con `enablePodMonitor: true`, **Prometheus auto-descubre y scrape todos los endpoints `/metrics` automáticamente** via CRDs `ServiceMonitor` y `PodMonitor`. No hay configuración manual de `scrape_configs`.

### Ajustes para CX32 (8 GB)

En 8 GB, el full stack de observabilidad cabe con ajustes mínimos:
```yaml
# En kube-prometheus-stack values.yaml:
prometheus:
  prometheusSpec:
    retention: 30d
    retentionSize: "20GB"
    resources:
      limits:
        memory: 512Mi
        cpu: 500m

alertmanager:
  enabled: true  # Incluir desde Fase 3

grafana:
  resources:
    limits:
      memory: 256Mi
```

**Si la presión de memoria es crítica en Fase 1:** Se puede diferir Grafana y Alertmanager hasta Fase 3, usando solo Prometheus + node-exporter para métricas básicas del cluster.

---

## 15. Cambio 10: Escalado Fase 3+ — Docker Swarm → Add k8s Worker Nodes

### Antes (Docker Swarm — plan previo)

Docker Swarm requería:
1. Convertir docker-compose.yml a docker-compose con `deploy:` stacks
2. Inicializar el swarm: `docker swarm init`
3. Cambiar el modelo mental de servicios a stacks Swarm
4. Reconfigurar networking overlay
5. Esta era una **refactorización estructural de operaciones** — violaba R2

### Después (Kubernetes Worker Nodes — nuevo plan)

Agregar capacidad al cluster es idéntico en Fase 1, 2 y 3+:

```bash
# 1. Crear nuevo VPS CX32 en Hetzner

# 2. Generar config del worker
talosctl gen config serenidad-prod https://<CONTROL_PLANE_IP>:6443 \
  --type worker \
  --output ./infra/clusters/hetzner-prod/talos/worker-01.yaml

# 3. Aplicar configuración al nuevo nodo
talosctl apply-config --insecure \
  --nodes <WORKER_IP> \
  --file ./infra/clusters/hetzner-prod/talos/worker-01.yaml

# 4. El worker se une al cluster automáticamente en ~2 minutos
kubectl get nodes  # Muestra el nuevo nodo en estado Ready
```

No hay cambios de código, no hay cambios de Helm values (excepto quizás actualizar `replicas: 2`), no hay migración de datos. El scheduler de Kubernetes distribuye los pods automáticamente.

**En Fase 3+:** Separar el control plane del nodo de datos:
```bash
# Remover el scheduling de pods del control plane
kubectl taint nodes <CONTROL_PLANE_NODE> \
  node-role.kubernetes.io/control-plane:NoSchedule

# Eliminar el patch allowSchedulingOnControlPlanes: true
talosctl apply-config --nodes <CONTROL_PLANE_IP> \
  --file ./patches/no-workers-on-control-plane.yaml
```

---

## 16. Nuevos ADRs (ADR-010 a ADR-014)

### ADR-010: Talos Linux v1.10.x como OS de Producción

**Decisión:** Talos Linux v1.10.x reemplaza Ubuntu 24.04 LTS como sistema operativo del VPS de producción.

**Razón:** Alineación máxima con R1 (Build It Right The First Time) y R4 (Soberanía Tecnológica):
- Inmutabilidad total: el OS no puede derivar de su configuración declarada. No existe drift de configuración.
- Superficie de ataque: 12 binarios únicos en PATH. Ubuntu: miles. Para una plataforma médica con datos sensibles, minimizar la superficie de ataque es obligatorio.
- Gestión declarativa: la misma filosofía que Kubernetes (declarative reconciliation) aplicada al OS mismo. Un archivo YAML controla el OS + el cluster completo.
- Sin SSH: elimina el vector de ataque más común en servidores Linux.
- Upgrade atómico: OS + Kubernetes actualizan juntos como una unidad. Sin divergencia posible entre OS y cluster.
- Hetzner nativo: ISO público oficial disponible desde abril 2025 con qemu-guest-agent.

**Rechazado:** Ubuntu 24.04 LTS (drift de config, SSH como vector, superficie de ataque grande, hardening manual continuo); Flatcar Container Linux (SSH disponible, no full OS immutability); NixOS (complejo, menor ecosistema k8s, curva de aprendizaje mayor).

**Trade-off aceptado:** Sin SSH en operaciones normales. Troubleshooting via `talosctl logs` + `talosctl dmesg`. Para incidentes graves: boot desde ISO recovery, acceso de emergencia via console Hetzner.

### ADR-011: Kubernetes 1.33.x (upstream via Talos) como Orquestador Definitivo desde Fase 1

**Decisión:** Kubernetes 1.33.x, gestionado por Talos, es el orquestador de producción desde el inicio de Fase 1. No hay Docker Compose en producción.

**Razón:** R2 (Inmutabilidad Arquitectónica — escalar sin refactorizaciones estructurales):
- El plan anterior (Docker Compose → Docker Swarm en Fase 3+) introducía dos migraciones estructurales de operations que violan R2.
- Kubernetes es el mismo modelo desde 1 nodo a 1,000. No hay "migración a k8s" futura; el modelo es definitivo desde el día uno.
- Kubernetes es el estándar de orquestación de la industria (CNCF graduated, 10+ años de madurez, millones de clusters globales).

**Clarificación técnica crítica:** "Talos + k3s" es técnicamente imposible. Talos gestiona upstream Kubernetes directamente. k3s es una distribución para OSes de propósito general. La combinación correcta es "Talos + Kubernetes upstream".

**Trade-off aceptado:** Curva de aprendizaje más alta en Fase 1 que Docker Compose. Compensado por: eliminación permanente de la deuda técnica de orquestación y del riesgo de migración futura a k8s.

**Docker Compose:** Se mantiene solo para desarrollo local (dev environment), no aparece en producción.

### ADR-012: cert-manager v1.x + Traefik v3.x como Stack de Ingress y TLS

**Decisión:** cert-manager v1.x (CNCF graduated) + Traefik v3.x como controlador de ingress y gestor de certificados TLS. Reemplaza Caddy v2.9+ standalone.

**Razón:** 
- Caddy Ingress Controller oficial (`caddyserver/ingress`) está marcado como WIP en el repositorio desde 2024. No es apto para producción médica.
- cert-manager es el estándar absoluto de TLS en Kubernetes. ~25M+ instancias. CNCF graduated.
- Traefik v3 implementa `ForwardAuth Middleware` — equivalente funcional exacto del `forward_auth` de Caddy para el IAM Domain Service. La seguridad del modelo IAM no se ve afectada.
- Traefik v3 soporta Gateway API — el estándar futuro de ingress k8s, reemplazando Ingress clásico.
- ingress-nginx está en proceso de deprecación (EOL Q1 2026) en favor de Gateway API.

**Rechazado:** Caddy Ingress Controller (WIP, no production-ready); ingress-nginx (deprecación inminente); Envoy/Istio (overkill para este escala, 1 GB+ RAM adicional).

**Qué NO cambia:** La lógica de autenticación del IAM Domain Service en el endpoint `/internal/validate-token` no cambia. Solo cambia el componente que llama a ese endpoint (Caddy `forward_auth` → Traefik `ForwardAuth Middleware`).

### ADR-013: CloudNativePG v1.x como Operador de PostgreSQL

**Decisión:** CloudNativePG v1.x (CNCF, respaldado por EDB) gestiona el ciclo de vida completo de PostgreSQL. Reemplaza el container `postgres:17-alpine` directo.

**Razón:**
- PostgreSQL en Kubernetes sin operator es inmanejable en producción: backups manuales propensos a fallos, sin self-healing, sin rolling upgrades, sin gestión de certificados TLS internos.
- Para datos de historia clínica médica (inmutables por diseño, regulados), backup WAL continuo con PITR es obligatorio. pg_dump diario permite una ventana de pérdida de hasta 24 horas. WAL continuo reduce esa ventana a minutos.
- CloudNativePG genera automáticamente los Kubernetes Secrets con las connection strings — los microservicios Go los consumen via `envFrom: secretRef` sin hardcodear credenciales.
- El overhead del operator (~100-150 MB) está completamente justificado para producción médica.
- Un solo Cluster CRD gestiona las 6 databases (iam_db, scheduling_db, clinical_db, billing_db, kratos_db, openfga_db) en la misma instancia PostgreSQL — igual que antes, solo la gestión cambia.

**Rechazado:** postgres:17-alpine directo (inmanejable en k8s producción); Zalando postgres-operator (menor madurez que CloudNativePG, menos adopción CNCF); CrunchyData PGO (complejo, mayor overhead).

### ADR-014: FluxCD v2.x como Sistema GitOps de Producción

**Decisión:** FluxCD v2.x (CNCF graduated) gestiona de forma declarativa el estado completo de producción desde Git.

**Razón:** 
- R1 (Build It Right The First Time) aplicado a operations: sin GitOps, la configuración de producción diverge del código en días o semanas. Con FluxCD, Git es la única fuente de verdad.
- Auditoría completa: cada cambio a producción tiene un commit en Git. Quién, qué, cuándo — siempre visible.
- Auto-reconciliation: si alguien aplica un cambio directo con `kubectl apply`, FluxCD lo revierte automáticamente al estado declarado en Git.
- CNCF graduated: el proyecto más maduro de GitOps junto con ArgoCD.

**Rechazado:** ArgoCD (más RAM, UI más rica pero innecesaria para este escala, ~500 MB+ RAM); Helm directo sin GitOps (sin reconciliation loop, sin auditoría); Jenkins/Argo Workflows (overkill para CI/CD de un solo developer team).

---

## 17. ADRs Actualizados (ADR-003, ADR-006)

### ADR-003: PostgreSQL 17 self-hosted (ACTUALIZADO)

**Actualización:** La decisión de usar PostgreSQL 17 self-hosted se mantiene. El mecanismo de deployment cambia:
- **Antes:** `docker run postgres:17-alpine` gestionado via Docker Compose
- **Después:** CloudNativePG operator v1.x + `Cluster` CRD con image `ghcr.io/cloudnative-pg/postgresql:17.x`

Los datos siguen en el mismo PostgreSQL 17.4. La instancia sigue siendo self-hosted en el VPS. No hay migración de datos entre versiones de PostgreSQL. CloudNativePG usa las mismas extensiones (pg_uuidv7, btree_gist, hstore) configuradas en el bootstrap del Cluster CRD.

### ADR-006: Caddy v2.9+ como reverse proxy (ACTUALIZADO → SUSTITUIDO)

**Actualización:** Caddy ya no es el reverse proxy de producción. Reemplazado por cert-manager + Traefik v3.x para el entorno Kubernetes. Ver ADR-012.

**Caddy en desarrollo local:** Caddy puede seguir usándose en el entorno de desarrollo local con Docker Compose para una experiencia de desarrollo más simple. No aparece en ningún manifest de producción.

La motivación original de ADR-006 sigue siendo válida: TLS automático sin certbot y forward_auth integrado son exactamente lo que Traefik v3 + cert-manager ofrecen en el contexto de Kubernetes.

---

## 18. Impacto en Fases: Mapa de Cambios por Fase

### Fase 1 — Cambios Sustanciales en Subcapas

| Subcapa | Antes | Después | Nivel de Cambio |
|---------|-------|---------|----------------|
| 1.A VPS + OS | Ubuntu 24.04 + Docker | **Talos Linux v1.10.x en CX32** | ⭐⭐⭐ Mayor |
| 1.A.x | — | **Talos bootstrap + FluxCD bootstrap** | ⭐⭐⭐ Nuevo |
| 1.B Reverse Proxy | Caddy v2.9+ standalone | **cert-manager + Traefik v3.x** | ⭐⭐⭐ Mayor |
| 1.C PostgreSQL | postgres:17-alpine container | **CloudNativePG v1.x Cluster CRD** | ⭐⭐ Significativo |
| 1.D Protobuf + buf | Sin cambios | Sin cambios | — |
| 1.E Ory Kratos | Docker container | **Helm chart ory/kratos** | ⭐ Menor (mismo SW) |
| 1.F IAM Domain Service | Docker container | **Kubernetes Deployment + Service** | ⭐ Menor (mismo código) |
| 1.G BFF Hono CF Workers | Sin cambios | Sin cambios | — |
| 1.H Qwik SPA CF Pages | Sin cambios | Sin cambios | — |
| 1.I Backups | pg_dump + rclone cron | **CloudNativePG WAL + barman-cloud** | ⭐⭐ Significativo |
| 1.x NUEVO | — | **Sealed Secrets + Secret management** | ⭐⭐ Nuevo |
| 1.x NUEVO | — | **CI/CD: GitHub Actions → ghcr.io** | ⭐ Nuevo |

### Fase 2 — Cambios Menores (Solo Mecanismo de Deploy)

Los microservicios Scheduling, Clinical y NATS son idénticos en código y configuración. Solo cambia cómo se despliegan:
- Docker Compose → Helm charts + Kubernetes Deployments
- Sus Helm charts se añaden al repositorio GitOps bajo `infra/apps/`
- El NATS Helm chart oficial incluye JetStream con las 4 streams configuradas via NACK (NATS Controller for Kubernetes)

### Fase 3 — Simplificado y Mejorado

- OpenFGA: Helm chart oficial `openfga/openfga` — más simple que Docker Compose
- Billing Service: Kubernetes Deployment — mismo código Go
- Observabilidad: kube-prometheus-stack ya incluye ServiceMonitor CRDs para auto-discovery — menos configuración manual que Docker Compose
- Escalado multi-nodo: simplemente añadir worker nodes CX32 — no hay "migración a Docker Swarm"

### Fase 4+ — Sin Cambio de Modelo

Multi-nodo en k8s es trivial vs el cambio de Docker Compose → Swarm → k8s que el plan anterior requería.

---

## 19. Estructura de Namespaces Kubernetes

```
serenidad-core        → IAM Service, Scheduling, Clinical, Billing
                          Ory Kratos, OpenFGA
                          (Workloads de aplicación)

serenidad-data        → CloudNativePG cluster (PostgreSQL)
                          NATS JetStream
                          (Workloads de persistencia y eventos)

serenidad-ops         → kube-prometheus-stack (Prometheus, Grafana, Alertmanager)
                          Loki + Promtail
                          Tempo
                          (Observabilidad — Fase 3)

cert-manager            → cert-manager controller
                          ClusterIssuers (Let's Encrypt prod/staging)

traefik                 → Traefik v3 DaemonSet
                          IngressRoutes, Middlewares

sealed-secrets          → Sealed Secrets controller

cnpg-system             → CloudNativePG operator
                          (El cluster PostgreSQL en serenidad-data)

flux-system             → FluxCD controllers (auto-generado por bootstrap)
```

**NetworkPolicy (Aislamiento):**
```yaml
# Solo serenidad-core puede conectar a serenidad-data
# serenidad-ops puede conectar a todos los namespaces para scrape de métricas
# traefik puede conectar a serenidad-core
# Ningún namespace puede conectar a cert-manager o flux-system
```

---

## 20. Estructura GitOps: Monorepo + FluxCD

Ver sección 13 para la estructura completa de directorios.

**Principio de GitOps:**
1. **Producción = Git** — ningún cambio manual con `kubectl apply` en producción
2. **PRs para cambios de infra** — igual que cambios de código
3. **Secrets en SealedSecrets** — todos los secretos sellados y en Git
4. **FluxCD reconcilia cada 1 minuto** — cualquier drift se corrige automáticamente

---

## 21. Mapa de Helm Charts por Servicio

| Servicio | Helm Repo | Chart | Namespace | Values |
|---------|-----------|-------|-----------|--------|
| cert-manager | `jetstack` | `cert-manager` | `cert-manager` | Estándar |
| Traefik v3 | `traefik` | `traefik` | `traefik` | Customizado |
| Sealed Secrets | `bitnami/sealed-secrets` | `sealed-secrets` | `sealed-secrets` | Estándar |
| CloudNativePG operator | `cnpg` | `cloudnative-pg` | `cnpg-system` | Estándar |
| CloudNativePG cluster | Custom CRD | — | `serenidad-data` | Cluster YAML |
| NATS JetStream | `nats` | `nats` | `serenidad-data` | JetStream enabled |
| Ory Kratos | `ory` | `kratos` | `serenidad-core` | identity schemas |
| OpenFGA | `openfga` | `openfga` | `serenidad-core` | 1 replica |
| IAM Service | Custom | `iam-service` | `serenidad-core` | Go image |
| Scheduling Service | Custom | `scheduling-service` | `serenidad-core` | Go image |
| Clinical Record | Custom | `clinical-service` | `serenidad-core` | Go image |
| Billing & Ops | Custom | `billing-service` | `serenidad-core` | Go image |
| kube-prometheus-stack | `prometheus-community` | `kube-prometheus-stack` | `serenidad-ops` | Ajustado para CX32 |
| Loki | `grafana` | `loki` | `serenidad-ops` | Simple scalable |
| Tempo | `grafana` | `tempo` | `serenidad-ops` | Single binary |
| Hetzner CCM | Custom manifest | — | `kube-system` | hcloud token |

---

## 22. Análisis de Costos: Antes vs Después

### Fase 1

| Componente | Antes | Después | Delta |
|-----------|-------|---------|-------|
| VPS | Hetzner CX31 ~$10/mes | **Hetzner CX32 ~$7.50/mes** | **-$2.50/mes** |
| Cloudflare Pages | $0 | $0 | — |
| Cloudflare Workers | $0 | $0 | — |
| Backblaze B2 | ~$0.05/mes | ~$0.05/mes | — |
| **Total Fase 1** | **~$10.05/mes** | **~$7.55/mes** | **-$2.50/mes** |

### Fase 3 (Full Stack)

| Componente | Antes | Después | Delta |
|-----------|-------|---------|-------|
| VPS | ~$10/mes | **CX32 ~$7.50/mes** | -$2.50 |
| Observabilidad | ~$0 (same VPS) | $0 (same CX32) | — |
| **Total Fase 3** | **~$25-30/mes** | **~$22-27/mes** | **-$3/mes** |

### Fase 4+ (Multi-Nodo)

| Configuración | Antes (Swarm) | Después (k8s) | Delta |
|--------------|--------------|--------------|-------|
| Control plane | CX31 ~$10/mes | CX32 ~$7.50/mes | -$2.50 |
| Worker node 1 | CX31 ~$10/mes | CX32 ~$7.50/mes | -$2.50 |
| Worker node 2 (si aplica) | CX41 ~$16/mes | CX42 ~$14/mes | -$2 |
| **Total 2 nodos** | **~$20/mes** | **~$15/mes** | **-$5/mes** |

**La actualización a Talos + k8s reduce el TCO en todas las fases** debido a que la línea CX32 de Hetzner es más barata que la CX31 anterior con mejores especificaciones.

---

## 23. Riesgos y Mitigaciones del Cambio

| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|-------------|---------|-----------|
| Curva de aprendizaje de Talos (no hay SSH) | Alta | Media | Invertir 1-2 semanas en laboratorio local con Talos antes de producción. Documentar runbooks en Git. |
| OOM en CX32 durante picos (8 GB) | Baja | Alta | Configurar resource limits en todos los pods. Prometheus alertas en >80% RAM. Upgrade a CX42 disponible sin downtime. |
| cert-manager falla al renovar TLS | Muy Baja | Alta | Usar `letsencrypt-staging` en dev, `letsencrypt-prod` en prod. Alertmanager alerta en certificados <30 días de expiración. |
| FluxCD diverge del estado Git | Muy Baja | Baja | FluxCD auto-reconcilia; alertar en FluxCD reconciliation failures. |
| CloudNativePG backup a B2 falla silenciosamente | Baja | Alta | Alertmanager alert en `CnpgBackupFailed`. Verificación semanal manual: `kubectl cnpg status serenidad-pg`. |
| Versión Kubernetes incompatible con chart | Baja | Media | Usar rangos de versión semántica en HelmRelease. Testear upgrades en staging primero. |
| Talos upgrade rompe el cluster | Muy Baja | Alta | Talos tiene upgrade atómico con rollback. Siempre upgrades en ventana de mantenimiento. |
| NATS data loss en pod restart | Muy Baja | Alta | JetStream con `storage: file` persiste en PersistentVolumeClaim. No usa emptyDir. |

---

*Documento generado con investigación Tavily abril 2026. Revisión: Abril 2026.*
