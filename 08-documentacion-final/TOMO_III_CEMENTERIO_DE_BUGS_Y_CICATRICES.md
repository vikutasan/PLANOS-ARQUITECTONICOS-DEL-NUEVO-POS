# TOMO III — CEMENTERIO DE BUGS Y CICATRICES

> **Documentación final del POS nuevo "R de Rico"** — Tomo III de VII.
> **Fuente principal:** [`DOCUMENTACION_MODULO_POS.md`](../../ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_MODULO_POS.md:1) §3 (el cementerio de bugs) y §4 (las 21 Reglas de Oro), [`GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md`](../../ESPECIFICACIONES%20DEL%20PROYECTO/GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md:1) §5, [`PLAN_MAESTRO_DEFINITIVO_POS.md`](../PLAN_MAESTRO_DEFINITIVO_POS.md:124) §4–§5, [`PLAN_DE_ABORDAJE_FASE_13_POR_PARTES.md`](../05-plan-de-construccion/PLAN_DE_ABORDAJE_FASE_13_POR_PARTES.md:225) §8.1 y §12.
> **Propósito de este tomo:** que una IA sin contexto conozca **cada bug que costó dinero real**, **qué cicatriz dejó**, **qué regla lo previene** y **qué pasa si alguien "limpia" esa cicatriz**. Este tomo **no se resume**: por regla dura del plan (§3), el cementerio se documenta **completo**.

---

## ÍNDICE DEL TOMO III

- [0. Cómo leer este tomo](#0-cómo-leer-este-tomo)
- [1. La regla dura: el cementerio NO se comprime](#1-la-regla-dura-el-cementerio-no-se-comprime)
- [2. Cicatriz vs. Deuda — la distinción que gobierna todo](#2-cicatriz-vs-deuda--la-distinción-que-gobierna-todo)
- [3. El cementerio de bugs (cronológico, completo)](#3-el-cementerio-de-bugs-cronológico-completo)
- [4. Las 21 Reglas de Oro supervivientes (v6.0 → v21)](#4-las-21-reglas-de-oro-supervivientes-v60--v21)
- [5. Las 9 Reglas de Oro críticas (las que no se pueden perder)](#5-las-9-reglas-de-oro-críticas-las-que-no-se-pueden-perder)
- [6. Las 5 cicatrices que NO se deben quitar](#6-las-5-cicatrices-que-no-se-deben-quitar)
- [7. Las 6 prohibiciones absolutas (del cementerio)](#7-las-6-prohibiciones-absolutas-del-cementerio)
- [8. Las 11 reglas arquitectónicas derivadas de la batalla](#8-las-11-reglas-arquitectónicas-derivadas-de-la-batalla)
- [9. Los 5 anti-patrones prohibidos](#9-los-5-anti-patrones-prohibidos)
- [10. Los 5 bugs del viejo POS y cómo los evita el nuevo](#10-los-5-bugs-del-viejo-pos-y-cómo-los-evita-el-nuevo)
- [11. Advertencia para futuras IAs](#11-advertencia-para-futuras-ias)
- [12. Matriz de trazabilidad: bug → cicatriz → regla → test](#12-matriz-de-trazabilidad-bug--cicatriz--regla--test)

---

## 0. CÓMO LEER ESTE TOMO

Cada bug del cementerio se presenta con **seis datos**:

| Campo | Qué es |
|-------|--------|
| **Fecha / Ticket** | Cuándo ocurrió y con qué identificador se le conoce. |
| **Síntoma** | Lo que el usuario vio (lo que se reportó). |
| **Causa raíz** | El defecto real (lo que de verdad estaba mal). |
| **Costo** | El daño medible: dinero, cuentas perdidas, tiempo de operación. |
| **Cicatriz** | La guarda que quedó en el código para que no vuelva a pasar. |
| **Regla / Test** | La regla de negocio o Regla de Oro que la codifica, y el test que la prueba. |

**La unidad de conservación es bug → cicatriz → test.** Una cicatriz sin test es una cicatriz que se puede borrar sin que nadie lo note. Por eso cada cicatriz de este tomo tiene su test nombrado.

---

## 1. LA REGLA DURA: EL CEMENTERIO NO SE COMPRIME

> **Fuente:** [`PLAN_DE_LA_DOCUMENTACION_FINAL_DEL_NUEVO_POS.md`](../05-plan-de-construccion/PLAN_DE_LA_DOCUMENTACION_FINAL_DEL_NUEVO_POS.md:1) §3 y §8.

El plan de la documentación final establece una **regla dura**:

> **"No se resume el cementerio de bugs. Por regla dura (§3), no se comprime."**

Y en los criterios de aceptación (§8):

> **"El cementerio de bugs y las cicatrices están completos (nada resumido 'para ahorrar espacio')."**
> **"Las 6 prohibiciones y las 10 reglas de batalla están con su caso real de origen."**

**¿Por qué esta regla?** Porque el cementerio **es el activo más valioso** del proyecto. Cada entrada del cementerio es una lección que **ya se pagó con dinero real**. Un resumen "para ahorrar espacio" destruye exactamente lo que hace valioso al documento: el **caso concreto** que da fuerza a la regla. Una regla sin su caso de origen es un dogma; una regla **con** su caso de origen es una **lección**.

**La consecuencia práctica:** este tomo es **largo a propósito**. Si alguien lo "optimiza" recortando incidentes, está violando la regla dura y degradando el activo.

---

## 2. CICATRIZ VS. DEUDA — LA DISTINCIÓN QUE GOBIERNA TODO

> **Fuente:** [`METODOLOGIA_DE_INGENIERIA_INVERSA_Y_DISENO.md`](../METODOLOGIA_DE_INGENIERIA_INVERSA_Y_DISENO.md:1).

Antes de leer el cementerio hay que entender **la distinción más importante** de todo el proyecto:

| Concepto | Definición | Qué se hace |
|----------|-----------|-------------|
| **Cicatriz** | Guarda nacida de un **bug real que ya se pagó**. Es código que existe **porque algo falló** y no debe volver a fallar. | **Se conserva.** Se porta **con su test**. |
| **Deuda** | Atajo que **oculta un fallo** o **acopla módulos**. Es código que existe porque fue más rápido escribirlo mal. | **Se elimina.** Se sustituye por un patrón correcto. |

**El error más peligroso de una IA sin contexto:** confundir una cicatriz con deuda y "limpiarla". Una cicatriz parece código raro, redundante o defensivo — y lo es, **a propósito**. Borrarla reintroduce el bug que la originó.

**Ejemplo canónico:**
- El **DRAFT GUARD** (una terminal no pisa el ticket de otra) parece una validación redundante. **Es una cicatriz.** Si se quita, dos cajeros pisan la misma venta.
- Un `try/except pass` en la ruta crítica parece "manejo de errores defensivo". **Es deuda.** Si se conserva, los fallos se vuelven invisibles.

**La prueba de fuego:** *¿esta guarda nació de un bug que costó dinero?* Si sí → cicatriz, se conserva. *¿Esta guarda oculta un fallo o acopla módulos?* Si sí → deuda, se elimina.

---

## 3. EL CEMENTERIO DE BUGS (CRONOLÓGICO, COMPLETO)

> **Fuente:** [`DOCUMENTACION_MODULO_POS.md`](../../ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_MODULO_POS.md:1) §3.

Los incidentes se listan en **orden cronológico**. Cada uno se documenta con su síntoma, causa raíz, costo, cicatriz y regla.

---

### 3.1 Ticket #906 — el carrito de $124 se convirtió en $2

**Fecha:** operación temprana del POS viejo.
**Síntoma:** un cajero tenía un carrito de **8 productos ($124)**. Al agregar un noveno producto, el carrito **se redujo a 1 producto ($2)**. La venta se cobró mal.

**Causa raíz:** el **auto-save** leía una **variable de estado dentro de un callback asíncrono** (una *stale closure*). El callback capturó el estado del carrito **en el momento en que se creó** (con 1 producto), no el estado actual (con 8). Cuando el auto-save disparó, **sobrescribió** el carrito de 8 productos con el carrito de 1 producto que había capturado.

**Línea de tiempo forense (T=0s a T=30s):**

| Tiempo | Evento |
|--------|--------|
| T=0s | El cajero agrega el producto 1. El carrito tiene 1 ítem ($2). |
| T=2s | El auto-save crea un callback que captura `carrito = [producto1]`. |
| T=5s | El cajero agrega los productos 2–8. El carrito tiene 8 ítems ($124). |
| T=15s | El auto-save dispara. El callback **lee su captura obsoleta** (`[producto1]`). |
| T=16s | El auto-save **sobrescribe** el carrito del servidor con 1 ítem. |
| T=30s | El cajero cobra. El ticket dice **$2**, no $124. |

**Costo:** venta cobrada por **$2 en lugar de $124** (pérdida de $122 en un solo ticket; el patrón se repitió).

**Cicatriz:** **Regla 1 — Referencias Mutables (`useRef`) vs. Closures.** El estado que un callback asíncrono necesita leer **debe vivir en un `useRef`**, no en una variable capturada por closure. El `ref` siempre apunta al valor **actual**.

**Regla / Test:** Regla 1; prohibición absoluta #3 ("NO leer variables de estado dentro de callbacks asíncronos — usar `useRef`").

---

### 3.2 Ticket #125 — la cuenta de $205 desapareció del Pizarrón

**Fecha:** operación temprana del POS viejo.
**Síntoma:** un cajero tenía una cuenta de **$205**. Pulsó el botón del **Pizarrón**. La pantalla **se limpió** antes de que el servidor confirmara. La cuenta **nunca llegó** al Pizarrón. El cliente se fue sin pagar.

**Causa raíz:** el frontend **limpió el carrito de forma optimista** (asumió que el envío tendría éxito) **antes** de recibir la confirmación HTTP 200 del servidor. El envío falló silenciosamente (red), pero el carrito ya estaba limpio. La cuenta se perdió.

**Costo:** cuenta de **$205 perdida** (el cliente no pagó).

**Cicatriz:** **Regla 12 — Verificación Post-Envío al Pizarrón** + **Regla 13 — Bloqueo de Botón Sin Conexión.** No se limpia el carrito hasta que el servidor **confirme** (HTTP 200) **y** se verifique que el ticket existe en la base de datos.

**Regla / Test:** Regla 12, Regla 13; prohibición absoluta #2 ("NO hacer `clearCart()` sin confirmación HTTP 200 Y verificación post-envío").

---

### 3.3 La Hora Pico de Falsos Positivos — modales 409 en cascada

**Fecha:** operación temprana del POS viejo.
**Síntoma:** durante la hora pico, con el WiFi lento (**>15s de latencia**), el **auto-save** lanzaba **modales de error 409 en cascada**. Los cajeros no podían trabajar: cada intento de guardar abría un modal de conflicto.

**Causa raíz:** el auto-save periódico disparaba escrituras que competían entre sí. Con latencia alta, varias escrituras quedaban en vuelo y chocaban (conflicto de versión 409). El sistema **no distinguía** entre un conflicto real (otro usuario editó) y un conflicto **espurio** (la misma terminal compitiendo consigo misma).

**Costo:** **bloqueo operativo** durante la hora pico (la franja de mayor venta del día).

**Cicatriz:** **eliminación del auto-save periódico** (persistencia atómica por ítem, v6.0) + **Regla 16 — Retries Simétricos** + **Regla 18 — Retries en Checkout con Filtro de Errores de Negocio.** Los conflictos espurios se reintentan; solo los conflictos reales se muestran.

**Regla / Test:** Regla 16, Regla 18; prohibición absoluta #1 ("NO reintroducir auto-save, timers ni `setInterval`").

---

### 3.4 Terminal Fantasma OMEGA — 2 terminales ocupadas 2+ días

**Fecha:** 23 de Abril de 2026.
**Síntoma:** la terminal **OMEGA** aparecía **ocupada** en el selector, pero **nadie la estaba usando**. Estuvo así **más de 2 días**. Los cajeros no podían tomar OMEGA.

**Causa raíz:** **tres bugs interconectados:**
1. **`CashSession` sin TTL:** una sesión de caja se creaba y **nunca expiraba**. Si el cajero cerraba el navegador sin cerrar la sesión, la sesión quedaba "abierta" para siempre.
2. **Doble ocupación:** el sistema permitía que **dos registros** de ocupación coexistieran para la misma terminal (uno huérfano + uno nuevo).
3. **Heartbeat sin purga:** el *heartbeat* de la terminal **no purgaba** las sesiones muertas. La terminal seguía "latido" aunque el navegador estuviera cerrado.

**Costo:** **2 terminales inutilizadas durante 2+ días** (capacidad de venta reducida).

**Cicatriz:** **Regla 8 — Garbage Collector (Limpieza de Zombis)** + **Regla 11 — Reciclaje de Tickets (Máximo 5 Minutos)** + **Regla 17 — Auto-Reconciliación Post-Fallo.** Las sesiones muertas se purgan; los locks huérfanos se liberan.

**Regla / Test:** Regla 8, Regla 11, Regla 17; RN-44 (`rn44_limpieza_throttled`), RN-47 (`rn47_al_expirar_pasa_a_cancelled`).

---

### 3.5 Cuenta Fantasma $453 — T3 (la cuenta que nunca llegó)

**Fecha:** 11 de Junio de 2026.
**Síntoma:** **Yami** (cajera de T3) tenía una cuenta de **$453**. La envió al Pizarrón. La cuenta **nunca apareció**. El cliente se fue. La cuenta quedó "fantasma": existía en la mente de Yami pero no en el sistema.

**Causa raíz:** **corte silencioso de WiFi** en el instante del envío. El frontend limpió el carrito (optimista) y el POST **nunca llegó** al servidor. No hubo error visible: el WiFi se cortó sin que el navegador lo detectara a tiempo.

**Línea de tiempo forense:**

| Tiempo | Evento |
|--------|--------|
| T=0s | Yami pulsa "Enviar al Pizarrón". El carrito tiene $453. |
| T=0s | El frontend limpia el carrito (optimista). |
| T=0s | El POST sale hacia el servidor. |
| T=0s | El WiFi se corta. El POST se pierde. |
| T=1s | El frontend muestra "enviado" (no hubo respuesta de error). |
| T=∞ | La cuenta de $453 **no existe** en la base de datos. |

**Costo:** cuenta de **$453 perdida** (el cliente no pagó).

**Cicatriz (solución v6.1, 3 capas):**
1. **Botón bloqueado sin conexión** (Regla 13): si no hay red, el botón de envío está deshabilitado.
2. **Banner rojo fijo** (Regla 12): un error de red muestra un banner **permanente** (no un toast que desaparece).
3. **Verificación post-envío** (Regla 12): tras el POST, el frontend **consulta al servidor** si el ticket existe realmente.

**Regla / Test:** Regla 12, Regla 13; `test_verificacion_post_envio` en [`test_f3_atomico.py`](../../NUEVO-POS/apps/api/tests/test_f3_atomico.py:267).

---

### 3.6 Ticket Secuestrado por la CAJA — T5

**Fecha:** 16 de Junio de 2026.
**Síntoma:** el ticket **V34538 ($2,550)** fue capturado en la terminal **T5**, pero apareció en el Pizarrón **bajo la terminal CAJA**. El ticket quedó "secuestrado": pertenecía a T5 pero se mostraba como de CAJA.

**Causa raíz:** el campo **`terminal_id` fue sobrescrito**. En algún punto del flujo, el código **reemplazó** el `terminal_id` de origen (T5) por el `terminal_id` de la terminal que procesaba (CAJA). La identidad de origen se perdió.

**Costo:** ticket de **$2,550** con trazabilidad rota (no se sabía de qué terminal salió).

**Cicatriz:** **Regla 14 — Inmutabilidad de la Terminal de Origen.** El `terminal_id` de un ticket **nunca se reescribe** una vez creado. Se eliminó el código que lo sobrescribía.

**Regla / Test:** Regla 14; RN-12 (`rn12_terminal_id_inmutable`) en [`rules/registry.py`](../../NUEVO-POS/apps/api/rules/registry.py:158).

---

### 3.7 Bucle de Destrucción T2/T4 — terminales expulsadas en bucle

**Fecha:** 18 de Junio de 2026.
**Síntoma:** las terminales **T2 y T4** eran **expulsadas** del sistema en un **bucle infinito**. El usuario entraba, era expulsado, volvía a entrar, era expulsado. Aparecían errores **404** repetidos.

**Causa raíz:** faltaba un **`await db.commit()`**. El código hacía `db.flush()` (que envía los cambios a la transacción pero **no los confirma**) y **asumía que ya estaban persistidos**. La siguiente operación no encontraba el registro (404) porque la transacción nunca se confirmó.

**Costo:** **2 terminales inutilizadas** en bucle (bloqueo operativo).

**Cicatriz:** **REGLA: `db.flush()` ≠ persist.** Todo cambio debe terminar en `await db.commit()`. Se agregó el commit faltante.

**Regla / Test:** RN-63 (`rn63_evento_antes_del_commit`); el patrón de commit explícito en [`routers/pos.py`](../../NUEVO-POS/apps/api/routers/pos.py:1).

---

### 3.8 Error 500 Silencioso por Schema Incongruente — el Lazy Load

**Fecha:** 12 de Julio de 2026.
**Síntoma:** al leer un ticket, el servidor devolvía **Error 500** de forma **silenciosa** (sin log claro). La pantalla quedaba en blanco.

**Causa raíz:** el esquema Pydantic de respuesta todavía tenía un campo residual **`technical_sheet`** que **no existía** en el modelo. Al serializar, Pydantic intentaba acceder a una relación **no cargada** (*lazy load*) fuera del contexto async, lo que provocaba **`MissingGreenlet`** (SQLAlchemy no puede hacer I/O implícito en un contexto async).

**Costo:** pantalla en blanco (bloqueo operativo intermitente).

**Cicatriz:** **Regla 15 — Respuesta Ligera en Operaciones Atómicas.** Se creó **`ProductLightResponse`**: una proyección de **campos escalares** que **no toca relaciones**. Se eliminó el campo residual del esquema.

**Regla / Test:** Regla 15; `test_respuesta_ligera_max_5_campos` en [`test_f3_atomico.py`](../../NUEVO-POS/apps/api/tests/test_f3_atomico.py:240).

---

### 3.9 Terminales Fantasma V2 — el `useEffect` con dependencias de objeto

**Fecha:** 31 de Agosto de 2026.
**Síntoma:** volvieron a aparecer **terminales fantasma** (ocupadas sin usuario). El bug de OMEGA (3.4) había "regresado" de otra forma.

**Causa raíz:** el `useEffect` de limpieza tenía **dependencias de objeto** (un objeto que se recreaba en cada render). React lo interpretaba como "las dependencias cambiaron" en **cada render**, así que el efecto de limpieza **se disparaba en bucle**, liberando y re-tomando el lock constantemente. El resultado: locks huérfanos.

**Costo:** terminales fantasma (capacidad de venta reducida).

**Cicatriz:** **Regla 21 — Guardián del `useEffect` de Re-sincronización de Refs** + **Hallazgo H4 (v15).** El `useEffect` de limpieza debe tener dependencias **`[]`** (vacío) y leer el estado actual vía **`refs`**, no vía objetos en las dependencias.

**Regla / Test:** Regla 21; Hallazgo H4 de la corrección v15.

---

### 3.10 Veracidad de Ocupación v12 — la landing "mentía"

**Fecha:** 13 de Septiembre de 2026.
**Síntoma:** la **landing** (selector de terminales) mostraba un estado de ocupación **incorrecto**. Decía que una terminal estaba libre cuando estaba ocupada, o viceversa. El usuario tomaba una terminal "libre" que en realidad estaba en uso.

**Causa raíz:** la landing **no distinguía** entre los estados reales de una terminal. Mostraba un booleano (libre/ocupada) cuando la realidad tiene **tres estados**. Se manifestó en **3 formas:**
1. **CAJA:** mostraba CAJA libre cuando había un lock huérfano.
2. **T2:** mostraba T2 ocupada cuando el usuario ya había salido.
3. **Lock huérfano:** un lock sin dueño vivo aparecía como "ocupado".

**Costo:** usuarios tomando terminales ocupadas (confusión operativa, doble ocupación).

**Cicatriz:** **Regla 20 — Guardián de Simetría** + **`resolveCardState()`** con **3 estados** (libre / mío / ajeno) + **`sendBeacon` al cerrar** (Hallazgo H3). La landing ahora resuelve el estado real de cada tarjeta.

**Regla / Test:** Regla 20; Hallazgo H3; los 3 estados de terminal.

---

### 3.11 `test_bloque9d_3bugs.py` frágiles — la trampa de la paginación

**Fecha:** 14 de Septiembre de 2026.
**Síntoma:** los tests `test_bloque9d_3bugs.py` **fallaban de forma intermitente**. A veces pasaban, a veces no, sin cambios en el código.

**Causa raíz:** los tests usaban **`limit=100`** para buscar un registro. Cuando la base de datos crecía más allá de 100 registros, el registro buscado **quedaba fuera de la primera página** y el test no lo encontraba. Era una **trampa de paginación**: el test dependía del volumen de datos.

**Costo:** falsos negativos en CI (pérdida de confianza en los tests).

**Cicatriz:** los tests ahora usan **`search=ACC_PREFIX`** (búsqueda dirigida) en lugar de `limit=100`. El test busca **exactamente** lo que necesita, sin depender del volumen.

**Regla / Test:** el patrón de búsqueda dirigida en los tests de puerta.

---

### 3.12 Cuentas Perdidas por Salida sin Enviar — v7.0.3 (6 defectos interconectados)

**Fecha:** 20 de Septiembre de 2026.
**Síntoma:** al **salir de una terminal sin enviar** la cuenta, la cuenta se **perdía**. El usuario cerraba sesión con un carrito abierto y la cuenta desaparecía sin aviso.

**Causa raíz:** **6 defectos interconectados:**
1. **Sin contrato de resultado:** las acciones no distinguían entre "éxito", "fallo de red" y "fallo de negocio". El consumidor no sabía qué había pasado.
2. **Consumidor no fail-safe:** ante un resultado ambiguo, el consumidor **asumía éxito** y limpiaba el carrito.
3. **Limpieza asimétrica:** unas rutas de salida limpiaban los refs y otras no, dejando **refs residuales**.
4. **Sin backdrop:** el modal de salida no bloqueaba la interacción de fondo.
5. **Sin beacon de force logout:** al forzar el cierre de sesión, no se liberaba el lock.
6. **Sin tests guardianes:** nada probaba que las rutas de salida fueran simétricas.

**Costo:** **cuentas perdidas** al salir sin enviar (pérdida de ventas).

**Cicatriz (solución v7.0.3):**
1. **Contrato `{outcome, reason}`:** toda acción de persistencia devuelve este contrato y **nunca lanza**.
2. **Consumidor fail-safe:** ante `outcome !== 'ok'`, el consumidor **conserva** el carrito.
3. **Limpieza espejo:** toda rama de salida limpia **los mismos refs**.
4. **Backdrop:** el modal bloquea la interacción de fondo.
5. **Beacon de force logout:** `sendBeacon` libera el lock al cerrar.
6. **Tests guardianes:** pruebas que verifican la simetría de las rutas.

**Regla / Test:** Regla 6 (Protección Contra Olvidos — Exit Modal), Regla 19 (Limpieza Única vía `buildResetPatch()`); [`sessionReset.js`](../../NUEVO-POS/apps/pos/src/state/sessionReset.js:77).

---

### 3.13 Lógica de Bloqueo H1–H8 — v15 (8 hallazgos)

**Fecha:** 20 de Septiembre de 2026.
**Síntoma:** la lógica de **bloqueo de terminal** tenía **8 defectos** descubiertos en auditoría.

**Causa raíz (los 8 hallazgos):**

| Hallazgo | Defecto |
|----------|---------|
| **H1** | Dependencias de `useEffect` con **objetos** en vez de **primitivos**. |
| **H2** | Timestamps **sin UTC** (mezcla de zonas horarias). |
| **H3** | Al cerrar la pestaña **no se liberaba** el lock ni se persistía el carrito. |
| **H4** | `useEffect` de limpieza con dependencias de objeto (bucle). |
| **H5** | Estados de terminal **no distinguidos** (libre/mío/ajeno). |
| **H6** | Locks **huérfanos** no purgados. |
| **H7** | Falta de **verificación post-envío**. |
| **H8** | Rutas de salida **asimétricas**. |

**Costo:** múltiples fallos de ocupación y pérdida de estado.

**Cicatriz (solución v15, 7 fases):** corrección de los 8 hallazgos en 7 fases de trabajo.

**Regla / Test:** Hallazgos H1–H8; Reglas 14, 15, 16, 17, 19, 20, 21.

---

### 3.14 Asimetría de Limpieza v17 — A1 a A5

**Fecha:** 20 de Septiembre de 2026.
**Síntoma:** las **rutas de salida** de la aplicación limpiaban el estado de forma **distinta**. Unas limpiaban más refs que otras.

**Causa raíz (los 5 hallazgos A1–A5):** cada ruta de salida tenía su **propia** lógica de limpieza, escrita a mano. Con el tiempo, divergieron: unas limpiaban 5 refs, otras 3, otras 7. Los **refs residuales** causaban estado sucio entre terminales.

**Costo:** estado sucio entre terminales (bugs intermitentes difíciles de reproducir).

**Cicatriz:** **Regla 19 — Limpieza Única de Sesión vía `buildResetPatch()`.** Se creó **una sola función** (`buildResetPatch()`) que produce el patch de limpieza. **Todas** las rutas aplican **el mismo** patch.

**Regla / Test:** Regla 19; [`sessionReset.js`](../../NUEVO-POS/apps/pos/src/state/sessionReset.js:77).

---

### 3.15 v18 — A6: la ruta que se olvidó del patch

**Fecha:** 20 de Septiembre de 2026.
**Síntoma:** tras aplicar `buildResetPatch()` en v17, **una ruta seguía sin limpiar** correctamente.

**Causa raíz (A6):** **`handleForceLogout`** era **la única ruta** que **no aplicaba** el patch. Se había escapado de la corrección de v17.

**Costo:** estado sucio al forzar el cierre de sesión.

**Cicatriz:** se aplicó `buildResetPatch()` también en `handleForceLogout`. **Regla 19** se cerró en v18.

**Regla / Test:** Regla 19 (cerrada en v18).

---

### 3.16 v19 — R1 corregida, A3 aceptada

**Fecha:** 20 de Septiembre de 2026.
**Síntoma:** auditoría de seguimiento sobre las correcciones de v17/v18.

**Causa raíz (2 hallazgos):**
- **R1:** una **eliminación redundante** de `localStorage` fue **corregida** (se eliminó la redundancia).
- **A3:** una **asimetría de estilo** fue **aceptada** (no era un bug, era una diferencia cosmética intencional).

**Costo:** ninguno (era limpieza de la limpieza).

**Cicatriz:** depuración de v19. La distinción **corregida vs. aceptada** es en sí misma una lección: no toda asimetría es un bug.

**Regla / Test:** Regla 19 (depurada en v19).

---

### 3.17 v20 — el Guardián de Simetría y la lección del falso verde

**Fecha:** 20 de Septiembre de 2026.
**Síntoma:** se necesitaba una prueba **automática** de que las 4 rutas de salida aplicaran el mismo patch. Sin ella, la asimetría podía volver.

**Causa raíz:** no existía un **guardián** que verificara la simetría. Los tests existentes probaban el comportamiento, no la **estructura** de las rutas.

**Costo:** riesgo de regresión silenciosa.

**Cicatriz:** **Regla 20 — Guardián de Simetría de la Aplicación del Patch.** Un test que verifica que **las 4 rutas de salida** aplican `buildResetPatch()`.

**La lección del falso verde:** el guardián usaba una función **`stripJsComments()`** que, mal implementada, **eliminaba código real** junto con los comentarios. El guardián pasaba (verde) aunque el código estuviera mal. **Lección:** un guardián mal escrito da **falsa confianza**. El guardián también necesita su propio test.

**Regla / Test:** Regla 20; el guardián de simetría.

---

### 3.18 v21 — el Guardián del `useEffect` de Re-sincronización de Refs

**Fecha:** 20 de Septiembre de 2026.
**Síntoma:** tras cambiar de terminal, los **refs quedaban obsoletos**. El `useEffect` que debía re-sincronizarlos no siempre lo hacía.

**Causa raíz:** el `useEffect` de re-sincronización de refs podía **omitirse** o **ejecutarse con dependencias incorrectas**, dejando refs apuntando a la terminal anterior.

**Costo:** estado sucio tras cambio de terminal.

**Cicatriz:** **Regla 21 — Guardián del `useEffect` de Re-sincronización de Refs.** Un test que verifica que el `useEffect` de re-sincronización existe y tiene las dependencias correctas.

**Regla / Test:** Regla 21; el guardián del `useEffect` de re-sincronización.

---

## 4. LAS 21 REGLAS DE ORO SUPERVIVIENTES (v6.0 → v21)

> **Fuente:** [`DOCUMENTACION_MODULO_POS.md`](../../ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_MODULO_POS.md:1) §4.

Las **21 Reglas de Oro** son las cicatrices **codificadas**. Cada una nació de un bug del cementerio (§3). Se listan **completas**, con su obligación/prohibición y su porqué.

| # | Regla | Qué exige / prohíbe | ¿Por qué? |
|---|-------|---------------------|-----------|
| **1** | Referencias Mutables (`useRef`) vs. Closures | El estado que un callback asíncrono lee debe vivir en un `useRef`. | Ticket #906: el auto-save leyó una closure obsoleta y sobrescribió $124 con $2. |
| **2** | Mutex para Acciones Finales (`actionMutexRef`) | Las acciones finales (cobrar, enviar) se serializan con un mutex. | Evita que dos acciones finales corran a la vez y se pisen. |
| **3** | Zero-Loss en Acciones Finales | Ninguna acción final puede perder datos: o completa o revierte. | Una acción final a medias pierde la venta. |
| **4** | El Candado Anti-Wipe (v4.6) | Un carrito con ítems no puede ser borrado por un estado vacío entrante. | Evita que un estado vacío del servidor borre un carrito lleno. |
| **5** | Visibilidad en el Pizarrón (DRAFT vs. OPEN) | Un ticket DRAFT no es visible en el Pizarrón; solo OPEN. | Evita mostrar cuentas a medio capturar. |
| **6** | Protección Contra Olvidos (Exit Modal) — v7.0.3 | Al salir con carrito abierto, un modal obliga a decidir. | v7.0.3: cuentas perdidas al salir sin enviar. |
| **7** | El "Draft Guard" (Blindaje Backend) | Una terminal no pisa el ticket DRAFT de otra. | Dos cajeros editando la misma venta. |
| **8** | Garbage Collector (Limpieza de Zombis) | Las sesiones y locks muertos se purgan. | Terminal Fantasma OMEGA: 2 terminales muertas 2+ días. |
| **9** | Sync Obligatorio de `cartRef` en Recuperación (v4.5) | Al recuperar una cuenta, `cartRef` se re-sincroniza. | Evita que el ref quede apuntando al carrito anterior. |
| **10** | Búsqueda Exacta por `account_num` (v4.5) | La recuperación busca por `account_num` exacto. | Evita recuperar la cuenta equivocada. |
| **11** | Reciclaje de Tickets — Máximo 5 Minutos (v4.8) | Un ticket DRAFT vacío se recicla tras 5 minutos. | Evita huecos en la numeración de folios. |
| **12** | Verificación Post-Envío al Pizarrón (v6.1) | Tras enviar, se verifica en BD que el ticket existe. | Cuenta Fantasma $453: el envío se perdió sin error. |
| **13** | Bloqueo de Botón Sin Conexión (v6.1) | Sin red, el botón de envío está deshabilitado. | Evita el envío optimista que se pierde. |
| **14** | Inmutabilidad de la Terminal de Origen | El `terminal_id` de un ticket nunca se reescribe. | Ticket Secuestrado T5: el ticket de T5 apareció bajo CAJA. |
| **15** | Respuesta Ligera en Operaciones Atómicas (v7.0) | Las respuestas atómicas devuelven solo campos escalares. | Error 500 por Lazy Load (`MissingGreenlet`). |
| **16** | Retries Simétricos en Operaciones Atómicas (v7.0.1) | Todas las operaciones atómicas reintentan igual (3 intentos, backoff 1s/2s/3s). | Asimetría v7.0.1: unas operaciones reintentaban y otras no. |
| **17** | Auto-Reconciliación Post-Fallo (v7.0.2) | Tras un fallo, el sistema se reconcilia con el servidor. | Evita estado local divergente tras un fallo. |
| **18** | Retries en Checkout con Filtro de Errores de Negocio (v7.0.2) | El checkout reintenta solo errores de red, no de negocio. | Evita reintentar un 400 (que fallará siempre). |
| **19** | Limpieza Única de Sesión vía `buildResetPatch()` (v17, cerrado v18, depurado v19) | Todas las rutas de salida aplican el mismo patch. | Asimetría v17 (A1–A5) y v18 (A6). |
| **20** | Guardián de Simetría de la Aplicación del Patch (v20) | Un test verifica que las 4 rutas aplican el patch. | v20: la asimetría podía volver sin un guardián. |
| **21** | Guardián del `useEffect` de Re-sincronización de Refs (v21) | Un test verifica el `useEffect` de re-sincronización. | v21: refs obsoletas tras cambio de terminal. |

---

## 5. LAS 9 REGLAS DE ORO CRÍTICAS (LAS QUE NO SE PUEDEN PERDER)

> **Fuente:** [`GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md`](../../ESPECIFICACIONES%20DEL%20PROYECTO/GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md:1) §5.0.

De las 21 Reglas de Oro, **9 son críticas**: su pérdida reintroduce un bug que costó dinero. Se listan con el **incidente que las originó**.

| Regla | Qué prohíbe / exige | Incidente que la originó |
|-------|---------------------|--------------------------|
| **Regla 5** | El POS no lee tablas de otro módulo; solo contratos. | Acoplamiento por BD (AC-01 a AC-10). |
| **Regla 7** | No `try/except pass` en la ruta crítica. | Fallos invisibles (DEUDA-04). |
| **Regla 11** | El folio es local; la identidad es el UUID. | Colisión de folios entre sucursales (DB-02). |
| **Regla 14** | DRAFT GUARD: una terminal no pisa el ticket de otra. | Dos cajeros editando la misma venta. |
| **Regla 15** | Anti-degradación: un payload incompleto (>50% menos) no borra líneas. | Pérdida silenciosa de ventas. |
| **Regla 16** | Bloqueo optimista (`version`): no sobrescribir cambios concurrentes. | Pérdida del último cambio. |
| **Regla 19** | Toda ruta de cierre aplica `buildResetPatch()`. | Estado sucio entre terminales. |
| **Regla 20** | Guardián de simetría: las 4 rutas de salida aplican el mismo patch. | Rutas divergentes (v20). |
| **Regla 21** | Guardián del `useEffect` de re-sincronización de refs. | Refs obsoletas tras cambio de terminal (v21). |

---

## 6. LAS 5 CICATRICES QUE NO SE DEBEN QUITAR

> **Fuente:** [`GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md`](../../ESPECIFICACIONES%20DEL%20PROYECTO/GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md:1) §5.1.

**El error más peligroso de una IA sin contexto: limpiar las cicatrices.** Estas 5 parecen código redundante o defensivo. **No lo son.** Cada una evita un bug que costó dinero.

| Cicatriz | Por qué existe | Qué pasa si la quitas |
|----------|---------------|-----------------------|
| **DRAFT GUARD** | Evita que dos terminales editen el mismo ticket. | Dos cajeros pisan la misma venta. |
| **Anti-degradación (>50%)** | Evita que un payload incompleto borre líneas. | Se pierden ventas silenciosamente. |
| **Bloqueo optimista (`version`)** | Evita sobrescribir cambios concurrentes. | Se pierde el último cambio. |
| **Reciclaje de folios** | Evita huecos en la numeración. | Reportes con folios saltados. |
| **Idempotencia de emergencia** | Evita duplicar tickets al reconectar. | Ventas duplicadas. |

**La prueba de fuego:** *¿esta guarda nació de un bug que costó dinero?* Si sí → **cicatriz, se conserva con su test**.

---

## 7. LAS 6 PROHIBICIONES ABSOLUTAS (DEL CEMENTERIO)

> **Fuente:** [`PLAN_MAESTRO_DEFINITIVO_POS.md`](../PLAN_MAESTRO_DEFINITIVO_POS.md:124) §4.

Estas 6 prohibiciones **no son opiniones**: cada una nació de un bug del cementerio (§3).

| # | Prohibición | Bug que la originó |
|---|-------------|--------------------|
| **1** | NO reintroducir auto-save, timers ni `setInterval` (persistencia atómica por ítem, v6.0). | Ticket #906 ($124→$2) y la Hora Pico de Falsos Positivos. |
| **2** | NO hacer `clearCart()` sin confirmación HTTP 200 **Y** verificación post-envío (v6.1). | Ticket #125 ($205) y Cuenta Fantasma $453. |
| **3** | NO leer variables de estado dentro de callbacks asíncronos — usar `useRef`. | Ticket #906 ($124→$2). |
| **4** | NO almacenar candados de terminal en RAM de Python — solo en PostgreSQL (`terminal_locks`). | Terminal Fantasma OMEGA. |
| **5** | NO generar folios en el frontend — solo el backend vía secuencia atómica de PostgreSQL. | Colisión de folios (DB-02). |
| **6** | NO duplicar módulos que el ERP ya tiene (login, gestión de empleados, perfiles). | Acoplamiento y duplicación (AC-01 a AC-10). |

---

## 8. LAS 11 REGLAS ARQUITECTÓNICAS DERIVADAS DE LA BATALLA

> **Fuente:** [`PLAN_MAESTRO_DEFINITIVO_POS.md`](../PLAN_MAESTRO_DEFINITIVO_POS.md:138) §5.

Estas 11 reglas se derivaron de la **operación real** (7 meses, 6+ terminales, 30,000+ tickets).

| Regla | Origen | Implementación |
|-------|--------|----------------|
| **Verificar, no asumir** | Autocrítica F7.5 v3.0→v3.1 | Verificar antes de nombrar tabla/campo/contrato/endpoint. |
| **Contrato de resultado discriminado** | Incidente v7.0.3 (cuentas perdidas) | `{ outcome, reason }`. |
| **Verificación post-envío** | Incidente $453 (cuenta fantasma) | Verificar que el ticket existe en la BD. |
| **`withRetries` centralizado** | Asimetría v7.0.1 | 3 intentos, backoff 1s/2s/3s. |
| **Respuesta ligera** | Optimización v7.0 (rush hour) | 5 campos escalares. |
| **Timestamps UTC** | Hallazgo H2 | `utcnow()` siempre. |
| **Primitivos en deps** | Hallazgo H1 | `useEffect` deps = primitivos. |
| **3 estados de terminal** | Incidente v12 (veracidad) | libre / mío / ajeno. |
| **`sendBeacon` al cerrar** | Hallazgo H3 | `beforeunload` libera lock + persiste carrito. |
| **Espejo de limpieza** | Incidente v7.0.3 (refs residuales) | Toda rama de salida limpia los mismos refs. |
| **Banner rojo fijo** | Incidente $453 | Error de red = banner permanente. |

---

## 9. LOS 5 ANTI-PATRONES PROHIBIDOS

> **Fuente:** [`GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md`](../../ESPECIFICACIONES%20DEL%20PROYECTO/GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md:1) §5.2.

Estos 5 anti-patrones son **deuda**, no cicatriz. Se **eliminan**, no se conservan.

| # | Anti-patrón | Por qué está prohibido |
|---|-------------|------------------------|
| **1** | `try/except pass` en la ruta crítica (DEUDA-04). | Oculta fallos: la venta se pierde sin que nadie lo sepa. |
| **2** | Leer tablas de otro módulo (AC-01 a AC-10). | Acopla módulos: el cambio de uno rompe al otro. |
| **3** | Usar el folio como clave (DB-02). | El folio es local; colisiona entre sucursales. |
| **4** | Hardcodear el offset de zona horaria (`+6h`) (RN-81, DB-04). | Rompe con el horario de verano y con otras zonas. |
| **5** | `UPDATE stock` en vez de insertar en el ledger (RN-61 a RN-66). | Pierde la trazabilidad del inventario. |

---

## 10. LOS 5 BUGS DEL VIEJO POS Y CÓMO LOS EVITA EL NUEVO

> **Fuente:** [`PLAN_DE_ABORDAJE_FASE_13_POR_PARTES.md`](../05-plan-de-construccion/PLAN_DE_ABORDAJE_FASE_13_POR_PARTES.md:225) §8.1.

Esta tabla es la **síntesis ejecutiva** del cementerio: los 5 bugs más representativos del viejo POS y la regla del nuevo POS que los previene.

| Bug del viejo POS | Cómo lo evita el nuevo POS |
|-------------------|----------------------------|
| **BUG 1 — secuestro de `terminal_id`** | **RN-12:** `terminal_id` es inmutable. |
| **BUG 2 — zona horaria (tickets nocturnos)** | **RN-78/RN-79:** UTC + `local_day_bounds_utc`. |
| **BUG 3 — `Decimal + float` (error 500)** | **E-09:** dinero en `Numeric`, nunca `Float`. |
| **BUG 4 — lazy loading (pantalla en blanco)** | **Regla 15:** proyección de campos escalares. |
| **BUG 5 — URL con `window.location.hostname`** | **`CONFIG.API_BASE_URL`** centralizado. |

---

## 11. ADVERTENCIA PARA FUTURAS IAs

> **Fuente:** [`PLAN_DE_ABORDAJE_FASE_13_POR_PARTES.md`](../05-plan-de-construccion/PLAN_DE_ABORDAJE_FASE_13_POR_PARTES.md:1) §12.

Si eres una IA sin contexto leyendo este tomo, **lee esta sección dos veces**:

1. **NO reconstruyas el libro mayor ni el outbox.** Ya existen y funcionan. Reconstruirlos reintroduce bugs de doble contabilidad.
2. **NO leas `tickets` desde auditoría.** La auditoría consume el **contrato**, no la tabla. Leer la tabla acopla los módulos (anti-patrón #2).
3. **NO uses `+6h` hardcodeado.** Usa `utcnow()` y `local_day_bounds_utc()`. El offset hardcodeado rompe con el horario de verano (anti-patrón #4).
4. **NO expongas `SELECT *`.** Expón proyecciones de campos escalares (Regla 15). El `SELECT *` trae relaciones que provocan Lazy Load (`MissingGreenlet`).
5. **NO limpies las cicatrices.** Las 5 cicatrices de §6 parecen redundantes. **No lo son.** Cada una evita un bug que costó dinero.
6. **NO confundas cicatriz con deuda.** Una cicatriz nació de un bug pagado (se conserva). Una deuda oculta un fallo o acopla módulos (se elimina). Ver §2.

---

## 12. MATRIZ DE TRAZABILIDAD: BUG → CICATRIZ → REGLA → TEST

Esta matriz cierra el tomo: cada bug del cementerio, su cicatriz, la regla que la codifica y el test que la prueba. **Una cicatriz sin test es una cicatriz que se puede borrar sin que nadie lo note.**

| Bug (§3) | Cicatriz | Regla | Test |
|----------|----------|-------|------|
| Ticket #906 ($124→$2) | `useRef` en callbacks | Regla 1 | `test_cicatriz_*` (front) |
| Ticket #125 ($205) | Verificación post-envío | Regla 12 | `test_verificacion_post_envio` |
| Hora Pico de Falsos Positivos | Sin auto-save + retries | Reglas 16, 18 | `test_f3_atomico.py` |
| Terminal Fantasma OMEGA | Garbage Collector | Reglas 8, 11, 17 | RN-44, RN-47 |
| Cuenta Fantasma $453 | 3 capas (botón, banner, verificación) | Reglas 12, 13 | `test_verificacion_post_envio` |
| Ticket Secuestrado T5 | `terminal_id` inmutable | Regla 14 | `test_rn12` |
| Bucle de Destrucción T2/T4 | `db.commit()` explícito | RN-63 | `test_rn63` |
| Error 500 por Lazy Load | `ProductLightResponse` | Regla 15 | `test_respuesta_ligera_max_5_campos` |
| Terminales Fantasma V2 | `useEffect` deps `[]` + refs | Regla 21 | guardián del `useEffect` |
| Veracidad de Ocupación v12 | `resolveCardState()` + 3 estados | Regla 20 | guardián de simetría |
| `test_bloque9d` frágiles | `search=ACC_PREFIX` | — | los propios tests |
| Cuentas Perdidas v7.0.3 | `{outcome, reason}` + espejo | Reglas 6, 19 | `test_f3_comportamiento.py` |
| Lógica de Bloqueo H1–H8 | 7 fases de corrección | Reglas 14–21 | varios |
| Asimetría v17 (A1–A5) | `buildResetPatch()` | Regla 19 | guardián de simetría |
| v18 (A6) | `handleForceLogout` + patch | Regla 19 | guardián de simetría |
| v19 (R1/A3) | depuración | Regla 19 | — |
| v20 (falso verde) | guardián de simetría | Regla 20 | guardián de simetría |
| v21 | guardián del `useEffect` | Regla 21 | guardián del `useEffect` |

**Las 5 cicatrices con test dedicado** (de [`test_f3_comportamiento.py`](../../NUEVO-POS/apps/api/tests/test_f3_comportamiento.py:740)):

| Test | Cicatriz que protege |
|------|----------------------|
| `test_cicatriz_draft_guard` | DRAFT GUARD |
| `test_cicatriz_anti_degradacion` | Anti-degradación (>50%) |
| `test_cicatriz_bloqueo_optimista` | Bloqueo optimista (`version`) |
| `test_cicatriz_reciclaje_de_folios` | Reciclaje de folios |
| `test_cicatriz_idempotencia_de_emergencia` | Idempotencia de emergencia |

---

> **Fin del Tomo III.** El cementerio está completo: **18 incidentes**, **21 Reglas de Oro**, **9 críticas**, **5 cicatrices intocables**, **6 prohibiciones absolutas**, **11 reglas de batalla**, **5 anti-patrones** y **5 bugs representativos**. Nada resumido "para ahorrar espacio" (regla dura §1).