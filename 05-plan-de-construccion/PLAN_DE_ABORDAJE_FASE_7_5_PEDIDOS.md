# 📋 PLAN DE ABORDAJE — FASE 7.5: PEDIDOS PROGRAMADOS (el puente POS → Pedidos → Producción)

**Versión:** 3.0 (política de pago configurable desde Vista General — DT-06)
**Fecha:** 30 Sep 2026
**Autor:** Arquitecto del Nuevo POS
**Estado:** Propuesta — pendiente de aprobación del dueño
**Precedente:** `PLAN_DE_ABORDAJE_FASE_7_POR_PARTES.md` §12.3 (lo diferido)
**Cambios vs v2.0:** ver §12 (registro de cambios)

---

## §0. LA POLÍTICA DE PAGO DE PEDIDOS — DECLARADA EN VISTA GENERAL (DT-06)

### §0.1 La decisión del dueño (30 Sep 2026)

> *"Vamos a definir las políticas del negocio desde el módulo que se creará llamado **Vista General**.
> Desde ahí se establecerá si se puede procesar sin pagar ni dejar anticipo, si se procesa con
> anticipo y de qué porcentaje, y si solo se procesa si está cubierto al 100%. El POS deberá acatar
> esta política en el procesamiento de pedidos."*

**Esta decisión es arquitectónicamente superior a la v2.0** (que fijaba "no ticket no food" como una
constante del POS). La v2.0 **hardcodeaba** una política de negocio dentro del POS — exactamente lo
que **DT-06 prohíbe**:

> **DT-06.1:** *"Los valores que afectan a todos los módulos se declaran una sola vez, en el módulo
> Vista General, y se persisten en `system_settings`. Ningún módulo los define por su cuenta."*

La política de pago de pedidos **es** un valor transversal: afecta al POS, a Pedidos, a Producción y
a Caja. Por lo tanto **no puede vivir en el POS**. Vive en Vista General.

### §0.2 La política declarada como valor transversal (4.º valor de DT-06)

Se declara un **cuarto valor transversal** en DT-06, junto a zona horaria, moneda y sucursal:

| Clave en `system_settings` | Tipo | Valores posibles | Default seguro |
|---|---|---|---|
| `orders.payment_policy` | enum | `SIN_PAGO` \| `ANTICIPO` \| `PAGO_COMPLETO` | **`PAGO_COMPLETO`** |
| `orders.deposit_percent` | entero (0–100) | Solo aplica si `payment_policy = ANTICIPO` | `50` |

**Los 3 modos:**

| Modo | Significado | Qué hace el POS |
|---|---|---|
| **`SIN_PAGO`** | Se puede procesar el pedido sin pagar ni dejar anticipo | Proyecta el pedido a `orders` al **crear** el ticket (estado `TENTATIVO`) |
| **`ANTICIPO`** | Se procesa con un anticipo de `deposit_percent` % | Proyecta cuando el pago acumulado ≥ `deposit_percent` % del total |
| **`PAGO_COMPLETO`** | Solo se procesa si está cubierto al 100% | Proyecta **solo** cuando el ticket pasa a `PAID` |

### §0.3 El default seguro y la degradación (DT-07)

> **Regla dura:** si el POS **no puede leer** la política (Vista General no existe todavía, o el
> endpoint falla), **asume `PAGO_COMPLETO`** — el modo más restrictivo — y **nunca bloquea la venta**.

Esto es **DT-07 aplicado a la configuración**: la ausencia de configuración degrada al
comportamiento más conservador, no al más permisivo. Un POS que no sabe la política **no puede**
regalar comida.

**Consecuencia inmediata:** la Fase 7.5 funciona **hoy**, sin Vista General, con el default
`PAGO_COMPLETO` (que es exactamente la política "no ticket no food" de la v2.0). Cuando Vista General
exista, **el POS no se toca**: empieza a leer la política real.

### §0.4 Por qué esto NO retrasa la Fase 7.5

| Pregunta | Respuesta |
|---|---|
| ¿Necesito Vista General para construir F7.5? | **NO.** El POS lee la política con un default seguro. |
| ¿Necesito el endpoint `GET /settings`? | **NO.** Se lee con fallback; si no existe, usa `PAGO_COMPLETO`. |
| ¿Cambia el trabajo del POS cuando Vista General exista? | **NO.** El POS ya lee la política; solo cambia el valor que recibe. |

---

## §1. EL HALLAZGO QUE ORIGINA ESTA FASE

### §1.1 Lo que el dueño detectó

> *"He detectado una omisión del plan: el POS viejo tiene la funcionalidad de procesar pedidos y
> enviarlos a producción y creo que en este nuevo POS esto no se contempló."*

### §1.2 El diagnóstico: NO es una omisión del PLAN, es una omisión de la EJECUCIÓN

| Dimensión | Veredicto | Evidencia |
|---|---|---|
| **¿El plan lo documentó?** | **SÍ, en 4 lugares** | Ver §1.3 |
| **¿El código tiene el cimiento?** | **SÍ, y más completo de lo que parecía** | Ver §1.4 |
| **¿El código tiene el puente?** | **NO** | No existe proyección a `orders`; los contratos 15/16 están declarados pero sin implementar |
| **¿Es una regresión funcional?** | **SÍ** | El POS viejo SÍ procesa pedidos y los manda a producción |

### §1.3 Los 4 lugares donde el plan ya lo documentó

1. **`CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md` §8** — contratos **15** y **16**.
2. **`PLANO ARQUITECTONICO PARA EL NUEVO POS.md` §1.12** — reglas **RN-55..RN-59**.
3. **`ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md`** — **F-19**, **RN-67..RN-70**, **AC-02**.
4. **`MODELO_DE_DATOS_DEL_NUEVO_POS.md` §4.1** — la tabla `orders`.

### §1.4 El cimiento real (corregido tras la autocrítica)

**La v1.0 de este plan dijo que el cimiento era `models/orders.py`. Era incompleto.**

El cimiento real son **DOS** estructuras, y la primera ya está completa:

| Estructura | Qué es | Estado |
|---|---|---|
| **`tickets.order_*`** (9 campos) | La **copia de trabajo del POS**. El POS la captura y la muestra. | **YA EXISTE** ([`models/pos.py`](../NUEVO-POS/apps/api/models/pos.py:75)) |
| **`orders`** | La **proyección gobernada por Pedidos**. Tiene el ciclo de 14 estados y el `earliest_ready_at` autoritativo. | **YA EXISTE** ([`models/orders.py`](../NUEVO-POS/apps/api/models/orders.py:21)) |

**El contrato 15 es el puente entre ambas.** El POS escribe `tickets.order_*` (su propia tabla,
sin violar la frontera) y **proyecta** a `orders` vía el contrato.

---

## §2. LA REGLA DURA QUE GOBIERNA ESTA FASE

### §2.1 P-01 / A-02 — Frontera por contratos

El POS viejo **viola** la frontera: importa el modelo `Order` y escribe la tabla `orders`
directamente. El POS nuevo **no puede** hacer eso.

> **El POS nuevo escribe `tickets.order_*` (su tabla) y proyecta a `orders` vía el contrato 15.
> Nunca importa `Order`. Nunca escribe `orders` directamente.**

### §2.2 El estado del pedido NO lo gobierna el POS

Los **14 estados** del ciclo de vida del pedido los gobierna **Pedidos/Producción**. El POS solo
**proyecta** el estado del ticket al pedido. El **cuándo** proyecta lo dicta la política (§0).

### §2.3 La regla de oro: la venta nunca se bloquea (DT-07)

Si la proyección a `orders` falla, **la venta se completa igual**. Pero — y aquí está el matiz
crítico — la proyección ocurre **dentro de la misma transacción** del cobro. Si la proyección
falla, se registra el error y se reconcilia después, **sin deshacer el cobro**.

> **Matiz:** la política de pago decide **cuándo** se proyecta (al crear, al alcanzar el anticipo, o
> al pagar completo), **no** si la venta se bloquea. El pago manda; la proyección es best-effort
> dentro de la transacción.

### §2.4 La política se lee, no se escribe (DT-06)

> **El POS es un CONSUMIDOR de la política, nunca su dueño.** No la define, no la persiste, no la
> sobrescribe. La lee de `system_settings` (vía Vista General) y la acata.

---

## §3. ALCANCE DEFINIDO

### §3.1 Lo que SÍ entra en F7.5a (ejecutable hoy)

| # | Entregable | Descripción |
|---|---|---|
| 1 | **Extender `POST /pos/tickets`** | Acepta y persiste los 9 campos del pedido en `tickets.order_*` |
| 2 | **Lector de política con default seguro** | `services/politica_pedidos.py` — lee `orders.payment_policy` de `system_settings`; si falta, `PAGO_COMPLETO` |
| 3 | **Servicio interno de proyección** | `services/pedidos.py` — proyecta `tickets.order_*` → `orders` (contrato 15), **en la misma transacción**, **cuando la política lo permite** |
| 4 | **Endpoints 15/16** | `POST /orders/from-ticket` y `GET /orders/by-ticket/{ticket_id}` — para **consumidores externos** (Pedidos, CRM), no para el POS |
| 5 | **Hook `useOrderProgramming`** | Estado local del pedido (tipo, datos, cálculo de `earliest_ready_at` para mostrar) |
| 6 | **Modal heredado** | `ProgramacionPedidoModal.jsx` — **UX heredada del POS viejo** (§6.8 del Plan Maestro) |
| 7 | **Botón "PEDIDO" en el header** | `POSHeader.jsx` — el toggle VENTA_DIRECTA ↔ PEDIDO |
| 8 | **Panel de pedido en el checkout** | Muestra el pedido programado antes de cobrar |
| 9 | **Gates** | API + frontend, con los criterios corregidos (§5) |

### §3.2 Lo que NO entra en F7.5a (explícito)

| # | Fuera de alcance | Por qué |
|---|---|---|
| 1 | **La máquina de 14 estados** | Vive en el módulo Pedidos/Producción |
| 2 | **La pantalla de Producción (KDS)** | Es del módulo Producción |
| 3 | **El reparto (rutas, choferes, mapas)** | Es del módulo Reparto |
| 4 | **La notificación al cliente** | Es la Fase 8 (CRM + Notificaciones) |
| 5 | **El cálculo de `delivery_fee` por distancia** | Requiere geocodificación; queda en 0 |
| 6 | **Pedidos creados por otros canales** | El POS solo proyecta lo suyo; otros canales son de Pedidos |
| 7 | **La UI de Vista General para editar la política** | Es de Vista General (F7.5b) |

> **Nota crítica:** los entregables 1–9 funcionan **hoy**, sin depender de módulos inexistentes.
> El servicio de proyección es autocontenido: escribe la tabla `orders` que ya existe. Cuando el
> módulo Pedidos real se construya, **solo cambia quién implementa el contrato** — el POS no se toca.

### §3.3 F7.5b — lo diferido (cuando Vista General exista)

| # | Entregable | Cuándo |
|---|---|---|
| 1 | **UI de Vista General** para editar `orders.payment_policy` y `orders.deposit_percent` | Cuando se reconstruya Vista General |
| 2 | **`GET /settings`** que exponga la política | Con Vista General |
| 3 | **El POS lee la política real** | Automático: el POS ya la lee; solo cambia el valor |

> **El POS no se toca en F7.5b.** Esa es la prueba de que la frontera está bien trazada.

---

## §4. LA UX HEREDADA DEL POS VIEJO (§6.8 del Plan Maestro)

> *"Cuando un componente ya existe en el viejo POS, su INTEGRACIÓN se hereda; solo su
> IMPLEMENTACIÓN se reescribe."*

### §4.1 Cómo funciona en el POS viejo (verificado en el código)

| Pieza | Comportamiento heredado |
|---|---|
| **`POSHeader`** | Botón **"PEDIDO"** que llama `onOrderTypeChange('PEDIDO')`. En modo PEDIDO aparece **"Programación del Pedido"** |
| **`ProgramacionPedidoModal`** | Captura: tipo de entrega (PICKUP/DOMICILIO), `committed_at`, nombre/teléfono, tipo de empaque (PROPIO/VENTA), dirección, notas. Calcula `earliestReady` con `calcMaxLeadTime(cart)` |
| **`CheckoutScreen`** | Panel de detalle del pedido cuando `orderData` existe |
| **Estado** | `showProgramacion` + `orderData` en el componente raíz |

### §4.2 La diferencia con el POS nuevo

El [`POSHeader.jsx`](../NUEVO-POS/apps/pos/src/components/POSHeader.jsx:47) del POS nuevo **no
tiene** los props `orderType` / `orderData` / `onOrderTypeChange`. Hay que **añadirlos**.

### §4.3 El modal: qué se hereda y qué se reescribe

| Se HEREDA (la integración) | Se REESCRIBE (la implementación) |
|---|---|
| Los campos del formulario | El estilo (tema del POS nuevo) |
| El flujo (toggle → modal → panel) | La persistencia (campos en `POST /pos/tickets`, no escritura directa) |
| El cálculo de `earliestReady` | El uso de `Number()` para el dinero (DT-02) |
| La validación de campos | Los tests (gate propio) |

---

## §5. SUB-FASES (de adentro hacia afuera)

### §5.0 — Extender `POST /pos/tickets` con los campos del pedido (API) + gate

**Objetivo:** que el POS pueda **persistir** el pedido en su propia tabla.

**Entregables:**
- Extender `CrearTicketEntrada` en [`schemas.py`](../NUEVO-POS/apps/api/schemas.py:100) con los
  9 campos opcionales del pedido.
- Extender `crear_ticket` en [`pos.py`](../NUEVO-POS/apps/api/routers/pos.py:244) para persistirlos
  en `tickets.order_*`.
- **Gate:** criterios:
  1. `POST /pos/tickets` con `order_type=PEDIDO` **persiste los 9 campos**.
  2. `POST /pos/tickets` sin campos de pedido deja los defaults (`VENTA_DIRECTA`, etc.).
  3. Un `order_type` inválido es **422**.
  4. `delivery_type` inválido (no PICKUP/DOMICILIO) es **422**.
  5. `packaging_type` inválido (no PROPIO/VENTA) es **422**.

**Por qué primero:** sin esto, el modal captura datos que se pierden (D-2 de la autocrítica).

### §5.1 — Lector de política con default seguro (API) + gate

**Objetivo:** `services/politica_pedidos.py` — lee la política de `system_settings` (DT-06).

**Entregables:**
- `services/politica_pedidos.py` con `leer_politica(db) -> PoliticaPedidos`:
  - Lee `orders.payment_policy` y `orders.deposit_percent` de `system_settings`.
  - **Si falta o es inválida → `PAGO_COMPLETO` + `deposit_percent=100`** (default seguro, §0.3).
  - **Nunca lanza excepción**: la ausencia de configuración no puede romper el POS (DT-07).
- **Gate:** criterios:
  1. Sin fila en `system_settings` → devuelve `PAGO_COMPLETO`.
  2. Con `orders.payment_policy = SIN_PAGO` → devuelve `SIN_PAGO`.
  3. Con `orders.payment_policy = ANTICIPO` y `deposit_percent = 30` → devuelve `ANTICIPO` + `30`.
  4. Con un valor corrupto (`"BASURA"`) → devuelve `PAGO_COMPLETO` (no lanza).
  5. Con `deposit_percent` fuera de rango (150) → lo acota a 100.

**Por qué antes de la proyección:** la proyección **consulta** la política para decidir cuándo
proyectar. Sin el lector, la proyección no sabe cuándo actuar.

### §5.2 — Servicio interno de proyección (API) + gate

**Objetivo:** `services/pedidos.py` — proyecta `tickets.order_*` → `orders` (contrato 15).

**Entregables:**
- `services/pedidos.py` con `proyectar_pedido(db, ticket, politica)`:
  - Idempotente por `ticket_id` (si existe, actualiza).
  - Calcula `earliest_ready_at` (autoritativo).
  - **Decide el estado según la política** (§0.2):
    - `SIN_PAGO` → proyecta al crear, `TENTATIVO`.
    - `ANTICIPO` → proyecta cuando `pagado ≥ deposit_percent %`, `TENTATIVO`.
    - `PAGO_COMPLETO` → proyecta solo en `PAID`, `PAGADO`.
  - **Se ejecuta en la misma transacción** del guardado/cobro.
- Invocación desde `crear_ticket` y `cobrar_ticket` en [`pos.py`](../NUEVO-POS/apps/api/routers/pos.py:244).
- **Gate:** criterios:
  1. Con `PAGO_COMPLETO`: cobrar un ticket PEDIDO **crea la fila en `orders`** con `status=PAGADO`.
  2. Con `PAGO_COMPLETO`: cobrar un ticket VENTA_DIRECTA **no crea fila** en `orders` (RN-59).
  3. Con `SIN_PAGO`: crear un ticket PEDIDO **crea la fila** con `status=TENTATIVO`.
  4. Con `ANTICIPO` (30%): un pago del 20% **no** proyecta; uno del 30% **sí** proyecta.
  5. La proyección es **idempotente**: re-cobrar no duplica.
  6. `earliest_ready_at` se calcula y se persiste.
  7. Si la proyección falla, **el cobro se completa** (DT-07) y se registra el error.
  8. `delivery_fee` se persiste como `Numeric(12,2)` (no Float).

### §5.3 — Endpoints 15/16 para consumidores externos (API) + gate

**Objetivo:** exponer los contratos 15/16 por HTTP para **otros** consumidores.

**Entregables:**
- `routers/orders.py` con `POST /orders/from-ticket` y `GET /orders/by-ticket/{ticket_id}`.
- Registrar en [`main.py`](../NUEVO-POS/apps/api/main.py:1).
- **Gate:** criterios:
  1. Los contratos 15 y 16 están declarados.
  2. `POST /orders/from-ticket` responde con `order_id`, `status`, `earliest_ready_at`.
  3. Un ticket que no es PEDIDO responde **409**.
  4. Un `ticket_id` inexistente responde **404**.
  5. `GET /orders/by-ticket/{ticket_id}` devuelve la **proyección** (no la fila completa).
  6. `delivery_fee` viaja como **STRING** en el cable (DT-02, regla derivada 7).
  7. Un ticket sin pedido responde **404**.

### §5.4 — El hook `useOrderProgramming` (frontend)

**Objetivo:** el estado local del pedido. **No llama a Pedidos.**

**Entregables:**
- `hooks/useOrderProgramming.js`:
  - `orderType` (`VENTA_DIRECTA` | `PEDIDO`) + `setOrderType`.
  - `orderData` + `guardarPedido(datos)` / `limpiarPedido()`.
  - `calcularEarliestReady(lineas)` — espejo local para **mostrar** antes de cobrar.
  - `camposParaTicket()` — devuelve los 9 campos para incluirlos en `POST /pos/tickets`.
- **Gate:** criterios:
  1. `orderType` arranca en `VENTA_DIRECTA`.
  2. `calcularEarliestReady` devuelve la fecha correcta según el lead time máximo.
  3. `camposParaTicket()` devuelve los 9 campos con los defaults correctos (PICKUP, PROPIO).
  4. `limpiarPedido` resetea el estado.
  5. **No** hace ninguna llamada de red (verificado con un fetch espiado).

### §5.5 — El modal heredado (frontend)

**Objetivo:** `components/ProgramacionPedidoModal.jsx` con la UX del POS viejo, reescrito.

**Entregables:**
- El modal con: toggle PICKUP/DOMICILIO, banner de "listo a partir de", input `committed_at`,
  campos de cliente, toggle de empaque (PROPIO/VENTA), dirección, notas.
- Estilo con el tema del POS nuevo.
- Dinero con `Number()` (DT-02).
- **Gate:** criterios:
  1. Renderiza los campos heredados.
  2. El toggle PICKUP/DOMICILIO cambia la visibilidad de la dirección.
  3. `onSave` recibe el payload completo.
  4. El botón de guardar está deshabilitado si faltan campos obligatorios.
  5. Los targets táctiles son ≥44px (R-04).

### §5.6 — El cableado en la pantalla viva (frontend)

**Objetivo:** conectar todo en `RetailVisionPOS.jsx` + `POSHeader.jsx`.

**Entregables:**
- [`POSHeader.jsx`](../NUEVO-POS/apps/pos/src/components/POSHeader.jsx:47): añadir `orderType`,
  `orderData`, `onOrderTypeChange`, `onAbrirProgramacion`. Botón "PEDIDO" + botón "Programación
  del Pedido" (visible solo en modo PEDIDO).
- [`RetailVisionPOS.jsx`](../NUEVO-POS/apps/pos/src/RetailVisionPOS.jsx:79): instanciar
  `useOrderProgramming`; pasar los props al header; renderizar el modal; **al crear el ticket,
  incluir `camposParaTicket()` en el cuerpo de `POST /pos/tickets`**.
- [`CheckoutScreen.jsx`](../NUEVO-POS/apps/pos/src/components/CheckoutScreen.jsx:1): panel de
  detalle del pedido cuando `orderData` existe.
- **Gate:** criterios:
  1. El botón "PEDIDO" cambia el `orderType`.
  2. El botón "Programación del Pedido" solo aparece en modo PEDIDO.
  3. Al guardar el modal, `orderData` se puebla.
  4. Al crear un ticket PEDIDO, el cuerpo de `POST /pos/tickets` **incluye los 9 campos**.
  5. Al crear un ticket VENTA_DIRECTA, el cuerpo **no** incluye campos de pedido (RN-59).
  6. El frontend **no** llama a `/orders/from-ticket` (verificado con fetch espiado).
  7. El frontend **no** lee ni escribe `system_settings` (la política la aplica el backend).

### §5.7 — Cierre: ficha + CI + documentación

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
| **A-02 / P-01** (frontera por contratos) | F7.5.2 (proyección interna) + F7.5.6 (el frontend no llama a Pedidos) |
| **DT-06** (configuración en Vista General) | F7.5.1 (el POS **lee** la política, no la define) |
| **DT-06.3.2** (un solo lugar de declaración) | F7.5.1 (el POS no tiene selector de política) |
| **DT-07** (degradación: la venta nunca se bloquea) | F7.5.1 (default seguro) + F7.5.2 (criterio 7) |
| **Política `PAGO_COMPLETO`** (default seguro) | F7.5.1 (criterio 1) + F7.5.2 (criterios 1–2) |
| **Política `SIN_PAGO`** | F7.5.2 (criterio 3) |
| **Política `ANTICIPO`** | F7.5.2 (criterio 4) |
| **RN-55** (el pedido se deriva del ticket) | F7.5.2 (mapeo del estado) |
| **RN-56** (PICKUP por defecto) | F7.5.4 (default) + F7.5.5 (el toggle arranca en PICKUP) |
| **RN-57** (PROPIO por defecto) | F7.5.4 (default) + F7.5.5 (el toggle arranca en PROPIO) |
| **RN-58** (proyección idempotente) | F7.5.2 (criterio 5) |
| **RN-59** (VENTA_DIRECTA no genera pedido) | F7.5.2 (criterio 2) + F7.5.6 (criterio 5) |
| **RN-67..RN-70** (el puente del pedido) | F7.5.2 + F7.5.3 |
| **DT-02** (dinero STRING en el cable) | F7.5.3 (criterio 6) + F7.5.5 |
| **§6.8** (UX heredada del viejo POS) | F7.5.5 + F7.5.6 |
| **R-04** (target ≥44px) | F7.5.5 (criterio 5) |
| **A-01** (regla con su test) | Todas las sub-fases |

---

## §7. RIESGOS Y MITIGACIONES

| Riesgo | Probabilidad | Impacto | Mitigación |
|---|---|---|---|
| La proyección duplica lógica que luego tendrá el módulo Pedidos | Media | Bajo | El contrato es la frontera: cuando el módulo real exista, se cambia el proveedor |
| El cálculo de `earliest_ready_at` diverge entre POS y proveedor | Media | Medio | El POS solo lo **muestra**; el valor autoritativo lo devuelve el proveedor |
| El modal heredado arrastra el estilo del POS viejo | Baja | Bajo | Se reescribe con el tema del POS nuevo (§4.3) |
| La proyección bloquea el cobro si falla | Baja | Alto | Best-effort explícito (DT-07); el gate lo verifica (F7.5.2 criterio 7) |
| `delivery_fee` se envía como número y rompe el contrato | Media | Medio | El gate lo verifica (F7.5.3 criterio 6) |
| El frontend llama a Pedidos por error (viola la frontera) | Media | Alto | El gate lo verifica (F7.5.6 criterio 6) |
| **El POS hardcodea la política de pago** | Media | **Alto** | **DT-06: el POS la lee; el gate verifica que no la define (F7.5.6 criterio 7)** |
| **La política corrupta rompe el POS** | Baja | **Alto** | **Default seguro `PAGO_COMPLETO`; el gate lo verifica (F7.5.1 criterios 1 y 4)** |

---

## §8. DEFINICIÓN DE TERMINADO (DoD)

La Fase 7.5a está terminada cuando:

1. ✅ `POST /pos/tickets` **persiste** los 9 campos del pedido en `tickets.order_*`.
2. ✅ El POS **lee** la política de `system_settings`; si falta, asume `PAGO_COMPLETO`.
3. ✅ Con `PAGO_COMPLETO`: cobrar un ticket PEDIDO **proyecta** a `orders` con `status=PAGADO`, en la misma transacción.
4. ✅ Con `SIN_PAGO` y `ANTICIPO`: la proyección respeta el modo (gate F7.5.2 criterios 3–4).
5. ✅ Cobrar un ticket VENTA_DIRECTA **no** proyecta (RN-59).
6. ✅ El POS **nunca** importa `Order` ni escribe `orders` (verificado por el guard de frontera).
7. ✅ El POS **nunca** define ni persiste la política (verificado por el gate F7.5.6 criterio 7).
8. ✅ El frontend **nunca** llama a `/orders/from-ticket` (verificado por el gate).
9. ✅ Si la proyección falla, la venta se completa igual (DT-07).
10. ✅ El operador puede marcar un ticket como PEDIDO, programarlo y cobrarlo.
11. ✅ Todos los gates verdes; CI completo verde.
12. ✅ La ficha documenta la evidencia.
13. ✅ El plan de Fase 7 §12.3 marca lo diferido como resuelto.

---

## §9. LO QUE ESTA FASE **NO** RESUELVE (y hay que decirlo)

1. **El módulo Pedidos/Producción no existe todavía.** Esta fase implementa el **contrato**, no el
   módulo. El pedido se guarda en `orders`; la máquina de 14 estados y el KDS son de otra fase.
2. **Vista General no existe todavía.** El POS lee la política con un **default seguro**
   (`PAGO_COMPLETO`). La UI para cambiarla es de Vista General (F7.5b).
3. **El reparto no funciona.** `delivery_fee` queda en 0; no hay geocodificación ni rutas.
4. **El cliente no recibe notificación.** Eso es la Fase 8 (CRM + Notificaciones).
5. **Los pedidos de otros canales (teléfono, WhatsApp) no se crean aquí.** El POS solo proyecta
   lo suyo; el estado `TENTATIVO` de otros canales es de Pedidos.

> **Esto es intencional y correcto.** El principio "de adentro hacia afuera" dice: se construye la
> pieza con su puerta, y cuando los módulos vecinos existan, **el POS no se toca**.

---

## §10. POR QUÉ ESTA FASE ES LA CORRECTA AHORA

1. **Cierra una regresión funcional real:** el POS viejo procesaba pedidos; el nuevo no.
2. **El cimiento ya está:** `tickets.order_*` (los 9 campos) y la tabla `orders`.
3. **No depende de módulos inexistentes:** la proyección es autocontenida y la política tiene default.
4. **Respeta la frontera:** el POS escribe su tabla y proyecta por contrato.
5. **Respeta DT-06:** la política se declara en Vista General; el POS la **lee**, no la define.
6. **Respeta DT-07:** la ausencia de configuración degrada al modo más seguro, sin bloquear la venta.
7. **Prepara la Fase 8:** cuando llegue CRM + Notificaciones, el pedido ya estará registrado.

---

## §11. LA PRUEBA DE QUE LA FRONTERA ESTÁ BIEN TRAZADA

> **Cuando Vista General exista, el POS no se toca.**

Esa es la prueba. Si al construir Vista General hubiera que **modificar el POS** para que lea la
política, la frontera estaría mal trazada. Con este diseño:

| Escenario | Qué cambia en el POS |
|---|---|
| Vista General no existe (hoy) | Nada. El POS usa el default `PAGO_COMPLETO`. |
| Vista General existe y declara `SIN_PAGO` | Nada. El POS ya lee la política; recibe otro valor. |
| Vista General existe y declara `ANTICIPO 30%` | Nada. El POS ya lee la política; recibe otro valor. |
| Se cambia la política en caliente | Nada. El POS la relee en la siguiente operación. |

**El POS es un consumidor puro.** No tiene estado de política, no la cachea, no la escribe.

---

## §12. REGISTRO DE CAMBIOS (v2.0 → v3.0)

| # | Cambio | Origen |
|---|---|---|
| 1 | **Reescrita §0:** la política de pago ya no es una constante ("no ticket no food") sino un **valor transversal configurable** en Vista General | Decisión del dueño (30 Sep 2026) |
| 2 | **Declarado `orders.payment_policy` + `orders.deposit_percent`** como 4.º valor de DT-06 | Propuesta 1 aceptada |
| 3 | **Añadido el default seguro `PAGO_COMPLETO`** con degradación DT-07 | Propuesta 2 aceptada |
| 4 | **Dividida la fase en F7.5a (hoy) + F7.5b (cuando Vista General exista)** | Propuesta 3 aceptada |
| 5 | **Añadida F7.5.1:** lector de política con default seguro | Consecuencia de §0 |
| 6 | **Reescrita F7.5.2:** la proyección decide el estado **según la política** (3 modos) | Consecuencia de §0 |
| 7 | **Añadido el criterio "el POS no lee ni escribe `system_settings`"** (F7.5.6 criterio 7) | DT-06 |
| 8 | **Añadidos 2 riesgos** (hardcodear la política; política corrupta) | DT-06 / DT-07 |
| 9 | **Añadida §11** (la prueba de la frontera) | Consecuencia de §0 |
| 10 | **Actualizado el DoD** (13 criterios, incluye la lectura de la política) | Consecuencia de §0 |

### Cambios heredados de v1.0 → v2.0 (se conservan)

| # | Cambio | Origen |
|---|---|---|
| 1 | **Añadida §0** (la política del negocio) | Respuesta del dueño (30 Sep 2026) |
| 2 | **Corregida §1.4:** el cimiento son `tickets.order_*` + `orders`, no solo `orders` | Autocrítica D-1 |
| 3 | **Añadida F7.5.0:** extender `POST /pos/tickets` con los 9 campos | Autocrítica D-2 |
| 4 | **Reescrita la proyección:** servicio **interno** (no endpoints que el POS consuma) | Autocrítica D-3 |
| 5 | **Eliminado `ordersService.js`** del frontend (el POS no llama a Pedidos) | Autocrítica D-3 |
| 6 | **Añadidos los endpoints 15/16** solo para consumidores externos | Autocrítica D-3 |
| 7 | **Corregidos los gates:** el frontend envía campos, no llama contratos | Autocrítica D-3 |

---

**FIN DEL PLAN — FASE 7.5: PEDIDOS PROGRAMADOS (v3.0)**
