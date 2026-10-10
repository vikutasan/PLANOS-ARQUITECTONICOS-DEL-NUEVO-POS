# PLAN DE FIX — BUG-08: la CAJA no puede cobrar cuentas de otras terminales

**Fecha:** 10 de octubre de 2026
**Estado:** PLANEADO (pendiente de aprobación)
**Severidad:** Alta — bloquea la operación real de la caja
**Repos:** `NUEVO-POS` (código) · `PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS` (este documento)

---

## 1. EL SÍNTOMA REPORTADO (verbatim del usuario)

> "quise cobrar una cuenta y salio ⚠️ No se pudo completar
> No hay turno de caja abierto para esta terminal
> Cerrar"

El operador habilitó una terminal como **CAJA**, abrió su turno de caja, recuperó del pizarrón una cuenta que pertenecía a **OTRA terminal** (p. ej. TERM-01) y al pulsar **CONFIRMAR PAGO** el backend respondió con el 400 de RN-49.

---

## 2. LA CAUSA RAÍZ (confirmada leyendo el código)

En [`cobrar_ticket()`](../NUEVO-POS/apps/api/routers/pos.py:499) el backend deriva el turno de caja de la **terminal de ORIGEN del ticket**, no de la terminal que COBRA:

```python
# pos.py:528  ← EL BUG
sesion_caja = await _sesion_caja_activa_o_400(db, ticket.terminal_id)
```

Y [`_sesion_caja_activa_o_400()`](../NUEVO-POS/apps/api/routers/pos.py:125) busca un turno `OPEN` **de esa terminal**:

```python
select(CashSession).where(
    CashSession.terminal_id == terminal_id,   # ← la terminal del TICKET
    CashSession.status == "OPEN",
)
```

**Consecuencia:** si la cuenta nació en TERM-01 (que no tiene turno abierto) y la CAJA (TERM-06) sí lo tiene, el cobro se rechaza con "No hay turno de caja abierto para esta terminal" — aunque la CAJA esté perfectamente habilitada.

### 2.1 Por qué el frontend NO lo puede arreglar solo

El frontend **ya tiene** el turno correcto en estado ([`turnoCaja`](../NUEVO-POS/apps/pos/src/RetailVisionPOS.jsx:216), poblado por [`refrescarTurnoCaja()`](../NUEVO-POS/apps/pos/src/RetailVisionPOS.jsx:380) con `caja.obtenerTurnoActivo(terminalEfectiva)`). El problema es que **no lo envía**: [`acciones.cobrar()`](../NUEVO-POS/apps/pos/src/RetailVisionPOS.jsx:779) solo manda `{ ticketId, version }`, y [`CobrarTicketEntrada`](../NUEVO-POS/apps/api/schemas.py:224) **no tiene** el campo `cash_session_id`. El backend, al no recibirlo, cae al comportamiento viejo (derivar de `ticket.terminal_id`).

---

## 3. CÓMO LO RESUELVE EL VIEJO POS (el precedente que manda)

El usuario pidió explícitamente: *"revisa como resuelve esta pregunta el viejo pos y ahi esta tu respuesta"*. La investigación confirmó el modelo:

1. **El frontend envía el turno de la terminal que COBRA.**
   [`apps/pos/hooks/useTicketActions.js:164-175`](../../apps/pos/hooks/useTicketActions.js) arma el payload con `cash_session_id: cashSessionId`, donde `cashSessionId` proviene de la terminal seleccionada (la CAJA), no de la cuenta.

2. **El backend escribe lo que el cliente manda, sin reinterpretarlo.**
   [`apps/api/modules/pos/service.py:146`](../../apps/api/modules/pos/service.py): `db_ticket.cash_session_id = ticket.cash_session_id`.

3. **El `terminal_id` del ticket NUNCA se sobreescribe.**
   Comentario textual en [`apps/api/modules/pos/service.py:164-165`](../../apps/api/modules/pos/service.py):
   > *"El terminal_id original NUNCA se sobreescribe. La CAJA u otra terminal que cobra solo registra el cashed_by_id y cash_session_id."*

**Lección del incidente "Ticket Secuestrado por la CAJA" (16/Jun/2026):** el bug histórico fue que la CAJA **reescribía** el `terminal_id` del ticket al cobrarlo, "secuestrando" la cuenta. El fix del viejo POS fue separar las dos responsabilidades: **el `terminal_id` es la trazabilidad del ORIGEN (inmutable); el `cash_session_id` es dónde se contó el dinero (lo elige quien cobra).** Este plan respeta esa separación al pie de la letra.

---

## 4. EL PRINCIPIO RECTOR (vocabulario del nuevo POS)

> **Toda terminal es una caja en potencia.**

No existe "la terminal CAJA" como entidad especial. Cualquier terminal con un turno de caja `OPEN` puede cobrar cuentas de cualquier otra. El dinero se cuenta en **la caja que lo recibió** (RN-53), y la cuenta conserva **su terminal de origen** para siempre (RN-12, `terminal_id` inmutable).

---

## 5. LA DECISIÓN ARQUITECTÓNICA

**El turno de caja lo determina la terminal que COBRA, y el frontend lo declara; el backend lo VALIDA (nunca lo confía).**

- **Contrato 5 (`CobrarTicketEntrada`)** gana un campo **opcional** `cash_session_id: UUID | None`.
- **Retrocompatibilidad:** si el campo falta, el backend cae al comportamiento actual (derivar de `ticket.terminal_id`). Ningún cliente viejo se rompe.
- **Seguridad E-13 (el backend es la autoridad):** el backend **no confía** en el id recibido. Lo valida con **tres comprobaciones** antes de usarlo:
  1. **Existe** el `CashSession` con ese id → si no, 400.
  2. **Está `OPEN`** (RN-55: una sesión cerrada es inmutable) → si no, 400.
  3. **Su terminal tiene una sesión de terminal activa** (RN-24: no se opera sin sesión) → si no, 400.
- **`ticket.terminal_id` NUNCA se toca.** El cobro solo escribe `cash_session_id`, `status`, `version`, `payment_details` y `cashed_by_name`.

---

## 6. VERIFICACIÓN PREVIA OBLIGATORIA (REGLA DURA 2)

Antes de escribir una sola línea de código, confirmar leyendo el archivo completo:

- [ ] [`cobrar_ticket()`](../NUEVO-POS/apps/api/routers/pos.py:499) **no** asigna `ticket.terminal_id` en ninguna rama (confirmado en la lectura: solo toca `status`, `version`, `payment_details`, `cash_session_id`, `cashed_by_name`).
- [ ] Firma de [`_sesion_activa_o_404()`](../NUEVO-POS/apps/api/routers/pos.py:109) es `(db, terminal_id) -> TerminalSession` — reutilizable para la validación 3.
- [ ] El import de `CashSession` ya existe en `pos.py` (lo usa `_sesion_caja_activa_o_400`).
- [ ] `UUID` ya está importado en `schemas.py` (lo usa `AbrirTurnoEntrada`).

---

## 7. LOS CAMBIOS, ARCHIVO POR ARCHIVO

### 7.1 Backend — [`schemas.py`](../NUEVO-POS/apps/api/schemas.py:224) (contrato 5)

Añadir a `CobrarTicketEntrada`:

```python
# BUG-08 — El turno de caja de la terminal que COBRA (no la de origen del
# ticket). Una terminal con turno abierto puede cobrar cuentas de OTRAS
# terminales; el dinero debe contarse en la caja que lo recibió (RN-53).
# Opcional por retrocompatibilidad: si falta, el backend cae al
# comportamiento anterior (turno de la terminal del ticket). El backend
# VALIDA el turno recibido (E-13): existe + OPEN (RN-55) + su terminal
# tiene sesión activa (RN-24).
cash_session_id: UUID | None = None
```

### 7.2 Backend — [`contracts/registry.py`](../NUEVO-POS/apps/api/contracts/registry.py:211) (contrato del cobro)

> **Nota:** el contrato 5 del registro es `pos.eventos_auditables` (Auditoría). El contrato del cobro es el de la matriz del POS (`pos.cobrar_ticket`). Localizar su entrada en `CONTRATOS` y añadir `"cash_session_id": "UUID | None (opcional, BUG-08)"` a su `entrada`, más una garantía: *"El turno de caja lo determina la terminal que COBRA; el backend lo valida (existe + OPEN + sesión activa). El `terminal_id` del ticket NUNCA se sobreescribe (RN-12)."*

### 7.3 Backend — [`pos.py`](../NUEVO-POS/apps/api/routers/pos.py:125) (helper nuevo + uso)

**Helper nuevo** (junto a `_sesion_caja_activa_o_400`):

```python
async def _sesion_caja_por_id_o_400(db: AsyncSession, cash_session_id: UUID) -> CashSession:
    """Valida un turno de caja DECLARADO por el cliente (E-13) y lo devuelve.

    BUG-08 — La terminal que COBRA declara su turno; el backend NO lo confía.
    Tres validaciones:
      1. El turno existe.
      2. Está OPEN (RN-55: una sesión cerrada es inmutable).
      3. La terminal del turno tiene una sesión de terminal activa (RN-24).
    """
    sesion = (
        await db.execute(select(CashSession).where(CashSession.id == cash_session_id))
    ).scalars().first()
    if sesion is None:
        raise ReglaViolada("RN-49", "El turno de caja indicado no existe", 400)
    if sesion.status != "OPEN":
        raise ReglaViolada("RN-55", "El turno de caja indicado está cerrado", 400)
    # RN-24: la terminal del turno debe tener sesión activa.
    await _sesion_activa_o_404(db, sesion.terminal_id)
    return sesion
```

**Uso en `cobrar_ticket`** (reemplaza la línea 528):

```python
# BUG-08 — El turno lo determina la terminal que COBRA. Si el cliente lo
# declara, se VALIDA (E-13); si no, se cae al comportamiento retrocompatible
# (turno de la terminal de origen del ticket).
if entrada.cash_session_id is not None:
    sesion_caja = await _sesion_caja_por_id_o_400(db, entrada.cash_session_id)
else:
    sesion_caja = await _sesion_caja_activa_o_400(db, ticket.terminal_id)
```

> `ticket.terminal_id` **no se toca** en ninguna rama. El resto del endpoint queda igual.

### 7.4 Frontend — [`useTicketActions.js`](../NUEVO-POS/apps/pos/src/hooks/useTicketActions.js:243) (`cobrar`)

Reenviar el turno si el llamador lo pasa en `opciones`:

```js
cliente.cobrarTicket(idTicket, {
  payment_details: paymentDetails,
  version,
  ...(opciones.cashSessionId ? { cash_session_id: opciones.cashSessionId } : {}),
}),
```

Actualizar el JSDoc de `cobrar` para documentar `opciones.cashSessionId`.

### 7.5 Frontend — [`RetailVisionPOS.jsx`](../NUEVO-POS/apps/pos/src/RetailVisionPOS.jsx:779) (llamada de cobro)

Pasar el turno de la terminal que cobra (ya está en `turnoVigente`):

```js
const pagado = await acciones.cobrar(paymentDetails, {
  ticketId: ticketIdRef.current,
  version: carrito.version,
  cashSessionId: turnoVigente.cash_session_id,
});
```

> `turnoVigente` ya se calcula en la guarda de RN-49 (líneas 685-692) y es el turno de `terminalEfectiva` (la que cobra). No hay que añadir estado nuevo.

---

## 8. LOS TESTS (la compuerta)

### Backend — `apps/api/tests/test_f3_atomico.py` (o ficha nueva `test_bug08_caja.py`)

| # | Test | Qué prueba |
|---|------|-----------|
| 1 | `test_caja_cobra_cuenta_de_otra_terminal` | Turno OPEN en TERM-06, ticket en TERM-01 sin turno → **200**, ticket PAID. |
| 2 | `test_cobro_no_sobreescribe_terminal_id` | Tras el cobro, `ticket.terminal_id` sigue siendo `TERM-01` (RN-12). |
| 3 | `test_cobro_retrocompatible_sin_cash_session_id` | Sin el campo → usa el turno de la terminal del ticket (comportamiento viejo intacto). |
| 4 | `test_cobro_turno_inexistente_da_400` | `cash_session_id` de un UUID que no existe → 400. |
| 5 | `test_cobro_turno_cerrado_da_400` | `cash_session_id` de un turno `CLOSED` → 400 (RN-55). |
| 6 | `test_cobro_turno_sin_sesion_de_terminal_da_400` | Turno OPEN pero su terminal sin sesión activa → 400 (RN-24). |
| 7 | `test_cobro_regresion_turno_de_la_propia_terminal` | El caso normal (misma terminal) sigue dando 200. |

### Frontend — `apps/pos/src/hooks/hooks.f3_3.test.jsx` (o ficha nueva)

| # | Test | Qué prueba |
|---|------|-----------|
| 8 | `cobrar reenvía cash_session_id cuando se pasa` | `api.cobrarTicket` recibe `{ payment_details, version, cash_session_id }`. |
| 9 | `cobrar omite cash_session_id cuando no se pasa` | El cuerpo NO incluye la clave (retrocompat). |
| 10 | `RetailVisionPOS pasa el turno de la terminal que cobra` | El `cashSessionId` enviado es el de `turnoVigente`, no el del ticket. |

### Contrato

| # | Test | Qué prueba |
|---|------|-----------|
| 11 | `test_contrato_cobro_declara_cash_session_id` | El registro de contratos declara el campo en la entrada del cobro. |

---

## 9. ORDEN DE EJECUCIÓN

1. **Verificación previa** (§6) — leer y confirmar.
2. **Backend:** `schemas.py` → `contracts/registry.py` → `pos.py` (helper + uso).
3. **Backend tests** 1-7 + contrato 11 → correr pytest en Docker.
4. **Frontend:** `useTicketActions.js` → `RetailVisionPOS.jsx`.
5. **Frontend tests** 8-10 → correr vitest.
6. **Suites completas** sin regresiones (backend 356+, frontend 812+).
7. **Ficha** `FICHA_FIX_BUG08_CAJA_COBRA_CUENTAS_AJENAS.md` + actualizar `ARQUITECTURA_TERMINALES_Y_CAJA.md` y `DECISIONES_ARQUITECTONICAS_DEL_NUEVO_POS.md`.
8. **Commit + push** (dos repos si aplica).

---

## 10. LO QUE ESTE PLAN **NO** HACE (límites explícitos)

- **NO** toca `ticket.terminal_id` (RN-12, trazabilidad del origen).
- **NO** cambia RN-49 ni RN-55 (siguen vigentes; solo se elige *cuál* turno se valida).
- **NO** rompe la retrocompatibilidad (el campo es opcional).
- **NO** confía en el cliente (E-13: el backend valida las 3 condiciones).
- **NO** introduce "la terminal CAJA" como entidad especial (toda terminal es una caja en potencia).
- **NO** arregla las deudas (a) naming BUG-02/03 ni (c) `_es_admin()` — fuera de alcance.

---

## 11. RIESGOS Y MITIGACIONES

| Riesgo | Mitigación |
|--------|-----------|
| Un cliente viejo no manda `cash_session_id` | Campo opcional + rama retrocompatible (test 3). |
| Un cliente malicioso manda un turno ajeno | Validación E-13: existe + OPEN + sesión activa (tests 4-6). |
| El arqueo cuenta el dinero en la caja equivocada | El turno es el de la terminal que COBRA (RN-53); test 1 lo verifica. |
| Se "secuestra" el ticket (incidente 16/Jun/2026) | `terminal_id` inmutable; test 2 lo blinda. |
