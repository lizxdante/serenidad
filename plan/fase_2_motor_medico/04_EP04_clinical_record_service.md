# EP-04 — Clinical Record Service (Event Sourcing + CQRS)

**Épica:** Como desarrollador de plataforma médica, necesito implementar el Clinical Record Service usando Event Sourcing y CQRS para registrar encuentros clínicos, observaciones, y diagnósticos con audit trail completo, para cumplir con requisitos HIPAA de auditoría médica y permitir queries temporales del estado del paciente.

**Origen:** Fase 2 (2.D) del roadmap arquitectónico + research/08_evaluacion_tecnologica_fase2_motor_medico.md  
**Prioridad:** Crítica  
**Sprint:** S4–S5 (4 semanas)  
**Dependencias Entrantes:** EP-01 (NATS), EP-02 (Outbox), EP-03 (Scheduling patterns)  
**Dependencias Salientes:** Ninguna (último servicio Fase 2)

---

## HU-04.1 — Event Store Schema y Partitioning

**Como** ingeniero de datos,  
**quiero** crear el event store schema en PostgreSQL con partitioning por tiempo y aggregate versioning,  
**para que** pueda almacenar millones de eventos clínicos de manera eficiente con optimistic concurrency control.

### Contexto técnico

**Event store design**:
- Tabla `events`: append-only, JSONB payload
- Unique constraint: `(aggregate_id, version)` para optimistic locking
- Partitioning: RANGE por `created_at` (mensual)
- Tabla `snapshots`: estado agregado cada N eventos
- Tabla `projections`: read models (clinical_records_view)

### Tareas y Subtareas

#### T-04.1.1 — Crear event store table con partitioning

- **ST-04.1.1.1** — Crear migration `001_create_event_store.up.sql`.
  ```sql
  -- Event store (partitioned by time)
  CREATE TABLE IF NOT EXISTS events (
      id uuid DEFAULT gen_random_uuid(),
      aggregate_id uuid NOT NULL,
      aggregate_type text NOT NULL,
      version int NOT NULL,
      event_type text NOT NULL,
      payload jsonb NOT NULL,
      metadata jsonb,
      created_at timestamptz NOT NULL DEFAULT now(),
      position bigserial,
      PRIMARY KEY (id, created_at)
  ) PARTITION BY RANGE (created_at);

  -- Unique constraint for optimistic concurrency
  CREATE UNIQUE INDEX ux_events_aggregate_version 
      ON events(aggregate_id, version, created_at);

  -- Query indexes
  CREATE INDEX idx_events_aggregate 
      ON events(aggregate_id, created_at);
  
  CREATE INDEX idx_events_type_time 
      ON events(aggregate_type, created_at);

  CREATE INDEX idx_events_position 
      ON events(position);

  COMMENT ON TABLE events IS 'Event store for Event Sourcing (clinical records)';
  COMMENT ON COLUMN events.position IS 'Global monotonic position for projections';
  ```
  - **CA:** Tabla `events` creada (partitioned).

- **ST-04.1.1.2** — Crear particiones iniciales (6 meses).
  ```sql
  -- Current month
  CREATE TABLE events_2026_04 PARTITION OF events
      FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');

  -- Next 5 months
  CREATE TABLE events_2026_05 PARTITION OF events
      FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
  
  CREATE TABLE events_2026_06 PARTITION OF events
      FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

  CREATE TABLE events_2026_07 PARTITION OF events
      FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');

  CREATE TABLE events_2026_08 PARTITION OF events
      FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');

  CREATE TABLE events_2026_09 PARTITION OF events
      FOR VALUES FROM ('2026-09-01') TO ('2026-10-01');
  ```
  - **CA:** 6 particiones creadas.

- **ST-04.1.1.3** — Crear función para auto-crear particiones futuras.
  ```sql
  CREATE OR REPLACE FUNCTION create_next_month_partition()
  RETURNS void AS $$
  DECLARE
      partition_date date := date_trunc('month', now() + interval '1 month');
      partition_name text := 'events_' || to_char(partition_date, 'YYYY_MM');
      start_date text := partition_date::text;
      end_date text := (partition_date + interval '1 month')::text;
  BEGIN
      EXECUTE format(
          'CREATE TABLE IF NOT EXISTS %I PARTITION OF events FOR VALUES FROM (%L) TO (%L)',
          partition_name, start_date, end_date
      );
  END;
  $$ LANGUAGE plpgsql;

  COMMENT ON FUNCTION create_next_month_partition IS 'Auto-create next month partition (run monthly via cron)';
  ```
  - **CA:** Función creada.

#### T-04.1.2 — Crear snapshots table

- **ST-04.1.2.1** — Crear migration `002_create_snapshots.up.sql`.
  ```sql
  CREATE TABLE IF NOT EXISTS snapshots (
      aggregate_id uuid PRIMARY KEY,
      aggregate_type text NOT NULL,
      version int NOT NULL,
      state jsonb NOT NULL,
      created_at timestamptz NOT NULL DEFAULT now()
  );

  CREATE INDEX idx_snapshots_type ON snapshots(aggregate_type);

  COMMENT ON TABLE snapshots IS 'Aggregate snapshots for performance (every N events)';
  ```
  - **CA:** Tabla `snapshots` creada.

#### T-04.1.3 — Crear projection read models

- **ST-04.1.3.1** — Crear migration `003_create_projections.up.sql`.
  ```sql
  -- Clinical records view (denormalized read model)
  CREATE TABLE IF NOT EXISTS clinical_records_view (
      patient_id uuid PRIMARY KEY,
      full_name text NOT NULL,
      date_of_birth date NOT NULL,
      medical_record_number text UNIQUE NOT NULL,
      encounter_count int NOT NULL DEFAULT 0,
      last_encounter_date timestamptz,
      chronic_conditions jsonb DEFAULT '[]'::jsonb,
      allergies jsonb DEFAULT '[]'::jsonb,
      current_medications jsonb DEFAULT '[]'::jsonb,
      version int NOT NULL DEFAULT 0,
      updated_at timestamptz NOT NULL DEFAULT now()
  );

  CREATE INDEX idx_clinical_mrn ON clinical_records_view(medical_record_number);
  CREATE INDEX idx_clinical_updated ON clinical_records_view(updated_at);

  -- Encounters table (projection)
  CREATE TABLE IF NOT EXISTS encounters_view (
      id uuid PRIMARY KEY,
      patient_id uuid NOT NULL REFERENCES clinical_records_view(patient_id),
      provider_id uuid NOT NULL,
      encounter_date timestamptz NOT NULL,
      encounter_type text NOT NULL,
      chief_complaint text,
      diagnosis jsonb,
      observations jsonb,
      status text NOT NULL DEFAULT 'in_progress',
      created_at timestamptz NOT NULL
  );

  CREATE INDEX idx_encounters_patient ON encounters_view(patient_id, encounter_date);
  CREATE INDEX idx_encounters_provider ON encounters_view(provider_id, encounter_date);
  ```
  - **CA:** Projection tables creadas.

#### T-04.1.4 — Crear crypto-shredding tables (GDPR)

- **ST-04.1.4.1** — Crear migration `004_create_encryption_keys.up.sql`.
  ```sql
  -- Encryption keys for crypto-shredding (GDPR right to erasure)
  CREATE TABLE IF NOT EXISTS encryption_keys (
      id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
      patient_id uuid NOT NULL UNIQUE,
      key_material bytea NOT NULL,  -- Encrypted with KMS master key
      created_at timestamptz NOT NULL DEFAULT now(),
      deleted_at timestamptz,
      deletion_reason text
  );

  CREATE INDEX idx_encryption_keys_patient ON encryption_keys(patient_id) WHERE deleted_at IS NULL;

  COMMENT ON TABLE encryption_keys IS 'Patient-specific encryption keys for crypto-shredding (GDPR)';
  COMMENT ON COLUMN encryption_keys.deleted_at IS 'Crypto-shredding timestamp (key destroyed)';
  ```
  - **CA:** Crypto-shredding schema creado.

### Criterios de Aceptación de la HU-04.1

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Tabla `events` particionada existe | `\d+ events` muestra partitions |
| 2 | 6 particiones mensuales creadas | `\d events_2026_*` |
| 3 | Unique constraint aggregate versioning | Insert duplicado (same agg_id + version) → error |
| 4 | Tabla `snapshots` existe | `\d snapshots` |
| 5 | Projection tables creadas | `\d clinical_records_view, encounters_view` |
| 6 | Encryption keys table existe | `\d encryption_keys` |

### Definition of Done — HU-04.1

- [ ] 4 migrations ejecutadas exitosamente.
- [ ] Partitioning configurado y funcional.
- [ ] Indexes optimizados creados.
- [ ] Función auto-crear particiones implementada.
- [ ] Schema documentado en README.

---

## HU-04.2 — Command Handlers (CQRS Write Side)

**Como** desarrollador,  
**quiero** implementar command handlers que validen negocio, produzcan eventos, y los persistan con optimistic concurrency,  
**para que** las operaciones de escritura sean consistentes y auditables.

### Contexto técnico

**Commands**:
- `CreatePatientRecord`: Crear expediente clínico inicial
- `OpenEncounter`: Iniciar encuentro clínico
- `RecordObservation`: Registrar observación (signos vitales, etc.)
- `AddDiagnosis`: Agregar diagnóstico
- `CloseEncounter`: Cerrar encuentro

**Events**:
- `PatientRecordCreated`
- `EncounterOpened`
- `ObservationRecorded`
- `DiagnosisAdded`
- `EncounterClosed`

### Tareas y Subtareas

#### T-04.2.1 — Implementar domain aggregates

- **ST-04.2.1.1** — Crear `internal/domain/patient.go`.
  ```go
  package domain

  import (
      "errors"
      "time"
      "github.com/google/uuid"
  )

  type Patient struct {
      ID                  uuid.UUID
      MedicalRecordNumber string
      FullName            string
      DateOfBirth         time.Time
      Encounters          []Encounter
      Version             int
      UncommittedEvents   []Event
  }

  type Encounter struct {
      ID             uuid.UUID
      ProviderID     uuid.UUID
      EncounterDate  time.Time
      Type           string
      ChiefComplaint string
      Observations   []Observation
      Diagnoses      []Diagnosis
      Status         string
  }

  type Observation struct {
      Type      string
      Value     string
      Unit      string
      Timestamp time.Time
  }

  type Diagnosis struct {
      Code        string  // ICD-10
      Description string
      Timestamp   time.Time
  }

  // Command: CreatePatientRecord
  func NewPatient(mrn, fullName string, dob time.Time) (*Patient, error) {
      if mrn == "" || fullName == "" {
          return nil, errors.New("invalid patient data")
      }

      patient := &Patient{
          ID:                  uuid.New(),
          MedicalRecordNumber: mrn,
          FullName:            fullName,
          DateOfBirth:         dob,
          Version:             0,
      }

      event := PatientRecordCreatedEvent{
          EventID:             uuid.New(),
          PatientID:           patient.ID,
          MedicalRecordNumber: mrn,
          FullName:            fullName,
          DateOfBirth:         dob,
          OccurredAt:          time.Now(),
      }

      patient.UncommittedEvents = append(patient.UncommittedEvents, event)
      return patient, nil
  }

  // Command: OpenEncounter
  func (p *Patient) OpenEncounter(providerID uuid.UUID, encounterType, chiefComplaint string) error {
      if providerID == uuid.Nil {
          return errors.New("provider_id required")
      }

      encounter := Encounter{
          ID:             uuid.New(),
          ProviderID:     providerID,
          EncounterDate:  time.Now(),
          Type:           encounterType,
          ChiefComplaint: chiefComplaint,
          Status:         "in_progress",
      }

      p.Encounters = append(p.Encounters, encounter)

      event := EncounterOpenedEvent{
          EventID:        uuid.New(),
          PatientID:      p.ID,
          EncounterID:    encounter.ID,
          ProviderID:     providerID,
          EncounterType:  encounterType,
          ChiefComplaint: chiefComplaint,
          OccurredAt:     time.Now(),
      }

      p.UncommittedEvents = append(p.UncommittedEvents, event)
      return nil
  }

  // Command: RecordObservation
  func (p *Patient) RecordObservation(encounterID uuid.UUID, obsType, value, unit string) error {
      // Find encounter
      for i, enc := range p.Encounters {
          if enc.ID == encounterID {
              if enc.Status != "in_progress" {
                  return errors.New("encounter is not in progress")
              }

              obs := Observation{
                  Type:      obsType,
                  Value:     value,
                  Unit:      unit,
                  Timestamp: time.Now(),
              }

              p.Encounters[i].Observations = append(p.Encounters[i].Observations, obs)

              event := ObservationRecordedEvent{
                  EventID:     uuid.New(),
                  PatientID:   p.ID,
                  EncounterID: encounterID,
                  Type:        obsType,
                  Value:       value,
                  Unit:        unit,
                  OccurredAt:  time.Now(),
              }

              p.UncommittedEvents = append(p.UncommittedEvents, event)
              return nil
          }
      }

      return errors.New("encounter not found")
  }
  ```
  - **CA:** Domain aggregate Patient implementado.

#### T-04.2.2 — Implementar event store repository

- **ST-04.2.2.1** — Crear `internal/repository/eventstore.go`.
  ```go
  package repository

  import (
      "context"
      "encoding/json"
      "errors"
      "github.com/jackc/pgx/v5/pgxpool"
      "github.com/google/uuid"
      "github.com/serenidad/clinical/internal/domain"
  )

  type EventStore struct {
      pool *pgxpool.Pool
  }

  func NewEventStore(pool *pgxpool.Pool) *EventStore {
      return &EventStore{pool: pool}
  }

  // Load aggregate from events
  func (es *EventStore) LoadAggregate(ctx context.Context, aggregateID uuid.UUID) (*domain.Patient, error) {
      // Check for snapshot first
      var snapshotVersion int
      var snapshotState []byte
      err := es.pool.QueryRow(ctx, `
          SELECT version, state FROM snapshots WHERE aggregate_id = $1
      `, aggregateID).Scan(&snapshotVersion, &snapshotState)

      var patient *domain.Patient
      if err == nil {
          // Load from snapshot
          patient = &domain.Patient{}
          json.Unmarshal(snapshotState, patient)
      } else {
          patient = &domain.Patient{ID: aggregateID, Version: 0}
      }

      // Load events after snapshot version
      rows, err := es.pool.Query(ctx, `
          SELECT event_type, payload FROM events
          WHERE aggregate_id = $1 AND version > $2
          ORDER BY version
      `, aggregateID, patient.Version)

      if err != nil {
          return nil, err
      }
      defer rows.Close()

      for rows.Next() {
          var eventType string
          var payload []byte
          rows.Scan(&eventType, &payload)

          // Apply event to aggregate
          // (omitted for brevity: unmarshal event and apply to patient state)
      }

      return patient, nil
  }

  // Save events with optimistic concurrency
  func (es *EventStore) SaveEvents(ctx context.Context, aggregateID uuid.UUID, expectedVersion int, events []domain.Event) error {
      tx, err := es.pool.Begin(ctx)
      if err != nil {
          return err
      }
      defer tx.Rollback(ctx)

      // Optimistic concurrency check
      var currentVersion int
      err = tx.QueryRow(ctx, `
          SELECT COALESCE(MAX(version), 0) FROM events WHERE aggregate_id = $1
      `, aggregateID).Scan(&currentVersion)

      if err != nil {
          return err
      }

      if currentVersion != expectedVersion {
          return errors.New("concurrency conflict: aggregate version mismatch")
      }

      // Append events
      for i, event := range events {
          version := expectedVersion + i + 1
          payload, _ := json.Marshal(event)

          _, err := tx.Exec(ctx, `
              INSERT INTO events (aggregate_id, aggregate_type, version, event_type, payload, metadata)
              VALUES ($1, $2, $3, $4, $5, $6)
          `, aggregateID, "Patient", version, event.Type(), payload, nil)

          if err != nil {
              return err
          }

          // Insert into outbox (same transaction)
          tx.Exec(ctx, `
              INSERT INTO outbox (aggregate_type, aggregate_id, event_type, payload)
              VALUES ($1, $2, $3, $4)
          `, "clinical", aggregateID, event.Type(), payload)
      }

      return tx.Commit(ctx)
  }
  ```
  - **CA:** EventStore repository implementado.

#### T-04.2.3 — Implementar command handlers

- **ST-04.2.3.1** — Crear `internal/handlers/commands.go`.
  ```go
  package handlers

  import (
      "context"
      "github.com/google/uuid"
      "github.com/serenidad/clinical/internal/domain"
      "github.com/serenidad/clinical/internal/repository"
  )

  type CommandHandler struct {
      eventStore *repository.EventStore
  }

  func NewCommandHandler(es *repository.EventStore) *CommandHandler {
      return &CommandHandler{eventStore: es}
  }

  func (h *CommandHandler) CreatePatientRecord(ctx context.Context, mrn, fullName string, dob time.Time) (*domain.Patient, error) {
      patient, err := domain.NewPatient(mrn, fullName, dob)
      if err != nil {
          return nil, err
      }

      err = h.eventStore.SaveEvents(ctx, patient.ID, 0, patient.UncommittedEvents)
      if err != nil {
          return nil, err
      }

      return patient, nil
  }

  func (h *CommandHandler) OpenEncounter(ctx context.Context, patientID, providerID uuid.UUID, encType, complaint string) error {
      patient, err := h.eventStore.LoadAggregate(ctx, patientID)
      if err != nil {
          return err
      }

      if err := patient.OpenEncounter(providerID, encType, complaint); err != nil {
          return err
      }

      return h.eventStore.SaveEvents(ctx, patient.ID, patient.Version, patient.UncommittedEvents)
  }
  ```
  - **CA:** Command handlers implementados.

### Criterios de Aceptación de la HU-04.2

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | CreatePatientRecord genera evento | Query `events` muestra PatientRecordCreated |
| 2 | Optimistic concurrency funciona | Concurrent writes mismo aggregate → error |
| 3 | Events persisten en partition correcta | Insert evento → va a partition del mes actual |
| 4 | Outbox recibe eventos (misma TX) | Query `outbox` post-command |
| 5 | LoadAggregate rehidrata estado | Load → patient state correcto |

### Definition of Done — HU-04.2

- [ ] Domain aggregates implementados.
- [ ] EventStore repository funcional.
- [ ] Command handlers testeados (unit + integration).
- [ ] Optimistic concurrency validado con tests.

---

## HU-04.3 — Snapshot Strategy

**Como** ingeniero de performance,  
**quiero** implementar snapshots automáticos cada 50 eventos,  
**para que** la rehydration de aggregates sea rápida incluso con miles de eventos.

### Tareas y Subtareas

#### T-04.3.1 — Implementar snapshot creation

- **ST-04.3.1.1** — Crear `internal/repository/snapshots.go`.
  ```go
  func (es *EventStore) CreateSnapshot(ctx context.Context, aggregate *domain.Patient) error {
      state, _ := json.Marshal(aggregate)

      _, err := es.pool.Exec(ctx, `
          INSERT INTO snapshots (aggregate_id, aggregate_type, version, state)
          VALUES ($1, $2, $3, $4)
          ON CONFLICT (aggregate_id) DO UPDATE
          SET version = EXCLUDED.version, state = EXCLUDED.state, created_at = now()
      `, aggregate.ID, "Patient", aggregate.Version, state)

      return err
  }
  ```
  - **CA:** CreateSnapshot implementado.

- **ST-04.3.1.2** — Modificar SaveEvents para auto-snapshot cada 50 eventos.
  ```go
  func (es *EventStore) SaveEvents(...) error {
      // ... existing code ...
      tx.Commit(ctx)

      // Auto-snapshot every 50 events
      newVersion := expectedVersion + len(events)
      if newVersion % 50 == 0 {
          aggregate, _ := es.LoadAggregate(ctx, aggregateID)
          es.CreateSnapshot(ctx, aggregate)
      }

      return nil
  }
  ```
  - **CA:** Auto-snapshot configurado.

### Criterios de Aceptación de la HU-04.3

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Snapshot se crea cada 50 eventos | Insert 50 eventos → snapshot exists |
| 2 | LoadAggregate usa snapshot | Load con 100 eventos → solo 50 eventos replayed |
| 3 | Snapshot actualiza estado correcto | Snapshot state refleja eventos aplicados |

### Definition of Done — HU-04.3

- [ ] Snapshot logic implementado.
- [ ] Auto-snapshot cada 50 eventos funcional.
- [ ] Tests validando performance improvement.

---

## HU-04.4 — Projection Workers (CQRS Read Side)

**Como** consumidor de APIs,  
**quiero** consultar datos clínicos vía read models optimizados,  
**para que** las queries sean rápidas sin replay de eventos en tiempo real.

### Tareas y Subtareas

#### T-04.4.1 — Implementar projection worker

- **ST-04.4.1.1** — Crear `internal/projections/clinical_view.go`.
  ```go
  package projections

  import (
      "context"
      "encoding/json"
      "time"
      "github.com/jackc/pgx/v5/pgxpool"
  )

  type ClinicalViewProjector struct {
      pool *pgxpool.Pool
  }

  func (p *ClinicalViewProjector) Run(ctx context.Context) error {
      var lastPosition int64

      for {
          select {
          case <-ctx.Done():
              return ctx.Err()
          default:
              p.processEvents(ctx, &lastPosition)
              time.Sleep(100 * time.Millisecond)
          }
      }
  }

  func (p *ClinicalViewProjector) processEvents(ctx context.Context, lastPos *int64) error {
      rows, _ := p.pool.Query(ctx, `
          SELECT position, aggregate_id, event_type, payload
          FROM events
          WHERE position > $1 AND aggregate_type = 'Patient'
          ORDER BY position
          LIMIT 100
      `, *lastPos)
      defer rows.Close()

      for rows.Next() {
          var pos int64
          var aggID uuid.UUID
          var eventType string
          var payload []byte

          rows.Scan(&pos, &aggID, &eventType, &payload)

          // Handle event
          switch eventType {
          case "PatientRecordCreated":
              var evt PatientRecordCreatedEvent
              json.Unmarshal(payload, &evt)
              p.handlePatientCreated(ctx, evt)
          case "EncounterOpened":
              var evt EncounterOpenedEvent
              json.Unmarshal(payload, &evt)
              p.handleEncounterOpened(ctx, evt)
          }

          *lastPos = pos
      }

      return nil
  }

  func (p *ClinicalViewProjector) handlePatientCreated(ctx context.Context, evt PatientRecordCreatedEvent) {
      p.pool.Exec(ctx, `
          INSERT INTO clinical_records_view (patient_id, full_name, date_of_birth, medical_record_number)
          VALUES ($1, $2, $3, $4)
          ON CONFLICT (patient_id) DO NOTHING
      `, evt.PatientID, evt.FullName, evt.DateOfBirth, evt.MedicalRecordNumber)
  }
  ```
  - **CA:** Projection worker implementado.

### Criterios de Aceptación de la HU-04.4

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Projection worker procesa eventos | Worker logs muestra event processing |
| 2 | clinical_records_view actualizado | Insert evento → view refleja cambio en <1s |
| 3 | Lag monitoring funcional | Métrica `projection_lag_seconds` exportada |

### Definition of Done — HU-04.4

- [ ] Projection worker implementado.
- [ ] Read models actualizándose en tiempo real.
- [ ] Lag metrics exportadas a Prometheus.

---

## Resumen de Entregables EP-04

**Database**:
- Event store particionado (RANGE mensual)
- Snapshots table
- Projection read models
- Crypto-shredding schema (GDPR)

**Código**:
- Domain aggregates (Patient, Encounter)
- Command handlers (CQRS write side)
- EventStore repository (optimistic concurrency)
- Snapshot strategy (auto cada 50 eventos)
- Projection workers (CQRS read side)

**Kubernetes**:
- Clinical service deployment
- Projection worker deployment
- ServiceMonitor Prometheus

---

**Tiempo estimado EP-04**: 4 semanas (Sprint 4-5)  
**Esfuerzo**: ~120 horas ingeniería + 40 horas testing
