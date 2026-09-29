# 🧱 PLAN DE ABORDAJE — FASE 4 POR PARTES (Gestor de Caja)

> **Fecha:** 29 Sep 2026
> **Versión:** 1.0
> **Autor:** Antigravity + Víctor (dueño de R de Rico)
> **Estado:** PROPUESTO — pendiente de aprobación del dueño antes de escribir código
> **Documento padre:** [`PLAN_MAESTRO_DEFINITIVO_POS.md`](../PLAN_MAESTRO_DEFINITIVO_POS.md:304) §7 (Fase 4)
> **Contrato de comportamiento:** [`PROMPT_DEL_ARQUITECTO_DEL_NUEVO_POS.md`](../06-prompt-del-arquitecto/PROMPT_DEL_ARQUITECTO_DEL_NUEVO_POS.md:1)
> **Precedente:** [`PLAN_DE_ABORDAJE_FASE_3_POR_PARTES.md`](PLAN_DE_ABORDAJE_FASE_3_POR_PARTES.md:1) v1.2 (Fase 3, cerrada y en verde)

> [!IMPORTANT]
> **Este documento es un artefacto de diseño.** No contiene código de producción.
> El código va en `NUEVO-POS`; los planos van en `PLANOS-ARQUITECTONICOS` (Plan Maestro §1).

> [!CAUTION]
> **LECTURA OBLIGATORIA ANTES DE CUALQUIER OTRA COSA: §3 (el defecto de lectura del `estado_hoy`).**
> La columna "Estado hoy" de la matriz de contratos describe el **ERP VIEJO**, no `NUEVO-POS`.
> Leerla como si describiera `NUEVO-POS` produce un diagnóstico falso. §3 lo desambigua.

---

## 0. PROPÓSITO DE ESTE DOCUMENTO

La Fase 4 ("Gestor de Caja") es la **cuarta rebanada vertical** del Plan Maestro §7. El diagnóstico §3 del Plan Maestro la lista como una de las piezas que DeepSeek **NO construyó** (56 KB en el POS viejo), y el §9 la incluye como resultado esperado #4: *"Gestor de caja → Abrir/cerrar turno, cortes"*.

Este documento:

1. **Registra el diagnóstico** del estado real de `NUEVO-POS` al 29 Sep 2026 para la caja.
2. **Justifica** por qué la Fase 4 se parte en 5 sub-fases (4.0 prerrequisito + 4.1–4.4).
3. **Define** cada sub-fase con su puerta verificable y su ficha de evidencia.
4. **Se autocritica** contra la evidencia del repositorio (§10) y corrige los defectos hallados.
5. **Delimita** qué NO entra en Fase 4 (§9).

---

## 1. DIAGNÓSTICO — ESTADO REAL DE LA CAJA EN `NUEVO-POS`

### 1.1 Lo que YA existe (cimiento)

| Pieza | Evidencia | Estado |
|---|---|---|
| Tabla `cash_sessions` | [`apps/api/models/cash.py:18`](../../NUEVO-POS/apps/api/models/cash.py:18) + migración `0001` | ✅ Existe |
| Tabla `cash_movements` | [`apps/api/models/cash.py:55`](../../NUEVO-POS/apps/api/models/cash.py:55) + migración `0001` | ✅ Existe |
| Columna `Ticket.cash_session_id` | [`apps/api/models/pos.py:99`](../../NUEVO-POS/apps/api/models/pos.py:99) | ✅ Existe (pero **nunca se puebla**) |
| Reglas RN-49 a RN-60 (categorías C.8 y C.9) | [`apps/api/rules/registry.py:401`](../../NUEVO-POS/apps/api/rules/registry.py:401) | ✅ Implementadas y probadas |
| Contratos 9–14 declarados | [`apps/api/contracts/registry.py:219`](../../NUEVO-POS/apps/api/contracts/registry.py:219) | ✅ Declarados |

### 1.2 Lo que NO existe (el hueco real)

| Pieza | Evidencia de la ausencia | Impacto |
|---|---|---|
| **`routers/cash.py`** | `list_files` de `routers/` → solo `catalog.py`, `ia.py`, `pos.py` | Los 6 endpoints `/cash/*` no existen |
| **`include_router(cash.router)`** | [`apps/api/main.py:68`](../../NUEVO-POS/apps/api/main.py:68) solo incluye catalog, pos, ia | Los endpoints no se sirven |
| **Esquemas Pydantic de caja** | [`apps/api/schemas.py`](../../NUEVO-POS/apps/api/schemas.py:1) no tiene `AbrirTurnoEntrada` ni similares | Sin contrato HTTP |
| **`cashService.js`** | No existe en `apps/pos/src/services/` | El POS no puede hablar con la caja |
| **`GestorDeCaja.jsx`** | No existe en `apps/pos/src/` | No hay pantalla de caja |
| **`CorteTicketTemplate.jsx`** | No existe en `apps/pos/src/components/` | No hay plantilla de corte |
| **Poblado de `Ticket.cash_session_id`** | [`routers/pos.py:274`](../../NUEVO-POS/apps/api/routers/pos.py:274) `cobrar_ticket` no lo asigna | **El arqueo daría siempre cero** |

### 1.3 La dependencia oculta (hallazgo de la autocrítica)

El contrato 13 (`caja.cerrar_turno`) devuelve `esperado` vs `capturado`. El `esperado` se calcula con **RN-53**:

```text
efectivo_esperado = fondo + entradas − salidas + ventas_en_efectivo
```

Las "ventas en efectivo" salen de los **tickets cobrados** de la sesión, ligados por `Ticket.cash_session_id`. Pero [`cobrar_ticket`](../../NUEVO-POS/apps/api/routers/pos.py:274) **nunca asigna `cash_session_id`**. Por lo tanto, sin resolver esto **primero**, el Gestor de Caja sería una cáscara: abriría y cerraría turnos, pero el arqueo daría **cero**.

> **Conclusión del diagnóstico:** la Fase 4 tiene un **prerrequisito** (4.0) que la Fase 3 no resolvió, porque la Fase 3 solo entregó el flujo E.1 (venta directa) y la caja estaba fuera de su alcance (§10.3 del plan de Fase 3).

---

## 2. POR QUÉ SE PARTE EN SUB-FASES

La Fase 4 del Plan Maestro §7 lista **3 archivos** (`GestorDeCaja.jsx`, `cashService.js`, `CorteTicketTemplate.jsx`) porque **asume que el backend de caja ya existe**. La evidencia (§1.2) demuestra que **no existe**. Por eso el corte real es de **5 sub-fases**, derivado del orden interno de rebanada vertical (`Endpoint → Servicio → Test → Componente`) más el prerrequisito detectado:

| Sub-fase | Rol en la rebanada | Nivel |
|---|---|---|
| **4.0** prerrequisito | Ligar el ticket a la sesión de caja al cobrar | Negocio (dato) |
| **4.1** backend | `routers/cash.py` + esquemas + `include_router` | Negocio (dato + contrato) |
| **4.2** servicio | `cashService.js` | Negocio (comportamiento) |
| **4.3** pantalla | `GestorDeCaja.jsx` | Negocio (superficie) |
| **4.4** plantilla | `CorteTicketTemplate.jsx` | Negocio (superficie) |

---

## 3. EL DEFECTO DE LECTURA DEL `estado_hoy` (desambiguación permanente)

> **Propósito:** que este error de lectura **no pueda repetirse**.

**El hecho:** la matriz de contratos ([`contracts/registry.py:7-26`](../../NUEVO-POS/apps/api/contracts/registry.py:7)) tiene una columna **"Estado hoy"** que dice `Ya existe` para los contratos 9–14 (caja). Es tentador leer eso como *"los endpoints de caja ya están construidos en NUEVO-POS"*. **Es falso.**

**Lo que significa realmente:** esa matriz proviene del **Documento 9 §10** — la **ingeniería inversa del ERP VIEJO**. La columna describe el estado del **ERP en operación**, no de `NUEVO-POS`. Prueba: el contrato 1 dice `Ya existe (parcial)` y **sí está implementado** en NUEVO-POS; los contratos 9–14 dicen `Ya existe` porque el **ERP viejo** tiene módulo de caja, pero en NUEVO-POS **no hay `routers/cash.py`**.

**La regla de decisión (aplicar siempre):**

1. **¿La columna `estado_hoy` describe NUEVO-POS o el ERP viejo?** → **El ERP viejo.** Es la matriz de ingeniería inversa.
2. **¿Cómo sé si un contrato está implementado en NUEVO-POS?** → **Buscando el endpoint en `routers/`**, no leyendo `estado_hoy`.
3. **¿Hay conflicto entre `estado_hoy` y el código?** → **Gana el código.** Es la evidencia (E-14).

**Firma de la desambiguación:** este registro se añadió en la v1.0 tras detectar que la propuesta inicial de Fase 4 leyó `estado_hoy` como si describiera NUEVO-POS (defecto D-1 de §10).

---

## 4. SUB-FASE 4.0 — PRERREQUISITO: LIGAR EL TICKET A LA SESIÓN DE CAJA

> **Por qué existe:** sin esto, el arqueo da cero (§1.3). Es la dependencia oculta que la Fase 3 no resolvió.

### 4.0.1 Qué se construye

| Archivo | Cambio |
|---|---|
| [`apps/api/routers/pos.py`](../../NUEVO-POS/apps/api/routers/pos.py:274) | `cobrar_ticket` asigna `ticket.cash_session_id` desde la sesión de caja activa de la terminal |
| [`apps/api/rules/registry.py`](../../NUEVO-POS/apps/api/rules/registry.py:401) | (si hace falta) una regla que valide que hay sesión de caja abierta antes de cobrar |

### 4.0.2 Decisión de diseño

El cobro debe **exigir** una sesión de caja abierta (RN-49) y **ligar** el ticket a ella. Si no hay sesión abierta, el cobro falla con un `reason` claro (`{outcome:'error', reason:'No hay turno de caja abierto'}`). Esto es coherente con la operación real: no se cobra sin caja abierta.

### 4.0.3 Puerta 4.0

```text
Comando : docker compose exec -T api pytest tests/test_f4_caja.py -k "liga_ticket" -v
Criterio: un ticket cobrado con sesión abierta queda con cash_session_id = id de la sesión.
          Un cobro sin sesión abierta falla con reason claro.
Resultado esperado: PASA
```

---

## 5. SUB-FASE 4.1 — BACKEND DE CAJA

### 5.1.1 Qué se construye

| Archivo | Qué resuelve | Contrato |
|---|---|---|
| [`apps/api/routers/cash.py`](../../NUEVO-POS/apps/api/routers/cash.py:1) | 6 endpoints de caja | 9–14 |
| [`apps/api/schemas.py`](../../NUEVO-POS/apps/api/schemas.py:1) | Esquemas Pydantic de entrada/salida | — |
| [`apps/api/main.py`](../../NUEVO-POS/apps/api/main.py:68) | `include_router(cash.router)` | — |
| [`apps/api/tests/test_f4_caja.py`](../../NUEVO-POS/apps/api/tests/test_f4_caja.py:1) | Puerta F4.1 | — |

### 5.1.2 Los 6 endpoints

| # | Contrato | Operación | Regla clave |
|---|---|---|---|
| 9 | `caja.sesion_activa` | `GET /cash/active-session` | — |
| 10 | `caja.abrir_turno` | `POST /cash/open-session` | RN-49, RN-50 |
| 11 | `caja.registrar_movimiento` | `POST /cash/movements` | RN-51, RN-52 |
| 12 | `caja.resumen_del_turno` | `GET /cash/session-summary/{id}` | RN-53, RN-60 |
| 13 | `caja.cerrar_turno` | `POST /cash/close-session` | RN-54, RN-55 |
| 14 | `caja.reporte_diario` | `GET /cash/daily-report/{fecha}` | RN-56, RN-59 |

> **Nota:** el contrato 14 (`reporte_diario`) se declara pero su **consumidor es "POS / Estadísticas"**. Se implementa el endpoint, pero la **pantalla** de reporte diario **no** entra en Fase 4 (§9).

### 5.1.3 Puerta 4.1

```text
Comando : docker compose exec -T api pytest tests/test_f4_caja.py -v
Criterio: los 6 contratos responden; RN-49 (una sesión por terminal),
          RN-50 (fondo no negativo), RN-51 (tipo válido), RN-52 (movimiento
          solo si abierta), RN-54 (conteos al cerrar), RN-55 (cerrada inmutable).
Resultado esperado: PASA
```

---

## 6. SUB-FASE 4.2 — SERVICIO DE CAJA (`cashService.js`)

### 6.2.1 Qué se construye

| Archivo | Qué resuelve |
|---|---|
| [`apps/pos/src/services/cashService.js`](../../NUEVO-POS/apps/pos/src/services/cashService.js:1) | Las 6 llamadas HTTP con `{outcome, reason}` |
| [`apps/pos/src/services/cashService.test.js`](../../NUEVO-POS/apps/pos/src/services/cashService.test.js:1) | Test con API simulada |

### 6.2.2 Decisión de diseño

El servicio sigue el patrón de [`client.js`](../../NUEVO-POS/apps/pos/src/api/client.js:1): cada función devuelve `{outcome, reason, data}` y **nunca lanza**. Reutiliza [`aOutcome`](../../NUEVO-POS/apps/pos/src/utils/outcome.js:68) y [`withRetries`](../../NUEVO-POS/apps/pos/src/utils/withRetries.js:55) para las llamadas de escritura (abrir, movimiento, cerrar).

### 6.2.3 Puerta 4.2

```text
Comando : npm run test   (cwd: NUEVO-POS)
Criterio: cashService.test.js en verde; cada función devuelve {outcome, reason}.
Resultado esperado: PASA
```

---

## 7. SUB-FASE 4.3 — PANTALLA (`GestorDeCaja.jsx`)

### 7.3.1 Qué se construye

| Archivo | Qué resuelve |
|---|---|
| [`apps/pos/src/GestorDeCaja.jsx`](../../NUEVO-POS/apps/pos/src/GestorDeCaja.jsx:1) | Pantalla completa: abrir turno, movimientos, resumen, arqueo, cierre |
| [`apps/pos/src/GestorDeCaja.test.jsx`](../../NUEVO-POS/apps/pos/src/GestorDeCaja.test.jsx:1) | Test de componente |

### 7.3.2 Flujo de la pantalla

1. **Sin turno abierto** → formulario de apertura (fondo inicial).
2. **Con turno abierto** → resumen en vivo (esperado vs contado), lista de movimientos, botón de cierre.
3. **Al cerrar** → captura de conteos físicos (efectivo, crédito, débito) y muestra la **diferencia** (descuadre).

### 7.3.3 Puerta 4.3

```text
Comando : npm run test   (cwd: NUEVO-POS)
Criterio: GestorDeCaja.test.js en verde; los 3 estados (sin turno / abierto / cierre)
          se renderizan y las acciones llaman al servicio correcto.
Resultado esperado: PASA
```

---

## 8. SUB-FASE 4.4 — PLANTILLA DE CORTE (`CorteTicketTemplate.jsx`)

### 8.4.1 Qué se construye

| Archivo | Qué resuelve |
|---|---|
| [`apps/pos/src/components/CorteTicketTemplate.jsx`](../../NUEVO-POS/apps/pos/src/components/CorteTicketTemplate.jsx:1) | Plantilla de impresión del corte |
| [`apps/pos/src/components/CorteTicketTemplate.test.jsx`](../../NUEVO-POS/apps/pos/src/components/CorteTicketTemplate.test.jsx:1) | Test de componente |

### 8.4.2 Decisión de diseño

La plantilla **solo renderiza** el corte (no imprime). La impresión física real es Fase 6. Sigue el patrón de [`SalesReceipt.jsx`](../../NUEVO-POS/apps/pos/src/components/SalesReceipt.jsx:34).

### 8.4.3 Puerta 4.4

```text
Comando : npm run test   (cwd: NUEVO-POS)
Criterio: CorteTicketTemplate.test.js en verde; renderiza esperado, contado y diferencia.
Resultado esperado: PASA
```

---

## 9. FUERA DE ALCANCE DE FASE 4

| Qué | Por qué no entra |
|---|---|
| Pantalla de **reporte diario** (contrato 14) | Su consumidor es "POS / Estadísticas"; es otra rebanada |
| **Impresión física** del corte | Es Fase 6 (Impresión + PDF) |
| **Job de TTL** de 24 h (lección OMEGA) | Es infraestructura, no una rebanada de negocio |
| **Pizarrón de cuentas** | Es Fase 5 |
| **CRM / notificaciones** | Es Fase 8 |

**Fase 4 entrega:** el flujo completo de caja (abrir turno → movimientos → resumen → arqueo → cierre → plantilla de corte), con su backend, su servicio y su pantalla. Nada más.

---

## 10. AUTOCRÍTICA (defectos hallados en la propuesta inicial)

| # | Defecto | Gravedad | Evidencia | Corrección |
|---|---|---|---|---|
| **D-1** | **Lectura falsa del `estado_hoy`.** La propuesta inicial dijo *"los contratos dicen `Ya existe` pero los endpoints no existen"*, como si el `estado_hoy` mintiera. En realidad describe el **ERP viejo**. | **Alta** | [`contracts/registry.py:7-26`](../../NUEVO-POS/apps/api/contracts/registry.py:7) (matriz del Documento 9) | **§3 desambiguación permanente** |
| **D-2** | **Corte de sub-fases sin justificar.** La propuesta inicial inventó 4 sub-fases sin derivarlas del orden interno. | Media | Plan Maestro §7 Fase 4 (3 archivos) | **§2** deriva el corte del orden `Endpoint → Servicio → Test → Componente` |
| **D-3** | **Dependencia oculta no detectada.** La propuesta inicial afirmó que la regla dura #4 bastaba, sin ver que `Ticket.cash_session_id` nunca se puebla → arqueo cero. | **Alta** | [`routers/pos.py:274`](../../NUEVO-POS/apps/api/routers/pos.py:274) | **§4 Sub-fase 4.0** (prerrequisito) |
| **D-4** | **Sin delimitación de alcance.** La propuesta inicial no dijo qué NO entra. | Baja | Plan de Fase 3 §10.3 (precedente) | **§9 Fuera de alcance** |

### 10.1 Alineación con el objetivo del proyecto

El objetivo (Plan Maestro §1) es construir la versión mejorada del POS *"como si se hubiera diseñado desde un principio por un arquitecto senior"*, y el §9 incluye el Gestor de Caja como resultado esperado #4. El §3 del Plan Maestro lista el Gestor de Caja (56 KB) entre lo que DeepSeek **NO construyó**.

| Verificación | Resultado |
|---|---|
| ¿Respeta la regla dura (no tocar el ERP)? | ✅ Todo el trabajo va en `NUEVO-POS` |
| ¿Respeta las 6 prohibiciones? | ✅ Sin timers, sin clearCart sin verificación, sin estado en async |
| ¿Respeta las 10 reglas de batalla? | ✅ `{outcome,reason}`, withRetries, UTC, respuesta ligera |
| ¿Respeta la frontera por contratos (A-02)? | ✅ El POS consume los contratos 9–14, no lee las tablas |
| ¿Sigue el orden "de adentro hacia afuera" (§10.6)? | ✅ Rebanadas verticales: 4.0 → 4.1 → 4.2 → 4.3 → 4.4 |
| ¿Cada sub-fase tiene puerta verificable? | ✅ §4.0.3, §5.1.3, §6.2.3, §7.3.3, §8.4.3 |
| ¿Contribuye al objetivo? | ✅ Es el resultado esperado #4; recupera el 80% que DeepSeek no hizo |

### 10.2 Riesgo residual aceptado

| Riesgo | Mitigación |
|---|---|
| El prerrequisito 4.0 toca el backend de tickets (Fase 3) | Se aísla como sub-fase propia con su puerta; no se relaja ninguna puerta de F3 |
| El contrato 14 podría requerir revisión del arquitecto | Se implementa el endpoint pero la pantalla queda fuera (§9) |
| El orden 4.0→4.1→4.2→4.3→4.4 es estricto | La regla dura #4 lo exige: ninguna sub-fase sin la puerta anterior en verde |

---

*Plan de Abordaje de la Fase 4 por Partes. **Versión 1.0**. Alineado al Plan Maestro v1.2 §7 (Fase 4) y §10.6, y al Prompt del Arquitecto v1.2.*
