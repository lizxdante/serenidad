# EP-02 — Infraestructura Base VPS

**Épica:** Como equipo de infraestructura, necesitamos un servidor VPS Hetzner CX32 con Talos Linux y Kubernetes 1.33.x funcionando como cluster single-node con el Hetzner CCM operativo, para tener la base de cómputo donde desplegar todos los componentes de serenidad.

**Origen:** Secciones §6 (V-01), §7 (V-02), §8 (V-03), §9 (V-04) del documento de decisión.
**Prioridad:** Crítica — Bloqueante para EP-03, EP-04, EP-05, EP-10.
**Sprint:** S1
**Dependencias Entrantes:** EP-01 (cuentas, tooling, monorepo, SSH key).
**Dependencias Salientes:** EP-03 (FluxCD), EP-04 (Networking), EP-05 (Base de Datos), EP-10 (Hardening).

---

## HU-02.1 — Aprovisionamiento del VPS Hetzner CX32 (V-01)

**Como** ingeniero de plataforma,
**quiero** tener un servidor VPS Hetzner CX32 aprovisionado con Floating IP y firewall básico configurado,
**para que** disponga de la infraestructura de cómputo necesaria para instalar Talos Linux y Kubernetes.

### Tareas y Subtareas

#### T-02.1.1 — Crear el servidor CX32

- **ST-02.1.1.1** — Verificar disponibilidad del tipo de servidor CX32 en Hetzner.
  - **CA:** `hcloud server-type list | grep CX32` muestra 4 cores, 8 GB RAM, 80 GB SSD.
- **ST-02.1.1.2** — Seleccionar datacenter (`nbg1-dc3` Nuremberg o `hel1-dc2` Helsinki).
  - **CA:** Datacenter elegido y documentado en `.envrc`.
- **ST-02.1.1.3** — Crear servidor `serenidad-prod-01` con imagen temporal Ubuntu 24.04 y SSH key.
  - **CA:** `hcloud server describe serenidad-prod-01` muestra el servidor en estado `running`.
  - **CA:** Server type es CX32 (4 vCPU, 8 GB RAM, 80 GB NVMe).
  - **CA:** SSH key `serenidad-admin-key` asociada al servidor.
- **ST-02.1.1.4** — Guardar IP pública asignada en `.envrc` como `VPS_IP`.
  - **CA:** `VPS_IP` tiene un valor IPv4 válido.
  - **CA:** `ping $VPS_IP` responde correctamente.

#### T-02.1.2 — Crear y asignar Floating IP

- **ST-02.1.2.1** — Crear Floating IP IPv4 en la misma región que el servidor.
  - **CA:** `hcloud floating-ip describe serenidad-prod-fip` muestra la IP creada.
  - **CA:** La IP es IPv4 y la región coincide con la del servidor (nbg1 o hel1).
- **ST-02.1.2.2** — Asignar la Floating IP al servidor `serenidad-prod-01`.
  - **CA:** `hcloud floating-ip describe serenidad-prod-fip` muestra `Server: serenidad-prod-01`.
- **ST-02.1.2.3** — Guardar Floating IP en `.envrc` como `FLOATING_IP`.
  - **CA:** `FLOATING_IP` tiene un valor IPv4 válido, diferente de `VPS_IP`.

#### T-02.1.3 — Configurar Firewall de Hetzner

- **ST-02.1.3.1** — Crear firewall `serenidad-prod-fw`.
  - **CA:** `hcloud firewall describe serenidad-prod-fw` muestra el firewall creado.
- **ST-02.1.3.2** — Agregar regla de ingreso TCP puerto 80 desde 0.0.0.0/0 y ::/0 (HTTP — ACME).
  - **CA:** Regla visible en `hcloud firewall describe serenidad-prod-fw`.
  - **CA:** Descripción: "HTTP Let's Encrypt ACME".
- **ST-02.1.3.3** — Agregar regla de ingreso TCP puerto 443 desde 0.0.0.0/0 y ::/0 (HTTPS).
  - **CA:** Regla visible, descripción: "HTTPS aplicacion".
- **ST-02.1.3.4** — Obtener IP actual de trabajo (`curl -s https://ifconfig.me`).
  - **CA:** Se obtiene una IP válida.
- **ST-02.1.3.5** — Agregar regla de ingreso TCP puerto 50000 restringida a IP de trabajo (Talos API).
  - **CA:** `source-ips` es `${MY_IP}/32` (no 0.0.0.0/0).
  - **CA:** Descripción: "Talos API (solo IP de trabajo)".
- **ST-02.1.3.6** — Agregar regla de ingreso TCP puerto 6443 restringida a IP de trabajo (K8s API).
  - **CA:** `source-ips` es `${MY_IP}/32`.
  - **CA:** Descripción: "Kubernetes API (solo IP de trabajo)".
- **ST-02.1.3.7** — Aplicar firewall al servidor `serenidad-prod-01`.
  - **CA:** `hcloud firewall describe serenidad-prod-fw` muestra `Applied To: server serenidad-prod-01`.
- **ST-02.1.3.8** — Verificar reglas completas del firewall.
  - **CA:** 4 reglas de ingreso configuradas (80, 443, 50000, 6443).
  - **CA:** Puertos 50000 y 6443 restringidos a IP de trabajo.

#### T-02.1.4 — Verificar acceso SSH al servidor (Ubuntu temporal)

- **ST-02.1.4.1** — Conectar al servidor via SSH con la key de Hetzner.
  - **CA:** `ssh -i ~/.ssh/hetzner_serenidad_ed25519 root@${VPS_IP}` → prompt de Ubuntu.
- **ST-02.1.4.2** — Verificar hardware del servidor.
  - **CA:** `nproc` → 4.
  - **CA:** `free -h` → ~7.7 GB RAM.
  - **CA:** `df -h /` → ~80 GB disco.
- **ST-02.1.4.3** — Identificar y anotar nombre del disco principal.
  - **CA:** `lsblk` → disco principal identificado (generalmente `/dev/sda`).
  - **CA:** Nombre del disco documentado para usar en V-02.

### Criterios de Aceptación de la HU-02.1

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-1 | Servidor CX32 running | `hcloud server describe serenidad-prod-01` → status: running |
| CA-2 | Floating IP asignada | `hcloud floating-ip describe serenidad-prod-fip` → assigned |
| CA-3 | Firewall aplicado | 4 reglas de ingreso; 50000 y 6443 restringidos |
| CA-4 | SSH funcional | Conexión SSH exitosa al servidor |
| CA-5 | Hardware verificado | 4 vCPU, ~8 GB RAM, ~80 GB disco |
| CA-6 | IPs documentadas en .envrc | `VPS_IP` y `FLOATING_IP` con valores válidos |

### Definition of Done — HU-02.1

- [ ] Servidor CX32 creado y running en Hetzner.
- [ ] Floating IP creada, asignada, y documentada.
- [ ] Firewall con 4 reglas aplicado al servidor.
- [ ] Acceso SSH verificado y hardware confirmado.
- [ ] `VPS_IP` y `FLOATING_IP` guardados en `.envrc`.

---

## HU-02.2 — Instalación de Talos Linux en el VPS (V-02)

**Como** ingeniero de plataforma,
**quiero** reemplazar Ubuntu temporal por Talos Linux v1.10.x en el VPS,
**para que** el servidor tenga un OS inmutable y seguro optimizado para Kubernetes.

**Dependencia:** HU-02.1 completada (servidor aprovisionado, SSH funcional).

### Tareas y Subtareas

#### T-02.2.1 — Obtener versión de Talos y preparar imagen

- **ST-02.2.1.1** — Consultar la última versión estable de Talos Linux.
  - **CA:** `TALOS_VERSION` contiene un valor tipo `v1.10.x`.
- **ST-02.2.1.2** — Crear archivo de schematic con extensión `qemu-guest-agent`.
  - **CA:** Archivo `/tmp/talos-hetzner-schematic.yaml` creado con `siderolabs/qemu-guest-agent`.
- **ST-02.2.1.3** — Subir schematic al Talos Image Factory y obtener `SCHEMATIC_ID`.
  - **CA:** `SCHEMATIC_ID` es un hash hexadecimal válido.
  - **CA:** `curl` al endpoint de Image Factory retorna 200.
- **ST-02.2.1.4** — Construir URL de imagen `hcloud-amd64.raw.xz`.
  - **CA:** `IMAGE_URL` tiene formato `https://factory.talos.dev/image/{SCHEMATIC_ID}/{TALOS_VERSION}/hcloud-amd64.raw.xz`.

#### T-02.2.2 — Activar modo rescue y flashear Talos

- **ST-02.2.2.1** — Activar modo rescue en el servidor Hetzner.
  - **CA:** `hcloud server enable-rescue` ejecuta sin error.
  - **CA:** Tipo de rescue: `linux64`.
- **ST-02.2.2.2** — Reiniciar servidor para entrar en modo rescue.
  - **CA:** `hcloud server reset serenidad-prod-01` ejecuta sin error.
- **ST-02.2.2.3** — Esperar ~60 segundos y conectar al rescue system via SSH.
  - **CA:** SSH conecta al rescue system (prompt diferente al de Ubuntu).
- **ST-02.2.2.4** — Verificar disco principal en rescue system con `lsblk`.
  - **CA:** Disco `/dev/sda` (o el identificado en T-02.1.4) visible.
- **ST-02.2.2.5** — Descargar y flashear imagen Talos en el disco (`curl | xz | dd`).
  - **CA:** Comando `dd` completa sin error.
  - **CA:** `sync` ejecuta para asegurar escritura en disco.
- **ST-02.2.2.6** — Reiniciar servidor para que arranque desde Talos.
  - **CA:** `reboot` ejecuta desde la sesión rescue.

#### T-02.2.3 — Verificar Talos en modo mantenimiento

- **ST-02.2.3.1** — Esperar ~90 segundos para que Talos arranque.
  - **CA:** Timer de espera completado.
- **ST-02.2.3.2** — Verificar que Talos responde en puerto 50000 (modo mantenimiento).
  - **CA:** `talosctl version --insecure --nodes ${VPS_IP}` retorna versión de Talos v1.10.x en Client y Server.
- **ST-02.2.3.3** — Si no responde: verificar firewall permite puerto 50000 desde IP de trabajo.
  - **CA:** Troubleshooting documentado si fue necesario.

### Criterios de Aceptación de la HU-02.2

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-1 | Talos Linux instalado | `talosctl version --insecure --nodes ${VPS_IP}` → Server Tag: v1.10.x |
| CA-2 | qemu-guest-agent incluido | Schematic ID con extensión qemu-guest-agent usado en la imagen |
| CA-3 | Talos en maintenance mode | Responde en puerto 50000 sin credenciales (insecure) |
| CA-4 | Ubuntu removido | El servidor ya no tiene Ubuntu; solo Talos |

### Definition of Done — HU-02.2

- [ ] Talos Linux v1.10.x flasheado en el disco del servidor.
- [ ] Servidor arranca correctamente desde Talos.
- [ ] `talosctl version --insecure` confirma versión correcta.
- [ ] Rescue mode desactivado (servidor arrancó normalmente).

---

## HU-02.3 — Bootstrap de Talos + Kubernetes 1.33.x (V-03)

**Como** ingeniero de plataforma,
**quiero** tener un cluster de Kubernetes 1.33.x funcional sobre Talos Linux con namespaces creados,
**para que** pueda desplegar componentes de infraestructura y aplicaciones.

**Dependencia:** HU-02.2 completada (Talos en modo mantenimiento).

### Tareas y Subtareas

#### T-02.3.1 — Crear patch de configuración para Hetzner CX32

- **ST-02.3.1.1** — Crear directorio `infra/clusters/hetzner-prod/talos/patches/`.
  - **CA:** Directorio existe.
- **ST-02.3.1.2** — Crear archivo de patch `hetzner-cx32.yaml` con configuración específica:
  - Red: hostname `serenidad-prod-01`, interfaz `eth0` DHCP + alias Floating IP.
  - Disco: `/dev/sda`, bootloader true, wipe false.
  - Tiempo: NTP servers `0.de.pool.ntp.org`, `1.de.pool.ntp.org`, `time.cloudflare.com`.
  - Kubelet: provider-id hcloud, nodeIP validSubnets.
  - Sysctls: ip_forward, bridge-nf-call-iptables, bridge-nf-call-ip6tables, vm.max_map_count, inotify limits, kernel.pid_max.
  - Cluster: `allowSchedulingOnControlPlanes: true`.
  - API server: certSANs con VPS_IP, FLOATING_IP, 127.0.0.1.
  - Red cluster: podSubnets 10.244.0.0/16, serviceSubnets 10.96.0.0/12, CNI flannel.
  - etcd: advertisedSubnets.
  - **CA:** Archivo YAML válido, contiene todas las secciones documentadas en §8.1.
  - **CA:** Variables `${VPS_IP}` y `${FLOATING_IP}` sustituidas con valores reales.

#### T-02.3.2 — Generar secrets y configuración del cluster

- **ST-02.3.2.1** — Sustituir variables en el patch con `envsubst` (o manualmente).
  - **CA:** Archivo resultante contiene IPs reales, no placeholders `${...}`.
- **ST-02.3.2.2** — Generar secrets del cluster con `talosctl gen secrets`.
  - **CA:** Archivo `infra/clusters/hetzner-prod/talos/secrets.yaml` generado.
  - **CA:** Archivo contiene claves criptográficas (NO commitear).
- **ST-02.3.2.3** — Backup del archivo `secrets.yaml` en password manager.
  - **CA:** Backup verificado y documentado.
  - **CA:** Nota: "Si pierdes este archivo, no podrás regenerar las credenciales del cluster."
- **ST-02.3.2.4** — Generar configuración del cluster con `talosctl gen config`.
  - **CA:** Se generan 3 archivos: `controlplane.yaml`, `worker.yaml`, `talosconfig`.
  - **CA:** El endpoint del cluster es `https://${FLOATING_IP}:6443`.
  - **CA:** Los secrets y el patch se incorporan correctamente.
- **ST-02.3.2.5** — Agregar archivos sensibles a `.gitignore`.
  - **CA:** `controlplane.yaml`, `worker.yaml`, `talosconfig`, `secrets.yaml`, `kubeconfig` están en `.gitignore`.
  - **CA:** `git status` no los muestra como no rastreados.

#### T-02.3.3 — Aplicar configuración y bootstrap de etcd

- **ST-02.3.3.1** — Aplicar configuración del control plane al servidor Talos.
  - **CA:** `talosctl apply-config --insecure --nodes ${VPS_IP} --file controlplane.yaml` ejecuta sin error.
- **ST-02.3.3.2** — Esperar ~90 segundos a que Talos reconfigure.
  - **CA:** Timer de espera completado.
- **ST-02.3.3.3** — Verificar que Talos responde con credenciales nuevas en la Floating IP.
  - **CA:** `talosctl --talosconfig ${TALOSCONFIG} version --nodes ${FLOATING_IP}` → versión correcta.
- **ST-02.3.3.4** — Ejecutar bootstrap de etcd (UNA SOLA VEZ).
  - **CA:** `talosctl bootstrap --nodes ${FLOATING_IP}` ejecuta sin error.
  - **CA:** Se documenta que este comando NUNCA debe ejecutarse de nuevo en este cluster.
- **ST-02.3.3.5** — Esperar ~3 minutos para que el API server esté disponible.
  - **CA:** Timer de espera completado.

#### T-02.3.4 — Obtener kubeconfig y verificar cluster

- **ST-02.3.4.1** — Obtener kubeconfig con `talosctl kubeconfig`.
  - **CA:** Archivo `infra/clusters/hetzner-prod/kubeconfig` generado.
  - **CA:** El kubeconfig apunta a `FLOATING_IP:6443`.
- **ST-02.3.4.2** — Exportar `KUBECONFIG` y verificar nodo.
  - **CA:** `kubectl get nodes -o wide` muestra `serenidad-prod-01` en estado `Ready` con roles `control-plane`, versión `v1.33.x`.
- **ST-02.3.4.3** — Verificar todos los pods del sistema.
  - **CA:** `kubectl get pods -A` → todos los pods de `kube-system` en estado `Running` o `Completed`.
  - **CA:** Pods presentes: flannel, coredns, kube-apiserver, kube-controller-manager, kube-scheduler, etcd.
- **ST-02.3.4.4** — Verificar salud general del cluster con `talosctl health`.
  - **CA:** `talosctl health` → `[OK] etcd is healthy`, `[OK] control plane is healthy`, `[OK] all nodes are ready`.
- **ST-02.3.4.5** — (Opcional) Crear aliases de trabajo (`kp`, `tp`).
  - **CA:** Aliases funcionales en el shell.

#### T-02.3.5 — Crear namespaces del cluster

- **ST-02.3.5.1** — Aplicar manifiesto de namespaces con `kubectl apply`.
  - **CA:** Namespaces creados: `traefik`, `cert-manager`, `serenidad-secrets`, `cnpg-system`, `serenidad-data`, `serenidad-core`, `serenidad-ops`.
- **ST-02.3.5.2** — Verificar labels en namespaces.
  - **CA:** `serenidad-data` y `serenidad-core` tienen label `environment: production`.
  - **CA:** Todos los namespaces tienen label `app.kubernetes.io/managed-by: flux`.
- **ST-02.3.5.3** — Verificar con `kubectl get namespaces`.
  - **CA:** 7 namespaces custom + los de sistema (`default`, `kube-system`, `kube-public`, `kube-node-lease`).

### Criterios de Aceptación de la HU-02.3

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-1 | Kubernetes 1.33.x single-node Ready | `kubectl get nodes` → `Ready control-plane v1.33.x` |
| CA-2 | Todos los pods del sistema Running | `kubectl get pods -A` → 0 pods en error |
| CA-3 | etcd healthy | `talosctl health` → `[OK] etcd is healthy` |
| CA-4 | 7 namespaces custom creados | `kubectl get ns` → traefik, cert-manager, serenidad-secrets, cnpg-system, serenidad-data, serenidad-core, serenidad-ops |
| CA-5 | kubeconfig funcional | `kubectl cluster-info` → API server accesible |
| CA-6 | Secrets de Talos respaldados | `secrets.yaml` en password manager |
| CA-7 | Archivos sensibles en .gitignore | `git status` no muestra archivos de Talos |

### Definition of Done — HU-02.3

- [ ] Cluster Kubernetes 1.33.x operativo y healthy.
- [ ] Todos los pods de sistema en Running.
- [ ] 7 namespaces creados con labels correctos.
- [ ] kubeconfig generado y funcional.
- [ ] Secrets de Talos respaldados en password manager.
- [ ] Archivos sensibles excluidos de Git.

---

## HU-02.4 — Hetzner Cloud Controller Manager (V-04)

**Como** ingeniero de plataforma,
**quiero** tener el Hetzner CCM desplegado y operativo en el cluster,
**para que** Kubernetes reconozca que corre en Hetzner, el nodo pierda el taint `uninitialized`, y tenga metadata de zona disponible.

**Dependencia:** HU-02.3 completada (cluster K8s funcional con namespaces).

### Tareas y Subtareas

#### T-02.4.1 — Crear namespace y Secret para el CCM

- **ST-02.4.1.1** — Crear namespace `hcloud-system`.
  - **CA:** `kubectl get ns hcloud-system` → namespace existe.
- **ST-02.4.1.2** — Crear Secret `hcloud` en `hcloud-system` con el API token de Hetzner.
  - **CA:** `kubectl get secret hcloud -n hcloud-system` → existe con 2 data keys (`token`, `network`).
  - **CA:** El valor de `token` coincide con `$HCLOUD_TOKEN`.

#### T-02.4.2 — Desplegar el CCM

- **ST-02.4.2.1** — Aplicar manifiesto oficial del CCM desde el repositorio hcloud.
  - **CA:** `kubectl apply -f https://github.com/hetznercloud/hcloud-cloud-controller-manager/releases/latest/download/ccm.yaml` ejecuta sin error.
- **ST-02.4.2.2** — Verificar que el deployment del CCM está Running.
  - **CA:** `kubectl get pods -n hcloud-system` → `hcloud-cloud-controller-manager-xxx` en estado `Running`.
- **ST-02.4.2.3** — Verificar tolerations del CCM (debe tolerar nodos uninitialized).
  - **CA:** Pod tiene toleration para `node.cloudprovider.kubernetes.io/uninitialized`.

#### T-02.4.3 — Verificar integración del CCM con el cluster

- **ST-02.4.3.1** — Verificar que el nodo NO tiene taint `uninitialized`.
  - **CA:** `kubectl describe node serenidad-prod-01 | grep Taint` → sin taint del CCM.
- **ST-02.4.3.2** — Verificar labels de topología Hetzner en el nodo.
  - **CA:** `kubectl get node serenidad-prod-01 -o yaml | grep topology.kubernetes.io` → `region: eu-central`, `zone: nbg1` (o equivalente).
- **ST-02.4.3.3** — Verificar logs del CCM sin errores críticos.
  - **CA:** `kubectl logs deployment/hcloud-cloud-controller-manager -n hcloud-system` → sin líneas `level=error` o `FATAL`.

### Criterios de Aceptación de la HU-02.4

| # | Criterio | Verificación |
|---|----------|--------------|
| CA-1 | CCM Running | `kubectl get pods -n hcloud-system` → Running |
| CA-2 | Nodo sin taint uninitialized | `kubectl get node -o json \| jq '.items[0].spec.taints'` → null o sin taint del CCM |
| CA-3 | Labels de topología Hetzner presentes | `topology.kubernetes.io/region` y `zone` presentes |
| CA-4 | Secret del token creado | `kubectl get secret hcloud -n hcloud-system` → existe |
| CA-5 | Sin errores críticos en logs | Logs del CCM sin FATAL ni errores de autenticación |

### Definition of Done — HU-02.4

- [ ] CCM desplegado y en estado Running.
- [ ] Nodo sin taint `uninitialized`.
- [ ] Labels de topología Hetzner visibles en el nodo.
- [ ] Sin errores críticos en logs del CCM.

---

## Resumen de Dependencias Internas EP-02

```
HU-02.1 (VPS + Floating IP + Firewall)
  └──► HU-02.2 (Instalar Talos)
        └──► HU-02.3 (Bootstrap K8s + Namespaces)
              └──► HU-02.4 (Hetzner CCM)
```

Cada HU es estrictamente secuencial: no se puede iniciar la siguiente sin completar la anterior.
