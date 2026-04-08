# Orden de Implementación Exhaustivo por Capas y Subcapas
## Serenidad — Clínica Digital de Salud Mental Global

| Campo | Valor |
|-------|-------|
| **Proyecto** | Serenidad — Clínica Digital de Salud Mental |
| **Versión del documento** | 1.0 |
| **Basado en** | target_arch v3.1 — Stack Definitivo (Abril 2026) |
| **Autor** | djca / Roo Architect Mode |
| **Fecha** | Abril 2026 |
| **Propósito** | Definir el orden de construcción capa por capa, subcapa por subcapa, con máxima granularidad, dependencias explícitas y criterios de aceptación verificables para cada unidad de trabajo. |

---

## Tabla de Contenidos

1. [Principios de Ordenación](#1-principios-de-ordenación)
2. [Mapa de Capas Completo](#2-mapa-de-capas-completo)
3. [Fase 1 — Cimiento del VPS: Infraestructura + IAM + Gateway](#3-fase-1--cimiento-del-vps-infraestructura--iam--gateway)
   - 3.1 [Subcapa 1.A — Infraestructura Base VPS](#31-subcapa-1a--infraestructura-base-vps)
   - 3.2 [Subcapa 1.B — Ingress + TLS: cert-manager v1.x + Traefik v3.x](#32-subcapa-1b--ingress--tls-cert-manager-v1x--traefik-v3x)
   - 3.3 [Subcapa 1.C — Persistencia: CloudNativePG v1.x + PostgreSQL 17.4](#33-subcapa-1c--persistencia-cloudnativepg-v1x--postgresql-174)
   - 3.4 [Subcapa 1.D — Schemas Proto y buf.build (Fundación Serialización)](#34-subcapa-1d--schemas-proto-y-bufbuild-fundación-serialización)
   - 3.5 [Subcapa 1.E — IAM: Ory Kratos v1.3.1 (AuthN)](#35-subcapa-1e--iam-ory-kratos-v131-authn)
   - 3.6 [Subcapa 1.F — IAM: IAM Domain Service en Go 1.25.x](#36-subcapa-1f--iam-iam-domain-service-en-go-125x)
   - 3.7 [Subcapa 1.G — BFF: Cloudflare Workers + Hono v4.x](#37-subcapa-1g--bff-cloudflare-workers--hono-v4x)
   - 3.8 [Subcapa 1.H — Frontend SPA: Qwik v2.0 en Cloudflare Pages](#38-subcapa-1h--frontend-spa-qwik-v20-en-cloudflare-pages)
   - 3.9 [Subcapa 1.I — Backups: pg_dump + rclone + Backblaze B2](#39-subcapa-1i--backups-pgdump--rclone--backblaze-b2)
4. [Fase 2 — Motor Médico: Event Bus + Scheduling + Clinical](#4-fase-2--motor-médico-event-bus--scheduling--clinical)
   - 4.1 [Subcapa 2.A — Event Bus: NATS JetStream v2.11.x](#41-subcapa-2a--event-bus-nats-jetstream-v211x)
   - 4.2 [Subcapa 2.B — Outbox Pattern: Fundación Cross-Service](#42-subcapa-2b--outbox-pattern-fundación-cross-service)
   - 4.3 [Subcapa 2.C — Scheduling Service](#43-subcapa-2c--scheduling-service)
   - 4.4 [Subcapa 2.D — Clinical Record Service](#44-subcapa-2d--clinical-record-service)
   - 4.5 [Subcapa 2.E — Landing Page: Astro v6.x en Cloudflare Pages](#45-subcapa-2e--landing-page-astro-v6x-en-cloudflare-pages)
5. [Fase 3 — Plataforma Completa: Billing + OpenFGA + Observabilidad](#5-fase-3--plataforma-completa-billing--openfga--observabilidad)
   - 5.1 [Subcapa 3.A — IAM: OpenFGA v1.x (AuthZ Zanzibar)](#51-subcapa-3a--iam-openfga-v1x-authz-zanzibar)
   - 5.2 [Subcapa 3.B — Billing & Ops Service](#52-subcapa-3b--billing--ops-service)
   - 5.3 [Subcapa 3.C — Observabilidad: PLG Stack Completo](#53-subcapa-3c--observabilidad-plg-stack-completo)
   - 5.4 [Subcapa 3.D — Alertas: Alertmanager → Telegram](#54-subcapa-3d--alertas-alertmanager--telegram)
6. [Fase 4+ — Evolución Futura](#6-fase-4--evolución-futura)
7. [Grafo de Dependencias Global](#7-grafo-de-dependencias-global)
8. [Matriz de Dependencias por Componente](#8-matriz-de-dependencias-por-componente)
9. [Definición de Hecho por Capa](#9-definición-de-hecho-por-capa)

---

## 1. Principios de Ordenación

Todo el orden de implementación está gobernado por las siguientes reglas de precedencia, derivadas directamente de las restricciones de diseño definidas en [`target_arch/descripcion.md`](../target_arch/descripcion.md):

### 1.1 Regla de Dependencia Fuerte
Un componente **X** debe implementarse antes que **Y** si Y necesita X para compilar, ejecutarse, o validar su correcto funcionamiento en entorno real. No existen dependencias circulares en este orden.

### 1.2 Regla del Valor de Negocio Primero
Dentro de un mismo nivel de dependencia, se prioriza el componente que genera valor de negocio verificable (e.g. un pago procesado, un diagnóstico guardado) sobre el que solo aporta calidad interna (e.g. trazabilidad distribuida).

### 1.3 Regla de Infraestructura Antes que Servicio
Toda infraestructura de soporte (red, TLS, base de datos, broker de eventos) debe estar funcionando y verificada antes de desplegar el código de negocio que la usa.

### 1.4 Regla de Seguridad Antes que Funcionalidad
El stack de autenticación (Kratos) y el mecanismo de firma JWT (IAM Domain Service) deben existir antes de que cualquier endpoint de dominio médico sea expuesto al exterior.

### 1.5 Regla de Observabilidad Antes que Complejidad
Cada microservicio debe tener su endpoint `/health` y `/metrics` antes de ser considerado listo para producción, independientemente de si la capa de observabilidad completa (Grafana/Prometheus) está desplegada.

### 1.6 Regla del Outbox como Ciudadano de Primera Clase
El Outbox Pattern es transversal. Debe diseñarse y verificarse como parte de **cada** microservicio que emita eventos, no como un añadido posterior. Su implementación es parte de la definición de hecho de cada servicio Go.

---

## 2. Mapa de Capas Completo

El sistema se estructura en **9 capas arquitectónicas** con múltiples subcapas cada una. A continuación el inventario completo antes de entrar en el detalle de cada una:

```
CAPA 1: Infraestructura Base (VPS + Talos + cert-manager + CloudNativePG)
  1.A  VPS Hetzner CX32 + Talos Linux v1.10.x + Kubernetes 1.33.x + FluxCD bootstrap
  1.B  cert-manager v1.x + Traefik v3.x + Sealed Secrets (ingress + TLS)
  1.C  CloudNativePG v1.x + PostgreSQL 17.4: 6 databases, WAL → B2
  1.D  (legacy — subsumido por 1.A y 1.C)
  1.E  Extensiones PG: pg_uuidv7, btree_gist, hstore

CAPA 2: Serialización (Protobuf + buf.build)
  2.A  Repositorio de schemas .proto
  2.B  buf.build toolchain (buf.yaml, buf.gen.yaml)
  2.C  Schemas de eventos IAM (proto3)
  2.D  Schemas de eventos Scheduling (proto3)
  2.E  Schemas de eventos Clinical (proto3)
  2.F  Schemas de eventos Billing (proto3)
  2.G  Generación de código Go tipado desde .proto

CAPA 3: IAM — Identity and Access Management
  3.A  Ory Kratos v1.3.1 (AuthN): Helm chart + DB (CloudNativePG) + identity schemas
  3.B  IAM Domain Service (Go 1.25.x): JWT enriquecido + Outbox + DIDs
  3.C  OpenFGA v1.x (AuthZ): Helm chart + DB (CloudNativePG) + authorization model
  3.D  Traefik ForwardAuth Middleware integrado con IAM Domain Service

CAPA 4: API Gateway / BFF
  4.A  Cloudflare Workers: Hono v4.x BFF
  4.B  Verificación JWT EdDSA en el edge
  4.C  Edge routing y caché

CAPA 5: Frontend — Portal Médico y Paciente
  5.A  Qwik v2.0 SPA en Cloudflare Pages (@qwik.dev/qwik)
  5.B  Sistema de diseño: Tailwind v4 + Serenidad Minimal
  5.C  Flujo de autenticación UI (Passkeys / Kratos self-service flows)
  5.D  Portal del Médico: agenda, historia clínica, consulta
  5.E  Portal del Paciente: citas, historial, pagos

CAPA 6: Event Bus
  6.A  NATS JetStream v2.11.x: contenedor Docker + configuración
  6.B  4 Streams permanentes por dominio
  6.C  Outbox Worker transversal (librería Go compartida)

CAPA 7: Microservicios Core — Dominio Médico
  7.A  Scheduling Service (Go 1.25.x)
       7.A.1 DB: scheduling_db con TSTZRANGE + EXCLUDE USING GIST
       7.A.2 Dominio: Appointment, Slot, AvailabilityRule
       7.A.3 API REST: endpoints de agenda y disponibilidad
       7.A.4 FHIR R4 Appointment output
       7.A.5 Outbox: AppointmentBooked, AppointmentCancelled
       7.A.6 Consumidor NATS: DoctorOffboarded
  7.B  Clinical Record Service (Go 1.25.x)
       7.B.1 DB: clinical_db — Event Store puro append-only
       7.B.2 Dominio: EHR, Encounter, ClinicalEvent
       7.B.3 openEHR Canonical JSON: arquetipos y composiciones
       7.B.4 API REST: endpoints de historia clínica
       7.B.5 Row-Level Security: políticas RLS sobre JWT
       7.B.6 Proyección/Read Model: diagnoses_view, encounters_view
       7.B.7 Outbox: DiagnosisRecorded, ConsultationFinished
       7.B.8 Consumidor NATS: AppointmentBooked → iniciar Encounter
  7.C  Billing & Ops Service (Go 1.25.x)
       7.C.1 DB: billing_db — transacciones financieras inmutables
       7.C.2 Dominio: FinancialTransaction, Invoice, PaymentGateway
       7.C.3 Multi-gateway: Izipay (LATAM), Stripe (global), Conekta (México)
       7.C.4 API REST: endpoints de facturación y recibos
       7.C.5 Outbox: PaymentCaptured, InvoiceGenerated
       7.C.6 Consumidor NATS: ConsultationFinished → iniciar cobro

CAPA 8: Observabilidad
  8.A  Prometheus v3.x: métricas de todos los servicios
  8.B  Loki v3.x + Promtail v3.x: logs centralizados
  8.C  Tempo v2.9+: distributed tracing
  8.D  Grafana v11.x: dashboards unificados
  8.E  Alertmanager v0.27+: reglas de alerta → Telegram bot

CAPA 9: Backups y Continuidad
  9.A  CloudNativePG WAL archiving continuo → Backblaze B2 (barman-cloud)
  9.B  ScheduledBackup CRD: snapshot diario 02:00 UTC
  9.C  Política de retención: 30 días base backup, 90 días WAL
  9.D  Verificación semanal de restauración (kubectl cnpg restore --dry-run)

CAPA 10: Landing Page y Marketing
  10.A Astro v6.x + TypeScript 6.0 en Cloudflare Pages
  10.B SEO: Lighthouse 99.2, zero JS por defecto
  10.C Integración con BFF para formulario de contacto
```

---

## 3. Fase 1 — Cimiento del VPS: Infraestructura + IAM + Gateway

> **Objetivo estratégico:** Construir los cimientos permanentes del sistema. Primer paso real del enfoque "Build It Right The First Time".
>
> **Stack definitivo:** Talos Linux v1.10.x + Kubernetes 1.33.x en Hetzner CX32 (8 GB). Ver análisis completo en [`plans/06_talos_k8s_decision_y_cambios_en_cadena.md`](./06_talos_k8s_decision_y_cambios_en_cadena.md).

---

### 3.1 Subcapa 1.A — Infraestructura Base: Talos Linux + Kubernetes en Hetzner CX32

**Prerequisito:** Ninguno dentro de Fase 1. Primera tarea.

**Por qué primero:** Todo lo demás de la Fase 1 corre como pods en Kubernetes. Sin el cluster, nada puede desplegarse.

**Por qué Talos + Kubernetes desde Fase 1 y no Docker Compose:**
Kubernetes desde Fase 1 elimina la deuda técnica de orquestación que existiría con Docker Compose → Docker Swarm → k8s. Agregar workers en Fase 4+ es `talosctl apply-config`, no una migración arquitectónica. Alineación con R1 (Build It Right) y R2 (Inmutabilidad Arquitectónica). Ver ADR-010, ADR-011.

#### 3.1.1 — Aprovisionamiento del VPS Hetzner CX32

1. En Hetzner Cloud Console: crear servidor con la siguiente configuración:
   - **Tipo:** CX32 (4 vCPU AMD, **8 GB RAM**, 80 GB SSD NVMe)
   - **OS:** **NO seleccionar Ubuntu** — seleccionar ISO público de Talos Linux.
     - En "ISO Images (Public)" buscar: `talos` o usar el schematic ID oficial:
       `ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515`
       (incluye qemu-guest-agent requerido por Hetzner)
   - **Datacenter:** FSN1 (Falkenstein) o NBG1 — mejor latencia hacia LATAM vía Frankfurt IXPs
   - **SSH Key:** NO aplica — Talos no tiene SSH. No configurar.
   - **Firewall:** Crear con reglas:
     - Entrada TCP 50000 (Talos API — talosctl) — solo desde IPs del equipo
     - Entrada TCP 6443 (Kubernetes API) — solo desde IPs del equipo
     - Entrada TCP 80 (HTTP) — cualquier origen (ACME challenge cert-manager)
     - Entrada TCP 443 (HTTPS) — cualquier origen
     - Salida: permitir todo

2. Asignar una Floating IP al servidor (DNS estable ante reemplazos del VPS).

3. Crear DNS records en Cloudflare:
   - `api.sereni.dad` → A record → IP del VPS (Cloudflare Proxy: OFF — cert-manager ACME requiere acceso directo)
   - `sereni.dad` → CNAME → Cloudflare Pages (Proxy: ON)
   - `app.sereni.dad` → CNAME → Cloudflare Pages (Proxy: ON)

#### 3.1.2 — Bootstrap de Talos Linux (reemplaza "hardening de Ubuntu")

> **Nota:** En Talos no existe SSH, no existe apt, no existe bash. Todo se gestiona via `talosctl` API con mTLS. El "hardening" es inherente al OS — no es un paso post-instalación.

**En la máquina de desarrollo (no en el servidor):**

1. Instalar `talosctl`:
   ```bash
   # macOS
   brew install siderolabs/tap/talosctl
   # Linux
   curl -sL https://talos.dev/install | sh
   ```

2. Crear el patch de configuración para nodo único:
   ```yaml
   # infra/clusters/hetzner-prod/talos/patches/single-node.yaml
   machine:
     network:
       hostname: serenidad-prod-01
     kubelet:
       extraArgs:
         node-labels: "node-role.kubernetes.io/worker="
   cluster:
     allowSchedulingOnControlPlanes: true  # Single-node: pods en el control plane
     network:
       cni:
         name: flannel  # CNI ligero para single-node
     extraManifests:
       - https://github.com/hetznercloud/hcloud-cloud-controller-manager/releases/latest/download/ccm.yaml
   ```

3. Generar machine config del cluster:
   ```bash
   # Generar secrets del cluster (solo una vez; commitear ENCRYPTED o usar SOPS)
   talosctl gen secrets --output-file infra/clusters/hetzner-prod/talos/secrets.yaml

   # Generar configuración del controlplane
   talosctl gen config serenidad-prod https://<VPS_IP>:6443      --with-secrets infra/clusters/hetzner-prod/talos/secrets.yaml      --config-patch @infra/clusters/hetzner-prod/talos/patches/single-node.yaml      --output-dir infra/clusters/hetzner-prod/talos/

   # Genera: controlplane.yaml, worker.yaml, talosconfig
   ```

4. Aplicar configuración al nodo (el VPS debe estar booteando desde el ISO Talos):
   ```bash
   # Esperar a que el nodo entre en maintenance mode (~2 min desde boot)
   talosctl apply-config --insecure      --nodes <VPS_IP>      --file infra/clusters/hetzner-prod/talos/controlplane.yaml

   # Bootstrap etcd (solo una vez, en el primer nodo)
   talosctl bootstrap --nodes <VPS_IP>      --talosconfig infra/clusters/hetzner-prod/talos/talosconfig
   ```

5. Obtener kubeconfig:
   ```bash
   talosctl kubeconfig      --nodes <VPS_IP>      --talosconfig infra/clusters/hetzner-prod/talos/talosconfig      --output infra/clusters/hetzner-prod/kubeconfig

   # Verificar cluster
   kubectl --kubeconfig infra/clusters/hetzner-prod/kubeconfig get nodes
   # → serenidad-prod-01   Ready   control-plane   ~2m
   ```

#### 3.1.3 — Bootstrap de FluxCD (GitOps Controller)

Una vez el cluster está operativo, instalar FluxCD para que toda la infraestructura posterior se gestione desde Git:

```bash
# Verificar prerequisitos
flux check --pre

# Bootstrap FluxCD apuntando al repositorio
flux bootstrap github   --owner=serenidad   --repository=serenidad-platform   --branch=main   --path=./infra/clusters/hetzner-prod   --personal   --kubeconfig infra/clusters/hetzner-prod/kubeconfig

# FluxCD crea: flux-system namespace + controllers + GitRepository
# A partir de aqui, todo cambio de infra se hace via Git
```

**Artefactos resultantes:**
- `infra/clusters/hetzner-prod/talos/controlplane.yaml`
- `infra/clusters/hetzner-prod/talos/talosconfig`
- `infra/clusters/hetzner-prod/kubeconfig`
- `infra/clusters/hetzner-prod/flux-system/` (auto-generado por FluxCD)
- Namespace `flux-system` con FluxCD controllers corriendo

**Criterio de aceptación:**
- `kubectl get nodes` muestra `serenidad-prod-01 Ready`
- `kubectl get pods -n flux-system` muestra todos los pods de FluxCD en estado `Running`
- `flux get all` muestra el `GitRepository` y `Kustomization` sincronizados
- `talosctl health` sin errores



### 3.2 Subcapa 1.B — Ingress + TLS: cert-manager v1.x + Traefik v3.x

> **ADR-012:** cert-manager v1.x + Traefik v3.x reemplaza Caddy v2.9+ para el entorno Kubernetes. Ver [`plans/06_talos_k8s_decision_y_cambios_en_cadena.md`](./06_talos_k8s_decision_y_cambios_en_cadena.md).

**Prerequisito:** 3.1 completo (cluster Kubernetes operativo, FluxCD bootstrap). DNS `api.sereni.dad` apuntando al VPS (Cloudflare Proxy: OFF — cert-manager ACME requiere acceso HTTP directo al CX32 por el puerto 80).

**Por qué antes de PostgreSQL:** Traefik es la puerta de entrada a todos los servicios. cert-manager necesita tiempo para emitir el certificado Let's Encrypt via ACME HTTP-01 challenge (propagación DNS + validación). Además, el ForwardAuth Middleware de Traefik se configura aquí y se activa cuando el IAM Service esté listo en 3.6.

#### 3.2.1 — HelmRelease: cert-manager

Crear `infra/infrastructure/cert-manager/helmrelease.yaml`:

```yaml
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
        enabled: true
```

Crear `infra/infrastructure/cert-manager/clusterissuer.yaml`:

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

#### 3.2.2 — HelmRelease: Traefik v3.x

Crear `infra/infrastructure/traefik/helmrelease.yaml`:

```yaml
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
      kind: DaemonSet
    service:
      type: LoadBalancer
    ports:
      web:
        port: 80
        hostPort: 80
      websecure:
        port: 443
        hostPort: 443
        tls:
          enabled: true
    additionalArguments:
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
    metrics:
      prometheus:
        enabled: true
        serviceMonitor:
          enabled: true
```

#### 3.2.3 — ForwardAuth Middleware (activar en subcapa 1.F)

Crear `infra/apps/iam-service/forwardauth-middleware.yaml` — este objeto se crea ahora pero el IAM Service que lo sirve se despliega en 3.6:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: iam-forward-auth
  namespace: serenidad-core
spec:
  forwardAuth:
    address: "http://iam-service.serenidad-core.svc.cluster.local:8080/internal/validate-token"
    authResponseHeaders:
      - "X-User-ID"
      - "X-User-Role"
      - "X-Tenant-ID"
    trustForwardHeader: false
```

#### 3.2.4 — IngressRoute: api.sereni.dad (versión final)

Crear `infra/apps/gateway/ingressroute.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: api-serenidad
  namespace: serenidad-core
spec:
  entryPoints:
    - websecure
  routes:
    - match: "Host(`api.sereni.dad`) && PathPrefix(`/auth`)"
      kind: Rule
      services:
        - name: ory-kratos-public
          port: 4433
    - match: "Host(`api.sereni.dad`) && PathPrefix(`/api/iam`)"
      kind: Rule
      middlewares:
        - name: iam-forward-auth
      services:
        - name: iam-service
          port: 8080
    - match: "Host(`api.sereni.dad`) && PathPrefix(`/api/scheduling`)"
      kind: Rule
      middlewares:
        - name: iam-forward-auth
      services:
        - name: scheduling-service
          port: 8081
    - match: "Host(`api.sereni.dad`) && PathPrefix(`/api/clinical`)"
      kind: Rule
      middlewares:
        - name: iam-forward-auth
      services:
        - name: clinical-service
          port: 8082
    - match: "Host(`api.sereni.dad`) && PathPrefix(`/api/billing`)"
      kind: Rule
      middlewares:
        - name: iam-forward-auth
      services:
        - name: billing-service
          port: 8083
  tls:
    certResolver: letsencrypt-prod
    secretName: api-serenidad-tls
```

#### 3.2.5 — Verificación del TLS automático

```bash
# Verificar que cert-manager emitió el certificado
kubectl get certificate -n serenidad-core
kubectl describe certificate api-serenidad-tls -n serenidad-core

# Verificar Traefik está operativo
kubectl get pods -n traefik
kubectl logs -n traefik -l app.kubernetes.io/name=traefik --tail=20

# Verificar que el ClusterIssuer está listo
kubectl get clusterissuer letsencrypt-prod

# Test de TLS (el servicio de destino puede devolver 503 aún — lo que importa es que TLS funciona)
curl -v https://api.sereni.dad/auth/health/ready
```

**Artefactos resultantes:** `infra/infrastructure/cert-manager/` (HelmRelease + ClusterIssuer), `infra/infrastructure/traefik/` (HelmRelease), `infra/apps/iam-service/forwardauth-middleware.yaml`, `infra/apps/gateway/ingressroute.yaml`.

**Criterio de aceptación:** `kubectl get certificate -n serenidad-core` muestra `READY=True`. `curl -I https://api.sereni.dad` retorna respuesta TLS válida (certificado emitido por Let's Encrypt R11). Traefik DaemonSet está Running.

---

### 3.3 Subcapa 1.C — Persistencia: CloudNativePG v1.x + PostgreSQL 17.4

> **ADR-013:** CloudNativePG v1.x (CNCF operator) reemplaza el container `postgres:17-alpine` directo. Ver [`plans/06_talos_k8s_decision_y_cambios_en_cadena.md`](./06_talos_k8s_decision_y_cambios_en_cadena.md).

**Prerequisito:** 3.1 completo (Kubernetes + FluxCD). cnpg-system namespace creado por FluxCD bootstrap. No depende de Traefik para arrancar, pero los servicios que consumen PG sí dependen de que las 6 databases estén disponibles.

**Por qué antes que los servicios:** CloudNativePG es la dependencia de datos de todos los microservicios. Sin el Cluster CRD operativo, ningún servicio puede inicializar sus migraciones via init containers.

#### 3.3.1 — HelmRelease: CloudNativePG Operator

Crear `infra/infrastructure/cnpg/helmrelease.yaml`:

```yaml
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
      podMonitorEnabled: true
```

#### 3.3.2 — Cluster CRD: PostgreSQL 17.4 con 6 databases

Crear `infra/infrastructure/cnpg/cluster.yaml`:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: serenidad-pg
  namespace: serenidad-data
spec:
  instances: 1
  imageName: ghcr.io/cloudnative-pg/postgresql:17.4

  postgresql:
    parameters:
      max_connections: "200"
      shared_buffers: "256MB"
      effective_cache_size: "768MB"
      maintenance_work_mem: "64MB"
      checkpoint_completion_target: "0.9"
      wal_buffers: "16MB"
      default_statistics_target: "100"
    pg_hba:
      - host all all 10.0.0.0/8 scram-sha-256

  # Extensiones habilitadas en todas las databases
  # pg_uuidv7, btree_gist, hstore se instalan via init containers en cada servicio

  storage:
    size: 20Gi
    storageClass: local-path

  # WAL archiving continuo → Backblaze B2 (configurado en 3.I)
  backup:
    retentionPolicy: "30d"
    barmanObjectStore:
      destinationPath: "s3://serenidad-pg-backups/wal"
      endpointURL: "https://s3.us-west-004.backblazeb2.com"
      s3Credentials:
        accessKeyId:
          name: b2-backup-credentials
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: b2-backup-credentials
          key: SECRET_ACCESS_KEY

  monitoring:
    enablePodMonitor: true

  # CloudNativePG auto-genera Secrets con connection strings
  # serenidad-pg-app → connection string para los servicios
  # serenidad-pg-superuser → solo para administración
```

#### 3.3.3 — Inicialización de databases y extensiones

CloudNativePG crea un único cluster PostgreSQL. Las 6 databases se crean via un Job de inicialización que corre una sola vez. Crear `infra/infrastructure/cnpg/init-databases-job.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pg-init-databases
  namespace: serenidad-data
  annotations:
    "helm.sh/hook": post-install
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: pg-init
        image: postgres:17-alpine
        env:
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef:
              name: serenidad-pg-superuser
              key: password
        - name: PGHOST
          value: "serenidad-pg-rw.serenidad-data.svc.cluster.local"
        - name: PGUSER
          value: "postgres"
        command:
        - /bin/sh
        - -c
        - |
          # Crear las 6 databases del sistema
          psql -c "CREATE DATABASE iam_db        ENCODING 'UTF8' LC_COLLATE='en_US.UTF-8' LC_CTYPE='en_US.UTF-8' TEMPLATE template0;" || true
          psql -c "CREATE DATABASE scheduling_db ENCODING 'UTF8' LC_COLLATE='en_US.UTF-8' LC_CTYPE='en_US.UTF-8' TEMPLATE template0;" || true
          psql -c "CREATE DATABASE clinical_db   ENCODING 'UTF8' LC_COLLATE='en_US.UTF-8' LC_CTYPE='en_US.UTF-8' TEMPLATE template0;" || true
          psql -c "CREATE DATABASE billing_db    ENCODING 'UTF8' LC_COLLATE='en_US.UTF-8' LC_CTYPE='en_US.UTF-8' TEMPLATE template0;" || true
          psql -c "CREATE DATABASE kratos_db     ENCODING 'UTF8' LC_COLLATE='en_US.UTF-8' LC_CTYPE='en_US.UTF-8' TEMPLATE template0;" || true
          psql -c "CREATE DATABASE openfga_db    ENCODING 'UTF8' LC_COLLATE='en_US.UTF-8' LC_CTYPE='en_US.UTF-8' TEMPLATE template0;" || true

          # Extensiones por database
          psql -d iam_db        -c "CREATE EXTENSION IF NOT EXISTS pg_uuidv7; CREATE EXTENSION IF NOT EXISTS hstore;"
          psql -d scheduling_db -c "CREATE EXTENSION IF NOT EXISTS pg_uuidv7; CREATE EXTENSION IF NOT EXISTS btree_gist;"
          psql -d clinical_db   -c "CREATE EXTENSION IF NOT EXISTS pg_uuidv7;"
          psql -d billing_db    -c "CREATE EXTENSION IF NOT EXISTS pg_uuidv7;"

          # Revocar acceso público (seguridad)
          psql -d iam_db        -c "REVOKE ALL ON SCHEMA public FROM PUBLIC;"
          psql -d scheduling_db -c "REVOKE ALL ON SCHEMA public FROM PUBLIC;"
          psql -d clinical_db   -c "REVOKE ALL ON SCHEMA public FROM PUBLIC;"
          psql -d billing_db    -c "REVOKE ALL ON SCHEMA public FROM PUBLIC;"

          echo "Databases inicializadas exitosamente."
```

> **Nota sobre connection strings:** CloudNativePG genera automáticamente un Secret `serenidad-pg-app` con el campo `uri` que contiene el connection string completo. Los microservicios Go consumen este Secret via variable de entorno en su Deployment. Las migraciones de cada servicio se ejecutan como init containers usando `golang-migrate` antes de que el contenedor principal arranque.

#### 3.3.4 — Verificación del Cluster CloudNativePG

```bash
# Estado del cluster
kubectl get cluster serenidad-pg -n serenidad-data

# Pods del cluster (primary + replicas en Fase 4+)
kubectl get pods -n serenidad-data -l cnpg.io/cluster=serenidad-pg

# Conexión de prueba via kubectl
kubectl exec -n serenidad-data \
  $(kubectl get pod -n serenidad-data -l cnpg.io/cluster=serenidad-pg,role=primary -o name) \
  -- psql -U postgres -c "\l"

# Verificar extensiones en clinical_db
kubectl exec -n serenidad-data \
  $(kubectl get pod -n serenidad-data -l cnpg.io/cluster=serenidad-pg,role=primary -o name) \
  -- psql -U postgres -d clinical_db -c "\dx"

# Verificar Secret autogenerado con connection string
kubectl get secret serenidad-pg-app -n serenidad-data -o jsonpath='{.data.uri}' | base64 -d
```

**Artefactos resultantes:** `infra/infrastructure/cnpg/` (HelmRelease + Cluster CRD + init Job), 6 databases con extensiones instaladas, Secret `serenidad-pg-app` con connection string auto-generado por CloudNativePG.

**Criterio de aceptación:** `kubectl get cluster serenidad-pg -n serenidad-data` muestra `STATUS=Cluster in healthy state`. Las 6 databases existen. `pg_uuidv7` está instalado en `iam_db`. `btree_gist` está instalado en `scheduling_db`. El Secret `serenidad-pg-app` tiene un `uri` válido con formato `postgresql://app:PASSWORD@serenidad-pg-rw.serenidad-data.svc.cluster.local/postgres`.

---

### 3.4 Subcapa 1.D — Schemas Proto y buf.build (Fundación Serialización)

**Prerequisito:** Estructura de directorios del monorepo (3.1.3 — infra/k8s/ setup). No depende de ningún servicio en ejecución.

**Por qué antes que los microservicios Go:** Los schemas Protobuf son el **contrato** entre servicios. Deben existir y generar código válido antes de que cualquier microservicio Go pueda importar los tipos de eventos. Un cambio en los schemas después de que los servicios están escritos implica regenerar código y recompilar todo.

#### 3.4.1 — Instalación de buf CLI

```bash
# En el VPS y en la máquina local del desarrollador
curl -sSL "https://github.com/bufbuild/buf/releases/latest/download/buf-Linux-x86_64" -o /usr/local/bin/buf
chmod +x /usr/local/bin/buf
buf --version  # Verificar
```

#### 3.4.2 — Configuración: `packages/events/buf.yaml`

```yaml
version: v2
modules:
  - path: proto
deps:
  - buf.build/googleapis/googleapis   # Para google.protobuf.Timestamp
breaking:
  use:
    - FILE                            # Detectar breaking changes de schema
lint:
  use:
    - STANDARD                        # Linting estándar buf.build
```

#### 3.4.3 — Configuración: `packages/events/buf.gen.yaml`

```yaml
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go    # Genera structs Go
    out: gen/go
    opt:
      - paths=source_relative
  - remote: buf.build/grpc/go               # Genera clientes gRPC (para futuro)
    out: gen/go
    opt:
      - paths=source_relative
      - require_unimplemented_servers=false
```

#### 3.4.4 — Schema IAM: `packages/events/proto/iam/v1/events.proto`

```protobuf
syntax = "proto3";
package serenidad.iam.v1;
option go_package = "serenidad/packages/events/gen/go/iam/v1;iamv1";

import "google/protobuf/timestamp.proto";

// Evento emitido cuando un nuevo usuario completa el registro
message UserRegisteredEvent {
  string event_id   = 1;  // UUIDv7 — identificador único del evento
  string user_id    = 2;  // UUIDv7 — ID del usuario en el sistema de dominio
  string kratos_id  = 3;  // UUID de Ory Kratos (identidad original)
  string email      = 4;
  string role       = 5;  // "patient" | "doctor" | "admin"
  string tenant_id  = 6;  // UUIDv7 — organización/clínica
  string did        = 7;  // did:web:sereni.dad:users:{user_id}
  google.protobuf.Timestamp registered_at = 8;
}

// Evento emitido cuando un médico completa su perfil profesional
message DoctorOnboardedEvent {
  string event_id        = 1;  // UUIDv7
  string doctor_id       = 2;  // UUIDv7 (mismo que user_id del UserRegisteredEvent)
  string specialty_snomed = 3; // Código SNOMED-CT de la especialidad
  string license_number  = 4;  // Número de cédula profesional
  string country_iso2    = 5;  // "PE" | "MX" | "CO" | etc.
  google.protobuf.Timestamp onboarded_at = 6;
}

// Evento emitido cuando un médico sale de la plataforma
message DoctorOffboardedEvent {
  string event_id  = 1;  // UUIDv7
  string doctor_id = 2;  // UUIDv7
  string reason    = 3;  // "voluntary" | "disciplinary" | "license_expired"
  google.protobuf.Timestamp offboarded_at = 4;
}
```

#### 3.4.5 — Schema Scheduling: `packages/events/proto/scheduling/v1/events.proto`

```protobuf
syntax = "proto3";
package serenidad.scheduling.v1;
option go_package = "serenidad/packages/events/gen/go/scheduling/v1;schedulingv1";

import "google/protobuf/timestamp.proto";

// FHIR R4 Appointment Reference (para interoperabilidad)
message FHIRAppointmentRef {
  string resource_id   = 1;  // UUID del recurso FHIR
  string resource_type = 2;  // Siempre "Appointment"
  string fhir_json     = 3;  // JSON completo del FHIR R4 Appointment
}

// Evento emitido cuando se reserva una cita exitosamente
message AppointmentBookedEvent {
  string event_id        = 1;  // UUIDv7
  string appointment_id  = 2;  // UUIDv7 (PK del Appointment en scheduling_db)
  string patient_id      = 3;  // UUIDv7
  string doctor_id       = 4;  // UUIDv7
  google.protobuf.Timestamp start_time = 5;
  google.protobuf.Timestamp end_time   = 6;
  string service_snomed_code = 7;      // Código SNOMED-CT del servicio médico
  string timezone_iana       = 8;      // "America/Lima" | "America/Mexico_City"
  FHIRAppointmentRef fhir_ref = 9;
  google.protobuf.Timestamp booked_at = 10;
}

// Evento emitido cuando se cancela una cita
message AppointmentCancelledEvent {
  string event_id        = 1;  // UUIDv7
  string appointment_id  = 2;  // UUIDv7
  string cancelled_by    = 3;  // UUIDv7 del usuario que canceló
  string reason          = 4;  // "patient_request" | "doctor_unavailable" | "payment_timeout"
  google.protobuf.Timestamp cancelled_at = 5;
}
```

#### 3.4.6 — Schema Clinical: `packages/events/proto/clinical/v1/events.proto`

```protobuf
syntax = "proto3";
package serenidad.clinical.v1;
option go_package = "serenidad/packages/events/gen/go/clinical/v1;clinicalv1";

import "google/protobuf/timestamp.proto";

// Composición openEHR Canonical JSON (el corazón de la historia clínica)
message OpenEHRComposition {
  string archetype_id   = 1;  // e.g. "openEHR-EHR-COMPOSITION.encounter.v1"
  bytes  canonical_json = 2;  // openEHR Canonical JSON completo, serializado como bytes
  int32  schema_version = 3;  // Versión del archetype para evolución futura
}

// Evento emitido cuando el médico registra un diagnóstico
message DiagnosisRecordedEvent {
  string event_id     = 1;  // UUIDv7
  string ehr_id       = 2;  // UUIDv7 — ID del EHR del paciente
  string patient_id   = 3;  // UUIDv7
  string doctor_id    = 4;  // UUIDv7
  string encounter_id = 5;  // UUIDv7 — ID de la consulta
  string appointment_id = 6; // UUIDv7 — referencia al Appointment que originó la consulta
  OpenEHRComposition composition = 7;
  string icd11_code   = 8;  // Para indexación rápida sin deserializar openEHR
  string icd11_display = 9; // Texto legible del diagnóstico
  google.protobuf.Timestamp recorded_at = 10;
}

// Evento emitido cuando la consulta médica termina
message ConsultationFinishedEvent {
  string event_id       = 1;  // UUIDv7
  string encounter_id   = 2;  // UUIDv7
  string appointment_id = 3;  // UUIDv7
  string patient_id     = 4;  // UUIDv7
  string doctor_id      = 5;  // UUIDv7
  string service_type   = 6;  // Tipo de servicio para calcular el monto a cobrar
  google.protobuf.Timestamp finished_at = 7;
}
```

#### 3.4.7 — Schema Billing: `packages/events/proto/billing/v1/events.proto`

```protobuf
syntax = "proto3";
package serenidad.billing.v1;
option go_package = "serenidad/packages/events/gen/go/billing/v1;billingv1";

import "google/protobuf/timestamp.proto";

// Evento emitido cuando un pago es capturado exitosamente
message PaymentCapturedEvent {
  string event_id        = 1;  // UUIDv7
  string transaction_id  = 2;  // UUIDv7 (PK de la transacción en billing_db)
  string appointment_id  = 3;  // UUIDv7
  string patient_id      = 4;  // UUIDv7
  string doctor_id       = 5;  // UUIDv7
  int64  amount_cents    = 6;  // Monto en centavos (evitar float para dinero)
  string currency_iso4217 = 7; // "PEN" | "MXN" | "USD" | "COP"
  string gateway         = 8;  // "izipay" | "stripe" | "conekta"
  string gateway_tx_id   = 9;  // ID de transacción del gateway externo
  google.protobuf.Timestamp captured_at = 10;
}

// Evento emitido cuando se genera una factura
message InvoiceGeneratedEvent {
  string event_id       = 1;  // UUIDv7
  string invoice_id     = 2;  // UUIDv7
  string transaction_id = 3;  // UUIDv7
  string patient_id     = 4;  // UUIDv7
  string doctor_id      = 5;  // UUIDv7
  string invoice_number = 6;  // Número de factura (formato por país)
  string tax_country_iso2 = 7; // "PE" | "MX" | "CO"
  google.protobuf.Timestamp issued_at = 8;
}
```

#### 3.4.8 — Generación de código Go

```bash
cd packages/events
buf dep update       # Descargar dependencias (googleapis)
buf generate         # Genera código Go en gen/go/
buf breaking --against '.git#branch=main'  # Detectar breaking changes
```

El código generado en `gen/go/` **no debe editarse manualmente**. Es generado automáticamente y debe estar en el repositorio Git para que los servicios Go puedan importarlo sin necesidad de regenerar en cada build.

**Artefactos resultantes:** `packages/events/` con todos los `.proto`, `buf.yaml`, `buf.gen.yaml`, y el código Go generado en `gen/go/`.

**Criterio de aceptación:** `buf generate` ejecuta sin errores. Los tipos Go están disponibles en `gen/go/iam/v1`, `gen/go/scheduling/v1`, etc. `buf breaking` no reporta breaking changes respecto a la versión anterior (útil en CI).

---

### 3.5 Subcapa 1.E — IAM: Ory Kratos v1.3.1 (AuthN)

**Prerequisito:** 4.3 (PostgreSQL 17.4 con `kratos_db` creada y usuario `kratos_user`).

**Por qué antes que el IAM Domain Service Go:** Kratos es el proveedor de identidad. El IAM Domain Service (Go) llama a la Admin API de Kratos para verificar sesiones. Sin Kratos funcionando, el IAM Service no puede compilar sus dependencias de runtime ni verificar su funcionamiento.

#### 3.5.1 — Identity Schemas

Los esquemas definen qué campos tiene cada tipo de usuario en Kratos.

**`infra/kratos/schemas/patient.json`:**
```json
{
  "$id": "https://api.sereni.dad/schemas/identity/patient.json",
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
```

> El detalle completo de las Subcapas 1.E a 3.D, el grafo de dependencias, la matriz de componentes y la definición de hecho por capa se encuentran en el documento complementario:
> **[`plans/05b_orden_implementacion_capas_detalle.md`](./05b_orden_implementacion_capas_detalle.md)**