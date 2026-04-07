# EP-04 — Networking y TLS

**Épica:** Como equipo de infraestructura, necesitamos cert-manager emitiendo certificados Let's Encrypt, Traefik v3.x como Ingress Controller con middlewares de seguridad, y DNS de Cloudflare configurado, para que el tráfico HTTPS llegue de forma segura desde internet hasta los servicios del cluster.

**Origen:** Secciones §12 (V-07), §13 (V-08), §16 (V-11) del documento de decisión.
**Prioridad:** Crítica — Bloqueante para EP-07 (IngressRoutes de Kratos e IAM).
**Sprint:** S2
**Dependencias Entrantes:** EP-03 (FluxCD + estructura GitOps para reconciliar HelmReleases).
**Dependencias Salientes:** EP-07 (Kratos/IAM necesitan IngressRoutes + TLS válido), EP-08 (BFF proxy al backend via HTTPS).

---

## HU-04.1 — cert-manager v1.x + Let's Encrypt (V-07)

**Como** ingeniero de plataforma,
**quiero** tener cert-manager v1.x desplegado con ClusterIssuers de Let's Encrypt (staging y producción),
**para que** los certificados TLS se emitan y renueven automáticamente para los dominios públicos de serenidad.

**Dependencia:** HU-03.2 (estructura GitOps con directorio `infra/infrastructure/cert-manager/`).

### Tareas y Subtareas

#### T-04.1.1 — Crear HelmRepository de Jetstack

- **ST-04.1.1.1** — Crear archivo `infra/infrastructure/cert-manager/helmrepository.yaml`.
  - **CA:** `apiVersion: source.toolkit.fluxcd.io/v1`, `kind: HelmRepository`.
  - **CA:** URL: `https://charts.jetstack.io`.
  - **CA:** Namespace: `flux-system`, intervalo: `1h`.
- **ST-04.1.1.2** — Commitear y pushear.
  - **CA:** FluxCD reconcilia el HelmRepository sin error.
  - **CA:** `flux get source helm jetstack` → `True`.

#### T-04.1.2 — Crear HelmRelease de cert-manager

- **ST-04.1.2.1** — Crear archivo `infra/infrastructure/cert-manager/helmrelease.yaml`.
  - **CA:** Chart: `cert-manager`, version: `>=1.15.0 <2.0.0`.
  - **CA:** Namespace: `cert-manager`.
  - **CA:** Values incluyen: `crds.enabled: true`, `webhook.enabled: true`.
  - **CA:** Resources: requests `cpu: 20m, memory: 64Mi`, limits `cpu: 200m, memory: 128Mi`.
  - **CA:** Prometheus metrics habilitadas, servicemonitor deshabilitado (Fase 3).
- **ST-04.1.2.2** — Commitear y pushear.
  - **CA:** FluxCD reconcilia el HelmRelease sin error.

#### T-04.1.3 — Verificar despliegue de cert-manager

- **ST-04.1.3.1** — Verificar pods de cert-manager.
  - **CA:** `kubectl get pods -n cert-manager` → 3 pods Running: `cert-manager`, `cert-manager-cainjector`, `cert-manager-webhook`.
- **ST-04.1.3.2** — Verificar que los CRDs están instalados.
  - **CA:** `kubectl get crd | grep cert-manager` → CRDs de Certificate, ClusterIssuer, Issuer, etc.
- **ST-04.1.3.3** — Verificar logs sin errores críticos.
  - **CA:** `kubectl logs deployment/cert-manager -n cert-manager` → sin errores FATAL.

#### T-04.1.4 — Crear ClusterIssuers (staging y producción)

- **ST-04.1.4.1** — Crear archivo `infra/infrastructure/cert-manager/clusterissuers.yaml`.
  - **CA:** Contiene 2 ClusterIssuers:
    - `letsencrypt-staging`: server `https://acme-staging-v02.api.letsencrypt.org/directory`.
    - `letsencrypt-prod`: server `https://acme-v02.api.letsencrypt.org/directory`.
  - **CA:** Ambos usan solver HTTP-01 con ingress class `traefik`.
  - **CA:** Email configurado: `admin@sereni.dad`.
  - **CA:** `privateKeySecretRef` diferente para cada issuer.
- **ST-04.1.4.2** — Commitear y pushear.
  - **CA:** FluxCD aplica los ClusterIssuers.
- **ST-04.1.4.3** — Verificar ClusterIssuers.
  - **CA:** `kubectl get clusterissuers` → `letsencrypt-staging True`, `letsencrypt-prod True`.

#### T-04.1.5 — Test de emisión de certificado con staging

- **ST-04.1.5.1** — Crear Certificate de prueba con issuer staging para `api.sereni.dad`.
  - **CA:** `kubectl apply -f` Certificate temporal en namespace `serenidad-core`.
  - **CA:** `issuerRef.name: letsencrypt-staging`, `issuerRef.kind: ClusterIssuer`.
- **ST-04.1.5.2** — Monitorear estado del Certificate (1-3 minutos).
  - **CA:** `kubectl describe certificate test-certificate -n serenidad-core` → `Status: True, Reason: Ready`.
  - **CA:** Si falla: verificar puerto 80 abierto en firewall Hetzner, DNS resuelve correctamente.
- **ST-04.1.5.3** — Eliminar Certificate de prueba tras verificación exitosa.
  - **CA:** `kubectl delete certificate test-certificate -n serenidad-core` → eliminado.
  - **CA:** Secret TLS de prueba también eliminado.

#### T-04.1.6 — Crear kustomization.yaml del componente cert-manager

- **ST-04.1.6.1** — Crear `infra/infrastructure/cert-manager/kustomization.yaml`.
  - **CA:** Resources: `helmrepository.yaml`, `helmrelease.yaml`, `clusterissuers.yaml`.
- **ST-04.1.6.2** — Commitear y verificar reconciliación.
  - **CA:** FluxCD reconcilia sin error.

### Criterios de Aceptación de la HU-04.1

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-1 | cert-manager desplegado | 3 pods Running en `cert-manager` namespace |
| CA-2 | ClusterIssuers Ready | `kubectl get clusterissuers` → staging y prod en True |
| CA-3 | Certificado staging emitido | Test Certificate con staging issuer → Ready |
| CA-4 | CRDs instalados | `kubectl get crd | grep cert-manager` → presentes |
| CA-5 | HelmRelease gestionado por FluxCD | `flux get helmrelease cert-manager -n cert-manager` → Ready |

### Definition of Done — HU-04.1

- [ ] cert-manager v1.x desplegado via FluxCD HelmRelease.
- [ ] 2 ClusterIssuers creados y en estado Ready.
- [ ] Test de emisión con staging exitoso.
- [ ] Manifests commiteados en estructura GitOps.
- [ ] Sin errores en logs de cert-manager.

---

## HU-04.2 — Traefik v3.x Ingress Controller (V-08)

**Como** ingeniero de plataforma,
**quiero** tener Traefik v3.x desplegado como DaemonSet con hostPort en :80/:443 y middlewares de seguridad configurados,
**para que** el tráfico HTTP/HTTPS del VPS se enrute correctamente a los servicios del cluster con redirección automática HTTP→HTTPS, ForwardAuth, y headers de seguridad.

**Dependencia:** HU-04.1 completada (cert-manager para resolución de certificados ACME).

### Tareas y Subtareas

#### T-04.2.1 — Crear HelmRepository de Traefik

- **ST-04.2.1.1** — Crear archivo `infra/infrastructure/traefik/helmrepository.yaml`.
  - **CA:** URL: `https://helm.traefik.io/traefik`.
  - **CA:** Namespace: `flux-system`, intervalo: `1h`.
- **ST-04.2.1.2** — Commitear y pushear.
  - **CA:** `flux get source helm traefik` → `True`.

#### T-04.2.2 — Crear HelmRelease de Traefik

- **ST-04.2.2.1** — Crear archivo `infra/infrastructure/traefik/helmrelease.yaml`.
  - **CA:** `dependsOn`: cert-manager (namespace `cert-manager`).
  - **CA:** Chart: `traefik`, version: `>=30.0.0 <31.0.0`.
  - **CA:** Deployment como `DaemonSet`.
  - **CA:** Puertos configurados:
    - `web`: port 8000, exposedPort 80, hostPort 80, redirectTo websecure.
    - `websecure`: port 8443, exposedPort 443, hostPort 443, TLS enabled.
  - **CA:** Service type: `ClusterIP` (no LoadBalancer).
  - **CA:** IngressClass habilitado como default.
  - **CA:** Providers: kubernetesCRD con `allowCrossNamespace: true`, kubernetesIngress habilitado.
  - **CA:** Logs: general level INFO, access logs enabled.
  - **CA:** API dashboard habilitado, insecure false.
  - **CA:** Certificados ACME configurados via `additionalArguments`:
    - certResolver `letsencrypt`, httpchallenge entrypoint `web`.
    - Email: `admin@sereni.dad`.
    - Storage: `/data/acme.json`.
  - **CA:** Persistencia habilitada: storageClass `local-path`, size `128Mi`.
  - **CA:** Resources: requests `cpu: 50m, memory: 64Mi`, limits `cpu: 300m, memory: 256Mi`.
- **ST-04.2.2.2** — Commitear y pushear.
  - **CA:** FluxCD reconcilia el HelmRelease.

#### T-04.2.3 — Crear middlewares de seguridad

- **ST-04.2.3.1** — Crear archivo `infra/infrastructure/traefik/middlewares.yaml` con 3 middlewares:
  - **Middleware 1: `iam-forward-auth`**
    - ForwardAuth address: `http://iam-service.serenidad-core.svc.cluster.local:8080/internal/validate-token`.
    - `trustForwardHeader: true`.
    - `authResponseHeaders`: `X-User-ID`, `X-User-Role`, `X-Tenant-ID`, `X-User-DID`.
    - `authRequestHeaders`: `Authorization`, `Cookie`.
    - **CA:** Middleware define la delegación de autenticación al IAM Service.
  - **Middleware 2: `redirect-to-https`**
    - `redirectScheme`: scheme `https`, permanent `true`.
    - **CA:** Middleware fuerza redirección HTTP → HTTPS.
  - **Middleware 3: `security-headers`**
    - `frameDeny: true`, `contentTypeNosniff: true`, `browserXssFilter: true`.
    - `referrerPolicy: strict-origin-when-cross-origin`.
    - `X-Powered-By: ""` (ocultar servidor).
    - HSTS: `stsSeconds: 31536000`, `stsIncludeSubdomains: true`, `stsPreload: true`.
    - **CA:** Headers de seguridad configurados según mejores prácticas OWASP.
- **ST-04.2.3.2** — Commitear y pushear.
  - **CA:** `kubectl get middlewares -n traefik` → 3 middlewares presentes.

#### T-04.2.4 — Verificar despliegue de Traefik

- **ST-04.2.4.1** — Verificar pod de Traefik.
  - **CA:** `kubectl get pods -n traefik` → 1 pod Running (DaemonSet en single-node).
- **ST-04.2.4.2** — Verificar hostPort en el VPS.
  - **CA:** `curl -I http://${FLOATING_IP}` → `HTTP/1.1 308 Permanent Redirect` (HTTP→HTTPS).
- **ST-04.2.4.3** — Verificar HTTPS en el VPS (sin IngressRoutes aún).
  - **CA:** `curl -k -I https://${FLOATING_IP}` → `HTTP/1.1 404 Not Found` (Traefik responde, sin rutas).
- **ST-04.2.4.4** — Verificar IngressClass.
  - **CA:** `kubectl get ingressclass` → `traefik traefik.io/ingress-lb`.
- **ST-04.2.4.5** — Verificar middlewares aplicados.
  - **CA:** `kubectl get middlewares -A` → 3 middlewares en namespace `traefik`.

#### T-04.2.5 — Crear kustomization.yaml del componente Traefik

- **ST-04.2.5.1** — Crear `infra/infrastructure/traefik/kustomization.yaml`.
  - **CA:** Resources: `helmrepository.yaml`, `helmrelease.yaml`, `middlewares.yaml`.
- **ST-04.2.5.2** — Commitear y verificar.
  - **CA:** FluxCD reconcilia sin error.

### Criterios de Aceptación de la HU-04.2

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-1 | Traefik DaemonSet Running | `kubectl get pods -n traefik` → 1 pod Running |
| CA-2 | hostPort :80 escuchando | `curl -I http://${FLOATING_IP}` → 308 redirect |
| CA-3 | hostPort :443 escuchando | `curl -k -I https://${FLOATING_IP}` → respuesta (404 OK) |
| CA-4 | HTTP→HTTPS redirect | Status 308 en :80 |
| CA-5 | 3 middlewares creados | `kubectl get middlewares -n traefik` → iam-forward-auth, redirect-to-https, security-headers |
| CA-6 | IngressClass default | `kubectl get ingressclass` → traefik como default |
| CA-7 | dependsOn cert-manager | HelmRelease espera a cert-manager antes de desplegar |

### Definition of Done — HU-04.2

- [ ] Traefik v3.x desplegado como DaemonSet via FluxCD.
- [ ] Puertos 80 y 443 escuchando en el VPS via hostPort.
- [ ] Redirección HTTP→HTTPS funcional.
- [ ] 3 middlewares de seguridad aplicados.
- [ ] IngressClass `traefik` configurado como default.
- [ ] Manifests en estructura GitOps.

---

## HU-04.3 — DNS y Cloudflare (V-11)

**Como** ingeniero de plataforma,
**quiero** tener los registros DNS configurados en Cloudflare apuntando al VPS con proxy activo y SSL/TLS en modo Full (strict),
**para que** `api.sereni.dad` resuelva correctamente al VPS con protección DDoS y HTTPS end-to-end.

**Dependencia:** HU-04.2 completada (Traefik escuchando en :80/:443 del VPS).

### Tareas y Subtareas

#### T-04.3.1 — Crear registro DNS A para api.sereni.dad

- **ST-04.3.1.1** — Crear registro A `api` apuntando a `FLOATING_IP` con proxy habilitado.
  - **CA:** `curl -s -X POST` al API de Cloudflare retorna `success: true`.
  - **CA:** Registro A: name `api`, content `${FLOATING_IP}`, proxied `true`, TTL auto.
- **ST-04.3.1.2** — Verificar propagación DNS (1-5 minutos con Cloudflare Proxy).
  - **CA:** `dig api.sereni.dad @1.1.1.1` → resuelve a IP de Cloudflare (NO la IP del VPS — proxy activo).
- **ST-04.3.1.3** — Verificar que el tráfico llega al VPS a través de Cloudflare.
  - **CA:** `curl -I https://api.sereni.dad` → respuesta del VPS (puede ser 404 si no hay rutas aún).
  - **CA:** Header `cf-ray` presente en la respuesta (confirma que pasa por Cloudflare).

#### T-04.3.2 — Configurar SSL/TLS en Cloudflare

- **ST-04.3.2.1** — Establecer modo SSL/TLS a "Full (strict)".
  - **CA:** `curl -s -X PATCH` al API de Cloudflare para setting `ssl` con value `strict` → `success: true`.
  - **CA:** Cloudflare verifica el certificado Let's Encrypt del VPS (no acepta self-signed).
- **ST-04.3.2.2** — Habilitar HSTS en Cloudflare.
  - **CA:** `curl -s -X PATCH` al API de Cloudflare para `security_header` → `success: true`.
  - **CA:** HSTS: `enabled: true`, `max_age: 31536000`, `include_subdomains: true`, `preload: true`, `nosniff: true`.

#### T-04.3.3 — Verificar HTTPS end-to-end

- **ST-04.3.3.1** — Verificar que HTTPS funciona con certificado de confianza.
  - **CA:** `curl https://api.sereni.dad` → sin warnings TLS, sin `-k`.
- **ST-04.3.3.2** — Verificar certificado emitido.
  - **CA:** `openssl s_client -connect api.sereni.dad:443 -servername api.sereni.dad` → certificado de Cloudflare (edge) o Let's Encrypt (origin).
- **ST-04.3.3.3** — Verificar headers de seguridad HSTS.
  - **CA:** Response incluye `strict-transport-security: max-age=31536000; includeSubDomains; preload`.

### Criterios de Aceptación de la HU-04.3

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-1 | DNS A record creado | `dig api.sereni.dad @1.1.1.1` → resuelve (IP de Cloudflare) |
| CA-2 | Proxy Cloudflare activo | Header `cf-ray` en respuesta HTTPS |
| CA-3 | SSL/TLS Full (strict) | Cloudflare verifica certificado del origin (Let's Encrypt) |
| CA-4 | HSTS habilitado | Header `strict-transport-security` en respuesta |
| CA-5 | HTTPS sin warnings | `curl https://api.sereni.dad` → sin errores TLS |
| CA-6 | IP del VPS oculta | `dig api.sereni.dad` → IP de Cloudflare, no del VPS |

### Definition of Done — HU-04.3

- [ ] Registro DNS A para `api.sereni.dad` creado con proxy Cloudflare.
- [ ] SSL/TLS en modo Full (strict).
- [ ] HSTS habilitado con max_age 1 año.
- [ ] HTTPS end-to-end verificado sin warnings.
- [ ] IP real del VPS oculta detrás de Cloudflare.

---

## Resumen de Dependencias Internas EP-04

```
HU-04.1 (cert-manager + ClusterIssuers)
  └──► HU-04.2 (Traefik v3.x — dependsOn cert-manager)
        └──► HU-04.3 (DNS Cloudflare — necesita Traefik escuchando)
```

Cadena estrictamente secuencial: cert-manager → Traefik → DNS.
El test de certificado staging (T-04.1.5) requiere que DNS ya esté parcialmente configurado o que se use resolución directa por IP.
