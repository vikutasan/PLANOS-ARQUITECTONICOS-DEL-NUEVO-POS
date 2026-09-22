# CONTRATOS ENTRE MÓDULOS DEL NUEVO POS

**Documento 9 — Las fronteras del edificio nuevo**

> **Propósito de este documento.** El [`MODELO_DE_DATOS_DEL_NUEVO_POS.md`](./MODELO_DE_DATOS_DEL_NUEVO_POS.md)
> dice **qué tablas** tiene el POS nuevo. Este documento dice **cómo se comunica** el POS
> con los demás módulos del ERP **sin leer nunca una tabla ajena**.
>
> **Regla de oro que este documento hace cumplir.** *Un módulo no lee las tablas de otro
> módulo.* Si el POS necesita saber el stock de un producto, **no** consulta
> `stock_almacen`: pide el dato al módulo de Almacenes por su contrato. Si necesita saber
> quién es el cajero, **no** consulta `employees`: pide el dato al módulo de Seguridad.
>
> **Por qué importa.** Hoy el POS lee `products.stock` y `products.warehouse` directamente
> (ver [`catalog/models.py`](../../apps/api/modules/catalog/models.py:66)). Eso es deuda:
> crea una **doble fuente de verdad** y **hardcodea la sucursal**, lo que rompe el principio
> SaaS (una instalación por sucursal). Este documento define el sustituto correcto.
>
> **Anclaje.** Este documento está anclado al commit `5802f45` (V23) del ERP actual. La
> ingeniería inversa se hizo sobre `fe9f6ed` (tag `v22-estable-fe9f6ed`). Cada acoplamiento
> que se corrige se cita con su `archivo:línea`.

---

## SECCIÓN 0 — CÓMO LEER ESTE DOCUMENTO

Cada contrato se presenta con esta estructura:

```
CONTRATO <nombre>
  Consumidor:   <quién pide el dato>  (el POS)
  Proveedor:    <quién lo entrega>    (el módulo dueño de la tabla)
  Acoplamiento actual:  <cómo se hace HOY>  →  <archivo:línea>
  Problema:     <por qué está mal>
  Contrato nuevo:  <la operación, su entrada y su salida>
  Garantías:    <qué promete el proveedor>
  Errores:      <qué devuelve cuando no puede>
```

**Los 3 principios que rigen TODOS los contratos:**

| # | Principio | Qué significa |
|---|-----------|---------------|
| **P-01** | **El dueño de la tabla es el único que la escribe** | Solo Almacenes escribe `stock_almacen`. Solo Seguridad escribe `employees`. |
| **P-02** | **El consumidor pide por operación, no por tabla** | El POS pide "dame el stock de estos 5 SKU", no "dame la tabla `stock_almacen`". |
| **P-03** | **El contrato es estable; la tabla es libre** | Almacenes puede refactorizar su esquema sin romper al POS, mientras respete el contrato. |

---

## SECCIÓN 1 — CONTRATO DE PRODUCTOS (Catálogo → POS)

### 1.1 Acoplamiento actual

```
Hoy el POS lee directamente:
  products.stock       →  apps/api/modules/catalog/models.py:66   (OBSOLETO)
  products.warehouse   →  apps/api/modules/catalog/models.py:72   (HARDCODE)
```

**Problema.** `products.stock` es una **doble fuente de verdad**: el stock real vive en
`stock_almacen` (Almacenes), pero el catálogo guarda una copia que se desincroniza.
`products.warehouse` **hardcodea la sucursal** dentro del producto, lo que impide que el
mismo catálogo sirva a dos sucursales (rompe el principio SaaS).

### 1.2 Contrato nuevo

```
CONTRATO catalogo.productos_para_venta
  Consumidor:   POS
  Proveedor:    Catálogo
  Operación:    GET /catalog/products?channel=<PANADERIA|HELADERIA>&active=true
  Entrada:      channel (obligatorio), active (default true)
  Salida:       Lista de ProductLight:
                  id            UUID
                  sku           String
                  name          String
                  price         Numeric(12,2)
                  category_id   UUID
                  image_url     String NULL
                  is_active     Boolean
  Garantías:
    - NUNCA incluye `stock` ni `warehouse` (esos no son del catálogo).
    - El precio es el vigente al momento de la consulta.
    - Solo devuelve productos activos si `active=true`.
  Errores:
    - 404 si el canal no existe.
    - Lista vacía (200) si no hay productos: NO es error.
```

**Nota de diseño.** El POS **no necesita** el stock para vender en mostrador: el stock se
descuenta al cobrar, vía el contrato de Almacenes (§2). El POS solo necesita el stock para
**mostrar disponibilidad** en heladería; para eso usa el contrato §2.3.

---

## SECCIÓN 2 — CONTRATO DE ALMACENES (Almacenes → POS)

### 2.1 Acoplamiento actual

```
Hoy el POS descuenta stock escribiendo directo:
  products.stock -= quantity   →  (implícito en el flujo de checkout)
  warehouse_events             →  apps/api/modules/warehouse/models.py:110
```

**Problema.** El POS escribe en tablas de Almacenes. Eso viola P-01: el POS no es dueño
del inventario. Además, el descuento no es atómico con el cobro: si falla a la mitad, el
stock queda inconsistente.

### 2.2 Contrato nuevo — descontar al cobrar

```
CONTRATO almacenes.consumir_por_venta
  Consumidor:   POS
  Proveedor:    Almacenes
  Operación:    POST /warehouse/consume
  Entrada:      {
                  evento_id:   UUID   ← idempotencia (el POS lo genera)
                  ticket_id:   UUID
                  almacen_id:  UUID
                  items: [ { sku: String, cantidad: Numeric(12,3) } ]
                }
  Salida:       {
                  evento_id:   UUID
                  aplicado:    Boolean
                  movimientos: [ { sku, cantidad_antes, cantidad_despues } ]
                }
  Garantías:
    - IDEMPOTENTE por `evento_id`: repetir la llamada NO descuenta dos veces.
    - ATÓMICO: o se aplican todos los items, o no se aplica ninguno.
    - Escribe un asiento en `movimientos_inventario` por cada item.
    - Si un SKU no tiene stock suficiente, NO falla: registra el movimiento igual
      (el negocio permite vender sin stock; el faltante se ve en el reporte).
  Errores:
    - 409 si `evento_id` ya fue aplicado con OTRO contenido (conflicto real).
    - 400 si `almacen_id` no existe.
```

**Por qué `evento_id` lo genera el POS.** Es el patrón **Outbox** (acción canónica A-04):
el POS guarda el evento en su propia tabla `warehouse_events` **dentro de la misma
transacción** que el cobro. Si la red falla, el evento queda pendiente y se reintenta con
el mismo `evento_id`, garantizando que no se descuente dos veces.

### 2.3 Contrato nuevo — consultar disponibilidad

```
CONTRATO almacenes.disponibilidad
  Consumidor:   POS (solo para mostrar, no para vender)
  Proveedor:    Almacenes
  Operación:    GET /warehouse/availability?almacen_id=<UUID>&skus=<csv>
  Entrada:      almacen_id (obligatorio), skus (lista separada por coma)
  Salida:       [ { sku, cantidad_actual, unidad } ]
  Garantías:
    - Devuelve 0 para los SKU sin fila (no 404).
    - Es una LECTURA: no bloquea, no reserva.
  Errores:
    - 400 si falta `almacen_id`.
```

**Advertencia de diseño.** Este contrato es **informativo**. El POS lo usa para pintar
"quedan 3" en la tarjeta del producto. **Nunca** lo usa para decidir si vende o no: la
venta siempre se permite y el descuento se hace en §2.2.

---

## SECCIÓN 3 — CONTRATO DE PRODUCCIÓN (Producción → POS)

### 3.1 Acoplamiento actual

```
Hoy el POS lee directamente:
  doughs  →  (módulo de Producción, leído desde el POS para saber qué hay en vitrina)
```

**Problema.** El POS consulta una tabla de Producción. Si Producción cambia su esquema,
el POS se rompe. Además, el POS no debería saber de "masas": debería saber de "productos
disponibles para vender ahora".

### 3.2 Contrato nuevo

```
CONTRATO produccion.disponible_para_vender
  Consumidor:   POS
  Proveedor:    Producción
  Operación:    GET /production/available?channel=<PANADERIA>&date=<YYYY-MM-DD>
  Entrada:      channel (obligatorio), date (default hoy local)
  Salida:       [ { sku, cantidad_disponible, lote, hora_salida } ]
  Garantías:
    - `cantidad_disponible` es lo que Producción declara vendible HOY.
    - No expone recetas, masas ni insumos: solo el resultado vendible.
  Errores:
    - 200 con lista vacía si Producción no ha declarado nada (no es error).
```

**Nota.** Este contrato es **opcional para el POS de mostrador** (que vende lo que hay en
vitrina física). Es **obligatorio para el POS de heladería**, que sí necesita saber qué
sabores están producidos hoy.

---

## SECCIÓN 4 — CONTRATO DE AUDITORÍA (POS → Auditoría)

### 4.1 Acoplamiento actual

```
Hoy el POS escribe su propia auditoría:
  pos_audit.audit_pos_write()  →  apps/api/modules/pos/pos_audit.py:35
```

**Problema.** No es un problema: es una **cicatriz valiosa**. El POS audita sus propias
escrituras porque nació de un bug real (operaciones sin rastro). Se conserva.

### 4.2 Contrato nuevo — el POS es PROVEEDOR aquí

```
CONTRATO pos.eventos_auditables
  Consumidor:   Auditoría (y Estadísticas)
  Proveedor:    POS
  Operación:    El POS publica cada escritura en su tabla `pos_audit_log`:
                  id            UUID
                  endpoint      String
                  payload       JSON
                  response_code Integer
                  terminal_id   String
                  empleado_id   UUID NULL
                  created_at    DateTime(timezone=True)
  Garantías:
    - Toda escritura del POS deja rastro (no hay excepción).
    - El log es append-only: nunca se actualiza ni se borra.
  Errores:
    - Si el log falla, la escritura de negocio NO se revierte (el log es best-effort).
```

**Nota de dirección.** Este contrato va **en sentido contrario** a los demás: aquí el POS
es el **proveedor** y Auditoría es el **consumidor**. Es el único contrato donde el POS
expone datos.

---

## SECCIÓN 5 — CONTRATO DE ESTADÍSTICAS (POS → Estadísticas)

### 5.1 Acoplamiento actual

```
Hoy Estadísticas lee directamente:
  tickets, ticket_items, cash_sessions  →  tablas del POS
```

**Problema.** Estadísticas lee las tablas del POS. Si el POS cambia su esquema, los
reportes se rompen. Además, los reportes hacen consultas pesadas sobre las tablas
transaccionales, compitiendo con la venta.

### 5.2 Contrato nuevo

```
CONTRATO pos.resumen_de_venta
  Consumidor:   Estadísticas
  Proveedor:    POS
  Operación:    GET /pos/analytics/summary?from=<ISO>&to=<ISO>&terminal_id=<opcional>
  Entrada:      from, to (obligatorios, en hora LOCAL de México), terminal_id (opcional)
  Salida:       {
                  rango: { from, to, timezone: "America/Mexico_City" },
                  total_ventas:      Numeric(12,2),
                  numero_tickets:    Integer,
                  ticket_promedio:   Numeric(12,2),
                  por_canal:         [ { channel, total, tickets } ],
                  por_terminal:      [ { terminal_id, total, tickets } ],
                  por_dia:           [ { fecha_local, total, tickets } ]
                }
  Garantías:
    - Los límites `from`/`to` se interpretan en hora LOCAL, no UTC.
      (Esto corrige el bug D-28: tickets de las 23:30 local aparecían en el día
      equivocado. Ver [`test_bloque9d_3bugs.py`](../../apps/api/tests/test_bloque9d_3bugs.py:88).)
    - Solo cuenta tickets con status PAID.
    - Es una LECTURA agregada: no expone tickets individuales.
  Errores:
    - 400 si `from` > `to`.
    - 400 si el rango supera 366 días (protección contra consultas que tumban la BD).
```

**Por qué el POS es el proveedor.** El POS es el dueño de `tickets`. Estadísticas no debe
leerlo: debe pedirle el resumen. Así el POS puede cambiar su esquema sin romper reportes.

---

## SECCIÓN 6 — CONTRATO DE SEGURIDAD (Seguridad → POS)

### 6.1 Acoplamiento actual

```
Hoy el POS lee directamente:
  employees  →  (módulo de Seguridad, leído para saber quién capturó/cobró)
```

**Problema.** El POS lee `employees` para poblar `captured_by_id` y `cashed_by_id`. Eso
acopla el POS al esquema de Seguridad. Si Seguridad cambia `employees`, el POS se rompe.

### 6.2 Contrato nuevo

```
CONTRATO seguridad.identidad_del_empleado
  Consumidor:   POS
  Proveedor:    Seguridad
  Operación:    GET /security/employees/{id}/identity
  Entrada:      id (UUID del empleado)
  Salida:       {
                  id:           UUID,
                  nombre:       String,
                  rol:          String,
                  activo:       Boolean,
                  permisos:     [ String ]
                }
  Garantías:
    - Devuelve SOLO identidad y permisos: nunca PIN, nunca hash, nunca salario.
    - Si el empleado está inactivo, `activo=false` (no 404).
  Errores:
    - 404 si el id no existe.
```

### 6.3 Contrato nuevo — validar PIN de caja

```
CONTRATO seguridad.validar_pin
  Consumidor:   POS (pantalla GestorDeCaja)
  Proveedor:    Seguridad
  Operación:    POST /security/validate-pin
  Entrada:      { empleado_id: UUID, pin: String }
  Salida:       { valido: Boolean, empleado: { id, nombre, rol } }
  Garantías:
    - El PIN NUNCA viaja de vuelta: solo viaja la respuesta booleana.
    - El POS NUNCA guarda el PIN ni el hash.
    - Rate-limited: 5 intentos por minuto por empleado.
  Errores:
    - 429 si se excede el rate limit.
    - 200 con `valido=false` si el PIN es incorrecto (no es error de red).
```

**Anclaje.** Hoy el POS valida el PIN contra `employees` directamente (ver
[`GestorDeCaja.jsx`](../../apps/pos/components/GestorDeCaja.jsx:233)). Mañana lo pide a
Seguridad.

---

## SECCIÓN 7 — CONTRATO DE CAJA (POS ↔ Caja)

> **Nota importante.** Este contrato **ya existe y funciona** en el ERP actual. No es deuda:
> es un contrato **bien hecho** que se documenta aquí como referencia. El POS ya habla con
> Caja por endpoints limpios, sin leer sus tablas. Es el modelo a imitar por los demás.

### 7.1 Acoplamiento actual

```
Hoy el POS habla con Caja por endpoints (correcto):
  cash_sessions   →  apps/api/modules/cash/models.py:7
  cash_movements  →  apps/api/modules/cash/models.py:32
  Endpoints       →  apps/api/modules/cash/router.py:11
  Frontend        →  apps/pos/components/GestorDeCaja.jsx:74
```

**No hay problema.** El POS **no** lee `cash_sessions` ni `cash_movements` directamente:
pide a Caja por su API. Esto cumple P-01, P-02 y P-03. Se documenta para que sirva de
ejemplo y para que el POS nuevo lo replique igual.

### 7.2 Contrato existente — consultar la sesión activa

```
CONTRATO caja.sesion_activa
  Consumidor:   POS
  Proveedor:    Caja
  Operación:    GET /cash/sessions/{terminal_id}/active
  Entrada:      terminal_id (String)
  Salida:       {
                  id:            UUID,
                  terminal_id:   String,
                  employee_id:   UUID,
                  employee_name: String,
                  opening_float: Numeric(12,2),
                  status:        String,   ← OPEN | CLOSED
                  opened_at:     DateTime(timezone=True),
                  closed_at:     DateTime(timezone=True) NULL
                }
  Garantías:
    - Devuelve la sesión OPEN de esa terminal, o 404 si no hay ninguna.
    - El POS usa esto para saber si puede cobrar (no hay caja → no hay cobro).
  Errores:
    - 404 si no hay sesión activa en esa terminal.
```

### 7.3 Contrato existente — abrir el turno

```
CONTRATO caja.abrir_turno
  Consumidor:   POS (pantalla GestorDeCaja)
  Proveedor:    Caja
  Operación:    POST /cash/sessions/open
  Entrada:      {
                  terminal_id:   String,
                  employee_id:   UUID,
                  employee_name: String,
                  opening_float: Numeric(12,2)
                }
  Salida:       CashSession (igual que §7.2)
  Garantías:
    - Solo puede haber UNA sesión OPEN por terminal.
    - `employee_name` se desnormaliza para reportes rápidos (decisión consciente).
  Errores:
    - 400 si ya hay una sesión abierta en esa terminal.
```

### 7.4 Contrato existente — registrar cobro y movimientos

```
CONTRATO caja.registrar_movimiento
  Consumidor:   POS
  Proveedor:    Caja
  Operación:    POST /cash/sessions/{session_id}/movements
  Entrada:      {
                  movement_type: String,   ← ENTRADA | SALIDA
                  amount:        Numeric(12,2),
                  concept:       String
                }
  Salida:       CashMovement
  Garantías:
    - Registra una entrada o salida de efectivo (propinas, refuerzo, retiro).
    - El cobro de tickets NO usa este endpoint: el POS asigna `cash_session_id`
      al ticket al cobrar, y Caja lo lee desde ahí.
  Errores:
    - 404 si la sesión no existe.
```

**Cómo se enlaza el cobro con la caja.** El POS **no** llama a Caja para cada cobro. Al
cobrar, el POS escribe `tickets.cash_session_id` (ver Documento 8, §1.2). Caja lee sus
tickets por esa columna. Así el cobro es **una sola transacción** en el POS, y Caja
consolida después. Esto es correcto y se conserva.

### 7.5 Contrato existente — resumen y corte

```
CONTRATO caja.resumen_del_turno
  Consumidor:   POS (pantalla de corte)
  Proveedor:    Caja
  Operación:    GET /cash/sessions/{session_id}/summary
  Salida:       CashSummaryResponse (totales por forma de pago, movimientos, esperado)
  Garantías:
    - Calcula lo esperado a partir de los tickets PAID + los movimientos.
    - Es una LECTURA: no cierra nada.
  Errores:
    - 404 si la sesión no existe.

CONTRATO caja.cerrar_turno
  Consumidor:   POS (pantalla de corte)
  Proveedor:    Caja
  Operación:    POST /cash/sessions/{session_id}/close
  Entrada:      {
                  physical_cash:   Numeric(12,2),
                  physical_credit: Numeric(12,2),
                  physical_debit:  Numeric(12,2)
                }
  Salida:       CashCloseResponse (esperado vs capturado, diferencia)
  Garantías:
    - Marca la sesión como CLOSED y guarda los montos físicos capturados.
    - Devuelve la diferencia (descuadre) para que el cajero la vea.
    - Una sesión CLOSED ya no acepta movimientos.
  Errores:
    - 400 si la sesión ya está cerrada.
```

### 7.6 Contrato existente — reporte diario consolidado

```
CONTRATO caja.reporte_diario
  Consumidor:   POS / Estadísticas
  Proveedor:    Caja
  Operación:    GET /cash/daily-report/{fecha}
  Entrada:      fecha (YYYY-MM-DD, hora local)
  Salida:       Reporte agrupado por canal (PANADERÍA/HELADERÍA) y por cajero/terminal
  Garantías:
    - Agrupa por canal y por cajero/terminal.
    - Usa la fecha LOCAL, no UTC.
  Errores:
    - 400 si el formato de fecha es inválido.
```

**Anclaje.** Todo este contrato ya existe en
[`cash/router.py`](../../apps/api/modules/cash/router.py:11) y se consume desde
[`GestorDeCaja.jsx`](../../apps/pos/components/GestorDeCaja.jsx:74). El POS nuevo lo
replica **igual**: no hay nada que corregir aquí.

---

## SECCIÓN 8 — CONTRATO DE PEDIDOS (POS ↔ Pedidos)

> **Nota importante.** El **levantamiento de pedidos ya existe y funciona** en el ERP actual:
> hay tabla, servicio, endpoints e interfaz. Lo que está **mal trazado es la frontera**: el POS
> **importa la clase `Order` y escribe la tabla `orders` él mismo**. Eso viola P-01. Este
> contrato define el sustituto correcto. A diferencia de Caja (que ya está bien), aquí sí hay
> deuda que corregir.

### 8.1 Acoplamiento actual

```
Hoy el POS escribe la tabla de Pedidos directamente (INCORRECTO):
  from modules.orders.models import Order  →  apps/api/modules/pos/service.py:11
  _sync_order_from_ticket()                →  apps/api/modules/pos/service.py:306
  Order(...) / db.add(new_order)           →  apps/api/modules/pos/service.py:332
  Tabla orders                             →  apps/api/modules/orders/models.py:12
  Endpoints de Pedidos                     →  apps/api/modules/orders/router.py:15
  Frontend (levantamiento)                 →  apps/pos/components/ProgramacionPedidoModal.jsx:13
```

**Problema.** El POS **importa la clase `Order`** ([`pos/service.py`](../../apps/api/modules/pos/service.py:11))
y hace `db.add(new_order)` ([línea 332](../../apps/api/modules/pos/service.py:332)). Es decir,
el POS **escribe una tabla que no es suya**. Eso rompe P-01 y crea dos problemas concretos:

1. **Doble dueño de la tabla.** Si Pedidos cambia su esquema (añade un estado, renombra una
   columna), el POS se rompe en silencio porque comparte la clase.
2. **Lógica de negocio duplicada.** El cálculo de `earliest_ready_at`, la validación de
   `delivery_type` y el ciclo de 14 estados viven en Pedidos, pero el POS los salta al
   escribir directo.

**Lo que SÍ está bien y se conserva.** El puente es **automático y transaccional**: al guardar
un ticket `PEDIDO`, el POS crea el pedido en la misma operación (ver
[`create_ticket()`](../../apps/api/modules/pos/service.py:59)). Eso evita que un pedido quede
sin registro. El POS nuevo conserva esa automaticidad, pero **pidiendo por contrato** en vez
de escribir la tabla.

### 8.2 Contrato nuevo — registrar el pedido desde el ticket

```
CONTRATO pedidos.registrar_desde_ticket
  Consumidor:   POS
  Proveedor:    Pedidos
  Operación:    POST /orders/from-ticket
  Entrada:      {
                  ticket_id:       UUID,
                  order_type:      String,   ← PEDIDO
                  status_ticket:   String,   ← OPEN | PAID
                  delivery_type:   String,   ← PICKUP | DOMICILIO
                  customer_name:   String NULL,
                  customer_phone:  String NULL,
                  committed_at:    DateTime(timezone=True) NULL,
                  packaging_type:  String,   ← PROPIO | VENTA
                  delivery_address: Text NULL,
                  notes:           Text NULL
                }
  Salida:       {
                  order_id:        UUID,
                  status:          String,   ← TENTATIVO | PAGADO
                  earliest_ready_at: DateTime(timezone=True)
                }
  Garantías:
    - Idempotente por `ticket_id`: si el pedido ya existe, lo ACTUALIZA (no duplica).
    - El mapeo de estado es del proveedor: OPEN → TENTATIVO, PAID → PAGADO.
    - Pedidos calcula `earliest_ready_at`; el POS no lo inventa.
    - Se ejecuta en la MISMA transacción del guardado del ticket.
  Errores:
    - 404 si el ticket no existe.
    - 409 si el ticket no es de tipo PEDIDO.
```

### 8.3 Contrato nuevo — leer el pedido de un ticket

```
CONTRATO pedidos.pedido_del_ticket
  Consumidor:   POS (checkout y ticket impreso)
  Proveedor:    Pedidos
  Operación:    GET /orders/by-ticket/{ticket_id}
  Entrada:      ticket_id (UUID)
  Salida:       {
                  order_id:          UUID,
                  delivery_type:     String,
                  status:            String,
                  customer_name:     String NULL,
                  customer_phone:    String NULL,
                  committed_at:      DateTime(timezone=True) NULL,
                  packaging_type:    String,
                  delivery_address:  Text NULL,
                  delivery_fee:      Numeric(12,2) NULL,
                  notes:             Text NULL
                }
  Garantías:
    - Devuelve el pedido asociado al ticket, o 404 si no tiene.
    - Devuelve una PROYECCIÓN, no la fila completa (cumple O-23).
  Errores:
    - 404 si el ticket no existe o no tiene pedido.
```

### 8.4 Lo que el POS NO hace con Pedidos

| Acción | Por qué NO |
|--------|-----------|
| `from modules.orders.models import Order` | El POS no importa modelos de Pedidos. |
| `db.add(Order(...))` | El POS no escribe la tabla `orders`. |
| Calcular `earliest_ready_at` | Es regla de negocio de Pedidos (tiempos de producción). |
| Cambiar el `status` del pedido a estados de Producción/Pickup/Reparto | Esos 14 estados los gobierna Pedidos, no el POS. |
| Leer `orders` para las suites de Producción | Esas suites son de Pedidos; el POS no las alimenta. |

**Anclaje.** Hoy el puente vive en
[`pos/service.py:306`](../../apps/api/modules/pos/service.py:306) y la tabla en
[`orders/models.py:12`](../../apps/api/modules/orders/models.py:12). El POS nuevo mueve ese
puente al módulo Pedidos y lo consume por endpoint.

---

## SECCIÓN 9 — CONTRATO DE VISIÓN (Visión → POS)

### 9.1 Acoplamiento actual

```
Hoy el POS llama a su propio motor de visión:
  POSService.predict_vision()  →  apps/api/modules/pos/service.py:854
  VisionScanner.jsx            →  apps/pos/VisionScanner.jsx:103
```

**Problema.** El motor de visión vive **dentro** del módulo POS. Eso mezcla dos
responsabilidades: vender y reconocer imágenes. Si Visión crece (más modelos, más cámaras),
arrastra al POS.

### 9.2 Contrato nuevo

```
CONTRATO vision.reconocer_producto
  Consumidor:   POS (pantalla de venta con cámara)
  Proveedor:    Visión
  Operación:    POST /vision/predict
  Entrada:      { frame_base64: String, channel: String, top_k: Integer = 3 }
  Salida:       {
                  detecciones: [
                    { sku, nombre, confianza: Float(0..1), bbox: {x,y,w,h} }
                  ],
                  modelo_version: String
                }
  Garantías:
    - Devuelve hasta `top_k` candidatos ordenados por confianza.
    - Si no reconoce nada, devuelve lista vacía (200), no error.
    - El POS decide qué hacer con la detección: Visión solo sugiere.
  Errores:
    - 400 si `frame_base64` está vacío o excede 5 MB.
    - 503 si el modelo no está cargado.
```

**Nota.** Este contrato **saca** el motor de visión del POS. El POS queda como
**consumidor** de Visión, igual que es consumidor de Almacenes o Seguridad.

---

## SECCIÓN 10 — MATRIZ DE CONTRATOS

| # | Contrato | Consumidor | Proveedor | Reemplaza a | Estado hoy |
|---|----------|-----------|-----------|-------------|------------|
| 1 | `catalogo.productos_para_venta` | POS | Catálogo | Lectura de `products` | Ya existe (parcial) |
| 2 | `almacenes.consumir_por_venta` | POS | Almacenes | Escritura de `products.stock` | **Deuda** |
| 3 | `almacenes.disponibilidad` | POS | Almacenes | Lectura de `products.stock` | **Deuda** |
| 4 | `produccion.disponible_para_vender` | POS | Producción | Lectura de `doughs` | **Deuda** |
| 5 | `pos.eventos_auditables` | Auditoría | POS | (ya existe) | Cicatriz |
| 6 | `pos.resumen_de_venta` | Estadísticas | POS | Lectura de `tickets` | **Deuda** |
| 7 | `seguridad.identidad_del_empleado` | POS | Seguridad | Lectura de `employees` | **Deuda** |
| 8 | `seguridad.validar_pin` | POS | Seguridad | Lectura de `employees` | **Deuda** |
| 9 | `caja.sesion_activa` | POS | Caja | (ya existe) | **Ya existe** |
| 10 | `caja.abrir_turno` | POS | Caja | (ya existe) | **Ya existe** |
| 11 | `caja.registrar_movimiento` | POS | Caja | (ya existe) | **Ya existe** |
| 12 | `caja.resumen_del_turno` | POS | Caja | (ya existe) | **Ya existe** |
| 13 | `caja.cerrar_turno` | POS | Caja | (ya existe) | **Ya existe** |
| 14 | `caja.reporte_diario` | POS / Estadísticas | Caja | (ya existe) | **Ya existe** |
| 15 | `pedidos.registrar_desde_ticket` | POS | Pedidos | Escritura directa de `orders` | **Deuda** |
| 16 | `pedidos.pedido_del_ticket` | POS | Pedidos | Lectura de `orders` | **Deuda** |
| 17 | `vision.reconocer_producto` | POS | Visión | Motor interno del POS | **Deuda** |

**Lectura de la matriz.** De 17 contratos: **1 es cicatriz** (se conserva), **1 ya existe
parcial** (catálogo, se formaliza), **6 ya existen completos** (Caja, se documentan como
referencia) y **9 son deuda** (se corrigen en el POS nuevo).

**Lo notable:** el módulo de Caja es el **único módulo que ya cumple la Regla de Oro #5 al
100%**. El POS nunca lee sus tablas. Es la prueba de que el patrón funciona y el modelo a
imitar por Almacenes, Producción, Pedidos, Seguridad y Visión.

**El caso de Pedidos es distinto al de Caja.** Pedidos **existe y funciona** (tabla, servicio,
6 endpoints, interfaz), pero su frontera está **mal trazada**: el POS importa la clase `Order`
y escribe la tabla. Por eso aparece como **deuda** y no como "ya existe": el módulo está, lo
que falta es que el POS deje de escribir en él.

---

## SECCIÓN 11 — OBSERVACIONES

| # | Observación | Acción |
|---|-------------|--------|
| **O-19** | El contrato §2.2 usa `evento_id` generado por el POS. Es el patrón Outbox (A-04). | Implementar la tabla `warehouse_events` en el POS nuevo (ya está en el Documento 8, §5). |
| **O-20** | El contrato §5.2 corrige el bug D-28 (límites locales vs UTC). | Documentar la zona horaria en la firma del contrato (ya hecho). |
| **O-21** | El contrato §6.3 (validar PIN) es el único que expone un secreto en tránsito. | Exigir HTTPS y rate-limit obligatorio. |
| **O-22** | El contrato §8.2 saca Visión del POS. | Mover `VisionScanner.jsx` y `service.predict_vision` al módulo Visión. |
| **O-23** | Ningún contrato devuelve tablas completas: todos devuelven proyecciones. | Mantener esta regla: si un contrato devuelve `SELECT *`, está mal diseñado. |
| **O-24** | **El contrato de Caja YA EXISTE en el código** (ver §7). No es un pendiente: es un contrato completo y bien hecho. | Documentarlo como referencia (hecho en §7) y usarlo como modelo para los demás módulos. |
| **O-25** | El cobro se enlaza a la caja por `tickets.cash_session_id`, no por una llamada por cobro. | Conservar este diseño: mantiene el cobro en una sola transacción. |
| **O-26** | Caja es el único módulo que ya cumple la Regla de Oro #5 al 100%. | Usarlo como caso de referencia al migrar Almacenes, Producción, Seguridad y Visión. |
| **O-27** | **El levantamiento de pedidos YA EXISTE** (tabla `orders`, 6 endpoints, `ProgramacionPedidoModal.jsx`), pero el POS **escribe la tabla `orders` directamente** (ver §8.1). | Mover el puente `_sync_order_from_ticket` al módulo Pedidos y consumirlo por `pedidos.registrar_desde_ticket`. |
| **O-28** | El puente POS → Pedidos es **automático y transaccional** (se dispara al guardar el ticket PEDIDO). Eso es correcto y se conserva. | Mantener la automaticidad, pero por contrato: el POS pide, Pedidos escribe. |
| **O-29** | El ciclo de **14 estados** del pedido (Producción → Pickup → Reparto → Final) lo gobierna Pedidos, no el POS. | El POS nunca cambia el `status` a estados de producción; solo registra y consulta. |

---

## SECCIÓN 12 — CIERRE

**Lo que este documento deja claro:**

1. El POS nuevo **no lee ni escribe** tablas de otros módulos. Pide por contrato.
2. Los 9 acoplamientos de deuda quedan identificados con su sustituto exacto.
3. El único contrato donde el POS es proveedor es el de auditoría y el de estadísticas:
   el POS expone **resúmenes**, nunca tablas.
4. El patrón Outbox (`evento_id`) garantiza que el descuento de stock sea idempotente.

**Lo que sigue.** El [`CRITERIOS_DE_ACEPTACION_DEL_NUEVO_POS.md`](./CRITERIOS_DE_ACEPTACION_DEL_NUEVO_POS.md)
define **cuándo se considera terminada** la construcción del POS nuevo: qué pruebas deben
pasar, qué contratos deben estar vivos y qué deuda debe estar a cero.
