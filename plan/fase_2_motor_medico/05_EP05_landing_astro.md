# EP-05 — Landing Page (Astro v6 + Cloudflare Pages)

**Épica:** Como equipo de marketing, necesito una landing page pública profesional con SSG, Core Web Vitals optimizados, SEO healthcare-compliant, y deployment automatizado en Cloudflare Pages, para atraer pacientes potenciales y educarles sobre servicios de salud mental.

**Origen:** Fase 2 (2.E) del roadmap arquitectónico + research/08_evaluacion_tecnologica_fase2_motor_medico.md  
**Prioridad:** Alta (no bloqueante para servicios backend)  
**Sprint:** S6  
**Dependencias Entrantes:** Ninguna (independiente de backend)  
**Dependencias Salientes:** Ninguna

---

## HU-05.1 — Astro v6 SSG Project Scaffold

**Como** desarrollador frontend,  
**quiero** crear un proyecto Astro v6 con configuración SSG, SEO optimizado, y Core Web Vitals targets,  
**para que** la landing page sea rápida, indexable, y cumpla con estándares de accesibilidad healthcare.

### Contexto técnico

**Stack**:
- Astro v6 (SSG mode)
- TailwindCSS v4 (styling)
- Lucide Icons (iconografía)
- View Transitions API (navegación smooth)

**Targets de performance**:
- LCP < 2.0s
- CLS < 0.1
- INP < 200ms
- TTFB < 600ms

### Tareas y Subtareas

#### T-05.1.1 — Crear proyecto Astro

- **ST-05.1.1.1** — Inicializar proyecto Astro.
  ```bash
  cd frontend
  npm create astro@latest landing -- --template minimal --typescript strict
  cd landing
  ```
  - **CA:** Proyecto Astro creado.

- **ST-05.1.1.2** — Instalar dependencias.
  ```bash
  npm install
  npm install -D tailwindcss@next @tailwindcss/typography
  npm install -D @astrojs/sitemap
  npm install lucide-astro
  ```
  - **CA:** Dependencias instaladas.

#### T-05.1.2 — Configurar Astro para SSG

- **ST-05.1.2.1** — Crear `astro.config.mjs`.
  ```js
  import { defineConfig } from 'astro/config';
  import sitemap from '@astrojs/sitemap';

  export default defineConfig({
    site: 'https://sereni.dad',
    output: 'static',
    integrations: [sitemap()],
    vite: {
      build: {
        cssMinify: 'lightningcss',
      },
    },
    image: {
      service: {
        entrypoint: 'astro/assets/services/sharp',
      },
    },
    compressHTML: true,
    build: {
      inlineStylesheets: 'auto',
    },
  });
  ```
  - **CA:** Config Astro SSG configurado.

#### T-05.1.3 — Configurar TailwindCSS

- **ST-05.1.3.1** — Crear `tailwind.config.mjs`.
  ```js
  /** @type {import('tailwindcss').Config} */
  export default {
    content: ['./src/**/*.{astro,html,js,jsx,md,mdx,svelte,ts,tsx,vue}'],
    theme: {
      extend: {
        colors: {
          primary: '#3B82F6',
          secondary: '#8B5CF6',
          accent: '#10B981',
        },
        fontFamily: {
          sans: ['Inter Variable', 'sans-serif'],
          display: ['Lexend', 'sans-serif'],
        },
      },
    },
    plugins: [
      require('@tailwindcss/typography'),
    ],
  };
  ```
  - **CA:** TailwindCSS configurado.

- **ST-05.1.3.2** — Crear `src/styles/global.css`.
  ```css
  @tailwind base;
  @tailwind components;
  @tailwind utilities;

  @layer base {
    html {
      @apply scroll-smooth;
    }
    
    body {
      @apply font-sans antialiased;
    }
  }
  ```
  - **CA:** Global styles creados.

#### T-05.1.4 — Crear layout base

- **ST-05.1.4.1** — Crear `src/layouts/BaseLayout.astro`.
  ```astro
  ---
  import '../styles/global.css';

  interface Props {
    title: string;
    description: string;
    image?: string;
  }

  const { title, description, image = '/og-image.jpg' } = Astro.props;
  const canonicalURL = new URL(Astro.url.pathname, Astro.site);
  ---

  <!doctype html>
  <html lang="es">
    <head>
      <meta charset="UTF-8" />
      <meta name="viewport" content="width=device-width, initial-scale=1.0" />
      
      <!-- Primary Meta Tags -->
      <title>{title}</title>
      <meta name="title" content={title} />
      <meta name="description" content={description} />
      <link rel="canonical" href={canonicalURL} />
      
      <!-- Open Graph / Facebook -->
      <meta property="og:type" content="website" />
      <meta property="og:url" content={canonicalURL} />
      <meta property="og:title" content={title} />
      <meta property="og:description" content={description} />
      <meta property="og:image" content={new URL(image, Astro.site)} />
      
      <!-- Twitter -->
      <meta property="twitter:card" content="summary_large_image" />
      <meta property="twitter:url" content={canonicalURL} />
      <meta property="twitter:title" content={title} />
      <meta property="twitter:description" content={description} />
      <meta property="twitter:image" content={new URL(image, Astro.site)} />
      
      <!-- Favicon -->
      <link rel="icon" type="image/svg+xml" href="/favicon.svg" />
      
      <!-- Fonts (preload critical) -->
      <link rel="preconnect" href="https://fonts.googleapis.com" />
      <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
      
      <!-- Sitemap -->
      <link rel="sitemap" href="/sitemap-index.xml" />
    </head>
    <body>
      <slot />
    </body>
  </html>
  ```
  - **CA:** BaseLayout creado.

### Criterios de Aceptación de la HU-05.1

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Proyecto Astro compila | `npm run build` exitoso |
| 2 | Output es SSG (static) | `dist/` contiene solo HTML/CSS/JS estáticos |
| 3 | TailwindCSS funciona | Clases Tailwind se aplican correctamente |
| 4 | Sitemap generado | `dist/sitemap-index.xml` existe |
| 5 | Meta tags SEO presentes | View source muestra og:tags, twitter:tags |

### Definition of Done — HU-05.1

- [ ] Proyecto Astro v6 scaffolded.
- [ ] TailwindCSS configurado y funcional.
- [ ] BaseLayout con SEO meta tags completo.
- [ ] Build genera sitio estático sin errores.

---

## HU-05.2 — Páginas y Contenido Healthcare

**Como** visitante del sitio,  
**quiero** navegar por páginas informativas sobre servicios de salud mental con diseño profesional y accesible,  
**para que** pueda entender la oferta de Serenidad y decidir si es adecuado para mí.

### Contexto técnico

**Páginas principales**:
- Homepage: Hero + features + CTA
- Servicios: Lista de servicios clínicos
- Nosotros: Misión, visión, equipo
- Contacto: Formulario + información

**Healthcare-specific requirements**:
- Disclaimer: "Este sitio NO maneja PHI"
- Accesibilidad: WCAG 2.1 AA
- Privacy policy link visible
- HIPAA compliance notice

### Tareas y Subtareas

#### T-05.2.1 — Crear Homepage

- **ST-05.2.1.1** — Crear `src/pages/index.astro`.
  ```astro
  ---
  import BaseLayout from '../layouts/BaseLayout.astro';
  import { Image } from 'astro:assets';
  import heroImage from '../assets/hero-mental-health.jpg';
  import { Heart, Shield, Users } from 'lucide-astro';
  ---

  <BaseLayout
    title="Serenidad — Salud Mental Accesible y Profesional"
    description="Clínica digital de salud mental. Terapia online con psicólogos certificados. Accesible, confidencial, y centrado en tu bienestar."
  >
    <!-- Hero Section -->
    <section class="relative h-screen flex items-center">
      <Image
        src={heroImage}
        alt="Persona en sesión de terapia online"
        class="absolute inset-0 w-full h-full object-cover"
        loading="eager"
        fetchpriority="high"
        format="avif"
        fallbackFormat="webp"
      />
      <div class="absolute inset-0 bg-gradient-to-r from-primary/90 to-secondary/80"></div>
      
      <div class="relative container mx-auto px-4 text-white">
        <h1 class="text-5xl md:text-7xl font-display font-bold mb-6">
          Tu bienestar mental,<br />nuestra prioridad
        </h1>
        <p class="text-xl md:text-2xl mb-8 max-w-2xl">
          Conéctate con psicólogos certificados desde la comodidad de tu hogar. 
          Terapia online accesible, confidencial, y centrada en ti.
        </p>
        <div class="flex gap-4">
          <a href="/registro" class="bg-white text-primary px-8 py-4 rounded-lg font-semibold hover:bg-gray-100 transition">
            Comienza hoy
          </a>
          <a href="/servicios" class="border-2 border-white px-8 py-4 rounded-lg font-semibold hover:bg-white/10 transition">
            Conoce más
          </a>
        </div>
      </div>
    </section>

    <!-- Features Section -->
    <section class="py-20 bg-gray-50">
      <div class="container mx-auto px-4">
        <h2 class="text-4xl font-display font-bold text-center mb-12">
          ¿Por qué elegir Serenidad?
        </h2>
        
        <div class="grid md:grid-cols-3 gap-8">
          <div class="bg-white p-8 rounded-xl shadow-sm">
            <Heart class="w-12 h-12 text-accent mb-4" />
            <h3 class="text-2xl font-semibold mb-4">Centrado en ti</h3>
            <p class="text-gray-600">
              Terapia personalizada con psicólogos especializados en tus necesidades específicas.
            </p>
          </div>

          <div class="bg-white p-8 rounded-xl shadow-sm">
            <Shield class="w-12 h-12 text-accent mb-4" />
            <h3 class="text-2xl font-semibold mb-4">100% Confidencial</h3>
            <p class="text-gray-600">
              Tus datos están protegidos con cifrado de grado médico y cumplimiento HIPAA.
            </p>
          </div>

          <div class="bg-white p-8 rounded-xl shadow-sm">
            <Users class="w-12 h-12 text-accent mb-4" />
            <h3 class="text-2xl font-semibold mb-4">Profesionales Certificados</h3>
            <p class="text-gray-600">
              Todos nuestros terapeutas están licenciados y con experiencia comprobada.
            </p>
          </div>
        </div>
      </div>
    </section>

    <!-- HIPAA Disclaimer -->
    <section class="py-8 bg-blue-50 border-t border-blue-100">
      <div class="container mx-auto px-4 text-center text-sm text-gray-600">
        <p>
          <strong>Aviso importante:</strong> Este sitio web público NO maneja información médica protegida (PHI). 
          Para acceder a servicios clínicos, utiliza nuestro <a href="https://app.sereni.dad" class="text-primary underline">portal seguro</a>.
        </p>
      </div>
    </section>

    <!-- Structured Data (Schema.org) -->
    <script type="application/ld+json" set:html={JSON.stringify({
      "@context": "https://schema.org",
      "@type": "MedicalOrganization",
      "name": "Serenidad",
      "url": "https://sereni.dad",
      "logo": "https://sereni.dad/logo.png",
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
  - **CA:** Homepage creada con hero, features, disclaimer.

#### T-05.2.2 — Crear página de servicios

- **ST-05.2.2.1** — Crear `src/pages/servicios.astro`.
  ```astro
  ---
  import BaseLayout from '../layouts/BaseLayout.astro';

  const services = [
    {
      title: "Terapia Individual",
      description: "Sesiones personalizadas para ansiedad, depresión, estrés y más.",
      duration: "50 minutos",
      price: "Desde $45 USD"
    },
    {
      title: "Terapia de Pareja",
      description: "Fortalece tu relación con terapia profesional online.",
      duration: "60 minutos",
      price: "Desde $60 USD"
    },
    {
      title: "Terapia Familiar",
      description: "Mejora la comunicación y resuelve conflictos familiares.",
      duration: "60 minutos",
      price: "Desde $70 USD"
    },
  ];
  ---

  <BaseLayout
    title="Servicios de Salud Mental — Serenidad"
    description="Terapia individual, de pareja y familiar online. Profesionales certificados especializados en ansiedad, depresión, y bienestar emocional."
  >
    <section class="py-20">
      <div class="container mx-auto px-4">
        <h1 class="text-5xl font-display font-bold mb-6">Nuestros Servicios</h1>
        <p class="text-xl text-gray-600 mb-12 max-w-3xl">
          Ofrecemos terapia profesional adaptada a tus necesidades. 
          Todos nuestros terapeutas están licenciados y especializados.
        </p>

        <div class="grid md:grid-cols-2 lg:grid-cols-3 gap-8">
          {services.map(service => (
            <div class="border rounded-xl p-6 hover:shadow-lg transition">
              <h2 class="text-2xl font-semibold mb-4">{service.title}</h2>
              <p class="text-gray-600 mb-4">{service.description}</p>
              <div class="flex justify-between text-sm text-gray-500 mb-6">
                <span>⏱️ {service.duration}</span>
                <span class="font-semibold text-primary">{service.price}</span>
              </div>
              <a href="/registro" class="block text-center bg-primary text-white py-3 rounded-lg hover:bg-primary/90 transition">
                Agendar cita
              </a>
            </div>
          ))}
        </div>
      </div>
    </section>
  </BaseLayout>
  ```
  - **CA:** Página servicios creada.

#### T-05.2.3 — Crear formulario de contacto (edge function)

- **ST-05.2.3.1** — Crear `src/pages/contacto.astro`.
  ```astro
  ---
  import BaseLayout from '../layouts/BaseLayout.astro';
  ---

  <BaseLayout
    title="Contacto — Serenidad"
    description="¿Tienes preguntas? Contáctanos y un miembro de nuestro equipo te responderá pronto."
  >
    <section class="py-20">
      <div class="container mx-auto px-4 max-w-2xl">
        <h1 class="text-4xl font-display font-bold mb-6">Contáctanos</h1>
        
        <form method="POST" action="/api/contact" class="space-y-6">
          <div>
            <label for="name" class="block text-sm font-medium mb-2">Nombre</label>
            <input 
              type="text" 
              id="name" 
              name="name" 
              required 
              class="w-full px-4 py-3 border rounded-lg focus:ring-2 focus:ring-primary"
            />
          </div>

          <div>
            <label for="email" class="block text-sm font-medium mb-2">Email</label>
            <input 
              type="email" 
              id="email" 
              name="email" 
              required 
              class="w-full px-4 py-3 border rounded-lg focus:ring-2 focus:ring-primary"
            />
          </div>

          <div>
            <label for="message" class="block text-sm font-medium mb-2">Mensaje</label>
            <textarea 
              id="message" 
              name="message" 
              rows="5" 
              required 
              class="w-full px-4 py-3 border rounded-lg focus:ring-2 focus:ring-primary"
            ></textarea>
          </div>

          <button 
            type="submit" 
            class="w-full bg-primary text-white py-4 rounded-lg font-semibold hover:bg-primary/90 transition"
          >
            Enviar mensaje
          </button>
        </form>

        <p class="text-sm text-gray-500 mt-6">
          Este formulario NO debe usarse para emergencias médicas. Si tienes una emergencia, 
          llama al 911 o acude al hospital más cercano.
        </p>
      </div>
    </section>
  </BaseLayout>
  ```
  - **CA:** Formulario contacto creado.

### Criterios de Aceptación de la HU-05.2

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Homepage renderiza correctamente | Visual QA en navegador |
| 2 | Hero image optimizado (AVIF) | Network tab muestra formato AVIF |
| 3 | Structured data presente | Google Rich Results Test valida schema |
| 4 | HIPAA disclaimer visible | Disclaimer aparece en homepage |
| 5 | Página servicios lista correctamente | 3 servicios visibles |
| 6 | Formulario contacto funcional | POST /api/contact (TODO: implementar endpoint) |

### Definition of Done — HU-05.2

- [ ] 3 páginas principales creadas (home, servicios, contacto).
- [ ] Structured data (Schema.org) implementado.
- [ ] HIPAA disclaimer visible en todas las páginas.
- [ ] Accesibilidad WCAG 2.1 AA validada (Lighthouse).

---

## HU-05.3 — Deployment Cloudflare Pages + CI/CD

**Como** DevOps,  
**quiero** configurar deployment automatizado en Cloudflare Pages con CI/CD GitLab,  
**para que** cada push a main despliegue automáticamente la landing page con preview environments por PR.

### Tareas y Subtareas

#### T-05.3.1 — Configurar Cloudflare Pages project

- **ST-05.3.1.1** — Crear proyecto en Cloudflare Pages dashboard.
  - **Pasos**:
    1. Login a Cloudflare dashboard
    2. Pages → Create a project
    3. Connect Git (GitLab)
    4. Seleccionar repositorio `serenidad/serenidad-platform`
    5. Configurar build:
       - Framework: Astro
       - Build command: `npm run build`
       - Build output directory: `dist`
       - Root directory: `frontend/landing`
  - **CA:** Proyecto Cloudflare Pages creado.

- **ST-05.3.1.2** — Configurar custom domain.
  - **Pasos**:
    1. Pages project → Custom domains
    2. Agregar `sereni.dad`
    3. Configurar DNS CNAME en Cloudflare DNS:
       ```
       sereni.dad CNAME serenidad-landing.pages.dev
       ```
  - **CA:** Domain `sereni.dad` apunta a Cloudflare Pages.

#### T-05.3.2 — Configurar environment variables

- **ST-05.3.2.1** — Configurar variables de build en Cloudflare Pages.
  - **Variables**:
    ```
    NODE_VERSION=22
    PUBLIC_SITE_URL=https://sereni.dad
    ```
  - **CA:** Environment variables configuradas.

#### T-05.3.3 — Crear GitLab CI/CD pipeline

- **ST-05.3.3.1** — Crear `.gitlab-ci.yml` para landing.
  ```yaml
  stages:
    - test
    - deploy

  test:landing:
    stage: test
    image: node:22
    script:
      - cd frontend/landing
      - npm ci
      - npm run build
      - npx lighthouse-ci --upload-target=temporary-public-storage
    only:
      - merge_requests
      - main

  deploy:landing:production:
    stage: deploy
    image: node:22
    script:
      - cd frontend/landing
      - npm ci
      - npm run build
      - npx wrangler pages deploy dist --project-name=serenidad-landing
    environment:
      name: production
      url: https://sereni.dad
    only:
      - main
    when: manual
  ```
  - **CA:** CI/CD pipeline creado.

#### T-05.3.4 — Configurar Lighthouse CI thresholds

- **ST-05.3.4.1** — Crear `lighthouserc.json`.
  ```json
  {
    "ci": {
      "collect": {
        "staticDistDir": "./dist",
        "numberOfRuns": 3
      },
      "assert": {
        "assertions": {
          "categories:performance": ["error", {"minScore": 0.9}],
          "categories:accessibility": ["error", {"minScore": 0.9}],
          "categories:seo": ["error", {"minScore": 0.95}],
          "largest-contentful-paint": ["error", {"maxNumericValue": 2000}],
          "cumulative-layout-shift": ["error", {"maxNumericValue": 0.1}],
          "total-blocking-time": ["error", {"maxNumericValue": 200}]
        }
      }
    }
  }
  ```
  - **CA:** Lighthouse CI configurado con thresholds.

### Criterios de Aceptación de la HU-05.3

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Cloudflare Pages project activo | Dashboard muestra proyecto |
| 2 | Custom domain `sereni.dad` funcional | `curl https://sereni.dad` → 200 |
| 3 | CI pipeline pasa tests | GitLab pipeline verde |
| 4 | Lighthouse CI valida performance | LCP < 2s, CLS < 0.1 |
| 5 | Deployment manual a producción | Manual trigger deploy → site actualizado |
| 6 | Preview deployments por PR | Merge request → preview URL generado |

### Definition of Done — HU-05.3

- [ ] Cloudflare Pages project configurado.
- [ ] Custom domain `sereni.dad` activo con HTTPS.
- [ ] GitLab CI/CD pipeline funcional.
- [ ] Lighthouse CI gating con thresholds.
- [ ] Preview deployments automáticos por PR.

---

## Resumen de Entregables EP-05

**Código**:
- Astro v6 SSG project
- TailwindCSS v4 styling
- 3 páginas principales (home, servicios, contacto)
- BaseLayout con SEO optimizado
- Structured data (Schema.org)
- HIPAA disclaimer

**Performance**:
- LCP < 2.0s (target met)
- CLS < 0.1 (target met)
- Core Web Vitals optimizados
- Image optimization (AVIF/WebP)

**Deployment**:
- Cloudflare Pages project
- Custom domain `sereni.dad`
- GitLab CI/CD pipeline
- Lighthouse CI gating
- Preview environments

**Validación end-to-end**:
```bash
# 1. Build local
cd frontend/landing
npm run build

# 2. Preview local
npm run preview

# 3. Lighthouse audit
npx lighthouse http://localhost:4321 --view

# 4. Deploy production
git push origin main
# → GitLab pipeline → manual deploy → https://sereni.dad live

# 5. Verificar Core Web Vitals
# PageSpeed Insights: https://pagespeed.web.dev/analysis?url=https://sereni.dad
```

---

## Riesgos y Mitigaciones EP-05

| Riesgo | Impacto | Probabilidad | Mitigación |
|--------|---------|--------------|------------|
| LCP regression (images) | Medio | Media | Lighthouse CI gating + AVIF format |
| SEO metadata faltante | Alto | Baja | BaseLayout template + validation |
| Forms sin backend | Medio | Alta | Documentar endpoint /api/contact pendiente |
| Accesibilidad no cumplida | Alto | Baja | axe-core testing + WCAG checklist |

---

**Tiempo estimado EP-05**: 2 semanas (Sprint 6)  
**Esfuerzo**: ~40 horas diseño + desarrollo + 10 horas testing + 5 horas deployment
