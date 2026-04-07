# Informe de Cambios Tecnológicos Recomendados — Serenidad 2026

| Campo | Valor |
|-------|-------|
| **Proyecto** | Serenamente — Clínica Digital |
| **Tipo** | Informe Técnico de Actualización |
| **Fecha** | Abril 2026 |
| **Autor** | djca + Investigación Tavily |

---

## Resumen Ejecutivo

**Decisión Principal: CAMBIAR de Sealed Secrets a SOPS+age** ✅ Validado como correcto.

Investigación exhaustiva identifica **7 cambios tecnológicos críticos** para garantizar compatibilidad futura (TypeScript 7, Qwik 2+) y mejores prácticas (SOPS para compliance HIPAA).

**Riesgo Global**: MEDIO-ALTO sin cambios, BAJO con implementación de recomendaciones.

---

## 1. SECRET MANAGEMENT: Sealed Secrets → SOPS+age

### ✅ RECOMENDACIÓN: CAMBIAR A SOPS+AGE

**Justificación (evidencia Tavily)**:
- **Multi-cluster**: SOPS superior con age keys por ambiente
- **Compliance HIPAA**: KMS centralizado + audit trails nativos
- **GitOps**: Flux integración nativa (flux.toolkit/sops)
- **Rotación de keys**: `sops rotate` más robusto que sealed-secrets
- **Blast radius**: Age keys distribuidas vs private key única en cluster

**Sealed Secrets debilidades identificadas**:
- Controller private key = single point of failure
- Pérdida de key = SealedSecrets irrecuperables
- Multi-cluster require keys separadas (no reuso)
- Menor control granular sobre IAM/KMS policies

**Implementación SOPS+age**:
```yaml
# .sops.yaml
creation_rules:
  - path_regex: secrets/prod/.*
    age: >-
      age1prod...,
      age1backup...
  - path_regex: secrets/staging/.*
    age: age1staging...
```

**Plan de migración**:
1. Generar age keys por ambiente (prod/staging/dev)
2. Configurar `.sops.yaml` con creation_rules
3. Cifrar secrets existentes: `sops -e -i secret.yaml`
4. Configurar Flux Kustomization con sops decryption
5. Backup age private keys en vault seguro (600 perms)
6. Implementar `sops rotate` en pipeline mensual

**Costo**: $0 (age open source, KMS opcional)

---

## 2. TYPESCRIPT 7.0 — Breaking Changes Críticos

### ⚠️ RIESGO CRÍTICO IDENTIFICADO

**TypeScript 7 "Corsa"**: Reescritura nativa en Go (2026-2027)

**Breaking Changes Confirmados**:
1. **API Strada → Corsa incompatible**: Tooling actual NO funcionará
2. **Stricter defaults by default**: Más errores de tipo
3. **Performance**: ~10× faster, ~70% menos memoria
4. **Watch mode**: Posibles regresiones

**Impacto en Serenidad**:
- Qwik SPA (TS 6.0 → 7.0)
- Hono BFF (TS 6.0 → 7.0)  
- Astro landing (TS 6.0 → 7.0)

**Recomendación**: 
```
MANTENER TypeScript 6.0 como puente
```

TypeScript 6.0 = última versión JS-based, diseñada como puente hacia 7.0.

**Plan de migración TS 7**:
1. **Fase 0 (Q2 2026)**: Instalar `@typescript/native-preview` side-by-side
2. **Fase 1 (Q3 2026)**: CI parallel checks (TS 6 + TS 7 native)
3. **Fase 2 (Q4 2026)**: Fix deprecations señaladas por TS 6
4. **Fase 3 (Q1 2027)**: Canary deploy con TS 7 si Qwik/Astro compatibles
5. **Rollback strategy**: Mantener `typescript@6` en package.json

---

## 3. QWIK 2.0 — Migration Path

### ✅ ADOPCIÓN CORRECTA (ya en descripcion.md)

**Breaking Changes v1 → v2**:
- Package namespace: `@builder.io/*` → `@qwik.dev/*`
- API renames: `useWatch$` → `useTask$`, `useRef` → `useSignal`
- Router: `routeConfig` nuevo formato trie
- Optimizer: Split a `@qwik.dev/optimizer`

**Qwik 2 + TypeScript 7 compatibility**: 🔴 NO CONFIRMADA (evidence gap)

**Recomendación**:
```
Proceder con Qwik 2.0 + TypeScript 6.0
NO actualizar a TS 7 hasta validación oficial Qwik
```

**Migration checklist**:
- [ ] Actualizar imports: `@builder.io/qwik` → `@qwik.dev/core`
- [ ] Reemplazar `useWatch$` → `useTask$`
- [ ] Reemplazar `useRef` → `useSignal`
- [ ] Actualizar `routeLoader$` / `routeAction$`
- [ ] Agregar `@qwik.dev/optimizer` a devDependencies

---

## 4. OTROS COMPONENTES — Status y Recomendaciones

### CloudNativePG ✅ MANTENER
- **Status**: CNCF project, producción-ready
- **PostgreSQL 17**: Soporte confirmado (17.6+ para major upgrades)
- **Kubernetes 1.33+**: Soporte explícito en v1.29+
- **HIPAA**: Técnicamente apto (PGAudit, TLS, PITR)
- **Gap**: No BAA público (requiere vendor como EDB para contrato)

**Acción**: Ninguna. CloudNativePG v1.x correcto.

### NATS JetStream 2.11+ ✅ MANTENER
- **Breaking changes 2.10 → 2.11**: Documentados, migration guide disponible
- **NATS 3.0**: No anunciado (evidence gap)
- **Performance**: Async flush (v2.12), per-message TTL
- **Kubernetes**: Helm charts (operator deprecated)

**Acción**: Actualizar a v2.11.14 (incluido en descripcion.md). NATS 3.0 monitoring.

### Go 1.25+ ✅ MANTENER
- **Green Tea GC**: ~10% reducción GC time (default en 1.26)
- **Small-object allocation**: ~30% faster (1-512 bytes)
- **Breaking**: cgo pkg-config, crypto/ecdh randomness deprecations
- **Controller-runtime/client-go**: Validar versiones

**Acción**: Go 1.25.x correcto. Planear 1.26 en Q3 2026.

### Talos Linux 1.10+ ✅ MANTENER
- **Kubernetes 1.33**: Soporte explícito (v1.11/v1.12)
- **Immutability**: Boot partition 2 GiB, UKI/SecureBoot
- **TPM disk encryption**: Validar PCR configs en upgrades
- **SELinux**: Experimental (solo fresh installs)

**Acción**: Talos 1.10+ correcto. Testing exhaustivo antes de 1.11/1.12.

### FluxCD v2 ✅ MANTENER
- **Roadmap**: v2.9 (Q2 2026), v2.10 (Q3 2026)
- **OCI artifacts**: GA desde v2.6
- **SOPS integration**: Nativa (Mozilla SOPS guide)
- **API migrations**: `flux migrate` tool disponible

**Acción**: FluxCD v2.x correcto. Migrar deprecated APIs antes de v2.9.

---

## 5. TABLA DE CAMBIOS RECOMENDADOS

| # | Componente | Acción | Prioridad | Timing | Esfuerzo |
|---|------------|--------|-----------|--------|----------|
| **1** | **Secret Management** | **Sealed Secrets → SOPS+age** | **CRÍTICA** | **Fase 1** | **2-3 días** |
| 2 | TypeScript | Mantener 6.0, monitorear 7.0 | Alta | Q3-Q4 2026 | 1 semana (testing) |
| 3 | Qwik | v2.0 confirmado correcto | Media | Fase 1 | Ya implementado |
| 4 | CloudNativePG | Mantener v1.x | Baja | - | - |
| 5 | NATS | Mantener 2.11.x | Baja | - | - |
| 6 | Go | Planear 1.26 | Media | Q3 2026 | 2-3 días (testing) |
| 7 | Talos | Validar 1.11/1.12 | Media | Q2-Q3 2026 | 1 semana (staging) |

---

## 6. ROADMAP DE IMPLEMENTACIÓN

### Fase 1 — Inmediato (Abril-Mayo 2026)
```
✅ CRÍTICO: Implementar SOPS+age
- Generar age keys
- Configurar .sops.yaml
- Migrar secrets actuales
- Configurar Flux decryption
- Backup keys en vault
```

### Fase 2 — Q2 2026
```
- Monitorear TypeScript 7 native-preview
- Instalar @typescript/native-preview side-by-side
- CI parallel type-checks
```

### Fase 3 — Q3 2026
```
- Evaluar Go 1.26 (Green Tea GC default)
- Testing Talos 1.11/1.12 en staging
- FluxCD v2.9 migration (deprecated APIs)
```

### Fase 4 — Q4 2026 - Q1 2027
```
- Decision point: TypeScript 7 production (si Qwik compatible)
- NATS 3.0 evaluation (si anunciado)
```

---

## 7. MÉTRICAS DE ÉXITO

| Métrica | Objetivo | Validación |
|---------|----------|------------|
| Secret rotation | < 5 min automated | `sops rotate` pipeline |
| TS 7 compatibility | 100% tests pass | CI parallel checks |
| Upgrade downtime | < 30 min rolling | Talos/k8s upgrades |
| Compliance audit | HIPAA-ready | SOPS KMS + audit logs |

---

## 8. RIESGOS Y MITIGACIONES

### RIESGO CRÍTICO: TypeScript 7 incompatibilidad
**Mitigación**: Side-by-side install, mantener TS 6 hasta Qwik/Astro validados

### RIESGO ALTO: Secret key loss (SOPS age)
**Mitigación**: Backup age keys en 3 locations, 600 perms, rotation mensual

### RIESGO MEDIO: Talos upgrade failures
**Mitigación**: Staging identical hardware, etcd snapshots, maintenance ISO ready

---

## CONCLUSIÓN

**Cambio a SOPS+age: VALIDADO Y RECOMENDADO** ✅

La investigación confirma que SOPS+age es superior para:
- Multi-cluster healthcare architecture
- HIPAA compliance (KMS + audit)
- Disaster recovery (distributed keys)
- GitOps best practices

**Próximos Pasos Inmediatos**:
1. Implementar SOPS+age (Fase 1, 2-3 días)
2. Setup TypeScript 7 monitoring (CI preview)
3. Validar stack en staging antes de producción

**Firma Digital**: Investigación completada con Tavily Pro Research (Abril 2026)
