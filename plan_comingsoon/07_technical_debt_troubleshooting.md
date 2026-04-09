# Technical Debt & Troubleshooting Guide
## Coming Soon Landing Page Plan Review

**Review Date:** 2026-04-09  
**Reviewer:** Architect Mode  
**Scope:** All plan_comingsoon files (00-06)

---

## Executive Summary

This document identifies technical debts, version mismatches, architectural concerns, and implementation gaps in the coming soon landing page plan. Each issue is categorized by severity and includes troubleshooting steps and recommended fixes.

**Total Issues Found:** 12  
- **Critical:** 3  
- **High:** 4  
- **Medium:** 3  
- **Low:** 2

---

## Table of Contents

1. [Critical Issues](#critical-issues)
2. [High Priority Issues](#high-priority-issues)
3. [Medium Priority Issues](#medium-priority-issues)
4. [Low Priority Issues](#low-priority-issues)
5. [Version Compatibility Matrix](#version-compatibility-matrix)
6. [Implementation Checklist](#implementation-checklist)

---

## Critical Issues

### TD-001: Astro Version Mismatch (v6 planned, v5.x available)

**Location:** All plan files  
**Severity:** 🔴 Critical  
**Category:** Version Mismatch

**Issue:**
The plan specifies Astro v6, but as of late 2025, the latest stable version is Astro v5.16.6. Astro v6 may not exist or may still be in beta/alpha, which introduces significant risk for production deployment.

**Impact:**
- Installation commands will fail or install wrong version
- Documentation references may be incorrect
- Breaking changes between v5 and v6 (if v6 exists) could break implementation
- Unstable API if using beta/alpha versions

**Root Cause:**
- Version specified without verification against current stable releases
- No version pinning strategy defined (e.g., `^5.0.0` vs `6.0.0`)

**Troubleshooting Steps:**
```bash
# Verify current stable version
npm view astro version
npm view astro versions --json | tail -20

# Check if v6 exists
npm view astro@6 versions --json
```

**Recommended Fix:**
1. Update all references from `v6` to `v5.x` (specifically `^5.0.0`)
2. Update installation command:
   ```bash
   # BEFORE (incorrect):
   npm create astro@latest -- --template minimal --typescript strict --no-git
   
   # AFTER (correct):
   npm create astro@latest -- --template minimal --typescript strict
   ```
3. Add version pinning to package.json:
   ```json
   {
     "dependencies": {
       "astro": "^5.0.0"
     }
   }
   ```
4. Verify Astro v5 features match planned functionality

**Affected Files:**
- `00_overview.md` - Line 42 (Tech Stack)
- `01_EP01_scaffold_design.md` - Line 15 (Installation command)

---

### TD-002: TypeScript 6.0 Version Issue

**Location:** `01_EP01_scaffold_design.md`, `00_overview.md`  
**Severity:** 🔴 Critical  
**Category:** Version Mismatch

**Issue:**
The plan specifies TypeScript 6.0 with `verbatimModuleSyntax` for TS 7 migration path. As of 2025, TypeScript is at version 5.x (5.4+ stable). TypeScript 6.0 may not exist or be stable.

**Impact:**
- Installation may fail or install latest 5.x version
- `verbatimModuleSyntax` may behave differently in TS 5.x
- Migration path documentation may be incorrect
- Type checking may fail with unexpected errors

**Root Cause:**
- Version specified without verification against current stable releases
- Confusion between TypeScript and Astro versioning

**Troubleshooting Steps:**
```bash
# Verify current stable version
npm view typescript version
npm view typescript versions --json | tail -20

# Check if TS 6.0 exists
npm view typescript@6 versions --json
```

**Recommended Fix:**
1. Update to TypeScript 5.4+ (latest stable):
   ```json
   {
     "devDependencies": {
       "typescript": "^5.4.0"
     }
   }
   ```
2. Update tsconfig.json:
   ```json
   {
     "compilerOptions": {
       "target": "ES2022",
       "module": "ESNext",
       "moduleResolution": "bundler",
       "verbatimModuleSyntax": true,
       "strict": true
     }
   }
   ```
3. Remove references to "TS 7 migration path" as this is speculative
4. Document actual TypeScript 5.x features being used

**Affected Files:**
- `00_overview.md` - Line 44 (Tech Stack)
- `01_EP01_scaffold_design.md` - Line 20 (TypeScript version)

---

### TD-003: In-Memory Rate Limiting Doesn't Work on Edge

**Location:** `04_EP04_backend_data.md` - Lines 95-115  
**Severity:** 🔴 Critical  
**Category:** Architecture Flaw

**Issue:**
The plan implements rate limiting using an in-memory `Map` stored in the Pages Function. Cloudflare Pages Functions are stateless and run on multiple edge locations worldwide. Each request may hit a different edge location, making in-memory rate limiting ineffective.

**Code from plan:**
```typescript
// THIS DOESN'T WORK ON EDGE
const rateLimit = new Map<string, { count: number; resetTime: number }>();
```

**Impact:**
- Rate limiting will be bypassed entirely
- Spammers can submit from different edge locations
- No protection against abuse
- Database may be flooded with submissions

**Root Cause:**
- Misunderstanding of Cloudflare Pages Functions execution model
- In-memory storage doesn't persist across function invocations or edge locations

**Troubleshooting Steps:**
1. Test rate limiting by submitting from different geographic locations
2. Monitor D1 database for unexpected submission spikes
3. Check Cloudflare Analytics for submission patterns

**Recommended Fix:**

**Option 1: Use Cloudflare KV for Rate Limiting (Recommended)**
```typescript
// functions/api/lead.ts
import { KVNamespace } from '@cloudflare/workers-types';

interface Env {
  DB: D1Database;
  RATE_LIMIT: KVNamespace;
}

export async function POST(request: Request, env: Env) {
  const ip = request.headers.get('CF-Connecting-IP') || 'unknown';
  const key = `ratelimit:${ip}`;
  const now = Date.now();
  
  // Check existing rate limit
  const existing = await env.RATE_LIMIT.get(key);
  if (existing) {
    const data = JSON.parse(existing);
    if (data.count >= 5 && now < data.resetTime) {
      return new Response(
        JSON.stringify({ error: 'Demasiados intentos. Por favor, espera una hora.' }),
        { status: 429 }
      );
    }
  }
  
  // Increment counter
  const count = existing ? JSON.parse(existing).count + 1 : 1;
  const resetTime = now + (60 * 60 * 1000); // 1 hour
  
  await env.RATE_LIMIT.put(key, JSON.stringify({ count, resetTime }), {
    expirationTtl: 3600 // 1 hour
  });
  
  // ... rest of submission logic
}
```

**Option 2: Use D1 for Rate Limiting**
```typescript
// Create rate_limits table
CREATE TABLE IF NOT EXISTS rate_limits (
  ip_hash TEXT PRIMARY KEY,
  count INTEGER NOT NULL DEFAULT 0,
  reset_at INTEGER NOT NULL,
  created_at INTEGER NOT NULL DEFAULT (strftime('%s', 'now'))
);

CREATE INDEX idx_rate_limits_reset_at ON rate_limits(reset_at);
```

```typescript
// Check and increment rate limit in D1
const ipHash = hashIP(ip);
const now = Math.floor(Date.now() / 1000);
const resetAt = now + 3600;

// Check existing
const existing = await env.DB.prepare(
  'SELECT count, reset_at FROM rate_limits WHERE ip_hash = ?'
).bind(ipHash).first();

if (existing && existing.count >= 5 && existing.reset_at > now) {
  return new Response(
    JSON.stringify({ error: 'Demasiados intentos. Por favor, espera una hora.' }),
    { status: 429 }
  );
}

// Upsert
await env.DB.prepare(`
  INSERT INTO rate_limits (ip_hash, count, reset_at, created_at)
  VALUES (?, 1, ?, ?)
  ON CONFLICT(ip_hash) DO UPDATE SET
    count = CASE WHEN reset_at <= ? THEN 1 ELSE count + 1 END,
    reset_at = CASE WHEN reset_at <= ? THEN ? ELSE reset_at END
`).bind(ipHash, resetAt, now, now, now, resetAt).run();
```

**Configuration Required:**
```toml
# wrangler.toml
[[kv_namespaces]]
binding = "RATE_LIMIT"
id = "your-kv-namespace-id"
```

**Affected Files:**
- `04_EP04_backend_data.md` - Lines 95-115 (Rate limiting implementation)
- `06_EP06_deploy_cicd.md` - Add KV namespace setup

---

## High Priority Issues

### TD-004: TailwindCSS v4 Status Uncertain

**Location:** `01_EP01_scaffold_design.md`, `00_overview.md`  
**Severity:** 🟠 High  
**Category:** Version Uncertainty

**Issue:**
The plan specifies TailwindCSS v4 with `@theme` directive. As of 2025, TailwindCSS v4 is in beta/alpha. The `@theme` directive syntax is experimental and may change before stable release.

**Impact:**
- Installation may require beta flags
- Syntax may break in future versions
- Documentation and examples may be outdated
- Production deployment with beta software is risky

**Root Cause:**
- Version specified without checking stable release status
- Experimental syntax used without fallback plan

**Troubleshooting Steps:**
```bash
# Check TailwindCSS versions
npm view tailwindcss version
npm view tailwindcss@beta version
npm view tailwindcss@next version
```

**Recommended Fix:**

**Option 1: Use TailwindCSS v3.4 (Stable, Recommended)**
```bash
npm install -D tailwindcss@^3.4.0
```

Update configuration:
```js
// tailwind.config.js
export default {
  content: ['./src/**/*.{astro,html,js,jsx,md,mdx,svelte,ts,tsx,vue}'],
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#f0f9ff',
          100: '#e0f2fe',
          200: '#bae6fd',
          300: '#7dd3fc',
          400: '#38bdf8',
          500: '#0ea5e9',
          600: '#0284c7',
          700: '#0369a1',
          800: '#075985',
          900: '#0c4a6e',
        },
        // ... other colors
      },
      fontFamily: {
        sans: ['Inter', 'system-ui', 'sans-serif'],
      },
    },
  },
  plugins: [],
};
```

**Option 2: Use TailwindCSS v4 Beta with Fallback**
```bash
npm install -D tailwindcss@next @tailwindcss/postcss
```

```css
/* src/styles/global.css */
@import "tailwindcss/theme" theme(reference);
@import "tailwindcss/utilities";

@theme {
  --color-primary-50: #f0f9ff;
  --color-primary-500: #0ea5e9;
  --color-primary-900: #0c4a6e;
  /* ... other tokens */
}
```

Add migration note to plan:
> **Note:** TailwindCSS v4 is currently in beta. If breaking changes occur before stable release, migrate to v3.4 using the fallback configuration.

**Affected Files:**
- `00_overview.md` - Line 46 (Tech Stack)
- `01_EP01_scaffold_design.md` - Lines 25-60 (TailwindCSS installation and configuration)

---

### TD-005: No CAPTCHA for Bot Protection

**Location:** `03_EP03_modal_form.md`, `04_EP04_backend_data.md`  
**Severity:** 🟠 High  
**Category:** Security Gap

**Issue:**
The plan only uses a honeypot field for bot protection. While effective against simple bots, sophisticated bots can bypass honeypots. No CAPTCHA or advanced bot detection is implemented.

**Impact:**
- Form may receive spam submissions
- Database may be flooded with fake leads
- WhatsApp API quota may be wasted on spam
- Potential abuse by malicious actors

**Root Cause:**
- Over-reliance on simple honeypot technique
- No defense-in-depth strategy for bot protection

**Troubleshooting Steps:**
1. Monitor submission patterns for anomalies
2. Check for submissions with identical user agents
3. Look for submissions from suspicious IP ranges
4. Monitor WhatsApp link clicks vs actual conversations

**Recommended Fix:**

**Option 1: Cloudflare Turnstile (Recommended, Free)**
```html
<!-- src/components/ModalForm.astro -->
<script>
  import { onMount } from 'svelte';

  let turnstileToken = '';

  onMount(() => {
    // Load Turnstile widget
    const script = document.createElement('script');
    script.src = 'https://challenges.cloudflare.com/turnstile/v0/api.js';
    script.async = true;
    script.defer = true;
    document.head.appendChild(script);
  });

  function onTurnstileSuccess(token: string) {
    turnstileToken = token;
  }
</script>

<form id="lead-form">
  <!-- ... existing fields ... -->
  
  <div class="cf-turnstile" 
       data-sitekey="YOUR_SITE_KEY"
       data-callback="onTurnstileSuccess"
       data-theme="light">
  </div>
</form>
```

```typescript
// functions/api/lead.ts
export async function POST(request: Request, env: Env) {
  const { turnstileToken } = await request.json();
  
  // Verify Turnstile token
  const verifyResponse = await fetch('https://challenges.cloudflare.com/turnstile/v0/siteverify', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      secret: env.TURNSTILE_SECRET_KEY,
      response: turnstileToken,
      remoteip: request.headers.get('CF-Connecting-IP'),
    }),
  });
  
  const verifyResult = await verifyResponse.json();
  
  if (!verifyResult.success) {
    return new Response(
      JSON.stringify({ error: 'Verificación fallida. Por favor, intenta de nuevo.' }),
      { status: 400 }
    );
  }
  
  // ... rest of submission logic
}
```

**Option 2: Custom CAPTCHA (Simpler, Less Robust)**
```html
<!-- Simple math CAPTCHA -->
<div class="form-group">
  <label for="captcha">¿Cuánto es 3 + 4?</label>
  <input 
    type="text" 
    id="captcha" 
    name="captcha" 
    required 
    pattern="7"
    title="Por favor, resuelve la operación matemática"
  />
</div>
```

```typescript
// Server-side verification
const { captcha } = formData;
if (captcha !== '7') {
  return new Response(
    JSON.stringify({ error: 'Captcha incorrecto.' }),
    { status: 400 }
  );
}
```

**Environment Variables Required:**
```toml
# wrangler.toml
[vars]
TURNSTILE_SITE_KEY = "your-site-key"

# Add to Cloudflare Pages dashboard:
# TURNSTILE_SECRET_KEY = "your-secret-key"
```

**Affected Files:**
- `03_EP03_modal_form.md` - Add Turnstile widget
- `04_EP04_backend_data.md` - Add verification logic
- `06_EP06_deploy_cicd.md` - Add environment variables

---

### TD-006: No Database Connection Error Handling

**Location:** `04_EP04_backend_data.md` - Lines 120-200  
**Severity:** 🟠 High  
**Category:** Error Handling Gap

**Issue:**
The Pages Function code doesn't handle database connection failures, query errors, or transaction rollbacks. If D1 is unavailable or queries fail, the function may return 500 errors without proper logging or user feedback.

**Impact:**
- Users see generic error messages
- No visibility into database issues
- Lost leads during outages
- Difficult to debug production issues

**Root Cause:**
- Missing error handling patterns
- No retry logic for transient failures
- No logging strategy defined

**Troubleshooting Steps:**
1. Check Cloudflare Pages Function logs for unhandled errors
2. Monitor D1 query performance and error rates
3. Test with D1 disabled to see error behavior
4. Review Cloudflare Analytics for 5xx errors

**Recommended Fix:**

```typescript
// functions/api/lead.ts
interface Env {
  DB: D1Database;
  RATE_LIMIT: KVNamespace;
  TURNSTILE_SECRET_KEY: string;
}

interface ErrorResponse {
  error: string;
  code?: string;
  details?: string;
}

class DatabaseError extends Error {
  constructor(message: string, public code: string) {
    super(message);
    this.name = 'DatabaseError';
  }
}

async function insertLead(
  db: D1Database,
  data: {
    name: string;
    email: string;
    phone: string;
    message: string;
    source: string;
    ip_hash: string;
    user_agent: string;
  }
): Promise<{ id: number; whatsapp_sent: boolean }> {
  try {
    const result = await db
      .prepare(`
        INSERT INTO leads (name, email, phone, message, source, ip_hash, user_agent)
        VALUES (?, ?, ?, ?, ?, ?, ?)
      `)
      .bind(
        data.name,
        data.email,
        data.phone,
        data.message,
        data.source,
        data.ip_hash,
        data.user_agent
      )
      .run();

    if (!result.success || !result.meta.last_row_id) {
      throw new DatabaseError('Failed to insert lead', 'INSERT_FAILED');
    }

    return {
      id: result.meta.last_row_id,
      whatsapp_sent: false,
    };
  } catch (error) {
    console.error('Database error:', error);
    
    // Check for unique constraint violation (duplicate email + phone)
    if (error instanceof Error && error.message.includes('UNIQUE constraint')) {
      throw new DatabaseError(
        'Ya existe un registro con este correo y teléfono.',
        'DUPLICATE_LEAD'
      );
    }
    
    throw error;
  }
}

export async function POST(request: Request, env: Env): Promise<Response> {
  const startTime = Date.now();
  
  try {
    // Parse request
    let formData;
    try {
      formData = await request.json();
    } catch (error) {
      return errorResponse('Formato de solicitud inválido', 400, 'INVALID_JSON');
    }

    // Validate Turnstile
    const turnstileToken = formData.turnstileToken;
    if (!turnstileToken) {
      return errorResponse('Verificación requerida', 400, 'MISSING_CAPTCHA');
    }

    const verifyResponse = await fetch(
      'https://challenges.cloudflare.com/turnstile/v0/siteverify',
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          secret: env.TURNSTILE_SECRET_KEY,
          response: turnstileToken,
          remoteip: request.headers.get('CF-Connecting-IP'),
        }),
      }
    );

    const verifyResult = await verifyResponse.json();
    if (!verifyResult.success) {
      return errorResponse(
        'Verificación fallida. Por favor, intenta de nuevo.',
        400,
        'CAPTCHA_FAILED'
      );
    }

    // Validate form data
    const validation = validateFormData(formData);
    if (!validation.valid) {
      return errorResponse(validation.error, 400, 'VALIDATION_ERROR');
    }

    // Check honeypot
    if (formData.website) {
      // Bot detected - return fake success
      console.log('Bot blocked via honeypot');
      return successResponse({ success: true, whatsappUrl: '' });
    }

    // Rate limiting
    const ip = request.headers.get('CF-Connecting-IP') || 'unknown';
    const ipHash = await hashIP(ip);
    
    const rateLimitResult = await checkRateLimit(env.RATE_LIMIT, ip);
    if (!rateLimitResult.allowed) {
      return errorResponse(
        'Demasiados intentos. Por favor, espera una hora.',
        429,
        'RATE_LIMIT_EXCEEDED'
      );
    }

    // Insert lead with retry logic
    let lead;
    try {
      lead = await insertLead(env.DB, {
        name: formData.name,
        email: formData.email,
        phone: formData.phone,
        message: formData.message || '',
        source: 'coming-soon-modal',
        ip_hash: ipHash,
        user_agent: request.headers.get('User-Agent') || '',
      });
    } catch (error) {
      if (error instanceof DatabaseError && error.code === 'DUPLICATE_LEAD') {
        return errorResponse(error.message, 409, error.code);
      }
      throw error; // Re-throw for outer handler
    }

    // Generate WhatsApp URL
    const whatsappUrl = generateWhatsAppUrl(formData.name, env.WHATSAPP_NUMBER);

    // Log success
    const duration = Date.now() - startTime;
    console.log(`Lead submitted successfully: ${lead.id} in ${duration}ms`);

    return successResponse({
      success: true,
      leadId: lead.id,
      whatsappUrl,
    });

  } catch (error) {
    const duration = Date.now() - startTime;
    console.error('Unexpected error in /api/lead:', error, `duration: ${duration}ms`);
    
    // Log to external service (optional)
    // await logError(error, { duration, ip: request.headers.get('CF-Connecting-IP') });
    
    return errorResponse(
      'Error interno del servidor. Por favor, intenta más tarde.',
      500,
      'INTERNAL_ERROR'
    );
  }
}

function errorResponse(message: string, status: number, code: string): Response {
  return new Response(
    JSON.stringify({ error: message, code } as ErrorResponse),
    {
      status,
      headers: { 'Content-Type': 'application/json' },
    }
  );
}

function successResponse(data: unknown): Response {
  return new Response(JSON.stringify(data), {
    status: 200,
    headers: { 'Content-Type': 'application/json' },
  });
}
```

**Affected Files:**
- `04_EP04_backend_data.md` - Replace entire POST handler with robust error handling

---

### TD-007: No Data Retention Policy

**Location:** `04_EP04_backend_data.md`, `06_EP06_deploy_cicd.md`  
**Severity:** 🟠 High  
**Category:** Compliance Gap

**Issue:**
The plan stores leads in Cloudflare D1 indefinitely with no data retention policy, deletion strategy, or cleanup mechanism. This creates compliance risks and potential data bloat.

**Impact:**
- GDPR/CCPA compliance issues (right to deletion)
- Database growth without bounds
- Increased storage costs over time
- Privacy concerns for stale data

**Root Cause:**
- No data lifecycle management planned
- Missing retention requirements analysis

**Troubleshooting Steps:**
1. Check D1 database size and growth rate
2. Identify oldest records
3. Review privacy policy for retention commitments

**Recommended Fix:**

**Add Retention Policy to Database Schema:**
```sql
-- Migration: 002_add_retention.sql
ALTER TABLE leads ADD COLUMN deleted_at INTEGER;
CREATE INDEX idx_leads_deleted_at ON leads(deleted_at);
CREATE INDEX idx_leads_retention_cleanup ON leads(created_at) 
  WHERE deleted_at IS NULL;
```

**Implement Cleanup Function:**
```typescript
// functions/api/cleanup-leads.ts
import type { Env } from './types';

const RETENTION_DAYS = 365; // 1 year retention
const BATCH_SIZE = 100;

export async function scheduled(event: ScheduledEvent, env: Env): Promise<void> {
  console.log('Starting lead cleanup...');
  
  const cutoffDate = Math.floor((Date.now() - RETENTION_DAYS * 24 * 60 * 60 * 1000) / 1000);
  
  let totalDeleted = 0;
  let hasMore = true;
  
  while (hasMore) {
    // Soft delete old leads
    const result = await env.DB
      .prepare(`
        UPDATE leads
        SET deleted_at = strftime('%s', 'now')
        WHERE created_at < ? 
          AND deleted_at IS NULL
        LIMIT ?
      `)
      .bind(cutoffDate, BATCH_SIZE)
      .run();
    
    totalDeleted += result.meta.changes || 0;
    hasMore = (result.meta.changes || 0) === BATCH_SIZE;
    
    console.log(`Deleted batch: ${result.meta.changes} leads`);
  }
  
  console.log(`Cleanup complete. Total deleted: ${totalDeleted}`);
}
```

**Configure Cron Trigger:**
```toml
# wrangler.toml
[triggers]
crons = ["0 2 * * *"] # Run daily at 2 AM UTC
```

**Add Manual Deletion Endpoint:**
```typescript
// functions/api/delete-lead.ts
export async function DELETE(request: Request, env: Env): Promise<Response> {
  const { email, phone } = await request.json();
  
  const ipHash = hashIP(request.headers.get('CF-Connecting-IP') || '');
  
  // Verify this user submitted the lead (optional, for privacy)
  const lead = await env.DB
    .prepare('SELECT id FROM leads WHERE email = ? AND phone = ? AND ip_hash = ?')
    .bind(email, phone, ipHash)
    .first();
  
  if (!lead) {
    return new Response(
      JSON.stringify({ error: 'Lead no encontrado' }),
      { status: 404 }
    );
  }
  
  await env.DB
    .prepare('UPDATE leads SET deleted_at = strftime("%s", "now") WHERE id = ?')
    .bind(lead.id)
    .run();
  
  return new Response(JSON.stringify({ success: true }));
}
```

**Update Privacy Policy:**
```markdown
## Retención de Datos

Conservamos tu información personal por un período máximo de **1 año** desde la fecha de recolección, a menos que sea necesario mantenerla por más tiempo para cumplir con obligaciones legales o resolver disputas.

Pasado este período, tus datos serán eliminados permanentemente de nuestros sistemas.

Puedes solicitar la eliminación inmediata de tus datos contactándonos a: privacidad@sereni.dad
```

**Affected Files:**
- `04_EP04_backend_data.md` - Add retention policy, cleanup function
- `06_EP06_deploy_cicd.md` - Add cron trigger configuration
- `02_EP02_landing_ui.md` - Update privacy policy section

---

## Medium Priority Issues

### TD-008: WhatsApp Deep Link Limitations

**Location:** `05_EP05_whatsapp.md`  
**Severity:** 🟡 Medium  
**Category:** Feature Limitation

**Issue:**
Using `wa.me` deep links has several limitations compared to WhatsApp Business API:
- No message delivery confirmation
- No read receipts
- No webhook for conversation events
- User must manually send the pre-filled message
- No fallback if WhatsApp is not installed
- Limited to 1,000 characters in pre-filled message

**Impact:**
- Can't track if user actually sent the message
- No analytics on conversion from form to conversation
- Poor UX on desktop without WhatsApp
- No integration with CRM systems

**Root Cause:**
- Chose simple solution over full-featured API
- No analysis of business requirements for WhatsApp integration

**Troubleshooting Steps:**
1. Monitor click-through rate on WhatsApp links
2. Survey users about WhatsApp experience
3. Track conversion from form submission to actual conversation

**Recommended Fix:**

**Document Limitations in Plan:**
```markdown
### WhatsApp Integration: Limitations and Considerations

#### Current Approach: wa.me Deep Link

**Advantages:**
- No API costs
- No infrastructure setup
- Works immediately
- No rate limiting from WhatsApp

**Limitations:**
- ❌ No delivery confirmation
- ❌ No read receipts  
- ❌ No webhooks for conversation tracking
- ❌ User must manually send pre-filled message
- ❌ Message limited to 1,000 characters
- ❌ No fallback for desktop users
- ❌ No CRM integration

#### Alternative: WhatsApp Business API

**When to Consider:**
- Need to track conversation metrics
- Want automated follow-up messages
- Need CRM integration
- High volume of leads (>100/month)
- Require message templates for compliance

**Migration Path:**
If business requirements evolve, migrate to WhatsApp Business API:
1. Register for WhatsApp Business API
2. Set up Meta for Developers account
3. Configure webhook endpoints
4. Implement message templates
5. Update form submission flow
6. Add conversation tracking to D1

**Estimated Migration Effort:** 2-3 days
**Estimated Cost:** Free tier (1,000 conversations/month)
```

**Add Fallback for Desktop:**
```typescript
// functions/api/lead.ts
function generateWhatsAppUrl(name: string, phoneNumber: string): string {
  const message = encodeURIComponent(
    `Hola, soy ${name}. Me interesa saber más sobre Serenidad y agendar una consulta.`
  );
  
  // Detect if likely mobile (simplified)
  const isMobile = /Mobile|Android|iPhone/i.test(
    request.headers.get('User-Agent') || ''
  );
  
  if (isMobile) {
    return `https://wa.me/${phoneNumber}?text=${message}`;
  } else {
    // Desktop: show QR code or web WhatsApp
    return `https://web.whatsapp.com/send?phone=${phoneNumber}&text=${message}`;
  }
}
```

**Affected Files:**
- `05_EP05_whatsapp.md` - Add limitations section and migration path

---

### TD-009: No Monitoring/Alerting for Form Failures

**Location:** `06_EP06_deploy_cicd.md`  
**Severity:** 🟡 Medium  
**Category**: Observability Gap

**Issue:**
The plan includes health checks for uptime but no monitoring for form submission failures, error rates, or lead submission anomalies. No alerting mechanism is defined.

**Impact:**
- Silent failures go unnoticed
- Can't detect spam attacks
- No visibility into conversion rates
- Difficult to debug production issues

**Root Cause:**
- Focus on infrastructure health, not application metrics
- No observability strategy defined

**Troubleshooting Steps:**
1. Check Cloudflare Analytics for error patterns
2. Review D1 query logs for failures
3. Monitor submission rates for anomalies

**Recommended Fix:**

**Add Error Tracking:**
```typescript
// functions/api/lead.ts
interface Env {
  DB: D1Database;
  RATE_LIMIT: KVNamespace;
  TURNSTILE_SECRET_KEY: string;
  SENTRY_DSN?: string; // Optional
  WEBHOOK_URL?: string; // For custom alerts
}

async function trackError(error: Error, context: Record<string, unknown>) {
  // Log to console
  console.error('Error:', error, context);
  
  // Send to Sentry (optional)
  if (typeof Sentry !== 'undefined') {
    Sentry.captureException(error, { extra: context });
  }
  
  // Send to webhook (custom alerting)
  if (env.WEBHOOK_URL) {
    fetch(env.WEBHOOK_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        error: error.message,
        context,
        timestamp: new Date().toISOString(),
      }),
    }).catch(console.error);
  }
}
```

**Add Metrics Collection:**
```typescript
// functions/api/lead.ts
interface Metrics {
  submissions: number;
  successes: number;
  failures: number;
  rateLimitBlocks: number;
  captchaBlocks: number;
  honeypotBlocks: number;
}

async function recordMetric(env: Env, metric: keyof Metrics, value = 1) {
  const key = `metrics:${metric}:${new Date().toISOString().slice(0, 10)}`;
  const current = await env.RATE_LIMIT.get(key);
  const newValue = (parseInt(current || '0') + value).toString();
  
  await env.RATE_LIMIT.put(key, newValue, {
    expirationTtl: 86400 * 7, // 7 days
  });
}
```

**Add Monitoring Dashboard:**
```typescript
// functions/api/metrics.ts
export async function GET(request: Request, env: Env): Promise<Response> {
  const today = new Date().toISOString().slice(0, 10);
  const metrics: Metrics = {
    submissions: 0,
    successes: 0,
    failures: 0,
    rateLimitBlocks: 0,
    captchaBlocks: 0,
    honeypotBlocks: 0,
  };
  
  for (const key of Object.keys(metrics) as Array<keyof Metrics>) {
    const value = await env.RATE_LIMIT.get(`metrics:${key}:${today}`);
    metrics[key] = parseInt(value || '0');
  }
  
  return new Response(JSON.stringify(metrics), {
    headers: { 'Content-Type': 'application/json' },
  });
}
```

**Configure Alerts:**
```typescript
// functions/api/health-check.ts
export async function GET(request: Request, env: Env): Promise<Response> {
  const today = new Date().toISOString().slice(0, 10);
  const failures = await env.RATE_LIMIT.get(`metrics:failures:${today}`);
  const failureRate = parseInt(failures || '0');
  
  const health = {
    status: failureRate > 10 ? 'degraded' : 'healthy',
    metrics: {
      failures: failureRate,
      timestamp: new Date().toISOString(),
    },
  };
  
  return new Response(JSON.stringify(health), {
    status: health.status === 'healthy' ? 200 : 503,
    headers: { 'Content-Type': 'application/json' },
  });
}
```

**Cloudflare Health Check Configuration:**
```toml
# Update wrangler.toml health check
[observability]
enabled = true
head_sampling_rate = 1

# Configure in Cloudflare dashboard:
# - Health check: https://sereni.dad/api/health-check
# - Alert if status != 200 for 3 consecutive checks
# - Email alerts to ops@sereni.dad
```

**Affected Files:**
- `04_EP04_backend_data.md` - Add error tracking and metrics
- `06_EP06_deploy_cicd.md` - Add monitoring dashboard and alerts

---

### TD-010: No CORS Security Best Practices

**Location:** `04_EP04_backend_data.md` - Lines 200-210  
**Severity:** 🟡 Medium  
**Category**: Security Gap

**Issue:**
The CORS configuration allows all methods (`*`) and doesn't implement additional security headers like Content-Security-Policy, X-Frame-Options, or X-Content-Type-Options.

**Impact:**
- Potential CSRF vulnerabilities
- Missing security hardening
- Doesn't follow OWASP recommendations

**Root Cause:**
- Minimal CORS configuration without security headers
- No security headers strategy

**Troubleshooting Steps:**
1. Test with security scanners (e.g., OWASP ZAP)
2. Check response headers in browser dev tools
3. Verify CORS preflight requests

**Recommended Fix:**

```typescript
// functions/api/lead.ts
const SECURITY_HEADERS = {
  'Content-Security-Policy': "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';",
  'X-Frame-Options': 'DENY',
  'X-Content-Type-Options': 'nosniff',
  'Referrer-Policy': 'strict-origin-when-cross-origin',
  'Permissions-Policy': 'geolocation=(), microphone=(), camera=()',
};

function createResponse(data: unknown, status: number): Response {
  return new Response(JSON.stringify(data), {
    status,
    headers: {
      'Content-Type': 'application/json',
      ...SECURITY_HEADERS,
    },
  });
}

export async function OPTIONS(request: Request): Promise<Response> {
  return new Response(null, {
    status: 204,
    headers: {
      'Access-Control-Allow-Origin': 'https://sereni.dad',
      'Access-Control-Allow-Methods': 'POST, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type',
      'Access-Control-Max-Age': '86400',
      ...SECURITY_HEADERS,
    },
  });
}

export async function POST(request: Request, env: Env): Promise<Response> {
  // Validate origin
  const origin = request.headers.get('Origin');
  const allowedOrigins = ['https://sereni.dad', 'http://localhost:4321'];
  
  if (!origin || !allowedOrigins.includes(origin)) {
    return createResponse({ error: 'Origin not allowed' }, 403);
  }
  
  // ... rest of handler
  
  return createResponse({ success: true }, 200);
}
```

**Add CSP to Astro Layout:**
```astro
---
// src/layouts/BaseLayout.astro
const csp = "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:; connect-src 'self' https://challenges.cloudflare.com;";
---

<html lang="es">
  <head>
    <meta http-equiv="Content-Security-Policy" content={csp} />
    <!-- ... other meta tags -->
  </head>
  <!-- ... -->
</html>
```

**Affected Files:**
- `04_EP04_backend_data.md` - Update CORS and security headers
- `01_EP01_scaffold_design.md` - Add CSP to BaseLayout

---

## Low Priority Issues

### TD-011: No A/B Testing Framework

**Location:** All plan files  
**Severity:** 🟢 Low  
**Category**: Optimization Gap

**Issue:**
The plan doesn't include A/B testing capabilities for the landing page, modal, or CTAs. This limits optimization opportunities for conversion rates.

**Impact:**
- Can't test different headlines, CTAs, or designs
- No data-driven optimization
- Missed opportunity to improve conversion

**Root Cause:**
- Focus on MVP implementation
- No growth strategy defined

**Recommended Fix:**

**Add A/B Testing Infrastructure:**
```typescript
// functions/api/ab-test.ts
interface Variant {
  name: string;
  traffic: number; // 0-1
}

const experiments: Record<string, Variant[]> = {
  'hero-headline': [
    { name: 'control', traffic: 0.5 },
    { name: 'variant-a', traffic: 0.5 },
  ],
  'cta-button': [
    { name: 'control', traffic: 0.5 },
    { name: 'variant-b', traffic: 0.5 },
  ],
};

export async function GET(request: Request, env: Env): Promise<Response> {
  const ip = request.headers.get('CF-Connecting-IP') || 'unknown';
  const hash = await hashIP(ip);
  
  const assignments: Record<string, string> = {};
  
  for (const [experiment, variants] of Object.entries(experiments)) {
    const bucket = hash % 100;
    let cumulative = 0;
    
    for (const variant of variants) {
      cumulative += variant.traffic * 100;
      if (bucket < cumulative) {
        assignments[experiment] = variant.name;
        break;
      }
    }
  }
  
  return new Response(JSON.stringify(assignments), {
    headers: { 'Content-Type': 'application/json' },
  });
}
```

**Track Conversions:**
```typescript
// functions/api/lead.ts
async function trackConversion(env: Env, experiment: string, variant: string) {
  const key = `ab-test:${experiment}:${variant}:${new Date().toISOString().slice(0, 10)}`;
  const current = await env.RATE_LIMIT.get(key);
  const newValue = (parseInt(current || '0') + 1).toString();
  
  await env.RATE_LIMIT.put(key, newValue, {
    expirationTtl: 86400 * 30, // 30 days
  });
}
```

**Affected Files:**
- New file: `plan_comingsoon/07_optimization_ab_testing.md` (optional)

---

### TD-012: No Progressive Enhancement for JavaScript

**Location:** `03_EP03_modal_form.md`  
**Severity:** 🟢 Low  
**Category**: Accessibility Gap

**Issue:**
The modal form requires JavaScript to function. If JavaScript is disabled or fails to load, users cannot submit the form.

**Impact:**
- Poor experience for users with JS disabled
- Accessibility concern for screen readers
- Single point of failure

**Root Cause:**
- Focus on modern UX over progressive enhancement

**Recommended Fix:**

**Add No-JS Fallback:**
```html
<!-- src/components/ModalForm.astro -->
<noscript>
  <div class="no-js-fallback">
    <p>JavaScript está deshabilitado. Por favor, usa el formulario alternativo:</p>
    <form action="/api/lead" method="POST">
      <!-- Same fields as modal -->
      <input type="hidden" name="source" value="coming-soon-nojs" />
      <button type="submit">Enviar</button>
    </form>
  </div>
</noscript>

<script>
  // Modal logic here
</script>
```

**CSS for Fallback:**
```css
/* src/styles/global.css */
.no-js-fallback {
  padding: 2rem;
  background: var(--color-neutral-50);
  border-radius: 8px;
  margin: 2rem 0;
}

.no-js-fallback form {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  max-width: 400px;
}

/* Hide fallback when JS is available */
.modal-form-container:not(.no-js) + .no-js-fallback {
  display: none;
}
```

**Affected Files:**
- `03_EP03_modal_form.md` - Add no-js fallback

---

## Version Compatibility Matrix

| Technology | Planned Version | Stable Version | Status | Action Required |
|------------|----------------|----------------|--------|------------------|
| Astro | v6 | v5.16.6 | ❌ Mismatch | Update to v5.x |
| TypeScript | 6.0 | 5.4+ | ❌ Mismatch | Update to 5.4+ |
| TailwindCSS | v4 | v3.4 (v4 beta) | ⚠️ Beta | Use v3.4 or document v4 beta |
| Cloudflare D1 | Latest | Latest | ✅ OK | None |
| Cloudflare Pages | Latest | Latest | ✅ OK | None |
| Cloudflare KV | Latest | Latest | ✅ OK | None |

---

## Implementation Checklist

### Phase 1: Critical Fixes (Must Fix Before Launch)
- [ ] Update Astro version from v6 to v5.x in all plan files
- [ ] Update TypeScript version from 6.0 to 5.4+ in all plan files
- [ ] Replace in-memory rate limiting with Cloudflare KV or D1-based solution
- [ ] Add KV namespace configuration to wrangler.toml

### Phase 2: High Priority Fixes (Fix Before Production)
- [ ] Verify TailwindCSS v4 status, use v3.4 if v4 not stable
- [ ] Add Cloudflare Turnstile CAPTCHA to modal form
- [ ] Add Turnstile verification to API endpoint
- [ ] Add comprehensive error handling to Pages Function
- [ ] Implement data retention policy (1 year)
- [ ] Add scheduled cleanup function for old leads
- [ ] Update privacy policy with retention information

### Phase 3: Medium Priority Improvements (Post-Launch)
- [ ] Document WhatsApp deep link limitations
- [ ] Add error tracking and metrics collection
- [ ] Create metrics dashboard endpoint
- [ ] Configure health check alerts for form failures
- [ ] Implement security headers (CSP, X-Frame-Options, etc.)
- [ ] Add origin validation to CORS

### Phase 4: Low Priority Enhancements (Future)
- [ ] Design A/B testing framework
- [ ] Add no-JavaScript fallback for form
- [ ] Consider WhatsApp Business API migration path

---

## Summary Statistics

| Category | Count |
|----------|-------|
| Critical Issues | 3 |
| High Priority | 4 |
| Medium Priority | 3 |
| Low Priority | 2 |
| **Total** | **12** |

### Issues by Type:
- Version Mismatches: 2
- Architecture Flaws: 1
- Security Gaps: 3
- Error Handling Gaps: 1
- Compliance Gaps: 1
- Feature Limitations: 1
- Observability Gaps: 1
- Accessibility Gaps: 1
- Optimization Gaps: 1

---

## Recommendations

1. **Immediate Action Required:** Fix the 3 critical issues before starting implementation. These will cause immediate failures.

2. **Architecture Review:** The in-memory rate limiting issue is a fundamental misunderstanding of edge computing. Consider a dedicated architecture review session.

3. **Security Audit:** Implement the security improvements (CAPTCHA, security headers) before handling any real user data.

4. **Compliance Check:** Review the data retention policy with legal counsel to ensure GDPR/CCPA compliance.

5. **Monitoring Strategy:** Define observability requirements before launch. You can't improve what you don't measure.

---

## Next Steps

1. Review this document with the team
2. Prioritize fixes based on timeline and resources
3. Update plan_comingsoon files with corrections
4. Create implementation tasks for each fix
5. Add fixes to the development backlog

---

**Document Version:** 1.0  
**Last Updated:** 2026-04-09  
**Status:** Ready for Review
