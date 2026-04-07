# EP-01 — Event Bus: NATS JetStream Deployment

**Épica:** Como arquitecto de plataforma, necesito desplegar un cluster NATS JetStream production-ready con alta disponibilidad, persistencia, y monitoreo, para que los microservices puedan comunicarse de forma asíncrona con delivery guarantees y audit trail completo.

**Origen:** Fase 2 (2.A) del roadmap arquitectónico + research/08_evaluacion_tecnologica_fase2_motor_medico.md  
**Prioridad:** Crítica — Bloqueante para EP-02, EP-03, EP-04  
**Sprint:** S1  
**Dependencias Entrantes:** Fase 1 completa (PostgreSQL, FluxCD, Kubernetes cluster)  
**Dependencias Salientes:** EP-02 (Outbox Worker), EP-03 (Scheduling), EP-04 (Clinical)

---

## HU-01.1 — Deployment NATS JetStream Cluster

**Como** ingeniero de plataforma,  
**quiero** desplegar un cluster NATS JetStream de 3 nodos con persistencia y HA,  
**para que** los eventos del sistema se puedan publicar y consumir con delivery guarantees (at-least-once) y fault tolerance.

### Contexto técnico

- **Helm chart**: `nats/nats` (oficial CNCF)
- **Topología**: 3-node StatefulSet con headless Service
- **Storage**: File storage con block volumes (gp3/pd-ssd, NO NFS)
- **Resources baseline**: 4 CPU / 8 GiB RAM por pod (production)
- **Anti-affinity**: Spread replicas across nodes/zones

### Tareas y Subtareas

#### T-01.1.1 — Preparar values.yaml para Helm chart

- **ST-01.1.1.1** — Crear archivo `infrastructure/nats/values-production.yaml` en el monorepo.
  - **CA:** Archivo existe con configuración completa.
  
- **ST-01.1.1.2** — Configurar cluster de 3 nodos con JetStream enabled.
  ```yaml
  nats:
    jetstream:
      enabled: true
      fileStorage:
        enabled: true
        size: 100Gi
        storageClassName: fast-ssd
    cluster:
      enabled: true
      replicas: 3
  
  config:
    cluster:
      enabled: true
  ```
  - **CA:** JetStream y clustering habilitados correctamente.

- **ST-01.1.1.3** — Configurar resource requests/limits production-ready.
  ```yaml
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 4
      memory: 8Gi
  ```
  - **CA:** Resources ajustados según baseline de investigación Tavily.

- **ST-01.1.1.4** — Configurar pod anti-affinity para HA.
  ```yaml
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app.kubernetes.io/name
              operator: In
              values:
                - nats
        topologyKey: kubernetes.io/hostname
  ```
  - **CA:** Pods se distribuyen en nodos diferentes (validar con `kubectl get pods -o wide`).

- **ST-01.1.1.5** — Configurar PodDisruptionBudget para preservar quorum.
  ```yaml
  podDisruptionBudget:
    enabled: true
    minAvailable: 2
  ```
  - **CA:** PDB previene disrupciones que rompan quorum (2/3 servers mínimo).

#### T-01.1.2 — Configurar StorageClass y PVCs

- **ST-01.1.2.1** — Crear StorageClass `fast-ssd` si no existe (Hetzner/cloud provider específico).
  - **Hetzner ejemplo**:
  ```yaml
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: fast-ssd
  provisioner: csi.hetzner.cloud
  parameters:
    type: ssd
  allowVolumeExpansion: true
  volumeBindingMode: WaitForFirstConsumer
  ```
  - **CA:** StorageClass existe y soporta `allowVolumeExpansion: true`.

- **ST-01.1.2.2** — Validar que Hetzner CSI driver está instalado (Fase 1).
  - **CA:** `kubectl get csidriver` muestra `csi.hetzner.cloud`.

- **ST-01.1.2.3** — Configurar volumeClaimTemplates en StatefulSet (manejado por Helm).
  - **CA:** Cada pod NATS obtiene PVC de 100 GiB automáticamente.

#### T-01.1.3 — Configurar autenticación y seguridad

- **ST-01.1.3.1** — Generar NKeys para autenticación (usuarios + system accounts).
  ```bash
  nk -gen user > nats-user.nk
  nk -gen account > nats-system.nk
  ```
  - **CA:** NKeys generadas y almacenadas en password manager.

- **ST-01.1.3.2** — Crear Kubernetes Secret con NKeys.
  ```bash
  kubectl create secret generic nats-auth \
    --from-file=user.nk=nats-user.nk \
    --from-file=system.nk=nats-system.nk \
    -n nats-system
  ```
  - **CA:** Secret `nats-auth` existe en namespace `nats-system`.

- **ST-01.1.3.3** — Configurar TLS certificates (cert-manager integration).
  ```yaml
  nats:
    tls:
      enabled: true
      cert: /etc/nats-certs/tls.crt
      key: /etc/nats-certs/tls.key
      ca: /etc/nats-certs/ca.crt
  ```
  - **CA:** TLS habilitado para client y route connections.

- **ST-01.1.3.4** — Crear Certificate CRD para auto-renovación de certs.
  ```yaml
  apiVersion: cert-manager.io/v1
  kind: Certificate
  metadata:
    name: nats-server-tls
    namespace: nats-system
  spec:
    secretName: nats-server-tls
    issuerRef:
      name: letsencrypt-prod
      kind: ClusterIssuer
    dnsNames:
      - nats.nats-system.svc.cluster.local
      - "*.nats-headless.nats-system.svc.cluster.local"
  ```
  - **CA:** Certificate se crea y renueva automáticamente.

#### T-01.1.4 — Deploy vía FluxCD

- **ST-01.1.4.1** — Crear HelmRepository CRD para nats chart.
  ```yaml
  apiVersion: source.toolkit.fluxcd.io/v1beta2
  kind: HelmRepository
  metadata:
    name: nats
    namespace: flux-system
  spec:
    interval: 1h
    url: https://nats-io.github.io/k8s/helm/charts/
  ```
  - **CA:** `flux get sources helm` muestra repositorio `nats` como Ready.

- **ST-01.1.4.2** — Crear HelmRelease CRD.
  ```yaml
  apiVersion: helm.toolkit.fluxcd.io/v2beta1
  kind: HelmRelease
  metadata:
    name: nats
    namespace: nats-system
  spec:
    interval: 10m
    chart:
      spec:
        chart: nats
        version: ">=1.0.0 <2.0.0"
        sourceRef:
          kind: HelmRepository
          name: nats
          namespace: flux-system
    valuesFrom:
      - kind: ConfigMap
        name: nats-values
    install:
      createNamespace: true
    upgrade:
      remediation:
        retries: 3
  ```
  - **CA:** HelmRelease reconcilia sin errores.

- **ST-01.1.4.3** — Commit y push cambios a Git (GitOps).
  - **CA:** FluxCD detecta cambios y aplica configuración automáticamente.

- **ST-01.1.4.4** — Validar deployment exitoso.
  ```bash
  kubectl get pods -n nats-system
  kubectl get statefulset -n nats-system
  flux get helmreleases -n nats-system
  ```
  - **CA:** 3 pods NATS corriendo (1/1 Ready), StatefulSet healthy.

### Criterios de Aceptación de la HU-01.1

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | 3 pods NATS corriendo en nodos distintos | `kubectl get pods -n nats-system -o wide` |
| 2 | Cada pod tiene PVC de 100 GiB | `kubectl get pvc -n nats-system` |
| 3 | Cluster formado correctamente | `kubectl logs nats-0 -n nats-system` (log "Route connection created") |
| 4 | TLS habilitado en todas las conexiones | `nats server list --tlscert=... --tlskey=...` |
| 5 | PodDisruptionBudget activo (minAvailable=2) | `kubectl get pdb -n nats-system` |
| 6 | FluxCD reconcilia HelmRelease sin errores | `flux get helmreleases nats` → Ready |

### Definition of Done — HU-01.1

- [ ] StatefulSet `nats` con 3 replicas running.
- [ ] PVCs provisionados y bound (100 GiB cada uno).
- [ ] TLS certificates válidos y auto-renovables (cert-manager).
- [ ] NKeys configuradas y secret creado.
- [ ] Anti-affinity funcionando (pods en nodos diferentes).
- [ ] PodDisruptionBudget configurado y activo.
- [ ] Cluster routes establecidas (logs confirman).

---

## HU-01.2 — Configurar Streams de JetStream

**Como** desarrollador de microservices,  
**quiero** tener 4 streams JetStream pre-configurados (IAM, Scheduling, Clinical, Billing),  
**para que** cada dominio tenga su canal de eventos aislado con configuración específica de retención y replicas.

### Contexto técnico

Streams configurados según tabla de diseño:

| Stream | Subjects | Retention | Replicas | Max Age | Storage Estimate |
|--------|----------|-----------|----------|---------|------------------|
| `IAM_EVENTS` | `iam.>` | Limits | 3 | 8760h (1 año) | ~10 GiB |
| `SCHEDULING_EVENTS` | `scheduling.>` | Limits | 3 | 17520h (2 años) | ~50 GiB |
| `CLINICAL_EVENTS` | `clinical.>` | Interest | 3 | ∞ (sin límite) | ∞ (archival manual) |
| `BILLING_EVENTS` | `billing.>` | Limits | 3 | 87600h (10 años) | ~100 GiB |

### Tareas y Subtareas

#### T-01.2.1 — Crear stream IAM_EVENTS

- **ST-01.2.1.1** — Crear archivo `infrastructure/nats/streams/iam-events.yaml`.
  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: nats-stream-iam
    namespace: nats-system
  data:
    stream.json: |
      {
        "name": "IAM_EVENTS",
        "subjects": ["iam.>"],
        "retention": "limits",
        "max_age": 31536000000000000,
        "storage": "file",
        "replicas": 3,
        "discard": "old"
      }
  ```
  - **CA:** ConfigMap creado.

- **ST-01.2.1.2** — Crear Job para aplicar stream configuration.
  ```yaml
  apiVersion: batch/v1
  kind: Job
  metadata:
    name: nats-stream-iam-create
    namespace: nats-system
  spec:
    template:
      spec:
        containers:
        - name: nats-stream
          image: natsio/nats-box:latest
          command:
            - nats
            - stream
            - add
            - --config=/config/stream.json
            - --server=nats://nats.nats-system.svc:4222
          volumeMounts:
            - name: config
              mountPath: /config
        volumes:
          - name: config
            configMap:
              name: nats-stream-iam
        restartPolicy: OnFailure
  ```
  - **CA:** Job completa exitosamente.

- **ST-01.2.1.3** — Validar stream creado.
  ```bash
  nats stream info IAM_EVENTS --server=nats://nats.nats-system.svc:4222
  ```
  - **CA:** Stream existe con replicas=3, retention=limits, max_age=1 year.

#### T-01.2.2 — Crear stream SCHEDULING_EVENTS

- **ST-01.2.2.1** — Crear ConfigMap similar a IAM_EVENTS.
  ```json
  {
    "name": "SCHEDULING_EVENTS",
    "subjects": ["scheduling.>"],
    "retention": "limits",
    "max_age": 63072000000000000,
    "storage": "file",
    "replicas": 3
  }
  ```
  - **CA:** ConfigMap creado.

- **ST-01.2.2.2** — Crear Job para aplicar configuración.
  - **CA:** Job completa, stream creado.

- **ST-01.2.2.3** — Validar stream.
  - **CA:** `nats stream info SCHEDULING_EVENTS` → max_age=2 years, replicas=3.

#### T-01.2.3 — Crear stream CLINICAL_EVENTS (special: interest retention)

- **ST-01.2.3.1** — Crear ConfigMap con retention="interest".
  ```json
  {
    "name": "CLINICAL_EVENTS",
    "subjects": ["clinical.>"],
    "retention": "interest",
    "storage": "file",
    "replicas": 3,
    "discard": "new"
  }
  ```
  - **CA:** ConfigMap creado con `retention: "interest"` (eventos permanecen hasta consumo).

- **ST-01.2.3.2** — Aplicar vía Job.
  - **CA:** Job completa.

- **ST-01.2.3.3** — Validar stream.
  - **CA:** Stream tiene `retention: interest`, NO tiene max_age.

- **ST-01.2.3.4** — Documentar archival policy para clinical events.
  - **Runbook**: Eventos clínicos persisten indefinidamente; implementar archival a S3/B2 después de 7 años (compliance HIPAA).
  - **CA:** Documento `docs/runbooks/clinical-events-archival.md` creado.

#### T-01.2.4 — Crear stream BILLING_EVENTS (10 years retention)

- **ST-01.2.4.1** — Crear ConfigMap con max_age=10 years.
  ```json
  {
    "name": "BILLING_EVENTS",
    "subjects": ["billing.>"],
    "retention": "limits",
    "max_age": 315360000000000000,
    "storage": "file",
    "replicas": 3
  }
  ```
  - **CA:** ConfigMap creado.

- **ST-01.2.4.2** — Aplicar vía Job.
  - **CA:** Job completa.

- **ST-01.2.4.3** — Validar stream.
  - **CA:** `nats stream info BILLING_EVENTS` → max_age=10 years.

### Criterios de Aceptación de la HU-01.2

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | 4 streams creados | `nats stream list` muestra IAM, SCHEDULING, CLINICAL, BILLING |
| 2 | Replicas=3 en todos los streams | `nats stream info <NAME>` → replicas: 3 |
| 3 | CLINICAL_EVENTS usa retention="interest" | `nats stream info CLINICAL_EVENTS` → retention: interest |
| 4 | Cada stream tiene subjects correctos | Info muestra subjects patterns (iam.>, scheduling.>, etc.) |
| 5 | Storage type="file" en todos | Info muestra storage: file |

### Definition of Done — HU-01.2

- [ ] 4 streams creados y configurados correctamente.
- [ ] ConfigMaps + Jobs commiteados en Git (GitOps).
- [ ] Validación manual con `nats stream info` exitosa.
- [ ] Runbook de archival para CLINICAL_EVENTS documentado.

---

## HU-01.3 — Monitoring y Alertas Prometheus

**Como** SRE,  
**quiero** tener métricas de NATS JetStream expuestas a Prometheus con dashboards y alertas críticas,  
**para que** pueda detectar problemas de storage, lag de consumers, y nodos down antes de que afecten producción.

### Contexto técnico

- **Metrics port**: 7777 (NATS monitoring HTTP)
- **Exporter**: nativo de NATS (no requiere sidecar)
- **Grafana dashboard**: ID 14725 (NATS JetStream official)
- **Alertas críticas**: Storage high, consumer lag, instance down

### Tareas y Subtareas

#### T-01.3.1 — Configurar ServiceMonitor para Prometheus

- **ST-01.3.1.1** — Habilitar metrics endpoint en NATS Helm values.
  ```yaml
  nats:
    exporter:
      enabled: true
      port: 7777
  ```
  - **CA:** Port 7777 expuesto en pods NATS.

- **ST-01.3.1.2** — Crear ServiceMonitor CRD.
  ```yaml
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    name: nats-metrics
    namespace: nats-system
  spec:
    selector:
      matchLabels:
        app.kubernetes.io/name: nats
    endpoints:
      - port: metrics
        path: /metrics
        interval: 30s
  ```
  - **CA:** Prometheus scrapes métricas NATS cada 30s.

- **ST-01.3.1.3** — Validar scraping en Prometheus UI.
  ```promql
  up{job="nats-metrics"}
  ```
  - **CA:** Query retorna `1` para 3 targets (3 pods NATS).

#### T-01.3.2 — Importar Grafana dashboard oficial NATS

- **ST-01.3.2.1** — Descargar dashboard JSON (ID 14725).
  ```bash
  curl -o nats-jetstream-dashboard.json \
    https://grafana.com/api/dashboards/14725/revisions/latest/download
  ```
  - **CA:** JSON descargado.

- **ST-01.3.2.2** — Crear ConfigMap con dashboard.
  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: grafana-dashboard-nats
    namespace: monitoring
    labels:
      grafana_dashboard: "1"
  data:
    nats-jetstream.json: |
      <DASHBOARD_JSON_CONTENT>
  ```
  - **CA:** ConfigMap creado (Grafana sidecar lo importa automáticamente).

- **ST-01.3.2.3** — Validar dashboard visible en Grafana.
  - **CA:** Dashboard "NATS JetStream" aparece en folder "NATS".

#### T-01.3.3 — Configurar PrometheusRule con alertas críticas

- **ST-01.3.3.1** — Crear PrometheusRule CRD.
  ```yaml
  apiVersion: monitoring.coreos.com/v1
  kind: PrometheusRule
  metadata:
    name: nats-alerts
    namespace: nats-system
  spec:
    groups:
      - name: nats.jetstream
        interval: 30s
        rules:
          - alert: NATSJetStreamStorageHigh
            expr: |
              (gnatsd_jetstream_storage_used / gnatsd_jetstream_max_storage) * 100 > 80
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "NATS JetStream storage > 80%"
              description: "Stream {{ $labels.stream }} storage at {{ $value }}%"
          
          - alert: NATSConsumerLagHigh
            expr: |
              sum(gnatsd_jetstream_consumer_num_pending) by (stream, consumer) > 10000
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "Consumer lag > 10k messages"
              description: "Consumer {{ $labels.consumer }} lag: {{ $value }}"
          
          - alert: NATSInstanceDown
            expr: |
              up{job="nats-metrics"} == 0
            for: 1m
            labels:
              severity: critical
            annotations:
              summary: "NATS instance down"
              description: "Instance {{ $labels.instance }} unreachable"
  ```
  - **CA:** PrometheusRule creado.

- **ST-01.3.3.2** — Validar rules cargadas en Prometheus.
  - **CA:** Prometheus UI → Alerts muestra 3 reglas NATS.

- **ST-01.3.3.3** — Simular alerta de storage high (opcional, testing).
  ```bash
  # Publicar mensajes hasta > 80% storage
  nats pub test.large "$(dd if=/dev/zero bs=1M count=10)" --count=1000
  ```
  - **CA:** Alerta `NATSJetStreamStorageHigh` se dispara en Prometheus UI.

#### T-01.3.4 — Documentar runbook de troubleshooting

- **ST-01.3.4.1** — Crear `docs/runbooks/nats-troubleshooting.md`.
  - Contenido:
    - Cómo verificar cluster health: `nats server list`
    - Cómo revisar stream info: `nats stream info <NAME>`
    - Cómo purgar mensajes viejos: `nats stream purge <NAME>`
    - Cómo expandir PVCs: `kubectl edit pvc <PVC_NAME>`
    - Cómo hacer backup de stream: `nats stream backup <NAME>`
  - **CA:** Runbook documentado.

### Criterios de Aceptación de la HU-01.3

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Prometheus scrapes métricas NATS | Query `up{job="nats-metrics"}` → 3 targets |
| 2 | Grafana dashboard importado | Dashboard visible en UI |
| 3 | 3 alertas configuradas | Prometheus Alerts muestra rules NATS |
| 4 | Alerta de storage funciona | Simulación dispara alerta correctamente |
| 5 | Runbook documentado | Archivo `docs/runbooks/nats-troubleshooting.md` existe |

### Definition of Done — HU-01.3

- [ ] ServiceMonitor creado y Prometheus scraping activo.
- [ ] Dashboard Grafana 14725 importado y funcional.
- [ ] PrometheusRule con 3 alertas críticas configurado.
- [ ] Alertas visibles en Prometheus UI.
- [ ] Runbook de troubleshooting documentado y versionado en Git.

---

## Resumen de Entregables EP-01

**Infraestructura**:
- 3-node NATS JetStream cluster (StatefulSet)
- 3 PVCs de 100 GiB cada uno (fast-ssd)
- TLS certificates (cert-manager)
- NKeys authentication

**Streams**:
- `IAM_EVENTS` (1 year retention)
- `SCHEDULING_EVENTS` (2 years retention)
- `CLINICAL_EVENTS` (interest retention, ∞ age)
- `BILLING_EVENTS` (10 years retention)

**Observabilidad**:
- Prometheus ServiceMonitor
- Grafana dashboard 14725
- 3 PrometheusRules (storage, lag, instance down)
- Runbook de troubleshooting

**Validación end-to-end**:
```bash
# 1. Verificar cluster health
nats server list --server=nats://nats.nats-system.svc:4222

# 2. Listar streams
nats stream list

# 3. Publicar mensaje de prueba
nats pub iam.user.created '{"user_id":"123"}' --server=nats://nats.nats-system.svc:4222

# 4. Consumir mensaje
nats sub "iam.>" --server=nats://nats.nats-system.svc:4222

# 5. Verificar métricas
curl http://nats-0.nats-headless.nats-system.svc:7777/metrics
```

---

## Riesgos y Mitigaciones EP-01

| Riesgo | Impacto | Probabilidad | Mitigación |
|--------|---------|--------------|------------|
| Quorum loss (2/3 pods down) | Crítico | Baja | PodDisruptionBudget minAvailable=2 |
| Storage full | Alto | Media | Alertas Prometheus + auto-expand PVCs |
| Certificate expiration | Medio | Baja | cert-manager auto-renewal |
| Backup strategy undefined | Alto | Alta | Documentar `nats stream backup` en runbook |

---

**Tiempo estimado EP-01**: 2 semanas (Sprint 1)  
**Esfuerzo**: ~40 horas ingeniería + 10 horas testing/validación
