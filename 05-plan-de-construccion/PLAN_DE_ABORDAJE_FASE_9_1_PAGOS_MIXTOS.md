# PLAN DE ABORDAJE — FASE 9.1 POR PARTES

**Proyecto:** POS Nuevo "R de Rico"
**Fase:** 9.1 — Pagos mixtos (capacidad de backend)
**Versión del plan:** 1.0
**Fecha:** 30 Sep 2026
**Estado:** 📝 PROPUESTO — pendiente de aprobación
**Autor:** Arquitecto del Nuevo POS

---

## 0. POR QUÉ EXISTE ESTA FASE

La F9.0 rescató tres piezas de UX del viejo POS **sin tocar el backend**. Dejó
explícitamente fuera **una capacidad mayor**: los **pagos mixtos** (abonar, en la
misma cuenta, efectivo + tarjeta + transferencia, en cualquier combinación).

El viejo POS **sí** los tenía: [`CheckoutScreen.jsx`](../../../ERP-R-DE-RICO/apps/pos/components/CheckoutScreen.jsx:25)
mantiene un arreglo `payments[]` donde cada abono tiene `{method, amount, received, cambio, type}`
y solo permite finalizar cuando `totalPaid >= total`. El nuevo POS, en cambio,
cobra con **un solo método** ([`CheckoutScreen.jsx`](../../../NUEVO-POS/apps/pos/src/components/CheckoutScreen.jsx:170)
envía `{metodo, recibido, cambio}`).

Esta fase **NO es un rescate de UI**: es un **cambio de contrato de cobro**. Por eso
tiene su propio plan, sus reglas de negocio y sus tests.

---

## 1. DIAGNÓSTICO VERIFICADO (REGLA DURA 2 — verificar, no asumir)

Antes de proponer nada, se leyó el código real. Hallazgos:

### 1.1 El contrato de cobro hoy acepta un solo método

- [`CobrarTicketEntrada`](../../../NUEVO-POS/apps/api/schemas.py:164) recibe
  `payment_details: dict[str, Any]` **libre** (JSONB) + `version`.
- [`cobrar_ticket`](../../../NUEVO-POS/apps/api/routers/pos.py:339) guarda
  `ticket.payment_details = entrada.payment_details` **tal cual**, sin validar su forma.
- El POS nuevo envía `{metodo, recibido, cambio}` — **un** método.

### 1.2 ⚠️ EL HALLAZGO CRÍTICO: el arqueo lee UN solo método

[`_ventas_en_efectivo()`](../../../NUEVO-POS/apps/api/routers/cash.py:88) es la función
que alimenta el **efectivo esperado** del corte (RN-53). Hoy hace:

```python
detalles = t.payment_details or {}
metodo = str(detalles.get("metodo", "")).upper()   # ← UN solo string
if metodo in {"EFECTIVO", "CREDITO", "DEBITO", "TRANSFERENCIA"}:
    pagos.append({"metodo": metodo, "monto": t.total})  # ← el TOTAL completo
```

**Consecuencia si se envían pagos mixtos sin tocar esto:** un ticket de $100 pagado
$40 efectivo + $60 tarjeta se clasificaría como **$100 al método que venga en
`metodo`** (o se ignoraría si `metodo` no existe). El arqueo saldría **mal** — es
exactamente la clase de bug silencioso que la cicatriz de los $453 nos enseñó a temer.

**Conclusión:** los pagos mixtos **exigen** cambiar el backend. No es opcional.

### 1.3 Las reglas de clasificación YA existen y son reutilizables

- [`rn57_clasificar_por_metodo()`](../../../NUEVO-POS/apps/api/rules/registry.py:459)
  valida que el método sea `EFECTIVO|CREDITO|DEBITO|TRANSFERENCIA`.
- [`rn58_clasificacion_alimenta_resumen()`](../../../NUEVO-POS/apps/api/rules/registry.py:467)
  ya suma una **lista** de pagos `[{metodo, monto}]` a un resumen por método.

Es decir: la infraestructura de clasificación **ya soporta N pagos**. Lo que falta es
que el cobro **guarde** N pagos y que el arqueo **los lea**.

### 1.4 El viejo POS ya resolvió la UX (y sus trampas)

El viejo POS maneja con cuidado:
- **Efectivo con cambio:** `realAbono = min(amount, pendingAmount)`, `cambio = amount - realAbono`.
  El cambio **no** se cuenta como pago; solo el abono real.
- **Edición/borrado de un abono** antes de finalizar.
- **Bloqueo de doble-click** (`isProcessing` antes del `await`).
- **No finalizar con saldo pendiente** (`totalPaid >= total`).

Estas trampas se heredan como **requisitos**, no se reinventan.

---

## 2. ALCANCE

### 2.1 Lo que SÍ hace esta fase

| # | Entregable | Capa | Riesgo |
|---|-----------|------|--------|
| 1 | **Contrato de cobro acepta N pagos** (`payment_details.pagos[]`) | Backend (schema + router) | Medio (toca el cobro) |
| 2 | **Regla nueva RN-94: la suma de pagos = total** | Backend (rules) | Bajo |
| 3 | **Arqueo lee N pagos** (`_ventas_en_efectivo` clasifica por lista) | Backend (cash) | Medio (toca el corte) |
| 4 | **UI de pagos mixtos** (agregar/editar/borrar abonos) | Frontend (CheckoutScreen) | Medio |
| 5 | **Retrocompatibilidad:** un pago único sigue funcionando igual | Backend | Bajo |

### 2.2 Lo que NO hace esta fase (explícito)

- **NO** cambia el modelo de datos (`payment_details` sigue siendo JSONB). No hay migración.
- **NO** toca la tabla `ticket_items` ni el cobro atómico.
- **NO** implementa devoluciones ni pagos parciales que dejen saldo (el ticket se cobra
  completo o no se cobra).
- **NO** toca el sistema de temas ni la impresión térmica (el ticket impreso ya lee
  `payment_details`; se ajustará solo si el formato lo requiere — ver §3.5).
- **NO** introduce una cola local (decisión de F9.0, se mantiene).

---

## 3. SUB-FASES (de adentro hacia afuera)

### 3.1 F9.1.0 — Contrato y regla (el cimiento)

**Objetivo:** que el backend **entienda y valide** N pagos, sin cambiar la UI todavía.

**Cambios:**
1. **Schema** ([`schemas.py`](../../../NUEVO-POS/apps/api/schemas.py:164)): documentar la
   forma canónica de `payment_details`:
   ```json
   {
     "pagos": [
       {"metodo": "EFECTIVO", "monto": "40.00", "recibido": "50.00", "cambio": "10.00"},
       {"metodo": "TARJETA",  "monto": "60.00", "tipo": "DEBITO"}
     ],
     "cajero": "Nombre"
   }
   ```
   Se mantiene `dict[str, Any]` (JSONB libre) para **no romper** tickets viejos.
2. **Regla nueva RN-94** en [`rules/registry.py`](../../../NUEVO-POS/apps/api/rules/registry.py:1):
   `rn94_suma_de_pagos_cuadra_total(pagos, total)` → lanza `ReglaViolada` si
   `sum(monto) != total` (tolerancia 0.00; el dinero es Decimal, DT-02).
3. **Regla nueva RN-95:** `rn95_metodos_de_pago_validos(pagos)` → reutiliza
   `rn57_clasificar_por_metodo` sobre cada pago (no duplica la lista de válidos).
4. **Router** ([`pos.py`](../../../NUEVO-POS/apps/api/routers/pos.py:339)): en
   `cobrar_ticket`, si `payment_details` trae `pagos[]`, validar RN-94/RN-95 **antes**
   de guardar. Si trae la forma vieja (`metodo`), se **normaliza** a `pagos[]` de un
   elemento (retrocompatibilidad).

**Puerta (gate):** tests de las reglas RN-94/RN-95 + test de que un cobro viejo
(`{metodo, recibido, cambio}`) sigue funcionando y se normaliza.

**Riesgo:** bajo. No cambia el comportamiento observable para el caso de un pago.

---

### 3.2 F9.1.1 — El arqueo lee N pagos (la corrección crítica)

**Objetivo:** que [`_ventas_en_efectivo()`](../../../NUEVO-POS/apps/api/routers/cash.py:88)
clasifique **cada abono** por su método, no el total por un método.

**Cambios:**
1. Reescribir `_ventas_en_efectivo` para:
   - Si `payment_details` trae `pagos[]` → construir `[{metodo, monto}]` con **cada**
     abono y pasarlo a `rn58_clasificacion_alimenta_resumen`.
   - Si trae la forma vieja (`metodo`) → comportamiento actual (un pago = total).
   - **Nunca** sumar el total completo a un método cuando hay varios pagos.
2. Aplicar el mismo criterio en el **reporte diario** (línea ~359 de `cash.py`) si
   clasifica por método.

**Puerta (gate):** test de caja con un ticket mixto ($40 efectivo + $60 tarjeta) que
verifique que el **efectivo esperado sube solo $40**, no $100. Este es el test que
**blinda contra la cicatriz**.

**Riesgo:** medio. Es la corrección que evita el bug silencioso. Se hace **antes** de
la UI para que la UI nunca pueda producir un arqueo malo.

---

### 3.3 F9.1.2 — Servicio y hook de cobro (frontera del POS)

**Objetivo:** que el POS pueda **enviar** N pagos por el contrato, con la disciplina
de servicios del proyecto.

**Cambios:**
1. **Servicio** (`checkoutService.js` o extender el existente): función pura que
   **construye** el `payment_details` canónico a partir de una lista de abonos de UI.
   Devuelve `{outcome, reason, data}` (patrón del proyecto), nunca lanza.
2. **Validación de frontera:** el servicio **no** deja construir un payload cuya suma
   no cuadre (defensa en profundidad; el backend igual valida RN-94).
3. **Hook** (`useCheckout` o extender `useTicketActions`): expone
   `agregarPago`, `editarPago`, `borrarPago`, `puedeCobrar`, `cambio`, `faltante`.
   Inyectable para tests (patrón del proyecto).

**Puerta (gate):** tests del servicio (construye payload correcto, rechaza suma
incorrecta) + tests del hook (agregar/editar/borrar, cálculo de cambio y faltante).

**Riesgo:** bajo. Es lógica pura, testeable sin red.

---

### 3.4 F9.1.3 — UI de pagos mixtos (la cara visible)

**Objetivo:** llevar la UX del viejo POS al nuevo, con los tokens del tema.

**Cambios en** [`CheckoutScreen.jsx`](../../../NUEVO-POS/apps/pos/src/components/CheckoutScreen.jsx:48):
1. Lista de **abonos** (chips/tarjetas) con método, monto, y cambio si es efectivo.
2. Botón **"Agregar pago"** (usa el monto capturado + el método seleccionado).
3. **Editar** y **borrar** un abono antes de confirmar.
4. **Resumen en vivo:** total, abonado, **faltante**, **cambio**.
5. **Bloqueo de confirmar** si `faltante > 0` (con mensaje claro).
6. Se **conserva** el [`TecladoNumerico`](../../../NUEVO-POS/apps/pos/src/components/TecladoNumerico.jsx:63)
   de F9.0.2 y los billetes rápidos.
7. Se **conserva** el caso de un solo pago (no romper el flujo actual).

**Puerta (gate):** tests de componente — agregar 2 pagos, ver faltante, editar, borrar,
confirmar habilitado solo cuando cuadra; y el caso de un pago único (regresión).

**Riesgo:** medio. Es UI con estado; se blinda con tests y con el gate de F9.1.1 ya verde.

---

### 3.5 F9.1.4 — Impresión y cierre (evidencia)

**Objetivo:** que el ticket impreso muestre los pagos mixtos y cerrar la fase.

**Cambios:**
1. Revisar [`ticketGenerator.js`](../../../NUEVO-POS/apps/pos/src/utils/ticketGenerator.js:122):
   si imprime el método, que liste los abonos cuando haya varios. Si ya imprime
   `payment_details` genérico, ajustar el formato.
2. **Ficha de cierre** `FICHA_F9_1_PAGOS_MIXTOS.md` con evidencia.
3. **Plan Maestro:** registrar F9.1 como CERRADA (subir versión).
4. **CI completo** verde + commit/push en ambos repos + registro de hashes.

**Puerta (gate):** `npm run ci` verde; test de impresión con 2 pagos.

**Riesgo:** bajo.

---

## 4. CRITERIOS GLOBALES DE ACEPTACIÓN

| Criterio | Cómo se verifica |
|----------|------------------|
| Un cobro de un solo pago sigue funcionando **idéntico** | Test de regresión del flujo actual |
| Un cobro mixto (efectivo + tarjeta) se guarda con N pagos | Test de API |
| El **arqueo** clasifica cada abono por su método | Test de caja: $40 efectivo + $60 tarjeta → efectivo esperado +$40 |
| La suma de pagos **debe** cuadrar el total | RN-94 + test que rechaza suma incorrecta |
| El cambio **no** se cuenta como pago | Test: recibido $50, abono $40 → cambio $10, efectivo +$40 |
| La UI bloquea confirmar con saldo pendiente | Test de componente |
| **Ningún** color hardcodeado nuevo | Guard de CI |
| **Ninguna** tabla ajena leída por el POS | Guard de CI (A-02) |
| `npm run ci` verde | CI completo |

---

## 5. TRAZABILIDAD (regla → test)

| Regla | Enunciado | Test |
|-------|-----------|------|
| RN-94 | La suma de los pagos cuadra el total del ticket | `test_f9_1_pagos.py::test_rn94` |
| RN-95 | Cada pago usa un método válido (RN-57) | `test_f9_1_pagos.py::test_rn95` |
| RN-53 | Efectivo esperado = fondo + entradas − salidas + ventas en efectivo | `test_f9_1_pagos.py::test_arqueo_mixto` |
| RN-58 | La clasificación alimenta el resumen | `test_f9_1_pagos.py::test_resumen_mixto` |

---

## 6. LO QUE ESTA FASE **NO** HACE (para evitar confusión futura)

- **NO** permite dejar saldo pendiente (no es venta a crédito; eso es otro módulo).
- **NO** implementa devoluciones.
- **NO** cambia el modelo de datos ni requiere migración.
- **NO** construye cola local.
- **NO** toca el cobro atómico de ítems (F3.2).

---

## 7. RIESGOS Y MITIGACIONES

| Riesgo | Prob. | Impacto | Mitigación |
|--------|-------|---------|-----------|
| El arqueo clasifica mal un ticket mixto | Media | **Alto** (dinero) | F9.1.1 se hace **antes** de la UI; test explícito del arqueo mixto |
| Romper el cobro de un solo pago | Media | Alto | Normalización a `pagos[]` + test de regresión |
| El cambio se cuenta como pago | Media | Alto | Test: cambio no suma al efectivo |
| Doble-click en confirmar | Baja | Medio | `procesando` antes del `await` (patrón heredado) |
| Regresión en el gate de Fase 3/4 | Baja | Medio | CI completo; props nuevas opcionales |

---

## 8. BITÁCORA DE CAMBIOS

| Versión | Fecha | Cambio |
|---------|-------|--------|
| 1.0 | 30 Sep 2026 | Plan inicial. 5 sub-fases (F9.1.0–F9.1.4). Diagnóstico verificado del arqueo. |

---

## 9. AUTOCRÍTICA

**¿Esta fase contribuye al objetivo del proyecto?** Sí, y de forma más profunda que
F9.0:

- **A favor:** recupera una capacidad **real de negocio** (cobrar mixto), no cosmética.
  Corrige un **riesgo de dinero** que hoy está latente (el arqueo asume un solo método).
  Reutiliza reglas que ya existen (RN-57/RN-58), sin duplicar.
- **En contra / límite:** toca el **cobro**, que es el corazón del POS. Por eso se hace
  "de adentro hacia afuera": primero el contrato y el arqueo (F9.1.0/F9.1.1), luego la
  UI (F9.1.3). La UI **nunca** puede producir un arqueo malo porque el backend ya lo
  impide.
- **Riesgo de sobre-ingeniería:** bajo. No se cambia el modelo de datos; `payment_details`
  ya es JSONB libre. Se añaden 2 reglas pequeñas y se corrige 1 función.
- **Honestidad:** el hallazgo de §1.2 (el arqueo lee un solo método) es la razón de ser
  de esta fase. Sin él, "pagos mixtos" sería un parche de UI que **rompería el corte en
  silencio**. Se dice claro.

**Conclusión:** la fase es **estructural y necesaria**. Se ejecuta por partes, con el
arqueo blindado antes de tocar la UI.
