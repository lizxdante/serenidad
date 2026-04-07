# Fase 1 — Plan Ágil de Infraestructura: Resumen de Épicas

**Proyecto:** serenidad — Clínica Digital de Salud Mental Global
**Sprint Goal:** Entorno de producción completo en VPS Hetzner CX32 con Talos Linux, flujo end-to-end funcional (registro → login Passkey → JWT → dashboard).
**Fuente:** `decisions/07b_fase1_vps_hetzner_produccion.md`
**Fecha:** Abril 2026

---

## Inventario de Épicas

| ID | Épica | Tareas Origen | HUs | Prioridad | Sprint Sugerido |
|----|-------|---------------|-----|-----------|-----------------|
| EP-01 | Preparación y Cuentas Externas | Prereqs §4 + V-00 | 4 | Crítica | S1 |
| EP-02 | Infraestructura Base VPS | V-01 a V-04 | 4 | Crítica | S1 |
| EP-03 | GitOps y Gestión de Configuración | V-05, V-06, V-09 | 3 | Crítica | S1–S2 |
| EP-04 | Networking y TLS | V-07, V-08, V-11 | 3 | Crítica | S2 |
| EP-05 | Base de Datos y Backups | V-10, V-17 | 2 | Crítica | S2 |
| EP-06 | Schemas de Eventos Protobuf | V-12 | 1 | Alta | S2 |
| EP-07 | IAM — Autenticación y Autorización | V-13, V-14 | 2 | Crítica | S3 |
| EP-08 | Frontend y BFF | V-15, V-16 | 2 | Crítica | S3 |
| EP-09 | CI/CD Pipeline | V-18 | 1 | Alta | S3 |
| EP-10 | Seguridad y Hardening | V-19 | 1 | Alta | S3–S4 |
| EP-11 | Observabilidad y Operaciones Día-2 | §25, §26, §27 | 3 | Media | S4 |

**Total: 11 Épicas · 26 Historias de Usuario · ~120 Tareas · ~300 Subtareas**

---

## Diagrama de Dependencias entre Épicas

```
EP-01 Preparación y Cuentas
  └──► EP-02 Infraestructura Base VPS
        └──► EP-03 GitOps y Configuración
              ├──► EP-04 Networking y TLS
              │     └──► EP-07 IAM (parcial: IngressRoutes dependen de Traefik+DNS)
              ├──► EP-05 Base de Datos y Backups
              │     └──► EP-07 IAM (parcial: Kratos y IAM Service dependen de PG)
              └──► EP-06 Schemas Protobuf
                    └──► EP-07 IAM (parcial: eventos Protobuf en Outbox)
                          └──► EP-08 Frontend y BFF
                                └──► EP-09 CI/CD Pipeline
        └──► EP-10 Seguridad y Hardening (paralelo desde EP-02 en adelante)
              └──► EP-11 Observabilidad y Operaciones
```

## Diagrama de Dependencias entre Tareas (detalle)

```
V-00 Preparar monorepo
  └─► V-01 Aprovisionamiento Hetzner CX32
       └─► V-02 Instalar Talos Linux (modo rescue)
            └─► V-03 Bootstrap Talos + Kubernetes 1.33.x
                 ├─► V-04 Hetzner CCM
                 ├─► V-05 FluxCD v2.x bootstrap
                 │    └─► V-06 Estructura GitOps del repo
                 │         ├─► V-07 cert-manager v1.x
                 │         │    └─► V-08 Traefik v3.x
                 │         │         └─► V-11 DNS Cloudflare
                 │         ├─► V-09 SOPS + age (cifrado GitOps)
                 │         │    └─► V-12 Protobuf schemas
                 │         └─► V-10 CloudNativePG + PG17.4
                 │              └─► V-17 ScheduledBackup → B2
                 │                   └─► V-13 Ory Kratos v1.3.1
                 │                        └─► V-14 IAM Domain Service Go
                 │                             ├─► V-15 BFF Hono CF Workers
                 │                             │    └─► V-16 Qwik SPA CF Pages
                 │                             └─► V-18 GitLab CI/CD
                 └─► V-19 Firewall y hardening de red
```

---

## Definition of Done (DoD) Global — Fase 1

Cada Historia de Usuario se considera **Done** cuando:

1. **Código/Config commiteado** en la rama `main` del monorepo.
2. **Secrets cifrados** con SOPS+age (ningún secret en texto plano en Git).
3. **FluxCD reconcilia** sin errores (`flux get all` → todos Ready).
4. **Criterios de Aceptación** de la HU verificados y documentados.
5. **Backup de credenciales críticas** en password manager.
6. **Documentación** de troubleshooting actualizada si aplica.
7. **Sin regresiones** en componentes previamente desplegados.

## Criterios de Aceptación Globales de Fase 1

La Fase 1 está **completa** cuando se cumple el flujo end-to-end:

```
1. Usuario abre https://app.sereni.dad
2. Registra Passkey (WebAuthn/FIDO2)
3. Verifica email via Resend
4. Inicia sesión con Passkey
5. Frontend intercambia session token por JWT Ed25519
6. IAM Service crea perfil + evento UserRegistered en Outbox (misma TX)
7. Frontend usa JWT para llamadas autenticadas
8. BFF verifica JWT y propaga claims al backend
9. Dashboard del usuario se muestra correctamente
10. Todo el flujo completa en < 3 segundos
```

---

## Archivos del Plan

| Archivo | Contenido |
|---------|-----------|
| `00_resumen_epicas.md` | Este documento — visión general |
| `01_EP01_preparacion_cuentas.md` | Épica 1: Cuentas externas y preparación del monorepo |
| `02_EP02_infraestructura_vps.md` | Épica 2: VPS Hetzner, Talos, K8s, CCM |
| `03_EP03_gitops_configuracion.md` | Épica 3: FluxCD, estructura GitOps, SOPS+age |
| `04_EP04_networking_tls.md` | Épica 4: cert-manager, Traefik, DNS Cloudflare |
| `05_EP05_base_datos_backups.md` | Épica 5: CloudNativePG, PostgreSQL 17.4, Backblaze B2 |
| `06_EP06_schemas_eventos.md` | Épica 6: Protobuf + buf.build |
| `07_EP07_iam_autenticacion.md` | Épica 7: Ory Kratos, IAM Domain Service Go |
| `08_EP08_frontend_bff.md` | Épica 8: BFF Hono CF Workers, Qwik SPA CF Pages |
| `09_EP09_cicd_pipeline.md` | Épica 9: GitLab CI/CD pipelines |
| `10_EP10_seguridad_hardening.md` | Épica 10: NetworkPolicies, firewall hardening |
| `11_EP11_observabilidad_operaciones.md` | Épica 11: Health checks, monitoreo, troubleshooting, día-2 |
