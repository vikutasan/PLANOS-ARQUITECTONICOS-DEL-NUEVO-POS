# CRITERIOS DE ACEPTACIÓN DEL NUEVO POS

**Documento 10 — La definición de "obra terminada"**

> **Propósito de este documento.** Los documentos 1-9 dicen **qué** construir y **cómo**.
> Este documento dice **cuándo se considera terminado**. Sin este documento, "terminado"
> es una opinión; con él, es una lista verificable.
>
> **Regla de oro de este documento.** *Ninguna casilla se marca por confianza: se marca
> por evidencia.* Cada criterio tiene un **cómo se verifica** concreto (un comando, una
> prueba, una consulta). Si no se puede verificar, no es un criterio: es un deseo.
>
> **Anclaje.** Este documento está anclado al commit `fe9f6ed` (tag `v22-estable-fe9f6ed`)
> del ERP actual. Los criterios de paridad se comparan contra el POS actual en ese commit.

---

## SECCIÓN 0 — CÓMO LEER ESTE DOCUMENTO

Cada criterio se presenta así:

```
CA-<n>  <título>
  Qué exige:      <la condición>
  Cómo se verifica:  <el comando o la prueba exacta>
  Evidencia:      <qué se guarda como prueba>
  Bloqueante:     <SÍ: sin esto no se despliega | NO: se puede diferir>
```

**Los 4 niveles de aceptación:**

| Nivel | Nombre | Qué significa |
|-------|--------|---------------|
| **N-1** | **Paridad funcional** | El POS nuevo hace **todo** lo que hace el actual. |
| **N-2** | **Corrección estructural** | La deuda de los documentos 8 y 9 está a cero. |
| **N-3** | **Blindaje** | Las cicatrices están portadas con sus pruebas. |
| **N-4** | **Operación** | El POS nuevo aguanta un día real de trabajo. |

**Regla de despliegue.** No se despliega a producción hasta que **N-1, N-2 y N-3 estén
completos**. N-4 se valida en un día de operación real antes de retirar el POS viejo.

---

## SECCIÓN 1 — N-1: PARIDAD FUNCIONAL

### CA-01 — Los 5 flujos del POS funcionan

```
Qué exige:      Los 5 flujos canónicos del POS actual funcionan en el POS nuevo:
                  1. Venta directa (agregar → cobrar → imprimir)
                  2. Cuenta abierta (abrir → agregar → recuperar → cobrar)
                  3. Pedido programado (programar → guardar → recuperar)
                  4. Cierre de caja (abrir turno → movimientos → corte)
                  5. Salida de terminal (guardar → liberar → reentrar)
Cómo se verifica:  Prueba manual guiada, flujo por flujo, en el navegador.
                   Cada flujo se ejecuta 3 veces: normal, con error de red, con doble clic.
Evidencia:      Video o captura de cada flujo + el ticket impreso de cada uno.
Bloqueante:     SÍ
```

### CA-02 — Las 81 reglas de negocio (RN-XX) se cumplen

```
Qué exige:      Las 81 reglas documentadas en la ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md
                se cumplen en el POS nuevo.
Cómo se verifica:  Matriz de trazabilidad: cada RN-XX apunta a la prueba que la cubre.
                   Las reglas sin prueba automática se prueban a mano y se documentan.
Evidencia:      La matriz de trazabilidad con las 81 filas, cada una con su prueba.
Bloqueante:     SÍ
```

### CA-03 — Las 26 interfaces existen y son fieles

```
Qué exige:      Las 26 interfaces del ESPECIFICACION_DE_INTERFACES_POS.md existen en el
                POS nuevo con su estructura, controles y estados.
Cómo se verifica:  Recorrido de las 26 fichas, comparando contra el POS actual lado a lado.
Evidencia:      Captura de cada interfaz en los 3 modos (Mostrador, Compacto, Móvil).
Bloqueante:     SÍ
```

### CA-04 — El contrato de respuesta es idéntico

```
Qué exige:      Las operaciones atómicas por ítem devuelven el mismo contrato discriminado
                que el POS actual (éxito con ticket ligero / error de negocio / error de red).
Cómo se verifica:  Ejecutar la suite de contratos:
                   npx vitest run apps/pos/hooks/useTicketActions.exitContract.test.js
Evidencia:      Salida de la suite en verde.
Bloqueante:     SÍ
```

---

## SECCIÓN 2 — N-2: CORRECCIÓN ESTRUCTURAL

### CA-05 — Cero lecturas a tablas ajenas

```
Qué exige:      El POS nuevo NO lee ni escribe ninguna tabla de otro módulo.
                Los 7 acoplamientos de deuda del Documento 9 están sustituidos por contratos.
Cómo se verifica:  Búsqueda estática en el código del POS nuevo:
                   - No debe existir `products.stock` ni `products.warehouse`.
                   - No debe existir `from modules.catalog.models import Product` para stock.
                   - No debe existir `from modules.warehouse.models import ...`.
                   - No debe existir `from modules.employees.models import ...`.
Evidencia:      Salida de la búsqueda (debe ser vacía) + la matriz de contratos del Documento 9.
Bloqueante:     SÍ
```

### CA-06 — Todas las PK son UUID

```
Qué exige:      Ninguna tabla del POS nuevo usa entero autoincremental como PK.
Cómo se verifica:  Inspección del esquema:
                   - `\d tickets` en psql: la columna `id` debe ser `uuid`.
                   - Repetir para las 17 tablas del Documento 8.
Evidencia:      Salida de `\d` de cada tabla.
Bloqueante:     SÍ
```

### CA-07 — Todos los timestamps son UTC explícito

```
Qué exige:      Ninguna columna de fecha es `timestamp without time zone`.
Cómo se verifica:  Consulta al catálogo de PostgreSQL:
                   SELECT table_name, column_name, data_type
                   FROM information_schema.columns
                   WHERE table_schema='public' AND data_type LIKE 'timestamp%';
                   Todas deben decir `timestamp with time zone`.
Evidencia:      Salida de la consulta.
Bloqueante:     SÍ
```

### CA-08 — El dinero nunca es Float

```
Qué exige:      Ninguna columna monetaria es `double precision` ni `real`.
Cómo se verifica:  Consulta al catálogo:
                   SELECT table_name, column_name, data_type
                   FROM information_schema.columns
                   WHERE table_schema='public'
                     AND data_type IN ('double precision','real')
                     AND column_name ~ 'total|precio|price|monto|importe|fee';
                   Debe devolver 0 filas.
Evidencia:      Salida de la consulta (0 filas).
Bloqueante:     SÍ
```

### CA-09 — El descuento de stock es idempotente

```
Qué exige:      Llamar dos veces a `almacenes.consumir_por_venta` con el mismo `evento_id`
                descuenta UNA sola vez.
Cómo se verifica:  Prueba automática:
                   - Enviar el evento.
                   - Reenviar el mismo evento.
                   - Verificar que `stock_almacen.cantidad_actual` bajó una sola vez.
Evidencia:      La prueba en verde + el asiento único en `movimientos_inventario`.
Bloqueante:     SÍ
```

### CA-10 — El bug D-28 (día local) está corregido

```
Qué exige:      Un ticket de las 23:30 hora local aparece en el día local correcto,
                no en el día UTC siguiente.
Cómo se verifica:  Ejecutar la suite del bug:
                   pytest apps/api/tests/test_bloque9d_3bugs.py -v
Evidencia:      Salida en verde (los 3 bugs cubiertos).
Bloqueante:     SÍ
```

---

## SECCIÓN 3 — N-3: BLINDAJE (LAS CICATRICES)

### CA-11 — El DRAFT GUARD está portado

```
Qué exige:      Un ticket DRAFT de otra terminal NO se puede cobrar desde esta terminal.
Cómo se verifica:  pytest apps/api/tests/test_pos_checkout.py::test_03_draft_guard_otra_terminal_400 -v
Evidencia:      Salida en verde.
Bloqueante:     SÍ
```

### CA-12 — El bloqueo optimista está portado

```
Qué exige:      Enviar una operación con `version` obsoleta devuelve 409, no corrompe datos.
Cómo se verifica:  pytest apps/api/tests/test_pos_atomic_ops.py -v
                   (cubre test_05, test_10, test_14: los 3 casos de version obsoleta)
Evidencia:      Salida en verde.
Bloqueante:     SÍ
```

### CA-13 — El guardado de emergencia está portado

```
Qué exige:      Al cerrar el navegador con carrito, el ticket se guarda (o se marca FAILED),
                y el guardado es idempotente.
Cómo se verifica:  pytest apps/api/tests/test_pos_emergency_save.py -v
Evidencia:      Salida en verde (6 pruebas).
Bloqueante:     SÍ
```

### CA-14 — La auditoría capturó/cobró está portada

```
Qué exige:      Todo ticket PAID tiene `captured_by_id` y `cashed_by_id` no nulos.
Cómo se verifica:  Consulta:
                   SELECT count(*) FROM tickets
                   WHERE status='PAID' AND (captured_by_id IS NULL OR cashed_by_id IS NULL);
                   Debe devolver 0.
Evidencia:      Salida de la consulta (0) + el reporte de auditoría del POS.
Bloqueante:     SÍ
```

### CA-15 — Los guardianes de arquitectura están portados

```
Qué exige:      Las 5 suites de arquitectura del POS actual pasan en el POS nuevo:
                   - architecture.test.js (v7.0.3, v18, v19, v20, v21)
                   - sessionReset.asymmetry.test.js
                   - sessionReset.equivalence.test.js
Cómo se verifica:  npx vitest run apps/pos/state/ apps/pos/hooks/
Evidencia:      Salida en verde de todas las suites.
Bloqueante:     SÍ
```

### CA-16 — El Outbox está portado

```
Qué exige:      Si la red falla al cobrar, el evento de almacén queda pendiente y se
                reintenta sin duplicar el descuento.
Cómo se verifica:  Prueba manual: desconectar la red, cobrar, reconectar, verificar que
                   el stock se descontó una sola vez.
Evidencia:      Captura del estado de `warehouse_events` antes y después.
Bloqueante:     SÍ
```

---

## SECCIÓN 4 — N-4: OPERACIÓN

### CA-17 — Un día real sin incidentes

```
Qué exige:      El POS nuevo opera un día completo de trabajo real sin:
                   - Pérdida de tickets.
                   - Descuadre de caja.
                   - Tickets duplicados.
                   - Bloqueos de terminal no liberados.
Cómo se verifica:  Operación en paralelo con el POS viejo durante un día.
                   Al cierre: comparar totales de ambos POS.
Evidencia:      El corte de caja de ambos POS + la diferencia (debe ser 0).
Bloqueante:     SÍ (para retirar el POS viejo)
```

### CA-18 — El rendimiento no degrada

```
Qué exige:      Agregar un ítem al ticket responde en menos de 300 ms en la red local.
Cómo se verifica:  Medir el tiempo de `POST /pos/tickets/items/add` 100 veces.
                   El percentil 95 debe ser < 300 ms.
Evidencia:      El reporte de tiempos.
Bloqueante:     NO (se puede optimizar después)
```

### CA-19 — La impresión es idéntica

```
Qué exige:      El ticket impreso es idéntico al del POS actual (formato, logo, corte).
Cómo se verifica:  Imprimir el mismo ticket en ambos POS y comparar lado a lado.
Evidencia:      Los dos tickets físicos.
Bloqueante:     SÍ
```

### CA-20 — El despliegue por sucursal funciona

```
Qué exige:      Instalar el POS nuevo en una sucursal nueva desde cero, siguiendo la
                GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md, sin ayuda del autor.
Cómo se verifica:  Un colaborador instala una sucursal de prueba siguiendo solo la guía.
Evidencia:      La sucursal funcionando + las notas del colaborador sobre la guía.
Bloqueante:     SÍ (para escalar a más sucursales)
```

### CA-21 — El dinero se presenta por un solo camino

```
Qué exige:      Todo monto monetario que se muestra al usuario pasa por UN ÚNICO
                formateador (`formatMoney`), ligado a la moneda del negocio
                (`business_currency`). Ningún componente formatea dinero por su cuenta.

                La regla tiene 4 partes:
                  1. UN SOLO FORMATEADOR. Existe una función `formatMoney(valor)` que
                     usa `Intl.NumberFormat` con `style: 'currency'` y la moneda
                     configurada. Es el único camino para mostrar dinero.
                  2. CERO `toFixed(2)` EN COMPONENTES. Ningún componente de UI usa
                     `toFixed(2)` ni `toLocaleString` para dinero. (Hoy hay 80
                     ocurrencias sueltas en 12 componentes del POS actual: esa es la
                     deuda que este criterio elimina.)
                  3. REDONDEO DECLARADO. El redondeo es half-up (el que espera un
                     cajero), no el half-even que `Intl` aplica por defecto. Se fija
                     explícitamente, no se deja al azar del navegador.
                  4. EL SELECTOR NO CONVIERTE. `business_currency` declara la moneda en
                     la que se capturan los precios. NO es un conversor de divisas.
                     Cambiarlo NO altera ningún monto guardado; solo cambia cómo se ve.

Cómo se verifica:  Dos pruebas:
                  a) Búsqueda estática en el código del POS nuevo:
                     - No debe existir `toFixed(2)` en ningún componente de UI.
                     - No debe existir `toLocaleString` aplicado a un monto.
                     - Debe existir exactamente UNA definición de `formatMoney`.
                  b) Prueba automática del formateador:
                     - `formatMoney(1234.5)` con moneda MXN devuelve `$1,234.50`.
                     - `formatMoney(1234.505)` redondea half-up a `$1,234.51`.
                     - `formatMoney(0)` devuelve `$0.00`.
                     - Un valor nulo/indefinido devuelve `$0.00` (nunca `NaN`).
Evidencia:      Salida de la búsqueda (0 `toFixed(2)`, 1 `formatMoney`) + la prueba en verde.
Bloqueante:     SÍ
```

---

## SECCIÓN 5 — MATRIZ DE ACEPTACIÓN

| # | Criterio | Nivel | Bloqueante | Verificación |
|---|----------|-------|-----------|--------------|
| CA-01 | Los 5 flujos funcionan | N-1 | SÍ | Prueba manual guiada |
| CA-02 | Las 81 RN-XX se cumplen | N-1 | SÍ | Matriz de trazabilidad |
| CA-03 | Las 26 interfaces existen | N-1 | SÍ | Recorrido de fichas |
| CA-04 | El contrato de respuesta es idéntico | N-1 | SÍ | `vitest exitContract` |
| CA-05 | Cero lecturas a tablas ajenas | N-2 | SÍ | Búsqueda estática |
| CA-06 | Todas las PK son UUID | N-2 | SÍ | `\d` de 17 tablas |
| CA-07 | Timestamps UTC explícito | N-2 | SÍ | Consulta a `information_schema` |
| CA-08 | Dinero nunca Float | N-2 | SÍ | Consulta a `information_schema` |
| CA-09 | Descuento idempotente | N-2 | SÍ | Prueba de doble envío |
| CA-10 | Bug D-28 corregido | N-2 | SÍ | `pytest test_bloque9d_3bugs` |
| CA-11 | DRAFT GUARD portado | N-3 | SÍ | `pytest test_03` |
| CA-12 | Bloqueo optimista portado | N-3 | SÍ | `pytest test_pos_atomic_ops` |
| CA-13 | Guardado de emergencia portado | N-3 | SÍ | `pytest test_pos_emergency_save` |
| CA-14 | Auditoría capturó/cobró portada | N-3 | SÍ | Consulta SQL |
| CA-15 | Guardianes de arquitectura portados | N-3 | SÍ | `vitest state/ hooks/` |
| CA-16 | Outbox portado | N-3 | SÍ | Prueba de red caída |
| CA-17 | Un día real sin incidentes | N-4 | SÍ | Operación en paralelo |
| CA-18 | Rendimiento < 300 ms p95 | N-4 | NO | Medición de 100 llamadas |
| CA-19 | Impresión idéntica | N-4 | SÍ | Comparación física |
| CA-20 | Despliegue por sucursal | N-4 | SÍ | Instalación por un colaborador |
| CA-21 | El dinero se presenta por un solo camino | N-2 | SÍ | Búsqueda estática + prueba del formateador |

**Resumen:** 21 criterios. **19 bloqueantes**, 2 diferibles (CA-18).

---

## SECCIÓN 6 — LO QUE ESTE DOCUMENTO NO CUBRE

| Tema | Por qué no está aquí | Dónde se cubre |
|------|----------------------|----------------|
| Contrato de Caja (POS ↔ Caja) | **Ya está documentado** (SECCIÓN 7 del Documento 9): el módulo existe y funciona | [`CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md`](./CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md) §7 |
| Contrato de Pedidos (POS ↔ Pedidos) | **Ya está documentado** (SECCIÓN 8 del Documento 9): el levantamiento existe, falta corregir la frontera | [`CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md`](./CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md) §8 |
| Migración de datos históricos | Es un proyecto aparte, no parte de la construcción | Plan de migración |
| Capacitación de cajeros | Es operación, no construcción | Guía de capacitación |
| Consolidación central (hub) | Ya está en el modelo de despliegue | `MODELO_DESPLIEGUE_Y_CONSOLIDACION_CENTRAL.md` |

---

## SECCIÓN 7 — CIERRE

**Lo que este documento deja claro:**

1. "Terminado" tiene **21 criterios verificables**, no una opinión.
2. **19 son bloqueantes**: sin ellos no se despliega.
3. Cada criterio tiene un **cómo se verifica** concreto: un comando, una prueba, una consulta.
4. La paridad funcional (N-1) se exige **antes** de la corrección estructural (N-2):
   primero que haga lo mismo, después que lo haga mejor.
5. Las cicatrices (N-3) son **bloqueantes**: un POS que pierde el DRAFT GUARD no es
   "el POS nuevo", es una regresión.
6. El dinero tiene **un solo camino de presentación** (CA-21): un formateador, cero
   `toFixed(2)` sueltos, redondeo declarado, y un selector que **declara** la moneda
   pero **nunca la convierte**.

**Con este documento, la fase de planificación queda cerrada.** Los 10 documentos maestros
cubren: qué debe ser (1), qué hace hoy (2), cómo se ve (3), cómo se despliega (4), cómo se
construye (5), cómo se adapta (6), cómo se toca (7), cómo se guardan los datos (8), cómo se
comunican los módulos (9) y cuándo está terminado (10).
