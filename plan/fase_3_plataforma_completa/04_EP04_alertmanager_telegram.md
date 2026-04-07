# EP-04 — Alertmanager → Telegram Integration

**Épica**: Como equipo de operaciones, necesito integración nativa Alertmanager → Telegram para recibir alertas críticas en tiempo real en grupo Telegram dedicado con templates HTML, grouping por severity, y send_resolved automático, para respuesta rápida a incidentes 24/7.

**Origen**: Fase 3 (3.D) + research/09_evaluacion_tecnologica_fase3_plataforma_completa.md  
**Prioridad**: Media  
**Sprint**: S9 (1 semana)  
**Dependencias Entrantes**: EP-03 (Alertmanager deployed)  
**Dependencias Salientes**: Ninguna

---

## HU-04.1 — Telegram Bot Creation

**Como** SRE, **quiero** crear bot Telegram con token seguro en Kubernetes Secret, **para que** Alertmanager pueda enviar notificaciones.

### Tareas

#### T-04.1.1 — Crear Bot vía BotFather

```bash
# 1. Buscar @BotFather en Telegram
# 2. /newbot
# 3. Nombre: "Serenidad Alerts Bot"
# 4. Username: serenidad_alerts_bot
# 5. Copiar bot token: 1234567890:ABCdefGHIjklMNOpqrsTUVwxyz
```

**CA**: Bot creado, token obtenido

#### T-04.1.2 — Crear Grupo Telegram

```bash
# 1. Crear grupo Telegram: "Serenidad Alerts"
# 2. Agregar @serenidad_alerts_bot al grupo
# 3. Enviar mensaje test en grupo
# 4. Obtener chat_id:
curl https://api.telegram.org/bot<TOKEN>/getUpdates

# Buscar chat.id en response (será negativo para grupos, ej: -1001234567890)
```

**CA**: Grupo creado, chat_id obtenido

#### T-04.1.3 — Kubernetes Secret

```bash
kubectl create secret generic telegram-bot-token \
  --from-literal=token='1234567890:ABCdefGHIjklMNOpqrsTUVwxyz' \
  --from-literal=chat_id='-1001234567890' \
  -n monitoring
```

**CA**: Secret creado, verificado

---

## HU-04.2 — Alertmanager Native Receiver

**Como** platform engineer, **quiero** configurar native telegram_configs receiver en Alertmanager, **para que** alertas se envíen automáticamente sin relay adicional.

### Tareas

#### T-04.2.1 — Alertmanager Config

Crear `infrastructure/k8s/monitoring/alertmanager-config.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-telegram-config
  namespace: monitoring
stringData:
  alertmanager.yaml: |
    global:
      resolve_timeout: 5m
    
    templates:
      - '/etc/alertmanager/templates/*.tmpl'
    
    route:
      receiver: telegram-critical
      group_by: ['alertname', 'namespace', 'severity']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      
      routes:
        - match:
            severity: critical
          receiver: telegram-critical
          repeat_interval: 1h
          group_wait: 10s
        
        - match:
            severity: warning
          receiver: telegram-warning
          repeat_interval: 4h
    
    receivers:
      - name: telegram-critical
        telegram_configs:
          - bot_token_file: /etc/telegram/token
            chat_id: <CHAT_ID>  # Replace with actual chat_id from Secret
            parse_mode: HTML
            send_resolved: true
            message: |
              {{ range .Alerts }}
              <b>🔴 {{ .Labels.alertname }}</b>
              <b>Severity:</b> {{ .Labels.severity }}
              <b>Namespace:</b> {{ .Labels.namespace }}
              {{ if .Labels.service }}<b>Service:</b> {{ .Labels.service }}{{ end }}
              
              <b>Description:</b>
              {{ .Annotations.description }}
              
              <b>Started:</b> {{ .StartsAt | date "2006-01-02 15:04:05" }}
              {{ if .EndsAt }}
              <b>Resolved:</b> {{ .EndsAt | date "2006-01-02 15:04:05" }}
              {{ end }}
              
              <a href="{{ .GeneratorURL }}">View in Prometheus</a>
              ---
              {{ end }}
      
      - name: telegram-warning
        telegram_configs:
          - bot_token_file: /etc/telegram/token
            chat_id: <CHAT_ID>
            parse_mode: HTML
            send_resolved: true
            message: |
              {{ range .Alerts }}
              <b>⚠️ {{ .Labels.alertname }}</b>
              <b>Severity:</b> {{ .Labels.severity }}
              <b>Description:</b> {{ .Annotations.description }}
              {{ end }}
```

**CA**: Config YAML creado

#### T-04.2.2 — Mount Token Secret

Modificar Helm values `infrastructure/k8s/monitoring/prometheus-values.yaml`:

```yaml
alertmanager:
  alertmanagerSpec:
    replicas: 3
    
    volumes:
      - name: telegram-secret
        secret:
          secretName: telegram-bot-token
    
    volumeMounts:
      - name: telegram-secret
        mountPath: /etc/telegram
        readOnly: true
    
    configSecret: alertmanager-telegram-config
```

**CA**: Secret mounted, Alertmanager config aplicado

#### T-04.2.3 — Apply Config

```bash
# Update Helm release
helm upgrade prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values infrastructure/k8s/monitoring/prometheus-values.yaml \
  --version 82.8.0

# Verify Alertmanager pods restarted
kubectl rollout status statefulset/alertmanager-prometheus-stack-kube-prom-alertmanager -n monitoring
```

**CA**: Alertmanager pods running con nueva config

---

## HU-04.3 — Testing y Validation

**Como** SRE, **quiero** test alert delivery end-to-end, **para que** verifique grouping, templates y resolved notifications funcionan.

### Tareas

#### T-04.3.1 — Trigger Test Alert

Crear `infrastructure/k8s/monitoring/test-alert.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: alert-test-trigger
  namespace: serenidad
  labels:
    app: test-trigger
spec:
  containers:
    - name: trigger
      image: alpine:latest
      command: ['sh', '-c', 'exit 1']
  restartPolicy: Never
```

```bash
kubectl apply -f infrastructure/k8s/monitoring/test-alert.yaml

# Pod fallará → ServiceDown alert después 1min
```

**CA**: Test alert triggered

#### T-04.3.2 — Verify Telegram Delivery

Verificar en grupo Telegram:
- Mensaje HTML formateado correctamente
- Severity, namespace, description visibles
- Link a Prometheus funcional
- Grouped alerts en mismo mensaje

**CA**: Mensaje recibido en Telegram con formato correcto

#### T-04.3.3 — Verify Resolved Notification

```bash
# Delete test pod → alert se resuelve
kubectl delete pod alert-test-trigger -n serenidad

# Esperar ~5min → resolved notification
```

**CA**: Resolved message recibido en Telegram

---

## Resumen Entregables EP-04

**Telegram**:
- Bot creado (@serenidad_alerts_bot)
- Grupo Telegram ("Serenidad Alerts")
- Token en Kubernetes Secret

**Alertmanager**:
- Native telegram_configs configurado
- Templates HTML custom
- Routing por severity (critical 1h repeat, warning 4h repeat)
- Grouping por alertname + namespace
- Send resolved enabled

**Testing**:
- Test alert delivery funcional
- Grouping validado
- Resolved notifications validadas
- Template rendering correcto

**Validación**:
```bash
# 1. Verificar Alertmanager config
kubectl get secret alertmanager-telegram-config -n monitoring -o yaml

# 2. Check Alertmanager logs
kubectl logs -n monitoring alertmanager-prometheus-stack-kube-prom-alertmanager-0 | grep telegram

# 3. Trigger manual test alert
kubectl apply -f infrastructure/k8s/monitoring/test-alert.yaml

# 4. Verificar en Telegram
# → Mensaje debe aparecer en <2min

# 5. Verificar resolved
kubectl delete pod alert-test-trigger -n serenidad
# → Resolved message en <5min
```

**Alerting Rules Activas** (de EP-03):
- ServiceDown (critical, 1m)
- HighErrorRate (warning, 2m)
- PostgreSQLConnectionPoolExhausted (critical, 5m)
- NATSStreamHighLag (warning, 5m)

---

**Tiempo estimado EP-04**: 1 semana (Sprint 9)  
**Esfuerzo**: ~15 horas configuración + 5 horas testing
