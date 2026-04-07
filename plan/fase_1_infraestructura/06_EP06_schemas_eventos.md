# EP-06 — Schemas de Eventos Protobuf

**Épica:** Como equipo de desarrollo, necesitamos los schemas Protobuf definidos, validados con linting, y con código Go generado para los eventos de dominio de Fase 1, para que todos los microservicios compartan contratos estrictos e inmutables para la comunicación asíncrona via Outbox → NATS JetStream.

**Origen:** Sección §17 (V-12) del documento de decisión.
**Prioridad:** Alta — Prerequisito para el Outbox del IAM Domain Service.
**Sprint:** S2
**Dependencias Entrantes:** EP-03 HU-03.3 (SOPS configurado, repo GitOps preparado para commits).
**Dependencias Salientes:** EP-07 (IAM Domain Service usa tipos Protobuf para serializar eventos en Outbox).

---

## HU-06.1 — Schemas Protobuf + buf.build CLI (V-12)

**Como** desarrollador backend,
**quiero** tener los schemas Protobuf de eventos de dominio definidos para IAM, Scheduling y Clinical, con código Go generado y compilando,
**para que** el IAM Domain Service pueda serializar eventos `UserRegistered` y `DoctorOnboarded` en la tabla outbox usando tipos Protobuf fuertemente tipados.

**Dependencia:** EP-01 HU-01.2 (buf CLI instalado).

### Tareas y Subtareas

#### T-06.1.1 — Inicializar workspace de buf

- **ST-06.1.1.1** — Navegar a `packages/events/` y ejecutar `buf config init`.
  - **CA:** Archivo `buf.yaml` generado en `packages/events/`.
- **ST-06.1.1.2** — Editar `buf.yaml` con configuración del proyecto.
  - **CA:** Contenido:
    - `version: v2`.
    - `modules[0].path: proto`.
    - `modules[0].name: buf.build/serenidad/events`.
    - `lint.use: [STANDARD]`.
    - `breaking.use: [FILE]`.
  - **CA:** Archivo válido y parseable por `buf`.
- **ST-06.1.1.3** — Crear `buf.gen.yaml` para generación de código Go.
  - **CA:** Contenido:
    - `version: v2`.
    - `managed.enabled: true`.
    - `managed.override[0].file_option: go_package_prefix` con valor `gitlab.com/serenidad/serenidad-platform/packages/events/gen`.
    - Plugin `buf.build/protocolbuffers/go` → `out: gen/go`, `opt: paths=source_relative`.
    - Plugin `buf.build/grpc/go` → `out: gen/go`, `opt: paths=source_relative, require_unimplemented_servers=false`.
  - **CA:** Archivo válido y parseable por `buf`.

#### T-06.1.2 — Definir schema de eventos IAM v1

- **ST-06.1.2.1** — Crear directorio `packages/events/proto/iam/v1/`.
  - **CA:** Directorio existe.
- **ST-06.1.2.2** — Crear archivo `packages/events/proto/iam/v1/events.proto`.
  - **CA:** `syntax = "proto3"`, `package iam.v1`.
  - **CA:** `go_package` apunta a `gitlab.com/serenidad/serenidad-platform/packages/events/gen/iam/v1;iamv1`.
  - **CA:** Import de `google/protobuf/timestamp.proto`.
  - **CA:** Mensaje `UserRegistered` con campos:
    - `string event_id = 1` (UUIDv7).
    - `string user_id = 2` (UUIDv7).
    - `string tenant_id = 3` (UUIDv7).
    - `string email = 4` (hash SHA-256).
    - `string role = 5` (patient|doctor|admin).
    - `string did = 6` (DID web).
    - `google.protobuf.Timestamp occurred_at = 7`.
    - `google.protobuf.Timestamp emitted_at = 8`.
  - **CA:** Mensaje `DoctorOnboarded` con campos:
    - `string event_id = 1`.
    - `string doctor_id = 2` (UUIDv7).
    - `string tenant_id = 3` (UUIDv7).
    - `string specialty = 4` (psychiatry|psychology|general).
    - `string license_number = 5`.
    - `string country_code = 6` (ISO 3166-1 alpha-2).
    - `google.protobuf.Timestamp occurred_at = 7`.
    - `google.protobuf.Timestamp emitted_at = 8`.

#### T-06.1.3 — Definir schema de eventos Scheduling v1

- **ST-06.1.3.1** — Crear directorio `packages/events/proto/scheduling/v1/`.
  - **CA:** Directorio existe.
- **ST-06.1.3.2** — Crear archivo `packages/events/proto/scheduling/v1/events.proto`.
  - **CA:** `syntax = "proto3"`, `package scheduling.v1`.
  - **CA:** `go_package` correcto.
  - **CA:** Mensaje `AppointmentBooked` con 11 campos: event_id, appointment_id, patient_id, doctor_id, tenant_id, starts_at, ends_at, modality, fhir_resource_json, occurred_at, emitted_at.
  - **CA:** Mensaje `AppointmentCancelled` con 6 campos: event_id, appointment_id, cancelled_by_id, reason, occurred_at, emitted_at.

#### T-06.1.4 — Definir schema de eventos Clinical v1

- **ST-06.1.4.1** — Crear directorio `packages/events/proto/clinical/v1/`.
  - **CA:** Directorio existe.
- **ST-06.1.4.2** — Crear archivo `packages/events/proto/clinical/v1/events.proto`.
  - **CA:** `syntax = "proto3"`, `package clinical.v1`.
  - **CA:** `go_package` correcto.
  - **CA:** Mensaje `ConsultationFinished` con 10 campos: event_id, consultation_id, appointment_id, patient_id, doctor_id, tenant_id, openehr_json, duration_minutes, occurred_at, emitted_at.
  - **CA:** Mensaje `DiagnosisRecorded` con 10 campos: event_id, diagnosis_id, clinical_record_id, patient_id, doctor_id, tenant_id, icd10_code, description, occurred_at, emitted_at.

#### T-06.1.5 — Lint y validación de schemas

- **ST-06.1.5.1** — Ejecutar `buf lint` en `packages/events/`.
  - **CA:** `buf lint` completa sin errores (sin output = sin errores).
- **ST-06.1.5.2** — Verificar que todos los field numbers son únicos y secuenciales dentro de cada mensaje.
  - **CA:** Revisión manual o automatizada de field numbers.
- **ST-06.1.5.3** — Verificar que `go_package` es correcto en cada archivo .proto.
  - **CA:** Cada archivo tiene un `go_package` único y válido.

#### T-06.1.6 — Generar código Go desde schemas

- **ST-06.1.6.1** — Ejecutar `buf generate` en `packages/events/`.
  - **CA:** Se generan archivos `*.pb.go` en:
    - `gen/go/iam/v1/`.
    - `gen/go/scheduling/v1/`.
    - `gen/go/clinical/v1/`.
- **ST-06.1.6.2** — Inicializar módulo Go en `gen/go/`.
  - **CA:** `go mod init gitlab.com/serenidad/serenidad-platform/packages/events/gen`.
  - **CA:** `go mod tidy` completa sin errores.
- **ST-06.1.6.3** — Verificar que el código generado compila.
  - **CA:** `go build ./...` en `gen/go/` completa sin errores.

#### T-06.1.7 — Verificar breaking changes (baseline)

- **ST-06.1.7.1** — Ejecutar `buf breaking` contra la rama main.
  - **CA:** En el primer commit no hay baseline de comparación, por lo que el comando no reporta breaking changes.
  - **CA:** Para commits futuros, breaking changes serán detectados.

#### T-06.1.8 — Commitear schemas y código generado

- **ST-06.1.8.1** — Commitear todo el directorio `packages/events/`.
  - **CA:** `git add packages/events/`.
  - **CA:** `git log --oneline -1` → `feat: add protobuf event schemas for IAM, Scheduling, and Clinical domains (Phase 1)`.
  - **CA:** Push a `main` exitoso.

### Criterios de Aceptación de la HU-06.1

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-1 | 3 archivos .proto definidos | `find packages/events/proto -name "*.proto"` → 3 archivos |
| CA-2 | 6 mensajes Protobuf definidos | UserRegistered, DoctorOnboarded, AppointmentBooked, AppointmentCancelled, ConsultationFinished, DiagnosisRecorded |
| CA-3 | buf lint sin errores | `buf lint` → sin output |
| CA-4 | Código Go generado | `find packages/events/gen -name "*.pb.go"` → archivos generados |
| CA-5 | Código Go compila | `go build ./...` en `gen/go/` → sin errores |
| CA-6 | buf.yaml y buf.gen.yaml válidos | `buf` los parsea correctamente |
| CA-7 | Schemas commiteados en main | `git log --oneline -1` → commit de schemas |

### Definition of Done — HU-06.1

- [ ] 3 archivos .proto con 6 mensajes de eventos de dominio.
- [ ] `buf lint` sin errores.
- [ ] Código Go generado y compilando.
- [ ] `buf.yaml` y `buf.gen.yaml` configurados.
- [ ] Schemas y código generado commiteados en `main`.
- [ ] Baseline establecido para detección de breaking changes futuros.

---

## Resumen de Dependencias Internas EP-06

```
T-06.1.1 (buf init) ──► T-06.1.2, T-06.1.3, T-06.1.4 (schemas — paralelos)
T-06.1.2, T-06.1.3, T-06.1.4 ──► T-06.1.5 (lint)
T-06.1.5 ──► T-06.1.6 (generar código Go)
T-06.1.6 ──► T-06.1.7 (breaking changes baseline)
T-06.1.7 ──► T-06.1.8 (commitear)

Dependencia externa:
  T-06.1.6 (código Go generado) ──► EP-07 T-07.2.x (IAM Service usa tipos Protobuf en Outbox)
```
