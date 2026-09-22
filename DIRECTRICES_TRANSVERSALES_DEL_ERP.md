# DIRECTRICES TRANSVERSALES DEL ERP

**Documento 12 del proyecto del Nuevo POS**
**Estado:** Vigente
**Ámbito:** Todo el ERP (todos los módulos, todas las sucursales)
**Naturaleza:** Compendio de reglas que **no pertenecen a un módulo**. Pertenecen a todos.

---

## SECCIÓN 0 — CÓMO LEER ESTE DOCUMENTO

Este documento no describe un módulo. Describe las **reglas que atraviesan todos los módulos**.

La diferencia es importante:

- Un documento de módulo dice: *"así funciona Almacenes"*.
- Una directriz transversal dice: *"así se comporta el tiempo en Almacenes, en Caja, en RRHH y en Grandeza — y así se verifica"*.

**Una directriz transversal no se "aplica" a un módulo. Se "verifica" contra un módulo.**

Por eso cada directriz tiene cuatro elementos obligatorios:

| Elemento | Qué es | Por qué |
|---|---|---|
| **La regla** | La norma, en una frase que no admite interpretación | Sin ambigüedad no hay verificación |
| **El ancla** | El archivo y la línea donde vive la implementación canónica | Sin ancla, la regla es una opinión |
| **La verificación** | Cómo se comprueba que un módulo la cumple | Sin verificación, la regla es un deseo |
| **La matriz** | El estado de cada módulo frente a la regla | Sin matriz, no se sabe qué falta |

**Regla de uso:** cuando se construye o se revisa un módulo, se recorre esta lista y se marca cada directriz como **Cumple**, **Parcial** o **No cumple**. Un módulo no se declara terminado con una directriz en "No cumple".

---

## SECCIÓN 1 — EL PRINCIPIO QUE LAS UNIFICA

Todas las directrices de este documento nacen del mismo principio:

> **Lo que es igual en todos los módulos no se decide en cada módulo. Se decide una vez, aquí, y se verifica en cada módulo.**

Esto no es una preferencia de estilo. Es una consecuencia de haber encontrado el mismo problema repetido en varios módulos:

- El **tiempo** estaba implementado en **5 lugares distintos** (`core/timezone.py`, `core/timestamps.py`, `core/serialization.py`, `apps/shared/timezone.js`, `apps/shared/TimezoneContext.jsx`).
- El **dinero** estaba formateado en **80 lugares distintos** (`toFixed(2)` suelto en 12 componentes), sin un solo formateador.
- La **identidad** (UUID vs folio) se confundía en varios módulos.
- El **inventario** se escribía desde fuera del módulo dueño (`products.stock`).

Cada uno de esos es un caso del mismo error: **una decisión transversal tomada localmente, muchas veces, de formas distintas.**

---

## SECCIÓN 2 — DT-01: TIEMPO

### DT-01.1 — La regla

> **Todo timestamp se guarda en UTC. Se muestra en hora local. La conversión es de presentación, nunca de almacenamiento.**

### DT-01.2 — El ancla

| Capa | Archivo | Qué hace |
|---|---|---|
| Almacenamiento | `apps/api/core/timestamps.py` | `utcnow()` — la única fuente de "ahora" |
| Configuración | `apps/api/core/timezone.py` | `get_business_tz(db)` lee `business_timezone` de `system_settings` (caché 5 min) |
| Cálculo | `apps/api/core/timezone.py` | `local_now`, `utc_to_local`, `local_day_bounds_utc`, `to_local_date_str` |
| Transporte | `apps/api/core/serialization.py` | `iso_utc()` — serializa siempre con marca UTC |
| Presentación | `apps/shared/timezone.js` | `formatLocal`, `formatLocalTime`, `formatLocalDate`, `parseUtc` |
| Presentación | `apps/shared/TimezoneContext.jsx` | `TimezoneProvider` + `useTimezone()` — el contexto global |

### DT-01.3 — Las reglas derivadas

1. **Nunca `datetime.now()`.** Siempre `utcnow()`. `datetime.now()` devuelve la hora del servidor, que no es la hora del negocio.
2. **Nunca `datetime.utcnow()` de Python 3.12+.** Está deprecado y devuelve un naive. Usar `utcnow()` del proyecto.
3. **Toda columna de tiempo es `DateTime(timezone=True)`.** Un `DateTime` naive es una bomba de tiempo: no se sabe en qué zona está.
4. **Los límites del día se calculan en hora local, no en UTC.** `local_day_bounds_utc(tz, fecha)` devuelve el rango UTC que corresponde al día local. Un ticket de las 23:30 local pertenece al día local, no al día UTC.
5. **Las columnas `Date` (como `journey_date`) son fechas locales.** No se convierten. Son la fecha del calendario del negocio.
6. **El frontend nunca formatea fechas por su cuenta.** Usa `useTimezone()`. Un `new Date().toLocaleString()` suelto es una violación.

### DT-01.4 — La verificación

| # | Verificación | Cómo |
|---|---|---|
| V-01 | No existe `datetime.now()` en el código de producción | Búsqueda estática: `datetime\.now\(\)` |
| V-02 | No existe `datetime.utcnow()` | Búsqueda estática: `datetime\.utcnow\(\)` |
| V-03 | No existe `DateTime` sin `timezone=True` en columnas nuevas | Revisión del modelo de datos |
| V-04 | El frontend no usa `toLocaleString`/`toLocaleDateString` fuera de `timezone.js` | Búsqueda estática en `apps/` |
| V-05 | Los reportes por día usan `local_day_bounds_utc` | Prueba: ticket a las 23:30 local aparece en el día local |

**Prueba de referencia:** `apps/api/tests/test_bloque9d_3bugs.py` (los 3 bugs de zona horaria). Es el guardián de esta directriz.

### DT-01.5 — La matriz de cumplimiento

| Módulo | Cumple | Evidencia / Deuda |
|---|---|---|
| POS | ✅ | `test_bloque9d_3bugs.py` cubre los 3 bugs |
| Caja | ✅ | Usa `local_day_bounds_utc` en `generar_reporte_diario` |
| Analytics | ✅ | Corregido en el bloque 9d |
| Grandeza | ⚠️ Parcial | `journey_date` es `Date` local (correcto), pero hay `Intl.DateTimeFormat('en-US')` suelto en `GrandezaDriverUI.jsx:83` |
| Almacenes | ⚠️ Parcial | `MovimientoInventario.timestamp` sin verificar `timezone=True` |
| RRHH | ⚠️ Parcial | Pendiente de auditoría |
| Producción | ⚠️ Parcial | Pendiente de auditoría |
| Pedidos | ⚠️ Parcial | Pendiente de auditoría |

**Hallazgo previo:** `DOCUMENTACION_MODULO_POS.md:360` (H5) ya había detectado que *"no existía una regla explícita de timestamps en la documentación, pese a ser un principio transversal"*. Esta directriz cierra ese hallazgo.

---

## SECCIÓN 3 — DT-02: DINERO

### DT-02.1 — La regla

> **El dinero se guarda en `Numeric(12,2)`, nunca en `Float`. La moneda del negocio (`business_currency`) declara en qué moneda se captura; no convierte. La presentación pasa por un solo formateador (`formatMoney`); ningún componente formatea dinero por su cuenta.**

### DT-02.2 — El ancla

| Capa | Archivo | Qué hace |
|---|---|---|
| Almacenamiento | `apps/api/modules/*/models.py` | Columnas de dinero en `Numeric(12,2)` |
| Configuración | `system_settings.business_currency` | **✅ Creado en V23** (22 Sep 2026) — sembrado en `seed_settings()`, expuesto en `GET /settings/currency` |
| Presentación | `apps/shared/money.js` | **Por crear (V24)** — `formatMoney`, `parseMoney`, `roundMoney` |
| Presentación | `apps/shared/MoneyContext.jsx` | **Por crear (V24)** — espejo de `TimezoneContext.jsx` |

### DT-02.3 — Las reglas derivadas

1. **Nunca `Float` para dinero.** `Float` no representa `0.10` exactamente. `Numeric(12,2)` sí. Un centavo perdido por ticket, multiplicado por miles de tickets, es un descuadre de caja.
2. **El redondeo es half-up y se declara.** `Intl.NumberFormat` usa half-even por defecto (bancario). El negocio usa half-up (comercial). La diferencia se declara explícitamente, no se hereda.
3. **Un solo formateador.** `formatMoney(monto, moneda)`. Ningún componente usa `toFixed(2)`, `Intl.NumberFormat` ni concatenación de símbolo.
4. **El selector de moneda declara, no convierte.** Cambiar la moneda cambia el **símbolo**, nunca el **número**. Un selector que cambia el número es un conversor de divisas, y un conversor de divisas descuadra la caja.
5. **La moneda se guarda una sola vez, en la configuración del negocio.** No se guarda por ticket, ni por producto, ni por sucursal (salvo que la sucursal opere en otra moneda, lo cual es una decisión explícita y documentada).
6. **El dinero no se suma en el frontend.** Los totales vienen del backend. El frontend solo formatea.

### DT-02.4 — La verificación

| # | Verificación | Cómo |
|---|---|---|
| V-06 | No existe `Float` en columnas de dinero | Búsqueda estática: `Float` en `models.py` + revisión semántica |
| V-07 | No existe `toFixed(2)` en componentes | Búsqueda estática: `toFixed\(2\)` en `apps/` |
| V-08 | Existe un solo `formatMoney` | Búsqueda estática: `formatMoney` |
| V-09 | El redondeo está declarado | Prueba: `formatMoney(0.125)` → `0.13` (half-up), no `0.12` |
| V-10 | El selector no convierte | Prueba: cambiar moneda no altera el valor numérico |

**Prueba de referencia:** por crear — `apps/api/tests/test_money_rounding.py` y `apps/shared/money.test.js`.

### DT-02.5 — La matriz de cumplimiento

| Módulo | Cumple | Evidencia / Deuda |
|---|---|---|
| POS | ✅ | `tickets.total`, `ticket_items.unit_price/subtotal` en `Numeric(12,2)` |
| Caja | ✅ | `opening_float`, `physical_cash`, `amount` en `Numeric(12,2)` |
| Catálogo | ✅ | `products.price/cost` en `Numeric(12,2)` |
| Heladería | ✅ | `base_price`, `price_per_scoop`, `unit_price` en `Numeric(12,2)` |
| Grandeza | ✅ | **Migrado en V23** (22 Sep 2026): 13 columnas a `Numeric(12,2)` (`b2b_price`, `cash_fund`, `cash_expected`, `cash_received`, `sale_amount`, `payment_received`, `change_given`, `total_exchange_amount`, `total_fresh_amount`, `unit_price`, `amount`, `total_amount`, `advance_payment`) |
| RRHH | ✅ | **Migrado en V23** (22 Sep 2026): 22 columnas a `Numeric(12,2)` (nómina, uniformes, fondo de cobertura, PSG, vacaciones, finiquito) |
| Pedidos | ✅ | **Migrado en V23** (22 Sep 2026): `orders.delivery_fee` a `Numeric(12,2)` |
| Almacenes | ⚠️ Parcial | `cantidad_actual`, `stock_minimo` en `Float` (cantidades, no dinero — pero conviene revisar) |
| Producción | ⚠️ Parcial | Pesos y gramos en `Float` (correcto para pesos, no para dinero) |
| Frontend (todos) | ❌ No cumple | 80 `toFixed(2)` en 12 componentes; no existe `formatMoney` |

**Deuda saldada (V23, 22 Sep 2026):** la migración `apps/api/migrations_applied/migrate_float_to_decimal.py` cubría solo **10 columnas** (tickets, ticket_items, products, cash_sessions, cash_movements). V23 la extendió con **36 columnas más** (Grandeza 13, RRHH 22, Pedidos 1), todas a `Numeric(12,2)`. Se conservaron deliberadamente en `Float` **9 columnas no monetarias** (GPS, distancias, porcentajes, puntajes). Respaldo previo: `database_backups/backup_pre_v23_decimal_20260922.sql`.

**Deuda pendiente (V24):** el **frontend** sigue sin `formatMoney` — hay ~80 `toFixed(2)` en 12 componentes. V23 solo corrigió el **tipo** en BD y modelos; el formateador único y la eliminación de `toFixed(2)` son alcance de V24.

---

## SECCIÓN 4 — DT-03: IDENTIDAD

### DT-03.1 — La regla

> **El UUID identifica. El folio comunica. Nunca se intercambian.**

### DT-03.2 — El ancla

| Capa | Archivo | Qué hace |
|---|---|---|
| Almacenamiento | `apps/api/modules/*/models.py` | PK `String` con UUID (nuevo modelo) |
| Presentación | — | El folio es un campo aparte, visible al usuario |

### DT-03.3 — Las reglas derivadas

1. **La PK es UUID.** Un entero autoincremental no se puede fusionar entre sucursales sin colisión.
2. **El folio es un campo de negocio.** Es visible, es corto, es humano. No es la PK.
3. **El folio no se reutiliza.** Un folio cancelado no se recicla.
4. **El UUID nunca se muestra al usuario.** El usuario ve el folio.

### DT-03.4 — La verificación

| # | Verificación | Cómo |
|---|---|---|
| V-11 | Las PK nuevas son UUID | Revisión del modelo de datos |
| V-12 | El folio es un campo aparte | Revisión del modelo de datos |
| V-13 | El frontend no muestra UUIDs | Revisión de interfaces |

### DT-03.5 — La matriz de cumplimiento

| Módulo | Cumple | Evidencia / Deuda |
|---|---|---|
| POS | ⚠️ Parcial | `Ticket.id` es entero; `account_num` es el folio visible |
| Almacenes | ✅ | `MovimientoInventario.id` es `String` UUID |
| Grandeza | ⚠️ Parcial | Pendiente de auditoría |
| Resto | ⚠️ Parcial | Pendiente de auditoría |

---

## SECCIÓN 5 — DT-04: INVENTARIO

### DT-04.1 — La regla

> **El inventario es un ledger inmutable. Solo el módulo dueño escribe. Nadie lee `products.stock`.**

### DT-04.2 — El ancla

| Capa | Archivo | Qué hace |
|---|---|---|
| Almacenamiento | `apps/api/modules/warehouse/models.py` | `MovimientoInventario` (ledger), `WarehouseEvent` (idempotencia) |
| Contrato | `CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md` | P-01: el dueño de la tabla es el único que la escribe |

### DT-04.3 — Las reglas derivadas

1. **El ledger es inmutable.** Un movimiento no se edita ni se borra. Se compensa con otro movimiento.
2. **La idempotencia se garantiza con `evento_id`.** Un evento repetido no duplica el movimiento.
3. **Nadie lee `products.stock`.** El stock se consulta al módulo dueño, por operación.
4. **El stock es derivado, no almacenado.** Si se puede calcular del ledger, no se guarda.

### DT-04.4 — La verificación

| # | Verificación | Cómo |
|---|---|---|
| V-14 | No existe `products.stock` en el código nuevo | Búsqueda estática |
| V-15 | Todo movimiento tiene `evento_id` | Revisión del modelo |
| V-16 | No hay `UPDATE`/`DELETE` sobre el ledger | Búsqueda estática |

### DT-04.5 — La matriz de cumplimiento

| Módulo | Cumple | Evidencia / Deuda |
|---|---|---|
| Almacenes | ✅ | `MovimientoInventario` + `WarehouseEvent` con `evento_id` |
| POS | ⚠️ Parcial | Emite `WarehouseEvent`; no escribe el ledger directamente |
| Producción | ⚠️ Parcial | Pendiente de auditoría |
| Resto | ⚠️ Parcial | Pendiente de auditoría |

---

## SECCIÓN 6 — DT-05: AUDITORÍA

### DT-05.1 — La regla

> **Toda operación que mueve dinero o inventario deja rastro: quién, cuándo, qué.**

### DT-05.2 — El ancla

| Capa | Archivo | Qué hace |
|---|---|---|
| Almacenamiento | `apps/api/modules/audit/` | Registro de operaciones |
| Contrato | `CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md` | A-04: outbox transaccional |

### DT-05.3 — Las reglas derivadas

1. **Se registra `capturó` y `cobró` por separado.** No siempre es la misma persona.
2. **El registro es transaccional con la operación.** Si la operación falla, el registro no queda.
3. **El registro es append-only.** No se edita ni se borra.
4. **El registro incluye el `terminal_id`.** Sin terminal, no se sabe desde dónde se hizo.

### DT-05.4 — La verificación

| # | Verificación | Cómo |
|---|---|---|
| V-17 | Toda operación de dinero registra `capturó`/`cobró` | Revisión de servicios |
| V-18 | El registro es transaccional | Revisión de transacciones |
| V-19 | El registro incluye `terminal_id` | Revisión del modelo |

### DT-05.5 — La matriz de cumplimiento

| Módulo | Cumple | Evidencia / Deuda |
|---|---|---|
| POS | ✅ | Auditoría con `terminal_id`, `capturó`, `cobró` |
| Caja | ✅ | Movimientos con usuario |
| Grandeza | ⚠️ Parcial | Pendiente de auditoría |
| Resto | ⚠️ Parcial | Pendiente de auditoría |

---

## SECCIÓN 6.5 — DT-06: CONFIGURACIÓN DEL NEGOCIO

### DT-06.1 — La regla

> **Los valores que afectan a todos los módulos (zona horaria, moneda, sucursal) se declaran una sola vez, en el módulo Vista General, y se persisten en `system_settings`. Ningún módulo los define por su cuenta. Ningún módulo los sobrescribe.**

### DT-06.2 — El ancla

| Capa | Archivo | Qué hace |
|---|---|---|
| Almacenamiento | `system_settings` (tabla) | Guarda `business_timezone`, `business_currency`, `sucursal_id` |
| Selección | **Módulo Vista General** | La interfaz donde el humano elige |
| Distribución | `apps/shared/TimezoneContext.jsx` | Contexto global de tiempo (existe hoy) |
| Distribución | `apps/shared/MoneyContext.jsx` | Contexto global de dinero (**por crear**) |
| Consumo | Cada módulo | Lee el contexto; nunca define el valor |

### DT-06.3 — Las reglas derivadas

1. **Vista General es un módulo del ERP, no del POS.** Es hermano del POS, no hijo. El POS lo consume, no lo contiene.
2. **Vista General es el único lugar donde se declaran los valores transversales.** No hay un selector de zona horaria en Caja, ni un selector de moneda en Almacenes.
3. **El valor se persiste en `system_settings`.** No se persiste en el frontend, ni en `localStorage`, ni en cada módulo.
4. **El frontend lo distribuye por contexto.** Un contexto por valor transversal (`TimezoneContext`, `MoneyContext`). Los componentes lo consumen con un hook (`useTimezone()`, `useMoney()`).
5. **El valor tiene un solo endpoint de lectura.** `GET /settings` (o `GET /settings/timezone` + `GET /settings/currency`). No hay un endpoint por módulo.
6. **Cambiar el valor no cambia los datos guardados.** Cambiar la zona horaria cambia cómo se **muestra** el tiempo; no reescribe los timestamps. Cambiar la moneda cambia el **símbolo**; no reescribe los montos. (Esto es la cara de configuración de DT-01 y DT-02.)

### DT-06.4 — La verificación

| # | Verificación | Cómo |
|---|---|---|
| V-20 | Existe un solo lugar donde se declaran los valores transversales | Búsqueda estática: un solo componente selector |
| V-21 | Los valores se persisten en `system_settings` | Revisión del modelo de datos |
| V-22 | Existe un contexto por valor transversal | Búsqueda estática: `TimezoneContext`, `MoneyContext` |
| V-23 | Ningún módulo define su propio valor transversal | Búsqueda estática: no hay `business_timezone`/`business_currency` fuera de `system_settings` |
| V-24 | Cambiar el valor no altera los datos guardados | Prueba: cambiar zona horaria no modifica timestamps en BD |

### DT-06.5 — La matriz de cumplimiento

| Módulo | Cumple | Evidencia / Deuda |
|---|---|---|
| Vista General | ⏳ Pendiente | **Por reconstruir.** Su rol ya está decidido (este documento); su especificación funcional se escribirá en su propia FASE 1 |
| POS | ✅ | Consume `TimezoneContext`; no define zona horaria |
| Caja | ✅ | Consume el contexto; no define valores |
| Resto | ⚠️ Parcial | Pendiente de verificar que ninguno define valores transversales |

**Nota importante:** Vista General **no se especifica aquí**. Este documento solo declara **su rol transversal** (dónde se declaran los valores). Su especificación funcional completa (pantallas, campos, validaciones) es FASE 1 de su propio módulo y se escribirá cuando se reconstruya. Declarar el rol ahora evita que la IA constructora invente el selector en cada módulo.

---

## SECCIÓN 7 — LA MATRIZ MAESTRA

Estado de cada módulo frente a cada directriz. **Un módulo no se declara terminado con un ❌.**

| Módulo | DT-01 Tiempo | DT-02 Dinero | DT-03 Identidad | DT-04 Inventario | DT-05 Auditoría | DT-06 Config |
|---|---|---|---|---|---|---|
| **Vista General** | — | — | — | — | — | ⏳ |
| POS | ✅ | ✅ | ⚠️ | ⚠️ | ✅ | ✅ |
| Caja | ✅ | ✅ | ⚠️ | — | ✅ | ✅ |
| Catálogo | — | ✅ | ⚠️ | ⚠️ | ⚠️ | ⚠️ |
| Heladería | — | ✅ | ⚠️ | — | ⚠️ | ⚠️ |
| Almacenes | ⚠️ | ⚠️ | ✅ | ✅ | ⚠️ | ⚠️ |
| Grandeza | ⚠️ | ❌ | ⚠️ | — | ⚠️ | ⚠️ |
| RRHH | ⚠️ | ❌ | ⚠️ | — | ⚠️ | ⚠️ |
| Pedidos | ⚠️ | ❌ | ⚠️ | — | ⚠️ | ⚠️ |
| Producción | ⚠️ | ⚠️ | ⚠️ | ⚠️ | ⚠️ | ⚠️ |
| Analytics | ✅ | ⚠️ | — | — | — | ⚠️ |
| Frontend | ⚠️ | ❌ | ⚠️ | — | — | ⚠️ |

**Leyenda:** ✅ Cumple · ⚠️ Parcial · ❌ No cumple · ⏳ Pendiente · — No aplica

**Nota sobre Vista General:** es el módulo donde se **declaran** los valores transversales. Su fila está en ⏳ porque aún no se reconstruye, pero su rol ya está decidido (DT-06). Los demás módulos lo consumen.

---

## SECCIÓN 8 — CÓMO SE USA ESTE DOCUMENTO AL ARREGLAR EL ERP

El usuario va a arreglar el ERP **módulo por módulo**. Este documento es la guía de ese trabajo.

**El procedimiento, por cada módulo:**

1. **Antes de tocar el módulo**, se recorre la matriz maestra y se anota qué directrices están en ⚠️ o ❌ para ese módulo.
2. **Se arregla el módulo** según su propio plan (FASE 1 a FASE 6 de la metodología).
3. **Al terminar**, se vuelve a recorrer la matriz. Las directrices que estaban en ❌ deben pasar a ✅ o a ⚠️ con una deuda declarada.
4. **Se actualiza la matriz maestra** en este documento.

**Lo que NO se hace:**

- **No se arregla una directriz transversal "de golpe" en todos los módulos.** Se arregla módulo por módulo, cuando el módulo se toca. Arreglar los 10 módulos a la vez es la forma más segura de romper el ERP que corre.
- **No se toca el ERP que corre sin autorización.** Este documento vive en el repo de planos. Cuando una directriz se va a aplicar al ERP, se pide autorización explícita.
- **No se declara una directriz "cumplida" sin la verificación.** La matriz se actualiza con evidencia, no con optimismo.

---

## SECCIÓN 9 — LO QUE ESTE DOCUMENTO NO ES

- **No es un documento de módulo.** No describe cómo funciona Almacenes ni Caja. Eso está en los documentos 1 a 11.
- **No es la metodología.** La metodología dice *cómo* se construye un módulo. Este documento dice *qué reglas debe cumplir* al construirse.
- **No es un plan de acción.** No tiene fases ni fechas. Es una referencia que se consulta.
- **No reemplaza a los criterios de aceptación.** Los criterios (Documento 10) dicen cuándo un módulo está terminado. Este documento dice qué reglas transversales debe cumplir para estarlo. Se complementan: CA-21 (dinero) es la cara de aceptación de DT-02.

---

## SECCIÓN 10 — CIERRE

Este documento existe porque el mismo error se cometió muchas veces en lugares distintos:

- El tiempo, en 5 lugares.
- El dinero, en 80 lugares.
- La identidad, confundida en varios módulos.
- El inventario, escrito desde fuera.

**La regla que lo resume todo:**

> **Lo transversal se decide una vez y se verifica en cada módulo. Nunca se decide en cada módulo.**

**Las 6 directrices vigentes:**

| ID | Directriz | Estado |
|---|---|---|
| DT-01 | Tiempo: UTC almacena, local muestra | Vigente |
| DT-02 | Dinero: `Numeric(12,2)` guarda, un formateador muestra, el selector declara | Vigente |
| DT-03 | Identidad: UUID identifica, folio comunica | Vigente |
| DT-04 | Inventario: ledger inmutable, solo el dueño escribe | Vigente |
| DT-05 | Auditoría: quién, cuándo, qué — transaccional | Vigente |
| DT-06 | Configuración: los valores transversales se declaran una sola vez, en Vista General | Vigente |

**La simetría completa (DT-06):**

```
system_settings  →  Vista General  →  contexto global  →  cada módulo
   (guarda)           (declara)        (distribuye)        (consume)
```

Hoy están documentados el primero, el tercero y el cuarto. **DT-06 declara el segundo**, que era el eslabón que faltaba.

**Documentos relacionados:**

- `METODOLOGIA_DE_INGENIERIA_INVERSA_Y_DISENO.md` — §3.4 (reglas transversales), FASE 2 (casilla de dinero), §9 (errores 9 y 10)
- `CRITERIOS_DE_ACEPTACION_DEL_NUEVO_POS.md` — CA-21 (dinero), CA-13 a CA-16 (tiempo)
- `MODELO_DE_DATOS_DEL_NUEVO_POS.md` — C-01 (UUID), C-02 (UTC), C-03 (Numeric), O-19 (business_currency)
- `CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md` — P-01 a P-03, A-01 a A-06
- `PLANO ARQUITECTONICO PARA EL NUEVO POS.md` — Sección 6 (módulos fuera del alcance del POS)
- `README.md` — reglas de oro 9 (tiempo) y 11 (dinero)
