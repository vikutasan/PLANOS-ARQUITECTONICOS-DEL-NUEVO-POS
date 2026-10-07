# TOMO I — VISIÓN Y ARQUITECTURA

> **Documentación final del POS nuevo "R de Rico"** — Tomo I de VII.
> **Fuente principal:** [`PLAN_MAESTRO_DEFINITIVO_POS.md`](../PLAN_MAESTRO_DEFINITIVO_POS.md:1) §1–§5, §8, §10.
> **Propósito de este tomo:** que una IA sin contexto entienda **qué es** el POS nuevo, **por qué** se construyó así, **qué está prohibido** y **cómo se verifica** cada afirmación.

---

## ÍNDICE DEL TOMO I

1. [El objetivo central](#1-el-objetivo-central)
2. [El POS es el primer módulo de un ERP reconstruido](#2-el-pos-es-el-primer-módulo-de-un-erp-reconstruido)
3. [El ERP viejo es el ORÁCULO, no el MODELO](#3-el-erp-viejo-es-el-oráculo-no-el-modelo)
4. [Las 2 reglas duras inquebrantables](#4-las-2-reglas-duras-inquebrantables)
5. [Las 6 prohibiciones absolutas](#5-las-6-prohibiciones-absolutas)
6. [Las 10 reglas arquitectónicas derivadas de la batalla](#6-las-10-reglas-arquitectónicas-derivadas-de-la-batalla)
7. [Los 16 estándares de calidad del constructor](#7-los-16-estándares-de-calidad-del-constructor)
8. [El principio "de adentro hacia afuera"](#8-el-principio-de-adentro-hacia-afuera)
9. [Las 5 lecciones de integración (F4.5 → F10.6)](#9-las-5-lecciones-de-integración-f45--f106)
10. [La frontera por contratos (A-02 / O-23)](#10-la-frontera-por-contratos-a-02--o-23)
11. [Autonomía vs. Observabilidad](#11-autonomía-vs-observabilidad)
12. [Los estándares de base de datos (C-01 a C-04)](#12-los-estándares-de-base-de-datos-c-01-a-c-04)
13. [Los 5 greps de CI](#13-los-5-greps-de-ci)
14. [La matriz de trazabilidad regla → test](#14-la-matriz-de-trazabilidad-regla--test)
15. [Los dos repositorios del proyecto](#15-los-dos-repositorios-del-proyecto)
16. [El resultado esperado](#16-el-resultado-esperado)

---

## 1. EL OBJETIVO CENTRAL

**Construir la versión mejorada del módulo "Punto de Venta IA" de R de Rico**, de forma que sin perder la funcionalidad actual e incorporando otras nuevas, parezca que este módulo se diseñó desde un principio por un arquitecto senior.

Esta nueva versión:

- **Incorpora el aprendizaje documentado** de haber sido probado en operación real (**7 meses, 6+ terminales, 30,000+ tickets**).
- **Nace tomando en cuenta las cicatrices** documentadas en la DOCUMENTACIÓN MAESTRA del POS viejo.
- **Produce un POS robusto** fraguado en la batalla pero con calidad de arquitectura senior.
- **Sienta las bases** del nuevo ERP que se construirá después (sistema de temas, contratos, motor compartido).

### 1.1 ¿Por qué se reescribió? — el diagnóstico

El POS viejo **funciona y está en producción**, pero su arquitectura es **parchada**: se construyó sobre la marcha, agregando y modificando sin un plano. El POS nuevo **no es un capricho estético**: es la corrección de defectos que costaron dinero real.

**Lo que DeepSeek construyó (~20% del módulo POS):**

| Categoría | Archivos | Estado |
|---|---|---|
| `RetailVisionPOS.jsx` | 1 (11 KB vs 40 KB del viejo) | Simplificado: flujo básico catálogo→ticket→cobro |
| Componentes | 5 de 12 | Funcionales pero sin las funciones reales |
| Hooks | 1 (`useModo.js` — nuevo, no existe en el viejo) | 0 de 9 hooks del viejo |
| Servicios | 0 de 2 | Nada |
| Utils | 0 de 3 | Nada |
| State | 0 de 1 | Nada |
| Config | 1 | ✅ Completo |
| API backend | Endpoints básicos de catálogo y sesión | Faltan terminales, caja, tickets atómicos |

**Lo que DeepSeek NO construyó (~80% del módulo POS):**

- ❌ **Selector de Terminales** (landing page — 31 KB)
- ❌ **Gestor de Caja** (cortes, arqueos, turnos — 56 KB)
- ❌ **Panel de Voz** (agregar por voz — 14.7 KB)
- ❌ **Visor de Cámara IA** (reconocimiento visual — 4.4 KB)
- ❌ **Programación de Pedidos** (pedidos programados — 21.6 KB)
- ❌ **Pizarrón de Cuentas** (cuentas abiertas en paralelo — 8.4 KB)
- ❌ **9 hooks** del POS viejo (carrito, sesión, terminal lock, voz, visión, red, escáner, beforeunload, ticket actions)
- ❌ **Checkout completo** (solo tenía 5.5 KB de 28.7 KB)
- ❌ **Persistencia atómica** (modelo SaaS v6.0)
- ❌ **Contratos de resultado discriminado** (v7.0.3)

**Lo que NOSOTROS construimos (la arquitectura del edificio):**

- ✅ **Theme Engine** — motor de temas compartido con 3 variantes (38 tests)
- ✅ **Contratos de módulo** — sistema de validación WCAG AA
- ✅ **Tokens CSS** — 6 base + 3 madera, con variables en runtime
- ✅ **Estética replicada** — header, tarjetas, ticket, categorías como el POS viejo
- ✅ **Extractor IA de branding** — endpoint para extraer tokens desde imágenes
- ✅ **Tailwind configurado** — con tokens semánticos en vez de hex sueltos

**La conclusión:** la arquitectura (cimientos) está sólida. Las funciones (habitaciones) se construyeron sobre ella respetando las lecciones de la documentación de batalla.

---

## 2. EL POS ES EL PRIMER MÓDULO DE UN ERP RECONSTRUIDO

> **Aclaración explícita del dueño (29 Sep 2026).** Se documenta aquí para que no exista posibilidad de confusión cuando se construyan los demás módulos.

**La visión completa:**

1. **Esto no es un proyecto del POS.** Es el **primer módulo** de una reconstrucción del ERP completo. El método que se usa aquí (diagnóstico → plan por partes → sub-fases con gate → ficha → commit) se **replicará** en cada módulo del ERP.
2. **El POS se integra con los demás módulos del ERP**, siempre **por contrato**:
   - **Centro de IA** (voz, visión, OCR, NLU) — ver **DT-07**.
   - **CRM / Notificaciones** (clientes, lealtad, envío de tickets) — ver **Fase 8**.
   - **Vista General** (zona horaria, moneda, sucursal) — ver **DT-06**.
   - **Estadísticas** (contexto diario, resúmenes de venta) — ver **§3** y **F10.4**.
3. **Lo que sí podemos hacer ahora** es **dejar el POS preparado** para que, cuando cada módulo sea rehecho (uno a uno, por el dueño), la integración sea limpia y no haya que reescribir el POS. Eso significa:
   - **Declarar los contratos** que faltan (empezando por los de IA en F7.0).
   - **No inventar** el motor de IA, ni el CRM, ni el selector de valores transversales dentro del POS.
   - **Acoplar por contrato** a los módulos viejos que ya existen (p. ej. Estadísticas en F10.4), sin heredar su arquitectura parchada.
   - **Respetar la degradación elegante**: si un módulo del ERP cae, el POS sigue vendiendo.

**La regla que lo resume:**

> **El POS es hermano de los demás módulos del ERP, no su padre. Consume por contrato; nunca los contiene.**

---

## 3. EL ERP VIEJO ES EL ORÁCULO, NO EL MODELO

> **Aclaración explícita del dueño (30 Sep 2026).** Corrige el supuesto más peligroso que se podría hacer al leer el Plan Maestro.

**El hecho, sin adornos:**

1. **El ERP que corre el POS viejo NO está vacío.** Existe, funciona, y tiene módulos reales (Estadísticas, CRM, Almacenes, RRHH, Auditoría, Reparto Grandeza, etc.). **No hay que inventarlos desde cero.**
2. **Pero ese ERP se construyó SOBRE LA MARCHA**, módulo por módulo, según las necesidades operativas del día a día, **agregando y modificando cosas** sin una arquitectura planificada de antemano. Es, en palabras del dueño, una **"arquitectura parchada"**.
3. **Consecuencia directa:** cada módulo viejo arrastra los mismos vicios que el POS viejo arrastraba (acoplamiento, tablas compartidas, lógica duplicada, ausencia de contratos, ausencia de tests). **No se puede asumir que un módulo viejo es "la versión buena" solo porque existe y funciona.**
4. **El trabajo del dueño es explícito y secuencial:** construir la **versión mejorada de cada módulo** (con el mismo método de este Plan Maestro) e **irlos integrando UNO A UNO** con el POS nuevo.

**Lo que esto cambia en la forma de trabajar:**

| Antes (supuesto incorrecto) | Ahora (hecho documentado) |
|---|---|
| "Los módulos del ERP aún no existen; el POS se prepara para cuando existan." | "Los módulos existen, pero están parchados; el POS se integra con ellos **por contrato** y cada uno se rehará a su tiempo." |
| "Integrar = esperar a que el módulo nazca." | "Integrar = **acoplar por contrato** al módulo viejo hoy, y **rehacerlo** cuando le toque su fase." |
| "Si el módulo viejo funciona, sirve tal cual." | "Que funcione no significa que sirva: hay que **profesionalizarlo y acoplarlo**." |

**La regla que lo resume:**

> **El ERP viejo es el ORÁCULO, no el MODELO.** Se le consulta qué debe hacer cada módulo (su comportamiento, sus reglas, su UX heredada); **no** se le copia su arquitectura parchada. Cada módulo se rehará con el método del POS, uno a uno, y se acoplará por contrato.

**Corolario operativo (evita dos errores opuestos):**

- **Error A — "hay que inventar el módulo":** falso. El módulo existe; hay que **profesionalizarlo y acoplarlo**, no construirlo de cero.
- **Error B — "el módulo viejo ya sirve":** falso. Su arquitectura es parchada; **no se hereda su implementación, solo su comportamiento** (misma lógica que §6.8: *la integración se hereda, la implementación se reescribe*).

**Aplicación inmediata (F10.4):** el "Contexto diario post-corte" (B-02) es el primer caso de este patrón. El módulo **Estadísticas ya existe** (backend `apps/api/modules/analytics/` + frontend `apps/analytics/` + tabla `daily_contexts` + endpoints `GET/PUT /analytics/context`). Por lo tanto **no se inventa nada**: el POS nuevo **se acopla por contrato** al endpoint que Estadísticas ya expone.

---

## 4. LAS 2 REGLAS DURAS INQUEBRANTABLES

### REGLA DURA 1 — PROHIBIDO tocar, modificar o interrumpir el ERP de R de Rico que corre en este servidor

- El ERP viejo corre en `localhost:5000` (frontend) y `localhost:5001` (API) con PostgreSQL en `5433`.
- El POS nuevo corre en `localhost:5100` (frontend) y `localhost:5101` (API) con su **propia** base de datos.
- **Cero dependencias cruzadas.** Este proyecto corre 100% en paralelo sin estorbar al otro.
- El POS nuevo tiene su propio repo (`NUEVO-POS`), su propio `package.json`, su propio backend.
- Cuando esté listo, **reemplazará** al viejo. Hasta entonces, no lo toca.

### REGLA DURA 2 — VERIFICAR, NO ASUMIR

**Ninguna afirmación sobre el código, el esquema o el estado del sistema se escribe sin haberla verificado contra la fuente real.**

- Un plan que **nombra** una tabla, un modelo, un campo, un contrato o un endpoint **debe verificar que existe** antes de nombrarlo. Si no existe, se declara como trabajo a construir — no se asume.
- Un plan que **afirma** que una función hace algo (p. ej. "hace commit al final") **debe leer la función** y citar la línea. No se infiere por el nombre.
- Un plan que **cuenta** campos, reglas o estados **debe abrir el contrato/registro** y contarlos. No se estima de memoria.
- Cuando la verificación contradice el plan, **manda la verificación**. El plan se corrige; no se ejecuta sobre una suposición.
- **Origen:** autocrítica del plan F7.5 (v3.0 → v3.1). El plan v3.0 nombró `system_settings` (que no existía), afirmó que `crear_ticket` era "misma transacción" (hacía `commit()` temprano), contó 9 campos donde el contrato exige 10, e ignoró el default de `order_status`. Los 4 defectos se detectaron **solo al verificar contra el código real**. La lección: *"Un plan que nombra una tabla debe verificar que la tabla existe."*
- **Cómo se verifica:** con las herramientas de lectura (`read_file`, `search_files`, `list_files`) sobre el repo real, citando archivo y línea. La evidencia de la verificación se registra en la ficha de la fase.

---

## 5. LAS 6 PROHIBICIONES ABSOLUTAS

> Extraídas de **7 meses de operación real + 1 error de construcción del POS nuevo**. **Toda línea de código del POS nuevo las respeta.**

| # | Prohibición | Origen (el caso real) |
|---|---|---|
| **1** | **NO** reintroducir auto-save, timers ni `setInterval` para guardar el carrito. La persistencia es **atómica por ítem** (v6.0). | Race conditions y closures viejos que perdían ítems. |
| **2** | **NO** hacer `clearCart()` sin confirmación HTTP 200 del servidor **Y** verificación post-envío (v6.1). | Incidente **$453** (cuenta fantasma): se limpió el carrito sin confirmar que el ticket existía en la BD. |
| **3** | **NO** leer variables de estado (`cart`, `currentAccountNum`) dentro de callbacks asíncronos — usar siempre `useRef`. | **Ticket #906**: el total pasó de **$124 a $2** por leer estado obsoleto en un closure. |
| **4** | **NO** almacenar candados de terminal en RAM de Python — solo en PostgreSQL (`terminal_locks`). | Los locks en RAM se perdían al reiniciar el proceso; dos cajeros creían tener la misma terminal. |
| **5** | **NO** generar folios en el frontend — solo el backend los genera vía secuencia atómica de PostgreSQL. | Folios duplicados o reciclados al recargar el navegador. |
| **6** | **NO** duplicar módulos que el ERP ya tiene (login, gestión de empleados, perfiles). El POS es un MÓDULO del ERP, no una app suelta. Si el ERP ya lo resuelve, el POS lo recibe como prop — no lo reconstruye. | Se construyó un login propio y el dueño lo rechazó correctamente por **YAGNI**. |

---

## 6. LAS 10 REGLAS ARQUITECTÓNICAS DERIVADAS DE LA BATALLA

| Regla | Origen | Implementación |
|---|---|---|
| **Verificar, no asumir** | Autocrítica F7.5 v3.0→v3.1 (4 defectos) | Antes de nombrar una tabla/campo/contrato/endpoint en un plan, verificar que existe (archivo + línea). Antes de afirmar qué hace una función, leerla. Antes de contar campos/reglas/estados, abrirlos y contarlos. Si la verificación contradice el plan, manda la verificación. |
| **Contrato de resultado discriminado** | Incidente v7.0.3 (cuentas perdidas) | Toda función de persistencia retorna `{ outcome, reason }`. PROHIBIDO asumir "no lanzar excepción" = éxito. |
| **Verificación post-envío** | Incidente **$453** (cuenta fantasma) | Después de HTTP 200, verificar que el ticket existe en la BD. |
| **withRetries centralizado** | Asimetría v7.0.1 | Todas las operaciones usan el mismo patrón: 3 intentos, backoff 1s/2s/3s. |
| **Respuesta ligera** | Optimización v7.0 (rush hour) | Operaciones atómicas devuelven 5 campos escalares, no JOINs completos. |
| **Timestamps UTC** | Hallazgo H2 | `utcnow()` siempre, nunca `datetime.now()`. Store UTC, Display Local. |
| **Primitivos en deps** | Hallazgo H1 | `useEffect` deps = primitivos (`currentUser?.id`), no objetos. |
| **3 estados de terminal** | Incidente v12 (veracidad) | libre / mío / ajeno. NUNCA colapsar a 2 ramas. |
| **sendBeacon al cerrar** | Hallazgo H3 | `beforeunload` libera lock + persiste carrito vía beacon. |
| **Espejo de limpieza** | Incidente v7.0.3 (refs residuales) | Toda rama de salida limpia exactamente los mismos refs que la rama de éxito. |
| **Banner rojo fijo** | Incidente **$453** | Error de red = banner permanente + botón bloqueado, no toast efímero. |

> **Nota:** la tabla lista 11 filas porque "Verificar, no asumir" es a la vez una **Regla Dura** (§4) y la primera **regla de batalla**. Las 10 reglas de batalla propiamente dichas son las 10 restantes.

---

## 7. LOS 16 ESTÁNDARES DE CALIDAD DEL CONSTRUCTOR

> **Origen:** [`PROMPT_DEL_ARQUITECTO_DEL_NUEVO_POS.md`](../06-prompt-del-arquitecto/PROMPT_DEL_ARQUITECTO_DEL_NUEVO_POS.md).
> **Estos son estándares obligatorios, no sugerencias. Cada uno tiene un criterio de verificación.**

| # | Estándar | Qué exige | Cómo se verifica |
|---|---|---|---|
| E-01 | **KISS** | La solución más simple que resuelve el problema | ¿Se puede explicar en una frase? |
| E-02 | **YAGNI** | No construir lo no pedido | ¿Hay código sin requisito que lo respalde? |
| E-03 | **DRY** | Una sola fuente de verdad | ¿El mismo valor se define en 2+ lugares? |
| E-04 | **SOLID** | Responsabilidad única; frontera por contratos | Test de arquitectura: 0 imports a modelos ajenos |
| E-05 | **Fail-fast** | Falla ruidosamente; sin silencios | CI: 0 `try/except pass` en ruta crítica |
| E-06 | **Idempotencia** | Reintentar no duplica efectos | Test: ejecutar 2× produce el mismo estado |
| E-07 | **Trazabilidad** | Toda regla tiene su test | Matriz `regla → test` completa |
| E-08 | **Sin números mágicos** | Todo valor de negocio se declara en config | 0 literales de negocio hardcodeados |
| E-09 | **Dinero decimal** | `Numeric(12,2)`, nunca `Float` | Esquema: 0 columnas de dinero `Float` |
| E-10 | **Tiempo UTC** | UTC en BD, local en pantalla | Esquema: 0 `DateTime` naive |
| E-11 | **Identidad ≠ folio** | UUID global, folio local | Ninguna regla usa folio como identidad |
| E-12 | **Ledger inmutable** | El stock se deriva, no se sobrescribe | Test: el ledger rechaza `UPDATE` directo |
| E-13 | **Seguridad en backend** | El backend valida todo | Ninguna validación vive solo en el frontend |
| E-14 | **Evidencia, no opinión** | Cada puerta se prueba con comando + salida | Cada fase cierra con evidencia |
| E-15 | **Cero código basura** | Sin `console.log()`, placeholders, código muerto | CI: greps automáticos |
| E-16 | **Funciones atómicas** | Máx. 20 líneas por función, máx. 3 niveles de anidamiento | Revisión de código |

**Personalidad del constructor:**

> *"Actúa como un ingeniero senior con 15+ años construyendo POS y ERPs en producción, que ha tenido que mantener código heredado y sabe exactamente cómo se degrada un sistema. Copias el comportamiento del POS viejo, NO su deuda."*

**Lo que NUNCA hará:**

1. Tocar el ERP viejo.
2. Entregar código basura (provisional, placeholders, `console.log`).
3. Escribir `try/except pass` en ruta crítica.
4. Usar `Float` para dinero o `DateTime` naive.
5. Usar el folio como identidad.
6. Leer tablas de otro módulo.
7. Avanzar de fase con pruebas fallando.
8. Decir "ya funciona" sin mostrar evidencia.
9. **Asumir sin verificar** — nombrar una tabla/campo/contrato/endpoint sin comprobar que existe, afirmar qué hace una función sin leerla, o contar campos/reglas/estados sin abrirlos y contarlos (REGLA DURA 2).

---

## 8. EL PRINCIPIO "DE ADENTRO HACIA AFUERA"

> **Origen:** PLAN_DE_CONSTRUCCION, §0.2. Diferencia intencional con DeepSeek.

### 8.1 Cómo lo proponía DeepSeek (por capas horizontales)

DeepSeek diseñó un orden de construcción **por capas**, donde se completa una capa entera del edificio antes de subir a la siguiente:

```
F1. TODAS las tablas de datos (17 tablas)          ← cimiento completo
F2. TODOS los contratos entre módulos (17)         ← frontera completa
F3. TODAS las 81 reglas de negocio con tests       ← comportamiento completo
F4. TODOS los tests guardianes                     ← blindaje completo
F5. TODAS las 26 interfaces                        ← superficie completa
F6. Consolidación central                          ← techo
```

> **Nota (F11.0, 30 Sep 2026):** este diagrama describe el plan **propuesto por DeepSeek** y se conserva como historia. La realidad construida difiere: **31 contratos** (no 17), **95 reglas** (no 81) y **24 interfaces** (no 26 — el "DEFECTO DEL PLANO 26 vs 24" quedó registrado en [`FICHA_F5_SUPERFICIE.md`](../../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F5_SUPERFICIE.md)).

**Ventaja:** garantiza que nunca construyes una pared sin cimiento. Es académicamente puro.

**Desventaja:** hasta completar F5 (la 5ª capa), no tienes NADA usable. No puedes enseñarle una pantalla funcional al dueño del negocio hasta que hayas construido las 17 tablas, los 17 contratos y las 81 reglas. Eso puede tardar semanas sin mostrar progreso visible.

### 8.2 Cómo lo hacemos nosotros (por funciones verticales)

Nuestro plan construye **rebanadas verticales** — cada fase entrega una función completa de piso a techo (dato + contrato + regla + test + interfaz):

```
Fase 1: Terminal Selector   → tabla terminal_locks + endpoint + hook + UI
Fase 2: Sesión               → tabla sessions + servicio + hook + UI
Fase 3: POS Completo         → tablas tickets/items + 9 hooks + 12 endpoints + UI
Fase 4: Gestor de Caja       → tablas cash + servicio + UI
...
```

**Ventaja:** al terminar la Fase 1, ya puedes abrir el POS y ver las terminales. Al terminar la Fase 3, ya puedes cobrar. El dueño ve progreso real en cada fase, puede probar, puede opinar.

**Desventaja:** podrías construir una tabla sin respetar los estándares (UUID, UTC, Numeric). Por eso existen la §12 y la §13 como guardias.

### 8.3 La garantía: el orden interno de cada rebanada

Dentro de cada fase, respetamos el orden de DeepSeek:

```
Cada fase internamente sigue:
  1. Endpoint (dato + contrato)    ← cimiento de la rebanada
  2. Hook (regla de negocio)       ← comportamiento de la rebanada
  3. Test (guardián)               ← blindaje de la rebanada
  4. Componente (interfaz)         ← superficie de la rebanada
```

> **En resumen:** DeepSeek proponía construir TODOS los cimientos, luego TODAS las paredes, luego TODOS los techos. Nosotros construimos una habitación completa a la vez (cimiento + pared + techo), pero cada habitación respeta el mismo orden interno. El resultado final es el mismo edificio — la diferencia es que el nuestro se puede ir probando habitación por habitación.

---

## 9. LAS 5 LECCIONES DE INTEGRACIÓN (F4.5 → F10.6)

> Estas lecciones son el corazón de la documentación final. Cada una nació de un defecto real que **pasó todas las compuertas** y aun así llegó a producción. Son la razón por la que existe este tomo.

### 9.1 LA LECCIÓN DE LA MICRO-FASE F4.5 — el paso de INTEGRACIÓN también es una compuerta

> **Origen:** Micro-fase correctiva F4.5 "Montaje del Gestor de Caja" (30 Sep 2026).
> **Ficha:** [`FICHA_F4_5_MONTAJE_CAJA.md`](../../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F4_5_MONTAJE_CAJA.md).

**El hecho:** el `GestorDeCaja.jsx` (467 líneas) se construyó, se probó y pasó su compuerta en verde. El backend de caja (`cash.py`, 6 endpoints) y el servicio (`cashService.js`) también. **Pero nadie lo montó en la pantalla.** El componente quedó **huérfano**: existía, estaba probado, y **ningún usuario podía llegar a él**.

**El impacto:** RN-49 exige una sesión de caja `OPEN` para cobrar. Sin punto de entrada para abrir el turno, **el POS no podía cobrar**. Un sistema con todas sus piezas verdes era, sin embargo, funcionalmente inoperante.

**La causa raíz:** cada sub-fase pasó su compuerta **en aislamiento**. La compuerta de F4.3 verificaba "el componente funciona"; la de F4.4 verificaba "el corte se genera". **Ninguna compuerta verificaba "el usuario puede llegar al componente".** El paso de INTEGRACIÓN no estaba declarado como compuerta.

**La lección (regla nueva):**

> *"el componente existe y pasa su test" ≠ "el usuario puede llegar a él".*
>
> Toda sub-fase que construye un **componente de superficie** (una pantalla, un panel, un overlay) debe declarar explícitamente **su punto de entrada** — el botón, el gesto o la ruta que lo hace alcanzable — y **probarlo**. La integración no es un detalle de cierre: es una compuerta más.

**Cómo se previene en adelante:**

1. En el plan de cada fase, la sub-fase de superficie debe responder por escrito: *"¿desde dónde llega el usuario a este componente?"*.
2. El test de integración debe montar la **pantalla real** (no el componente aislado) y verificar que el punto de entrada existe y abre el componente.
3. Al cerrar una fase, revisar que **ningún componente construido quede sin punto de entrada** (grep de imports vs. renders).

**Corolario operativo:** una guarda nueva en la ruta crítica (como la de F4.5.3, que exige turno abierto para cobrar) **obliga** a revisar todos los tests que tocan esa ruta. En F4.5 esto rompió dos tests existentes (`f8_6` y `f3_cierre`) que cobraban sin declarar el contrato de caja; se corrigieron sembrando el turno `OPEN`. La guarda era correcta; lo que faltaba era que los tests declararan el contrato que la pantalla ahora consume.

### 9.2 LA LECCIÓN DE LA FASE 10 — la COMPLETITUD del conjunto también es una compuerta

> **Origen:** Fase 10 "Auditoría de Paridad" (30 Sep 2026).
> **Ficha:** [`FICHA_F10_PARIDAD.md`](../../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F10_PARIDAD.md).

La lección de §9.1 tenía una **segunda mitad**:

> *"el componente existe y pasa su test" ≠ "el conjunto está completo".*

**El hecho:** el dueño, revisando el Gestor de Terminales, notó que el botón **"Copiar URL"** del viejo POS **no existía** en el nuevo. Se verificó el código: **ni el botón ni la lógica** estaban. No era un componente huérfano — era una **pieza que nunca se construyó**, porque ninguna sub-fase la había pedido explícitamente.

**La causa raíz — la misma clase de defecto, tercera instancia:**

| # | Instancia | Fase | Qué pasó |
|---|-----------|------|----------|
| 1 | `GestorDeCaja` huérfano | F4.5 | El componente existía y pasaba su test, pero **nadie podía llegar a él**. |
| 2 | `payment_details` no expuesto | F9.1.4a | El dato se persistía, pero **no se exponía** en la salida. |
| 3 | "Copiar URL" omitido | F10 | La pieza existía en el viejo POS, pero **nunca se construyó** en el nuevo. |

Las tres comparten la raíz: **"de adentro hacia afuera" verifica cada pieza en aislamiento, pero no verifica la COMPLETITUD del conjunto contra el viejo POS.** Cada sub-fase pasa su compuerta; ninguna compuerta compara el inventario completo.

**La lección (regla nueva):**

> *"el componente existe y pasa su test" ≠ "el conjunto está completo".*
>
> Antes de declarar un módulo terminado, se debe ejecutar una **auditoría de paridad** contra el sistema de referencia: inventariar **todo** lo que el viejo hacía, clasificar cada pieza (PORTADA / OMITIDA / HUÉRFANA / DESCARTADA) y **cerrar cada OMITIDA o justificarla por escrito**. La completitud no se asume: se audita.

**Cómo se previene en adelante:**

1. Al cerrar un módulo, ejecutar una **auditoría de paridad** contra el sistema de referencia (el viejo POS, el ERP, el contrato). No basta con que "todo lo construido pase": hay que verificar que **nada de lo esperado falte**.
2. Cada pieza del inventario se clasifica en una de las 4 categorías, y cada **OMITIDA** se cierra o se documenta como **DESCARTADA** con su razón.
3. La auditoría se ejecuta **antes** de la documentación final: no se documenta un sistema sin verificar que está completo.

**Corolario:** la documentación final (§11) es el **último** entregable, no el primero. Se escribe sobre un sistema **verificado completo**, no sobre uno que "parece" completo porque todas sus piezas verdes pasaron sus tests.

### 9.3 LA LECCIÓN DE LA FASE 10.4 — el inventario de componentes no ve las integraciones

> **Origen:** Fase 10.4 "Contexto diario post-corte" (30 Sep 2026).
> **Ficha:** [`FICHA_F10_4_CONTEXTO_DIARIO.md`](../../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F10_4_CONTEXTO_DIARIO.md).

**El hecho:** la auditoría de paridad de la Fase 10 inventarió **componentes** (pantallas, hooks, servicios, endpoints) y los clasificó uno por uno. Todo lo que aparecía en el inventario estaba portado. Sin embargo, el POS viejo **escribía un "contexto diario"** —un resumen del día— en la tabla `daily_contexts` del módulo de **Estadísticas**. Esa escritura **no era un componente del POS**: era una **integración** POS → Estadísticas. Como no era un componente, **no apareció en el inventario** y por tanto **no se detectó su ausencia**.

**La causa raíz:** el inventario de paridad se construyó **componente por componente** (una lista de piezas del POS). Una **integración** no es una pieza del POS: es un **borde** entre dos módulos. Los bordes no viven en el inventario de piezas de ninguno de los dos módulos, así que **ningún inventario por componentes los ve**. El defecto no fue "olvidamos portar X": fue "nuestro método de auditoría **no tenía una categoría** para X".

**La lección (regla nueva):**

> *"el inventario de componentes" ≠ "el inventario de integraciones".*
>
> La auditoría de paridad debe inventariar **dos capas**: (1) los **componentes** de cada módulo, y (2) las **integraciones** entre módulos (quién escribe en las tablas de quién, quién llama a la operación de quién). Una integración que existe en el sistema de referencia y no en el nuevo es una **OMITIDA**, aunque no sea "un componente".

**Cómo se previene en adelante:**

1. La auditoría de paridad se ejecuta **dos veces**: una sobre el inventario de componentes y otra sobre el inventario de **bordes** (integraciones, contratos, escrituras cruzadas).
2. Cada borde del sistema de referencia se clasifica igual que un componente (PORTADO / OMITIDO / HUÉRFANO / DESCARTADO).
3. El contexto diario se portó como **contrato** (contrato 23, `analytics.registrar_contexto_diario`), no como componente: el POS **llama** a Estadísticas, no escribe su tabla.

**Corolario:** un sistema puede tener el **100% de sus componentes** portados y seguir **incompleto**, porque le faltan los **bordes**. La completitud se mide sobre componentes **y** sobre integraciones.

### 9.4 LA LECCIÓN DE LA FASE 10.5 — el inventario de componentes no ve los FLUJOS DE DATOS

> **Origen:** Fase 10.5 "Paridad de datos de caja" (30 Sep 2026).
> **Ficha:** [`FICHA_F10_5_PARIDAD_DE_DATOS_DE_CAJA.md`](../../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F10_5_PARIDAD_DE_DATOS_DE_CAJA.md).

**El hecho — el "trasplante de corazón":** el corte de caja del POS viejo devolvía un `CashSummaryResponse` con **8 campos**. El nuevo POS devolvía un `ResumenTurnoSalida` (contrato 12) con **2 campos**. **Faltaban 7 campos.** Además, el nombre del usuario (`usuario_nombre`) **no viajaba** en la respuesta del corte (contrato 10): el dato existía en la base, pero **no se exponía**. El componente "corte de caja" existía en ambos sistemas; **el flujo de datos que lo atravesaba era distinto**.

**La causa raíz:** el inventario de paridad comparó **componentes** ("¿existe el corte de caja? Sí") pero **no comparó los datos que cada componente transporta**. Un componente puede existir en ambos lados y, sin embargo, **transportar un subconjunto de los datos**. El inventario veía el **órgano** (el corte) pero no la **sangre** (los campos que fluyen por él).

**La lección (regla nueva):**

> *"el componente existe en ambos lados" ≠ "el flujo de datos es equivalente".*
>
> La auditoría de paridad debe comparar, para cada componente compartido, **el conjunto de campos de entrada y de salida** (el contrato de datos), no solo la existencia del componente. Un campo que viaja en el viejo y no en el nuevo es una **OMITIDA de datos**.

**Cómo se previene en adelante:**

1. Para cada componente compartido, comparar **campo por campo** la entrada y la salida (el esquema Pydantic / el contrato).
2. Todo campo presente en el viejo y ausente en el nuevo se documenta como **OMITIDA de datos** y se cierra o se justifica.
3. El contrato de datos se declara **explícitamente** (esquema Pydantic), de modo que la comparación sea mecánica y no dependa de leer el código a ojo.

**Corolario:** la paridad de un componente no es binaria (existe / no existe): es **cuantitativa** (¿cuántos de sus campos viajan?). Un componente "portado" con la mitad de sus campos es un componente **a medias**.

### 9.5 LA LECCIÓN DE LA FASE 10.6 — el inventario de componentes no ve la PARIDAD DE OPERACIÓN

> **Origen:** Fase 10.6 "Paridad de operación de caja" (30 Sep 2026).
> **Ficha:** [`FICHA_F10_6_PARIDAD_DE_OPERACION_DE_CAJA.md`](../../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F10_6_PARIDAD_DE_OPERACION_DE_CAJA.md).

**El hecho — cuatro sub-defectos de la misma clase:**

| # | Sub-fase | Qué faltaba | Naturaleza |
|---|----------|-------------|------------|
| 1 | F10.6.1 | **Teclado táctil** ausente | Una **forma de operar** (cómo se captura el monto) |
| 2 | F10.6.2 | **Eliminar movimiento** ausente (contrato 29) | Una **operación** (borrar un movimiento de caja) |
| 3 | F10.6.3 | **Hora y concepto** ausentes en la lista | Un **dato de operación** (cuándo y por qué) |
| 4 | F10.6.4 | **Impresión del corte** no cableada | Una **acción de operación** (imprimir el corte) |

Los cuatro son la **misma clase de defecto**: el componente existía (o existía su equivalente), pero **la forma en que el cajero lo opera** —el teclado, el botón de borrar, la hora, el botón de imprimir— **no se había portado**. El inventario veía el componente; **no veía la operación**.

**La causa raíz:** el inventario de paridad comparó **qué existe**, no **cómo se opera**. La operación (el gesto, el botón, el flujo de la mano del cajero) **no es un componente**: es un **modo de uso** que vive dentro de la implementación. Al reescribir la implementación, **se perdió la operación**.

**La lección (regla nueva) — y el refinamiento de §6.8:**

> *"el componente existe" ≠ "el cajero puede operarlo como en el viejo".*
>
> La auditoría de paridad debe comparar, para cada componente compartido, **la operación**: qué gestos, botones, atajos y flujos tiene el viejo y cuáles tiene el nuevo. Una operación presente en el viejo y ausente en el nuevo es una **OMITIDA de operación**.

**Refinamiento de §6.8 ("la integración se hereda, la implementación se reescribe"):** la frase era **incompleta**. No basta con heredar la **integración** (el borde) y reescribir la **implementación** (el código): hay **decisiones operativas** que viven **dentro** de la implementación (el teclado táctil, el botón de borrar, la hora en la lista) y que **se pierden al reescribir**. La regla correcta es:

> *"la INTEGRACIÓN se hereda, la OPERACIÓN se hereda, la IMPLEMENTACIÓN se reescribe."*

**Cómo se previene en adelante:**

1. Para cada componente compartido, comparar **la operación**: gestos, botones, atajos, flujos de la mano del cajero.
2. Toda operación presente en el viejo y ausente en el nuevo se documenta como **OMITIDA de operación** y se cierra o se justifica.
3. Al reescribir una implementación, **inventariar primero las decisiones operativas** que contiene, para no perderlas en el trasplante.

**Corolario — las tres capas de la paridad:** la paridad de un módulo se mide en **tres capas**, y las tres son compuertas:

| Capa | Pregunta | Lección |
|------|----------|---------|
| **Componentes** | ¿existe la pieza? | §9.2 (F10) |
| **Datos** | ¿viajan todos los campos? | §9.4 (F10.5) |
| **Operación** | ¿el usuario la opera igual? | §9.5 (F10.6) |

Y por encima de las tres, los **bordes** (§9.3, F10.4): ¿existen las integraciones entre módulos? Un módulo está completo solo cuando las **cuatro** capas están verificadas.

---

## 10. LA FRONTERA POR CONTRATOS (A-02 / O-23)

> **Fuente:** [`PLAN_MAESTRO_DEFINITIVO_POS.md`](../PLAN_MAESTRO_DEFINITIVO_POS.md:873) §10.2.

**La regla dura A-02:** *el POS **no importa** los modelos de otros módulos.* Ningún archivo del POS hace `from models.estadisticas import ...` ni `from models.crm import ...`. El POS **no lee** las tablas de otro módulo, y **no escribe** en ellas.

**Cómo se resuelve cada dependencia:** por **contrato explícito**. Cada vez que el POS necesita algo de otro módulo, se declara un **contrato** en [`contracts/registry.py`](../../NUEVO-POS/apps/api/contracts/registry.py:1) con:

- **`nombre`** — el identificador del contrato (p. ej. `crm.beneficios_para_ticket`).
- **`proveedor`** — el módulo que **responde** (CRM, Estadísticas, Notificaciones…).
- **`consumidor`** — el módulo que **pregunta** (POS).
- **`operacion`** — la operación que el proveedor expone (no una tabla: una **operación**).
- **`firma`** — la forma de la entrada y la salida.

**Los 10 acoplamientos reemplazados:** el POS viejo tenía **10 acoplamientos directos** (AC-01 a AC-10) con otros módulos —importaba sus modelos, leía sus tablas, escribía en ellas—. Cada uno se reemplazó por un **contrato**. La tabla de equivalencia (acoplamiento → contrato) es la prueba de que **ningún acoplamiento directo sobrevive**.

**El test de arquitectura:** existe un test que **falla si el POS importa un modelo ajeno**. No es una convención: es una **compuerta de CI**. Si un desarrollador (o una IA) escribe `from models.<otro_modulo> import ...` dentro del POS, el test se pone rojo y el commit no pasa.

**Por qué importa:** un acoplamiento directo **congela** la tabla del otro módulo. Si el POS lee `daily_contexts` directamente, Estadísticas **no puede cambiar** esa tabla sin romper el POS. El contrato **desacopla**: Estadísticas puede cambiar su tabla mientras **mantenga la operación** del contrato. La frontera por contratos es lo que permite que el ERP se rehaga **módulo por módulo** (§8.2) sin que un módulo rompa a otro.

---

## 11. AUTONOMÍA VS. OBSERVABILIDAD

> **Fuente:** [`DOCUMENTACION_MODULO_MONITOREO_DE_RED.md`](../../ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_MODULO_MONITOREO_DE_RED.md:1) §11.3.

El POS nuevo persigue **dos propiedades distintas** que suelen confundirse. Son **complementarias**, no la misma cosa:

| Propiedad | Definición | Módulo que la provee |
|-----------|------------|----------------------|
| **Autonomía** | *"Puedo seguir operando aunque el otro módulo esté caído."* | El POS mismo (no depende de CRM/Notificaciones para cobrar) |
| **Observabilidad** | *"Puedo probar qué pasó aunque el otro módulo esté caído."* | Auditoría y Control (F13) |

**Autonomía:** el POS **cobra** aunque CRM esté caído (RN-87), aunque Notificaciones esté caído (RN-88). El cobro **no depende** de que el CRM responda ni de que el ticket se envíe. Los fallos de los módulos vecinos **no tumban** el POS.

**Observabilidad:** el POS **registra** cada escritura en `pos_audit_log` (F13), de modo que **aunque el módulo de Auditoría y Control esté caído**, el POS **sigue escribiendo su log** y **puede probar** qué pasó. La observabilidad **no depende** de que el consumidor del log esté vivo.

**Las dos observabilidades (complementarias, NO fusionadas):**

- **Monitoreo de Red** — observabilidad de **infraestructura** (¿el servidor responde? ¿la red está viva? ¿cuál es la latencia?).
- **Auditoría y Control** — observabilidad de **operación de negocio** (¿quién cobró? ¿qué se escribió? ¿cuándo?).

Son **dos módulos distintos** que responden **dos preguntas distintas**. Fusionarlos sería un error: la infraestructura y el negocio tienen ciclos de vida, dueños y consumidores diferentes.

**La consecuencia de diseño:** el POS **nunca** espera a que el observador esté vivo para operar. Escribe su log **en su propia transacción** (guardián `LogDeAuditoria`, [`guards/audit.py`](../../NUEVO-POS/apps/api/guards/audit.py:1)) y **confirma el asiento al cerrar sin excepción**; si la escritura falla, **hace rollback** (no se audita lo que no ocurrió). La observabilidad es **local al POS** y **no bloquea** la operación.

---

## 12. LOS ESTÁNDARES DE BASE DE DATOS (C-01 A C-04)

> **Fuente:** [`PLAN_MAESTRO_DEFINITIVO_POS.md`](../PLAN_MAESTRO_DEFINITIVO_POS.md:881) §10.3.

Cuatro estándares **obligatorios** en toda tabla del POS nuevo. No son sugerencias: son **compuertas**.

| # | Estándar | Regla | Por qué |
|---|----------|-------|---------|
| **C-01** | **PK = UUID** | Toda clave primaria es un `UUID`, no un entero autoincremental. | Permite generar ids **sin consultar la base** (idempotencia, reintentos, multi-sucursal futuro). Un entero autoincremental **filtra** el volumen de negocio y **colisiona** al consolidar. |
| **C-02** | **`DateTime(timezone=True)` en UTC** | Todo timestamp es **consciente de zona** y se almacena en **UTC**. | "Store UTC, Display Local". Un `DateTime()` naive **pierde** la zona y produce bugs de horario (ver el bug de la vista general). La conversión a hora local es **de presentación**, no de almacenamiento. |
| **C-03** | **`Numeric(12,2)` para dinero** | Todo monto es `Numeric(12,2)`, **nunca** `Float`. | Un `Float` **no representa** decimales exactos: `0.1 + 0.2 ≠ 0.3`. En dinero, eso es **centavos perdidos** que no cuadran. `Numeric` es exacto. |
| **C-04** | **Columna `version` para bloqueo optimista** | Toda entidad mutable tiene una columna `version` que se incrementa en cada escritura. | Permite **detectar escrituras concurrentes** (RN-25/RN-26): si dos terminales editan el mismo ticket, la segunda recibe **409** y no pisa a la primera. Sin `version`, la última escritura **gana en silencio** y se pierde trabajo. |

**Cómo se verifica:** los estándares C-01 a C-04 se verifican con **greps de CI** (§13) y con **tests de esquema**. Un modelo con `Float` en un campo de dinero, o con `DateTime()` naive, **rompe la compuerta**.

**La utilidad central:** [`core/timestamps.py`](../../NUEVO-POS/apps/api/core/timestamps.py:1) expone `utcnow()` —la **única** fuente de tiempo del POS— para que ningún módulo llame a `datetime.now()` directamente (que sería naive y local). C-02 se cumple **por construcción**: todos usan `utcnow()`.

---

## 13. LOS 5 GREPS DE CI

> **Fuente:** [`PLAN_MAESTRO_DEFINITIVO_POS.md`](../PLAN_MAESTRO_DEFINITIVO_POS.md:892) §10.4.

Desde el **día 1**, el CI ejecuta **5 greps** que **fallan el build** si encuentran un patrón prohibido. No son linters de estilo: son **guardas de arquitectura**.

| # | Patrón prohibido | Qué detecta | Por qué es una compuerta |
|---|------------------|-------------|--------------------------|
| **1** | `except.*pass` | Un `except` que **silencia** la excepción con `pass`. | Silenciar un error en la ruta crítica **oculta** un fallo (E-05). Un error silenciado es un error que **nadie verá** hasta que sea un desastre. |
| **2** | `console.log` | Un `console.log` olvidado en el frontend. | El logging de producción **no** es `console.log`: contamina la consola, filtra datos y no es observable. |
| **3** | `TODO` sin formato | Un `TODO` sin dueño ni fecha. | Un `TODO` huérfano es **deuda invisible**. El formato obliga a declarar **quién** y **cuándo**. |
| **4** | `Float` en modelos de dinero | Un campo de dinero tipado como `Float`. | Viola C-03: **centavos perdidos**. |
| **5** | `DateTime()` naive | Un timestamp sin `timezone=True`. | Viola C-02: **bugs de horario**. |

**Cómo se ejecuta:** los greps viven en [`scripts/guards.mjs`](../../NUEVO-POS/scripts/guards.mjs:1) y corren en el CI. El resultado esperado es **7/7 guardas verdes** (las 5 anteriores más 2 guardas adicionales de arquitectura).

**Por qué greps y no solo tests:** un test verifica **comportamiento**; un grep verifica **forma**. Hay defectos que **no se manifiestan** en un test (un `console.log` no rompe ningún test) pero que **degradan** el sistema. Los greps cierran esa brecha: verifican que el **código fuente** no contiene el patrón, aunque el comportamiento pase.

---

## 14. LA MATRIZ DE TRAZABILIDAD REGLA → TEST

> **Fuente:** [`PLAN_MAESTRO_DEFINITIVO_POS.md`](../PLAN_MAESTRO_DEFINITIVO_POS.md:904) §10.5.

**El principio A-01:** *las reglas de negocio se portan **con** su test.* Una regla sin test es una regla que **se puede perder** sin que nadie lo note.

**La matriz:** cada fase produce una **matriz de trazabilidad** que mapea **cada regla** a **su test**. La matriz es un artefacto **verificable**: no es una promesa, es una tabla que el CI puede comprobar.

**Las 95 reglas:** el POS nuevo tiene **95 reglas de negocio** (RN-01 a RN-95), declaradas en [`rules/registry.py`](../../NUEVO-POS/apps/api/rules/registry.py:1). Cada una tiene:

- **`numero`** — el identificador (RN-01…RN-95).
- **`enunciado`** — la regla en lenguaje natural.
- **`categoria`** — una de las **17 categorías**.
- **`test`** — el nombre del test que la verifica.

**La compuerta:** el test [`test_f3_comportamiento.py`](../../NUEVO-POS/apps/api/tests/test_f3_comportamiento.py:1) verifica que:

1. La matriz tiene **95 reglas**.
2. Los números van de **RN-01 a RN-95**.
3. **Ninguna regla está sin test.**
4. Cada regla tiene **enunciado**.
5. Las **17 categorías** están presentes.

**Las reglas sin test se marcan como "riesgo de pérdida":** si una regla no tiene test, se documenta explícitamente como **riesgo de pérdida** —una regla que puede desaparecer en una refactorización sin que el CI lo detecte—. El CI **falla** si una regla **crítica** no tiene guardián.

**Por qué importa:** la trazabilidad regla → test es lo que hace que las 95 reglas sean **reales** y no **aspiracionales**. Sin la matriz, las reglas serían un documento que se desactualiza; con la matriz, son un **contrato verificado** en cada commit.

---

## 15. LOS DOS REPOSITORIOS DEL PROYECTO

> **Fuente:** [`PLAN_MAESTRO_DEFINITIVO_POS.md`](../PLAN_MAESTRO_DEFINITIVO_POS.md:30) §"Repositorios del proyecto".

El proyecto vive en **dos repositorios separados**, con **responsabilidades distintas**:

| Repositorio | Contenido | Naturaleza |
|-------------|-----------|------------|
| **`NUEVO-POS`** | El **código** del POS: backend (`apps/api`), frontend (`apps/pos`), migraciones, tests, fichas (`docs/05-plan-de-construccion`). | **Artefacto**: lo que se despliega y se ejecuta. |
| **`PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS`** | Los **planos**: el Plan Maestro, los planes de fase, los planes de abordaje, la documentación final (estos 7 tomos). | **Diseño**: lo que se lee para entender y decidir. |

**Por qué separados:**

1. **Ciclos de vida distintos.** El código cambia con cada commit; los planos cambian con cada decisión arquitectónica. Mezclarlos haría que el historial de git fuera ilegible.
2. **Consumidores distintos.** El código lo consume el **runtime**; los planos los consume una **IA o un arquitecto** que necesita contexto.
3. **La documentación final (estos tomos) vive en el repo de planos**, no en el de código: es **diseño**, no artefacto.

**La regla de oro:** el código **referencia** los planos (por ruta relativa `../PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS/...`), pero **no los duplica**. Un plano vive en **un solo lugar**: el repo de planos.

**La estructura de la documentación final:**

```
PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS/
└── 08-documentacion-final/
    ├── README.md                                  ← índice maestro
    ├── TOMO_I_VISION_Y_ARQUITECTURA.md            ← este documento
    ├── TOMO_II_LAS_95_REGLAS_DE_NEGOCIO.md
    ├── TOMO_III_CEMENTERIO_DE_BUGS_Y_CICATRICES.md
    ├── TOMO_IV_CONTRATOS_Y_FRONTERAS.md
    ├── TOMO_V_SUPERFICIE_E_INTERFACES.md
    ├── TOMO_VI_ACTA_DE_OBRA.md
    └── TOMO_VII_GUIA_PARA_LA_PROXIMA_IA.md
```

---

## 16. EL RESULTADO ESPERADO

> **Fuente:** [`PLAN_MAESTRO_DEFINITIVO_POS.md`](../PLAN_MAESTRO_DEFINITIVO_POS.md:838) §9.

Cuando el POS nuevo está completo y corriendo en `localhost:5100/`, se observan **12 cosas**:

1. **Selector de Terminales** — la landing page: se elige la terminal antes de operar.
2. **Sesión y control de acceso** — resuelta por el ERP (no por el POS).
3. **POS completo** — catálogo, carrito, cobro, con las reglas de negocio aplicadas.
4. **Gestor de Caja** — turnos, movimientos, corte, arqueo, impresión del corte.
5. **Pizarrón de Cuentas Abiertas** — cuentas en paralelo, recuperables por folio.
6. **Impresión + PDF de Catálogo** — el ticket y el catálogo imprimibles.
7. **Voz + Visión IA + Selector de Temas** — tres modos de topología de IA, temas conmutables.
8. **Integración con CRM y Notificaciones** — beneficios al cobrar, ticket por WhatsApp/Email (vía outbox).
9. **UX rescatada del viejo POS** — la operación heredada (teclado táctil, botones, flujos).
10. **Pagos mixtos** — efectivo + tarjeta + transferencia en un mismo ticket.
11. **Auditoría y Control** — cada escritura registrada en `pos_audit_log`, consultable por terminal y rango.
12. **Observabilidad** — el indicador de red con latencia y semáforo de 3 estados.

**Lo que NO se ve pero sostiene todo:** los **31 contratos**, las **24 interfaces**, las **95 reglas**, los **16 estándares**, las **5 guardas de CI**, los **2 repositorios** y las **4 capas de paridad** (componentes, datos, operación, bordes). El resultado visible es la **punta del iceberg**; la arquitectura es lo que **no se ve** y lo que hace que el POS **no se rompa**.

**La prueba final:** una IA sin contexto, leyendo **solo estos 7 tomos**, debe poder **entender** el POS, **extenderlo** sin violar sus reglas, y **no repetir** ninguno de los errores documentados en el Tomo III. Si lo logra, la documentación cumplió su propósito.

---

> **Fin del Tomo I.** Continúa en el **Tomo II — Las 95 reglas de negocio**.