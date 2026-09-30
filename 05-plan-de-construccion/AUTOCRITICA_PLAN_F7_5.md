# 🔍 AUTOCRÍTICA DEL PLAN F7.5 — PEDIDOS PROGRAMADOS

**Versión:** 1.0
**Fecha:** 30 Sep 2026
**Objeto criticado:** `PLAN_DE_ABORDAJE_FASE_7_5_PEDIDOS.md` v1.0 (commit `2a7c6ac`)
**Método:** verificación de cada supuesto del plan contra el código real (no contra mi memoria)

---

## §0. VEREDICTO GENERAL

El plan v1.0 tiene **la dirección correcta** (construir el puente POS → Pedidos) pero
**3 defectos de fondo** que, si no se corrigen, producirían trabajo duplicado y una violación
de la frontera que el propio plan dice defender.

| # | Defecto | Severidad | Consecuencia si no se corrige |
|---|---|---|---|
| **D-1** | El plan **ignora que `Ticket` ya tiene los 9 campos del pedido** | **ALTA** | Se construiría un router de pedidos que duplica datos que el ticket ya guarda |
| **D-2** | El plan **no define quién escribe los campos del pedido en el ticket** | **ALTA** | El modal capturaría datos que nunca llegan a la BD |
| **D-3** | El plan **asume que el POS debe llamar a Pedidos al cobrar**, pero el contrato 15 dice "en la MISMA transacción del guardado del ticket" | **MEDIA** | Se diseñaría una llamada HTTP donde el contrato pide atomicidad |

---

## §1. D-1 — EL PLAN IGNORÓ QUE EL TICKET YA TIENE LOS CAMPOS DEL PEDIDO

### §1.1 La evidencia (verificada en el código)

[`models/pos.py`](../NUEVO-POS/apps/api/models/pos.py:75) — la clase `Ticket` ya declara:

```python
order_type: Mapped[str] = mapped_column(String, nullable=False, default="VENTA_DIRECTA")
order_status: Mapped[str] = mapped_column(String, nullable=False, default="PROGRAMADO PARA SER PREPARADO")
delivery_type: Mapped[str | None] = mapped_column(String, nullable=True)
customer_name: Mapped[str | None] = mapped_column(String, nullable=True)
customer_phone: Mapped[str | None] = mapped_column(String, nullable=True)
committed_at: Mapped[object | None] = mapped_column(DateTime(timezone=True), nullable=True)
packaging_type: Mapped[str | None] = mapped_column(String, nullable=True)
delivery_address: Mapped[str | None] = mapped_column(Text, nullable=True)
order_notes: Mapped[str | None] = mapped_column(Text, nullable=True)
```

Y la migración [`0001_initial_schema.py`](../NUEVO-POS/apps/api/migrations/versions/0001_initial_schema.py:120) los creó.

### §1.2 Por qué esto invalida parte del plan v1.0

Mi plan v1.0 dijo: *"el cimiento ya está: `models/orders.py`, `Ticket.order_type`, la tabla `orders`"*.
Eso fue **incompleto y engañoso**. La verdad es más fuerte:

> **El POS ya tiene el modelo de datos COMPLETO del pedido, dentro del propio ticket.**

Esto significa que hay **dos lugares** donde vive el pedido:
1. **`tickets`** — los 9 campos desnormalizados (lo que el POS captura y muestra).
2. **`orders`** — la tabla normalizada (lo que Pedidos/Producción gobierna).

### §1.3 La pregunta que el plan v1.0 no se hizo

**¿Por qué existen los dos?** La respuesta está en la arquitectura:

- **`tickets.order_*`** = la **copia de trabajo del POS**. El POS necesita mostrar el pedido
  mientras lo captura, sin depender de que el módulo Pedidos exista.
- **`orders`** = la **proyección gobernada por Pedidos**. Es la que tiene el ciclo de 14 estados,
  el `earliest_ready_at` autoritativo y el `delivery_fee`.

**El contrato 15 es el puente entre ambos.** El POS escribe `tickets.order_*` (su copia) y
**proyecta** a `orders` vía el contrato.

### §1.4 La corrección

El plan debe decir explícitamente:

1. **El POS escribe `tickets.order_*`** — es su propia tabla, no viola la frontera.
2. **El POS proyecta a `orders` vía el contrato 15** — no escribe `orders` directamente.
3. **`POST /pos/tickets` debe aceptar los campos del pedido** en su entrada (hoy no lo hace:
   el `CrearTicketEntrada` no los tiene). Esto es un **entregable que el plan v1.0 omitió**.

---

## §2. D-2 — EL PLAN NO DEFINIÓ QUIÉN ESCRIBE LOS CAMPOS DEL PEDIDO

### §2.1 El hueco

Mi plan v1.0 describió el modal (F7.5.3) y el cableado (F7.5.4), pero **nunca dijo cómo los
datos del modal llegan a `tickets.order_*`**.

El flujo real que falta definir:

```
Modal captura datos
   → useOrderProgramming.guardarPedido(datos)
   → ??? (¿quién los persiste?)
   → tickets.order_* (la copia del POS)
   → contrato 15 → orders (la proyección)
```

### §2.2 Las dos opciones (y cuál es correcta)

| Opción | Cómo | Veredicto |
|---|---|---|
| **A. Al crear el ticket** | `POST /pos/tickets` recibe los campos del pedido y los guarda | **CORRECTA** — una sola escritura, atómica |
| **B. Endpoint aparte** | `PATCH /pos/tickets/{id}/order` | **INCORRECTA** — dos escrituras, no atómica, y el contrato 15 pide atomicidad |

**La opción A es la correcta**, y el plan v1.0 no la eligió porque no se hizo la pregunta.

### §2.3 La corrección

El plan debe añadir un entregable explícito:

> **F7.5.0b — Extender `POST /pos/tickets`** para aceptar los campos del pedido
> (`order_type`, `delivery_type`, `customer_name`, `customer_phone`, `committed_at`,
> `packaging_type`, `delivery_address`, `order_notes`) y persistirlos en `tickets.order_*`.

Sin esto, el modal captura datos que se pierden.

---

## §3. D-3 — EL PLAN CONFUNDIÓ "PROYECTAR" CON "LLAMAR POR HTTP"

### §3.1 La contradicción

El contrato 15 dice, en sus garantías ([`registry.py`](../NUEVO-POS/apps/api/contracts/registry.py:374)):

> *"Se ejecuta en la MISMA transacción del guardado del ticket."*

Pero mi plan v1.0 (F7.5.4) dijo:

> *"al cobrar, si `orderType === 'PEDIDO'`, llamar `registrarPedido(ticketId, status)`"*

**Eso es una llamada HTTP desde el frontend**, que **no puede** ser atómica con el guardado del
ticket. El plan se contradijo a sí mismo.

### §3.2 La resolución correcta

La atomicidad solo se logra **dentro del backend**. Entonces:

- **El frontend NO llama al contrato 15.** El frontend solo envía los campos del pedido en
  `POST /pos/tickets`.
- **El backend, dentro de la misma transacción**, escribe `tickets.order_*` **y** proyecta a
  `orders` (contrato 15 implementado como **función interna**, no como llamada HTTP).

### §3.3 La corrección

El plan debe reescribir F7.5.4:

> El frontend **no** llama a `/orders/from-ticket`. El backend, en `POST /pos/tickets` y en
> `POST /pos/tickets/{id}/pay`, proyecta el pedido a `orders` **en la misma transacción**.

Y el contrato 15 se implementa como **servicio interno** (`services/pedidos.py`), no como
endpoint HTTP que el POS consuma. El endpoint HTTP existe para **otros consumidores** (el módulo
Pedidos, el CRM), no para el POS.

---

## §4. LO QUE EL PLAN v1.0 HIZO BIEN (para no tirarlo todo)

| Acierto | Por qué se conserva |
|---|---|
| **La distinción plan vs ejecución** | Correcta y verificada: el plan documentó, la ejecución omitió |
| **La regla de frontera (A-02/P-01)** | Correcta: el POS no importa `Order` |
| **El estado del pedido no lo gobierna el POS** | Correcto: los 14 estados son de Pedidos |
| **DT-07 (la venta nunca se bloquea)** | Correcto y crítico |
| **La UX heredada del POS viejo (§6.8)** | Correcto: el modal se hereda en integración |
| **La trazabilidad regla → sub-fase** | Correcta y útil |
| **La definición de terminado** | Correcta, salvo que le faltan los criterios de D-1/D-2 |

---

## §5. EL PLAN CORREGIDO (v2.0) — CAMBIOS CONCRETOS

### §5.1 Sub-fases corregidas

| Sub-fase | v1.0 | v2.0 (corregida) |
|---|---|---|
| **F7.5.0** | Router de pedidos (contratos 15/16 como endpoints) | **Servicio interno `services/pedidos.py`** (proyección atómica) + **endpoints 15/16** para consumidores externos |
| **F7.5.0b** | *(no existía)* | **Extender `POST /pos/tickets`** para aceptar y persistir los campos del pedido |
| **F7.5.1** | `ordersService.js` (llama a los contratos) | **Se elimina** — el frontend no llama a los contratos; solo envía campos en `POST /pos/tickets` |
| **F7.5.2** | `useOrderProgramming` (registra vía servicio) | `useOrderProgramming` **solo gestiona estado local**; no llama a Pedidos |
| **F7.5.3** | Modal heredado | Igual (sin cambios) |
| **F7.5.4** | Cableado: al cobrar, llamar `registrarPedido` | Cableado: **enviar los campos del pedido en `POST /pos/tickets`**; el backend proyecta |
| **F7.5.5** | Cierre | Igual + verificar que `tickets.order_*` se puebla |

### §5.2 Criterios de gate corregidos

**Gate de API (F7.5.0) — añadir:**
- `POST /pos/tickets` con `order_type=PEDIDO` **persiste los 9 campos** en `tickets.order_*`.
- La proyección a `orders` ocurre **en la misma transacción** (si falla, el ticket no se crea).
- `POST /pos/tickets` con `order_type=VENTA_DIRECTA` **no** crea fila en `orders` (RN-59).

**Gate de frontend (F7.5.4) — corregir:**
- ~~"Al cobrar un ticket PEDIDO, se llama `registrarPedido`"~~ → **"Al crear un ticket PEDIDO,
  el cuerpo de `POST /pos/tickets` incluye los campos del pedido"**.

### §5.3 La pregunta que el plan v2.0 debe responder al dueño

> **¿El POS debe poder crear un pedido SIN cobrar?** (es decir, un pedido programado que se
> paga después). El POS viejo sí lo permite. Si la respuesta es sí, el contrato 15 debe poder
> ejecutarse en `POST /pos/tickets` (ticket OPEN → `TENTATIVO`) **y** en `pay` (PAID → `PAGADO`).

Esto es una **decisión de alcance** que el plan v1.0 no planteó y que cambia el diseño.

---

## §6. LECCIONES DE ESTA AUTOCRÍTICA

1. **Verificar el modelo de datos ANTES de planear el router.** Asumí que el cimiento era
   `models/orders.py`; el cimiento real es `Ticket.order_*`, que ya existía.
2. **Preguntar "¿quién escribe esto?" para cada dato que se captura.** El plan v1.0 describió
   la captura (el modal) sin describir la persistencia.
3. **Leer las garantías del contrato, no solo su firma.** El contrato 15 pide atomicidad; mi
   plan propuso una llamada HTTP. La garantía estaba escrita y no la leí con cuidado.
4. **Una autocrítica sin verificación es teatro.** Los 3 defectos se encontraron **leyendo el
   código**, no releyendo el plan.

---

## §7. RECOMENDACIÓN

**No ejecutar el plan v1.0.** Reescribirlo como v2.0 con:
1. El servicio interno de proyección (no endpoints que el POS consuma).
2. La extensión de `POST /pos/tickets` con los campos del pedido.
3. La eliminación de `ordersService.js` del frontend.
4. La pregunta de alcance al dueño (§5.3).

El trabajo de F7.5.3 (el modal) y F7.5.5 (cierre) se conserva casi intacto.

---

**FIN DE LA AUTOCRÍTICA — F7.5**
