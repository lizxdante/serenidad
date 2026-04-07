# Arquitectura Actualizada — Serenidad 2026

| Campo | Valor |
|-------|-------|
| **Proyecto** | Serenidad — Clínica Digital |
| **Versión** | v2026 — Post-Investigación |
| **Fecha** | Abril 2026 |

---

## CAMBIOS PRINCIPALES vs Versión Anterior

### 1. SECRET MANAGEMENT ⚠️ CAMBIO CRÍTICO

**ANTES**: Sealed Secrets v0.27+
**AHORA**: **SOPS + age**

```yaml
# Nuevo componente agregado
| Secret Management | SOPS + age | latest | Cifrado GitOps, age keys por ambiente |
```

**Razón**: 
- Multi-cluster superior
- HIPAA compliance (KMS + audit)
- Rotación de keys más robusta
- Sin single point of failure

### 2. TYPESCRIPT — Estrategia Actualizada

**Versión actual**: TypeScript 6.0 (correcto como puente)
**TS 7.0**: Reescritura Go, breaking changes API

**Estrategia**:
- Mantener TS 6.0 en 2026
- Monitorear TS 7 native-preview
- Side-by-side testing en CI
- NO migrar hasta Qwik 2 + TS 7 validados oficialmente

### 3. OTROS COMPONENTES — Sin Cambios Necesarios

| Componente | Status | Acción |
|------------|--------|--------|
| Qwik 2.0 | ✅ Correcto | Ninguna |
| CloudNativePG | ✅ Correcto | Ninguna |
| NATS 2.11.x | ✅ Correcto | Ninguna |
| Talos 1.10.x | ✅ Correcto | Testing 1.11+ |
| FluxCD v2.x | ✅ Correcto | Ninguna |
| Go 1.25.x | ✅ Correcto | Planear 1.26 Q3 |

---

## STACK ACTUALIZADO (Solo Cambios)

### 3.x Secret Management (NUEVO)

| Componente | Tecnología | Versión | Notas |
|-----------|------------|---------|-------|
| GitOps Secrets | **SOPS + age** | **latest** | Reemplaza Sealed Secrets. Age keys por ambiente. |
| Encryption Keys | age-keygen | latest | Keys distribuidas, backup en vault |
| Flux Integration | Mozilla SOPS | native | Flux Kustomization decryption automática |

**Configuración**:
```yaml
# .sops.yaml
creation_rules:
  - path_regex: secrets/prod/.*
    age: >-
      age1prod_primary...,
      age1prod_backup...
  - path_regex: secrets/staging/.*
    age: age1staging...
```

**Migration de Sealed Secrets**:
1. Generar age keys: `age-keygen -o key.txt`
2. Crear `.sops.yaml` en repo
3. Cifrar: `sops -e -i secret.yaml`
4. Update Flux Kustomization:
```yaml
spec:
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```
5. Backup age private keys (permisos 600)
6. Rotation mensual: `sops rotate --add-age NEW --rm-age OLD`

---

## ADR ACTUALIZADO

### ADR-015: SOPS + age como Secret Management (NUEVO)

**Decisión**: SOPS + age reemplaza Sealed Secrets para gestión de secretos en GitOps.

**Razón**: 
- Multi-cluster: age keys separadas por ambiente vs sealed-secrets key única
- HIPAA: KMS centralizado opcional + audit trails nativos
- DR: Keys distribuidas, no single point of failure
- Rotación: `sops rotate` más robusto
- GitOps: Integración nativa Flux

**Rechazado**: Sealed Secrets (controller private key = SPOF, pérdida key = irrecuperable, multi-cluster complejo)

### ADR-016: TypeScript 6.0 como Puente a TS 7 (ACTUALIZADO)

**Decisión**: Mantener TypeScript 6.0, NO migrar a TS 7 hasta 2027.

**Razón**:
- TS 7 "Corsa" = reescritura Go con breaking changes API
- TS 6.0 = última versión JS-based, diseñada como puente
- Qwik 2 + TS 7 compatibility NO confirmada
- Performance wins TS 7 (~10×) no justifican riesgo sin validación

**Plan**: Side-by-side testing Q3 2026, migration Q1 2027 si ecosistema ready.

---

## ROADMAP ACTUALIZADO

### Fase 1 — Ahora (Abril-Mayo 2026)
```
CRÍTICO: Implementar SOPS + age
- Generar age keys (prod/staging/dev)
- Configurar .sops.yaml
- Migrar secrets existentes
- Flux Kustomization decryption
- Backup keys en vault seguro
```

### Fase 2 — Q2-Q3 2026
- TypeScript 7 monitoring (native-preview)
- Go 1.26 evaluation (Green Tea GC)
- Talos 1.11/1.12 staging tests

### Fase 4+ — Q1 2027+
- TypeScript 7 migration (si validado)
- NATS 3.0 (si anunciado)

---

## REFERENCIAS ACTUALIZADAS

| # | Documento | Cambios |
|---|-----------|---------|
| [NEW] | `informe_cambios_2026.md` | Investigación Tavily completa |
| [1] | `descripcion.md` | Versión original (NO sobrescrita) |

---

**Validación**: Investigación Tavily Pro, Abril 2026
