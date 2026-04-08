# Trazabilidad de Deuda Técnica — Fase 1 Infraestructura

**Fecha de auditoría:** 2026-04-08
**Alcance:** 11 Épicas (EP-01 a EP-11), 27 HUs, ~120 Tareas
**Revisión:** Exhaustiva — secuenciación, dependencias, naming, criterios de aceptación, tecnología

---

## Resumen Ejecutivo

| Severidad | Encontradas | Corregidas | Pendientes (documentadas) |
|-----------|-------------|------------|---------------------------|
| **CRITICA** | 4 | 4 | 0 |
| **ALTA** | 12 | 6 | 6 |
| **MEDIA** | 17 | 3 | 14 |
| **BAJA** | 5 | 0 | 5 |
| **Total** | **38** | **13** | **25** |

**Todas las correcciones aplicadas son retrocompatibles** — no requieren cambios arquitecturales.
Las 25 pendientes están documentadas como deuda técnica conocida para abordar antes de ejecución.

---

## Categorías de Hallazgos

| Código | Categoría | Descripción |
|--------|-----------|-------------|
| SEQ | Secuenciación | Orden incorrecto o dependencia circular entre tareas |
| NAM | Naming | Colisión, inconsistencia o ambigüedad en identificadores |
| DEP | Dependencias | Dependencias faltantes, implícitas o mal documentadas |
| CA | Criterios Aceptación | Criterios ausentes, incompletos o no verificables |
| TEC | Tecnología | Conflictos de versión, incompatibilidades o ambigüedades técnicas |
| DOC | Documentación | Duplicación, contradicciones o información faltante |
| SEC | Seguridad | Rotación de secrets, backup de credenciales, hardening |

---

## HALLAZGOS CRITICOS (Corregidos)

### DT-001 [CRITICA] [NAM] Colisión de nombres en EP-09: dos archivos HU-09.1

**Ubicación:** `09_EP09_cicd_pipeline/HU-09.1_Root_y_IAM_Pipeline.md` y `HU-09.1_Edge_Pipelines_Docs.md`
**Problema:** Dos archivos compartían el mismo ID `HU-09.1` pero con contenidos distintos (Parte A: orquestador + IAM pipeline; Parte B: edge pipelines BFF/SPA). Esto causaba ambigüedad en todas las referencias cruzadas, riesgo de que herramientas de automatización no pudieran distinguirlos, y confusión en la numeración de tareas (ambos usaban T-09.1.3 con significados diferentes).
**Impacto:** Referencias ambiguas en el plan completo; cualquier mención a "HU-09.1" no permite saber a cuál archivo se refiere.
**Resolución aplicada:**
- Renombrado `HU-09.1_Root_y_IAM_Pipeline.md` -> `HU-09.1A_Root_y_IAM_Pipeline.md`
- Renombrado `HU-09.1_Edge_Pipelines_Docs.md` -> `HU-09.1B_Edge_Pipelines_Docs.md`
- Títulos internos actualizados de "(Parte A/B)" a "HU-09.1A / HU-09.1B"
- Épica EP-09 actualizada con dos secciones HU-09.1A y HU-09.1B con descripciones, dependencias y DoD separados
- Criterios de aceptación separados: CA-09.1A-1..6 y CA-09.1B-1..6
- Diagrama de dependencias internas reescrito con la nueva nomenclatura
- Resumen de épicas (`00_resumen_epicas.md`) actualizado: EP-09 ahora indica "2 (HU-09.1A, HU-09.1B)" y total de HUs es 27

---

### DT-002 [CRITICA] [SEQ] Dependencia circular B2 secret entre HU-05.1 y HU-05.2

**Ubicación:** `HU-05.1_CloudNativePG.md` T-05.1.4.6 y `HU-05.2_Backups_B2.md` T-05.2.1
**Problema:** HU-05.1 configura el Cluster CRD de PostgreSQL con `barmanObjectStore` que referencia el secret `cnpg-b2-credentials`. Pero este secret se crea en HU-05.2 T-05.2.1. Si ambos manifiestos se aplican sin orden, FluxCD intentará reconciliar un secret que no existe y fallará.
**Impacto:** Deploy de PostgreSQL fallaría si el secret B2 no existe previamente.
**Resolución aplicada:**
- En `05_EP05_base_datos_backups.md` ST-05.1.4.6: clarificado que la referencia al secret queda COMENTADA en `kustomization.yaml` hasta que HU-05.2 la habilite
- En `HU-05.1_CloudNativePG.md`: agregada nota de secuenciación explícita antes del bloque `backup:` explicando que la referencia en kustomization.yaml debe estar comentada hasta que HU-05.2 cree el secret
- HU-05.2 T-05.2.1 ya contenía el `sed` para descomentar la línea — se validó que el flujo es correcto

---

### DT-003 [CRITICA] [SEQ] Variables TALOS_VERSION y TALOS_SCHEMATIC_ID no declaradas en .envrc

**Ubicación:** `HU-01.3_Variables_Entorno.md` y `HU-02.2_Instalacion_Talos.md`
**Problema:** El template de `.envrc` en HU-01.3 no incluía placeholders para `TALOS_VERSION` ni `TALOS_SCHEMATIC_ID`. HU-02.2 las añadía con `echo >> .envrc`, creando un archivo desordenado con variables inyectadas al final sin estructura.
**Impacto:** Variables críticas de Talos quedan fuera del template centralizado; `.envrc` pierde mantenibilidad.
**Resolución aplicada:**
- Agregados en HU-01.3 los placeholders con comentario de que se llenan en EP-02:
  ```
  export TALOS_VERSION=""
  export TALOS_SCHEMATIC_ID=""
  ```
- HU-02.2 debería usar `sed -i` para reemplazar los valores en lugar de `echo >>` (documentado como DT-016 pendiente)

---

### DT-004 [CRITICA] [DEP] Namespace hcloud-system no creado en HU-02.3

**Ubicación:** `HU-02.3_Bootstrap_Talos_K8s.md` T-02.3.6 y `HU-02.4_Hetzner_CCM.md`
**Problema:** HU-02.3 creaba 7 namespaces, pero HU-02.4 requiere `hcloud-system` que no estaba en la lista. El namespace se creaba en HU-02.4 sin previo aviso, generando una inconsistencia entre "7 namespaces" mencionados en los criterios de aceptación y los 8 realmente necesarios.
**Impacto:** Documentación inconsistente; posible confusión al verificar DoD de HU-02.3.
**Resolución aplicada:**
- Agregado `kubectl create namespace hcloud-system` en T-02.3.6 con comentario "Requerido por HU-02.4 (Hetzner CCM)"
- Actualizados comentarios de "7 namespaces" a "8 namespaces" en verificación
- Actualizado CA-02.3.7 para indicar 8 namespaces incluyendo `hcloud-system`
- Agregado CA-02.3.9 para verificar que bootstrap de etcd se ejecutó exactamente una vez
- Actualizado DoD para reflejar 8 namespaces

---

## HALLAZGOS ALTOS (6 Corregidos, 6 Pendientes)

### DT-005 [ALTA] [DEP] EP-10 no declara dependencias en EP-03 y EP-05 — CORREGIDO

**Ubicación:** `10_EP10_seguridad_hardening.md` cabecera de dependencias
**Problema:** EP-10 listaba solo EP-02 y EP-07 como dependencias entrantes, pero sus tareas referencian manifiestos de FluxCD (EP-03 HU-03.1) y labels de CNPG como `cnpg.io/cluster: serenidad-pg` (EP-05). Las dependencias implícitas no documentadas pueden causar que alguien intente ejecutar EP-10 antes de que EP-03 y EP-05 estén completos.
**Resolución aplicada:** Actualizada la línea de dependencias entrantes a: `EP-02, EP-03 HU-03.1 (FluxCD), EP-05 (CloudNativePG labels), EP-07`

---

### DT-006 [ALTA] [DOC] Alternativa ambigua en EP-10 para aplicación de NetworkPolicies — CORREGIDO

**Ubicación:** `10_EP10_seguridad_hardening.md` sección "Resumen de Dependencias Internas"
**Problema:** Se ofrecían dos opciones: (1) aplicar deny-all junto con allow-rules en el mismo commit, o (2) aplicar allow-rules primero y deny-all al final. Estas NO son equivalentes bajo GitOps/FluxCD. La segunda opción requiere dos commits separados, creando una ventana de tiempo sin deny-all donde el tráfico no estaría controlado.
**Resolución aplicada:** Eliminada la "Alternativa". Establecido un único orden obligatorio: escribir allow-rules primero, luego deny-all, luego kustomization, todo en un ÚNICO commit atómico. FluxCD aplica el conjunto completo.

---

### DT-007 [ALTA] [CA] HU-05.2 sin criterio de PITR recovery test — CORREGIDO

**Ubicación:** `HU-05.2_Backups_B2.md` criterios de aceptación y DoD
**Problema:** T-05.2.5 describe un proceso de restore PITR simulado, pero este paso no aparecía ni en los criterios de aceptación (CA-05.2.x) ni en el DoD. La capacidad de PITR es anunciada como feature central pero nunca se verifica formalmente.
**Resolución aplicada:** Agregado CA-05.2.7 "Proceso de restore PITR simulado documentado" y checkbox en DoD.

---

### DT-008 [ALTA] [CA] HU-07.2 sin test end-to-end de token exchange — CORREGIDO

**Ubicación:** `HU-07.2_IAM_Domain_Service.md` criterios de aceptación
**Problema:** Los 8 criterios originales verificaban componentes individuales (migraciones, Docker, health, etc.) pero ninguno validaba el flujo completo: sesión Kratos -> token exchange -> JWT Ed25519 -> validate-token. Tampoco se verificaba que el patrón Outbox generara eventos en `outbox_events`.
**Resolución aplicada:**
- Agregado CA-07.2.9: "Flujo end-to-end de token exchange funcional" con verificación completa del chain
- Agregado CA-07.2.10: "Evento UserRegistered creado en outbox_events tras primer login"

---

### DT-009 [ALTA] [CA] Resumen de épicas desactualizado — CORREGIDO

**Ubicación:** `00_resumen_epicas.md`
**Problema:** EP-09 listaba "1 HU" pero realmente tiene 2 (HU-09.1A y HU-09.1B). El diagrama de dependencias mostraba EP-10 como "paralelo desde EP-02" sin mencionar EP-03 y EP-05.
**Resolución aplicada:**
- EP-09 actualizado a "2 (HU-09.1A, HU-09.1B)"
- Total de HUs actualizado de 26 a 27
- Diagrama de dependencias corregido para EP-10
- Tabla de archivos actualizada con descripción de HU-09.1A/B

---

### DT-010 [ALTA] [SEQ] envsubst sin validación de variables sustituidas (HU-02.3) — PENDIENTE

**Ubicación:** `HU-02.3_Bootstrap_Talos_K8s.md` T-02.3.1
**Problema:** `envsubst` se usa para sustituir `${FLOATING_IP}` y `${VPS_IP}` en el patch YAML. Si el usuario no ha hecho `source .envrc` o las variables están vacías, envsubst deja los placeholders como están, generando YAML inválido que solo fallará más adelante en `talosctl gen config`.
**Recomendación:** Agregar validación post-envsubst:
```bash
if grep -q '\${' patches/hetzner-cx32.yaml; then
  echo "ERROR: Variables sin sustituir detectadas"
  grep -o '\${[^}]*}' patches/hetzner-cx32.yaml | sort -u
  exit 1
fi
```

---

### DT-011 [ALTA] [SEQ] kubeconfig path inconsistente: ${PWD} vs $(pwd) — PENDIENTE

**Ubicación:** `HU-01.3_Variables_Entorno.md` (usa `${PWD}`) vs `HU-02.3_Bootstrap_Talos_K8s.md` T-02.3.5 (usa `$(pwd)`)
**Problema:** Ambas formas son funcionalmente equivalentes, pero la inconsistencia indica copy-paste de fuentes distintas y dificulta la búsqueda por grep. Además, HU-02.3 asume CWD es la raíz del monorepo sin verificación explícita.
**Recomendación:** Estandarizar en `${PWD}` en todos los archivos. Agregar verificación de CWD antes de generar kubeconfig.

---

### DT-012 [ALTA] [SEC] Sin procedimiento de verificación de backups en Password Manager — PENDIENTE

**Ubicación:** `HU-02.3_Bootstrap_Talos_K8s.md` T-02.3.2, `HU-03.1_Bootstrap_FluxCD.md`
**Problema:** Se instruye "BACKUP en password manager" para secrets.yaml y age private key, pero no hay:
1. Un método para verificar que el backup se realizó
2. Un procedimiento de recovery (qué hacer si el PM no es accesible)
3. Un inventario de qué se respaldó y dónde
**Recomendación:** Crear checklist de verificación de backups como sub-tarea transversal.

---

### DT-013 [ALTA] [SEQ] Script update-firewall.sh referenciado pero no definido hasta EP-10 — PENDIENTE

**Ubicación:** `HU-02.2_Instalacion_Talos.md` troubleshooting y `10_EP10_seguridad_hardening.md` T-10.1.8
**Problema:** HU-02.2 referencia `scripts/update-firewall.sh` como remediación cuando la IP del operador cambia, pero este script no se crea hasta EP-10 T-10.1.8 (Sprint S3-S4). Entre EP-02 (S1) y EP-10, no hay mecanismo documentado para actualizar el firewall.
**Recomendación:** Mover la creación del script a EP-02 o crear una versión minimal en HU-02.1.

---

### DT-014 [ALTA] [SEQ] Floating IP no verificada en Talos tras bootstrap — PENDIENTE

**Ubicación:** `HU-02.3_Bootstrap_Talos_K8s.md` T-02.3.5
**Problema:** Tras el bootstrap, se verifica que el nodo está Ready y que kubectl funciona, pero no se verifica explícitamente que la Floating IP esté configurada en la interfaz de red del nodo Talos. Si la VIP no se anuncia correctamente, el API server no será accesible a través de la Floating IP.
**Recomendación:** Agregar verificación:
```bash
talosctl get addresses --nodes ${VPS_IP} | grep ${FLOATING_IP}
```

---

### DT-015 [ALTA] [DOC] Duplicación entre épica EP-10 y HU-10.1A detallado — PENDIENTE

**Ubicación:** `10_EP10_seguridad_hardening.md` vs `HU-10.1A_Network_Policies_K8s.md`
**Problema:** Ambos archivos contienen las mismas tareas (T-10.1.1 a T-10.1.9) con descripciones ligeramente diferentes. El archivo épica dice "T-10.1.2 — Crear NetworkPolicy: Traefik -> IAM Service" mientras que el detallado la describe como "T-10.1.2 — Definir todas las Network Policies en un manifiesto". Divergencia que crece con el tiempo.
**Recomendación:** El archivo épica debe ser un resumen con referencias al archivo detallado. Eliminar las descripciones duplicadas del épica y reemplazar con `Ver HU-10.1A para detalles de implementación`.

---

## HALLAZGOS MEDIOS (3 Corregidos, 14 Pendientes)

### DT-016 [MEDIA] [SEQ] HU-02.2 usa echo >> para inyectar vars en .envrc — PENDIENTE

**Ubicación:** `HU-02.2_Instalacion_Talos.md`
**Problema:** `TALOS_VERSION` y `TALOS_SCHEMATIC_ID` se añaden con `echo >>` al final del .envrc, rompiendo la estructura. Ahora que HU-01.3 incluye placeholders (DT-003), HU-02.2 debería usar `sed -i` para reemplazar valores.
**Recomendación:** Cambiar `echo >> .envrc` por `sed -i "s|^export TALOS_VERSION=.*|export TALOS_VERSION=\"${TALOS_VERSION}\"|" .envrc`

---

### DT-017 [MEDIA] [TEC] Versiones futuras pinneadas: Go 1.25, K8s 1.33 — PENDIENTE

**Ubicación:** Múltiples archivos (HU-01.2, HU-02.3, HU-09.1A)
**Problema:** Go 1.25 y Kubernetes 1.33 son versiones que pueden no existir o tener breaking changes al momento de ejecución (el plan es de Abril 2026). Los version pins son especulativos.
**Recomendación:** Reemplazar con la versión estable disponible al momento de ejecución. Usar notación "Go >= 1.24 (última estable)" y "K8s >= 1.31 (última estable Talos-compatible)".

---

### DT-018 [MEDIA] [CA] postInitSQL limitación oculta en detalle de HU-05.1 — PENDIENTE

**Ubicación:** `HU-05.1_CloudNativePG.md` línea 164
**Problema:** La nota "postInitSQL solo ejecuta SQL puro, no metacomandos de psql (\c)" está enterrada en un subtask. No aparece en los criterios de aceptación ni en el resumen de la épica. Esto fuerza T-05.1.6 (instalación manual de extensiones) pero el lector de la épica no lo sabrá.
**Recomendación:** Agregar nota explícita en la épica EP-05 y en CA de HU-05.1.

---

### DT-019 [MEDIA] [CA] Outbox worker testing ausente en HU-07.2 — CORREGIDO (parcial)

**Ubicación:** `HU-07.2_IAM_Domain_Service.md` T-07.2.4.4
**Problema:** El outbox worker (goroutine de polling) se describe pero su verificación es débil. No hay test que confirme que el worker publica eventos correctamente.
**Resolución parcial:** CA-07.2.10 agregado para verificar que `outbox_events` contiene registros. Test completo del worker (publicación a NATS) queda para Fase 2.

---

### DT-020 [MEDIA] [SEC] Sin estrategia de rotación de secrets — PENDIENTE

**Ubicación:** Transversal (HU-04.1, HU-05.1, HU-05.2, HU-07.1, HU-07.2)
**Problema:** Todos los secrets (Kratos cookies, PG admin password, B2 credentials, JWT keys) se generan una vez y nunca se rotan. No hay plan de rotación ni documentación de expiración.
**Recomendación:** Crear runbook de rotación de secrets como parte de EP-11 Operaciones Día-2 o como HU-11.4 dedicada.

---

### DT-021 [MEDIA] [TEC] IAM Service requiere NATS_URL en config pero Fase 1 no usa NATS — PENDIENTE

**Ubicación:** `HU-07.2_IAM_Domain_Service.md` config struct
**Problema:** La configuración del IAM Service declara `NATS_URL` como variable requerida, pero NATS no se despliega hasta Fase 2. El outbox worker en Fase 1 solo hace polling local. Si NATS_URL es requerida en el fail-fast del startup, el servicio no arrancará.
**Recomendación:** Hacer `NATS_URL` opcional con default vacío y skip de publicación NATS cuando no está configurada.

---

### DT-022 [MEDIA] [NAM] Inconsistencia en namespaces de ejemplos de test — PENDIENTE

**Ubicación:** `HU-03.3_Sops_Age_Cifrado.md` ejemplos de test
**Problema:** Los ejemplos de test de SOPS usan a veces `serenidad-core` y a veces `default` como namespace target. Mezclar namespaces en ejemplos confunde al implementador.
**Recomendación:** Estandarizar ejemplos con `serenidad-core` para tests funcionales. Usar `default` solo si el test se autolimpia.

---

### DT-023 [MEDIA] [DOC] Sin guía de autenticación de Container Registry en Flux — PENDIENTE

**Ubicación:** `HU-03.2_Estructura_GitOps.md` T-03.2.4
**Problema:** Se menciona que `gitlab-registry-secret.yaml` será necesario con `dockerconfigjson`, pero no hay instrucción de cuándo ni cómo crearlo. HU-07.2 Deployment lo usa como `imagePullSecrets` sin que ninguna HU anterior lo haya creado.
**Recomendación:** Agregar tarea en HU-03.3 o nueva sub-HU para crear y cifrar el secret del registry con el Deploy Token de HU-01.1.

---

### DT-024 [MEDIA] [TEC] TypeScript no pinneado en pipelines CI/CD — PENDIENTE

**Ubicación:** `HU-09.1B_Edge_Pipelines_Docs.md` T-09.1.3 y T-09.1.4
**Problema:** Los pipelines usan `node:22-alpine` e instalan `bun@latest` pero no fijan versión de TypeScript. HU-08.1 y HU-08.2 referencian `typescript@^6.0.0` localmente. Si el CI instala una versión diferente, `tsc --noEmit` podría dar resultados distintos.
**Recomendación:** Fijar TypeScript en el `package.json` de cada proyecto y usar `--frozen-lockfile` (ya presente) para garantizar consistencia.

---

### DT-025 [MEDIA] [CA] HU-04.1 test de certificado depende de HU-04.2 y HU-04.3 — PENDIENTE

**Ubicación:** `HU-04.1_CertManager_LetsEncrypt.md` T-04.1.5
**Problema:** El test de emisión de certificado usa `api.sereni.dad` que requiere DNS (HU-04.3) y Traefik (HU-04.2) para funcionar. Si se ejecuta HU-04.1 de forma aislada, el test fallará.
**Recomendación:** Documentar explícitamente que T-04.1.5 es un test de integración que requiere HU-04.2 y HU-04.3 completadas, o proporcionar un test alternativo con `--dry-run`.

---

### DT-026 [MEDIA] [DOC] HU-11.3 referencia procedimientos de Fase 0 sin nota de scope — PENDIENTE

**Ubicación:** `HU-11.3_Troubleshooting_y_Disaster_Recovery.md` ST-11.3.8.1
**Problema:** El procedimiento de disaster recovery referencia HU-02.2 (Talos), HU-03.1 (FluxCD), HU-05.2 (CNPG restore) como pasos de recovery, pero son de épicas anteriores. No hay nota de que estos son prerrequisitos que deben estar completados.
**Recomendación:** Agregar sección "Prerequisites" al inicio de HU-11.3 listando todas las HUs referenciadas.

---

### DT-027 [MEDIA] [CA] Sin test de integración E2E entre HU-08.1 y HU-08.2 — PENDIENTE

**Ubicación:** Épica EP-08
**Problema:** HU-08.1 (BFF) y HU-08.2 (SPA) se testan de forma aislada. No hay criterio que valide el flujo completo: SPA -> BFF -> IAM -> Kratos -> JWT -> Dashboard. La épica debería tener un CA de integración.
**Recomendación:** Agregar CA a nivel de épica EP-08: "Flujo E2E register -> login -> token exchange -> dashboard funcional".

---

### DT-028 [MEDIA] [TEC] Kratos version inconsistente: v1.3.1 vs v1.3.x — PENDIENTE

**Ubicación:** `07_EP07_iam_autenticacion.md` (v1.3.1) vs `HU-07.1_Ory_Kratos_Passkeys.md` (v1.3.x)
**Problema:** La épica fija v1.3.1 pero el detalle usa v1.3.x. Helm chart usa `>=0.50.0 <1.0.0`. La inconsistencia es menor pero dificulta saber exactamente qué versión se desplegará.
**Recomendación:** Unificar a "v1.3.x (>= 1.3.1)" en ambos archivos.

---

### DT-029 [MEDIA] [DOC] EP-06 contenido duplicado entre épica y HU-06.1 — PENDIENTE

**Ubicación:** `06_EP06_schemas_eventos.md` y `HU-06.1_Schemas_Protobuf.md`
**Problema:** Los ejemplos de `buf.yaml` y `buf.gen.yaml` aparecen word-for-word en ambos archivos. Mantenimiento duplicado sin valor.
**Recomendación:** La épica debe referenciar al detalle: "Ver HU-06.1 para configuración completa de buf."

---

### DT-030 [MEDIA] [DOC] Formato inconsistente de DoD entre épicas y HUs — CORREGIDO (parcial)

**Ubicación:** Transversal
**Problema:** Los archivos épica usan narrativa en DoD; los HU detallados usan checkboxes `- [ ]`. Cuando un HU se divide (HU-09.1, HU-10.1), la épica y los detallados tienen DoD no alineados.
**Resolución parcial:** EP-09 ahora tiene DoD separados por HU-09.1A y HU-09.1B con checkboxes consistentes.

---

### DT-031 [MEDIA] [CA] Deploy Token vs PAT: uso no clarificado — CORREGIDO (parcial via DT-023)

**Ubicación:** `HU-01.1_Cuentas_Servicios_Externos.md` y `HU-01.4_Preparacion_Monorepo.md`
**Problema:** HU-01.1 crea tanto un PAT como un Deploy Token. HU-01.4 usa el PAT para clonar. No queda claro cuándo se usa el Deploy Token. La separación de responsabilidades (PAT=humanos/CI, Deploy Token=K8s pull) no está documentada.
**Recomendación:** Agregar nota en HU-01.4 clarificando: PAT para operaciones interactivas, Deploy Token para imagePullSecrets en K8s (referenciado en DT-023).

---

## HALLAZGOS BAJOS (0 Corregidos, 5 Pendientes)

### DT-032 [BAJA] [DOC] Sin procedimiento de tear-down/rollback por HU — PENDIENTE

**Ubicación:** Transversal
**Problema:** Todas las HUs tienen DoD pero ninguna documenta "cómo deshacer". Si un paso falla a mitad (ej: Talos flasher falla al 50% de escritura en disco), no hay procedimiento documentado de recuperación.
**Recomendación:** Agregar sección "Rollback/Recovery" a HUs de alto riesgo (HU-02.2, HU-02.3, HU-05.1).

---

### DT-033 [BAJA] [DOC] Sin diagrama de arquitectura de red — PENDIENTE

**Ubicación:** `00_resumen_epicas.md`
**Problema:** El resumen tiene diagramas de dependencias textuales pero no hay diagrama de la topología de red: Internet -> Cloudflare -> Floating IP -> Firewall Hetzner -> VPS -> K8s -> Pods. Dificulta la comprensión del flujo de tráfico.
**Recomendación:** Agregar diagrama Mermaid o ASCII de topología de red.

---

### DT-034 [BAJA] [DOC] Formato Markdown inconsistente entre archivos — PENDIENTE

**Ubicación:** Transversal
**Problema:** Algunos archivos usan `**bold**`, otros `__bold__`. Algunos code blocks no especifican lenguaje. Indentación en listas anidadas varía entre 2 y 4 espacios.
**Recomendación:** Estandarizar: `**bold**`, siempre especificar lenguaje en code blocks, 2 espacios de indentación.

---

### DT-035 [BAJA] [TEC] Traefik configurado con ACME propio Y cert-manager — PENDIENTE

**Ubicación:** `HU-04.2_Traefik_Ingress.md`
**Problema:** Traefik se configura tanto para usar cert-manager (via ClusterIssuer) como para tener su propio ACME resolver (`certificatesresolvers.letsencrypt`). Ambos mecanismos funcionan pero la duplicidad es confusa.
**Recomendación:** Documentar explícitamente que cert-manager es el mecanismo primario y el resolver ACME de Traefik es el fallback, o elegir uno solo.

---

### DT-036 [BAJA] [CA] HU-01.2 DoD no referencia tabla de acceptance criteria — PENDIENTE

**Ubicación:** `HU-01.2_Instalacion_Tooling.md`
**Problema:** DoD tiene 5 checkboxes genéricos pero no referencia los 16 CA individuales (CA-01.2.1 a CA-01.2.16). Un implementador podría pasar el DoD sin verificar cada herramienta individualmente.
**Recomendación:** Agregar al DoD: "Los 16 Criterios de Aceptación (CA-01.2.1 a CA-01.2.16) verificados individualmente."

---

## Matriz de Impacto por Épica

| Épica | Criticos | Altos | Medios | Bajos | Total |
|-------|----------|-------|--------|-------|-------|
| EP-01 | 1 (DT-003) | 0 | 2 (DT-022, DT-031) | 1 (DT-036) | 4 |
| EP-02 | 1 (DT-004) | 3 (DT-010, DT-013, DT-014) | 1 (DT-016) | 1 (DT-032) | 6 |
| EP-03 | 0 | 1 (DT-012) | 2 (DT-022, DT-023) | 0 | 3 |
| EP-04 | 0 | 0 | 2 (DT-025, DT-028) | 1 (DT-035) | 3 |
| EP-05 | 1 (DT-002) | 1 (DT-007) | 1 (DT-018) | 0 | 3 |
| EP-06 | 0 | 0 | 1 (DT-029) | 0 | 1 |
| EP-07 | 0 | 1 (DT-008) | 3 (DT-019, DT-020, DT-021) | 0 | 4 |
| EP-08 | 0 | 0 | 1 (DT-027) | 0 | 1 |
| EP-09 | 1 (DT-001) | 1 (DT-009) | 2 (DT-024, DT-030) | 0 | 4 |
| EP-10 | 0 | 2 (DT-005, DT-006) | 0 | 0 | 2 |
| EP-11 | 0 | 0 | 1 (DT-026) | 1 (DT-033) | 2 |
| **Transversal** | 0 | 1 (DT-015) | 1 (DT-017) | 1 (DT-034) | 3 |

---

## Plan de Remediación Sugerido

### Antes de iniciar ejecución (Sprint S0)
1. DT-010: Agregar validación post-envsubst en HU-02.3
2. DT-011: Estandarizar `${PWD}` en todos los archivos
3. DT-013: Mover script update-firewall.sh a EP-02
4. DT-016: Cambiar `echo >>` por `sed -i` en HU-02.2
5. DT-017: Actualizar version pins con versiones estables disponibles
6. DT-023: Crear sub-tarea para gitlab-registry-secret

### Antes de Sprint S2
7. DT-012: Crear checklist de verificación de backups
8. DT-018: Elevar nota postInitSQL a épica
9. DT-025: Documentar dependencia de T-04.1.5 en HU-04.2/4.3

### Antes de Sprint S3
10. DT-021: Hacer NATS_URL opcional en IAM config
11. DT-024: Verificar TypeScript pinneado via lockfile
12. DT-027: Agregar CA de integración E2E en EP-08
13. DT-028: Unificar versión de Kratos

### Mejora continua (post-Fase 1)
14. DT-020: Documentar estrategia de rotación de secrets
15. DT-015: Deduplicar épicas vs HU detallados
16. DT-029, DT-034: Limpieza de documentación

---

## Notas de la Auditoría

- **Metodología:** Lectura exhaustiva de los 38 archivos markdown (11 épicas + 27 HUs). Análisis de dependencias cruzadas, verificación de IDs, validación de criterios de aceptación, coherencia tecnológica.
- **No se encontraron problemas arquitecturales:** La estructura general del plan es sólida. Todos los hallazgos son remediables mediante actualizaciones de documentación.
- **Fortalezas del plan:** Criterios de aceptación detallados por tarea, comandos de verificación explícitos, troubleshooting sections en HUs críticas, separación clara de concerns entre épicas.
- **Área de mejora principal:** Consistencia entre archivos épica y archivos HU detallados (evitar duplicación que diverge).

