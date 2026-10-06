# 🛡️ PLAN DE ABORDAJE — FASE 13 POR PARTES (Auditoría y Control)

> **Fecha:** 6 Oct 2026
> **Versión:** 1.0
> **Autor:** Antigravity + Víctor (dueño de R de Rico)
> **Estado:** PROPUESTO — pendiente de aprobación del dueño antes de escribir código
> **Documento padre:** [`PLAN_MAESTRO_DEFINITIVO_POS.md`](../PLAN_MAESTRO_DEFINITIVO_POS.md:1) §7 (Fases) + §8.1 (el POS es el primer módulo de un ERP reconstruido)
> **Contrato de comportamiento:** [`PROMPT_DEL_ARQUITECTO_DEL_NUEVO_POS.md`](../06-prompt-del-arquitecto/PROMPT_DEL_ARQUITECTO_DEL_NUEVO_POS.md:1)
> **Documentación del viejo POS (el oráculo):** [`DOCUMENTACION_MODULO_AUDITORIA_Y_CONTROL.md`](../../ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_MODULO_AUDITORIA_Y_CONTROL.md:1)
> **Precedentes:** [`PLAN_DE_ABORDAJE_FASE_5_POR_PARTES.md`](PLAN_DE_ABORDAJE_FASE_5_POR_PARTES.md:1) (cerrada) · [`PLAN_DE_ABORDAJE_FASE_6_POR_PARTES.md`](PLAN_DE_ABORDAJE_FASE_6_POR_PARTES.md:1) (cerrada)

> [!IMPORTANT]
> **Este documento es un artefacto de diseño.** No contiene código de producción.
> El código va en `NUEVO-POS`; los planos van en `PLANOS-ARQUITECTONICOS` (Plan Maestro §1).

> [!CAUTION]
> **LECTURA OBLIGATORIA ANTES DE CUALQUIER OTRA COSA: §3 (la afirmación falsa del inventario).**
> El inventario de la Fase 13 suele presentarse como *"libro mayor inmutable + outbox + log de auditoría"*.
> **La mayor parte de eso YA ESTÁ CONSTRUIDO.** §3 lo demuestra con evidencia y aísla el hueco real.

---

## 0. PROPÓSITO DE ESTE DOCUMENTO

La Fase 13 ("Auditoría y Control") es la **rebanada de observabilidad** del POS: la capacidad de
**probar lo que pasó** aunque el módulo consumidor esté caído.

Este documento:

1. **Registra el diagnóstico** del estado real de `NUEVO-POS` al 6 Oct 2026 para auditoría y control.
2. **Documenta la afirmación falsa** del inventario (§3) y establece la regla de decisión permanente.
3. **Justifica** por qué la Fase 13 se parte en 3 sub-fases (13.0–13.2).
4. **Define** cada sub-fase con su puerta verificable y su ficha de evidencia.
5. **Se autocritica** contra la evidencia del repositorio (§10) y corrige los defectos hallados.
6. **Delimita** qué NO entra en Fase 13 (§9).

---

## 1. DIAGNÓSTICO — ESTADO REAL DE LA AUDITORÍA EN `NUEVO-POS`

### 1.1 Lo que YA existe (cimiento)

| Pieza | Evidencia | Estado |
|---|---|---|
| Libro mayor inmutable `movimientos_inventario` | [`migrations/versions/0001_initial_schema.py:287`](../../NUEVO-POS/apps/api/migrations/versions/0001_initial_schema.py:287) | ✅ Existe |
| Trigger `trg_ledger_inmutable` (rechaza UPDATE/DELETE) | [`0001_initial_schema.py:401`](../../NUEVO-POS/apps/api/migrations/versions/0001_initial_schema.py:401) | ✅ Instalado y probado |
| Outbox transaccional `warehouse_events` | [`0001_initial_schema.py:309`](../../NUEVO-POS/apps/api/migrations/versions/0001_initial_schema.py:309) | ✅ Existe |
| Guardián del outbox (misma transacción, sin silencios) | [`guards/outbox.py:55`](../../NUEVO-POS/apps/api/guards/outbox.py:55) | ✅ Implementado |
| Reglas RN-61 a RN-66 (ledger + eventos) | [`rules/registry.py:521`](../../NUEVO-POS/apps/api/rules/registry.py:521) | ✅ Implementadas y probadas |
| Outbox de consolidación multi-sucursal | [`consolidacion/registry.py:177`](../../NUEVO-POS/apps/api/consolidacion/registry.py:177) | ✅ Implementado |
| Reglas RN-75/RN-76/RN-77 (auditoría) | [`rules/registry.py:637`](../../NUEVO-POS/apps/api/rules/registry.py:637) | ⚠️ **Declaradas, sin endpoint** |
| Contrato 5 `pos.eventos_auditables` | [`contracts/registry.py:205`](../../NUEVO-POS/apps/api/contracts/registry.py:205) | ⚠️ Declarado (`estado_hoy="Cicatriz"`), **sin endpoint** |

### 1.2 Lo que NO existe (el hueco real)

| Pieza | Evidencia de la ausencia | Impacto |
|---|---|---|
| **Endpoint `GET /pos/auditable-events`** | [`routers/pos.py`](../../NUEVO-POS/apps/api/routers/pos.py:1) no tiene esa ruta | Auditoría no tiene de dónde leer |
| **Esquema `EventoAuditableSalida`** | [`schemas.py`](../../NUEVO-POS/apps/api/schemas.py:1) no lo tiene | Sin contrato HTTP |
| **Tabla `pos_audit_log`** | No existe en `models/` ni en migraciones | RN-75 no persiste nada |
| **`test_rn75` / `test_rn76` / `test_rn77`** | Búsqueda en `tests/`: 0 resultados | Deuda D-11.1 (reglas sin test) |
| **Servicio de auditoría en el POS** | No existe en `apps/pos/src/services/` | El POS no puede consultar su propia auditoría |

### 1.3 La dependencia oculta (hallazgo de la autocrítica)

El contrato 5 declara `entrada={"desde", "hasta"}` y `salida={"eventos": [...]}`. Pero **no declara
de dónde salen los eventos**. Hay dos fuentes candidatas y el contrato no elige:

1. **La tabla `tickets`** (proyección de ventas) — pero el contrato prohíbe exponer la tabla.
2. **Un log de auditoría dedicado** (`pos_audit_log`) — que **no existe**.

Sin resolver esta ambigüedad, el endpoint no puede escribirse sin violar la frontera (A-02).
Es la misma clase de dependencia oculta que la Fase 5 tuvo con el listado de cuentas
(§1.3 del plan de F5).

> **Conclusión del diagnóstico:** la Fase 13 tiene un **prerrequisito** (13.0) que ninguna fase
> anterior resolvió, porque el libro mayor y el outbox cubren **inventario**, no **auditoría de
> escrituras POS**. El hueco real es el **log de auditoría** + su **contrato de lectura**.

---

## 2. POR QUÉ SE PARTE EN SUB-FASES

El inventario presenta la Fase 13 como "libro mayor + outbox + log de auditoría" en un solo bloque.
La evidencia (§1) demuestra que **los dos primeros ya existen** y solo el tercero falta. Por eso el
corte real es de **3 sub-fases**, derivado del orden interno de rebanada vertical
(`Dato → Contrato → Superficie`) más el prerrequisito detectado:

| Sub-fase | Rol en la rebanada | Nivel |
|---|---|---|
| **13.0** prerrequisito | Tabla `pos_audit_log` + migración + reglas RN-75/76/77 con test | Negocio (dato + regla) |
| **13.1** contrato | Contrato 5 + endpoint `GET /pos/auditable-events` + esquema | Negocio (contrato) |
| **13.2** superficie | `auditService.js` + panel de consulta (opcional, según §9) | Negocio (superficie) |

---

## 3. LA AFIRMACIÓN FALSA DEL INVENTARIO (regla de decisión permanente)

> **Afirmación del inventario:** *"F13 — libro mayor inmutable + eventos outbox + log de auditoría (RN-75/76/77)."*

**Esa afirmación es PARCIALMENTE FALSA.** La evidencia:

| Componente de la afirmación | ¿Existe? | Evidencia |
|---|---|---|
| Libro mayor inmutable | ✅ **SÍ** | `movimientos_inventario` + `trg_ledger_inmutable` |
| Eventos outbox | ✅ **SÍ** | `warehouse_events` + `guards/outbox.py` |
| Log de auditoría (RN-75/76/77) | ❌ **NO** | Reglas declaradas, sin tabla, sin endpoint, sin test |

### 3.1 La regla de decisión permanente

> **REGLA DE DECISIÓN F13:** Antes de construir cualquier pieza de la Fase 13, se verifica contra
> el repositorio si **ya existe**. Si existe, se **documenta como cimiento** y NO se reconstruye.
> Solo se construye el **hueco real** (el log de auditoría + su contrato de lectura).

Esta regla es la aplicación directa de §10.6.2 (*"la COMPLETITUD del conjunto también es una
compuerta"*): un inventario que lista como "pendiente" algo ya construido **infla el trabajo** y
**oculta el hueco real**.

---

## 4. SUB-FASE 13.0 — EL LOG DE AUDITORÍA (prerrequisito)

### 4.1 Qué construye

1. **Tabla `pos_audit_log`** (migración `0004_pos_audit_log.py`):
   - `id` (UUID, PK)
   - `endpoint` (String) — la ruta de la escritura
   - `payload` (JSONB) — el cuerpo de la petición (sin datos sensibles)
   - `codigo` (Integer) — el código de respuesta HTTP
   - `terminal_id` (String, indexado) — para la consulta por terminal (RN-77)
   - `usuario_id` (String, nullable)
   - `extras` (JSONB) — metadatos adicionales
   - `timestamp` (DateTime(timezone=True), indexado) — **UTC** (RN-78)
   - Índice compuesto `(terminal_id, timestamp)` para RN-77.

2. **Reglas RN-75/76/77 con test real** (cierra la deuda D-11.1):
   - `test_rn75` — cada escritura POS se registra.
   - `test_rn76` — el registro incluye endpoint, payload, código y extras.
   - `test_rn77` — la consulta filtra por terminal y rango de fechas.

3. **Guardián de escritura** (`guards/audit.py`): una función que registra la escritura en la
   **misma transacción** que la operación (mismo patrón que el outbox, RN-63). Si la operación
   falla, el registro no se persiste.

### 4.2 Puerta verificable

| Criterio | Verificación |
|---|---|
| La tabla existe con los índices declarados | `test_f1_cimiento.py` (lista de tablas) |
| RN-75/76/77 tienen `test_rnXX` que **ejecuta la función** | `test_f3_comportamiento.py` |
| El registro ocurre en la misma transacción | Test de rollback: si la operación falla, no hay registro |
| El timestamp es UTC con tzinfo | Guardián E-10 (DateTime naive) |

### 4.3 Ficha de evidencia

`FICHA_F13_0_LOG_AUDITORIA.md`

---

## 5. SUB-FASE 13.1 — EL CONTRATO 5 (endpoint de lectura)

### 5.1 Qué construye

1. **Esquema `EventoAuditableSalida`** en `schemas.py`:
   - `tipo` (String), `ticket_id` (UUID), `usuario_id` (String), `timestamp` (DateTime), `detalle` (JSONB).
   - **Proyección de campos escalares** (Regla 15): nunca `SELECT *`.

2. **Endpoint `GET /pos/auditable-events`** en `routers/pos.py`:
   - Entrada: `desde`, `hasta` (DateTime con timezone).
   - Salida: `{"eventos": [...]}`.
   - **Lee de `pos_audit_log`**, nunca de `tickets` (frontera A-02).
   - Valida el rango de fechas (400 si es inválido).

3. **Actualización del contrato 5** en `contracts/registry.py`:
   - `estado_hoy`: de `"Cicatriz"` a `"Implementado"`.
   - Documentar que la fuente es `pos_audit_log`, no `tickets`.

### 5.2 Puerta verificable

| Criterio | Verificación |
|---|---|
| El endpoint responde 200 con el rango válido | `test_f13_auditoria.py` |
| El endpoint responde 400 con rango inválido | `test_f13_auditoria.py` |
| La salida tiene EXACTAMENTE los 5 campos declarados | Test de proyección (Regla 15) |
| El POS NO lee la tabla `tickets` para auditar | Guardián de frontera (A-02) |
| El contrato 5 ya no dice "Cicatriz" | `test_f2_frontera.py` |

### 5.3 Ficha de evidencia

`FICHA_F13_1_CONTRATO_AUDITORIA.md`

---

## 6. SUB-FASE 13.2 — LA SUPERFICIE (opcional)

### 6.1 Qué construye (solo si el dueño lo aprueba)

1. **`auditService.js`** en `apps/pos/src/services/` — consume el contrato 5 con el patrón
   `{outcome, reason, data}` (nunca lanza).
2. **Panel de consulta** — una vista de solo lectura que lista los eventos por terminal y rango.

### 6.2 Por qué es opcional

El módulo **Auditoría** es un **consumidor externo** del POS (contrato 5). La superficie de
consulta puede vivir en el módulo de Auditoría, no en el POS. Construirla en el POS solo tiene
sentido si el dueño quiere consultar la auditoría **desde la misma terminal**.

> **Decisión pendiente:** §9.2. Si el dueño no la pide, la Fase 13 cierra en 13.1.

### 6.3 Ficha de evidencia

`FICHA_F13_2_SUPERFICIE_AUDITORIA.md` (solo si se construye)

---

## 7. TRAZABILIDAD — REGLA → TEST → FICHA

| Regla | Test | Ficha | Sub-fase |
|---|---|---|---|
| RN-75 | `test_rn75` | F13.0 | 13.0 |
| RN-76 | `test_rn76` | F13.0 | 13.0 |
| RN-77 | `test_rn77` | F13.0 | 13.0 |
| Contrato 5 | `test_f13_auditoria.py` | F13.1 | 13.1 |
| Frontera A-02 | `test_f2_frontera.py` | F13.1 | 13.1 |

---

## 8. LO QUE SE HEREDA DEL VIEJO POS (§6.8)

> **La INTEGRACIÓN se hereda, la IMPLEMENTACIÓN se reescribe.**

Del [`DOCUMENTACION_MODULO_AUDITORIA_Y_CONTROL.md`](../../ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_MODULO_AUDITORIA_Y_CONTROL.md:1)
se hereda la **integración** (qué pregunta la gerencia), no la implementación:

| Se hereda (integración) | Se reescribe (implementación) |
|---|---|
| Trazabilidad: quién capturó, quién cobró, en qué terminal | El POS viejo lo hacía con `cashed_by_id` + `cash_session_id`; el nuevo lo hace con `pos_audit_log` |
| Filtro por "día operativo" (no por día UTC) | El POS viejo usaba `+6h` hardcodeado; el nuevo usa `local_day_bounds_utc` (RN-79) |
| Consulta por terminal y rango | El POS viejo leía `tickets`; el nuevo lee `pos_audit_log` (frontera A-02) |

### 8.1 Los 5 bugs del cementerio del viejo POS — cómo los evita el nuevo

| Bug del viejo POS | Cómo lo evita el nuevo POS |
|---|---|
| **BUG 1** — secuestro de `terminal_id` | RN-12: `terminal_id` es inmutable |
| **BUG 2** — zona horaria (tickets nocturnos) | RN-78/RN-79: UTC + `local_day_bounds_utc` |
| **BUG 3** — `Decimal + float` (error 500) | E-09: dinero en `Numeric`, nunca `Float` |
| **BUG 4** — lazy loading (pantalla en blanco) | Regla 15: proyección de campos escalares |
| **BUG 5** — URL con `window.location.hostname` | `CONFIG.API_BASE_URL` centralizado |

---

## 9. QUÉ NO ENTRA EN LA FASE 13

### 9.1 Fuera de alcance (explícito)

- **Reconstruir el libro mayor** — ya existe (§1.1). Prohibido tocarlo.
- **Reconstruir el outbox** — ya existe (§1.1). Prohibido tocarlo.
- **El módulo de Auditoría completo** (UI de gerencia, reporte diario consolidado) — es un
  **módulo externo** del ERP, no parte del POS. El POS solo **expone** el contrato 5.
- **El reporte diario consolidado** — vive en el módulo de Estadísticas (contrato 6).

### 9.2 Decisión pendiente del dueño

> **¿Se construye la superficie de consulta (13.2) dentro del POS, o se deja al módulo de Auditoría?**

- **Opción A:** Fase 13 cierra en 13.1 (solo el contrato). La UI vive en Auditoría.
- **Opción B:** Fase 13 incluye 13.2 (panel de consulta en el POS).

---

## 10. AUTOCRÍTICA

| # | Defecto detectado | Corrección |
|---|---|---|
| **A1** | El inventario infla la Fase 13 con piezas ya construidas. | §3 establece la regla de decisión F13. |
| **A2** | El contrato 5 no declara su fuente de datos. | §5.1 la fija: `pos_audit_log`, nunca `tickets`. |
| **A3** | RN-75/76/77 están declaradas sin test (deuda D-11.1). | §4.1 las dota de `test_rnXX` real. |
| **A4** | El contrato 5 dice `estado_hoy="Cicatriz"` pero no hay endpoint. | §5.1 lo actualiza a `"Implementado"`. |
| **A5** | La superficie (13.2) podría no ser deseable. | §6.2 la marca como opcional y §9.2 la deja a decisión del dueño. |

---

## 11. RESUMEN DE ACCIONES

| Sub-fase | Entregable | Puerta | Ficha |
|---|---|---|---|
| **13.0** | Tabla `pos_audit_log` + RN-75/76/77 con test | `test_rn75/76/77` + cimiento | `FICHA_F13_0_LOG_AUDITORIA.md` |
| **13.1** | Contrato 5 + `GET /pos/auditable-events` | `test_f13_auditoria.py` + frontera | `FICHA_F13_1_CONTRATO_AUDITORIA.md` |
| **13.2** | `auditService.js` + panel (opcional) | Test de servicio + UI | `FICHA_F13_2_SUPERFICIE_AUDITORIA.md` |

**Cierre de Fase 13:** CI verde (front + back + guards) + las 3 fichas + commit/push.

---

## 12. ADVERTENCIA PARA FUTURAS IAs

> **NO reconstruyas el libro mayor ni el outbox.** Ya existen y están probados.
> **NO leas la tabla `tickets` desde el endpoint de auditoría.** Usa `pos_audit_log` (frontera A-02).
> **NO uses `+6h` hardcodeado para el "día operativo".** Usa `local_day_bounds_utc` (RN-79).
> **NO expongas `SELECT *`.** Proyecta los 5 campos escalares (Regla 15).
