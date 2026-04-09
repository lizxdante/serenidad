# EP-CS01 — Project Scaffold & Design System

**Épica:** Como desarrollador frontend, necesito un proyecto Astro v6 correctamente configurado con TailwindCSS v4, design tokens healthcare-appropriate, y estructura de archivos escalable, para que la landing page coming soon sea rápida, mantenible y evolucione hacia la landing completa de Fase 2.

**Prioridad:** Must Have — Fundacional
**Sprint:** Sprint 1
**Dependencias Entrantes:** Ninguna
**Dependencias Salientes:** EP-CS02, EP-CS03

---

## HU-CS01.1 — Astro v6 SSG Project Scaffold

**Como** desarrollador frontend,
**quiero** crear un proyecto Astro v6 con configuración SSG, TypeScript 6.0 strict, y estructura de archivos organizada,
**para que** la landing page tenga una base sólida, tipada, y preparada para evolucionar.

### Contexto Técnico

- **Runtime:** Node.js 22 LTS
- **Package manager:** npm (consistente con el ecosistema Astro)
- **Output mode:** `static` (SSG puro — zero server-side rendering)
- **TypeScript:** 6.0 con `verbatimModuleSyntax: true` (preparación para TS 7)
- **Estructura:** Monorepo path `frontend/comingsoon/`

### Tareas y Subtareas

#### T-CS01.1.1 — Inicializar proyecto Astro

- **ST-CS01.1.1.1** — Crear proyecto Astro v6.
  ```bash
  mkdir -p frontend/comingsoon
  cd frontend/comingsoon
  npm create astro@latest . -- --template minimal --typescript strict
  ```
  - **CA:** Proyecto Astro creado con `package.json`, `astro.config.mjs`, `tsconfig.json`.

- **ST-CS01.1.1.2** — Verificar versión Astro.
  ```bash
  npx astro --version
  # Esperado: astro v6.x.x
  ```
  - **CA:** Astro v6.x confirmado.

- **ST-CS01.1.1.3** — Actualizar TypeScript a 6.0.
  ```bash
  npm install -D typescript@^6.0.0
  npx tsc --version
  # Esperado: Version 6.x.x
  ```
  - **CA:** TypeScript 6.0 instalado.

#### T-CS01.1.2 — Instalar dependencias

- **ST-CS01.1.2.1** — Instalar TailwindCSS v4.
  ```bash
  npm install -D tailwindcss@next @tailwindcss/vite
  ```
  - **CA:** TailwindCSS v4 instalado.

- **ST-CS01.1.2.2** — Instalar integraciones Astro.
  ```bash
  npm install -D @astrojs/sitemap
  npm install lucide-astro
  ```
  - **CA:** `@astrojs/sitemap` y `lucide-astro` instalados.

- **ST-CS01.1.2.3** — Instalar dependencias de desarrollo.
  ```bash
  npm install -D wrangler
  npm install -D lighthouse
  ```
  - **CA:** `wrangler` y `lighthouse` instalados.

#### T-CS01.1.3 — Configurar Astro para SSG

- **ST-CS01.1.3.1** — Crear `astro.config.mjs`.
  ```js
  import { defineConfig } from 'astro/config';
  import tailwindcss from '@tailwindcss/vite';
  import sitemap from '@astrojs/sitemap';

  export default defineConfig({
    site: 'https://sereni.dad',
    output: 'static',
    integrations: [sitemap()],
    vite: {
      plugins: [tailwindcss()],
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
  - **CA:** Config SSG con TailwindCSS v4, sitemap, y image optimization.

#### T-CS01.1.4 — Configurar TypeScript 6.0

- **ST-CS01.1.4.1** — Crear `tsconfig.json`.
  ```json
  {
    "extends": "astro/tsconfigs/strict",
    "compilerOptions": {
      "verbatimModuleSyntax": true,
      "noUncheckedIndexedAccess": true,
      "exactOptionalPropertyTypes": true,
      "noImplicitOverride": true,
      "skipLibCheck": true,
      "resolveJsonModule": true
    }
  }
  ```
  - **CA:** TypeScript strict con `verbatimModuleSyntax` (preparado para TS 7).

#### T-CS01.1.5 — Crear estructura de directorios

- **ST-CS01.1.5.1** — Crear estructura completa.
  ```bash
  mkdir -p src/{styles,layouts,components,scripts,pages,types}
  mkdir -p public
  mkdir -p functions/api
  mkdir -p schema
  ```
  - **CA:** Directorios creados.

#### T-CS01.1.6 — Configurar wrangler.toml para D1

- **ST-CS01.1.6.1** — Crear `wrangler.toml`.
  ```toml
  name = "serenidad-comingsoon"
  compatibility_date = "2026-04-01"
  compatibility_flags = ["nodejs_compat"]

  [[d1_databases]]
  binding = "DB"
  database_name = "serenidad-leads"
  database_id = "<DATABASE_ID>"  # Se obtiene tras crear la DB
  ```
  - **CA:** Wrangler configurado con binding D1.

### Criterios de Aceptación — HU-CS01.1

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Proyecto Astro compila | `npm run build` exitoso |
| 2 | Output es SSG (static) | `dist/` contiene solo HTML/CSS/JS estáticos |
| 3 | TypeScript 6.0 strict | `npx tsc --noEmit` sin errores |
| 4 | TailwindCSS v4 funciona | Clases utility se aplican correctamente |
| 5 | Estructura de directorios completa | Todos los directorios existen |
| 6 | wrangler.toml configurado | D1 binding presente |

### Definition of Done — HU-CS01.1

- [ ] Proyecto Astro v6 scaffolded con TypeScript 6.0 strict
- [ ] TailwindCSS v4 configurado y funcional
- [ ] Estructura de directorios creada según especificación
- [ ] `npm run build` genera sitio estático sin errores
- [ ] wrangler.toml con D1 binding configurado
- [ ] tsconfig.json con `verbatimModuleSyntax: true`

---

## HU-CS01.2 — Design System & Base Layout

**Como** diseñador/desarrollador frontend,
**quiero** un design system coherente con tokens de color, tipografía y espaciado apropiados para salud mental, y un layout base con SEO meta tags completo,
**para que** la landing page transmita calma, profesionalismo y confianza desde el primer render.

### Contexto Técnico

**Principios de diseño para salud mental:**
- Colores fríos y suaves (azules, verdes, violetas pálidos)
- Tipografía legible y amigable
- Espaciado generoso (respira)
- Sin elementos agresivos o urgentes
- Contraste suficiente para accesibilidad

### Tareas y Subtareas

#### T-CS01.2.1 — Configurar TailwindCSS v4 con design tokens

- **ST-CS01.2.1.1** — Crear `src/styles/global.css`.
  ```css
  @import "tailwindcss";

  @theme {
    /* Colores primarios */
    --color-primary: #3B82F6;
    --color-primary-dark: #2563EB;
    --color-primary-light: #93C5FD;

    /* Colores secundarios */
    --color-secondary: #8B5CF6;
    --color-secondary-dark: #7C3AED;
    --color-secondary-light: #C4B5FD;

    /* Colores accent */
    --color-accent: #10B981;
    --color-accent-dark: #059669;
    --color-accent-light: #6EE7B7;

    /* Colores WhatsApp */
    --color-whatsapp: #25D366;
    --color-whatsapp-dark: #128C7E;

    /* Colores neutros */
    --color-bg: #FFFFFF;
    --color-bg-alt: #F8FAFC;
    --color-bg-dark: #0F172A;
    --color-text: #1E293B;
    --color-text-muted: #64748B;
    --color-text-light: #94A3B8;
    --color-border: #E2E8F0;

    /* Tipografía */
    --font-sans: 'Inter Variable', 'Inter', system-ui, sans-serif;
    --font-display: 'Lexend Variable', 'Lexend', sans-serif;

    /* Sombras */
    --shadow-sm: 0 1px 2px 0 rgb(0 0 0 / 0.05);
    --shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.1);
    --shadow-lg: 0 10px 15px -3px rgb(0 0 0 / 0.1);
    --shadow-xl: 0 20px 25px -5px rgb(0 0 0 / 0.1);

    /* Border radius */
    --radius-sm: 0.375rem;
    --radius-md: 0.5rem;
    --radius-lg: 0.75rem;
    --radius-xl: 1rem;
    --radius-2xl: 1.5rem;
    --radius-full: 9999px;

    /* Transiciones */
    --transition-fast: 150ms ease;
    --transition-base: 250ms ease;
    --transition-slow: 350ms ease;
  }

  @layer base {
    html {
      scroll-behavior: smooth;
    }

    body {
      font-family: var(--font-sans);
      color: var(--color-text);
      background-color: var(--color-bg);
      -webkit-font-smoothing: antialiased;
      -moz-osx-font-smoothing: grayscale;
    }

    h1, h2, h3, h4, h5, h6 {
      font-family: var(--font-display);
    }
  }
  ```
  - **CA:** Design tokens definidos como CSS custom properties via `@theme`.

#### T-CS01.2.2 — Crear BaseLayout con SEO completo

- **ST-CS01.2.2.1** — Crear `src/layouts/BaseLayout.astro`.
  ```astro
  ---
  import '../styles/global.css';

  interface Props {
    title: string;
    description: string;
    image?: string;
    canonicalUrl?: string;
  }

  const {
    title,
    description,
    image = '/og-image.jpg',
    canonicalUrl,
  } = Astro.props;

  const siteUrl = Astro.site?.toString() || 'https://sereni.dad';
  const canonical = canonicalUrl || new URL(Astro.url.pathname, siteUrl).toString();
  const ogImage = new URL(image, siteUrl).toString();
  ---
  <!doctype html>
  <html lang="es" class="scroll-smooth">
    <head>
      <meta charset="UTF-8" />
      <meta name="viewport" content="width=device-width, initial-scale=1.0" />

      <!-- Primary Meta Tags -->
      <title>{title}</title>
      <meta name="title" content={title} />
      <meta name="description" content={description} />
      <meta name="robots" content="index, follow" />
      <link rel="canonical" href={canonical} />

      <!-- Open Graph -->
      <meta property="og:type" content="website" />
      <meta property="og:url" content={canonical} />
      <meta property="og:title" content={title} />
      <meta property="og:description" content={description} />
      <meta property="og:image" content={ogImage} />
      <meta property="og:locale" content="es_MX" />
      <meta property="og:site_name" content="Serenidad" />

      <!-- Twitter -->
      <meta property="twitter:card" content="summary_large_image" />
      <meta property="twitter:url" content={canonical} />
      <meta property="twitter:title" content={title} />
      <meta property="twitter:description" content={description} />
      <meta property="twitter:image" content={ogImage} />

      <!-- Favicon -->
      <link rel="icon" type="image/svg+xml" href="/favicon.svg" />

      <!-- Fonts: preload critical -->
      <link rel="preconnect" href="https://fonts.googleapis.com" />
      <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
      <link
        href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600&family=Lexend:wght@600;700;800&display=swap"
        rel="stylesheet"
      />

      <!-- Sitemap -->
      <link rel="sitemap" href="/sitemap-index.xml" />

      <!-- Manifest -->
      <link rel="manifest" href="/manifest.json" />

      <!-- Theme Color -->
      <meta name="theme-color" content="#3B82F6" />
    </head>
    <body class="min-h-screen flex flex-col">
      <main class="flex-1">
        <slot />
      </main>
    </body>
  </html>
  ```
  - **CA:** BaseLayout con SEO meta tags, Open Graph, Twitter Cards, fonts preload.

#### T-CS01.2.3 — Crear assets estáticos

- **ST-CS01.2.3.1** — Crear `public/favicon.svg`.
  ```svg
  <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 32 32">
    <rect width="32" height="32" rx="8" fill="#3B82F6"/>
    <path d="M8 16 C8 11.6 11.6 8 16 8 C20.4 8 24 11.6 24 16 C24 20.4 20.4 24 16 24" 
          stroke="white" stroke-width="2.5" fill="none" stroke-linecap="round"/>
    <circle cx="16" cy="16" r="3" fill="white"/>
  </svg>
  ```
  - **CA:** Favicon SVG creado.

- **ST-CS01.2.3.2** — Crear `public/robots.txt`.
  ```
  User-agent: *
  Allow: /

  Sitemap: https://sereni.dad/sitemap-index.xml
  ```
  - **CA:** robots.txt creado.

- **ST-CS01.2.3.3** — Crear `public/manifest.json`.
  ```json
  {
    "name": "Serenidad — Salud Mental",
    "short_name": "Serenidad",
    "description": "Clínica digital de salud mental. Terapia online accesible y profesional.",
    "start_url": "/",
    "display": "standalone",
    "background_color": "#FFFFFF",
    "theme_color": "#3B82F6",
    "icons": [
      {
        "src": "/favicon.svg",
        "sizes": "any",
        "type": "image/svg+xml"
      }
    ]
  }
  ```
  - **CA:** Web manifest creado.

#### T-CS01.2.4 — Crear tipos TypeScript compartidos

- **ST-CS01.2.4.1** — Crear `src/types/lead.ts`.
  ```typescript
  /** Payload del formulario de lead */
  export interface LeadSubmission {
    name: string;
    email: string;
    phone: string;
    message?: string;
  }

  /** Respuesta exitosa del API */
  export interface LeadCreated {
    id: number;
    status: 'created' | 'duplicate';
    whatsapp_url: string;
  }

  /** Error de validación del API */
  export interface ValidationError {
    error: 'VALIDATION_ERROR';
    details: Array<{
      field: string;
      message: string;
    }>;
  }

  /** Error del servidor */
  export interface ServerError {
    error: 'INTERNAL_ERROR';
    message: string;
  }

  /** Unión de todas las respuestas posibles */
  export type LeadApiResponse = LeadCreated | ValidationError | ServerError;
  ```
  - **CA:** Tipos compartidos definidos con `export type` (compatible TS 7).

### Criterios de Aceptación — HU-CS01.2

| # | Criterio | Verificación |
|---|----------|--------------|
| 1 | Design tokens aplicados | Colores, fonts y spacing se renderizan correctamente |
| 2 | BaseLayout renderiza | HTML válido con todos los meta tags |
| 3 | SEO meta tags completos | og:title, og:description, og:image, twitter:card presentes |
| 4 | Fonts preload | Google Fonts cargan con `display=swap` |
| 5 | Favicon visible | Tab del navegador muestra icono |
| 6 | Tipos TypeScript sin errores | `npx tsc --noEmit` pasa |
| 7 | Contraste WCAG AA | Todos los colores texto/fondo tienen ratio ≥ 4.5:1 |

### Definition of Done — HU-CS01.2

- [ ] TailwindCSS v4 con design tokens healthcare definidos
- [ ] BaseLayout con SEO completo (meta tags, OG, Twitter, canonical)
- [ ] Assets estáticos (favicon, robots.txt, manifest.json)
- [ ] Tipos TypeScript compartidos para el formulario
- [ ] Contraste verificado con herramienta de accesibilidad
- [ ] Build exitoso sin errores de TypeScript

---

## Resumen de Entregables EP-CS01

**Código:**
- Proyecto Astro v6 con TypeScript 6.0 strict
- TailwindCSS v4 con design tokens healthcare
- BaseLayout con SEO meta tags completo
- Tipos TypeScript compartidos
- Assets estáticos (favicon, robots.txt, manifest.json)

**Infraestructura:**
- wrangler.toml con D1 binding configurado
- Estructura de directorios lista para desarrollo

**Validación:**
```bash
# 1. Instalar dependencias
cd frontend/comingsoon && npm install

# 2. Verificar TypeScript
npx tsc --noEmit  # Sin errores

# 3. Build
npm run build     # Exitoso, dist/ generado

# 4. Preview local
npm run preview   # http://localhost:4321
```
