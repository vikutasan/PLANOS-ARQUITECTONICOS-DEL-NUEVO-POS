# HALLAZGOS — Evaluación del Proceso de Pedidos (5 Oct 2026)

**Sesión:** Antigravity — Evaluación funcional del flujo de pedidos viejo POS vs nuevo POS.
**Commits:**
- `4bfea16` — P1: guardia de empaque en 2 capas.
- `f85c36d` — P5: política de pago mínimo (Vista General → POS).
- `c0b2732` — P5c: ticket de pedido con estado de pago condicional.
**Repositorio:** `github.com/vikutasan/NUEVO-POS`.

---

## 1. Hallazgos del Modal de Programación de Pedidos

### P1 — ⛔ Falta cobrar empaques como producto (CORREGIDA parcialmente)

**Hallazgo:** El viejo POS (`ProgramacionPedidoModal.jsx`, 354 líneas) listaba
todos los productos con `nature === 'EMPAQUE'` del catálogo y permitía al cajero
agregarlos al carrito directamente desde el modal. El nuevo POS solo marca
`PROPIO` / `CAJA` sin catálogo de empaques ni agregado al carrito.

**Decisión:** NO portar el selector de empaques DENTRO del modal (SRP: el modal
solo captura datos del pedido, no modifica el carrito). En cambio, implementar
una **guardia de 2 capas** que asegura que el cajero no olvide cobrar el empaque:

**Implementación — Capa 1 (badge pasivo en header):**
- `POSHeader.jsx`: nuevos props `empaqueRequerido` y `empaqueEnCarrito`.
- Si `empaqueRequerido && !empaqueEnCarrito`, el badge cambia de
  "📦 Pedido tentativo" (verde) → "📦 ⚠️ Sin empaque" (rojo, pulsante).
- Es un recordatorio visual constante pero no intrusivo.

**Implementación — Capa 2 (modal al cobrar):**
- `RetailVisionPOS.jsx`: nuevo estado `avisoEmpaque`.
- `abrirCheckoutConGuardia()` sustituye a `setCheckoutAbierto(true)` en los
  2 puntos donde el cajero pulsa "Cobrar" (desktop y móvil).
- Si `empaqueRequerido && !empaqueEnCarrito`:
  - Se muestra modal: "📦 Sin empaque en la cuenta".
  - Opción 1: "Volver y agregar empaque" (cierra modal, cajero agrega).
  - Opción 2: "Cobrar sin empaque" (abre checkout normalmente).
- NO bloquea: es advertencia, no prohibición. El cajero tiene la última palabra.

**Por qué 2 capas:**
- El badge (Capa 1) avisa TEMPRANO mientras el cajero arma la cuenta.
- El modal (Capa 2) avisa TARDE como red de seguridad al cobrar.
- Juntos cubren todo el flujo sin ser invasivos.

**Por qué NO poner la guardia al cerrar el modal de programación:**
El cajero podría agregar productos (y empaques) DESPUÉS de programar el pedido.
Advertir al cerrar el modal sería un falso positivo.

### P2 — ⚠️ Clave `DOMICILIO` vs `DELIVERY` (NO CORREGIDA — deliberado)

**Hallazgo:** El viejo POS usa `DOMICILIO` como valor del tipo de entrega;
el nuevo usa `DELIVERY`.

**Decisión:** Dejar `DELIVERY`. Razones:
1. El viejo POS casi no procesa pedidos (decisión del usuario).
2. No hay migración de datos pendiente.
3. `DELIVERY` es más estandarizado para futuras integraciones.

### P3 — ✅ Doble copia del ticket (F12.2 — NUEVO)

**Hallazgo:** El viejo POS imprime una sola copia del ticket de pedido.
El nuevo imprime doble copia (CLIENTE + COMERCIO) con la plantilla F12.2.

**Decisión:** Feature nueva, ya implementada. Superior al viejo POS.

### P4 — ✅ Lectura fresca del pedido (contrato 16 — NUEVO)

**Hallazgo:** El viejo POS lee el `orderData` inline con el ticket (puede
quedar stale si otra terminal lo modificó). El nuevo POS hace un GET separado
(contrato 16) para traer la versión fresca.

**Decisión:** Feature nueva, ya implementada. Superior al viejo POS.

---

## 2. Cumplimiento DT-09 del modal de programación

| Regla | Estado |
|-------|--------|
| R-01 (sin anchos fijos) | ✅ `max-w-[640px]` fluido |
| R-02 (mínimo 10px) | ✅ `text-[10px]` mínimo |
| R-03 (3 modos) | ✅ `grid-cols-1 sm:grid-cols-2` |
| R-04 (≥44px) | ✅ `min-h-tactil` en todos los botones |
| Tokens semánticos | ✅ `bg-fondo-panel`, `text-crema-ticket`, `bg-acento` |
| Accesibilidad | ✅ `role="dialog"`, `aria-modal`, `aria-label`, `aria-pressed` |

---

## 3. Archivos modificados

| Archivo | Cambio |
|---------|--------|
| `POSHeader.jsx` | Props `empaqueRequerido` + `empaqueEnCarrito`, badge condicional |
| `RetailVisionPOS.jsx` | Estado `avisoEmpaque`, `abrirCheckoutConGuardia()`, modal de advertencia, `politicaPagoPedido`, guardia P5 |
| `client.js` | `getSettingValue(key)` — lectura fail-safe de settings de Vista General |
| `OrderProgrammingModal.jsx` | Prop `porcentajePagoMinimo`, texto de confirmación dinámico |
| `ticketGenerator.js` | `bloqueEstadoPagoPedido()` — estado de pago condicional (100% limpio / parcial con desglose) |
| `service.py` (ERP) | Seed `order_min_payment_pct` en `system_settings` |

---

## 4. Hallazgos de Política de Pago (P5)

### P5 — ⛔ No existía política configurable de pago para pedidos (CORREGIDA)

**Hallazgo:** Ni el viejo POS ni el nuevo tenían una forma de configurar qué
porcentaje del pago se requiere para enviar un pedido a preparación. El texto
"100%" estaba hardcodeado.

**Decisión:** Crear contrato #18 (`configuracion.leer_politica`): el POS lee
la política de Vista General al montar. Default seguro: 100% si no hay conexión.
Se sembraron las siguientes piezas:

1. **ERP viejo:** seed `order_min_payment_pct = "100"` en `system_settings`.
2. **POS:** `getSettingValue()` en client + estado `politicaPagoPedido` + guardia en `confirmarCobro`.
3. **Modal:** texto dinámico de confirmación (100% vs "al menos el X%").

### P5c — ⛔ Ticket de pedido no reflejaba el estado de pago (CORREGIDA)

**Hallazgo:** El ticket siempre decía "PAGADO - PENDIENTE DE RECOLECCIÓN"
sin importar si el pago era total o parcial.

**Decisión (Opción C):** Formato condicional:
- **Pago completo (≥100%):** Muestra "PAGADO AL 100% - PENDIENTE DE RECOLECCIÓN/ENTREGA"
  (sin ruido de porcentajes — limpio).
- **Pago parcial (<100%):** Muestra desglose:
  - `CUBIERTO (X%): $monto`
  - `RESTANTE (Y%): $monto`
  - `COBRAR RESTANTE AL RECOGER/ENTREGAR`

El porcentaje se pasa por `ticket.payment_covered_pct` (default 100).
Hoy siempre es 100% porque el checkout no permite pagos parciales aún.
Cuando se implemente, el ticket lo refleja automáticamente.

---

## 5. DT-10 — Contrato obligatorio

Esta sesión originó la directriz transversal **DT-10**: toda funcionalidad
inter-modular DEBE tener su contrato formal en `CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md`
antes de considerarse terminada. Nació del near-miss de P5.
