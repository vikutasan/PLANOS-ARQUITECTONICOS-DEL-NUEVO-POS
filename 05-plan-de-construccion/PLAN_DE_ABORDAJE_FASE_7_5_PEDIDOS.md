# 📋 PLAN DE ABORDAJE — FASE 7.5: PEDIDOS PROGRAMADOS (el puente POS → Pedidos → Producción)

**Versión:** 1.0
**Fecha:** 30 Sep 2026
**Autor:** Arquitecto del Nuevo POS
**Estado:** Propuesta — pendiente de aprobación del dueño
**Precedente:** `PLAN_DE_ABORDAJE_FASE_7_POR_PARTES.md` §12.3 (lo diferido)

---

## §1. EL HALLAZGO QUE ORIGINA ESTA FASE

### §1.1 Lo que el dueño detectó

> *"He detectado una omisión del plan: el POS viejo tiene la funcionalidad de procesar pedidos y
> enviarlos a producción y creo que en este nuevo POS esto no se contempló."*

### §1.2 El diagnóstico: NO es una omisión del PLAN, es una omisión de la EJECUCIÓN

Tras revisar los 4 documentos rectores y el código, la conclusión es precisa:

| Dimensión | Veredicto | Evidencia |
|---|---|---|
| **¿El plan lo documentó?** | **SÍ, en 4 lugares** | Ver §1.3 |
| **¿El código tiene el cimiento?** | **SÍ** | `models/orders.py`, `Ticket.order_type`, tabla `orders` en la migración 0001 |
| **¿El código tiene el puente?** | **NO** | No existe router de pedidos; los contratos 15/16 están declarados pero sin implementar |
| **¿Es una regresión funcional?** | **SÍ** | El POS viejo SÍ procesa pedidos y los manda a producción |

**Conclusión:** el plan está completo. Lo que falta es **ejecutar** el puente que el plan ya
describió. Esta fase cierra esa brecha.

### §1.3 Los 4 lugares donde el plan ya lo documentó

1. **`CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md` §8** — contratos **15** (`pedidos.registrar_desde_ticket`,
   `POST /orders/from-ticket`) y **16** (`pedidos.pedido_del_ticket`, `GET /orders/by-ticket/{ticket_id}`),
   con entradas, salidas, garantías y errores completos.
2. **`PLANO ARQUITECTONICO PARA EL NUEVO POS.md` §1.12** — reglas **RN-55..RN-59** (el puente del pedido).
3. **`ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md`** — **F-19**, **RN-67..RN-70**, **AC-02**.
4. **`MODELO_DE_DATOS_DEL_NUEVO_POS.md` §4.1** — la tabla `orders`.

### §1.4 Por qué se difirió (y por qué ahora toca)

`PLAN_DE_ABORDAJE_FASE_7_POR_PARTES.md` §12.3 lo difirió a "Fase 7.5 o Fase 8" porque:
1. No es voz, ni visión, ni temas — no encajaba en el hilo de la Fase 7.
2. **H-7:** su alcance no estaba definido con el dueño.

**Hoy H-7 se resuelve:** el dueño confirmó que la funcionalidad existe en el POS viejo y debe
existir en el nuevo. El alcance se define en §3 de este documento.

---

## §2. LA REGLA DURA QUE GOBIERNA ESTA FASE

### §2.1 P-01 / A-02 — Frontera por contratos

El POS viejo **viola** la frontera: importa el modelo `Order` y escribe la tabla `orders`
directamente. El POS nuevo **no puede** hacer eso.

> **El POS nuevo consume los contratos 15 y 16. Nunca importa `Order`. Nunca escribe `orders`.**

Esto es lo que hace que el POS nuevo sea un módulo del ERP y no un monolito encubierto.

### §2.2 El estado del pedido NO lo gobierna el POS

Los **14 estados** del ciclo de vida del pedido (TENTATIVO → PAGADO → TURNO_ASIGNADO →
EN_PREPARACION → PREPARADO_ENFRIAMIENTO → PREPARADO_REPOSO → LISTO_EMPAQUE → EN_EMPAQUE_PICKUP →
LISTO_PICKUP_SIN_EMPAQUE → LISTO_PICKUP_EMPACADO → EN_EMPAQUE_REPARTO → LISTO_REPARTO_EMPACADO →
EN_RUTA → ENTREGADO | CANCELADO) los gobierna **Pedidos/Producción**, no el POS.

El POS solo **proyecta** el estado del ticket al pedido (`OPEN → TENTATIVO`, `PAID → PAGADO`).
El resto de la máquina de estados vive en el módulo Pedidos (que aún no existe como módulo
separado, pero el contrato ya está declarado).

### §2.3 La regla de oro: la venta nunca se bloquea

Si el proveedor de Pedidos falla, **la venta se completa igual**. El pedido se registra
best-effort y se reconcilia después. Esto es DT-07 aplicado al puente.

---

## §3. ALCANCE DEFINIDO CON EL DUEÑO

### §3.1 Lo que SÍ entra en F7.5

| # | Entregable | Descripción |
|---|---|---|
| 1 | **Router de pedidos (API)** | Implementa los contratos 15 y 16: `POST /orders/from-ticket` y `GET /orders/by-ticket/{ticket_id}` |
| 2 | **Servicio de pedidos (frontend)** | `ordersService.js` — capa fina sobre los contratos 15/16 |
| 3 | **Hook `useOrderProgramming`** | Estado del pedido programado + cálculo de `earliest_ready_at` local (espejo del proveedor) |
| 4 | **Modal heredado** | `ProgramacionPedidoModal.jsx` — **UX heredada del POS viejo** (§6.8 del Plan Maestro) |
| 5 | **Botón "PEDIDO" en el header** | `POSHeader.jsx` — el toggle VENTA_DIRECTA ↔ PEDIDO |
| 6 | **Panel de pedido en el checkout** | Muestra el pedido programado antes de cobrar |
| 7 | **Gate de API + gate de frontend** | Criterios verificables |

### §3.2 Lo que NO entra en F7.5 (explícito)

| # | Fuera de alcance | Por qué |
|---|---|---|
| 1 | **La máquina de 14 estados** | Vive en el módulo Pedidos/Producción, que es otra fase |
| 2 | **La pantalla de Producción (KDS)** | Es del módulo Producción, no del POS |
| 3 | **El reparto (rutas, choferes, mapas)** | Es del módulo Reparto |
| 4 | **La notificación al cliente (WhatsApp/Email)** | Es la Fase 8 (CRM + Notificaciones) |
| 5 | **El cálculo de `delivery_fee` por distancia** | Requiere geocodificación; se deja en 0 y se documenta |

> **Nota crítica:** los entregables 1–7 funcionan **hoy**, sin depender de módulos inexistentes.
> El router de pedidos es autocontenido: escribe la tabla `orders` que ya existe. Cuando el módulo
> Pedidos real se construya, **solo cambia quién implementa el contrato** — el POS no se toca.

---

## §4. LA UX HEREDADA DEL POS VIEJO (§6.8 del Plan Maestro)

> *"Cuando un componente ya existe en el viejo POS, su INTEGRACIÓN se hereda; solo su
> IMPLEMENTACIÓN se reescribe."*

### §4.1 Cómo funciona en el POS viejo (verificado en el código)

| Pieza | Comportamiento heredado |
|---|---|
| **`POSHeader`** | Botón **"PEDIDO"** que llama `onOrderTypeChange('PEDIDO')`. Cuando el tipo es PEDIDO, aparece un segundo botón **"Programación del Pedido"** |
| **`ProgramacionPedidoModal`** | Captura: tipo de entrega (PICKUP/DOMICILIO), `committed_at`, nombre/teléfono del cliente, tipo de empaque (PROPIO/VENTA), dirección, notas. Calcula `earliestReady` con `calcMaxLeadTime(cart)` |
| **`CheckoutScreen`** | Muestra un panel de detalle del pedido cuando `orderData` existe |
| **Estado** | `showProgramacion` + `orderData` en el componente raíz |

### §4.2 La diferencia con el POS nuevo

El `POSHeader.jsx` del POS nuevo **no tiene** los props `orderType` / `orderData` /
`onOrderTypeChange`. Hay que **añadirlos** (es la implementación que se reescribe).

### §4.3 El modal: qué se hereda y qué se reescribe

| Se HEREDA (la integración) | Se REESCRIBE (la implementación) |
|---|---|
| Los campos del formulario | El estilo (tema del POS nuevo) |
| El flujo (toggle → modal → panel) | La llamada al backend (contrato 15, no escritura directa) |
| El cálculo de `earliestReady` | El uso de `Number()` para el dinero (DT-02) |
| La validación de campos | Los tests (gate propio) |

---

## §5. SUB-FASES (de adentro hacia afuera)

### §5.0 — El router de pedidos (API) + gate

**Objetivo:** implementar los contratos 15 y 16 en `apps/api/routers/orders.py`.

**Entregables:**
- `routers/orders.py` con:
  - `POST /orders/from-ticket` (contrato 15) — idempotente por `ticket_id`; mapea `OPEN → TENTATIVO`,
    `PAID → PAGADO`; calcula `earliest_ready_at`; 404 si el ticket no existe; 409 si no es PEDIDO.
  - `GET /orders/by-ticket/{ticket_id}` (contrato 16) — proyección, no la fila completa; 404 si no hay.
- Registrar el router en `main.py`.
- Schemas Pydantic en `schemas.py` (entrada/salida de ambos contratos).
- **Gate:** `tests/test_f7_5_pedidos.py` con criterios:
  1. Los contratos 15 y 16 están declarados en el registro.
  2. `POST /orders/from-ticket` crea el pedido y devuelve `order_id`, `status`, `earliest_ready_at`.
  3. Es **idempotente**: dos llamadas con el mismo `ticket_id` no duplican (actualizan).
  4. `OPEN → TENTATIVO` y `PAID → PAGADO` (mapeo del proveedor).
  5. Un ticket que no es PEDIDO responde **409**.
  6. Un `ticket_id` inexistente responde **404**.
  7. `GET /orders/by-ticket/{ticket_id}` devuelve la proyección con los campos del contrato 16.
  8. `delivery_fee` viaja como **STRING** en el cable (DT-02, regla derivada 7).
  9. Un ticket sin pedido responde **404**.

**Por qué primero:** es la pieza con su puerta. Sin ella, el frontend no tiene con qué hablar.

### §5.1 — El servicio de pedidos (frontend)

**Objetivo:** `services/ordersService.js` — capa fina sobre los contratos 15/16.

**Entregables:**
- `registrarPedidoDesdeTicket(cuerpo)` → `POST /orders/from-ticket`.
- `obtenerPedidoDelTicket(ticketId)` → `GET /orders/by-ticket/{ticket_id}`.
- Coerción de ids a texto (lección de F7.7c: `aIdTexto()`).
- **Gate:** `ordersService.f7_5.test.jsx` — verifica las rutas, los cuerpos y el manejo de errores.

### §5.2 — El hook `useOrderProgramming`

**Objetivo:** el estado del pedido programado + el cálculo local de `earliest_ready_at`.

**Entregables:**
- `hooks/useOrderProgramming.js`:
  - `orderType` (`VENTA_DIRECTA` | `PEDIDO`) + `setOrderType`.
  - `orderData` + `guardarPedido(datos)` / `limpiarPedido()`.
  - `calcularEarliestReady(lineas)` — espejo local del cálculo del proveedor (para mostrar antes de guardar).
  - `registrarPedido(ticketId, statusTicket)` — llama al servicio; best-effort (no bloquea la venta).
- **Gate:** `useOrderProgramming.f7_5.test.jsx` — criterios:
  1. `orderType` arranca en `VENTA_DIRECTA`.
  2. `calcularEarliestReady` devuelve la fecha correcta según el lead time máximo.
  3. `registrarPedido` llama al servicio con el `ticket_id` y el `status_ticket` correctos.
  4. Si el servicio falla, **no lanza** (best-effort) y la venta continúa.
  5. `limpiarPedido` resetea el estado.

### §5.3 — El modal heredado

**Objetivo:** `components/ProgramacionPedidoModal.jsx` con la UX del POS viejo, reescrito.

**Entregables:**
- El modal con: toggle PICKUP/DOMICILIO, banner de "listo a partir de", input `committed_at`,
  campos de cliente, toggle de empaque (PROPIO/VENTA), dirección, notas.
- Estilo con el tema del POS nuevo (no el del viejo).
- Dinero con `Number()` (DT-02).
- **Gate:** `ProgramacionPedidoModal.f7_5.test.jsx` — criterios:
  1. Renderiza los campos heredados.
  2. El toggle PICKUP/DOMICILIO cambia la visibilidad de la dirección.
  3. `onSave` recibe el payload completo.
  4. El botón de guardar está deshabilitado si faltan campos obligatorios.
  5. Los targets táctiles son ≥44px (R-04).

### §5.4 — El cableado en la pantalla viva

**Objetivo:** conectar todo en `RetailVisionPOS.jsx` + `POSHeader.jsx`.

**Entregables:**
- `POSHeader.jsx`: añadir `orderType`, `orderData`, `onOrderTypeChange`, `onAbrirProgramacion`.
  Botón "PEDIDO" + botón "Programación del Pedido" (visible solo en modo PEDIDO).
- `RetailVisionPOS.jsx`: instanciar `useOrderProgramming`; pasar los props al header; renderizar
  el modal; al cobrar, si `orderType === 'PEDIDO'`, llamar `registrarPedido(ticketId, status)`.
- `CheckoutScreen.jsx`: panel de detalle del pedido cuando `orderData` existe.
- **Gate:** `RetailVisionPOS.f7_5.test.jsx` — criterios:
  1. El botón "PEDIDO" cambia el `orderType`.
  2. El botón "Programación del Pedido" solo aparece en modo PEDIDO.
  3. Al guardar el modal, `orderData` se puebla.
  4. Al cobrar un ticket PEDIDO, se llama `registrarPedido`.
  5. Al cobrar un ticket VENTA_DIRECTA, **no** se llama `registrarPedido` (RN-59).
  6. Si el registro del pedido falla, la venta se completa igual.

### §5.5 — Cierre: ficha + CI + documentación

**Entregables:**
- `FICHA_F7_5_PEDIDOS.md` con la evidencia.
- CI completo verde (lint + Node + Vitest + pytest + guards).
- Actualizar `PLAN_DE_ABORDAJE_FASE_7_POR_PARTES.md` §12.3 (marcar lo diferido como resuelto).
- Actualizar el Plan Maestro (Fase 7.5 en la lista de fases).
- Commit + push.

---

## §6. TRAZABILIDAD REGLA → SUB-FASE

| Regla / Directriz | Sub-fase que la cumple |
|---|---|
| **A-02 / P-01** (frontera por contratos) | F7.5.0 (el POS consume contratos, no escribe `orders`) |
| **RN-55** (el pedido se deriva del ticket) | F7.5.0 (mapeo `OPEN → TENTATIVO`, `PAID → PAGADO`) |
| **RN-56** (PICKUP por defecto) | F7.5.3 (el toggle arranca en PICKUP) |
| **RN-57** (PROPIO por defecto) | F7.5.3 (el toggle de empaque arranca en PROPIO) |
| **RN-58** (proyección idempotente) | F7.5.0 (idempotencia por `ticket_id`) |
| **RN-59** (VENTA_DIRECTA no genera pedido) | F7.5.4 (criterio 5 del gate) |
| **RN-67..RN-70** (el puente del pedido) | F7.5.0 + F7.5.4 |
| **DT-02** (dinero STRING en el cable) | F7.5.0 (criterio 8) + F7.5.3 |
| **DT-07** (degradación: la venta nunca se bloquea) | F7.5.2 (criterio 4) + F7.5.4 (criterio 6) |
| **§6.8** (UX heredada del viejo POS) | F7.5.3 + F7.5.4 |
| **R-04** (target ≥44px) | F7.5.3 (criterio 5) |
| **A-01** (regla con su test) | Todas las sub-fases |

---

## §7. RIESGOS Y MITIGACIONES

| Riesgo | Probabilidad | Impacto | Mitigación |
|---|---|---|---|
| El router de pedidos duplica lógica que luego tendrá el módulo Pedidos | Media | Bajo | El contrato es la frontera: cuando el módulo real exista, se cambia el proveedor, no el POS |
| El cálculo de `earliest_ready_at` diverge entre POS y proveedor | Media | Medio | El POS solo lo **muestra** antes de guardar; el valor autoritativo lo devuelve el proveedor |
| El modal heredado arrastra el estilo del POS viejo | Baja | Bajo | Se reescribe con el tema del POS nuevo (§4.3) |
| El registro del pedido bloquea la venta si falla | Baja | Alto | Best-effort explícito (DT-07); el gate lo verifica (criterio 6) |
| `delivery_fee` se envía como número y rompe el contrato | Media | Medio | El gate lo verifica (criterio 8); se envía como STRING |

---

## §8. DEFINICIÓN DE TERMINADO (DoD)

La Fase 7.5 está terminada cuando:

1. ✅ Los contratos 15 y 16 están **implementados** (no solo declarados).
2. ✅ El POS **nunca** importa `Order` ni escribe `orders` (verificado por el guard de frontera).
3. ✅ El operador puede marcar un ticket como PEDIDO, programarlo y cobrarlo.
4. ✅ Al cobrar un PEDIDO, el pedido se registra (best-effort).
5. ✅ Al cobrar una VENTA_DIRECTA, **no** se registra pedido.
6. ✅ Si el registro falla, la venta se completa igual.
7. ✅ Todos los gates verdes; CI completo verde.
8. ✅ La ficha documenta la evidencia.
9. ✅ El plan de Fase 7 §12.3 marca lo diferido como resuelto.

---

## §9. LO QUE ESTA FASE **NO** RESUELVE (y hay que decirlo)

1. **El módulo Pedidos/Producción no existe todavía.** Esta fase implementa el **contrato**, no el
   módulo. El pedido se guarda en la tabla `orders`; la máquina de 14 estados y el KDS de producción
   son de otra fase.
2. **El reparto no funciona.** `delivery_fee` queda en 0; no hay geocodificación ni rutas.
3. **El cliente no recibe notificación.** Eso es la Fase 8 (CRM + Notificaciones).

> **Esto es intencional y correcto.** El principio "de adentro hacia afuera" dice: se construye la
> pieza con su puerta, y cuando los módulos vecinos existan, **el POS no se toca**. La Fase 7.5
> cierra la brecha funcional (el POS vuelve a procesar pedidos) sin inventar módulos que no tocan.

---

## §10. POR QUÉ ESTA FASE ES LA CORRECTA AHORA

1. **Cierra una regresión funcional real:** el POS viejo procesaba pedidos; el nuevo no. Eso es una
   pérdida de capacidad, no una simplificación.
2. **El cimiento ya está:** `models/orders.py`, `Ticket.order_type`, la tabla `orders` y los
   contratos 15/16 declarados. Solo falta el puente.
3. **No depende de módulos inexistentes:** el router es autocontenido.
4. **Respeta la frontera:** el POS consume contratos, no escribe tablas ajenas.
5. **Prepara la Fase 8:** cuando llegue CRM + Notificaciones, el pedido ya estará registrado y
   tendrá a quién notificar.

---

**FIN DEL PLAN — FASE 7.5: PEDIDOS PROGRAMADOS**
