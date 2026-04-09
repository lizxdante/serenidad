# EP-CS03 — Modal Form & Validation

**Épica:** Como visitante interesado, quiero un formulario modal elegante y fácil de completar que valide mis datos en tiempo real, para que pueda dejar mi información de contacto sin fricción y ser contactado por WhatsApp.

**Prioridad:** Must Have — Core functionality
**Sprint:** Sprint 2
**Dependencias Entrantes:** EP-CS01 (scaffold + design system), EP-CS02 (landing UI con CTAs)
**Dependencias Salientes:** EP-CS04 (backend API), EP-CS05 (WhatsApp integration)

---

## HU-CS03.1 — Modal Component & Trigger

**Como** visitante que hace click en el CTA,
**quiero** que se abra un modal elegante con el formulario de contacto,
**para que** pueda dejar mis datos sin abandonar la página.

### Principios de UX

- **Modal overlay:** Backdrop oscuro semi-transparente que enfoca la atención
- **Animación suave:** Fade in/out, sin parpadeos
- **Focus trap:** Tab/Shift+Tab cicla dentro del modal
- **Escape key:** Cierra el modal
- **Click fuera:** Cierra el modal
- **Scroll lock:** Body no scrollea cuando modal está abierto
- **Accesibilidad:** `role="dialog"`, `aria-modal="true"`, `aria-labelledby`

### Tareas y Subtareas

#### T-CS03.1.1 — Crear componente LeadModal

- **ST-CS03.1.1.1** — Crear `src/components/LeadModal.astro`.

  **Layout del Modal:**
  ```
  ┌─────────────────────────────────────────────────────┐
  │ ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │
  │ ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │
  │ ░░░░░░░┌─────────────────────────┐░░░░░░░░░░░░░░░░ │
  │ ░░░░░░░│  ✕                      │░░░░░░░░░░░░░░░░ │
  │ ░░░░░░░│                         │░░░░░░░░░░░░░░░░ │
  │ ░░░░░░░│  Sé el primero en       │░░░░░░░░░░░░░░░░ │
  │ ░░░░░░░│  acceder a Serenidad    │░░░░░░░░░░░░░░░░ │
  │ ░░░░░░░│                         │░░░░░░░░░░░░░░░░ │
  │ ░░░░░░░│  Nombre _______________ │░░░░░░░░░░░░░░░░ │
  │ ░░░░░░░│  Email  _______________ │░░░░░░░░░░░░░░░░ │
  │ ░░░░░░░│  Teléf  _______________ │░░░░░░░░░░░░░░░░ │
  │ ░░░░░░░│  Mensaj _______________ │░░░░░░░░░░░░░░░░ │
  │ ░░░░░░░│        _______________ │░░░░░░░░░░░░░░░░ │
  │ ░░░░░░░│                         │░░░░░░░░░░░░░░░░ │
  │ ░░░░░░░│  [💬 Enviar y contactar │░░░░░░░░░░░░░░░░ │
  │ ░░░░░░░│       por WhatsApp]     │░░░░░░░░░░░░░░░░ │
  │ ░░░░░░░│                         │░░░░░░░░░░░░░░░░ │
  │ ░░░░░░░│  Tus datos están seguros│░░░░░░░░░░░░░░░░ │
  │ ░░░░░░░└─────────────────────────┘░░░░░░░░░░░░░░░░ │
  │ ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │
  └─────────────────────────────────────────────────────┘
  ```

  **Código del componente:**
  ```astro
  ---
  ---

  <!-- Modal Backdrop + Container -->
  <div
    id="lead-modal"
    class="fixed inset-0 z-50 hidden"
    role="dialog"
    aria-modal="true"
    aria-labelledby="modal-title"
  >
    <!-- Backdrop -->
    <div
      id="modal-backdrop"
      class="absolute inset-0 bg-black/50 backdrop-blur-sm transition-opacity"
      aria-hidden="true"
    ></div>

    <!-- Modal Content -->
    <div class="relative flex items-center justify-center min-h-full p-4">
      <div
        id="modal-content"
        class="relative bg-white rounded-2xl shadow-xl w-full max-w-md p-8 transform transition-all"
      >
        <!-- Close button -->
        <button
          id="modal-close"
          class="absolute top-4 right-4 text-text-muted hover:text-text transition-fast p-1 rounded-lg hover:bg-gray-100"
          aria-label="Cerrar formulario"
        >
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"
               fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"
               stroke-linejoin="round">
            <line x1="18" y1="6" x2="6" y2="18"></line>
            <line x1="6" y1="6" x2="18" y2="18"></line>
          </svg>
        </button>

        <!-- Header -->
        <div class="mb-6">
          <h2 id="modal-title" class="text-2xl font-display font-bold text-text mb-2">
            Sé el primero en acceder
          </h2>
          <p class="text-sm text-text-muted">
            Deja tus datos y te contactaremos por WhatsApp para agendar tu primera consulta.
          </p>
        </div>

        <!-- Form -->
        <form id="lead-form" class="space-y-4" novalidate>
          <!-- Honeypot field (anti-spam) -->
          <div class="hidden" aria-hidden="true">
            <label for="website">Website</label>
            <input type="text" id="website" name="website" tabindex="-1" autocomplete="off" />
          </div>

          <!-- Name -->
          <div>
            <label for="lead-name" class="block text-sm font-medium text-text mb-1">
              Nombre completo <span class="text-red-500">*</span>
            </label>
            <input
              type="text"
              id="lead-name"
              name="name"
              required
              minlength="2"
              autocomplete="name"
              placeholder="Tu nombre completo"
              class="w-full px-4 py-3 border border-border rounded-lg focus:ring-2 focus:ring-primary focus:border-primary transition-fast text-text placeholder:text-text-light"
              aria-describedby="name-error"
            />
            <p id="name-error" class="text-red-500 text-xs mt-1 hidden" role="alert"></p>
          </div>

          <!-- Email -->
          <div>
            <label for="lead-email" class="block text-sm font-medium text-text mb-1">
              Email <span class="text-red-500">*</span>
            </label>
            <input
              type="email"
              id="lead-email"
              name="email"
              required
              autocomplete="email"
              placeholder="tu@email.com"
              class="w-full px-4 py-3 border border-border rounded-lg focus:ring-2 focus:ring-primary focus:border-primary transition-fast text-text placeholder:text-text-light"
              aria-describedby="email-error"
            />
            <p id="email-error" class="text-red-500 text-xs mt-1 hidden" role="alert"></p>
          </div>

          <!-- Phone -->
          <div>
            <label for="lead-phone" class="block text-sm font-medium text-text mb-1">
              Teléfono <span class="text-red-500">*</span>
            </label>
            <input
              type="tel"
              id="lead-phone"
              name="phone"
              required
              autocomplete="tel"
              placeholder="+52 55 1234 5678"
              class="w-full px-4 py-3 border border-border rounded-lg focus:ring-2 focus:ring-primary focus:border-primary transition-fast text-text placeholder:text-text-light"
              aria-describedby="phone-error"
            />
            <p id="phone-error" class="text-red-500 text-xs mt-1 hidden" role="alert"></p>
          </div>

          <!-- Message (optional) -->
          <div>
            <label for="lead-message" class="block text-sm font-medium text-text mb-1">
              Mensaje <span class="text-text-light">(opcional)</span>
            </label>
            <textarea
              id="lead-message"
              name="message"
              rows="3"
              maxlength="500"
              placeholder="¿En qué podemos ayudarte?"
              class="w-full px-4 py-3 border border-border rounded-lg focus:ring-2 focus:ring-primary focus:border-primary transition-fast text-text placeholder:text-text-light resize-none"
            ></textarea>
          </div>

          <!-- Submit button -->
          <button
            type="submit"
            id="lead-submit"
            class="w-full inline-flex items-center justify-center gap-2 bg-whatsapp text-white py-4 rounded-xl font-semibold text-lg hover:bg-whatsapp-dark transition-base shadow-lg focus:ring-2 focus:ring-whatsapp focus:ring-offset-2 disabled:opacity-50 disabled:cursor-not-allowed"
          >
            <span id="submit-text">💬 Enviar y contactar por WhatsApp</span>
            <span id="submit-loading" class="hidden">
              <svg class="animate-spin h-5 w-5 inline" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z"></path>
              </svg>
              Enviando...
            </span>
          </button>

          <!-- Privacy notice -->
          <p class="text-xs text-text-light text-center">
            Al enviar, aceptas nuestra
            <a href="/privacidad" class="text-primary underline hover:text-primary-dark">política de privacidad</a>.
            Tus datos están seguros.
          </p>
        </form>

        <!-- Success state (hidden by default) -->
        <div id="lead-success" class="hidden text-center py-4">
          <div class="w-16 h-16 bg-accent/10 rounded-full flex items-center justify-center mx-auto mb-4">
            <svg class="w-8 h-8 text-accent" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"
                 stroke="currentColor" stroke-width="2">
              <path stroke-linecap="round" stroke-linejoin="round" d="M5 13l4 4L19 7" />
            </svg>
          </div>
          <h3 class="text-xl font-display font-bold text-text mb-2">¡Registro exitoso!</h3>
          <p class="text-text-muted mb-4">
            Te estamos redirigiendo a WhatsApp para completar tu registro...
          </p>
        </div>

        <!-- Error state (hidden by default) -->
        <div id="lead-error" class="hidden text-center py-4">
          <div class="w-16 h-16 bg-red-100 rounded-full flex items-center justify-center mx-auto mb-4">
            <svg class="w-8 h-8 text-red-500" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"
                 stroke="currentColor" stroke-width="2">
              <path stroke-linecap="round" stroke-linejoin="round" d="M6 18L18 6M6 6l12 12" />
            </svg>
          </div>
          <h3 class="text-xl font-display font-bold text-text mb-2">Error al enviar</h3>
          <p id="error-message" class="text-text-muted mb-4"></p>
          <button
            id="retry-button"
            class="text-primary underline hover:text-primary-dark"
          >
            Intentar de nuevo
          </button>
        </div>
      </div>
    </div>
  </div>
  ```
  - **CA:** Modal con backdrop, close button, formulario completo, success/error states.
  - **CA:** Honeypot field para anti-spam (hidden, aria-hidden, tabindex -1).
  - **CA:** Focus ring visible en todos los inputs y buttons.

---

## HU-CS03.2 — Client-Side Form Logic

**Como** visitante que completa el formulario,
**quiero** validación en tiempo real, feedback claro de errores, y un envío sin fricción,
**para que** pueda completar el registro rápidamente y sin frustración.

### Principios de Validación

- **Progressive disclosure:** Errores solo después de blur en campo individual
- **Clear messages:** Mensajes de error específicos en español
- **Real-time feedback:** Corrección inmediata al resolver error
- **Submit validation:** Validación completa al enviar
- **Loading state:** Botón muestra spinner durante envío
- **No alert/prompt/confirm:** Solo inline messages

### Tareas y Subtareas

#### T-CS03.2.1 — Crear script de formulario

- **ST-CS03.2.1.1** — Crear `src/scripts/lead-form.ts`.

  **Estructura del script:**
  ```typescript
  /**
   * lead-form.ts — Client-side form logic for the Coming Soon lead modal.
   *
   * Responsibilities:
   * 1. Open/close modal from CTA buttons
   * 2. Client-side validation with real-time feedback
   * 3. Form submission to /api/lead
   * 4. WhatsApp redirect on success
   * 5. Focus trap within modal
   *
   * This is the ONLY JavaScript on the page. Loaded via <script> tag in LeadModal.astro.
   * Bundle target: < 3KB gzipped.
   */

  // ── Types ──────────────────────────────────────────────────────────────
  interface LeadSubmission {
    name: string;
    email: string;
    phone: string;
    message?: string;
  }

  interface LeadCreated {
    id: number;
    status: 'created' | 'duplicate';
    whatsapp_url: string;
  }

  interface FieldError {
    field: string;
    message: string;
  }

  // ── Constants ──────────────────────────────────────────────────────────
  const WHATSAPP_NUMBER = '5215512345678'; // Configurable
  const API_ENDPOINT = '/api/lead';

  // ── DOM References ─────────────────────────────────────────────────────
  const modal = document.getElementById('lead-modal')!;
  const backdrop = document.getElementById('modal-backdrop')!;
  const closeBtn = document.getElementById('modal-close')!;
  const form = document.getElementById('lead-form') as HTMLFormElement;
  const submitBtn = document.getElementById('lead-submit') as HTMLButtonElement;
  const submitText = document.getElementById('submit-text')!;
  const submitLoading = document.getElementById('submit-loading')!;
  const successView = document.getElementById('lead-success')!;
  const errorView = document.getElementById('lead-error')!;
  const errorMessage = document.getElementById('error-message')!;
  const retryBtn = document.getElementById('retry-button')!;

  // ── Validation ─────────────────────────────────────────────────────────
  const validators = {
    name: (value: string): string | null => {
      if (!value.trim()) return 'El nombre es obligatorio';
      if (value.trim().length < 2) return 'El nombre debe tener al menos 2 caracteres';
      if (!/^[a-zA-ZáéíóúÁÉÍÓÚñÑüÜ\s]+$/.test(value.trim())) return 'El nombre solo puede contener letras y espacios';
      return null;
    },
    email: (value: string): string | null => {
      if (!value.trim()) return 'El email es obligatorio';
      if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value.trim())) return 'Ingresa un email válido';
      return null;
    },
    phone: (value: string): string | null => {
      if (!value.trim()) return 'El teléfono es obligatorio';
      const digits = value.replace(/[\s\-\(\)\+]/g, '');
      if (digits.length < 8) return 'El teléfono debe tener al menos 8 dígitos';
      if (digits.length > 15) return 'El teléfono no puede tener más de 15 dígitos';
      if (!/^\d+$/.test(digits)) return 'El teléfono solo puede contener números';
      return null;
    },
  } as const;

  function showFieldError(fieldName: string, message: string): void {
    const errorEl = document.getElementById(`${fieldName}-error`);
    const inputEl = document.getElementById(`lead-${fieldName}`) as HTMLInputElement;
    if (errorEl) {
      errorEl.textContent = message;
      errorEl.classList.remove('hidden');
    }
    if (inputEl) {
      inputEl.classList.add('border-red-500');
      inputEl.classList.remove('border-border');
    }
  }

  function clearFieldError(fieldName: string): void {
    const errorEl = document.getElementById(`${fieldName}-error`);
    const inputEl = document.getElementById(`lead-${fieldName}`) as HTMLInputElement;
    if (errorEl) {
      errorEl.textContent = '';
      errorEl.classList.add('hidden');
    }
    if (inputEl) {
      inputEl.classList.remove('border-red-500');
      inputEl.classList.add('border-border');
    }
  }

  function validateField(fieldName: keyof typeof validators): boolean {
    const input = form.elements.namedItem(fieldName) as HTMLInputElement;
    const error = validators[fieldName](input.value);
    if (error) {
      showFieldError(fieldName, error);
      return false;
    }
    clearFieldError(fieldName);
    return true;
  }

  function validateAll(): boolean {
    const results = [
      validateField('name'),
      validateField('email'),
      validateField('phone'),
    ];
    return results.every(Boolean);
  }

  // ── Modal Control ──────────────────────────────────────────────────────
  function openModal(): void {
    modal.classList.remove('hidden');
    document.body.style.overflow = 'hidden';
    // Focus first input
    const firstInput = form.querySelector('input:not([tabindex="-1"])') as HTMLInputElement;
    if (firstInput) firstInput.focus();
  }

  function closeModal(): void {
    modal.classList.add('hidden');
    document.body.style.overflow = '';
    // Reset form state
    resetForm();
  }

  function resetForm(): void {
    form.reset();
    form.classList.remove('hidden');
    successView.classList.add('hidden');
    errorView.classList.add('hidden');
    ['name', 'email', 'phone'].forEach(clearFieldError);
    submitBtn.disabled = false;
    submitText.classList.remove('hidden');
    submitLoading.classList.add('hidden');
  }

  // ── Form Submission ────────────────────────────────────────────────────
  async function submitForm(): Promise<void> {
    // Check honeypot
    const honeypot = form.elements.namedItem('website') as HTMLInputElement;
    if (honeypot.value) return; // Bot detected, silently ignore

    if (!validateAll()) return;

    // Loading state
    submitBtn.disabled = true;
    submitText.classList.add('hidden');
    submitLoading.classList.remove('hidden');

    const data: LeadSubmission = {
      name: (form.elements.namedItem('name') as HTMLInputElement).value.trim(),
      email: (form.elements.namedItem('email') as HTMLInputElement).value.trim(),
      phone: (form.elements.namedItem('phone') as HTMLInputElement).value.trim(),
      message: (form.elements.namedItem('message') as HTMLTextAreaElement).value.trim() || undefined,
    };

    try {
      const response = await fetch(API_ENDPOINT, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data),
      });

      if (!response.ok) {
        const errorData = await response.json();
        if (response.status === 400 && errorData.details) {
          // Show field-specific errors from server
          for (const detail of errorData.details as FieldError[]) {
            showFieldError(detail.field, detail.message);
          }
          submitBtn.disabled = false;
          submitText.classList.remove('hidden');
          submitLoading.classList.add('hidden');
          return;
        }
        throw new Error(errorData.message || 'Error del servidor');
      }

      const result: LeadCreated = await response.json();

      // Show success
      form.classList.add('hidden');
      successView.classList.remove('hidden');

      // Redirect to WhatsApp after brief delay
      setTimeout(() => {
        window.location.href = result.whatsapp_url;
      }, 1500);

    } catch (error) {
      // Show error state
      form.classList.add('hidden');
      errorView.classList.remove('hidden');
      errorMessage.textContent = error instanceof Error ? error.message : 'Error desconocido. Intenta de nuevo.';
    }
  }

  // ── Event Listeners ────────────────────────────────────────────────────

  // Open modal from any CTA button
  document.querySelectorAll('#hero-cta, .features-cta').forEach((btn) => {
    btn.addEventListener('click', (e) => {
      e.preventDefault();
      openModal();
    });
  });

  // Close modal
  closeBtn.addEventListener('click', closeModal);
  backdrop.addEventListener('click', closeModal);
  document.addEventListener('keydown', (e) => {
    if (e.key === 'Escape' && !modal.classList.contains('hidden')) {
      closeModal();
    }
  });

  // Real-time validation on blur
  ['name', 'email', 'phone'].forEach((fieldName) => {
    const input = form.elements.namedItem(fieldName) as HTMLInputElement;
    input.addEventListener('blur', () => validateField(fieldName as keyof typeof validators));
    input.addEventListener('input', () => clearFieldError(fieldName as keyof typeof validators));
  });

  // Form submit
  form.addEventListener('submit', (e) => {
    e.preventDefault();
    submitForm();
  });

  // Retry button
  retryBtn.addEventListener('click', () => {
    errorView.classList.add('hidden');
    form.classList.remove('hidden');
    submitBtn.disabled = false;
    submitText.classList.remove('hidden');
    submitLoading.classList.add('hidden');
  });
  ```
  - **CA:** Script completo con validación, submit, modal control, y focus management.
  - **CA:** Honeypot anti-spam implementado.
  - **CA:** Loading state con spinner durante envío.
  - **CA:** Success state con redirección a WhatsApp.
  - **CA:** Error state con retry.

- **ST-CS03.2.1.2** — Integrar script en LeadModal.astro.

  Agregar al final de `LeadModal.astro`:
  ```astro
  <script>
    import '../scripts/lead-form';
  </script>
  ```
  - **CA:** Script se carga solo cuando el modal está presente en la página.

#### T-CS03.2.2 — Focus trap implementation

- **ST-CS03.2.2.1** — Implementar focus trap dentro del modal.

  Agregar al script `lead-form.ts`:
  ```typescript
  // Focus trap
  function handleTabKey(e: KeyboardEvent): void {
    if (e.key !== 'Tab') return;
    if (modal.classList.contains('hidden')) return;

    const focusable = modal.querySelectorAll<HTMLElement>(
      'button, input, textarea, [tabindex]:not([tabindex="-1"])'
    );
    const first = focusable[0];
    const last = focusable[focusable.length - 1];

    if (e.shiftKey) {
      if (document.activeElement === first) {
        e.preventDefault();
        last.focus();
      }
    } else {
      if (document.activeElement === last) {
        e.preventDefault();
        first.focus();
      }
    }
  }

  document.addEventListener('keydown', handleTabKey);
  ```
  - **CA:** Tab/Shift+Tab cicla dentro del modal sin escapar al fondo.

### Criterios de Aceptación — HU-CS03.2

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Validación en tiempo real | Error aparece al blur, desaparece al corregir |
| 2 | Mensajes de error en español | "El nombre es obligatorio", "Ingresa un email válido" |
| 3 | Loading state visible | Spinner aparece durante envío |
| 4 | Success state funcional | Checkmark + mensaje de redirección |
| 5 | Error state con retry | Mensaje de error + botón reintentar |
| 6 | Focus trap funciona | Tab no sale del modal |
| 7 | Escape cierra modal | Tecla Escape cierra el modal |
| 8 | Click fuera cierra modal | Click en backdrop cierra |
| 9 | Honeypot funciona | Bot con campo website lleno no envía |
| 10 | Body scroll lock | Body no scrollea con modal abierto |

### Definition of Done — HU-CS03.2

- [ ] Script lead-form.ts implementado con validación, submit, modal control
- [ ] Focus trap implementado y verificado
- [ ] Anti-spam honeypot implementado
- [ ] Loading, success y error states funcionales
- [ ] Validación en tiempo real con mensajes en español
- [ ] Script bundle < 3KB gzipped

---

## Criterios de Aceptación Global — EP-CS03

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-EP03-1 | CTA buttons abren el modal | Click en hero CTA o features CTA → modal visible |
| CA-EP03-2 | Modal muestra formulario completo | 4 campos + submit button + privacy link |
| CA-EP03-3 | Validación client-side funciona | Errores mostrados para campos inválidos |
| CA-EP03-4 | Formulario envía datos al API | POST /api/lead con JSON payload |
| CA-EP03-5 | Success state redirige a WhatsApp | WhatsApp se abre con mensaje pre-llenado |
| CA-EP03-6 | Error state permite reintentar | Botón retry → formulario visible de nuevo |
| CA-EP03-7 | Accesibilidad completa | role=dialog, aria-modal, focus trap, escape key |
| CA-EP03-8 | Total JS < 5KB gzipped | Build output verificado |

### Definition of Done — EP-CS03

- [ ] Componente LeadModal.astro con formulario completo
- [ ] Script lead-form.ts con validación, submit, modal control
- [ ] Focus trap implementado
- [ ] Anti-spam honeypot implementado
- [ ] Loading, success, error states funcionales
- [ ] Accesibilidad WCAG 2.1 AA (role=dialog, aria-modal, focus trap)
- [ ] Total JS bundle < 5KB gzipped

---

## Resumen de Entregables EP-CS03

**Componentes:**
- `LeadModal.astro` — Modal container con formulario HTML
- `lead-form.ts` — Client-side logic (validación, submit, modal control)

**UX States:**
- Default: Formulario visible con 4 campos
- Loading: Spinner en botón submit
- Success: Checkmark + mensaje + redirección WhatsApp
- Error: Mensaje de error + botón retry

**Anti-spam:**
- Honeypot field (hidden website input)
- Server-side rate limiting (ver EP-CS04)

**Accesibilidad:**
- `role="dialog"`, `aria-modal="true"`, `aria-labelledby`
- Focus trap con Tab/Shift+Tab
- Escape key cierra modal
- Error messages con `role="alert"`
