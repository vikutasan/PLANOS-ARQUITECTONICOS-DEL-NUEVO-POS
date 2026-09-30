# 🔍 AUTOCRÍTICA DEL PLAN F7.5 v3.0 — Verificada contra el código

**Fecha:** 30 Sep 2026
**Autor:** Arquitecto del Nuevo POS
**Objeto:** `PLAN_DE_ABORDAJE_FASE_7_5_PEDIDOS.md` v3.0 (commit `bcc901a`)
**Método:** cada afirmación del plan se contrastó contra el código real en `../NUEVO-POS/apps/api`.

---

## §1. VEREDICTO GENERAL

El plan v3.0 **mejoró sustancialmente** respecto a v2.0: la política de pago dejó de ser una
constante hardcodeada y pasó a ser un valor transversal de DT-06. **Eso está bien y se sostiene.**

**Pero la verificación contra el código revela 4 defectos nuevos**, uno de ellos **BLOQUEANTE**:
el plan asume que la tabla `system_settings` existe, y **no existe**.

| # | Defecto | Severidad | Bloquea la ejecución |
|---|---|---|---|
| **D-4** | `system_settings` **no existe** como tabla ni como modelo | **CRÍTICA** | **SÍ** |
| **D-5** | `crear_ticket` hace `commit()` temprano → la proyección `SIN_PAGO` no puede ser "misma transacción" | **ALTA** | SÍ (para `SIN_PAGO`) |
| **D-6** | El plan habla de "9 campos" pero el contrato 15 exige **10** (incluye `status_ticket`) | **MEDIA** | No, pero rompe el gate |
| **D-7** | `Ticket.order_status` tiene default `"PROGRAMADO PARA SER PREPARADO"` — el plan lo ignora | **MEDIA** | No |

---

## §2. D-4 (CRÍTICA) — `system_settings` NO EXISTE

### §2.1 La afirmación del plan

> **§0.2:** *"Se declara un cuarto valor transversal en DT-06... `orders.payment_policy`... se
> persiste en `system_settings`."*
> **§5.1:** *"`services/politica_pedidos.py` — lee `orders.payment_policy` y `orders.deposit_percent`
> de `system_settings`."*

### §2.2 La realidad del código

**La tabla `system_settings` NO existe.** Los 17 modelos declarados en
[`models/__init__.py`](../NUEVO-POS/apps/api/models/__init__.py:17) son:

```
terminal_sessions, tickets, ticket_items, terminal_locks,
cash_sessions, cash_movements,
categories, products, product_technical_sheets,
orders,
almacenes, stock_almacen, movimientos_inventario,
warehouse_events, warehouse_eventos_sin_almacen,
heladeria_product_config, ticket_item_components
```

**No hay `system_settings`.** La búsqueda de `SystemSetting` en todo `apps/api` devuelve **solo
comentarios y docstrings** en `consolidacion/registry.py` y `consolidacion/__init__.py`:

> `# Default 23:30, configurable en SystemSetting. La clave es consolidacion.hora_cierre.`

Es una **referencia a un mecanismo que aún no se construyó**. El `SyncCierreDeDia.desde_setting()`
recibe el valor **como parámetro** (`valor: str | None`) — no lo lee de ninguna tabla.

### §2.3 La consecuencia

**F7.5.1 no se puede ejecutar como está escrito.** No hay tabla que leer. El plan tiene 3 salidas:

| Opción | Qué implica | Veredicto |
|---|---|---|
| **A. Crear `system_settings` en F7.5.1** | Añadir el modelo + migración + seed del default | **Correcta**, pero es trabajo no declarado |
| **B. Leer de una variable de entorno** | `ORDERS_PAYMENT_POLICY` en `.env` | **Incorrecta**: viola DT-06 (no es Vista General) |
| **C. Hardcodear `PAGO_COMPLETO`** | Sin lector | **Incorrecta**: es lo que la v3.0 quiso evitar |

**La opción A es la única correcta.** Pero el plan **no la declara**. Es un defecto de alcance:
F7.5.1 es en realidad **dos** sub-fases (crear la tabla + el lector).

### §2.4 La corrección propuesta

**Dividir F7.5.1 en dos:**

- **F7.5.1a — Crear `system_settings`** (modelo + migración + seed). Es la **primera piedra de
  Vista General** en el nuevo POS. Se declara como tal.
- **F7.5.1b — El lector de política** (`services/politica_pedidos.py`) con default seguro.

> **Nota importante:** crear `system_settings` **no es invadir Vista General**. Es crear la
> **infraestructura de almacenamiento** que DT-06 ya declaró (`system_settings` es la capa de
> almacenamiento de DT-06.2). Vista General es la **UI**; la tabla es compartida. El plan debe
> decirlo explícitamente para que no parezca una violación de frontera.

---

## §3. D-5 (ALTA) — `crear_ticket` hace `commit()` temprano

### §3.1 La afirmación del plan

> **§5.2:** *"Se ejecuta en la misma transacción del guardado/cobro."*
> **§0.2:** *"`SIN_PAGO` → Proyecta el pedido a `orders` al **crear** el ticket."*

### §3.2 La realidad del código

[`crear_ticket`](../NUEVO-POS/apps/api/routers/pos.py:294) hace:

```python
db.add(ticket)
await db.commit()          # ← línea 295: COMMIT TEMPRANO
# ... recarga el ticket ...
return _ticket_a_salida(ticket)
```

**El commit ocurre antes de que el plan pueda proyectar.** Para el modo `SIN_PAGO`, la proyección
debería ocurrir **antes** de ese commit para ser "la misma transacción". Pero el ticket necesita
estar persistido para que `orders.ticket_id` (FK) lo referencie.

### §3.3 La consecuencia

El plan dice "misma transacción" pero **no reconoce que `crear_ticket` ya cierra la transacción**.
Hay 3 salidas:

| Opción | Qué implica | Veredicto |
|---|---|---|
| **A. Mover el commit después de la proyección** | Reestructurar `crear_ticket` | **Correcta**, pero toca código estable |
| **B. Proyectar en una segunda transacción** | `commit()` → proyectar → `commit()` | **Aceptable** si se documenta como "best-effort" |
| **C. Solo proyectar en `PAID`** | Ignorar `SIN_PAGO` en el POS | **Incorrecta**: contradice la política |

**La opción A es la correcta para `SIN_PAGO`.** El plan debe declarar que `crear_ticket` se
reestructura: `db.add(ticket)` → `flush()` (para obtener el id) → proyectar → `commit()`.

### §3.4 La corrección propuesta

Añadir a F7.5.2 un entregable explícito:

> **Reestructurar `crear_ticket`:** reemplazar `commit()` por `flush()` + proyección + `commit()`,
> para que la proyección de `SIN_PAGO` ocurra en la misma transacción. **Verificar que el gate de
> F3.2 (atómico) sigue verde** — es un cambio en código con puerta propia.

> **Riesgo añadido:** `crear_ticket` tiene un gate de Fase 3.2 (persistencia atómica). Tocarlo
> puede romperlo. El plan debe declararlo como riesgo y verificar ese gate.

---

## §4. D-6 (MEDIA) — El contrato 15 exige 10 campos, no 9

### §4.1 La afirmación del plan

> **§5.0:** *"Extender `CrearTicketEntrada`... con los **9 campos** opcionales del pedido."*
> **§5.4:** *"`camposParaTicket()` — devuelve los **9 campos**."*

### §4.2 La realidad del código

El contrato 15 ([`registry.py`](../NUEVO-POS/apps/api/contracts/registry.py:353)) declara **10**
campos de entrada:

```python
entrada={
    "ticket_id": "UUID",
    "order_type": "String",
    "status_ticket": "String",      # ← el 10.º, que el plan omite
    "delivery_type": "String",
    "customer_name": "String NULL",
    "customer_phone": "String NULL",
    "committed_at": "DateTime(timezone=True) NULL",
    "packaging_type": "String",
    "delivery_address": "Text NULL",
    "notes": "Text NULL",
}
```

`status_ticket` es **el estado del ticket** (`OPEN`/`PAID`), que el proveedor usa para mapear
(`OPEN → TENTATIVO`, `PAID → PAGADO`). El plan lo omite porque asumió que el POS solo proyecta en
`PAID` — pero con la política configurable, el POS **también** proyecta en `OPEN` (`SIN_PAGO`).

### §4.3 La consecuencia

El gate de F7.5.0 ("persiste los 9 campos") **no verifica `status_ticket`**, y el de F7.5.3
("responde con `order_id`, `status`, `earliest_ready_at`") tampoco. El contrato quedaría
**parcialmente implementado**.

### §4.4 La corrección propuesta

Cambiar "9 campos" por **"los 10 campos del contrato 15"** en §5.0, §5.4 y §5.6. Añadir un criterio
al gate de F7.5.3: *"`status_ticket` viaja en la entrada y determina el mapeo del estado."*

---

## §5. D-7 (MEDIA) — `Ticket.order_status` tiene un default que el plan ignora

### §5.1 La realidad del código

[`Ticket.order_status`](../NUEVO-POS/apps/api/models/pos.py:78) tiene:

```python
order_status: Mapped[str] = mapped_column(
    String, nullable=False, default="PROGRAMADO PARA SER PREPARADO"
)
```

**El plan nunca menciona `order_status`.** Habla de `order_type`, `delivery_type`, etc., pero no de
este campo, que **ya existe con un default**.

### §5.2 La consecuencia

Hay **dos** estados en juego y el plan los confunde:

| Campo | Dónde vive | Quién lo gobierna |
|---|---|---|
| `tickets.order_status` | La copia de trabajo del POS | **El POS** (es su tabla) |
| `orders.status` | La proyección | **Pedidos** (14 estados) |

El plan dice "el POS solo proyecta el estado" pero **no dice qué escribe en `tickets.order_status`**.
¿Lo deja en el default? ¿Lo actualiza? ¿Se mapea a `orders.status`?

### §5.3 La corrección propuesta

Añadir a §5.0 un entregable: **declarar explícitamente qué hace el POS con `tickets.order_status`**.
Propuesta: el POS lo deja en su default (`PROGRAMADO PARA SER PREPARADO`) y **no lo usa** para la
proyección — la proyección usa `tickets.status` (`OPEN`/`PAID`), que es el que el contrato 15 pide
como `status_ticket`. Documentarlo para que no haya ambigüedad.

---

## §6. LO QUE LA v3.0 HIZO BIEN (y se conserva)

| # | Acierto | Por qué se sostiene |
|---|---|---|
| 1 | **La política como valor transversal de DT-06** | Es la decisión correcta; el código no la contradice |
| 2 | **El default seguro `PAGO_COMPLETO`** | Correcto: degrada al modo conservador |
| 3 | **La división F7.5a / F7.5b** | Correcta: el POS no se toca cuando Vista General exista |
| 4 | **La frontera P-01/A-02** | Correcta: el POS escribe su tabla, proyecta por contrato |
| 5 | **La §11 (la prueba de la frontera)** | Correcta y valiosa |
| 6 | **El cimiento `tickets.order_*` + `orders`** | Verificado: ambos existen |

---

## §7. LA CORRECCIÓN PROPUESTA (v3.1)

| # | Cambio | Defecto que corrige |
|---|---|---|
| 1 | **Dividir F7.5.1 en F7.5.1a (crear `system_settings`) + F7.5.1b (el lector)** | D-4 |
| 2 | **Declarar que crear `system_settings` es la infraestructura de DT-06, no una invasión de Vista General** | D-4 |
| 3 | **Añadir a F7.5.2: reestructurar `crear_ticket` (`flush()` + proyección + `commit()`)** | D-5 |
| 4 | **Añadir el riesgo "tocar `crear_ticket` puede romper el gate de F3.2"** | D-5 |
| 5 | **Cambiar "9 campos" por "los 10 campos del contrato 15"** | D-6 |
| 6 | **Añadir el criterio `status_ticket` al gate de F7.5.3** | D-6 |
| 7 | **Declarar qué hace el POS con `tickets.order_status`** | D-7 |
| 8 | **Añadir a F7.5.0 el criterio "`order_status` queda en su default"** | D-7 |

---

## §8. LAS 4 LECCIONES

1. **Un plan que nombra una tabla debe verificar que la tabla existe.** La v3.0 asumió
   `system_settings` por analogía con DT-06; el código dice que no existe. **Verificar, no asumir.**

2. **"Misma transacción" es una afirmación verificable.** El plan la repitió 3 veces sin comprobar
   que `crear_ticket` ya cierra la transacción. **Una garantía de atomicidad exige leer el código
   que la implementa.**

3. **Los contratos son la fuente de verdad de los campos.** El plan dijo "9 campos" de memoria; el
   contrato 15 declara 10. **Contar contra el contrato, no contra la memoria.**

4. **Los defaults existentes son decisiones tomadas.** `order_status` ya tiene un default; ignorarlo
   es dejar una ambigüedad. **Todo campo preexistente que el plan toque debe declararse.**

---

## §9. RECOMENDACIÓN

**No ejecutar F7.5a todavía.** Escribir la **v3.1** con las 8 correcciones de §7, y **volver a
verificar** contra el código. La v3.1 debe:

1. Declarar la creación de `system_settings` como primer entregable (F7.5.1a).
2. Declarar la reestructuración de `crear_ticket` como entregable explícito (F7.5.2).
3. Hablar de "los 10 campos del contrato 15", no de "9 campos".
4. Declarar el destino de `tickets.order_status`.

**El costo de corregir el plan ahora es de minutos. El costo de descubrir D-4 a mitad de la
ejecución es de horas** (habría que parar, crear la tabla, migrar, y rehacer el lector).

---

**FIN DE LA AUTOCRÍTICA — PLAN F7.5 v3.0**
