# EP-CS02 — Landing Page UI

**Épica:** Como visitante del sitio, quiero ver una landing page profesional y calmada que comunique la propuesta de valor de Serenidad con claridad, para que pueda entender qué ofrece la clínica digital y decidir si quiero dejar mis datos.

**Prioridad:** Must Have — Experiencia de usuario
**Sprint:** Sprint 1
**Dependencias Entrantes:** EP-CS01 (scaffold + design system)
**Dependencias Salientes:** EP-CS06 (deployment)

---

## HU-CS02.1 — Hero Section

**Como** visitante que llega a sereni.dad,
**quiero** ver inmediatamente un hero impactante con la propuesta de valor y un CTA claro,
**para que** en menos de 5 segundos entienda qué es Serenidad y qué acción tomar.

### Principios de Diseño

- **Above the fold:** Todo el hero visible sin scroll en viewport estándar (1440×900)
- **Jerarquía visual:** Headline → Subheadline → CTA → Trust signal
- **Emoción:** Calma, confianza, esperanza — NO urgencia ni miedo
- **Performance:** Zero JS en hero, solo HTML/CSS

### Tareas y Subtareas

#### T-CS02.1.1 — Crear componente Hero

- **ST-CS02.1.1.1** — Crear `src/components/Hero.astro`.

  **Estructura del Hero:**
  ```
  ┌──────────────────────────────────────────────────────┐
  │  [Navigation Bar]                                     │
  │   Logo    Inicio   Servicios   Contacto              │
  │                                                        │
  │  ┌─────────────────────────────────────────────────┐  │
  │  │                                                  │  │
  │  │   Tu bienestar mental,                          │  │
  │  │   nuestra prioridad.                            │  │
  │  │                                                  │  │
  │  │   Terapia online profesional con                │  │
  │  │   psicólogos certificados. Accesible,           │  │
  │  │   confidencial y centrada en ti.                │  │
  │  │                                                  │  │
  │  │   [💬 Agendar consulta]  [Conocer más ↓]       │  │
  │  │                                                  │  │
  │  │   ✓ 100% confidencial  ✓ Profesionales         │  │
  │  │     certificados        certificados            │  │
  │  │                                                  │  │
  │  └─────────────────────────────────────────────────┘  │
  │                                                        │
  │  Próximamente — Déjanos ayudarte                      │
  └──────────────────────────────────────────────────────┘
  ```

  **Código del componente:**
  ```astro
  ---
  import { Phone, Shield, Users } from 'lucide-astro';
  ---

  <section class="relative min-h-screen flex items-center bg-gradient-to-br from-primary/5 via-bg to-secondary/5">
    <!-- Decorative elements -->
    <div class="absolute inset-0 overflow-hidden" aria-hidden="true">
      <div class="absolute -top-40 -right-40 w-80 h-80 bg-primary/10 rounded-full blur-3xl"></div>
      <div class="absolute -bottom-40 -left-40 w-80 h-80 bg-secondary/10 rounded-full blur-3xl"></div>
    </div>

    <div class="relative container mx-auto px-6 py-20 lg:py-32">
      <div class="max-w-3xl mx-auto text-center lg:text-left">
        <!-- Badge -->
        <div class="inline-flex items-center gap-2 bg-accent/10 text-accent-dark px-4 py-2 rounded-full text-sm font-medium mb-8">
          <span class="w-2 h-2 bg-accent rounded-full animate-pulse"></span>
          Próximamente — Regístrate ahora
        </div>

        <!-- Headline -->
        <h1 class="text-4xl sm:text-5xl lg:text-6xl font-display font-bold text-text leading-tight mb-6">
          Tu bienestar mental,
          <span class="text-primary">nuestra prioridad</span>
        </h1>

        <!-- Subheadline -->
        <p class="text-lg sm:text-xl text-text-muted max-w-2xl mb-10 leading-relaxed">
          Terapia online profesional con psicólogos certificados.
          Accesible, confidencial y centrada en ti.
          <strong class="text-text">Regístrate y sé el primero en acceder.</strong>
        </p>

        <!-- CTAs -->
        <div class="flex flex-col sm:flex-row gap-4 justify-center lg:justify-start mb-12">
          <button
            id="hero-cta"
            class="inline-flex items-center justify-center gap-2 bg-whatsapp text-white px-8 py-4 rounded-xl font-semibold text-lg hover:bg-whatsapp-dark transition-shadow-lg focus:ring-2 focus:ring-whatsapp focus:ring-offset-2"
            aria-label="Agendar consulta y abrir formulario de contacto"
          >
            💬 Agendar consulta
          </button>
          <a
            href="#features"
            class="inline-flex items-center justify-center gap-2 border-2 border-primary text-primary px-8 py-4 rounded-xl font-semibold hover:bg-primary/5 transition-base focus:ring-2 focus:ring-primary focus:ring-offset-2"
          >
            Conocer más ↓
          </a>
        </div>

        <!-- Trust signals -->
        <div class="flex flex-wrap gap-6 justify-center lg:justify-start text-sm text-text-muted">
          <div class="flex items-center gap-2">
            <Shield class="w-4 h-4 text-accent" />
            <span>100% confidencial</span>
          </div>
          <div class="flex items-center gap-2">
            <Users class="w-4 h-4 text-accent" />
            <span>Profesionales certificados</span>
          </div>
          <div class="flex items-center gap-2">
            <Phone class="w-4 h-4 text-accent" />
            <span>Consulta por WhatsApp</span>
          </div>
        </div>
      </div>
    </div>
  </section>
  ```
  - **CA:** Hero section renderiza con headline, subheadline, CTAs y trust signals.
  - **CA:** CTA principal tiene color WhatsApp (#25D366) como indicador visual.
  - **CA:** Decorative elements NO afectan layout (position absolute, aria-hidden).

#### T-CS02.1.2 — Responsive behavior del Hero

- **ST-CS02.1.2.1** — Verificar responsive breakpoints.
  - **Mobile (<640px):** Stack vertical, texto centrado, CTAs full-width
  - **Tablet (640-1024px):** Stack vertical, texto centrado, CTAs inline
  - **Desktop (>1024px):** Texto alineado izquierda, CTAs inline, más espaciado
  - **CA:** Hero se adapta correctamente en los 3 breakpoints.

---

## HU-CS02.2 — Features Section

**Como** visitante que scrollea hacia abajo,
**quiero** ver 3 pilares de valor de Serenidad presentados de forma clara y visual,
**para que** entienda los beneficios clave de la plataforma.

### Tareas y Subtareas

#### T-CS02.2.1 — Crear componente Features

- **ST-CS02.2.1.1** — Crear `src/components/Features.astro`.

  **Layout de las Features:**
  ```
  ┌──────────────────────────────────────────────────────┐
  │              ¿Por qué elegir Serenidad?              │
  │                                                      │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
  │  │  🧠      │  │  🔒      │  │  🌍      │          │
  │  │ Centrado │  │ Privacid │  │ Accesibl │          │
  │  │ en ti    │  │ absoluta │  │ global   │          │
  │  │          │  │          │  │          │          │
  │  │ Terapia  │  │ Cifrado  │  │ Desde    │          │
  │  │ personal │  │ de grado │  │ cualquier│          │
  │  │ izada... │  │ médico...│  │ lugar... │          │
  │  └──────────┘  └──────────┘  └──────────┘          │
  │                                                      │
  │         [💬 Quiero agendar consulta]                 │
  └──────────────────────────────────────────────────────┘
  ```

  **Código del componente:**
  ```astro
  ---
  import { Brain, Lock, Globe } from 'lucide-astro';
  ---

  <section id="features" class="py-20 lg:py-28 bg-bg-alt">
    <div class="container mx-auto px-6">
      <!-- Section header -->
      <div class="text-center mb-16">
        <h2 class="text-3xl sm:text-4xl font-display font-bold text-text mb-4">
          ¿Por qué elegir Serenidad?
        </h2>
        <p class="text-lg text-text-muted max-w-2xl mx-auto">
          Construimos una clínica digital que prioriza tu bienestar,
          tu privacidad y tu accesibilidad.
        </p>
      </div>

      <!-- Feature cards -->
      <div class="grid md:grid-cols-3 gap-8 max-w-5xl mx-auto">
        <!-- Card 1: Centrado en ti -->
        <div class="bg-white p-8 rounded-2xl shadow-sm hover:shadow-md transition-base border border-border">
          <div class="w-14 h-14 bg-primary/10 rounded-xl flex items-center justify-center mb-6">
            <Brain class="w-7 h-7 text-primary" />
          </div>
          <h3 class="text-xl font-display font-semibold text-text mb-3">
            Centrado en ti
          </h3>
          <p class="text-text-muted leading-relaxed">
            Terapia personalizada con psicólogos especializados en tus necesidades.
            Cada sesión se adapta a tu ritmo y tus objetivos.
          </p>
        </div>

        <!-- Card 2: Privacidad absoluta -->
        <div class="bg-white p-8 rounded-2xl shadow-sm hover:shadow-md transition-base border border-border">
          <div class="w-14 h-14 bg-secondary/10 rounded-xl flex items-center justify-center mb-6">
            <Lock class="w-7 h-7 text-secondary" />
          </div>
          <h3 class="text-xl font-display font-semibold text-text mb-3">
            Privacidad absoluta
          </h3>
          <p class="text-text-muted leading-relaxed">
            Cifrado de grado médico y cumplimiento HIPAA.
            Tus datos están protegidos con los más altos estándares de seguridad.
          </p>
        </div>

        <!-- Card 3: Accesible globalmente -->
        <div class="bg-white p-8 rounded-2xl shadow-sm hover:shadow-md transition-base border border-border">
          <div class="w-14 h-14 bg-accent/10 rounded-xl flex items-center justify-center mb-6">
            <Globe class="w-7 h-7 text-accent" />
          </div>
          <h3 class="text-xl font-display font-semibold text-text mb-3">
            Accesible globalmente
          </h3>
          <p class="text-text-muted leading-relaxed">
            Desde cualquier lugar del mundo. Terapia online que se adapta
            a tu zona horaria y tu idioma.
          </p>
        </div>
      </div>

      <!-- Secondary CTA -->
      <div class="text-center mt-12">
        <button
          class="features-cta inline-flex items-center justify-center gap-2 bg-whatsapp text-white px-8 py-4 rounded-xl font-semibold hover:bg-whatsapp-dark transition-base shadow-lg focus:ring-2 focus:ring-whatsapp focus:ring-offset-2"
          aria-label="Agendar consulta y abrir formulario"
        >
          💬 Quiero agendar consulta
        </button>
      </div>
    </div>
  </section>
  ```
  - **CA:** 3 feature cards renderizan con iconos Lucide.
  - **CA:** Secondary CTA replica el comportamiento del hero CTA.
  - **CA:** Cards tienen hover effect suave (shadow-md).

---

## HU-CS02.3 — Footer, Disclaimer & Page Assembly

**Como** visitante y como compliance officer,
**quiero** ver un footer con información legal, un disclaimer de no-PHI, y la página completa ensamblada correctamente,
**para que** el sitio cumpla con regulaciones healthcare y ofrezca navegación completa.

### Tareas y Subtareas

#### T-CS02.3.1 — Crear componente Disclaimer

- **ST-CS02.3.1.1** — Crear `src/components/Disclaimer.astro`.
  ```astro
  ---
  ---

  <section class="py-6 bg-primary/5 border-t border-primary/10">
    <div class="container mx-auto px-6 text-center text-sm text-text-muted max-w-3xl">
      <p>
        <strong class="text-text">Aviso importante:</strong>
        Este sitio web público NO maneja información médica protegida (PHI).
        Los datos recopilados en el formulario son únicamente de contacto
        y no constituyen una relación médico-paciente.
        Para acceder a servicios clínicos, utiliza nuestro
        <a href="https://app.sereni.dad" class="text-primary underline hover:text-primary-dark">
          portal seguro
        </a>.
      </p>
      <p class="mt-2 text-xs text-text-light">
        Si tienes una emergencia de salud mental, llama al 911 o a la línea nacional de prevención del suicidio: 800-911-2000.
      </p>
    </div>
  </section>
  ```
  - **CA:** Disclaimer visible con texto de no-PHI y emergencias.

#### T-CS02.3.2 — Crear componente Footer

- **ST-CS02.3.2.1** — Crear `src/components/Footer.astro`.
  ```astro
  ---
  const currentYear = new Date().getFullYear();
  ---

  <footer class="py-12 bg-bg-dark text-white">
    <div class="container mx-auto px-6">
      <div class="grid md:grid-cols-3 gap-8 mb-8">
        <!-- Brand -->
        <div>
          <h3 class="text-xl font-display font-bold mb-4">Serenidad</h3>
          <p class="text-gray-400 text-sm leading-relaxed">
            Clínica digital de salud mental.
            Terapia online accesible, profesional y centrada en tu bienestar.
          </p>
        </div>

        <!-- Links -->
        <div>
          <h4 class="font-semibold mb-4">Legal</h4>
          <ul class="space-y-2 text-sm text-gray-400">
            <li><a href="/privacidad" class="hover:text-white transition-fast">Política de privacidad</a></li>
            <li><a href="/terminos" class="hover:text-white transition-fast">Términos de uso</a></li>
          </ul>
        </div>

        <!-- Contact -->
        <div>
          <h4 class="font-semibold mb-4">Contacto</h4>
          <ul class="space-y-2 text-sm text-gray-400">
            <li>
              <a
                href="https://wa.me/5215512345678?text=Hola%2C%20me%20interesa%20saber%20más%20sobre%20Serenidad"
                class="hover:text-white transition-fast"
                target="_blank"
                rel="noopener noreferrer"
              >
                💬 WhatsApp
              </a>
            </li>
            <li>
              <a href="mailto:contacto@sereni.dad" class="hover:text-white transition-fast">
                ✉️ contacto@sereni.dad
              </a>
            </li>
          </ul>
        </div>
      </div>

      <!-- Copyright -->
      <div class="border-t border-gray-700 pt-8 text-center text-sm text-gray-500">
        <p>&copy; {currentYear} Serenidad. Todos los derechos reservados.</p>
      </div>
    </div>
  </footer>
  ```
  - **CA:** Footer con 3 columnas: brand, legal, contacto.
  - **CA:** Link de WhatsApp en footer (número placeholder, configurable).

#### T-CS02.3.3 — Ensamblar página principal

- **ST-CS02.3.3.1** — Crear `src/pages/index.astro`.
  ```astro
  ---
  import BaseLayout from '../layouts/BaseLayout.astro';
  import Hero from '../components/Hero.astro';
  import Features from '../components/Features.astro';
  import Disclaimer from '../components/Disclaimer.astro';
  import Footer from '../components/Footer.astro';
  import LeadModal from '../components/LeadModal.astro';
  ---

  <BaseLayout
    title="Serenidad — Salud Mental Accesible y Profesional | Próximamente"
    description="Clínica digital de salud mental. Terapia online con psicólogos certificados. Accesible, confidencial y centrada en tu bienestar. Regístrate y sé el primero en acceder."
  >
    <Hero />
    <Features />
    <Disclaimer />
    <Footer />
    <LeadModal />

    <!-- Structured Data -->
    <script type="application/ld+json" set:html={JSON.stringify({
      "@context": "https://schema.org",
      "@type": "MedicalOrganization",
      "name": "Serenidad",
      "url": "https://sereni.dad",
      "description": "Clínica digital de salud mental con terapia online profesional",
      "medicalSpecialty": "Psychiatry",
      "contactPoint": {
        "@type": "ContactPoint",
        "contactType": "Customer Service",
        "availableLanguage": ["es", "en"]
      }
    })} />
  </BaseLayout>
  ```
  - **CA:** Página principal ensamblada con todos los componentes.
  - **CA:** Structured data (Schema.org MedicalOrganization) presente.

#### T-CS02.3.4 — Crear página de privacidad

- **ST-CS02.3.4.1** — Crear `src/pages/privacidad.astro`.
  ```astro
  ---
  import BaseLayout from '../layouts/BaseLayout.astro';
  import Footer from '../components/Footer.astro';
  ---

  <BaseLayout
    title="Política de Privacidad — Serenidad"
    description="Política de privacidad de Serenidad, clínica digital de salud mental."
  >
    <section class="py-20">
      <div class="container mx-auto px-6 max-w-3xl">
        <h1 class="text-4xl font-display font-bold mb-8">Política de Privacidad</h1>

        <div class="prose prose-lg max-w-none text-text-muted">
          <h2>1. Datos que recopilamos</h2>
          <p>
            En esta página de registro ("Coming Soon"), recopilamos únicamente
            los siguientes datos de contacto:
          </p>
          <ul>
            <li>Nombre completo</li>
            <li>Dirección de correo electrónico</li>
            <li>Número de teléfono</li>
            <li>Mensaje opcional</li>
          </ul>

          <h2>2. Finalidad del tratamiento</h2>
          <p>
            Los datos recopilados se utilizan exclusivamente para:
          </p>
          <ul>
            <li>Contactarte para informarte sobre el lanzamiento de Serenidad</li>
            <li>Iniciar una conversación por WhatsApp si así lo solicitas</li>
            <li>Enviar comunicaciones relacionadas con el servicio</li>
          </ul>

          <h2>3. No manejamos PHI</h2>
          <p>
            Este sitio web NO recopila, almacena ni procesa información médica
            protegida (PHI, Protected Health Information). Los datos recopilados
            son exclusivamente de contacto comercial.
          </p>

          <h2>4. Almacenamiento y seguridad</h2>
          <p>
            Los datos se almacenan en infraestructura de Cloudflare (D1) con
            cifrado en tránsito (HTTPS/TLS) y en reposo. Solo el equipo autorizado
            de Serenidad tiene acceso a los datos.
          </p>

          <h2>5. Tus derechos</h2>
          <p>
            Puedes solicitar en cualquier momento:
          </p>
          <ul>
            <li>Acceso a tus datos personales</li>
            <li>Rectificación de datos incorrectos</li>
            <li>Eliminación de tus datos</li>
            <li>Revocación del consentimiento</li>
          </ul>
          <p>
            Para ejercer estos derechos, contacta a:
            <a href="mailto:privacidad@sereni.dad">privacidad@sereni.dad</a>
          </p>

          <h2>6. Contacto</h2>
          <p>
            Para cualquier duda sobre esta política, contacta a:
            <a href="mailto:privacidad@sereni.dad">privacidad@sereni.dad</a>
          </p>

          <p class="text-sm text-text-light mt-8">
            Última actualización: Abril 2026
          </p>
        </div>
      </div>
    </section>
    <Footer />
  </BaseLayout>
  ```
  - **CA:** Página de privacidad con contenido completo.

### Criterios de Aceptación — HU-CS02.3

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Disclaimer visible en homepage | Texto "NO maneja PHI" presente |
| 2 | Footer con links legales | Política de privacidad accesible |
| 3 | Página privacidad renderiza | `/privacidad` retorna HTML |
| 4 | Structured data válido | Google Rich Results Test pasa |
| 5 | Número de emergencia visible | 911 / línea nacional visible |

### Definition of Done — HU-CS02.3

- [ ] Disclaimer componente creado e integrado
- [ ] Footer con brand, legal, contacto
- [ ] Página principal ensamblada con todos los componentes
- [ ] Página de privacidad creada
- [ ] Structured data (Schema.org) implementado
- [ ] Número de emergencia visible

---

## Criterios de Aceptación Global — EP-CS02

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-EP02-1 | Hero visible above the fold sin scroll | Viewport 1440×900 muestra headline + CTA |
| CA-EP02-2 | 3 feature cards renderizan correctamente | Iconos + texto visibles en desktop y mobile |
| CA-EP02-3 | CTAs abren el modal de formulario | Click en CTA → modal aparece |
| CA-EP02-4 | Responsive en 3 breakpoints | Mobile, tablet, desktop verificados |
| CA-EP02-5 | Disclaimer visible | Sección de no-PHI presente |
| CA-EP02-6 | Footer completo | 3 columnas + copyright |
| CA-EP02-7 | SEO meta tags presentes | og:title, og:description, canonical |
| CA-EP02-8 | Structured data válido | Schema.org MedicalOrganization |
| CA-EP02-9 | Accesibilidad WCAG 2.1 AA | Lighthouse Accessibility ≥ 95 |
| CA-EP02-10 | Performance Lighthouse ≥ 95 | Lighthouse Performance ≥ 95 |

### Definition of Done — EP-CS02

- [ ] Hero section con headline, subheadline, CTAs y trust signals
- [ ] Features section con 3 cards y secondary CTA
- [ ] Disclaimer de no-PHI visible
- [ ] Footer con brand, legal, contacto y copyright
- [ ] Página principal ensamblada
- [ ] Página de privacidad creada
- [ ] Responsive en mobile, tablet, desktop
- [ ] Structured data (Schema.org) implementado
- [ ] Lighthouse Performance ≥ 95, Accessibility ≥ 95

---

## Resumen de Entregables EP-CS02

**Componentes Astro:**
- `Hero.astro` — Hero section con CTA principal
- `Features.astro` — 3 feature cards con secondary CTA
- `Disclaimer.astro` — Aviso de no-PHI y emergencias
- `Footer.astro` — Footer con 3 columnas

**Páginas:**
- `index.astro` — Landing page completa
- `privacidad.astro` — Política de privacidad

**SEO:**
- Structured data (Schema.org MedicalOrganization)
- Meta tags completos (OG, Twitter, canonical)
- robots.txt y sitemap

**Validación:**
```bash
# Build
npm run build

# Preview
npm run preview

# Lighthouse
npx lighthouse http://localhost:4321 --view
# Targets: Performance ≥ 95, Accessibility ≥ 95, SEO ≥ 95
```
