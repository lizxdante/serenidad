# Evaluación Tecnológica Fase 2: Motor Médico — Stack Validado 2026

> **Contexto:** Este documento evalúa y valida las tecnologías seleccionadas para la Fase 2 "Motor Médico" del proyecto Serenamente. La evaluación cubre NATS JetStream, Go microservices patterns, Outbox pattern transaccional, Event Sourcing para clinical records, y Astro v6 deployment en Cloudflare Pages. Para cada dimensión se proporciona **recomendaciones finales validadas** con investigación Tavily Pro (Abril 2026).

---

## 1. NATS JetStream v2.11.x — Message Broker para Healthcare

### Plan actual: NATS JetStream v2.11.x con 4 streams

NATS JetStream es un message broker cloud-native con persistencia, delivery guarantees, y clustering. La versión 2.11.x incluye breaking changes respecto a 2.10.

### Arquitectura recomendada para healthcare PHI workloads

| Criterio | **Recomendación validada 2026** | Fuente |
|----------|--------------------------------|--------|
| **Deployment method** | Helm chart oficial (nats/nats v3+) como StatefulSet | Tavily Research |
| **HA topology** | 3-node cluster, R3 streams (replicationFactor: 3) | NATS docs + Tavily |
| **Storage backend** | File storage con block volumes (NO NFS) | NATS best practices |
| **Resource baseline** | CPU: 4 cores, RAM: 8 GiB por server (production) | Community validated |
| **Persistent volumes** | gp3 (AWS), pd-ssd (GCP), Premium SSD (Azure), Ceph RBD | Tavily + vendor docs |
| **Security** | TLS everywhere, NKeys/JWT auth, NetworkPolicies | HIPAA compliance pattern |
| **Monitoring** | Prometheus exporter puerto 7777, Grafana dashboard 14725 | NATS official |
| **Backup strategy** | `nats stream backup` + CSI VolumeSnapshots | Disaster recovery |

### Configuración Helm recomendada (production baseline)

```yaml
# values-production.yaml
nats:
  jetstream:
    enabled: true
    fileStorage:
      enabled: true
      size: 100Gi
      storageClassName: fast-ssd  # gp3 / pd-ssd
  cluster:
    enabled: true
    replicas: 3
  
config:
  cluster:
    enabled: true
  
resources:
  requests:
    cpu: 500m
    memory: 1Gi
  limits:
    cpu: 4
    memory: 8Gi

podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    - topologyKey: kubernetes.io/hostname

podDisruptionBudget:
  enabled: true
  minAvailable: 2  # Preserve quorum for 3-node

auth:
  enabled: true
  nkeys:
    users:
      - nkey: <NKEY_SEED>
```

### Streams recomendados Fase 2

| Stream | Subjects | Retention | Replicas | Max Age | Storage |
|--------|----------|-----------|----------|---------|---------|
| `IAM_EVENTS` | `iam.>` | Limits (1 year) | 3 | 8760h | ~10 GiB |
| `SCHEDULING_EVENTS` | `scheduling.>` | Limits (2 years) | 3 | 17520h | ~50 GiB |
| `CLINICAL_EVENTS` | `clinical.>` | Interest (∞) | 3 | ∞ | ∞ (pruning manual) |
| `BILLING_EVENTS` | `billing.>` | Limits (10 years) | 3 | 87600h | ~100 GiB |

**Nota crítica HIPAA**: Stream `CLINICAL_EVENTS` debe configurarse con `interest` retention (eventos permanecen hasta que consumers los procesan) debido a requisitos de auditoría médica. Implementar archival a cold storage (S3/B2) para cumplir retención de 7-10 años.

### Sizing guidance (mensajes/seg → storage)

Cálculo conservador (community validated):
- **1,000 msg/s @ 1 KiB payload, 24h retention, R3**: ~265 GiB/stream
- **Serenamente baseline (100 msg/s promedio)**: ~26.5 GiB/stream/día

**Recomendación inicial**: 100 GiB PVC por server (300 GiB total cluster) con `allowVolumeExpansion: true`.

### Operación y troubleshooting

**Upgrade path 2.10 → 2.11**:
1. Leer breaking changes: https://docs.nats.io/running-a-nats-service/upgrading
2. Pre-upgrade backup: `nats stream backup --all`
3. Lame Duck Mode: `nats-server --signal ldm`
4. Rolling restart con Helm: `helm upgrade nats nats/nats --reuse-values`
5. Post-upgrade validation: `nats stream list`, verificar consumers

**Alertas Prometheus críticas**:
- `JetStreamStorageHigh`: `(gnatsd_jetstream_storage_used / max_storage) > 0.8`
- `ConsumerLagHigh`: `sum(js_consumer_lag) > 10000`
- `InstanceDown`: `up{job="nats"} == 0`

### ✅ Veredicto: NATS JetStream v2.11.x — **MANTENER y VALIDADO**

NATS JetStream es la elección correcta para Serenamente:
- **Open source puro** (Apache 2.0) → soberanía tecnológica ✓
- **Performance**: ~1M msg/s en hardware modesto
- **HIPAA-ready**: TLS, auth, audit trails, encryption at rest (disk level)
- **Operational maturity**: Helm charts production-ready, Prometheus exporter

**Gap identificado**: No existe BAA público de Synadia. Solución: self-hosted cluster con disk encryption + KMS.

---

## 2. Go Microservices — Chi v5 + pgx v5 Patterns

### Plan actual: Go 1.25.x con Chi router y pgxpool

Go 1.25 incluye mejoras de runtime (Green Tea GC, small-object allocation optimization) y slog enhancements.

### Arquitectura de microservicio recomendada

```
┌─────────────────────────────────────────┐
│  HTTP Layer (Chi v5 router)            │
│  - Middleware: RequestID, Logger,      │
│    Recoverer, Timeout (30s)            │
│  - Route groups: /v1/api, /internal    │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│  Domain Layer                           │
│  - Aggregates (clinical, scheduling)    │
│  - Commands & Events                    │
│  - Business rules validation            │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│  Persistence (pgxpool v5)               │
│  - Connection pool: MinConns=5,         │
│    MaxConns=25, MaxConnLifetime=1h     │
│  - Transaction pattern: defer Rollback  │
│  - Outbox pattern: same TX              │
└─────────────────────────────────────────┘
```

### Chi v5 — Configuración baseline

```go
func NewRouter(pool *pgxpool.Pool) chi.Router {
    r := chi.NewRouter()
    
    // Core middleware
    r.Use(middleware.RequestID)
    r.Use(middleware.RealIP)
    r.Use(middleware.Logger)
    r.Use(middleware.Recoverer)
    r.Use(middleware.Timeout(30 * time.Second))
    
    // API routes
    r.Route("/v1", func(r chi.Router) {
        r.Route("/appointments", func(r chi.Router) {
            r.Post("/", createAppointmentHandler(pool))
            r.Get("/{id}", getAppointmentHandler(pool))
        })
    })
    
    // Internal routes (health, metrics)
    r.Get("/healthz", healthCheckHandler(pool))
    r.Get("/readyz", readinessHandler(pool))
    
    return r
}
```

**Principios validados**:
- Context propagation: `r.Context()` → `pool.QueryRow(ctx, ...)` para cancellation
- Middleware timeout: 30s default, ajustable por ruta
- Structured logging: usar `slog` (Go 1.25+) con request ID correlation

### pgxpool v5 — Tuning y patterns

**Pool configuration**:
```go
cfg, _ := pgxpool.ParseConfig(dsn)
cfg.MinConns = 5          // Warm connections
cfg.MaxConns = 25         // Per-pod ceiling
cfg.MaxConnLifetime = time.Hour
cfg.MaxConnIdleTime = 30 * time.Minute
cfg.HealthCheckPeriod = 1 * time.Minute

pool, err := pgxpool.NewWithConfig(ctx, cfg)
```

**Transaction pattern (safe)**:
```go
tx, err := pool.Begin(ctx)
if err != nil { return err }
defer tx.Rollback(ctx)  // Safe no-op if committed

// Business logic
if err := updateAggregate(ctx, tx); err != nil {
    return err  // Rollback happens
}

// Outbox insert (same TX)
if err := insertOutboxEvent(ctx, tx, event); err != nil {
    return err
}

return tx.Commit(ctx)  // Atomic commit
```

**PgBouncer tradeoffs**:
- **NO recomendado inicialmente**: pgxpool maneja pooling eficientemente in-process
- **Considerar PgBouncer si**: >500 concurrent connections, o multi-service connection sharing
- **Caveat**: PgBouncer transaction mode ROMPE prepared statements (pgx auto-prepares)

### Kubernetes deployment pattern

**Resource requests/limits (baseline)**:
```yaml
resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi
```

**Probes**:
```yaml
readinessProbe:
  httpGet:
    path: /readyz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3

livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3

lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]  # Allow drain
```

**Graceful shutdown (Go)**:
```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt)
defer stop()

srv := &http.Server{Addr: ":8080", Handler: router}

go func() {
    if err := srv.ListenAndServe(); err != http.ErrServerClosed {
        log.Fatal(err)
    }
}()

<-ctx.Done()
shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

srv.Shutdown(shutdownCtx)
pool.Close()  // May block; use timeout wrapper if needed
```

### ✅ Veredicto: Go 1.25 + Chi v5 + pgx v5 — **VALIDADO**

Stack maduro y production-ready:
- **Chi**: Lightweight, middleware-first, context-native
- **pgx v5**: Fastest PG driver, prepared statements, native types
- **Go 1.25**: Green Tea GC, small-object alloc improvements

---

## 3. Outbox Pattern — Transactional Messaging

### Patrón recomendado: Polling + SELECT FOR UPDATE SKIP LOCKED

**Schema (PostgreSQL)**:
```sql
CREATE TABLE outbox (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type text NOT NULL,
    aggregate_id uuid NOT NULL,
    event_type text NOT NULL,
    payload jsonb NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now(),
    published_at timestamptz,
    retry_count int NOT NULL DEFAULT 0,
    trace_context text
);

CREATE INDEX idx_outbox_unpublished 
    ON outbox(created_at) 
    WHERE published_at IS NULL;
```

**Implementación Go (relay worker)**:
```go
// Dispatcher (polling loop)
func runOutboxRelay(ctx context.Context, pool *pgxpool.Pool, natsConn *nats.Conn) {
    ticker := time.NewTicker(500 * time.Millisecond)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            processBatch(ctx, pool, natsConn, 100)
        }
    }
}

// Batch processor
func processBatch(ctx context.Context, pool *pgxpool.Pool, nc *nats.Conn, limit int) {
    tx, _ := pool.Begin(ctx)
    defer tx.Rollback(ctx)
    
    rows, _ := tx.Query(ctx, `
        SELECT id, aggregate_type, event_type, payload
        FROM outbox
        WHERE published_at IS NULL
        ORDER BY created_at
        LIMIT $1
        FOR UPDATE SKIP LOCKED
    `, limit)
    
    var batch []OutboxEvent
    for rows.Next() {
        var e OutboxEvent
        rows.Scan(&e.ID, &e.AggType, &e.EventType, &e.Payload)
        batch = append(batch, e)
    }
    rows.Close()
    tx.Commit(ctx)
    
    // Publish outside transaction
    for _, evt := range batch {
        subject := fmt.Sprintf("%s.%s", evt.AggType, evt.EventType)
        if err := nc.Publish(subject, evt.Payload); err != nil {
            handleRetry(ctx, pool, evt)
            continue
        }
        markPublished(ctx, pool, evt.ID)
    }
}
```

**Retry strategy (exponential backoff)**:
```go
func handleRetry(ctx context.Context, pool *pgxpool.Pool, evt OutboxEvent) {
    if evt.RetryCount >= 5 {
        moveToDLQ(ctx, pool, evt)
        return
    }
    
    _, err := pool.Exec(ctx, `
        UPDATE outbox
        SET retry_count = retry_count + 1
        WHERE id = $1
    `, evt.ID)
}
```

**LISTEN/NOTIFY optimization** (opcional):
```go
// Wakeup poller on INSERT via trigger
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

### Idempotency (consumer side)

**Deduplication table**:
```sql
CREATE TABLE message_dedup (
    message_id uuid PRIMARY KEY,
    processed_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX idx_dedup_ttl 
    ON message_dedup(processed_at)
    WHERE processed_at > now() - interval '7 days';
```

**Consumer pattern**:
```go
func handleMessage(ctx context.Context, pool *pgxpool.Pool, msg *nats.Msg) {
    var evt Event
    json.Unmarshal(msg.Data, &evt)
    
    // Check idempotency
    var exists bool
    pool.QueryRow(ctx, `
        SELECT EXISTS(SELECT 1 FROM message_dedup WHERE message_id = $1)
    `, evt.ID).Scan(&exists)
    
    if exists {
        msg.Ack()  // Already processed
        return
    }
    
    tx, _ := pool.Begin(ctx)
    defer tx.Rollback(ctx)
    
    // Process event
    applyEvent(ctx, tx, evt)
    
    // Record idempotency
    tx.Exec(ctx, `
        INSERT INTO message_dedup (message_id) VALUES ($1)
    `, evt.ID)
    
    tx.Commit(ctx)
    msg.Ack()
}
```

### ✅ Veredicto: Outbox Pattern — **VALIDADO**

Patrón correcto para at-least-once delivery con atomicidad:
- **Correctness**: Business write + outbox en misma TX
- **Performance**: SKIP LOCKED permite workers paralelos sin deadlocks
- **Scalability**: Batch size ~100 rows, autovacuum tuning necesario

---

## 4. Event Sourcing — Clinical Records

### Arquitectura CQRS + Event Store en PostgreSQL

**Event store schema**:
```sql
CREATE TABLE events (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_id uuid NOT NULL,
    aggregate_type text NOT NULL,
    version int NOT NULL,
    event_type text NOT NULL,
    payload jsonb NOT NULL,
    metadata jsonb,
    created_at timestamptz NOT NULL DEFAULT now(),
    position bigserial
);

CREATE UNIQUE INDEX ux_events_aggregate_version 
    ON events(aggregate_id, version);

CREATE INDEX idx_events_aggregate 
    ON events(aggregate_id, created_at);

-- Partition by time (RANGE)
-- Implementar pg_partman para crear particiones automáticas
```

**Aggregate snapshot table**:
```sql
CREATE TABLE snapshots (
    aggregate_id uuid PRIMARY KEY,
    aggregate_type text NOT NULL,
    version int NOT NULL,
    state jsonb NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now()
);
```

**Projection table (read model)**:
```sql
CREATE TABLE clinical_records_view (
    patient_id uuid PRIMARY KEY,
    full_name text NOT NULL,
    date_of_birth date NOT NULL,
    medical_history jsonb,
    last_encounter_at timestamptz,
    version int NOT NULL,  -- For optimistic locking on writes
    updated_at timestamptz NOT NULL
);
```

### Command handler pattern (Go)

```go
type CreateEncounterCommand struct {
    PatientID    uuid.UUID
    ProviderID   uuid.UUID
    ChiefComplaint string
    IdempotencyKey string
}

func (h *CommandHandler) CreateEncounter(ctx context.Context, cmd CreateEncounterCommand) error {
    // 1. Check idempotency
    if h.isDuplicate(ctx, cmd.IdempotencyKey) {
        return nil  // Already processed
    }
    
    // 2. Load aggregate events
    events := h.loadEvents(ctx, cmd.PatientID)
    
    // 3. Rehydrate aggregate state
    patient := rehydratePatient(events)
    
    // 4. Business logic
    newEvents, err := patient.CreateEncounter(cmd)
    if err != nil {
        return err
    }
    
    // 5. Append events (with optimistic concurrency check)
    return h.appendEvents(ctx, cmd.PatientID, patient.Version, newEvents, cmd.IdempotencyKey)
}

func (h *CommandHandler) appendEvents(ctx context.Context, aggID uuid.UUID, expectedVersion int, events []Event, idempKey string) error {
    tx, _ := h.pool.Begin(ctx)
    defer tx.Rollback(ctx)
    
    // Optimistic concurrency check
    var currentVersion int
    tx.QueryRow(ctx, `
        SELECT COALESCE(MAX(version), 0) FROM events WHERE aggregate_id = $1
    `, aggID).Scan(&currentVersion)
    
    if currentVersion != expectedVersion {
        return ErrConcurrencyConflict
    }
    
    // Append events
    for i, evt := range events {
        version := expectedVersion + i + 1
        _, err := tx.Exec(ctx, `
            INSERT INTO events (aggregate_id, aggregate_type, version, event_type, payload, metadata)
            VALUES ($1, $2, $3, $4, $5, $6)
        `, aggID, "Patient", version, evt.Type, evt.Payload, evt.Metadata)
        if err != nil {
            return err
        }
    }
    
    // Record idempotency
    tx.Exec(ctx, `INSERT INTO idempotency_keys (key) VALUES ($1)`, idempKey)
    
    // Outbox for integration events
    tx.Exec(ctx, `
        INSERT INTO outbox (aggregate_type, aggregate_id, event_type, payload)
        VALUES ($1, $2, $3, $4)
    `, "clinical", aggID, events[0].Type, events[0].Payload)
    
    return tx.Commit(ctx)
}
```

### Projection worker (catch-up subscription)

```go
func runProjectionWorker(ctx context.Context, pool *pgxpool.Pool) {
    var lastPosition int64
    
    for {
        rows, _ := pool.Query(ctx, `
            SELECT position, aggregate_id, event_type, payload
            FROM events
            WHERE position > $1
            ORDER BY position
            LIMIT 100
        `, lastPosition)
        
        for rows.Next() {
            var pos int64
            var aggID uuid.UUID
            var evtType string
            var payload []byte
            
            rows.Scan(&pos, &aggID, &evtType, &payload)
            
            // Update read model
            updateProjection(ctx, pool, evtType, payload)
            
            lastPosition = pos
        }
        rows.Close()
        
        time.Sleep(100 * time.Millisecond)
    }
}
```

### GDPR / HIPAA compliance considerations

**Right to erasure (GDPR Article 17)**:
- **NO borrar eventos** (inmutabilidad del audit trail)
- **Solución recomendada**: Crypto-shredding
  - Cifrar PII en `payload` con key derivada del patient_id
  - Al ejercer "derecho al olvido": destruir encryption key
  - Evento permanece pero payload es irrecuperable

**Ejemplo schema con encryption**:
```sql
CREATE TABLE events (
    -- ... campos existentes
    payload_encrypted bytea NOT NULL,
    encryption_key_id uuid NOT NULL REFERENCES encryption_keys(id)
);

CREATE TABLE encryption_keys (
    id uuid PRIMARY KEY,
    patient_id uuid NOT NULL,
    key_material bytea NOT NULL,  -- Encrypted with KMS
    deleted_at timestamptz  -- Crypto-shredding timestamp
);
```

**Audit logging**:
- Todo acceso a clinical records debe loguearse con:
  - User ID, timestamp, action (READ/WRITE), patient ID
  - Retención: 7-10 años (requisito HIPAA)

### ✅ Veredicto: Event Sourcing para Clinical Records — **VALIDADO CON CAVEATS**

Event Sourcing es apropiado para clinical records debido a:
- **Audit trail completo**: Inmutabilidad + versioning
- **Temporal queries**: "Estado del paciente en fecha X"
- **Compliance**: HIPAA audit requirements naturalmente satisfechos

**Caveats**:
- **GDPR**: Requiere crypto-shredding, NO es delete nativo
- **Performance**: Snapshots cada 50-100 eventos para aggregates largos
- **Complexity**: Mayor complejidad que CRUD, justificado solo si audit trail es crítico

---

## 5. Astro v6 — Landing Page en Cloudflare Pages

### Plan actual: Astro v6.x SSG en Cloudflare Pages

Astro v6 (beta en Abril 2026) incluye mejoras de performance y mejor integración con edge runtimes.

### Rendering strategy recomendada

| Tipo de página | Strategy | Justificación |
|----------------|----------|---------------|
| Homepage | SSG | SEO máximo, Core Web Vitals óptimos |
| Servicios clínicos | SSG | Contenido estático, cambios infrecuentes |
| Blog / Recursos | SSG + ISR | Content updates sin rebuild completo |
| Formularios contacto | Hybrid (SSG + edge middleware) | Form submission via CF Workers |
| Dashboard preview | SSR | Contenido personalizado (NO PHI) |

### Configuración recomendada

**astro.config.mjs** (SSG puro):
```js
import { defineConfig } from 'astro/config';

export default defineConfig({
  output: 'static',
  site: 'https://sereni.dad',
  integrations: [],
  vite: {
    build: {
      cssMinify: 'lightningcss',
      rollupOptions: {
        output: {
          manualChunks: {
            'vendor': ['react', 'react-dom']  // Si se usa React islands
          }
        }
      }
    }
  },
  image: {
    service: {
      entrypoint: 'astro/assets/services/sharp'  // Build-time optimization
    }
  }
});
```

**Performance optimizations**:
```astro
---
// src/pages/index.astro
import { Image } from 'astro:assets';
import heroImage from '../assets/hero.jpg';
---

<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    
    <!-- Preload LCP image -->
    <link rel="preload" as="image" href={heroImage.src} fetchpriority="high">
    
    <!-- Inline critical CSS -->
    <style is:inline>
        /* Critical above-the-fold CSS */
    </style>
</head>
<body>
    <Image 
        src={heroImage} 
        alt="Serenamente - Salud Mental" 
        loading="eager"
        format="avif"
        fallbackFormat="webp"
    />
</body>
</html>
```

### Cloudflare Pages deployment

**Build configuration** (cloudflare pages settings):
```bash
Build command: npm run build
Build output directory: dist
Node version: 22
```

**Environment variables** (Cloudflare Pages dashboard):
```
# Build-time (public)
PUBLIC_SITE_URL=https://sereni.dad

# Runtime (private - NO usar para PHI)
# (Pages no maneja PHI; solo marketing content)
```

**CI/CD con GitLab** (.gitlab-ci.yml):
```yaml
stages:
  - test
  - deploy

test:
  stage: test
  image: node:22
  script:
    - npm ci
    - npm run lint
    - npm run build
    - npx lighthouse-ci --upload-target=temporary-public-storage

deploy_production:
  stage: deploy
  image: node:22
  only:
    - main
  script:
    - npx wrangler pages deploy dist --project-name=serenidad-landing
  environment:
    name: production
    url: https://sereni.dad
```

### SEO Healthcare best practices

**Structured data (JSON-LD)**:
```astro
---
const organizationSchema = {
  "@context": "https://schema.org",
  "@type": "MedicalOrganization",
  "name": "Serenamente",
  "url": "https://sereni.dad",
  "logo": "https://sereni.dad/logo.png",
  "contactPoint": {
    "@type": "ContactPoint",
    "telephone": "+1-xxx-xxx-xxxx",
    "contactType": "Customer Service",
    "availableLanguage": ["es", "en"]
  },
  "medicalSpecialty": "Psychiatry"
};
---

<script type="application/ld+json" set:html={JSON.stringify(organizationSchema)} />
```

**Healthcare-specific meta tags**:
```html
<meta name="robots" content="index, follow">
<meta name="googlebot" content="index, follow, max-snippet:-1, max-image-preview:large">

<!-- HIPAA disclaimer -->
<meta name="description" content="Servicios de salud mental. Este sitio NO maneja información médica protegida (PHI). Para consultas médicas, use nuestro portal seguro.">
```

### Core Web Vitals targets (healthcare)

| Métrica | Target | Estrategia |
|---------|--------|------------|
| **LCP** | < 2.5s | Preload hero image, CDN caching, AVIF format |
| **CLS** | < 0.1 | Size attributes on images, no layout shifts |
| **INP** | < 200ms | Defer non-critical JS, use islands architecture |
| **TTFB** | < 600ms | Cloudflare CDN (300+ PoPs globally) |

### ✅ Veredicto: Astro v6 SSG en Cloudflare Pages — **VALIDADO**

Stack óptimo para landing healthcare:
- **SEO**: SSG pre-renderizado = indexación inmediata
- **Performance**: Cloudflare CDN global, Core Web Vitals excelentes
- **Cost**: $0/mes tier gratuito (500 builds/mes)
- **Security**: NO maneja PHI, solo marketing content público

**IMPORTANTE**: Landing page NO debe recolectar ni mostrar PHI. Formularios de contacto deben enviar a backend protegido con BAA.

---

## Conclusiones y Roadmap de Implementación

### Validación global del stack Fase 2

| Tecnología | Status | Risk Level | Action |
|------------|--------|------------|--------|
| NATS JetStream 2.11.x | ✅ Validado | LOW | Implementar según Helm chart |
| Go 1.25 + Chi + pgx | ✅ Validado | LOW | Usar patterns documentados |
| Outbox Pattern | ✅ Validado | MEDIUM | Testing exhaustivo de retries |
| Event Sourcing | ✅ Con caveats | MEDIUM-HIGH | Crypto-shredding para GDPR |
| Astro v6 CF Pages | ✅ Validado | LOW | NO manejar PHI en landing |

### Roadmap de implementación (priorizado)

**Sprint 1-2: Event Bus**
1. Deploy NATS cluster (3 nodes, Helm)
2. Crear 4 streams (IAM, Scheduling, Clinical, Billing)
3. Configurar TLS + NKeys auth
4. Setup Prometheus monitoring

**Sprint 3: Outbox Infrastructure**
1. Crear schema `outbox` + indexes
2. Implementar relay worker (polling + SKIP LOCKED)
3. Testing: concurrent workers, retries, DLQ
4. LISTEN/NOTIFY optimization

**Sprint 4-5: Scheduling Service**
1. Scaffold Go service (Chi router)
2. Implementar commands: CreateAppointment, CancelAppointment
3. Outbox integration (eventos a NATS)
4. Kubernetes deployment + probes

**Sprint 6-7: Clinical Record Service**
1. Event store schema + partitioning
2. Command handlers (CreateEncounter, RecordObservation)
3. Snapshot strategy (cada 50 eventos)
4. Projection workers (read models)
5. Crypto-shredding para GDPR

**Sprint 8: Landing Page**
1. Astro v6 project scaffold
2. Homepage + servicios pages (SSG)
3. SEO optimization (structured data, meta tags)
4. Cloudflare Pages deployment + CI/CD

### Métricas de éxito

| Métrica | Target | Validación |
|---------|--------|------------|
| NATS message throughput | > 1,000 msg/s | `nats bench` |
| Outbox lag | < 1s p95 | Prometheus query |
| Event store write latency | < 50ms p95 | Custom metrics |
| Landing page LCP | < 2.0s | Lighthouse CI |
| Service uptime | 99.9% | Prometheus + alerting |

### Gaps y próximos pasos

**Documentación pendiente**:
- [ ] Runbook operacional NATS (upgrade, troubleshooting)
- [ ] Crypto-shredding implementation guide (GDPR)
- [ ] Disaster recovery playbook (NATS + PostgreSQL)
- [ ] Load testing suite (K6 scripts)

**Validaciones técnicas**:
- [ ] Performance testing: 10,000 msg/s sustained (NATS)
- [ ] Chaos testing: pod kills, network partitions
- [ ] GDPR compliance audit (crypto-shredding)
- [ ] HIPAA security review (TLS, auth, audit logs)

---

**Firma de investigación**: Tavily Pro Research + Community Validated Patterns (Abril 2026)
