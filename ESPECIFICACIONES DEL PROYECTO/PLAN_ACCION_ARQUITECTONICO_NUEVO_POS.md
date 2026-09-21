# PLAN DE ACCIÓN ARQUITECTÓNICO — NUEVO POS
## Traducción de la opinión del analista en acciones concretas

> **Documento complementario** al [`PLANO ARQUITECTONICO PARA EL NUEVO POS.md`](PLANO ARQUITECTONICO PARA EL NUEVO POS.md:1).
> El plano describe **qué debe ser** el nuevo POS. Este documento describe **qué acciones concretas**
> incorporar a la arquitectura para que el plano no se quede en intención.
> **Regla dura:** no se toca el ERP. Este documento es un artefacto de diseño.

---

## SECCIÓN 0 — DE OPINIÓN A ACCIÓN

### 0.1 El diagnóstico en una frase

> **No tenemos un POS malo; tenemos un POS bueno atrapado en una frontera mala.**

El POS actual fue **endurecido por la operación**: cada guarda rara (DRAFT GUARD, anti-degradación,
idempotencia) es la cicatriz de un bug real que ya se pagó. El problema no es la lógica —está probada—
sino la **delimitación**: el POS nació *dentro* del ERP, no *al lado*.

### 0.2 Las 5 tesis que se traducen en acciones

| # | Tesis | Acción arquitectónica derivada |
|---|-------|-------------------------------|
| **T1** | El POS fue endurecido por la operación; sus guardas son activos, no ruido. | **A-01**: Portar las 81 reglas **con sus tests**, no solo la regla. |
| **T2** | El problema no es el POS, es la frontera (10 acoplamientos). | **A-02**: Frontera por contratos + **prohibición de leer tablas ajenas**. |
| **T3** | Una regla no automatizada es solo una intención (RN-81 violada). | **A-03**: Cada regla crítica tiene un **test guardián** que falla si se viola. |
| **T4** | `try/except pass` en inventario es la deuda más peligrosa. | **A-04**: **Outbox transaccional** obligatorio; el evento vive en la misma transacción. |
| **T5** | El folio `V####` parece global pero es local. | **A-05**: **Identidad (UUID) ≠ Presentación (folio)**; separación explícita. |

### 0.3 El riesgo central del rewrite: perder las cicatrices

El mayor peligro de limpiar la arquitectura es **limpiar también las cicatrices**. Es decir:
quitar el DRAFT GUARD porque "con contratos limpios ya no hace falta", o relajar la anti-degradación
porque "el nuevo diseño no envía payloads incompletos". Eso reproduce, en 6 meses, los mismos bugs
que el POS actual ya pagó por aprender.

**Contramedida estructural (A-01):** ninguna regla migra sin su test. La unidad de migración
**no es la regla, es la regla + su prueba**.

---

## SECCIÓN 1 — LAS 5 ACCIONES ARQUITECTÓNICAS

### A-01 — Portar las reglas CON sus tests (no solo las reglas)

**Problema que resuelve:** el rewrite pierde las cicatrices.

**Acción concreta:**
1. Cada una de las 81 reglas (RN-01 a RN-81) se migra como una **unidad de dos partes**:
   - la regla (comportamiento), y
   - el **test que la prueba** (portado del POS actual o escrito nuevo si no existía).
2. El criterio de aceptación de una regla migrada es: **"el test pasa en el nuevo POS"**, no
   "la regla está implementada".
3. Las reglas **sin test en el POS actual** se marcan como **riesgo de pérdida** y se les escribe
   un test antes de migrar.

**Evidencia en el POS actual (tests existentes a portar):**

| Suite actual | Reglas que cubre | Acción |
|--------------|------------------|--------|
| [`test_pos_atomic_ops.py`](apps/api/tests/test_pos_atomic_ops.py:1) | RN-17–RN-30 (ítems, versión, DRAFT, estados) | Portar los 14 tests |
| [`test_pos_checkout.py`](apps/api/tests/test_pos_checkout.py:1) | RN-09–RN-16, RN-67–RN-70 (checkout, proyección) | Portar los 7 tests |
| [`test_pos_emergency_save.py`](apps/api/tests/test_pos_emergency_save.py:1) | RN-40 (idempotencia, fallback) | Portar los 6 tests |
| [`test_bloque9d_3bugs.py`](apps/api/tests/test_bloque9d_3bugs.py:1) | RN-78–RN-80 (zona horaria) | Portar los 9 tests |
| [`architecture.test.js`](apps/pos/state/architecture.test.js:1) | Guardas de arquitectura frontend | Portar como guardas del nuevo POS |

**Reglas sin test hoy (riesgo de pérdida):** RN-37–RN-40 (anti-degradación), RN-44–RN-48
(limpieza de borradores), RN-81 (cero offsets). **Acción:** escribir test antes de migrar.

**Criterio de aceptación:** existe una matriz `regla → test` donde **ninguna regla queda sin test**.

---

### A-02 — Frontera por contratos + prohibición de leer tablas ajenas

**Problema que resuelve:** los 10 acoplamientos (AC-01 a AC-10).

**Acción concreta:**
1. El nuevo POS **no importa** modelos de otros módulos. Ni `Product`, ni `Order`, ni
   `WarehouseEvent`, ni `Employee`, ni `CashSession`, ni `Settings`.
2. Cada dependencia externa se resuelve por un **contrato explícito** (interfaz), no por tabla.
3. Se establece una **regla de arquitectura automatizada**: un test que falla si el módulo POS
   importa un modelo de otro módulo.

**Mapeo acoplamiento → contrato:**

| Acoplamiento actual | Contrato que lo reemplaza | Acción |
|---------------------|---------------------------|--------|
| AC-01 POS → `products` | Contrato **Catálogo** | `obtenerArticuloVendible(id)` |
| AC-02 POS → `orders` | Contrato **Pedidos** | `emitirIntencionDePedido(ticket)` |
| AC-03 POS → `warehouse_events` | Contrato **Almacenes** | `emitirConsumo(evento)` (outbox) |
| AC-04 POS → `employees` | Contrato **Seguridad** | `obtenerOperador(id)` |
| AC-05 POS → `cash_sessions` | Contrato **Caja** | `obtenerSesionDeCaja(terminal)` |
| AC-06 POS → `settings` | Contrato **Configuración** | `obtenerConfiguracion(clave)` |
| AC-07 Caja → `tickets` | Contrato **POS** | `obtenerTicketsPagados(rango)` |
| AC-08 POS → `heladeria_product_config` | Contrato **Heladería** | `obtenerConfiguracionDeCanal(canal)` |
| AC-09 POS → `pos_audit.log` | Contrato **Auditoría** | `registrarEvento(evento)` |
| AC-10 POS → `product_technical_sheet` | Contrato **Catálogo** | `obtenerFichaTecnica(id)` |

**Criterio de aceptación:** un test de arquitectura falla si el POS importa un modelo ajeno.

---

### A-03 — Cada regla crítica tiene un test guardián

**Problema que resuelve:** RN-81 fue violada en producción ([`cash/service.py:189`](apps/api/modules/cash/service.py:189))
porque **nada la vigilaba**.

**Acción concreta:**
1. Identificar las reglas **críticas** (las que, si se violan, causan daño de negocio):
   - RN-81 (cero offsets hardcodeados)
   - RN-25–RN-30 (concurrencia optimista)
   - RN-31–RN-36 (DRAFT GUARD)
   - RN-37–RN-40 (anti-degradación)
   - RN-61–RN-66 (inventario como libro mayor)
   - RN-78–RN-80 (zona horaria)
2. Cada regla crítica tiene un **test guardián** que **falla** si se viola.
3. Los guardianes se ejecutan en CI; una violación **bloquea el merge**.

**Ejemplo de guardián (RN-81):**
> Un test que escanea el código del nuevo POS buscando patrones de offset hardcodeado
> (`+6h`, `timedelta(hours=6)`, `-6`, etc.) y **falla** si encuentra alguno fuera de la
> utilidad de zona de negocio.

**Criterio de aceptación:** ninguna regla crítica puede violarse sin que CI falle.

---

### A-04 — Outbox transaccional obligatorio (no `try/except pass`)

**Problema que resuelve:** DEUDA-04 / RC-04 — el evento de inventario se emite con
`try/except pass` ([`service.py:47`](apps/api/modules/pos/service.py:47)); si falla, la venta
se cobra pero el inventario miente.

**Acción concreta:**
1. El evento de consumo de inventario se persiste en la **misma transacción** que el ticket.
   Si el ticket se guarda, el evento existe. **No hay "casi".**
2. Un **procesador asíncrono** (job dedicado) lee el outbox y aplica el consumo en Almacenes.
3. El fallo del procesador es **observable** (log + métrica + reintento), nunca silencioso.
4. Se elimina el patrón `try/except pass` de la ruta crítica de la venta.

**Contraste:**

| Aspecto | POS actual | Nuevo POS |
|---------|-----------|-----------|
| Persistencia del evento | `try/except pass` (puede perderse) | Misma transacción (atómico) |
| Fallo | Silencioso | Observable (log + métrica + reintento) |
| Consistencia | Eventual y frágil | Garantizada por transacción |
| Procesamiento | Inmediato en el request | Job dedicado (desacoplado) |

**Criterio de aceptación:** no existe ningún `try/except pass` en la ruta crítica de la venta.

---

### A-05 — Identidad (UUID) ≠ Presentación (folio)

**Problema que resuelve:** DB-02 — el folio `V####` **parece** global pero es local; colisiona
si se comparte BD entre sucursales.

**Acción concreta:**
1. El **identificador real** de un ticket es un **UUID v4** (global, único, no adivinable).
2. El **folio visible** (`V####`) es una **etiqueta de presentación**, generada por sucursal.
3. La separación es **explícita en el modelo**: `id` (UUID) vs. `folio` (presentación).
4. Ninguna lógica de negocio depende del folio; solo la impresión y la búsqueda humana.

**Contraste:**

| Aspecto | POS actual | Nuevo POS |
|---------|-----------|-----------|
| Identidad | `id` entero (local) | `id` UUID v4 (global) |
| Presentación | `account_num` `V####` | `folio` `V####` (por sucursal) |
| Colisión entre sucursales | Posible | Imposible |
| Confusión identidad/presentación | Alta | Eliminada por diseño |

**Criterio de aceptación:** ninguna regla de negocio usa el folio como identificador.

---

## SECCIÓN 2 — MATRIZ DE TRAZABILIDAD: HALLAZGO → ACCIÓN

| Hallazgo (del doc de ingeniería inversa) | Acción que lo resuelve |
|------------------------------------------|------------------------|
| DEUDA-01 (POS escribe tablas ajenas) | **A-02** (frontera por contratos) |
| DEUDA-02 (POS conoce esquema de Almacenes) | **A-02** + **A-04** (contrato + outbox) |
| DEUDA-03 (POS conoce esquema de Pedidos) | **A-02** (contrato de Pedidos) |
| DEUDA-04 (`try/except pass`) | **A-04** (outbox transaccional) |
| DEUDA-05 (limpieza en el request) | **A-04** (job dedicado) |
| AC-01 a AC-10 (acoplamientos) | **A-02** (contratos) |
| DB-01 (servicio monolítico) | **A-02** (separación por contratos) |
| DB-02 (folio local parece global) | **A-05** (UUID ≠ folio) |
| DB-04 (offset +6h hardcodeado) | **A-03** (test guardián de RN-81) |
| DB-05 (throttle en memoria) | **A-04** (job dedicado) |
| DB-07 (umbral mágico 50%) | **A-03** (test guardián de RN-37) |
| DB-09 (auditoría a archivo) | **A-02** (contrato de Auditoría) |
| DB-10 (umbral visión 0.35) | **A-03** (configurabilidad + guardián) |
| RC-01 (dos terminales escriben) | **A-01** (portar test de concurrencia) |
| RC-02 (dos reservan el mismo DRAFT) | **A-01** (portar test de reserva) |
| RC-03 (candado vence) | **A-01** (portar test de heartbeat) |
| RC-04 (evento huérfano) | **A-04** (outbox transaccional) |

**Lectura:** las 5 acciones cubren **los 29 hallazgos**. No queda ninguno sin acción asignada.

---

## SECCIÓN 3 — ORDEN DE EJECUCIÓN

| Orden | Acción | Depende de | Por qué en este orden |
|-------|--------|-----------|----------------------|
| 1 | **A-02** (frontera por contratos) | — | Define las fronteras; todo lo demás se apoya en ellas |
| 2 | **A-05** (UUID ≠ folio) | A-02 | Es una decisión de identidad; debe fijarse antes de persistir |
| 3 | **A-04** (outbox transaccional) | A-02 | El outbox es un contrato con Almacenes |
| 4 | **A-01** (portar reglas + tests) | A-02, A-04, A-05 | Migra el comportamiento sobre la frontera ya definida |
| 5 | **A-03** (tests guardianes) | A-01 | Los guardianes vigilan las reglas ya migradas |

**Principio rector:** primero se **redibuja la frontera** (A-02), luego se **fija la identidad**
(A-05), luego se **garantiza la consistencia** (A-04), luego se **migra el comportamiento**
(A-01), y finalmente se **blinda** (A-03).

---

## SECCIÓN 4 — CRITERIOS DE ACEPTACIÓN GLOBALES

| # | Criterio | Cómo se verifica |
|---|----------|------------------|
| G1 | Ninguna regla migra sin test | Matriz `regla → test` completa |
| G2 | El POS no importa modelos ajenos | Test de arquitectura (falla si importa) |
| G3 | Ninguna regla crítica puede violarse sin que CI falle | Suite de guardianes en verde/rojo |
| G4 | No hay `try/except pass` en la ruta crítica | Búsqueda automatizada en CI |
| G5 | Ninguna regla de negocio usa el folio como identidad | Revisión de la matriz de trazabilidad |
| G6 | El ERP permanece intacto | `git status` del ERP limpio; HEAD `fe9f6ed` |

---

## SECCIÓN 5 — DECLARACIÓN DE LA REGLA DURA

> **NO SE TOCA EL ERP INSTALADO Y CORRIENDO.**
> **NO SE TOCA NINGUNO DE SUS MÓDULOS.**
> **NO SE MODIFICA NI UNA LÍNEA DEL POS ACTUAL.**

Este documento es un artefacto de diseño derivado de la ingeniería inversa del POS actual
(commit `fe9f6ed`, tag `v22-estable-fe9f6ed`). No contiene código de producción. El ERP
permanece intacto y operando.

---

*Documento complementario al plano fundacional. Versión 1.0. Anclado al commit `fe9f6ed`.*
