# PLAN DE CONSTRUCCIÓN DEL NUEVO POS
## Etapa 5 — Cómo se construye el POS "como si hubiera nacido así"

> **Documento complementario** al [`PLANO ARQUITECTONICO PARA EL NUEVO POS.md`](../PLANO%20ARQUITECTONICO%20PARA%20EL%20NUEVO%20POS.md:1),
> al [`PLAN_ACCION_ARQUITECTONICO_NUEVO_POS.md`](../ESPECIFICACIONES%20DEL%20PROYECTO/PLAN_ACCION_ARQUITECTONICO_NUEVO_POS.md:1)
> y al [`MODELO_DESPLIEGUE_Y_CONSOLIDACION_CENTRAL.md`](../ESPECIFICACIONES%20DEL%20PROYECTO/MODELO_DESPLIEGUE_Y_CONSOLIDACION_CENTRAL.md:1).
> El plano dice **qué debe ser**. El plan de acción dice **qué acciones ejecutar**. El modelo de
> despliegue dice **dónde vive**. Este documento dice **en qué orden se construye y cómo se
> verifica cada paso**.
>
> **Regla dura:** no se toca el ERP. Este documento es un artefacto de diseño. No contiene
> código de producción.
>
> **Anclaje:** commit `5802f45` (V23) del ERP actual; ingeniería inversa sobre `fe9f6ed`
> (tag `v22-estable-fe9f6ed`).

---

## SECCIÓN 0 — POR QUÉ EXISTE ESTE DOCUMENTO

### 0.1 El problema que resuelve

Los documentos anteriores describen **el destino** (el plano) y **las reglas del viaje**
(las 5 acciones). Ninguno describe **la secuencia de obra**: qué se construye primero,
qué depende de qué, y cómo se sabe que un paso terminó antes de empezar el siguiente.

Sin esta secuencia, el riesgo es el mismo que produjo el "parcheo por acumulación" del POS
actual: construir por urgencia, no por dependencia. Este documento impone el orden.

### 0.2 El principio rector

> **Se construye de adentro hacia afuera: primero el cimiento (datos), luego la frontera
> (contratos), luego el comportamiento (reglas + tests), luego la superficie (interfaces),
> y al final la consolidación (central).**

Cada capa solo puede empezar cuando la anterior está **verificada**, no cuando está "escrita".

### 0.3 Lo que este documento NO es

- **No es un cronograma.** No hay fechas ni estimaciones de esfuerzo. Hay **dependencias**.
- **No es código.** Describe *qué* construir y *cómo verificarlo*, no *cómo escribirlo*.
- **No es una reescritura del POS actual.** El POS actual sigue vivo y no se toca.

---

## SECCIÓN 1 — LAS 7 FASES DE CONSTRUCCIÓN

La obra se divide en 7 fases. Cada fase tiene: **entrada** (qué necesita), **salida**
(qué produce) y **puerta** (el criterio que debe pasar para avanzar).

| Fase | Nombre | Entrada | Salida | Puerta |
|------|--------|---------|--------|--------|
| **F0** | Andamiaje | Nada | Repo nuevo, CI, esqueleto de carpetas | CI corre en verde con 0 tests |
| **F1** | Cimiento de datos | F0 | 17 tablas con UUID/UTC/ledger | Migraciones aplican y revierten limpias |
| **F2** | Frontera (contratos) | F1 | 17 contratos entre módulos | Test de arquitectura: 0 imports ajenos |
| **F3** | Comportamiento (reglas) | F2 | 81 reglas portadas **con sus tests** | Matriz `regla → test` completa |
| **F4** | Guardianes | F3 | Test guardián por regla crítica | CI falla si se viola una regla crítica |
| **F5** | Superficie (interfaces) | F4 | 26 interfaces del POS | Paridad funcional con el POS actual |
| **F6** | Consolidación (central) | F5 | Outbox + contrato de consolidación | Sync de cierre de día verificada |

> **Regla de avance:** ninguna fase empieza sin que la **puerta** de la anterior esté en verde.
> Si una puerta falla, se corrige la fase actual; no se avanza "dejando pendiente".

---

## SECCIÓN 2 — FASE 0: ANDAMIAJE

### 2.1 Objetivo

Crear el proyecto nuevo **al lado** del ERP, sin tocar el ERP. Un repositorio limpio con
CI funcionando y la estructura de carpetas vacía pero definida.

### 2.2 Entregables

- Repositorio nuevo (separado del ERP).
- Pipeline de CI que corre en cada push (lint + tests).
- Esqueleto de carpetas según la estructura del plano.
- `README` que declara la regla dura y el anclaje.

### 2.3 Puerta de salida

- [ ] El CI corre en verde con **0 tests** (el pipeline funciona aunque no haya pruebas).
- [ ] El repo del ERP sigue con `git status` limpio y HEAD `5802f45`.
- [ ] La estructura de carpetas coincide con la del plano.

---

## SECCIÓN 3 — FASE 1: CIMIENTO DE DATOS

### 3.1 Objetivo

Construir el modelo de datos **antes** que cualquier lógica. Es el cimiento: si la identidad
(UUID), el tiempo (UTC) y el inventario (ledger) no nacen correctos, todo lo que se construya
encima hereda el defecto.

### 3.2 Entregables

- Las **17 tablas** del [`MODELO_DE_DATOS_DEL_NUEVO_POS.md`](../MODELO_DE_DATOS_DEL_NUEVO_POS.md:1).
- **C-01:** PK entero → **UUID** en todas las tablas.
- **C-02:** `DateTime` naive → **`DateTime(timezone=True)` UTC**.
- **C-03:** dinero en **`Numeric(12,2)`** (nunca `Float`).
- **C-04:** columna `version` para **bloqueo optimista**.
- El **ledger de inventario** (asientos inmutables; el stock se deriva, no se sobrescribe).

### 3.3 Puerta de salida

- [ ] Las migraciones **aplican y revierten** sin error.
- [ ] Ninguna columna de dinero es `Float` (verificable por consulta al esquema).
- [ ] Ningún `DateTime` es naive (verificable por consulta al esquema).
- [ ] El ledger rechaza un `UPDATE` directo (verificable por test).

---

## SECCIÓN 4 — FASE 2: FRONTERA (CONTRATOS)

### 4.1 Objetivo

Definir **cómo el POS habla con los demás módulos** antes de escribir una sola regla de
negocio. La frontera es lo que el POS actual **no** tiene: hoy lee tablas ajenas
(`Product`, `Order`, `WarehouseEvent`, `Employee`). Aquí se prohíbe.

### 4.2 Entregables

- Los **17 contratos** del [`CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md`](../CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md:1).
- **A-02:** frontera por contratos + **prohibición de leer tablas ajenas**.
- El **test de arquitectura** que falla si el POS importa un modelo de otro módulo.

### 4.3 Puerta de salida

- [ ] El test de arquitectura pasa: **0 imports** del POS a modelos ajenos.
- [ ] Cada contrato tiene su firma (entrada/salida) documentada.
- [ ] Ningún contrato expone una tabla; todos exponen una operación.

---

## SECCIÓN 5 — FASE 3: COMPORTAMIENTO (REGLAS + TESTS)

### 5.1 Objetivo

Portar las **81 reglas de negocio** (RN-01 a RN-81) del POS actual, **cada una con su test**.
La unidad de migración no es la regla: es **regla + test**.

### 5.2 Entregables

- Las **81 reglas** implementadas sobre los contratos de F2.
- **A-01:** cada regla portada **con su test** (no se acepta una regla sin prueba).
- La **matriz `regla → test`** completa (trazabilidad).

### 5.3 Puerta de salida

- [ ] La matriz `regla → test` está **completa** (81 de 81).
- [ ] Ninguna regla migró sin test (verificable por la matriz).
- [ ] Las **cicatrices** (DRAFT GUARD, anti-degradación, bloqueo optimista, reciclaje de
      folios, idempotencia de emergencia) están presentes y probadas.

---

## SECCIÓN 6 — FASE 4: GUARDIANES

### 6.1 Objetivo

Convertir cada regla crítica en un **test guardián** que **falla si la regla se viola**.
Una regla no automatizada es solo una intención (RN-81 fue violada precisamente por eso).

### 6.2 Entregables

- **A-03:** un test guardián por regla crítica.
- **A-04:** **Outbox transaccional** (el evento vive en la misma transacción; elimina
  `try/except pass`).
- **A-05:** separación explícita **identidad (UUID) ≠ presentación (folio)**.

### 6.3 Puerta de salida

- [ ] El CI **falla** si se viola una regla crítica (verificable rompiendo una a propósito).
- [ ] **0** `try/except pass` en la ruta crítica (búsqueda automatizada en CI).
- [ ] Ninguna regla de negocio usa el folio como identidad.

---

## SECCIÓN 7 — FASE 5: SUPERFICIE (INTERFACES)

### 7.1 Objetivo

Construir las **26 interfaces** del POS sobre el comportamiento ya probado. La superficie
es lo último porque depende de todo lo anterior: no se pinta una pared antes del cimiento.

### 7.2 Entregables

- Las **26 interfaces** del [`ESPECIFICACION_DE_INTERFACES_POS.md`](../ESPECIFICACION_DE_INTERFACES_POS.md:1).
- La **regla responsiva** del [`ESPECIFICACION_RESPONSIVA_Y_ERGONOMIA_TACTIL.md`](../ESPECIFICACION_RESPONSIVA_Y_ERGONOMIA_TACTIL.md:1)
  (los 28 contenedores nacen fluidos, no de ancho fijo).
- La **paleta canónica** (`#c1d72e`, `#0a0a0a`, `#fdfbf7`, `rounded-[35px]`).

### 7.3 Puerta de salida

- [ ] **Paridad funcional** con el POS actual (los 6 flujos E.1 a E.6 replicados).
- [ ] Ningún contenedor crítico tiene ancho fijo en píxeles.
- [ ] La paleta canónica se respeta en las 26 interfaces.

---

## SECCIÓN 8 — FASE 6: CONSOLIDACIÓN (CENTRAL)

### 8.1 Objetivo

Conectar cada sucursal con el servidor central, **al final**, cuando el POS ya funciona
solo. La consolidación es una capa asíncrona de cierre de día, no un requisito del cobro.

### 8.2 Entregables

- El **contrato de consolidación** (principios P1-P6, payload JSON, outbox).
- La **sync al cierre del día** (default `23:30`, configurable en `SystemSetting`).
- La **resolución de conflictos** (tabla `sync_conflictos`, revisión manual, nunca silenciosa).

### 8.3 Puerta de salida

- [ ] La sync de cierre de día envía ventas y caja al central.
- [ ] Si el central cae, **las sucursales siguen operando** (el central no es transaccional).
- [ ] Un conflicto se registra para revisión manual; **nunca** se resuelve en silencio.

---

## SECCIÓN 9 — MATRIZ DE TRAZABILIDAD (FASES ↔ ACCIONES ↔ HALLAZGOS)

| Fase | Acciones que ejecuta | Hallazgos que cierra |
|------|----------------------|----------------------|
| **F1 — Cimiento** | (C-01 a C-04 del modelo de datos) | DEUDA-01 (dinero Float), DEUDA-02 (PK entero), DB-01 (DateTime naive) |
| **F2 — Frontera** | **A-02** | AC-01 a AC-10 (los 10 acoplamientos) |
| **F3 — Comportamiento** | **A-01** | DEUDA-03, DEUDA-05 (reglas sin test) |
| **F4 — Guardianes** | **A-03, A-04, A-05** | DEUDA-04 (`try/except pass`), DB-02 (folio como identidad), DB-04 (offset hardcodeado) |
| **F5 — Superficie** | (regla responsiva) | DB-05 a DB-10 (debilidades de interfaz) |
| **F6 — Consolidación** | (contrato de consolidación) | RC-01 a RC-04 (riesgos de concurrencia) |

> **Nota:** los códigos de hallazgo (DEUDA-XX, AC-XX, DB-XX, RC-XX) provienen de la
> [`ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md`](../ESPECIFICACIONES%20DEL%20PROYECTO/ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md:1) §F.

---

## SECCIÓN 10 — EL ORDEN DE LECTURA PARA QUIEN CONSTRUYE

```
1. GUÍA MAESTRA          → contexto y regla dura
        ↓
2. PLANO                 → qué debe ser
        ↓
3. ESPECIFICACIÓN        → qué hace hoy (fuente de verdad funcional)
        ↓
4. PLAN DE ACCIÓN        → las 5 acciones obligatorias
        ↓
5. MODELO DE DESPLIEGUE  → dónde vive
        ↓
6. ESTE DOCUMENTO        → en qué orden se construye y cómo se verifica
```

**Principio:** primero el **contexto**, luego el **diseño**, luego la **realidad**,
luego la **ejecución**, luego el **despliegue**, y al final **la secuencia de obra**.

---

## SECCIÓN 11 — DECLARACIÓN DE LA REGLA DURA

> **NO SE TOCA EL ERP INSTALADO Y CORRIENDO.**
> **NO SE TOCA NINGUNO DE SUS MÓDULOS.**
> **NO SE MODIFICA NI UNA LÍNEA DEL POS ACTUAL.**

Este documento es un artefacto de diseño. No contiene código de producción.
El ERP permanece intacto y operando; su HEAD es `5802f45` (V23) al 22 Sep 2026. La ingeniería
inversa se hizo sobre `fe9f6ed` (tag `v22-estable-fe9f6ed`).

---

*Plan de construcción del Nuevo POS. Versión 1.0. Anclado al commit `5802f45` (V23);
ingeniería inversa sobre `fe9f6ed`. 7 fases, 6 puertas de verificación.*
