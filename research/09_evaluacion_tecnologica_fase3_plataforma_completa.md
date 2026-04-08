# Evaluación Tecnológica Fase 3 — Plataforma Completa (Abril 2026)

| Campo | Valor |
|-------|-------|
| **Proyecto** | Serenidad — Clínica Digital de Salud Mental |
| **Versión** | 1.0 |
| **Fecha** | Abril 2026 |
| **Autor** | djca / Cascade AI + Tavily Research Pro |

---

## Resumen Ejecutivo

Validación de tecnologías Fase 3: OpenFGA v1.x (AuthZ), kube-prometheus-stack v82 (métricas), Loki v3 (logs), Tempo v2.9 (traces), Alertmanager→Telegram, Billing multi-gateway.

**Conclusión**: Todas production-ready con Helm charts oficiales y casos healthcare validados.

---

## OpenFGA v1.x — Authorization ReBAC

### Deployment
- **Helm**: `openfga/openfga` chart oficial
- **Replicas**: 3 (HA)
- **Backend**: PostgreSQL (CloudNativePG externo)
- **Resources**: 2 CPU / 4 Gi RAM baseline

### Healthcare Model
```yaml
type patient
  relations
    define owner: [user]
    define care_team: [user]
    define viewer: [user] or care_team

type clinical_record
  relations
    define patient: [patient]
    define viewer: viewer from patient
```

### Performance
- Check queries: <5ms p99
- Config crítico: `OPENFGA_DATASTORE_MAX_OPEN_CONNS=300`

### Recomendaciones
✅ Helm + PostgreSQL externo, caching Redis, integration via middleware Go
⚠️ ListObjects queries costosas, testing exhaustivo model

---

## kube-prometheus-stack v82

### Componentes
Prometheus Operator v0.82, Alertmanager v0.27+, Grafana v11, node-exporter, kube-state-metrics

### Values Producción
```yaml
prometheus:
  prometheusSpec:
    replicas: 2
    retention: 15d
    storage: 100Gi
    walCompression: true
alertmanager:
  replicas: 3
grafana:
  replicas: 2
  persistence: 10Gi
```

### ServiceMonitor Pattern
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: scheduling-service
spec:
  selector:
    matchLabels:
      app: scheduling-service
  endpoints:
    - port: http-metrics
      interval: 30s
```

### Recomendaciones
✅ v82.8.0, WAL compression, ServiceMonitors todos los servicios
⚠️ CRDs gestión manual, PVC sizing crítico

---

## Loki v3 — Log Aggregation

### Deployment
- **Mode**: Microservices (no monolithic)
- **Storage**: S3/MinIO
- **Replicas**: write=3, read=3, backend=2

### Promtail Config
```yaml
scrape_configs:
  - job_name: kubernetes-pods
    relabel_configs:
      - source_labels: [__meta_kubernetes_namespace]
        action: keep
        regex: serenidad
    pipeline_stages:
      - multiline:
          firstline: '^\d{4}-\d{2}-\d{2}'
      - json:
          expressions:
            level: level
            trace_id: trace_id
```

### LogQL Queries
```logql
{namespace="serenidad"} |= "error" | json
sum by (app) (rate({namespace="serenidad"} |= "error" [5m]))
```

### Recomendaciones
✅ Microservices, S3 storage, retention 30d, max 15 labels
⚠️ Cardinality crítica, compactor continuo

---

## Tempo v2.9 — Distributed Tracing

### Architecture
Distributor → Ingester → S3, Query Frontend → Querier

### OTLP Integration
```yaml
traces:
  otlp:
    grpc:
      enabled: true
    http:
      enabled: true
```

### Tail Sampling
```yaml
tail_sampling:
  policies:
    - name: error-policy
      type: status_code
      status_code: [ERROR]
    - name: probabilistic
      probabilistic:
        sampling_percentage: 10
```

### Recomendaciones
✅ OTLP protocol, tail sampling 10-20%, S3 backend, retention 30d
⚠️ Query performance depende S3 latency

---

## Alertmanager → Telegram

### Native Receiver (Recomendado)
```yaml
receivers:
  - name: telegram-critical
    telegram_configs:
      - bot_token_file: /etc/alertmanager/secrets/token
        chat_id: -1001234567890
        parse_mode: HTML
        send_resolved: true
```

### Security
- Bot token en Kubernetes Secret
- RBAC restrictivo
- Templates HTML

### Recomendaciones
✅ Native telegram_configs, grouping alerts, templates HTML
⚠️ Rate limiting Telegram, chat_id integer

---

## Billing Service — Multi-Gateway

### Architecture
```
Client → Billing Service
  ├→ Idempotency (Redis)
  ├→ PSP Façade (Stripe/Conekta/Izipay)
  ├→ Financial Ledger (append-only)
  └→ Outbox (NATS)
```

### PSP Coverage
| PSP | Región | Settlement |
|-----|--------|------------|
| Stripe | Global | T+2 |
| Conekta | México | T+2 |
| Izipay | Perú/Colombia | T+3 |

### Ledger Schema
```sql
CREATE TABLE ledger_entries (
    id uuid PRIMARY KEY,
    account_debit text NOT NULL,
    account_credit text NOT NULL,
    amount numeric(15,2) NOT NULL,
    operation_type text CHECK (operation_type IN ('INSERT', 'REVERSAL')),
    psp_transaction_id text,
    idempotency_key text UNIQUE
);
```

### Recomendaciones
✅ ACL pattern, ledger append-only, idempotency Redis 24h, tokenization
⚠️ PCI-DSS scope (SAQ A/A-EP), LATAM invoicing por país

---

## Roadmap Implementación

**Sprint 7**: OpenFGA + kube-prometheus-stack (2 sem)
**Sprint 8**: Loki + Tempo + Grafana integration (2 sem)
**Sprint 9**: Billing Service + Alertmanager Telegram (2 sem)

**Total Fase 3**: 6 semanas (~1.5 meses)
