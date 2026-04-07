# Informe Challenger v3: Stack Final Bajo Restricciones Reales
## PRO + Costo Mínimo + Open Source + Control Total + No Reinventar la Rueda

**Proyecto:** Serenamente — Clínica Digital de Salud Mental Global  
**Versión:** 3.0 — Revisión final bajo todas las restricciones correctas  
**Fecha:** Abril 2026  
**Autor:** djca / Roo Architect Mode

> **⚠️ ACTUALIZACIÓN POST-CHALLENGER (Abril 2026):**
> La Sección 1 de este documento ("VPS + Docker Compose + Caddy") representa la decisión evaluada
> en el análisis challenger, pero fue **superada** por la decisión ADR-010/ADR-011/ADR-012/ADR-013.
> La decisión final es: **Talos Linux v1.10.x + Kubernetes 1.33.x** (no Docker Compose),
> **cert-manager + Traefik v3.x** (no Caddy), **CloudNativePG v1.x** (no postgres:17-alpine directo),
> **Hetzner CX32** (no CX31).
> Ver decisión completa y justificación en [`plans/06_talos_k8s_decision_y_cambios_en_cadena.md`](./06_talos_k8s_decision_y_cambios_en_cadena.md).

---

## Las Tres Restricciones Activas — Definitivas

```
RESTRICCIÓN 1: Equipo PRO
  El solopreneur domina todas las tecnologías involucradas.
  Go, Protobuf, NATS, PostgreSQL, OpenEHR, FHIR, Zanzibar, Docker — TODO.
  La dificultad técnica NO es un factor de decisión.

RESTRICCIÓN 2: Costo Mínimo + Open Source + Control Total
  Se PREFIERE software open source siempre que sea posible.
  Se PREFIERE autoalojado (self-hosted) sobre managed services de pago.
  El control total del sistema y los datos es un valor en sí mismo.
  El gasto mensual de servicios SaaS es el enemigo a minimizar.

RESTRICCIÓN 3: No Reinventar la Rueda
  Si existe una librería, framework o herramienta open source probada
  que resuelva el problema, se usa. No se construye desde cero lo que ya existe.
  Esto aplica tanto a nivel de código como de infraestructura.
```

### Tabla de Pesos de Criterios v3.0

| Criterio | Peso | Justificación |
|----------|------|---------------|
| **Costo total de propiedad** | ★★★★★★ | Restricción primaria — minimizar gastos mensuales |
| **Open Source / Control total** | ★★★★★★ | Restricción primaria — soberanía del sistema y datos |
| **No reinventar la rueda** | ★★★★★ | Usar soluciones probadas que ya resuelven el problema |
| **Anti vendor lock-in** | ★★★★★ | Dato médico permanente, self-hosted = máximo control |
| **Performance P99** | ★★★★☆ | Importante pero secundario al costo |
| **Madurez / Estabilidad** | ★★★★☆ | Herramientas probadas en producción |
| ~~Curva de aprendizaje~~ | N/A | **ELIMINADO** — equipo PRO |

> **Principio rector:** Un VPS de $10/mes con NATS + PostgreSQL + todos los microservicios Go es $120/año. El equivalente en servicios managed puede costar $200-500/mes. Con equipo PRO capaz de operar todo, esa diferencia es simplemente margen de negocio perdido.

---

## Índice

1. [El Modelo de Despliegue Base: VPS + Docker Compose + Caddy](#1-el-modelo-de-despliegue-base-vps--docker-compose--caddy)
2. [Challenger: Event Bus — NATS Self-Hosted Confirmado](#2-challenger-event-bus--nats-self-hosted-confirmado)
3. [Challenger: Serialización — Protobuf Confirmado](#3-challenger-serialización--protobuf-confirmado)
4. [Challenger: Persistencia — PostgreSQL Self-Hosted Confirmado](#4-challenger-persistencia--postgresql-self-hosted-confirmado)
5. [Challenger: IAM — Kratos + OpenFGA Decidido](#5-challenger-iam--kratos--openfga-decidido)
6. [Challenger: API Gateway — Cloudflare Workers + Caddy](#6-challenger-api-gateway--cloudflare-workers--caddy)
7. [Challenger: Frontend — Qwik + Astro Confirmados](#7-challenger-frontend--qwik--astro-confirmados)
8. [Challenger: Backend — Go Confirmado con Especificaciones](#8-challenger-backend--go-confirmado-con-especificaciones)
9. [Challenger: Estándares Médicos — OpenEHR como Paradigma](#9-challenger-estándares-médicos--openehr-como-paradigma)
10. [Challenger: Observabilidad — Stack OSS Completo](#10-challenger-observabilidad--stack-oss-completo)
11. [El Stack Target Definitivo v3.0](#11-el-stack-target-definitivo-v30)
12. [Análisis de Costo Total de Propiedad](#12-análisis-de-costo-total-de-propiedad)
13. [Tabla de Veredictos Finales v3.0](#13-tabla-de-veredictos-finales-v30)

---

## 1. El Modelo de Despliegue Base: VPS + Docker Compose + Caddy

### 1.1 La Premisa Central

Con un equipo PRO que prioriza costo y control total, el modelo de despliegue es **un VPS bien dimensionado con Docker Compose** para producción. No Kubernetes (overkill para el volumen inicial), no managed services (costo innecesario con equipo PRO).

```
Proveedor recomendado: Hetzner Cloud (mejor relación precio/performance en Europa y US)

FASE 1-2: 1 VPS único con todo
  Instancia: CX31 = 2 vCPU AMD, 4GB RAM, 80GB SSD = €8.99/mes (~$10/mes)
  
  En este VPS corre TODO:
  ├── nats              (JetStream message broker)
  ├── postgres          (PostgreSQL 17, 4 databases separadas)
  ├── ory-kratos        (AuthN service)
  ├── openfga           (AuthZ service — Zanzibar)
  ├── iam-service       (Go microservicio — dominio IAM)
  ├── scheduling-service(Go microservicio)
  ├── clinical-service  (Go microservicio)
  ├── billing-service   (Go microservicio)
  ├── grafana+prometheus+loki (observabilidad)
  └── caddy             (reverse proxy + TLS automático)

FASE 3: 2 VPS con Docker Swarm
  VPS-1: Servicios Go + NATS + Caddy       →  €8.99/mes
  VPS-2: PostgreSQL (réplica de lectura)   →  €5.99/mes
  Total: €15/mes

FASE 4+: k3s (Kubernetes ligero) en Hetzner
  Solo cuando el volumen justifique la complejidad

Si los datos clínicos deben estar en LATAM por regulación:
  → Kamatera (México, Brasil — centros de datos propios)
  → Vultr Santiago de Chile o São Paulo
  → DigitalOcean NYC (latencia ~80ms para LATAM — aceptable)
```

### 1.2 Caddy — No Reinventar la Rueda de TLS

```
Caddy (caddyserver.com):
  Licencia:  Apache 2.0
  Costo:     $0
  Función:   Reverse proxy con TLS automático vía Let's Encrypt
  Reemplaza: Nginx + Certbot + cron de renovación de certificados

Caddyfile completo para Serenamente:
  api.serenamente.com {
    # Forward auth: IAM Service valida JWT en cada request
    forward_auth /api/scheduling/* iam-service:8080 {
      uri /internal/validate-token
      copy_headers X-User-ID X-User-Role X-Tenant-ID
    }
    forward_auth /api/clinical/* iam-service:8080 {
      uri /internal/validate-token
      copy_headers X-User-ID X-User-Role
    }
    forward_auth /api/billing/* iam-service:8080 {
      uri /internal/validate-token
      copy_headers X-User-ID X-User-Role
    }
    
    # Routing a microservicios
    handle /api/scheduling/*  { reverse_proxy scheduling-service:8081 }
    handle /api/clinical/*    { reverse_proxy clinical-service:8082 }
    handle /api/billing/*     { reverse_proxy billing-service:8083 }
    
    # Rutas IAM (sin JWT validation — son los flujos de login)
    handle /auth/*            { reverse_proxy ory-kratos:4433 }
    handle /api/iam/*         { reverse_proxy iam-service:8080 }
    
    # Cabeceras de seguridad
    header {
      Strict-Transport-Security "max-age=31536000; includeSubDomains"
      X-Content-Type-Options "nosniff"
      X-Frame-Options "DENY"
    }
  }

→ TLS automático. Let's Encrypt gestiona los certificados.
→ Zero configuración manual de certificados.
→ No hay certbot, no hay cron de renovación.
→ Esto es NO reinventar la rueda: Caddy ya resuelve TLS + proxy.
```

---

## 2. Challenger: Event Bus — NATS Self-Hosted Confirmado

### 2.1 Veredicto Directo

**NATS JetStream self-hosted es la elección correcta y no cambia.**

### 2.2 El Análisis de Alternativas

```
¿Por qué no Redpanda Cloud?
  → Tiene costo variable: $0 en free tier (limitado) → $40-100+/mes en producción real
  → Con equipo PRO, operar NATS self-hosted es trivial
  → $0 vs $40-100/mes es una diferencia que no se justifica
  
¿Por qué no Redpanda Self-Hosted?
  → Redpanda es C++ thread-per-core: requiere más RAM y CPU que NATS
  → Para el volumen de Serenamente (miles de eventos/día, no millones):
    NATS es MÁS que suficiente y mucho más eficiente en recursos
  → El ecosistema Kafka (Flink, Connect) no se necesita en las fases iniciales
  → Path de migración: cuando el volumen lo justifique, migrar de NATS a Redpanda
    self-hosted = cambiar client library + connection string en los servicios Go
  
¿Por qué no Kafka/Apache?
  → JVM = 500MB-1GB de RAM solo para el broker
  → ZooKeeper o KRaft: complejidad operativa innecesaria
  → Overkill para el volumen médico inicial de Serenamente
  
NATS JetStream es la herramienta correcta:
  Licencia:    Apache 2.0
  Costo:       $0 (binario ~20MB)
  RAM idle:    ~50MB
  Throughput:  10M+ msgs/sec (overkill para Serenamente)
  Latencia:    P99 <1ms
  JetStream:   Persistencia durable en disco
  Clustering:  Soportado (3 nodos para HA en Fase 3)
  CNCF:        Proyecto graduado
```

### 2.3 Configuración de Producción

```yaml
# docker-compose.yml - NATS con JetStream
  nats:
    image: nats:2.10-alpine
    command:
      - "--jetstream"
      - "--store_dir=/data"
      - "--http_port=8222"
      - "--max_payload=10MB"
      - "--cluster_name=serenamente-prod"
      - "--tls"
      - "--tlscert=/certs/nats.crt"
      - "--tlskey=/certs/nats.key"
      - "--tlsca=/certs/ca.crt"
    volumes:
      - nats-data:/data
      - ./certs:/certs:ro
    restart: unless-stopped
    # Solo accesible desde la red Docker interna
    # NO exponer puerto 4222 al exterior

# Streams iniciales (ejecutar via nats CLI una vez):
# nats stream add SERENAMENTE_IAM      --subjects "iam.>"        --retention limits --max-age 365d  --storage file
# nats stream add SERENAMENTE_SCHED    --subjects "scheduling.>" --retention limits --max-age 730d  --storage file
# nats stream add SERENAMENTE_CLINICAL --subjects "clinical.>"   --retention limits --max-age 0     --storage file  # infinito
# nats stream add SERENAMENTE_BILLING  --subjects "billing.>"    --retention limits --max-age 3650d --storage file  # 10 años fiscal
```

---

## 3. Challenger: Serialización — Protobuf Confirmado

### 3.1 Veredicto

**Protocol Buffers v3 con buf.build es la elección correcta.** La propuesta de Avro (versión anterior) asumía tener Redpanda Cloud con Schema Registry incluido. Sin ese managed service, Protobuf es técnicamente superior para el setup self-hosted.

### 3.2 Por Qué Protobuf Gana en Self-Hosted

```
El argumento central:

Avro requiere Schema Registry para ser safe en producción:
  → Sin SR, productores y consumidores pueden desincronizarse silenciosamente
  → Con SR (Apicurio, Confluent open source): otro servicio a desplegar y mantener
  → Apicurio Registry: Java, ~512MB RAM adicionales
  
Protobuf NO requiere Schema Registry:
  → Schema evolution es NATIVA: campo nuevo = field number nuevo = backward compat
  → Campo eliminado = marcar como `reserved` = safe
  → Productores y consumidores evolucionan independientemente sin coordinación
  → El .proto file en Git ES el schema registry
  
En el contexto self-hosted:
  Avro:    necesita Schema Registry (componente adicional) + goavro library
  Protobuf: solo necesita buf.build CLI (free) + protobuf-go library
  
  Protobuf tiene MENOS componentes adicionales en el stack self-hosted.
  
Performance Protobuf vs JSON (para contexto):
  Tamaño: Protobuf ~40 bytes vs JSON ~100 bytes (60% más compacto)
  Speed:  Protobuf 5-10x más rápido que JSON
  → Para el event store de Clinical Record (BYTEA en PostgreSQL),
    Protobuf reduce el tamaño del event store significativamente
    
buf.build toolchain:
  → buf CLI: open source (BSL license, libre para uso)
  → buf.build registry: tier gratuito para proyectos individuales
  → buf generate: genera código Go desde .proto files
  → buf breaking: detecta breaking changes antes de hacer push
  → buf lint: valida convenciones en los schemas
```

### 3.3 Estructura del Monorepo para Protobuf

```
packages/events/
├── buf.yaml
├── buf.gen.yaml
├── buf.work.yaml
└── proto/
    ├── iam/v1/
    │   ├── user_registered.proto
    │   ├── doctor_onboarded.proto
    │   └── doctor_offboarded.proto
    ├── scheduling/v1/
    │   ├── appointment_booked.proto
    │   └── appointment_cancelled.proto
    ├── clinical/v1/
    │   ├── consultation_started.proto
    │   ├── diagnosis_recorded.proto
    │   └── consultation_finished.proto
    └── billing/v1/
        ├── payment_captured.proto
        └── invoice_generated.proto

# buf.gen.yaml — generación de código Go:
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen/go
    opt:
      - paths=source_relative
  - remote: buf.build/grpc/go
    out: gen/go
    opt:
      - paths=source_relative

# El código generado se commitea al monorepo
# → Sin paso de compilación en CI para consumir los tipos
# → Cada servicio Go importa desde packages/events/gen/go
```

---

## 4. Challenger: Persistencia — PostgreSQL Self-Hosted Confirmado

### 4.1 Veredicto

**PostgreSQL 17 self-hosted en el VPS es la elección correcta bajo costo-mínimo + open source.**

### 4.2 Análisis de Costo

```
Costo de 4 bases de datos PostgreSQL:

  Self-hosted en VPS existente:
    → $0 adicional (corre en el VPS de $10/mes ya presupuestado)
    → PostgreSQL 17 es open source (PostgreSQL License)
    → pgxpool funciona igual en cualquier PostgreSQL

  Neon (managed serverless): $0-76+/mes por 4 proyectos
  Railway (managed): $20-40/mes
  Supabase: $25-100+/mes
  AWS RDS: $50-200+/mes

Veredicto de costo:
  Self-hosted: $0 adicional vs managed: $20-200+/mes adicionales
  Con equipo PRO capaz de operar PostgreSQL, no hay justificación.
```

### 4.3 Patrón de Despliegue

```yaml
# docker-compose.yml — Fase 1-2: Una instancia, 4 databases separadas
  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: serenamente_admin
      POSTGRES_PASSWORD: ${PG_ADMIN_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./infra/postgres/init.sql:/docker-entrypoint-initdb.d/01_init.sql
    restart: unless-stopped
    # NO exponer puerto 5432 al exterior
    # Solo accesible desde la red Docker interna

# init.sql — crear databases y usuarios aislados:
CREATE DATABASE iam_db;
CREATE DATABASE scheduling_db;
CREATE DATABASE clinical_db;
CREATE DATABASE billing_db;

CREATE USER iam_user WITH PASSWORD '${IAM_DB_PASSWORD}';
CREATE USER scheduling_user WITH PASSWORD '${SCHEDULING_DB_PASSWORD}';
CREATE USER clinical_user WITH PASSWORD '${CLINICAL_DB_PASSWORD}';
CREATE USER billing_user WITH PASSWORD '${BILLING_DB_PASSWORD}';

-- Aislamiento completo: cada usuario solo accede a su propia DB
GRANT ALL PRIVILEGES ON DATABASE iam_db TO iam_user;
GRANT ALL PRIVILEGES ON DATABASE scheduling_db TO scheduling_user;
GRANT ALL PRIVILEGES ON DATABASE clinical_db TO clinical_user;
GRANT ALL PRIVILEGES ON DATABASE billing_db TO billing_user;

-- REVOKE acceso cruzado (RLS adicional a nivel de DB)
REVOKE ALL ON DATABASE iam_db FROM scheduling_user, clinical_user, billing_user;
REVOKE ALL ON DATABASE scheduling_db FROM iam_user, clinical_user, billing_user;
REVOKE ALL ON DATABASE clinical_db FROM iam_user, scheduling_user, billing_user;
REVOKE ALL ON DATABASE billing_db FROM iam_user, scheduling_user, clinical_user;
```

### 4.4 Backups — No Reinventar la Rueda

```bash
#!/bin/bash
# /opt/serenamente/scripts/backup-postgres.sh
# Cron: 0 2 * * * /opt/serenamente/scripts/backup-postgres.sh

set -euo pipefail

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR=/var/backups/serenamente/postgres
DOCKER_COMPOSE_DIR=/opt/serenamente

# Crear directorio de backup
mkdir -p $BACKUP_DIR

# Backup de cada database
for DB in iam_db scheduling_db clinical_db billing_db; do
    USER="${DB%_db}_user"
    echo "Backing up $DB..."
    docker compose -f $DOCKER_COMPOSE_DIR/docker-compose.yml \
        exec -T postgres \
        pg_dump -U $USER -d $DB \
        | gzip > $BACKUP_DIR/${DB}_${DATE}.sql.gz
done

# Subir a Backblaze B2 (rclone — open source, Apache 2.0)
# rclone es la alternativa open source a aws-cli para object storage
rclone copy $BACKUP_DIR serenamente-b2:serenamente-backups/postgres/

# Retener solo los últimos 30 días localmente
find $BACKUP_DIR -name "*.sql.gz" -mtime +30 -delete

echo "Backup completado: $DATE"
```

```
Costo de Backblaze B2:
  → $0.006/GB/mes almacenado (el más barato del mercado)
  → $0.01/GB de descarga (solo se paga al restaurar)
  → Para 50GB de datos clínicos comprimidos: ~15GB backups → $0.09/mes
  → Prácticamente $0

Alternativa si los datos deben estar en Europa:
  → Wasabi (€0.006/GB/mes, GDPR compliant, Frankfurt region)
  → O Hetzner Storage Box (1TB por €3.81/mes, incluye SFTP)
```

---

## 5. Challenger: IAM — Kratos + OpenFGA Decidido

### 5.1 El Único Cambio Real Respecto al Target Original

El target original usa **Ory Keto** para AuthZ (autorización). La propuesta es reemplazarlo con **OpenFGA** — sin cambiar nada más del stack IAM.

### 5.2 Por Qué OpenFGA sobre Ory Keto

```
Ambos son open source. Ambos implementan Google Zanzibar. Ambos son $0.
La diferencia es técnica y de DX:

Ory Keto:
  → Paradigma Zanzibar correcto
  → API de relation tuples funcional
  → Namespace config en YAML (puede ser verboso)
  → Comunidad más pequeña (a Abril 2026)
  → Últimas releases con menor cadencia que OpenFGA
  
OpenFGA:
  → Paradigma Zanzibar correcto (misma semántica)
  → DSL de modelado más expresivo:
      type clinical_record
        relations
          define owner: [patient]
          define assigned_doctor: [doctor]
          define can_write: assigned_doctor
          define can_read: owner or assigned_doctor or (assigned_doctor but not suspended)
  → Playground oficial: https://play.fga.dev (modelar sin escribir código)
  → SDK Go oficial: github.com/openfga/go-sdk
  → Backing: Okta/Auth0 → mayor longevidad y recursos de desarrollo
  → Watch API: stream de cambios de relaciones (útil para invalidar caches)
  → Comunidad más activa (2024-2026)
  → Docker: openfga/openfga:latest (binario Go, ~50MB RAM)
  
Migración de Keto a OpenFGA: los conceptos son idénticos.
  Keto "relation tuple" = OpenFGA "tuple"
  Keto "namespace" = OpenFGA "type"
  Keto "check" = OpenFGA "check"
  La sintaxis del SDK cambia; la arquitectura no.
```

### 5.3 Stack IAM Completo en Docker

```yaml
# docker-compose.yml — stack IAM completo
  ory-kratos:
    image: oryd/kratos:v1.3
    depends_on: [postgres]
    environment:
      DSN: postgres://kratos_user:${KRATOS_DB_PASSWORD}@postgres:5432/kratos_db
      SECRETS_COOKIE: ${KRATOS_COOKIE_SECRET}
      SECRETS_CIPHER: ${KRATOS_CIPHER_SECRET}
    volumes:
      - ./infra/kratos/kratos.yml:/etc/config/kratos/kratos.yml:ro
      - ./infra/kratos/identity_schemas:/etc/config/kratos/schemas:ro
    command: serve -c /etc/config/kratos/kratos.yml
    restart: unless-stopped

  openfga:
    image: openfga/openfga:latest
    depends_on: [postgres]
    environment:
      OPENFGA_DATASTORE_ENGINE: postgres
      OPENFGA_DATASTORE_URI: postgres://openfga_user:${OPENFGA_DB_PASSWORD}@postgres:5432/openfga_db
      OPENFGA_AUTHN_METHOD: preshared
      OPENFGA_AUTHN_PRESHARED_KEYS: ${OPENFGA_API_KEY}
    command: run
    restart: unless-stopped
    # Puertos 8080 (HTTP) y 8081 (gRPC) solo internos

  iam-service:
    image: serenamente/iam-service:latest
    depends_on: [ory-kratos, openfga, postgres, nats]
    environment:
      KRATOS_PUBLIC_URL: http://ory-kratos:4433
      KRATOS_ADMIN_URL: http://ory-kratos:4434
      OPENFGA_URL: http://openfga:8080
      OPENFGA_STORE_ID: ${OPENFGA_STORE_ID}
      DB_URL: postgres://iam_user:${IAM_DB_PASSWORD}@postgres:5432/iam_db
      NATS_URL: nats://nats:4222
      JWT_PRIVATE_KEY: ${JWT_ED25519_PRIVATE_KEY}  # Ed25519 para JWTs
    restart: unless-stopped
```

### 5.4 El Modelo de Autorización OpenFGA para Serenamente

```
# Modelo completo de relaciones médicas en OpenFGA DSL
model
  schema 1.1

type user

type organization
  relations
    define admin: [user]
    define member: [user] or admin

type doctor
  relations
    define user: [user]
    define organization: [organization]

type patient
  relations
    define user: [user]
    define primary_doctor: [doctor]
    define shared_doctor: [doctor]
    define can_access: primary_doctor or shared_doctor

type clinical_record
  relations
    define patient: [patient]
    define can_read: (user from patient) or (user from primary_doctor from patient) or (user from shared_doctor from patient)
    define can_write: (user from primary_doctor from patient)

type appointment
  relations
    define doctor: [doctor]
    define patient: [patient]
    define can_view: (user from doctor) or (user from patient)

# Relaciones de ejemplo:
# "Doctor Dr. Smith tiene acceso de escritura a historia HC-001 del Paciente P-456"
# → tuple: {user: "doctor:dr-smith", relation: "primary_doctor", object: "patient:p-456"}
# → tuple: {user: "patient:p-456", relation: "patient", object: "clinical_record:hc-001"}
# → Check: can Dr. Smith can_write on clinical_record:hc-001? → YES (via transitivity)
```

---

## 6. Challenger: API Gateway — Cloudflare Workers + Caddy

### 6.1 La Estrategia de Dos Capas

```
CAPA 1: Caddy (en el VPS) — proxy + routing + forward auth
  → TLS automático
  → Routing a microservicios Go
  → Forward auth: delega verificación JWT al IAM Service
  → Rate limiting (Caddy rate limit module)
  → $0 (incluido en el VPS)

CAPA 2: Cloudflare Workers con Hono.js — BFF para el frontend
  → El frontend Qwik ya está en Cloudflare Pages
  → Workers son GRATUITOS para el volumen inicial (100K req/día)
  → Transforma requests del frontend antes de llegar al VPS
  → Puede hacer edge caching de responses públicas
  → El código Hono es 100% portable a cualquier runtime
  → $0 hasta 100K req/día; $5/mes para 10M req/mes si supera el free tier

¿Por qué no un Gateway dedicado en el VPS?
  → Con Caddy haciendo forward_auth, ya tenemos validación de JWT en cada ruta
  → Añadir un proceso de Gateway separado (Kong, Envoy) sería reinventar la rueda
  → Kong es pesado (Java + PostgreSQL adicional para plugins)
  → Envoy es complejo (YAML extenso, curva alta incluso para PRO)
  → Caddy + Cloudflare Workers cubren el 95% de los casos de uso de un API Gateway
    para el volumen de Serenamente
```

### 6.2 El BFF en Cloudflare Workers

```typescript
// apps/bff/src/index.ts
import { Hono } from 'hono'
import { jwt } from 'hono/jwt'
import { cors } from 'hono/cors'

type Bindings = {
  VPS_API_URL: string
  JWT_PUBLIC_KEY: string
}

const app = new Hono<{ Bindings: Bindings }>()

app.use('*', cors({ origin: ['https://app.serenamente.com'] }))

// Verificar JWT emitido por el IAM Service
app.use('/api/*', (c, next) => {
  return jwt({ secret: c.env.JWT_PUBLIC_KEY, alg: 'EdDSA' })(c, next)
})

// Proxy a los microservicios vía el VPS (a través de Caddy)
app.all('/api/*', async (c) => {
  const url = new URL(c.req.url)
  const targetUrl = `${c.env.VPS_API_URL}${url.pathname}${url.search}`
  
  return fetch(targetUrl, {
    method: c.req.method,
    headers: {
      ...Object.fromEntries(c.req.raw.headers),
      'X-Forwarded-For': c.req.header('CF-Connecting-IP') ?? '',
    },
    body: c.req.raw.body,
  })
})

export default app

// wrangler.toml para deploy:
// [vars]
// VPS_API_URL = "https://api.serenamente.com"
```

---

## 7. Challenger: Frontend — Qwik + Astro Confirmados

```
Bajo las restricciones de costo-mínimo + open source:

Qwik (portal médico + paciente):
  Licencia:  MIT
  Costo:     $0 (Cloudflare Pages tier gratuito)
  Razón:     Resumability → TTI ~50ms → mejor UX en móviles LATAM
  Deploy:    git push → CI → Cloudflare Pages build → deploy automático

Astro 5 (landing page + marketing):
  Licencia:  MIT
  Costo:     $0 (mismo Cloudflare Pages)
  Razón:     Lighthouse 99.2 → mejor SEO → más conversiones → más ingresos
  Deploy:    mismo pipeline, proyecto separado en Cloudflare Pages

Cloudflare Pages:
  Tier gratuito:   Proyectos ilimitados, 500 builds/mes, bandwidth unlimited
  Costo:           $0 para el volumen inicial de Serenamente
  
  Si supera el tier gratuito: Cloudflare Pages Pro = $25/mes
  → Incluye 5,000 builds/mes, analytics avanzado

VEREDICTO: Qwik + Astro en Cloudflare Pages. Ambos open source. Costo $0.
```

---

## 8. Challenger: Backend — Go Confirmado con Especificaciones

```
Con equipo PRO y restricción de costo-mínimo:

Go vs Rust:
  → Rust tiene mejor performance absoluta
  → Para el dominio médico (IO-bound vs CPU-bound):
    La consulta SQL tarda 1-5ms. La serialización Protobuf tarda <0.1ms.
    La diferencia de runtime entre Go y Rust es IRRELEVANTE en este contexto.
  → Rust no tiene librerías FHIR/OpenEHR maduras
  → Go tiene un ecosistema médico incipiente pero funcional
  → Tiempo de implementación en Go es 2-3x más rápido que Rust
  → Para solopreneur: velocidad de implementación = velocidad de negocio
  → VEREDICTO: Go es la elección correcta

Librerías Go confirmadas (no reinventar la rueda):
  github.com/jackc/pgx/v5/pgxpool      → Driver PostgreSQL de alta performance
  github.com/nats-io/nats.go           → Cliente NATS oficial
  google.golang.org/protobuf           → Protobuf (librería oficial Google)
  github.com/openfga/go-sdk            → Cliente OpenFGA oficial
  github.com/ory/kratos-client-go      → Cliente Kratos oficial
  github.com/golang-jwt/jwt/v5         → JWT signing/verification
  github.com/google/uuid               → UUIDv7 nativo desde v4.0.0+
  github.com/go-chi/chi/v5             → HTTP router minimalista (2K stars)
  github.com/golang-migrate/migrate/v4 → Migraciones de PostgreSQL
  go.opentelemetry.io/otel             → OpenTelemetry para tracing
  github.com/prometheus/client_golang  → Métricas Prometheus

Framework HTTP: Chi en lugar de stdlib puro
  → Chi provee routing con params, grupos, middlewares
  → Sin magia: implementa net/http.Handler estándar
  → No reinventar: Chi resuelve el routing sin overhead

Estructura de cada microservicio Go:
  services/scheduling/
  ├── cmd/main.go          → Entry point
  ├── internal/
  │   ├── domain/          → Entidades y lógica de negocio pura
  │   ├── app/             → Casos de uso / Application services
  │   ├── adapters/        → HTTP handlers (Chi), NATS consumers
  │   └── infra/           → pgxpool, NATS connection, config
  ├── migrations/          → SQL migrations (golang-migrate)
  └── Dockerfile           → Multi-stage build (builder + alpine)
```

---

## 9. Challenger: Estándares Médicos — OpenEHR como Paradigma

### 9.1 OpenEHR + FHIR Confirmados

```
La dualidad OpenEHR-as-storage + FHIR-as-API sigue siendo correcta.
No hay alternativa mejor para el dominio clínico a largo plazo.
```

### 9.2 La Decisión Crítica: Sin Servidor OpenEHR Hasta Fase 4

```
Opción A: Desplegar EHRbase (servidor openEHR open source, Java)
  → Provee AQL (Archetype Query Language) para queries clínicas
  → Gestiona arquetipos automáticamente
  PERO:
  → Java runtime: ~500MB-1GB de RAM solo para EHRbase
  → En el VPS de 4GB: 12-25% del RAM total solo para el servidor OpenEHR
  → Complejidad: AQL es un lenguaje adicional que aprender y mantener
  → Otro proceso en Docker Compose que monitorear y actualizar
  → Para Serenamente Fase 1-3: la complejidad no vale el beneficio

Opción B: OpenEHR como paradigma de modelado (sin servidor openEHR)
  → Los eventos clínicos se MODELAN con arquetipos openEHR:
    el payload es openEHR Canonical JSON (formato estándar)
  → Se almacenan como BYTEA (Protobuf que serializa el Canonical JSON)
    en el event store de PostgreSQL
  → Las proyecciones/read models son tablas PostgreSQL normales
  → No se necesita AQL: las queries son SQL estándar sobre los read models
  → No se necesita un servidor openEHR
  → Costo adicional: $0

Estructura del evento clínico (paradigma openEHR sin servidor):
  {
    "_type": "COMPOSITION",
    "archetype_id": "openEHR-EHR-COMPOSITION.encounter.v1",
    "uid": "01HV3K9P5N7Q8R6M4J2X0WBY3Z",  # UUIDv7
    "composer": {"_type": "PARTY_PROXY", "external_ref": {"id": {"value": "doctor-uuid"}}},
    "content": [{
      "_type": "EVALUATION",
      "archetype_id": "openEHR-EHR-EVALUATION.problem_diagnosis.v1",
      "data": {
        "_type": "ITEM_TREE",
        "items": [{
          "_type": "ELEMENT",
          "archetype_node_id": "at0002",
          "value": {
            "_type": "DV_CODED_TEXT",
            "value": "Trastorno depresivo mayor, episodio único",
            "defining_code": {
              "terminology_id": {"value": "ICD-11"},
              "code_string": "6A70"
            }
          }
        }]
      }
    }]
  }

→ Este JSON se serializa en Protobuf y se guarda en BYTEA en PostgreSQL
→ Una proyección materializada en PostgreSQL extrae los campos clave para queries:
  CREATE TABLE clinical_diagnoses_projection (
    event_id UUID PRIMARY KEY,  -- UUIDv7
    patient_id UUID NOT NULL,
    doctor_id UUID NOT NULL,
    icd11_code TEXT NOT NULL,
    diagnosis_name TEXT NOT NULL,
    recorded_at TIMESTAMPTZ NOT NULL
  );
→ El Clinical Record Service mantiene esta proyección en sync con el event store
→ El médico hace queries en SQL estándar, no en AQL

VEREDICTO: OpenEHR como paradigma de modelado, PostgreSQL como storage,
SQL como query language. Sin EHRbase hasta Fase 4+.
```

---

## 10. Challenger: Observabilidad — Stack OSS Completo

### 10.1 El Stack Estándar Open Source — No Reinventar la Rueda

```
PLG Stack (Prometheus + Loki + Grafana) + Tempo:

Componente     Función              Licencia        Costo
Prometheus     Métricas (scrape)   Apache 2.0      $0
Loki           Log aggregation     AGPL 3.0        $0 (self-hosted)
Promtail       Log collector       Apache 2.0      $0
Tempo          Distributed tracing Apache 2.0      $0
Grafana        Dashboards unified  AGPL 3.0        $0 (self-hosted)
Alertmanager   Alertas             Apache 2.0      $0

Dashboards pre-construidos (NO reinventar la rueda):
  → Grafana Dashboard ID 14981: NATS JetStream overview
  → Grafana Dashboard ID 9628:  PostgreSQL overview
  → Grafana Dashboard ID 12708: Go microservices overview
  → Grafana Dashboard ID 3662:  Prometheus 2.0 stats
  Importar estos IDs en Grafana → dashboards completos sin escribir una línea de PromQL

Instrumentación en Go (no reinventar la rueda):
  En cada microservicio Go, añadir:
  import "github.com/prometheus/client_golang/prometheus/promhttp"
  
  // En main.go:
  http.Handle("/metrics", promhttp.Handler())
  // → Prometheus scrape automáticamente todas las métricas de Go runtime
  
  // Para tracing:
  import "go.opentelemetry.io/otel"
  // → OpenTelemetry instrumenta el http.Client y el pgxpool automáticamente
```

### 10.2 Alertas para el Solopreneur

```
Alertas críticas (Alertmanager → Telegram/Email):
  
  - PostgreSQL: Tamaño del event store clínico > 50% del disco
  - PostgreSQL: Queries lentas > 1s en producción
  - NATS: Consumidores con lag > 1000 mensajes
  - NATS: Errores de publicación
  - Servicios Go: Error rate > 1% en endpoints críticos
  - Servicios Go: Memoria > 80% del límite del container
  - VPS: CPU > 80% por más de 5 minutos
  - VPS: Disco > 80% utilización
  - Caddy: Certificado TLS a menos de 7 días de vencer (aunque Let's Encrypt renueva solo)
  - Backup: El cron de backup no corrió en las últimas 25h
  
Alertas enviadas a Telegram (bot gratuito):
  → Telegram bot API es gratuita
  → Alertmanager tiene webhook receiver nativo
  → El solopreneur recibe alertas en el móvil sin costo adicional
  → No se necesita PagerDuty, Opsgenie, ni nada pagado
```

---

## 11. El Stack Target Definitivo v3.0

```
══════════════════════════════════════════════════════════════════
  SERENAMENTE — STACK TARGET v3.0 (Abril 2026)
  Restricciones: PRO + Costo Mínimo + Open Source + No Reinventar
══════════════════════════════════════════════════════════════════

INFRAESTRUCTURA BASE (~$10/mes):
  └── 1 VPS Hetzner CX31 (2 vCPU, 4GB RAM, 80GB) = €8.99/mes
      Docker Compose para orchestration
      Caddy para reverse proxy + TLS automático (Let's Encrypt)
      Docker volumes para persistencia de datos

CAPA DE PRESENTACIÓN ($0 — Cloudflare Pages, tier gratuito):
  ├── Qwik SPA → Portal médico + portal paciente (alta interactividad)
  └── Astro 5 → Landing page + marketing + SEO (Lighthouse 99.2)

API GATEWAY / BFF ($0 — Cloudflare Workers, tier gratuito hasta 100K req/día):
  ├── Hono.js en Cloudflare Workers → JWT verify + edge routing para el frontend
  └── Caddy en VPS → reverse proxy + forward_auth + routing interno

IAM — $0 (self-hosted en VPS via Docker):
  ├── Ory Kratos v1.3 → AuthN: Passkeys/WebAuthn, sesiones, MFA, flows
  ├── OpenFGA → AuthZ: Zanzibar relation tuples [CAMBIO: reemplaza Ory Keto]
  └── IAM Domain Service (Go + Chi) → JWT enriquecido + UUIDv7 + DIDs + Outbox

MICROSERVICIOS CORE (Go + Chi + pgxpool) — $0 (Docker en VPS):
  ├── Scheduling Service → FHIR Appointment + EXCLUDE GIST + NATS Outbox
  ├── Clinical Record Service → Event Store puro (openEHR Canonical JSON)
  └── Billing & Ops Service → Transacciones financieras + multi-pasarela

EVENT BUS — $0 (NATS 2.10 self-hosted en VPS):
  └── NATS JetStream
      Streams persistidos en disco, retención por dominio:
      iam: 1 año | scheduling: 2 años | clinical: indefinido | billing: 10 años

SERIALIZACIÓN — $0:
  └── Protocol Buffers v3 + buf.build (CLI open source, tier gratuito)
      Schema evolution nativa, sin Schema Registry adicional

PERSISTENCIA — $0 (PostgreSQL 17 self-hosted en VPS):
  └── 1 instancia PostgreSQL, 4 databases aisladas (Fase 1-2)
      Separar en 4 instancias Docker al escalar (Fase 3+)
      pg_dump + rclone → Backblaze B2 (~$0.05/mes para <10GB)

PATRONES ARQUITECTÓNICOS (sin cambio respecto al target original):
  ├── Event Sourcing puro → Clinical Record (BYTEA Protobuf, append-only)
  ├── Outbox Pattern → publicación atómica en misma transacción PG
  ├── UUIDv7 → primary keys en todos los servicios
  ├── Row-Level Security → PostgreSQL RLS alimentado por JWT enriquecido
  └── Database-per-service → via databases separadas en PG (misma instancia)

ESTÁNDARES MÉDICOS:
  ├── FHIR R4 → Output APIs + interoperabilidad externa
  └── openEHR Canonical JSON → formato de eventos clínicos (paradigma)
      Sin servidor EHRbase hasta Fase 4+ [SIMPLIFICACIÓN respecto al target original]

OBSERVABILIDAD — $0 (self-hosted en VPS):
  ├── Prometheus → métricas de todos los servicios + PG + NATS
  ├── Loki + Promtail → logs de containers Docker
  ├── Tempo → distributed tracing vía OpenTelemetry
  ├── Grafana → dashboards (importados desde grafana.com, no construidos manualmente)
  └── Alertmanager → alertas a Telegram (gratuito)

TLS / SEGURIDAD — $0:
  └── Caddy + Let's Encrypt → TLS automático, sin certbot, sin cron

HERRAMIENTAS DE DESARROLLO — $0:
  ├── buf.build CLI → gestión de .proto files
  ├── nats CLI → debugging de mensajes y streams
  ├── golang-migrate → migraciones de PostgreSQL
  └── Docker Compose → entorno local = producción (paridad perfecta)
```

---

## 12. Análisis de Costo Total de Propiedad

### 12.1 Costo Mensual por Fases

```
FASE 0 (MVP actual en Cloudflare Pages + Turso):
  Cloudflare Pages:     $0 (tier gratuito)
  Turso DB:             $0 (tier gratuito)
  ─────────────────────────────────────────
  TOTAL:                $0/mes

FASE 1 (VPS + stack completo sin observabilidad avanzada):
  Hetzner CX31:         €8.99/mes  (~$10/mes)
  Cloudflare Pages:     $0 (tier gratuito)
  Cloudflare Workers:   $0 (tier gratuito: 100K req/día)
  Backblaze B2 backups: $0.05/mes
  Dominio DNS:          ~$1/mes (ya existente)
  ─────────────────────────────────────────
  TOTAL:                ~$11/mes

FASE 2 (mismo VPS, más servicios activos):
  Hetzner CX41 (upgrade): €15.90/mes (~$17/mes)
  ─────────────────────────────────────────
  TOTAL:                ~$18/mes

FASE 3 (2 VPS: servicios + PostgreSQL dedicado):
  Hetzner CX41 (servicios):    €15.90/mes
  Hetzner CX21 (PostgreSQL):   €4.99/mes
  Cloudflare Workers Paid:     $5/mes (si supera 100K req/día)
  ─────────────────────────────────────────
  TOTAL:                ~$27/mes

FASE 4+ (múltiples VPS + servicios adicionales):
  3-5 VPS Hetzner:             €25-50/mes
  Dominio + extras:            $5/mes
  ─────────────────────────────────────────
  TOTAL:                ~$30-60/mes
```

### 12.2 Comparación con el Stack Managed Equivalente

```
Si se hubieran tomado decisiones de managed services:
  Redpanda Cloud (Dedicated):    $200+/mes
  Neon Scale (4 proyectos):      $276/mes
  Zitadel Cloud Pro:             $49/mes
  Datadog (observabilidad):      $100+/mes
  ─────────────────────────────────────────
  TOTAL managed:                 ~$625+/mes

Stack v3.0 self-hosted:          ~$11-60/mes (según fase)

Ahorro anual (vs managed):       $6,780-7,380/año
En 5 años:                       $33,900-36,900 de diferencia

Esta diferencia = el presupuesto para contratar un especialista
en informática médica para los arquetipos openEHR, o para marketing
del negocio médico.
```

---

## 13. Tabla de Veredictos Finales v3.0

| Componente | Target Original | Veredicto v3.0 | Tipo |
|------------|----------------|----------------|------|
| **Infraestructura base** | Sin definir | VPS Hetzner + Docker Compose | ADICIÓN CRÍTICA |
| **TLS** | Sin definir | Caddy + Let's Encrypt | ADICIÓN |
| **Frontend App** | Qwik | Qwik (Cloudflare Pages, $0) | CONFIRMAR |
| **Landing Page** | Implícito en Qwik | Astro 5 (Cloudflare Pages, $0) | ADICIÓN |
| **API Gateway** | [3] Genérico (Fase 3) | Caddy + Hono/CF Workers (Fase 0.5, $0) | ADELANTAR + ESPECIFICAR |
| **IAM AuthN** | Ory Kratos | Ory Kratos v1.3 (self-hosted) | CONFIRMAR |
| **IAM AuthZ** | **Ory Keto** | **OpenFGA** (self-hosted) | **CAMBIO** |
| **IAM Domain** | Go Service | Go + Chi + pgxpool | CONFIRMAR + ESPECIFICAR |
| **Backend** | Go | Go + Chi + pgxpool | CONFIRMAR + ESPECIFICAR |
| **Event Bus** | NATS JetStream | NATS JetStream 2.10 (self-hosted) | CONFIRMAR + ESPECIFICAR |
| **Serialización** | Protobuf | Protobuf v3 + buf.build | CONFIRMAR + ESPECIFICAR |
| **Persistencia** | PostgreSQL ×4 | PostgreSQL 17 self-hosted (1 instancia, 4 DBs) | CONFIRMAR + SIMPLIFICAR inicio |
| **Backups** | Sin definir | pg_dump + Backblaze B2 (~$0) | ADICIÓN CRÍTICA |
| **OpenEHR server** | Implícito | Sin EHRbase (openEHR como paradigma en JSON) | SIMPLIFICAR |
| **FHIR R4** | APIs externas | FHIR R4 output APIs | CONFIRMAR |
| **Observabilidad** | Sin definir | Prometheus + Loki + Tempo + Grafana (self-hosted) | ADICIÓN CRÍTICA |
| **UUIDv7** | UUIDv7 | UUIDv7 (PG17 + github.com/google/uuid) | CONFIRMAR |
| **RLS** | PostgreSQL RLS | PostgreSQL RLS via JWT enriquecido | CONFIRMAR |
| **Outbox Pattern** | Outbox → NATS | Outbox → Protobuf → NATS | CONFIRMAR |
| **Event Sourcing** | Append-only | BYTEA (Protobuf) append-only en PG | CONFIRMAR |

### El Único Cambio Real al Target Original

```
De todos los componentes del target original, solo hay UN cambio técnico real:

  Ory Keto → OpenFGA

Todo lo demás es:
  - Confirmación (el target original estaba bien elegido)
  - Adición de componentes que estaban "sin definir" (infraestructura, observabilidad)
  - Simplificación del inicio (openEHR como paradigma sin servidor EHRbase)
  - Especificación de detalles de implementación (Chi en lugar de HTTP stdlib puro)

Esto valida que el target original fue diseñado correctamente.
El challenger solo afinó los detalles de implementación.
```

---

*Documento v3.0 generado en Abril 2026.*  
*Aplica las restricciones definitivas: equipo PRO + costo mínimo + open source + control total + no reinventar la rueda.*  
*Siguiente: [`04_gap_map_y_ruta_migracion.md`](./04_gap_map_y_ruta_migracion.md)*
