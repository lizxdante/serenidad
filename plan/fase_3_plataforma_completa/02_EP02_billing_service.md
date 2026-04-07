# EP-02 — Billing & Ops Service (Multi-Gateway LATAM)

**Épica**: Como equipo de producto, necesito un servicio de billing multi-gateway que soporte Stripe (global), Conekta (México) e Izipay (Perú/Colombia) con ledger financiero inmutable, idempotency, reconciliación automática y compliance PCI-DSS reducido mediante tokenization, para monetizar consultas médicas en LATAM y globalmente.

**Origen**: Fase 3 (3.B) + research/09_evaluacion_tecnologica_fase3_plataforma_completa.md  
**Prioridad**: Crítica  
**Sprint**: S8-S9 (2 semanas)  
**Dependencias Entrantes**: Fase 2 (NATS + Outbox), EP-01 (OpenFGA para invoice authz)  
**Dependencias Salientes**: Ninguna

---

## HU-02.1 — Billing Service Scaffold + Financial Ledger

**Como** backend developer, **quiero** scaffold Go service con financial ledger append-only y PSP façade pattern, **para que** tengamos base sólida para multi-gateway payments con audit trail completo.

### Tareas

#### T-02.1.1 — Estructura Proyecto

```bash
mkdir -p services/billing/{cmd/server,internal/{domain,handlers,repository,psp,idempotency},migrations}

cd services/billing
go mod init github.com/serenidad/billing

go get github.com/go-chi/chi/v5
go get github.com/jackc/pgx/v5/pgxpool
go get github.com/google/uuid
go get github.com/shopspring/decimal
go get github.com/redis/go-redis/v9
```

**CA**: Estructura creada, dependencies instaladas

#### T-02.1.2 — Ledger Schema (PostgreSQL)

Crear `migrations/001_create_ledger.up.sql`:

```sql
CREATE TABLE IF NOT EXISTS ledger_entries (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at timestamptz NOT NULL DEFAULT now(),
    
    -- Double-entry bookkeeping
    account_debit text NOT NULL,
    account_credit text NOT NULL,
    amount numeric(15,2) NOT NULL CHECK (amount > 0),
    currency char(3) NOT NULL,
    
    -- Operation type (immutability enforcement)
    operation_type text NOT NULL CHECK (operation_type IN ('INSERT', 'REVERSAL')),
    description text NOT NULL,
    
    -- Linking
    related_invoice_id uuid,
    related_appointment_id uuid,
    related_patient_id uuid,
    
    -- PSP metadata
    source_psp text NOT NULL,
    psp_transaction_id text,
    psp_metadata jsonb,
    
    -- Idempotency
    idempotency_key text UNIQUE NOT NULL,
    
    -- Audit
    created_by uuid NOT NULL,
    metadata jsonb
);

CREATE INDEX idx_ledger_created ON ledger_entries(created_at DESC);
CREATE INDEX idx_ledger_invoice ON ledger_entries(related_invoice_id);
CREATE INDEX idx_ledger_patient ON ledger_entries(related_patient_id);
CREATE INDEX idx_ledger_idempotency ON ledger_entries(idempotency_key);
CREATE INDEX idx_ledger_psp_txn ON ledger_entries(psp_transaction_id);

COMMENT ON TABLE ledger_entries IS 'Immutable financial ledger (append-only)';
COMMENT ON COLUMN ledger_entries.operation_type IS 'INSERT for new entries, REVERSAL for refunds/chargebacks';
```

**CA**: Migration aplicada, tabla `ledger_entries` existe

#### T-02.1.3 — Domain Model

Crear `internal/domain/ledger.go`:

```go
package domain

import (
    "time"
    "github.com/google/uuid"
    "github.com/shopspring/decimal"
)

type OperationType string

const (
    OperationInsert   OperationType = "INSERT"
    OperationReversal OperationType = "REVERSAL"
)

type LedgerEntry struct {
    ID                  uuid.UUID
    CreatedAt           time.Time
    AccountDebit        string
    AccountCredit       string
    Amount              decimal.Decimal
    Currency            string
    OperationType       OperationType
    Description         string
    RelatedInvoiceID    *uuid.UUID
    RelatedAppointmentID *uuid.UUID
    RelatedPatientID    *uuid.UUID
    SourcePSP           string
    PSPTransactionID    *string
    PSPMetadata         map[string]interface{}
    IdempotencyKey      string
    CreatedBy           uuid.UUID
    Metadata            map[string]interface{}
}

type ChargeRequest struct {
    Amount         decimal.Decimal
    Currency       string
    PatientID      uuid.UUID
    InvoiceID      uuid.UUID
    AppointmentID  uuid.UUID
    Description    string
    PaymentMethod  string
    IdempotencyKey string
}

type RefundRequest struct {
    OriginalChargeID uuid.UUID
    Amount           decimal.Decimal
    Reason           string
    IdempotencyKey   string
}
```

**CA**: Domain models definidos

### Criterios Aceptación HU-02.1

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Ledger table creada | `\d ledger_entries` en psql |
| 2 | Immutability enforced | CHECK constraint operation_type |
| 3 | Domain models compilados | `go build ./...` exitoso |

---

## HU-02.2 — PSP Façade Multi-Gateway

**Como** payment engineer, **quiero** PSP façade (Anti-Corruption Layer) que abstraiga Stripe, Conekta e Izipay con smart routing por país, **para que** integremos múltiples gateways sin acoplar domain logic.

### Tareas

#### T-02.2.1 — PSP Interface

Crear `internal/psp/interface.go`:

```go
package psp

import (
    "context"
    "github.com/serenidad/billing/internal/domain"
    "github.com/shopspring/decimal"
)

type PaymentGateway interface {
    Name() string
    Charge(ctx context.Context, req ChargeRequest) (*ChargeResponse, error)
    Refund(ctx context.Context, req RefundRequest) (*RefundResponse, error)
    GetCharge(ctx context.Context, chargeID string) (*ChargeResponse, error)
}

type ChargeRequest struct {
    Amount        decimal.Decimal
    Currency      string
    PaymentMethod string
    CustomerID    string
    Description   string
    Metadata      map[string]string
}

type ChargeResponse struct {
    ID            string
    Status        string
    Amount        decimal.Decimal
    Currency      string
    Created       int64
    Paid          bool
    PaymentMethod string
    Metadata      map[string]string
}

type RefundRequest struct {
    ChargeID string
    Amount   decimal.Decimal
    Reason   string
    Metadata map[string]string
}

type RefundResponse struct {
    ID       string
    ChargeID string
    Amount   decimal.Decimal
    Status   string
    Created  int64
}
```

**CA**: Interface definido

#### T-02.2.2 — Stripe Adapter

Crear `internal/psp/stripe.go`:

```go
package psp

import (
    "context"
    "github.com/stripe/stripe-go/v81"
    "github.com/stripe/stripe-go/v81/charge"
    "github.com/stripe/stripe-go/v81/refund"
)

type StripeGateway struct {
    apiKey string
}

func NewStripeGateway(apiKey string) *StripeGateway {
    stripe.Key = apiKey
    return &StripeGateway{apiKey: apiKey}
}

func (g *StripeGateway) Name() string {
    return "stripe"
}

func (g *StripeGateway) Charge(ctx context.Context, req ChargeRequest) (*ChargeResponse, error) {
    params := &stripe.ChargeParams{
        Amount:      stripe.Int64(req.Amount.IntPart() * 100), // cents
        Currency:    stripe.String(req.Currency),
        Description: stripe.String(req.Description),
    }
    
    if req.PaymentMethod != "" {
        params.Source = &stripe.SourceParams{
            Token: stripe.String(req.PaymentMethod),
        }
    }
    
    for k, v := range req.Metadata {
        params.AddMetadata(k, v)
    }
    
    ch, err := charge.New(params)
    if err != nil {
        return nil, err
    }
    
    return &ChargeResponse{
        ID:       ch.ID,
        Status:   string(ch.Status),
        Amount:   decimal.NewFromInt(ch.Amount / 100),
        Currency: string(ch.Currency),
        Created:  ch.Created,
        Paid:     ch.Paid,
    }, nil
}

func (g *StripeGateway) Refund(ctx context.Context, req RefundRequest) (*RefundResponse, error) {
    params := &stripe.RefundParams{
        Charge: stripe.String(req.ChargeID),
        Amount: stripe.Int64(req.Amount.IntPart() * 100),
    }
    
    if req.Reason != "" {
        params.Reason = stripe.String(req.Reason)
    }
    
    r, err := refund.New(params)
    if err != nil {
        return nil, err
    }
    
    return &RefundResponse{
        ID:       r.ID,
        ChargeID: r.Charge.ID,
        Amount:   decimal.NewFromInt(r.Amount / 100),
        Status:   string(r.Status),
        Created:  r.Created,
    }, nil
}
```

**CA**: Stripe adapter implementado

#### T-02.2.3 — Conekta Adapter (México)

Crear `internal/psp/conekta.go`:

```go
package psp

import (
    "context"
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
)

type ConektaGateway struct {
    apiKey  string
    baseURL string
}

func NewConektaGateway(apiKey string) *ConektaGateway {
    return &ConektaGateway{
        apiKey:  apiKey,
        baseURL: "https://api.conekta.io",
    }
}

func (g *ConektaGateway) Name() string {
    return "conekta"
}

func (g *ConektaGateway) Charge(ctx context.Context, req ChargeRequest) (*ChargeResponse, error) {
    payload := map[string]interface{}{
        "amount":      req.Amount.IntPart() * 100, // centavos
        "currency":    req.Currency,
        "description": req.Description,
        "payment_source": map[string]string{
            "type":  "card",
            "token_id": req.PaymentMethod,
        },
    }
    
    body, _ := json.Marshal(payload)
    httpReq, _ := http.NewRequestWithContext(ctx, "POST", g.baseURL+"/charges", bytes.NewBuffer(body))
    httpReq.Header.Set("Authorization", "Bearer "+g.apiKey)
    httpReq.Header.Set("Content-Type", "application/json")
    httpReq.Header.Set("Accept", "application/vnd.conekta-v2.0.0+json")
    
    client := &http.Client{}
    resp, err := client.Do(httpReq)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode != 200 {
        return nil, fmt.Errorf("conekta charge failed: %d", resp.StatusCode)
    }
    
    var result map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&result)
    
    return &ChargeResponse{
        ID:       result["id"].(string),
        Status:   result["status"].(string),
        Amount:   req.Amount,
        Currency: req.Currency,
        Paid:     result["status"].(string) == "paid",
    }, nil
}

func (g *ConektaGateway) Refund(ctx context.Context, req RefundRequest) (*RefundResponse, error) {
    // Similar implementation for Conekta refunds
    // TODO: Implement
    return nil, fmt.Errorf("not implemented")
}
```

**CA**: Conekta adapter implementado

#### T-02.2.4 — Gateway Router (Smart Routing)

Crear `internal/psp/router.go`:

```go
package psp

import (
    "context"
    "fmt"
)

type GatewayRouter struct {
    gateways map[string]PaymentGateway
}

func NewGatewayRouter() *GatewayRouter {
    return &GatewayRouter{
        gateways: make(map[string]PaymentGateway),
    }
}

func (r *GatewayRouter) RegisterGateway(name string, gateway PaymentGateway) {
    r.gateways[name] = gateway
}

func (r *GatewayRouter) SelectGateway(country, currency string) (PaymentGateway, error) {
    // Smart routing rules
    switch country {
    case "MX":
        if gw, ok := r.gateways["conekta"]; ok {
            return gw, nil
        }
    case "PE", "CO":
        if gw, ok := r.gateways["izipay"]; ok {
            return gw, nil
        }
    }
    
    // Default: Stripe
    if gw, ok := r.gateways["stripe"]; ok {
        return gw, nil
    }
    
    return nil, fmt.Errorf("no gateway available for country %s", country)
}

func (r *GatewayRouter) Charge(ctx context.Context, country string, req ChargeRequest) (*ChargeResponse, error) {
    gateway, err := r.SelectGateway(country, req.Currency)
    if err != nil {
        return nil, err
    }
    
    return gateway.Charge(ctx, req)
}
```

**CA**: Router con smart routing implementado

### Criterios Aceptación HU-02.2

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Stripe adapter funcional | Test charge exitoso |
| 2 | Conekta adapter funcional | Test charge MX exitoso |
| 3 | Router selecciona gateway | País MX → Conekta, default → Stripe |

---

## HU-02.3 — Idempotency con Redis

**Como** reliability engineer, **quiero** idempotency layer con Redis TTL 24h, **para que** retries de charges no dupliquen cobros y tengamos exactly-once semantics.

### Tareas

#### T-02.3.1 — Idempotency Service

Crear `internal/idempotency/service.go`:

```go
package idempotency

import (
    "context"
    "encoding/json"
    "time"
    "github.com/redis/go-redis/v9"
)

type Service struct {
    redis *redis.Client
}

func NewService(redis *redis.Client) *Service {
    return &Service{redis: redis}
}

func (s *Service) Execute(
    ctx context.Context,
    key string,
    ttl time.Duration,
    fn func() (interface{}, error),
) (interface{}, error) {
    // Check if key exists
    val, err := s.redis.Get(ctx, key).Result()
    if err == nil {
        // Key exists → return cached result
        var result interface{}
        json.Unmarshal([]byte(val), &result)
        return result, nil
    }
    
    if err != redis.Nil {
        return nil, err
    }
    
    // Key doesn't exist → execute operation
    result, err := fn()
    if err != nil {
        return nil, err
    }
    
    // Store result in Redis
    resultJSON, _ := json.Marshal(result)
    s.redis.Set(ctx, key, resultJSON, ttl)
    
    return result, nil
}
```

**CA**: Service implementado

#### T-02.3.2 — Handler Integration

Crear `internal/handlers/charge.go`:

```go
package handlers

import (
    "encoding/json"
    "net/http"
    "time"
    "github.com/serenidad/billing/internal/idempotency"
)

type BillingHandler struct {
    idempotency *idempotency.Service
    billingRepo *repository.BillingRepository
}

func (h *BillingHandler) CreateCharge(w http.ResponseWriter, r *http.Request) {
    idempotencyKey := r.Header.Get("Idempotency-Key")
    if idempotencyKey == "" {
        http.Error(w, "Idempotency-Key header required", http.StatusBadRequest)
        return
    }
    
    var req domain.ChargeRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }
    
    req.IdempotencyKey = idempotencyKey
    
    result, err := h.idempotency.Execute(
        r.Context(),
        "charge:"+idempotencyKey,
        24*time.Hour,
        func() (interface{}, error) {
            return h.billingRepo.CreateCharge(r.Context(), req)
        },
    )
    
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(result)
}
```

**CA**: Handler usa idempotency

### Criterios Aceptación HU-02.3

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Redis conectado | Ping retorna PONG |
| 2 | Idempotency key required | Request sin key → 400 |
| 3 | Retry retorna cached | 2 requests mismo key → mismo result |
| 4 | TTL 24h configurado | Redis TTL key = 86400s |

---

## HU-02.4 — Reconciliation Job

**Como** finance ops, **quiero** reconciliation job diario que compare ledger entries con PSP settlements, **para que** detectemos discrepancias automáticamente y tengamos audit trail completo.

### Tareas

#### T-02.4.1 — Reconciliation Service

Crear `internal/reconciliation/service.go`:

```go
package reconciliation

import (
    "context"
    "log"
    "time"
)

type Service struct {
    ledgerRepo      *repository.LedgerRepository
    stripeClient    *psp.StripeGateway
    conektaClient   *psp.ConektaGateway
}

func (s *Service) ReconcileDaily(ctx context.Context, date time.Time) error {
    log.Printf("Starting reconciliation for %s", date.Format("2006-01-02"))
    
    // 1. Fetch Stripe settlements
    stripeSettlements, err := s.fetchStripeSettlements(ctx, date)
    if err != nil {
        return err
    }
    
    // 2. Match against ledger
    for _, settlement := range stripeSettlements {
        ledgerEntry, err := s.ledgerRepo.FindByPSPTxnID(settlement.ID)
        if err != nil {
            s.flagException("missing_ledger_entry", settlement)
            continue
        }
        
        // 3. Verify amounts (accounting for fees)
        if settlement.NetAmount != ledgerEntry.Amount {
            s.flagException("amount_mismatch", settlement)
        }
    }
    
    // Repeat for Conekta, Izipay...
    
    log.Printf("Reconciliation completed for %s", date.Format("2006-01-02"))
    return nil
}

func (s *Service) flagException(reason string, data interface{}) {
    // Store exception in exceptions table for manual review
    log.Printf("EXCEPTION: %s - %+v", reason, data)
}
```

**CA**: Service implementado

#### T-02.4.2 — CronJob Kubernetes

Crear `infrastructure/k8s/billing/cronjob.yaml`:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: billing-reconciliation
  namespace: serenidad
spec:
  schedule: "0 2 * * *"  # Daily 02:00 UTC
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: reconciliation
              image: registry.gitlab.com/serenidad/billing:latest
              command: ["/app/billing-service", "reconcile", "--date", "yesterday"]
              env:
                - name: DATABASE_URL
                  valueFrom:
                    secretKeyRef:
                      name: postgres-credentials
                      key: connection-string
                - name: STRIPE_API_KEY
                  valueFrom:
                    secretKeyRef:
                      name: stripe-credentials
                      key: api-key
          restartPolicy: OnFailure
```

**CA**: CronJob configurado

### Criterios Aceptación HU-02.4

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | CronJob creado | `kubectl get cronjob -n serenidad` |
| 2 | Job ejecuta daily | Logs muestran ejecución 02:00 UTC |
| 3 | Exceptions flagged | Exceptions table contiene entries |

---

## Resumen Entregables EP-02

**Código**:
- Billing Service Go (Chi + pgxpool)
- Financial ledger append-only (PostgreSQL)
- PSP façade (Stripe, Conekta, Izipay adapters)
- Smart routing por país
- Idempotency layer (Redis 24h TTL)
- Reconciliation service

**Base de Datos**:
- `ledger_entries` table (immutable)
- `invoices` table
- `exceptions` table (reconciliation)

**Kubernetes**:
- Deployment (2 replicas, HPA)
- CronJob (reconciliation daily)
- Secrets (Stripe, Conekta, Izipay keys)
- NetworkPolicy

**Validación**:
```bash
# 1. Crear charge Stripe
curl -X POST https://api.sereni.dad/v1/billing/charges \
  -H "Authorization: Bearer $JWT" \
  -H "Idempotency-Key: charge-abc-123" \
  -d '{
    "amount": 45.00,
    "currency": "USD",
    "patient_id": "p123",
    "invoice_id": "inv-456",
    "payment_method": "tok_visa"
  }'

# 2. Verificar ledger entry
psql -c "SELECT * FROM ledger_entries WHERE idempotency_key='charge-abc-123';"

# 3. Verificar Outbox event
psql -c "SELECT * FROM outbox WHERE aggregate_type='billing' ORDER BY created_at DESC LIMIT 1;"

# 4. Test idempotency (retry)
curl -X POST https://api.sereni.dad/v1/billing/charges \
  -H "Idempotency-Key: charge-abc-123" \
  # → Same result, no duplicate charge

# 5. Verificar reconciliation job
kubectl logs -n serenidad $(kubectl get pods -n serenidad -l job-name=billing-reconciliation --sort-by=.metadata.creationTimestamp -o name | tail -1)
```

---

**Tiempo estimado EP-02**: 2 semanas (Sprint 8-9)  
**Esfuerzo**: ~80 horas ingeniería + 20 horas testing + 10 horas PSP integration
