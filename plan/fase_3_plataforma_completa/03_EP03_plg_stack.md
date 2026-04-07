# EP-03 — PLG Stack (Prometheus + Loki + Grafana + Tempo)

**Épica**: Como equipo de operaciones, necesito stack completo de observabilidad PLG (Prometheus para métricas, Loki para logs, Grafana para dashboards, Tempo para traces) con storage S3/MinIO, retention policies, y correlación traces-to-logs-to-metrics, para monitorear salud de plataforma y debugging rápido de incidentes healthcare.

**Origen**: Fase 3 (3.C) + research/09_evaluacion_tecnologica_fase3_plataforma_completa.md  
**Prioridad**: Alta  
**Sprint**: S7-S8 (2 semanas)  
**Dependencias Entrantes**: Fase 1 (Kubernetes + storage), Fase 2 (servicios a monitorear)  
**Dependencias Salientes**: EP-04 (Alertmanager usa Prometheus)

---

## HU-03.1 — kube-prometheus-stack Deployment

**Como** SRE, **quiero** desplegar kube-prometheus-stack v82 con Prometheus, Alertmanager, Grafana y exporters, **para que** tengamos métricas time-series de toda la plataforma.

### Tareas

#### T-03.1.1 — Helm Install kube-prometheus-stack

Crear `infrastructure/k8s/monitoring/prometheus-values.yaml`:

```yaml
prometheus:
  prometheusSpec:
    replicas: 2
    retention: 15d
    retentionSize: 90GB
    walCompression: true
    
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: standard
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi
    
    resources:
      requests:
        cpu: 1000m
        memory: 4Gi
      limits:
        cpu: 4000m
        memory: 8Gi
    
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    
    additionalScrapeConfigs: []

alertmanager:
  alertmanagerSpec:
    replicas: 3
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: standard
          resources:
            requests:
              storage: 10Gi
    
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi

grafana:
  enabled: true
  replicas: 2
  
  adminPassword: <USE_SECRET>
  
  persistence:
    enabled: true
    size: 10Gi
    storageClassName: standard
  
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 1Gi
  
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: Prometheus
          type: prometheus
          url: http://prometheus-operated:9090
          isDefault: true
          jsonData:
            timeInterval: 30s
        - name: Loki
          type: loki
          url: http://loki-gateway:3100
          jsonData:
            derivedFields:
              - datasourceUid: tempo
                matcherRegex: "trace_id=(\\w+)"
                name: TraceID
                url: "$${__value.raw}"
        - name: Tempo
          type: tempo
          url: http://tempo-query-frontend:3100
          jsonData:
            tracesToLogs:
              datasourceUid: loki
              tags: ['trace_id']
            serviceMap:
              datasourceUid: prometheus

kubeStateMetrics:
  enabled: true

nodeExporter:
  enabled: true

prometheusOperator:
  resources:
    requests:
      cpu: 200m
      memory: 256Mi
```

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace monitoring

helm install prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values infrastructure/k8s/monitoring/prometheus-values.yaml \
  --version 82.8.0
```

**CA**: Prometheus + Alertmanager + Grafana deployed, pods running

#### T-03.1.2 — PrometheusRules Critical

Crear `infrastructure/k8s/monitoring/prometheusrules.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: serenidad-critical-alerts
  namespace: monitoring
  labels:
    prometheus: kube-prometheus
spec:
  groups:
    - name: serenidad.critical
      interval: 30s
      rules:
        - alert: ServiceDown
          expr: up{namespace="serenidad"} == 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "Service {{ $labels.job }} down"
            description: "{{ $labels.instance }} unreachable for >1m"
        
        - alert: HighErrorRate
          expr: |
            rate(http_requests_total{status=~"5..",namespace="serenidad"}[5m]) > 0.05
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: "High 5xx rate on {{ $labels.service }}"
        
        - alert: PostgreSQLConnectionPoolExhausted
          expr: |
            (pg_stat_database_numbackends / pg_settings_max_connections) > 0.9
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "PostgreSQL connection pool near limit"
        
        - alert: NATSStreamHighLag
          expr: nats_jetstream_stream_lag_total > 1000
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "NATS stream {{ $labels.stream }} lag >1000"
```

**CA**: PrometheusRule creada, Prometheus reload config

### Criterios HU-03.1

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Prometheus scraping targets | All targets UP en Prometheus UI |
| 2 | Grafana accesible | `kubectl port-forward -n monitoring svc/prometheus-stack-grafana 3000:80` |
| 3 | Alerting rules loaded | Prometheus Rules muestra serenidad-critical-alerts |

---

## HU-03.2 — Loki v3 Log Aggregation

**Como** developer, **quiero** Loki v3 agregando logs de todos los pods con retention 30d y S3 storage, **para que** pueda query logs centralizados con LogQL.

### Tareas

#### T-03.2.1 — Loki Helm Install

Crear `infrastructure/k8s/monitoring/loki-values.yaml`:

```yaml
loki:
  commonConfig:
    replication_factor: 3
  
  schemaConfig:
    configs:
      - from: 2026-04-01
        store: tsdb
        object_store: s3
        schema: v13
        index:
          prefix: loki_index_
          period: 24h
  
  storage:
    type: s3
    bucketNames:
      chunks: loki-chunks-serenidad
      ruler: loki-ruler-serenidad
    s3:
      endpoint: s3.amazonaws.com
      region: us-east-1
      s3ForcePathStyle: false
  
  limits_config:
    retention_period: 720h  # 30 días
    max_query_length: 721h
    split_queries_by_interval: 24h
    per_stream_rate_limit: 5MB
    per_stream_rate_limit_burst: 15MB
    max_streams_per_user: 10000

write:
  replicas: 3
  resources:
    requests:
      cpu: 500m
      memory: 2Gi
    limits:
      cpu: 2000m
      memory: 4Gi

read:
  replicas: 3
  resources:
    requests:
      cpu: 500m
      memory: 2Gi

backend:
  replicas: 2
  resources:
    requests:
      cpu: 250m
      memory: 1Gi

gateway:
  enabled: true
  replicas: 2

compactor:
  enabled: true
  replicas: 1

promtail:
  enabled: true
  config:
    clients:
      - url: http://loki-gateway:3100/loki/api/v1/push
    
    positions:
      filename: /tmp/positions.yaml
    
    scrape_configs:
      - job_name: kubernetes-pods
        kubernetes_sd_configs:
          - role: pod
        
        relabel_configs:
          - source_labels: [__meta_kubernetes_namespace]
            action: keep
            regex: serenidad
          - source_labels: [__meta_kubernetes_pod_name]
            target_label: pod
          - source_labels: [__meta_kubernetes_namespace]
            target_label: namespace
          - source_labels: [__meta_kubernetes_pod_container_name]
            target_label: container
          - source_labels: [__meta_kubernetes_pod_label_app]
            target_label: app
          - action: labeldrop
            regex: pod_template_hash
        
        pipeline_stages:
          - multiline:
              firstline: '^\d{4}-\d{2}-\d{2}'
              max_wait_time: 3s
          - json:
              expressions:
                level: level
                msg: message
                trace_id: trace_id
          - labels:
              level:
              trace_id:
```

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install loki grafana/loki \
  --namespace monitoring \
  --values infrastructure/k8s/monitoring/loki-values.yaml \
  --version 3.7.0
```

**CA**: Loki deployed, Promtail DaemonSet running

### Criterios HU-03.2

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Loki receiving logs | LogQL query `{namespace="serenidad"}` retorna logs |
| 2 | Promtail pods all nodes | DaemonSet pod per node |
| 3 | S3 chunks storing | S3 bucket contiene objetos |

---

## HU-03.3 — Tempo v2.9 Distributed Tracing

**Como** developer, **quiero** Tempo v2.9 recibiendo traces OTLP con tail sampling y S3 backend, **para que** pueda debug requests lentos end-to-end.

### Tareas

#### T-03.3.1 — Tempo Helm Install

Crear `infrastructure/k8s/monitoring/tempo-values.yaml`:

```yaml
traces:
  otlp:
    grpc:
      enabled: true
    http:
      enabled: true

distributor:
  replicas: 1
  resources:
    requests:
      cpu: 200m
      memory: 512Mi

ingester:
  replicas: 3
  persistence:
    enabled: true
    size: 50Gi
    storageClass: standard
  resources:
    requests:
      cpu: 500m
      memory: 2Gi

querier:
  replicas: 2
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilizationPercentage: 70

queryFrontend:
  replicas: 2

compactor:
  replicas: 1
  config:
    compaction:
      block_retention: 720h  # 30 días
      compacted_block_retention: 1h

metricsGenerator:
  enabled: true
  replicas: 1
  config:
    storage:
      remote_write:
        - url: http://prometheus-operated:9090/api/v1/write
          send_exemplars: true

storage:
  trace:
    backend: s3
    s3:
      bucket: tempo-traces-serenidad
      endpoint: s3.amazonaws.com
      region: us-east-1

memcached:
  enabled: true
  replicas: 3
```

```bash
helm install tempo grafana/tempo-distributed \
  --namespace monitoring \
  --values infrastructure/k8s/monitoring/tempo-values.yaml \
  --version 1.x
```

**CA**: Tempo deployed, distributor accepting OTLP

#### T-03.3.2 — OpenTelemetry Collector Sidecar

Crear `infrastructure/k8s/otel/collector-config.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: serenidad
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    
    processors:
      batch:
        timeout: 10s
        send_batch_size: 1024
      
      tail_sampling:
        decision_wait: 10s
        policies:
          - name: error-policy
            type: status_code
            status_code:
              status_codes: [ERROR]
          - name: probabilistic
            type: probabilistic
            probabilistic:
              sampling_percentage: 10
          - name: latency-policy
            type: latency
            latency:
              threshold_ms: 1000
    
    exporters:
      otlp:
        endpoint: tempo-distributor.monitoring.svc:4317
        tls:
          insecure: true
    
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [tail_sampling, batch]
          exporters: [otlp]
```

**CA**: OTEL Collector ConfigMap creado

### Criterios HU-03.3

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Tempo accepting traces | OTLP push exitoso |
| 2 | Tail sampling working | Solo 10-20% traces persisted |
| 3 | Grafana Tempo datasource | Search traces funciona |

---

## HU-03.4 — Grafana Dashboards Integration

**Como** SRE, **quiero** dashboards Grafana con traces-to-logs linking y business metrics, **para que** tenga single pane of glass para observabilidad.

### Tareas

#### T-03.4.1 — Dashboard Kubernetes Overview

Importar dashboard ID 15760 (Kubernetes cluster monitoring):

```bash
# Via Grafana UI: Import → 15760
```

**CA**: Dashboard muestra cluster health

#### T-03.4.2 — Dashboard Serenidad Services

Crear `infrastructure/grafana/dashboards/serenidad-services.json`:

```json
{
  "dashboard": {
    "title": "Serenidad Services Overview",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [{
          "expr": "sum(rate(http_requests_total{namespace=\"serenidad\"}[5m])) by (service)"
        }]
      },
      {
        "title": "Error Rate",
        "targets": [{
          "expr": "sum(rate(http_requests_total{status=~\"5..\",namespace=\"serenidad\"}[5m])) by (service)"
        }]
      },
      {
        "title": "P95 Latency",
        "targets": [{
          "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{namespace=\"serenidad\"}[5m])) by (service, le))"
        }]
      }
    ]
  }
}
```

**CA**: Dashboard creado, muestra RED metrics

### Criterios HU-03.4

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Dashboards loaded | Grafana UI muestra dashboards |
| 2 | Traces-to-logs linking | Click trace → muestra logs |
| 3 | All datasources connected | Datasources status green |

---

## Resumen Entregables EP-03

**Stack Completo**:
- Prometheus v2 (métricas time-series, 15d retention, 100Gi PVC)
- Loki v3 (logs aggregation, 30d retention, S3 chunks)
- Tempo v2.9 (traces OTLP, 30d retention, S3 blocks)
- Grafana v11 (dashboards unified, traces-to-logs-to-metrics)
- Alertmanager v0.27 (alert routing, 3 replicas HA)

**Observability Coverage**:
- ServiceMonitor: todos los servicios Go
- Promtail: todos los pods namespace serenidad
- OTLP: tail sampling 10% + 100% errors
- PrometheusRules: 15+ alerting rules

**Dashboards**:
- Kubernetes cluster overview
- Serenidad services RED metrics
- PostgreSQL metrics
- NATS JetStream metrics
- Business metrics (appointments, payments)

**Validación**:
```bash
# 1. Verificar Prometheus targets
kubectl port-forward -n monitoring svc/prometheus-stack-prometheus 9090:9090
# → http://localhost:9090/targets (All UP)

# 2. Query logs Loki
kubectl port-forward -n monitoring svc/loki-gateway 3100:3100
curl -G -s "http://localhost:3100/loki/api/v1/query" \
  --data-urlencode 'query={namespace="serenidad"}' \
  --data-urlencode 'limit=10'

# 3. Search traces Tempo
kubectl port-forward -n monitoring svc/tempo-query-frontend 3100:3100
# Grafana → Explore → Tempo → Search

# 4. Grafana UI
kubectl port-forward -n monitoring svc/prometheus-stack-grafana 3000:80
# → http://localhost:3000 (admin/password)
```

---

**Tiempo estimado EP-03**: 2 semanas (Sprint 7-8)  
**Esfuerzo**: ~70 horas ingeniería + 20 horas dashboards + 10 horas testing
