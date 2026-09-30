# 📋 PLAN DE ABORDAJE — FASE 7.5: PEDIDOS PROGRAMADOS (el puente POS → Pedidos → Producción)

**Versión:** 3.1 (correcciones de la autocrítica verificadas contra el código)
**Fecha:** 30 Sep 2026
**Autor:** Arquitecto del Nuevo POS
**Estado:** Propuesta — pendiente de aprobación del dueño
**Precedente:** `PLAN_DE_ABORDAJE_FASE_7_POR_PARTES.md` §12.3 (lo diferido)
**Autocrítica que origina esta versión:** `AUTOCRITICA_PLAN_F7_5_V3.md` (4 defectos, 1 crítico)
**Cambios vs v3.0:** ver §12 (registro de cambios)

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

### §0.5 La infraestructura de almacenamiento: `system_settings` (corrección D-4)

> **La autocrítica D-4 reveló que `system_settings` NO EXISTE en el código.**
> Los 17 modelos de [`models/__init__.py`](../NUEVO-POS/apps/api/models/__init__.py:17) no la incluyen.
> `SystemSetting` aparece **solo en comentarios** de `consolidacion/registry.py`.

**Esto obliga a declarar un entregable nuevo:** crear la tabla `system_settings`. Y hay que decir
**por qué no es una invasión de frontera**:

| Pregunta | Respuesta |
|---|---|
| ¿`system_settings` es de Vista General? | **NO.** Es la **capa de almacenamiento** que DT-06.2 ya declaró. |
| ¿Vista General qué es entonces? | La **UI** que escribe en `system_settings`. |
| ¿Quién más la usa? | Todos los módulos que consumen valores transversales (Consolidación ya la referencia). |
| ¿Crearla ahora viola la frontera? | **NO.** Es crear la infraestructura compartida, no la UI de otro módulo. |

> **Analogía:** crear `system_settings` es como tender la tubería. Vista General es la llave que la
> abre. El POS es un grifo que la consume. Tender la tubería no es "ser Vista General".

**Esta es la primera piedra de Vista General en el nuevo POS, y se declara como tal.**

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

El cimiento real son **TRES** estructuras, y las dos primeras ya existen:

| Estructura | Qué es | Estado |
|---|---|---|
| **`tickets.order_*`** (9 campos) | La **copia de trabajo del POS**. El POS la captura y la muestra. | **YA EXISTE** ([`models/pos.py:75-89`](../NUEVO-POS/apps/api/models/pos.py:75)) |
| **`orders`** | La **proyección gobernada por Pedidos**. Tiene el ciclo de 14 estados y el `earliest_ready_at` autoritativo. | **YA EXISTE** ([`models/orders.py:21`](../NUEVO-POS/apps/api/models/orders.py:21)) |
| **`system_settings`** | La **capa de almacenamiento de DT-06**. Guarda la política de pago. | **NO EXISTE** — se crea en F7.5.1a (D-4) |

**Los 9 campos de `tickets.order_*`** (verificados en [`models/pos.py:75-89`](../NUEVO-POS/apps/api/models/pos.py:75)):

| # | Campo | Tipo | Default |
|---|---|---|---|
| 1 | `order_type` | String NOT NULL | `VENTA_DIRECTA` |
| 2 | `order_status` | String NOT NULL | `PROGRAMADO PARA SER PREPARADO` |
| 3 | `delivery_type` | String NULL | — |
| 4 | `customer_name` | String NULL | — |
| 5 | `customer_phone` | String NULL | — |
| 6 | `committed_at` | DateTime(tz) NULL | — |
| 7 | `packaging_type` | String NULL | — |
| 8 | `delivery_address` | Text NULL | — |
| 9 | `order_notes` | Text NULL | — |

**El contrato 15 es el puente entre `tickets.order_*` y `orders`.** El POS escribe `tickets.order_*`
(su propia tabla, sin violar la frontera) y **proyecta** a `orders` vía el contrato.

### §1.5 Los 10 campos del contrato 15 (corrección D-6)

> **La autocrítica D-6 reveló que el plan hablaba de "9 campos" pero el contrato 15 exige 10.**

El contrato 15 ([`registry.py:353-364`](../NUEVO-POS/apps/api/contracts/registry.py:353)) declara:

| # | Campo de entrada | Origen en el POS |
|---|---|---|
| 1 | `ticket_id` | `ticket.id` |
| 2 | `order_type` | `tickets.order_type` |
| 3 | **`status_ticket`** | **`tickets.status`** (`OPEN`/`PAID`) — **el 10.º, que la v3.0 omitía** |
| 4 | `delivery_type` | `tickets.delivery_type` |
| 5 | `customer_name` | `tickets.customer_name` |
| 6 | `customer_phone` | `tickets.customer_phone` |
| 7 | `committed_at` | `tickets.committed_at` |
| 8 | `packaging_type` | `tickets.packaging_type` |
| 9 | `delivery_address` | `tickets.delivery_address` |
| 10 | `notes` | `tickets.order_notes` |

**`status_ticket` NO es una columna nueva:** se **deriva** de `tickets.status`. El proveedor lo usa
para mapear (`OPEN → TENTATIVO`, `PAID → PAGADO`). El POS lo envía; no lo almacena.

### §1.6 El destino de `tickets.order_status` (corrección D-7)

> **La autocrítica D-7 reveló que `Ticket.order_status` ya tiene un default que el plan ignoraba.**

[`Ticket.order_status`](../NUEVO-POS/apps/api/models/pos.py:78) tiene
`default="PROGRAMADO PARA SER PREPARADO"`. Hay **dos** estados en juego y no se deben confundir:

| Campo | Dónde vive | Quién lo gobierna | Qué hace el POS |
|---|---|---|---|
| `tickets.order_status` | La copia de trabajo del POS | **El POS** (es su tabla) | Lo deja en su **default**; **no** lo usa para proyectar |
| `orders.status` | La proyección | **Pedidos** (14 estados) | Lo **recibe** del proveedor; no lo inventa |

> **Decisión explícita:** el POS **no escribe** `tickets.order_status` (queda en su default) y **no lo
> usa** para la proyección. La proyección usa `tickets.status` (`OPEN`/`PAID`), que es lo que el
> contrato 15 pide como `status_ticket`. Esto elimina la ambigüedad que la v3.0 dejaba abierta.

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

### §2.5 La atomicidad exige reestructurar los commits (corrección D-5)

> **La autocrítica D-5 reveló que `crear_ticket` hace `commit()` temprano, y la verificación
> adicional reveló que `cobrar_ticket` también.**

| Endpoint | Línea | Problema |
|---|---|---|
| [`crear_ticket`](../NUEVO-POS/apps/api/routers/pos.py:294) | 294–295 | `db.add(ticket)` → `await db.commit()` — cierra la transacción antes de poder proyectar |
| [`cobrar_ticket`](../NUEVO-POS/apps/api/routers/pos.py:350) | 350 | `await db.commit()` — cierra la transacción antes de poder proyectar |

**La corrección:** en ambos, reemplazar `commit()` por `flush()` + proyección + `commit()`:

```python
# crear_ticket (SIN_PAGO proyecta aquí)
db.add(ticket)
await db.flush()              # ← obtiene el id SIN cerrar la transacción
await proyectar_pedido(db, ticket, politica)   # ← misma transacción
await db.commit()

# cobrar_ticket (PAGO_COMPLETO / ANTICIPO proyectan aquí)
ticket.status = rn14_ciclo_de_vida("PAID")
...
await db.flush()              # ← persiste el cambio de estado SIN cerrar
await proyectar_pedido(db, ticket, politica)   # ← misma transacción
await db.commit()
```

> **Riesgo declarado:** `crear_ticket` tiene un gate de Fase 3.2 (persistencia atómica) y
> `cobrar_ticket` tiene gates de Fase 3.2 y 4.0. Tocarlos puede romperlos. **El gate de F7.5.2 debe
> incluir la verificación de que esos gates siguen verdes.**

---

## §3. ALCANCE DEFINIDO

### §3.1 Lo que SÍ entra en F7.5a (ejecutable hoy)

| # | Entregable | Descripción |
|---|---|---|
| 1 | **Extender `POST /pos/tickets`** | Acepta y persiste los 9 campos del pedido en `tickets.order_*` |
| 2 | **Crear `system_settings`** | Modelo + migración + seed del default (D-4) — la infraestructura de DT-06 |
| 3 | **Lector de política con default seguro** | `services/politica_pedidos.py` — lee `orders.payment_policy` de `system_settings`; si falta, `PAGO_COMPLETO` |
| 4 | **Servicio interno de proyección** | `services/pedidos.py` — proyecta `tickets.order_*` → `orders` (contrato 15), **en la misma transacción**, **cuando la política lo permite** |
| 5 | **Reestructurar `crear_ticket` y `cobrar_ticket`** | `commit()` → `flush()` + proyección + `commit()` (D-5) |
| 6 | **Endpoints 15/16** | `POST /orders/from-ticket` y `GET /orders/by-ticket/{ticket_id}` — para **consumidores externos** (Pedidos, CRM), no para el POS |
| 7 | **Hook `useOrderProgramming`** | Estado local del pedido (tipo, datos, cálculo de `earliest_ready_at` para mostrar) |
| 8 | **Modal heredado** | `ProgramacionPedidoModal.jsx` — **UX heredada del POS viejo** (§6.8 del Plan Maestro) |
| 9 | **Botón "PEDIDO" en el header** | `POSHeader.jsx` — el toggle VENTA_DIRECTA ↔ PEDIDO |
| 10 | **Panel de pedido en el checkout** | Muestra el pedido programado antes de cobrar |
| 11 | **Gates** | API + frontend, con los criterios corregidos (§5) |

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

> **Nota crítica:** los entregables 1–11 funcionan **hoy**, sin depender de módulos inexistentes.
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
  **9 campos** del pedido (opcionales).
- Extender `crear_ticket` en [`pos.py`](../NUEVO-POS/apps/api/routers/pos.py:244) para persistirlos
  en `tickets.order_*`.
- **Declarar explícitamente** que `tickets.order_status` **queda en su default** y **no se usa** para
  proyectar (D-7).
- **Gate:** criterios:
  1. `POST /pos/tickets` con `order_type=PEDIDO` **persiste los 9 campos**.
  2. `POST /pos/tickets` sin campos de pedido deja los defaults (`VENTA_DIRECTA`, etc.).
  3. Un `order_type` inválido es **422**.
  4. `delivery_type` inválido (no PICKUP/DOMICILIO) es **422**.
  5. `packaging_type` inválido (no PROPIO/VENTA) es **422**.
  6. **`order_status` queda en su default** `"PROGRAMADO PARA SER PREPARADO"` (D-7).

**Por qué primero:** sin esto, el modal captura datos que se pierden (D-2 de la autocrítica v1.0).

### §5.1a — Crear `system_settings` (API) + gate

**Objetivo:** crear la **capa de almacenamiento de DT-06** (D-4).

**Entregables:**
- `models/settings.py` con `SystemSetting`:
  - `clave` (String, PK) — clave punteada (ej. `orders.payment_policy`).
  - `valor` (Text) — el valor serializado.
  - `updated_at` (DateTime(tz)).
- Registrarlo en [`models/__init__.py`](../NUEVO-POS/apps/api/models/__init__.py:17) (pasa de 17 a 18 modelos).
- Migración Alembic.
- **Seed del default:** `orders.payment_policy = 'PAGO_COMPLETO'` y `orders.deposit_percent = '50'`.
- **Gate:** criterios:
  1. La tabla `system_settings` existe y tiene las columnas `clave`, `valor`, `updated_at`.
  2. El seed inserta `orders.payment_policy = 'PAGO_COMPLETO'`.
  3. El seed inserta `orders.deposit_percent = '50'`.
  4. Insertar una clave duplicada falla (PK).

> **Nota de frontera:** crear `system_settings` **no es invadir Vista General** (§0.5). Es la
> infraestructura compartida que DT-06.2 ya declaró.

### §5.1b — Lector de política con default seguro (API) + gate

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
  - **Envía los 10 campos del contrato 15**, incluido `status_ticket` derivado de `tickets.status` (D-6).
  - **Decide el estado según la política** (§0.2):
    - `SIN_PAGO` → proyecta al crear, `TENTATIVO`.
    - `ANTICIPO` → proyecta cuando `pagado ≥ deposit_percent %`, `TENTATIVO`.
    - `PAGO_COMPLETO` → proyecta solo en `PAID`, `PAGADO`.
  - **Se ejecuta en la misma transacción** del guardado/cobro.
- **Reestructurar `crear_ticket` y `cobrar_ticket`** en [`pos.py`](../NUEVO-POS/apps/api/routers/pos.py:294):
  `commit()` → `flush()` + proyección + `commit()` (D-5).
- **Gate:** criterios:
  1. Con `PAGO_COMPLETO`: cobrar un ticket PEDIDO **crea la fila en `orders`** con `status=PAGADO`.
  2. Con `PAGO_COMPLETO`: cobrar un ticket VENTA_DIRECTA **no crea fila** en `orders` (RN-59).
  3. Con `SIN_PAGO`: crear un ticket PEDIDO **crea la fila** con `status=TENTATIVO`.
  4. Con `ANTICIPO` (30%): un pago del 20% **no** proyecta; uno del 30% **sí** proyecta.
  5. La proyección es **idempotente**: re-cobrar no duplica.
  6. `earliest_ready_at` se calcula y se persiste.
  7. Si la proyección falla, **el cobro se completa** (DT-07) y se registra el error.
  8. `delivery_fee` se persiste como `Numeric(12,2)` (no Float).
  9. **`status_ticket` se deriva de `tickets.status`** y determina el mapeo (D-6).
  10. **Los gates de F3.2 (atómico) y F4.0 (caja) siguen verdes** tras reestructurar los commits (D-5).

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
  8. **`status_ticket` viaja en la entrada** y determina el mapeo del estado (D-6).

### §5.4 — El hook `useOrderProgramming` (frontend)

**Objetivo:** el estado local del pedido. **No llama a Pedidos.**

**Entregables:**
- `hooks/useOrderProgramming.js`:
  - `orderType` (`VENTA_DIRECTA` | `PEDIDO`) + `setOrderType`.
  - `orderData` + `guardarPedido(datos)` / `limpiarPedido()`.
  - `calcularEarliestReady(lineas)` — espejo local para **mostrar** antes de cobrar.
  - `camposParaTicket()` — devuelve los **9 campos** para incluirlos en `POST /pos/tickets`.
- **Gate:** criterios:
  1. `orderType` arranca en `VENTA_DIRECTA`.
  2. `calcularEarliestReady` devuelve la fecha correcta según el lead time máximo.
  3. `camposParaTicket()` devuelve los 9 campos con los defaults correctos (PICKUP, PROPIO).
  4. `limpiarPedido` resetea el estado.
  5. **No** hace ninguna llamada de red (verificado con un fetch espiado).

### §5.5 — El modal heredado (frontend)

**Objetivo:** `components/ProgramacionPedidoModal.jsx` con la UX del POS viejo, reescrita.

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
| **DT-06** (configuración en Vista General) | F7.5.1a (crear `system_settings`) + F7.5.1b (el POS **lee** la política, no la define) |
| **DT-06.2** (la capa de almacenamiento) | F7.5.1a (la tabla `system_settings`) |
| **DT-06.3.2** (un solo lugar de declaración) | F7.5.1b (el POS no tiene selector de política) |
| **DT-07** (degradación: la venta nunca se bloquea) | F7.5.1b (default seguro) + F7.5.2 (criterio 7) |
| **Política `PAGO_COMPLETO`** (default seguro) | F7.5.1b (criterio 1) + F7.5.2 (criterios 1–2) |
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
| **Contrato 15** (los 10 campos) | F7.5.0 (9 persistidos) + F7.5.2 (criterio 9: `status_ticket`) + F7.5.3 (criterio 8) |
| **F3.2 / F4.0** (atomicidad y caja) | F7.5.2 (criterio 10: los gates siguen verdes) |

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
| **La política corrupta rompe el POS** | Baja | **Alto** | **Default seguro `PAGO_COMPLETO`; el gate lo verifica (F7.5.1b criterios 1 y 4)** |
| **Reestructurar `crear_ticket`/`cobrar_ticket` rompe los gates de F3.2/F4.0** | Media | **Alto** | **El gate de F7.5.2 criterio 10 verifica que siguen verdes; cambio quirúrgico (`commit()` → `flush()`)** |
| **Crear `system_settings` se percibe como invadir Vista General** | Baja | Medio | **§0.5 lo declara: es la infraestructura de DT-06.2, no la UI** |

---

## §8. DEFINICIÓN DE TERMINADO (DoD)

La Fase 7.5a está terminada cuando:

1. ✅ `POST /pos/tickets` **persiste** los 9 campos del pedido en `tickets.order_*`.
2. ✅ La tabla `system_settings` **existe** con el seed del default (D-4).
3. ✅ El POS **lee** la política de `system_settings`; si falta, asume `PAGO_COMPLETO`.
4. ✅ Con `PAGO_COMPLETO`: cobrar un ticket PEDIDO **proyecta** a `orders` con `status=PAGADO`, en la misma transacción.
5. ✅ Con `SIN_PAGO` y `ANTICIPO`: la proyección respeta el modo (gate F7.5.2 criterios 3–4).
6. ✅ Cobrar un ticket VENTA_DIRECTA **no** proyecta (RN-59).
7. ✅ El POS **nunca** importa `Order` ni escribe `orders` (verificado por el guard de frontera).
8. ✅ El POS **nunca** define ni persiste la política (verificado por el gate F7.5.6 criterio 7).
9. ✅ El frontend **nunca** llama a `/orders/from-ticket` (verificado por el gate).
10. ✅ Si la proyección falla, la venta se completa igual (DT-07).
11. ✅ El operador puede marcar un ticket como PEDIDO, programarlo y cobrarlo.
12. ✅ **Los gates de F3.2 y F4.0 siguen verdes** tras reestructurar los commits (D-5).
13. ✅ **`status_ticket` se deriva de `tickets.status`** y viaja en el contrato 15 (D-6).
14. ✅ **`tickets.order_status` queda en su default** y no se usa para proyectar (D-7).
15. ✅ Todos los gates verdes; CI completo verde.
16. ✅ La ficha documenta la evidencia.
17. ✅ El plan de Fase 7 §12.3 marca lo diferido como resuelto.

---

## §9. LO QUE ESTA FASE **NO** RESUELVE (y hay que decirlo)

1. **El módulo Pedidos/Producción no existe todavía.** Esta fase implementa el **contrato**, no el
   módulo. El pedido se guarda en `orders`; la máquina de 14 estados y el KDS son de otra fase.
2. **Vista General no existe todavía.** El POS lee la política con un **default seguro**
   (`PAGO_COMPLETO`). La UI para cambiarla es de Vista General (F7.5b). **La tabla `system_settings`
   sí se crea** (es la infraestructura), pero su UI no.
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

## §12. REGISTRO DE CAMBIOS (v3.0 → v3.1)

| # | Cambio | Defecto que corrige |
|---|---|---|
| 1 | **Dividida F7.5.1 en F7.5.1a (crear `system_settings`) + F7.5.1b (el lector)** | D-4 (CRÍTICA) |
| 2 | **Añadida §0.5:** crear `system_settings` es la infraestructura de DT-06, no una invasión de Vista General | D-4 |
| 3 | **Añadido a F7.5.2: reestructurar `crear_ticket` Y `cobrar_ticket` (`flush()` + proyección + `commit()`)** | D-5 |
| 4 | **Añadida §2.5** con el detalle de los dos commits tempranos y el código de la corrección | D-5 |
| 5 | **Añadido el riesgo "tocar los commits puede romper los gates de F3.2/F4.0"** | D-5 |
| 6 | **Añadido el criterio 10 a F7.5.2** (los gates de F3.2/F4.0 siguen verdes) | D-5 |
| 7 | **Añadida §1.5:** los 10 campos del contrato 15, con `status_ticket` derivado de `tickets.status` | D-6 |
| 8 | **Añadido el criterio 9 a F7.5.2 y el criterio 8 a F7.5.3** (`status_ticket`) | D-6 |
| 9 | **Añadida §1.6:** el destino de `tickets.order_status` (queda en su default, no se usa) | D-7 |
| 10 | **Añadido el criterio 6 a F7.5.0** (`order_status` queda en su default) | D-7 |
| 11 | **Actualizado el cimiento (§1.4):** ahora son TRES estructuras (se añade `system_settings`) | D-4 |
| 12 | **Actualizado el DoD** (17 criterios, incluye D-4/D-5/D-6/D-7) | Todos |
| 13 | **Actualizada la trazabilidad** (DT-06.2, contrato 15, F3.2/F4.0) | Todos |

### Cambios heredados de v2.0 → v3.0 (se conservan)

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

**FIN DEL PLAN — FASE 7.5: PEDIDOS PROGRAMADOS (v3.1)**
