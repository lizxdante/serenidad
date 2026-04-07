# EP-02 — Outbox Worker Transversal

**Épica:** Como desarrollador de microservices, necesito un mecanismo de Outbox pattern transaccional con relay worker que publique eventos a NATS de forma confiable (at-least-once), para garantizar que los eventos del sistema se propaguen correctamente incluso ante fallos temporales de red o NATS.

**Origen:** Fase 2 (2.B) del roadmap arquitectónico + research/08_evaluacion_tecnologica_fase2_motor_medico.md  
**Prioridad:** Crítica — Bloqueante para EP-03, EP-04  
**Sprint:** S2  
**Dependencias Entrantes:** EP-01 (NATS JetStream funcionando)  
**Dependencias Salientes:** EP-03 (Scheduling Service), EP-04 (Clinical Record Service)

---

## HU-02.1 — Schema Outbox y Deduplication

**Como** desarrollador de microservice,  
**quiero** tener una tabla `outbox` en PostgreSQL con indexes optimizados y una tabla de deduplicación,  
**para que** pueda insertar eventos en la misma transacción que mis cambios de negocio y el relay worker pueda procesarlos eficientemente.

### Contexto técnico

**Schema design**:
- Tabla `outbox`: almacena eventos pendientes de publicación
- Tabla `message_dedup`: deduplicación consumer-side
- Index parcial en `outbox(created_at) WHERE published_at IS NULL`
- Pattern: SELECT FOR UPDATE SKIP LOCKED para concurrency

### Tareas y Subtareas

#### T-02.1.1 — Crear migration para tabla outbox

- **ST-02.1.1.1** — Crear archivo `services/shared/migrations/002_create_outbox.up.sql`.
  ```sql
  CREATE TABLE IF NOT EXISTS outbox (
      id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
      aggregate_type text NOT NULL,
      aggregate_id uuid NOT NULL,
      event_type text NOT NULL,
      payload jsonb NOT NULL,
      created_at timestamptz NOT NULL DEFAULT now(),
      published_at timestamptz,
      retry_count int NOT NULL DEFAULT 0,
      trace_context text,
      failure_reason text
  );

  -- Partial index for efficient unpublished selection
  CREATE INDEX idx_outbox_unpublished 
      ON outbox(created_at) 
      WHERE published_at IS NULL;

  -- Index for cleanup queries
  CREATE INDEX idx_outbox_published 
      ON outbox(published_at) 
      WHERE published_at IS NOT NULL;

  COMMENT ON TABLE outbox IS 'Transactional outbox for reliable event publishing';
  COMMENT ON COLUMN outbox.trace_context IS 'OpenTelemetry trace context for distributed tracing';
  ```
  - **CA:** Migration file creado.

- **ST-02.1.1.2** — Crear migration down correspondiente.
  ```sql
  DROP INDEX IF EXISTS idx_outbox_published;
  DROP INDEX IF EXISTS idx_outbox_unpublished;
  DROP TABLE IF EXISTS outbox;
  ```
  - **CA:** Down migration creado.

- **ST-02.1.1.3** — Ejecutar migration con golang-migrate.
  ```bash
  migrate -path services/shared/migrations \
          -database "postgres://user:pass@localhost/serenidad_db" \
          up
  ```
  - **CA:** Tabla `outbox` existe en base de datos.

#### T-02.1.2 — Crear tabla de deduplicación

- **ST-02.1.2.1** — Crear migration `003_create_message_dedup.up.sql`.
  ```sql
  CREATE TABLE IF NOT EXISTS message_dedup (
      message_id uuid PRIMARY KEY,
      processed_at timestamptz NOT NULL DEFAULT now()
  );

  -- TTL cleanup index (para borrar mensajes > 7 días)
  CREATE INDEX idx_dedup_ttl 
      ON message_dedup(processed_at)
      WHERE processed_at > now() - interval '7 days';

  COMMENT ON TABLE message_dedup IS 'Consumer-side idempotency deduplication';
  ```
  - **CA:** Migration creado.

- **ST-02.1.2.2** — Ejecutar migration.
  - **CA:** Tabla `message_dedup` existe.

#### T-02.1.3 — Crear tabla Dead Letter Queue (DLQ)

- **ST-02.1.3.1** — Crear migration `004_create_outbox_dlq.up.sql`.
  ```sql
  CREATE TABLE IF NOT EXISTS outbox_dlq (
      id uuid PRIMARY KEY,
      aggregate_type text NOT NULL,
      aggregate_id uuid NOT NULL,
      event_type text NOT NULL,
      payload jsonb NOT NULL,
      original_created_at timestamptz NOT NULL,
      failed_at timestamptz NOT NULL DEFAULT now(),
      retry_count int NOT NULL,
      failure_reason text NOT NULL
  );

  CREATE INDEX idx_dlq_failed_at ON outbox_dlq(failed_at);
  CREATE INDEX idx_dlq_aggregate ON outbox_dlq(aggregate_id);
  ```
  - **CA:** DLQ table creado.

#### T-02.1.4 — Crear helper functions PostgreSQL (opcional)

- **ST-02.1.4.1** — Crear function para LISTEN/NOTIFY trigger.
  ```sql
  CREATE OR REPLACE FUNCTION notify_outbox_insert()
  RETURNS trigger AS $$
  BEGIN
      PERFORM pg_notify('outbox_events', '');
      RETURN NEW;
  END;
  $$ LANGUAGE plpgsql;

  CREATE TRIGGER outbox_insert_trigger
  AFTER INSERT ON outbox
  FOR EACH ROW EXECUTE FUNCTION notify_outbox_insert();
  ```
  - **CA:** Trigger creado (optimization para relay worker).

### Criterios de Aceptación de la HU-02.1

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Tabla `outbox` existe con schema correcto | `\d outbox` en psql |
| 2 | Index parcial en unpublished creado | `\di` muestra `idx_outbox_unpublished` |
| 3 | Tabla `message_dedup` existe | `\d message_dedup` |
| 4 | Tabla `outbox_dlq` existe | `\d outbox_dlq` |
| 5 | Trigger LISTEN/NOTIFY configurado | Insert en `outbox` dispara notificación |

### Definition of Done — HU-02.1

- [ ] 3 migrations ejecutadas exitosamente.
- [ ] Indexes creados y optimizados.
- [ ] Trigger LISTEN/NOTIFY funcional.
- [ ] Schema documentado en README.md del monorepo.

---

## HU-02.2 — Relay Worker (Outbox Publisher)

**Como** plataforma de microservices,  
**quiero** un worker Go que procese la tabla outbox y publique eventos a NATS con retry logic y DLQ,  
**para que** los eventos escritos en transacciones se publiquen eventualmente con at-least-once guarantees.

### Contexto técnico

**Worker design**:
- Polling loop cada 500ms (con LISTEN/NOTIFY optimization)
- Batch size: 100 eventos por iteración
- Pattern: SELECT FOR UPDATE SKIP LOCKED
- Retry strategy: exponential backoff (max 5 retries)
- DLQ: mover a `outbox_dlq` después de 5 fallos

### Tareas y Subtareas

#### T-02.2.1 — Scaffold Go project para outbox worker

- **ST-02.2.1.1** — Crear directorio `services/outbox-worker/`.
  ```bash
  mkdir -p services/outbox-worker/{cmd,internal/{worker,publisher,metrics}}
  ```
  - **CA:** Estructura de directorios creada.

- **ST-02.2.1.2** — Crear `go.mod`.
  ```bash
  cd services/outbox-worker
  go mod init github.com/serenidad/outbox-worker
  ```
  - **CA:** `go.mod` creado.

- **ST-02.2.1.3** — Agregar dependencias.
  ```bash
  go get github.com/jackc/pgx/v5/pgxpool
  go get github.com/nats-io/nats.go
  go get github.com/prometheus/client_golang/prometheus
  ```
  - **CA:** Dependencias descargadas.

#### T-02.2.2 — Implementar dispatcher (polling loop)

- **ST-02.2.2.1** — Crear `internal/worker/dispatcher.go`.
  ```go
  package worker

  import (
      "context"
      "time"
      "github.com/jackc/pgx/v5/pgxpool"
      "github.com/nats-io/nats.go"
  )

  type Dispatcher struct {
      pool      *pgxpool.Pool
      natsConn  *nats.Conn
      batchSize int
      interval  time.Duration
  }

  func NewDispatcher(pool *pgxpool.Pool, nc *nats.Conn, batchSize int) *Dispatcher {
      return &Dispatcher{
          pool:      pool,
          natsConn:  nc,
          batchSize: batchSize,
          interval:  500 * time.Millisecond,
      }
  }

  func (d *Dispatcher) Run(ctx context.Context) error {
      ticker := time.NewTicker(d.interval)
      defer ticker.Stop()

      for {
          select {
          case <-ctx.Done():
              return ctx.Err()
          case <-ticker.C:
              if err := d.processBatch(ctx); err != nil {
                  // Log error but continue
                  log.Printf("batch processing error: %v", err)
              }
          }
      }
  }
  ```
  - **CA:** Dispatcher implementado.

- **ST-02.2.2.2** — Implementar `processBatch` con SKIP LOCKED.
  ```go
  func (d *Dispatcher) processBatch(ctx context.Context) error {
      tx, err := d.pool.Begin(ctx)
      if err != nil {
          return err
      }
      defer tx.Rollback(ctx)

      rows, err := tx.Query(ctx, `
          SELECT id, aggregate_type, aggregate_id, event_type, payload, trace_context
          FROM outbox
          WHERE published_at IS NULL
          ORDER BY created_at
          LIMIT $1
          FOR UPDATE SKIP LOCKED
      `, d.batchSize)
      if err != nil {
          return err
      }
      defer rows.Close()

      var events []OutboxEvent
      for rows.Next() {
          var e OutboxEvent
          if err := rows.Scan(&e.ID, &e.AggType, &e.AggID, &e.EventType, &e.Payload, &e.TraceCtx); err != nil {
              return err
          }
          events = append(events, e)
      }

      tx.Commit(ctx)

      // Publish outside transaction
      for _, evt := range events {
          d.publishEvent(ctx, evt)
      }

      return nil
  }
  ```
  - **CA:** Batch processing con SKIP LOCKED implementado.

#### T-02.2.3 — Implementar publisher con retry logic

- **ST-02.2.3.1** — Crear `internal/publisher/nats.go`.
  ```go
  package publisher

  import (
      "context"
      "encoding/json"
      "fmt"
      "github.com/nats-io/nats.go"
      "github.com/jackc/pgx/v5/pgxpool"
  )

  type NATSPublisher struct {
      nc   *nats.Conn
      pool *pgxpool.Pool
  }

  func (p *NATSPublisher) Publish(ctx context.Context, evt OutboxEvent) error {
      subject := fmt.Sprintf("%s.%s", evt.AggType, evt.EventType)
      
      // Publish to NATS
      if err := p.nc.Publish(subject, evt.Payload); err != nil {
          return p.handleRetry(ctx, evt, err)
      }

      // Mark as published
      return p.markPublished(ctx, evt.ID)
  }

  func (p *NATSPublisher) markPublished(ctx context.Context, id uuid.UUID) error {
      _, err := p.pool.Exec(ctx, `
          UPDATE outbox
          SET published_at = now()
          WHERE id = $1
      `, id)
      return err
  }
  ```
  - **CA:** Publisher implementado.

- **ST-02.2.3.2** — Implementar exponential backoff retry.
  ```go
  func (p *NATSPublisher) handleRetry(ctx context.Context, evt OutboxEvent, publishErr error) error {
      _, err := p.pool.Exec(ctx, `
          UPDATE outbox
          SET retry_count = retry_count + 1,
              failure_reason = $2
          WHERE id = $1
      `, evt.ID, publishErr.Error())

      if err != nil {
          return err
      }

      // Check if should move to DLQ
      var retryCount int
      p.pool.QueryRow(ctx, `SELECT retry_count FROM outbox WHERE id = $1`, evt.ID).Scan(&retryCount)

      if retryCount >= 5 {
          return p.moveToDLQ(ctx, evt, publishErr.Error())
      }

      return nil
  }

  func (p *NATSPublisher) moveToDLQ(ctx context.Context, evt OutboxEvent, reason string) error {
      tx, _ := p.pool.Begin(ctx)
      defer tx.Rollback(ctx)

      // Insert into DLQ
      _, err := tx.Exec(ctx, `
          INSERT INTO outbox_dlq (id, aggregate_type, aggregate_id, event_type, payload, 
                                  original_created_at, retry_count, failure_reason)
          SELECT id, aggregate_type, aggregate_id, event_type, payload, 
                 created_at, retry_count, $2
          FROM outbox WHERE id = $1
      `, evt.ID, reason)

      if err != nil {
          return err
      }

      // Delete from outbox
      _, err = tx.Exec(ctx, `DELETE FROM outbox WHERE id = $1`, evt.ID)
      if err != nil {
          return err
      }

      return tx.Commit(ctx)
  }
  ```
  - **CA:** Retry + DLQ implementado.

#### T-02.2.4 — Implementar métricas Prometheus

- **ST-02.2.4.1** — Crear `internal/metrics/prometheus.go`.
  ```go
  package metrics

  import "github.com/prometheus/client_golang/prometheus"

  var (
      EventsPublished = prometheus.NewCounterVec(
          prometheus.CounterOpts{
              Name: "outbox_events_published_total",
              Help: "Total events successfully published",
          },
          []string{"aggregate_type", "event_type"},
      )

      EventsFailed = prometheus.NewCounterVec(
          prometheus.CounterOpts{
              Name: "outbox_events_failed_total",
              Help: "Total events failed to publish",
          },
          []string{"aggregate_type", "event_type"},
      )

      EventsDLQ = prometheus.NewCounter(
          prometheus.CounterOpts{
              Name: "outbox_events_dlq_total",
              Help: "Total events moved to DLQ",
          },
      )

      BatchProcessingDuration = prometheus.NewHistogram(
          prometheus.HistogramOpts{
              Name:    "outbox_batch_processing_seconds",
              Help:    "Batch processing duration",
              Buckets: prometheus.DefBuckets,
          },
      )
  )

  func init() {
      prometheus.MustRegister(EventsPublished, EventsFailed, EventsDLQ, BatchProcessingDuration)
  }
  ```
  - **CA:** Métricas Prometheus definidas.

- **ST-02.2.4.2** — Agregar instrumentación en publisher.
  ```go
  func (p *NATSPublisher) Publish(ctx context.Context, evt OutboxEvent) error {
      if err := p.nc.Publish(subject, evt.Payload); err != nil {
          metrics.EventsFailed.WithLabelValues(evt.AggType, evt.EventType).Inc()
          return p.handleRetry(ctx, evt, err)
      }

      metrics.EventsPublished.WithLabelValues(evt.AggType, evt.EventType).Inc()
      return p.markPublished(ctx, evt.ID)
  }
  ```
  - **CA:** Métricas instrumentadas.

#### T-02.2.5 — Crear main.go y Dockerfile

- **ST-02.2.5.1** — Crear `cmd/outbox-worker/main.go`.
  ```go
  package main

  import (
      "context"
      "net/http"
      "os"
      "os/signal"

      "github.com/jackc/pgx/v5/pgxpool"
      "github.com/nats-io/nats.go"
      "github.com/prometheus/client_golang/prometheus/promhttp"
      "github.com/serenidad/outbox-worker/internal/worker"
  )

  func main() {
      ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt)
      defer stop()

      // Connect to PostgreSQL
      pool, err := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
      if err != nil {
          panic(err)
      }
      defer pool.Close()

      // Connect to NATS
      nc, err := nats.Connect(os.Getenv("NATS_URL"))
      if err != nil {
          panic(err)
      }
      defer nc.Close()

      // Start metrics server
      go func() {
          http.Handle("/metrics", promhttp.Handler())
          http.ListenAndServe(":9090", nil)
      }()

      // Start dispatcher
      dispatcher := worker.NewDispatcher(pool, nc, 100)
      if err := dispatcher.Run(ctx); err != nil {
          panic(err)
      }
  }
  ```
  - **CA:** Main entrypoint creado.

- **ST-02.2.5.2** — Crear `Dockerfile`.
  ```dockerfile
  FROM golang:1.25-alpine AS builder
  WORKDIR /app
  COPY go.mod go.sum ./
  RUN go mod download
  COPY . .
  RUN CGO_ENABLED=0 go build -o outbox-worker ./cmd/outbox-worker

  FROM alpine:latest
  RUN apk --no-cache add ca-certificates
  WORKDIR /root/
  COPY --from=builder /app/outbox-worker .
  EXPOSE 9090
  CMD ["./outbox-worker"]
  ```
  - **CA:** Dockerfile multi-stage creado.

#### T-02.2.6 — Crear Kubernetes Deployment

- **ST-02.2.6.1** — Crear `infrastructure/k8s/outbox-worker/deployment.yaml`.
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: outbox-worker
    namespace: serenidad
  spec:
    replicas: 2  # Multiple workers for throughput
    selector:
      matchLabels:
        app: outbox-worker
    template:
      metadata:
        labels:
          app: outbox-worker
      spec:
        containers:
        - name: worker
          image: registry.gitlab.com/serenidad/outbox-worker:latest
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: connection-string
            - name: NATS_URL
              value: "nats://nats.nats-system.svc:4222"
          ports:
            - containerPort: 9090
              name: metrics
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /metrics
              port: 9090
            initialDelaySeconds: 10
            periodSeconds: 10
  ```
  - **CA:** Deployment YAML creado.

- **ST-02.2.6.2** — Crear Service para métricas.
  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: outbox-worker-metrics
    namespace: serenidad
    labels:
      app: outbox-worker
  spec:
    ports:
      - port: 9090
        name: metrics
    selector:
      app: outbox-worker
  ```
  - **CA:** Service creado.

- **ST-02.2.6.3** — Crear ServiceMonitor para Prometheus.
  ```yaml
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    name: outbox-worker
    namespace: serenidad
  spec:
    selector:
      matchLabels:
        app: outbox-worker
    endpoints:
      - port: metrics
        interval: 30s
  ```
  - **CA:** ServiceMonitor creado.

### Criterios de Aceptación de la HU-02.2

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Worker procesa batch de 100 eventos | Logs muestran batch processing |
| 2 | Eventos se publican a NATS correctamente | `nats sub "iam.>"` recibe eventos |
| 3 | Retry logic funciona (max 5 retries) | Simular fallo NATS, verificar retry_count |
| 4 | DLQ recibe eventos fallidos | Query `outbox_dlq` muestra eventos post-5 retries |
| 5 | Métricas expuestas en :9090/metrics | `curl localhost:9090/metrics` |
| 6 | Deployment Kubernetes funcional | `kubectl get pods -n serenidad` → 2 workers running |

### Definition of Done — HU-02.2

- [ ] Código Go compilado y testeado.
- [ ] Unit tests para dispatcher y publisher (>80% coverage).
- [ ] Integration test con PostgreSQL + NATS testcontainers.
- [ ] Dockerfile y imagen publicada en GitLab registry.
- [ ] Deployment en Kubernetes funcionando.
- [ ] ServiceMonitor configurado, Prometheus scraping métricas.

---

## Resumen de Entregables EP-02

**Database**:
- Tabla `outbox` con indexes optimizados
- Tabla `message_dedup` para consumer idempotency
- Tabla `outbox_dlq` para Dead Letter Queue
- Trigger LISTEN/NOTIFY (optimization)

**Worker**:
- Outbox worker Go (dispatcher + publisher)
- SELECT FOR UPDATE SKIP LOCKED pattern
- Exponential backoff retry (max 5)
- DLQ automation
- Prometheus metrics

**Kubernetes**:
- Deployment (2 replicas)
- Service (metrics endpoint)
- ServiceMonitor (Prometheus integration)

**Validación end-to-end**:
```bash
# 1. Insert evento en outbox (desde app)
INSERT INTO outbox (aggregate_type, aggregate_id, event_type, payload)
VALUES ('iam', gen_random_uuid(), 'user.created', '{"user_id":"123"}'::jsonb);

# 2. Verificar worker procesa
kubectl logs -f deployment/outbox-worker -n serenidad

# 3. Verificar evento publicado a NATS
nats sub "iam.user.created" --server=nats://nats.nats-system.svc:4222

# 4. Verificar métricas
curl http://outbox-worker-metrics.serenidad.svc:9090/metrics | grep outbox_events_published_total
```

---

## Riesgos y Mitigaciones EP-02

| Riesgo | Impacto | Probabilidad | Mitigación |
|--------|---------|--------------|------------|
| Outbox table bloat | Alto | Alta | Cleanup job (delete published > 7 days) |
| Worker crash perdiendo batch | Medio | Media | Idempotency keys + SKIP LOCKED (safe retry) |
| NATS temporalmente down | Medio | Media | Retry logic + exponential backoff |
| DLQ sin monitoreo | Alto | Baja | Alert en Prometheus si DLQ > 10 eventos |

---

**Tiempo estimado EP-02**: 2 semanas (Sprint 2)  
**Esfuerzo**: ~50 horas ingeniería + 15 horas testing
