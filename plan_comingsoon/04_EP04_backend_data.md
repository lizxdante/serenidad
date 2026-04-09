# EP-CS04 — Backend API & Database

**Épica:** Como sistema backend, necesito un API endpoint en Cloudflare Pages Functions que reciba los datos del formulario, los valide server-side, los persista en Cloudflare D1, y retorne la URL de WhatsApp — garantizando integridad de datos, deduplicación y protección contra abuso.

**Prioridad:** Must Have — Core functionality
**Sprint:** Sprint 2
**Dependencias Entrantes:** EP-CS01 (wrangler.toml con D1 binding), EP-CS03 (formulario que envía datos)
**Dependencias Salientes:** EP-CS05 (WhatsApp URL generation)

---

## HU-CS04.1 — Cloudflare D1 Database Setup

**Como** desarrollador backend,
**quiero** una base de datos D1 configurada con el schema de leads y migraciones versionadas,
**para que** los datos de contacto se almacenen de forma estructurada y consultable.

### Tareas y Subtareas

#### T-CS04.1.1 — Crear base de datos D1

- **ST-CS04.1.1.1** — Crear la base de datos D1 via wrangler CLI.
  ```bash
  cd frontend/comingsoon
  npx wrangler d1 create serenidad-leads
  ```
  - **CA:** Output muestra `database_id`. Copiar a `wrangler.toml`.

- **ST-CS04.1.1.2** — Actualizar `wrangler.toml` con el database_id real.
  ```toml
  [[d1_databases]]
  binding = "DB"
  database_name = "serenidad-leads"
  database_id = "<ID_OBTENIDO>"
  ```
  - **CA:** `wrangler.toml` actualizado.

#### T-CS04.1.2 — Crear migración inicial

- **ST-CS04.1.2.1** — Crear `schema/001_create_leads.sql`.
  ```sql
  -- Migration: 001_create_leads
  -- Description: Initial leads table for Coming Soon landing page
  -- Author: djca
  -- Date: April 2026

  CREATE TABLE IF NOT EXISTS leads (
      -- Primary key: auto-increment integer (D1/SQLite native)
      id          INTEGER PRIMARY KEY AUTOINCREMENT,

      -- Contact data
      name        TEXT NOT NULL CHECK(length(trim(name)) >= 2),
      email       TEXT NOT NULL CHECK(email LIKE '%@%.%'),
      phone       TEXT NOT NULL CHECK(length(replace(replace(replace(replace(phone, ' ', ''), '-', ''), '(', ''), ')', '')) >= 8),
      message     TEXT CHECK(message IS NULL OR length(message) <= 500),

      -- Metadata
      source      TEXT NOT NULL DEFAULT 'coming_soon_landing',
      ip_hash     TEXT,           -- SHA-256 hash of client IP (privacy-preserving)
      user_agent  TEXT,           -- Browser user agent for debugging
      whatsapp_sent INTEGER NOT NULL DEFAULT 0,  -- 0=no, 1=yes (redirected to WhatsApp)

      -- Timestamps
      created_at  TEXT NOT NULL DEFAULT (datetime('now')),
      updated_at  TEXT NOT NULL DEFAULT (datetime('now')),

      -- Constraints
      UNIQUE(email, phone)  -- Prevent exact duplicates
  );

  -- Indexes for common queries
  CREATE INDEX idx_leads_email ON leads(email);
  CREATE INDEX idx_leads_phone ON leads(phone);
  CREATE INDEX idx_leads_created_at ON leads(created_at DESC);
  CREATE INDEX idx_leads_source ON leads(source);

  -- Trigger: auto-update updated_at on row modification
  CREATE TRIGGER update_lead_timestamp
  AFTER UPDATE ON leads
  FOR EACH ROW
  BEGIN
      UPDATE leads SET updated_at = datetime('now') WHERE id = NEW.id;
  END;
  ```
  - **CA:** Schema SQL creado con tabla leads, indexes, constraints y trigger.

- **ST-CS04.1.2.2** — Aplicar migración local.
  ```bash
  npx wrangler d1 execute serenidad-leads --local --file=schema/001_create_leads.sql
  ```
  - **CA:** Tabla `leads` creada en D1 local.

- **ST-CS04.1.2.3** — Aplicar migración remota (producción).
  ```bash
  npx wrangler d1 execute serenidad-leads --remote --file=schema/001_create_leads.sql
  ```
  - **CA:** Tabla `leads` creada en D1 remoto.

#### T-CS04.1.3 — Verificar schema

- **ST-CS04.1.3.1** — Verificar tabla creada.
  ```bash
  npx wrangler d1 execute serenidad-leads --local --command="SELECT sql FROM sqlite_master WHERE name='leads';"
  ```
  - **CA:** Output muestra el CREATE TABLE statement.

- **ST-CS04.1.3.2** — Verificar indexes.
  ```bash
  npx wrangler d1 execute serenidad-leads --local --command="SELECT name FROM sqlite_master WHERE type='index' AND tbl_name='leads';"
  ```
  - **CA:** 4 indexes listados: idx_leads_email, idx_leads_phone, idx_leads_created_at, idx_leads_source.

### Criterios de Aceptación — HU-CS04.1

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | D1 database creada | `wrangler d1 list` muestra `serenidad-leads` |
| 2 | Tabla leads existe | SELECT sql FROM sqlite_master retorna schema |
| 3 | 4 indexes creados | idx_leads_email, phone, created_at, source |
| 4 | UNIQUE constraint funciona | INSERT duplicado retorna error |
| 5 | Trigger updated_at funciona | UPDATE cambia updated_at automáticamente |
| 6 | CHECK constraints funcionan | INSERT con email inválido retorna error |

### Definition of Done — HU-CS04.1

- [ ] D1 database `serenidad-leads` creada (local + remoto)
- [ ] Migración 001_create_leads.sql aplicada
- [ ] Tabla leads con todos los campos, constraints e indexes
- [ ] Trigger de updated_at funcionando
- [ ] wrangler.toml con database_id correcto

---

## HU-CS04.2 — API Endpoint POST /api/lead

**Como** frontend de la landing page,
**quiero** un endpoint POST /api/lead que valide los datos, los persista en D1, y retorne la URL de WhatsApp,
**para que** los leads se guarden de forma confiable y el usuario sea redirigido a WhatsApp.

### Contexto Técnico

**Cloudflare Pages Functions:**
- Archivos en `functions/` se mapean automáticamente a rutas API
- `functions/api/lead.ts` → `POST /api/lead`
- Runtime: Cloudflare Workers (V8 isolate)
- Bindings: `DB` (D1 database)
- TypeScript 6.0 con tipado estricto

### Tareas y Subtareas

#### T-CS04.2.1 — Implementar Pages Function

- **ST-CS04.2.1.1** — Crear `functions/api/lead.ts`.

  ```typescript
  /**
   * POST /api/lead — Cloudflare Pages Function
   *
   * Receives lead form submissions from the Coming Soon landing page.
   * Validates input, deduplicates, persists to D1, returns WhatsApp URL.
   *
   * Rate limit: 5 submissions per IP per hour (in-memory, per-isolate).
   * Anti-spam: Honeypot field check + rate limiting.
   */

  // ── Types ──────────────────────────────────────────────────────────────
  interface Env {
    DB: D1Database;
    WHATSAPP_NUMBER: string;  // Configured as Pages Function env var
  }

  interface LeadInput {
    name?: unknown;
    email?: unknown;
    phone?: unknown;
    message?: unknown;
    website?: unknown;  // Honeypot
  }

  interface ValidationErrorDetail {
    field: string;
    message: string;
  }

  // ── Rate Limiting (per-isolate, in-memory) ─────────────────────────────
  const rateLimitMap = new Map<string, { count: number; resetAt: number }>();
  const RATE_LIMIT_MAX = 5;
  const RATE_LIMIT_WINDOW_MS = 60 * 60 * 1000; // 1 hour

  function checkRateLimit(ip: string): boolean {
    const now = Date.now();
    const entry = rateLimitMap.get(ip);

    if (!entry || now > entry.resetAt) {
      rateLimitMap.set(ip, { count: 1, resetAt: now + RATE_LIMIT_WINDOW_MS });
      return true;
    }

    if (entry.count >= RATE_LIMIT_MAX) {
      return false;
    }

    entry.count++;
    return true;
  }

  // ── Validation ─────────────────────────────────────────────────────────
  function validateInput(data: LeadInput): {
    valid: boolean;
    errors: ValidationErrorDetail[];
    sanitized: {
      name: string;
      email: string;
      phone: string;
      message: string | null;
    } | null;
  } {
    const errors: ValidationErrorDetail[] = [];

    // Honeypot check
    if (data.website) {
      // Bot detected — return fake success
      return { valid: true, errors: [], sanitized: null };
    }

    // Name validation
    const name = typeof data.name === 'string' ? data.name.trim() : '';
    if (!name) {
      errors.push({ field: 'name', message: 'El nombre es obligatorio' });
    } else if (name.length < 2) {
      errors.push({ field: 'name', message: 'El nombre debe tener al menos 2 caracteres' });
    } else if (!/^[a-zA-ZáéíóúÁÉÍÓÚñÑüÜ\s]+$/.test(name)) {
      errors.push({ field: 'name', message: 'El nombre solo puede contener letras y espacios' });
    }

    // Email validation
    const email = typeof data.email === 'string' ? data.email.trim().toLowerCase() : '';
    if (!email) {
      errors.push({ field: 'email', message: 'El email es obligatorio' });
    } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      errors.push({ field: 'email', message: 'Ingresa un email válido' });
    }

    // Phone validation
    const phone = typeof data.phone === 'string' ? data.phone.trim() : '';
    const phoneDigits = phone.replace(/[\s\-\(\)\+]/g, '');
    if (!phone) {
      errors.push({ field: 'phone', message: 'El teléfono es obligatorio' });
    } else if (phoneDigits.length < 8) {
      errors.push({ field: 'phone', message: 'El teléfono debe tener al menos 8 dígitos' });
    } else if (phoneDigits.length > 15) {
      errors.push({ field: 'phone', message: 'El teléfono no puede tener más de 15 dígitos' });
    }

    // Message validation (optional)
    const message = typeof data.message === 'string' ? data.message.trim() : null;
    if (message && message.length > 500) {
      errors.push({ field: 'message', message: 'El mensaje no puede exceder 500 caracteres' });
    }

    if (errors.length > 0) {
      return { valid: false, errors, sanitized: null };
    }

    return {
      valid: true,
      errors: [],
      sanitized: { name, email, phone, message },
    };
  }

  // ── WhatsApp URL Generation ────────────────────────────────────────────
  function generateWhatsAppUrl(
    phoneNumber: string,
    leadName: string,
    leadMessage: string | null,
  ): string {
    const textParts = [
      `Hola, soy ${leadName}.`,
      'Me interesa saber más sobre Serenidad y agendar una consulta.',
    ];
    if (leadMessage) {
      textParts.push(`Mi consulta: ${leadMessage}`);
    }
    const text = encodeURIComponent(textParts.join(' '));
    return `https://wa.me/${phoneNumber}?text=${text}`;
  }

  // ── Hash IP for privacy ────────────────────────────────────────────────
  async function hashIp(ip: string): Promise<string> {
    const encoder = new TextEncoder();
    const data = encoder.encode(ip + '-serenidad-salt-2026');
    const hashBuffer = await crypto.subtle.digest('SHA-256', data);
    const hashArray = Array.from(new Uint8Array(hashBuffer));
    return hashArray.map((b) => b.toString(16).padStart(2, '0')).join('').substring(0, 16);
  }

  // ── Handler ────────────────────────────────────────────────────────────
  export const onRequestPost: PagesFunction<Env> = async (context) => {
    const { request, env } = context;

    // CORS headers
    const corsHeaders = {
      'Access-Control-Allow-Origin': 'https://sereni.dad',
      'Access-Control-Allow-Methods': 'POST, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type',
      'Content-Type': 'application/json',
    };

    // Rate limiting
    const clientIp = request.headers.get('CF-Connecting-IP') || 'unknown';
    if (!checkRateLimit(clientIp)) {
      return new Response(
        JSON.stringify({
          error: 'RATE_LIMITED',
          message: 'Demasiados intentos. Intenta de nuevo en una hora.',
        }),
        { status: 429, headers: corsHeaders },
      );
    }

    // Parse body
    let body: LeadInput;
    try {
      body = await request.json() as LeadInput;
    } catch {
      return new Response(
        JSON.stringify({ error: 'INVALID_JSON', message: 'Body debe ser JSON válido' }),
        { status: 400, headers: corsHeaders },
      );
    }

    // Validate
    const validation = validateInput(body);

    // Honeypot triggered — return fake success (no data saved)
    if (validation.valid && validation.sanitized === null) {
      const fakeUrl = generateWhatsAppUrl(
        env.WHATSAPP_NUMBER || '5215512345678',
        'amigo',
        null,
      );
      return new Response(
        JSON.stringify({ id: 0, status: 'created', whatsapp_url: fakeUrl }),
        { status: 201, headers: corsHeaders },
      );
    }

    if (!validation.valid) {
      return new Response(
        JSON.stringify({ error: 'VALIDATION_ERROR', details: validation.errors }),
        { status: 400, headers: corsHeaders },
      );
    }

    const { name, email, phone, message } = validation.sanitized!;

    // Hash IP for privacy-preserving storage
    const ipHash = await hashIp(clientIp);
    const userAgent = request.headers.get('User-Agent') || null;

    // Deduplication check
    const existing = await env.DB.prepare(
      'SELECT id FROM leads WHERE email = ? AND phone = ?'
    ).bind(email, phoneDigits(phone)).first();

    if (existing) {
      // Lead already exists — return existing with WhatsApp URL
      const whatsappUrl = generateWhatsAppUrl(
        env.WHATSAPP_NUMBER || '5215512345678',
        name,
        message,
      );
      return new Response(
        JSON.stringify({ id: existing.id as number, status: 'duplicate', whatsapp_url: whatsappUrl }),
        { status: 200, headers: corsHeaders },
      );
    }

    // Insert new lead
    const result = await env.DB.prepare(
      `INSERT INTO leads (name, email, phone, message, ip_hash, user_agent, whatsapp_sent)
       VALUES (?, ?, ?, ?, ?, ?, 1)`
    ).bind(name, email, phone, message, ipHash, userAgent).run();

    const leadId = result.meta.last_row_id;

    // Generate WhatsApp URL
    const whatsappUrl = generateWhatsAppUrl(
      env.WHATSAPP_NUMBER || '5215512345678',
      name,
      message,
    );

    return new Response(
      JSON.stringify({ id: leadId, status: 'created', whatsapp_url: whatsappUrl }),
      { status: 201, headers: corsHeaders },
    );
  };

  // ── CORS Preflight ─────────────────────────────────────────────────────
  export const onRequestOptions: PagesFunction = async () => {
    return new Response(null, {
      status: 204,
      headers: {
        'Access-Control-Allow-Origin': 'https://sereni.dad',
        'Access-Control-Allow-Methods': 'POST, OPTIONS',
        'Access-Control-Allow-Headers': 'Content-Type',
        'Access-Control-Max-Age': '86400',
      },
    });
  };

  // ── Helper ─────────────────────────────────────────────────────────────
  function phoneDigits(phone: string): string {
    return phone.replace(/[\s\-\(\)\+]/g, '');
  }
  ```
  - **CA:** Pages Function implementada con validación, deduplicación, rate limiting, D1 insert.
  - **CA:** Honeypot check retorna fake success (no guarda datos de bots).
  - **CA:** IP hasheada con SHA-256 para privacy-preserving storage.
  - **CA:** CORS headers configurados para sereni.dad.
  - **CA:** Rate limiting: 5 submissions por IP por hora.

### Criterios de Aceptación — HU-CS04.2

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | POST /api/lead retorna 201 | curl con datos válidos → 201 Created |
| 2 | Lead insertado en D1 | SELECT * FROM leads muestra el registro |
| 3 | Validación server-side | POST con email inválido → 400 con details |
| 4 | Deduplicación funciona | POST mismo email+phone → 200 con status duplicate |
| 5 | Rate limiting funciona | 6° POST desde misma IP → 429 |
| 6 | Honeypot funciona | POST con website lleno → 201 fake (no insert) |
| 7 | WhatsApp URL generada | Response incluye wa.me URL válida |
| 8 | CORS preflight funciona | OPTIONS /api/lead → 204 con headers |
| 9 | IP hasheada | D1 muestra hash, no IP real |
| 10 | TypeScript compila | Sin errores de tipos |

### Definition of Done — HU-CS04.2

- [ ] Pages Function `functions/api/lead.ts` implementada
- [ ] Validación server-side con mensajes en español
- [ ] Deduplicación por email + phone
- [ ] Rate limiting por IP (5/hora)
- [ ] Anti-spam honeypot
- [ ] IP hasheada con SHA-256
- [ ] CORS configurado
- [ ] TypeScript compila sin errores

---

## HU-CS04.3 — Data Export & Admin Query

**Como** administrador de Serenidad,
**quiero** poder consultar y exportar los leads almacenados en D1,
**para que** pueda dar seguimiento a los contactos y migrar los datos a PostgreSQL cuando la infraestructura esté lista.

### Tareas y Subtareas

#### T-CS04.3.1 — Crear script de exportación

- **ST-CS04.3.1.1** — Crear `scripts/export-leads.sh`.
  ```bash
  #!/bin/bash
  # Export all leads from D1 to CSV
  # Usage: ./scripts/export-leads.sh

  set -euo pipefail

  TIMESTAMP=$(date +%Y%m%d_%H%M%S)
  OUTPUT_DIR="exports"
  OUTPUT_FILE="${OUTPUT_DIR}/leads_${TIMESTAMP}.csv"

  mkdir -p "${OUTPUT_DIR}"

  echo "Exporting leads from D1..."

  npx wrangler d1 execute serenidad-leads --remote --command="
    SELECT
      id,
      name,
      email,
      phone,
      message,
      source,
      whatsapp_sent,
      created_at,
      updated_at
    FROM leads
    ORDER BY created_at DESC;
  " --json > "${OUTPUT_DIR}/leads_${TIMESTAMP}_raw.json"

  echo "Exported to ${OUTPUT_DIR}/leads_${TIMESTAMP}_raw.json"
  echo "Total leads:"
  npx wrangler d1 execute serenidad-leads --remote --command="SELECT COUNT(*) as total FROM leads;"
  ```
  - **CA:** Script de exportación creado.

#### T-CS04.3.2 — Crear query de estadísticas

- **ST-CS04.3.2.1** — Documentar queries útiles en `schema/queries.sql`.
  ```sql
  -- Total leads
  SELECT COUNT(*) as total_leads FROM leads;

  -- Leads por día (últimos 30 días)
  SELECT DATE(created_at) as date, COUNT(*) as count
  FROM leads
  WHERE created_at >= datetime('now', '-30 days')
  GROUP BY DATE(created_at)
  ORDER BY date DESC;

  -- Leads duplicados (mismo email, diferente phone)
  SELECT email, COUNT(*) as count
  FROM leads
  GROUP BY email
  HAVING count > 1
  ORDER BY count DESC;

  -- Exportar a CSV-compatible format
  SELECT
    id, name, email, phone, message, source,
    CASE WHEN whatsapp_sent = 1 THEN 'yes' ELSE 'no' END as whatsapp,
    created_at
  FROM leads
  ORDER BY created_at DESC;
  ```
  - **CA:** Queries documentadas para administración.

### Criterios de Aceptación — HU-CS04.3

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Script export ejecuta | `./scripts/export-leads.sh` genera JSON |
| 2 | Queries documentadas | queries.sql contiene 4+ queries útiles |
| 3 | Count query funciona | Retorna número total de leads |

### Definition of Done — HU-CS04.3

- [ ] Script de exportación a JSON creado
- [ ] Queries de administración documentadas
- [ ] Verificación manual de export exitosa

---

## Criterios de Aceptación Global — EP-CS04

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-EP04-1 | D1 database creada y migrada | Tabla leads accesible |
| CA-EP04-2 | POST /api/lead funciona E2E | curl → 201 → D1 tiene registro |
| CA-EP04-3 | Validación server-side completa | 400 para datos inválidos |
| CA-EP04-4 | Deduplicación funciona | 200 duplicate para repetidos |
| CA-EP04-5 | Rate limiting activo | 429 después de 5 requests |
| CA-EP04-6 | Anti-spam honeypot | Bots reciben fake success |
| CA-EP04-7 | Data exportable | Script genera JSON export |

### Definition of Done — EP-CS04

- [ ] D1 database con schema completo (tabla, indexes, trigger)
- [ ] Pages Function POST /api/lead implementada y testeada
- [ ] Validación server-side con mensajes en español
- [ ] Deduplicación por email + phone
- [ ] Rate limiting por IP
- [ ] Anti-spam honeypot
- [ ] CORS configurado
- [ ] Script de exportación creado
- [ ] Queries de administración documentadas

---

## Resumen de Entregables EP-CS04

**Database:**
- `schema/001_create_leads.sql` — Migración D1 completa
- `schema/queries.sql` — Queries de administración

**API:**
- `functions/api/lead.ts` — Pages Function con POST handler

**Scripts:**
- `scripts/export-leads.sh` — Exportación de leads

**Seguridad:**
- Rate limiting: 5 req/IP/hora
- Honeypot anti-spam
- IP hasheada (SHA-256)
- CORS restrictivo (solo sereni.dad)
- Server-side validation completa

**Testing commands:**
```bash
# Test local
curl -X POST http://localhost:8788/api/lead \
  -H "Content-Type: application/json" \
  -d '{"name":"Test User","email":"test@example.com","phone":"+5215512345678"}'

# Test validation error
curl -X POST http://localhost:8788/api/lead \
  -H "Content-Type: application/json" \
  -d '{"name":"","email":"invalid","phone":"123"}'

# Test rate limit
for i in $(seq 1 6); do
  curl -X POST http://localhost:8788/api/lead \
    -H "Content-Type: application/json" \
    -d "{\"name\":\"Test $i\",\"email\":\"test$i@example.com\",\"phone\":\"+521551234567$i\"}"
done
# 6th request should return 429
```
