# Evaluación de Infraestructura: Plan Actual vs Alternativas Modernas

> **Contexto:** Este documento evalúa la idoneidad del stack planificado en `07b_fase1_vps_hetzner_produccion.md` frente a alternativas actualizadas (2025). La evaluación cubre sistema operativo del nodo, distribución Kubernetes, herramienta GitOps, gestión de secretos, ingress controller, operador de PostgreSQL y proveedor de cómputo. Para cada dimensión se concluye con una **recomendación final** para el proyecto Serenidad.

---

## 1. Sistema Operativo del Nodo Kubernetes

### Plan actual: Talos Linux v1.10.x

Talos Linux es un OS inmutable, diseñado exclusivamente para ejecutar Kubernetes. No tiene shell SSH; toda la gestión es API-only via `talosctl`. Las actualizaciones son imagen A/B con rollback automático en caso de fallo de boot.

### Alternativas evaluadas

| Criterio | **Talos Linux** ✅ Plan actual | **k3s en Alpine/Debian** | **RKE2 en Ubuntu** |
|----------|-------------------------------|--------------------------|---------------------|
| **Superficie de ataque** | Mínima: sin SSH, sin package manager, OS inmutable de solo lectura | Media: OS convencional + shell + package manager | Media-alta: OS convencional + stack Rancher completo |
| **Overhead del plano de control** | ~500 MB RAM (etcd + apiserver + kubelet) | ~300–400 MB RAM (k3s usa SQLite en single-node) | ~1.2–1.8 GB RAM (etcd standalone + componentes) |
| **Complejidad day-0 (instalación)** | Alta: requiere flashear el disco desde rescue mode | Baja: `curl -sfL https://get.k3s.io \| sh` | Media: packages RPM/DEB + config YAML |
| **Complejidad day-2 (operación)** | Media: sin SSH, debugging solo via `talosctl`; upgrade A/B con rollback | Baja: SSH convencional, `kubectl`, upgrades con `system-upgrade-controller` | Media: upgrades via package manager + restart, mayor footprint |
| **GitOps compatibility** | Excelente: FluxCD y ArgoCD se bootstrapean sin SSH via `talosctl` | Excelente: compatible con cualquier herramienta GitOps | Excelente: compatible con todo el ecosistema Rancher/CNCF |
| **Single-node production** | Soportado y documentado; recomendado para producción constrained | Diseñado y optimizado específicamente para esto | Diseñado para multi-node; single-node posible pero sobredimensionado |
| **Kubernetes upstream** | Kubernetes puro sin modificaciones (certificado CNCF) | Kubernetes + distribución ligera (CNCF certificado) | Kubernetes + distribución hardened (CNCF certificado) |
| **Upgrade de Kubernetes** | Un solo comando `talosctl upgrade`; A/B rollback automático | `system-upgrade-controller` o manual | Package manager + restart de servicios |
| **Idoneidad para CX32 (8 GB RAM)** | ✅ Holgada: ~500 MB para el plano de control | ✅ Óptima: ~300 MB para el plano de control | ⚠️ Ajustada: ~1.5 GB solo para el plano de control |
| **Madurez (2025)** | v1.10.x, CNCF certified, activamente desarrollado | v1.30.x, ampliamente usado en edge/IoT/prod | v1.32.x, CNCF certified, EOL de RKE1 en Jul 2025 |

### Análisis

**Talos Linux es la elección más defensiva para Serenidad.** El proyecto maneja datos clínicos sensibles (PHI/PII), y la ausencia de SSH y el modelo de OS inmutable reducen significativamente la superficie de ataque comparado con un VPS convencional. El overhead adicional de complejidad day-0 se justifica por:

1. La producción directa desde el día 1 exige máxima seguridad desde el inicio.
2. El CX32 (8 GB RAM) tiene holgura suficiente para el overhead de Talos.
3. El upgrade A/B con rollback automático reduce el riesgo operacional en single-node.

**k3s sería la alternativa si el criterio fuera exclusivamente simplicidad operacional** (inicio más rápido, debugging más familiar). Sin embargo, para datos de salud, el modelo de seguridad de Talos es netamente superior.

RKE2 no es recomendable para este caso: su overhead de RAM (~1.5 GB solo para el plano de control) consume el 19% de la RAM del CX32 antes de arrancar un solo microservicio.

### ✅ Veredicto: Talos Linux — **MANTENER**

---

## 2. Distribución / Runtime Kubernetes

### Plan actual: Kubernetes 1.33.x sobre Talos Linux

Talos ejecuta Kubernetes upstream sin modificaciones. No aplica ninguna distribución en el sentido clásico; es Kubernetes puro con containerd.

### Alternativas evaluadas

| Distribución | RAM base (plano control) | SSH/shell | Kubernetes upstream | Ideal para |
|-------------|--------------------------|-----------|--------------------|-----------------------|
| **Talos (upstream K8s)** | ~500 MB | ❌ No (por diseño) | ✅ Puro | Producción inmutable, seguridad máxima |
| **k3s** | ~300 MB | ✅ Sí | ✅ Casi puro (SQLite/Kine) | Edge, IoT, single-node resource-constrained |
| **RKE2** | ~1.5 GB | ✅ Sí | ✅ Hardened | Multi-node enterprise, necesidad de CIS compliance |
| **microk8s** | ~600 MB | ✅ Sí | ✅ Sí | Dev local, Ubuntu ecosistema |

### ✅ Veredicto: Kubernetes upstream sobre Talos — **MANTENER**

---

## 3. Herramienta GitOps

### Plan actual: FluxCD v2.x

FluxCD es una colección de controllers CNCF Graduated que implementan el patrón GitOps. Cada controller hace una sola cosa: source-controller (lee Git/Helm/OCI), kustomize-controller (aplica Kustomizations), helm-controller (gestiona HelmReleases), etc.

### Comparativa FluxCD vs ArgoCD

| Criterio | **FluxCD v2.x** ✅ Plan actual | **ArgoCD v2.x** |
|----------|-------------------------------|------------------|
| **Footprint en single-node** | ~200–500 MB RAM total (todos los controllers) | ~1.5–3.5 GB RAM (controller + server + repo-server + Redis) |
| **UI web** | ❌ No nativa; existe Weave GitOps (separado) | ✅ Rica UI web incluida out-of-the-box |
| **Curva de aprendizaje** | Alta: múltiples CRDs (GitRepository, Kustomization, HelmRelease…) | Media: UI visual reduce la curva inicial |
| **Integración GitLab** | Nativa: `flux bootstrap gitlab`, Receiver para webhooks | Via webhook + CLI; OIDC login con GitLab como provider |
| **SOPS nativo** | ✅ Decryption nativo via `spec.decryption.provider: sops` | ❌ Requiere plugin `argocd-vault-plugin` o init container |
| **Image automation** | ✅ image-reflector + image-automation controllers incluidos | ❌ Requiere Argo Image Updater (herramienta separada) |
| **Modularidad** | Excelente: instalar solo los controllers necesarios | Media: todos los componentes se instalan juntos |
| **GitLab Receiver webhook** | ✅ Soporte nativo para disparar reconciliación desde push | ✅ Via webhook genérico |
| **CNCF status** | Graduated | Graduated |
| **Multi-cluster** | Sí, via sharding y múltiples instancias | Sí, ApplicationSets + managed clusters |

### Análisis

**Para un CX32 con 8 GB RAM compartido entre plano de control, FluxCD y microservicios, FluxCD es el claro ganador.** ArgoCD en configuración mínima requiere ~1.5 GB RAM solo para sus componentes (controller + server + repo-server + Redis), lo que consumiría ~18% de la RAM total del nodo antes de ejecutar ningún workload de la aplicación.

FluxCD idle consume ~200 MB, dejando ~7.3 GB para workloads. Además:
- La integración `flux bootstrap gitlab` es first-class y sin fricciones.
- El soporte nativo de SOPS y de image automation elimina dependencias adicionales.
- Para Fase 1 sin necesidad de UI web, la ausencia de dashboard de FluxCD no es una pérdida.

ArgoCD podría considerarse en Fase 3+ cuando el cluster escale a multi-nodo y el budget de RAM sea mayor.

### ✅ Veredicto: FluxCD v2.x — **MANTENER**

---

## 4. Gestión de Secretos

### Plan actual: Sealed Secrets v0.27+

Sealed Secrets cifra los Kubernetes Secrets con la clave pública del cluster (RSA-4096) y los persiste en Git como `SealedSecret` CRDs. El controller en el cluster los descifra en tiempo real.

### Comparativa de alternativas

| Criterio | **Sealed Secrets** ✅ Plan actual | **External Secrets Operator (ESO)** | **SOPS + age** |
|----------|------------------------------------|--------------------------------------|-----------------|
| **Modelo de cifrado** | RSA-4096 / AES-256 (asimétrico, clave en cluster) | Sin cifrado en Git; secretos viven en vault externo (AWS SM, HashiCorp Vault, GitLab Variables, etc.) | AES-256-GCM client-side; clave master fuera del cluster (age/PGP/KMS) |
| **Dónde vive la clave privada** | Kubernetes Secret en `kube-system` | Credenciales de acceso al vault externo (IAM role, token) | Archivo `age.key` fuera del cluster (o KMS) |
| **Overhead en cluster** | ~50 mCPU / 64 MB RAM (controller) | ~100 mCPU / 256 MB RAM (controller) | **Ninguno** en runtime (decrypt ocurre en FluxCD o CI) |
| **Dependencias externas** | Ninguna (autosuficiente) | Requiere vault externo (AWS Secrets Manager, Vault, etc.) | Ninguna en runtime; `age` CLI en CI/CD |
| **Integración con FluxCD** | ✅ Apply de `SealedSecret` CRDs | ✅ Apply de `ExternalSecret` CRDs | ✅ **Nativo**: `spec.decryption.provider: sops` |
| **Rotación de claves** | Automática (30 días default); backup obligatorio de la clave privada | Gestionada por el vault externo | Manual: `sops rotate` + re-cifrar + commit |
| **Riesgo de pérdida de clave** | 🔴 **ALTO**: si se pierde la clave privada del cluster, todos los secrets son irrecuperables | 🟡 Medio: depende del vault externo | 🟡 Medio: depende del backup de la clave `age` |
| **Auditoría de acceso** | Solo vía Kubernetes audit logs | ✅ Logs nativos del vault externo (CloudTrail, Vault audit log) | Solo vía Git history |
| **Costo operacional Fase 1** | Bajo: `kubeseal` + commit | Medio-alto: configurar vault externo | Bajo: `age` CLI + commit |
| **Complejidad** | Baja | Media-Alta | Baja-Media |

### Análisis

**Para Fase 1, Sealed Secrets es la opción correcta.** No requiere infraestructura adicional, el modelo de GitOps es intuitivo (todos los secretos van a Git cifrados), y el overhead en cluster es mínimo.

**La alternativa más moderna y segura a largo plazo es SOPS + age** integrado con FluxCD nativo: no requiere controller extra en el cluster, la clave `age` puede guardarse en el password manager, y la rotación es explícita. Si el proyecto crece, SOPS + KMS (AWS KMS o Google Cloud KMS) añade auditoría centralizada sin dependencia de un controller en el cluster.

**ESO tiene sentido cuando ya existe un vault externo** (HashiCorp Vault Enterprise, AWS Secrets Manager) en la organización. Para Serenidad en Fase 1 sin vault previo, añadiría complejidad sin beneficio proporcional.

**Riesgo crítico de Sealed Secrets a monitorear:** el backup de la clave privada del cluster es absolutamente obligatorio. Si el cluster se destruye y no hay backup de la clave, todos los `SealedSecret` son ilegibles y los secretos deben regenerarse. El plan actual ya contempla este backup en el password manager.

### ✅ Veredicto: Sealed Secrets — **MANTENER en Fase 1**
### 🔄 Recomendación futura (Fase 3): Migrar a **SOPS + age** integrado nativamente con FluxCD para eliminar el controller adicional y mejorar la auditoría.

---

## 5. Ingress Controller

### Plan actual: Traefik v3.x (DaemonSet)

Traefik v3 incluye soporte nativo para CRDs `IngressRoute` de Traefik, middlewares, ForwardAuth, y TLS via cert-manager. Se despliega como DaemonSet en el nodo.

### Comparativa de alternativas

| Criterio | **Traefik v3.x** ✅ Plan actual | **ingress-nginx** | **Envoy Gateway (Kubernetes Gateway API)** | **Cilium Gateway API** |
|----------|--------------------------------|-------------------|--------------------------------------------|------------------------|
| **Resource overhead** | ~30–80 MB RAM | ~50–100 MB RAM | ~100–200 MB RAM (Envoy + controller) | ~50–80 MB RAM (integrado con Cilium CNI) |
| **API nativa** | IngressRoute CRDs (propietario) + Kubernetes Ingress + Gateway API (v3) | Kubernetes Ingress + partial Gateway API | **Kubernetes Gateway API** (estándar CNCF) | **Kubernetes Gateway API** (estándar CNCF) |
| **ForwardAuth / JWT middleware** | ✅ Nativo: Middleware ForwardAuth | ❌ Requiere annotations o plugins externos | ⚠️ Via ExtAuth filter de Envoy | ❌ No nativo |
| **Let's Encrypt (ACME)** | ✅ Integración nativa (alternativa a cert-manager) | Via cert-manager o annotations | Via cert-manager | Via cert-manager |
| **Observabilidad** | Métricas Prometheus + dashboard UI | Métricas Prometheus | Métricas Prometheus nativas de Envoy | Métricas Prometheus via Cilium |
| **Madurez** | v3.x (2024-2025), muy madura | v1.x, extremadamente madura | v1.x, CNCF Incubating (2025) | Reciente, ligado a Cilium |
| **Portabilidad de config** | ⚠️ IngressRoutes propietarios (lock-in parcial) | ⚠️ Annotations propietarias | ✅ Estándar Kubernetes (máxima portabilidad) | ✅ Estándar Kubernetes |
| **Casos de uso en plan actual** | ForwardAuth → IAM validate-token, TLS, HTTP→HTTPS redirect, security headers | Lo mismo via annotations más verbose | Lo mismo via HTTPRoute + ExtAuthz | Limitado sin ForwardAuth nativo |

### Análisis

**Traefik v3 es la elección correcta para el plan actual.** El middleware ForwardAuth es esencial para el patrón de autenticación centralizada (Traefik → IAM /internal/validate-token) y Traefik lo implementa nativamente. ingress-nginx requeriría configuración adicional (auth-url annotation) que es menos elegante y tiene menos soporte para casos complejos.

**La alternativa más moderna es Envoy Gateway (Kubernetes Gateway API)**, que en 2025 alcanzó madurez suficiente para producción. Sin embargo, para reemplazar ForwardAuth en Envoy Gateway se necesita un ExtAuthz filter que añade complejidad y un componente adicional.

**Traefik v3 añade soporte para Kubernetes Gateway API** (además de sus propios IngressRoute CRDs), lo que mitiga el lock-in y permite migración gradual al estándar CNCF en el futuro.

### ✅ Veredicto: Traefik v3.x — **MANTENER**

---

## 6. Operador de PostgreSQL

### Plan actual: CloudNativePG (CNPG) v1.x + PostgreSQL 17.4

CNPG es el operador de PostgreSQL más maduro del ecosistema CNCF. Gestiona failover automático, WAL archiving, Point-in-Time Recovery (PITR), upgrades sin downtime y backups a S3-compatible.

### Comparativa de alternativas

| Criterio | **CloudNativePG** ✅ Plan actual | **Zalando Postgres Operator** | **Percona Operator for PostgreSQL** | **PostgreSQL directo (sin operador)** |
|----------|---------------------------------|-------------------------------|--------------------------------------|---------------------------------------|
| **PITR (Point-in-Time Recovery)** | ✅ Nativo (WAL archiving a S3/B2) | ✅ Vía WAL-E/WAL-G | ✅ Vía pgBackRest | ❌ Manual |
| **Failover automático** | ✅ Raft-based, segundos | ✅ Patroni-based | ✅ Patroni-based | ❌ |
| **Backups a Backblaze B2 (S3-compatible)** | ✅ Nativo | ✅ Via WAL-G | ✅ Via pgBackRest | ❌ Manual |
| **ScheduledBackup CRD** | ✅ Nativo | ✅ Vía CRD | ✅ Vía CRD | ❌ |
| **Resource overhead del operador** | ~50 MB RAM | ~100 MB RAM | ~150 MB RAM | **0** (sin operador) |
| **PgBouncer integrado** | ✅ Pooler CRD nativo | ❌ Separado | ✅ Via CRD | ❌ |
| **Monitoring (PodMonitor)** | ✅ Nativo (`monitoring.enablePodMonitor`) | ✅ Vía PostgresTeam | ✅ Vía CRD | ❌ Manual |
| **CNCF status** | Sandbox → Incubating (2025) | No CNCF | No CNCF | N/A |
| **Adopción** | Adoptado por EDB (empresa), muy activo | Usado por Zalando en producción | Enterprise-grade con soporte comercial | N/A |
| **Idoneidad single-node** | ✅ Funciona con 1 instancia; HA opcional | ✅ Funciona con 1 instancia | ✅ Funciona con 1 instancia | ✅ |

### Análisis

**CloudNativePG es la mejor opción para Kubernetes-native PostgreSQL** y la que mejor integra con el ecosistema CNCF y GitOps. El soporte nativo de PITR con WAL archiving a Backblaze B2 (S3-compatible) sin configuración adicional es un diferenciador clave para la estrategia de backup del plan.

La imagen oficial `ghcr.io/cloudnative-pg/postgresql:17.4` se distribuye desde GitHub Container Registry (no es nuestro registry), lo cual es correcto mantenerlo así — es el canal oficial de distribución del proyecto.

### ✅ Veredicto: CloudNativePG v1.x — **MANTENER**

---

## 7. Proveedor de Cómputo

### Plan actual: Hetzner Cloud CX32 (~€6.80/mes)

VPS con 4 vCPU AMD EPYC, 8 GB RAM, 80 GB NVMe SSD en datacenter EU (Nuremberg o Helsinki).

### Comparativa de alternativas

| Proveedor | Instancia equiv. | Precio/mes | vCPU | RAM | SSD | Datacenter EU | Notas |
|-----------|-----------------|------------|------|-----|-----|----------------|-------|
| **Hetzner CX32** ✅ | CX32 | ~€6.80 | 4 vCPU AMD EPYC | 8 GB | 80 GB NVMe | ✅ nbg1/hel1 | Mejor ratio precio/rendimiento en EU |
| **DigitalOcean** | s-4vcpu-8gb | ~$48/mes | 4 vCPU | 8 GB | 160 GB SSD | ✅ Amsterdam/Frankfurt | 7x más caro |
| **Linode/Akamai** | Dedicated 4GB | ~$36/mes | 2 vCPU dedicado | 4 GB | 80 GB | ✅ Frankfurt | Menos RAM, 5x más caro |
| **OVHcloud** | B2-15 | ~€15/mes | 4 vCPU | 15 GB | 100 GB SSD | ✅ Gravelines/Roubaix | 2x más caro, pero más RAM |
| **Contabo VPS XL** | VPS XL | ~€7.49/mes | 6 vCPU | 16 GB | 400 GB SSD | ✅ Munich | Más barato por GB RAM, pero peor soporte y SLA más débil |
| **Scaleway DEV1-M** | DEV1-M | ~€8.99/mes | 3 vCPU | 4 GB | 40 GB SSD | ✅ Paris/Amsterdam | Menos RAM |
| **AWS t3.medium** | t3.medium | ~$30/mes + almacenamiento | 2 vCPU | 4 GB | EBS separado | ✅ eu-west | 5x más caro solo en instancia |

### Análisis

**Hetzner CX32 es imbatible en la relación precio/rendimiento para Europa.** A €6.80/mes ofrece el stack completo (Talos + K8s + todos los microservicios) con holgura. Ninguna alternativa comparada ofrece 4 vCPU + 8 GB RAM + NVMe por menos de €8/mes en datacenter EU.

**Consideraciones adicionales a favor de Hetzner:**
- API REST completa y bien documentada (`hcloud` CLI).
- Floating IPs gratuitas o de bajo costo (€1.19/mes), esenciales para el plan de recuperación ante fallos.
- Soporte para Talos Linux sin configuraciones especiales.
- CCM (Cloud Controller Manager) oficial y bien mantenido para Kubernetes.
- SLA 99.9% para CX32.

### ✅ Veredicto: Hetzner CX32 — **MANTENER**

---

## 8. Container Registry

### Plan actual: GitLab Container Registry (`registry.gitlab.com`)

El registry de contenedores de GitLab está incluido en el tier gratuito de GitLab.com con 10 GB de storage por proyecto.

### Comparativa de alternativas

| Registry | Precio (tier gratuito) | Integración GitLab CI | Auth Kubernetes | Notas |
|---------|------------------------|----------------------|-----------------|-------|
| **GitLab Container Registry** ✅ | Incluido en GitLab.com (10 GB) | ✅ Nativa (`$CI_REGISTRY_*`) | Via imagePullSecret + Deploy Token | Primera opción natural con GitLab CI |
| **GitHub Container Registry (ghcr.io)** | Incluido en GitHub (packages) | ❌ Requiere token GitLab → GitHub | Via PAT o Deploy Token | No aplica si el monorepo está en GitLab |
| **Docker Hub** | 200 pulls/6h en free tier ⚠️ | Via `docker login` manual | Via imagePullSecret | Rate limiting agresivo en free tier |
| **Cloudflare R2 + OCI registry** | Experimental (no es registry nativo) | Manual | Complejo | No recomendado para producción |
| **AWS ECR Public** | Gratis para imágenes públicas; privado ~$0.10/GB | Via AWS credentials | Via IRSA o imagePullSecret | Sobredimensionado para este caso |
| **Harbor (self-hosted)** | Free (self-hosted en el propio VPS) | Via Docker login | Via imagePullSecret | Consume ~1 GB RAM extra; no recomendado en single-node |

### ✅ Veredicto: GitLab Container Registry — **CORRECTO** (ya en el plan actualizado)

---

## 9. CI/CD Platform

### Plan actual (actualizado): GitLab CI/CD

### Comparativa

| Criterio | **GitLab CI/CD** ✅ | **GitHub Actions** | **Tekton** | **Drone CI** |
|----------|--------------------|-------------------|-----------|--------------|
| **Integración con el monorepo** | ✅ Nativa (monorepo en GitLab.com) | ❌ Monorepo en GitLab, trigger cross-platform complejo | Self-hosted, CRDs en cluster | Self-hosted |
| **Minutos gratis/mes** | 400 min (shared runners) | 2000 min (pero monorepo está en GitLab) | Ilimitado (self-hosted) | Ilimitado (self-hosted) |
| **GitLab Container Registry** | ✅ `$CI_REGISTRY_*` auto-inyectadas | ❌ Requiere token cruzado | Requiere configuración | Requiere configuración |
| **Pipelines as code** | `.gitlab-ci.yml` en el repo | `.github/workflows/*.yml` | YAML CRDs en cluster | `.drone.yml` |
| **Overhead operacional** | Ninguno (SaaS) | Ninguno (SaaS) | Alto (operación del cluster de CI) | Medio (Docker o K8s como runner) |

### ✅ Veredicto: GitLab CI/CD — **CORRECTO** (ya en el plan actualizado)

---

## 10. Resumen Ejecutivo y Recomendaciones

### Tabla de decisiones

| Componente | Plan actual | Alternativa principal | Decisión | Justificación clave |
|-----------|-------------|----------------------|----------|---------------------|
| OS del nodo | Talos Linux v1.10.x | k3s en Alpine Linux | ✅ **MANTENER** Talos | Máxima seguridad para datos clínicos; OS inmutable sin SSH; upgrade A/B |
| K8s distribution | Kubernetes 1.33.x upstream | k3s (embedded dist) | ✅ **MANTENER** upstream | Cero modificaciones al API; compatibilidad máxima con el ecosistema |
| GitOps | FluxCD v2.x | ArgoCD v2.x | ✅ **MANTENER** FluxCD | 5-10x menor footprint RAM; bootstrap GitLab nativo; SOPS nativo |
| Secrets | Sealed Secrets v0.27+ | SOPS + age | ✅ **MANTENER Fase 1**, **migrar a SOPS en Fase 3** | Sealed Secrets suficiente ahora; SOPS más ligero y auditable a futuro |
| Ingress | Traefik v3.x | Envoy Gateway (GW API) | ✅ **MANTENER** Traefik | ForwardAuth nativo esencial para el patrón de AuthN del plan |
| PostgreSQL | CloudNativePG v1.x | Zalando Operator | ✅ **MANTENER** CNPG | PITR nativo a B2; mejor integración CNCF; PodMonitor nativo para Fase 3 |
| Cómputo | Hetzner CX32 | OVH / Contabo | ✅ **MANTENER** Hetzner | Mejor ratio €/rendimiento en EU; API + CCM Kubernetes oficiales |
| Container Registry | GitLab Container Registry | Docker Hub | ✅ **CORRECTO** (ya actualizado) | Incluido en GitLab.com; integración CI nativa |
| CI/CD | GitLab CI/CD | GitHub Actions | ✅ **CORRECTO** (ya actualizado) | Monorepo en GitLab; registry nativo; zero overhead |

### Único cambio recomendado a futuro

**Fase 3: Migrar Sealed Secrets → SOPS + age**

FluxCD tiene soporte nativo para SOPS (`spec.decryption.provider: sops`). Migrar a SOPS elimina el controller de Sealed Secrets (~64 MB RAM), mejora la auditoría (git history + log de decryption en FluxCD), y permite integrar KMS (AWS KMS / Google Cloud KMS) en Fase 4+ para cumplimiento HIPAA/SOC-2 si aplica.

La migración requiere:
1. Generar clave `age`: `age-keygen -o age.key`
2. Crear secret en el cluster: `kubectl create secret generic sops-age -n flux-system --from-file=age.agekey=age.key`
3. Añadir `decryption` en las Kustomizations de FluxCD
4. Re-cifrar los `SealedSecrets` existentes con SOPS: `sops --age $(cat age.key.pub) -e secret.yaml > secret.enc.yaml`
5. Eliminar los `SealedSecret` CRDs y el controller de Sealed Secrets

---

## 11. Conclusión

El stack planificado en `07b_fase1_vps_hetzner_produccion.md` **es sólido y moderno para 2025**. Las decisiones técnicas (Talos + FluxCD + Traefik + CNPG + Hetzner) representan el estado del arte para producción Kubernetes single-node en EU con el presupuesto disponible.

La actualización más significativa fue **reemplazar GitHub/ghcr.io por GitLab/registry.gitlab.com**, que es la elección natural dado que el monorepo vive en GitLab. Esta migración es coherente con el principio de usar plataformas integradas y reduce la fricción operacional.

El único vector de mejora estructural identificado es la migración futura de Sealed Secrets a SOPS, pero esto es una optimización de Fase 3, no un bloqueante para Fase 1.

**El plan está listo para ejecutarse.**
