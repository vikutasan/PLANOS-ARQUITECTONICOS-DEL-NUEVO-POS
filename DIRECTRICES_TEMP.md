# DIRECTRICES TRANSVERSALES DEL ERP

**Documento 12 del proyecto del Nuevo POS**
**Estado:** Vigente
**Ãmbito:** Todo el ERP (todos los mÃ³dulos, todas las sucursales)
**Naturaleza:** Compendio de reglas que **no pertenecen a un mÃ³dulo**. Pertenecen a todos.

---

## SECCIÃ“N 0 â€” CÃ“MO LEER ESTE DOCUMENTO

Este documento no describe un mÃ³dulo. Describe las **reglas que atraviesan todos los mÃ³dulos**.

La diferencia es importante:

- Un documento de mÃ³dulo dice: *"asÃ­ funciona Almacenes"*.
- Una directriz transversal dice: *"asÃ­ se comporta el tiempo en Almacenes, en Caja, en RRHH y en Grandeza â€” y asÃ­ se verifica"*.

**Una directriz transversal no se "aplica" a un mÃ³dulo. Se "verifica" contra un mÃ³dulo.**

Por eso cada directriz tiene cuatro elementos obligatorios:

| Elemento | QuÃ© es | Por quÃ© |
|---|---|---|
| **La regla** | La norma, en una frase que no admite interpretaciÃ³n | Sin ambigÃ¼edad no hay verificaciÃ³n |
| **El ancla** | El archivo y la lÃ­nea donde vive la implementaciÃ³n canÃ³nica | Sin ancla, la regla es una opiniÃ³n |
| **La verificaciÃ³n** | CÃ³mo se comprueba que un mÃ³dulo la cumple | Sin verificaciÃ³n, la regla es un deseo |
| **La matriz** | El estado de cada mÃ³dulo frente a la regla | Sin matriz, no se sabe quÃ© falta |

**Regla de uso:** cuando se construye o se revisa un mÃ³dulo, se recorre esta lista y se marca cada directriz como **Cumple**, **Parcial** o **No cumple**. Un mÃ³dulo no se declara terminado con una directriz en "No cumple".

---

## SECCIÃ“N 1 â€” EL PRINCIPIO QUE LAS UNIFICA

Todas las directrices de este documento nacen del mismo principio:

> **Lo que es igual en todos los mÃ³dulos no se decide en cada mÃ³dulo. Se decide una vez, aquÃ­, y se verifica en cada mÃ³dulo.**

Esto no es una preferencia de estilo. Es una consecuencia de haber encontrado el mismo problema repetido en varios mÃ³dulos:

- El **tiempo** estaba implementado en **5 lugares distintos** (`core/timezone.py`, `core/timestamps.py`, `core/serialization.py`, `apps/shared/timezone.js`, `apps/shared/TimezoneContext.jsx`).
- El **dinero** estaba formateado en **80 lugares distintos** (`toFixed(2)` suelto en 12 componentes), sin un solo formateador.
- La **identidad** (UUID vs folio) se confundÃ­a en varios mÃ³dulos.
- El **inventario** se escribÃ­a desde fuera del mÃ³dulo dueÃ±o (`products.stock`).

Cada uno de esos es un caso del mismo error: **una decisiÃ³n transversal tomada localmente, muchas veces, de formas distintas.**

---

## SECCIÃ“N 2 â€” DT-01: TIEMPO

### DT-01.1 â€” La regla

> **Todo timestamp se guarda en UTC. Se muestra en hora local. La conversiÃ³n es de presentaciÃ³n, nunca de almacenamiento.**

### DT-01.2 â€” El ancla

| Capa | Archivo | QuÃ© hace |
|---|---|---|
| Almacenamiento | `apps/api/core/timestamps.py` | `utcnow()` â€” la Ãºnica fuente de "ahora" |
| ConfiguraciÃ³n | `apps/api/core/timezone.py` | `get_business_tz(db)` lee `business_timezone` de `system_settings` (cachÃ© 5 min) |
| CÃ¡lculo | `apps/api/core/timezone.py` | `local_now`, `utc_to_local`, `local_day_bounds_utc`, `to_local_date_str` |
| Transporte | `apps/api/core/serialization.py` | `iso_utc()` â€” serializa siempre con marca UTC |
| PresentaciÃ³n | `apps/shared/timezone.js` | `formatLocal`, `formatLocalTime`, `formatLocalDate`, `parseUtc` |
| PresentaciÃ³n | `apps/shared/TimezoneContext.jsx` | `TimezoneProvider` + `useTimezone()` â€” el contexto global |

### DT-01.3 â€” Las reglas derivadas

1. **Nunca `datetime.now()`.** Siempre `utcnow()`. `datetime.now()` devuelve la hora del servidor, que no es la hora del negocio.
2. **Nunca `datetime.utcnow()` de Python 3.12+.** EstÃ¡ deprecado y devuelve un naive. Usar `utcnow()` del proyecto.
3. **Toda columna de tiempo es `DateTime(timezone=True)`.** Un `DateTime` naive es una bomba de tiempo: no se sabe en quÃ© zona estÃ¡.
4. **Los lÃ­mites del dÃ­a se calculan en hora local, no en UTC.** `local_day_bounds_utc(tz, fecha)` devuelve el rango UTC que corresponde al dÃ­a local. Un ticket de las 23:30 local pertenece al dÃ­a local, no al dÃ­a UTC.
5. **Las columnas `Date` (como `journey_date`) son fechas locales.** No se convierten. Son la fecha del calendario del negocio.
6. **El frontend nunca formatea fechas por su cuenta.** Usa `useTimezone()`. Un `new Date().toLocaleString()` suelto es una violaciÃ³n.

### DT-01.4 â€” La verificaciÃ³n

| # | VerificaciÃ³n | CÃ³mo |
|---|---|---|
| V-01 | No existe `datetime.now()` en el cÃ³digo de producciÃ³n | BÃºsqueda estÃ¡tica: `datetime\.now\(\)` |
| V-02 | No existe `datetime.utcnow()` | BÃºsqueda estÃ¡tica: `datetime\.utcnow\(\)` |
| V-03 | No existe `DateTime` sin `timezone=True` en columnas nuevas | RevisiÃ³n del modelo de datos |
| V-04 | El frontend no usa `toLocaleString`/`toLocaleDateString` fuera de `timezone.js` | BÃºsqueda estÃ¡tica en `apps/` |
| V-05 | Los reportes por dÃ­a usan `local_day_bounds_utc` | Prueba: ticket a las 23:30 local aparece en el dÃ­a local |

**Prueba de referencia:** `apps/api/tests/test_bloque9d_3bugs.py` (los 3 bugs de zona horaria). Es el guardiÃ¡n de esta directriz.

### DT-01.5 â€” La matriz de cumplimiento

| MÃ³dulo | Cumple | Evidencia / Deuda |
|---|---|---|
| POS | âœ… | `test_bloque9d_3bugs.py` cubre los 3 bugs |
| Caja | âœ… | Usa `local_day_bounds_utc` en `generar_reporte_diario` |
| Analytics | âœ… | Corregido en el bloque 9d |
| Grandeza | âš ï¸ Parcial | `journey_date` es `Date` local (correcto), pero hay `Intl.DateTimeFormat('en-US')` suelto en `GrandezaDriverUI.jsx:83` |
| Almacenes | âš ï¸ Parcial | `MovimientoInventario.timestamp` sin verificar `timezone=True` |
| RRHH | âš ï¸ Parcial | Pendiente de auditorÃ­a |
| ProducciÃ³n | âš ï¸ Parcial | Pendiente de auditorÃ­a |
| Pedidos | âš ï¸ Parcial | Pendiente de auditorÃ­a |

**Hallazgo previo:** `DOCUMENTACION_MODULO_POS.md:360` (H5) ya habÃ­a detectado que *"no existÃ­a una regla explÃ­cita de timestamps en la documentaciÃ³n, pese a ser un principio transversal"*. Esta directriz cierra ese hallazgo.

---

## SECCIÃ“N 3 â€” DT-02: DINERO

### DT-02.1 â€” La regla

> **El dinero se guarda en `Numeric(12,2)`, nunca en `Float`. La moneda del negocio (`business_currency`) declara en quÃ© moneda se captura; no convierte. La presentaciÃ³n pasa por un solo formateador (`formatMoney`); ningÃºn componente formatea dinero por su cuenta.**

### DT-02.2 â€” El ancla

| Capa | Archivo | QuÃ© hace |
|---|---|---|
| Almacenamiento | `apps/api/modules/*/models.py` | Columnas de dinero en `Numeric(12,2)` |
| ConfiguraciÃ³n | `system_settings.business_currency` | **âœ… Creado en V23** (22 Sep 2026) â€” sembrado en `seed_settings()`, expuesto en `GET /settings/currency` |
| PresentaciÃ³n | `apps/shared/money.js` | **Por crear (V24)** â€” `formatMoney`, `parseMoney`, `roundMoney` |
| PresentaciÃ³n | `apps/shared/MoneyContext.jsx` | **Por crear (V24)** â€” espejo de `TimezoneContext.jsx` |

### DT-02.3 â€” Las reglas derivadas

1. **Nunca `Float` para dinero.** `Float` no representa `0.10` exactamente. `Numeric(12,2)` sÃ­. Un centavo perdido por ticket, multiplicado por miles de tickets, es un descuadre de caja.
2. **El redondeo es half-up y se declara.** `Intl.NumberFormat` usa half-even por defecto (bancario). El negocio usa half-up (comercial). La diferencia se declara explÃ­citamente, no se hereda.
3. **Un solo formateador.** `formatMoney(monto, moneda)`. NingÃºn componente usa `toFixed(2)`, `Intl.NumberFormat` ni concatenaciÃ³n de sÃ­mbolo.
4. **El selector de moneda declara, no convierte.** Cambiar la moneda cambia el **sÃ­mbolo**, nunca el **nÃºmero**. Un selector que cambia el nÃºmero es un conversor de divisas, y un conversor de divisas descuadra la caja.
5. **La moneda se guarda una sola vez, en la configuraciÃ³n del negocio.** No se guarda por ticket, ni por producto, ni por sucursal (salvo que la sucursal opere en otra moneda, lo cual es una decisiÃ³n explÃ­cita y documentada).
6. **El dinero no se suma en el frontend.** Los totales vienen del backend. El frontend solo formatea.
7. **El dinero que llega de la API viaja como STRING, y se coercionar antes de operarlo.** Pydantic serializa `Decimal` como **cadena** (`"price":"12.00"`), no como nÃºmero, para **preservar la precisiÃ³n decimal** (evitar el error de coma flotante). Es correcto y **no se cambia en el backend**. La consecuencia es del lado del consumidor: **todo valor de dinero que llega por el cable se coercionar con `Number()` antes de operarlo** (formatear, comparar, sumar). Un `|| 0` **no protege**: un string no vacÃ­o es *truthy*, asÃ­ que `"12.00" || 0` evalÃºa a `"12.00"` (el string), y `.toFixed` â€”que solo existe en `Number`â€” lanza `TypeError`. **Origen de esta regla:** HALLAZGO 5 (F7.7e, 30 Sep 2026) â€” `ProductCard.jsx` hacÃ­a `(producto.price || 0).toFixed(2)` y tumbÃ³ la pantalla completa del POS.

### DT-02.4 â€” La verificaciÃ³n

| # | VerificaciÃ³n | CÃ³mo |
|---|---|---|
| V-06 | No existe `Float` en columnas de dinero | BÃºsqueda estÃ¡tica: `Float` en `models.py` + revisiÃ³n semÃ¡ntica |
| V-07 | No existe `toFixed(2)` en componentes | BÃºsqueda estÃ¡tica: `toFixed\(2\)` en `apps/` |
| V-08 | Existe un solo `formatMoney` | BÃºsqueda estÃ¡tica: `formatMoney` |
| V-09 | El redondeo estÃ¡ declarado | Prueba: `formatMoney(0.125)` â†’ `0.13` (half-up), no `0.12` |
| V-10 | El selector no convierte | Prueba: cambiar moneda no altera el valor numÃ©rico |
| V-11 | NingÃºn `.toFixed()` se aplica sobre un valor sin coercionar | BÃºsqueda estÃ¡tica de `.toFixed(` + **prueba con precio STRING** (el valor tal como lo devuelve la API) |

**Nota sobre V-07 y V-11 (por quÃ© V-07 sola no basta):** V-07 detecta que existe un `toFixed(2)`, pero **no distingue si es seguro**. El bug de HALLAZGO 5 (`(producto.price || 0).toFixed(2)`) pasaba V-07 sin problema: el `toFixed(2)` estaba ahÃ­, y el grep lo encontraba. Lo que V-07 no ve es que el valor **podÃ­a ser un string**. V-11 cierra ese hueco: no pregunta *"Â¿usas `toFixed`?"* sino *"Â¿coercionaste antes de usarlo?"*.

**Nota sobre los fixtures de dinero (regla de testing):** los fixtures de los gates usan el dinero **como STRING**, igual que la API real (`price: '12.00'`, no `price: 12.00`). Un fixture numÃ©rico **oculta** el bug: un nÃºmero sÃ­ tiene `.toFixed`, asÃ­ que la prueba pasa aunque el cÃ³digo real falle. **Origen:** HALLAZGO 5 â€” el gate de F7.7e no existÃ­a, y los gates previos usaban precios numÃ©ricos, por eso el bug solo apareciÃ³ con datos reales.

**Prueba de referencia:** por crear â€” `apps/api/tests/test_money_rounding.py` y `apps/shared/money.test.js`. **Ya existe (POS nuevo):** `apps/pos/src/components/ProductCard.f7_7e.test.jsx` (8 tests con precios STRING) es la primera prueba de V-11 en el POS nuevo.

### DT-02.5 â€” La matriz de cumplimiento

| MÃ³dulo | Cumple | Evidencia / Deuda |
|---|---|---|
| POS | âœ… | `tickets.total`, `ticket_items.unit_price/subtotal` en `Numeric(12,2)` |
| Caja | âœ… | `opening_float`, `physical_cash`, `amount` en `Numeric(12,2)` |
| CatÃ¡logo | âœ… | `products.price/cost` en `Numeric(12,2)` |
| HeladerÃ­a | âœ… | `base_price`, `price_per_scoop`, `unit_price` en `Numeric(12,2)` |
| Grandeza | âœ… | **Migrado en V23** (22 Sep 2026): 13 columnas a `Numeric(12,2)` (`b2b_price`, `cash_fund`, `cash_expected`, `cash_received`, `sale_amount`, `payment_received`, `change_given`, `total_exchange_amount`, `total_fresh_amount`, `unit_price`, `amount`, `total_amount`, `advance_payment`) |
| RRHH | âœ… | **Migrado en V23** (22 Sep 2026): 22 columnas a `Numeric(12,2)` (nÃ³mina, uniformes, fondo de cobertura, PSG, vacaciones, finiquito) |
| Pedidos | âœ… | **Migrado en V23** (22 Sep 2026): `orders.delivery_fee` a `Numeric(12,2)` |
| Almacenes | âš ï¸ Parcial | `cantidad_actual`, `stock_minimo` en `Float` (cantidades, no dinero â€” pero conviene revisar) |
| ProducciÃ³n | âš ï¸ Parcial | Pesos y gramos en `Float` (correcto para pesos, no para dinero) |
| Frontend (ERP viejo) | âŒ No cumple | 80 `toFixed(2)` en 12 componentes; no existe `formatMoney` |
| Frontend (POS nuevo) | âš ï¸ Parcial | Ya tiene formateadores que coercionan con `Number()` (`formatearMoneda` en `SalesReceipt`/`POSOverlays`/`TicketTemplate`/`CorteTicketTemplate`; `formatearPrecio` en `ProductCard`). **Falta** el `formatMoney` Ãºnico compartido (`apps/shared/money.js`, V24) y migrar los formateadores locales a Ã©l. |

**Deuda saldada (V23, 22 Sep 2026):** la migraciÃ³n `apps/api/migrations_applied/migrate_float_to_decimal.py` cubrÃ­a solo **10 columnas** (tickets, ticket_items, products, cash_sessions, cash_movements). V23 la extendiÃ³ con **36 columnas mÃ¡s** (Grandeza 13, RRHH 22, Pedidos 1), todas a `Numeric(12,2)`. Se conservaron deliberadamente en `Float` **9 columnas no monetarias** (GPS, distancias, porcentajes, puntajes). Respaldo previo: `database_backups/backup_pre_v23_decimal_20260922.sql`.

**Deuda pendiente (V24):** el **frontend** sigue sin `formatMoney` â€” hay ~80 `toFixed(2)` en 12 componentes. V23 solo corrigiÃ³ el **tipo** en BD y modelos; el formateador Ãºnico y la eliminaciÃ³n de `toFixed(2)` son alcance de V24.

---

## SECCIÃ“N 4 â€” DT-03: IDENTIDAD

### DT-03.1 â€” La regla

> **El UUID identifica. El folio comunica. Nunca se intercambian.**

### DT-03.2 â€” El ancla

| Capa | Archivo | QuÃ© hace |
|---|---|---|
| Almacenamiento | `apps/api/modules/*/models.py` | PK `String` con UUID (nuevo modelo) |
| PresentaciÃ³n | â€” | El folio es un campo aparte, visible al usuario |

### DT-03.3 â€” Las reglas derivadas

1. **La PK es UUID.** Un entero autoincremental no se puede fusionar entre sucursales sin colisiÃ³n.
2. **El folio es un campo de negocio.** Es visible, es corto, es humano. No es la PK.
3. **El folio no se reutiliza.** Un folio cancelado no se recicla.
4. **El UUID nunca se muestra al usuario.** El usuario ve el folio.

### DT-03.4 â€” La verificaciÃ³n

| # | VerificaciÃ³n | CÃ³mo |
|---|---|---|
| V-11 | Las PK nuevas son UUID | RevisiÃ³n del modelo de datos |
| V-12 | El folio es un campo aparte | RevisiÃ³n del modelo de datos |
| V-13 | El frontend no muestra UUIDs | RevisiÃ³n de interfaces |

### DT-03.5 â€” La matriz de cumplimiento

| MÃ³dulo | Cumple | Evidencia / Deuda |
|---|---|---|
| POS | âš ï¸ Parcial | `Ticket.id` es entero; `account_num` es el folio visible |
| Almacenes | âœ… | `MovimientoInventario.id` es `String` UUID |
| Grandeza | âš ï¸ Parcial | Pendiente de auditorÃ­a |
| Resto | âš ï¸ Parcial | Pendiente de auditorÃ­a |

---

## SECCIÃ“N 5 â€” DT-04: INVENTARIO

### DT-04.1 â€” La regla

> **El inventario es un ledger inmutable. Solo el mÃ³dulo dueÃ±o escribe. Nadie lee `products.stock`.**

### DT-04.2 â€” El ancla

| Capa | Archivo | QuÃ© hace |
|---|---|---|
| Almacenamiento | `apps/api/modules/warehouse/models.py` | `MovimientoInventario` (ledger), `WarehouseEvent` (idempotencia) |
| Contrato | `CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md` | P-01: el dueÃ±o de la tabla es el Ãºnico que la escribe |

### DT-04.3 â€” Las reglas derivadas

1. **El ledger es inmutable.** Un movimiento no se edita ni se borra. Se compensa con otro movimiento.
2. **La idempotencia se garantiza con `evento_id`.** Un evento repetido no duplica el movimiento.
3. **Nadie lee `products.stock`.** El stock se consulta al mÃ³dulo dueÃ±o, por operaciÃ³n.
4. **El stock es derivado, no almacenado.** Si se puede calcular del ledger, no se guarda.

### DT-04.4 â€” La verificaciÃ³n

| # | VerificaciÃ³n | CÃ³mo |
|---|---|---|
| V-14 | No existe `products.stock` en el cÃ³digo nuevo | BÃºsqueda estÃ¡tica |
| V-15 | Todo movimiento tiene `evento_id` | RevisiÃ³n del modelo |
| V-16 | No hay `UPDATE`/`DELETE` sobre el ledger | BÃºsqueda estÃ¡tica |

### DT-04.5 â€” La matriz de cumplimiento

| MÃ³dulo | Cumple | Evidencia / Deuda |
|---|---|---|
| Almacenes | âœ… | `MovimientoInventario` + `WarehouseEvent` con `evento_id` |
| POS | âš ï¸ Parcial | Emite `WarehouseEvent`; no escribe el ledger directamente |
| ProducciÃ³n | âš ï¸ Parcial | Pendiente de auditorÃ­a |
| Resto | âš ï¸ Parcial | Pendiente de auditorÃ­a |

---

## SECCIÃ“N 6 â€” DT-05: AUDITORÃA

### DT-05.1 â€” La regla

> **Toda operaciÃ³n que mueve dinero o inventario deja rastro: quiÃ©n, cuÃ¡ndo, quÃ©.**

### DT-05.2 â€” El ancla

| Capa | Archivo | QuÃ© hace |
|---|---|---|
| Almacenamiento | `apps/api/modules/audit/` | Registro de operaciones |
| Contrato | `CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md` | A-04: outbox transaccional |

### DT-05.3 â€” Las reglas derivadas

1. **Se registra `capturÃ³` y `cobrÃ³` por separado.** No siempre es la misma persona.
2. **El registro es transaccional con la operaciÃ³n.** Si la operaciÃ³n falla, el registro no queda.
3. **El registro es append-only.** No se edita ni se borra.
4. **El registro incluye el `terminal_id`.** Sin terminal, no se sabe desde dÃ³nde se hizo.

### DT-05.4 â€” La verificaciÃ³n

| # | VerificaciÃ³n | CÃ³mo |
|---|---|---|
| V-17 | Toda operaciÃ³n de dinero registra `capturÃ³`/`cobrÃ³` | RevisiÃ³n de servicios |
| V-18 | El registro es transaccional | RevisiÃ³n de transacciones |
| V-19 | El registro incluye `terminal_id` | RevisiÃ³n del modelo |

### DT-05.5 â€” La matriz de cumplimiento

| MÃ³dulo | Cumple | Evidencia / Deuda |
|---|---|---|
| POS | âœ… | AuditorÃ­a con `terminal_id`, `capturÃ³`, `cobrÃ³` |
| Caja | âœ… | Movimientos con usuario |
| Grandeza | âš ï¸ Parcial | Pendiente de auditorÃ­a |
| Resto | âš ï¸ Parcial | Pendiente de auditorÃ­a |

---

## SECCIÃ“N 6.5 â€” DT-06: CONFIGURACIÃ“N DEL NEGOCIO

### DT-06.1 â€” La regla

> **Los valores que afectan a todos los mÃ³dulos (zona horaria, moneda, sucursal, polÃ­tica de pago de pedidos) se declaran una sola vez, en el mÃ³dulo Vista General, y se persisten en `system_settings`. NingÃºn mÃ³dulo los define por su cuenta. NingÃºn mÃ³dulo los sobrescribe.**

### DT-06.2 â€” El ancla

| Capa | Archivo | QuÃ© hace |
|---|---|---|
| Almacenamiento | `system_settings` (tabla) | Guarda `business_timezone`, `business_currency`, `sucursal_id`, `orders.payment_policy`, `orders.deposit_percent` |
| SelecciÃ³n | **MÃ³dulo Vista General** | La interfaz donde el humano elige |
| DistribuciÃ³n | `apps/shared/TimezoneContext.jsx` | Contexto global de tiempo (existe hoy) |
| DistribuciÃ³n | `apps/shared/MoneyContext.jsx` | Contexto global de dinero (**por crear**) |
| DistribuciÃ³n | `services/politica_pedidos.py` (backend) | Lector de la polÃ­tica de pago con **default seguro** (**por crear**, F7.5.1) |
| Consumo | Cada mÃ³dulo | Lee el contexto; nunca define el valor |

**Los 4 valores transversales declarados:**

| Clave en `system_settings` | Tipo | Valores | Default seguro |
|---|---|---|---|
| `business_timezone` | texto | Zona IANA (ej. `America/Mexico_City`) | `America/Mexico_City` |
| `business_currency` | texto | CÃ³digo ISO (ej. `MXN`) | `MXN` |
| `sucursal_id` | texto | Identificador de sucursal | La sucursal principal |
| `orders.payment_policy` | enum | `SIN_PAGO` \| `ANTICIPO` \| `PAGO_COMPLETO` | **`PAGO_COMPLETO`** |
| `orders.deposit_percent` | entero (0â€“100) | Solo aplica si `payment_policy = ANTICIPO` | `50` |

> **La polÃ­tica de pago de pedidos** (aÃ±adida el 30 Sep 2026 por decisiÃ³n del dueÃ±o) define **cuÃ¡ndo
> el POS puede proyectar un pedido a `orders`**: sin pago (`SIN_PAGO`), con anticipo (`ANTICIPO` +
> `deposit_percent`), o solo cubierto al 100% (`PAGO_COMPLETO`). El POS la **lee**; nunca la define.
> Ver [`PLAN_DE_ABORDAJE_FASE_7_5_PEDIDOS.md`](./05-plan-de-construccion/PLAN_DE_ABORDAJE_FASE_7_5_PEDIDOS.md) Â§0.

### DT-06.3 â€” Las reglas derivadas

1. **Vista General es un mÃ³dulo del ERP, no del POS.** Es hermano del POS, no hijo. El POS lo consume, no lo contiene.
2. **Vista General es el Ãºnico lugar donde se declaran los valores transversales.** No hay un selector de zona horaria en Caja, ni un selector de moneda en Almacenes, ni un selector de polÃ­tica de pago en el POS.
3. **El valor se persiste en `system_settings`.** No se persiste en el frontend, ni en `localStorage`, ni en cada mÃ³dulo.
4. **El frontend lo distribuye por contexto.** Un contexto por valor transversal (`TimezoneContext`, `MoneyContext`). Los componentes lo consumen con un hook (`useTimezone()`, `useMoney()`).
5. **El valor tiene un solo endpoint de lectura.** `GET /settings` (o `GET /settings/timezone` + `GET /settings/currency`). No hay un endpoint por mÃ³dulo.
6. **Cambiar el valor no cambia los datos guardados.** Cambiar la zona horaria cambia cÃ³mo se **muestra** el tiempo; no reescribe los timestamps. Cambiar la moneda cambia el **sÃ­mbolo**; no reescribe los montos. (Esto es la cara de configuraciÃ³n de DT-01 y DT-02.)
7. **La ausencia de un valor transversal degrada al default seguro, nunca al permisivo.** Si un mÃ³dulo no puede leer un valor (porque Vista General no existe todavÃ­a o el endpoint falla), asume el valor **mÃ¡s conservador** y **nunca bloquea la operaciÃ³n**. Ejemplo: si el POS no puede leer `orders.payment_policy`, asume `PAGO_COMPLETO` â€” no regala comida. (Esto es DT-07 aplicado a la configuraciÃ³n.)

### DT-06.4 â€” La verificaciÃ³n

| # | VerificaciÃ³n | CÃ³mo |
|---|---|---|
| V-20 | Existe un solo lugar donde se declaran los valores transversales | BÃºsqueda estÃ¡tica: un solo componente selector |
| V-21 | Los valores se persisten en `system_settings` | RevisiÃ³n del modelo de datos |
| V-22 | Existe un contexto por valor transversal | BÃºsqueda estÃ¡tica: `TimezoneContext`, `MoneyContext` |
| V-23 | NingÃºn mÃ³dulo define su propio valor transversal | BÃºsqueda estÃ¡tica: no hay `business_timezone`/`business_currency`/`orders.payment_policy` fuera de `system_settings` |
| V-24 | Cambiar el valor no altera los datos guardados | Prueba: cambiar zona horaria no modifica timestamps en BD |
| V-25 | La ausencia de un valor transversal degrada al default seguro | Prueba: sin fila en `system_settings`, el lector devuelve `PAGO_COMPLETO` (no lanza) |

### DT-06.5 â€” La matriz de cumplimiento

| MÃ³dulo | Cumple | Evidencia / Deuda |
|---|---|---|
| Vista General | âœ… | **FASE 1 completada** (22 Sep 2026) â€” ver [`ESPECIFICACION_FUNCIONAL_VISTA_GENERAL.md`](./ESPECIFICACIONES%20DEL%20PROYECTO/ESPECIFICACION_FUNCIONAL_VISTA_GENERAL.md). Declara zona horaria, moneda y sucursal; 35 reglas VG; 4 interfaces. **Deuda:** la UI para `orders.payment_policy` se aÃ±ade al reconstruirse (F7.5b) |
| POS | âœ… | Consume `TimezoneContext`; no define zona horaria. **Deuda:** lee `orders.payment_policy` con default seguro (F7.5.1); no la define |
| Caja | âœ… | Consume el contexto; no define valores |
| Resto | âš ï¸ Parcial | Pendiente de verificar que ninguno define valores transversales |

**Nota importante:** Vista General **no se especifica aquÃ­**. Este documento solo declara **su rol transversal** (dÃ³nde se declaran los valores). Su especificaciÃ³n funcional completa (pantallas, campos, validaciones) es FASE 1 de su propio mÃ³dulo y se escribirÃ¡ cuando se reconstruya. Declarar el rol ahora evita que la IA constructora invente el selector en cada mÃ³dulo.

---

## SECCIÃ“N 6.6 â€” DT-07: LA INTELIGENCIA ARTIFICIAL DEL NEGOCIO

> **Origen de esta directriz:** aclaraciÃ³n explÃ­cita del dueÃ±o (29 Sep 2026). Se documenta aquÃ­
> para que **no exista posibilidad de confusiÃ³n** cuando se construyan los mÃ³dulos de IA.

### DT-07.1 â€” La regla

> **Las capacidades de inteligencia artificial (voz, visiÃ³n, OCR, NLU) se gestionan en un solo mÃ³dulo del ERP llamado "Centro de IA". NingÃºn mÃ³dulo del ERP â€”incluido el POSâ€” contiene el motor de IA ni importa sus dependencias (`torch`, `whisper`, `ultralytics`, `tesseract`). Cada mÃ³dulo consume la IA por contrato.**

### DT-07.2 â€” El ancla

| Capa | Archivo / MÃ³dulo | QuÃ© hace |
|---|---|---|
| Motor | `ai-local/` (contenedor `ia-local`) | Ejecuta los modelos (YOLO, Whisper, Ollama, Tesseract). Habla HTTP. |
| Gateway | `apps/api/modules/ai/` | Traduce HTTP del ERP al motor. **PolÃ­tica inviolable: cualquier fallo del motor â†’ 503 `IA_NO_DISPONIBLE`. Nunca propaga un 500 al mÃ³dulo consumidor.** |
| GestiÃ³n | **MÃ³dulo "Centro de IA"** (`apps/ai/`) | La interfaz donde el humano ve el estado del motor, entrena la visiÃ³n, diagnostica la voz y lee capturas (OCR). Es un **mÃ³dulo paraguas** del ERP. |
| Consumo | Cada mÃ³dulo (POS, Almacenes, Grandezaâ€¦) | Pide la capacidad por **contrato**; nunca importa el motor. |

### DT-07.3 â€” Las reglas derivadas

1. **El Centro de IA es un mÃ³dulo del ERP, no del POS.** Es hermano del POS, no hijo. El POS lo consume, no lo contiene. (SimetrÃ­a exacta con DT-06 y Vista General.)
2. **El Centro de IA vive en `apps/ai/`, no en `apps/pos/`.** Es transversal: la voz la usan el POS y Almacenes; la visiÃ³n la usan el POS y Almacenes; el estado del motor interesa a todos.
3. **NingÃºn mÃ³dulo importa el motor de IA.** El ERP solo conoce `AI_LOCAL_URL` y habla HTTP. Si el motor se cae, cada mÃ³dulo sigue operando en modo manual.
4. **La IA es asistiva, nunca bloqueante.** Un fallo de la IA **nunca** impide una venta, un cobro o un movimiento de inventario. (Cara de IA de la Regla de Oro #7.)
5. **La frontera se declara por contrato.** Cada capacidad de IA que un mÃ³dulo consume se declara en el registro de contratos (`apps/api/contracts/registry.py`) con su firma completa. Un mÃ³dulo **no** llama a la IA por convenciÃ³n implÃ­cita.
6. **El Centro de IA no decide datos de negocio.** El motor de IA nunca decide un `client_id` ni un `product_id`; solo **sugiere**. La decisiÃ³n es del humano (contrato human-in-the-loop).

### DT-07.4 â€” La verificaciÃ³n

| # | VerificaciÃ³n | CÃ³mo |
|---|---|---|
| V-25 | Existe un solo lugar donde se gestiona la IA | BÃºsqueda estÃ¡tica: un solo mÃ³dulo `apps/ai/` |
| V-26 | NingÃºn mÃ³dulo importa el motor de IA | BÃºsqueda estÃ¡tica: no hay `import torch`/`whisper`/`ultralytics` fuera de `ai-local/` |
| V-27 | Cada capacidad de IA consumida tiene su contrato declarado | RevisiÃ³n del registro de contratos |
| V-28 | Un fallo del motor no rompe al consumidor | Prueba: apagar `ia-local` y verificar que el POS sigue vendiendo |
| V-29 | El motor nunca decide un `client_id`/`product_id` | RevisiÃ³n del contrato human-in-the-loop |

### DT-07.5 â€” La matriz de cumplimiento

| MÃ³dulo | Cumple | Evidencia / Deuda |
|---|---|---|
| Centro de IA | â³ | **Documentado** en [`DOCUMENTACION_CENTRO_IA.md`](./ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_CENTRO_IA.md) (v27.4, 4 capacidades). **AÃºn no reconstruido** en el ERP nuevo. |
| POS | âš ï¸ | Consume voz y visiÃ³n por contrato. **Deuda:** el contrato de voz aÃºn no estÃ¡ declarado en el registro (se declara en F7.0). El contrato 17 (visiÃ³n) sÃ­ existe. |
| Almacenes | âš ï¸ | Consume voz. Pendiente de declarar su contrato. |
| Resto | âš ï¸ | Pendiente de verificar que ninguno importa el motor. |

**Nota importante:** el Centro de IA **no se especifica aquÃ­**. Este documento solo declara **su rol transversal** (dÃ³nde se gestiona la IA). Su especificaciÃ³n funcional completa ya existe en [`DOCUMENTACION_CENTRO_IA.md`](./ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_CENTRO_IA.md) y se reescribirÃ¡ cuando el mÃ³dulo se reconstruya. Declarar el rol ahora evita que la IA constructora invente un motor de IA dentro de cada mÃ³dulo.

### DT-07.6 â€” El estado de la frontera POS â†” Centro de IA (hoy)

| Capacidad | Contrato declarado | Estado |
|---|---|---|
| VisiÃ³n (conteo de pan) | `17 vision.reconocer_producto` | âœ… Declarado (proveedor: VisiÃ³n â†’ **debe reetiquetarse a "Centro de IA"**) |
| Voz (dictado manos libres) | â€” | âŒ **No declarado.** Se declara en F7.0. |
| NLU (interpretar intenciÃ³n) | â€” | âŒ **No declarado.** Se declara en F7.0. |
| OCR (leer capturas) | â€” | âŒ **No declarado.** Lo consume Grandeza, no el POS. |

**Consecuencia:** hoy el POS hablarÃ­a con el Centro de IA por **convenciÃ³n implÃ­cita**, no por contrato declarado. Eso viola la Regla Dura del proyecto (A-02). La Fase 7 del POS empieza por **cerrar esta brecha** (F7.0).

---

## SECCIÃ“N 6.7 â€” DT-08: LA CAPTURA POR VISIÃ“N CENITAL

> **Origen de esta directriz:** aclaraciÃ³n explÃ­cita del dueÃ±o (29 Sep 2026). Se documenta aquÃ­
> para que **no exista posibilidad de confusiÃ³n** cuando se construya el hardware de captura y
> cuando se ajuste el umbral de confianza de la visiÃ³n.

### DT-08.1 â€” La regla

> **La visiÃ³n del POS opera sobre una cÃ¡mara cenital (montada sobre el mostrador, mirando hacia abajo) con iluminaciÃ³n dedicada que elimina las sombras. El umbral de confianza de la visiÃ³n (RN-72, hoy 0.35) es un valor calibrado para ese montaje cenital, no un valor universal, y es configurable desde el Centro de IA. La captura por visiÃ³n es un flujo persistente de "escaneo de charola", no una captura bajo demanda.**

### DT-08.2 â€” El ancla

| Capa | Archivo / MÃ³dulo | QuÃ© hace |
|---|---|---|
| Hardware | CÃ¡mara cenital + iluminaciÃ³n dedicada | Punto de vista fijo y controlado sobre el mostrador. Elimina sombras y el "apuntado" manual del operador. |
| Contrato | `17 vision.reconocer_producto` | Recibe el frame y devuelve detecciones con confianza. Debe declarar el `modo_captura` (`cenital` \| `manual`). |
| Umbral | RN-72 (`rules/registry.py`) | 0.35 es el valor **calibrado para el modo cenital**. Configurable desde el Centro de IA. |
| Consumo | `apps/pos/src/hooks/useVision.js` + `VisionVisor.jsx` | Visor persistente que sugiere productos de la charola. Nunca agrega al carrito solo. |

### DT-08.3 â€” Las reglas derivadas

1. **El punto de vista es fijo y controlado.** La cÃ¡mara cenital con iluminaciÃ³n dedicada convierte la visiÃ³n en un "escÃ¡ner de charola": el operador coloca los productos y el sistema los reconoce sin apuntar. Esto es lo que hace que la visiÃ³n **agilice** la captura en lugar de estorbarla.
2. **El umbral 0.35 es de calibraciÃ³n, no universal.** RN-72 fija 0.35 como el valor calibrado para el montaje cenital con iluminaciÃ³n controlada. No es un valor mÃ¡gico ni aplicable a una webcam de laptop. Es **configurable desde el Centro de IA** (DT-07).
3. **La captura es un flujo persistente, no bajo demanda.** El visor cenital permanece abierto durante la venta (modo "escÃ¡ner de charola"), no se abre y cierra por cada producto. Esto es una consecuencia directa del punto de vista fijo.
4. **La visiÃ³n sigue siendo asistiva (RN-74).** Aunque el montaje sea controlado, la visiÃ³n **sugiere**, nunca decide. El humano confirma. La regla de oro no cambia con el hardware.
5. **El `modo_captura` se declara en el contrato.** El contrato 17 distingue `cenital` de `manual` para que el Centro de IA pueda aplicar la calibraciÃ³n correcta y para que el POS sepa quÃ© flujo de UI ofrecer.
6. **La degradaciÃ³n elegante no cambia.** Si el Centro de IA o la cÃ¡mara fallan, el POS sigue vendiendo en modo manual (DT-07, regla derivada 4).

### DT-08.4 â€” La verificaciÃ³n

| # | VerificaciÃ³n | CÃ³mo |
|---|---|---|
| V-30 | El contrato 17 declara el `modo_captura` | RevisiÃ³n del registro de contratos |
| V-31 | El umbral de visiÃ³n es configurable, no hardcodeado | RevisiÃ³n: RN-72 lee el umbral de configuraciÃ³n del Centro de IA |
| V-32 | El visor cenital es persistente, no bajo demanda | RevisiÃ³n de `VisionVisor.jsx`: el visor no se monta/desmonta por producto |
| V-33 | La visiÃ³n nunca agrega al carrito sola | Prueba: una detecciÃ³n no produce una lÃ­nea de carrito sin confirmaciÃ³n (RN-74) |

### DT-08.5 â€” La matriz de cumplimiento

| MÃ³dulo | Cumple | Evidencia / Deuda |
|---|---|---|
| Centro de IA | â³ | Debe exponer el umbral configurable y la calibraciÃ³n cenital. **AÃºn no reconstruido.** |
| POS | âš ï¸ | `useVision` + `VisionVisor` consumen el contrato 17. **Deuda:** el contrato 17 aÃºn no declara `modo_captura` (se ajusta en F7.3). |
| Almacenes | â³ | PodrÃ­a reutilizar la captura cenital para recepciÃ³n de mercancÃ­a. Pendiente. |

**Nota importante:** esta directriz **no especifica el hardware**. Solo declara que la visiÃ³n del POS asume un montaje cenital con iluminaciÃ³n controlada, y que el umbral 0.35 es de calibraciÃ³n (configurable), no universal. Documentarlo ahora evita que un futuro ajuste del umbral se haga a ciegas o que alguien asuma que la visiÃ³n funciona igual con una webcam de laptop.

---

## SECCIÃ“N 7 â€” LA MATRIZ MAESTRA

Estado de cada mÃ³dulo frente a cada directriz. **Un mÃ³dulo no se declara terminado con un âŒ.**

| MÃ³dulo | DT-01 Tiempo | DT-02 Dinero | DT-03 Identidad | DT-04 Inventario | DT-05 AuditorÃ­a | DT-06 Config | DT-07 IA | DT-08 VisiÃ³n | DT-09 Responsivo |
|---|---|---|---|---|---|---|---|---|---|
| **Vista General** | â€” | â€” | â€” | â€” | â€” | â³ | â€” | â€” | âš ï¸ |
| **Centro de IA** | â€” | â€” | â€” | â€” | â€” | â€” | â³ | â³ | âš ï¸ |
| POS | âœ… | âœ… | âš ï¸ | âš ï¸ | âœ… | âœ… | âš ï¸ | âš ï¸ | âœ… |
| Caja | âœ… | âœ… | âš ï¸ | â€” | âœ… | âœ… | â€” | â€” | âš ï¸ |
| CatÃ¡logo | â€” | âœ… | âš ï¸ | âš ï¸ | âš ï¸ | âš ï¸ | â€” | â€” | âš ï¸ |
| HeladerÃ­a | â€” | âœ… | âš ï¸ | â€” | âš ï¸ | âš ï¸ | â€” | â€” | âš ï¸ |
| Almacenes | âš ï¸ | âš ï¸ | âœ… | âœ… | âš ï¸ | âš ï¸ | âš ï¸ | â³ | âœ… |
| Grandeza | âš ï¸ | âŒ | âš ï¸ | â€” | âš ï¸ | âš ï¸ | âš ï¸ | â€” | âš ï¸ |
| RRHH | âš ï¸ | âŒ | âš ï¸ | â€” | âš ï¸ | âš ï¸ | â€” | â€” | âœ… |
| Pedidos | âš ï¸ | âŒ | âš ï¸ | â€” | âš ï¸ | âš ï¸ | â€” | â€” | âš ï¸ |
| ProducciÃ³n | âš ï¸ | âš ï¸ | âš ï¸ | âš ï¸ | âš ï¸ | âš ï¸ | â€” | â€” | âš ï¸ |
| Analytics | âœ… | âš ï¸ | â€” | â€” | â€” | âš ï¸ | â€” | â€” | âš ï¸ |
| Frontend | âš ï¸ | âŒ | âš ï¸ | â€” | â€” | âš ï¸ | â€” | â€” | âš ï¸ |

**Leyenda:** âœ… Cumple Â· âš ï¸ Parcial Â· âŒ No cumple Â· â³ Pendiente Â· â€” No aplica

**Nota sobre Vista General:** es el mÃ³dulo donde se **declaran** los valores transversales. Su fila estÃ¡ en â³ porque aÃºn no se reconstruye, pero su rol ya estÃ¡ decidido (DT-06). Los demÃ¡s mÃ³dulos lo consumen.

**Nota sobre el Centro de IA:** es el mÃ³dulo donde se **gestionan** las capacidades de IA (DT-07) y donde se **calibra** el umbral de la visiÃ³n cenital (DT-08). Su fila estÃ¡ en â³ porque aÃºn no se reconstruye, pero su rol ya estÃ¡ decidido. Los demÃ¡s mÃ³dulos lo consumen por contrato.

---

## SECCIÃ“N 8 â€” CÃ“MO SE USA ESTE DOCUMENTO AL ARREGLAR EL ERP

El usuario va a arreglar el ERP **mÃ³dulo por mÃ³dulo**. Este documento es la guÃ­a de ese trabajo.

**El procedimiento, por cada mÃ³dulo:**

1. **Antes de tocar el mÃ³dulo**, se recorre la matriz maestra y se anota quÃ© directrices estÃ¡n en âš ï¸ o âŒ para ese mÃ³dulo.
2. **Se arregla el mÃ³dulo** segÃºn su propio plan (FASE 1 a FASE 6 de la metodologÃ­a).
3. **Al terminar**, se vuelve a recorrer la matriz. Las directrices que estaban en âŒ deben pasar a âœ… o a âš ï¸ con una deuda declarada.
4. **Se actualiza la matriz maestra** en este documento.

**Lo que NO se hace:**

- **No se arregla una directriz transversal "de golpe" en todos los mÃ³dulos.** Se arregla mÃ³dulo por mÃ³dulo, cuando el mÃ³dulo se toca. Arreglar los 10 mÃ³dulos a la vez es la forma mÃ¡s segura de romper el ERP que corre.
- **No se toca el ERP que corre sin autorizaciÃ³n.** Este documento vive en el repo de planos. Cuando una directriz se va a aplicar al ERP, se pide autorizaciÃ³n explÃ­cita.
- **No se declara una directriz "cumplida" sin la verificaciÃ³n.** La matriz se actualiza con evidencia, no con optimismo.

---

## SECCIÃ“N 9 â€” LO QUE ESTE DOCUMENTO NO ES

- **No es un documento de mÃ³dulo.** No describe cÃ³mo funciona Almacenes ni Caja. Eso estÃ¡ en los documentos 1 a 11.
- **No es la metodologÃ­a.** La metodologÃ­a dice *cÃ³mo* se construye un mÃ³dulo. Este documento dice *quÃ© reglas debe cumplir* al construirse.
- **No es un plan de acciÃ³n.** No tiene fases ni fechas. Es una referencia que se consulta.
- **No reemplaza a los criterios de aceptaciÃ³n.** Los criterios (Documento 10) dicen cuÃ¡ndo un mÃ³dulo estÃ¡ terminado. Este documento dice quÃ© reglas transversales debe cumplir para estarlo. Se complementan: CA-21 (dinero) es la cara de aceptaciÃ³n de DT-02.

---

## SECCIÃ“N 10 â€” CIERRE

Este documento existe porque el mismo error se cometiÃ³ muchas veces en lugares distintos:

- El tiempo, en 5 lugares.
- El dinero, en 80 lugares.
- La identidad, confundida en varios mÃ³dulos.
- El inventario, escrito desde fuera.

**La regla que lo resume todo:**

> **Lo transversal se decide una vez y se verifica en cada mÃ³dulo. Nunca se decide en cada mÃ³dulo.**

**Las 9 directrices vigentes:**

| ID | Directriz | Estado |
|---|---|---|
| DT-01 | Tiempo: UTC almacena, local muestra | Vigente |
| DT-02 | Dinero: `Numeric(12,2)` guarda, un formateador muestra, el selector declara; **el dinero viaja como STRING por el cable y se coercionar con `Number()`** | Vigente |
| DT-03 | Identidad: UUID identifica, folio comunica | Vigente |
| DT-04 | Inventario: ledger inmutable, solo el dueÃ±o escribe | Vigente |
| DT-05 | AuditorÃ­a: quiÃ©n, cuÃ¡ndo, quÃ© â€” transaccional | Vigente |
| DT-06 | ConfiguraciÃ³n: los valores transversales se declaran una sola vez, en Vista General | Vigente |
| DT-07 | IA: las capacidades de IA se gestionan una sola vez, en el Centro de IA; cada mÃ³dulo las consume por contrato | Vigente |
| DT-08 | VisiÃ³n cenital: la captura por visiÃ³n asume cÃ¡mara cenital con iluminaciÃ³n dedicada; el umbral 0.35 es de calibraciÃ³n (configurable), no universal | Vigente |
| DT-09 | Responsividad y lenguaje visual: todo componente nace responsivo (R-01 a R-04), tÃ¡ctil (â‰¥44px), y respeta los tokens de diseÃ±o de la nueva arquitectura | Vigente |

**La simetrÃ­a completa (DT-06):**

```
system_settings  â†’  Vista General  â†’  contexto global  â†’  cada mÃ³dulo
   (guarda)           (declara)        (distribuye)        (consume)
```

Hoy estÃ¡n documentados el primero, el tercero y el cuarto. **DT-06 declara el segundo**, que era el eslabÃ³n que faltaba.

**La simetrÃ­a completa (DT-07):**

```
ai-local (motor)  â†’  Centro de IA  â†’  Gateway (503)  â†’  cada mÃ³dulo
   (ejecuta)          (gestiona)       (traduce)         (consume por contrato)
```

**DT-07 declara el rol del Centro de IA**, que era el eslabÃ³n que faltaba para que el POS (y Almacenes, y Grandeza) no inventen un motor de IA dentro de sÃ­ mismos.

**La simetrÃ­a completa (DT-08):**

```
cÃ¡mara cenital + luz  â†’  contrato 17 (modo_captura)  â†’  umbral calibrado  â†’  visor persistente
   (captura fija)          (declara el modo)            (configurable)        (sugiere, no decide)
```

**DT-08 declara el montaje de captura de la visiÃ³n**, que era el supuesto implÃ­cito detrÃ¡s del umbral 0.35 (RN-72). Sin esta directriz, un futuro ajuste del umbral se harÃ­a a ciegas o alguien asumirÃ­a que la visiÃ³n funciona igual con una webcam de laptop.

**Documentos relacionados:**

- `METODOLOGIA_DE_INGENIERIA_INVERSA_Y_DISENO.md` â€” Â§3.4 (reglas transversales), FASE 2 (casilla de dinero), Â§9 (errores 9 y 10)
- `CRITERIOS_DE_ACEPTACION_DEL_NUEVO_POS.md` â€” CA-21 (dinero), CA-13 a CA-16 (tiempo)
- `MODELO_DE_DATOS_DEL_NUEVO_POS.md` â€” C-01 (UUID), C-02 (UTC), C-03 (Numeric), O-19 (business_currency)
- `CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md` â€” P-01 a P-03, A-01 a A-06
- `PLANO ARQUITECTONICO PARA EL NUEVO POS.md` â€” SecciÃ³n 6 (mÃ³dulos fuera del alcance del POS)
- `README.md` â€” reglas de oro 9 (tiempo) y 11 (dinero)
- `ESPECIFICACIONES DEL PROYECTO/DOCUMENTACION_CENTRO_IA.md` â€” la especificaciÃ³n funcional del Centro de IA (DT-07)
- `docs/SPEC_AI_GATEWAY_TRANSVERSAL.md` â€” la especificaciÃ³n del Gateway transversal (DT-07)
- `05-plan-de-construccion/PLAN_DE_ABORDAJE_FASE_7_POR_PARTES.md` â€” Â§8 (sub-fase F7.3, visiÃ³n cenital) (DT-08)
- `PLAN_MAESTRO_DEFINITIVO_POS.md` â€” Â§Fase 7 (nota de la cÃ¡mara cenital) (DT-08)

---

## SECCIÃ“N 12 â€” DT-04: TRANSACCIONALIDAD DE CARRITOS Y FOLIOS

### DT-04.1 â€” La regla

> **El carrito vive en memoria volÃ¡til pura hasta su envÃ­o explÃ­cito. La persistencia es atÃ³mica con reintentos (`withRetries`), los folios se generan 100% en backend, y la limpieza de pantalla tras envÃ­o exige verificaciÃ³n post-creaciÃ³n, nunca un ciego HTTP 200.**

### DT-04.2 â€” El contexto y las reglas secundarias

Para evitar incidentes histÃ³ricos de sobreescrituras por closures ("folios quemados" por pruebas de operadores, pÃ©rdida de comandas por fallos de red), se aplican las siguientes reglas inquebrantables a **cualquier mÃ³dulo del ERP que maneje compras, ventas o carritos** (POS, HeladerÃ­a, Pedidos, ProducciÃ³n):

1. **Cero Auto-Saves Masivos:** Prohibido usar temporizadores (`setInterval`) para el estado del carrito. Mientras el usuario estÃ© seleccionando productos de prueba o armando el carrito, los datos deben vivir exclusivamente en la memoria volÃ¡til del frontend.
2. **Atomicidad por Ã­tem:** Todo agregado/borrado de Ã­tem es atÃ³mico hacia la base de datos (si el sistema no opera puramente en local), incorporando reintentos automÃ¡ticos (`withRetries`) ante micro-cortes de red.
3. **VerificaciÃ³n Post-EnvÃ­o Obligatoria (Anti-Falsos Positivos):** El frontend jamÃ¡s debe limpiar el carrito basÃ¡ndose en un Ã©xito ciego (HTTP 200 aislado). Tras enviar la comanda, se debe ejecutar una verificaciÃ³n secundaria (GET) para confirmar que los datos se han persistido. Si la red falla, se debe aplicar un bloqueo visual (ej. banner rojo persistente) reteniendo los datos para evitar que el cajero pierda el pedido.
4. **GestiÃ³n de Folios 100% Backend:** El frontend jamÃ¡s calcula, asume o genera folios. Toda la numeraciÃ³n es generada atÃ³micamente por secuencias de PostgreSQL. El backend puede implementar lÃ³gica de reciclaje de folios vacÃ­os creados en los Ãºltimos 5 minutos para mitigar el impacto de pruebas rÃ¡pidas.
5. **Garbage Collector AsÃ­ncrono:** El backend ejecuta una limpieza asÃ­ncrona para eliminar fÃ­sicamente borradores vacÃ­os muy antiguos, y mandar a estatus `CANCELLED` los borradores con productos abandonados por mÃ¡s de 24 horas, manteniendo intacta la trazabilidad para auditorÃ­a.

---

## SECCIÃ“N 13 â€” DT-05: GESTIÃ“N DE TERMINALES, SESIONES Y CANDADOS

### DT-05.1 â€” La regla

> **Toda terminal habilitada como caja opera bajo exclusividad garantizada por un candado de latido activo (Heartbeat). El cÃ³digo del frontend debe usar referencias primitivas inmutables para el ciclo de vida del candado, y el flujo de caja debe concluir atÃ³micamente liberando la estaciÃ³n al pool general.**

### DT-05.2 â€” Reglas Inquebrantables (AdiÃ³s a las Terminales Fantasma)

Para erradicar definitivamente los incidentes histÃ³ricos de estaciones bloqueadas indefinidamente ("Gavetas Secuestradas"), bucles de reconexiÃ³n por pÃ©rdida de sesiÃ³n y falsos desbloqueos por referencias inestables, cualquier mÃ³dulo que maneje terminales fÃ­sicas debe regirse por los siguientes principios:

1. **Candado por Latido Activo (Heartbeat y TTL)**
   - **Exclusividad de OperaciÃ³n:** Cuando una estaciÃ³n es habilitada como caja o zona de cobro, el sistema adquiere un candado digital vinculado exclusivamente a ese usuario y terminal, impidiendo el uso simultÃ¡neo o la mezcla de turnos.
   - **ExpiraciÃ³n AutomÃ¡tica por Ausencia:** El candado depende de un latido constante (heartbeat) emitido periÃ³dicamente (ej. cada 10 segundos) desde el cliente. Si la pestaÃ±a se cierra de golpe, la aplicaciÃ³n se apaga o la red se interrumpe de forma permanente, los latidos cesan. El backend detectarÃ¡ la inactividad y liberarÃ¡ el candado automÃ¡ticamente, evitando que la terminal quede bloqueada como "fantasma".

2. **Estabilidad de Referencias en el Frontend (Cero Falsos Desbloqueos)**
   - **Inmutabilidad del Estado de SesiÃ³n:** Queda estrictamente prohibido utilizar objetos volÃ¡tiles o referencias dinÃ¡micas de React directamente en los arrays de dependencia de los ciclos de vida (`useEffect`) que controlan las peticiones de desbloqueo.
   - Se deben emplear exclusivamente primitivos estables (`usuarioRef.current`, `terminalRef.current`) para garantizar que la comunicaciÃ³n en segundo plano corra de forma limpia sin interrumpir al cajero ni disparar Ã³rdenes de `unlock` accidentales durante la venta.

3. **Ciclo de Cierre y LiberaciÃ³n AtÃ³mica (Corte de Caja)**
   - **Retorno Obligatorio al Estado Base:** El flujo operativo de una caja debe concluir formalmente mediante un proceso de cierre explÃ­cito (ej. `ESTADOS.CIERRE` en el Corte de Caja).
   - Una vez cotejado el flujo fÃ­sico (dinero en caja) contra el sistema y confirmado el cierre del turno, la aplicaciÃ³n debe disparar de forma atÃ³mica la orden de liberaciÃ³n (`liberarLock`), apagando el latido y devolviendo inmediatamente la terminal al pool general (estado `free`), dejÃ¡ndola limpia y disponible para el siguiente operador o turno.

