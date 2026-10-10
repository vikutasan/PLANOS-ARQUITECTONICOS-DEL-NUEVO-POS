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

## SECCIÓN 7 — REGISTRO DE BUGS RECIENTES (BUG-01 … BUG-08)

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

> **Deuda de nomenclatura detectada:** BUG-02 y BUG-03 no siguen la convención `FICHA_FIX_BUG0N_*`. Este registro los hace localizables sin renombrar archivos (renombrar rompería enlaces relativos y referencias en tests). Si en el futuro se decide uniformar, este documento es el punto de partida.

---

## SECCIÓN 8 — CÓMO USAR ESTE DOCUMENTO

1. **Si buscas una decisión:** encuéntrala en la tabla de su familia (Secciones 2–6) y sigue el enlace a la fuente.
2. **Si vas a tomar una decisión nueva:** primero verifica que no exista ya (Regla Dura 2: *verificar, no asumir*). Si es transversal, va a `DIRECTRICES_TRANSVERSALES_DEL_ERP.md`; si es de arquitectura, al Plan Maestro §10.
3. **Si resuelves un bug:** documenta la cicatriz en su ficha `FICHA_FIX_*` y añade una fila a la Sección 7.
4. **Si una decisión cambia:** cámbiala **en su fuente** y actualiza solo el enlace aquí. Este documento nunca duplica el contenido normativo.

---

## SECCIÓN 9 — RESUMEN EN UNA FRASE

> **Las decisiones del proyecto no faltaban: estaban dispersas. Este documento es el índice que las ordena y las enlaza, sin duplicar ninguna.**
