# 🧱 PLAN DE ABORDAJE — FASE 3 POR PARTES

> **Fecha:** 29 Sep 2026
> **Versión:** 1.2 (desambiguación del principio rector)
> **Autor:** Antigravity + Víctor (dueño de R de Rico)
> **Estado:** PROPUESTO — pendiente de aprobación del dueño antes de escribir código
> **Documento padre:** [`PLAN_MAESTRO_DEFINITIVO_POS.md`](../PLAN_MAESTRO_DEFINITIVO_POS.md:1) §7 (Fase 3)
> **Contrato de comportamiento:** [`PROMPT_DEL_ARQUITECTO_DEL_NUEVO_POS.md`](../06-prompt-del-arquitecto/PROMPT_DEL_ARQUITECTO_DEL_NUEVO_POS.md:1)

> [!WARNING]
> **v1.2 — Esta versión corrige 6 defectos encontrados en la autocrítica de la v1.0/v1.1.**
> Ver **§10 AUTOCRÍTICA** al final. Los defectos son de fondo, no de forma:
> (D-1) colisión de numeración de fases, (D-2) endpoints sin contrato declarado,
> (D-3) puertas con un runner de tests que no existe, (D-4) runner equivocado en el backend,
> (D-5) alcance de F3-Maestro mal delimitado, (D-6) **doble significado de "de adentro hacia afuera"**.
> **No ejecutar la v1.0 ni la v1.1.** Usar esta v1.2.

> [!CAUTION]
> **LECTURA OBLIGATORIA ANTES DE CUALQUIER OTRA COSA: §3 (principio rector).**
> El término "de adentro hacia afuera" tiene **dos significados** en este proyecto.
> El vigente es **rebanadas verticales** (Plan Maestro §10.6). El descartado es
> **capas horizontales** (documento discontinuado). Confundirlos invierte el orden de las
> sub-fases. §3 lo desambigua de forma permanente.

---

## 0. PROPÓSITO DE ESTE DOCUMENTO

La Fase 3 ("POS Completo — el corazón") es la más pesada de las 8 fases: el Plan Maestro §7 le asigna **12 archivos** y concentra **7 de las 10 reglas arquitectónicas derivadas de la batalla** (§5 del Plan Maestro). Ejecutarla de un solo golpe es imprudente.

Este documento:

1. **Registra el diagnóstico** del estado real de `NUEVO-POS` al 29 Sep 2026 (Fases 1 y 2 construidas).
2. **Justifica** por qué la Fase 3 se parte en 5 sub-fases (3.0 prerrequisito + 3.1–3.4).
3. **Define** cada sub-fase con su puerta verificable y su ficha de evidencia.
4. **Deja constancia** para poder retomar el trabajo si se interrumpe (corte de luz, cambio de sesión, etc.).
5. **Se autocritica** contra la evidencia del repositorio (§10) y corrige los defectos hallados.

> [!IMPORTANT]
> **Este documento es un artefacto de diseño.** No contiene código de producción. El código va en `NUEVO-POS`; los planos van en `PLANOS-ARQUITECTONICOS` (Plan Maestro §1).

---

## 1. DIAGNÓSTICO — ESTADO REAL DE `NUEVO-POS` AL 29 SEP 2026

> [!IMPORTANT]
> **Aclaración de numeración (defecto D-1 de la v1.0).**
> En este repositorio conviven **dos numeraciones distintas** de "fase":
>
> | Numeración | Documento | Fases | Estado |
> |---|---|---|---|
> | **Plan de Construcción** | `docs/05-plan-de-construccion/` | F1 Cimiento · F2 Frontera · **F3 Comportamiento** · F4 Guardianes · F5 Superficie · F6 Consolidación | **F3 CERRADA** (ver [`FICHA_F3_COMPORTAMIENTO.md`](../../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F3_COMPORTAMIENTO.md:1)) |
> | **Plan Maestro** | [`PLAN_MAESTRO_DEFINITIVO_POS.md`](../PLAN_MAESTRO_DEFINITIVO_POS.md:1) §7 | F1 Terminales · F2 Sesión · **F3 POS Completo** · F4 Caja · F5 Pizarrón · F6 Impresión · F7 IA · F8 CRM | F1/F2 hechas; **F3 POS Completo PENDIENTE** |
>
> **Este documento trata la "Fase 3" del PLAN MAESTRO (POS Completo).**
> La "Fase 3" del Plan de Construcción (Comportamiento) **ya está cerrada** y es un
> **cimiento** sobre el que se apoya este trabajo, no un trabajo pendiente.
> Para evitar ambigüedad, en adelante se escribe **"F3-Maestro"**.

### 1.1 Backend (Fases 1 y 2 — COMPLETAS)

| Artefacto | Ruta | Estado |
|---|---|---|
| Modelos transaccionales | `apps/api/models/pos.py` | ✅ `TerminalSession`, `Ticket`, `TicketItem`, `TerminalLock` |
| Las 81 reglas de negocio | `apps/api/rules/registry.py` | ✅ RN-01 … RN-81 implementadas como funciones puras |
| Los 17 contratos | `apps/api/contracts/registry.py` | ✅ Declarados con firma completa |
| Router del POS | `apps/api/routers/pos.py` | ⚠️ **Solo 3 endpoints** |
| Matriz regla → test | `apps/api/tests/test_f3_comportamiento.py` | ✅ 81/81 + 5 cicatrices probadas |
| **Runner de tests API** | `apps/api/pytest.ini` + Docker | ✅ **pytest real** (`docker compose exec api pytest`) |
| **Runner de tests frontend** | `scripts/test.mjs` | ❌ **STUB F0 — siempre exit 0, NO corre tests** |
| **Vitest en el frontend** | `apps/pos/package.json` | ❌ **NO declarado** (no hay script `test`) |
| **CI** | `.github/workflows/ci.yml` | ⚠️ Corre `npm run test` = el stub → verde vacío |

**Endpoints existentes hoy (solo 3):**

| Método | Ruta | Contrato | Reglas aplicadas |
|---|---|---|---|
| `GET` | `/pos/session-active` | #9 `caja.sesion_activa` | RN-24 |
| `POST` | `/pos/tickets` | #3 `pos.crear_ticket` | RN-10, RN-14, RN-16, RN-18, RN-19, RN-20, RN-21, RN-22, RN-24 |
| `POST` | `/pos/tickets/{id}/pay` | #5 `pos.cobrar_ticket` | RN-14, RN-23, RN-25, RN-27 |

### 1.2 Frontend (esqueleto de Fase 1/2)

| Artefacto | Ruta | Estado |
|---|---|---|
| POS raíz | `apps/pos/src/RetailVisionPOS.jsx` | ⚠️ Monolito con `useState` locales |
| Cliente API | `apps/pos/src/api/client.js` | ⚠️ 4 funciones, sin reintentos |
| Componentes | `SalesReceipt.jsx`, `CheckoutScreen.jsx`, `ProductGrid.jsx`, `CategoryBar.jsx` | ⚠️ Versiones mínimas |
| Hooks | `useModo.js`, `useTerminals.js`, `useAuth.js` | ⚠️ Ninguno del POS real |

### 1.3 Conclusión del diagnóstico

> **El cimiento (datos + contratos + reglas) YA ESTÁ COMPLETO.**
> Lo que falta en F3-Maestro es la **rebanada vertical de comportamiento + interfaz**: los hooks, los endpoints atómicos, la persistencia y la UI real.

### 1.4 Hallazgo crítico de infraestructura de pruebas (defecto D-3)

La v1.0 de este plan escribió puertas como `npm run test -- withRetries`. **Ese comando no prueba nada hoy.** Evidencia:

| Hecho | Evidencia |
|---|---|
| `npm run test` es un **stub** que solo cuenta archivos y **siempre** hace `exit 0` | [`scripts/test.mjs:41-53`](../../NUEVO-POS/scripts/test.mjs:41) |
| El frontend **no tiene** script `test` ni Vitest | [`apps/pos/package.json:7-11`](../../NUEVO-POS/apps/pos/package.json:7) |
| El CI corre ese stub → **verde vacío** | [`.github/workflows/ci.yml:30`](../../NUEVO-POS/.github/workflows/ci.yml:30) |
| El backend **sí** tiene runner real (pytest en Docker) | [`FICHA_F3_COMPORTAMIENTO.md:10`](../../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F3_COMPORTAMIENTO.md:10) |

> **Consecuencia:** cualquier puerta de frontend de este plan exige **primero** construir el runner real (Vitest). Eso es un **prerrequisito P0**, no una tarea de F3-Maestro. Ver §4.0.

---

## 2. POR QUÉ LA FASE 3 ES PESADA (Y POR QUÉ HAY QUE PARTIRLA)

El riesgo de la Fase 3 **no es el volumen de archivos**: es que cada archivo depende de una **decisión de arquitectura que hoy no está tomada**. Los tres huecos críticos:

### Hueco 1 — No existe el modelo de persistencia atómica por ítem (v6.0)

Hoy `crearVenta()` manda el carrito completo en un solo `POST`. El modelo SaaS v6.0 exige que **cada ítem se persista individualmente** con su propio `version`, y que `clearCart()` solo ocurra tras HTTP 200 **+ verificación post-envío**.

> **Implicación:** esto cambia el **contrato del backend**, no solo el frontend. Por eso hay una sub-fase dedicada al backend.

### Hueco 2 — No existe el contrato `{outcome, reason}`

Hoy `confirmarCobro()` asume "no lanzar excepción = éxito". La prohibición #2 del Plan Maestro y la Regla de Oro v7.0.3 exigen **resultado discriminado**.

> **Implicación:** es una utilidad transversal de la que dependen todos los hooks.

### Hueco 3 — No existe `withRetries` ni `sessionReset`

Son utilidades transversales de las que dependen **todos** los hooks. Si se construyen mal, **contaminan las 5 fases restantes** (4 a 8).

> **Implicación:** deben construirse y probarse **primero**, con una puerta barata.

---

## 3. PRINCIPIO RECTOR DE LA PARTICIÓN

> [!CAUTION]
> **DESAMBIGUACIÓN OBLIGATORIA — "de adentro hacia afuera" tiene DOS significados en este proyecto.**
> Este es el punto que causó la confusión que originó la v1.2. Leer antes de continuar.
>
> | Término | Documento | Significado | ¿Vigente? |
> |---|---|---|---|
> | **"De adentro hacia afuera" (capas horizontales)** | [`PLAN_DE_CONSTRUCCION_DEL_NUEVO_POS.md:40`](../PLANES%20DESCONTINUADOS%20DEL%20NUEVO%20POS/PLAN_DE_CONSTRUCCION_DEL_NUEVO_POS.md:40) | Terminar **TODAS** las tablas, luego **TODOS** los contratos, luego **TODAS** las reglas, luego **TODAS** las interfaces. Hasta la 5ª capa no hay nada usable. | ❌ **DESCARTADO** (documento discontinuado) |
> | **"Rebanadas verticales"** | [`PLAN_MAESTRO_DEFINITIVO_POS.md:516`](../PLAN_MAESTRO_DEFINITIVO_POS.md:516) §10.6 | Cada fase entrega **una función completa de piso a techo** (dato + contrato + regla + test + interfaz). Se puede probar al terminar cada fase. | ✅ **VIGENTE** |
>
> **El Plan Maestro conservó el NOMBRE "de adentro hacia afuera" pero cambió el SIGNIFICADO.**
> Cuando este documento dice "de adentro hacia afuera", se refiere **siempre** a la definición
> vigente del Plan Maestro §10.6: **rebanadas verticales**. Nunca a las capas horizontales
> del documento discontinuado.

**Nivel macro (entre sub-fases): rebanadas verticales.** Cada sub-fase de este plan entrega una
parte funcional de la rebanada, no una capa completa del edificio. Por eso 3.2 (backend) y 3.3
(hooks) y 3.4 (UI) se construyen en secuencia dentro de la **misma** rebanada (el flujo E.1),
en lugar de completar "todos los endpoints del POS" antes de tocar "todos los hooks".

**Nivel micro (dentro de cada sub-fase): el orden interno del Plan Maestro §10.6.**

```text
Cada sub-fase internamente sigue:
  1. Endpoint (dato + contrato)    ← cimiento de la rebanada
  2. Hook (regla de negocio)       ← comportamiento de la rebanada
  3. Test (guardián)               ← blindaje de la rebanada
  4. Componente (interfaz)         ← superficie de la rebanada
```

**Nota sobre 3.1 (utilidades transversales):** `withRetries`, `outcome`, `sessionReset`,
`useBeforeUnload` y `useNetworkHealth` **no son "comportamiento de negocio"**: son
**infraestructura** que el comportamiento consume. Por eso van **antes** del endpoint (3.2)
sin violar el orden interno: el orden `Endpoint → Hook` describe el orden de las **piezas de
negocio**, y las utilidades son el andamio sobre el que ambas se apoyan.

Y respeta la regla dura #4 del Prompt del Arquitecto:

> **NINGUNA FASE EMPIEZA sin que la PUERTA de la anterior esté en verde.**

---

## 4. LAS 5 SUB-FASES DE LA FASE 3

```text
3.0 (runner)  ──►  3.1 (utilidades)  ──►  3.2 (backend atómico)  ──►  3.3 (hooks)  ──►  3.4 (UI)
 prerrequisito      infraestructura         cimiento de la rebanada      comportamiento      superficie
 sin deps           sin deps                depende de 3.0+3.1           depende de 3.2      depende de 3.3
```

> **Cambio v1.1:** se añade **3.0** como prerrequisito (defecto D-3) y **3.2 se reordena**
> para declarar contratos antes de endpoints (defecto D-2).
>
> **Cambio v1.2:** se **confirma** este orden como el correcto tras desambiguar el principio
> rector (§3). El orden 3.1 (infraestructura) → 3.2 (cimiento) → 3.3 (comportamiento) → 3.4
> (superficie) **respeta las rebanadas verticales** del Plan Maestro §10.6. **NO se invierte.**

---

### FASE 3.0 — Prerrequisito: runner de tests real (Vitest)

> **Sin esto, ninguna puerta de frontend de este plan es verificable.**

**Por qué existe:** [`scripts/test.mjs`](../../NUEVO-POS/scripts/test.mjs:41) es un stub F0 que siempre hace `exit 0`. El frontend no tiene Vitest. Las puertas de 3.1, 3.3 y 3.4 son de frontend → **no se pueden verificar** hasta que exista el runner.

**Archivos a construir:**

| Archivo | Qué resuelve |
|---|---|
| `apps/pos/package.json` | Añadir `devDependencies`: `vitest`, `@testing-library/react`, `jsdom`; script `"test": "vitest run"` |
| `apps/pos/vitest.config.js` | Entorno `jsdom`, `globals: true` |
| `scripts/test.mjs` | Reemplazar el stub por el runner real: invoca `vitest run` (frontend) **y** `pytest` (API), sin cambiar el contrato `npm run test` |

**Puerta 3.0:**

```text
Comando : npm run test
Salida  :
  ✓ vitest ejecuta los tests existentes (theme.test.js, theme-engine.test.js)
  ✓ pytest ejecuta los tests de la API (92 tests de F3-Comportamiento)
  ✓ el runner FALLA (exit 1) si un test falla  ← prueba negativa obligatoria
Resultado: PASA / NO PASA
```

> **Prueba negativa obligatoria:** se debe demostrar que el runner **falla** cuando un test falla. Un runner que siempre pasa es exactamente el defecto que estamos corrigiendo.

**Riesgo si se salta:** todas las puertas de frontend son teatro; el proyecto cree estar verificado sin estarlo (viola E-14 "evidencia, no opinión").

---

### FASE 3.1 — Cimientos de comportamiento (utilidades transversales)

> **Sin esto, ningún hook puede ser correcto.**

**Archivos a construir:**

| Archivo | Qué resuelve | Cicatriz / Regla |
|---|---|---|
| `apps/pos/src/utils/withRetries.js` | 3 intentos, backoff 1s/2s/3s, centralizado | v7.0.1 (asimetría) |
| `apps/pos/src/utils/outcome.js` | Contrato `{ outcome, reason }` | v7.0.3 (cuentas perdidas) |
| `apps/pos/src/state/sessionReset.js` | `buildResetPatch()` — limpieza espejo explícita | Regla 19 |
| `apps/pos/src/hooks/useBeforeUnload.js` | `sendBeacon` al cerrar pestaña | H3 |
| `apps/pos/src/hooks/useNetworkHealth.js` | Banner rojo fijo + botón bloqueado | v6.1 ($453) |

**Puerta 3.1 (comando + salida esperada):**

```text
Comando : npm run test -- withRetries outcome sessionReset
          (requiere Fase 3.0 en verde; sin ella este comando no prueba nada)
Salida  :
  ✓ withRetries reintenta 3 veces con backoff 1s/2s/3s
  ✓ withRetries es idempotente (2× = mismo estado)
  ✓ outcome discrimina éxito de fallo sin asumir excepción
  ✓ sessionReset limpia EXACTAMENTE los mismos refs en toda rama de salida
  ✓ useBeforeUnload registra sendBeacon y lo limpia al desmontar
  ✓ useNetworkHealth expone banner fijo + botón bloqueado
Resultado: PASA / NO PASA
```

**Riesgo si se salta:** los hooks de 3.3 heredan un `withRetries` asimétrico y el bug reaparece en producción (incidente v7.0.1).

---

### FASE 3.2 — Backend atómico (contratos + endpoints que faltan)

> **El frontend no puede ser atómico si el backend no lo es.**

> [!CAUTION]
> **Defecto D-2 de la v1.0 corregido aquí.**
> La v1.0 listaba 5 endpoints atómicos **sin contrato declarado**. Los 17 contratos de
> [`contracts/registry.py`](../../NUEVO-POS/apps/api/contracts/registry.py:64) **no incluyen**
> ninguno de ellos (el contrato #3 es `almacenes.disponibilidad`, no `pos.añadir_item`).
> La regla **A-02 (frontera por contratos)** y la puerta de F2 exigen que **ningún endpoint
> exista sin contrato**. Por tanto **3.2 empieza declarando los contratos**, no escribiendo endpoints.

**Paso 1 — Declarar los contratos nuevos** en [`contracts/registry.py`](../../NUEVO-POS/apps/api/contracts/registry.py:64) (numerados 18–22):

| # | Nombre | Operación | Garantía clave |
|---|---|---|---|
| 18 | `pos.añadir_item` | `POST /pos/tickets/{id}/items` | Idempotente por `item_id`; aplica RN-17/18/19 |
| 19 | `pos.cambiar_cantidad` | `PATCH /pos/tickets/{id}/items/{item_id}` | Bloqueo optimista por `version` (RN-25/27) |
| 20 | `pos.quitar_item` | `DELETE /pos/tickets/{id}/items/{item_id}` | Anti-degradación RN-37 |
| 21 | `pos.leer_ticket` | `GET /pos/tickets/{id}` | **Respuesta ligera ≤5 campos** (Regla 15) |
| 22 | `pos.verificar_envio` | `POST /pos/tickets/{id}/verify` | Verificación post-envío (v6.1 $453) |

**Paso 2 — Implementar los endpoints** en [`routers/pos.py`](../../NUEVO-POS/apps/api/routers/pos.py:1), cada uno citando su contrato y sus reglas.

**Puerta 3.2:**

```text
Comando : docker compose exec api pytest tests/test_f3_atomico.py -v
          (runner REAL de la API — NO `npm run test`)
Salida  :
  ✓ test_contratos_18_a_22_declarados         (frontera A-02)
  ✓ test_idempotencia_item_no_duplica         (2× POST = mismo estado)
  ✓ test_concurrencia_optimista_409           (version obsoleto → 409)
  ✓ test_respuesta_ligera_max_5_campos        (≤5 campos escalares)
  ✓ test_verificacion_post_envio              (el ticket existe en BD)
  ✓ test_anti_degradacion_rechaza_50pct       (RN-37)
Resultado: PASA / NO PASA
```

**Riesgo si se salta:** el frontend implementa "atomicidad" sobre un backend que sigue siendo monolítico → falsa sensación de seguridad. Y si se saltan los contratos, se rompe la frontera A-02 y la puerta de F2 deja de ser válida.

---

### FASE 3.3 — Hooks del POS (el corazón)

> **Aquí vive la lógica de negocio del frontend.**

**Hooks a construir:**

| Hook | Responsabilidad | Cicatriz |
|---|---|---|
| `apps/pos/src/hooks/useCart.js` | Carrito con persistencia atómica por ítem | v6.0 SaaS |
| `apps/pos/src/hooks/useTicketActions.js` | Acciones con contrato `{outcome, reason}` | v7.0.3 |
| `apps/pos/src/hooks/useTerminalLocking.js` | Heartbeat + locks (complementa Fase 1) | OMEGA |
| `apps/pos/src/hooks/useBarcodeScanner.js` | Lector de código de barras | — |

**Reglas duras de esta sub-fase:**

- `useRef` para todo callback asíncrono (Ticket #906, prohibición #3).
- `useEffect` deps = primitivos (H1).
- `clearCart()` solo con HTTP 200 **+ verificación post-envío** (prohibición #2).

**Puerta 3.3:**

```text
Comando : npm run test -- useCart useTicketActions
          (requiere Fase 3.0 en verde; sin ella este comando no prueba nada)
Salida  :
  ✓ clearCart solo ocurre tras HTTP 200 + verificación
  ✓ callback async lee useRef, no estado cerrado (Ticket #906)
  ✓ useTicketActions devuelve { outcome, reason } en toda rama
  ✓ simetría de limpieza: éxito y fallo limpian los mismos refs
  ✓ useEffect deps son primitivos (H1)
Resultado: PASA / NO PASA
```

**Riesgo si se salta:** se repite el Ticket #906 ($124 → $2) y el incidente de la cuenta fantasma ($453).

---

### FASE 3.4 — Interfaz real (componentes)

> **La superficie visible, ya sobre cimientos sólidos.**

**Componentes a construir / completar:**

| Componente | Qué completa |
|---|---|
| `apps/pos/src/components/POSHeader.jsx` | Estado cuenta, tipo venta, indicador de red |
| `apps/pos/src/components/SalesReceipt.jsx` | Edición de cantidad, banner de estado |
| `apps/pos/src/components/CheckoutScreen.jsx` | Efectivo, tarjeta, cambio, validación |
| `apps/pos/src/components/POSOverlays.jsx` | Modales de confirmación |
| `apps/pos/src/RetailVisionPOS.jsx` | Refactor: de monolito a orquestador de hooks |

**Puerta 3.4:**

```text
Comando : npm run guards && npm run test
          (requiere Fase 3.0 en verde; sin ella `npm run test` no prueba nada)
Salida  :
  ✓ 0 console.log
  ✓ 0 try/except pass en ruta crítica
  ✓ 0 Float en modelos de dinero
  ✓ 0 DateTime() naive
  ✓ 0 TODO sin formato declarado
  ✓ Flujo E.1 (venta directa) probado manualmente
Resultado: PASA / NO PASA
```

---

## 5. TABLA RESUMEN DE SUB-FASES

| Sub-fase | Objetivo | Archivos | Depende de | Puerta |
|---|---|---|---|---|
| **3.0** | Runner de tests real (Vitest + pytest) | 3 | — | `npm run test` (con prueba negativa) |
| **3.1** | Utilidades transversales | 5 | 3.0 | `npm run test` (utilidades) |
| **3.2** | Contratos 18–22 + backend atómico | 5 contratos + 5 endpoints + tests | 3.0, 3.1 | `docker compose exec api pytest tests/test_f3_atomico.py` |
| **3.3** | Hooks del POS | 4 hooks | 3.2 | `npm run test` (hooks) |
| **3.4** | Interfaz real | 5 componentes | 3.3 | `npm run guards` + flujo E.1 |

---

## 6. CRITERIOS DE ACEPTACIÓN DE LA FASE 3 COMPLETA

La Fase 3 cierra cuando **las 5 sub-fases** tienen su puerta en verde y se cumple:

1. **Paridad funcional E.1** — la venta directa funciona igual que en el POS viejo.
2. **0 de las 5 deudas** identificadas en la auditoría.
3. **0 de los 10 acoplamientos** (test de arquitectura: 0 imports a modelos ajenos).
4. **Persistencia atómica por ítem** verificada con test de idempotencia.
5. **`clearCart()` con verificación post-envío** verificada con test.
6. **Contrato `{outcome, reason}`** en toda función de persistencia.
7. **5 greps de §7.4** en verde.
8. **Ficha de evidencia** por cada sub-fase, adjunta al repo.

---

## 7. ORDEN DE EJECUCIÓN Y PUNTO DE REANUDACIÓN

```text
[ ] 3.0 — Runner de tests real (Vitest + pytest)   ← PRERREQUISITO
[ ] 3.1 — Cimientos de comportamiento
[ ] 3.2 — Contratos 18–22 + backend atómico
[ ] 3.3 — Hooks del POS
[ ] 3.4 — Interfaz real
```

> [!NOTE]
> **Punto de reanudación:** si el trabajo se interrumpe, retomar por la **primera sub-fase con casilla vacía**. No se avanza a la siguiente sin la ficha de evidencia de la anterior.

---

## 8. RECOMENDACIÓN DE ARRANQUE

Empezar por **Fase 3.0** (runner de tests) y **luego 3.1** porque:

- **3.0 es obligatoria primero:** sin un runner que falle de verdad, ninguna puerta de frontend es verificable. Construirla primero convierte todas las puertas siguientes en evidencia real (E-14).
- **3.1 es la de menor riesgo y mayor apalancamiento:** 5 archivos pequeños que **todas** las sub-fases siguientes consumen.
- Permite validar el **ciclo completo del arquitecto** (leer → planear → construir → verificar → reportar) con una puerta barata antes de tocar el corazón transaccional.
- Si `withRetries` o `sessionReset` se diseñan mal, se detecta aquí y no en la Fase 3.3, donde el costo de corrección es **10× mayor**.

**Primera tarea concreta:** construir el runner real (3.0) con su **prueba negativa** (un test que falla debe hacer fallar el runner). Después, diseñar las firmas de `withRetries`, `outcome`, `sessionReset`, `useBeforeUnload` y `useNetworkHealth` con sus tests, y presentarlas para aprobación **antes** de implementarlas (Prompt del Arquitecto §5, paso 2).

---

## 9. REGLA DURA INQUEBRANTABLE

> [!CAUTION]
> **NO SE TOCA EL ERP INSTALADO Y CORRIENDO.**
> Todo el trabajo de la Fase 3 se construye **solo en `NUEVO-POS`**.
> El ERP en `ERP-R-DE-RICO` permanece intacto (HEAD `5802f45`, V23).

---

## 10. AUTOCRÍTICA (v1.0 → v1.2)

> Esta sección es el resultado de revisar la v1.0 **contra la evidencia del repositorio**,
> no contra la memoria. Cada defecto cita el archivo y la línea que lo prueban.

### 10.1 Defectos encontrados

| # | Defecto | Gravedad | Evidencia | Corrección |
|---|---|---|---|---|
| **D-1** | **Colisión de numeración de fases.** La v1.0 llamó "Fase 3" a lo que el Plan Maestro §7 llama Fase 3, pero el Plan de Construcción ya tiene una "F3 Comportamiento" **cerrada**. Ambigüedad real. | Alta | [`FICHA_F3_COMPORTAMIENTO.md:1`](../../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F3_COMPORTAMIENTO.md:1) vs [`PLAN_MAESTRO_DEFINITIVO_POS.md:275`](../PLAN_MAESTRO_DEFINITIVO_POS.md:275) | §1 tabla de numeración + término **"F3-Maestro"** |
| **D-2** | **Endpoints sin contrato.** La v1.0 propuso 5 endpoints atómicos que **no existen** en los 17 contratos. Viola A-02 y la puerta de F2. | **Crítica** | [`contracts/registry.py:64-386`](../../NUEVO-POS/apps/api/contracts/registry.py:64) (no hay `pos.añadir_item`) | 3.2 ahora **declara contratos 18–22 primero** |
| **D-3** | **Puertas con un runner inexistente.** La v1.0 usó `npm run test` para frontend, pero ese script es un **stub que siempre pasa**. | **Crítica** | [`scripts/test.mjs:41-53`](../../NUEVO-POS/scripts/test.mjs:41) + [`apps/pos/package.json:7-11`](../../NUEVO-POS/apps/pos/package.json:7) | Nueva **Fase 3.0** (runner real + prueba negativa) |
| **D-4** | **Runner equivocado en el backend.** La v1.0 usó `pytest` a secas; el entorno real es Docker. | Media | [`FICHA_F3_COMPORTAMIENTO.md:10`](../../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F3_COMPORTAMIENTO.md:10) | Puerta 3.2 = `docker compose exec api pytest` |
| **D-5** | **Alcance de F3-Maestro mal delimitado.** La v1.0 no dijo qué **NO** entra en F3-Maestro (p. ej. caja, pizarrón, impresión, IA, CRM son F4–F8). | Media | [`PLAN_MAESTRO_DEFINITIVO_POS.md:304-408`](../PLAN_MAESTRO_DEFINITIVO_POS.md:304) | §10.3 "Fuera de alcance" |
| **D-6** | **Doble significado de "de adentro hacia afuera".** El término se usó con dos sentidos opuestos: (a) capas horizontales (documento discontinuado) y (b) rebanadas verticales (Plan Maestro vigente). Esto llevó a proponer **invertir** el orden de las sub-fases, lo que habría **desalineado** el plan. | **Crítica** | [`PLAN_DE_CONSTRUCCION_DEL_NUEVO_POS.md:40`](../PLANES%20DESCONTINUADOS%20DEL%20NUEVO%20POS/PLAN_DE_CONSTRUCCION_DEL_NUEVO_POS.md:40) (descartado) vs [`PLAN_MAESTRO_DEFINITIVO_POS.md:516`](../PLAN_MAESTRO_DEFINITIVO_POS.md:516) (vigente) | **§3 desambiguación permanente** + §10.5 |

### 10.2 Alineación con el objetivo del proyecto

El objetivo (Plan Maestro §1) es construir la versión mejorada del POS **"como si se hubiera diseñado desde un principio por un arquitecto senior"**, sin tocar el ERP vivo.

| Verificación | Resultado |
|---|---|
| ¿Respeta la regla dura (no tocar el ERP)? | ✅ §9 lo declara; todo el trabajo va en `NUEVO-POS` |
| ¿Respeta las 6 prohibiciones? | ✅ 3.1/3.3 las citan explícitamente (clearCart, useRef, sin timers) |
| ¿Respeta las 10 reglas de batalla? | ✅ `{outcome,reason}`, withRetries, respuesta ligera, UTC, sendBeacon, limpieza espejo |
| ¿Respeta los 16–18 estándares (E-01…E-18)? | ⚠️ **Parcial en v1.0** — violaba E-14 (evidencia, no opinión) al usar un runner falso. **Corregido en v1.1** |
| ¿Respeta la frontera por contratos (A-02)? | ❌ **No en v1.0** — endpoints sin contrato. **Corregido en v1.1** |
| ¿Sigue el orden "de adentro hacia afuera" (§10.6)? | ✅ **Sí, en su definición VIGENTE (rebanadas verticales).** Ver §3 para la desambiguación. Orden interno: Endpoint → Hook → Test → Componente |
| ¿Cada sub-fase tiene puerta verificable? | ⚠️ **No en v1.0** (runner falso). **Corregido en v1.1** |

### 10.3 Fuera de alcance de F3-Maestro (defecto D-5)

Para evitar que F3-Maestro se convierta en un cajón de sastre, **NO entra** aquí:

| Fase | Qué es | Por qué no entra |
|---|---|---|
| F4 | Gestor de Caja | Es su propia rebanada (contratos 9–14 ya existen) |
| F5 | Pizarrón de Cuentas Abiertas | Depende de F3-Maestro terminada |
| F6 | Impresión + PDF de Catálogo | Módulo aparte |
| F7 | Voz + Visión IA + Temas | Módulo aparte |
| F8 | CRM y Notificaciones | Módulo aparte |

**F3-Maestro entrega:** el flujo **E.1 (venta directa)** completo y atómico, con sus hooks, endpoints y UI. Nada más.

### 10.4 Riesgo residual aceptado

| Riesgo | Mitigación |
|---|---|
| Construir 3.0 (runner) es trabajo "no visible" para el dueño | Se documenta como prerrequisito con su propia ficha de evidencia |
| Los contratos 18–22 podrían requerir revisión del arquitecto | Se presentan **antes** de implementar (Prompt §5, paso 2) |
| El orden 3.0→3.1→3.2→3.3→3.4 es estricto | La regla dura #4 lo exige: ninguna fase sin la puerta anterior en verde |

### 10.5 REGISTRO PERMANENTE DE DESAMBIGUACIÓN (defecto D-6)

> **Propósito:** que este error de lectura **no pueda repetirse**. Cualquier persona o IA que
> lea este plan (o el Plan Maestro) debe encontrar aquí la respuesta antes de proponer un
> cambio de orden.

**El hecho:** el término **"de adentro hacia afuera"** aparece en dos documentos con **dos
significados incompatibles**:

| # | Documento | Significado | Estado |
|---|---|---|---|
| 1 | [`PLAN_DE_CONSTRUCCION_DEL_NUEVO_POS.md:40`](../PLANES%20DESCONTINUADOS%20DEL%20NUEVO%20POS/PLAN_DE_CONSTRUCCION_DEL_NUEVO_POS.md:40) | **Capas horizontales:** datos → contratos → reglas → interfaces. Terminar una capa entera antes de la siguiente. | ❌ **DESCARTADO** — el propio documento dice en su encabezado que fue *"reemplazado por el PLAN MAESTRO DEFINITIVO"* |
| 2 | [`PLAN_MAESTRO_DEFINITIVO_POS.md:516`](../PLAN_MAESTRO_DEFINITIVO_POS.md:516) §10.6 | **Rebanadas verticales:** cada fase entrega una función completa de piso a techo. | ✅ **VIGENTE** — es el documento padre de este plan |

**La regla de decisión (aplicar siempre):**

1. **¿El documento está en `PLANES DESCONTINUADOS DEL NUEVO POS/`?** → **No manda.** Es historia.
2. **¿El documento es el `PLAN_MAESTRO_DEFINITIVO_POS.md`?** → **Manda.** Es el documento padre.
3. **¿Hay conflicto entre ambos?** → **Gana el Plan Maestro**, sin excepción.

**La consecuencia sobre el orden de sub-fases (para que nadie lo invierta):**

```text
CORRECTO (rebanadas verticales — Plan Maestro §10.6 vigente):
  3.0 runner → 3.1 utilidades → 3.2 backend → 3.3 hooks → 3.4 UI

INCORRECTO (capas horizontales — documento discontinuado):
  ✗ "primero TODOS los contratos, luego TODAS las utilidades, luego TODOS los hooks"
  ✗ Invertir 3.1 y 3.2 para poner contratos antes que utilidades
```

**Por qué el orden correcto respeta las rebanadas verticales:**

| Sub-fase | Rol en la rebanada | Nivel |
|---|---|---|
| 3.0 runner | Andamio de verificación | Infraestructura |
| 3.1 utilidades | Andamio de comportamiento | Infraestructura |
| 3.2 backend | **Cimiento** de la rebanada (dato + contrato) | Negocio |
| 3.3 hooks | **Comportamiento** de la rebanada | Negocio |
| 3.4 UI | **Superficie** de la rebanada | Negocio |

Las sub-fases 3.0 y 3.1 son **infraestructura** (andamio), no "comportamiento de negocio".
Por eso pueden ir antes del cimiento sin violar el orden interno `Endpoint → Hook → Test →
Componente`, que describe el orden de las **piezas de negocio** (3.2 → 3.3 → 3.4).

**Firma de la desambiguación:** este registro se añadió en la v1.2 tras detectar que la v1.1
estaba a punto de invertir el orden por leer el documento discontinuado como si mandara.

---

*Plan de Abordaje de la Fase 3 por Partes. **Versión 1.2 (desambiguación del principio rector)**. Alineado al Plan Maestro v1.2 §7 y §10.6, y al Prompt del Arquitecto v1.2.*
