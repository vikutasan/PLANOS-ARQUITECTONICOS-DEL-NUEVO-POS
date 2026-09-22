# ESPECIFICACIÓN FUNCIONAL DEL MÓDULO POS
## Ingeniería Inversa — El "Qué", no el "Cómo"

> **Documento:** Especificación funcional obtenida por ingeniería inversa del código actual del módulo POS.
> **Autor:** Analista de Sistemas Senior (rol asumido).
> **Fecha:** 2026-09-21.
> **Alcance:** Módulo POS del ERP R de Rico (v7.0.3 / v22), incluyendo sus acoplamientos con Caja, Almacenes, Pedidos, Catálogo y Visión.
> **Regla dura:** Este documento NO modifica el ERP en producción. Es un artefacto de análisis.
> **Anclaje original:** commit `fe9f6ed` (tag `v22-estable-fe9f6ed`).
> **Re-anclaje:** 22 Sep 2026 — commit `5802f45` (V23). Ver §A.4.

---

## §A.4 — NOTA DE RE-ANCLAJE (22 Sep 2026)

> **Por qué existe esta nota.** Este documento se escribió leyendo el POS en el commit
> `fe9f6ed`. Desde entonces el ERP avanzó a `5802f45` (V23). El **comportamiento del POS
> no cambió**, pero **dos cosas sí**, y quien reconstruya el Nuevo POS debe saberlo.

### A.4.1 Qué cambió entre `fe9f6ed` y `5802f45` que toca al POS

| Cambio | Versión | Qué hizo | Efecto sobre este documento |
|--------|---------|----------|-----------------------------|
| **Operaciones atómicas por ítem** | V22 | Cada operación de ítem (agregar/actualizar/eliminar) es transaccional y valida `version` | **Ya estaba documentado** (RN-17 a RN-30, F-12 a F-14). Sin cambio. |
| **Dinero `Float` → `Numeric(12,2)`** | V23 | 36 columnas de dinero migradas en todo el ERP, incluidas `tickets.total`, `ticket_items.unit_price`, `ticket_items.subtotal` | **Este documento ya declaraba `Numeric(12,2)`** ([§D.1.2](:299), [§D.1.3](:315)) porque se leyó el modelo, no la BD. Ahora **la BD coincide con el modelo**. Deuda saldada. |
| **`business_currency` sembrado** | V23 | Nueva clave en `system_settings` + `GET /settings/currency` | El POS **consume** configuración (AC-06). El contrato de Configuración ahora expone moneda. Ver [`ESPECIFICACION_FUNCIONAL_VISTA_GENERAL.md`](ESPECIFICACION_FUNCIONAL_VISTA_GENERAL.md:1). |

### A.4.2 Qué NO cambió

- Las **81 reglas** (RN-01 a RN-81) siguen vigentes sin excepción.
- Los **29 hallazgos** (5 deudas + 10 acoplamientos + 10 debilidades + 4 riesgos) siguen vigentes.
- Las **33 funcionalidades** (F-01 a F-33) siguen vigentes.
- El **offset hardcodeado** de [`cash/service.py:189`](../../apps/api/modules/cash/service.py:189) (DB-04) **sigue ahí**: V23 no lo tocó. RN-81 sigue violada en Caja.

### A.4.3 La consecuencia práctica

El criterio de aceptación **G6** del [`PLAN_ACCION_ARQUITECTONICO_NUEVO_POS.md`](PLAN_ACCION_ARQUITECTONICO_NUEVO_POS.md:231)
decía *"HEAD `fe9f6ed`"*. **Era falso desde V19.** Se corrigió a `5802f45` en el mismo
commit que esta nota. La regla dura **no cambia**: el ERP no se toca; solo se actualiza
la referencia de qué versión se leyó.

---

## SECCIÓN A — PROPÓSITO Y MÉTODO

### A.1 Propósito

Extraer, desde el código en ejecución, **qué hace** el POS (funcionalidades, reglas de negocio, datos que entran/procesan/guardan) y **dónde duele** (deudas, acoplamientos innecesarios, debilidades de diseño, riesgos de concurrencia). El objetivo es producir la base funcional sobre la que se diseñará el **nuevo POS** para futuras sucursales, copiando el **comportamiento** y no la **deuda**.

### A.2 Método

1. **Lectura del código** (fuente de verdad): `apps/api/modules/pos/{service,models,router,occupancy,schemas,pos_audit}.py`, `apps/api/modules/cash/{service,models}.py`, `apps/api/modules/warehouse/models.py`, `apps/api/modules/orders/models.py`, `apps/api/modules/catalog/models.py`, `apps/api/modules/heladeria/models.py`, `apps/api/core/{timestamps,timezone,serialization}.py`, y la suite de pruebas `apps/api/tests/test_pos_*.py`.
2. **Abstracción**: se ignoró la estructura del código (clases, nombres de métodos, capas) y se describió el **comportamiento observable**: qué entra, qué se valida, qué se persiste, qué se emite.
3. **Extracción de reglas**: cada condicional, guarda, umbral, formato y transición de estado se tradujo a una regla de negocio numerada (RN-xx).
4. **Detección de hallazgos**: se marcaron deudas conocidas, acoplamientos directos a tablas de otros módulos, debilidades de diseño y riesgos de concurrencia.
5. **Trazabilidad**: cada funcionalidad se liga a sus reglas y a las entidades que toca.

### A.3 Convenciones

- **RN-xx**: Regla de Negocio.
- **F-xx**: Funcionalidad.
- **AC-xx**: Acoplamiento innecesario (Acoplamiento).
- **DB-xx**: Debilidad de diseño (Design Weakness).
- **RC-xx**: Riesgo de concurrencia (Race Condition).
- **DEUDA-xx**: Deuda técnica conocida.
- Los timestamps se almacenan en **UTC naive** y se muestran en la zona de negocio (`America/Mexico_City`, UTC-6).

---

## SECCIÓN B — FUNCIONALIDADES ACTUALES (33)

### Grupo 1 — Sesión de terminal y ocupación

| ID | Funcionalidad | Descripción funcional |
|----|---------------|-----------------------|
| **F-01** | Abrir sesión de terminal | Registra una sesión de trabajo por terminal (caja abierta) con operador y fondo inicial. |
| **F-02** | Consultar sesión activa | Devuelve la sesión abierta de una terminal, si existe. |
| **F-03** | Bloquear terminal (lock) | Un operador toma posesión exclusiva de una terminal por un TTL. |
| **F-04** | Liberar terminal (unlock) | El poseedor libera la terminal; solo el dueño puede hacerlo. |
| **F-05** | Forzar desbloqueo | Un administrador libera una terminal ocupada por otro. |
| **F-06** | Heartbeat de terminal | Renueva el TTL del candado mientras el operador sigue activo. |
| **F-07** | Estado de terminales | Reporta ocupación, sesión de caja y salud de cada terminal. |

### Grupo 2 — Tickets: creación y reserva

| ID | Funcionalidad | Descripción funcional |
|----|---------------|-----------------------|
| **F-08** | Crear ticket (checkout completo) | Crea o actualiza un ticket con sus líneas y total en una sola operación. |
| **F-09** | Reservar ticket | Obtiene un ticket vacío (DRAFT) para una terminal, reutilizando uno existente si aplica. |
| **F-10** | Generar folio consecutivo | Asigna un folio visible con formato `V####` por sesión. |
| **F-11** | Recuperar ticket por cuenta | Busca un ticket por su número de cuenta (folio visible). |

### Grupo 3 — Tickets: operaciones atómicas por ítem (v22)

| ID | Funcionalidad | Descripción funcional |
|----|---------------|-----------------------|
| **F-12** | Agregar ítem | Agrega o incrementa una línea del ticket y recalcula el total. |
| **F-13** | Actualizar cantidad | Cambia la cantidad de una línea y recalcula el total. |
| **F-14** | Eliminar ítem | Quita una línea del ticket y recalcula el total. |

### Grupo 4 — Consultas de tickets

| ID | Funcionalidad | Descripción funcional |
|----|---------------|-----------------------|
| **F-15** | Listar tickets abiertos | Devuelve los tickets no cerrados (cuentas abiertas). |
| **F-16** | Listar tickets con filtros | Filtra por terminal, estado, texto y fecha local. |
| **F-17** | Borradores por terminal | Lista los DRAFT de una terminal. |
| **F-18** | Reporte de borradores por expirar | Reporta DRAFT con ítems próximos a expirar. |

### Grupo 5 — Ciclo de vida y sincronización

| ID | Funcionalidad | Descripción funcional |
|----|---------------|-----------------------|
| **F-19** | Sincronizar pedido desde ticket | Proyecta el ticket hacia el módulo de Pedidos cuando corresponde. |
| **F-20** | Guardado de emergencia | Persiste el carrito ante cierre inminente del navegador. |
| **F-21** | Limpieza de borradores obsoletos | Cancela DRAFT vacíos o vencidos para liberar terminales. |

### Grupo 6 — Caja (cortes)

| ID | Funcionalidad | Descripción funcional |
|----|---------------|-----------------------|
| **F-22** | Abrir sesión de caja | Abre una sesión de caja con fondo inicial. |
| **F-23** | Registrar movimiento de caja | Registra entradas/salidas de efectivo con concepto. |
| **F-24** | Eliminar movimiento de caja | Borra un movimiento registrado. |
| **F-25** | Calcular resumen de caja | Calcula esperado vs. contado y clasifica pagos. |
| **F-26** | Cerrar sesión de caja | Cierra la sesión con conteo físico y desglose. |
| **F-27** | Historial de sesiones | Lista sesiones de caja por terminal y fecha. |
| **F-28** | Reporte diario | Genera el reporte de ventas del día local. |

### Grupo 7 — Inventario (eventos)

| ID | Funcionalidad | Descripción funcional |
|----|---------------|-----------------------|
| **F-29** | Emitir evento de almacén | Emite un `WarehouseEvent` para que Almacenes procese el consumo. |

### Grupo 8 — Visión

| ID | Funcionalidad | Descripción funcional |
|----|---------------|-----------------------|
| **F-30** | Predicción por visión | Identifica productos por imagen (motor ORB). |
| **F-31** | Carga de imágenes de entrenamiento | Sube imágenes etiquetadas por SKU. |

### Grupo 9 — Auditoría

| ID | Funcionalidad | Descripción funcional |
|----|---------------|-----------------------|
| **F-32** | Auditoría de operaciones POS | Registra cada escritura POS en un log de auditoría. |
| **F-33** | Consulta de auditoría | Lee el log de auditoría por terminal y rango de fechas. |

---

## SECCIÓN C — REGLAS DE NEGOCIO (81)

### C.1 Sesión de terminal y ocupación (RN-01 a RN-08)

- **RN-01** — Una terminal solo puede tener **una** sesión de trabajo abierta a la vez.
- **RN-02** — Una sesión de terminal se identifica por `terminal_id` y operador (`employee_id`).
- **RN-03** — Un candado de terminal (`TerminalLock`) es **exclusivo**: solo un ocupante a la vez.
- **RN-04** — El candado tiene un **TTL** (por defecto 15 minutos); al vencer, se considera libre.
- **RN-05** — Solo el **dueño** del candado puede liberarlo (`unlock`); otro recibe 403.
- **RN-06** — Un **administrador** puede forzar el desbloqueo de una terminal ajena.
- **RN-07** — El **heartbeat** renueva el TTL del candado del dueño.
- **RN-08** — Los candados vencidos se purgan antes de reportar el estado de terminales.

### C.2 Tickets: identidad y folio (RN-09 a RN-16)

- **RN-09** — Un ticket se identifica internamente por `id` (PK) y externamente por `account_num` (folio visible).
- **RN-10** — El folio visible tiene formato `V####` (V + consecutivo), generado por sesión.
- **RN-11** — El folio es **local a la sucursal**; no es un identificador global.
- **RN-12** — El `terminal_id` de un ticket **nunca se sobrescribe** una vez asignado.
- **RN-13** — Un ticket pertenece a un **canal** (`channel`), por defecto `PANADERIA`.
- **RN-14** — Un ticket tiene un `status` con ciclo de vida definido (DRAFT, OPEN, PAID, CANCELLED, etc.).
- **RN-15** — Un ticket tiene un `version` (entero) para control de concurrencia optimista.
- **RN-16** — El `total` del ticket es la suma de los subtotales de sus líneas.

### C.3 Tickets: líneas (RN-17 a RN-24)

- **RN-17** — Un producto aparece **una sola vez** por ticket; agregarlo de nuevo incrementa la cantidad.
- **RN-18** — El `unit_price` de la línea se **congela** al momento de agregarla.
- **RN-19** — El `subtotal` de la línea es `unit_price × quantity`.
- **RN-20** — La cantidad de una línea es un entero positivo.
- **RN-21** — No se puede agregar un producto **inexistente** (404).
- **RN-22** — No se puede agregar un producto **inactivo** (400).
- **RN-23** — No se puede modificar un ticket en estado `PAID` (400).
- **RN-24** — No se puede operar sobre un ticket de una sesión **inactiva** (400).

### C.4 Concurrencia optimista (RN-25 a RN-30)

- **RN-25** — Toda operación de escritura sobre un ticket valida el `version` recibido.
- **RN-26** — Si el `version` recibido es **obsoleto**, se responde 409 (conflicto).
- **RN-27** — Cada escritura exitosa **incrementa** el `version` en 1.
- **RN-28** — El `version` protege contra escrituras concurrentes de dos terminales.
- **RN-29** — Tras una escritura atómica, se expira la caché de sesión (`db.expire_all()`) para releer el estado real.
- **RN-30** — La reserva de ticket usa `skip_locked` para evitar bloquear a otras terminales.

### C.5 DRAFT GUARD (RN-31 a RN-36)

- **RN-31** — Un ticket en estado `DRAFT` pertenece a la terminal que lo creó.
- **RN-32** — Otra terminal **no puede** escribir sobre un DRAFT ajeno (400).
- **RN-33** — La misma terminal **sí puede** continuar su propio DRAFT.
- **RN-34** — Un DRAFT sin ítems puede reutilizarse para una nueva venta.
- **RN-35** — Un DRAFT con ítems no se reutiliza; se crea uno nuevo.
- **RN-36** — La guarda de DRAFT se evalúa **antes** de aplicar cambios.

### C.6 Anti-degradación de líneas (RN-37 a RN-40)

- **RN-37** — Al sincronizar líneas, no se permite una reducción mayor al **50%** del total de líneas.
- **RN-38** — La anti-degradación protege contra pérdida accidental de líneas por payloads incompletos.
- **RN-39** — Si la reducción supera el umbral, la operación se rechaza.
- **RN-40** — La sincronización de líneas es **idempotente** para el mismo conjunto.

### C.7 Reserva y limpieza de borradores (RN-41 a RN-48)

- **RN-41** — La reserva busca primero un DRAFT vacío reutilizable de la terminal.
- **RN-42** — Un DRAFT vacío solo se reutiliza si fue creado dentro de los últimos **5 minutos**.
- **RN-43** — Un DRAFT vacío más antiguo se descarta y se crea uno nuevo.
- **RN-44** — La limpieza de borradores obsoletos está **throttled** (máximo 1 vez por minuto).
- **RN-45** — Un DRAFT vacío con más de **1 hora** se considera obsoleto.
- **RN-46** — Un DRAFT con ítems tiene un **TTL** de expiración.
- **RN-47** — Al expirar, el DRAFT pasa a `CANCELLED`.
- **RN-48** — La limpieza libera la terminal para nuevas ventas.

### C.8 Caja: sesión y movimientos (RN-49 a RN-56)

- **RN-49** — Solo puede existir **una** sesión de caja `OPEN` por terminal.
- **RN-50** — Una sesión de caja registra `opening_float` (fondo inicial).
- **RN-51** — Los movimientos de caja son `ENTRADA` o `SALIDA` con monto y concepto.
- **RN-52** — Un movimiento puede eliminarse mientras la sesión esté abierta.
- **RN-53** — El efectivo esperado = fondo inicial + entradas − salidas + ventas en efectivo.
- **RN-54** — Al cerrar, se registra el conteo físico (`physical_cash`), crédito y débito.
- **RN-55** — Una sesión cerrada es inmutable.
- **RN-56** — El historial de sesiones se filtra por terminal y fecha local.

### C.9 Caja: clasificación de pagos (RN-57 a RN-60)

- **RN-57** — Los pagos se clasifican por método (efectivo, crédito, débito, transferencia).
- **RN-58** — La clasificación alimenta el resumen y el reporte diario.
- **RN-59** — El reporte diario usa el **día local** de negocio, no el día UTC.
- **RN-60** — El resumen distingue efectivo esperado de efectivo contado.

### C.10 Inventario: eventos (RN-61 a RN-66)

- **RN-61** — El inventario es un **libro mayor inmutable** (`movimientos_inventario`); nunca se hace `UPDATE stock`.
- **RN-62** — El POS **no** descuenta stock directamente; emite un `WarehouseEvent`.
- **RN-63** — El `WarehouseEvent` se inserta **antes** del commit de la transacción POS.
- **RN-64** — Si la emisión del evento falla, el POS **no** falla (try/except pass).
- **RN-65** — El `WarehouseEvent` tiene estado `PENDIENTE`, `PROCESADO` o `FALLIDO`.
- **RN-66** — Un movimiento de inventario se liga a su evento por `evento_id`; el índice único `(evento_id, item_id)` garantiza idempotencia.

### C.11 Pedidos: proyección (RN-67 a RN-70)

- **RN-67** — Un ticket puede proyectarse a un `Order` (pedido) cuando corresponde.
- **RN-68** — La relación ticket↔pedido es 1:1 (`ticket_id` único).
- **RN-69** — El pedido tiene 14 estados de ciclo de vida.
- **RN-70** — El pedido registra tipo de entrega, empaque y datos de reparto (lat/lng/distancia/costo).

### C.12 Visión (RN-71 a RN-74)

- **RN-71** — La predicción por visión usa el motor ORB existente.
- **RN-72** — Una detección se acepta si su confianza supera el umbral **0.35**.
- **RN-73** — Las imágenes de entrenamiento se etiquetan por SKU.
- **RN-74** — La visión es **asistiva**: no bloquea la venta manual.

### C.13 Auditoría (RN-75 a RN-77)

- **RN-75** — Cada escritura POS se registra en un log de auditoría (`pos_audit.log`).
- **RN-76** — El registro incluye endpoint, payload, código de respuesta y extras.
- **RN-77** — La auditoría se consulta por terminal y rango de fechas.

### C.14 Tiempo y zona horaria (RN-78 a RN-80)

- **RN-78** — Todos los timestamps se almacenan en **UTC naive** (`utcnow()`).
- **RN-79** — La conversión a hora local usa la zona de negocio (`get_business_tz`).
- **RN-80** — Los límites del día local se calculan con `local_day_bounds_utc`.

### C.15 Regla transversal (RN-81)

- **RN-81** — **Ninguna** capa debe hardcodear un offset de zona horaria; siempre debe usar la utilidad de zona de negocio.

### C.16 Resumen cuantitativo

| Categoría | Rango | Total |
|-----------|-------|-------|
| Sesión de terminal y ocupación | RN-01 – RN-08 | 8 |
| Tickets: identidad y folio | RN-09 – RN-16 | 8 |
| Tickets: líneas | RN-17 – RN-24 | 8 |
| Concurrencia optimista | RN-25 – RN-30 | 6 |
| DRAFT GUARD | RN-31 – RN-36 | 6 |
| Anti-degradación de líneas | RN-37 – RN-40 | 4 |
| Reserva y limpieza de borradores | RN-41 – RN-48 | 8 |
| Caja: sesión y movimientos | RN-49 – RN-56 | 8 |
| Caja: clasificación de pagos | RN-57 – RN-60 | 4 |
| Inventario: eventos | RN-61 – RN-66 | 6 |
| Pedidos: proyección | RN-67 – RN-70 | 4 |
| Visión | RN-71 – RN-74 | 4 |
| Auditoría | RN-75 – RN-77 | 3 |
| Tiempo y zona horaria | RN-78 – RN-80 | 3 |
| Regla transversal | RN-81 | 1 |
| **TOTAL** | | **81** |

---

## SECCIÓN D — MODELO DE DATOS (QUÉ ENTRA, SE PROCESA Y SE GUARDA)

### D.1 Entidades propias del POS

#### D.1.1 `TerminalSession` (sesión de trabajo)

| Campo | Tipo | Nulo | Descripción |
|-------|------|------|-------------|
| `id` | Integer PK | No | Identificador |
| `terminal_id` | String | No | Identificador de terminal |
| `employee_id` | Integer FK | Sí | Operador (`employees.id`) |
| `employee_name` | String | Sí | Nombre del operador |
| `status` | String | No | `OPEN` / `CLOSED` |
| `opened_at` | DateTime UTC | No | Apertura |
| `closed_at` | DateTime UTC | Sí | Cierre |

**Invariante:** una sola sesión `OPEN` por `terminal_id`.

#### D.1.2 `Ticket` (ticket / cuenta)

| Campo | Tipo | Nulo | Descripción |
|-------|------|------|-------------|
| `id` | Integer PK | No | Identificador |
| `account_num` | String | No | Folio visible (`V####`) |
| `session_id` | Integer FK | Sí | Sesión de terminal |
| `terminal_id` | String | Sí | Terminal (nunca se sobrescribe) |
| `channel` | String | No | Canal (por defecto `PANADERIA`) |
| `status` | String | No | Estado del ciclo de vida |
| `total` | Numeric(12,2) | No | Total del ticket |
| `version` | Integer | No | Control de concurrencia optimista |
| `captured_by_id` | Integer FK | Sí | Operador que capturó |
| `created_at` | DateTime UTC | No | Creación |
| `updated_at` | DateTime UTC | Sí | Última modificación |

**Invariante:** `PAID` es inmutable.

#### D.1.3 `TicketItem` (línea de ticket)

| Campo | Tipo | Nulo | Descripción |
|-------|------|------|-------------|
| `id` | Integer PK | No | Identificador |
| `ticket_id` | Integer FK | Sí | → `tickets.id` |
| `product_id` | Integer FK | Sí | → `products.id` |
| `quantity` | Integer | No | Cantidad |
| `unit_price` | Numeric(12,2) | No | Precio congelado |
| `subtotal` | Numeric(12,2) | No | `unit_price × quantity` |

**Invariante:** un producto aparece una sola vez por ticket (se incrementa cantidad).

#### D.1.4 `TerminalLock` (candado de terminal)

| Campo | Tipo | Nulo | Descripción |
|-------|------|------|-------------|
| `id` | Integer PK | No | Identificador |
| `terminal_id` | String | No | Terminal bloqueada (única) |
| `occupier_id` | Integer | No | Operador que posee el candado |
| `occupier_name` | String | Sí | Nombre del ocupante |
| `locked_at` | DateTime UTC | No | Momento del bloqueo |
| `expires_at` | DateTime UTC | No | Vencimiento del TTL |

**Invariante:** un solo candado por `terminal_id`; vencido el TTL, se considera libre.

#### D.1.5 `CashSession` (sesión de caja)

| Campo | Tipo | Nulo | Descripción |
|-------|------|------|-------------|
| `id` | Integer PK | No | Identificador |
| `terminal_id` | String | No | Terminal |
| `employee_id` | Integer FK | Sí | Operador |
| `employee_name` | String | Sí | Nombre del operador |
| `opening_float` | Numeric(12,2) | No | Fondo inicial |
| `status` | String | No | `OPEN` / `CLOSED` |
| `opened_at` | DateTime UTC | No | Apertura |
| `closed_at` | DateTime UTC | Sí | Cierre |
| `physical_cash` | Numeric(12,2) | Sí | Efectivo contado |
| `credit_total` | Numeric(12,2) | Sí | Total crédito |
| `debit_total` | Numeric(12,2) | Sí | Total débito |

**Invariante:** una sola sesión `OPEN` por `terminal_id`.

#### D.1.6 `CashMovement` (movimiento de caja)

| Campo | Tipo | Nulo | Descripción |
|-------|------|------|-------------|
| `id` | Integer PK | No | Identificador |
| `session_id` | Integer FK | No | → `cash_sessions.id` |
| `movement_type` | String | No | `ENTRADA` / `SALIDA` |
| `amount` | Numeric(12,2) | No | Monto |
| `concept` | String | Sí | Concepto |

### D.2 Entidades de otros módulos que el POS toca

#### D.2.1 `WarehouseEvent` (evento de almacén)

| Campo | Tipo | Nulo | Descripción |
|-------|------|------|-------------|
| `id` | Integer PK | No | Identificador |
| `event_type` | String | No | Tipo de evento |
| `payload` | JSON | Sí | Datos del evento |
| `estado` | String | No | `PENDIENTE` / `PROCESADO` / `FALLIDO` |
| `created_at` | DateTime UTC | No | Creación |

#### D.2.2 `MovimientoInventario` (libro mayor inmutable)

| Campo | Tipo | Nulo | Descripción |
|-------|------|------|-------------|
| `id` | Integer PK | No | Identificador |
| `evento_id` | Integer FK | Sí | → `warehouse_events.id` |
| `item_id` | Integer FK | No | Insumo/producto |
| `cantidad` | Numeric | No | Cantidad (con signo) |
| `tipo` | String | No | Tipo de movimiento |
| `created_at` | DateTime UTC | No | Creación |

**Índice único:** `(evento_id, item_id)` → idempotencia.

#### D.2.3 `Order` (pedido)

| Campo | Tipo | Nulo | Descripción |
|-------|------|------|-------------|
| `id` | Integer PK | No | Identificador |
| `ticket_id` | Integer FK | No | → `tickets.id` (único) |
| `status` | String | No | Uno de 14 estados |
| `delivery_type` | String | Sí | Tipo de entrega |
| `packaging_type` | String | Sí | Tipo de empaque |
| `delivery_lat` | Float | Sí | Latitud |
| `delivery_lng` | Float | Sí | Longitud |
| `delivery_distance_km` | Float | Sí | Distancia |
| `delivery_fee` | Numeric | Sí | Costo de envío |

### D.3 Diagrama conceptual de relaciones

```
TerminalSession 1 ──── * Ticket 1 ──── * TicketItem
        │                  │
        │                  └──── 1 Order (proyección)
        │
        └──── 1 TerminalLock (por terminal)

CashSession 1 ──── * CashMovement
     │
     └──── * Ticket (pagos del periodo)

Ticket ──(emite)──> WarehouseEvent ──(procesa)──> MovimientoInventario
```

### D.4 Qué entra, qué se procesa, qué se guarda

| Operación | Entra | Se procesa | Se guarda |
|-----------|-------|------------|-----------|
| Crear ticket | Líneas, terminal, canal, operador | Validación de productos, cálculo de total, folio | `Ticket` + `TicketItem` (+ `WarehouseEvent`) |
| Agregar ítem | `account_num`, `product_id`, `quantity`, `version` | DRAFT GUARD, validación de producto, recálculo | `TicketItem` + `Ticket` (total, version) |
| Actualizar cantidad | `account_num`, `product_id`, `quantity`, `version` | Validación de versión, recálculo | `TicketItem` + `Ticket` |
| Eliminar ítem | `account_num`, `product_id`, `version` | Validación de existencia, recálculo | `TicketItem` (borrado) + `Ticket` |
| Reservar ticket | `terminal_id`, operador, canal | Búsqueda de DRAFT vacío, folio | `Ticket` (DRAFT) |
| Abrir caja | `terminal_id`, operador, fondo | Validación de sesión única | `CashSession` |
| Movimiento de caja | `session_id`, tipo, monto, concepto | Validación de sesión abierta | `CashMovement` |
| Cerrar caja | `session_id`, conteo físico | Cálculo de esperado vs. contado | `CashSession` (cierre) |
| Guardado de emergencia | Carrito, `account_num`, `terminal_id` | Idempotencia, fallback de sesión | `Ticket` + `TicketItem` |
| Predicción por visión | Imagen, terminal | Motor ORB, umbral 0.35 | (sin persistencia de venta) |

---

## SECCIÓN E — FLUJOS FUNCIONALES

### E.1 Flujo: Venta directa (mostrador)

1. El operador **abre sesión de terminal** (F-01) y **toma el candado** (F-03).
2. El sistema **reserva un ticket** (F-09): reutiliza un DRAFT vacío reciente o crea uno nuevo con folio `V####` (F-10).
3. El operador **agrega ítems** (F-12): cada ítem valida producto activo (RN-21, RN-22), congela precio (RN-18) y recalcula total (RN-16).
4. Cada escritura valida `version` (RN-25, RN-26) e incrementa `version` (RN-27).
5. Al cobrar, el ticket pasa a `PAID` (RN-14) y se vuelve **inmutable** (RN-23).
6. Se **emite un `WarehouseEvent`** (F-29) para que Almacenes procese el consumo (RN-62, RN-63).
7. El operador **libera el candado** (F-04) o el TTL vence (RN-04).

### E.2 Flujo: Pedido (con proyección a Order)

1. Igual que E.1 hasta el paso 3.
2. El ticket se **proyecta a un `Order`** (F-19) con tipo de entrega y datos de reparto (RN-67, RN-70).
3. La relación ticket↔pedido es 1:1 (RN-68).
4. El pedido avanza por sus 14 estados (RN-69).
5. El `WarehouseEvent` se emite igual que en E.1.

### E.3 Flujo: Recuperación de cuenta

1. El operador **busca el ticket por cuenta** (F-11) usando el folio visible.
2. El sistema devuelve el ticket con sus líneas.
3. El operador continúa la venta (agregar/actualizar/eliminar ítems).
4. Se aplican las mismas guardas (DRAFT GUARD, versión, estado).

### E.4 Flujo: Guardado de emergencia

1. El navegador detecta cierre inminente (`beforeunload`).
2. Se envía el carrito con `account_num` y `terminal_id` (F-20).
3. El backend es **idempotente**: si el ticket ya existe, no duplica (RN-40).
4. Si no hay sesión activa, se usa un **fallback de sesión**.
5. Si el payload está vacío, se ignora.
6. Ante error interno, se responde `FAILED` sin bloquear el cierre.

### E.5 Flujo: Corte de caja

1. El operador **abre sesión de caja** (F-22) con fondo inicial (RN-50).
2. Durante el turno, registra **movimientos** (F-23) de entrada/salida (RN-51).
3. Al cerrar, el sistema **calcula el resumen** (F-25): efectivo esperado vs. contado (RN-53, RN-60).
4. Se **clasifican los pagos** por método (RN-57).
5. Se **cierra la sesión** (F-26) registrando conteo físico, crédito y débito (RN-54).
6. La sesión cerrada es **inmutable** (RN-55).
7. El **reporte diario** (F-28) usa el día local de negocio (RN-59).

### E.6 Flujo: Ocupación de terminal

1. El operador **consulta el estado de terminales** (F-07).
2. Si la terminal está libre, **toma el candado** (F-03) con TTL (RN-04).
3. Mientras trabaja, envía **heartbeats** (F-06) para renovar el TTL (RN-07).
4. Si otro intenta tomar la terminal, recibe rechazo (RN-03).
5. Si el dueño no libera (F-04), un administrador puede **forzar el desbloqueo** (F-05, RN-06).
6. Los candados vencidos se purgan (RN-08).

---

## SECCIÓN F — HALLAZGOS (29)

### F.1 Deudas técnicas conocidas (5)

| ID | Deuda | Evidencia | Impacto |
|----|-------|-----------|---------|
| **DEUDA-01** | El POS escribe directamente en tablas de otros módulos | `service.py` toca `Product`, `Order`, `WarehouseEvent` | Acoplamiento fuerte; impide extraer el POS como servicio |
| **DEUDA-02** | El POS conoce el esquema de Almacenes | Inserta `WarehouseEvent` con payload propio | Cambios en Almacenes rompen el POS |
| **DEUDA-03** | El POS conoce el esquema de Pedidos | Crea/actualiza `Order` directamente | Cambios en Pedidos rompen el POS |
| **DEUDA-04** | La emisión de eventos usa `try/except pass` | `create_ticket` (43-54) | Fallos silenciosos; inventario puede desincronizarse |
| **DEUDA-05** | La limpieza de borradores se ejecuta en el request del usuario | `_cleanup_stale_empty_tickets` | Latencia acoplada al tráfico; no es un job dedicado |

### F.2 Acoplamientos innecesarios (10)

| ID | Acoplamiento | Ubicación | Por qué es innecesario |
|----|--------------|-----------|------------------------|
| **AC-01** | POS → `products` (lectura directa) | `_get_items_and_total` | Debería usar un contrato de Catálogo |
| **AC-02** | POS → `orders` (escritura directa) | `_sync_order_from_ticket` | Debería emitir un evento de Pedidos |
| **AC-03** | POS → `warehouse_events` (escritura directa) | `create_ticket` | Debería usar un bus de eventos |
| **AC-04** | POS → `employees` (lectura directa) | `captured_by_id` | Debería usar un contrato de RRHH |
| **AC-05** | POS → `cash_sessions` (lectura directa) | `get_terminals_status` | Debería consultar el contrato de Caja |
| **AC-06** | POS → `settings` (lectura directa) | Configuración de terminales | Debería usar un contrato de Configuración |
| **AC-07** | Caja → `tickets` (lectura directa) | `_obtener_tickets_pagados` | Debería usar el contrato del POS |
| **AC-08** | POS → `heladeria_product_config` (lectura) | Configuración de heladería | Debería usar el contrato de Heladería |
| **AC-09** | POS → archivo `pos_audit.log` (I/O directo) | `pos_audit.py` | Debería usar el contrato de Auditoría |
| **AC-10** | POS → `product_technical_sheet` (lectura) | Visión/heladería | Debería usar el contrato de Catálogo |

### F.3 Debilidades de diseño (10)

| ID | Debilidad | Evidencia | Consecuencia |
|----|-----------|-----------|--------------|
| **DB-01** | El POS mezcla orquestación y persistencia en un solo servicio | `POSService` (1000+ líneas) | Difícil de testear y de extraer |
| **DB-02** | El folio visible se genera por sesión, no global | `_generate_consecutive_ticket` | Colisiones si se comparte BD entre sucursales |
| **DB-03** | El `channel` está hardcodeado por defecto | `"PANADERIA"` | Nuevos canales requieren cambios de código |
| **DB-04** | Offset de zona horaria hardcodeado (+6h) | `cash/service.py:189-190` | Viola RN-81; rompe en horario de verano |
| **DB-05** | La limpieza de borradores usa throttle en memoria | `_last_gc_time` | No funciona con múltiples workers |
| **DB-06** | El guardado de emergencia acepta `dict` sin esquema | `emergency_save_ticket` | Validación débil; datos sucios |
| **DB-07** | La anti-degradación usa un umbral mágico (50%) | `_sync_ticket_items` | Regla no documentada ni configurable |
| **DB-08** | El TTL del candado está duplicado en varias capas | `occupancy.py` y `router.py` | Riesgo de inconsistencias |
| **DB-09** | La auditoría escribe a archivo, no a BD | `pos_audit.py` | No consultable, no transaccional |
| **DB-10** | El umbral de visión (0.35) está hardcodeado | `predict_vision` | No ajustable por sucursal |

### F.4 Riesgos de concurrencia (4)

| ID | Riesgo | Escenario | Mitigación actual |
|----|--------|-----------|-------------------|
| **RC-01** | Dos terminales escriben el mismo ticket | Ambas pasan la validación de versión a la vez | Concurrencia optimista (`version`) — mitiga, no elimina |
| **RC-02** | Dos terminales reservan el mismo DRAFT vacío | `_find_empty_ticket` con `skip_locked` | `skip_locked` reduce, pero no garantiza unicidad lógica |
| **RC-03** | El candado vence mientras el operador trabaja | TTL expira por red lenta | Heartbeat — depende de la red |
| **RC-04** | El `WarehouseEvent` se emite pero el commit falla | Evento huérfano | `try/except pass` — no hay compensación |

---

## SECCIÓN G — TRAZABILIDAD

### G.1 Funcionalidad → Reglas → Entidades

| Funcionalidad | Reglas | Entidades |
|---------------|--------|-----------|
| F-01 Abrir sesión de terminal | RN-01, RN-02 | `TerminalSession` |
| F-02 Consultar sesión activa | RN-01 | `TerminalSession` |
| F-03 Bloquear terminal | RN-03, RN-04 | `TerminalLock` |
| F-04 Liberar terminal | RN-05 | `TerminalLock` |
| F-05 Forzar desbloqueo | RN-06 | `TerminalLock` |
| F-06 Heartbeat | RN-07 | `TerminalLock` |
| F-07 Estado de terminales | RN-08 | `TerminalLock`, `TerminalSession`, `CashSession` |
| F-08 Crear ticket | RN-09–RN-16, RN-25–RN-30 | `Ticket`, `TicketItem`, `WarehouseEvent` |
| F-09 Reservar ticket | RN-41–RN-43 | `Ticket` |
| F-10 Generar folio | RN-10, RN-11 | `Ticket` |
| F-11 Recuperar por cuenta | RN-09 | `Ticket` |
| F-12 Agregar ítem | RN-17–RN-24, RN-25–RN-30, RN-31–RN-36 | `Ticket`, `TicketItem` |
| F-13 Actualizar cantidad | RN-19, RN-20, RN-25–RN-30 | `Ticket`, `TicketItem` |
| F-14 Eliminar ítem | RN-16, RN-25–RN-30 | `Ticket`, `TicketItem` |
| F-15 Listar abiertos | RN-14 | `Ticket` |
| F-16 Listar con filtros | RN-14, RN-78–RN-80 | `Ticket` |
| F-17 Borradores por terminal | RN-31, RN-34 | `Ticket` |
| F-18 Reporte de expiración | RN-45, RN-46 | `Ticket` |
| F-19 Sincronizar pedido | RN-67–RN-70 | `Order` |
| F-20 Guardado de emergencia | RN-40 | `Ticket`, `TicketItem` |
| F-21 Limpieza de borradores | RN-44–RN-48 | `Ticket` |
| F-22 Abrir caja | RN-49, RN-50 | `CashSession` |
| F-23 Movimiento de caja | RN-51 | `CashMovement` |
| F-24 Eliminar movimiento | RN-52 | `CashMovement` |
| F-25 Resumen de caja | RN-53, RN-60 | `CashSession`, `Ticket` |
| F-26 Cerrar caja | RN-54, RN-55 | `CashSession` |
| F-27 Historial de sesiones | RN-56 | `CashSession` |
| F-28 Reporte diario | RN-57–RN-59 | `CashSession`, `Ticket` |
| F-29 Emitir evento de almacén | RN-61–RN-66 | `WarehouseEvent`, `MovimientoInventario` |
| F-30 Predicción por visión | RN-71, RN-72, RN-74 | (sin persistencia) |
| F-31 Carga de entrenamiento | RN-73 | (archivos) |
| F-32 Auditoría de operaciones | RN-75, RN-76 | `pos_audit.log` |
| F-33 Consulta de auditoría | RN-77 | `pos_audit.log` |

### G.2 Cobertura de pruebas observada

| Suite | Cobertura funcional |
|-------|---------------------|
| `test_pos_atomic_ops.py` | 14 pruebas: operaciones atómicas por ítem, versión, DRAFT, estados |
| `test_pos_checkout.py` | 7 pruebas: checkout, DRAFT GUARD, proyección a Order, eventos |
| `test_pos_emergency_save.py` | 6 pruebas: guardado de emergencia, idempotencia, fallback |
| `test_bloque9d_3bugs.py` | 9 pruebas: zona horaria y día local |
| `architecture.test.js` | Guardas de arquitectura del frontend (v7.0.3, v18–v21) |

---

## SECCIÓN H — CONCLUSIONES

### H.1 Qué hace bien el POS actual

1. **Persistencia atómica por ítem** (v22): cada operación es transaccional y consistente.
2. **Concurrencia optimista**: el `version` protege contra escrituras simultáneas.
3. **DRAFT GUARD**: evita que una terminal pise el borrador de otra.
4. **Contrato de respuesta ligero**: `TicketLightResponse` reduce el tráfico.
5. **Idempotencia** en guardado de emergencia y en movimientos de inventario.
6. **Zona horaria centralizada** en la mayoría de las capas (con excepciones).
7. **Auditoría** de cada escritura POS.
8. **Cobertura de pruebas** significativa en los flujos críticos.

### H.2 Qué hace mal el POS actual

1. **Acoplamiento directo a tablas de otros módulos** (10 acoplamientos).
2. **Servicio monolítico** de 1000+ líneas que mezcla orquestación y persistencia.
3. **Fallos silenciosos** en la emisión de eventos (`try/except pass`).
4. **Offset de zona horaria hardcodeado** en Caja (viola RN-81).
5. **Folio local** que no escala a múltiples sucursales con BD compartida.
6. **Limpieza de borradores** acoplada al request del usuario.
7. **Auditoría a archivo**, no consultable ni transaccional.
8. **Umbrales mágicos** (50%, 0.35) no configurables.

### H.3 Recomendaciones para el nuevo POS (10)

1. **Extraer el POS como módulo con contratos explícitos**: exponer interfaces, no tablas.
2. **Emitir eventos en lugar de escribir en tablas ajenas**: bus de eventos con outbox transaccional.
3. **Separar orquestación de persistencia**: casos de uso + repositorios.
4. **Eliminar todo offset hardcodeado**: usar siempre la utilidad de zona de negocio.
5. **Folio global con UUID**: el folio visible es solo presentación local.
6. **Job dedicado de limpieza**: no acoplado al tráfico de usuarios.
7. **Auditoría en BD**: consultable, transaccional, con índices.
8. **Umbrales configurables por sucursal**: anti-degradación, visión, TTL.
9. **Outbox transaccional**: el evento se persiste en la misma transacción que el ticket.
10. **Contratos versionados**: cada módulo expone su contrato con versión explícita.

### H.4 Declaración de regla dura

> **REGLA DURA:** Este documento es un artefacto de análisis. **No se ha modificado ni se modificará el ERP en producción.** Todo trabajo derivado (el nuevo POS) se realizará en un proyecto y repositorio separados. El ERP actual permanece intacto y operando; su HEAD es `5802f45` (V23) al 22 Sep 2026. La ingeniería inversa se hizo sobre `fe9f6ed` (tag `v22-estable-fe9f6ed`); ver §A.4 para los cambios posteriores que tocan al POS.

---

*Fin del documento — Especificación Funcional del Módulo POS (Ingeniería Inversa).*
