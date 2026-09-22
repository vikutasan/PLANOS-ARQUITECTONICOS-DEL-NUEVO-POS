# MODELO DE DATOS DEL NUEVO POS

**Documento 8 — El cimiento del edificio nuevo**

> **Propósito de este documento.** El [`PLANO ARQUITECTONICO PARA EL NUEVO POS.md`](../PLANO%20ARQUITECTONICO%20PARA%20EL%20NUEVO%20POS.md)
> dice **qué debe ser** el nuevo POS. La [`ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md`](./ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md)
> dice **qué hace hoy**. Este documento dice **cómo se guardan los datos** en el POS nuevo:
> las tablas, las columnas, los tipos, los índices y las reglas de integridad.
>
> **Regla de oro de este documento.** El esquema nuevo **conserva el comportamiento** del
> POS actual (que es la fuente de verdad funcional) pero **corrige su estructura**: donde
> hoy hay un entero autoincremental, mañana habrá un UUID; donde hoy hay un `DateTime`
> naive, mañana habrá UTC explícito; donde hoy hay un `UPDATE stock`, mañana habrá un
> asiento en el ledger.
>
> **Anclaje.** Este documento está anclado al commit `5802f45` (V23) del ERP actual. La
> ingeniería inversa se hizo sobre `fe9f6ed` (tag `v22-estable-fe9f6ed`). Cada tabla del
> esquema actual se cita con su `archivo:línea`.

---

## SECCIÓN 0 — CÓMO LEER ESTE DOCUMENTO

Cada tabla del POS nuevo se presenta con esta estructura:

```
TABLA <nombre_nuevo>
  Origen:      <tabla actual>  →  <archivo:línea>
  Propósito:   <qué guarda y por qué>
  Columnas:    <nombre> <tipo> <restricciones>  ← <nota>
  Índices:     <los que hacen falta para que las consultas del POS no escaneen>
  Cambios:     <qué cambia respecto al esquema actual y por qué>
```

**Los 4 cambios estructurales que aplican a TODAS las tablas:**

| # | Cambio | Por qué |
|---|--------|---------|
| **C-01** | **PK entero → UUID** | El folio `V####` es local y se repite entre sucursales. La identidad real debe ser global. |
| **C-02** | **`DateTime` naive → `DateTime(timezone=True)` en UTC** | Hoy se guarda UTC sin tzinfo (ver [`core/timestamps.py`](../../apps/api/core/timestamps.py:21)). Mañana el tipo lo declara. |
| **C-03** | **`Numeric(12,2)` se conserva** | El dinero nunca se guarda en `Float`. Esto ya está bien hoy y no se toca. La moneda del negocio (`business_currency`) **declara** en qué moneda se captura; **no convierte**. La presentación pasa por un solo formateador (ver CA-21 del Documento 10). |
| **C-04** | **`version` (bloqueo optimista) se conserva y se generaliza** | Hoy solo `tickets` y `stock_almacen` lo tienen. Mañana toda tabla que el POS escriba en paralelo lo lleva. |

---

## SECCIÓN 1 — EL NÚCLEO TRANSACCIONAL (POS)

### 1.1 TABLA `terminal_sessions`

```
Origen:      terminal_sessions  →  apps/api/modules/pos/models.py:17
Propósito:   El turno de una terminal. Agrupa los tickets de una jornada de trabajo.
Columnas:
  id              UUID PK
  terminal_id     String NOT NULL INDEX
  opened_at       DateTime(timezone=True) NOT NULL DEFAULT utcnow
  closed_at       DateTime(timezone=True) NULL
  is_active       Boolean NOT NULL DEFAULT true
Índices:
  ix_terminal_sessions_terminal_active  (terminal_id, is_active)
Cambios:
  C-01, C-02. Se conserva la estructura: es correcta.
```

### 1.2 TABLA `tickets` — la tabla central del POS

```
Origen:      tickets  →  apps/api/modules/pos/models.py:28
Propósito:   El ticket de venta. Es la entidad que el POS crea, modifica y cobra.
Columnas:
  id                  UUID PK
  account_num         String UNIQUE NOT NULL INDEX   ← folio local V#### (NO es identidad)
  total               Numeric(12,2) NOT NULL DEFAULT 0
  payment_details     JSON NULL                      ← lista de pagos mixtos
  created_at          DateTime(timezone=True) NOT NULL DEFAULT utcnow
  status              String NOT NULL DEFAULT 'OPEN' ← OPEN | PAID | CANCELLED
  version             Integer NOT NULL DEFAULT 1     ← bloqueo optimista (C-04)
  order_type          String NOT NULL DEFAULT 'VENTA_DIRECTA'
  order_status        String NOT NULL DEFAULT 'PROGRAMADO PARA SER PREPARADO'
  delivery_type       String NULL                    ← PICKUP | DOMICILIO
  customer_name       String NULL
  customer_phone      String NULL
  committed_at        DateTime(timezone=True) NULL
  packaging_type      String NULL                    ← PROPIO | VENTA
  delivery_address    Text NULL
  order_notes         Text NULL
  terminal_id         String NULL INDEX
  channel             String NOT NULL DEFAULT 'PANADERIA' INDEX ← PANADERIA | HELADERIA
  customer_group_name String NULL
  session_id          UUID FK terminal_sessions.id
  cash_session_id     UUID FK cash_sessions.id NULL
  captured_by_id      UUID FK employees.id NULL      ← auditoría: quién capturó
  cashed_by_id        UUID FK employees.id NULL      ← auditoría: quién cobró
Índices:
  uq_tickets_account_num               (account_num) UNIQUE
  ix_tickets_terminal_status           (terminal_id, status)
  ix_tickets_channel_created           (channel, created_at)
  ix_tickets_cash_session              (cash_session_id)
Cambios:
  C-01, C-02, C-04. Se conserva la separación de canal (channel) y la auditoría
  capturó/cobró: son cicatrices valiosas, no deuda.
```

### 1.3 TABLA `ticket_items`

```
Origen:      ticket_items  →  apps/api/modules/pos/models.py:72
Propósito:   Las líneas del ticket. Cada producto agregado al carrito.
Columnas:
  id          UUID PK
  ticket_id   UUID FK tickets.id NOT NULL INDEX
  product_id  UUID FK products.id NOT NULL
  quantity    Integer NOT NULL DEFAULT 1
  unit_price  Numeric(12,2) NOT NULL
  subtotal    Numeric(12,2) NOT NULL
Índices:
  ix_ticket_items_ticket               (ticket_id)
  ix_ticket_items_product              (product_id)
Cambios:
  C-01. Se conserva la estructura. Nota: `unit_price` se congela al momento de la
  venta (no se recalcula desde el producto): eso es correcto y se mantiene.
```

### 1.4 TABLA `terminal_locks`

```
Origen:      terminal_locks  →  apps/api/modules/pos/models.py:7
Propósito:   Candado persistente de terminal. Reemplaza el diccionario en RAM que se
             perdía con reinicios.
Columnas:
  id              UUID PK
  terminal_id     String UNIQUE NOT NULL INDEX
  occupier_id     UUID FK employees.id NOT NULL
  occupier_name   String NOT NULL
  locked_at       DateTime(timezone=True) NOT NULL DEFAULT utcnow
Índices:
  uq_terminal_locks_terminal_id        (terminal_id) UNIQUE
Cambios:
  C-01, C-02. Se conserva: es una cicatriz (el candado en RAM era el bug).
```

---

## SECCIÓN 2 — LA CAJA (CASH)

### 2.1 TABLA `cash_sessions`

```
Origen:      cash_sessions  →  apps/api/modules/cash/models.py:7
Propósito:   El turno de un cajero. Se abre con fondo inicial y se cierra al final.
Columnas:
  id              UUID PK
  terminal_id     String NOT NULL INDEX
  employee_id     UUID FK employees.id NOT NULL
  employee_name   String NOT NULL                    ← desnormalizado para reportes
  opening_float   Numeric(12,2) NOT NULL DEFAULT 0
  status          String NOT NULL DEFAULT 'OPEN'     ← OPEN | CLOSED
  opened_at       DateTime(timezone=True) NOT NULL DEFAULT utcnow
  closed_at       DateTime(timezone=True) NULL
  physical_cash   Numeric(12,2) NULL                 ← conteo físico al cierre
  physical_credit Numeric(12,2) NULL
  physical_debit  Numeric(12,2) NULL
Índices:
  ix_cash_sessions_terminal_status     (terminal_id, status)
Cambios:
  C-01, C-02. Se conserva la desnormalización de `employee_name`: es una decisión
  consciente para reportes rápidos, no un accidente.
```

### 2.2 TABLA `cash_movements`

```
Origen:      cash_movements  →  apps/api/modules/cash/models.py:32
Propósito:   Entrada o salida de dinero durante un turno (propinas, refuerzos).
Columnas:
  id              UUID PK
  cash_session_id UUID FK cash_sessions.id NOT NULL INDEX
  movement_type   String NOT NULL                    ← ENTRADA | SALIDA
  amount          Numeric(12,2) NOT NULL
  concept         String NOT NULL
  created_at      DateTime(timezone=True) NOT NULL DEFAULT utcnow
Índices:
  ix_cash_movements_session            (cash_session_id)
Cambios:
  C-01, C-02. Se conserva.
```

---

## SECCIÓN 3 — EL CATÁLOGO (CATALOG)

### 3.1 TABLA `categories`

```
Origen:      categories  →  apps/api/modules/catalog/models.py:5
Propósito:   Categoría de producto. Define el destino de proyección hacia los POS.
Columnas:
  id                    UUID PK
  name                  String UNIQUE NOT NULL INDEX
  icon                  String NULL
  position              Integer NULL
  vision_enabled        Boolean NOT NULL DEFAULT false
  is_system             Boolean NOT NULL DEFAULT false
  heladeria_enabled     Boolean NOT NULL DEFAULT false
  pos_target            String NOT NULL DEFAULT 'PANADERIA' ← PANADERIA | HELADERIA | AMBOS
  heladeria_default_role String NULL                        ← SABOR | RECIPIENTE | EXTRA | ...
Índices:
  uq_categories_name                   (name) UNIQUE
  ix_categories_pos_target             (pos_target)
Cambios:
  C-01. Se conserva `pos_target` y `heladeria_default_role`: son la única decisión que
  el usuario toma por categoría, y el producto la hereda. Eso es diseño, no deuda.
```

### 3.2 TABLA `products`

```
Origen:      products  →  apps/api/modules/catalog/models.py:42
Propósito:   El producto vendible. Es la entidad que el POS muestra en la grilla.
Columnas:
  id            UUID PK
  sku           String UNIQUE NOT NULL INDEX
  barcode       String UNIQUE NULL INDEX
  name          String NOT NULL INDEX
  price         Numeric(12,2) NOT NULL
  cost          Numeric(12,2) NOT NULL DEFAULT 0
  warehouse     String NULL                          ← NO usar como ubicación real
  image_url     String NULL
  position      Integer NULL
  nature        String NOT NULL DEFAULT 'MANUFACTURADO' ← MANUFACTURADO | PREPARADO | REVENTA
  category_id   UUID FK categories.id
  active        Boolean NOT NULL DEFAULT true
Índices:
  uq_products_sku                      (sku) UNIQUE
  uq_products_barcode                  (barcode) UNIQUE
  ix_products_category_active          (category_id, active)
Cambios:
  C-01. **SE ELIMINA la columna `products.stock`.** Hoy existe
  (apps/api/modules/catalog/models.py:66) pero está marcada OBSOLETA: la fuente única
  de stock es `stock_almacen`. Mantenerla es la deuda D-STOCK (doble fuente de verdad).
  En el POS nuevo **no existe**: el stock se consulta por contrato al módulo Almacenes.
```

### 3.3 TABLA `product_technical_sheets`

```
Origen:      product_technical_sheets  →  apps/api/modules/catalog/models.py:82
Propósito:   Ficha técnica del producto (masas, horneado, tiempos, BOM).
Columnas:
  id                    UUID PK
  product_id            UUID FK products.id UNIQUE NOT NULL
  primary_mass_id       UUID FK doughs.id NULL
  primary_mass_grams    Float NULL
  secondary_mass_id     UUID FK doughs.id NULL
  secondary_mass_grams  Float NULL
  tertiary_mass_id      UUID FK doughs.id NULL
  tertiary_mass_grams   Float NULL
  weight_per_piece      Float NULL
  baking_temp_top       Float NULL
  baking_temp_bottom    Float NULL
  baking_time_min       Integer NULL
  steam_seconds         Integer NULL
  scoring_type          String NULL
  forming_procedure     Text NULL
  bom_extra             JSON NULL
  preparation_time_min  Integer NULL
  order_lead_time_hours Integer NULL                 ← lead time para Pedidos
  recipe_procedure      Text NULL
  modifiers             JSON NULL
  provider              String NULL
  original_barcode      String NULL
  unit_measure          String NULL
  min_stock             Integer NULL
  max_stock             Integer NULL
Índices:
  uq_pts_product_id                    (product_id) UNIQUE
Cambios:
  C-01. Se conserva. Nota: `order_lead_time_hours` es la regla que calcula
  `earliest_ready_at` en los pedidos; es una regla de negocio, no un campo decorativo.
```

---

## SECCIÓN 4 — LOS PEDIDOS (ORDERS)

### 4.1 TABLA `orders`

```
Origen:      orders  →  apps/api/modules/orders/models.py:12
Propósito:   Un pedido es una venta diferida con fecha de entrega compromiso.
             Se vincula 1:1 con un ticket del POS.
Columnas:
  id                    UUID PK
  ticket_id             UUID FK tickets.id UNIQUE NOT NULL
  delivery_type         String NOT NULL DEFAULT 'PICKUP'  ← PICKUP | DOMICILIO
  status                String NOT NULL DEFAULT 'TENTATIVO'
  customer_name         String NULL
  customer_phone        String NULL
  earliest_ready_at     DateTime(timezone=True) NULL      ← calculada por el sistema
  committed_at          DateTime(timezone=True) NULL      ← confirmada con el cliente
  packaging_type        String NOT NULL DEFAULT 'PROPIO'  ← PROPIO | VENTA
  delivery_address      Text NULL
  delivery_lat          Float NULL
  delivery_lng          Float NULL
  delivery_distance_km  Float NULL
  delivery_fee          Float NULL DEFAULT 0.0
  created_at            DateTime(timezone=True) NOT NULL DEFAULT utcnow
  updated_at            DateTime(timezone=True) NOT NULL DEFAULT utcnow
  notes                 Text NULL
Índices:
  uq_orders_ticket_id                  (ticket_id) UNIQUE
  ix_orders_status_committed           (status, committed_at)
Cambios:
  C-01, C-02. Se conserva el ciclo de vida de 14 estados (documentado en
  apps/api/modules/orders/models.py:25). Es una regla de negocio central.
```

---

## SECCIÓN 5 — EL ALMACÉN (WAREHOUSE) — el ledger inmutable

### 5.1 TABLA `almacenes`

```
Origen:      almacenes  →  apps/api/modules/warehouse/models.py:54
Propósito:   Un almacén físico (seco, refrigerado, congelado).
Columnas:
  id              UUID PK
  nombre          String NOT NULL
  zona_termica    String NOT NULL                    ← SECO | REFRIGERADO | CONGELADO
  proposito       String NOT NULL                    ← ALMACENAMIENTO | EXHIBICION_VENTA | EQUIPAMIENTO
  sucursal_id     UUID NULL
  foto_url        String NULL
  planograma_url  String NULL
  pautas_acomodo  JSON NOT NULL DEFAULT []
  activo          Boolean NOT NULL DEFAULT true
  created_at      DateTime(timezone=True) NOT NULL DEFAULT utcnow
Índices:
  ix_almacenes_sucursal_activo         (sucursal_id, activo)
Cambios:
  C-01, C-02. Se conserva.
```

### 5.2 TABLA `stock_almacen` — el saldo (con bloqueo optimista)

```
Origen:      stock_almacen  →  apps/api/modules/warehouse/models.py:74
Propósito:   El saldo actual de un ítem en un almacén. Es un CACHE del ledger.
Columnas:
  id                    UUID PK
  almacen_id            UUID FK almacenes.id NOT NULL
  item_id               String NOT NULL INDEX        ← SKU
  item_type             String NOT NULL              ← PRODUCTO | INSUMO
  cantidad_actual       Float NOT NULL DEFAULT 0.0
  stock_minimo          Float NOT NULL DEFAULT 0.0
  stock_maximo          Float NOT NULL DEFAULT 0.0
  fecha_ingreso         DateTime(timezone=True) NOT NULL DEFAULT utcnow
  dias_anaquel_alerta   Integer NULL
  version               Integer NOT NULL DEFAULT 1   ← bloqueo optimista (C-04)
  ultima_actualizacion  DateTime(timezone=True) NOT NULL DEFAULT utcnow
Índices:
  uq_stock_almacen_almacen_item        (almacen_id, item_id) UNIQUE
  ix_stock_almacen_item                (item_id)
Cambios:
  C-01, C-02. **Regla dura:** esta tabla es un CACHE derivado del ledger
  `movimientos_inventario`. Nunca se escribe directamente sin un asiento en el ledger.
```

### 5.3 TABLA `movimientos_inventario` — el ledger inmutable

```
Origen:      movimientos_inventario  →  apps/api/modules/warehouse/models.py:90
Propósito:   El asiento contable del inventario. Cada entrada/salida es una fila.
             NUNCA se hace UPDATE de stock: se inserta un movimiento.
Columnas:
  id                  UUID PK
  almacen_origen_id   UUID NULL
  almacen_destino_id  UUID NULL
  item_id             String NOT NULL INDEX
  item_type           String NOT NULL
  cantidad            Float NOT NULL
  tipo_movimiento     String NOT NULL                ← ENTRADA_COMPRA | SALIDA_VENTA | MERMA...
  metodo_captura      String NOT NULL
  usuario_id          UUID NOT NULL
  notas               String NULL
  lote_entrada_id     UUID NULL
  evento_id           Integer NULL                   ← idempotencia del Outbox
  timestamp           DateTime(timezone=True) NOT NULL DEFAULT utcnow
Índices:
  ix_mov_inv_item_timestamp            (item_id, timestamp)
  uq_movimiento_evento_item            (evento_id, item_id) UNIQUE  ← anti-doble-descuento
Cambios:
  C-01, C-02. Se conserva el índice único `uq_movimiento_evento_item`: es la cicatriz
  que impide descontar el mismo SKU dos veces si el evento se reprocesa.
```

### 5.4 TABLA `warehouse_events` — el Outbox transaccional

```
Origen:      warehouse_events  →  apps/api/modules/warehouse/models.py:110
Propósito:   El evento que el POS emite al cobrar, para que Almacenes descuente stock
             de forma asíncrona y confiable. Es el patrón Outbox.
Columnas:
  id           UUID PK
  ticket_id    UUID UNIQUE NOT NULL INDEX
  items_json   JSON NOT NULL
  estado       String NOT NULL DEFAULT 'PENDIENTE'   ← PENDIENTE | PROCESADO | FALLIDO
  intentos     Integer NOT NULL DEFAULT 0
  error_log    String NULL
  sucursal_id  UUID NULL INDEX
  created_at   DateTime(timezone=True) NOT NULL DEFAULT utcnow
Índices:
  uq_warehouse_events_ticket_id        (ticket_id) UNIQUE
  ix_warehouse_events_estado           (estado)
Cambios:
  C-01, C-02. Se conserva. Este es el mecanismo que elimina el `try/except pass` de la
  ruta crítica (Regla de Oro #7).
```

### 5.5 TABLA `warehouse_eventos_sin_almacen` — el diagnóstico

```
Origen:      warehouse_eventos_sin_almacen  →  apps/api/modules/warehouse/models.py:125
Propósito:   Registra los SKUs que NO se pudieron descontar, para que el operador
             corrija la configuración. Reemplaza el "ignorar en silencio".
Columnas:
  id          UUID PK
  evento_id   UUID NULL INDEX
  ticket_id   UUID NULL
  sku         String NULL INDEX
  cantidad    Float NULL
  motivo      String NOT NULL                        ← SIN_SKU | SIN_STOCK_SUFICIENTE | SIN_ALMACEN_VENTA
  detalle     String NULL
  created_at  DateTime(timezone=True) NOT NULL DEFAULT utcnow
Índices:
  ix_we_sin_almacen_evento             (evento_id)
  ix_we_sin_almacen_sku                (sku)
Cambios:
  C-01, C-02. Se conserva: es una cicatriz (el silencio era el bug).
```

---

## SECCIÓN 6 — LA HELADERÍA (HELADERIA) — el producto compuesto

### 6.1 TABLA `heladeria_product_config`

```
Origen:      heladeria_product_config  →  apps/api/modules/heladeria/models.py:11
Propósito:   Extiende un producto del catálogo con configuración de heladería.
             Ejemplo: 'Chocolate' → component_type='SABOR'.
Columnas:
  id              UUID PK
  product_id      UUID FK products.id UNIQUE NOT NULL
  component_type  String NOT NULL                    ← RECIPIENTE | SABOR | EXTRA | BEBIDA_BASE | TAMAÑO
  max_scoops      Integer NULL                       ← solo RECIPIENTE
  base_price      Numeric(12,2) NULL
  price_per_scoop Numeric(12,2) NULL
  is_available    Boolean NOT NULL DEFAULT true      ← toggle "AGOTAR SABOR"
  position        Integer NOT NULL DEFAULT 0
Índices:
  uq_heladeria_config_product          (product_id) UNIQUE
  ix_heladeria_config_type_available   (component_type, is_available)
Cambios:
  C-01. Se conserva. Nota: esta tabla es una PROYECCIÓN del catálogo (ver
  apps/api/modules/heladeria/sync.py), no una fuente de verdad paralela.
```

### 6.2 TABLA `ticket_item_components`

```
Origen:      ticket_item_components  →  apps/api/modules/heladeria/models.py:40
Propósito:   Los componentes de un TicketItem compuesto (un helado armado).
             Un helado = 1 TicketItem con N TicketItemComponents.
Columnas:
  id              UUID PK
  ticket_item_id  UUID FK ticket_items.id NOT NULL INDEX
  product_id      UUID FK products.id NULL
  component_type  String NOT NULL                    ← RECIPIENTE | BOLA_1 | BOLA_2 | BOLA_3 | EXTRA
  component_name  String NOT NULL
  unit_price      Numeric(12,2) NOT NULL DEFAULT 0
  quantity        Integer NOT NULL DEFAULT 1
Índices:
  ix_ticket_item_components_item       (ticket_item_id)
Cambios:
  C-01. Se conserva. Nota: `component_name` se desnormaliza a propósito: el ticket
  debe poder imprimirse aunque el producto se renombre después.
```

---

## SECCIÓN 7 — LAS TABLAS QUE EL POS NUEVO **NO** TENDRÁ

Estas tablas existen hoy y son **deuda** que el POS nuevo elimina:

| Tabla/columna actual | Por qué se elimina | Sustituto |
|----------------------|--------------------|-----------|
| `products.stock` | Doble fuente de verdad (D-STOCK) | Contrato con Almacenes → `stock_almacen` |
| `products.warehouse` | Hardcode de sucursal, prohibido por el principio SaaS | `stock_almacen.almacen_id` |
| Cualquier lectura directa a `employees` | Acoplamiento entre módulos | Contrato de Seguridad (Documento 9) |
| Cualquier lectura directa a `doughs` | Acoplamiento con Producción | Contrato de Producción (Documento 9) |

**Regla dura (Regla de Oro #5):** *El POS no lee tablas ajenas. Solo contratos.*

---

## SECCIÓN 8 — MATRIZ DE TRAZABILIDAD

| Tabla nueva | Tabla actual | Archivo:línea | Cambio principal |
|-------------|--------------|---------------|------------------|
| `terminal_sessions` | `terminal_sessions` | `pos/models.py:17` | UUID + UTC |
| `tickets` | `tickets` | `pos/models.py:28` | UUID + UTC |
| `ticket_items` | `ticket_items` | `pos/models.py:72` | UUID |
| `terminal_locks` | `terminal_locks` | `pos/models.py:7` | UUID + UTC |
| `cash_sessions` | `cash_sessions` | `cash/models.py:7` | UUID + UTC |
| `cash_movements` | `cash_movements` | `cash/models.py:32` | UUID + UTC |
| `categories` | `categories` | `catalog/models.py:5` | UUID |
| `products` | `products` | `catalog/models.py:42` | UUID, **sin `stock`** |
| `product_technical_sheets` | `product_technical_sheets` | `catalog/models.py:82` | UUID |
| `orders` | `orders` | `orders/models.py:12` | UUID + UTC |
| `almacenes` | `almacenes` | `warehouse/models.py:54` | UUID + UTC |
| `stock_almacen` | `stock_almacen` | `warehouse/models.py:74` | UUID + UTC |
| `movimientos_inventario` | `movimientos_inventario` | `warehouse/models.py:90` | UUID + UTC |
| `warehouse_events` | `warehouse_events` | `warehouse/models.py:110` | UUID + UTC |
| `warehouse_eventos_sin_almacen` | `warehouse_eventos_sin_almacen` | `warehouse/models.py:125` | UUID + UTC |
| `heladeria_product_config` | `heladeria_product_config` | `heladeria/models.py:11` | UUID |
| `ticket_item_components` | `ticket_item_components` | `heladeria/models.py:40` | UUID |

---

## SECCIÓN 9 — OBSERVACIONES (O-XX)

> Estas observaciones son notas para el POS nuevo. **No se aplican al POS actual.**

| # | Observación | Destino |
|---|-------------|---------|
| **O-13** | `products.stock` sigue existiendo en el ERP actual y es una trampa para el programador nuevo. | **Eliminar** en el POS nuevo |
| **O-14** | `products.warehouse` tiene un hardcode histórico de sucursal. | **Eliminar** en el POS nuevo |
| **O-15** | El folio `account_num` es `UNIQUE` global hoy, pero en un esquema multisucursal debe ser único **por sucursal**. | **Corregir**: `UNIQUE (sucursal_id, account_num)` |
| **O-16** | `orders.delivery_fee` es `Float`, no `Numeric`. El dinero nunca va en `Float`. | **Corregir** a `Numeric(12,2)` |
| **O-17** | `stock_almacen.cantidad_actual` es `Float`; para piezas enteras debería ser `Integer`, y para granel `Numeric`. | **Evaluar** por `item_type` |
| **O-18** | No hay tabla de `sucursales` en el esquema actual; `sucursal_id` es un `String` suelto. | **Crear** tabla `sucursales` y usar FK |
| **O-19** | No existe una configuración de moneda del negocio (`business_currency`). Hoy cada componente formatea el dinero por su cuenta (80 `toFixed(2)` sueltos en 12 componentes). | **Crear** `business_currency` en `system_settings` + un único `formatMoney` (ver CA-21 del Documento 10) |

---

## SECCIÓN 10 — CIERRE

Este documento es el **cimiento** del nuevo POS. Sin él, quien construya tendría que
inventar el esquema, y ahí es donde se cuelan las deudas que este proyecto busca evitar.

**Lo que este documento garantiza:**

1. **Toda tabla tiene origen trazable** a `archivo:línea` del ERP actual.
2. **Todo cambio estructural está justificado** (C-01 a C-04 + sección 7).
3. **Toda deuda conocida está marcada para eliminación** (sección 7 + O-13 a O-19).
4. **Toda cicatriz está marcada para conservación** (bloqueo optimista, Outbox,
   auditoría capturó/cobró, diagnóstico sin-almacén).

**El siguiente documento** ([`CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md`](./CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md))
define **cómo se comunican** estos datos entre módulos, para que el POS nunca vuelva a
leer una tabla ajena.
