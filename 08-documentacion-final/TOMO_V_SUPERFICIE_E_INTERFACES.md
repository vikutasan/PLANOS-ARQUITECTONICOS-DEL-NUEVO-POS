# TOMO V — SUPERFICIE E INTERFACES

> **Fuente principal:** `apps/api/superficie/registry.py` (registro de las 24 interfaces), `Documento 6` (responsividad), `Documento 7` (fichas 01–24), `Documento 5 §E` (los 6 flujos).
> **Propósito:** documentar la cara visible del POS nuevo: las 24 interfaces que el operador toca, la paleta canónica que las identifica, las 4 reglas duras de responsividad que las gobiernan, los 10 contenedores críticos que se parametrizaron, los 6 flujos funcionales que la puerta F5 exige, y las interfaces que se DESCARTARON (y por qué). Este tomo es el puente entre los contratos (Tomo IV) y la obra construida (Tomo VI).

---

## ÍNDICE

- [0. Cómo leer este tomo](#0-cómo-leer-este-tomo)
- [1. La superficie como contrato visual](#1-la-superficie-como-contrato-visual)
- [2. La paleta canónica y los radios](#2-la-paleta-canónica-y-los-radios)
- [3. Las 4 reglas duras de responsividad](#3-las-4-reglas-duras-de-responsividad)
- [4. Los 3 modos de layout](#4-los-3-modos-de-layout)
- [5. Las 24 interfaces (inventario completo)](#5-las-24-interfaces-inventario-completo)
- [6. Los 10 contenedores críticos parametrizados](#6-los-10-contenedores-críticos-parametrizados)
- [7. Los 6 flujos funcionales (paridad F5)](#7-los-6-flujos-funcionales-paridad-f5)
- [8. Las interfaces descartadas (y por qué)](#8-las-interfaces-descartadas-y-por-qué)
- [9. El defecto del plano: 26 vs 24](#9-el-defecto-del-plano-26-vs-24)
- [10. El test de la puerta F5](#10-el-test-de-la-puerta-f5)
- [11. Matriz de trazabilidad: interfaz → tipo → anclaje → modos](#11-matriz-de-trazabilidad-interfaz--tipo--anclaje--modos)

---

## 0. Cómo leer este tomo

Este tomo documenta la **superficie** del POS: todo lo que el operador ve y toca. La superficie es la cara visible de los contratos del Tomo IV: cada botón, cada panel, cada modal es el punto donde un contrato se hace tangible.

El tomo responde a cinco preguntas:

1. **¿Cuál es la identidad visual?** → §2 (paleta canónica y radios).
2. **¿Qué reglas gobiernan el layout?** → §3 (las 4 reglas duras) y §4 (los 3 modos).
3. **¿Cuáles son las 24 interfaces?** → §5 (inventario completo).
4. **¿Qué se parametrizó para que fuera responsivo?** → §6 (los 10 contenedores críticos).
5. **¿Qué se descartó y por qué?** → §8 (las interfaces descartadas).

**La regla de lectura:** la superficie NO es decoración. Es la implementación visible de la arquitectura. Un panel que lee una tabla ajena viola A-02 aunque se vea bien. Un contenedor con ancho fijo viola R-01 aunque funcione en la pantalla del desarrollador.

---

## 1. La superficie como contrato visual

La superficie del POS es un **contrato visual** con el operador. Igual que los contratos del Tomo IV definen cómo hablan los módulos, la superficie define cómo habla el sistema con el humano.

### 1.1 Los tres compromisos de la superficie

| Compromiso | Qué significa | Regla que lo protege |
|------------|---------------|----------------------|
| **Identidad estable** | La paleta y los radios no cambian entre modos ni entre pantallas | §2 (paleta canónica) |
| **Responsividad real** | Los 3 modos son explícitos, no accidentales | R-01 a R-04 |
| **Paridad funcional** | Los 6 flujos del viejo POS se replican | §7 (flujos E.1–E.6) |

### 1.2 La superficie hereda la UX, no la implementación

La decisión D-08 del Plan Maestro es tajante: **la UX del viejo POS se hereda; la implementación se reescribe.** Esto significa:

- **Se hereda:** el flujo de venta, la disposición de los paneles, la lógica de los modales, la sensación de operación.
- **Se reescribe:** el código, los contenedores, la responsividad, la gestión de estado.

El operador no debe notar la diferencia en el flujo; el desarrollador debe notar la diferencia en el código. La superficie es el punto donde esta decisión se hace visible.

### 1.3 La superficie es la puerta F5

La FASE 5 (Superficie) tiene su propia puerta de calidad. El test de la puerta F5 recorre el registro de interfaces y verifica:

1. Que las 24 interfaces existan.
2. Que ninguna tenga ancho fijo en su contenedor raíz (R-01), salvo las exentas (impresión).
3. Que la paleta canónica se respete.
4. Que los 3 modos se declaren (R-03).

La superficie no se acepta por inspección visual: se acepta por una puerta que la verifica mecánicamente.

---

## 2. La paleta canónica y los radios

La paleta canónica es la identidad visual del POS. Sus valores viajan intactos en los 3 modos.

### 2.1 La paleta canónica (Documento 7 §2.1)

| Token | Valor | Uso |
|-------|-------|-----|
| `acento_principal` | `#c1d72e` | El verde lima característico del POS. Botones primarios, acentos, el "sello" de la marca. |
| `fondo_profundo` | `#0a0a0a` | El negro de fondo principal. |
| `fondo_profundo_alt` | `#080808` | El negro alterno (variación sutil). |
| `fondo_panel` | `#1a1a1a` | El gris oscuro de los paneles y modales. |
| `crema_ticket` | `#fdfbf7` | El crema del papel del ticket. |
| `rojo_peligro` | `#ef4444` | El rojo de peligro (red-500). Acciones destructivas. |

### 2.2 Los radios canónicos (Documento 7 §2.3)

| Radio | Uso |
|-------|-----|
| `rounded-[35px]` | Tarjetas de producto, contenedores principales. |
| `rounded-[40px]` | Contenedores secundarios. |
| `rounded-[50px]` | Contenedores de énfasis. |

### 2.3 La regla de la paleta

> **La paleta canónica es intocable. Un componente que use un color fuera de la paleta NO se acepta en revisión.**

La paleta no es una sugerencia estética: es la identidad del POS. Cambiar un color es cambiar la marca. La puerta F5 verifica que la paleta se respete.

### 2.4 La excepción de la paleta

Las pantallas de Grandeza (interfaces 4–7) usan colores adicionales porque son pantallas de administración y reparto, no del POS transaccional:

| Interfaz | Colores adicionales |
|----------|---------------------|
| 4 GrandezaParamsUI | `#3b82f6` (azul), `#6366f1` (índigo) |
| 5 GrandezaDailyUI | `#f59e0b` (ámbar), `#10b981` (esmeralda) |
| 6 GrandezaDriverUI | `#f59e0b` (ámbar) |
| 7 RepartoPanGrandezaUI | `#f59e0b` (ámbar) |

Estos colores son de las suites de Grandeza, no del POS transaccional. El POS transaccional (interfaces 1–3, 8–24) usa estrictamente la paleta canónica.

---

## 3. Las 4 reglas duras de responsividad

Las 4 reglas duras (Documento 6 §3) son innegociables. Un componente que las viole NO se acepta en revisión.

### 3.1 R-01 — Cero anchos absolutos en contenedores raíz

> **`w-[...px]` prohibido en el contenedor raíz.**

Un contenedor raíz con ancho fijo no puede adaptarse a los 3 modos. La regla permite `max-w-[...]` (fluido), `min-w-[...]` (tablas con scroll) y anchos acotados a un breakpoint (`sm:`, `md:`, `lg:`, `xl:`, `2xl:`), porque no fijan el ancho raíz en todos los modos.

**El patrón de detección:** el registro usa una expresión regular (`PATRON_ANCHO_FIJO`) que detecta `w-[<n>px]` sin acotar, excluyendo `max-w-`, `min-w-` y los prefijos de breakpoint.

### 3.2 R-02 — Tipografía que escala, no que se fija

> **`text-[7px]`/`[8px]`/`[9px]` prohibido como base legible.**

La tipografía debe escalar con el modo. Una base de 7px es ilegible en móvil. La excepción son las plantillas de impresión (interfaces 23–24), que usan `font-mono` con tamaños fijos porque el papel térmico tiene un ancho físico fijo.

### 3.3 R-03 — Los 3 modos son explícitos, no implícitos

> **MÓVIL base, `md:` COMPACTO, `lg:` MOSTRADOR.**

Los 3 modos se declaran explícitamente en el código. No se asume que un componente "se verá bien" en móvil: se declara cómo se ve. El registro verifica que cada interfaz declare los 3 modos (`declara_los_3_modos`).

### 3.4 R-04 — Targets táctiles de 44×44px mínimo

> **44×44px mínimo, con ≥8px de separación entre adyacentes.**

Los botones y controles táctiles deben ser lo bastante grandes para el dedo. Un botón de 20px es inusable en una tablet. La separación de 8px evita toques accidentales en el botón vecino.

### 3.5 La jerarquía de las reglas

| Regla | Qué protege | Consecuencia de violarla |
|-------|-------------|--------------------------|
| R-01 | El layout raíz | El componente no se adapta a los modos |
| R-02 | La legibilidad | El texto es ilegible en móvil |
| R-03 | La explicitud | El modo móvil es un accidente |
| R-04 | La usabilidad táctil | El operador toca el botón equivocado |

---

## 4. Los 3 modos de layout

El POS tiene 3 modos de layout (Documento 6 §2). El modo MOSTRADOR es la referencia de verdad y es INTOCABLE. COMPACTO y MÓVIL son ADICIONES, no sustituciones.

### 4.1 MOSTRADOR (≥1024px)

El modo de referencia. Es el layout del POS en la pantalla grande del mostrador. **Es intocable:** no se modifica para acomodar los otros modos. Los otros modos se construyen alrededor de él.

### 4.2 COMPACTO (768–1023px)

El modo de tablet. El layout se estrecha: el ticket lateral se colapsa, el grid de productos reduce columnas, los modales se ensanchan. Es una ADICIÓN al MOSTRADOR, no un reemplazo.

### 4.3 MÓVIL (<768px)

El modo de teléfono. El layout se apila: el ticket se convierte en panel inferior deslizable, el grid pasa a 2 columnas, las acciones van a una barra inferior. Es una ADICIÓN, no un reemplazo.

### 4.4 La regla de los 3 modos

> **El modo MOSTRADOR es la verdad. COMPACTO y MÓVIL son adiciones. Nunca se sacrifica el MOSTRADOR para acomodar los otros.**

Esta regla es la que evita el error clásico de "responsive design" que degrada la experiencia de escritorio para que quepa en móvil. En el POS, el escritorio manda.

---

## 5. Las 24 interfaces (inventario completo)

Las 24 interfaces se agrupan en 5 tipos. El inventario es la fuente única de verdad.

### 5.1 Resumen por tipo

| Tipo | Cantidad | Componentes |
|------|----------|-------------|
| Pantallas raíz | 7 | RetailVisionPOS, TableServicePOS, VisionTrainingUI, GrandezaParamsUI, GrandezaDailyUI, GrandezaDriverUI, RepartoPanGrandezaUI |
| Modales | 6 | CheckoutScreen, GestionPersonal, GestorDeCaja, ProgramacionPedidoModal, TerminalSelector, OpenAccountsCorkboard |
| Paneles y overlays | 5 | SalesReceipt, POSHeader, POSOverlays, VisionVisor, VisionScanner |
| Composición | 4 | ProductGrid, ProductCard, CategoryBar, CategoryEditor |
| Impresión | 2 | TicketTemplate, CorteTicketTemplate |
| **TOTAL** | **24** | |

### 5.2 §3 Pantallas raíz (7)

| # | Nombre | Anclaje | Modos (MOSTRADOR / COMPACTO / MÓVIL) |
|---|--------|---------|--------------------------------------|
| 1 | RetailVisionPOS | `apps/pos/RetailVisionPOS.jsx` | 2 columnas / ticket estrecho + grid 3 col / ticket panel inferior + grid 2 col |
| 2 | TableServicePOS | `apps/pos/TableServicePOS.jsx` | grid 4 col + comanda lateral / grid 2 col + comanda apilada / grid 2 col + comanda inferior |
| 3 | VisionTrainingUI | `apps/pos/VisionTrainingUI.jsx` | panel 450px / panel w-full bajo contenido / panel w-full 1 col |
| 4 | GrandezaParamsUI | `apps/pos/GrandezaParamsUI.jsx` | tabs con columnas laterales / tabs apiladas / 1 col scroll |
| 5 | GrandezaDailyUI | `apps/pos/GrandezaDailyUI.jsx` | columnas / columnas apiladas / 1 col scroll |
| 6 | GrandezaDriverUI | `apps/pos/GrandezaDriverUI.jsx` | mapa / mapa + lista apilados / 1 col + barra inferior |
| 7 | RepartoPanGrandezaUI | `apps/pos/RepartoPanGrandezaUI.jsx` | grid / grid reducido / 1 col scroll |

### 5.3 §4 Modales (6)

| # | Nombre | Anclaje | Modos |
|---|--------|---------|-------|
| 8 | CheckoutScreen | `apps/pos/components/CheckoutScreen.jsx` | modal 1100px + col lateral 320px / modal 800px / modal w-full 1 col |
| 9 | GestionPersonal | `apps/pos/components/GestionPersonal.jsx` | modal 800×600 + numpad 280px / modal 800px / modal w-full + numpad w-full |
| 10 | GestorDeCaja | `apps/pos/components/GestorDeCaja.jsx` | modal + teclado 380px / teclado w-full bajo contenido / 1 col + barra inferior |
| 11 | ProgramacionPedidoModal | `apps/pos/components/ProgramacionPedidoModal.jsx` | modal centrado / modal w-full / modal w-full 1 col |
| 12 | TerminalSelector | `apps/pos/components/TerminalSelector.jsx` | grid de terminales / grid reducido / 1 col scroll |
| 13 | OpenAccountsCorkboard | `apps/pos/components/OpenAccountsCorkboard.jsx` | corcho en grid / grid reducido / 1 col scroll |

### 5.4 §5 Paneles y overlays (5)

| # | Nombre | Anclaje | Modos |
|---|--------|---------|-------|
| 14 | SalesReceipt | `apps/pos/components/SalesReceipt.jsx` | panel lateral 420px / panel colapsable / panel inferior deslizable |
| 15 | POSHeader | `apps/pos/components/POSHeader.jsx` | header completo / header compacto / header reducido + barra inferior |
| 16 | POSOverlays | `apps/pos/components/POSOverlays.jsx` | 3 overlays / overlays fluidos / overlays fluidos + banner inferior |
| 17 | VisionVisor | `apps/pos/components/VisionVisor.jsx` | visor a pantalla completa / visor reducido / visor a pantalla completa |
| 18 | VisionScanner | `apps/pos/VisionScanner.jsx` | escáner con estados / escáner reducido / escáner a pantalla completa |

> **Nota de conteo:** `POSOverlays.jsx` exporta 3 overlays (`ForceLogoutModal`, `OfflineBanner`, `ToastNotification`) que se cuentan como un solo archivo.

### 5.5 §6 Composición (4)

| # | Nombre | Anclaje | Modos |
|---|--------|---------|-------|
| 19 | ProductGrid | `apps/pos/components/ProductGrid.jsx` | grid 4 col / grid 3 col / grid 2 col |
| 20 | ProductCard | `apps/pos/components/ProductCard.jsx` | tarjeta rounded-[35px] con cascada de 5 niveles / fluida / fluida |
| 21 | CategoryBar | `apps/pos/components/CategoryBar.jsx` | barra + botón ESCANER IA / scroll horizontal / scroll horizontal |
| 22 | CategoryEditor | `apps/pos/CategoryEditor.jsx` | editor / editor fluido / editor fluido 1 col |

> **Nota de conteo:** `CategoryEditor` vive en `apps/pos/` (no en `components/`) pero es composición del POS.

### 5.6 §7 Impresión (2) — EXENTAS de responsividad

| # | Nombre | Anclaje | Modos |
|---|--------|---------|-------|
| 23 | TicketTemplate | `apps/pos/components/TicketTemplate.jsx` | papel térmico 58mm/80mm — ancho físico fijo |
| 24 | CorteTicketTemplate | `apps/pos/components/CorteTicketTemplate.jsx` | papel térmico — ancho físico fijo |

Las interfaces 23–24 están **exentas de responsividad** (`exenta_responsiva=True`) porque el papel térmico tiene un ancho físico fijo. Su contenedor raíz usa `w-[58mm]` y `w-[80mm]`, que violarían R-01 si no estuvieran exentas. La exención es explícita y justificada: no es un descuido, es una decisión.

---

## 6. Los 10 contenedores críticos parametrizados

El Documento 6 §4.1 identificó 28 contenedores de ancho fijo a parametrizar. De ellos, **10 son CRÍTICOS** del POS transaccional (bloquean el cobro). Cada uno declara su ancho actual (rígido) y su ancho nuevo (fluido).

### 6.1 La tabla de los 10 críticos

| Archivo | Línea | Ancho actual (rígido) | Ancho nuevo (fluido) |
|---------|-------|-----------------------|----------------------|
| SalesReceipt.jsx | 41 | `w-[420px]` | `w-full max-w-[420px]` |
| CheckoutScreen.jsx | 132 | `w-[1100px]` | `w-full max-w-[1100px]` |
| CheckoutScreen.jsx | 133 | `w-[800px]` | `w-full max-w-[800px]` |
| CheckoutScreen.jsx | 322 | `w-[320px] flex-shrink-0` | `w-full lg:w-[320px] lg:flex-shrink-0` |
| GestionPersonal.jsx | 123 | `w-[800px] h-[600px]` | `w-full max-w-[800px] h-full max-h-[600px]` |
| GestionPersonal.jsx | 241 | `w-[280px]` | `w-full sm:w-[280px]` |
| GestorDeCaja.jsx | 853 | `w-[380px]` | `w-full lg:w-[380px]` |
| TableServicePOS.jsx | 134 | `w-[400px]` | `w-full lg:w-[400px]` |
| VisionTrainingUI.jsx | 80 | `w-[450px]` | `w-full lg:w-[450px]` |
| SalesReceipt.jsx | 206 | `w-[400px]` | `w-full max-w-[400px]` |

### 6.2 Los 3 patrones de parametrización

Los 10 contenedores se parametrizaron con 3 patrones:

1. **`w-full max-w-[Npx]`** — el contenedor ocupa todo el ancho disponible hasta un máximo. Usado en modales y paneles que deben encogerse en móvil pero no crecer sin límite en escritorio.
2. **`w-full lg:w-[Npx]`** — el contenedor ocupa todo el ancho en móvil, pero un ancho fijo en escritorio (≥1024px). Usado en columnas laterales que solo tienen sentido en MOSTRADOR.
3. **`w-full sm:w-[Npx]`** — el contenedor ocupa todo el ancho en móvil, pero un ancho fijo desde el breakpoint `sm`. Usado en el numpad de GestionPersonal.

### 6.3 Por qué estos 10 son críticos

Los 10 contenedores críticos son los que **bloquean el cobro** si no son responsivos. Un modal de checkout con ancho fijo de 1100px es inusable en una tablet de 800px. Un panel de ticket con ancho fijo de 420px no cabe en un teléfono de 375px. Estos 10 son la ruta crítica de la venta.

### 6.4 Los 18 contenedores no críticos

Los otros 18 contenedores (de los 28) son de pantallas de administración (Grandeza, reparto, configuración). No bloquean el cobro, pero también se parametrizaron para mantener la coherencia.

---

## 7. Los 6 flujos funcionales (paridad F5)

La puerta F5 exige **paridad funcional**: los 6 flujos del viejo POS se replican en el nuevo. Cada flujo declara sus interfaces y sus reglas de negocio.

### 7.1 E.1 — Venta directa (mostrador)

| Aspecto | Detalle |
|---------|---------|
| **Interfaces** | RetailVisionPOS, POSHeader, ProductGrid, ProductCard, CategoryBar, SalesReceipt, CheckoutScreen |
| **Reglas** | RN-14, RN-16, RN-18, RN-21, RN-22, RN-23, RN-25, RN-26, RN-27, RN-62, RN-63 |
| **Qué hace** | El flujo principal: elegir productos, armar el carrito, cobrar, imprimir. |

### 7.2 E.2 — Pedido (con proyección a Order)

| Aspecto | Detalle |
|---------|---------|
| **Interfaces** | RetailVisionPOS, SalesReceipt, ProgramacionPedidoModal |
| **Reglas** | RN-67, RN-68, RN-69, RN-70 |
| **Qué hace** | Convertir un ticket en un pedido programado (con datos de reparto). |

### 7.3 E.3 — Recuperación de cuenta

| Aspecto | Detalle |
|---------|---------|
| **Interfaces** | RetailVisionPOS, OpenAccountsCorkboard, SalesReceipt |
| **Reglas** | RN-31, RN-32, RN-33, RN-25, RN-26 |
| **Qué hace** | Recuperar una cuenta abierta del pizarrón y continuarla. |

### 7.4 E.4 — Guardado de emergencia

| Aspecto | Detalle |
|---------|---------|
| **Interfaces** | RetailVisionPOS, POSOverlays |
| **Reglas** | RN-40 |
| **Qué hace** | Guardar el ticket en curso cuando algo falla (red, cierre de pestaña). |

### 7.5 E.5 — Corte de caja

| Aspecto | Detalle |
|---------|---------|
| **Interfaces** | GestorDeCaja, CorteTicketTemplate |
| **Reglas** | RN-50, RN-51, RN-53, RN-54, RN-55, RN-57, RN-59, RN-60 |
| **Qué hace** | Cerrar el turno, contar el efectivo, imprimir el corte. |

### 7.6 E.6 — Ocupación de terminal

| Aspecto | Detalle |
|---------|---------|
| **Interfaces** | TerminalSelector, POSHeader, POSOverlays |
| **Reglas** | RN-03, RN-04, RN-06, RN-07, RN-08 |
| **Qué hace** | Tomar/liberar una terminal, gestionar el lock. |

### 7.7 La regla de los 6 flujos

> **Los 6 flujos son la paridad funcional con el viejo POS. Si un flujo no se replica, la puerta F5 falla.**

Los 6 flujos no son "features": son el contrato de paridad. El nuevo POS puede tener MÁS que el viejo, pero no puede tener MENOS en estos 6 flujos.

---

## 8. Las interfaces descartadas (y por qué)

No todo lo del viejo POS se portó. Algunas interfaces se descartaron deliberadamente. Documentarlas es tan importante como documentar las que se conservaron: evita que una futura IA las "rescate" por error.

### 8.1 `usePOSSession` — NO PORTAR (decisión D-12)

El viejo POS tenía un hook `usePOSSession` que gestionaba la sesión del POS. El nuevo POS **NO lo porta**. La decisión D-12 es final y no negociable.

**Qué hacía `usePOSSession` en el viejo POS:**
- Cargaba categorías y productos.
- Gestionaba la categoría activa.
- Generaba números de cuenta.
- Mantenía el estado de la sesión en `localStorage`.

**Por qué el nuevo POS NO lo necesita:**
- La carga de catálogo es un contrato (`catalogo.productos_para_venta`, contrato 1).
- La categoría activa es estado local de la UI, no de la sesión.
- La generación de folios es del servidor (RN-10), no del cliente.
- El estado del POS NO vive en `localStorage` (prohibido).

**Qué pasaría si se portara:** se reintroduciría el acoplamiento entre la sesión y el estado del carrito, que fue la causa de varios bugs del viejo POS (Cuenta Fantasma, Ticket Secuestrado).

**Dónde vive ahora cada responsabilidad:**

| Responsabilidad del viejo `usePOSSession` | Dónde vive ahora |
|-------------------------------------------|------------------|
| Carga de catálogo | Contrato 1 (`catalogo.productos_para_venta`) |
| Categoría activa | Estado local de la UI (`useState`) |
| Generación de folios | Servidor (RN-10) |
| Estado de sesión | Contrato 9 (`caja.sesion_activa`) |
| Persistencia local | NO EXISTE (prohibido `localStorage`) |

### 8.2 Auto-save periódico — NO RECREAR

El viejo POS tenía un auto-save periódico (cada N segundos) que guardaba el estado del carrito. El nuevo POS **NO lo recrea**.

**Por qué:** el auto-save periódico es una deuda que oculta fallos. Si el guardado falla, el auto-save lo reintenta silenciosamente y el operador no se entera. El nuevo POS guarda de forma **explícita y transaccional** (contrato 29), no periódica.

**La alternativa:** el guardado de emergencia (flujo E.4) es explícito: el operador lo dispara o el sistema lo dispara ante un evento concreto (cierre de pestaña, pérdida de red). No hay un timer silencioso.

### 8.3 `localStorage` para estado del POS — PROHIBIDO

El viejo POS usaba `localStorage` para persistir el estado del POS. El nuevo POS lo **prohíbe**.

**Por qué:** `localStorage` es estado no transaccional, no auditable y no sincronizado. Un carrito en `localStorage` puede divergir del servidor (Cuenta Fantasma). El nuevo POS tiene una lista explícita de claves prohibidas (`FORBIDDEN_KEYS` en `state/sessionReset.js`).

### 8.4 Detección de robo de lock — NO PORTAR (D1)

El viejo POS tenía una detección de "robo de lock" (otra terminal tomaba el lock). El nuevo POS **NO la porta** porque el lock se gestiona por contrato (RN-03, RN-04) y el servidor es la fuente de verdad.

### 8.5 Intervalos configurables — NO PORTAR (ahora) (D3)

El viejo POS tenía intervalos configurables para el heartbeat y el auto-save. El nuevo POS usa intervalos fijos por ahora. Es una mejora futura, no una necesidad actual.

### 8.6 La regla de las interfaces descartadas

> **Lo que se descarta, se documenta. Una interfaz descartada sin documentar es una invitación a que una futura IA la "rescate" por error.**

Esta sección existe precisamente para eso: para que la próxima IA sepa que `usePOSSession`, el auto-save periódico y el `localStorage` NO se portaron por decisión, no por olvido.

---

## 9. El defecto del plano: 26 vs 24

El Documento 7 dice "26 interfaces" en su título, en §0.3 (tabla) y en §10 (cierre). Pero su contenido enumerado y trazable es de **24**.

### 9.1 La evidencia del defecto

| Fuente | Dice | Realidad |
|--------|------|----------|
| Título del Documento 7 | "26 interfaces" | 24 |
| §0.3 (tabla) | "Composición: 5" | 4 nombres listados |
| §6 (título) | "Fichas — Composición (4)" | 4 fichas (19–22) |
| §8 (matriz de trazabilidad) | — | 24 filas (01–24) |
| Fichas del documento | — | 24 fichas (FICHA 01 a FICHA 24) |
| README del plano | "7 + 6 + 5 + 4 + 2" | = 24 |

### 9.2 La aritmética del defecto

La fila "Composición | 5" de §0.3 lista solo 4 nombres. La columna suma `7 + 6 + 5 + 4 + 2 = 24`, no 26. El "26" es un **error aritmético del plano**.

### 9.3 La decisión del registro

> **El registro se ancla a las 24 interfaces documentadas (las que tienen ficha y fila de trazabilidad). NO se inventan interfaces para forzar el 26.**

Esta decisión es importante: el registro NO infla el conteo para que cuadre con el título. Se ancla a la realidad trazable. El defecto se documenta, no se corrige inventando.

### 9.4 La lección del defecto

La lección es la misma que la de la FASE 10.6.2: **la completitud del conjunto también es una compuerta.** Un plano que dice "26" pero enumera 24 tiene un defecto de completitud. El registro lo detecta y lo documenta en lugar de esconderlo.

---

## 10. El test de la puerta F5

La FASE 5 (Superficie) tiene una puerta de calidad que se ejecuta en CI. Su propósito es verificar que la superficie se construyó según las reglas.

### 10.1 Qué verifica

El test de la puerta F5 recorre el registro de interfaces y verifica:

1. **Las 24 interfaces existen.** El conteo por tipo suma 24.
2. **Ninguna tiene ancho fijo en su contenedor raíz (R-01).** Salvo las exentas (impresión).
3. **La paleta canónica se respeta.** Los colores usados están en la paleta.
4. **Los 3 modos se declaran (R-03).** Cada interfaz declara MOSTRADOR, COMPACTO y MÓVIL.

### 10.2 Las funciones de verificación

El registro expone funciones que el test usa:

| Función | Qué devuelve |
|---------|--------------|
| `listar_interfaces()` | Las 24 interfaces, en orden. |
| `matriz_interfaz_tipo()` | La matriz `interfaz → tipo`. |
| `conteo_por_tipo()` | Cuántas interfaces hay de cada tipo (debe sumar 24). |
| `interfaces_con_ancho_fijo()` | Las que violan R-01 (excluye las exentas). |
| `interfaces_que_no_declaran_los_3_modos()` | Las que violan R-03. |

### 10.3 El código del test (conceptual)

```python
def test_puerta_f5_las_24_interfaces_existen():
    """La puerta F5: el registro tiene exactamente 24 interfaces."""
    assert len(listar_interfaces()) == 24
    assert sum(conteo_por_tipo().values()) == 24

def test_puerta_f5_ninguna_interfaz_con_ancho_fijo():
    """R-01: ninguna interfaz (salvo las exentas) fija el ancho raíz."""
    violadoras = interfaces_con_ancho_fijo()
    assert violadoras == (), (
        f"Interfaces que violan R-01: {[i.nombre for i in violadoras]}"
    )

def test_puerta_f5_todas_declaran_los_3_modos():
    """R-03: cada interfaz declara MOSTRADOR, COMPACTO y MÓVIL."""
    violadoras = interfaces_que_no_declaran_los_3_modos()
    assert violadoras == (), (
        f"Interfaces que no declaran los 3 modos: {[i.nombre for i in violadoras]}"
    )
```

### 10.4 Por qué es una puerta y no un test cualquiera

La puerta F5 es una **compuerta de arquitectura visual**: no verifica una función, verifica una PROPIEDAD de la superficie. Si alguien introduce un contenedor con ancho fijo (por ejemplo, para "que se vea bien" en su pantalla), la puerta lo detiene. Es el guardián automatizado de R-01 y R-03.

### 10.5 La lección de la puerta F5

La lección es la misma que la de la puerta F2 (Tomo IV): **la superficie no se verifica mirándola, se verifica ejecutando una puerta.** Un componente que "se ve bien" en la pantalla del desarrollador puede violar R-01 en una tablet. La puerta F5 lo comprueba mecánicamente.

---

## 11. Matriz de trazabilidad: interfaz → tipo → anclaje → modos

Esta matriz es el mapa completo de la superficie del POS. Cada fila es una interfaz; cada columna, su lugar en el sistema.

| # | Interfaz | Tipo | Anclaje (POS viejo) | Declara 3 modos | Ancho fijo |
|---|----------|------|---------------------|-----------------|------------|
| 1 | RetailVisionPOS | Pantalla raíz | `apps/pos/RetailVisionPOS.jsx` | Sí | No |
| 2 | TableServicePOS | Pantalla raíz | `apps/pos/TableServicePOS.jsx` | Sí | No |
| 3 | VisionTrainingUI | Pantalla raíz | `apps/pos/VisionTrainingUI.jsx` | Sí | No |
| 4 | GrandezaParamsUI | Pantalla raíz | `apps/pos/GrandezaParamsUI.jsx` | Sí | No |
| 5 | GrandezaDailyUI | Pantalla raíz | `apps/pos/GrandezaDailyUI.jsx` | Sí | No |
| 6 | GrandezaDriverUI | Pantalla raíz | `apps/pos/GrandezaDriverUI.jsx` | Sí | No |
| 7 | RepartoPanGrandezaUI | Pantalla raíz | `apps/pos/RepartoPanGrandezaUI.jsx` | Sí | No |
| 8 | CheckoutScreen | Modal | `apps/pos/components/CheckoutScreen.jsx` | Sí | No |
| 9 | GestionPersonal | Modal | `apps/pos/components/GestionPersonal.jsx` | Sí | No |
| 10 | GestorDeCaja | Modal | `apps/pos/components/GestorDeCaja.jsx` | Sí | No |
| 11 | ProgramacionPedidoModal | Modal | `apps/pos/components/ProgramacionPedidoModal.jsx` | Sí | No |
| 12 | TerminalSelector | Modal | `apps/pos/components/TerminalSelector.jsx` | Sí | No |
| 13 | OpenAccountsCorkboard | Modal | `apps/pos/components/OpenAccountsCorkboard.jsx` | Sí | No |
| 14 | SalesReceipt | Panel | `apps/pos/components/SalesReceipt.jsx` | Sí | No |
| 15 | POSHeader | Panel | `apps/pos/components/POSHeader.jsx` | Sí | No |
| 16 | POSOverlays | Overlay | `apps/pos/components/POSOverlays.jsx` | Sí | No |
| 17 | VisionVisor | Panel | `apps/pos/components/VisionVisor.jsx` | Sí | No |
| 18 | VisionScanner | Panel | `apps/pos/VisionScanner.jsx` | Sí | No |
| 19 | ProductGrid | Composición | `apps/pos/components/ProductGrid.jsx` | Sí | No |
| 20 | ProductCard | Composición | `apps/pos/components/ProductCard.jsx` | Sí | No |
| 21 | CategoryBar | Composición | `apps/pos/components/CategoryBar.jsx` | Sí | No |
| 22 | CategoryEditor | Composición | `apps/pos/CategoryEditor.jsx` | Sí | No |
| 23 | TicketTemplate | Impresión | `apps/pos/components/TicketTemplate.jsx` | Sí | Sí (exenta) |
| 24 | CorteTicketTemplate | Impresión | `apps/pos/components/CorteTicketTemplate.jsx` | Sí | Sí (exenta) |

### 11.1 Lectura de la matriz

- **24 interfaces** en total.
- **7 pantallas raíz, 6 modales, 5 paneles/overlays, 4 composición, 2 impresión.**
- **Todas declaran los 3 modos** (R-03 cumplida).
- **Solo 2 tienen ancho fijo** (las de impresión), y están **exentas** de R-01 porque el papel térmico tiene ancho físico fijo.

### 11.2 Las interfaces exentas de responsividad

Las únicas 2 interfaces con ancho fijo son las de impresión (23–24). Su exención es explícita (`exenta_responsiva=True`) y justificada: el papel térmico mide 58mm u 80mm, no se adapta al viewport. La exención NO es un descuido: es una decisión documentada.

### 11.3 La regla de oro de la matriz

> **Toda interfaz declara los 3 modos. Ninguna fija el ancho raíz (salvo las de impresión, exentas). La superficie se respeta en el papel (registro) y en la práctica (puerta F5).**

---

## Cierre del Tomo V

Este tomo documentó la **superficie** del POS nuevo: las 24 interfaces que el operador toca, la paleta canónica que las identifica, las 4 reglas duras de responsividad que las gobiernan, los 10 contenedores críticos que se parametrizaron, los 6 flujos funcionales que la puerta F5 exige, y las interfaces que se descartaron.

Los tres principios que la gobiernan:

1. **La identidad visual es intocable:** la paleta canónica y los radios no cambian.
2. **La responsividad es real:** los 3 modos son explícitos (R-03), sin anchos fijos (R-01), con tipografía que escala (R-02) y targets táctiles de 44×44px (R-04).
3. **La paridad funcional es un contrato:** los 6 flujos del viejo POS se replican (E.1–E.6).

La superficie no es decoración: es la implementación visible de la arquitectura. Un panel que lee una tabla ajena viola A-02 aunque se vea bien. Un contenedor con ancho fijo viola R-01 aunque funcione en la pantalla del desarrollador. La puerta F5 lo verifica mecánicamente.

**El Tomo VI (Acta de obra) documenta las 84 fichas de construcción: el registro de cómo se levantó esta superficie, fase por fase.**