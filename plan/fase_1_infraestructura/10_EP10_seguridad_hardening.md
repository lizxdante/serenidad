# EP-10 — Seguridad y Hardening

**Épica:** Como equipo de infraestructura, necesitamos NetworkPolicies de Kubernetes implementadas con modelo deny-all-by-default y reglas explícitas de comunicación entre pods, además de un script de actualización de firewall Hetzner para IPs dinámicas, para que el tráfico entre componentes esté estrictamente controlado en dos capas (firewall Hetzner + NetworkPolicies K8s) y se reduzca la superficie de ataque.

**Origen:** Sección §24 (V-19) del documento de decisión.
**Prioridad:** Alta — Segunda línea de defensa.
**Sprint:** S3–S4
**Dependencias Entrantes:** EP-02 (cluster K8s con flannel que soporta NetworkPolicies), EP-07 (IAM Service desplegado — para validar reglas de comunicación).
**Dependencias Salientes:** EP-11 (Observabilidad depende de red funcional).

---

## HU-10.1 — Firewall y Hardening de Red (V-19)

**Como** ingeniero de seguridad,
**quiero** tener NetworkPolicies de Kubernetes que implementen deny-all por defecto en los namespaces de producción con reglas explícitas para el tráfico permitido, y un script para actualizar el firewall de Hetzner ante cambios de IP de trabajo,
**para que** ningún pod pueda comunicarse con otro sin autorización explícita y el acceso administrativo al cluster esté siempre restringido a mi IP actual.

**Dependencias:** EP-02 HU-02.3 (cluster K8s con namespaces), EP-07 HU-07.2 (IAM Service desplegado — para reglas Traefik→IAM y IAM→PG).

### Tareas y Subtareas

#### T-10.1.1 — Crear NetworkPolicies deny-all por defecto

- **ST-10.1.1.1** — Crear directorio `infra/infrastructure/network-policies/`.
  - **CA:** Directorio existe (puede ya existir desde EP-03).
- **ST-10.1.1.2** — Crear archivo `infra/infrastructure/network-policies/deny-all-default.yaml`.
  - **CA:** NetworkPolicy `deny-all-ingress` en namespace `serenidad-core`:
    - `podSelector: {}` (todos los pods).
    - `policyTypes: [Ingress, Egress]`.
    - Sin reglas (denegar todo).
  - **CA:** NetworkPolicy `deny-all-ingress` en namespace `serenidad-data`:
    - `podSelector: {}`.
    - `policyTypes: [Ingress, Egress]`.
    - Sin reglas (denegar todo).
- **ST-10.1.1.3** — Commitear y pushear.
  - **CA:** FluxCD aplica las policies.
  - **CA:** `kubectl get networkpolicies -n serenidad-core` → `deny-all-ingress`.
  - **CA:** `kubectl get networkpolicies -n serenidad-data` → `deny-all-ingress`.

#### T-10.1.2 — Crear NetworkPolicy: Traefik → IAM Service

- **ST-10.1.2.1** — Agregar NetworkPolicy `allow-traefik-to-iam` al archivo o como archivo separado.
  - **CA:** Namespace: `serenidad-core`.
  - **CA:** `podSelector.matchLabels.app: iam-service`.
  - **CA:** `policyTypes: [Ingress]`.
  - **CA:** Ingress: from namespace `traefik` (via `namespaceSelector.matchLabels.kubernetes.io/metadata.name: traefik`).
  - **CA:** Port: TCP 8080.
- **ST-10.1.2.2** — Verificar que Traefik puede alcanzar el IAM Service.
  - **CA:** `curl https://api.sereni.dad/api/iam/health` → respuesta del IAM Service (no bloqueado por NetworkPolicy).
  - **CA:** Request desde otro pod (no Traefik) al IAM Service → bloqueado.

#### T-10.1.3 — Crear NetworkPolicy: IAM Service → PostgreSQL

- **ST-10.1.3.1** — Agregar NetworkPolicy `allow-iam-to-postgres`.
  - **CA:** Namespace: `serenidad-data`.
  - **CA:** `podSelector.matchLabels: cnpg.io/cluster: serenidad-pg`.
  - **CA:** `policyTypes: [Ingress]`.
  - **CA:** Ingress: from namespace `serenidad-core`.
  - **CA:** Port: TCP 5432.
- **ST-10.1.3.2** — Verificar conectividad IAM → PG.
  - **CA:** IAM Service puede conectar a PostgreSQL (health `/ready` → db connected).
  - **CA:** Pod en otro namespace (no serenidad-core) no puede conectar a PG en serenidad-data.

#### T-10.1.4 — Crear NetworkPolicy: Kratos → PostgreSQL

- **ST-10.1.4.1** — Verificar que la regla `allow-iam-to-postgres` cubre a Kratos (también en `serenidad-core`).
  - **CA:** Si Kratos está en `serenidad-core`, la regla que permite from `serenidad-core` ya lo cubre.
  - **CA:** Si no, crear regla adicional.
- **ST-10.1.4.2** — Verificar que Kratos puede conectar a `kratos_db`.
  - **CA:** Kratos health `/health/ready` → `{"status":"ok"}`.

#### T-10.1.5 — Crear NetworkPolicy: DNS egress para todos los pods

- **ST-10.1.5.1** — Agregar NetworkPolicy `allow-dns-egress` en `serenidad-core`.
  - **CA:** `podSelector: {}` (todos los pods).
  - **CA:** `policyTypes: [Egress]`.
  - **CA:** Egress: to namespace `kube-system`, ports UDP 53 y TCP 53.
- **ST-10.1.5.2** — Agregar misma policy en `serenidad-data`.
  - **CA:** Pods en `serenidad-data` también pueden resolver DNS.
- **ST-10.1.5.3** — Verificar resolución DNS desde pods.
  - **CA:** `kubectl exec -it deployment/iam-service -n serenidad-core -- nslookup serenidad-pg-rw.serenidad-data.svc.cluster.local` → resuelve.

#### T-10.1.6 — Crear NetworkPolicy: egress adicional para pods que necesitan acceso externo

- **ST-10.1.6.1** — Crear policy para IAM Service → Kratos Admin (egress interno).
  - **CA:** IAM Service puede hacer requests a `kratos-admin.serenidad-core.svc.cluster.local:4434`.
- **ST-10.1.6.2** — Crear policy para cert-manager → Let's Encrypt (egress externo HTTPS).
  - **CA:** cert-manager puede contactar servidores ACME de Let's Encrypt.
- **ST-10.1.6.3** — Crear policy para CloudNativePG → Backblaze B2 (egress externo HTTPS).
  - **CA:** Pod de PG puede enviar WAL y backups a B2 via HTTPS.

#### T-10.1.7 — Crear kustomization.yaml del componente network-policies

- **ST-10.1.7.1** — Crear `infra/infrastructure/network-policies/kustomization.yaml`.
  - **CA:** Resources: `deny-all-default.yaml` y cualquier archivo adicional de policies.
- **ST-10.1.7.2** — Commitear y verificar reconciliación.
  - **CA:** FluxCD reconcilia sin error.

#### T-10.1.8 — Crear script de actualización de firewall Hetzner

- **ST-10.1.8.1** — Crear archivo `scripts/update-firewall.sh`.
  - **CA:** Script:
    - `set -euo pipefail`.
    - `source .envrc`.
    - Obtiene IP actual con `curl -s https://ifconfig.me`.
    - Obtiene ID del firewall `serenidad-prod-fw` con `hcloud firewall list -o json | jq`.
    - Actualiza regla del puerto 50000 (Talos API) con `hcloud firewall replace-rule`.
    - Actualiza regla del puerto 6443 (K8s API) con `hcloud firewall replace-rule`.
    - Imprime mensaje de confirmación.
  - **CA:** Script funcional y probado.
- **ST-10.1.8.2** — Hacer ejecutable (`chmod +x`).
  - **CA:** Permisos de ejecución presentes.
- **ST-10.1.8.3** — Commitear y pushear.
  - **CA:** Script en `scripts/update-firewall.sh` en `main`.

#### T-10.1.9 — Verificar aislamiento de red completo

- **ST-10.1.9.1** — Test: pod en `default` namespace no puede alcanzar PG en `serenidad-data`.
  - **CA:** Timeout o connection refused.
- **ST-10.1.9.2** — Test: pod en `default` namespace no puede alcanzar IAM Service en `serenidad-core`.
  - **CA:** Timeout o connection refused.
- **ST-10.1.9.3** — Test: IAM Service puede alcanzar PG y Kratos.
  - **CA:** `/ready` → `{"status":"ok","db":"connected","kratos":"reachable"}`.
- **ST-10.1.9.4** — Test: Traefik puede alcanzar IAM Service y Kratos.
  - **CA:** Requests HTTPS proxiados correctamente.
- **ST-10.1.9.5** — Test: script `update-firewall.sh` actualiza reglas correctamente.
  - **CA:** `./scripts/update-firewall.sh` → firewall actualizado con nueva IP.
  - **CA:** `hcloud firewall describe serenidad-prod-fw` → IPs actualizadas en reglas 50000 y 6443.

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

IMPORTANTE: T-10.1.1 DEBE aplicarse junto con T-10.1.2 a T-10.1.6 en el mismo commit
para evitar downtime. Alternativa: aplicar las allow-rules primero, y deny-all al final.

T-10.1.7 (kustomization) — agrupa todo
T-10.1.8 (script firewall) — independiente de las NetworkPolicies
T-10.1.9 (verificación) — después de todo lo anterior

Recomendación: Aplicar en orden:
  1. Reglas allow (T-10.1.2 a T-10.1.6)
  2. Reglas deny-all (T-10.1.1) — en el mismo commit
  3. Verificar (T-10.1.9)
```
