# EP-03 — GitOps y Gestión de Configuración

**Épica:** Como equipo de infraestructura, necesitamos FluxCD v2.x operativo como motor GitOps, la estructura del repositorio definida para reconciliación declarativa, y SOPS+age configurado para cifrado de secretos, para que todo cambio a producción pase por Git como única fuente de verdad.

**Origen:** Secciones §10 (V-05), §11 (V-06), §14 (V-09) del documento de decisión.
**Prioridad:** Crítica — Bloqueante para EP-04, EP-05, EP-06, EP-07.
**Sprint:** S1–S2
**Dependencias Entrantes:** EP-02 (cluster K8s funcional con namespaces y CCM).
**Dependencias Salientes:** EP-04 (cert-manager/Traefik via FluxCD), EP-05 (CNPG via FluxCD), EP-06 (schemas commitados), EP-07 (Kratos/IAM via FluxCD).

---

## HU-03.1 — Bootstrap de FluxCD v2.x (V-05)

**Como** ingeniero de plataforma,
**quiero** tener FluxCD v2.x instalado en el cluster y sincronizando la rama `main` del monorepo de GitLab,
**para que** todo cambio en el repositorio se reconcilie automáticamente en el cluster de producción.

**Dependencia:** HU-02.4 completada (cluster K8s con CCM operativo).

### Tareas y Subtareas

#### T-03.1.1 — Pre-checks antes del bootstrap

- **ST-03.1.1.1** — Ejecutar `flux check --pre` para verificar compatibilidad del cluster.
  - **CA:** Output incluye `✓ Kubernetes 1.33.x >= 1.28.0` y `✓ prerequisites checks passed`.
- **ST-03.1.1.2** — Verificar que `GITLAB_TOKEN` está exportado en el entorno.
  - **CA:** `echo $GITLAB_TOKEN` retorna un valor no vacío.
- **ST-03.1.1.3** — Verificar que el repositorio GitLab es accesible con el token.
  - **CA:** `git ls-remote https://oauth2:${GITLAB_TOKEN}@gitlab.com/${GITLAB_USER}/${GITLAB_REPO}.git` retorna refs.

#### T-03.1.2 — Ejecutar bootstrap de FluxCD

- **ST-03.1.2.1** — Ejecutar `flux bootstrap gitlab` con los parámetros:
  - `--owner=${GITLAB_USER}`
  - `--repository=${GITLAB_REPO}`
  - `--branch=main`
  - `--path=./infra/clusters/hetzner-prod`
  - `--personal`
  - `--components-extra=image-reflector-controller,image-automation-controller`
  - **CA:** Comando completa sin error.
  - **CA:** FluxCD instala controllers en namespace `flux-system`.
  - **CA:** Deploy key creado automáticamente en el repo de GitLab.
  - **CA:** Manifests de FluxCD commiteados en `infra/clusters/hetzner-prod/flux-system/`.
  - **CA:** GitRepository y Kustomization creados apuntando al monorepo.

#### T-03.1.3 — Verificar sincronización de FluxCD

- **ST-03.1.3.1** — Verificar que todos los recursos FluxCD están Ready.
  - **CA:** `flux get all` → `gitrepository True`, `kustomization True`.
- **ST-03.1.3.2** — Verificar pods de FluxCD en namespace `flux-system`.
  - **CA:** Pods Running: `source-controller`, `kustomize-controller`, `helm-controller`, `notification-controller`, `image-reflector-controller`, `image-automation-controller`.
  - **CA:** 6 pods en estado Running.
- **ST-03.1.3.3** — Verificar que el GitRepository apunta al monorepo correcto.
  - **CA:** `flux get source git flux-system` → URL `gitlab.com/${GITLAB_USER}/${GITLAB_REPO}`, branch `main`.
- **ST-03.1.3.4** — Verificar que la Kustomization aplica el path correcto.
  - **CA:** `flux get kustomization flux-system` → path `./infra/clusters/hetzner-prod`, status `Applied revision: main@sha1:...`.

#### T-03.1.4 — Configurar SOPS+age para decryption en FluxCD

- **ST-03.1.4.1** — Generar clave age con `age-keygen`.
  - **CA:** Archivo `~/.config/sops/age/keys.txt` generado.
  - **CA:** Clave pública extraída en `AGE_PUBLIC_KEY`.
- **ST-03.1.4.2** — Backup de la clave PRIVADA age en password manager.
  - **CA:** Backup verificado y documentado como CRÍTICO.
  - **CA:** Nota: "Sin esta clave, los secrets cifrados en Git son irrecuperables."
- **ST-03.1.4.3** — Crear Secret de Kubernetes `sops-age` en `flux-system`.
  - **CA:** `kubectl get secret sops-age -n flux-system` → existe con key `age.agekey`.
- **ST-03.1.4.4** — Parchear la Kustomization `flux-system` para habilitar decryption SOPS.
  - **CA:** `kubectl get kustomization flux-system -n flux-system -o yaml | grep -A5 decryption` → `provider: sops`, `secretRef.name: sops-age`.
- **ST-03.1.4.5** — Crear archivo `.sops.yaml` en `infra/secrets/`.
  - **CA:** Archivo contiene: `path_regex: infra/secrets/.*\.yaml$`, `encrypted_regex: ^(data|stringData)$`, `age: ${AGE_PUBLIC_KEY}`.
- **ST-03.1.4.6** — Commitear y pushear `.sops.yaml`.
  - **CA:** `git log --oneline -1` → `feat: add SOPS age encryption rules for infra/secrets/`.
  - **CA:** Archivo visible en GitLab en `infra/secrets/.sops.yaml`.

### Criterios de Aceptación de la HU-03.1

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-1 | FluxCD v2.x sincronizando | `flux get all` → todos Ready |
| CA-2 | 6 controllers Running | `kubectl get pods -n flux-system` → 6 pods Running |
| CA-3 | GitRepository apunta a GitLab | `flux get source git` → URL correcta, branch main |
| CA-4 | SOPS+age configurado | `kubectl get secret sops-age -n flux-system` → existe |
| CA-5 | Decryption habilitado en Kustomization | `spec.decryption.provider: sops` presente |
| CA-6 | `.sops.yaml` commiteado | Archivo en `infra/secrets/.sops.yaml` en el repo |
| CA-7 | Clave age respaldada | Backup en password manager verificado |

### Definition of Done — HU-03.1

- [ ] FluxCD v2.x bootstrap completado y reconciliando.
- [ ] 6 controllers en Running.
- [ ] SOPS+age configurado para decryption automático.
- [ ] `.sops.yaml` commiteado en el repo.
- [ ] Clave age privada respaldada en password manager.
- [ ] Deploy key activo en GitLab para FluxCD.

---

## HU-03.2 — Estructura GitOps del Repositorio (V-06)

**Como** ingeniero de plataforma,
**quiero** tener la estructura de directorios GitOps completamente definida con el punto de entrada `kustomization.yaml` del cluster,
**para que** FluxCD pueda reconciliar toda la infraestructura y aplicaciones declarativamente en el orden de dependencia correcto.

**Dependencia:** HU-03.1 completada (FluxCD bootstrap exitoso).

### Tareas y Subtareas

#### T-03.2.1 — Crear el punto de entrada Kustomization del cluster

- **ST-03.2.1.1** — Crear archivo `infra/clusters/hetzner-prod/kustomization.yaml`.
  - **CA:** Archivo contiene `apiVersion: kustomize.config.k8s.io/v1beta1` y `kind: Kustomization`.
  - **CA:** `resources` incluye:
    - `flux-system/` (auto-generado)
    - `../../infrastructure/hcloud-ccm/`
    - `../../infrastructure/cert-manager/`
    - `../../infrastructure/traefik/`
    - `../../infrastructure/cnpg/`
    - `../../apps/kratos/`
    - `../../apps/iam-service/`
  - **CA:** Recursos de Fase 2 y 3 están comentados.
- **ST-03.2.1.2** — Commitear y pushear el kustomization.yaml.
  - **CA:** FluxCD detecta el cambio y reconcilia sin error.
  - **CA:** `flux get kustomization flux-system` → Applied revision actualizada.

#### T-03.2.2 — Crear estructura de directorios de infraestructura compartida

- **ST-03.2.2.1** — Crear directorio y placeholder `infra/infrastructure/hcloud-ccm/`.
  - **CA:** Directorio existe para que FluxCD adopte la gestión del CCM.
- **ST-03.2.2.2** — Crear directorio `infra/infrastructure/cert-manager/` con archivos placeholder.
  - **CA:** Directorio listo para recibir `helmrepository.yaml`, `helmrelease.yaml`, `clusterissuers.yaml`.
- **ST-03.2.2.3** — Crear directorio `infra/infrastructure/traefik/` con archivos placeholder.
  - **CA:** Directorio listo para recibir `helmrepository.yaml`, `helmrelease.yaml`, `middlewares.yaml`.
- **ST-03.2.2.4** — Crear directorio `infra/infrastructure/cnpg/` con archivos placeholder.
  - **CA:** Directorio listo para recibir `helmrepository.yaml`, `helmrelease.yaml`, `cluster.yaml`.
- **ST-03.2.2.5** — Crear directorio `infra/infrastructure/network-policies/`.
  - **CA:** Directorio listo para recibir NetworkPolicies (V-19).

#### T-03.2.3 — Crear estructura de directorios de aplicaciones

- **ST-03.2.3.1** — Crear directorio `infra/apps/kratos/` con archivos placeholder.
  - **CA:** Directorio listo para recibir `helmrelease.yaml`, `configmap-kratos.yaml`, `ingressroute.yaml`, subdirectorio `schemas/`.
- **ST-03.2.3.2** — Crear directorio `infra/apps/iam-service/` con archivos placeholder.
  - **CA:** Directorio listo para recibir `deployment.yaml`, `service.yaml`, `ingressroute.yaml`.

#### T-03.2.4 — Crear directorio de secrets cifrados

- **ST-03.2.4.1** — Verificar que `infra/secrets/` existe y contiene `.sops.yaml`.
  - **CA:** Directorio y archivo de reglas SOPS presentes.
- **ST-03.2.4.2** — Documentar los secrets que se crearán:
  - `iam-service-secrets.yaml` — JWT Ed25519 keys.
  - `kratos-secrets.yaml` — Cookie y cipher secrets.
  - `cnpg-serenidad-admin-creds.yaml` — Credenciales admin PG.
  - `cnpg-b2-credentials.yaml` — Credenciales Backblaze B2.
  - `gitlab-registry-secret.yaml` — dockerconfigjson para registry.
  - **CA:** Lista documentada y commiteada como README en `infra/secrets/`.

#### T-03.2.5 — Commitear estructura completa y verificar reconciliación

- **ST-03.2.5.1** — Commitear toda la estructura GitOps.
  - **CA:** `git log --oneline -1` → commit descriptivo.
- **ST-03.2.5.2** — Forzar reconciliación de FluxCD.
  - **CA:** `flux reconcile kustomization flux-system --with-source` → sin errores.
- **ST-03.2.5.3** — Verificar que FluxCD no tiene errores de directorios faltantes.
  - **CA:** `flux get kustomizations -A` → todos Ready (puede haber warnings por componentes pendientes, pero sin errores fatales).

### Criterios de Aceptación de la HU-03.2

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-1 | kustomization.yaml del cluster creado | Archivo en `infra/clusters/hetzner-prod/kustomization.yaml` con todas las referencias |
| CA-2 | Estructura de infra completa | Directorios cert-manager, traefik, cnpg, hcloud-ccm, network-policies existen |
| CA-3 | Estructura de apps completa | Directorios kratos, iam-service existen |
| CA-4 | Directorio de secrets con .sops.yaml | `infra/secrets/.sops.yaml` presente |
| CA-5 | FluxCD reconcilia sin errores fatales | `flux get all` → sin status `False` por errores de estructura |

### Definition of Done — HU-03.2

- [ ] Punto de entrada Kustomization creado con todas las referencias.
- [ ] Estructura completa de directorios para infra y apps.
- [ ] FluxCD reconcilia la nueva estructura sin errores fatales.
- [ ] Commit de estructura en `main`.

---

## HU-03.3 — SOPS + age: Cifrado de Secretos GitOps (V-09)

**Como** ingeniero de plataforma,
**quiero** tener un flujo operativo completo para cifrar, almacenar en Git, y descifrar automáticamente secretos de Kubernetes via SOPS+age,
**para que** los secrets sensibles (JWT keys, contraseñas de DB, tokens de API) estén seguros en el repositorio y se apliquen automáticamente al cluster.

**Dependencia:** HU-03.1 completada (SOPS+age configurado en FluxCD). Este HU operacionaliza lo configurado en HU-03.1.

### Tareas y Subtareas

#### T-03.3.1 — Crear script auxiliar de cifrado

- **ST-03.3.1.1** — Crear archivo `scripts/encrypt-secret.sh`.
  - **CA:** Script acepta parámetros: `<namespace> <secret-name> <key>=<value> [<key>=<value> ...]`.
  - **CA:** Script genera un Kubernetes Secret cifrado con SOPS+age.
  - **CA:** Output se guarda en `infra/secrets/${SECRET_NAME}.yaml`.
  - **CA:** Script verifica existencia de `.sops.yaml` antes de ejecutar.
- **ST-03.3.1.2** — Hacer ejecutable el script (`chmod +x`).
  - **CA:** `ls -la scripts/encrypt-secret.sh` → permisos de ejecución.
- **ST-03.3.1.3** — Commitear el script.
  - **CA:** Script en `main` en GitLab.

#### T-03.3.2 — Verificar flujo completo de cifrado y descifrado

- **ST-03.3.2.1** — Crear un secret de prueba cifrado con el script.
  - **CA:** `./scripts/encrypt-secret.sh default test-sops-secret TEST_KEY=test_value` genera `infra/secrets/test-sops-secret.yaml`.
  - **CA:** El archivo contiene datos cifrados (no texto plano en `data` o `stringData`).
- **ST-03.3.2.2** — Verificar descifrado local con `sops -d`.
  - **CA:** `sops -d infra/secrets/test-sops-secret.yaml` muestra el secret en texto plano con `TEST_KEY`.
- **ST-03.3.2.3** — Commitear, pushear, y verificar que FluxCD descifra y aplica.
  - **CA:** `flux reconcile kustomization flux-system --with-source` → sin errores de decryption.
  - **CA:** `kubectl get secret test-sops-secret -n default` → existe en el cluster.
  - **CA:** `kubectl get secret test-sops-secret -n default -o jsonpath='{.data.TEST_KEY}' | base64 -d` → `test_value`.
- **ST-03.3.2.4** — Limpiar secret de prueba (eliminar del repo y del cluster).
  - **CA:** Secret de prueba eliminado de Git y del cluster.

#### T-03.3.3 — Configurar Kustomizations de componentes para referenciar secrets

- **ST-03.3.3.1** — Documentar el patrón de inclusión de secrets desde cada componente.
  - **CA:** Documentación muestra que cada `kustomization.yaml` de componente referencia sus secrets desde `../../secrets/` o `../../../secrets/`.
- **ST-03.3.3.2** — Preparar plantilla de `kustomization.yaml` para `infra/infrastructure/cnpg/`.
  - **CA:** Plantilla incluye referencia a `cnpg-serenidad-admin-creds.yaml` y `cnpg-b2-credentials.yaml`.
- **ST-03.3.3.3** — Preparar plantilla de `kustomization.yaml` para `infra/apps/iam-service/`.
  - **CA:** Plantilla incluye referencia a `iam-service-secrets.yaml` y `gitlab-registry-secret.yaml`.

#### T-03.3.4 — Verificar logs de FluxCD para decryption

- **ST-03.3.4.1** — Verificar logs de `kustomize-controller` para errores de SOPS.
  - **CA:** `kubectl logs deployment/kustomize-controller -n flux-system | grep -i "sops\|decrypt\|age"` → sin errores.
- **ST-03.3.4.2** — Documentar troubleshooting de decryption.
  - **CA:** Pasos documentados: verificar secret `sops-age`, verificar `.sops.yaml`, verificar clave age.

### Criterios de Aceptación de la HU-03.3

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-1 | Script encrypt-secret.sh funcional | Genera YAML cifrado correctamente |
| CA-2 | Descifrado local exitoso | `sops -d` muestra texto plano |
| CA-3 | FluxCD descifra automáticamente | Secret aplicado al cluster tras push |
| CA-4 | Sin secrets en texto plano en Git | `grep -r "stringData" infra/secrets/` → solo datos cifrados |
| CA-5 | Patrón de referencia documentado | Cada componente sabe cómo referenciar sus secrets |

### Definition of Done — HU-03.3

- [ ] Script `encrypt-secret.sh` creado, probado, y commiteado.
- [ ] Flujo completo verificado: cifrar → commitear → FluxCD descifra → secret en cluster.
- [ ] Sin secrets en texto plano en ningún commit de Git.
- [ ] Patrones de referencia documentados para componentes.
- [ ] Troubleshooting de decryption documentado.

---

## Resumen de Dependencias Internas EP-03

```
HU-03.1 (FluxCD Bootstrap + SOPS config)
  ├──► HU-03.2 (Estructura GitOps)
  └──► HU-03.3 (SOPS operacionalización)
        └──► (Dependencia externa: EP-05, EP-07 usan secrets cifrados)

HU-03.2 es prerequisito para que FluxCD pueda reconciliar componentes de EP-04, EP-05, EP-07.
HU-03.3 es prerequisito para que los secrets de EP-05 (PG creds), EP-07 (JWT keys, Kratos secrets) se puedan crear.
```
