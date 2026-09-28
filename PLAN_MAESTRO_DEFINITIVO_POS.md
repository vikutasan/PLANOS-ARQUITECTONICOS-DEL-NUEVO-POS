# 🏗️ PLAN MAESTRO DEFINITIVO — POS Nuevo "R de Rico"

> **Fecha:** 28 Sep 2026  
> **Versión del plan:** 1.0  
> **Autor:** Antigravity + Víctor (dueño de R de Rico)

### Repositorios del proyecto

Este proyecto vive en **dos repositorios** separados con roles distintos:

| Repositorio | URL | Contenido |
|---|---|---|
| **NUEVO-POS** | [github.com/vikutasan/NUEVO-POS](https://github.com/vikutasan/NUEVO-POS) | El **código fuente** del POS nuevo: frontend (React/Vite), backend (FastAPI), theme engine, tests, configuración |
| **PLANOS-ARQUITECTÓNICOS** | [github.com/vikutasan/PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS](https://github.com/vikutasan/PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS) | La **documentación arquitectónica**: planes, auditorías, decisiones de diseño, especificaciones, diagramas — los planos del edificio |

> [!NOTE]
> El código va en `NUEVO-POS`. Los planos van en `PLANOS-ARQUITECTONICOS`. Si estás escribiendo código, estás en el repo equivocado si no es `NUEVO-POS`. Si estás documentando arquitectura, estás en el repo equivocado si no es `PLANOS-ARQUITECTONICOS`.

---

## 1. OBJETIVO CENTRAL

**Construir la versión mejorada del módulo "Punto de Venta IA" de R de Rico**, de forma que sin perder la funcionalidad actual e incorporando otras nuevas, parezca que este módulo se diseñó desde un principio por un arquitecto senior.

Esta nueva versión:
- **Incorpora el aprendizaje documentado** de haber sido probado en operación real (7 meses, 6+ terminales, 30,000+ tickets)
- **Nace tomando en cuenta las cicatrices** documentadas en la [DOCUMENTACIÓN MAESTRA del POS](https://github.com/vikutasan/ERP-R-DE-RICO-CON-POS-SIMPLIFICADO/blob/main/ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_MODULO_POS.md)
- **Produce un POS robusto** fraguado en la batalla pero con calidad de arquitectura senior
- **Sienta las bases** del nuevo ERP que se construirá después (sistema de temas, contratos, motor compartido)

---

## 2. REGLA DURA INQUEBRANTABLE

> [!CAUTION]
> **PROHIBIDO tocar, modificar o interrumpir el ERP de R de Rico que corre en este servidor.**
> 
> - El ERP viejo corre en `localhost:5000` (frontend) y `localhost:5001` (API) con PostgreSQL en `5433`
> - El POS nuevo corre en `localhost:5100` (frontend) y `localhost:5101` (API) con su **propia** base de datos
> - **Cero dependencias cruzadas.** Este proyecto corre 100% en paralelo sin estorbar al otro
> - El POS nuevo tiene su propio repo (`NUEVO-POS`), su propio `package.json`, su propio backend
> - Cuando esté listo, **reemplazará** al viejo. Hasta entonces, no lo toca.

---

## 3. DIAGNÓSTICO: QUÉ HIZO DEEPSEEK Y QUÉ ENCONTRAMOS

### Lo que DeepSeek construyó (~20% del módulo POS)

| Categoría | Archivos | Estado |
|---|---|---|
| `RetailVisionPOS.jsx` | 1 (11 KB vs 40 KB del viejo) | Simplificado: flujo básico catálogo→ticket→cobro |
| Componentes | 5 de 12 (CategoryBar, ProductCard, ProductGrid, SalesReceipt parcial, CheckoutScreen parcial) | Funcionales pero sin las funciones reales |
| Hooks | 1 (`useModo.js` — nuevo, no existe en el viejo) | 0 de 9 hooks del viejo |
| Servicios | 0 de 2 | Nada |
| Utils | 0 de 3 | Nada |
| State | 0 de 1 | Nada |
| Config | 1 | ✅ Completo |
| API backend | Endpoints básicos de catálogo y sesión | Faltan terminales, caja, tickets atómicos |

### Lo que DeepSeek NO construyó (~80% del módulo POS)

- ❌ **Selector de Terminales** (landing page — 31 KB)
- ❌ **Gestor de Caja** (cortes, arqueos, turnos — 56 KB)
- ❌ **Panel de Voz** (agregar por voz — 14.7 KB)
- ❌ **Visor de Cámara IA** (reconocimiento visual — 4.4 KB)
- ❌ **Programación de Pedidos** (pedidos programados — 21.6 KB)
- ❌ **Pizarrón de Cuentas** (cuentas abiertas en paralelo — 8.4 KB)
- ❌ **9 hooks** del POS viejo (carrito, sesión, terminal lock, voz, visión, red, escáner, beforeunload, ticket actions)
- ❌ **Checkout completo** (solo tiene 5.5 KB de 28.7 KB)
- ❌ **Persistencia atómica** (modelo SaaS v6.0)
- ❌ **Contratos de resultado discriminado** (v7.0.3)

### Lo que NOSOTROS construimos (arquitectura del edificio)

- ✅ **Theme Engine** — motor de temas compartido con 3 variantes (38 tests)
- ✅ **Contratos de módulo** — sistema de validación WCAG AA
- ✅ **Tokens CSS** — 6 base + 3 madera, con variables en runtime
- ✅ **Estética replicada** — header, tarjetas, ticket, categorías como el POS viejo
- ✅ **Extractor IA de branding** — endpoint para extraer tokens desde imágenes
- ✅ **Tailwind configurado** — con tokens semánticos en vez de hex sueltos

### Cómo lo corregimos

La arquitectura (cimientos) está sólida. Las funciones (habitaciones) se construyen sobre ella respetando las lecciones de la documentación de batalla.

---

## 4. LAS 5 PROHIBICIONES ABSOLUTAS (del cementerio de bugs)

Extraídas de 7 meses de operación real. **Toda línea de código del POS nuevo las respeta:**

> [!WARNING]
> 1. **NO** reintroducir auto-save, timers ni `setInterval` para guardar el carrito. La persistencia es **atómica por ítem** (v6.0)
> 2. **NO** hacer `clearCart()` sin confirmación HTTP 200 del servidor **Y** verificación post-envío (v6.1)
> 3. **NO** leer variables de estado (`cart`, `currentAccountNum`) dentro de callbacks asíncronos — usar siempre `useRef` (Ticket #906, $124→$2)
> 4. **NO** almacenar candados de terminal en RAM de Python — solo en PostgreSQL (`terminal_locks`)
> 5. **NO** generar folios en el frontend — solo el backend los genera vía secuencia atómica de PostgreSQL

---

## 5. REGLAS ARQUITECTÓNICAS DERIVADAS DE LA BATALLA

| Regla | Origen | Implementación |
|---|---|---|
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

## 6. FUNCIONES NUEVAS (que el POS viejo NO tiene)

### 6.1 Sistema de Temas (ya construido)

| Función | Estado | Descripción |
|---|---|---|
| Motor de temas compartido | ✅ Listo | `packages/theme-engine/` — cambia colores en runtime sin recompilar |
| 3 temas del POS | ✅ Listo | R de Rico Classic (default), Nocturno, Minimal |
| Tokens CSS semánticos | ✅ Listo | `bg-acento`, `text-crema-ticket` en vez de `#c1d72e` suelto |
| Validación WCAG AA | ✅ Listo | 38 tests automáticos de contraste |
| Selector de temas UI | 🔲 Fase 7 | Interfaz para que el cajero elija tema |
| Compatibilidad ERP futuro | ✅ Diseñado | El motor servirá para todos los módulos del nuevo ERP |

### 6.2 Exportar Catálogo a PDF (nueva)

> **Justificación del dueño:** "Para poder cobrar si se va la luz o el sistema, necesitamos tener una versión analógica del catálogo de productos con precios."

| Función | Descripción |
|---|---|
| Botón "📄 Exportar PDF" | Visible en la barra de categorías |
| Contenido del PDF | Grid de productos tal cual aparece en la interfaz: imagen + nombre + precio |
| Agrupación | Por categoría (como las pestañas del POS) |
| Formato | Optimizado para impresión en carta/oficio |
| Actualización | Se genera en el momento, siempre con precios vigentes |

---

## 7. FASES DE CONSTRUCCIÓN

### Fase 1 — Selector de Terminales (LA LANDING)
> **Sin esto no puedes ni abrir el POS**

**Archivos a construir:**
- `TerminalSelector.jsx` — tarjetas de terminales con 3 estados (libre/mío/ajeno)
- `useTerminalLocking.js` — heartbeat + locks en PostgreSQL + cleanup
- `terminalService.js` — API de terminales
- Ruta `/` → selector, ruta `/pos?terminal=X` → POS

**Lecciones integradas:**
- 3 estados visuales de terminal (v12: libre/mío/ajeno)
- Lock release vía `sendBeacon` al cerrar pestaña (H3)
- Heartbeat con purga de duplicados (incidente OMEGA)
- `utcnow()` para timestamps de lock (H2)

**Criterio de aceptación:**
- Abres `localhost:5100/` → ves tarjetas de terminales
- Clic en terminal → entras al POS
- Terminal queda "ocupada", muestra quién la tiene
- Al cerrar pestaña → lock se libera

---

### Fase 2 — Sesión y Control de Acceso
> **Sin esto cualquiera opera la caja**

**Archivos a construir:**
- `usePOSSession.js` — catálogo, folio, inicialización
- `securityService.js` — validación de roles (ADMIN/MANAGER/CAJERO)
- Flujo de login integrado con el selector

**Lecciones integradas:**
- Primitivos en deps (`currentUser?.id`, no el objeto) (H1)
- Folio generado SOLO por el backend (prohibición #5)

---

### Fase 3 — POS Completo (el corazón)
> **Completar lo que DeepSeek dejó a medias**

**Archivos a construir:**
- `useCart.js` — hook del carrito con persistencia atómica
- `useTicketActions.js` — acciones del ticket con contrato discriminado
- `CheckoutScreen.jsx` — checkout completo (efectivo, tarjeta, cambio)
- `SalesReceipt.jsx` — edición de cantidad, banner de estado
- `POSHeader.jsx` — funciones reales (estado cuenta, tipo venta, red)
- `POSOverlays.jsx` — modales de confirmación
- `useBeforeUnload.js` — protección de cierre con sendBeacon
- `useBarcodeScanner.js` — lector de código de barras
- `useNetworkHealth.js` — monitor de red
- `withRetries.js` — utilidad centralizada de reintentos
- `sessionReset.js` — limpieza espejo de estado
- `POSService.js` — API del POS (endpoints atómicos)

**Lecciones integradas:**
- Persistencia atómica por ítem (v6.0 SaaS) — NUNCA auto-save masivo
- `clearCart()` solo con HTTP 200 + verificación post-envío (v6.1)
- `useRef` para async callbacks (Ticket #906)
- Contrato `{ outcome, reason }` en handleTicketAction (v7.0.3)
- withRetries con 3 intentos y backoff (v7.0.1)
- Respuesta ligera para operaciones atómicas (v7.0)
- Banner rojo fijo + botón bloqueado cuando hay error de red (v6.1)
- Espejo de limpieza completo en toda rama de salida (v7.0.3)

---

### Fase 4 — Gestor de Caja
> **Cortes de caja, arqueos, turnos**

**Archivos a construir:**
- `GestorDeCaja.jsx` — pantalla completa
- `cashService.js` — API de caja
- `CorteTicketTemplate.jsx` — plantilla de impresión de corte

**Lecciones integradas:**
- CashSession con TTL (incidente OMEGA: sesiones sin cerrar > 24h)
- Lock de terminal asociado a CashSession

---

### Fase 5 — Pizarrón de Cuentas Abiertas
> **Varias cuentas en paralelo**

**Archivos a construir:**
- `OpenAccountsCorkboard.jsx` — pizarrón visual
- Lógica multi-cuenta (ya incluida en useTicketActions de Fase 3)

**Lecciones integradas:**
- Recuperación de cuenta = descargar versión fresca del servidor (v6.0)
- Terminal original NO se sobreescribe al cobrar desde otra terminal (incidente T5/CAJA)

---

### Fase 6 — Impresión + PDF de Catálogo
> **Sin esto no das ticket al cliente. El PDF es la novedad.**

**Archivos a construir:**
- `TicketTemplate.jsx` — plantilla de venta
- `ticketGenerator.js` — generador de HTML para impresión
- **`CatalogoPDF.jsx`** — ✨ NUEVO: genera PDF del grid de productos

**Funciones nuevas (PDF de catálogo):**
- Botón "📄 Exportar PDF" en la barra de categorías
- Genera un PDF con: nombre del negocio, fecha, y cada categoría con sus productos (imagen + nombre + precio)
- Formato optimizado para impresión carta/oficio
- Para cobrar manualmente si se va la luz o el sistema

---

### Fase 7 — Voz + Visión IA + Selector de Temas
> **Las funciones estrella + la UI del tema**

**Archivos a construir:**
- `VoiceCartPanel.jsx` — panel de voz
- `useVoiceCart.js` — hook de voz
- `voiceCartMapper.js` — mapeo de intents a productos
- `VisionVisor.jsx` — visor de cámara IA
- `useVision.js` — hook de visión
- `ProgramacionPedidoModal.jsx` — pedidos programados
- `ThemeSelector.jsx` — ✨ selector visual de temas (Fase 4 del theme engine)

---

## 8. DECISIONES ARQUITECTÓNICAS

| Decisión | Elección | Por qué |
|---|---|---|
| **Estado global** | React Context + useReducer | Suficiente para el POS. No necesita Redux/Zustand. Hooks propios como puente. |
| **Ruteo** | React Router v6 (`/` = selector, `/pos` = POS, `/caja` = gestor) | El POS viejo usaba un switch monolítico. Rutas = habitaciones separadas. |
| **Persistencia** | Modelo SaaS v6.0 (atómica por ítem) | Probado en batalla. Elimina race conditions y closures viejos. |
| **API** | FastAPI propia en puerto 5101 | Separada del ERP. Misma arquitectura que el viejo pero limpia desde cero. |
| **Tests** | Vitest para frontend, pytest para backend | Cada hook y servicio tiene tests. Contratos guardianes obligatorios. |
| **PDF** | html2pdf.js o jsPDF | Genera PDF en el navegador sin dependencias de servidor. |
| **Impresión** | window.print() con template HTML | Igual que el POS viejo. Funciona con térmicas de 80mm. |

---

## 9. RESULTADO ESPERADO

Cuando las 7 fases estén completas, al abrir `localhost:5100/` verás:

1. **Landing** → Selector de terminales (tarjetas con estado libre/mío/ajeno)
2. **Login** → El cajero se identifica
3. **POS** → Catálogo + ticket + checkout (idéntico al viejo pero limpio)
4. **Gestor de caja** → Abrir/cerrar turno, cortes
5. **Pizarrón** → Cuentas abiertas en paralelo
6. **Voz** → "Agrega dos baguettes" → aparecen en el ticket
7. **Cámara** → Reconoce productos en la charola
8. **PDF** → Exporta catálogo para cobro manual sin sistema
9. **Temas** → Elige entre 3 estilos visuales

**Todo construido sobre la arquitectura de un edificio planificado, no uno remendado.**

---

## 10. HERENCIA VALIOSA DE LOS PLANES ANTERIORES

Los siguientes puntos provienen de los planes de DeepSeek (`PLAN_DE_CONSTRUCCION`, `PLAN_ACCION_ARQUITECTONICO`, `PLAN_DE_IMPLEMENTACION_UI`) y son demasiado valiosos para perderse. Quedan incorporados a este Plan Maestro:

### 10.1 Las 81 reglas de negocio se portan CON su test (A-01)

> **Origen:** PLAN_ACCION_ARQUITECTONICO, Acción A-01

La unidad de migración **no es la regla, es la regla + su prueba**. Cada una de las 81 reglas (RN-01 a RN-81) documentadas en `ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md` se implementa con su test correspondiente.

**Reglas sin test hoy (riesgo de pérdida):** RN-37 a RN-40 (anti-degradación), RN-44 a RN-48 (limpieza de borradores), RN-81 (cero offsets). Se les escribe test ANTES de implementar.

### 10.2 Frontera por contratos — Prohibición de leer tablas ajenas (A-02)

> **Origen:** PLAN_ACCION_ARQUITECTONICO, Acción A-02

El POS nuevo **no importa** modelos de otros módulos. Cada dependencia se resuelve por contrato explícito. Los 10 acoplamientos (AC-01 a AC-10) del POS viejo se reemplazan por contratos definidos en `CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md`.

**Test de arquitectura:** falla si el POS importa un modelo ajeno.

### 10.3 Estándares de base de datos (C-01 a C-04)

> **Origen:** PLAN_DE_CONSTRUCCION, Fase 1

| Estándar | Regla |
|---|---|
| **C-01** | PK = UUID en todas las tablas (no enteros autoincrementales) |
| **C-02** | `DateTime(timezone=True)` UTC siempre (nunca naive) |
| **C-03** | Dinero en `Numeric(12,2)` (NUNCA `Float`) |
| **C-04** | Columna `version` para bloqueo optimista |

### 10.4 Los 5 greps de CI que se ejecutan desde el día 1

> **Origen:** PLAN_DE_CONSTRUCCION, Fase 0, §2.3

Estos 5 greps se ejecutan en cada push. Si alguno encuentra una violación, **el build falla**:

1. `except.*pass` → prohibido en ruta crítica
2. `console.log` → prohibido en producción
3. `TODO` sin formato → prohibido (usar `// TODO(nombre): descripción`)
4. `Float` en modelos de dinero → siempre `Numeric(12,2)`
5. `DateTime()` naive → siempre `DateTime(timezone=True)`

### 10.5 Matriz de trazabilidad regla → test

> **Origen:** PLAN_ACCION_ARQUITECTONICO, A-01 y A-03

Cada fase produce una **matriz verificable** donde se puede rastrear: para cada regla de negocio, ¿cuál es su test? Las reglas sin test se marcan como **riesgo de pérdida**. El CI falla si una regla crítica no tiene guardián.

### 10.6 El principio "de adentro hacia afuera"

> **Origen:** PLAN_DE_CONSTRUCCION, §0.2

> *Se construye de adentro hacia afuera: primero el cimiento (datos), luego la frontera (contratos), luego el comportamiento (reglas + tests), luego la superficie (interfaces).*

Nuestras 7 fases respetan este principio: la Fase 1 (Terminal Selector) requiere el endpoint de terminales (dato + contrato) antes de la interfaz. Cada fase interna tiene este orden: endpoint → hook → componente.

### 10.7 Documentos de referencia obligatoria por fase

| Antes de construir... | Consultar... |
|---|---|
| Cualquier pantalla | [ESPECIFICACION_DE_INTERFACES_POS.md](./ESPECIFICACION_DE_INTERFACES_POS.md) (las 26 fichas) |
| Cualquier componente | [ESPECIFICACION_RESPONSIVA_Y_ERGONOMIA_TACTIL.md](./ESPECIFICACION_RESPONSIVA_Y_ERGONOMIA_TACTIL.md) (R-01 a R-04) |
| Cualquier regla de negocio | [ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md](./ESPECIFICACIONES%20DEL%20PROYECTO/ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md) (RN-01 a RN-81) |
| Cualquier tabla/endpoint | [MODELO_DE_DATOS_DEL_NUEVO_POS.md](./MODELO_DE_DATOS_DEL_NUEVO_POS.md) |
| Cualquier frontera entre módulos | [CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md](./CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md) |

---

> [!IMPORTANT]
> **Este plan (v1.1) integra lo valioso de los 3 planes anteriores. Los documentos originales quedan marcados como SUPERSEDED con referencia a este plan.**

