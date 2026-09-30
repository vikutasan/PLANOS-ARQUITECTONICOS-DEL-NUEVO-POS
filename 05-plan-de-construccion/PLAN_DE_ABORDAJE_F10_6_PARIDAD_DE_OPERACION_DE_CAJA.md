# PLAN DE ABORDAJE — F10.6 "Paridad de operación del Gestor de Caja"

**Versión:** 1.0
**Fecha:** 30 de septiembre de 2026
**Estado:** PROPUESTO — pendiente de aprobación
**Fase padre:** F10 — Auditoría de Paridad (viejo POS vs. nuevo POS)
**Predecesoras:** F10.4 (contexto diario post-corte) y F10.5 (paridad de datos de caja) — ambas CERRADAS.

---

## 1. ORIGEN DE ESTA FASE

F10.4 y F10.5 cerraron dos brechas de **datos** (el contexto diario y el desglose del resumen).
Al cerrarlas, el usuario preguntó algo más profundo:

> *"¿el gestor de caja del nuevo POS tiene la sección de flujo de efectivo que sí tiene el viejo?
> ¿y tiene los teclados que de acuerdo a las necesidades reales de la operación se definieron en el viejo?"*

Se verificó **leyendo ambos archivos completos** (REGLA DURA 2: *verificar, no asumir*):

| Archivo | Líneas |
|---|---|
| Viejo POS — [`apps/pos/components/GestorDeCaja.jsx`](../../../ERP-R-DE-RICO/apps/pos/components/GestorDeCaja.jsx) | 908 |
| Nuevo POS — [`../NUEVO-POS/apps/pos/src/GestorDeCaja.jsx`](../NUEVO-POS/apps/pos/src/GestorDeCaja.jsx) | 583 |

El resultado: **la función de flujo de efectivo SÍ está portada, pero faltan cuatro cosas de la
operación real.** Esta fase las cierra.

---

## 2. HALLAZGOS VERIFICADOS (evidencia, no opinión)

### 2.1 Lo que SÍ está portado (no tocar)

| Elemento | Viejo POS | Nuevo POS |
|---|---|---|
| Apertura de turno con fondo | ✅ `handleConfirmarFondo` | ✅ `alAbrirTurno` |
| Flujo de efectivo (entradas/salidas) | ✅ Sección B | ✅ Sección "Movimientos" |
| Tipo ENTRADA / SALIDA | ✅ botones | ✅ botones |
| Monto + concepto | ✅ | ✅ |
| Lista de movimientos | ✅ | ✅ |
| Arqueo (efectivo/crédito/débito) | ✅ `FilaDiferencia` | ✅ inputs de conteo |
| Desglose del resumen | ✅ Sección C | ✅ F10.5 |
| Contexto diario post-corte | ✅ | ✅ F10.4 |

### 2.2 Las CUATRO brechas (el objeto de esta fase)

#### BRECHA #1 — Teclado táctil (la más visible para el cajero)

El viejo POS tiene una **columna completa de teclado numérico táctil** (líneas 852–902):

- Display "Valor en Pantalla" que refleja el campo enfocado (línea 857).
- Rejilla 3×4: `1-9`, `.`, `0`, `←` (línea 870).
- Botón `C` (limpiar) (línea 884).
- Botón `ENTER / CONTINUAR` que **avanza el foco** entre campos (línea 890).
- Handler `handleKeypadPress` (línea 115) con la máquina de foco
  `fondo → movimiento → cash → credit → debit`.

El nuevo POS usa **`<input type="number">` nativos** (líneas 278, 433, 475) — depende del
teclado del sistema operativo.

**Por qué importa operativamente (no es estética):** en una tablet de mostrador sin teclado
físico, el teclado numérico del SO tapa media pantalla y obliga a toques imprecisos. El viejo
POS lo resolvió con botones de 80px (`h-20`) y un `ENTER/CONTINUAR` que **encadena campos**:
el cajero nunca toca la pantalla para cambiar de rubro. Eso es una **decisión de operación
real**, no un detalle de diseño.

#### BRECHA #2 — Eliminar movimiento

El viejo POS permite borrar un movimiento mal capturado (`handleEliminarMovimiento`, línea 317;
botón `✕` en línea 617, visible solo si `!turnoFinalizado`).

El nuevo POS **no tiene forma de borrar un movimiento**. Y lo grave: **la regla existe**.
[`rn52_movimiento_eliminable_si_abierta`](../NUEVO-POS/apps/api/rules/registry.py:424) está
implementada y probada — pero **ningún endpoint la usa y ninguna UI la expone**.

Verificado: en [`routers/cash.py`](../NUEVO-POS/apps/api/routers/cash.py) solo existe
`POST /cash/movements`. **No hay `DELETE /cash/movements/{id}`.**

Es la MISMA clase de fallo que ya documentamos cinco veces: *la regla existe y pasa su test,
pero el usuario no puede llegar a ella.*

#### BRECHA #3 — Hora del movimiento

El viejo POS muestra la hora de cada movimiento (`formatHora(m.created_at)`, línea 609).

El nuevo POS muestra solo el motivo y el monto. El dato existe en la tabla
(`CashMovement.created_at`) pero **no se expone en `MovimientoResumen`**
([`schemas.py`](../NUEVO-POS/apps/api/schemas.py:355)) ni se muestra.

#### BRECHA #4 — Impresión del corte no cableada

El viejo POS **imprime el corte automáticamente** al cerrar el turno
(`setTimeout(handlePrintCorte, 500)`, línea 354) y tiene botón "📄 Imprimir Reporte" (línea 780).

El nuevo POS **no imprime el corte desde el Gestor de Caja**. El botón "Confirmar cierre"
(línea 561) solo cierra y abre el modal de contexto.

Y lo grave otra vez: **el componente existe**. [`CorteTicketTemplate.jsx`](../NUEVO-POS/apps/pos/src/components/CorteTicketTemplate.jsx)
(F6.1) y [`printService.js`](../NUEVO-POS/apps/pos/src/services/printService.js) (F6.2, con
`imprimirCorte`) están construidos y probados — pero **`GestorDeCaja.jsx` no los importa**.

---

## 3. LA LECCIÓN QUE ESTA FASE CODIFICA (§10.6.5)

Las cinco brechas anteriores (F4.5, F9.1.4a, F10/B-01, F10.4/B-02, F10.5) fueron de
**integración** y de **flujos de datos**. Esta fase revela una sexta variante:

> **"el inventario de componentes no ve la PARIDAD DE OPERACIÓN."**

Un componente puede existir, pasar su test, estar integrado y recibir los datos correctos —
y aun así **no reproducir la operación real** para la que fue diseñado. El teclado táctil no
es un componente que "falta": es una **decisión de operación** que se tomó en el viejo POS
mirando cómo trabaja el cajero, y que se perdió al reescribir la implementación.

Esto refina §6.8: *"la integración se hereda, la implementación se reescribe"* — pero hay
decisiones de operación que **viven dentro de la implementación** y que, si no se auditan
explícitamente, se pierden en la reescritura.

---

## 4. ALCANCE Y NO-ALCANCE

### Dentro del alcance
1. Portar el teclado táctil con su máquina de foco.
2. Exponer y cablear la eliminación de movimientos (endpoint + servicio + UI).
3. Exponer y mostrar la hora de cada movimiento.
4. Cablear la impresión del corte al cierre del turno.

### Fuera del alcance (se declara explícitamente)
- **Validación por PIN del cajero.** El viejo POS valida un PIN (`securityService.validarPin`)
  para identificar al responsable. El nuevo POS usa la **sesión y el lock del terminal**
  (`useTerminalLocking.js`), que es una decisión arquitectónica ya tomada (el "candado
  exclusivo", RN-03). **No se porta el PIN.** Se documenta como diferencia intencional.
- **Rediseño estético.** El teclado se porta funcionalmente, adaptado a los tokens de diseño
  del nuevo POS (`bg-fondo-panel`, `rounded-canon35`, `min-h-tactil`), no pixel a pixel.
- **Impresión automática silenciosa.** Se cablea la impresión; si debe ser automática o por
  botón se decide en F10.6.4 (ver §7.4).

---

## 5. PRINCIPIOS DE EJECUCIÓN

1. **Verificar, no asumir (REGLA DURA 2).** Cada sub-fase lee el archivo antes de editarlo.
2. **Una sub-fase = una compuerta.** No se avanza sin el test verde.
3. **El contrato `{outcome, reason}` se respeta.** El servicio nunca lanza; el componente
   decide con `esOk(resultado)`.
4. **PROHIBICIÓN #3.** Los callbacks leen del valor devuelto por el servicio, nunca de un
   cierre sobre el estado de React.
5. **Sin timers de auto-guardado (PROHIBICIÓN #1).** El teclado es síncrono; no introduce
   temporizadores.
6. **La regla se porta CON su test (A-01).** RN-52 ya tiene test; se añade el test de la
   integración (endpoint + UI).

---

## 6. SUB-FASES

### F10.6.1 — Teclado táctil con máquina de foco

**Objetivo:** que el cajero capture montos con botones grandes, sin depender del teclado del SO.

**Entregables:**
- Nuevo componente [`TecladoTactil.jsx`](../NUEVO-POS/apps/pos/src/components/TecladoTactil.jsx):
  - Props: `valor` (string), `onTecla(valor)`, `etiqueta` (opcional).
  - Renderiza el display "Valor en Pantalla" + rejilla 3×4 + `C` + `ENTER/CONTINUAR`.
  - **Puro y controlado:** no guarda estado propio; emite cada tecla al padre.
- En [`GestorDeCaja.jsx`](../NUEVO-POS/apps/pos/src/GestorDeCaja.jsx):
  - Estado `campoEnfocado` (`'fondo' | 'movimiento' | 'efectivo' | 'credito' | 'debito' | null`).
  - Los inputs pasan a `readOnly` y se enfocan con `onFocus`/`onClick`.
  - `alTecla(valor)` replica la lógica del viejo `handleKeypadPress`:
    - `C` → limpia el campo enfocado.
    - `←` → borra el último dígito.
    - `ENTER` → avanza el foco (`fondo → movimiento → efectivo → credito → debito → null`).
    - dígito/`.` → concatena, con guardas: un solo `.`, máximo 2 decimales.
  - Layout: en MOSTRADOR, el teclado va en una columna lateral; en COMPACTO/MÓVIL, debajo.
- Test [`GestorDeCaja.f10_6.test.jsx`](../NUEVO-POS/apps/pos/src/GestorDeCaja.f10_6.test.jsx):
  - Criterio 1: pulsar `5`, `0`, `0` en el campo fondo deja `fondoInicial === '500'`.
  - Criterio 2: `←` borra el último dígito; `C` limpia todo.
  - Criterio 3: un segundo `.` se ignora; un tercer decimal se ignora.
  - Criterio 4: `ENTER` en `fondo` mueve el foco a `movimiento`.
  - Criterio 5: `ENTER` en `debito` suelta el foco (`null`).

**Compuerta:** 5/5 verde + CI verde.

---

### F10.6.2 — Eliminar movimiento (endpoint + servicio + UI)

**Objetivo:** que el cajero pueda corregir un movimiento mal capturado mientras el turno está
abierto (RN-52).

**Entregables backend:**
- Nuevo contrato **29** `caja.eliminar_movimiento` en
  [`contracts/registry.py`](../NUEVO-POS/apps/api/contracts/registry.py):
  - Operación: `DELETE /cash/movements/{movement_id}`.
  - Proveedor: POS. Consumidor: POS.
  - Entrada: `{cash_session_id}` (para validar pertenencia).
- Endpoint en [`routers/cash.py`](../NUEVO-POS/apps/api/routers/cash.py):
  - `DELETE /cash/movements/{movement_id}`.
  - Valida con `rn52_movimiento_eliminable_si_abierta(sesion.status)` → 400 si cerrada.
  - Valida que el movimiento pertenece al turno → 404 si no.
  - Devuelve `{movement_id}`.
- **Actualizar el test de frontera:** [`test_f2_frontera.py`](../NUEVO-POS/apps/api/tests/test_f2_frontera.py:157)
  afirma **exactamente 28 contratos** → pasa a **29**. Este es un cambio obligatorio y
  consciente (no un efecto colateral).

**Entregables frontend:**
- [`client.js`](../NUEVO-POS/apps/pos/src/api/client.js): `eliminarMovimiento(cashSessionId, movementId)`.
- [`cashService.js`](../NUEVO-POS/apps/pos/src/services/cashService.js): `eliminarMovimiento(datos)`
  envuelto en `aOutcome`.
- [`GestorDeCaja.jsx`](../NUEVO-POS/apps/pos/src/GestorDeCaja.jsx): botón `✕` por movimiento,
  visible solo si el turno está abierto; al confirmar, refresca el resumen.

**Test:** [`test_f10_6_eliminar_movimiento.py`](../NUEVO-POS/apps/api/tests/test_f10_6_eliminar_movimiento.py):
- Criterio 1: borrar un movimiento abierto lo quita y ajusta el esperado.
- Criterio 2: borrar en un turno cerrado → 400 (RN-52).
- Criterio 3: borrar un movimiento de otro turno → 404.
- Criterio 4: el contrato 29 está declarado y su operación es un verbo HTTP.

**Compuerta:** 4/4 backend verde + 1 test frontend verde + CI verde.

---

### F10.6.3 — Hora del movimiento

**Objetivo:** que el cajero vea cuándo se registró cada movimiento (paridad con el viejo POS).

**Entregables:**
- [`schemas.py`](../NUEVO-POS/apps/api/schemas.py:355): añadir `creado_en: datetime | None`
  a `MovimientoResumen`.
- [`routers/cash.py`](../NUEVO-POS/apps/api/routers/cash.py:328): poblar `creado_en` desde
  `m.created_at`.
- [`GestorDeCaja.jsx`](../NUEVO-POS/apps/pos/src/GestorDeCaja.jsx:397): mostrar la hora
  formateada junto al motivo (usando el formateador local del POS, no `toLocaleString` crudo).

**Test:** extender [`test_f10_5_paridad_caja.py`](../NUEVO-POS/apps/api/tests/test_f10_5_paridad_caja.py)
o crear `test_f10_6_hora_movimiento.py`:
- Criterio 1: el resumen expone `creado_en` en cada movimiento.
- Criterio 2: el frontend renderiza la hora (test de componente).

**Compuerta:** tests verdes + CI verde.

---

### F10.6.4 — Cablear la impresión del corte

**Objetivo:** que al cerrar el turno el cajero pueda imprimir el corte, reutilizando F6.1 + F6.2.

**Entregables:**
- [`GestorDeCaja.jsx`](../NUEVO-POS/apps/pos/src/GestorDeCaja.jsx):
  - Importar `CorteTicketTemplate` y `imprimirCorte` de `printService.js`.
  - Al confirmar el cierre, generar el HTML del corte (vía `ticketGenerator.js` de F6.0) y
    llamar `imprimirCorte(html)`.
  - Añadir botón "📄 Imprimir corte" en el estado CIERRE (para reimprimir).
  - **Decisión de diseño:** la impresión se dispara **por acción del cajero** (botón), no
    automáticamente. Razón: el nuevo POS no asume que hay impresora configurada; un fallo
    silencioso de impresión no debe bloquear el cierre. Se documenta esta diferencia con el
    viejo POS (que imprimía automático) como **mejora intencional**.
- Test: extender `GestorDeCaja.f10_6.test.jsx`:
  - Criterio 1: tras cerrar el turno, existe el botón "Imprimir corte".
  - Criterio 2: pulsarlo llama a `imprimirCorte` con un HTML no vacío.
  - Criterio 3: si `imprimirCorte` devuelve `{outcome:'error'}`, se muestra el banner
    persistente (no se traga el error).

**Compuerta:** 3/3 verde + CI verde.

---

### F10.6.5 — Cierre de fase

**Entregables:**
- [`FICHA_F10_6_PARIDAD_DE_OPERACION_DE_CAJA.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F10_6_PARIDAD_DE_OPERACION_DE_CAJA.md):
  evidencia de las 4 brechas cerradas, con hashes de commit.
- Actualizar [`FICHA_F10_PARIDAD.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F10_PARIDAD.md):
  añadir las 4 brechas nuevas al inventario de paridad.
- Plan Maestro: añadir **§10.6.5** (la lección de F10.6) y actualizar §7 (Fase 10).
- Commit + push en ambos repos (NUEVO-POS y PLANOS).

**Compuerta:** CI verde + ambos repos sincronizados con `origin/main`.

---

## 7. ORDEN DE EJECUCIÓN Y DEPENDENCIAS

```
F10.6.1 (teclado)      ── independiente
F10.6.2 (eliminar)     ── independiente (backend + frontend)
F10.6.3 (hora)         ── depende de F10.6.2 (toca el mismo render de movimientos)
F10.6.4 (impresión)    ── depende de F10.6.1 (el layout del teclado afecta el estado CIERRE)
F10.6.5 (cierre)       ── depende de todas
```

**Orden recomendado:** 6.1 → 6.2 → 6.3 → 6.4 → 6.5.

**Riesgo identificado:** F10.6.1 y F10.6.4 tocan el mismo archivo (`GestorDeCaja.jsx`) y el
mismo layout. Se ejecutan en orden para evitar conflictos de diff. Si el archivo crece
demasiado (ya tiene 583 líneas), evaluar extraer el teclado a su propio componente (ya está
previsto en F10.6.1).

---

## 8. CRITERIOS DE ACEPTACIÓN DE LA FASE

1. El cajero puede capturar todos los montos con el teclado táctil, sin el teclado del SO.
2. El cajero puede borrar un movimiento mal capturado mientras el turno está abierto.
3. El cajero ve la hora de cada movimiento.
4. El cajero puede imprimir el corte al cerrar el turno.
5. Los 4 cambios tienen test y el CI está verde.
6. La diferencia del PIN se documenta como decisión intencional (no como brecha).
7. Ambos repos quedan sincronizados con `origin/main`.

---

## 9. LO QUE ESTA FASE **NO** HACE (para no confundir el alcance)

- No rediseña el Gestor de Caja.
- No porta la validación por PIN (decisión arquitectónica ya tomada).
- No cambia el backend de caja más allá de añadir el endpoint de borrado y el campo de hora.
- No toca el módulo de Estadísticas (eso es F11).
- No redacta la documentación final (§11) — eso es el entregable posterior a F10.

---

## 10. REFERENCIAS

- Viejo POS: [`apps/pos/components/GestorDeCaja.jsx`](../../../ERP-R-DE-RICO/apps/pos/components/GestorDeCaja.jsx)
- Nuevo POS: [`../NUEVO-POS/apps/pos/src/GestorDeCaja.jsx`](../NUEVO-POS/apps/pos/src/GestorDeCaja.jsx)
- Backend caja: [`../NUEVO-POS/apps/api/routers/cash.py`](../NUEVO-POS/apps/api/routers/cash.py)
- Reglas: [`../NUEVO-POS/apps/api/rules/registry.py`](../NUEVO-POS/apps/api/rules/registry.py) (RN-52)
- Contratos: [`../NUEVO-POS/apps/api/contracts/registry.py`](../NUEVO-POS/apps/api/contracts/registry.py)
- Impresión: [`../NUEVO-POS/apps/pos/src/services/printService.js`](../NUEVO-POS/apps/pos/src/services/printService.js)
- Plantilla de corte: [`../NUEVO-POS/apps/pos/src/components/CorteTicketTemplate.jsx`](../NUEVO-POS/apps/pos/src/components/CorteTicketTemplate.jsx)
- Plan Maestro §6.8, §10.6.1–§10.6.4
- Fichas: `FICHA_F10_4_*`, `FICHA_F10_5_*`, `FICHA_F10_PARIDAD.md`
