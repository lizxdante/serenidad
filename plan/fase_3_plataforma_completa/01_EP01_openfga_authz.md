# EP-01 — OpenFGA v1.x AuthZ (Zanzibar ReBAC)

**Épica**: Como desarrollador de plataforma médica, necesito implementar autorización granular basada en relaciones (ReBAC) usando OpenFGA v1.x con modelo Zanzibar para controlar acceso a registros clínicos, citas y datos sensibles con delegación care_team, cumpliendo requisitos HIPAA de acceso basado en roles y relaciones.

**Origen**: Fase 3 (3.A) + research/09_evaluacion_tecnologica_fase3_plataforma_completa.md  
**Prioridad**: Crítica  
**Sprint**: S7 (2 semanas)  
**Dependencias Entrantes**: Fase 1 (IAM Service + JWT), CloudNativePG  
**Dependencias Salientes**: EP-02 (Billing necesita AuthZ para invoices)

---

## HU-01.1 — Deployment OpenFGA Kubernetes

**Como** SRE, **quiero** desplegar OpenFGA cluster con 3 replicas HA y PostgreSQL backend externo, **para que** tengamos authorization service resiliente con <5ms p99 latency.

### Tareas

#### T-01.1.1 — Preparar PostgreSQL Database

```sql
-- En CloudNativePG cluster existente (Fase 1)
CREATE DATABASE openfga_db
  WITH OWNER = serenidade_admin
  ENCODING = 'UTF8'
  LC_COLLATE = 'en_US.UTF-8'
  LC_CTYPE = 'en_US.UTF-8';

GRANT ALL PRIVILEGES ON DATABASE openfga_db TO serenidade_admin;
```

**CA**: Database creada, accesible desde cluster Kubernetes

#### T-01.1.2 — Crear Kubernetes Secret

```bash
kubectl create secret generic openfga-postgres-credentials \
  --from-literal=uri='postgres://serenidade_admin:PASSWORD@postgres.databases.svc:5432/openfga_db?sslmode=require' \
  -n serenidad
```

**CA**: Secret creado, verificado con `kubectl get secret`

#### T-01.1.3 — Helm Chart Values

Crear `infrastructure/k8s/openfga/values.yaml`:

```yaml
replicaCount: 3

datastore:
  engine: postgres
  existingSecret: "openfga-postgres-credentials"
  uriSecret: "uri"

resources:
  requests:
    cpu: 2000m
    memory: 4Gi
  limits:
    cpu: 4000m
    memory: 8Gi

startupProbe:
  enabled: true
  initialDelaySeconds: 60
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 6

livenessProbe:
  enabled: true
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  enabled: true
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values:
                  - openfga
          topologyKey: kubernetes.io/hostname

serviceMonitor:
  enabled: true
  interval: 30s

env:
  OPENFGA_DATASTORE_MAX_OPEN_CONNS: "300"
  OPENFGA_LOG_FORMAT: "json"
  OPENFGA_MAX_CONCURRENT_READS_FOR_LIST_OBJECTS: "10"
  OPENFGA_MAX_CONCURRENT_READS_FOR_LIST_USERS: "10"
  OPENFGA_RESOLVE_NODE_LIMIT: "1000"
  OPENFGA_RESOLVE_NODE_BREADTH_LIMIT: "100"
```

**CA**: Values file creado con configuración production-ready

#### T-01.1.4 — Helm Install

```bash
helm repo add openfga https://openfga.github.io/helm-charts
helm repo update

helm install openfga openfga/openfga \
  --namespace serenidad \
  --values infrastructure/k8s/openfga/values.yaml \
  --version 0.2.x
```

**CA**: 3 pods running, probes healthy

#### T-01.1.5 — Service y NetworkPolicy

```yaml
# infrastructure/k8s/openfga/networkpolicy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: openfga-policy
  namespace: serenidad
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: openfga
  policyTypes:
    - Ingress
  ingress:
    # Solo servicios Go pueden llamar OpenFGA
    - from:
        - podSelector:
            matchLabels:
              app: scheduling-service
        - podSelector:
            matchLabels:
              app: clinical-service
        - podSelector:
            matchLabels:
              app: billing-service
      ports:
        - protocol: TCP
          port: 8080
```

**CA**: NetworkPolicy aplicada, conectividad validada

### Criterios Aceptación HU-01.1

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | 3 replicas OpenFGA running | `kubectl get pods -n serenidad \| grep openfga` |
| 2 | Probes healthy | Describe pod muestra probes passing |
| 3 | PostgreSQL connectivity | Logs no muestran connection errors |
| 4 | Service accessible | `curl http://openfga.serenidad.svc:8080/healthz` |
| 5 | ServiceMonitor scraping | Prometheus muestra target UP |

---

## HU-01.2 — Authorization Model Healthcare

**Como** security engineer, **quiero** definir authorization model Zanzibar para healthcare con patient ownership, care_team delegation y record viewing permissions, **para que** tengamos fine-grained access control compliant con HIPAA.

### Tareas

#### T-01.2.1 — Diseñar Authorization Model

Crear `infrastructure/openfga/authorization-model.yaml`:

```yaml
model_version: "1.1"
type_definitions:
  - type: user
  
  - type: organization
    relations:
      member:
        this: {}
      admin:
        this: {}
  
  - type: patient
    relations:
      owner:
        this: {}
      care_team:
        this: {}
      viewer:
        union:
          - this: {}
          - computedUserset:
              object: ""
              relation: care_team
  
  - type: clinical_record
    relations:
      patient:
        this: {}
      viewer:
        tupleToUserset:
          tupleset:
            object: ""
            relation: patient
          computedUserset:
            object: ""
            relation: viewer
      editor:
        tupleToUserset:
          tupleset:
            object: ""
            relation: patient
          computedUserset:
            object: ""
            relation: care_team
  
  - type: appointment
    relations:
      patient:
        this: {}
      provider:
        this: {}
      viewer:
        union:
          - computedUserset:
              object: ""
              relation: provider
          - tupleToUserset:
              tupleset:
                object: ""
                relation: patient
              computedUserset:
                object: ""
                relation: viewer
      canceller:
        union:
          - computedUserset:
              object: ""
              relation: provider
          - tupleToUserset:
              tupleset:
                object: ""
                relation: patient
              computedUserset:
                object: ""
                relation: owner
  
  - type: invoice
    relations:
      patient:
        this: {}
      viewer:
        union:
          - tupleToUserset:
              tupleset:
                object: ""
                relation: patient
              computedUserset:
                object: ""
                relation: owner
          - tupleToUserset:
              tupleset:
                object: ""
                relation: patient
              computedUserset:
                object: ""
                relation: care_team
```

**CA**: Model YAML creado siguiendo sintaxis OpenFGA

#### T-01.2.2 — Aplicar Model vía API

```bash
# Script apply-model.sh
#!/bin/bash
set -euo pipefail

OPENFGA_URL="http://openfga.serenidad.svc:8080"
MODEL_FILE="infrastructure/openfga/authorization-model.yaml"

# Convert YAML to JSON (OpenFGA API expects JSON)
yq eval -o=json "$MODEL_FILE" > /tmp/model.json

# Create store
STORE_ID=$(curl -s -X POST "$OPENFGA_URL/stores" \
  -H "Content-Type: application/json" \
  -d '{"name":"serenidad"}' | jq -r '.id')

echo "Store created: $STORE_ID"

# Upload authorization model
curl -X POST "$OPENFGA_URL/stores/$STORE_ID/authorization-models" \
  -H "Content-Type: application/json" \
  -d @/tmp/model.json

echo "Authorization model applied"
```

**CA**: Model aplicado exitosamente, store_id guardado

#### T-01.2.3 — Seed Test Tuples

```bash
# Script seed-test-data.sh
#!/bin/bash
OPENFGA_URL="http://openfga.serenidad.svc:8080"
STORE_ID="<STORE_ID>"

# Paciente P123 es owned por user:patient_p123
curl -X POST "$OPENFGA_URL/stores/$STORE_ID/write" \
  -H "Content-Type: application/json" \
  -d '{
    "writes": {
      "tuple_keys": [
        {
          "user": "user:patient_p123",
          "relation": "owner",
          "object": "patient:p123"
        },
        {
          "user": "user:dr_lopez",
          "relation": "care_team",
          "object": "patient:p123"
        },
        {
          "object": "clinical_record:rec_456",
          "relation": "patient",
          "user": "patient:p123"
        }
      ]
    }
  }'
```

**CA**: Test tuples insertadas exitosamente

#### T-01.2.4 — Validar Check Queries

```bash
# ¿Puede dr_lopez ver el record rec_456?
curl -X POST "$OPENFGA_URL/stores/$STORE_ID/check" \
  -H "Content-Type: application/json" \
  -d '{
    "tuple_key": {
      "user": "user:dr_lopez",
      "relation": "viewer",
      "object": "clinical_record:rec_456"
    }
  }'

# Esperado: { "allowed": true }
```

**CA**: Check query retorna `allowed: true` como esperado

### Criterios Aceptación HU-01.2

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Authorization model aplicado | API retorna 200 en POST model |
| 2 | Test tuples insertadas | Write tuples retorna 200 |
| 3 | Check queries funcionan | Check retorna allowed correctamente |
| 4 | Model versionado | YAML en Git con version tag |

---

## HU-01.3 — Go SDK Integration

**Como** backend developer, **quiero** integrar OpenFGA SDK en servicios Go con middleware para check automático en endpoints protegidos, **para que** autorización sea transparente y performante (<5ms).

### Tareas

#### T-01.3.1 — Install Go SDK

```bash
cd services/clinical
go get github.com/openfga/go-sdk@latest
```

**CA**: Dependency añadida a go.mod

#### T-01.3.2 — OpenFGA Client Factory

Crear `services/clinical/internal/authz/client.go`:

```go
package authz

import (
    "context"
    "os"
    
    openfga "github.com/openfga/go-sdk"
    "github.com/openfga/go-sdk/client"
)

type AuthZClient struct {
    fgaClient *client.OpenFgaClient
    storeID   string
}

func NewAuthZClient(ctx context.Context) (*AuthZClient, error) {
    configuration, err := client.NewConfiguration(client.Configuration{
        ApiUrl: os.Getenv("OPENFGA_API_URL"),
        StoreId: os.Getenv("OPENFGA_STORE_ID"),
    })
    if err != nil {
        return nil, err
    }
    
    fgaClient := client.NewOpenFgaClient(configuration)
    
    return &AuthZClient{
        fgaClient: fgaClient,
        storeID:   os.Getenv("OPENFGA_STORE_ID"),
    }, nil
}

func (c *AuthZClient) Check(ctx context.Context, user, relation, object string) (bool, error) {
    body := client.ClientCheckRequest{
        User:     user,
        Relation: relation,
        Object:   object,
    }
    
    data, err := c.fgaClient.Check(ctx).Body(body).Execute()
    if err != nil {
        return false, err
    }
    
    return data.GetAllowed(), nil
}
```

**CA**: Client factory implementado

#### T-01.3.3 — Authorization Middleware

Crear `services/clinical/internal/middleware/authz.go`:

```go
package middleware

import (
    "fmt"
    "net/http"
    "strings"
    
    "github.com/go-chi/chi/v5"
    "github.com/serenidad/clinical/internal/authz"
)

type AuthZMiddleware struct {
    authzClient *authz.AuthZClient
}

func NewAuthZMiddleware(client *authz.AuthZClient) *AuthZMiddleware {
    return &AuthZMiddleware{authzClient: client}
}

func (m *AuthZMiddleware) RequirePermission(objectType, relation string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Extract user_id from JWT (set by IAM middleware)
            userID := r.Context().Value("user_id").(string)
            
            // Extract resource ID from URL
            resourceID := chi.URLParam(r, "id")
            if resourceID == "" {
                http.Error(w, "Missing resource ID", http.StatusBadRequest)
                return
            }
            
            // Format OpenFGA entities
            user := fmt.Sprintf("user:%s", userID)
            object := fmt.Sprintf("%s:%s", objectType, resourceID)
            
            // Check authorization
            allowed, err := m.authzClient.Check(r.Context(), user, relation, object)
            if err != nil {
                http.Error(w, "Authorization check failed", http.StatusInternalServerError)
                return
            }
            
            if !allowed {
                http.Error(w, "Forbidden", http.StatusForbidden)
                return
            }
            
            next.ServeHTTP(w, r)
        })
    }
}
```

**CA**: Middleware implementado

#### T-01.3.4 — Apply Middleware to Routes

```go
// services/clinical/internal/handlers/router.go
func NewRouter(h *ClinicalHandler, authz *middleware.AuthZMiddleware) chi.Router {
    r := chi.NewRouter()
    
    r.Route("/v1/clinical-records", func(r chi.Router) {
        // GET requires viewer permission
        r.With(authz.RequirePermission("clinical_record", "viewer")).
            Get("/{id}", h.GetClinicalRecord)
        
        // POST requires editor permission
        r.With(authz.RequirePermission("clinical_record", "editor")).
            Post("/{id}/observations", h.AddObservation)
    })
    
    return r
}
```

**CA**: Rutas protegidas con middleware

### Criterios Aceptación HU-01.3

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | SDK instalado | go.mod contiene openfga/go-sdk |
| 2 | Client conecta a OpenFGA | Health check client exitoso |
| 3 | Middleware bloquea unauthorized | Request sin permission → 403 |
| 4 | Middleware permite authorized | Request con permission → 200 |
| 5 | Latency <5ms p99 | Prometheus muestra check_duration_ms |

---

## Resumen Entregables EP-01

**Infraestructura**:
- OpenFGA cluster (3 replicas, HA)
- PostgreSQL database (CloudNativePG)
- NetworkPolicy restrictiva
- ServiceMonitor Prometheus

**Authorization Model**:
- Healthcare model (patient, clinical_record, appointment, invoice)
- Care team delegation
- Tuple management scripts

**Go Integration**:
- SDK client factory
- Authorization middleware
- Protected routes (viewer, editor permissions)

**Validación**:
```bash
# 1. Verificar pods
kubectl get pods -n serenidad | grep openfga

# 2. Test check query
curl -X POST http://openfga.serenidad.svc:8080/stores/$STORE_ID/check \
  -d '{"tuple_key":{"user":"user:dr_lopez","relation":"viewer","object":"clinical_record:rec_456"}}'

# 3. Test middleware
curl https://api.sereni.dad/v1/clinical-records/rec_456 \
  -H "Authorization: Bearer $JWT_TOKEN"
  # → 200 si authorized, 403 si forbidden

# 4. Verificar latency
curl http://prometheus.monitoring.svc:9090/api/v1/query \
  -d 'query=histogram_quantile(0.99, openfga_check_duration_seconds_bucket)'
```

---

**Tiempo estimado EP-01**: 2 semanas (Sprint 7)  
**Esfuerzo**: ~60 horas ingeniería + 15 horas testing
