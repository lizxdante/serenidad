# Blueprint de Diagramas PlantUML — Actualización v4.0

**Propósito:** Historial de los diagramas PlantUML del target_arch y sus estados actuales. Los archivos `.puml` correspondientes ya han sido actualizados y reflejan el stack definitivo con Talos Linux + Kubernetes 1.33.x.

**Estado actual (v4.0 — Abril 2026):**
- `arch.puml`: Stack completo actualizado — Traefik v3.x, CloudNativePG, Talos+k8s, FluxCD GitOps
- `comp_iam_capa_core.puml`: Traefik ForwardAuth (reemplaza Caddy forward_auth), Helm-based deployment
- `comp_infra_operaciones.puml`: v4.0 — Topología Kubernetes/Talos completa (reemplaza Docker Compose)

**ADR de referencia:** [`plans/06_talos_k8s_decision_y_cambios_en_cadena.md`](../plans/06_talos_k8s_decision_y_cambios_en_cadena.md)

---

## 1. Contenido histórico de `target_arch/arch.puml` (pre-Kubernetes)

> **NOTA:** Este bloque es el contenido histórico pre-decisión Talos+k8s. El archivo `arch.puml` real ya ha sido actualizado a v4.0. Este bloque queda como referencia histórica del challenger analysis.



```plantuml
@startuml
!theme plain
skinparam componentStyle uml2
skinparam linetype ortho
skinparam padding 5
skinparam defaultTextAlignment center

title Arquitectura "Build It Right The First Time" v3.0\nClinica Global de Salud Mental - Stack Definitivo (Abril 2026)

' --- PALETA DE COLORES ---
!define COLOR_FASE_1  #FF5722
!define COLOR_FASE_2  #2196F3
!define COLOR_FASE_3  #9C27B0
!define COLOR_INFRA   #ECEFF1
!define COLOR_DB      #CFD8DC
!define COLOR_FRONT   #F5F5F5
!define COLOR_NATS    #27AE60
!define COLOR_OPS     #607D8B

' --- ESTILOS ---
skinparam component {
  BackgroundColor<<Fase_1>>  COLOR_FASE_1
  FontColor<<Fase_1>>        white
  BackgroundColor<<Fase_2>>  COLOR_FASE_2
  FontColor<<Fase_2>>        white
  BackgroundColor<<Fase_3>>  COLOR_FASE_3
  FontColor<<Fase_3>>        white
  BackgroundColor<<Front>>   COLOR_FRONT
  BorderColor<<Front>>       #BDBDBD
  BackgroundColor<<Gateway>> COLOR_INFRA
  BorderColor<<Gateway>>     #90A4AE
  BackgroundColor<<Broker>>  COLOR_NATS
  FontColor<<Broker>>        white
  BackgroundColor<<Ops>>     COLOR_OPS
  FontColor<<Ops>>           white
}

skinparam database {
  BackgroundColor<<Storage>> COLOR_DB
  BorderColor<<Storage>>     #78909C
}

skinparam cloud {
  BackgroundColor #E3F2FD
  BorderColor     #1565C0
}

' ==========================================================
' CAPA CLOUDFLARE
' ==========================================================
package "Cloudflare Edge - Tier Gratuito" {

  component "[CF Pages] Qwik SPA\n--\nPortal Medico y Paciente.\nResumability: TTI ~50ms.\nSin hidratacion." <<Front>> as Front

  component "[CF Pages] Astro 5\n--\nLanding Page y Marketing.\nLighthouse 99.2. Zero JS.\nSEO Optimo." <<Front>> as Astro

  component "[CF Workers] BFF / Hono.js\n--\nJWT Verify EdDSA.\nRouting y Edge Cache.\n100K req/dia gratuito." <<Gateway>> as BFF

}

' ==========================================================
' VPS HETZNER
' ==========================================================
package "VPS Hetzner - Docker Compose" {

  component "[Caddy] Reverse Proxy\n--\nTLS automatico Let\'s Encrypt.\nForward Auth al IAM Service.\nRouting a microservicios." <<Gateway>> as Caddy

  package "Dominio Medico y Negocio" {

    component "[1] IAM and Identity Service\n--\nOry Kratos AuthN Passkeys.\nOpenFGA AuthZ Zanzibar.\nJWT enriquecido UUIDv7." <<Fase_1>> as IAM

    component "[2] Scheduling Service\n--\nCalendly propio.\nZonas horarias y slots.\nOutput: FHIR Appointment.\nEXCLUDE USING GIST." <<Fase_2>> as Scheduling

    component "[2] Clinical Record Service\n--\nEl Cerebro Medico.\nopenEHR Canonical JSON.\nEvent Store puro + RLS." <<Fase_2>> as Clinical

    component "[3] Billing and Ops Service\n--\nFacturacion y recibos.\nMulti-pasarela de pago.\nIzipay Stripe Conekta." <<Fase_3>> as Billing

  }

  package "Columna Vertebral de Eventos" {
    component "NATS JetStream 2.10\n--\nMessage Broker Apache 2.0.\nEventos en Protobuf v3.\nRetencion por dominio medico." <<Broker>> as NATS
  }

  package "Persistencia y Event Stores - PostgreSQL 17" {

    database "DB_Identity\n--\nKratos DB y OpenFGA DB.\nDomain profiles.\nOutbox events.\nRLS Policies." <<Storage>> as DB_IAM

    database "DB_Scheduling\n--\nAppointments TSTZRANGE.\nEXCLUDE GIST constraints.\nOutbox Protobuf.\nFHIR resources JSONB." <<Storage>> as DB_Sched

    database "DB_Clinical\n--\nEvent Store PURO.\nPayload: BYTEA Protobuf.\nopenEHR Canonical JSON.\nPK: UUIDv7. Inmutable." <<Storage>> as DB_Clin

    database "DB_Billing\n--\nTransacciones Financieras.\nAppend-only inmutable.\nCurrency ISO 4217.\nTax country codes." <<Storage>> as DB_Bill

  }

  package "Observabilidad - PLG Stack" {
    component "Prometheus y Loki\ny Tempo y Grafana\n--\nMetricas, Logs, Tracing.\nDashboards pre-construidos.\nAlertas via Telegram." <<Ops>> as Obs
  }

}

cloud "Backblaze B2\npg_dump diario\n~0.05 usd/mes" as B2

' --- FLUJOS PRINCIPALES ---

Front   --> BFF   : "HTTPS / WebSocket"
Astro   --> BFF   : "HTTPS"
BFF     --> Caddy : "HTTPS al VPS"

Caddy --> IAM       : "Forward Auth + Routing"
Caddy --> Scheduling : "/api/scheduling"
Caddy --> Clinical   : "/api/clinical"
Caddy --> Billing    : "/api/billing"

IAM       -down-> DB_IAM   : "Go pgxpool"
Scheduling -down-> DB_Sched : "Go pgxpool"
Clinical  -down-> DB_Clin  : "Go pgxpool"
Billing   -down-> DB_Bill  : "Go pgxpool"

DB_IAM   .up.> NATS : "Outbox [UserRegistered]\n[DoctorOnboarded]"
DB_Sched .up.> NATS : "Outbox [AppointmentBooked]\n[AppointmentCancelled]"
DB_Clin  .up.> NATS : "Outbox [DiagnosisRecorded]\n[ConsultationFinished]"

NATS ..> Clinical   : "Consume [AppointmentBooked]"
NATS ..> Billing    : "Consume [ConsultationFinished]"
NATS ..> Scheduling : "Consume [DoctorOffboarded]"

DB_IAM  .right.> B2 : "backup"
DB_Clin .right.> B2 : "backup"
DB_Bill .right.> B2 : "backup"

Obs ..> IAM       : "Scrape /metrics"
Obs ..> Scheduling : "Scrape /metrics"
Obs ..> Clinical  : "Scrape /metrics"
Obs ..> Billing   : "Scrape /metrics"
Obs ..> NATS      : "Scrape :8222"

' --- NOTAS ---
note right of NATS
  **Protobuf v3 + buf.build**
  Schema evolution nativa.
  Sin Schema Registry adicional.
  .proto files en Git = contrato.
end note

note right of IAM
  **OpenFGA en lugar de Ory Keto**
  Zanzibar relation tuples.
  Mejor DX para relaciones medicas.
  Playground: play.fga.dev
end note

note bottom of DB_Clin
  **Event Store Puro**
  NUNCA UPDATE ni DELETE.
  openEHR Canonical JSON en BYTEA.
  Estado = replay desde eventos.
end note

@enduml
```

---

## 2. Contenido histórico de `target_arch/comp_iam_capa_core.puml` (pre-Kubernetes)

> **NOTA:** Versión histórica pre-decisión Talos+k8s. El archivo `comp_iam_capa_core.puml` real ya ha sido actualizado con referencias a Traefik ForwardAuth y Helm-based deployment.



```plantuml
@startuml
!theme plain
skinparam componentStyle uml2
skinparam linetype ortho
skinparam padding 8
skinparam defaultTextAlignment center

title Arquitectura Granular: IAM Component - Identity and Access Management v3.0

' --- PALETA DE COLORES ---
!define COLOR_GO     #00ADD8
!define COLOR_ORY    #5528FF
!define COLOR_FGA    #FF6B35
!define COLOR_DB     #CFD8DC
!define COLOR_NATS   #27AE60
!define COLOR_FRONT  #F5F5F5

skinparam component {
  BackgroundColor<<Go>>  COLOR_GO
  FontColor<<Go>>        white
  BackgroundColor<<Ory>> COLOR_ORY
  FontColor<<Ory>>       white
  BackgroundColor<<FGA>> COLOR_FGA
  FontColor<<FGA>>       white
  BackgroundColor<<Ext>> COLOR_FRONT
  BorderColor<<Ext>>     #BDBDBD
}

skinparam database {
  BackgroundColor<<Storage>> COLOR_DB
  BorderColor<<Storage>>     #78909C
}

skinparam rectangle {
  BackgroundColor transparent
  BorderColor     #455A64
  BorderStyle     dashed
}

' --- ACTORES EXTERNOS AL IAM ---
component "Caddy + CF Workers BFF" <<Ext>> as External
component "NATS JetStream\nEvent Bus Global" <<Ext>> as NATS
component "Scheduling, Clinical\ny demas Servicios Go" <<Ext>> as CoreServices

' --- IAM BOUNDED CONTEXT ---
rectangle "IAM Bounded Context - Microservicio Aislado" {

  component "IAM Domain Service\nGo + Chi + pgxpool\n--\nOrquestador de Identidad.\nEnriquece JWTs con UUIDv7.\nGestiona DIDs.\nPublica eventos via Outbox." <<Go>> as GoIAM

  rectangle "Ory Stack - Autenticacion" {
    component "Ory Kratos v1.3\nAuthN\n--\nPasskeys WebAuthn.\nGestion de Sesiones y MFA.\nCero contrasenas por defecto.\nFlows self-service." <<Ory>> as Kratos
  }

  rectangle "OpenFGA - Autorizacion Zanzibar" {
    component "OpenFGA\nAuthZ\n--\nMotor Zanzibar.\nRelation Tuples:\nDoctor X puede escribir\nen Historia Y.\nPlayground: play.fga.dev" <<FGA>> as OpenFGA
  }

  rectangle "IAM Database Cluster - PostgreSQL 17" {

    database "Kratos DB\n--\nCredenciales y hashes.\nSesiones activas.\nIdentity schemas.\nEsquema interno Ory." <<Storage>> as DB_Kratos

    database "OpenFGA DB\n--\nRelation Tuples.\nGrafos de permisos.\nStore ID por organizacion.\nConsistencia configurable." <<Storage>> as DB_OpenFGA

    database "IAM Domain DB\n--\nOutbox Table para NATS.\nPerfiles de usuarios.\nUUIDv7 como PK.\nRLS Policies." <<Storage>> as DB_Domain

  }
}

' --- FLUJOS DE COMUNICACION ---

' 1. Flujo de Login
External -down-> Kratos : "1. Challenge Passkey / WebAuthn"
Kratos -down-> DB_Kratos : "Verifica Credencial"
Kratos -up-> External : "Set-Cookie / Session Token"

' 2. Flujo de Acceso Autenticado
External -down-> GoIAM : "2. Peticion autenticada\nSession Token"
GoIAM -left-> Kratos : "Verifica Sesion activa\n(Kratos Admin API)"
GoIAM -right-> OpenFGA : "Consulta Permiso:\nCan doctor:X write on\nclinical_record:Y?"
OpenFGA -down-> DB_OpenFGA : "Evalua Grafo\nde Relaciones"

' 3. Persistencia y Event Sourcing
GoIAM -down-> DB_Domain : "Transaccion local:\nInserta Perfil + Evento Outbox"

' 4. Emision de Eventos
DB_Domain .up.> NATS : "Outbox Worker pgx\nPublica [UserRegistered]\n[DoctorOnboarded]"
NATS ..> CoreServices : "Distribuye eventos\na toda la clinica"

' 5. Verificacion en otros servicios
CoreServices .left.> GoIAM : "Valida JWT via\nclave publica Ed25519\nverificacion asincrona"

' --- NOTAS ---
note right of GoIAM
  **El JWT Enriquecido Ed25519:**
  GoIAM toma la sesion de Kratos,
  inyecta UUIDv7 + rol + tenant_id
  y firma con clave Ed25519.
  Este JWT alimenta el RLS de
  todos los PostgreSQL del sistema.
  
  Claims del JWT:
    sub: UUIDv7 del usuario
    role: doctor | patient | admin
    tenant_id: UUIDv7 de la org
    did: did:web:sereni.dad:...
end note

note right of OpenFGA
  **Paradigma Zanzibar:**
  No guardas "Rol = Doctor".
  Guardas tuplas de relacion:
  
  doctor:dr-smith
    -> assigned_to
    -> patient:p-456
  
  patient:p-456
    -> owns
    -> clinical_record:hc-001
  
  Check: can dr-smith can_write
  on clinical_record:hc-001?
  -> YES (por transitividad)
  
  Escala a millones de relaciones.
end note

note bottom of OpenFGA
  **Cambio respecto al diseno original:**
  OpenFGA reemplaza a Ory Keto.
  Misma semantica Zanzibar.
  Mejor DX y modelado de
  relaciones medicas complejas.
  Backing institucional: Okta/Auth0.
  Open Source: Apache 2.0.
end note

@enduml
```

---

## 3. Contenido histórico para `target_arch/comp_infra_operaciones.puml` (v3.0 — SUPERSEDIDO)

> **NOTA:** Este era el contenido v3.0 (Docker Compose). El archivo `comp_infra_operaciones.puml` real fue **completamente reescrito** a v4.0 con Talos Linux + Kubernetes. Este bloque queda únicamente como referencia histórica.



```plantuml
@startuml
!theme plain
skinparam componentStyle uml2
skinparam linetype ortho
skinparam padding 6
skinparam defaultTextAlignment center

title Diagrama de Infraestructura Operacional\nSerenidad v3.0 - VPS + Docker Compose

' --- COLORES ---
!define COLOR_DOCKER  #2496ED
!define COLOR_CADDY   #00AD9F
!define COLOR_VPS     #37474F
!define COLOR_CF      #F38020
!define COLOR_BACKUP  #4CAF50
!define COLOR_OBS     #607D8B

skinparam rectangle {
  BackgroundColor transparent
  BorderColor     #455A64
  BorderStyle     dashed
  FontStyle       bold
}

skinparam component {
  BackgroundColor<<Docker>> #E3F2FD
  BorderColor<<Docker>>     COLOR_DOCKER
  BackgroundColor<<Caddy>>  #E0F7FA
  BorderColor<<Caddy>>      COLOR_CADDY
  BackgroundColor<<CF>>     #FFF3E0
  BorderColor<<CF>>         COLOR_CF
  BackgroundColor<<Backup>> #E8F5E9
  BorderColor<<Backup>>     COLOR_BACKUP
  BackgroundColor<<Obs>>    #ECEFF1
  BorderColor<<Obs>>        COLOR_OBS
}

' ==========================================================
' DNS + CDN
' ==========================================================
rectangle "DNS - Cloudflare (Gratuito)" {
  component "sereni.dad\n app.sereni.dad\n api.sereni.dad\n--\nA record -> VPS IP\nCF Proxy: OFF para api\nCF Proxy: ON para app" <<CF>> as DNS
}

' ==========================================================
' CLOUDFLARE EDGE
' ==========================================================
rectangle "Cloudflare Edge - 0 usd/mes" {
  component "Cloudflare Pages\nQwik SPA + Astro 5\n--\nBuild: git push\nDeploy: automatico\n500 builds/mes gratis\nBandwidth ilimitado" <<CF>> as CFPages

  component "Cloudflare Workers\nHono.js BFF\n--\nJWT verify EdDSA\nEdge routing\n100K req/dia gratis\n0ms cold start" <<CF>> as CFWorkers
}

' ==========================================================
' VPS HETZNER CX31
' ==========================================================
rectangle "VPS Hetzner CX31 - ~10 usd/mes\n2 vCPU AMD / 4GB RAM / 80GB SSD / Ubuntu 22.04" {

  component "Caddy 2.x\n--\nReverse Proxy\nTLS automatico Let Encrypt\nForward Auth al IAM Service\nRouting por path" <<Caddy>> as Caddy

  rectangle "Docker Compose - Red Interna" {

    component "ory-kratos:4433\noryd/kratos:v1.3\n--\nAuthN service\nPasskeys WebAuthn\nSelf-service flows" <<Docker>> as Kratos

    component "openfga:8080\nopenfga/openfga:latest\n--\nAuthZ service\nZanzibar engine\ngRPC + HTTP" <<Docker>> as OpenFGA

    component "iam-service:8080\nserenidad/iam:latest\n--\nJWT enrichment\nOutbox worker\nDomain events" <<Docker>> as IAMSvc

    component "scheduling-service:8081\nserenidad/scheduling:latest\n--\nFHIR Appointment\nEXCLUDE GIST\nOutbox worker" <<Docker>> as SchedSvc

    component "clinical-service:8082\nserenidad/clinical:latest\n--\nEvent Store\nopenEHR JSON\nRLS enforced" <<Docker>> as ClinSvc

    component "billing-service:8083\nserenidad/billing:latest\n--\nMulti-gateway\nIzipay Stripe\nOutbox worker" <<Docker>> as BillSvc

    component "nats:4222\nnats:2.10-alpine\n--\nJetStream\nFile storage\nMonitoring :8222" <<Docker>> as NATS

    component "postgres:5432\npostgres:17-alpine\n--\n4 databases aisladas\niam_db scheduling_db\nclinical_db billing_db\n+ kratos_db openfga_db" <<Docker>> as PG

  }

  rectangle "Observabilidad - Docker Compose" {
    component "Prometheus :9090\nLoki :3100\nTempo :3200\nGrafana :3000\n--\nSolo acceso via SSH tunnel\nAlertas a Telegram bot" <<Obs>> as ObsStack
  }

}

' ==========================================================
' BACKUPS EXTERNOS
' ==========================================================
rectangle "Backups - Backblaze B2 - ~0.05 usd/mes" {
  component "pg_dump diario\n+ rclone\n--\nTodos los databases\nComprimidos gzip\nRetencion: 30 dias local\nBackblaze: 90 dias" <<Backup>> as BackupSvc
}

' ==========================================================
' RELACIONES
' ==========================================================

' Usuario → Cloudflare
[Navegador\nPaciente/Medico] --> CFPages : "HTTPS app.sereni.dad"
CFPages --> CFWorkers : "fetch() al BFF\npara llamadas API"

' BFF → VPS
CFWorkers --> Caddy : "HTTPS api.sereni.dad\n(puerto 443)"

' Caddy → Servicios internos
Caddy -down-> IAMSvc    : "/auth/* y /api/iam/*\nforward_auth para todos"
Caddy -down-> SchedSvc  : "/api/scheduling/*"
Caddy -down-> ClinSvc   : "/api/clinical/*"
Caddy -down-> BillSvc   : "/api/billing/*"

' Servicios → PostgreSQL
IAMSvc  -down-> PG : "iam_db\nkratos_db\nopenfga_db"
SchedSvc -down-> PG : "scheduling_db"
ClinSvc -down-> PG : "clinical_db"
BillSvc -down-> PG : "billing_db"

' IAM Stack interno
IAMSvc -right-> Kratos  : "Kratos Admin API"
IAMSvc -right-> OpenFGA : "OpenFGA Check API"

' Servicios → NATS
SchedSvc .up.> NATS : "Publish\nAppointmentBooked"
ClinSvc  .up.> NATS : "Publish\nDiagnosisRecorded"
IAMSvc   .up.> NATS : "Publish\nDoctorOnboarded"

NATS ..> ClinSvc  : "Subscribe\nAppointmentBooked"
NATS ..> BillSvc  : "Subscribe\nConsultationFinished"
NATS ..> SchedSvc : "Subscribe\nDoctorOffboarded"

' Backups
PG -right-> BackupSvc : "pg_dump cron 2am"

' Observabilidad
ObsStack ..> IAMSvc   : "Scrape /metrics"
ObsStack ..> SchedSvc : "Scrape /metrics"
ObsStack ..> ClinSvc  : "Scrape /metrics"
ObsStack ..> BillSvc  : "Scrape /metrics"
ObsStack ..> NATS     : "Scrape :8222/metrics"
ObsStack ..> PG       : "postgres_exporter"

' --- NOTAS ---
note right of PG
  **Fase 1-2: 1 instancia Docker**
  4 databases aisladas con RLS.
  Cada servicio Go usa su propio
  usuario PG con GRANT minimo.
  
  **Fase 3+: 4 instancias Docker**
  Separar en contenedores
  independientes por servicio
  cuando el aislamiento sea critico.
end note

note right of Caddy
  **Caddyfile - TLS automatico:**
  api.sereni.dad {
    forward_auth iam-service:8080 {
      uri /internal/validate-token
    }
    handle /api/scheduling/*  {
      reverse_proxy scheduling-service:8081
    }
    handle /api/clinical/* {
      reverse_proxy clinical-service:8082
    }
  }
  Let Encrypt renueva auto.
  Sin certbot. Sin cron.
end note

note bottom of BackupSvc
  **rclone + Backblaze B2:**
  rclone copy /backups B2:bucket/
  
  Coste: 0.006 USD/GB/mes
  Para 15GB de backups: 0.09/mes
  Casi cero costo de backup.
end note

@enduml
```

---

## Instrucciones para Code Mode

Para aplicar estos cambios, en Code Mode ejecutar:

1. **Reemplazar** el contenido de [`target_arch/arch.puml`](../target_arch/arch.puml) con el bloque PlantUML de la sección 1 (sin las marcas de código markdown).

2. **Reemplazar** el contenido de [`target_arch/comp_iam_capa_core.puml`](../target_arch/comp_iam_capa_core.puml) con el bloque PlantUML de la sección 2.

3. **Crear** el archivo [`target_arch/comp_infra_operaciones.puml`](../target_arch/comp_infra_operaciones.puml) con el contenido de la sección 3.

---

## Orden Final de Implementación — Referencia para los Diagramas

> Detalle completo en:
> - [`plans/05_orden_implementacion_capas_exhaustivo.md`](../plans/05_orden_implementacion_capas_exhaustivo.md)
> - [`plans/05b_orden_implementacion_capas_detalle.md`](../plans/05b_orden_implementacion_capas_detalle.md)

Los diagramas de este directorio representan el **estado final target** (v3.1). La tabla siguiente indica en qué fase y subcapa se construye cada componente visible en los diagramas:

| Componente (en diagramas) | Fase | Subcapa | Prerrequisito principal |
|---------------------------|------|---------|------------------------|
| Hetzner CX32 + Talos Linux v1.10.x + k8s 1.33.x + FluxCD | Fase 1 | 1.A | — |
| cert-manager v1.x + Traefik v3.x + Sealed Secrets | Fase 1 | 1.B | 1.A |
| CloudNativePG v1.x + PostgreSQL 17.4 (6 DBs) | Fase 1 | 1.C | 1.A |
| Schemas Protobuf + buf.build | Fase 1 | 1.D | 1.C |
| Ory Kratos v1.3.1 (AuthN, Helm: ory/kratos) | Fase 1 | 1.E | 1.C |
| IAM Domain Service Go (JWT Ed25519 + Outbox) + Traefik ForwardAuth | Fase 1 | 1.F | 1.C, 1.D, 1.E |
| BFF: CF Workers + Hono v4.x | Fase 1 | 1.G | 1.F |
| Qwik v2.0 SPA en CF Pages | Fase 1 | 1.H | 1.G |
| CloudNativePG ScheduledBackup CRD → B2 | Fase 1 | 1.I | 1.C |
| NATS JetStream v2.11.x (Helm: nats/nats) | Fase 2 | 2.A | 1.A–1.C |
| Outbox Worker (librería Go) | Fase 2 | 2.B | 2.A |
| Scheduling Service Go (Deployment + Service k8s) | Fase 2 | 2.C | 1.F, 2.A–2.B |
| Clinical Record Service Go (Deployment + Service k8s) | Fase 2 | 2.D | 1.F, 2.A–2.C |
| Landing Page Astro v6.x | Fase 2 | 2.E | 1.G |
| OpenFGA v1.x (AuthZ, Helm: openfga/openfga) | Fase 3 | 3.A | 1.C, 1.F |
| Billing & Ops Service Go (Deployment + Service k8s) | Fase 3 | 3.B | 1.F, 2.A–2.B, 2.D |
| kube-prometheus-stack v82.x + Loki v3.x + Tempo v2.9+ | Fase 3 | 3.C | 1.A |
| Alertmanager v0.27+ → Telegram | Fase 3 | 3.D | 3.C |

### Correspondencia diagrama → fase

| Diagrama | Componentes principales | Fases representadas |
|----------|------------------------|---------------------|
| `arch.puml` | Vista global de toda la arquitectura | Fases 1–3 (estado final target) |
| `comp_iam_capa_core.puml` | Detalle del IAM Bounded Context | Fase 1 (1.E, 1.F) + Fase 3 (3.A) |
| `comp_infra_operaciones.puml` | Topología Kubernetes + Talos Linux v4.0 | Fases 1–3 completo |
