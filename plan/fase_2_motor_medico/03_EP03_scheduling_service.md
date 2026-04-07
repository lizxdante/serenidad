# EP-03 — Scheduling Service (Agendamiento de Citas)

**Épica:** Como desarrollador de plataforma médica, necesito implementar el Scheduling Service que gestione la creación, cancelación y consulta de citas médicas con lógica de negocio, validaciones, y emisión de eventos vía Outbox pattern, para permitir que pacientes y providers agenden consultas de manera confiable.

**Origen:** Fase 2 (2.C) del roadmap arquitectónico + research/08_evaluacion_tecnologica_fase2_motor_medico.md  
**Prioridad:** Crítica  
**Sprint:** S3  
**Dependencias Entrantes:** EP-01 (NATS), EP-02 (Outbox Worker), Fase 1 (IAM Service + JWT)  
**Dependencias Salientes:** EP-04 (Clinical Record Service reutiliza patterns)

---

## HU-03.1 — Scaffold Go Service con Chi Router

**Como** desarrollador backend,  
**quiero** tener un servicio Go scaffold con Chi router, pgxpool, health checks, y estructura modular,  
**para que** pueda implementar comandos de negocio siguiendo best practices validadas.

### Contexto técnico

**Stack**:
- Go 1.25.x
- Chi v5 router
- pgxpool v5 (connection pooling)
- Middleware: RequestID, Logger, Recoverer, Timeout

**Estructura de directorios**:
```
services/scheduling/
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── domain/
│   │   ├── appointment.go
│   │   └── commands.go
│   ├── handlers/
│   │   └── http.go
│   ├── repository/
│   │   └── postgres.go
│   └── outbox/
│       └── writer.go
├── migrations/
│   └── 001_create_appointments.up.sql
├── Dockerfile
└── go.mod
```

### Tareas y Subtareas

#### T-03.1.1 — Crear estructura de proyecto

- **ST-03.1.1.1** — Crear directorios.
  ```bash
  mkdir -p services/scheduling/{cmd/server,internal/{domain,handlers,repository,outbox},migrations}
  ```
  - **CA:** Estructura creada.

- **ST-03.1.1.2** — Inicializar módulo Go.
  ```bash
  cd services/scheduling
  go mod init github.com/serenidad/scheduling
  ```
  - **CA:** `go.mod` creado.

- **ST-03.1.1.3** — Agregar dependencias.
  ```bash
  go get github.com/go-chi/chi/v5
  go get github.com/go-chi/chi/v5/middleware
  go get github.com/jackc/pgx/v5/pgxpool
  go get github.com/google/uuid
  go get github.com/prometheus/client_golang/prometheus
  ```
  - **CA:** Dependencias instaladas.

#### T-03.1.2 — Implementar router Chi con middleware

- **ST-03.1.2.1** — Crear `internal/handlers/router.go`.
  ```go
  package handlers

  import (
      "net/http"
      "time"
      "github.com/go-chi/chi/v5"
      "github.com/go-chi/chi/v5/middleware"
  )

  func NewRouter(h *AppointmentHandler) chi.Router {
      r := chi.NewRouter()

      // Core middleware
      r.Use(middleware.RequestID)
      r.Use(middleware.RealIP)
      r.Use(middleware.Logger)
      r.Use(middleware.Recoverer)
      r.Use(middleware.Timeout(30 * time.Second))

      // Health endpoints
      r.Get("/healthz", healthCheckHandler)
      r.Get("/readyz", readinessHandler(h.repo))

      // API routes
      r.Route("/v1", func(r chi.Router) {
          r.Route("/appointments", func(r chi.Router) {
              r.Post("/", h.CreateAppointment)
              r.Get("/{id}", h.GetAppointment)
              r.Delete("/{id}", h.CancelAppointment)
          })
      })

      // Metrics
      r.Handle("/metrics", promhttp.Handler())

      return r
  }

  func healthCheckHandler(w http.ResponseWriter, r *http.Request) {
      w.WriteHeader(http.StatusOK)
      w.Write([]byte(`{"status":"ok"}`))
  }

  func readinessHandler(repo Repository) http.HandlerFunc {
      return func(w http.ResponseWriter, r *http.Request) {
          if err := repo.Ping(r.Context()); err != nil {
              w.WriteHeader(http.StatusServiceUnavailable)
              return
          }
          w.WriteHeader(http.StatusOK)
          w.Write([]byte(`{"status":"ready"}`))
      }
  }
  ```
  - **CA:** Router implementado con middleware.

#### T-03.1.3 — Implementar main.go con graceful shutdown

- **ST-03.1.3.1** — Crear `cmd/server/main.go`.
  ```go
  package main

  import (
      "context"
      "log"
      "net/http"
      "os"
      "os/signal"
      "time"

      "github.com/jackc/pgx/v5/pgxpool"
      "github.com/serenidad/scheduling/internal/handlers"
      "github.com/serenidad/scheduling/internal/repository"
  )

  func main() {
      ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt)
      defer stop()

      // Database connection
      pool, err := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
      if err != nil {
          log.Fatalf("Unable to connect to database: %v", err)
      }
      defer pool.Close()

      // Repository
      repo := repository.NewPostgresRepo(pool)

      // Handlers
      handler := handlers.NewAppointmentHandler(repo)

      // Router
      router := handlers.NewRouter(handler)

      // HTTP Server
      srv := &http.Server{
          Addr:         ":8080",
          Handler:      router,
          ReadTimeout:  10 * time.Second,
          WriteTimeout: 10 * time.Second,
      }

      // Start server in goroutine
      go func() {
          log.Printf("Starting server on :8080")
          if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
              log.Fatalf("Server error: %v", err)
          }
      }()

      // Wait for interrupt
      <-ctx.Done()
      log.Println("Shutting down gracefully...")

      shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
      defer cancel()

      if err := srv.Shutdown(shutdownCtx); err != nil {
          log.Printf("HTTP shutdown error: %v", err)
      }

      pool.Close()
      log.Println("Server stopped")
  }
  ```
  - **CA:** Main con graceful shutdown implementado.

#### T-03.1.4 — Crear Dockerfile

- **ST-03.1.4.1** — Crear `Dockerfile`.
  ```dockerfile
  FROM golang:1.25-alpine AS builder
  WORKDIR /app
  COPY go.mod go.sum ./
  RUN go mod download
  COPY . .
  RUN CGO_ENABLED=0 GOOS=linux go build -o scheduling-service ./cmd/server

  FROM alpine:latest
  RUN apk --no-cache add ca-certificates
  WORKDIR /root/
  COPY --from=builder /app/scheduling-service .
  EXPOSE 8080
  CMD ["./scheduling-service"]
  ```
  - **CA:** Dockerfile multi-stage creado.

### Criterios de Aceptación de la HU-03.1

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Proyecto Go compila sin errores | `go build ./cmd/server` |
| 2 | Server inicia en :8080 | `curl localhost:8080/healthz` → 200 |
| 3 | Health check funciona | `/healthz` retorna `{"status":"ok"}` |
| 4 | Readiness check valida DB | `/readyz` retorna 200 si PG conectado |
| 5 | Graceful shutdown funciona | SIGTERM → server drain en <30s |

### Definition of Done — HU-03.1

- [ ] Código compila y tests pasan.
- [ ] Dockerfile construye imagen exitosamente.
- [ ] Health y readiness endpoints funcionan.
- [ ] Graceful shutdown testeado manualmente.

---

## HU-03.2 — Commands: Crear y Cancelar Citas

**Como** paciente o provider,  
**quiero** poder crear y cancelar citas médicas mediante API REST,  
**para que** pueda gestionar mi agenda de consultas de manera confiable con validaciones de negocio y eventos auditables.

### Contexto técnico

**Domain events**:
- `AppointmentCreated`: paciente agenda cita con provider
- `AppointmentCancelled`: cita cancelada (paciente o provider)

**Business rules**:
- Cita debe ser en el futuro (>= now + 1h)
- Provider debe tener disponibilidad en slot
- Paciente no puede tener >3 citas pendientes simultáneas
- Cancelación solo permitida si cita está >24h en el futuro

### Tareas y Subtareas

#### T-03.2.1 — Crear migration para appointments table

- **ST-03.2.1.1** — Crear `migrations/001_create_appointments.up.sql`.
  ```sql
  CREATE TABLE IF NOT EXISTS appointments (
      id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
      patient_id uuid NOT NULL,
      provider_id uuid NOT NULL,
      scheduled_at timestamptz NOT NULL,
      duration_minutes int NOT NULL DEFAULT 30,
      status text NOT NULL DEFAULT 'scheduled',
      chief_complaint text,
      notes text,
      created_at timestamptz NOT NULL DEFAULT now(),
      updated_at timestamptz NOT NULL DEFAULT now(),
      cancelled_at timestamptz,
      cancellation_reason text,
      CONSTRAINT valid_status CHECK (status IN ('scheduled', 'cancelled', 'completed', 'no_show'))
  );

  CREATE INDEX idx_appointments_patient ON appointments(patient_id, scheduled_at);
  CREATE INDEX idx_appointments_provider ON appointments(provider_id, scheduled_at);
  CREATE INDEX idx_appointments_status ON appointments(status) WHERE status = 'scheduled';

  COMMENT ON TABLE appointments IS 'Medical appointments scheduling';
  ```
  - **CA:** Migration creado.

- **ST-03.2.1.2** — Ejecutar migration.
  ```bash
  migrate -path migrations -database "$DATABASE_URL" up
  ```
  - **CA:** Tabla `appointments` existe.

#### T-03.2.2 — Implementar domain model y commands

- **ST-03.2.2.1** — Crear `internal/domain/appointment.go`.
  ```go
  package domain

  import (
      "time"
      "github.com/google/uuid"
  )

  type Appointment struct {
      ID               uuid.UUID
      PatientID        uuid.UUID
      ProviderID       uuid.UUID
      ScheduledAt      time.Time
      DurationMinutes  int
      Status           string
      ChiefComplaint   string
      Notes            string
      CreatedAt        time.Time
      UpdatedAt        time.Time
  }

  type CreateAppointmentCommand struct {
      PatientID       uuid.UUID `json:"patient_id"`
      ProviderID      uuid.UUID `json:"provider_id"`
      ScheduledAt     time.Time `json:"scheduled_at"`
      DurationMinutes int       `json:"duration_minutes"`
      ChiefComplaint  string    `json:"chief_complaint"`
  }

  type CancelAppointmentCommand struct {
      AppointmentID uuid.UUID `json:"appointment_id"`
      Reason        string    `json:"reason"`
      CancelledBy   uuid.UUID `json:"cancelled_by"` // user_id from JWT
  }
  ```
  - **CA:** Domain models definidos.

- **ST-03.2.2.2** — Crear `internal/domain/events.go`.
  ```go
  package domain

  import (
      "encoding/json"
      "time"
      "github.com/google/uuid"
  )

  type AppointmentCreatedEvent struct {
      EventID         uuid.UUID `json:"event_id"`
      AppointmentID   uuid.UUID `json:"appointment_id"`
      PatientID       uuid.UUID `json:"patient_id"`
      ProviderID      uuid.UUID `json:"provider_id"`
      ScheduledAt     time.Time `json:"scheduled_at"`
      DurationMinutes int       `json:"duration_minutes"`
      OccurredAt      time.Time `json:"occurred_at"`
  }

  func (e AppointmentCreatedEvent) ToJSON() ([]byte, error) {
      return json.Marshal(e)
  }

  type AppointmentCancelledEvent struct {
      EventID       uuid.UUID `json:"event_id"`
      AppointmentID uuid.UUID `json:"appointment_id"`
      CancelledBy   uuid.UUID `json:"cancelled_by"`
      Reason        string    `json:"reason"`
      OccurredAt    time.Time `json:"occurred_at"`
  }

  func (e AppointmentCancelledEvent) ToJSON() ([]byte, error) {
      return json.Marshal(e)
  }
  ```
  - **CA:** Event types definidos.

#### T-03.2.3 — Implementar repository con Outbox pattern

- **ST-03.2.3.1** — Crear `internal/repository/postgres.go`.
  ```go
  package repository

  import (
      "context"
      "errors"
      "time"
      "github.com/jackc/pgx/v5/pgxpool"
      "github.com/google/uuid"
      "github.com/serenidad/scheduling/internal/domain"
  )

  type PostgresRepo struct {
      pool *pgxpool.Pool
  }

  func NewPostgresRepo(pool *pgxpool.Pool) *PostgresRepo {
      return &PostgresRepo{pool: pool}
  }

  func (r *PostgresRepo) CreateAppointment(ctx context.Context, cmd domain.CreateAppointmentCommand) (*domain.Appointment, error) {
      // Business rule: validate scheduled_at is in future
      if cmd.ScheduledAt.Before(time.Now().Add(1 * time.Hour)) {
          return nil, errors.New("appointment must be scheduled at least 1 hour in future")
      }

      tx, err := r.pool.Begin(ctx)
      if err != nil {
          return nil, err
      }
      defer tx.Rollback(ctx)

      // Business rule: check patient doesn't have >3 pending appointments
      var pendingCount int
      err = tx.QueryRow(ctx, `
          SELECT COUNT(*) FROM appointments
          WHERE patient_id = $1 AND status = 'scheduled' AND scheduled_at > now()
      `, cmd.PatientID).Scan(&pendingCount)

      if err != nil {
          return nil, err
      }

      if pendingCount >= 3 {
          return nil, errors.New("patient already has 3 pending appointments")
      }

      // Insert appointment
      appt := &domain.Appointment{
          ID:              uuid.New(),
          PatientID:       cmd.PatientID,
          ProviderID:      cmd.ProviderID,
          ScheduledAt:     cmd.ScheduledAt,
          DurationMinutes: cmd.DurationMinutes,
          Status:          "scheduled",
          ChiefComplaint:  cmd.ChiefComplaint,
          CreatedAt:       time.Now(),
          UpdatedAt:       time.Now(),
      }

      _, err = tx.Exec(ctx, `
          INSERT INTO appointments (id, patient_id, provider_id, scheduled_at, duration_minutes, 
                                    status, chief_complaint, created_at, updated_at)
          VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
      `, appt.ID, appt.PatientID, appt.ProviderID, appt.ScheduledAt, appt.DurationMinutes,
         appt.Status, appt.ChiefComplaint, appt.CreatedAt, appt.UpdatedAt)

      if err != nil {
          return nil, err
      }

      // Create event
      event := domain.AppointmentCreatedEvent{
          EventID:         uuid.New(),
          AppointmentID:   appt.ID,
          PatientID:       appt.PatientID,
          ProviderID:      appt.ProviderID,
          ScheduledAt:     appt.ScheduledAt,
          DurationMinutes: appt.DurationMinutes,
          OccurredAt:      time.Now(),
      }

      payload, _ := event.ToJSON()

      // Insert into outbox (same transaction)
      _, err = tx.Exec(ctx, `
          INSERT INTO outbox (aggregate_type, aggregate_id, event_type, payload)
          VALUES ($1, $2, $3, $4)
      `, "scheduling", appt.ID, "AppointmentCreated", payload)

      if err != nil {
          return nil, err
      }

      // Commit transaction
      if err := tx.Commit(ctx); err != nil {
          return nil, err
      }

      return appt, nil
  }

  func (r *PostgresRepo) CancelAppointment(ctx context.Context, cmd domain.CancelAppointmentCommand) error {
      tx, err := r.pool.Begin(ctx)
      if err != nil {
          return err
      }
      defer tx.Rollback(ctx)

      // Fetch appointment
      var appt domain.Appointment
      err = tx.QueryRow(ctx, `
          SELECT id, patient_id, provider_id, scheduled_at, status
          FROM appointments WHERE id = $1
      `, cmd.AppointmentID).Scan(&appt.ID, &appt.PatientID, &appt.ProviderID, &appt.ScheduledAt, &appt.Status)

      if err != nil {
          return errors.New("appointment not found")
      }

      // Business rule: only scheduled appointments can be cancelled
      if appt.Status != "scheduled" {
          return errors.New("only scheduled appointments can be cancelled")
      }

      // Business rule: cancellation only if >24h in future
      if appt.ScheduledAt.Before(time.Now().Add(24 * time.Hour)) {
          return errors.New("appointment can only be cancelled if >24h in future")
      }

      // Update appointment
      _, err = tx.Exec(ctx, `
          UPDATE appointments
          SET status = 'cancelled', cancelled_at = now(), cancellation_reason = $2, updated_at = now()
          WHERE id = $1
      `, cmd.AppointmentID, cmd.Reason)

      if err != nil {
          return err
      }

      // Create event
      event := domain.AppointmentCancelledEvent{
          EventID:       uuid.New(),
          AppointmentID: cmd.AppointmentID,
          CancelledBy:   cmd.CancelledBy,
          Reason:        cmd.Reason,
          OccurredAt:    time.Now(),
      }

      payload, _ := event.ToJSON()

      // Insert into outbox
      _, err = tx.Exec(ctx, `
          INSERT INTO outbox (aggregate_type, aggregate_id, event_type, payload)
          VALUES ($1, $2, $3, $4)
      `, "scheduling", cmd.AppointmentID, "AppointmentCancelled", payload)

      if err != nil {
          return err
      }

      return tx.Commit(ctx)
  }

  func (r *PostgresRepo) GetAppointment(ctx context.Context, id uuid.UUID) (*domain.Appointment, error) {
      var appt domain.Appointment
      err := r.pool.QueryRow(ctx, `
          SELECT id, patient_id, provider_id, scheduled_at, duration_minutes, status, 
                 chief_complaint, notes, created_at, updated_at
          FROM appointments WHERE id = $1
      `, id).Scan(&appt.ID, &appt.PatientID, &appt.ProviderID, &appt.ScheduledAt, 
                  &appt.DurationMinutes, &appt.Status, &appt.ChiefComplaint, 
                  &appt.Notes, &appt.CreatedAt, &appt.UpdatedAt)

      if err != nil {
          return nil, err
      }

      return &appt, nil
  }

  func (r *PostgresRepo) Ping(ctx context.Context) error {
      return r.pool.Ping(ctx)
  }
  ```
  - **CA:** Repository con Outbox pattern implementado.

#### T-03.2.4 — Implementar HTTP handlers

- **ST-03.2.4.1** — Crear `internal/handlers/appointment.go`.
  ```go
  package handlers

  import (
      "encoding/json"
      "net/http"
      "github.com/go-chi/chi/v5"
      "github.com/google/uuid"
      "github.com/serenidad/scheduling/internal/domain"
      "github.com/serenidad/scheduling/internal/repository"
  )

  type AppointmentHandler struct {
      repo *repository.PostgresRepo
  }

  func NewAppointmentHandler(repo *repository.PostgresRepo) *AppointmentHandler {
      return &AppointmentHandler{repo: repo}
  }

  func (h *AppointmentHandler) CreateAppointment(w http.ResponseWriter, r *http.Request) {
      var cmd domain.CreateAppointmentCommand
      if err := json.NewDecoder(r.Body).Decode(&cmd); err != nil {
          http.Error(w, "invalid request body", http.StatusBadRequest)
          return
      }

      appt, err := h.repo.CreateAppointment(r.Context(), cmd)
      if err != nil {
          http.Error(w, err.Error(), http.StatusBadRequest)
          return
      }

      w.Header().Set("Content-Type", "application/json")
      w.WriteHeader(http.StatusCreated)
      json.NewEncoder(w).Encode(appt)
  }

  func (h *AppointmentHandler) GetAppointment(w http.ResponseWriter, r *http.Request) {
      idStr := chi.URLParam(r, "id")
      id, err := uuid.Parse(idStr)
      if err != nil {
          http.Error(w, "invalid appointment ID", http.StatusBadRequest)
          return
      }

      appt, err := h.repo.GetAppointment(r.Context(), id)
      if err != nil {
          http.Error(w, "appointment not found", http.StatusNotFound)
          return
      }

      w.Header().Set("Content-Type", "application/json")
      json.NewEncoder(w).Encode(appt)
  }

  func (h *AppointmentHandler) CancelAppointment(w http.ResponseWriter, r *http.Request) {
      idStr := chi.URLParam(r, "id")
      id, err := uuid.Parse(idStr)
      if err != nil {
          http.Error(w, "invalid appointment ID", http.StatusBadRequest)
          return
      }

      var cmd domain.CancelAppointmentCommand
      if err := json.NewDecoder(r.Body).Decode(&cmd); err != nil {
          http.Error(w, "invalid request body", http.StatusBadRequest)
          return
      }
      cmd.AppointmentID = id

      // TODO: Extract user_id from JWT (Fase 1 integration)
      // cmd.CancelledBy = extractUserFromJWT(r)

      if err := h.repo.CancelAppointment(r.Context(), cmd); err != nil {
          http.Error(w, err.Error(), http.StatusBadRequest)
          return
      }

      w.WriteHeader(http.StatusNoContent)
  }
  ```
  - **CA:** Handlers implementados.

### Criterios de Aceptación de la HU-03.2

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | POST /v1/appointments crea cita | Status 201, retorna appointment JSON |
| 2 | Business rule: cita en futuro | POST con `scheduled_at` < now+1h → 400 |
| 3 | Business rule: max 3 citas pendientes | 4ta cita mismo paciente → 400 error |
| 4 | Evento AppointmentCreated en outbox | Query `outbox` muestra evento post-create |
| 5 | GET /v1/appointments/{id} retorna cita | Status 200, JSON con appointment |
| 6 | DELETE /v1/appointments/{id} cancela | Status 204, outbox tiene AppointmentCancelled |
| 7 | Cancellation rule: >24h future | Cancelar cita <24h → 400 error |

### Definition of Done — HU-03.2

- [ ] Endpoints POST, GET, DELETE implementados.
- [ ] Business rules validadas con tests.
- [ ] Outbox pattern funcional (eventos en misma TX).
- [ ] Integration tests con testcontainers PostgreSQL.

---

## HU-03.3 — Deployment Kubernetes

**Como** SRE,  
**quiero** desplegar Scheduling Service en Kubernetes con HPA, PDB, y probes correctamente configurados,  
**para que** el servicio sea resiliente y escalable automáticamente según carga.

### Tareas y Subtareas

#### T-03.3.1 — Crear Kubernetes manifests

- **ST-03.3.1.1** — Crear `infrastructure/k8s/scheduling/deployment.yaml`.
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: scheduling-service
    namespace: serenidad
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: scheduling-service
    template:
      metadata:
        labels:
          app: scheduling-service
      spec:
        containers:
        - name: service
          image: registry.gitlab.com/serenidad/scheduling:latest
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: connection-string
          ports:
            - containerPort: 8080
              name: http
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 1Gi
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
                command: ["/bin/sh", "-c", "sleep 5"]
  ```
  - **CA:** Deployment YAML creado.

- **ST-03.3.1.2** — Crear Service.
  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: scheduling-service
    namespace: serenidad
  spec:
    selector:
      app: scheduling-service
    ports:
      - port: 80
        targetPort: 8080
        name: http
    type: ClusterIP
  ```
  - **CA:** Service creado.

- **ST-03.3.1.3** — Crear HPA (Horizontal Pod Autoscaler).
  ```yaml
  apiVersion: autoscaling/v2
  kind: HorizontalPodAutoscaler
  metadata:
    name: scheduling-service-hpa
    namespace: serenidad
  spec:
    scaleTargetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: scheduling-service
    minReplicas: 2
    maxReplicas: 10
    metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 70
      - type: Resource
        resource:
          name: memory
          target:
            type: Utilization
            averageUtilization: 80
  ```
  - **CA:** HPA configurado.

- **ST-03.3.1.4** — Crear PodDisruptionBudget.
  ```yaml
  apiVersion: policy/v1
  kind: PodDisruptionBudget
  metadata:
    name: scheduling-service-pdb
    namespace: serenidad
  spec:
    minAvailable: 1
    selector:
      matchLabels:
        app: scheduling-service
  ```
  - **CA:** PDB creado.

#### T-03.3.2 — Configurar Ingress / IngressRoute (Traefik)

- **ST-03.3.2.1** — Crear `infrastructure/k8s/scheduling/ingressroute.yaml`.
  ```yaml
  apiVersion: traefik.containo.us/v1alpha1
  kind: IngressRoute
  metadata:
    name: scheduling-api
    namespace: serenidad
  spec:
    entryPoints:
      - websecure
    routes:
      - match: Host(`api.sereni.dad`) && PathPrefix(`/v1/appointments`)
        kind: Rule
        services:
          - name: scheduling-service
            port: 80
        middlewares:
          - name: iam-forwardauth  # JWT validation from Fase 1
            namespace: iam-system
    tls:
      secretName: api-sereni-dad-tls
  ```
  - **CA:** IngressRoute creado.

#### T-03.3.3 — Configurar CI/CD GitLab

- **ST-03.3.3.1** — Crear `.gitlab-ci.yml` para scheduling service.
  ```yaml
  stages:
    - test
    - build
    - deploy

  test:scheduling:
    stage: test
    image: golang:1.25
    script:
      - cd services/scheduling
      - go test -v ./...
      - go vet ./...

  build:scheduling:
    stage: build
    image: docker:latest
    services:
      - docker:dind
    script:
      - cd services/scheduling
      - docker build -t $CI_REGISTRY_IMAGE/scheduling:$CI_COMMIT_SHA .
      - docker push $CI_REGISTRY_IMAGE/scheduling:$CI_COMMIT_SHA
    only:
      - main

  deploy:scheduling:
    stage: deploy
    image: bitnami/kubectl:latest
    script:
      - kubectl set image deployment/scheduling-service \
          service=$CI_REGISTRY_IMAGE/scheduling:$CI_COMMIT_SHA \
          -n serenidad
    only:
      - main
    when: manual
  ```
  - **CA:** CI/CD pipeline configurado.

### Criterios de Aceptación de la HU-03.3

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Deployment con 2 replicas running | `kubectl get pods -n serenidad` |
| 2 | Service accesible internamente | `curl http://scheduling-service.serenidad.svc/healthz` |
| 3 | HPA configurado correctamente | `kubectl get hpa -n serenidad` |
| 4 | PDB activo (minAvailable=1) | `kubectl get pdb -n serenidad` |
| 5 | IngressRoute funcional | `curl https://api.sereni.dad/v1/appointments` (con JWT) |
| 6 | CI/CD pipeline exitoso | GitLab pipeline verde |

### Definition of Done — HU-03.3

- [ ] Kubernetes manifests commiteados en Git.
- [ ] Deployment funcional con 2+ pods.
- [ ] Service accesible desde BFF (Fase 1).
- [ ] HPA y PDB configurados.
- [ ] IngressRoute con TLS + JWT validation.
- [ ] CI/CD pipeline testeado end-to-end.

---

## Resumen de Entregables EP-03

**Código**:
- Go service (Chi + pgxpool)
- Domain models (Appointment, Commands, Events)
- Repository con Outbox pattern
- HTTP handlers (Create, Get, Cancel)
- Health/readiness checks
- Graceful shutdown

**Base de Datos**:
- Tabla `appointments`
- Indexes optimizados
- Business rules enforcement

**Kubernetes**:
- Deployment (2 replicas, HPA)
- Service (ClusterIP)
- PodDisruptionBudget
- IngressRoute (Traefik + JWT)

**CI/CD**:
- GitLab pipeline (test, build, deploy)
- Docker image registry

**Validación end-to-end**:
```bash
# 1. Crear cita (con JWT válido)
curl -X POST https://api.sereni.dad/v1/appointments \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "patient_id": "uuid-paciente",
    "provider_id": "uuid-provider",
    "scheduled_at": "2026-04-15T10:00:00Z",
    "duration_minutes": 30,
    "chief_complaint": "Evaluación inicial ansiedad"
  }'

# 2. Verificar evento en outbox
kubectl exec -it postgres-0 -n databases -- \
  psql -U postgres -d serenidad_db -c \
  "SELECT * FROM outbox WHERE aggregate_type='scheduling' ORDER BY created_at DESC LIMIT 1;"

# 3. Verificar evento publicado a NATS
nats sub "scheduling.AppointmentCreated" --server=nats://nats.nats-system.svc:4222

# 4. Consultar cita creada
curl https://api.sereni.dad/v1/appointments/{id} \
  -H "Authorization: Bearer $JWT_TOKEN"

# 5. Cancelar cita
curl -X DELETE https://api.sereni.dad/v1/appointments/{id} \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -d '{"reason": "Paciente reprogramó"}'
```

---

## Riesgos y Mitigaciones EP-03

| Riesgo | Impacto | Probabilidad | Mitigación |
|--------|---------|--------------|------------|
| Race condition en disponibilidad | Alto | Media | Locking optimista o pessimista en slots |
| Overbooking de provider | Crítico | Baja | Validación de slots disponibles pre-commit |
| Outbox table bloat | Medio | Alta | Cleanup job automático (EP-02) |
| JWT validation failure | Alto | Baja | Integration tests con IAM Service (Fase 1) |

---

**Tiempo estimado EP-03**: 2 semanas (Sprint 3)  
**Esfuerzo**: ~60 horas ingeniería + 20 horas testing + 10 horas deployment
