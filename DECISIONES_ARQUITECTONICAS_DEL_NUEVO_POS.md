# DECISIONES ARQUITECTÓNICAS DEL NUEVO POS

**Documento 13 del proyecto del Nuevo POS**
**Estado:** Vigente
**Ámbito:** Todo el proyecto (POS nuevo + ERP futuro)
**Naturaleza:** Índice consolidado de decisiones. **No sustituye a las fuentes; las ordena y las enlaza.**

---

## SECCIÓN 0 — POR QUÉ EXISTE ESTE DOCUMENTO

Las decisiones arquitectónicas del proyecto **ya estaban documentadas**, pero **dispersas**:

- Las directrices transversales (DT-xx) viven en [`DIRECTRICES_TRANSVERSALES_DEL_ERP.md`](./DIRECTRICES_TRANSVERSALES_DEL_ERP.md).
- Las prohibiciones y reglas de batalla viven en el [`PLAN_MAESTRO_DEFINITIVO_POS.md`](./PLAN_MAESTRO_DEFINITIVO_POS.md) §4 y §5.
- Las acciones de arquitectura (A-xx) y los estándares de BD (C-xx) viven en el Plan Maestro §10.
- Las cicatrices (bugs resueltos) viven repartidas entre fichas de fase y el [`TOMO_III_CEMENTERIO_DE_BUGS_Y_CICATRICES.md`](./08-documentacion-final/TOMO_III_CEMENTERIO_DE_BUGS_Y_CICATRICES.md).

El problema no era falta de contenido: era **falta de un punto de entrada único**. Quien busca "¿dónde está decidido X?" tenía que saber de antemano en qué ficha mirar. Este documento cierra ese hueco.

> **Regla de uso:** este documento es un **índice**, no una fuente. Cuando una decisión cambie, se cambia **en su fuente** (la directriz, el Plan Maestro o la ficha) y aquí solo se actualiza el enlace. Nunca se duplica el contenido normativo.

---

## SECCIÓN 1 — LAS TRES FAMILIAS DE DECISIONES

| Familia | Qué decide | Dónde vive la fuente | Prefijo |
|---|---|---|---|
| **Directrices transversales** | Lo que es igual en todos los módulos (tiempo, dinero, identidad, inventario, auditoría, configuración, IA, visión, responsividad, contratos) | [`DIRECTRICES_TRANSVERSALES_DEL_ERP.md`](./DIRECTRICES_TRANSVERSALES_DEL_ERP.md) | `DT-xx` |
| **Acciones de arquitectura** | Cómo se porta y se protege el conocimiento (regla+test, frontera por contratos) | Plan Maestro §10 + [`PLAN_ACCION_ARQUITECTONICO_NUEVO_POS.md`](./PLANES%20DESCONTINUADOS%20DEL%20NUEVO%20POS/PLAN_ACCION_ARQUITECTONICO_NUEVO_POS.md) | `A-xx` |
| **Estándares de base de datos** | Cómo se modelan los datos (PK, tiempo, dinero, concurrencia) | Plan Maestro §10.3 | `C-xx` |

Y una cuarta familia, que no es una decisión sino su **consecuencia histórica**:

| Familia | Qué registra | Dónde vive la fuente |
|---|---|---|
| **Cicatrices (bugs resueltos)** | Qué se rompió y qué se aprendió | Fichas `FICHA_FIX_*` + [`TOMO_III`](./08-documentacion-final/TOMO_III_CEMENTERIO_DE_BUGS_Y_CICATRICES.md) |

---

## SECCIÓN 2 — DIRECTRICES TRANSVERSALES (DT-xx)

> **Fuente canónica:** [`DIRECTRICES_TRANSVERSALES_DEL_ERP.md`](./DIRECTRICES_TRANSVERSALES_DEL_ERP.md).
> Cada directriz trae **la regla**, **el ancla** (archivo + línea), **la verificación** y **la matriz de cumplimiento por módulo**.

| ID | Tema | La regla, en una frase | Sección fuente |
|---|---|---|---|
| **DT-01** | Tiempo | Todo timestamp se guarda en UTC y se muestra en hora local; la conversión es de presentación, nunca de almacenamiento | §2 |
| **DT-02** | Dinero | El dinero se guarda en `Numeric(12,2)`, nunca en `Float`; viaja como STRING y se coercionar con `Number()` antes de operarlo | §3 |
| **DT-03** | Identidad | El UUID identifica; el folio comunica. Nunca se intercambian | §4 |
| **DT-04** | Inventario | El inventario es un ledger inmutable; solo el módulo dueño escribe; nadie lee `products.stock` | §5 |
| **DT-05** | Auditoría | Toda operación que mueve dinero o inventario deja rastro: quién, cuándo, qué | §6 |
| **DT-06** | Configuración del negocio | Los valores que afectan a todos los módulos se declaran una sola vez, en Vista General, y se persisten en `system_settings` | §6.5 |
| **DT-07** | Inteligencia artificial | La IA se gestiona en un solo módulo (Centro de IA); ningún módulo contiene el motor ni importa sus dependencias; un fallo de IA nunca bloquea una venta | §6.6 |
| **DT-08** | Visión cenital | La visión opera sobre cámara cenital con iluminación dedicada; el umbral 0.35 es de calibración, no universal, y es configurable | §6.7 |
| **DT-09** | Responsividad y lenguaje visual | Todo componente nace responsivo (3 modos), cumple R-01 a R-04 y usa tokens semánticos; un componente que viole esto NO se acepta | §14 |
| **DT-10** | Contrato obligatorio inter-modular | Toda funcionalidad que cruce la frontera de un módulo DEBE tener su contrato redactado ANTES de considerarse terminada | §15 |

> **Nota sobre numeración:** el documento fuente contiene dos secciones rotuladas `DT-04` y dos rotuladas `DT-05` (§5/§12 y §6/§13). Las de §12 y §13 son **aplicaciones al POS** de las directrices de §5 y §6, no directrices nuevas. Este índice usa la numeración de §2–§6.5 y §14–§15, que es la canónica.

---

## SECCIÓN 3 — ACCIONES DE ARQUITECTURA (A-xx)

> **Fuente canónica:** Plan Maestro §10 + [`PLAN_ACCION_ARQUITECTONICO_NUEVO_POS.md`](./PLANES%20DESCONTINUADOS%20DEL%20NUEVO%20POS/PLAN_ACCION_ARQUITECTONICO_NUEVO_POS.md).

| ID | Decisión | Por qué | Dónde vive |
|---|---|---|---|
| **A-01** | La unidad de migración es **la regla + su test**, no la regla sola | Una regla sin test es una regla que se pierde en el siguiente refactor | Plan Maestro §10.1; [`TOMO_II`](./08-documentacion-final/TOMO_II_LAS_95_REGLAS_DE_NEGOCIO.md) §1 |
| **A-02** | **Frontera por contratos:** el POS no importa modelos de otros módulos; cada dependencia se resuelve por contrato explícito | Los 10 acoplamientos del POS viejo (AC-01 a AC-10) se reemplazan por contratos | Plan Maestro §10.2; [`TOMO_IV`](./08-documentacion-final/TOMO_IV_CONTRATOS_Y_FRONTERAS.md) §1 |
| **A-03** | Cada regla crítica tiene un **test guardián**; el CI falla si falta | Sin guardián, la regla se erosiona sin que nadie lo note | Plan Maestro §10.5 |
| **A-04** | **Outbox transaccional obligatorio** (no `try/except pass`) | Un evento perdido es un descuadre silencioso | `PLAN_ACCION_ARQUITECTONICO` §A-04 |
| **A-05** | **Identidad (UUID) ≠ Presentación (folio)** | Confundirlos rompe la fusión entre sucursales | `PLAN_ACCION_ARQUITECTONICO` §A-05; refuerza DT-03 |

---

## SECCIÓN 4 — ESTÁNDARES DE BASE DE DATOS (C-xx)

> **Fuente canónica:** Plan Maestro §10.3.

| ID | Estándar | Regla |
|---|---|---|
| **C-01** | PK | UUID en todas las tablas (no enteros autoincrementales) |
| **C-02** | Tiempo | `DateTime(timezone=True)` UTC siempre (nunca naive) |
| **C-03** | Dinero | `Numeric(12,2)` (NUNCA `Float`) |
| **C-04** | Concurrencia | Columna `version` para bloqueo optimista |

---

## SECCIÓN 5 — LAS 6 PROHIBICIONES ABSOLUTAS

> **Fuente canónica:** Plan Maestro §4. Extraídas de 7 meses de operación real + 1 error de construcción del POS nuevo. **Toda línea de código del POS nuevo las respeta.**

| # | Prohibición | Origen |
|---|---|---|
| 1 | **NO** reintroducir auto-save, timers ni `setInterval` para guardar el carrito. La persistencia es **atómica por ítem** | v6.0 |
| 2 | **NO** hacer `clearCart()` sin confirmación HTTP 200 del servidor **Y** verificación post-envío | v6.1 |
| 3 | **NO** leer variables de estado (`cart`, `currentAccountNum`) dentro de callbacks asíncronos — usar siempre `useRef` | Ticket #906 ($124→$2) |
| 4 | **NO** almacenar candados de terminal en RAM de Python — solo en PostgreSQL (`terminal_locks`) | Incidente de terminales fantasma |
| 5 | **NO** generar folios en el frontend — solo el backend los genera vía secuencia atómica de PostgreSQL | Incidente de folios duplicados |
| 6 | **NO** duplicar módulos que el ERP ya tiene (login, empleados, perfiles). El POS es un MÓDULO del ERP, no una app suelta | Login propio rechazado por YAGNI |

---

## SECCIÓN 6 — REGLAS ARQUITECTÓNICAS DERIVADAS DE LA BATALLA

> **Fuente canónica:** Plan Maestro §5. Cada regla nace de un incidente real.

| Regla | Origen | Implementación |
|---|---|---|
| **Verificar, no asumir** | Autocrítica F7.5 v3.0→v3.1 (4 defectos) | Antes de nombrar una tabla/campo/contrato/endpoint, verificar que existe (archivo + línea). Si la verificación contradice el plan, manda la verificación |
| **Contrato de resultado discriminado** | Incidente v7.0.3 (cuentas perdidas) | Toda función de persistencia retorna `{ outcome, reason }`. PROHIBIDO asumir "no lanzar excepción" = éxito |
| **Verificación post-envío** | Incidente $453 (cuenta fantasma) | Después de HTTP 200, verificar que el ticket existe en la BD |
| **withRetries centralizado** | Asimetría v7.0.1 | Todas las operaciones usan el mismo patrón: 3 intentos, backoff 1s/2s/3s |
| **Respuesta ligera** | Optimización v7.0 (rush hour) | Operaciones atómicas devuelven 5 campos escalares, no JOINs completos |
| **Timestamps UTC** | Hallazgo H2 | `utcnow()` siempre, nunca `datetime.now()`. Store UTC, Display Local |
| **Primitivos en deps** | Hallazgo H1 | `useEffect` deps = primitivos (`currentUser?.id`), no objetos |
| **3 estados de terminal** | Incidente v12 (veracidad) | libre / mío / ajeno. NUNCA colapsar a 2 ramas |
| **sendBeacon al cerrar** | Hallazgo H3 | `beforeunload` libera lock + persiste carrito vía beacon |
| **Espejo de limpieza** | Incidente v7.0.3 (refs residuales) | Toda rama de salida limpia exactamente los mismos refs que la rama de éxito |
| **Banner rojo fijo** | Incidente $453 | Error de red = banner permanente + botón bloqueado, no toast efímero |

---

## SECCIÓN 7 — REGISTRO DE BUGS RECIENTES (BUG-01 … BUG-10h) + PARIDAD F12.23 / F12.23b

> **Fuente canónica:** las fichas `FICHA_FIX_*` del repo [`NUEVO-POS`](https://github.com/vikutasan/NUEVO-POS) (`docs/05-plan-de-construccion/`) y el [`TOMO_III`](./08-documentacion-final/TOMO_III_CEMENTERIO_DE_BUGS_Y_CICATRICES.md).
>
> **Nota de nomenclatura:** BUG-01, BUG-04 y BUG-05 tienen ficha dedicada `FICHA_FIX_BUG0N_*`. BUG-02 y BUG-03 **no** la tienen: su contenido vive dentro de fichas de fase. Se registran aquí con su ubicación real para que sean localizables.

| Bug | Qué se rompió | Dónde vive la documentación | Lección |
|---|---|---|---|
| **BUG-01** | La sesión de terminal no se abría al seleccionar la terminal (error RN-24 al primer ticket) | `FICHA_FIX_BUG01_SESION_DE_TERMINAL.md` | La sesión de terminal es un requisito previo del ticket, no un efecto colateral |
| **BUG-02** | El color del post-it del pizarrón usaba el vocabulario viejo de ids (`T6`) en vez de `TERM-0X` | `FICHA_F12_6_PARIDAD_PIZARRON.md` §BUG-02 + test `OpenAccountsCorkboard.bug02.test.jsx` | Al heredar código del POS viejo, heredar también su vocabulario es un error |
| **BUG-03** | La paleta de colores del post-it no coincidía con el diseño; la asignación debía ser manual | `FICHA_F12_6_PARIDAD_PIZARRON.md` §BUG-03 + `PLAN_SELECTOR_COLOR_POST_IT.md` + `FICHA_F13_3_SELECTOR_COLOR_POST_IT.md` | Una paleta de diseño se adopta completa (21 colores), no por aproximación |
| **BUG-04** | La fecha de compromiso (`committed_at`) no se restauraba al recuperar un pedido del pizarrón | `FICHA_FIX_BUG04_FECHA_COMPROMISO.md` | El bloque `order_*` (contrato 3) nombra las notas `order_notes`, no `notes`; leer el campo equivocado deja el dato vacío |
| **BUG-05** | `CAJA` se trataba como una terminal (corrompía `terminal_config.json`) y el botón «Guardar cambios» del gestor parecía muerto | `FICHA_FIX_BUG05_CAJA_NO_ES_TERMINAL.md` + `ARQUITECTURA_TERMINALES_Y_CAJA.md` + `LECCIONES_DE_UI.md` | **Toda terminal es una caja en potencia; `CAJA` no es una terminal.** Y un `return` temprano puede dejar fuera un elemento transversal (el toast) |
| **BUG-06** | Al enviar un ticket desde TERM-06 aparecía «hay productos sin guardar en el servidor» aunque el ticket estaba completo | `FICHA_FIX_BUG06_VERIFY_POR_COBERTURA.md` | La verificación post-envío debe comparar lo que el servidor **tiene**, no lo que el cliente **cree** que envió |
| **BUG-07** | Un producto repetido en el carrito bloqueaba el envío: la verificación comparaba LÍNEAS del carrito contra FILAS del servidor, pero RN-17 fusiona duplicados en una sola fila | `FICHA_FIX_BUG07_VERIFY_POR_UNIDADES.md` | Verificar por **UNIDADES**, no por líneas: el servidor fusiona (RN-17) y la comparación debe respetar esa regla |
| **BUG-08** | Al cobrar una cuenta ajena desde otra terminal aparecía «No hay turno de caja abierto para esta terminal»: el backend derivaba el turno de caja de la terminal de **ORIGEN** del ticket, no de la que **COBRA** | `FICHA_FIX_BUG08_CAJA_COBRA_CUENTAS_AJENAS.md` + `ARQUITECTURA_TERMINALES_Y_CAJA.md` §10 | **El turno de caja pertenece a la terminal que COBRA, no a la del ticket.** El cliente declara `cash_session_id`; el backend lo **valida** (E-13) y **nunca** sobreescribe el `terminal_id` del ticket (RN-12) |
| **BUG-09** | Tras configurar el color de una terminal en el Gestor, los post-its del pizarrón seguían saliendo **amarillos** (color por defecto) incluso refrescando el navegador | `LECCIONES_DE_UI.md` §7 + test `terminalService.colores.test.jsx` | **Cambiar la FORMA de retorno de un servicio rompe a sus consumidores en silencio.** `fetchTerminalConfig()` pasó de devolver un arreglo a `{terminals, orden}`; el consumidor seguía iterándolo con `for...of` (lanza `TypeError` sobre un objeto) y un `catch {}` vacío lo tragaba, dejando el mapa de colores vacío. Centralizar la lectura en un helper tolerante a ambos formatos elimina la clase de bug |
| **BUG-10** | Con muchas cuentas, el pizarrón se desbordaba pero **no aparecía barra de scroll lateral**, a diferencia del POS viejo | `LECCIONES_DE_UI.md` §8 + test `OpenAccountsCorkboard.f5_3.test.jsx` (criterio 9) | **Un contenedor que crece sin límite deja el scroll «invisible».** El tablero crecía con el contenido y el scroll quedaba en el overlay del modal. Para una barra visible y contenida, el marco acota su altura (`max-h-[85vh] flex-col overflow-hidden`) y el hijo que desborda lleva `flex-1 overflow-y-auto` |
| **BUG-10b** | El primer arreglo de BUG-10 puso `flex-1 overflow-y-auto` **en el `<ul>` del grid**: en vez de aparecer la barra, los post-its se **encimaron** parcialmente unos sobre otros | `LECCIONES_DE_UI.md` §8 (segunda vuelta) + test `OpenAccountsCorkboard.f5_3.test.jsx` (criterio 9, actualizado) | **`flex-1` en un grid de tarjetas `aspect-square` comprime las filas y encima las tarjetas.** El scroll va en un **wrapper** de altura acotada (`flex-1 overflow-y-auto`); el grid dentro queda con alto automático (`content-start`, **sin** `flex-1` ni `overflow-y-auto`). Separación amplia (`gap-10`/`lg:gap-12`) para que la rotación (±3°) no toque al vecino |
| **BUG-10c** | Con el scroll ya resuelto, el post-it se veía **demasiado largo**: un PEDIDO con poco texto quedaba estirado y el total se iba al fondo, dejando un hueco vacío enorme en medio | `LECCIONES_DE_UI.md` §8 (tercera vuelta) + test `OpenAccountsCorkboard.f5_3.test.jsx` (criterio 10) | **Un `min-h` fijo desacopla el alto del ancho de la columna.** Con `min-h-[11rem]` + `justify-between`, el contenido escaso deja el hueco en el centro. Para una tarjeta tipo post-it, usar `aspect-square` (alto = ancho de columna), como el POS viejo, **no** un `min-h` fijo |
| **BUG-10d** | Tras BUG-10/10b/10c, la barra de scroll **seguía sin aparecer** y los post-its salían **«mordidos»** (recortados por abajo) | `LECCIONES_DE_UI.md` §8 (cuarta vuelta) + test `OpenAccountsCorkboard.f5_3.test.jsx` (criterio 9, actualizado) | **`flex-1` + `overflow-y-auto` solo desplazan si el padre flex tiene ALTURA DEFINIDA.** El tablero usaba `max-h-[85vh]` (un máximo, no una altura): `flex-1` resolvía a `auto`, el wrapper crecía y la barra nunca aparecía; y el `overflow-hidden` del marco recortaba las filas de abajo. Fix: `h-[85vh]` (altura definida) + `min-h-0` en el wrapper de scroll. El POS viejo no sufre esto porque su tablero es `aspect-[16/9]` |
| **BUG-10e** | Los arreglos de 10d eran correctos pero seguían sin verse: el navegador no actualizaba el código por culpa de un HMR roto | `LECCIONES_DE_UI.md` §8 (quinta vuelta) + test `OpenAccountsCorkboard.f5_3.test.jsx` (compuerta de fuente, ver nota) | **Exportar una función pura junto al componente rompe el Fast Refresh de Vite.** El módulo exportaba `colorDe` (función pura) además del componente por defecto; Vite no podía refrescar en caliente y el navegador nunca recibía los arreglos de 10b/10c/10d, dejando al usuario viendo el `max-h-[85vh]` original que recortaba los post-its. Al quitar el `export` de `colorDe` (la función permanece en el módulo, ya no se exporta) el HMR resucitó. Además, el modal padre (`fixed inset-0 ... overflow-y-auto`) secuestraba el scroll: se cambió a `items-center` sin `overflow-y-auto`, de modo que el scroll nace dentro del tablero. |
| **BUG-10f** | Con el HMR ya sano (10e), el pizarrón **seguía** sin barra y con post-its «mordidos»: el fix de 10d (`h-[85vh]`) era **insuficiente** | `LECCIONES_DE_UI.md` §8 (sexta vuelta) + test `OpenAccountsCorkboard.f5_3.test.jsx` (criterio 9 + compuerta de fuente, actualizados) | **Una altura en `vh` NO cabe si el contenedor está dentro de paddings anidados.** El tablero vivía dentro de DOS paddings (`p-4` del wrapper exterior + `p-4` del modal), así que su altura total era `85vh + 2rem + 2rem > 100vh` en pantallas normales; como el modal es `items-center` sin `overflow-y-auto`, el navegador recortaba arriba y abajo (post-its «mordidos») sin barra en ningún lado. La solución es la del POS viejo: **`aspect-[16/9]`** (la altura se DERIVA del ancho, acotado por `max-w-[1100px]`, así el tablero SIEMPRE cabe) + eliminar el `p-4` del wrapper exterior para no sumar altura. Regla: para un marco que debe caber siempre, derivar la altura del ancho (`aspect-*`), no fijarla en `vh` |
| **BUG-10g** | Con el HMR sano (10e) y el `aspect-[16/9]` aplicado (10f), el usuario **seguía** sin ver la barra tras refrescar: el `aspect` **solo** no basta | `LECCIONES_DE_UI.md` §8 (séptima vuelta) + test `OpenAccountsCorkboard.f5_3.test.jsx` (criterio 9 + compuerta de fuente, endurecidos) | **Un `aspect-*` no basta como única restricción de tamaño.** `aspect-[16/9]` fija el alto a partir del ANCHO (1100px → ~619px); en una ventana **BAJA** (p. ej. 1366×600 tras las barras del navegador) `619px + 2rem > 100vh`, el tablero desborda el modal `items-center` (sin `overflow-y-auto`) y el navegador lo recorta arriba y abajo (post-its «mordidos») **sin barra**. Fix: **`aspect-[16/9]` + `max-h-[calc(100vh-2rem)]`** — el `aspect` da la proporción, el `max-h` da el tope duro que garantiza que quepa. Regla: a un `aspect-*` añádele **siempre** un tope al viewport (`max-h-[calc(100vh-2rem)]`) |
| **BUG-10h** | Con el HMR sano (10e), el `aspect-[16/9]` (10f) y el tope `max-h-[calc(100vh-2rem)]` (10g), el usuario **seguía** sin ver la barra: el CSS era correcto, pero el **SO la ocultaba** | `LECCIONES_DE_UI.md` §8 (octava vuelta) + test `OpenAccountsCorkboard.f5_3.test.jsx` (criterio 9 + compuerta de fuente, endurecidos) | **`overflow-y: auto` no garantiza que la barra se VEA.** Un arnés por Chrome DevTools Protocol (CDP) midió el layout **computado** real y probó que el wrapper **sí desplaza** (`clientHeight=400` vs `scrollHeight=654`, `canScroll: true`) y que la barra ocupa **10px** clásicos (no overlay): el CSS era correcto. El culpable era la **política del SO**: en Windows 11 con «Ocultar automáticamente las barras de desplazamiento», `auto` usa barras **overlay** invisibles hasta hacer scroll. Fix: **`overflow-y-scroll`** (carril siempre reservado) + `scrollbar-gutter: stable` + propiedades estándar `scrollbar-width`/`scrollbar-color` + track con fondo visible. Regla: si la barra debe ser siempre visible, usa `scroll` (garantía del CSS), no `auto` (promesa del SO); y **mide el layout computado** antes de seguir tocando CSS a ciegas |
| **F12.23** | La cantidad de una línea del ticket se ajustaba con botones laterales `−`/`+`; el POS viejo, en cambio, **teclea** la cantidad (tocar la cantidad → teclado numérico). El `±` es cómodo para `1→2` pero **hostil** para `1→24` (24 taps) | `LECCIONES_DE_UI.md` §8 (novena vuelta) + test `SalesReceipt.f12_23.test.jsx` (8 tests) + `components.f3_4.test.jsx` + `RetailVisionPOS.f3_cierre.test.jsx` | **La paridad (§6.8) exige portar el PATRÓN de interacción, no solo el resultado.** Cuando el POS viejo resuelve algo con un gesto distinto (teclado numérico en vez de `±`), «mejorarlo» con el patrón que ya teníamos rompe la paridad de operación. La cantidad pasó a ser un **botón clicable** (`aria-label="Modificar cantidad de <producto>"`) que abre un **modal** (`role="dialog"`, `aria-modal="true"`) reutilizando el `TecladoNumerico.jsx` de F9.0.2; `OK` confirma, `Cancelar` cierra sin cambios, y `0` + `OK` **quita la línea** (regla heredada del POS viejo). La nueva prop `onCambiarCantidad(linea, nuevaCantidad)` emite la cantidad **exacta**; el contenedor la aplica (`cambiarCantidad`) o quita la línea (`quitarLinea`) si es `0`. Regla: la comodidad de un control depende del **rango** de valores; un `±` que sirve para `1→2` puede ser peor que un teclado para `1→24` |
| **F12.23b** | Tras portar el gesto (F12.23), tocar **varias veces** la misma ficha del producto creaba **varias líneas de cantidad 1** en vez de **una sola línea con la cantidad acumulada**, como hacía el POS viejo | `LECCIONES_DE_UI.md` §8 (décima vuelta) + test `useCart.f12_23b.test.jsx` (6 tests) | **La paridad de operación tiene DOS mitades: el gesto y la regla de acumulación.** El POS viejo fusionaba por `product.id` (`apps/pos/hooks/useCart.js:105`); el nuevo fusionaba por `item_id`, pero `agregarProducto` **no pasa** `item_id` y `anadirLinea` generaba un **UUID nuevo en cada tap**, así que el `find` nunca encontraba la línea previa → N líneas de 1. La regla RN-17 estaba **escrita en un comentario** pero **no implementada** en esa ruta. Fix: si el llamador **no** trae `item_id`, se fusiona por **`product_id`**; si **sí** lo trae, se conserva la ruta por identidad. Además, se resuelve la línea existente en el espejo `lineasRef.current` y se **reutiliza su `item_id`** en `anadirItem` para que el **backend también fusione** (contrato 18) en vez de duplicar la fila. Regla: portar solo el gesto deja un POS que *parece* igual pero **duplica** líneas; y una regla escrita **solo en un comentario** no es una regla, es una **intención** |

> **Nota sobre la compuerta de BUG-10e (corrección de auditoría):** la primera versión del test de BUG-10e era una **tautología**: simulaba `clientHeight=800`/`scrollHeight=1200` en jsdom y afirmaba `1200 > 800`, verdadero por construcción — pasaba con el código roto y con el sano. Se reemplazó por una **compuerta de fuente**: el test lee el código de [`RetailVisionPOS.jsx`](https://github.com/vikutasan/NUEVO-POS/blob/main/apps/pos/src/RetailVisionPOS.jsx) y de [`OpenAccountsCorkboard.jsx`](https://github.com/vikutasan/NUEVO-POS/blob/main/apps/pos/src/components/OpenAccountsCorkboard.jsx) y afirma que el bloque del modal **no** contiene `overflow-y-auto` (y sí `items-center`/`justify-center`/`role="dialog"`), y que el tablero usa `aspect-[16/9]` (no `max-h-[85vh]` ni `h-[85vh]`) con `min-h-0` en el wrapper de scroll. Se **verificó que falla** con el código viejo y **pasa** con el arreglo. Lección transversal: *una compuerta que no puede fallar no es una compuerta*.
>
> **Nota sobre BUG-10f (por qué 10d no bastó):** el fix de 10d (`h-[85vh]`) era **correcto en su diagnóstico** (un `flex-1` necesita altura definida) pero **incompleto en su aritmética**: no contó los paddings anidados. La compuerta de 10d afirmaba `h-[85vh]` y por eso **no podía detectar** el desborde real (el test no mide layout, solo lee clases). La compuerta de 10f se endureció: ahora **rechaza** `h-[85vh]` además de `max-h-[85vh]`, y exige `aspect-[16/9]`. Lección transversal: *una compuerta que solo verifica la clase que tú escribiste no verifica el efecto que el usuario ve*.
>
> **Nota sobre BUG-10g (por qué 10f no bastó):** el fix de 10f (`aspect-[16/9]`) era **correcto en su diagnóstico** (derivar la altura del ancho evita los paddings anidados) pero **incompleto en su alcance**: el `aspect` deriva el alto del **ancho**, y el ancho puede ser grande (1100px → ~619px). En una ventana **baja** ese alto más los paddings supera `100vh` y el tablero desborda el modal `items-center` (sin scroll), recortando los post-its. La compuerta de 10f exigía `aspect-[16/9]` pero **no** un tope de alto, así que **no podía detectar** el desborde en ventanas bajas. La compuerta de 10g **exige además** `max-h-[calc(100vh-2rem)]` en la línea del tablero (criterio 9 y compuerta de fuente). Lección transversal: *una compuerta que verifica la proporción pero no el tope no verifica que el marco quepa*.
>
> **Nota sobre BUG-10h (por qué 10g no bastó, y cómo se dejó de adivinar):** tras 10e/10f/10g el usuario **seguía** sin ver la barra. En lugar de seguir «arreglando» CSS a ciegas, se escribió un **arnés de medición** que lanza Chrome headless por el **Chrome DevTools Protocol** (CDP, `--remote-debugging-port` + el `WebSocket` global de Node 24), inyecta el markup exacto del tablero y lee el layout **computado** real. El veredicto: el wrapper **sí desplaza** (`clientHeight=400` vs `scrollHeight=654`, `canScroll: true`) y la barra **sí se renderiza** con **10px** de ancho clásico (no overlay). Es decir, **el CSS era correcto**; el problema era la **política del SO** (Windows 11 con «Ocultar automáticamente las barras de desplazamiento» usa barras overlay invisibles hasta hacer scroll). La compuerta de 10g exigía `overflow-y-auto`, que es precisamente lo que el SO oculta en modo overlay; la compuerta de 10h **exige** `overflow-y-scroll` (y **rechaza** `auto`) en el wrapper, y **exige** las propiedades estándar `scrollbar-gutter: stable`, `scrollbar-width: thin` y `scrollbar-color:` en el bloque `<style>`. Lección transversal: *una compuerta que exige `auto` no verifica que la barra se vea; `auto` es una promesa del SO, `scroll` es una garantía del CSS. Y ante un síntoma que resiste varios fixes, MIDE el layout computado antes de seguir tocando CSS*.
>
> **Deuda de nomenclatura detectada:** BUG-02 y BUG-03 no siguen la convención `FICHA_FIX_BUG0N_*`. Este registro los hace localizables sin renombrar archivos (renombrar rompería enlaces relativos y referencias en tests). Si en el futuro se decide uniformar, este documento es el punto de partida.
>
> **Deuda de nomenclatura (DEUDA-BUG08 · Obs. 1):** el campo `cash_session_id` del contrato 33 (`pos.cobrar_ticket`) es **heredado** del POS viejo y **engañoso**: en el cobro ajeno no es «la sesión de caja del ticket», sino **el turno de la caja que COBRA**. Se conserva por retrocompatibilidad de contrato (está en el contrato 33, en el frontend y en los tests). El vocabulario correcto se documenta en `ARQUITECTURA_TERMINALES_Y_CAJA.md` §10.6 y §11.1 y en el propio contrato 33. Si en el futuro se decide renombrar (p. ej. `turno_de_caja_que_cobra`), este registro es el punto de partida.

---

## SECCIÓN 8 — CÓMO USAR ESTE DOCUMENTO

1. **Si buscas una decisión:** encuéntrala en la tabla de su familia (Secciones 2–6) y sigue el enlace a la fuente.
2. **Si vas a tomar una decisión nueva:** primero verifica que no exista ya (Regla Dura 2: *verificar, no asumir*). Si es transversal, va a `DIRECTRICES_TRANSVERSALES_DEL_ERP.md`; si es de arquitectura, al Plan Maestro §10.
3. **Si resuelves un bug:** documenta la cicatriz en su ficha `FICHA_FIX_*` y añade una fila a la Sección 7.
4. **Si una decisión cambia:** cámbiala **en su fuente** y actualiza solo el enlace aquí. Este documento nunca duplica el contenido normativo.

---

## SECCIÓN 9 — RESUMEN EN UNA FRASE

> **Las decisiones del proyecto no faltaban: estaban dispersas. Este documento es el índice que las ordena y las enlaza, sin duplicar ninguna.**
