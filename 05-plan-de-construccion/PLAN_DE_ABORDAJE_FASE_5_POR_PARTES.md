# 🧱 PLAN DE ABORDAJE — FASE 5 POR PARTES (Pizarrón de Cuentas Abiertas)

> **Fecha:** 29 Sep 2026
> **Versión:** 1.0
> **Autor:** Antigravity + Víctor (dueño de R de Rico)
> **Estado:** PROPUESTO — pendiente de aprobación del dueño antes de escribir código
> **Documento padre:** [`PLAN_MAESTRO_DEFINITIVO_POS.md`](../PLAN_MAESTRO_DEFINITIVO_POS.md:318) §7 (Fase 5)
> **Contrato de comportamiento:** [`PROMPT_DEL_ARQUITECTO_DEL_NUEVO_POS.md`](../06-prompt-del-arquitecto/PROMPT_DEL_ARQUITECTO_DEL_NUEVO_POS.md:1)
> **Precedentes:** [`PLAN_DE_ABORDAJE_FASE_3_POR_PARTES.md`](PLAN_DE_ABORDAJE_FASE_3_POR_PARTES.md:1) v1.2 (cerrada) · [`PLAN_DE_ABORDAJE_FASE_4_POR_PARTES.md`](PLAN_DE_ABORDAJE_FASE_4_POR_PARTES.md:1) v1.0 (cerrada)

> [!IMPORTANT]
> **Este documento es un artefacto de diseño.** No contiene código de producción.
> El código va en `NUEVO-POS`; los planos van en `PLANOS-ARQUITECTONICOS` (Plan Maestro §1).

> [!CAUTION]
> **LECTURA OBLIGATORIA ANTES DE CUALQUIER OTRA COSA: §3 (la afirmación falsa del plan maestro).**
> El Plan Maestro §7 afirma que la "Lógica multi-cuenta (ya incluida en `useTicketActions` de Fase 3)".
> **Esa afirmación es FALSA.** §3 lo demuestra con evidencia y establece la regla de decisión.

---

## 0. PROPÓSITO DE ESTE DOCUMENTO

La Fase 5 ("Pizarrón de Cuentas Abiertas") es la **quinta rebanada vertical** del Plan Maestro §7. Permite **varias cuentas en paralelo** en una misma terminal: el cajero abre una cuenta, la deja "en el corcho", atiende otra, y luego recupera la primera.

Este documento:

1. **Registra el diagnóstico** del estado real de `NUEVO-POS` al 29 Sep 2026 para las cuentas abiertas.
2. **Documenta la afirmación falsa** del plan maestro (§3) y establece la regla de decisión permanente.
3. **Justifica** por qué la Fase 5 se parte en 4 sub-fases (5.0–5.3).
4. **Define** cada sub-fase con su puerta verificable y su ficha de evidencia.
5. **Se autocritica** contra la evidencia del repositorio (§10) y corrige los defectos hallados.
6. **Delimita** qué NO entra en Fase 5 (§9).

---

## 1. DIAGNÓSTICO — ESTADO REAL DE LAS CUENTAS ABIERTAS EN `NUEVO-POS`

### 1.1 Lo que YA existe (cimiento)

| Pieza | Evidencia | Estado |
|---|---|---|
| Tabla `tickets` con `status` (`OPEN`/`PAID`) | [`apps/api/models/pos.py:73`](../../NUEVO-POS/apps/api/models/pos.py:73) | ✅ Existe |
| Columna `Ticket.terminal_id` (indexada) | [`apps/api/models/pos.py:90`](../../NUEVO-POS/apps/api/models/pos.py:90) | ✅ Existe |
| Columna `Ticket.version` (bloqueo optimista) | [`apps/api/models/pos.py:74`](../../NUEVO-POS/apps/api/models/pos.py:74) | ✅ Existe |
| Reglas RN-31/RN-32/RN-33 (draft por terminal) | [`apps/api/rules/registry.py:288`](../../NUEVO-POS/apps/api/rules/registry.py:288) | ✅ Implementadas y probadas |
| Reglas RN-25/RN-26 (validación de versión) | [`apps/api/rules/registry.py:250`](../../NUEVO-POS/apps/api/rules/registry.py:250) | ✅ Implementadas y probadas |
| Contrato 21 `pos.leer_ticket` (leer UN ticket) | [`apps/api/contracts/registry.py:502`](../../NUEVO-POS/apps/api/contracts/registry.py:502) | ✅ Implementado (F3.2) |
| Interfaz 13 `OpenAccountsCorkboard` **declarada** | [`apps/api/superficie/registry.py:354`](../../NUEVO-POS/apps/api/superficie/registry.py:354) | ⚠️ Declarada, **no construida** |
| Flujo E.3 "Recuperación de cuenta" declarado | [`apps/api/superficie/registry.py:111`](../../NUEVO-POS/apps/api/superficie/registry.py:111) | ⚠️ Declarado, **no cableado** |

### 1.2 Lo que NO existe (el hueco real)

| Pieza | Evidencia de la ausencia | Impacto |
|---|---|---|
| **Contrato de cuentas abiertas** | Los 22 contratos de [`contracts/registry.py:73`](../../NUEVO-POS/apps/api/contracts/registry.py:73) no incluyen ninguno que liste cuentas. El 21 lee UN ticket por id | No hay frontera declarada para listar |
| **Endpoint `GET /pos/open-accounts`** | [`routers/pos.py`](../../NUEVO-POS/apps/api/routers/pos.py:1) no tiene esa ruta | El pizarrón no tiene de dónde leer |
| **Esquema `CuentaAbiertaSalida`** | [`apps/api/schemas.py`](../../NUEVO-POS/apps/api/schemas.py:1) no lo tiene | Sin contrato HTTP |
| **`openAccountsService.js`** | No existe en `apps/pos/src/services/` | El POS no puede pedir las cuentas |
| **`useOpenAccounts.js`** | No existe en `apps/pos/src/hooks/` | No hay estado multi-cuenta |
| **`OpenAccountsCorkboard.jsx`** | No existe en `apps/pos/src/components/` | No hay pizarrón visual |
| **Lógica multi-cuenta en `useTicketActions`** | [`useTicketActions.js:34`](../../NUEVO-POS/apps/pos/src/hooks/useTicketActions.js:34) solo maneja UN ticket | **La afirmación del plan maestro es falsa** |
| **Soporte multi-cuenta en `useCart`** | [`useCart.js:43`](../../NUEVO-POS/apps/pos/src/hooks/useCart.js:43) recibe UN `ticketId` y UNA `version` | El carrito es mono-cuenta por diseño |

### 1.3 La dependencia oculta (hallazgo de la autocrítica)

El pizarrón muestra **cuentas abiertas de una terminal**. Para que una cuenta aparezca en el corcho debe existir en el servidor con `status='OPEN'` y `terminal_id` poblado. Hoy:

- [`crear_ticket`](../../NUEVO-POS/apps/api/routers/pos.py:241) **sí** puebla `terminal_id` (viene del cuerpo).
- Pero **no hay forma de listar** los tickets `OPEN` de una terminal: el contrato 21 exige un `ticket_id` conocido.

Por lo tanto, sin un contrato de **listado**, el pizarrón no puede descubrir las cuentas. Es la misma clase de dependencia oculta que la Fase 4 tuvo con `cash_session_id` (§1.3 del plan de F4).

> **Conclusión del diagnóstico:** la Fase 5 tiene un **prerrequisito** (5.0) que la Fase 3 no resolvió, porque la Fase 3 solo entregó el flujo E.1 (venta directa) y las cuentas paralelas estaban fuera de su alcance.

---

## 2. POR QUÉ SE PARTE EN SUB-FASES

El Plan Maestro §7 lista **1 archivo** (`OpenAccountsCorkboard.jsx`) porque **asume que la lógica multi-cuenta ya existe**. La evidencia (§1.2) demuestra que **no existe**. Por eso el corte real es de **4 sub-fases**, derivado del orden interno de rebanada vertical (`Endpoint → Servicio → Hook → Componente`) más el prerrequisito detectado:

| Sub-fase | Rol en la rebanada | Nivel |
|---|---|---|
| **5.0** prerrequisito | Contrato 23 + endpoint `GET /pos/open-accounts` + esquema | Negocio (dato + contrato) |
| **5.1** servicio | `openAccountsService.js` + `listarCuentasAbiertas` en `client.js` | Negocio (comportamiento) |
| **5.2** hook | `useOpenAccounts.js` (lista, refresca, recupera) | Negocio (comportamiento) |
| **5.3** pantalla | `OpenAccountsCorkboard.jsx` | Negocio (superficie) |

---

## 3. LA AFIRMACIÓN FALSA DEL PLAN MAESTRO (desambiguación permanente)

> **Propósito:** que este error de lectura **no pueda repetirse**.

**El hecho:** el Plan Maestro §7 (Fase 5) dice textualmente:

> *"Lógica multi-cuenta (**ya incluida en `useTicketActions` de Fase 3**)."*

Es tentador leer eso como *"el hook ya tiene las funciones multi-cuenta; solo falta el componente"*. **Es falso.**

**La evidencia:** [`useTicketActions.js`](../../NUEVO-POS/apps/pos/src/hooks/useTicketActions.js:34) expone exactamente:

```js
return { ticket, enviando, ultimoOutcome, crearTicket, cobrar };
```

- `ticket` es **UN** objeto (no una lista).
- `crearTicket(items)` crea **UN** ticket.
- `cobrar(paymentDetails)` cobra **EL** ticket actual.
- **No hay** `cuentas`, `listarCuentas`, `recuperarCuenta`, `cambiarCuenta`.

**Lo que significa realmente:** la frase del plan maestro describe una **intención de diseño** (que la lógica multi-cuenta viva cerca de `useTicketActions`), no un **hecho construido**. La Fase 3 entregó el flujo E.1 (venta directa, mono-cuenta); las cuentas paralelas quedaron explícitamente fuera de su alcance.

**La regla de decisión (aplicar siempre):**

1. **¿La frase del plan describe lo construido o lo deseado?** → **Lo deseado.** El plan maestro es un plano, no un inventario.
2. **¿Cómo sé si una pieza existe en NUEVO-POS?** → **Buscando el archivo/función en el código**, no leyendo el plan.
3. **¿Hay conflicto entre el plan y el código?** → **Gana el código.** Es la evidencia (E-14).

**Firma de la desambiguación:** este registro se añadió en la v1.0 tras detectar que la propuesta inicial de Fase 5 confió en la frase del plan maestro (defecto D-1 de §10).

---

## 4. SUB-FASE 5.0 — PRERREQUISITO: CONTRATO + ENDPOINT DE CUENTAS ABIERTAS

> **Por qué existe:** sin esto, el pizarrón no tiene de dónde leer (§1.3). Es la dependencia oculta que la Fase 3 no resolvió.

### 4.0.1 Qué se construye

| Archivo | Cambio |
|---|---|
| [`apps/api/contracts/registry.py`](../../NUEVO-POS/apps/api/contracts/registry.py:525) | Añadir el **contrato 23** `pos.cuentas_abiertas` |
| [`apps/api/schemas.py`](../../NUEVO-POS/apps/api/schemas.py:357) | Añadir `CuentaAbiertaSalida` y `CuentasAbiertasSalida` |
| [`apps/api/routers/pos.py`](../../NUEVO-POS/apps/api/routers/pos.py:520) | Añadir `GET /pos/open-accounts?terminal_id=` |
| [`apps/api/tests/test_f2_frontera.py`](../../NUEVO-POS/apps/api/tests/test_f2_frontera.py:143) | Actualizar `test_criterio2_hay_exactamente_22_contratos` → **23** |
| [`apps/api/tests/test_f5_cuentas.py`](../../NUEVO-POS/apps/api/tests/test_f5_cuentas.py) | **Nuevo** — la puerta de F5.0 |

### 4.0.2 Decisión de diseño

El contrato 23 es de **SOLO LECTURA** y devuelve una **PROYECCIÓN** (O-23), nunca la tabla `tickets`. Respeta la **Regla 15** (respuesta ligera): cada cuenta expone **5 campos escalares**:

```text
{ id, account_num, status, total, version }
```

Filtra por `terminal_id` y `status='OPEN'`. Ordena por `created_at` ascendente (la cuenta más antigua primero, como un corcho real).

**Firma propuesta del contrato 23:**

```python
Contrato(
    numero=23,
    nombre="pos.cuentas_abiertas",
    consumidor="POS",
    proveedor="POS",
    operacion="GET /pos/open-accounts",
    entrada={"terminal_id": "String"},
    salida={"cuentas": "List[CuentaAbiertaSalida]"},
    garantias=(
        "Devuelve una PROYECCIÓN de las cuentas OPEN, no la tabla `tickets` (O-23).",
        "RESPUESTA LIGERA: cada cuenta expone EXACTAMENTE 5 campos escalares (Regla 15).",
        "Solo devuelve cuentas de la terminal pedida (RN-31).",
        "NO devuelve las líneas: leer las líneas es del contrato 21.",
    ),
    errores=("400 si `terminal_id` está vacío.",),
    estado_hoy="FASE 5.0",
)
```

### 4.0.3 Puerta de F5.0

`apps/api/tests/test_f5_cuentas.py` debe verificar:

1. El contrato 23 está declarado y tiene firma documentada.
2. `GET /pos/open-accounts` devuelve **solo** cuentas `OPEN` de la terminal pedida.
3. Cada cuenta devuelve **exactamente 5 campos** (Regla 15).
4. Una cuenta `PAID` **no** aparece.
5. Una cuenta de **otra** terminal **no** aparece (RN-31).
6. `terminal_id` vacío → **400**.

**Criterio de cierre:** `pytest tests/test_f5_cuentas.py` en verde + `test_f2_frontera.py` actualizado a 23 contratos en verde.

---

## 5. SUB-FASE 5.1 — SERVICIO FRONTEND

> **Por qué existe:** el componente no debe hablar HTTP directo; habla con un servicio que normaliza a `{outcome, reason}`.

### 5.1.1 Qué se construye

| Archivo | Cambio |
|---|---|
| [`apps/pos/src/api/client.js`](../../NUEVO-POS/apps/pos/src/api/client.js:269) | Añadir `listarCuentasAbiertas(terminalId)` |
| [`apps/pos/src/services/openAccountsService.js`](../../NUEVO-POS/apps/pos/src/services/openAccountsService.js) | **Nuevo** — envuelve la llamada en `{outcome, reason}` |
| [`apps/pos/src/services/openAccountsService.test.js`](../../NUEVO-POS/apps/pos/src/services/openAccountsService.test.js) | **Nuevo** — la puerta de F5.1 |

### 5.1.2 Decisión de diseño

Sigue el patrón exacto de [`cashService.js`](../../NUEVO-POS/apps/pos/src/services/cashService.js:1): usa `aOutcome` de [`utils/outcome.js`](../../NUEVO-POS/apps/pos/src/utils/outcome.js:68) y `withRetries` de [`utils/withRetries.js`](../../NUEVO-POS/apps/pos/src/utils/withRetries.js:55). Nunca lanza; siempre devuelve `{outcome, reason, data}`.

### 5.1.3 Puerta de F5.1

`openAccountsService.test.js` debe verificar:

1. Éxito → `{outcome:'ok', reason:null, data:{cuentas:[...]}}`.
2. Fallo de red → `{outcome:'error', reason:'red'}` (o el motivo mapeado).
3. Reintentos: usa `withRetries` (3 intentos).
4. `terminalId` vacío → no llama al cliente, devuelve error local.

**Criterio de cierre:** `npm run test` en verde con el nuevo archivo incluido.

---

## 6. SUB-FASE 5.2 — HOOK `useOpenAccounts`

> **Por qué existe:** el componente necesita estado (lista, carga, error) y acciones (refrescar, recuperar).

### 6.2.1 Qué se construye

| Archivo | Cambio |
|---|---|
| [`apps/pos/src/hooks/useOpenAccounts.js`](../../NUEVO-POS/apps/pos/src/hooks/useOpenAccounts.js) | **Nuevo** — lista, refresca, recupera |
| [`apps/pos/src/hooks/useOpenAccounts.test.jsx`](../../NUEVO-POS/apps/pos/src/hooks/useOpenAccounts.test.jsx) | **Nuevo** — la puerta de F5.2 |

### 6.2.2 Decisión de diseño

El hook expone:

```js
return { cuentas, cargando, error, refrescar, recuperarCuenta };
```

**`recuperarCuenta(ticketId)`** implementa la **lección v6.0**: *"Recuperación de cuenta = descargar versión fresca del servidor"*. NO usa el estado local; llama al contrato 21 (`pos.leer_ticket`) para traer la versión actual y evitar el bug de "trabajar sobre una copia vieja".

**Prohibición #3:** los callbacks asíncronos leen de `useRef`, nunca de estado cerrado.

**H1:** los `useEffect` dependen de primitivos (`terminalId`), no de objetos recreados.

### 6.2.3 Puerta de F5.2

`useOpenAccounts.test.jsx` debe verificar:

1. Al montar con `terminalId`, llama al servicio y puebla `cuentas`.
2. `refrescar()` vuelve a llamar al servicio.
3. `recuperarCuenta(id)` llama al contrato 21 y devuelve la versión fresca.
4. Error del servicio → `error` poblado, `cuentas` vacío.
5. Desmontar no deja timers ni llamadas pendientes (cleanup).

**Criterio de cierre:** `npm run test` en verde con el nuevo archivo incluido.

---

## 7. SUB-FASE 5.3 — COMPONENTE `OpenAccountsCorkboard`

> **Por qué existe:** es la superficie visible del pizarrón (interfaz 13 del registro).

### 7.3.1 Qué se construye

| Archivo | Cambio |
|---|---|
| [`apps/pos/src/components/OpenAccountsCorkboard.jsx`](../../NUEVO-POS/apps/pos/src/components/OpenAccountsCorkboard.jsx) | **Nuevo** — el pizarrón visual |
| [`apps/pos/src/components/OpenAccountsCorkboard.f5_3.test.jsx`](../../NUEVO-POS/apps/pos/src/components/OpenAccountsCorkboard.f5_3.test.jsx) | **Nuevo** — la puerta de F5.3 |

### 7.3.2 Decisión de diseño

Debe cumplir el registro de superficie ([`superficie/registry.py:354`](../../NUEVO-POS/apps/api/superficie/registry.py:354)):

- **Nombre exacto:** `OpenAccountsCorkboard` (el test `test_f5_superficie.py` lo exige).
- **Tipo:** Modal.
- **3 modos (R-03):** `MOSTRADOR` (grid), `COMPACTO` (grid reducido), `MOVIL` (1 columna).
- **Contenedor raíz:** `w-full max-w-[1000px] mx-auto` (fluido, R-01).
- **Paleta:** `#1a1a1a` (fondo-panel) + `#c1d72e` (acento).
- **Táctil:** cada tarjeta de cuenta ≥ 44px (`min-h-tactil`, R-04).

Cada tarjeta muestra: `account_num` (folio), `total`, y un botón "Recuperar" que dispara `recuperarCuenta(id)`.

### 7.3.3 Puerta de F5.3

`OpenAccountsCorkboard.f5_3.test.jsx` debe verificar:

1. Renderiza una tarjeta por cuenta.
2. Muestra el folio (`account_num`) y el total formateado.
3. El botón "Recuperar" llama a `recuperarCuenta` con el `id` correcto.
4. Estado vacío: "No hay cuentas abiertas".
5. Estado de carga: indicador visible.
6. Estado de error: mensaje visible.
7. Contenedor raíz fluido (`w-full`) y sin ancho fijo (R-01).
8. Cada tarjeta tiene `min-h-tactil` (R-04).

**Criterio de cierre:** `npm run test` en verde + `node scripts/guards.mjs` 7/7 en verde.

---

## 8. ORDEN DE EJECUCIÓN Y PUERTAS

```text
F5.0  contrato 23 + endpoint + esquema   →  pytest test_f5_cuentas.py + test_f2_frontera.py (23)
  │
  ▼
F5.1  openAccountsService.js             →  npm run test (servicio)
  │
  ▼
F5.2  useOpenAccounts.js                 →  npm run test (hook)
  │
  ▼
F5.3  OpenAccountsCorkboard.jsx          →  npm run test + guards 7/7
  │
  ▼
F5    Ficha de cierre de la Fase 5       →  commit + push
```

**Regla dura #4:** ninguna sub-fase empieza sin la puerta de la anterior en verde.

---

## 9. FUERA DE ALCANCE DE LA FASE 5 (delimitación explícita)

| Pieza | Por qué queda fuera | Dónde entra |
|---|---|---|
| **Cobrar desde otra terminal** (incidente T5/CAJA) | Requiere su propio contrato y su propia puerta; toca `cobrar_ticket` y la validación de `terminal_id` | **Deuda D-19** — fase futura |
| Impresión física del pizarrón | Es superficie de impresión | Fase 6 |
| Reporte diario de cuentas | Ya existe el reporte de caja | F4.1 (hecho) |
| CRM / notificaciones al cliente | Es integración externa | Fase 8 |
| Sincronización multi-sucursal | Es consolidación central | §6.6 (futura) |
| Edición de líneas desde el pizarrón | El pizarrón solo **lista y recupera**; editar es del POS | Fase 3 (hecho) |

---

## 10. AUTOCRÍTICA DEL PLAN (defectos hallados y corregidos)

### 10.1 Defectos de la propuesta inicial

| # | Defecto | Corrección aplicada |
|---|---|---|
| **D-1** | Confié en la frase del plan maestro ("lógica multi-cuenta ya incluida en `useTicketActions`") sin verificarla. **Es falsa.** | §3 documenta la afirmación falsa y establece la regla de decisión permanente. |
| **D-2** | Propuse "Fase 5 = solo el componente visual". Eso produciría un pizarrón decorativo sin datos. | §2 justifica las 4 sub-fases; §4 añade el prerrequisito del contrato 23. |
| **D-3** | No vi que "cobrar desde otra terminal" (incidente T5/CAJA) es una capacidad **inexistente** que el plan menciona como si existiera. | §9 la declara **fuera de alcance** y la registra como **deuda D-19**. |
| **D-4** | No delimité qué queda fuera de F5. | §9 delimita explícitamente 6 piezas fuera de alcance. |

### 10.2 Alineación con el objetivo del proyecto

| Verificación | Estado |
|---|---|
| ¿Contribuye al objetivo (POS completo que reemplaza al viejo)? | ✅ Sí — el pizarrón habilita varias cuentas en paralelo, hoy inexistente |
| ¿Respeta la frontera por contratos (A-02)? | ✅ Sí — F5.0 declara el contrato 23 **antes** del endpoint |
| ¿Respeta "de adentro hacia afuera" (rebanada vertical)? | ✅ Sí — Endpoint → Servicio → Hook → Componente |
| ¿Respeta la regla dura #4 (no empezar sin puerta previa en verde)? | ✅ Sí — F4 cerrada en verde (commit `2443620`) |
| ¿Documenta la lectura falsa del plan (E-14)? | ✅ Sí — §3 |
| ¿Respeta la Regla 15 (respuesta ligera)? | ✅ Sí — 5 campos por cuenta |
| ¿Respeta la lección v6.0 (versión fresca)? | ✅ Sí — F5.2 usa el contrato 21 para refrescar |

### 10.3 Riesgo residual

| Riesgo | Mitigación |
|---|---|
| El test `test_criterio2_hay_exactamente_22_contratos` fallará al añadir el 23 | F5.0 lo actualiza a 23 de forma explícita y justificada |
| `test_f5_superficie.py` ya espera `OpenAccountsCorkboard` como interfaz 13 | F5.3 usa el nombre exacto y declara los 3 modos |
| El pizarrón podría mostrar cuentas de otra terminal | F5.0 filtra por `terminal_id` y lo prueba (RN-31) |
| `useCart` es mono-cuenta; recuperar una cuenta podría pisar el carrito actual | F5.2 **no** toca `useCart`; solo devuelve la versión fresca. El cableado al carrito es decisión del POS (fuera de F5) |

---

## 11. TRAZABILIDAD — REGLA → SUB-FASE → PUERTA

| Regla | Sub-fase | Puerta |
|---|---|---|
| RN-31 (draft pertenece a su terminal) | F5.0 | `test_f5_cuentas.py` (filtro por terminal) |
| RN-32 (no escribir draft ajeno) | F5.0 | `test_f5_cuentas.py` (solo lectura) |
| RN-33 (misma terminal continúa su draft) | F5.2 | `useOpenAccounts.test.jsx` (recuperar) |
| RN-25 / RN-26 (validación de versión) | F5.2 | `useOpenAccounts.test.jsx` (versión fresca) |
| Regla 15 (respuesta ligera) | F5.0 | `test_f5_cuentas.py` (5 campos) |
| R-01 (sin ancho fijo) | F5.3 | `OpenAccountsCorkboard.f5_3.test.jsx` + guards |
| R-03 (3 modos) | F5.3 | `test_f5_superficie.py` |
| R-04 (táctil ≥ 44px) | F5.3 | `OpenAccountsCorkboard.f5_3.test.jsx` |
| O-23 (proyección, no tabla) | F5.0 | `test_f2_frontera.py` |

---

## 12. DEUDAS REGISTRADAS

| # | Deuda | Origen | Fase sugerida |
|---|---|---|---|
| **D-19** | Cobrar desde otra terminal sin sobreescribir la original (incidente T5/CAJA) | §9 de este plan | Fase futura (requiere contrato propio) |
| **D-20** | `useCart` es mono-cuenta; el cableado de "recuperar cuenta → carrito" no está definido | §10.3 de este plan | Fase futura (decisión de UX) |

---

## 13. VEREDICTO

La Fase 5 **no es "solo un componente"**. Es una rebanada vertical de 4 sub-fases con un prerrequisito (contrato 23) que la Fase 3 no resolvió. La afirmación del plan maestro de que la lógica multi-cuenta "ya está incluida" es **falsa** y queda documentada en §3 para que no vuelva a inducir a error.

**Este plan está listo para aprobación.** No se escribirá código hasta que el dueño lo apruebe (Prompt del Arquitecto §5, paso 2).
