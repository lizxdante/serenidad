# EP-10 — Seguridad y Hardening

**Épica:** Como equipo de infraestructura, necesitamos NetworkPolicies de Kubernetes implementadas con modelo deny-all-by-default y reglas explícitas de comunicación entre pods, además de un script de actualización de firewall Hetzner para IPs dinámicas, para que el tráfico entre componentes esté estrictamente controlado en dos capas (firewall Hetzner + NetworkPolicies K8s) y se reduzca la superficie de ataque.

**Origen:** Sección §24 (V-19) del documento de decisión.
**Prioridad:** Alta — Segunda línea de defensa.
**Sprint:** S3–S4
**Dependencias Entrantes:** EP-02 (cluster K8s con flannel que soporta NetworkPolicies), EP-03 HU-03.1 (FluxCD para Kustomization de manifiestos), EP-05 (CloudNativePG desplegado — labels `cnpg.io/cluster` en pods de PG), EP-07 (IAM Service desplegado — para validar reglas de comunicación).
**Dependencias Salientes:** EP-11 (Observabilidad depende de red funcional).

---

## HU-10.1 — Firewall y Hardening de Red (V-19)

**Como** ingeniero de seguridad,
**quiero** tener NetworkPolicies de Kubernetes que implementen deny-all por defecto en los namespaces de producción con reglas explícitas para el tráfico permitido, y un script para actualizar el firewall de Hetzner ante cambios de IP de trabajo,
**para que** ningún pod pueda comunicarse con otro sin autorización explícita y el acceso administrativo al cluster esté siempre restringido a mi IP actual.

**Dependencias:** EP-02 HU-02.3 (cluster K8s con namespaces), EP-07 HU-07.2 (IAM Service desplegado — para reglas Traefik→IAM y IAM→PG).

### Tareas y Subtareas

> **Implementación detallada:** Ver `HU-10.1A_Network_Policies_K8s.md` para los manifiestos YAML completos y comandos de verificación de las NetworkPolicies (T-10.1.1 a T-10.1.7, T-10.1.9). Ver `HU-10.1B_Firewall_Hetzner_Dinamico.md` para el script de firewall (T-10.1.8).

| Tarea | Descripción | Archivo detallado |
|-------|-------------|-------------------|
| T-10.1.1 | NetworkPolicies deny-all en `serenidad-core` y `serenidad-data` | HU-10.1A |
| T-10.1.2 | Allow: Traefik → IAM Service (port 8080) | HU-10.1A |
| T-10.1.3 | Allow: IAM Service → PostgreSQL (port 5432) | HU-10.1A |
| T-10.1.4 | Allow: Kratos → PostgreSQL (cubierto por regla serenidad-core) | HU-10.1A |
| T-10.1.5 | Allow: DNS egress (port 53 UDP/TCP) en ambos namespaces | HU-10.1A |
| T-10.1.6 | Allow: egress externo (cert-manager→LE, CNPG→B2, IAM→Kratos Admin) | HU-10.1A |
| T-10.1.7 | Kustomization del componente network-policies | HU-10.1A |
| T-10.1.8 | Script `update-firewall.sh` para IP dinámica | HU-10.1B |
| T-10.1.9 | Verificación completa de aislamiento de red | HU-10.1A |

### Criterios de Aceptación de la HU-10.1

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-1 | Deny-all por defecto | NetworkPolicies en serenidad-core y serenidad-data |
| CA-2 | Traefik → IAM permitido | Health del IAM accesible via Traefik |
| CA-3 | IAM → PG permitido | IAM readiness → db connected |
| CA-4 | Kratos → PG permitido | Kratos health → ok |
| CA-5 | DNS egress permitido | Pods resuelven DNS internamente |
| CA-6 | Aislamiento verificado | Pod en default no alcanza servicios en serenidad-core ni data |
| CA-7 | Script update-firewall.sh funcional | Actualiza IPs en firewall Hetzner |
| CA-8 | 2 capas de defensa | Firewall Hetzner (L3/L4) + NetworkPolicies K8s (L3/L4 interno) |

### Definition of Done — HU-10.1

- [ ] NetworkPolicies deny-all aplicadas en serenidad-core y serenidad-data.
- [ ] Reglas explícitas para: Traefik→IAM, IAM→PG, Kratos→PG, DNS egress.
- [ ] Reglas de egress para cert-manager, CNPG→B2.
- [ ] Aislamiento verificado con tests de conectividad positivos y negativos.
- [ ] Script `update-firewall.sh` creado y probado.
- [ ] Manifests commiteados en estructura GitOps.

---

## Resumen de Dependencias Internas EP-10

```
T-10.1.1 (deny-all) — se aplica primero, rompe conectividad
  ├──► T-10.1.2 (allow Traefik→IAM) — restaura acceso externo al IAM
  ├──► T-10.1.3 (allow IAM→PG) — restaura acceso del IAM a la DB
  ├──► T-10.1.4 (allow Kratos→PG) — restaura acceso de Kratos a la DB
  ├──► T-10.1.5 (allow DNS egress) — restaura resolución DNS
  └──► T-10.1.6 (allow egress externo) — restaura acceso a servicios externos

IMPORTANTE: T-10.1.1 (deny-all) DEBE aplicarse ATÓMICAMENTE junto con T-10.1.2 a
T-10.1.6 (allow-rules) en un ÚNICO commit para que FluxCD las aplique como conjunto.
Aplicar deny-all sin las excepciones causa downtime inmediato.

ORDEN DE IMPLEMENTACIÓN OBLIGATORIO:
  1. Escribir PRIMERO las reglas allow (T-10.1.2 a T-10.1.6)
  2. Escribir las reglas deny-all (T-10.1.1)
  3. Agrupar en kustomization (T-10.1.7)
  4. Commitear TODO en un solo push — FluxCD aplica el conjunto completo
  5. Verificar (T-10.1.9)

T-10.1.8 (script firewall) — independiente de las NetworkPolicies, puede hacerse en paralelo
```
