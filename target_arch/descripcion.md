# Documento de Arquitectura y Visión Maestra

| Campo | Valor |
|-------|-------|
| **Proyecto** | Serenidad — Clínica Digital de Salud Mental |
| **Autor** | djca |
| **Estado** | Definición Estratégica & Diseño de Alto Nivel — **VERSIÓN DEFINITIVA** |
| **Última revisión** | Abril 2026 — Challenger v3.0 + Versiones Actualizadas |

---

## Tabla de Contenidos

1. [Visión y Propósito](#1-visión-y-propósito)
2. [Restricciones de Diseño Intencionales](#2-restricciones-de-diseño-intencionales)
3. [Stack Tecnológico Definitivo](#3-stack-tecnológico-definitivo)
4. [Tabla de Versiones Pinneadas](#4-tabla-de-versiones-pinneadas)
5. [Patrones Arquitectónicos Invariables](#5-patrones-arquitectónicos-invariables)
6. [Decisiones de Diseño — ADRs](#6-decisiones-de-diseño--adrs)
7. [Roadmap de Construcción](#7-roadmap-de-construcción)
8. [Referencias Técnicas](#8-referencias-técnicas)
9. [Orden Final de Implementación](#9-orden-final-de-implementación)

---

## 1. Visión y Propósito

Construcción de una **clínica digital de salud mental con alcance global**, diseñada para iniciar operaciones con un único profesional médico e iterar hacia una plataforma multiespecialidad.

El propósito central no es una salida rápida al mercado, sino la consolidación de un **sistema informático médico que perdure por generaciones**, protegiendo la integridad de la historia clínica, garantizando la privacidad absoluta del paciente y ofreciendo ultra-velocidad en la experiencia del usuario.

---

## 2. Restricciones de Diseño Intencionales

Las siguientes reglas son **inquebrantables** en toda decisión de ingeniería:

| # | Restricción | Descripción |
|---|-------------|-------------|
| **R1** | No MVP — Build It Right The First Time | Cero deuda técnica asumida. Estándares definitivos desde el día cero. |
| **R2** | Inmutabilidad Arquitectónica | La arquitectura debe escalar de 1 a 10,000 médicos sin refactorizaciones estructurales. |
| **R3** | Desacoplamiento Absoluto | Lógica de negocio aislada de infraestructura, base de datos y frameworks de UI. |
| **R4** | Soberanía Tecnológica | Open source y self-hosted prioritariamente. Sin dependencia de terceros en el path crítico. |
| **R5** | Costo Mínimo de Propiedad | Software open source + self-hosted. TCO < $35/mes en fases iniciales. |
| **R6** | Ultravelocidad | TTI < 100ms en frontend. Latencia P99 < 50ms en APIs. |
| **R7** | No Reinventar la Rueda | Usar librerías y herramientas probadas de la industria. El equipo conecta, no construye herramientas. |

---

## 3. Stack Tecnológico Definitivo

### 3.1 Infraestructura Base

| Componente | Tecnología | Versión | Notas |
|-----------|------------|---------|-------|
| VPS | Hetzner Cloud CX32 | — | 4 vCPU AMD, **8 GB RAM**, 80 GB SSD ≈ €6.80/mes. Reemplaza CX31. |
| SO | **Talos Linux** | **v1.10.x** | OS inmutable, API-driven, zero SSH. Diseñado exclusivamente para Kubernetes. |
| Orquestación | **Kubernetes** | **1.33.x** (via Talos) | Definitivo desde Fase 1 hasta Fase 4+. No Docker Compose en producción. |
| GitOps / Deploy | **FluxCD** | **v2.x** | CNCF graduated. Todo el estado de producción declarado en Git. |
| Ingress + TLS | **cert-manager + Traefik** | cert-manager **v1.x** / Traefik **v3.x** | cert-manager CNCF graduated. Traefik ForwardAuth = equivalente Caddy forward_auth. |
| Package Manager k8s | **Helm** | **v3.x** | Todos los servicios se despliegan via Helm charts. |
| Secret Management | **Sealed Secrets** | **v0.27+** | SealedSecrets cifrados commiteables en Git (Bitnami/Helm). |
| Backups DB | **CloudNativePG WAL + barman-cloud** | — | WAL continuo → Backblaze B2 (S3-compatible). PITR disponible. |
| Container Registry | **ghcr.io** (GitHub Container Registry) | — | OCI compatible. Imágenes Go firmadas. Free tier para repos públicos/privados. |

### 3.2 Capa Frontend

| Componente | Tecnología | Versión | Notas |
|-----------|------------|---------|-------|
| Portal médico / paciente | Qwik | **v2.0** | Resumability, TTI ~50ms. Contiene breaking changes vs v1.x |
| Landing page + SEO | Astro | **v6.x** | Lighthouse 99.2, zero JS por defecto. v6 requerida para Qwik v2 |
| Language | TypeScript | **6.0** | Released March 2026. Última versión JS-based antes de 7.0 nativo en Go |
| Deploy | Cloudflare Pages | — | Tier gratuito: 500 builds/mes, bandwidth ilimitado |

> **Nota TypeScript 6.0:** Breaking changes respecto a 5.x incluyen: cambios en default settings, actualización de tipos DOM, mejoras de inferencia, y deprecaciones. Por ser la versión más reciente disponible, se usa en todo el código TypeScript del proyecto (frontend + BFF).

### 3.3 API Gateway / BFF

| Componente | Tecnología | Versión | Notas |
|-----------|------------|---------|-------|
| BFF Edge | Hono.js | **v4.x** | Cloudflare Workers, 100K req/día gratis |
| Runtime del BFF | Bun | **1.3.x** | Usado solo en dev local para el BFF; en CF Workers usa el runtime de CF |
| Routing interno VPS | Caddy `forward_auth` | — | Sin proceso adicional |

### 3.4 IAM — Identity & Access Management

| Componente | Tecnología | Versión | Notas |
|-----------|------------|---------|-------|
| AuthN | Ory Kratos | **v1.3.1** | Passkeys/WebAuthn, sesiones, MFA, flows self-service |
| AuthZ | OpenFGA | **v1.x (latest)** | Motor Zanzibar. Reemplaza Ory Keto. Backing: Okta/Auth0 |
| IAM Domain Service | Go + Chi + pgxpool | Go **1.25.x** | JWT enriquecido, Outbox, DIDs |

### 3.5 Microservicios Core

| Servicio | Lenguaje / Libs | Versión |
|---------|----------------|---------|
| Scheduling Service | Go + `go-chi/chi` + pgxpool | Go **1.25.x**, Chi **v5.x** |
| Clinical Record Service | Go + `go-chi/chi` + pgxpool | Go **1.25.x** |
| Billing & Ops Service | Go + `go-chi/chi` + pgxpool | Go **1.25.x** |

**Pasarelas de pago soportadas:** Izipay (LATAM), Stripe (global), Conekta (México)

### 3.6 Event Bus

| Componente | Tecnología | Versión | Notas |
|-----------|------------|---------|-------|
| Message Broker | NATS JetStream | **v2.11.x** | Self-hosted, Apache 2.0. Breaking changes desde 2.10 — ver upgrade guide |
| Go Client | nats.go | **v1.50.0** | Cliente oficial NATS para Go |
| Streams | 4 streams por dominio | — | IAM: 1a, Scheduling: 2a, Clinical: ∞, Billing: 10a |

> **Nota NATS 2.11:** Contiene breaking changes respecto a 2.10.x. Consultar la [guía de upgrade oficial](https://docs.nats.io/running-a-nats-service/upgrading) antes de migrar desde 2.10.

### 3.7 Serialización de Eventos

| Componente | Tecnología | Versión | Notas |
|-----------|------------|---------|-------|
| Formato | Protocol Buffers | **v3** (proto3 syntax) | Schema evolution nativa, sin Schema Registry |
| Toolchain | buf CLI (buf.build) | **latest** | Open source, tier gratuito. `buf generate` + `buf breaking` |

### 3.8 Persistencia

| Componente | Tecnología | Versión | Notas |
|-----------|------------|---------|-------|
| Base de datos | PostgreSQL | **17** (17.4 patch) | Self-hosted via CloudNativePG. `ghcr.io/cloudnative-pg/postgresql:17.x` |
| Operador PostgreSQL | **CloudNativePG** | **v1.x** | CNCF project. Respaldado por EDB. Lifecycle completo: backups WAL, PITR, self-healing. |
| Driver Go | pgx / pgxpool | **v5.7+** (`pgx/v5`) | `github.com/jackc/pgx/v5/pgxpool`. Sin cambios. |
| Primary Keys | UUIDv7 | — | PG17 nativo + `github.com/google/uuid` v1.6+ |
| Seguridad | Row-Level Security (RLS) | — | Alimentado por JWT enriquecido |
| Migraciones | golang-migrate | **v4.x** | Ejecutado como Kubernetes Job / init container antes de cada deploy. |

**Fase 1-3:** 1 Cluster CRD de CloudNativePG, 6 databases aisladas en la misma instancia.  
**Fase 4+:** Múltiples Cluster CRDs para aislamiento de compliance crítico (clinical_db separado).

### 3.9 Estándares Médicos

| Estándar | Rol | Implementación |
|---------|-----|----------------|
| FHIR R4 | Output APIs + interoperabilidad externa | Serialización JSON en endpoints REST |
| openEHR Canonical JSON | Formato de eventos clínicos | Como paradigma de modelado (sin servidor EHRbase hasta Fase 4+) |

> **Nota:** EHRbase (servidor openEHR en Java) consume 500MB-1GB RAM. Se omite hasta Fase 4+. Los arquetipos openEHR se usan como formato JSON dentro del Event Store PostgreSQL.

### 3.10 Observabilidad

| Componente | Tecnología | Versión |
|-----------|------------|---------|
| Métricas | Prometheus | **v3.x** |
| Logs | Loki + Promtail | **v3.x** |
| Tracing | Tempo | **v2.9+** |
| Dashboards | Grafana | **v11.x** |
| Alertas | Alertmanager → Telegram | **v0.27+** |

**Costo total observabilidad:** $0 (self-hosted, dashboards importados de grafana.com, alertas vía Telegram bot gratuito)

---

## 4. Tabla de Versiones Pinneadas

> Actualizado: Abril 2026. Prioridad: versión más reciente con breaking changes sobre versiones anteriores.

| Componente | Versión Pinneada | Breaking vs Anterior | Fuente |
|-----------|-----------------|---------------------|--------|
| **Talos Linux** | **v1.10.x** | SÍ (vs v1.9.x) — SELinux enforcing, systemd-boot | github.com/siderolabs/talos |
| **Kubernetes** | **1.33.x** (via Talos) | SÍ (vs 1.32.x) | kubernetes.io |
| **cert-manager** | **v1.x (latest)** | — | cert-manager.io |
| **Traefik** | **v3.x** | SÍ (vs v2.x) — new routing syntax, Gateway API | traefik.io |
| **CloudNativePG** | **v1.x (latest)** | — | cloudnative-pg.io |
| **FluxCD** | **v2.x (latest)** | SÍ (vs v1.x) — nueva API GitRepository/Kustomization | fluxcd.io |
| **Sealed Secrets** | **v0.27+** | No significativo | github.com/bitnami-labs/sealed-secrets |
| **Helm** | **v3.x (latest)** | No (vs v3.x anterior) | helm.sh |
| **Go** | 1.25.x (1.25.8+) | No (compatible con 1.24) | go.dev |
| **TypeScript** | **6.0** | **SÍ** (vs 5.x) — deprecaciones, DOM types, settings | devblogs.microsoft.com |
| **Qwik** | **v2.0** | **SÍ** (vs v1.x) — nueva API, nuevo paquete `@qwik.dev/*` | github.com/QwikDev |
| **Astro** | **v6.x** | **SÍ** (vs v5.x) — requerido para Qwik v2 | github.com/withastro |
| **NATS Server** | **v2.11.14** | **SÍ** (vs 2.10.x) — ver Upgrade Guide | github.com/nats-io |
| **NATS Go client** | v1.50.0 | No significativo | github.com/nats-io/nats.go |
| **PostgreSQL** | 17 (17.4) | SÍ (vs 16) — UUID generation nativa v7 | postgresql.org |
| **Ory Kratos** | v1.3.1 | No (desde v1.3.0) | github.com/ory/kratos |
| **OpenFGA** | v1.x (latest) | — | github.com/openfga/openfga |
| **Hono** | v4.x | SÍ (vs v3.x) | hono.dev |
| **pgx / pgxpool** | v5.7+ (`/v5`) | SÍ (vs v4) — nueva API | github.com/jackc/pgx |
| **Chi** | v5.x | No (desde v5.0) | github.com/go-chi/chi |
| **Caddy** | v2.9+ | No | caddyserver.com |
| **Grafana** | v11.x | SÍ (vs v10) | grafana.com |
| **Loki** | v3.x | SÍ (vs v2.x) | grafana.com/docs/loki |
| **Tempo** | v2.9+ | No | grafana.com/docs/tempo |
| **Prometheus** | v3.x | SÍ (vs v2.x) | prometheus.io |
| **Protocol Buffers** | proto3 syntax | — | protobuf.dev |
| **buf CLI** | latest | — | buf.build |

---

## 5. Patrones Arquitectónicos Invariables

Los siguientes patrones son **irrenunciables** y no deben modificarse en ninguna fase:

### Event Sourcing Puro — Clinical Record

```
La historia clínica es append-only.
NUNCA hay UPDATE ni DELETE en clinical_events.
Cada diagnóstico, prescripción o nota = 1 evento inmutable.
El estado actual se reconstruye reproduciendo el log de eventos.
Formato del payload: openEHR Canonical JSON serializado en Protobuf → BYTEA
```

### Outbox Pattern — Garantía de Entrega

```
El evento NATS y el dato de negocio se insertan en la MISMA transacción PostgreSQL.
Un worker Go independiente lee la tabla outbox (WHERE published_at IS NULL).
Publica en NATS y marca published_at = NOW().
Garantía: at-least-once delivery.
Consecuencia: todos los consumidores NATS deben ser idempotentes.
```

### UUIDv7 — Primary Keys Ordenadas Temporalmente

```
Los primeros 48 bits = timestamp en ms → orden temporal → BTree eficiente en PG.
Generación: PostgreSQL 17 nativo (gen_random_uuid() soporta v7) + github.com/google/uuid v1.6+
Sin coordinación entre servicios para generar IDs únicos.
```

### JWT Enriquecido + RLS

```go
// El IAM Domain Service firma con Ed25519:
// Claims: sub (UUIDv7), role, tenant_id, did
//
// Cada servicio Go inyecta en la sesión PG:
SET LOCAL app.user_id   = '<uuid_del_jwt>';
SET LOCAL app.user_role = '<role_del_jwt>';
// Las políticas RLS se evalúan automáticamente en cada query
```

### Database-per-Service

```
Cada microservicio tiene su propia database en PostgreSQL.
Ningún servicio puede leer directamente la DB de otro.
La única comunicación cross-service es:
  a) Eventos via NATS JetStream (async)
  b) API calls via el API Gateway (sync)
```

---

## 6. Decisiones de Diseño — ADRs

### ADR-001: Go 1.25.x como lenguaje de microservicios

**Decisión:** Go para todos los microservicios de dominio.  
**Razón:** Performance (150K+ req/s), goroutines para concurrencia event-driven, pgxpool como mejor driver PG, ecosistema NATS/Protobuf maduro, binario único para deploy.  
**Rechazado:** Rust (overkill IO-bound, sin ecosistema médico), TypeScript/Bun (GC no determinista para event stores críticos).

---

### ADR-002: NATS JetStream v2.11 self-hosted

**Decisión:** NATS JetStream v2.11.x self-hosted en VPS.  
**Razón:** Open source (Apache 2.0), $0, binario único (~20MB), latencia P99 <1ms.  
**Nota de versión:** v2.11 tiene breaking changes vs 2.10. Consultar upgrade guide.  
**Rechazado:** Redpanda Cloud (costo variable), Kafka/Redpanda self-hosted (mayor RAM/CPU sin beneficio al volumen médico).

---

### ADR-003: PostgreSQL 17 self-hosted via CloudNativePG (ACTUALIZADO)

**Decisión:** PostgreSQL 17.x self-hosted, gestionado por el operador CloudNativePG v1.x en Kubernetes.  
**Razón original mantenida:** Open source, $0, EXCLUDE USING GIST, RLS nativo, BYTEA para event store, UUIDv7 nativo en PG17.  
**Actualización vs versión anterior:** El container `postgres:17-alpine` directo (Docker) es reemplazado por CloudNativePG v1.x. El mismo PostgreSQL 17.4, ahora con gestión declarativa, backup WAL continuo hacia B2, PITR, y self-healing.  
**Rechazado:** Neon/Supabase/managed (costo innecesario con equipo PRO), SQLite/Turso (no adecuado para event stores), postgres container directo en k8s sin operator (inmanejable en producción — sin PITR, sin rolling upgrades).  
Ver: **ADR-013** (CloudNativePG) en [`plans/06_talos_k8s_decision_y_cambios_en_cadena.md`](../plans/06_talos_k8s_decision_y_cambios_en_cadena.md).

---

### ADR-004: Ory Kratos v1.3.1 + OpenFGA para IAM

**Decisión:** Ory Kratos v1.3.1 (AuthN) + OpenFGA v1.x (AuthZ) + IAM Domain Service (Go).  
**Razón:** Ambos open source y $0. OpenFGA tiene mejor DX que Ory Keto para relaciones médicas, playground en play.fga.dev.  
**Rechazado:** Zitadel (sin Zanzibar nativo), Auth0/Descope (managed, costo).

---

### ADR-005: Protocol Buffers v3 + buf.build

**Decisión:** Protobuf proto3 + buf.build para serialización de eventos.  
**Razón:** Schema evolution nativa sin Schema Registry externo, 60% más compacto que JSON, generación de código tipado.  
**Rechazado:** Avro (requiere Schema Registry externo en self-hosted), JSON puro (grande, lento para event stores).

---

### ADR-006: cert-manager v1.x + Traefik v3.x como ingress y TLS (REEMPLAZA Caddy en producción)

**Decisión original de Caddy:** Open source, TLS automático, forward_auth integrado — todos estos principios siguen vigentes.  
**Actualización:** En el contexto de Kubernetes, el Caddy Ingress Controller oficial (`caddyserver/ingress`) está marcado como WIP y no es apto para producción médica. Es reemplazado por:
- **cert-manager v1.x** (CNCF graduated): TLS automático vía Let's Encrypt, equivalente al CertMagic de Caddy.
- **Traefik v3.x**: `ForwardAuth Middleware` es el equivalente funcional exacto del `forward_auth` de Caddy. La seguridad del modelo IAM no cambia.
- **Caddy en local dev:** Caddy puede seguir usándose en entornos de desarrollo local con Docker Compose.  
Ver: **ADR-012** (cert-manager + Traefik) en [`plans/06_talos_k8s_decision_y_cambios_en_cadena.md`](../plans/06_talos_k8s_decision_y_cambios_en_cadena.md).

---

### ADR-007: openEHR como paradigma (sin servidor EHRbase hasta Fase 4+)

**Decisión:** Usar openEHR Canonical JSON como formato de eventos clínicos, sin desplegar EHRbase.  
**Razón:** EHRbase (Java) consume 500MB-1GB RAM en el VPS inicial de 4GB. Los arquetipos openEHR se usan como formato JSON dentro del Event Store PostgreSQL.

---

### ADR-008: TypeScript 6.0 en todo código TypeScript

**Decisión:** TypeScript 6.0 (released March 23, 2026) para Qwik SPA, Astro landing page y Hono BFF.  
**Razón:** Es la versión más reciente con breaking changes desde 5.x. Es la última versión basada en JavaScript antes de TypeScript 7.0 (nativo en Go). Prioridad: última versión > compatibilidad hacia atrás.  
**Breaking changes relevantes:** DOM types actualizados, deprecación de opciones legadas, mejoras de inferencia, nuevo target `es2025`.

---

### ADR-009: Qwik v2.0 + Astro v6.x

**Decisión:** Qwik v2.0 con paquete `@qwik.dev/qwik` (breaking change vs `@builder.io/qwik` v1.x). Astro v6.x requerido para soporte nativo de Qwik v2 vía `@qwik.dev/astro`.  
**Razón:** Versión más reciente con breaking changes. Qwik v2 tiene mejor DX, APIs más estables y es la versión activa mantenida.

---

### ADR-010: Talos Linux v1.10.x como OS de Producción

**Decisión:** Talos Linux v1.10.x reemplaza Ubuntu 24.04 LTS como sistema operativo del VPS.  
**Razón:** Inmutabilidad total del OS (ejecuta desde SquashFS firmado), superficie de ataque mínima (12 binarios en PATH vs miles en Ubuntu), gestión declarativa API-driven (talosctl), upgrade atómico OS+Kubernetes como una sola unidad, zero SSH en producción, Hetzner Cloud ISO oficial disponible desde abril 2025. Alineación máxima con R1 (Build It Right) y R4 (Soberanía Tecnológica).  
**Rechazado:** Ubuntu 24.04 LTS (drift de config, SSH como vector, hardening manual continuo); Flatcar Container Linux (SSH disponible, no full OS immutability); NixOS (menor ecosistema k8s).  
Ver análisis completo: [`plans/06_talos_k8s_decision_y_cambios_en_cadena.md`](../plans/06_talos_k8s_decision_y_cambios_en_cadena.md).

---

### ADR-011: Kubernetes 1.33.x (upstream via Talos) como Orquestador Definitivo desde Fase 1

**Decisión:** Kubernetes 1.33.x, gestionado por Talos, es el orquestador de producción desde Fase 1. No hay Docker Compose ni Docker Swarm en producción.  
**Razón:** R2 (escalar de 1 a 10,000 médicos sin refactorizaciones estructurales). Docker Compose → Swarm → k8s eran dos migraciones estructurales futuras. Kubernetes es el mismo modelo operativo de 1 nodo a N nodos. Agregar workers es `talosctl apply-config`, no una migración arquitectónica.  
**Nota crítica:** "Talos + k3s" es técnicamente imposible. Talos gestiona upstream Kubernetes directamente; k3s es una distribución para OSes de propósito general.  
**Docker Compose:** Únicamente para desarrollo local. No aparece en producción.

---

### ADR-012: cert-manager v1.x + Traefik v3.x como Stack de Ingress y TLS

**Decisión:** cert-manager v1.x (CNCF graduated) + Traefik v3.x reemplazan Caddy v2.9+ en el entorno Kubernetes de producción.  
**Razón:** Caddy Ingress Controller oficial está marcado como WIP. cert-manager es el estándar de TLS en Kubernetes (~25M+ instancias). Traefik v3 implementa `ForwardAuth Middleware` — equivalente funcional exacto del `forward_auth` de Caddy para el IAM Domain Service. Traefik v3 soporta Gateway API (estándar futuro).  
**Rechazado:** Caddy Ingress (WIP); ingress-nginx (EOL Q1 2026); Istio/Envoy (overkill, 1 GB+ RAM adicional).

---

### ADR-013: CloudNativePG v1.x como Operador PostgreSQL

**Decisión:** CloudNativePG v1.x (CNCF, respaldado por EDB) gestiona el ciclo de vida completo de PostgreSQL en Kubernetes.  
**Razón:** PostgreSQL en k8s sin operador es inmanejable en producción: sin backup WAL automático, sin PITR, sin self-healing. Para historia clínica médica inmutable, PITR con WAL continuo → B2 es obligatorio. pg_dump diario permite pérdida de hasta 24h de datos; WAL continuo reduce la ventana a minutos. CloudNativePG genera automáticamente Kubernetes Secrets con connection strings para los microservicios Go.  
**Rechazado:** postgres container directo (inmanejable en k8s producción); Zalando postgres-operator (menor madurez CNCF); CrunchyData PGO (mayor overhead).

---

### ADR-014: FluxCD v2.x como Sistema GitOps

**Decisión:** FluxCD v2.x (CNCF graduated) gestiona declarativamente el estado completo de producción desde Git.  
**Razón:** R1 (Build It Right The First Time) aplicado a operaciones: sin GitOps el estado de producción diverge inevitablemente del código. Con FluxCD, Git es la única fuente de verdad. Auditoría completa de cada cambio a producción. Auto-reconciliación: cualquier cambio manual directo es revertido al estado declarado en Git.  
**Rechazado:** ArgoCD (~500 MB+ RAM, UI innecesaria para este escala); Helm directo sin GitOps (sin reconciliation loop, sin auditoría).

---

## 7. Roadmap de Construcción

| Fase | Duración | Costo/mes | Objetivo |
|------|----------|-----------|----------|
| **Fase 1** | 3-4 meses | ~$7.50 | VPS CX32 + Talos + k8s + IAM + Gateway + CloudNativePG |
| **Fase 2** | 4-6 meses | ~$16 | Scheduling + Clinical Record + NATS JetStream |
| **Fase 3** | 3-4 meses | ~$22-27 | Billing + OpenFGA + kube-prometheus-stack completo |
| **Fase 4+** | — | ~$22-45 | Multi-nodo (add CX32 workers), EHRbase, multi-región, DIDs |

> Ver roadmap detallado: [`plans/04_gap_map_y_ruta_migracion.md`](../plans/04_gap_map_y_ruta_migracion.md)

---

## 8. Referencias Técnicas

| # | Documento | Descripción |
|---|-----------|-------------|
| [1] | [`plans/03_stack_alternatives_challenger.md`](../plans/03_stack_alternatives_challenger.md) | Challenger analysis v3.0 con decisiones finales |
| [2] | [`plans/04_gap_map_y_ruta_migracion.md`](../plans/04_gap_map_y_ruta_migracion.md) | Gap map y roadmap detallado |
| [3] | [`plans/01_analisis_comparativo_repo_vs_target.md`](../plans/01_analisis_comparativo_repo_vs_target.md) | Comparativa repo actual vs target |
| [4] | [`plans/02_target_arch_deep_dive.md`](../plans/02_target_arch_deep_dive.md) | Deep-dive técnico del target |
| [5] | [`target_arch/arch.d2`](./arch.d2) | Diagrama arquitectural general (D2 — render: `d2 --layout=elk arch.d2 arch.svg`) |
| [6] | [`target_arch/comp_iam_capa_core.d2`](./comp_iam_capa_core.d2) | Diagrama granular del IAM Component (D2) |
| [7] | [`target_arch/comp_infra_operaciones.d2`](./comp_infra_operaciones.d2) | Topología VPS + Kubernetes (Talos Linux v1.10.x) v4.0 (D2) |
| [5b] | [`target_arch/arch.puml`](./arch.puml) | Diagrama arquitectural general (PlantUML — legado) |
| [6b] | [`target_arch/comp_iam_capa_core.puml`](./comp_iam_capa_core.puml) | IAM Component (PlantUML — legado) |
| [7b] | [`target_arch/comp_infra_operaciones.puml`](./comp_infra_operaciones.puml) | Infraestructura Operacional (PlantUML — legado) |
| [8] | [`plans/06_talos_k8s_decision_y_cambios_en_cadena.md`](../plans/06_talos_k8s_decision_y_cambios_en_cadena.md) | ADR-010–014: Talos + k8s decision, R1-R7 validation, 18 cascade changes |
| [9] | [`plans/07_fase1_laptop_talos_desarrollo.md`](../plans/07_fase1_laptop_talos_desarrollo.md) | Fase 1 — Entorno local: laptop con Talos Linux (variante desarrollo) |
| [10] | [`plans/07b_fase1_vps_hetzner_produccion.md`](../plans/07b_fase1_vps_hetzner_produccion.md) | Fase 1 — Entorno VPS directo: Hetzner CX32 desde el día 1 (variante producción) |

---

## 9. Orden Final de Implementación

> Detalle exhaustivo completo en:
> - [`plans/05_orden_implementacion_capas_exhaustivo.md`](../plans/05_orden_implementacion_capas_exhaustivo.md) — Fases 1 a 4+, subcapas, prerrequisitos, criterios de aceptación
> - [`plans/05b_orden_implementacion_capas_detalle.md`](../plans/05b_orden_implementacion_capas_detalle.md) — Subcapas 1.E–3.D, grafo de dependencias, matriz de componentes, definición de hecho
> - [`plans/06_talos_k8s_decision_y_cambios_en_cadena.md`](../plans/06_talos_k8s_decision_y_cambios_en_cadena.md) — ADR Talos + Kubernetes: decisión definitiva, cambios en cadena, detalles de implementación

### Principios de Ordenación

| # | Regla | Descripción |
|---|-------|-------------|
| 1 | **Dependencia fuerte** | X debe existir antes que Y si Y lo necesita para compilar o funcionar |
| 2 | **Valor de negocio primero** | Se prioriza lo que genera valor verificable (pago procesado, diagnóstico guardado) sobre calidad interna |
| 3 | **Infraestructura antes que servicio** | Red, TLS, DB verificados antes de desplegar código de negocio |
| 4 | **Seguridad antes que funcionalidad** | IAM completo (Kratos + IAM Domain Service) antes de exponer cualquier endpoint médico |
| 5 | **Observabilidad antes que complejidad** | `/health` + `/metrics` son parte de la definición de hecho de cada microservicio |
| 6 | **Outbox como ciudadano de primera clase** | El Outbox Pattern se diseña e implementa junto con cada microservicio Go, nunca como añadido posterior |

### Mapa de Fases y Capas

```
FASE 1 — Cimiento Kubernetes: Infraestructura + IAM + Gateway (~$7.50/mes)
  1.A  Hetzner CX32 + Talos Linux v1.10.x + Kubernetes 1.33.x + FluxCD bootstrap
  1.B  cert-manager v1.x + Traefik v3.x: ingress + TLS automático + Sealed Secrets
  1.C  CloudNativePG v1.x + PostgreSQL 17.4: 6 databases aisladas, WAL → B2
  1.D  Schemas Proto + buf.build (fundación serialización)
  1.E  Ory Kratos v1.3.1 (AuthN) via Helm                  ← requiere 1.C
  1.F  IAM Domain Service Go (JWT Ed25519 + Outbox)          ← requiere 1.C, 1.D, 1.E
       Traefik ForwardAuth configurado → /internal/validate-token
  1.G  BFF: CF Workers + Hono v4.x                          ← requiere 1.F
  1.H  Qwik v2.0 SPA en CF Pages                            ← requiere 1.G
  1.I  CloudNativePG ScheduledBackup CRD → Backblaze B2

FASE 2 — Motor Médico: NATS + Scheduling + Clinical (~$16/mes)
  2.A  NATS JetStream v2.11.x + 4 streams (Helm: nats/nats)  ← requiere 1.A–1.C
  2.B  Outbox Worker transversal (librería Go)                ← requiere 2.A
  2.C  Scheduling Service (Deployment + Service k8s)          ← requiere 1.F, 2.A–2.B
  2.D  Clinical Record Service (Deployment + Service k8s)     ← requiere 1.F, 2.A–2.C
  2.E  Landing Page: Astro v6.x en CF Pages

FASE 3 — Plataforma Completa: Billing + AuthZ + Observabilidad (~$22–27/mes)
  3.A  OpenFGA v1.x (AuthZ Zanzibar, Helm: openfga/openfga)  ← requiere 1.C, 1.F
  3.B  Billing & Ops Service (Deployment + Service k8s)       ← requiere 1.F, 2.A–2.B, 2.D
  3.C  kube-prometheus-stack v82.x + Loki v3.x + Tempo v2.9+
  3.D  Alertmanager v0.27+ → Telegram bot                    ← requiere 3.C

FASE 4+ — Evolución Futura (~$22–45/mes)
  Add worker nodes CX32 sin downtime, EHRbase, DIDs, multi-región
  talosctl apply-config --nodes <NEW_IP> → cluster escala automáticamente
```

### Grafo de Dependencias Clave

```
Talos + k8s + FluxCD (1.A)
  └─► cert-manager + Traefik (1.B) ────────────────────┐
  └─► CloudNativePG (1.C) ──┐                          │
       └─► Protobuf (1.D)   │                          │
       └─► Kratos (1.E) ────┤                          │
            └─► IAM Go (1.F) ──► BFF (1.G) ────────────┴─► Qwik SPA (1.H)
                 └─► NATS (2.A) ──► Outbox (2.B)
                      └─► Scheduling (2.C)
                           └─► Clinical (2.D)
                                └─► Billing (3.B) ◄── OpenFGA (3.A)
                                     └─► PLG Stack (3.C) ──► Alertas (3.D)
```
