# EP-05 — Base de Datos y Backups

**Épica:** Como equipo de infraestructura, necesitamos CloudNativePG v1.x desplegado con PostgreSQL 17.4, las 6 databases creadas con extensiones y RLS, y backups continuos (WAL + base diario) a Backblaze B2, para disponer de la capa de persistencia con PITR disponible desde el primer día.

**Origen:** Secciones §15 (V-10) y §22 (V-17) del documento de decisión.
**Prioridad:** Crítica — Bloqueante para EP-07 (Kratos y IAM usan PostgreSQL).
**Sprint:** S2
**Dependencias Entrantes:** EP-03 (FluxCD + SOPS para secrets cifrados de credenciales PG y B2).
**Dependencias Salientes:** EP-07 (Kratos depende de `kratos_db`, IAM de `iam_db`).

---

## HU-05.1 — CloudNativePG v1.x + PostgreSQL 17.4 (V-10)

**Como** ingeniero de plataforma,
**quiero** tener el operador CloudNativePG desplegado y un cluster PostgreSQL 17.4 single-instance con 6 databases aisladas, extensiones habilitadas y parámetros de producción calibrados para el CX32,
**para que** todos los servicios de serenidad dispongan de bases de datos seguras, performantes y con aislamiento por dominio.

**Dependencia:** HU-03.1 (FluxCD), HU-03.3 (SOPS+age para secrets de credenciales PG).

### Tareas y Subtareas

#### T-05.1.1 — Crear HelmRepository de CloudNativePG

- **ST-05.1.1.1** — Crear archivo `infra/infrastructure/cnpg/helmrepository.yaml`.
  - **CA:** URL: `https://cloudnative-pg.github.io/charts`.
  - **CA:** Namespace: `flux-system`, intervalo: `1h`.
- **ST-05.1.1.2** — Commitear y pushear.
  - **CA:** `flux get source helm cnpg` → `True`.

#### T-05.1.2 — Crear HelmRelease del operador CloudNativePG

- **ST-05.1.2.1** — Crear archivo `infra/infrastructure/cnpg/helmrelease.yaml`.
  - **CA:** Chart: `cloudnative-pg`, version: `>=0.22.0`.
  - **CA:** Namespace: `cnpg-system`.
  - **CA:** Monitoring: `podMonitorEnabled: false` (Fase 3).
  - **CA:** Resources: requests `cpu: 20m, memory: 64Mi`, limits `cpu: 200m, memory: 128Mi`.
- **ST-05.1.2.2** — Commitear y pushear.
  - **CA:** FluxCD reconcilia el HelmRelease.
- **ST-05.1.2.3** — Verificar despliegue del operador.
  - **CA:** `kubectl get pods -n cnpg-system` → operator pod en estado Running.
  - **CA:** `kubectl get crd | grep cnpg` → CRDs de Cluster, Backup, ScheduledBackup presentes.

#### T-05.1.3 — Crear Secret de credenciales admin de PostgreSQL

- **ST-05.1.3.1** — Generar contraseña segura con `openssl rand -base64 32`.
  - **CA:** Contraseña generada con al menos 32 caracteres.
- **ST-05.1.3.2** — Cifrar secret con el script `encrypt-secret.sh`.
  - **CA:** `./scripts/encrypt-secret.sh serenidad-data cnpg-serenidad-admin-creds username=serenidad_admin password=${PG_ADMIN_PASSWORD}`.
  - **CA:** Archivo `infra/secrets/cnpg-serenidad-admin-creds.yaml` generado y cifrado.
- **ST-05.1.3.3** — Backup de la contraseña en password manager.
  - **CA:** Contraseña respaldada con label `PG_ADMIN_PASSWORD`.
- **ST-05.1.3.4** — Commitear y pushear el secret cifrado.
  - **CA:** `git log --oneline -1` → `feat: add cnpg admin credentials encrypted with SOPS`.
  - **CA:** FluxCD descifra y aplica el secret: `kubectl get secret cnpg-serenidad-admin-creds -n serenidad-data` → existe.

#### T-05.1.4 — Crear Cluster CRD de PostgreSQL 17.4

- **ST-05.1.4.1** — Crear archivo `infra/infrastructure/cnpg/cluster.yaml`.
  - **CA:** `imageName: ghcr.io/cloudnative-pg/postgresql:17.4`.
  - **CA:** `instances: 1` (single-node Fase 1).
  - **CA:** Parámetros de PostgreSQL calibrados para CX32 (8 GB RAM):
    - `shared_buffers: "512MB"` (~25% RAM para PG).
    - `effective_cache_size: "1536MB"` (~75% RAM estimación planner).
    - `maintenance_work_mem: "128MB"`.
    - `work_mem: "16MB"`.
    - `max_connections: "100"`.
    - `wal_buffers: "16MB"`.
    - `checkpoint_completion_target: "0.9"`.
    - `random_page_cost: "1.1"` (SSD NVMe).
    - `effective_io_concurrency: "200"` (SSD NVMe).
    - `min_wal_size: "256MB"`, `max_wal_size: "2GB"`.
  - **CA:** Logging configurado:
    - `log_min_duration_statement: "1000"` (queries >1s).
    - `log_checkpoints: "on"`, `log_lock_waits: "on"`.
  - **CA:** SSL habilitado.
  - **CA:** `shared_preload_libraries: ["pg_stat_statements"]`.
- **ST-05.1.4.2** — Configurar bootstrap con `postInitSQL` para crear las 6 databases:
  - `iam_db` con OWNER `serenidad_admin`, encoding UTF8, collate C.
  - `kratos_db` con OWNER `serenidad_admin`, encoding UTF8, collate C.
  - `openfga_db` con OWNER `serenidad_admin`, encoding UTF8, collate C.
  - `scheduling_db` con OWNER `serenidad_admin`, encoding UTF8, collate C.
  - `clinical_db` con OWNER `serenidad_admin`, encoding UTF8, collate C.
  - `billing_db` con OWNER `serenidad_admin`, encoding UTF8, collate C.
  - **CA:** 6 comandos `CREATE DATABASE` en `postInitSQL`.
- **ST-05.1.4.3** — Instalar extensiones en databases relevantes **post-bootstrap** (T-05.1.6).
  - `iam_db`: `pg_stat_statements`, `btree_gist`.
  - `scheduling_db`: `pg_stat_statements`, `btree_gist`.
  - `clinical_db`: `pg_stat_statements`.
  - `billing_db`: `pg_stat_statements`.
  - **LIMITACION:** `postInitSQL` de CNPG solo ejecuta SQL puro contra la DB del bootstrap (`postgres`). No soporta metacomandos de `psql` como `\c <db>`. Por tanto, las extensiones en cada DB individual se instalan manualmente vía `kubectl exec` en T-05.1.6, NO en `postInitSQL`.
  - **CA:** Extensiones instaladas y verificadas con `\dx` en cada base de datos tras T-05.1.6.
- **ST-05.1.4.4** — Configurar recursos del pod PostgreSQL.
  - **CA:** Requests: `memory: "256Mi"`, `cpu: "100m"`.
  - **CA:** Limits: `memory: "512Mi"`, `cpu: "1000m"`.
- **ST-05.1.4.5** — Configurar storage.
  - **CA:** `size: 20Gi`, `storageClass: local-path`.
- **ST-05.1.4.6** — Configurar backup a Backblaze B2 (placeholder — la referencia al secret `cnpg-b2-credentials` se activa en HU-05.2 T-05.2.1).
  - **CA:** `barmanObjectStore` configurado con:
    - `destinationPath: "s3://serenidad-pg-backups/wal"`.
    - `endpointURL: "https://s3.us-west-004.backblazeb2.com"` (ajustar por región).
    - `s3Credentials` referenciando secret `cnpg-b2-credentials`.
    - WAL compression gzip, maxParallel 2.
    - Data compression gzip, jobs 2.
  - **CA:** `retentionPolicy: "30d"`.
  - **NOTA DE SECUENCIACIÓN:** El secret `cnpg-b2-credentials` no existe aún en este punto. La línea en `kustomization.yaml` que lo referencia debe quedar COMENTADA hasta que HU-05.2 T-05.2.1 lo cree y lo habilite. Sin esto, FluxCD fallará al intentar reconciliar un secret inexistente.
- **ST-05.1.4.7** — Monitoring deshabilitado (Fase 3).
  - **CA:** `enablePodMonitor: false`.

#### T-05.1.5 — Crear kustomization.yaml del componente CNPG

- **ST-05.1.5.1** — Crear `infra/infrastructure/cnpg/kustomization.yaml`.
  - **CA:** Resources: `helmrepository.yaml`, `helmrelease.yaml`, `cluster.yaml`.
  - **CA:** Referencia a secrets: `../../../secrets/cnpg-serenidad-admin-creds.yaml`, `../../../secrets/cnpg-b2-credentials.yaml`.
- **ST-05.1.5.2** — Commitear todo y pushear.
  - **CA:** FluxCD reconcilia.

#### T-05.1.6 — Verificar cluster PostgreSQL

- **ST-05.1.6.1** — Esperar a que el Cluster CRD esté Ready (~3-5 minutos).
  - **CA:** `kubectl wait cluster serenidad-pg --for=condition=Ready -n serenidad-data --timeout=300s` → condición cumplida.
- **ST-05.1.6.2** — Verificar estado del cluster.
  - **CA:** `kubectl get cluster serenidad-pg -n serenidad-data` → `INSTANCES: 1, READY: 1, STATUS: Cluster in healthy state`.
- **ST-05.1.6.3** — Verificar secrets generados por CloudNativePG.
  - **CA:** `kubectl get secrets -n serenidad-data | grep cnpg` → secrets `serenidad-pg-app`, `serenidad-pg-superuser`, `serenidad-pg-ca`, `serenidad-pg-replication`.
- **ST-05.1.6.4** — Verificar que las 6 databases existen.
  - **CA:** `kubectl exec -it serenidad-pg-1 -n serenidad-data -- psql -U serenidad_admin -c "\l"` → lista `iam_db`, `kratos_db`, `openfga_db`, `scheduling_db`, `clinical_db`, `billing_db`.
- **ST-05.1.6.5** — Verificar extensiones instaladas.
  - **CA:** `psql -U serenidad_admin -d iam_db -c "\dx"` → `btree_gist`, `pg_stat_statements`.
  - **CA:** `psql -d scheduling_db -c "\dx"` → `btree_gist`, `pg_stat_statements`.
  - **CA:** `psql -d clinical_db -c "\dx"` → `pg_stat_statements`.
  - **CA:** `psql -d billing_db -c "\dx"` → `pg_stat_statements`.
- **ST-05.1.6.6** — Verificar UUIDv7 funcional en PG17.
  - **CA:** `psql -c "SELECT gen_random_uuid()::text"` → retorna UUID.
- **ST-05.1.6.7** — Verificar parámetros de PostgreSQL aplicados.
  - **CA:** `psql -c "SHOW shared_buffers"` → `512MB`.
  - **CA:** `psql -c "SHOW max_connections"` → `100`.
  - **CA:** `psql -c "SHOW random_page_cost"` → `1.1`.
- **ST-05.1.6.8** — Verificar SSL habilitado.
  - **CA:** `psql -c "SHOW ssl"` → `on`.

### Criterios de Aceptación de la HU-05.1

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-1 | Operador CNPG Running | `kubectl get pods -n cnpg-system` → operator Running |
| CA-2 | Cluster PG Healthy | `kubectl get cluster serenidad-pg -n serenidad-data` → Healthy |
| CA-3 | 6 databases creadas | `\l` lista las 6 databases |
| CA-4 | Extensiones instaladas | `\dx` en cada DB muestra extensiones correctas |
| CA-5 | Parámetros calibrados para CX32 | `SHOW shared_buffers` → 512MB |
| CA-6 | UUIDv7 funcional | `SELECT gen_random_uuid()` → UUID válido |
| CA-7 | SSL habilitado | `SHOW ssl` → on |
| CA-8 | Credenciales cifradas en Git | Secret SOPS en `infra/secrets/` |
| CA-9 | Storage 20Gi con local-path | `kubectl get pvc -n serenidad-data` → Bound, 20Gi |

### Definition of Done — HU-05.1

- [ ] Operador CloudNativePG desplegado y Running.
- [ ] Cluster PostgreSQL 17.4 en estado Healthy.
- [ ] 6 databases con extensiones y encoding correctos.
- [ ] Parámetros de PG calibrados para CX32.
- [ ] SSL habilitado.
- [ ] Credenciales cifradas con SOPS en Git.
- [ ] Contraseña admin respaldada en password manager.

---

## HU-05.2 — Backups: CloudNativePG ScheduledBackup → Backblaze B2 (V-17)

**Como** ingeniero de plataforma,
**quiero** tener WAL archiving continuo y backup base diario configurados hacia Backblaze B2,
**para que** disponga de Point-in-Time Recovery (PITR) ante cualquier pérdida de datos, con RPO cercano a cero.

**Dependencia:** HU-05.1 completada (cluster PG operativo con configuración de backup referenciando secrets B2).

### Tareas y Subtareas

#### T-05.2.1 — Crear Secret de credenciales de Backblaze B2

- **ST-05.2.1.1** — Cifrar credenciales B2 con el script `encrypt-secret.sh`.
  - **CA:** `./scripts/encrypt-secret.sh serenidad-data cnpg-b2-credentials B2_KEY_ID=${B2_KEY_ID} B2_APPLICATION_KEY=${B2_APPLICATION_KEY}`.
  - **CA:** Archivo `infra/secrets/cnpg-b2-credentials.yaml` generado y cifrado.
- **ST-05.2.1.2** — Commitear y pushear.
  - **CA:** `git log --oneline -1` → `feat: add backblaze B2 credentials encrypted with SOPS for WAL archiving`.
  - **CA:** FluxCD descifra y aplica: `kubectl get secret cnpg-b2-credentials -n serenidad-data` → existe.

#### T-05.2.2 — Verificar WAL archiving continuo

- **ST-05.2.2.1** — Verificar que el cluster PG inicia WAL archiving tras aplicarse el secret B2.
  - **CA:** `kubectl logs serenidad-pg-1 -n serenidad-data | grep -i "wal\|archive\|barman"` → mensajes de archiving exitoso.
  - **CA:** Sin mensajes de error de credenciales o conectividad a B2.
- **ST-05.2.2.2** — Verificar que WAL segments se archivan en B2.
  - **CA:** `b2 ls serenidad-pg-backups` → directorio `wal/wals/` con archivos.
  - **CA:** O via API de Backblaze/S3: archivos WAL presentes en el bucket.

#### T-05.2.3 — Crear ScheduledBackup CRD para backup base diario

- **ST-05.2.3.1** — Crear archivo `infra/infrastructure/cnpg/scheduled-backup.yaml`.
  - **CA:** `schedule: "0 2 * * *"` (02:00 UTC diario).
  - **CA:** `backupOwnerReference: self`.
  - **CA:** `cluster.name: serenidad-pg`.
  - **CA:** `immediate: true` (ejecutar un backup al crear el CRD).
- **ST-05.2.3.2** — Agregar al `kustomization.yaml` del componente CNPG.
  - **CA:** Resource `scheduled-backup.yaml` incluido.
- **ST-05.2.3.3** — Commitear y pushear.
  - **CA:** FluxCD aplica el ScheduledBackup.

#### T-05.2.4 — Verificar backup base inmediato

- **ST-05.2.4.1** — Verificar que el backup inmediato se ejecutó.
  - **CA:** `kubectl get backup -n serenidad-data` → backup con STATUS `Completed`.
- **ST-05.2.4.2** — Verificar detalles del backup.
  - **CA:** `kubectl describe backup <nombre> -n serenidad-data` → `Destination: s3://serenidad-pg-backups/wal/base/...`.
- **ST-05.2.4.3** — Verificar archivos en Backblaze B2.
  - **CA:** `b2 ls serenidad-pg-backups` → directorio `wal/base/` con archivos de backup.
- **ST-05.2.4.4** — Verificar PITR teórico.
  - **CA:** Diferencia entre `created_at` del backup y `NOW()` es conocida.
  - **CA:** WAL segments continuos disponibles desde el backup base hasta el momento actual.

#### T-05.2.5 — Verificar retención de backups

- **ST-05.2.5.1** — Verificar política de retención configurada.
  - **CA:** `retentionPolicy: "30d"` en el Cluster CRD (backups base > 30 días se eliminan).
  - **CA:** WAL retention implícita: 90 días (configuración por defecto de barman).

### Criterios de Aceptación de la HU-05.2

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-1 | Credenciales B2 cifradas en Git | `infra/secrets/cnpg-b2-credentials.yaml` cifrado con SOPS |
| CA-2 | WAL archiving funcionando | Logs de PG muestran archiving exitoso, sin errores |
| CA-3 | Backup base ejecutado con éxito | `kubectl get backup -n serenidad-data` → Completed |
| CA-4 | Archivos en B2 | `b2 ls serenidad-pg-backups` → WAL y base backup presentes |
| CA-5 | ScheduledBackup 02:00 UTC diario | `kubectl get scheduledbackup -n serenidad-data` → schedule correcto |
| CA-6 | PITR teóricamente disponible | WAL continuo + backup base = ventana de recuperación |
| CA-7 | Retención 30d configurada | `retentionPolicy: "30d"` en Cluster CRD |

### Definition of Done — HU-05.2

- [ ] Credenciales B2 cifradas con SOPS y aplicadas al cluster.
- [ ] WAL archiving continuo a B2 funcionando sin errores.
- [ ] Backup base inmediato completado y archivos en B2.
- [ ] ScheduledBackup diario a las 02:00 UTC configurado.
- [ ] Retención de 30 días configurada.
- [ ] PITR disponible desde el momento del primer backup.

---

## Resumen de Dependencias Internas EP-05

```
HU-05.1 (Operador CNPG + Cluster PG17.4 + 6 DBs)
  └──► HU-05.2 (Secret B2 + WAL archiving + ScheduledBackup)

HU-05.1 requiere:
  - EP-03 HU-03.1 (FluxCD para HelmRelease)
  - EP-03 HU-03.3 (SOPS para cifrar credenciales PG)

HU-05.2 requiere:
  - HU-05.1 (cluster PG operativo)
  - EP-01 HU-01.1 T-01.1.4 (cuenta Backblaze B2 con bucket y Application Key)
```
