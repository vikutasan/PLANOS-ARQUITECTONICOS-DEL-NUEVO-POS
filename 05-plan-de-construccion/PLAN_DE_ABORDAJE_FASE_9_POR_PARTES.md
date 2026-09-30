# PLAN DE ABORDAJE — FASE 9 POR PARTES

**Proyecto:** POS Nuevo "R de Rico"
**Fase:** 9 — Rescate de UX del viejo POS (y afinado de la UI por defecto)
**Versión del plan:** 1.1
**Fecha:** 30 Sep 2026
**Estado:** ✅ APROBADO — en ejecución
**Autor:** Arquitecto del Nuevo POS

---

## 0. POR QUÉ EXISTE ESTA FASE

El usuario invirtió mucho en la UX del **viejo POS** (`apps/pos/` del ERP). Antes de
definir la UI por defecto del módulo nuevo, pidió comparar ambas UX y rescatar lo que
valga la pena **sin mayor problema**.

La comparación (archivo por archivo) arrojó un hallazgo tranquilizador: **el POS nuevo
ya heredó ~80% de la UX del viejo** (ticket de papel, barra de categorías, visión como
modo de vista, header de 3 zonas, pizarrón, gestor de caja, panel de voz, programación
de pedido, edición de cantidad). El §6.8 del Plan Maestro — *"la integración se hereda,
la implementación se reescribe"* — se cumplió.

Quedan **tres piezas concretas** de bajo riesgo y alto valor, más **una capacidad mayor**
que NO es "sin mayor problema" porque toca el contrato de cobro.

Esta fase ejecuta las tres piezas de bajo riesgo. La capacidad mayor se planifica aparte
(F9.1).

---

## 1. ALCANCE

### 1.1 Lo que SÍ hace esta fase (F9.0)

| # | Pieza | Origen (viejo POS) | Riesgo |
|---|-------|--------------------|--------|
| 1 | **Modal de salida "Cuenta sin enviar"** | `RetailVisionPOS.jsx` (modal inline) | Muy bajo (puro frontend) |
| 2 | **Teclado numérico + cambio en vivo en el cobro** | `CheckoutScreen.jsx` (teclado 1-9/0/./C) | Bajo (UI del modal existente) |
| 3 | **`OfflineBanner` (solo estado de red, SIN conteo)** | `POSOverlays.jsx` (`OfflineBanner`) | Bajo (ya existe `useNetworkHealth`) |

> **DECISIÓN ARQUITECTÓNICA (v1.1) — No se construye cola local.** Se verificó
> contra el código real (REGLA DURA 2) que el POS nuevo **no tiene cola local**:
> persiste directo contra la API en cada acción ([`useCart.anadirLinea`](../../../NUEVO-POS/apps/pos/src/hooks/useCart.js:122)
> → `cliente.anadirItem`). Lo único en `localStorage` son preferencias de UI
> (`pos.tema`, `pos.ordenTerminales`), no operaciones. Por tanto, la pieza 3
> **NO muestra un conteo de pendientes** (no hay de dónde sacarlo sin inventarlo).
> Construir una cola local sería un motor offline-first que contradice el modelo
> "el servidor es la única fuente de verdad", reintroduce el riesgo de la cicatriz
> de $453 (cobrar sin red) y toca caja, folios y multi-terminal. Si algún día el
> dolor de red es real y medido, se abrirá una **fase propia** (p. ej. "F10 —
> Modo offline-first"), no un parche de UX.

### 1.2 Lo que NO hace esta fase (explícito)

- **NO** implementa **pagos mixtos** (abonar efectivo + tarjeta en la misma cuenta).
  Eso toca el contrato de cobro (`POST /pos/tickets/{id}/pay`) y las reglas de negocio.
  Va a **F9.1** con su propio plan, sus RN y sus tests.
- **NO** cambia el sistema de temas ni los colores. El nuevo POS usa tokens
  (`--madera`, `--acento`, `text-crema-ticket`); el viejo usaba colores hardcodeados.
  Rescatamos la **gramática de interacción**, no los valores de color.
- **NO** revierte el layout responsivo (MOSTRADOR/COMPACTO/MÓVIL). El viejo era
  desktop-only; el nuevo es fluido. Se conserva lo nuevo.
- **NO** toca el backend. Las tres piezas son 100% frontend.
- **NO** toca los contratos ni las reglas de negocio (RN). No hay migración.

---

## 2. PRINCIPIOS RECTORES (heredados, no negociables)

1. **"Verificar, no asumir" (REGLA DURA 2).** Cada pieza se verifica contra el código
   real antes de escribirla.
2. **"De adentro hacia afuera".** Primero el componente con su gate (test), luego el
   cableado en la pantalla real.
3. **Gate primero.** Ninguna pieza se considera terminada sin su test verde.
4. **La IA propone; el operador decide.** (No aplica aquí, pero se respeta el patrón.)
5. **Banners persistentes, no auto-ocultables** (Regla 19 / prohibición #2). El
   `OfflineBanner` NO se auto-oculta.
6. **Target táctil ≥ 44×44px** (R-04) en todo botón nuevo.
7. **Sin anchos absolutos** (R-01); los 3 modos de layout (R-03) se respetan.
8. **Ficha de evidencia + commit + push** al cerrar cada sub-fase.

---

## 3. SUB-FASES

### 3.1 F9.0.1 — Modal de salida "Cuenta sin enviar"

**Problema:** hoy, si el operador intenta salir (cambiar de estación / volver al
selector) con una cuenta abierta, el nuevo POS **no ofrece salvaguarda**. El viejo POS
ofrecía 3 caminos explícitos.

**Solución:** un overlay de confirmación con 3 acciones, reutilizando el patrón de
`OverlayConfirmar` (pero con 3 botones, no 2):

- **"📌 Enviar al Pizarrón y salir"** → deja la cuenta abierta (ya está persistida por
  la persistencia atómica por ítem) y sale.
- **"🚪 Salir sin enviar — perder cuenta"** → acción destructiva (color `peligro`).
- **"Cancelar — quedarme"** → cierra el modal.

**Archivos:**
- **Nuevo:** `apps/pos/src/components/ExitAccountModal.jsx` (o extender
  `POSOverlays.jsx` con `OverlaySalidaCuenta`).
- **Modificar:** `apps/pos/src/RetailVisionPOS.jsx` — interceptar `onBackToTerminals`
  cuando `carrito.lineas.length > 0`.
- **Test:** `apps/pos/src/components/ExitAccountModal.f9_0_1.test.jsx`.

**Gate (criterios de aceptación):**
1. Con el carrito vacío, salir NO muestra el modal (sale directo).
2. Con el carrito con ítems, salir MUESTRA el modal.
3. "Enviar al Pizarrón y salir" invoca el callback de salida sin borrar la cuenta.
4. "Salir sin enviar" invoca el callback de salida marcando la cuenta como perdida.
5. "Cancelar" cierra el modal y NO sale.
6. Los 3 botones respetan el target táctil ≥44px.
7. `role="dialog"` + `aria-modal="true"`.

**Riesgo:** Muy bajo. Puro frontend, sin API.

---

### 3.2 F9.0.2 — Teclado numérico + cambio en vivo en el cobro

**Problema:** el nuevo `CheckoutScreen` captura el efectivo con un `<input type="number">`
y botones rápidos ($50/$100/$200/$500). En una terminal táctil (tablet/mostrador sin
teclado físico), el input nativo es incómodo. El viejo POS tenía un **teclado numérico
grande en pantalla** (1-9, 0, ., C).

**Solución:** añadir un teclado numérico en pantalla al bloque de EFECTIVO, **sin quitar**
el input (se conservan ambos: el input para teclado físico, el teclado para táctil). El
cambio en vivo ya existe (`cambio` en `CheckoutScreen.jsx`); se conserva y se resalta.

**Archivos:**
- **Nuevo:** `apps/pos/src/components/TecladoNumerico.jsx` (componente puro, reutilizable).
- **Modificar:** `apps/pos/src/components/CheckoutScreen.jsx` — montar el teclado en el
  bloque de EFECTIVO.
- **Test:** `apps/pos/src/components/TecladoNumerico.f9_0_2.test.jsx` +
  ampliar `components.f3_4.test.jsx` (o un test nuevo del CheckoutScreen).

**Gate (criterios de aceptación):**
1. El teclado tiene teclas 1-9, 0, "." y "C".
2. Pulsar dígitos concatena al valor recibido.
3. "." solo se permite una vez.
4. "C" limpia el valor.
5. El cambio en vivo se recalcula al pulsar.
6. El botón "CONFIRMAR PAGO" sigue deshabilitado si el recibido < total.
7. Cada tecla respeta el target táctil ≥44px.
8. El input nativo sigue funcionando (no se rompe el caso de teclado físico).

**Riesgo:** Bajo. UI del modal existente; no toca el contrato de cobro.

---

### 3.3 F9.0.3 — `OfflineBanner` (solo estado de red, SIN conteo)

**Problema:** el viejo POS mostraba un banner con `pendingCount` + `isSyncing` (cuántas
operaciones están en cola local). El nuevo POS tiene indicador de red en el header, pero
**no tiene un banner fijo y visible** que avise de forma inequívoca que no hay conexión.

**Verificación (REGLA DURA 2) — RESUELTA:** se verificó el código real. El POS nuevo
**NO tiene cola local** (ver la DECISIÓN ARQUITECTÓNICA en §1.1). Por tanto, **no existe
un conteo de pendientes que mostrar**. El banner muestra **solo el estado de red**.

**Solución:** un banner fijo (no auto-ocultable) que aparece cuando `!enLinea`, con el
mensaje *"⚠️ Sin conexión — el cobro está bloqueado hasta que vuelva la red"*. Se
reutiliza el estado que ya expone [`useNetworkHealth`](../../../NUEVO-POS/apps/pos/src/hooks/useNetworkHealth.js:42)
(`{ enLinea, bannerVisible, botonBloqueado }`). **No se muestra ningún número.**

**Archivos:**
- **Modificar:** `apps/pos/src/components/POSOverlays.jsx` — añadir `OfflineBanner`
  (recibe `visible`; sin `pendingCount`).
- **Modificar:** `apps/pos/src/RetailVisionPOS.jsx` — montarlo cuando `!enLinea`.
- **Test:** `apps/pos/src/components/POSOverlays.f9_0_3.test.jsx`.

**Gate (criterios de aceptación):**
1. Con red, el banner NO se muestra.
2. Sin red, el banner SÍ se muestra.
3. El banner NO se auto-oculta (persistente).
4. El banner **NO muestra ningún conteo de pendientes** (no existe cola local).
5. El mensaje comunica que el cobro está bloqueado (coherente con `botonBloqueado`).
6. `role="alert"`.

**Riesgo:** Bajo. Verificación ya resuelta: no hay cola local → no hay conteo.

---

### 3.4 F9.0.4 — Cierre de la fase

- Correr `npm run ci` completo (lint + tests + guards).
- Escribir `FICHA_F9_UX_RESCATE.md` con: sub-fases, commits, criterios de aceptación,
  evidencia del CI, matriz de trazabilidad, y qué NO hizo la fase.
- Actualizar §7 del Plan Maestro (Fase 9 → ✅ CERRADA).
- Commit + push (NUEVO-POS + PLANOS).
- Registrar el hash real en la ficha (commit de seguimiento).

---

## 4. CRITERIOS DE ACEPTACIÓN DE LA FASE (globales)

1. Las 3 piezas están implementadas y cableadas en la pantalla real.
2. Cada pieza tiene su test verde.
3. `npm run ci` verde (lint 0 errores + todos los tests + los 7 guards limpios).
4. La ficha de cierre existe y está completa.
5. §7 del Plan Maestro actualizado.
6. Commits + push en ambos repos.
7. **Ningún** cambio en backend, contratos ni reglas de negocio.
8. **Ningún** color hardcodeado nuevo (se usan los tokens del tema).

---

## 5. TRAZABILIDAD

| Pieza | Origen (viejo POS) | Destino (nuevo POS) | Test |
|-------|--------------------|---------------------|------|
| Modal de salida | `RetailVisionPOS.jsx` (modal inline) | `ExitAccountModal.jsx` | `ExitAccountModal.f9_0_1.test.jsx` |
| Teclado numérico | `CheckoutScreen.jsx` (teclado) | `TecladoNumerico.jsx` | `TecladoNumerico.f9_0_2.test.jsx` |
| OfflineBanner | `POSOverlays.jsx` (`OfflineBanner` con `pendingCount`) | `POSOverlays.jsx` (`OfflineBanner` solo estado de red) | `POSOverlays.f9_0_3.test.jsx` |

---

## 6. LO QUE ESTA FASE **NO** HACE (para evitar confusión futura)

- **NO** implementa pagos mixtos → **F9.1**.
- **NO** cambia colores ni temas.
- **NO** revierte el layout responsivo.
- **NO** toca backend, contratos ni RN.
- **NO** construye una cola local (offline-first) → decisión arquitectónica en §1.1.
- **NO** muestra un conteo de pendientes (no existe cola local que lo respalde).

---

## 7. RIESGOS

| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|--------------|---------|------------|
| El modal de salida interfiere con el flujo de `onBackToTerminals` | Baja | Medio | Interceptar solo si `carrito.lineas.length > 0`; test explícito |
| El teclado numérico rompe el input nativo | Baja | Bajo | Conservar ambos; test del caso de teclado físico |
| No existe cola local para el conteo de pendientes | **Confirmada** | Nulo | Verificado: no hay cola local → el banner muestra solo el estado de red (§1.1) |
| Regresión en el gate de Fase 3 (componentes montados sin props nuevas) | Baja | Medio | Props nuevas opcionales; correr el CI completo |

---

## 8. BITÁCORA DE CAMBIOS

| Versión | Fecha | Cambio |
|---------|-------|--------|
| 1.0 | 30 Sep 2026 | Versión inicial. 3 piezas de bajo riesgo + cierre. F9.1 (pagos mixtos) se planifica aparte. |
| 1.1 | 30 Sep 2026 | Verificación REGLA DURA 2 resuelta: el POS nuevo NO tiene cola local. F9.0.3 pasa a "solo estado de red, SIN conteo". Se registra la DECISIÓN ARQUITECTÓNICA de no construir cola local (§1.1). Estado → APROBADO. |

---

## 9. AUTOCRÍTICA

**¿Esta fase contribuye al objetivo del proyecto?** Sí, de forma acotada y honesta:
- **A favor:** rescata UX que el usuario ya pagó y valoró, con riesgo mínimo y sin tocar
  la arquitectura. Mejora la operación real (salvaguarda de cuentas, cobro táctil, aviso
  de red).
- **En contra / límite:** es una fase **cosmética-operativa**, no estructural. No añade
  capacidad de negocio nueva. La capacidad que SÍ importa (pagos mixtos) se deja
  explícitamente fuera y se planifica aparte, para no mezclar "rescate de UI" con
  "cambio de contrato".
- **Riesgo de sobre-ingeniería:** bajo. Las 3 piezas son componentes pequeños y
  testeables. La disciplina de "gate primero" evita que crezcan.
- **Honestidad sobre el conteo de pendientes:** verificado que NO hay cola local, NO se
  inventa un número. Se dice la verdad en la UI (solo estado de red).
- **Honestidad sobre la cola local:** se evaluó construirla y se **rechazó** con
  argumentos (contradice "el servidor es la única verdad", reintroduce el riesgo de la
  cicatriz de $453, toca caja/folios/multi-terminal). Si el dolor de red es real y
  medido, será una fase propia, no un parche de UX.

**Conclusión:** la fase es correcta como **micro-fase de afinado**, siempre que se
respete el alcance y no se cuele F9.1 dentro.
