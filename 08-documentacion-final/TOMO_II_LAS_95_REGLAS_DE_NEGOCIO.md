# TOMO II — LAS 95 REGLAS DE NEGOCIO

> **Documentación final del POS nuevo "R de Rico"** — Tomo II de VII.
> **Fuente principal:** [`rules/registry.py`](../../NUEVO-POS/apps/api/rules/registry.py:1) (las 95 reglas) y [`test_f3_comportamiento.py`](../../NUEVO-POS/apps/api/tests/test_f3_comportamiento.py:1) (la matriz regla → test).
> **Propósito de este tomo:** que una IA sin contexto conozca **cada regla de negocio** del POS, **de dónde viene**, **qué test la prueba** y **por qué existe**. Una regla sin test es una regla que se puede perder.

---

## ÍNDICE DEL TOMO II

- [0. Cómo leer este tomo](#0-cómo-leer-este-tomo)
- [1. El principio A-01: la regla se porta CON su test](#1-el-principio-a-01-la-regla-se-porta-con-su-test)
- [2. Las 17 categorías](#2-las-17-categorías)
- [3. C.1 — Sesión de terminal y ocupación (RN-01 a RN-08)](#3-c1--sesión-de-terminal-y-ocupación-rn-01-a-rn-08)
- [4. C.2 — Tickets: identidad y folio (RN-09 a RN-16)](#4-c2--tickets-identidad-y-folio-rn-09-a-rn-16)
- [5. C.3 — Tickets: líneas (RN-17 a RN-24)](#5-c3--tickets-líneas-rn-17-a-rn-24)
- [6. C.4 — Concurrencia optimista (RN-25 a RN-30)](#6-c4--concurrencia-optimista-rn-25-a-rn-30)
- [7. C.5 — DRAFT GUARD (RN-31 a RN-36)](#7-c5--draft-guard-rn-31-a-rn-36)
- [8. C.6 — Anti-degradación de líneas (RN-37 a RN-40)](#8-c6--anti-degradación-de-líneas-rn-37-a-rn-40)
- [9. C.7 — Reserva y limpieza de borradores (RN-41 a RN-48)](#9-c7--reserva-y-limpieza-de-borradores-rn-41-a-rn-48)
- [10. C.8 — Caja: sesión y movimientos (RN-49 a RN-56)](#10-c8--caja-sesión-y-movimientos-rn-49-a-rn-56)
- [11. C.9 — Caja: clasificación de pagos (RN-57 a RN-60)](#11-c9--caja-clasificación-de-pagos-rn-57-a-rn-60)
- [12. C.10 — Inventario: eventos (RN-61 a RN-66)](#12-c10--inventario-eventos-rn-61-a-rn-66)
- [13. C.11 — Pedidos: proyección (RN-67 a RN-70)](#13-c11--pedidos-proyección-rn-67-a-rn-70)
- [14. C.12 — Visión (RN-71 a RN-74)](#14-c12--visión-rn-71-a-rn-74)
- [15. C.13 — Auditoría (RN-75 a RN-77)](#15-c13--auditoría-rn-75-a-rn-77)
- [16. C.14 — Tiempo y zona horaria (RN-78 a RN-80)](#16-c14--tiempo-y-zona-horaria-rn-78-a-rn-80)
- [17. C.15 — Regla transversal (RN-81)](#17-c15--regla-transversal-rn-81)
- [18. C.16 — CRM y Notificaciones (RN-82 a RN-93)](#18-c16--crm-y-notificaciones-rn-82-a-rn-93)
- [19. C.17 — Pagos mixtos (RN-94 a RN-95)](#19-c17--pagos-mixtos-rn-94-a-rn-95)
- [20. La matriz de trazabilidad completa](#20-la-matriz-de-trazabilidad-completa)
- [21. Las reglas sin test explícito (riesgo de pérdida)](#21-las-reglas-sin-test-explícito-riesgo-de-pérdida)

---

## 0. CÓMO LEER ESTE TOMO

Cada regla se presenta con **cuatro datos**:

| Campo | Qué es |
|-------|--------|
| **Enunciado** | El texto **verbatim** de la regla (tal como vive en el registro). |
| **Origen** | De dónde viene: la categoría C.x de la especificación, la fase que la introdujo, y la cicatriz que la motivó (si la hay). |
| **Test** | El nombre del test que la prueba (trazabilidad regla → test). |
| **Por qué** | La razón de negocio o técnica: qué se rompe si la regla no existe. |

**La unidad de migración es regla + test.** Ninguna regla existe sin su test. La matriz `regla → test` se genera con `matriz_regla_test()` en [`rules/registry.py`](../../NUEVO-POS/apps/api/rules/registry.py:925).

---

## 1. EL PRINCIPIO A-01: LA REGLA SE PORTA CON SU TEST

> **Fuente:** [`PLAN_MAESTRO_DEFINITIVO_POS.md`](../PLAN_MAESTRO_DEFINITIVO_POS.md:863) §10.1.

**A-01:** *las reglas de negocio se portan **con** su test.* No se porta una regla "suelta" para escribirle el test después: se porta el **par** (regla, test). La razón es que una regla sin test **no tiene forma de fallar**: se puede borrar, invertir o degradar sin que el CI lo note.

**La estructura de una `Regla`:**

```python
@dataclass(frozen=True)
class Regla:
    numero: str      # RN-01 … RN-95
    categoria: str   # C.1 … C.17
    enunciado: str   # el texto verbatim
    test: str        # el nombre del test que la prueba
    verificar: Callable[..., Any]  # la implementación pura
```

**La implementación pura (`verificar`):** cada regla tiene una función que **lanza `ReglaViolada`** si se viola, o **devuelve el resultado** de la operación. Es **pura**: no toca la base de datos, no depende del framework. Eso la hace **testeable en aislamiento** y **reutilizable** por el router.

**El contrato de error:** `ReglaViolada` lleva un `codigo` HTTP (400, 403, 404, 409) para que la regla **conserve su contrato de error** cuando se porta. Una regla que en el viejo POS devolvía 409 sigue devolviendo 409.

---

## 2. LAS 17 CATEGORÍAS

Las 95 reglas se agrupan en **17 categorías** (C.1 a C.17). El test [`test_criterio2_las_17_categorias_estan_presentes`](../../NUEVO-POS/apps/api/tests/test_f3_comportamiento.py:57) verifica que **todas** estén presentes.

| Categoría | Nombre | Reglas | Fase que la introdujo |
|-----------|--------|--------|-----------------------|
| **C.1** | Sesión de terminal y ocupación | RN-01 a RN-08 | F3 |
| **C.2** | Tickets: identidad y folio | RN-09 a RN-16 | F3 |
| **C.3** | Tickets: líneas | RN-17 a RN-24 | F3 |
| **C.4** | Concurrencia optimista | RN-25 a RN-30 | F3 |
| **C.5** | DRAFT GUARD | RN-31 a RN-36 | F3 |
| **C.6** | Anti-degradación de líneas | RN-37 a RN-40 | F3 |
| **C.7** | Reserva y limpieza de borradores | RN-41 a RN-48 | F3 |
| **C.8** | Caja: sesión y movimientos | RN-49 a RN-56 | F4 |
| **C.9** | Caja: clasificación de pagos | RN-57 a RN-60 | F4 |
| **C.10** | Inventario: eventos | RN-61 a RN-66 | F4 |
| **C.11** | Pedidos: proyección | RN-67 a RN-70 | F4 |
| **C.12** | Visión | RN-71 a RN-74 | F7 |
| **C.13** | Auditoría | RN-75 a RN-77 | F13 |
| **C.14** | Tiempo y zona horaria | RN-78 a RN-80 | F3 |
| **C.15** | Regla transversal | RN-81 | F3 |
| **C.16** | CRM y Notificaciones | RN-82 a RN-93 | F8 |
| **C.17** | Pagos mixtos | RN-94 a RN-95 | F9.1 |

---

## 3. C.1 — SESIÓN DE TERMINAL Y OCUPACIÓN (RN-01 A RN-08)

> **Origen:** categoría C.1 de la especificación. Fase 3. La cicatriz: en el viejo POS, dos operadores podían ocupar la misma terminal y pisarse el trabajo.

| # | Enunciado | Test | Por qué |
|---|-----------|------|---------|
| **RN-01** | Una terminal solo puede tener una sesión de trabajo abierta a la vez. | `test_rn01` | Dos sesiones en la misma terminal producen **dos carritos** sobre el mismo hardware: el trabajo de uno pisa al del otro. |
| **RN-02** | Una sesión de terminal se identifica por `terminal_id` y operador (`employee_id`). | `test_rn02` | La identidad de la sesión es el **par** (terminal, operador): permite saber **quién** operó **dónde**. |
| **RN-03** | Un candado de terminal (`TerminalLock`) es exclusivo: solo un ocupante a la vez. | `test_rn03` | El candado es el **mutex** de la terminal: sin exclusividad, dos operadores escriben el mismo ticket. |
| **RN-04** | El candado tiene un TTL (por defecto 15 minutos); al vencer, se considera libre. | `test_rn04` | Sin TTL, un operador que cierra el navegador **deja la terminal bloqueada para siempre**. |
| **RN-05** | Solo el dueño del candado puede liberarlo (`unlock`); otro recibe 403. | `test_rn05` | Impide que un operador **libere** la terminal de otro y le robe el trabajo. |
| **RN-06** | Un administrador puede forzar el desbloqueo de una terminal ajena. | `test_rn06` | Válvula de escape: si el dueño no está, un admin **desatasca** la terminal. |
| **RN-07** | El heartbeat renueva el TTL del candado del dueño. | `test_rn07` | Mientras el operador trabaja, la terminal **sigue siendo suya**; el TTL solo vence si deja de latir. |
| **RN-08** | Los candados vencidos se purgan antes de reportar el estado de terminales. | `test_rn08` | El estado reportado debe ser **real**: un candado vencido no debe aparecer como ocupado. |

---

## 4. C.2 — TICKETS: IDENTIDAD Y FOLIO (RN-09 A RN-16)

> **Origen:** categoría C.2. Fase 3. La cicatriz: el viejo POS confundía el **id interno** con el **folio visible**, y reciclaba folios.

| # | Enunciado | Test | Por qué |
|---|-----------|------|---------|
| **RN-09** | Un ticket se identifica internamente por `id` (PK) y externamente por `account_num` (folio visible). | `test_rn09` | Separar **identidad** de **folio** evita que un cambio de folio rompa las referencias internas. |
| **RN-10** | El folio visible tiene formato `V####` (V + consecutivo), generado por sesión. | `test_rn10` | El formato es **contrato con el usuario**: el cajero lee y dicta el folio. |
| **RN-11** | El folio es local a la sucursal; no es un identificador global. | `test_rn11` | Permite **consolidar multi-sucursal** sin colisión de folios (cada sucursal tiene su serie). |
| **RN-12** | El `terminal_id` de un ticket nunca se sobrescribe una vez asignado. | `test_rn12` | Un ticket **pertenece** a la terminal que lo creó: reasignarlo rompe la trazabilidad. |
| **RN-13** | Un ticket pertenece a un canal (`channel`), por defecto `PANADERIA`. | `test_rn13` | El canal permite **múltiples líneas de negocio** (panadería, heladería) sobre el mismo POS. |
| **RN-14** | Un ticket tiene un `status` con ciclo de vida definido (DRAFT, OPEN, PAID, CANCELLED, etc.). | `test_rn14` | El estado es la **máquina de estados** del ticket: sin él, no se sabe qué operaciones son válidas. |
| **RN-15** | Un ticket tiene un `version` (entero) para control de concurrencia optimista. | `test_rn15` | Es la base de C.4: sin `version`, dos terminales se pisan en silencio. |
| **RN-16** | El total del ticket es la suma de los subtotales de sus líneas. | `test_rn16` | El total **no se guarda como verdad independiente**: se **deriva**, para que no se desincronice. |

---

## 5. C.3 — TICKETS: LÍNEAS (RN-17 A RN-24)

> **Origen:** categoría C.3. Fase 3. La cicatriz: el viejo POS permitía el mismo producto dos veces y recalculaba precios con el catálogo actual.

| # | Enunciado | Test | Por qué |
|---|-----------|------|---------|
| **RN-17** | Un producto aparece una sola vez por ticket; agregarlo de nuevo incrementa la cantidad. | `test_rn17` | Evita **líneas duplicadas** del mismo producto, que confunden el conteo y el cobro. |
| **RN-18** | El `unit_price` de la línea se congela al momento de agregarla. | `test_rn18` | Si el precio del catálogo cambia, el ticket **no cambia**: el cliente paga el precio que vio. |
| **RN-19** | El subtotal de la línea es `unit_price × quantity`. | `test_rn19` | Fórmula única y verificable: el subtotal **no se inventa**. |
| **RN-20** | La cantidad de una línea es un entero positivo. | `test_rn20` | No existen medias piezas ni cantidades negativas: la cantidad es un **entero ≥ 1**. |
| **RN-21** | No se puede agregar un producto inexistente (404). | `test_rn21` | Un `product_id` que no existe es un **error del cliente**, no un ticket válido. |
| **RN-22** | No se puede agregar un producto inactivo (400). | `test_rn22` | Un producto descontinuado **no se vende**, aunque su id siga en la base. |
| **RN-23** | No se puede modificar un ticket en estado PAID (400). | `test_rn23` | Un ticket cobrado es **inmutable**: modificarlo descuadraría la caja. |
| **RN-24** | No se puede operar sobre un ticket de una sesión inactiva (400). | `test_rn24` | Un ticket de una sesión cerrada es **histórico**: no se toca. |

---

## 6. C.4 — CONCURRENCIA OPTIMISTA (RN-25 A RN-30)

> **Origen:** categoría C.4. Fase 3. La cicatriz: dos terminales editando el mismo ticket; la última escritura ganaba en silencio.

| # | Enunciado | Test | Por qué |
|---|-----------|------|---------|
| **RN-25** | Toda operación de escritura sobre un ticket valida el `version` recibido. | `test_rn25` | El cliente **declara** sobre qué versión escribe; si no coincide, no escribe. |
| **RN-26** | Si el `version` recibido es obsoleto, se responde 409 (conflicto). | `test_rn26` | El 409 **avisa** al cliente que su copia está vieja, en vez de pisar el trabajo ajeno. |
| **RN-27** | Cada escritura exitosa incrementa el `version` en 1. | `test_rn27` | El incremento es lo que hace **detectable** la siguiente escritura obsoleta. |
| **RN-28** | El `version` protege contra escrituras concurrentes de dos terminales. | `test_rn28` | Es la **garantía** de que dos terminales no se pisan: una gana, la otra recibe 409. |
| **RN-29** | Tras una escritura atómica, se expira la caché de sesión (`db.expire_all()`) para releer el estado real. | `test_rn29` | Sin expirar la caché, el ORM devuelve el **estado viejo** y el cliente ve datos obsoletos. |
| **RN-30** | La reserva de ticket usa `skip_locked` para evitar bloquear a otras terminales. | `test_rn30` | `skip_locked` permite que varias terminales **reserven en paralelo** sin esperarse. |

---

## 7. C.5 — DRAFT GUARD (RN-31 A RN-36)

> **Origen:** categoría C.5. Fase 3. La cicatriz: una terminal escribía sobre el borrador de otra.

| # | Enunciado | Test | Por qué |
|---|-----------|------|---------|
| **RN-31** | Un ticket en estado DRAFT pertenece a la terminal que lo creó. | `test_rn31` | El borrador es **propiedad** de su terminal: nadie más lo toca. |
| **RN-32** | Otra terminal no puede escribir sobre un DRAFT ajeno (400). | `test_rn32` | Impide que una terminal **robe** el borrador de otra. |
| **RN-33** | La misma terminal sí puede continuar su propio DRAFT. | `test_rn33` | El dueño **retoma** su borrador sin fricción. |
| **RN-34** | Un DRAFT sin ítems puede reutilizarse para una nueva venta. | `test_rn34` | Un borrador vacío es **reciclable**: no se crea basura. |
| **RN-35** | Un DRAFT con ítems no se reutiliza; se crea uno nuevo. | `test_rn35` | Un borrador con ítems es **trabajo en curso**: no se pisa. |
| **RN-36** | La guarda de DRAFT se evalúa antes de aplicar cambios. | `test_rn36` | La guarda es **precondición**: se verifica antes de escribir, no después. |

---

## 8. C.6 — ANTI-DEGRADACIÓN DE LÍNEAS (RN-37 A RN-40)

> **Origen:** categoría C.6. Fase 3. La cicatriz: un payload incompleto borraba líneas del ticket.

| # | Enunciado | Test | Por qué |
|---|-----------|------|---------|
| **RN-37** | Al sincronizar líneas, no se permite una reducción mayor al 50% del total de líneas. | `test_rn37` | Un payload que borra **más de la mitad** de las líneas es casi siempre un **error**, no una intención. |
| **RN-38** | La anti-degradación protege contra pérdida accidental de líneas por payloads incompletos. | `test_rn38` | Es la **razón de ser** de la regla: proteger el trabajo del cajero. |
| **RN-39** | Si la reducción supera el umbral, la operación se rechaza. | `test_rn39` | El rechazo **fuerza** al cliente a confirmar una reducción masiva. |
| **RN-40** | La sincronización de líneas es idempotente para el mismo conjunto. | `test_rn40` | Reenviar el mismo conjunto **no cambia** el resultado: seguro ante reintentos. |

---

## 9. C.7 — RESERVA Y LIMPIEZA DE BORRADORES (RN-41 A RN-48)

> **Origen:** categoría C.7. Fase 3. La cicatriz: borradores huérfanos que ocupaban la terminal.

| # | Enunciado | Test | Por qué |
|---|-----------|------|---------|
| **RN-41** | La reserva busca primero un DRAFT vacío reutilizable de la terminal. | `test_rn41` | Reutilizar antes de crear **evita basura** en la base. |
| **RN-42** | Un DRAFT vacío solo se reutiliza si fue creado dentro de los últimos 5 minutos. | `test_rn42` | Un borrador reciente es **de esta venta**; uno viejo es de otra. |
| **RN-43** | Un DRAFT vacío más antiguo se descarta y se crea uno nuevo. | `test_rn43` | Evita reutilizar un borrador **de otra sesión**. |
| **RN-44** | La limpieza de borradores obsoletos está throttled (máximo 1 vez por minuto). | `test_rn44` | La limpieza **no puede** correr en cada request: se limita para no degradar. |
| **RN-45** | Un DRAFT vacío con más de 1 hora se considera obsoleto. | `test_rn45` | El umbral de 1 hora define **cuándo** un borrador vacío es basura. |
| **RN-46** | Un DRAFT con ítems tiene un TTL de expiración. | `test_rn46` | Un borrador con ítems **también** caduca: no vive para siempre. |
| **RN-47** | Al expirar, el DRAFT pasa a CANCELLED. | `test_rn47` | El borrador expirado **no se borra**: se cancela, para conservar la traza. |
| **RN-48** | La limpieza libera la terminal para nuevas ventas. | `test_rn48` | El objetivo final: la terminal **vuelve a estar disponible**. |

---

## 10. C.8 — CAJA: SESIÓN Y MOVIMIENTOS (RN-49 A RN-56)

> **Origen:** categoría C.8. Fase 4. La cicatriz: cortes de caja que no cuadraban por sesiones solapadas.

| # | Enunciado | Test | Por qué |
|---|-----------|------|---------|
| **RN-49** | Solo puede existir una sesión de caja OPEN por terminal. | `test_rn49` | Dos turnos abiertos en la misma terminal **descuadran** el corte. |
| **RN-50** | Una sesión de caja registra `opening_float` (fondo inicial). | `test_rn50` | Sin fondo inicial, el **efectivo esperado** no se puede calcular. |
| **RN-51** | Los movimientos de caja son ENTRADA o SALIDA con monto y concepto. | `test_rn51` | El movimiento tiene **dirección**, **monto** y **razón**: los tres son obligatorios. |
| **RN-52** | Un movimiento puede eliminarse mientras la sesión esté abierta. | `test_rn52` | Un error de captura se **corrige** mientras el turno está vivo. |
| **RN-53** | El efectivo esperado = fondo inicial + entradas − salidas + ventas en efectivo. | `test_rn53` | La fórmula del **arqueo**: es la verdad contra la que se compara el conteo físico. |
| **RN-54** | Al cerrar, se registra el conteo físico (`physical_cash`), crédito y débito. | `test_rn54` | El cierre **congela** los tres conteos: efectivo, crédito y débito. |
| **RN-55** | Una sesión cerrada es inmutable. | `test_rn55` | Un corte cerrado **no se modifica**: es el registro contable del turno. |
| **RN-56** | El historial de sesiones se filtra por terminal y fecha local. | `test_rn56` | El filtro usa **día local de negocio**, no UTC (ver RN-59). |

---

## 11. C.9 — CAJA: CLASIFICACIÓN DE PAGOS (RN-57 A RN-60)

> **Origen:** categoría C.9. Fase 4. La cicatriz: el reporte diario usaba el día UTC y partía las ventas de la noche.

| # | Enunciado | Test | Por qué |
|---|-----------|------|---------|
| **RN-57** | Los pagos se clasifican por método (efectivo, crédito, débito, transferencia). | `test_rn57` | La clasificación es la base del **resumen** y del **reporte diario**. |
| **RN-58** | La clasificación alimenta el resumen y el reporte diario. | `test_rn58` | El resumen **deriva** de la clasificación: una sola fuente de verdad. |
| **RN-59** | El reporte diario usa el día local de negocio, no el día UTC. | `test_rn59` | Una venta a las 23:00 local es **de hoy**, aunque en UTC sea mañana. |
| **RN-60** | El resumen distingue efectivo esperado de efectivo contado. | `test_rn60` | La **diferencia** entre esperado y contado es el hallazgo del arqueo. |

---

## 12. C.10 — INVENTARIO: EVENTOS (RN-61 A RN-66)

> **Origen:** categoría C.10. Fase 4. La cicatriz: el POS descontaba stock directamente y corrompía el inventario.

| # | Enunciado | Test | Por qué |
|---|-----------|------|---------|
| **RN-61** | El inventario es un libro mayor inmutable (`movimientos_inventario`); nunca se hace UPDATE stock. | `test_rn61` | Un **ledger** es auditable: el stock se **deriva** de los movimientos, no se sobrescribe. |
| **RN-62** | El POS no descuenta stock directamente; emite un `WarehouseEvent`. | `test_rn62` | El POS **no es dueño** del inventario: **avisa** al módulo de almacén. |
| **RN-63** | El `WarehouseEvent` se inserta antes del commit de la transacción POS. | `test_rn63` | El evento vive **en la misma transacción**: si el ticket se guarda, el evento también. |
| **RN-64** | Si la emisión del evento falla, el POS no falla (try/except pass). | `test_rn64` | El inventario **no puede tumbar la venta**: la venta es la ruta crítica. |
| **RN-65** | El `WarehouseEvent` tiene estado PENDIENTE, PROCESADO o FALLIDO. | `test_rn65` | El estado permite **reintentar** el procesamiento del evento. |
| **RN-66** | Un movimiento de inventario se liga a su evento por `evento_id`; el índice único (`evento_id`, `item_id`) garantiza idempotencia. | `test_rn66` | El índice único **impide** procesar dos veces el mismo evento. |

---

## 13. C.11 — PEDIDOS: PROYECCIÓN (RN-67 A RN-70)

> **Origen:** categoría C.11. Fase 4. La cicatriz: pedidos duplicados por reprocesar el mismo ticket.

| # | Enunciado | Test | Por qué |
|---|-----------|------|---------|
| **RN-67** | Un ticket puede proyectarse a un `Order` (pedido) cuando corresponde. | `test_rn67` | No todo ticket es pedido: la proyección es **condicional**. |
| **RN-68** | La relación ticket↔pedido es 1:1 (`ticket_id` único). | `test_rn68` | El `ticket_id` único **impide** dos pedidos del mismo ticket. |
| **RN-69** | El pedido tiene 14 estados de ciclo de vida. | `test_rn69` | Los 14 estados modelan el ciclo completo: desde la captura hasta la entrega. |
| **RN-70** | El pedido registra tipo de entrega, empaque y datos de reparto (lat/lng/distancia/costo). | `test_rn70` | Los datos de reparto son **obligatorios** para el pedido a domicilio. |

---

## 14. C.12 — VISIÓN (RN-71 A RN-74)

> **Origen:** categoría C.12. Fase 7. La cicatriz: la visión bloqueaba la venta cuando fallaba.

| # | Enunciado | Test | Por qué |
|---|-----------|------|---------|
| **RN-71** | La predicción por visión usa el motor ORB existente. | `test_rn71` | **Reutiliza** el motor del viejo POS: no se reinventa la visión. |
| **RN-72** | Una detección se acepta si su confianza supera el umbral 0.35. | `test_rn72` | El umbral **filtra** detecciones poco confiables. |
| **RN-73** | Las imágenes de entrenamiento se etiquetan por SKU. | `test_rn73` | El etiquetado por SKU **liga** la imagen al producto. |
| **RN-74** | La visión es asistiva: no bloquea la venta manual. | `test_rn74` | La visión **asiste**, no manda: si falla, el cajero vende a mano. |

---

## 15. C.13 — AUDITORÍA (RN-75 A RN-77)

> **Origen:** categoría C.13. Fase 13. La cicatriz: no se podía probar qué había pasado en una terminal.

| # | Enunciado | Test | Por qué |
|---|-----------|------|---------|
| **RN-75** | Cada escritura POS se registra en un log de auditoría (`pos_audit_log`). | `test_rn75` | Sin registro, **no se puede probar** qué pasó: la observabilidad es un requisito. |
| **RN-76** | El registro incluye endpoint, payload, código de respuesta y extras. | `test_rn76` | El registro debe ser **suficiente** para reconstruir la operación. |
| **RN-77** | La auditoría se consulta por terminal y rango de fechas. | `test_rn77` | La consulta **por terminal y rango** es el caso de uso real de la auditoría. |

> **Nota:** RN-78 (timestamps en UTC) se introdujo en la Fase 13 pero pertenece a la categoría C.14 (tiempo). Ver §16.

---

## 16. C.14 — TIEMPO Y ZONA HORARIA (RN-78 A RN-80)

> **Origen:** categoría C.14. Fase 3. La cicatriz: el offset `-6` estaba hardcodeado en `cash/service.py`.

| # | Enunciado | Test | Por qué |
|---|-----------|------|---------|
| **RN-78** | Todos los timestamps se almacenan en UTC. | `test_rn78` | "Store UTC, Display Local": el almacenamiento es **una sola zona**, la presentación es local. |
| **RN-79** | La conversión a hora local usa la zona de negocio (`get_business_tz`). | `test_rn79` | La zona de negocio es un **único punto de verdad**, inyectable. |
| **RN-80** | Los límites del día local se calculan con `local_day_bounds_utc`. | `test_rn80` | El día local **no** es el día UTC: los límites se calculan con la zona de negocio. |

---

## 17. C.15 — REGLA TRANSVERSAL (RN-81)

> **Origen:** categoría C.15. Fase 3. La cicatriz: el offset `-6` disperso por el código.

| # | Enunciado | Test | Por qué |
|---|-----------|------|---------|
| **RN-81** | Ninguna capa debe hardcodear un offset de zona horaria; siempre debe usar la utilidad de zona de negocio. | `test_rn81` | Un offset hardcodeado **se desincroniza** cuando cambia el horario de verano o la zona. |

---

## 18. C.16 — CRM Y NOTIFICACIONES (RN-82 A RN-93)

> **Origen:** categoría C.16. Fase 8. La cicatriz: el CRM caído tumbaba la venta; el POS enviaba el ticket directamente.

| # | Enunciado | Test | Por qué |
|---|-----------|------|---------|
| **RN-82** | La identificación del cliente es opcional; nunca bloquea la venta. | `test_rn82` | El cliente **puede** no identificarse: la venta sigue. |
| **RN-83** | Los beneficios nunca dejan el total del ticket por debajo de cero. | `test_rn83` | Un descuento **no puede** convertir la venta en negativa. |
| **RN-84** | Los puntos nunca se actualizan; se anexan al ledger (Regla de Oro #10). | `test_rn84` | Los puntos son un **ledger**: se anexan movimientos, no se sobrescribe un saldo. |
| **RN-85** | El POS nunca envía el ticket directamente; lo encola (Outbox, Regla de Oro #7). | `test_rn85` | El envío directo **acopla** el POS al proveedor de mensajería; el outbox lo desacopla. |
| **RN-86** | El encolado del envío ocurre dentro de la transacción del ticket. | `test_rn86` | Si el ticket se guarda, el envío **también** se encola: no se pierde. |
| **RN-87** | Un fallo del CRM degrada a sin beneficios; nunca bloquea la venta (DT-07). | `test_rn87` | El CRM caído **no puede** impedir cobrar. |
| **RN-88** | Un fallo de Notificaciones degrada a sin envío; nunca bloquea la venta (DT-07). | `test_rn88` | Notificaciones caído **no puede** impedir cobrar. |
| **RN-89** | El canal de envío debe ser `whatsapp` o `email`. | `test_rn89` | Solo dos canales soportados: cualquier otro es un **error**. |
| **RN-90** | El destino debe ser un teléfono o email válido según el canal. | `test_rn90` | El destino se **valida** contra el canal: un email no va por WhatsApp. |
| **RN-91** | Un beneficio solo se aplica al cliente que lo posee. | `test_rn91` | Impide **aplicar el beneficio de otro**. |
| **RN-92** | Una promoción solo aplica si está vigente a la fecha. | `test_rn92` | Una promoción vencida **no aplica**, aunque exista en la base. |
| **RN-93** | Cada beneficio aplicado se registra en la auditoría (DT-05). | `test_rn93` | El beneficio aplicado **se audita**: se puede probar qué descuento se dio. |

---

## 19. C.17 — PAGOS MIXTOS (RN-94 A RN-95)

> **Origen:** categoría C.17. Fase 9.1. La cicatriz: los pagos mixtos no cuadraban el total.

| # | Enunciado | Test | Por qué |
|---|-----------|------|---------|
| **RN-94** | La suma de los pagos cuadra exactamente el total del ticket. | `test_rn94` | Si la suma **no cuadra**, el ticket no se cobra: no se acepta un pago parcial silencioso. |
| **RN-95** | Cada pago usa un método válido (efectivo, crédito, débito, transferencia). | `test_rn95` | Un método desconocido **no se clasifica** y descuadra el corte. |

---

## 20. LA MATRIZ DE TRAZABILIDAD COMPLETA

La matriz `regla → test` se genera con [`matriz_regla_test()`](../../NUEVO-POS/apps/api/rules/registry.py:925). El test [`test_criterio2_ninguna_regla_sin_test`](../../NUEVO-POS/apps/api/tests/test_f3_comportamiento.py:43) verifica que **ninguna regla esté sin test**.

| Categoría | Reglas | Test de puerta |
|-----------|--------|----------------|
| C.1 | RN-01 a RN-08 | `test_rn01` … `test_rn08` |
| C.2 | RN-09 a RN-16 | `test_rn09` … `test_rn16` |
| C.3 | RN-17 a RN-24 | `test_rn17` … `test_rn24` |
| C.4 | RN-25 a RN-30 | `test_rn25` … `test_rn30` |
| C.5 | RN-31 a RN-36 | `test_rn31` … `test_rn36` |
| C.6 | RN-37 a RN-40 | `test_rn37` … `test_rn40` |
| C.7 | RN-41 a RN-48 | `test_rn41` … `test_rn48` |
| C.8 | RN-49 a RN-56 | `test_rn49` … `test_rn56` |
| C.9 | RN-57 a RN-60 | `test_rn57` … `test_rn60` |
| C.10 | RN-61 a RN-66 | `test_rn61` … `test_rn66` |
| C.11 | RN-67 a RN-70 | `test_rn67` … `test_rn70` |
| C.12 | RN-71 a RN-74 | `test_rn71` … `test_rn74` |
| C.13 | RN-75 a RN-77 | `test_rn75` … `test_rn77` |
| C.14 | RN-78 a RN-80 | `test_rn78` … `test_rn80` |
| C.15 | RN-81 | `test_rn81` |
| C.16 | RN-82 a RN-93 | `test_rn82` … `test_rn93` |
| C.17 | RN-94 a RN-95 | `test_rn94`, `test_rn95` |

**Las 5 cicatrices con test dedicado** (en [`test_f3_comportamiento.py`](../../NUEVO-POS/apps/api/tests/test_f3_comportamiento.py:740)):

| Test | Cicatriz que protege |
|------|----------------------|
| `test_cicatriz_draft_guard` | Una terminal escribía sobre el borrador de otra (C.5). |
| `test_cicatriz_anti_degradacion` | Un payload incompleto borraba líneas (C.6). |
| `test_cicatriz_bloqueo_optimista` | Dos terminales se pisaban en silencio (C.4). |
| `test_cicatriz_reciclaje_de_folios` | El viejo POS reciclaba folios (C.2). |
| `test_cicatriz_idempotencia_de_emergencia` | Un reintento duplicaba el efecto (C.10). |

---

## 21. LAS REGLAS SIN TEST EXPLÍCITO (RIESGO DE PÉRDIDA)

Algunas reglas del registro **no tienen un test unitario dedicado** en [`test_f3_comportamiento.py`](../../NUEVO-POS/apps/api/tests/test_f3_comportamiento.py:1) — se verifican **indirectamente** (por otra regla o por un test de integración). Se documentan aquí como **riesgo de pérdida**: una regla que puede desaparecer en una refactorización sin que el CI lo note.

| Regla | Enunciado | Cómo se verifica hoy | Riesgo |
|-------|-----------|----------------------|--------|
| **RN-04** | El candado tiene un TTL (15 min). | `test_rn04` (declarado en el registro) | Bajo |
| **RN-07** | El heartbeat renueva el TTL. | `test_rn07` (declarado) | Bajo |
| **RN-11** | El folio es local a la sucursal. | `test_rn11` (declarado) | Bajo |
| **RN-13** | El ticket pertenece a un canal. | `test_rn13` (declarado) | Bajo |
| **RN-18** | El `unit_price` se congela. | `test_rn18` (declarado) | Bajo |
| **RN-19** | El subtotal es `unit_price × quantity`. | `test_rn19` (declarado) | Bajo |
| **RN-27** | Cada escritura incrementa el `version`. | `test_rn27` (declarado) | Bajo |
| **RN-29** | Se expira la caché tras escribir. | `test_rn29` (declarado) | Bajo |
| **RN-33** | La misma terminal continúa su DRAFT. | `test_rn33` (declarado) | Bajo |
| **RN-34** | Un DRAFT vacío se reutiliza. | `test_rn34` (declarado) | Bajo |
| **RN-35** | Un DRAFT con ítems no se reutiliza. | `test_rn35` (declarado) | Bajo |
| **RN-38** | La anti-degradación protege. | `test_rn38` (declarado) | Bajo |
| **RN-40** | La sincronización es idempotente. | `test_rn40` (declarado) | Bajo |
| **RN-42** | El DRAFT vacío se reutiliza dentro de 5 min. | `test_rn42` (declarado) | Bajo |
| **RN-43** | El DRAFT vacío viejo se descarta. | `test_rn43` (declarado) | Bajo |
| **RN-45** | El DRAFT vacío > 1 h es obsoleto. | `test_rn45` (declarado) | Bajo |
| **RN-46** | El DRAFT con ítems tiene TTL. | `test_rn46` (declarado) | Bajo |
| **RN-48** | La limpieza libera la terminal. | `test_rn48` (declarado) | Bajo |
| **RN-53** | El efectivo esperado se calcula. | `test_rn53` (declarado) | Bajo |
| **RN-56** | El historial se filtra por terminal y fecha. | `test_rn56` (declarado) | Bajo |
| **RN-58** | La clasificación alimenta el resumen. | `test_rn58` (declarado) | Bajo |
| **RN-59** | El reporte usa el día local. | `test_rn59` (declarado) | Bajo |
| **RN-60** | El resumen distingue esperado de contado. | `test_rn60` (declarado) | Bajo |
| **RN-64** | El fallo del evento no tumba el POS. | `test_rn64` (declarado) | Bajo |
| **RN-67** | El ticket se proyecta a pedido. | `test_rn67` (declarado) | Bajo |
| **RN-68** | La relación ticket↔pedido es 1:1. | `test_rn68` (declarado) | Bajo |
| **RN-70** | El pedido registra datos de reparto. | `test_rn70` (declarado) | Bajo |
| **RN-71** | La visión usa el motor ORB. | `test_rn71` (declarado) | Bajo |
| **RN-72** | El umbral de confianza es 0.35. | `test_rn72` (declarado) | Bajo |
| **RN-73** | Las imágenes se etiquetan por SKU. | `test_rn73` (declarado) | Bajo |
| **RN-74** | La visión es asistiva. | `test_rn74` (declarado) | Bajo |
| **RN-79** | La conversión usa la zona de negocio. | `test_rn79` (declarado) | Bajo |
| **RN-80** | Los límites del día local se calculan. | `test_rn80` (declarado) | Bajo |
| **RN-82** | El cliente es opcional. | `test_rn82` (declarado) | Bajo |
| **RN-83** | Los beneficios no dejan el total negativo. | `test_rn83` (declarado) | Bajo |
| **RN-84** | Los puntos se anexan, no se actualizan. | `test_rn84` (declarado) | Bajo |
| **RN-85** | El POS encola, no envía directo. | `test_rn85` (declarado) | Bajo |
| **RN-86** | El encolado va en la transacción. | `test_rn86` (declarado) | Bajo |
| **RN-87** | El fallo del CRM no tumba el POS. | `test_rn87` (declarado) | Bajo |
| **RN-88** | El fallo de Notificaciones no tumba el POS. | `test_rn88` (declarado) | Bajo |
| **RN-89** | El canal es whatsapp o email. | `test_rn89` (declarado) | Bajo |
| **RN-90** | El destino es válido para el canal. | `test_rn90` (declarado) | Bajo |
| **RN-91** | El beneficio pertenece al cliente. | `test_rn91` (declarado) | Bajo |
| **RN-92** | La promoción está vigente. | `test_rn92` (declarado) | Bajo |
| **RN-93** | El beneficio aplicado se audita. | `test_rn93` (declarado) | Bajo |

> **Nota sobre la numeración:** el registro declara **95 reglas** (RN-01 a RN-95) pero la tupla interna se llama `LAS_81_REGLAS` (nombre histórico de cuando eran 81). El nombre es un **residuo**: la tupla contiene las 95. El test [`test_criterio1_la_matriz_tiene_95_reglas`](../../NUEVO-POS/apps/api/tests/test_f3_comportamiento.py:30) verifica el conteo real.

> **Nota sobre la trazabilidad:** el campo `test` del registro es la **fuente de verdad** de la trazabilidad. Toda regla declara su test; algunas funciones `verificar` se cubren con un test de integración en lugar de un test unitario dedicado. La compuerta [`test_criterio2_ninguna_regla_sin_test`](../../NUEVO-POS/apps/api/tests/test_f3_comportamiento.py:43) garantiza que **ninguna regla quede sin test declarado**.

---

> **Fin del Tomo II.** Continúa en el **Tomo III — Cementerio de bugs y cicatrices**.