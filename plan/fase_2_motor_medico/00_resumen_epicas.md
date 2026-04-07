# Fase 2 — Plan Ágil Motor Médico: Resumen de Épicas

**Proyecto:** serenidad — Clínica Digital de Salud Mental Global  
**Sprint Goal:** Event bus funcional (NATS JetStream), servicios core (Scheduling + Clinical Records) con Event Sourcing, Outbox pattern implementado, y landing page pública.  
**Fuente:** `target_arch/descripcion.md` + `research/08_evaluacion_tecnologica_fase2_motor_medico.md`  
**Fecha:** Abril 2026

---

## Inventario de Épicas

| ID | Épica | Componentes | HUs | Prioridad | Sprint Sugerido |
|----|-------|-------------|-----|-----------|-----------------|
| EP-01 | Event Bus — NATS JetStream | NATS Helm, 4 Streams, Monitoring | 3 | Crítica | S1 |
| EP-02 | Outbox Worker Transversal | Schema, Relay Worker, DLQ | 2 | Crítica | S2 |
| EP-03 | Scheduling Service | Go Service, Commands, API | 3 | Crítica | S3 |
| EP-04 | Clinical Record Service | Event Store, CQRS, Projections | 4 | Crítica | S4–S5 |
| EP-05 | Landing Page Astro | SSG, SEO, CF Pages Deploy | 2 | Alta | S6 |

**Total: 5 Épicas · 14 Historias de Usuario · ~80 Tareas · ~220 Subtareas**

---

## Diagrama de Dependencias entre Épicas

```
Fase 1 completa (IAM + BFF + Qwik SPA + PostgreSQL + FluxCD)
  └──► EP-01 NATS JetStream
        ├──► EP-02 Outbox Worker (requiere NATS)
        │     └──► EP-03 Scheduling Service (requiere Outbox + IAM)
        │          └──► EP-04 Clinical Record Service (requiere Scheduling patterns)
        └──► EP-05 Landing Astro (paralelo, NO requiere backend)
```

### Grafo de Dependencias Detallado

```
[Fase 1: IAM Service + PostgreSQL + NATS prereqs]
  │
  ├─► [EP-01] NATS JetStream Cluster
  │     ├─► HU-01.1: Deploy NATS Helm chart (3-node StatefulSet)
  │     ├─► HU-01.2: Crear 4 Streams (IAM, Scheduling, Clinical, Billing)
  │     └─► HU-01.3: Monitoring + Alertas Prometheus
  │           │
  │           └─► [EP-02] Outbox Worker
  │                 ├─► HU-02.1: Schema outbox + indexes
  │                 └─► HU-02.2: Relay worker (polling + NATS publish)
  │                       │
  │                       ├─► [EP-03] Scheduling Service
  │                       │     ├─► HU-03.1: Go scaffold (Chi + pgxpool)
  │                       │     ├─► HU-03.2: Commands (Create/Cancel Appointment)
  │                       │     └─► HU-03.3: Kubernetes deployment
  │                       │           │
  │                       │           └─► [EP-04] Clinical Record Service
  │                       │                 ├─► HU-04.1: Event store schema
  │                       │                 ├─► HU-04.2: Command handlers (CQRS)
  │                       │                 ├─► HU-04.3: Snapshot strategy
  │                       │                 └─► HU-04.4: Projection workers
  │                       │
  │                       └─► (patterns reusables para Clinical)
  │
  └─► [EP-05] Landing Astro (independiente)
        ├─► HU-05.1: Astro v6 SSG project
        └─► HU-05.2: CF Pages deployment + CI/CD
```

---

## Definition of Done (DoD) Global — Fase 2

Cada Historia de Usuario se considera **Done** cuando:

1. **Código/Config commiteado** en la rama `main` del monorepo.
2. **Tests automatizados** pasando (unit + integration donde aplique).
3. **FluxCD reconcilia** sin errores (`flux get all` → todos Ready) para componentes k8s.
4. **Eventos publicados a NATS** correctamente (validado con `nats sub`).
5. **Métricas expuestas** en `/metrics` endpoint (Prometheus).
6. **Documentación** de API/runbook actualizada.
7. **Sin regresiones** en componentes de Fase 1.

## Criterios de Aceptación Globales de Fase 2

La Fase 2 está **completa** cuando se cumple el flujo end-to-end:

```
FLUJO SCHEDULING:
1. Paciente autenticado (JWT desde Fase 1) llama POST /v1/appointments
2. Scheduling Service valida disponibilidad y crea cita
3. Evento AppointmentCreated se escribe a outbox (misma TX)
4. Outbox worker publica a NATS stream "scheduling.AppointmentCreated"
5. Clinical Service consume evento y actualiza vista de citas del paciente
6. Frontend (Qwik SPA) obtiene cita vía BFF en < 2 segundos
7. Auditoría completa en event store (quién, cuándo, qué)

FLUJO CLINICAL:
1. Provider autenticado registra observación clínica (POST /v1/encounters/{id}/observations)
2. Clinical Service aplica comando, valida negocio, genera eventos
3. Eventos se persisten en event store (aggregate versioning)
4. Eventos se publican vía outbox → NATS
5. Projection worker actualiza read model (clinical_records_view)
6. Provider consulta historial completo vía GET /v1/patients/{id}/records
8. Todo el flujo completa en < 3 segundos

FLUJO LANDING:
1. Visitante anónimo accede https://sereni.dad
2. Homepage carga en < 2s (LCP < 2.0s)
3. Core Web Vitals: LCP < 2.5s, CLS < 0.1, INP < 200ms
4. SEO: structured data validado por Google Rich Results Test
5. Formulario contacto envía a backend seguro (NO PHI en landing)
```

---

## Stack Tecnológico Fase 2 (Resumen)

| Componente | Tecnología | Versión | Notas |
|-----------|------------|---------|-------|
| **Event Bus** | NATS JetStream | v2.11.14 | 3-node cluster, R3 streams |
| **Microservices** | Go + Chi + pgxpool | Go 1.25.x, Chi v5, pgx v5 | Connection pooling, graceful shutdown |
| **Messaging Pattern** | Transactional Outbox | PostgreSQL + polling | SKIP LOCKED, at-least-once delivery |
| **Clinical Records** | Event Sourcing + CQRS | PostgreSQL partitioned | Snapshots, projections, crypto-shredding |
| **Landing Page** | Astro v6 SSG | latest | Cloudflare Pages, Core Web Vitals |
| **Observability** | Prometheus + Grafana | v3.x / latest | Metrics, dashboards, alerts |

---

## Archivos del Plan

| Archivo | Contenido |
|---------|-----------|
| `00_resumen_epicas.md` | Este documento — visión general Fase 2 |
| `01_EP01_nats_jetstream.md` | Épica 1: Event bus NATS JetStream |
| `02_EP02_outbox_worker.md` | Épica 2: Outbox pattern transaccional |
| `03_EP03_scheduling_service.md` | Épica 3: Servicio de agendamiento de citas |
| `04_EP04_clinical_record_service.md` | Épica 4: Servicio de registros clínicos (Event Sourcing) |
| `05_EP05_landing_astro.md` | Épica 5: Landing page pública Astro v6 |

---

## Diferencias clave vs Fase 1

| Aspecto | Fase 1 | Fase 2 |
|---------|--------|--------|
| **Arquitectura** | CRUD tradicional (IAM) | Event-driven (NATS + Outbox) |
| **Persistencia** | Relacional normalizada | Event Sourcing + CQRS |
| **Messaging** | Síncrono (HTTP) | Asíncrono (NATS JetStream) |
| **Consistency** | Strong (ACID) | Eventual (at-least-once) |
| **Audit** | Application logs | Event store immutable |
| **Frontend** | SPA (Qwik) | SPA + Landing SSG (Astro) |

---

## Roadmap Visual (Sprints)

```
Sprint 1 (2 semanas):  EP-01 — NATS JetStream deployment
  ✓ 3-node cluster con Helm
  ✓ 4 streams configurados
  ✓ Monitoring Prometheus

Sprint 2 (2 semanas):  EP-02 — Outbox Worker
  ✓ Schema + indexes
  ✓ Relay worker funcionando
  ✓ Integration tests (retry + DLQ)

Sprint 3 (2 semanas):  EP-03 — Scheduling Service
  ✓ Go service scaffold
  ✓ Commands: Create/Cancel Appointment
  ✓ K8s deployment + probes

Sprint 4-5 (4 semanas): EP-04 — Clinical Record Service
  ✓ Event store + partitioning
  ✓ Command handlers (CQRS)
  ✓ Snapshots + projections
  ✓ GDPR crypto-shredding

Sprint 6 (2 semanas):  EP-05 — Landing Astro
  ✓ Homepage + servicios pages
  ✓ SEO optimization
  ✓ CF Pages deployment

Total: 12 semanas (~3 meses)
```

---

## Métricas de Éxito Fase 2

| Métrica | Target | Herramienta |
|---------|--------|-------------|
| **NATS throughput** | > 1,000 msg/s sustained | `nats bench` |
| **Outbox lag p95** | < 1 segundo | Prometheus histogram |
| **Event store write p95** | < 50 ms | Custom metrics |
| **API latency p95** | < 200 ms | Prometheus histogram |
| **Landing LCP** | < 2.0 segundos | Lighthouse CI |
| **Service uptime** | 99.9% (43 min/mes downtime) | Prometheus + alerting |
| **Event replay time** | < 5 min para 100K eventos | Manual testing |

---

## Riesgos y Mitigaciones

| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|--------------|---------|------------|
| **NATS cluster quorum loss** | Media | Alto | PodDisruptionBudget minAvailable=2, backups |
| **Outbox table bloat** | Alta | Medio | Autovacuum tuning, partition by time |
| **Event store query performance** | Media | Alto | Snapshots cada 50 eventos, projections |
| **GDPR compliance (crypto-shredding)** | Baja | Crítico | Design review + legal validation |
| **NATS message loss** | Baja | Alto | R3 streams + file storage + backups |
| **Outbox worker crash** | Media | Medio | Kubernetes restarts, idempotency keys |

---

## Próximos Pasos Post-Fase 2

**Fase 3 — Plataforma Completa**:
- OpenFGA v1.x (AuthZ Zanzibar)
- Billing & Ops Service
- kube-prometheus-stack + Loki + Tempo
- Alertmanager → Telegram

**Fase 4+ — Evolución**:
- Multi-node cluster (add CX32 workers)
- EHRbase (openEHR server)
- DIDs (Decentralized Identifiers)
- Multi-región (HA geográfica)
