# TOMO IV — CONTRATOS Y FRONTERAS

> **Documentación final del POS nuevo "R de Rico"** — Tomo IV de VII.
> **Fuente principal:** [`contracts/registry.py`](../../NUEVO-POS/apps/api/contracts/registry.py:1) (los 31 contratos) y [`PLAN_MAESTRO_DEFINITIVO_POS.md`](../PLAN_MAESTRO_DEFINITIVO_POS.md:873) §10.2.
> **Propósito de este tomo:** que una IA sin contexto entienda **qué es un contrato**, **por qué el POS no lee tablas ajenas**, y conozca **los 31 contratos** del POS con su firma, garantías y errores. Un contrato es una **operación**, nunca una tabla.

---

## ÍNDICE DEL TOMO IV

- [0. Cómo leer este tomo](#0-cómo-leer-este-tomo)
- [1. La frontera por contratos (A-02 / O-23)](#1-la-frontera-por-contratos-a-02--o-23)
- [2. La anatomía de un contrato](#2-la-anatomía-de-un-contrato)
- [3. Los 31 contratos (completos)](#3-los-31-contratos-completos)
- [4. El POS como consumidor vs. el POS como proveedor](#4-el-pos-como-consumidor-vs-el-pos-como-proveedor)
- [5. La regla de oro DT-07: un fallo ajeno nunca bloquea una venta](#5-la-regla-de-oro-dt-07-un-fallo-ajeno-nunca-bloquea-una-venta)
- [6. El patrón Outbox en los contratos](#6-el-patrón-outbox-en-los-contratos)
- [7. Las notas de frontera (CRM, Notificaciones, IA)](#7-las-notas-de-frontera-crm-notificaciones-ia)
- [8. El test de la puerta F2](#8-el-test-de-la-puerta-f2)
- [9. Matriz de trazabilidad: contrato → consumidor → proveedor → estado](#9-matriz-de-trazabilidad-contrato--consumidor--proveedor--estado)

---

## 0. CÓMO LEER ESTE TOMO

Cada contrato se presenta con **siete datos**:

| Campo | Qué es |
|-------|--------|
| **Número** | El índice del contrato en el registro (1 a 31). |
| **Nombre** | La operación en notación `proveedor.operacion`. |
| **Consumidor** | Quién pide (el POS, Auditoría, Estadísticas). |
| **Proveedor** | Quién responde (Catálogo, Caja, CRM, Centro de IA…). |
| **Operación** | El endpoint HTTP concreto. |
| **Entrada / Salida** | La firma: qué recibe y qué devuelve. |
| **Garantías / Errores** | Qué promete y qué puede fallar. |

**La unidad de frontera es contrato + proveedor.** El consumidor **pide por operación**; nunca lee la tabla del proveedor.

---

## 1. LA FRONTERA POR CONTRATOS (A-02 / O-23)

> **Fuente:** [`PLAN_MAESTRO_DEFINITIVO_POS.md`](../PLAN_MAESTRO_DEFINITIVO_POS.md:873) §10.2.

**A-02 — Frontera por contratos:** *el consumidor pide por OPERACIÓN, nunca lee la tabla del proveedor.*

**O-23 — Un contrato nunca expone una tabla:** el campo `tabla_expuesta` de todo `Contrato` es **siempre `None`**. Existe precisamente para poder **afirmarlo** en el test de la puerta.

**Los 3 principios de la frontera:**

| Principio | Enunciado |
|-----------|-----------|
| **P-01** | El **dueño** de una tabla es el único que la escribe. |
| **P-02** | El **consumidor** pide por **operación**, no por tabla. |
| **P-03** | El **contrato** es estable; la **tabla** es libre de cambiar. |

**¿Por qué?** Porque si el POS leyera la tabla `tickets` de Auditoría (o `employees` de Seguridad), cualquier cambio de esquema en el proveedor **rompería** al POS. El contrato **aísla**: el proveedor puede cambiar su tabla sin tocar el contrato.

**El anti-patrón que esto prohíbe:** leer tablas de otro módulo (anti-patrón #2 del Tomo III §9). Acopla módulos: el cambio de uno rompe al otro.

**La consecuencia práctica:** el POS expone un **RESUMEN** (una proyección), nunca su tabla `tickets` ni la fila cruda de `pos_audit_log`.

---

## 2. LA ANATOMÍA DE UN CONTRATO

> **Fuente:** [`contracts/registry.py`](../../NUEVO-POS/apps/api/contracts/registry.py:101).

```python
@dataclass(frozen=True)
class Contrato:
    numero: int
    nombre: str
    consumidor: str
    proveedor: str
    operacion: str
    entrada: dict[str, str]
    salida: dict[str, str]
    garantias: tuple[str, ...] = field(default_factory=tuple)
    errores: tuple[str, ...] = field(default_factory=tuple)
    estado_hoy: str = "Deuda"
    # Un contrato NUNCA expone una tabla. Este campo existe para poder
    # afirmarlo explícitamente en el test de la puerta.
    tabla_expuesta: None = None
```

**La firma legible** (`firma`): `proveedor.operacion(entrada) -> {salida}`.

**Los estados de un contrato** (`estado_hoy`):

| Estado | Significado |
|--------|-------------|
| **Ya existe** | El contrato está implementado y en uso. |
| **Ya existe (parcial)** | Implementado parcialmente. |
| **Implementado** | Implementado (usado por Auditoría). |
| **Cicatriz** | Nació de un bug; se conserva. |
| **Deuda** | Declarado pero aún no implementado (se porta en una fase futura). |
| **FASE X.Y** | Implementado en esa fase. |

---

## 3. LOS 31 CONTRATOS (COMPLETOS)

> **Fuente:** [`contracts/registry.py`](../../NUEVO-POS/apps/api/contracts/registry.py:131) — la tupla `CONTRATOS`.

### §1 Catálogo

#### Contrato 1 — `catalogo.productos_para_venta`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | Catálogo |
| **Operación** | `GET /catalog/products-for-sale` |
| **Entrada** | `channel: String`, `include_hidden: Boolean = false` |
| **Salida** | `productos: List[ProductoParaVenta]`, `categorias: List[CategoriaParaVenta]` |
| **Garantías** | Devuelve una PROYECCIÓN de venta, no la fila completa (O-23). Solo incluye productos visibles para el canal pedido. El precio ya viene resuelto por el proveedor. |
| **Errores** | 400 si el canal es inválido. |
| **Estado** | Ya existe (parcial) |

### §2 Almacenes

#### Contrato 2 — `almacenes.consumir_por_venta`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | Almacenes |
| **Operación** | `POST /warehouse/consume-for-sale` |
| **Entrada** | `evento_id: UUID`, `items: List[{item_id, item_type, cantidad: Numeric}]`, `ticket_id: UUID` |
| **Salida** | `aplicado: Boolean`, `movimientos: List[UUID]` |
| **Garantías** | Idempotente por `evento_id` (patrón Outbox, A-04 / O-19). El POS NO escribe `products.stock` ni `stock_almacen`. Se ejecuta en la MISMA transacción del guardado del ticket. |
| **Errores** | 409 si el `evento_id` ya fue aplicado con otro contenido. 422 si algún item no existe. |
| **Estado** | Deuda |

#### Contrato 3 — `almacenes.disponibilidad`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | Almacenes |
| **Operación** | `GET /warehouse/availability` |
| **Entrada** | `item_ids: List[String]`, `almacen_id: UUID NULL` |
| **Salida** | `disponibilidad: List[{item_id, item_type, cantidad_disponible: Numeric}]` |
| **Garantías** | Devuelve una PROYECCIÓN, no la tabla `stock_almacen` (O-23). El POS NO lee `products.stock` (columna eliminada en F1). |
| **Errores** | 400 si la lista de items está vacía. |
| **Estado** | Deuda |

### §3 Producción

#### Contrato 4 — `produccion.disponible_para_vender`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | Producción |
| **Operación** | `GET /production/available-to-sell` |
| **Entrada** | `fecha: Date (hora local)`, `channel: String` |
| **Salida** | `disponibles: List[{producto_id, cantidad: Numeric}]` |
| **Garantías** | Usa la fecha LOCAL, no UTC (corrige D-28 / O-20). El POS NO lee la tabla `doughs`. |
| **Errores** | 400 si el formato de fecha es inválido. |
| **Estado** | Deuda |

### §4 Auditoría (POS es PROVEEDOR)

#### Contrato 5 — `pos.eventos_auditables`

| Campo | Valor |
|-------|-------|
| **Consumidor** | Auditoría |
| **Proveedor** | POS |
| **Operación** | `GET /pos/auditable-events` |
| **Entrada** | `desde: DateTime(timezone=True)`, `hasta: DateTime(timezone=True)` |
| **Salida** | `eventos: List[{tipo, ticket_id, usuario_id, timestamp, detalle}]` |
| **Garantías** | El POS expone un RESUMEN de eventos, nunca su tabla `tickets`. Es una cicatriz: ya existe y se conserva. |
| **Errores** | 400 si el rango de fechas es inválido. |
| **Estado** | Implementado |

### §5 Estadísticas (POS es PROVEEDOR)

#### Contrato 6 — `pos.resumen_de_venta`

| Campo | Valor |
|-------|-------|
| **Consumidor** | Estadísticas |
| **Proveedor** | POS |
| **Operación** | `GET /pos/sales-summary` |
| **Entrada** | `desde: Date (hora local)`, `hasta: Date (hora local)` |
| **Salida** | `total_ventas: Numeric(12,2)`, `numero_tickets: Integer`, `por_canal: List[{canal, total: Numeric(12,2)}]` |
| **Garantías** | El POS expone un RESUMEN, nunca su tabla `tickets` (O-23). Usa la fecha LOCAL, no UTC. |
| **Errores** | 400 si el rango de fechas es inválido. |
| **Estado** | Deuda |

### §6 Seguridad

#### Contrato 7 — `seguridad.identidad_del_empleado`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | Seguridad |
| **Operación** | `GET /security/employees/{employee_id}` |
| **Entrada** | `employee_id: UUID` |
| **Salida** | `employee_id: UUID`, `nombre: String`, `perfil: String`, `permisos: List[String]` |
| **Garantías** | Devuelve una PROYECCIÓN, no la tabla `employees` (O-23). El POS NO lee `employees` directamente. |
| **Errores** | 404 si el empleado no existe. |
| **Estado** | Deuda |

#### Contrato 8 — `seguridad.validar_pin`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | Seguridad |
| **Operación** | `POST /security/employees/validate-pin` |
| **Entrada** | `pin: String` |
| **Salida** | `valido: Boolean`, `employee_id: UUID NULL` |
| **Garantías** | Único contrato que expone un secreto en tránsito (O-21). Exige HTTPS y rate-limit obligatorio (O-21). |
| **Errores** | 429 si se excede el rate-limit. 401 si el PIN es inválido. |
| **Estado** | Deuda |

### §7 Caja (ya existe, completo)

#### Contrato 9 — `caja.sesion_activa`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | Caja |
| **Operación** | `GET /cash/active-session` |
| **Entrada** | `terminal_id: UUID` |
| **Salida** | `cash_session_id: UUID NULL`, `abierta_en: DateTime(timezone=True) NULL` |
| **Garantías** | El POS NO lee la tabla `cash_sessions`. |
| **Errores** | — |
| **Estado** | Ya existe |

#### Contrato 10 — `caja.abrir_turno`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | Caja |
| **Operación** | `POST /cash/open-session` |
| **Entrada** | `terminal_id: UUID`, `usuario_id: UUID`, `monto_inicial: Numeric(12,2)`, `usuario_nombre: String NULL` (F10.5 — paridad de datos) |
| **Salida** | `cash_session_id: UUID`, `abierta_en: DateTime(timezone=True)` |
| **Garantías** | Una sola sesión abierta por terminal. |
| **Errores** | 409 si ya hay una sesión abierta en la terminal. |
| **Estado** | Ya existe |

#### Contrato 11 — `caja.registrar_movimiento`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | Caja |
| **Operación** | `POST /cash/movements` |
| **Entrada** | `cash_session_id: UUID`, `tipo: String`, `monto: Numeric(12,2)`, `motivo: Text NULL` |
| **Salida** | `movement_id: UUID` |
| **Garantías** | Una sesión CLOSED ya no acepta movimientos. |
| **Errores** | 400 si la sesión está cerrada. |
| **Estado** | Ya existe |

#### Contrato 12 — `caja.resumen_del_turno`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | Caja |
| **Operación** | `GET /cash/session-summary/{cash_session_id}` |
| **Entrada** | `cash_session_id: UUID` |
| **Salida** | `esperado: Numeric(12,2)`, `movimientos: List[{tipo, monto}]`, `fondo_inicial`, `total_entradas`, `total_salidas`, `total_credito`, `total_debito`, `total_ventas`, `num_transacciones` (F10.5 — paridad de datos, 8 campos del viejo `CashSummaryResponse`) |
| **Garantías** | Devuelve una PROYECCIÓN, no la tabla `cash_movements`. |
| **Errores** | 404 si la sesión no existe. |
| **Estado** | Ya existe |

#### Contrato 13 — `caja.cerrar_turno`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | Caja |
| **Operación** | `POST /cash/close-session` |
| **Entrada** | `cash_session_id: UUID`, `montos_fisicos: Numeric(12,2)` |
| **Salida** | `esperado: Numeric(12,2)`, `capturado: Numeric(12,2)`, `diferencia: Numeric(12,2)` |
| **Garantías** | Marca la sesión como CLOSED y guarda los montos físicos. Devuelve la diferencia (descuadre). Una sesión CLOSED ya no acepta movimientos. |
| **Errores** | 400 si la sesión ya está cerrada. |
| **Estado** | Ya existe |

#### Contrato 14 — `caja.reporte_diario`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS / Estadísticas |
| **Proveedor** | Caja |
| **Operación** | `GET /cash/daily-report/{fecha}` |
| **Entrada** | `fecha: Date (hora local)` |
| **Salida** | `reporte: List[{canal, cajero, terminal, total: Numeric(12,2)}]` |
| **Garantías** | Agrupa por canal y por cajero/terminal. Usa la fecha LOCAL, no UTC. |
| **Errores** | 400 si el formato de fecha es inválido. |
| **Estado** | Ya existe |

### §8 Pedidos

#### Contrato 15 — `pedidos.registrar_desde_ticket`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | Pedidos |
| **Operación** | `POST /orders/from-ticket` |
| **Entrada** | `ticket_id: UUID`, `order_type: String`, `status_ticket: String`, `delivery_type: String`, `customer_name: String NULL`, `customer_phone: String NULL`, `committed_at: DateTime NULL`, `packaging_type: String`, `delivery_address: Text NULL`, `notes: Text NULL` |
| **Salida** | `order_id: UUID`, `status: String`, `earliest_ready_at: DateTime(timezone=True)` |
| **Garantías** | Idempotente por `ticket_id`: si el pedido ya existe, lo ACTUALIZA. El mapeo de estado es del proveedor: OPEN → TENTATIVO, PAID → PAGADO. Pedidos calcula `earliest_ready_at`; el POS no lo inventa. Se ejecuta en la MISMA transacción del guardado del ticket. |
| **Errores** | 404 si el ticket no existe. 409 si el ticket no es de tipo PEDIDO. |
| **Estado** | Deuda |

#### Contrato 16 — `pedidos.pedido_del_ticket`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | Pedidos |
| **Operación** | `GET /orders/by-ticket/{ticket_id}` |
| **Entrada** | `ticket_id: UUID` |
| **Salida** | `order_id`, `delivery_type`, `status`, `customer_name`, `customer_phone`, `committed_at`, `packaging_type`, `delivery_address`, `delivery_fee`, `notes` |
| **Garantías** | Devuelve el pedido asociado al ticket, o 404 si no tiene. Devuelve una PROYECCIÓN, no la fila completa (O-23). |
| **Errores** | 404 si el ticket no existe o no tiene pedido. |
| **Estado** | Deuda |

### §9 Visión (proveedor: Centro de IA — DT-07)

#### Contrato 17 — `vision.reconocer_producto`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | Centro de IA |
| **Operación** | `POST /vision/predict` |
| **Entrada** | `frame_base64: String`, `channel: String`, `top_k: Integer = 3` |
| **Salida** | `detecciones: List[{sku, nombre, confianza: Float(0..1), bbox}]`, `modelo_version: String` |
| **Garantías** | Devuelve hasta `top_k` candidatos ordenados por confianza. Si no reconoce nada, devuelve lista vacía (200), no error. El POS decide qué hacer: Visión solo sugiere. |
| **Errores** | 400 si `frame_base64` está vacío o excede 5 MB. 503 si el modelo no está cargado. |
| **Estado** | Deuda |

### §10 POS atómico — FASE 3.2 (corrige el defecto D-2)

#### Contrato 18 — `pos.añadir_item`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | POS |
| **Operación** | `POST /pos/tickets/{ticket_id}/items` |
| **Entrada** | `ticket_id: UUID`, `item_id: String`, `product_id: UUID`, `quantity: Integer`, `version: Integer` |
| **Salida** | `ticket_id`, `item_id`, `version`, `total: Numeric(12,2)`, `lineas: List[{item_id, product_id, quantity, unit_price, subtotal}]` |
| **Garantías** | IDEMPOTENTE por `item_id`: repetir el POST deja el ticket en el MISMO estado (no duplica). El `unit_price` se congela desde el catálogo (RN-18). Un producto ya presente incrementa su cantidad (RN-17). Toda escritura valida el `version` (RN-25) y lo incrementa (RN-27). Devuelve una PROYECCIÓN (O-23). |
| **Errores** | 404 si el producto no existe (RN-21). 400 si está inactivo (RN-22), si la cantidad no es entero positivo (RN-20), si la sesión no está activa (RN-24), si el ticket está PAID (RN-23). 409 si el `version` no coincide (RN-25). |
| **Estado** | FASE 3.2 |

#### Contrato 19 — `pos.cambiar_cantidad`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | POS |
| **Operación** | `PATCH /pos/tickets/{ticket_id}/items/{item_id}` |
| **Entrada** | `ticket_id: UUID`, `item_id: String`, `quantity: Integer`, `version: Integer` |
| **Salida** | `ticket_id`, `item_id`, `version`, `total`, `lineas` |
| **Garantías** | BLOQUEO OPTIMISTA por `version`: si no coincide, responde 409 y NO escribe (RN-25/RN-26). El `unit_price` NO se recalcula (RN-18). El subtotal se recalcula (RN-19). Cada escritura incrementa el `version` (RN-27). Devuelve una PROYECCIÓN (O-23). |
| **Errores** | 404 si el ticket o el ítem no existen. 400 si la cantidad no es entero positivo (RN-20) o si el ticket está PAID (RN-23). 409 si el `version` no coincide (RN-25/RN-26). |
| **Estado** | FASE 3.2 |

#### Contrato 20 — `pos.quitar_item`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | POS |
| **Operación** | `DELETE /pos/tickets/{ticket_id}/items/{item_id}` |
| **Entrada** | `ticket_id: UUID`, `item_id: String`, `version: Integer` |
| **Salida** | `ticket_id`, `item_id`, `version`, `total`, `lineas` |
| **Garantías** | ANTI-DEGRADACIÓN (RN-37): si quitar la línea reduce el total de líneas en más del 50%, la operación se RECHAZA con 400. El total se recalcula (RN-16). Cada escritura incrementa el `version` (RN-27). Devuelve una PROYECCIÓN (O-23). |
| **Errores** | 404 si el ticket o el ítem no existen. 400 si la reducción supera el 50% (RN-37) o si el ticket está PAID (RN-23). 409 si el `version` no coincide (RN-25). |
| **Estado** | FASE 3.2 |

#### Contrato 21 — `pos.leer_ticket`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | POS |
| **Operación** | `GET /pos/tickets/{ticket_id}` |
| **Entrada** | `ticket_id: UUID` |
| **Salida** | `id: UUID`, `account_num: String`, `status: String`, `total: Numeric(12,2)`, `version: Integer` |
| **Garantías** | RESPUESTA LIGERA: devuelve EXACTAMENTE 5 campos escalares (Regla 15). NO devuelve las líneas. Devuelve una PROYECCIÓN (O-23). `account_num` es el folio (RN-10); `id` es la identidad (RN-09). |
| **Errores** | 404 si el ticket no existe. |
| **Estado** | FASE 3.2 |

#### Contrato 22 — `pos.verificar_envio`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | POS |
| **Operación** | `POST /pos/tickets/{ticket_id}/verify` |
| **Entrada** | `ticket_id: UUID`, `item_ids: List[String]` |
| **Salida** | `existe: Boolean`, `item_ids_persistidos: List[String]`, `faltantes: List[String]` |
| **Garantías** | VERIFICACIÓN POST-ENVÍO (v6.1 $453): confirma en la BASE DE DATOS que el ticket y sus ítems existen ANTES de que el frontend limpie el carrito. `existe` es True solo si el ticket está persistido. `faltantes` lista los `item_ids` que el cliente cree haber enviado pero que NO están en la BD. Es de SOLO LECTURA. |
| **Errores** | 404 si el ticket no existe. |
| **Estado** | FASE 3.2 |

### §11 Pizarrón de cuentas abiertas — FASE 5.0

#### Contrato 23 — `pos.cuentas_abiertas`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | POS |
| **Operación** | `GET /pos/open-accounts` |
| **Entrada** | `terminal_id: String` |
| **Salida** | `cuentas: List[CuentaAbiertaSalida]` |
| **Garantías** | Devuelve una PROYECCIÓN de las cuentas OPEN (O-23). RESPUESTA LIGERA: campos escalares explícitos (Regla 15). F12.6 — PARIDAD DE PRESENTACIÓN: incluye terminal, capturista, cliente, teléfono, tipo de pedido, tipo de entrega y hora de creación, como el viejo POS. Solo devuelve cuentas de la terminal pedida (RN-31). NO devuelve las líneas (es del contrato 21). |
| **Errores** | 400 si `terminal_id` está vacío. |
| **Estado** | FASE 5.0 + F12.6 |

### §12 IA — Voz (proveedor: Centro de IA — DT-07 / FASE 7.0)

#### Contrato 24 — `ia.transcribir_voz`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | Centro de IA |
| **Operación** | `POST /ai/voice/transcribe` |
| **Entrada** | `audio_base64: String`, `formato: String = 'webm'`, `idioma: String = 'es-MX'` |
| **Salida** | `texto: String`, `confianza: Float(0..1)`, `duracion_ms: Integer` |
| **Garantías** | Transcribe el audio a texto en español. NO interpreta la intención. El POS decide qué hacer con el texto. Si el audio es silencio o ininteligible, devuelve `texto` vacío (200), no error. |
| **Errores** | 400 si `audio_base64` está vacío o excede 10 MB. 503 `IA_NO_DISPONIBLE` si el motor de voz no está cargado. |
| **Estado** | FASE 7.0 |

#### Contrato 25 — `ia.interpretar_intencion`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | Centro de IA |
| **Operación** | `POST /ai/voice/parse-intent` |
| **Entrada** | `texto: String`, `contexto: Dict = {}` |
| **Salida** | `intent: String`, `entidades: Dict`, `confianza: Float(0..1)` |
| **Garantías** | Traduce el texto a una intención estructurada. El POS valida el intent contra su allowlist: la IA PROPONE, el operador CONFIRMA. Si no reconoce la intención, devuelve `intent='desconocido'` (200), no error. |
| **Errores** | 400 si `texto` está vacío. 503 `IA_NO_DISPONIBLE` si el motor NLU no está cargado. |
| **Estado** | FASE 7.0 |

### §13 CRM — Beneficios del cliente (proveedor: CRM — FASE 8.0)

#### Contrato 26 — `clientes.beneficios_para_ticket`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | CRM |
| **Operación** | `POST /crm/benefits/for-ticket` |
| **Entrada** | `cliente_id: String | None`, `telefono: String | None`, `total: String`, `lineas: List[Dict]` |
| **Salida** | `beneficios: List[Dict]`, `puntos_disponibles: Integer`, `puntos_a_ganar: Integer` |
| **Garantías** | Devuelve los beneficios aplicables al ticket (puntos, promociones). El CRM es el dueño del cálculo; el POS solo muestra y aplica. Si el cliente no está identificado, devuelve lista vacía (200), no error. |
| **Errores** | 400 si `total` no es un String decimal válido. 503 `CRM_NO_DISPONIBLE` si el CRM no responde; el POS degrada a sin beneficios. |
| **Estado** | FASE 8.0 |

### §14 Notificaciones — Envío del ticket (proveedor: Notificaciones — FASE 8.0)

#### Contrato 27 — `notificaciones.encolar_ticket`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | Notificaciones |
| **Operación** | `POST /notifications/enqueue-ticket` |
| **Entrada** | `ticket_id: String`, `canal: String = 'whatsapp'`, `destino: String`, `payload: Dict` |
| **Salida** | `encolado: Boolean`, `envio_id: String` |
| **Garantías** | ENCOLA el envío del ticket (patrón Outbox, Regla de Oro #7). El POS encola dentro de la transacción del ticket; el worker envía después. El POS NUNCA envía directamente: solo encola. |
| **Errores** | 400 si `destino` está vacío o `canal` no es soportado. 503 `NOTIFICACIONES_NO_DISPONIBLE` si la cola no responde; el POS no bloquea la venta. |
| **Estado** | FASE 8.0 |

### §15 Caja — Eliminar movimiento (proveedor: Caja — FASE 10.6.2)

#### Contrato 29 — `caja.eliminar_movimiento`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | Caja |
| **Operación** | `DELETE /cash/movements/{movement_id}` |
| **Entrada** | `movement_id: UUID` |
| **Salida** | `eliminado: Boolean` |
| **Garantías** | Solo se elimina un movimiento si la sesión está ABIERTA (RN-52). Una sesión CLOSED es inmutable (400). El viejo POS ya tenía esta operación; el nuevo POS la había OMITIDO. F10.6.2 restaura la paridad de operación. |
| **Errores** | 404 si el movimiento no existe. 400 si la sesión del movimiento está cerrada (RN-52). |
| **Estado** | FASE 10.6.2 |

### §16 POS — Lectura de líneas (proveedor: POS — FASE 12.10)

#### Contrato 30 — `pos.leer_lineas`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS (UI) |
| **Proveedor** | POS (núcleo) |
| **Operación** | `GET /pos/tickets/{ticket_id}/items` |
| **Entrada** | `ticket_id: UUID` |
| **Salida** | `lineas: List[LineaSalida]` |
| **Garantías** | Devuelve las líneas FRESCAS del ticket leídas de la base, no del estado en memoria del cliente. Es la operación que resuelve la recuperación de una cuenta del pizarrón (F12.18): al recuperar una cuenta, el POS NO confía en su carrito local; pide las líneas al servidor. |
| **Errores** | 404 si el ticket no existe. |
| **Estado** | FASE 12.10 |

**Por qué existe este contrato (la cicatriz que lo justifica):** el bug de la Cuenta Fantasma $453 (v6.1) nació de confiar en el estado local del carrito al recuperar una cuenta. El POS creía tener unas líneas que el servidor no tenía (o al revés). La cura fue una operación explícita de lectura de líneas: la fuente de verdad es la base, no el `useState`. Este contrato es la frontera que impide que el cliente invente el contenido de un ticket.

### §17 POS — Creación de ticket (proveedor: POS — FASE 12.9.1)

#### Contrato 29 — `pos.crear_ticket`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS (UI) |
| **Proveedor** | POS (núcleo) |
| **Operación** | `POST /pos/tickets` |
| **Entrada** | `terminal_id: String`, `items: List[LineaEntrada]`, `channel: String` |
| **Salida** | `id: UUID`, `folio: String`, `status: String`, `total: Decimal`, `version: Int` |
| **Garantías** | Crea el ticket en estado `draft` con folio asignado por el servidor (RN-10). El folio NUNCA lo inventa el cliente. La creación es idempotente por `terminal_id` + contenido (RN-33: la misma terminal continúa su draft). |
| **Errores** | 400 si no hay sesión de terminal activa (RN-24). 409 si la terminal ya tiene un draft incompatible. |
| **Estado** | FASE 12.9.1 |

> **Nota de numeración (residuo histórico):** el número `29` aparece dos veces en el registro — una para `caja.eliminar_movimiento` (§15) y otra para `pos.crear_ticket` (§17). Es un residuo de la evolución del registro, no un error de contenido. El registro tiene **31 entradas** en total. Se documenta aquí para que la próxima IA no se confunda al leer el `registry.py`.

### §18 Estadísticas — Contexto diario (proveedor: Estadísticas — FASE 10.4)

#### Contrato 28 — `pos.contexto_diario`

| Campo | Valor |
|-------|-------|
| **Consumidor** | POS |
| **Proveedor** | Estadísticas |
| **Operación** | `POST /analytics/context` |
| **Entrada** | `terminal_id: String`, `fecha: Date`, `resumen: Dict` |
| **Salida** | `recibido: Boolean` |
| **Garantías** | El POS ENVÍA su contexto diario a Estadísticas. Es un contrato de SALIDA del POS (el POS es proveedor de datos, Estadísticas es consumidor). El envío es best-effort: si Estadísticas no responde, el POS NO bloquea la operación (DT-07). |
| **Errores** | 503 `ESTADISTICAS_NO_DISPONIBLE` si el módulo no responde; el POS continúa. |
| **Estado** | **Deuda** |

> **Nota de numeración (residuo histórico):** el número `28` también aparece dos veces en el registro. Igual que el `29`, es un residuo. El contenido es correcto.

---

## 4. El POS como consumidor vs. el POS como proveedor

El POS es el módulo más activo del sistema, y por eso aparece en los dos lados de la frontera. Confundir los dos papeles es la fuente de la mitad de los bugs de integración. La distinción es tajante:

### 4.1 El POS como CONSUMIDOR (pide operaciones a otros)

Cuando el POS necesita algo de otro módulo, **pide una operación**, nunca lee una tabla. El POS es consumidor en:

| Contrato | Pide a | Para qué |
|----------|--------|----------|
| 1 `catalogo.productos_para_venta` | Catálogo | Saber qué se puede vender |
| 2 `almacenes.consumir_por_venta` | Almacenes | Descontar stock al vender |
| 3 `almacenes.disponibilidad` | Almacenes | Saber si hay stock |
| 4 `produccion.disponible_para_vender` | Producción | Saber qué está listo |
| 7 `seguridad.identidad_del_empleado` | Seguridad | Saber quién opera |
| 8 `seguridad.validar_pin` | Seguridad | Autorizar acciones |
| 9–14 `caja.*` | Caja | Abrir/cerrar turno, movimientos, resumen |
| 15–16 `pedidos.*` | Pedidos | Registrar y consultar pedidos |
| 17 `vision.reconocer_producto` | Centro de IA | Reconocer producto por imagen |
| 24–25 `ia.*` | IA | Transcribir voz, interpretar intención |
| 26 `clientes.beneficios_para_ticket` | CRM | Aplicar beneficios |
| 27 `notificaciones.encolar_ticket` | Notificaciones | Encolar el envío del ticket |
| 28 `pos.contexto_diario` | Estadísticas | Enviar contexto diario |

**La regla del consumidor (P-02):** el POS pregunta por OPERACIÓN. Nunca hace `SELECT * FROM productos` ni `SELECT * FROM stock_almacen`. Si necesita un dato que no tiene contrato, **se crea el contrato**, no se abre la tabla.

### 4.2 El POS como PROVEEDOR (otros le piden operaciones)

El POS también expone operaciones a otros módulos y a su propia UI. El POS es proveedor en:

| Contrato | Le pide | Quién |
|----------|---------|-------|
| 5 `pos.eventos_auditables` | Auditoría y Control | Los eventos auditables del POS |
| 6 `pos.resumen_de_venta` | Estadísticas | El resumen de una venta |
| 18–22 `pos.*` (atómico) | La UI del POS | Añadir/cambiar/quitar ítem, leer ticket, verificar envío |
| 23 `pos.cuentas_abiertas` | La UI del POS (pizarrón) | Las cuentas abiertas |
| 29 `pos.crear_ticket` | La UI del POS | Crear el ticket |
| 30 `pos.leer_lineas` | La UI del POS | Leer las líneas frescas |

**La regla del proveedor (P-01):** el POS es el ÚNICO que escribe sus tablas (`tickets`, `ticket_items`, `terminal_sessions`, `terminal_locks`, `pos_audit_log`). Nadie más las toca. Cuando Auditoría necesita saber qué pasó, **no lee `pos_audit_log`**: pide `pos.eventos_auditables` y recibe una proyección.

### 4.3 La asimetría intencional

Hay una asimetría deliberada entre los dos papeles:

- **Como consumidor, el POS es frágil:** si un proveedor falla, el POS debe degradar con gracia (DT-07). Nunca bloquea una venta por un fallo ajeno.
- **Como proveedor, el POS es sólido:** sus operaciones son transaccionales y atómicas. Si el POS dice que guardó, guardó. La verificación post-envío (contrato 22) existe precisamente para confirmar esto contra la base.

Esta asimetría es la que permite que el POS sea el corazón del sistema sin ser su punto único de fallo.

---

## 5. La regla de oro DT-07: un fallo ajeno nunca bloquea una venta

DT-07 es la decisión de diseño más importante de la frontera del POS. Su enunciado:

> **Un fallo de un módulo ajeno (CRM, Notificaciones, IA, Estadísticas) NUNCA bloquea una venta. El POS degrada y cobra igual.**

### 5.1 Cómo se implementa

El POS trata a los módulos ajenos como **opcionales en la ruta crítica**. La venta tiene un camino mínimo que NO depende de nadie:

```
[Catálogo] → [Carrito] → [Cobro] → [Ticket guardado] → [Venta cerrada]
     ↑            ↑           ↑            ↑
  puede fallar  local      local      transaccional
  (degrada)               (siempre)   (siempre)
```

Los módulos que SÍ pueden fallar sin bloquear:
- **CRM** (contrato 26): si no responde, el POS cobra **sin beneficios** y avisa "sin beneficios".
- **Notificaciones** (contrato 27): si no responde, el POS cobra **sin envío** y avisa "sin envío".
- **IA** (contratos 24–25): si no responde, el POS opera **en modo manual**.
- **Estadísticas** (contrato 28): si no responde, el POS **no envía el contexto** y sigue.

### 5.2 La traducción del fallo

El Gateway traduce cualquier fallo de un módulo ajeno a un código de negocio, nunca a un 500 crudo:

| Módulo caído | Código | Efecto en el POS |
|--------------|--------|------------------|
| CRM | 503 `CRM_NO_DISPONIBLE` | Cobra sin beneficios |
| Notificaciones | 503 `NOTIFICACIONES_NO_DISPONIBLE` | Cobra sin envío |
| IA | 503 `IA_NO_DISPONIBLE` | Modo manual |
| Estadísticas | 503 `ESTADISTICAS_NO_DISPONIBLE` | No envía contexto |

### 5.3 La cicatriz que lo justifica

DT-07 nació del bug del **Error 500 Lazy Load** (§3.8 del Tomo III): el POS intentaba cargar un dato de un módulo ajeno en medio del cobro, el módulo fallaba, y el 500 tumbaba la venta completa. La cura fue sacar a los módulos ajenos de la ruta crítica y degradar con gracia. **Nunca más una venta se pierde por un módulo que no es el POS.**

### 5.4 Lo que NO se degrada

La degradación tiene un límite. **NO se degrada** lo que es del propio POS:
- El guardado del ticket (contrato 29) es transaccional: si falla, la venta NO se cerró.
- El cobro (contrato 18–22) es atómico: si falla, el ticket queda en su estado previo.
- La verificación post-envío (contrato 22) es obligatoria: confirma contra la base.

La regla es: **se degrada lo ajeno, se blinda lo propio.**

---

## 6. El patrón Outbox en los contratos

El patrón Outbox es la Regla de Oro #7 y vive en la frontera del POS. Su enunciado:

> **El POS NUNCA envía un ticket directamente. Lo ENCOLA dentro de la transacción del ticket. Un worker lo envía después.**

### 6.1 Por qué existe

Si el POS enviara el ticket directamente (por WhatsApp/Email) dentro del cobro, tendría dos problemas:
1. **Acoplamiento:** el cobro dependería de que el servicio de mensajería responda.
2. **Inconsistencia:** si el envío falla después de guardar el ticket, el ticket existe pero el envío no, sin registro.

El Outbox resuelve ambos: el envío se **encola en la misma transacción** que guarda el ticket. Si la transacción commitea, el envío está garantizado (el worker lo hará). Si la transacción falla, el envío no se encoló (no hay envío de un ticket que no existe).

### 6.2 El contrato que lo implementa

El contrato 27 (`notificaciones.encolar_ticket`) es la cara visible del Outbox:

| Aspecto | Detalle |
|---------|---------|
| **Operación** | `POST /notifications/enqueue-ticket` |
| **Entrada** | `ticket_id`, `canal`, `destino`, `payload` |
| **Salida** | `encolado: Boolean`, `envio_id: String` |
| **Garantía** | ENCOLA, no envía. El POS encola dentro de la transacción del ticket. |
| **Regla** | RN-85 (envío por outbox, no directo), RN-86 (encolar en la misma transacción) |

### 6.3 El guardián que lo protege

El guardián `OutboxTransaccional` (`guards/outbox.py`) modela la transacción y garantiza:
- `guardar_ticket()` y `emitir_evento()` ocurren en la misma transacción.
- Si la transacción falla, el evento NO se aplica (no se envía lo que no se guardó).
- La excepción `SinSilenciosEnRutaCritica` (E-05) impide silenciar un fallo en la ruta crítica.

### 6.4 La cicatriz que lo justifica

El Outbox nació del bug del **Ticket Secuestrado T5** (§3.6 del Tomo III): el POS enviaba el ticket directamente y, si el envío fallaba a medias, el ticket quedaba en un estado inconsistente. La cura fue el Outbox: el envío es un evento encolado, no una llamada directa.

---

## 7. Las notas de frontera (CRM, Notificaciones, IA)

El registro de contratos (`contracts/registry.py`) incluye notas de frontera explícitas para los tres módulos ajenos más delicados. Estas notas son **contratos de comportamiento**, no sugerencias.

### 7.1 Frontera CRM (contrato 26)

> **El CRM es OPCIONAL. Un fallo del CRM NUNCA bloquea una venta.**

- El POS pide `clientes.beneficios_para_ticket` con el `cliente_id` (que puede ser `None`).
- Si el cliente es `None`, el POS cobra sin beneficios (RN-82: cliente opcional no bloquea).
- Si el CRM falla, el POS cobra sin beneficios y avisa.
- Los beneficios NUNCA hacen el total negativo (RN-83).
- Los puntos NO se actualizan: se ANEXAN (RN-84) — el POS no es dueño del saldo de puntos.

### 7.2 Frontera Notificaciones (contrato 27)

> **Notificaciones es OPCIONAL y ASÍNCRONO. El POS ENCOLA, no envía.**

- El POS encola el envío dentro de la transacción del ticket (Outbox).
- Si Notificaciones falla, el POS cobra sin envío y avisa (RN-88).
- El canal debe ser soportado (RN-89) y el destino válido para el canal (RN-90).
- El POS NUNCA envía directamente (RN-85).

### 7.3 Frontera IA (contratos 24–25)

> **La IA es ASISTIVA. Nunca bloquea una venta manual.**

- El POS pide `ia.transcribir_voz` y `ia.interpretar_intencion`.
- Si la IA falla, el POS opera en modo manual (RN-74: visión asistiva no bloquea venta manual).
- La IA propone; el humano dispone. La propuesta de la IA se muestra y el operador la acepta o la rechaza.
- El motor de IA es configurable (RN-71: motor ORB).

### 7.4 La regla común a las tres fronteras

Las tres fronteras comparten una regla: **el módulo ajeno es opcional; el POS es obligatorio.** El POS nunca delega en un módulo ajeno la responsabilidad de cerrar una venta. Si el módulo ajeno no está, el POS cierra la venta igual, con menos funciones.

---

## 8. El test de la puerta F2

La FASE 2 (Frontera) tiene una puerta de calidad que se ejecuta en CI. Su propósito es verificar que la frontera por contratos se respeta de verdad, no solo en el papel.

### 8.1 Qué verifica

El test de la puerta F2 itera el registro de contratos y verifica:

1. **Ningún contrato expone una tabla.** El campo `tabla_expuesta` de cada `Contrato` es siempre `None` (O-23). Si algún contrato expusiera una tabla, el test falla.
2. **Todos los contratos tienen consumidor y proveedor.** No hay contratos huérfanos.
3. **Todos los contratos tienen operación.** No hay contratos sin punto de entrada.
4. **El número de contratos es el esperado.** El registro tiene 31 entradas.

### 8.2 El código del test (conceptual)

```python
def test_puerta_f2_ningun_contrato_expone_tabla():
    """O-23: un contrato NUNCA expone una tabla."""
    for contrato in listar_contratos():
        assert contrato.tabla_expuesta is None, (
            f"El contrato {contrato.numero} ({contrato.nombre}) "
            f"expone una tabla: {contrato.tabla_expuesta}"
        )

def test_puerta_f2_todos_los_contratos_completos():
    """Todo contrato tiene consumidor, proveedor y operación."""
    for contrato in listar_contratos():
        assert contrato.consumidor
        assert contrato.proveedor
        assert contrato.operacion
```

### 8.3 Por qué es una puerta y no un test cualquiera

La puerta F2 es una **compuerta de arquitectura**: no verifica una función, verifica una PROPIEDAD del sistema. Si alguien intenta que un contrato exponga una tabla (por ejemplo, para "optimizar" una consulta), la puerta lo detiene. Es el guardián automatizado de A-02 y O-23.

### 8.4 La lección de la puerta F2

La lección (heredada de la FASE 10.6.2, "la completitud del conjunto también es una compuerta") es que **la frontera no se verifica leyendo el código, se verifica ejecutando una puerta**. Un contrato que "parece" respetar la frontera puede no respetarla. La puerta F2 lo comprueba mecánicamente.

---

## 9. Matriz de trazabilidad: contrato → consumidor → proveedor → estado

Esta matriz es el mapa completo de la frontera del POS. Cada fila es un contrato; cada columna, su lugar en el sistema.

| # | Contrato | Consumidor | Proveedor | Estado |
|---|----------|------------|-----------|--------|
| 1 | `catalogo.productos_para_venta` | POS | Catálogo | Ya existe (parcial) |
| 2 | `almacenes.consumir_por_venta` | POS | Almacenes | Deuda |
| 3 | `almacenes.disponibilidad` | POS | Almacenes | Deuda |
| 4 | `produccion.disponible_para_vender` | POS | Producción | Deuda |
| 5 | `pos.eventos_auditables` | Auditoría y Control | POS | Implementado (Cicatriz) |
| 6 | `pos.resumen_de_venta` | Estadísticas | POS | Deuda |
| 7 | `seguridad.identidad_del_empleado` | POS | Seguridad | Deuda |
| 8 | `seguridad.validar_pin` | POS | Seguridad | Deuda |
| 9 | `caja.sesion_activa` | POS | Caja | Ya existe |
| 10 | `caja.abrir_turno` | POS | Caja | Ya existe |
| 11 | `caja.registrar_movimiento` | POS | Caja | Ya existe |
| 12 | `caja.resumen_del_turno` | POS | Caja | Ya existe |
| 13 | `caja.cerrar_turno` | POS | Caja | Ya existe |
| 14 | `caja.reporte_diario` | POS | Caja | Ya existe |
| 15 | `pedidos.registrar_desde_ticket` | POS | Pedidos | Deuda |
| 16 | `pedidos.pedido_del_ticket` | POS | Pedidos | Deuda |
| 17 | `vision.reconocer_producto` | POS | Centro de IA | Deuda |
| 18 | `pos.añadir_item` | UI del POS | POS | Implementado |
| 19 | `pos.cambiar_cantidad` | UI del POS | POS | Implementado |
| 20 | `pos.quitar_item` | UI del POS | POS | Implementado |
| 21 | `pos.leer_ticket` | UI del POS | POS | Implementado |
| 22 | `pos.verificar_envio` | UI del POS | POS | Implementado |
| 23 | `pos.cuentas_abiertas` | UI del POS | POS | Implementado |
| 24 | `ia.transcribir_voz` | POS | IA | Implementado |
| 25 | `ia.interpretar_intencion` | POS | IA | Implementado |
| 26 | `clientes.beneficios_para_ticket` | POS | CRM | Implementado |
| 27 | `notificaciones.encolar_ticket` | POS | Notificaciones | Implementado |
| 28 | `pos.contexto_diario` | POS | Estadísticas | Deuda |
| 29 | `caja.eliminar_movimiento` | POS | Caja | Implementado |
| 29 | `pos.crear_ticket` | UI del POS | POS | Implementado |
| 30 | `pos.leer_lineas` | UI del POS | POS | Implementado |

### 9.1 Lectura de la matriz

- **31 contratos** en total (los números 28 y 29 se repiten por residuo histórico).
- **El POS es consumidor** en 17 contratos (1–17, 24–28).
- **El POS es proveedor** en 14 contratos (5–6, 18–23, 29–30).
- **Estados:** 6 "Ya existe" (Caja), 1 "Ya existe (parcial)" (Catálogo), 1 "Implementado (Cicatriz)" (Auditoría), 12 "Implementado", 11 "Deuda".

### 9.2 Las deudas de la frontera

Las 11 deudas son contratos declarados pero aún no implementados. Son la lista de trabajo de las próximas fases:

| Deuda | Contrato | Módulo que falta |
|-------|----------|------------------|
| 1 | `almacenes.consumir_por_venta` | Almacenes |
| 2 | `almacenes.disponibilidad` | Almacenes |
| 3 | `produccion.disponible_para_vender` | Producción |
| 4 | `pos.resumen_de_venta` | Estadísticas |
| 5 | `seguridad.identidad_del_empleado` | Seguridad |
| 6 | `seguridad.validar_pin` | Seguridad |
| 7 | `pedidos.registrar_desde_ticket` | Pedidos |
| 8 | `pedidos.pedido_del_ticket` | Pedidos |
| 9 | `vision.reconocer_producto` | Centro de IA |
| 10 | `pos.contexto_diario` | Estadísticas |
| 11 | `catalogo.productos_para_venta` (parcial) | Catálogo |

**Nota:** las deudas NO son fracasos. Son contratos declarados a propósito antes de implementarlos, para que la frontera exista desde el día 1. Un contrato en "Deuda" es una promesa de arquitectura, no una omisión.

### 9.3 La regla de oro de la matriz

> **Todo contrato tiene un consumidor y un proveedor. Ninguno expone una tabla. La frontera se respeta en el papel (registro) y en la práctica (puerta F2).**

---

## Cierre del Tomo IV

Este tomo documentó la **frontera por contratos** del POS nuevo: los 31 contratos que definen cómo el POS habla con el resto del sistema y cómo el resto habla con él. Los tres principios que lo gobiernan:

1. **A-02 — Frontera por contratos:** el consumidor pide por OPERACIÓN, nunca lee la tabla del proveedor.
2. **O-23 — Un contrato nunca expone una tabla:** `tabla_expuesta` es siempre `None`.
3. **DT-07 — Un fallo ajeno nunca bloquea una venta:** el POS degrada y cobra igual.

La frontera no es una convención: es una **propiedad verificada por la puerta F2**. Cualquier intento de romperla (leer una tabla ajena, exponer una tabla, bloquear una venta por un módulo caído) es detenido por un test, no por la disciplina de un programador.

**El Tomo V (Superficie e interfaces) documenta las 24 interfaces del POS: la cara visible de estos contratos.**