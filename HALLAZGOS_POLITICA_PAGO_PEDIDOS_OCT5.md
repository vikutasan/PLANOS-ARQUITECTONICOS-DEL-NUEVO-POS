# HALLAZGO P5 — Política de Pago Mínimo para Pedidos (5 Oct 2026)

**Sesión:** Antigravity — Política configurable de pago para procesamiento de pedidos.
**Commits:**
- NUEVO-POS: `f85c36d` — P5: política de pago mínimo (Vista General → POS)
- ERP-R-DE-RICO: `4211b38` — P5: seed `order_min_payment_pct` en `system_settings`

---

## 1. Problema

Ni el viejo POS ni el nuevo tenían una política configurable para el porcentaje
mínimo de pago requerido para enviar un pedido a preparación. Ambos tenían un
texto hardcodeado "solo cuando el pago sea recibido al 100%" sin validación real.

El usuario necesita poder establecer desde Vista General si un pedido requiere
pago completo (100%) o si un porcentaje menor (ej. 50% de anticipo) es suficiente
para mandar el pedido a cocina.

## 2. Diseño

```
Vista General (ERP viejo)            POS nuevo
─────────────────────────            ─────────
system_settings                      Al montar:
  key: order_min_payment_pct         GET /api/v1/settings/order_min_payment_pct
  value: "100"                        → estado: politicaPagoPedido
  category: "orders"                  → default: 100 si no hay conexión
  input_type: "number"

                                     Al programar pedido (modal):
                                      → Muestra la política real al confirmar
                                        ("100%" o "al menos el X%")

                                     Al cobrar un PEDIDO (confirmarCobro):
                                      → Valida: montoRecibido >= total * (pct/100)
                                      → Si no cumple: error descriptivo con montos
                                      → Si cumple: procesa normalmente
```

### Principios:
- **Default seguro:** 100% si no se puede leer la configuración.
- **Se lee UNA vez** al montar el POS (sin polling — Prohibición #1).
- **NO toca el ERP instalado** — solo agrega un setting al seed (aditivo).
- **El POS solo CONSUME** la política; no la establece.
- **La Vista General ya la tendrá** cuando se profesionalice.

## 3. Implementación

### ERP viejo (`service.py`):
```python
{
    "key": "order_min_payment_pct",
    "value": "100",
    "description": "Porcentaje mínimo de pago para enviar pedido (0-100).",
    "category": "orders",
    "input_type": "number"
}
```
Entrada ADITIVA: el bucle `seed_settings` solo inserta si la clave no existe.

### Nuevo POS:

| Archivo | Cambio |
|---------|--------|
| `client.js` | `getSettingValue(key)` — lee un setting individual (fail-safe) |
| `RetailVisionPOS.jsx` | Estado `politicaPagoPedido` (default 100), lectura al montar, guardia en `confirmarCobro` |
| `OrderProgrammingModal.jsx` | Prop `porcentajePagoMinimo`, texto dinámico en confirmación |

### Flujo de la guardia:
1. El POS monta → lee `order_min_payment_pct` → guarda en `politicaPagoPedido`.
2. Si no hay conexión o el setting no existe → queda en 100%.
3. Al cobrar un PEDIDO:
   - Calcula `minimoRequerido = total × (pct / 100)`.
   - Calcula `montoRecibido` (suma de abonos o pago único).
   - Si `montoRecibido < minimoRequerido` → bloquea con error descriptivo:
     *"Para enviar este pedido se requiere al menos el X% ($Y). Monto recibido: $Z."*
   - Si cumple → procesa normalmente.

## 4. Nota sobre pagos parciales

La guardia P5 está lista para cuando se implemente **pago parcial de pedidos**
en el CheckoutScreen. Actualmente el checkout siempre requiere pago completo
(para todas las transacciones). Cuando se permita pagar un anticipo (ej. 50%),
la guardia P5 será la frontera que valide que el anticipo cumple con la política.

**Trabajo pendiente:** Modificar CheckoutScreen para aceptar montos < total
cuando el tipo es PEDIDO y la política lo permite.
