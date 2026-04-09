# EP-11 — Observabilidad y Operaciones Día-2

**Épica:** Como equipo de operaciones, necesitamos health checks obligatorios en cada microservicio, consultas de estado rápidas documentadas, alertas básicas via Hetzner Monitoring, procedimientos de día-2 (deploy de emergencia, gestión de secretos, actualización de Talos, recuperación ante fallos), y troubleshooting exhaustivo documentado, para poder operar, diagnosticar y recuperar el sistema de producción con tiempos de respuesta predecibles.

**Origen:** Secciones §25 (Flujo de trabajo diario), §26 (Observabilidad mínima viable), §27 (Troubleshooting exhaustivo) del documento de decisión.
**Prioridad:** Media — No bloquea funcionalidad pero es esencial para operación.
**Sprint:** S4
**Dependencias Entrantes:** EP-02 (cluster K8s), EP-03 (FluxCD), EP-05 (CNPG), EP-07 (IAM + Kratos), EP-10 (NetworkPolicies).
**Dependencias Salientes:** Ninguna (cierra la Fase 1).

---

## HU-11.1 — Health Checks y Observabilidad Mínima Viable (§26)

**Como** ingeniero de operaciones,
**quiero** que cada microservicio Go implemente endpoints `/health`, `/ready`, y `/metrics`, y tener alertas básicas configuradas en Hetzner Monitoring,
**para que** Kubernetes pueda gestionar la disponibilidad de pods automáticamente y yo reciba notificaciones ante problemas de infraestructura.

**Dependencias:** EP-07 HU-07.2 (IAM Service ya tiene health checks; esta HU formaliza el estándar).

### Tareas y Subtareas

#### T-11.1.1 — Definir estándar de health checks para microservicios Go

- **ST-11.1.1.1** — Documentar el contrato obligatorio de endpoints de salud.
  - **CA:** Documento especifica:
    - `GET /health` → Liveness: `{"status":"ok","service":"<nombre>","version":"<semver>"}`.
    - `GET /ready` → Readiness: `{"status":"ok","db":"connected","<dep>":"reachable"}` o 503 con detalle.
    - `GET /metrics` → Formato Prometheus/OpenMetrics (para Fase 3).
  - **CA:** Documento incluye ejemplo de código Go de referencia.

- **ST-11.1.1.2** — Verificar que el IAM Service cumple el estándar.
  - **CA:** `GET /health` → 200 con `status`, `service`, `version`.
  - **CA:** `GET /ready` → 200 con `status`, `db`, `kratos` o 503 si hay problemas.
  - **CA:** `GET /metrics` → endpoint presente (puede estar vacío o con métricas básicas).

#### T-11.1.2 — Configurar alertas básicas en Hetzner Monitoring

- **ST-11.1.2.1** — Acceder a Hetzner Cloud Console → serenidad-prod-01 → Monitoring.
  - **CA:** Panel de monitoreo del servidor accesible.
- **ST-11.1.2.2** — Configurar alerta de CPU > 90% por más de 5 minutos.
  - **CA:** Alerta creada con umbral 90%, duración 5 min, notificación por email.
- **ST-11.1.2.3** — Configurar alerta de tráfico de red inusual.
  - **CA:** Alerta de network traffic creada.
- **ST-11.1.2.4** — Configurar alerta de disponibilidad del servidor (ping check).
  - **CA:** Alerta de server unavailability creada.
- **ST-11.1.2.5** — Verificar que las alertas envían notificaciones.
  - **CA:** Email de confirmación recibido al crear alertas (o test de notificación).

#### T-11.1.3 — Documentar consultas de estado rápidas

- **ST-11.1.3.1** — Crear documento o sección con comandos de diagnóstico rápido.
  - **CA:** Comandos documentados:
    - Estado general: `kubectl get pods -A --sort-by='.status.startTime'`.
    - Pods con problemas: `kubectl get pods -A | grep -v "Running\|Completed"`.
    - Logs en tiempo real: `kubectl logs -f deployment/<servicio> -n <ns>`.
    - Logs últimos N min: `kubectl logs deployment/<servicio> -n <ns> --since=10m`.
    - Eventos del cluster: `kubectl get events -A --sort-by='.lastTimestamp' | tail -20`.
    - Estado FluxCD: `flux get all --status-selector ready=false`.
    - Estado CNPG: `kubectl get cluster,backup,scheduledbackup -n serenidad-data`.
    - Top pods: `kubectl top pods -A --sort-by=memory`.
    - PVCs: `kubectl get pvc -A`.
  - **CA:** Documento incluye aliases recomendados (`kp`, `tp`, `fp`).

### Criterios de Aceptación de la HU-11.1

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-1 | Estándar de health checks documentado | Documento con /health, /ready, /metrics |
| CA-2 | IAM Service cumple estándar | 3 endpoints funcionales |
| CA-3 | 3 alertas Hetzner configuradas | CPU, network, availability |
| CA-4 | Comandos de diagnóstico documentados | ≥ 9 comandos con explicación |
| CA-5 | Aliases de trabajo documentados | kp, tp, fp |

### Definition of Done — HU-11.1

- [ ] Estándar de health checks documentado y aplicado al IAM Service.
- [ ] 3 alertas básicas configuradas en Hetzner Monitoring.
- [ ] Comandos de diagnóstico rápido documentados.
- [ ] Aliases de trabajo documentados.

---

## HU-11.2 — Procedimientos Operativos Día-2 (§25)

**Como** ingeniero de operaciones,
**quiero** tener procedimientos documentados y probados para el ciclo de desarrollo diario, deploy de emergencia (hotfix), gestión de nuevos secretos, rotación de claves age, y actualización de Talos Linux,
**para que** las operaciones recurrentes se ejecuten de forma consistente, segura, y sin improvisación.

**Dependencias:** EP-03 (FluxCD + SOPS), EP-07 (IAM Service para ciclo de deploy), EP-02 (Talos para actualización).

### Tareas y Subtareas

#### T-11.2.1 — Documentar ciclo de desarrollo de código Go

- **ST-11.2.1.1** — Crear documento con flujo estándar de desarrollo → producción.
  - **CA:** Pasos documentados:
    1. Código y tests locales: `cd services/iam && go test ./... -v -race`.
    2. Commit y push: desencadena pipeline CI.
    3. FluxCD detecta cambio en deployment.yaml y despliega (~1-3 min).
    4. Verificar rollout: `flux get all`, `kubectl rollout status`.
    5. Verificar logs: `kubectl logs -f`.
  - **CA:** Tiempos estimados incluidos (feedback local: ms, pipeline: ~5 min, FluxCD: ~1-3 min).

#### T-11.2.2 — Documentar procedimiento de deploy de emergencia (hotfix)

- **ST-11.2.2.1** — Crear documento con flujo de hotfix.
  - **CA:** Marcado como "SOLO PARA EMERGENCIAS".
  - **CA:** Pasos:
    1. Hacer cambio mínimo en el código.
    2. Build local de la imagen Docker con tag `hotfix-<timestamp>`.
    3. Login y push directo a GitLab Container Registry.
    4. `kubectl set image` para actualizar la imagen directamente (bypass FluxCD).
    5. Post-fix: commit normal, push, FluxCD re-sincroniza.
  - **CA:** Nota: FluxCD revertirá el hotfix manual a la versión en Git (comportamiento esperado).

#### T-11.2.3 — Documentar gestión de secretos nuevos

- **ST-11.2.3.1** — Crear documento con flujo de agregar nuevos secrets.
  - **CA:** Pasos:
    1. Cifrar con `./scripts/encrypt-secret.sh <namespace> <nombre> KEY=valor`.
    2. Commitear y pushear `infra/secrets/<nombre>.yaml`.
    3. FluxCD aplica automáticamente (~1 min).
    4. `kustomize-controller` descifra con clave age.

#### T-11.2.4 — Documentar procedimiento de rotación de clave age

- **ST-11.2.4.1** — Crear documento con flujo de rotación de clave age.
  - **CA:** Pasos:
    1. Generar nueva clave age: `age-keygen -o ~/.config/sops/age/keys-new.txt`.
    2. Extraer clave pública.
    3. Actualizar `.sops.yaml` con nueva clave pública.
    4. Re-cifrar todos los secrets: `for f in infra/secrets/*.yaml; do sops --rotate --in-place "$f"; done`.
    5. Actualizar secret `sops-age` en el cluster: `kubectl create secret generic sops-age ...`.
    6. Commitear y pushear secrets re-cifrados.
  - **CA:** Documentado como operación de seguridad ante compromiso de clave.

#### T-11.2.5 — Documentar procedimiento de actualización de Talos Linux

- **ST-11.2.5.1** — Crear documento con flujo de actualización de Talos.
  - **CA:** Pasos:
    1. Verificar versión actual: `talosctl version --nodes ${FLOATING_IP}`.
    2. Actualizar: `talosctl upgrade --nodes ${FLOATING_IP} --image ghcr.io/siderolabs/installer:<NEW_VERSION>`.
    3. El nodo se reinicia automáticamente.
    4. Verificar: `talosctl version`, `kubectl get nodes`.
  - **CA:** Nota sobre imagen oficial: `ghcr.io/siderolabs/installer` es de Siderolabs (no nuestra).

### Criterios de Aceptación de la HU-11.2

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-1 | Ciclo de desarrollo documentado | Flujo completo: local → CI → FluxCD → producción |
| CA-2 | Hotfix documentado | Procedimiento de emergencia con advertencias |
| CA-3 | Gestión de secrets documentada | Flujo: encrypt → commit → FluxCD descifra |
| CA-4 | Rotación de clave age documentada | 6 pasos de rotación completos |
| CA-5 | Actualización de Talos documentada | Verificar → upgrade → verificar |

### Definition of Done — HU-11.2

- [ ] 5 procedimientos operativos documentados.
- [ ] Cada procedimiento incluye pasos numerados, comandos exactos, y notas de seguridad.
- [ ] Procedimiento de hotfix marcado como "solo emergencias".
- [ ] Procedimiento de rotación de age incluye re-cifrado de todos los secrets.

#### T-11.2.X — Runbook de Rotación de Secrets

> **NOTA:** Este runbook se documenta en Fase 1 pero la rotación efectiva se ejecuta periódicamente post-despliegue.

**Inventario de secrets y frecuencia de rotación sugerida:**

| Secret | Ubicación | Frecuencia | Procedimiento |
|--------|-----------|------------|---------------|
| Kratos Cookie Secret | `iam-service-secrets` / SOPS | 6 meses | Generar nuevo con `openssl rand -base64 32`, cifrar con SOPS, push → FluxCD redeploy Kratos |
| Kratos Cipher Secret | `iam-service-secrets` / SOPS | 6 meses | Igual que cookie secret |
| PG Admin Password | `cnpg-serenidad-admin-creds` / SOPS | 12 meses | Generar nuevo, cifrar, push → `ALTER ROLE serenidad_admin PASSWORD 'nuevo'` |
| B2 Application Key | `cnpg-b2-credentials` / SOPS | 12 meses | Rotar en B2 console, actualizar .envrc, re-cifrar, push |
| JWT Ed25519 Keys | `iam-service-secrets` / SOPS | 12 meses | Generar nuevo par ed25519, cifrar, push → restart IAM (JWTs previos invalidos) |
| CF API Token | GitLab CI/CD variables | 12 meses | Rotar en Cloudflare dashboard, actualizar en GitLab CI vars |
| HCLOUD_TOKEN | `.envrc` + GitLab CI/CD | 12 meses | Rotar en Hetzner Cloud console, actualizar .envrc y GitLab CI vars |
| Resend API Key | `.envrc` / Kratos SMTP config | 12 meses | Rotar en Resend dashboard, actualizar .envrc, re-cifrar Kratos secret |

**Procedimiento general de rotación:**
1. Generar nuevo secret
2. Actualizar `.envrc` con el nuevo valor
3. Re-cifrar con `./scripts/encrypt-secret.sh`
4. Commit y push — FluxCD aplica automáticamente
5. Verificar que el servicio arranca correctamente post-rotación
6. Guardar backup del nuevo secret en Password Manager

---

## HU-11.3 — Troubleshooting y Recuperación ante Fallos (§27 + §25.5)

**Como** ingeniero de operaciones,
**quiero** tener una guía de troubleshooting exhaustiva para los problemas más comunes del cluster y un procedimiento de recuperación completa ante fallo del nodo,
**para que** cualquier incidencia se pueda diagnosticar y resolver en el menor tiempo posible, incluyendo la reconstrucción completa del entorno desde cero si fuera necesario.

**Dependencias:** Todos los componentes desplegados (EP-02 a EP-10).

### Tareas y Subtareas

#### T-11.3.1 — Documentar troubleshooting: nodo en NotReady

- **ST-11.3.1.1** — Crear sección de troubleshooting para nodo NotReady.
  - **CA:** Comandos documentados:
    - `kubectl describe node serenidad-prod-01 | grep -A20 "Conditions:"`.
    - `kubectl get node serenidad-prod-01 -o json | jq '.spec.taints'`.
    - Verificar CCM: `kubectl get pods -n hcloud-system`, logs.
    - Verificar etcd: `talosctl etcd status --nodes ${FLOATING_IP}`.
    - Reiniciar kubelet: `talosctl service kubelet restart --nodes ${FLOATING_IP}`.

#### T-11.3.2 — Documentar troubleshooting: FluxCD no sincroniza

- **ST-11.3.2.1** — Crear sección para problemas de FluxCD.
  - **CA:** Comandos:
    - `flux get all`.
    - Logs de controllers: `source-controller`, `kustomize-controller`, `helm-controller`.
    - Reconciliación manual: `flux reconcile source git flux-system`, `flux reconcile kustomization flux-system`.
    - Verificar deploy key en GitLab.

#### T-11.3.3 — Documentar troubleshooting: cert-manager no emite certificados

- **ST-11.3.3.1** — Crear sección para problemas de certificados.
  - **CA:** Comandos:
    - `kubectl describe certificate <nombre> -n <namespace>`.
    - `kubectl get challenges -A`.
    - `dig api.sereni.dad`.
    - Verificar pod de challenge ACME.
    - Logs de cert-manager.
    - Solución común: puerto :80 abierto en firewall Hetzner.

#### T-11.3.4 — Documentar troubleshooting: PostgreSQL no inicia

- **ST-11.3.4.1** — Crear sección para problemas de CNPG/PostgreSQL.
  - **CA:** Comandos:
    - `kubectl describe cluster serenidad-pg -n serenidad-data`.
    - Logs del pod PG.
    - PVCs: `kubectl get pvc -n serenidad-data`, `kubectl describe pvc`.
    - Verificar `local-path-provisioner`.
    - Verificar StorageClass.

#### T-11.3.5 — Documentar troubleshooting: Kratos no conecta a la DB

- **ST-11.3.5.1** — Crear sección para problemas de Kratos.
  - **CA:** Comandos:
    - Logs de Kratos filtrando `error|database|dsn`.
    - Verificar secret de credenciales.
    - Test de conectividad desde pod Kratos a PG.
    - Verificar NetworkPolicy no bloquea.

#### T-11.3.6 — Documentar troubleshooting: IAM Service retorna 500

- **ST-11.3.6.1** — Crear sección para problemas del IAM Service.
  - **CA:** Comandos:
    - Logs del IAM Service últimos 5 min.
    - Verificar secret JWT.
    - Verificar Kratos Admin API accesible desde IAM pod.
    - Verificar env vars del pod (excluyendo secrets).

#### T-11.3.7 — Documentar troubleshooting: Traefik no enruta correctamente

- **ST-11.3.7.1** — Crear sección para problemas de Traefik.
  - **CA:** Comandos:
    - `kubectl get ingressroutes -A`.
    - `kubectl get middlewares -A`.
    - Logs de Traefik DaemonSet.
    - Port-forward al dashboard Traefik: `kubectl port-forward -n traefik daemonset/traefik 9000:9000`.
    - Inspección de rutas: `curl http://localhost:9000/api/rawdata | jq '.routers | keys'`.
    - Test directo bypass Cloudflare: `curl -k --resolve api.sereni.dad:443:${FLOATING_IP} https://api.sereni.dad/...`.

#### T-11.3.8 — Documentar procedimiento de recuperación completa ante fallo del nodo

- **ST-11.3.8.1** — Crear documento de Disaster Recovery.
  - **CA:** Sección "Qué se pierde":
    - Estado del cluster (etcd).
    - Datos de PostgreSQL en disco local.
    - Secret `sops-age` del cluster.
  - **CA:** Sección "Qué se conserva":
    - Backups PG en B2 (WAL + base).
    - Secrets cifrados SOPS en Git.
    - Imágenes en `registry.gitlab.com`.
    - Toda la configuración en Git.
  - **CA:** Procedimiento paso a paso:
    1. Verificar estado del servidor en Hetzner: `hcloud server describe serenidad-prod-01`.
    2. Si parado: `hcloud server poweron`.
    3. Si corrupto: activar rescue mode, reinstalar Talos (repetir V-02).
    4. Aplicar configuración Talos con secrets originales.
    5. Bootstrap Kubernetes.
    6. Re-bootstrap FluxCD (repetir V-05).
    7. Recrear secret `sops-age` manualmente.
    8. CloudNativePG restaura PG desde B2: `kubectl cnpg restore serenidad-pg --backup <latest>`.
    9. O usar PITR para recuperar hasta un punto específico.
  - **CA:** RTO estimado: 30-60 minutos.
  - **CA:** RPO: WAL no archivado antes del fallo (~5 minutos máximo).

#### T-11.3.9 — Documentar archivos críticos y su estado en Git

- **ST-11.3.9.1** — Crear tabla de archivos críticos.
  - **CA:** Tabla incluye:
    | Archivo | ¿En Git? | ¿Cifrado? | Notas |
    |---------|----------|-----------|-------|
    | `talos/secrets.yaml` | NO | — | Backup en password manager |
    | `talos/talosconfig` | NO | — | Backup en password manager |
    | `kubeconfig` | NO | — | .gitignore |
    | `infra/secrets/.sops.yaml` | SÍ | — | Clave age pública (segura) |
    | `infra/secrets/*.yaml` | SÍ | SÍ (AES-256-GCM + age) | `sops -d` para inspeccionar |
    | `~/.config/sops/age/keys.txt` | NO | — | Backup OBLIGATORIO |
    | `.envrc` | NO | — | .gitignore |
    | `talos/patches/hetzner-cx32.yaml` | SÍ | — | Sin secretos |

### Criterios de Aceptación de la HU-11.3

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-1 | 7 secciones de troubleshooting | NotReady, FluxCD, cert-manager, PG, Kratos, IAM, Traefik |
| CA-2 | Cada sección con comandos específicos | ≥ 3 comandos de diagnóstico por sección |
| CA-3 | Disaster Recovery documentado | 9 pasos de recuperación completa |
| CA-4 | RTO y RPO documentados | RTO: 30-60 min, RPO: ~5 min |
| CA-5 | Tabla de archivos críticos | 8+ archivos con estado en Git y cifrado |
| CA-6 | Secretos críticos identificados | sops-age, talos secrets, kubeconfig, .envrc |

### Definition of Done — HU-11.3

- [ ] 7 secciones de troubleshooting con comandos específicos.
- [ ] Procedimiento de Disaster Recovery completo con 9 pasos.
- [ ] RTO y RPO estimados y documentados.
- [ ] Tabla de archivos críticos y su estado (Git, cifrado, backup).
- [ ] Toda la documentación commiteada en el repo.

---

## Resumen de Dependencias Internas EP-11

```
HU-11.1 (Health checks + alertas + comandos)
  └──► Independiente, puede ejecutarse tan pronto como EP-07 esté completa

HU-11.2 (Procedimientos operativos)
  └──► Requiere experiencia con EP-03 (FluxCD), EP-07 (IAM deploy cycle)

HU-11.3 (Troubleshooting + DR)
  └──► Requiere todos los componentes desplegados (EP-02 a EP-10)
  └──► Idealmente se documenta tras operar el sistema algunos días

Orden sugerido:
  1. HU-11.1 (estándar de health + alertas)
  2. HU-11.2 (procedimientos día-2)
  3. HU-11.3 (troubleshooting + DR)
```

---

## Verificación Final de Fase 1 — Flujo End-to-End

La Fase 1 está **COMPLETA** cuando el siguiente flujo funciona de extremo a extremo:

```
 1. Usuario abre https://app.sereni.dad en Chrome/Safari/Firefox
 2. Hace clic en "Registrarse"
 3. Ory Kratos muestra el flow de registro
 4. Usuario registra una Passkey (WebAuthn/FIDO2) con su dispositivo
 5. Kratos envía email de verificación via Resend
 6. Usuario verifica su email
 7. Usuario inicia sesión con su Passkey
 8. Kratos emite session token
 9. Frontend llama a POST /bff/api/iam/token/exchange con el session token
10. IAM Domain Service verifica la sesión con Kratos Admin API
11. IAM Domain Service crea el perfil de usuario en iam_db (si no existe)
12. IAM Domain Service inserta evento UserRegistered en outbox_events (MISMA TX)
13. IAM Domain Service firma JWT Ed25519 con claims: sub, role, tenant_id, did
14. Frontend recibe el JWT y lo almacena
15. Frontend hace GET /bff/api/iam/whoami con JWT en Authorization header
16. BFF verifica el JWT con la clave pública Ed25519
17. BFF propaga X-User-ID, X-User-Role, X-Tenant-ID al backend
18. Traefik ejecuta ForwardAuth → IAM /internal/validate-token → 200 OK
19. IAM Domain Service retorna el perfil del usuario
20. Frontend muestra el dashboard del usuario

✓ Todos los pasos completan en < 3 segundos en condiciones normales.
```
