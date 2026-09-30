# 🏗️ PLAN MAESTRO DEFINITIVO — POS Nuevo "R de Rico"

> **Fecha:** 28 Sep 2026 (última actualización: 30 Sep 2026)
> **Versión del plan:** 2.3
> **Autor:** Antigravity + Víctor (dueño de R de Rico)
>
> **Cambios v2.3:** Se añade **§8.2 "El ERP viejo existe, pero su arquitectura es
> parchada — se rehará módulo por módulo"**. Corrige el supuesto más peligroso del
> plan: los módulos del ERP **no están vacíos** (Estadísticas, CRM, Almacenes, RRHH,
> etc. existen y funcionan), pero se construyeron **sobre la marcha** con arquitectura
> parchada. El trabajo del dueño es rehacer cada módulo con el método del POS e
> integrarlos **uno a uno**. Regla: *"El ERP viejo es el ORÁCULO, no el MODELO."*
> Aplicación inmediata: F10.4 (Contexto diario) se acopla por contrato al módulo
> Estadísticas que **ya existe**; "profesionalizar/acoplar Estadísticas" es **F11**.
>
> **Cambios v2.2:** Fase **9.1 "Pagos mixtos"** cerrada. Un ticket puede cobrarse
> con varios métodos a la vez (efectivo + tarjeta + transferencia). El arqueo
> ahora suma **solo la parte en efectivo** (RN-53) — antes sumaba el total aunque
> parte fuera con tarjeta. Se añaden RN-94 (la suma de pagos cuadra el total, sin
> tolerancia) y RN-95 (métodos válidos). Se documenta en §7 (Fase 9.1) la lección
> de INTEGRACIÓN: el dato se persistía pero `TicketSalida` no lo exponía — misma
> clase de defecto que el `GestorDeCaja` huérfano (F4.5).
>
> **Cambios v2.1:** Micro-fase correctiva **F4.5 "Montaje del Gestor de Caja"**
> cerrada. Se documenta en §10.6.1 la lección de INTEGRACIÓN: *"el componente
> existe y pasa su test" ≠ "el usuario puede llegar a él"*. El `GestorDeCaja`
> era un componente huérfano; ahora tiene punto de entrada (botón "Caja" en el
> header) y guarda de cobro proactiva (RN-49).

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

## 2. REGLAS DURAS INQUEBRANTABLES

> [!CAUTION]
> **REGLA DURA 1 — PROHIBIDO tocar, modificar o interrumpir el ERP de R de Rico que corre en este servidor.**
>
> - El ERP viejo corre en `localhost:5000` (frontend) y `localhost:5001` (API) con PostgreSQL en `5433`
> - El POS nuevo corre en `localhost:5100` (frontend) y `localhost:5101` (API) con su **propia** base de datos
> - **Cero dependencias cruzadas.** Este proyecto corre 100% en paralelo sin estorbar al otro
> - El POS nuevo tiene su propio repo (`NUEVO-POS`), su propio `package.json`, su propio backend
> - Cuando esté listo, **reemplazará** al viejo. Hasta entonces, no lo toca.

> [!CAUTION]
> **REGLA DURA 2 — VERIFICAR, NO ASUMIR.**
>
> **Ninguna afirmación sobre el código, el esquema o el estado del sistema se escribe sin haberla verificado contra la fuente real.**
>
> - Un plan que **nombra** una tabla, un modelo, un campo, un contrato o un endpoint **debe verificar que existe** antes de nombrarlo. Si no existe, se declara como trabajo a construir — no se asume.
> - Un plan que **afirma** que una función hace algo (p. ej. "hace commit al final") **debe leer la función** y citar la línea. No se infiere por el nombre.
> - Un plan que **cuenta** campos, reglas o estados **debe abrir el contrato/registro** y contarlos. No se estima de memoria.
> - Cuando la verificación contradice el plan, **manda la verificación**. El plan se corrige; no se ejecuta sobre una suposición.
> - **Origen:** autocrítica del plan F7.5 (v3.0 → v3.1). El plan v3.0 nombró `system_settings` (que no existía), afirmó que `crear_ticket` era "misma transacción" (hacía `commit()` temprano), contó 9 campos donde el contrato exige 10, e ignoró el default de `order_status`. Los 4 defectos se detectaron **solo al verificar contra el código real**. La lección: *"Un plan que nombra una tabla debe verificar que la tabla existe."*
> - **Cómo se verifica:** con las herramientas de lectura (`read_file`, `search_files`, `list_files`) sobre el repo real, citando archivo y línea. La evidencia de la verificación se registra en la ficha de la fase.

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

## 4. LAS 6 PROHIBICIONES ABSOLUTAS (del cementerio de bugs)

Extraídas de 7 meses de operación real + 1 error de construcción del POS nuevo. **Toda línea de código del POS nuevo las respeta:**

> [!WARNING]
> 1. **NO** reintroducir auto-save, timers ni `setInterval` para guardar el carrito. La persistencia es **atómica por ítem** (v6.0)
> 2. **NO** hacer `clearCart()` sin confirmación HTTP 200 del servidor **Y** verificación post-envío (v6.1)
> 3. **NO** leer variables de estado (`cart`, `currentAccountNum`) dentro de callbacks asíncronos — usar siempre `useRef` (Ticket #906, $124→$2)
> 4. **NO** almacenar candados de terminal en RAM de Python — solo en PostgreSQL (`terminal_locks`)
> 5. **NO** generar folios en el frontend — solo el backend los genera vía secuencia atómica de PostgreSQL
> 6. **NO** duplicar módulos que el ERP ya tiene (login, gestión de empleados, perfiles). El POS es un MÓDULO del ERP, no una app suelta. Si el ERP ya lo resuelve, el POS lo recibe como prop — no lo reconstruye. *(Origen: se construyó un login propio y el dueño lo rechazó correctamente por YAGNI)*

---

## 5. REGLAS ARQUITECTÓNICAS DERIVADAS DE LA BATALLA

| Regla | Origen | Implementación |
|---|---|---|
| **Verificar, no asumir** | Autocrítica F7.5 v3.0→v3.1 (4 defectos) | Antes de nombrar una tabla/campo/contrato/endpoint en un plan, verificar que existe (archivo + línea). Antes de afirmar qué hace una función, leerla. Antes de contar campos/reglas/estados, abrirlos y contarlos. Si la verificación contradice el plan, manda la verificación |
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

### 6.3 Identificación de cliente al cobrar — CRM (nueva)

> **Especificación completa:** [PROPUESTA_CRM_Y_NOTIFICACIONES_DEL_NUEVO_POS.md](https://github.com/vikutasan/PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS/blob/main/PROPUESTA_CRM_Y_NOTIFICACIONES_DEL_NUEVO_POS.md)
> **Petición textual del dueño:** *"Llegará un momento en que tendremos una base de datos de clientes que nos será útil para administrar tarjetas de lealtad, enviar tickets, programar promociones y descuentos."*

| Función | Descripción |
|---|---|
| Botón "👤 Cliente" en el header | El cajero puede identificar al cliente en cualquier momento (antes o durante la carga de productos) |
| 3 caminos de identificación | **A:** Público general (no se identifica). **B:** Cliente registrado (teclea teléfono). **C:** Alta rápida (nombre + teléfono) |
| Teléfono como clave | Se normaliza con el criterio ya existente (`_normalizar_telefono()`) |
| Beneficios automáticos | El CRM decide qué descuento aplica. El POS nunca calcula descuentos, solo los muestra |
| Puntos de lealtad | Ledger inmutable (como inventario): se asientan, nunca se hacen UPDATE |
| Promociones configurables | "2x1 en conchas los martes". Ciclo de vida: BORRADOR → ACTIVA → VENCIDA/PAUSADA. Nunca se borran |
| Degradación elegante | Si el CRM cae, el POS cobra a precio de lista. **NUNCA bloquea la venta** |

**Dos tipos de beneficios:**
- **Tipo A (precio):** Se aplica ANTES de cobrar como línea negativa en el ticket (ej: "10% en pan dulce")
- **Tipo B (acumulable):** Se asienta DESPUÉS de cobrar en el ledger (ej: "1 punto por cada $10")

### 6.4 Envío de ticket por WhatsApp o Email (nueva)

> **Petición textual del dueño:** *"Deseo que el nuevo POS al momento de cobrar una cuenta me dé la opción de imprimir el ticket o enviarlo por WhatsApp o por e-mail."*

| Función | Descripción |
|---|---|
| Paso de entrega post-cobro | Después de "FINALIZAR VENTA" aparece: 🖨️ Imprimir / 📱 WhatsApp / ✉️ E-mail / Omitir |
| Imprimir siempre disponible | Es el comportamiento por defecto. WhatsApp/email son **adicionales**, nunca sustitutos |
| Patrón Outbox | El POS **encola** el mensaje en la misma transacción del ticket. El envío real lo hace un worker en segundo plano |
| NUNCA bloquea la venta | Si WhatsApp está caído, el ticket ya se cobró. El mensaje queda en cola y se reintenta |
| Idempotencia | Cada mensaje lleva `evento_id`. Un reintento no duplica el mensaje (constraint UNIQUE) |
| Plantillas | WhatsApp: texto plano con formato. Email: HTML reutilizando el generador de tickets |

**Arquitectura (2 módulos separados por contrato):**
- **Clientes (CRM):** Decide QUÉ beneficio aplica y A QUIÉN se le manda (tablas: `customers`, `loyalty_ledger`, `promotions`, `customer_benefits`)
- **Notificaciones:** Ejecuta CÓMO se entrega (tablas: `notification_outbox`, `notification_log`, `channel_config`)
- **2 contratos nuevos:** #18 `clientes.beneficios_para_ticket` y #19 `notificaciones.encolar_ticket`
- **8 tablas nuevas** (5 CRM + 3 Notificaciones)

### 6.5 Layout responsivo desde el día 1 (nueva)

> **Especificación completa:** [ESPECIFICACION_RESPONSIVA_Y_ERGONOMIA_TACTIL.md](https://github.com/vikutasan/PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS/blob/main/ESPECIFICACION_RESPONSIVA_Y_ERGONOMIA_TACTIL.md)

| POS viejo | POS nuevo |
|---|---|
| Anchos fijos: `w-[420px]`, `w-[1100px]`, `w-[800px]` | Anchos fluidos: `w-full max-w-[420px]` |
| Solo funciona en la tablet del mostrador | Funciona en cualquier tablet, laptop o pantalla |
| Cada sucursal con hardware distinto = "adaptación" manual | Se instala y funciona sin tocar CSS |

**4 reglas duras (R-01 a R-04):** estándar táctil 44×44px, estética del mostrador intocable, layout fluido obligatorio.

### 6.6 Consolidación central multi-sucursal (futura)

> **Especificación completa:** [MODELO_DESPLIEGUE_Y_CONSOLIDACION_CENTRAL.md](https://github.com/vikutasan/PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS/blob/main/ESPECIFICACIONES%20DEL%20PROYECTO/MODELO_DESPLIEGUE_Y_CONSOLIDACION_CENTRAL.md)

| POS viejo | POS nuevo |
|---|---|
| Una sola sucursal, sin capacidad de replicación | Diseñado para instalarse en N sucursales |
| No reporta a ningún servidor central | Sync al cierre del día (default 23:30) a servidor corporativo |

**Topología hub-and-spoke:** cada sucursal tiene su ERP completo (autónomo). Al cierre del día envía un resumen al servidor central. Si el central cae, las sucursales siguen operando.

> [!NOTE]
> La consolidación central es **fase futura** (no bloquea las fases 1–8). El POS se diseña DESDE AHORA con UUID como PK y outbox transaccional para que cuando se active la sync, no haya que rediseñar nada.

### 6.7 IA flexible — 3 modos de topología (nueva)

> **Especificación completa:** [ESPECIFICACION_IA_LOCAL_Y_MULTIMODAL.md](https://github.com/vikutasan/PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS/blob/main/07-ia-local/ESPECIFICACION_IA_LOCAL_Y_MULTIMODAL.md)

| POS viejo | POS nuevo |
|---|---|
| IA fija apuntando a un solo servidor | 3 modos seleccionables |

**3 modos de IA:**
- **M1 — Local por sucursal:** contenedor de IA junto al ERP local (para sucursales con GPU propia)
- **M2 — Local central:** un solo servidor de IA en la matriz, sirve a todas las sucursales
- **M3 — Nube:** servicio externo (para picos de demanda o modelos grandes)

**Regla de oro:** la IA NUNCA bloquea la venta. Si no está disponible, el POS funciona sin ella (fallback 503).

### 6.8 La UX heredada del viejo POS — la integración se hereda, la implementación se reescribe (nueva)

> **Origen:** hallazgo de F7.6 (29 Sep 2026). Al cablear voz/visión/temas en la pantalla real, se
> descubrió que el viejo POS **ya define cómo se integran** estos componentes. La primera versión
> del cableado los trataba como overlays flotantes y **contradecía** esa UX.

**El principio:**

> *Cuando un componente ya existe en el viejo POS, su **INTEGRACIÓN** se hereda; solo su
> **IMPLEMENTACIÓN** se reescribe. El viejo POS es la fuente de verdad para la integración;
> el nuevo POS lo es para la implementación.*

Es la **Regla Dura A-01** (portar con su test) aplicada a la UX: no se reinventa *cómo se usa* algo
que ya se usa bien; se reinventa *cómo se construye*.

| Componente | Integración heredada del viejo POS |
|---|---|
| **Visión** | **Modo de vista** (`viewMode === 'CAMERA'`) que **reemplaza el cuerpo**; se conmuta desde la barra de categorías. **No** es un overlay. |
| **Voz** | **Botón en el header** con `disabled={!vozDisponible}` → abre overlay de dictado. |
| **Tema** | **No existe** en el viejo POS → capacidad nueva → overlay nuevo. |

**Por qué importa para todo el ERP:** este principio gobierna la reconstrucción completa. Cuando un
módulo del viejo ERP ya define *cómo se usa*, el módulo nuevo **hereda ese uso** y solo reescribe
*cómo se construye*. Evita reinventar UX probada y evita romper la memoria muscular del operador.

**Verificación:** gate `RetailVisionPOS.f7_6.test.jsx` (11 criterios / 13 tests) + CI en verde.
Evidencia en `FICHA_F7_6_UX_HEREDADA.md`.

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

### Fase 2 — ~~Sesión y Control de Acceso~~ RESUELTA POR EL ERP
> **El login ya existe en el ERP. El POS NO lo duplica.**

> [!CAUTION]
> **DECISIÓN DEL DUEÑO (28 Sep 2026):** Se construyó un login propio (LoginScreen.jsx,
> useAuth.js, securityService.js) y el dueño lo rechazó correctamente:
> *"No le veo caso a tenerla ahora, ya que jamás la correré como app suelta."*
>
> **Regla resultante (prohibición #6):** NO duplicar módulos que el ERP ya tiene.
> El POS recibe `currentUser` como prop del ERP. No construye su propio login.
>
> Los archivos construidos se conservan en el repo como referencia pero NO se usan.

**Estado actual:**
- `App.jsx` usa un usuario demo hardcodeado (`{ id: 1, name: 'Victor', role: 'ADMIN' }`)
- TODO(integración): reemplazar por prop `currentUser` del ERP

**Lecciones integradas:**
- Primitivos en deps (`currentUser?.id`, no el objeto) (H1)
- Folio generado SOLO por el backend (prohibición #5)
- **YAGNI:** no construir lo que el ERP ya resuelve (prohibición #6)

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

> [!IMPORTANT]
> **Aclaración de alcance (29 Sep 2026):** la voz y la visión **no son del POS**. Son capacidades del
> **Centro de IA**, un módulo del ERP que aún no se reconstruye. Lo que esta fase construye es **solo
> el lado del POS**: los puntos de contacto para consumir esas capacidades **por contrato** cuando el
> Centro de IA exista. Ver **DT-07** en [`DIRECTRICES_TRANSVERSALES_DEL_ERP.md`](./DIRECTRICES_TRANSVERSALES_DEL_ERP.md).

> [!NOTE]
> **Aclaración del hardware de visión (29 Sep 2026):** la visión del POS opera sobre una **cámara
> cenital** (montada sobre el mostrador, mirando hacia abajo) con **iluminación dedicada que elimina
> las sombras**. Esto convierte la visión en un **"escáner de charola"**: el operador coloca los
> productos y el sistema los reconoce sin apuntar. Consecuencias directas:
> - El **umbral 0.35 (RN-72)** es un valor **calibrado para ese montaje cenital**, no universal, y es
>   **configurable desde el Centro de IA**.
> - El visor de visión es un **flujo persistente** (permanece abierto durante la venta), no una
>   captura bajo demanda.
> - El contrato 17 debe declarar el **`modo_captura`** (`cenital` | `manual`).
> - La visión **sigue siendo asistiva** (RN-74): sugiere, nunca decide. La regla de oro no cambia.
>
> Ver **DT-08** en [`DIRECTRICES_TRANSVERSALES_DEL_ERP.md`](./DIRECTRICES_TRANSVERSALES_DEL_ERP.md).

**Sub-fase F7.0 — Declarar la frontera POS ↔ Centro de IA (PRIMERO, sin UI):**

Antes de portar una sola línea de voz o visión, se declaran los contratos que faltan en
[`apps/api/contracts/registry.py`](../../NUEVO-POS/apps/api/contracts/registry.py:1). Hoy el POS
hablaría con el Centro de IA por **convención implícita**, lo que viola la Regla Dura (A-02).

| Acción | Detalle |
|---|---|
| Reetiquetar el contrato 17 | `vision.reconocer_producto`: proveedor pasa de `"Visión"` a `"Centro de IA"` |
| Declarar contrato de voz | `ia.transcribir_voz` (proveedor: Centro de IA) |
| Declarar contrato de NLU | `ia.interpretar_intencion` (proveedor: Centro de IA) |
| Documentar la degradación | Todo contrato de IA devuelve 503 `IA_NO_DISPONIBLE`; el POS sigue vendiendo |

**Archivos a construir (F7.1 en adelante):**
- `VoiceCartPanel.jsx` — panel de voz
- `useVoiceCart.js` — hook de voz (consume `ia.transcribir_voz` + `ia.interpretar_intencion`)
- `voiceCartMapper.js` — mapeo de intents a productos
- `VisionVisor.jsx` — visor de cámara IA (consume contrato 17)
- `useVision.js` — hook de visión
- `ProgramacionPedidoModal.jsx` — pedidos programados
- `ThemeSelector.jsx` — ✨ selector visual de temas (Fase 4 del theme engine)

**Orden de ejecución propuesto:** F7.0 (contratos) → F7.1 (temas UI, menor riesgo) → F7.2 (voz) → F7.3 (visión).

**Sub-fase F7.5 — El puente POS → Pedidos (CERRADA, 30 Sep 2026):**

El POS viejo **violaba A-02/P-01**: importaba el modelo `Order` y escribía la tabla `orders`
**directamente** desde el frontend. El POS nuevo construye el puente **por contrato**, de adentro
hacia afuera:

| Pieza | Detalle |
|---|---|
| Contrato 15 (proyección) | `proyectar_pedido(db, ticket)` — `ticket → order` en la **misma transacción** (idempotente, RN-58) |
| Contrato 16 (consulta) | `GET /orders/by-ticket/{ticket_id}` — proyección explícita (O-23) |
| DT-06.2 (almacenamiento) | Tabla `system_settings` (modelo `SystemSetting` + migración `0002`) |
| DT-07 (degradación) | `leer_politica_pago` con default seguro `PAGO_COMPLETO`; la venta nunca se bloquea |
| Frontend | `useOrderProgramming` + `OrderProgrammingModal.jsx` (UX heredada del viejo POS, §6.8) + cableado en `RetailVisionPOS.jsx` |

Gate de **7 criterios / 11 tests** en verde; CI completo verde. Evidencia:
[`FICHA_F7_5_PEDIDOS.md`](../../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F7_5_PEDIDOS.md).
**Diferido a F7.5b:** que el POS lea la política **real** de Vista General (hoy usa el default seguro).

---

### Fase 8 — Integración con CRM y Notificaciones (lado POS) — ✅ CERRADA (30 Sep 2026)
> **El POS se prepara para hablar con dos módulos futuros del ERP: Clientes (CRM) y Notificaciones.**

> [!IMPORTANT]
> **ESTADO: CERRADA.** Los 9 criterios de aceptación se cumplen, los 7 guardianes están
> limpios y todos los tests están en verde. Evidencia completa en
> [`FICHA_F8_CRM_NOTIFICACIONES.md`](../../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F8_CRM_NOTIFICACIONES.md).
> Sub-fases: F8.0 (`e594de0`) · F8.1 (`b894f1d`) · F8.2 (`c42d83c`) · F8.3 (`e20a9a2`) ·
> F8.4 (`70685f5`) · F8.5 (`d8bf8ca`) · F8.6 (`99e94cb`) · F8.7 (cierre).

> [!IMPORTANT]
> **Aclaración de alcance:** El CRM y Notificaciones son **módulos independientes del ERP**, no son parte del POS. Tendrán su propio plan, sus propias tablas, su propio código. Lo que esta fase construye es **solo el lado del POS** — los puntos de contacto para comunicarse con esos módulos cuando existan.

> [!NOTE]
> Esta fase **no bloquea** las fases 1–7. El POS funciona completo sin ella.

**Especificación completa de los módulos CRM y Notificaciones:** [PROPUESTA_CRM_Y_NOTIFICACIONES_DEL_NUEVO_POS.md](https://github.com/vikutasan/PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS/blob/main/PROPUESTA_CRM_Y_NOTIFICACIONES_DEL_NUEVO_POS.md) (40 KB)

---

**Lo que SÍ se construye en esta fase (lado POS):**

| Archivo | Qué hace | Contrato que consume |
|---|---|---|
| `CustomerIdentificationPanel.jsx` | Botón "👤 Cliente" en el header + panel para teclear teléfono | #18 `clientes.beneficios_para_ticket` |
| `TicketDeliveryPanel.jsx` | Paso post-cobro: 🖨️ Imprimir / 📱 WhatsApp / ✉️ Email / Omitir | #19 `notificaciones.encolar_ticket` |

Son **2 componentes** y **2 llamadas a contrato**. Nada más.

**Lo que NO se construye en esta fase (es de otro módulo):**

| Componente | Pertenece a | Módulo |
|---|---|---|
| Pantalla de gestión de clientes | ❌ No es del POS | Módulo Clientes (CRM) |
| Pantalla de promociones | ❌ No es del POS | Módulo Clientes (CRM) |
| Tablas `customers`, `loyalty_ledger`, `promotions`, `customer_benefits` | ❌ No es del POS | Módulo Clientes (CRM) |
| Worker de WhatsApp/SMTP | ❌ No es del POS | Módulo Notificaciones |
| Tablas `notification_outbox`, `notification_log`, `channel_config` | ❌ No es del POS | Módulo Notificaciones |

---

**Lecciones integradas (lado POS):**
- Si el CRM no responde, el POS cobra a precio de lista (**NUNCA bloquea la venta**)
- Si Notificaciones no responde, el ticket se imprime normalmente. El envío queda pendiente
- El POS muestra los beneficios que el CRM devuelve pero **nunca los calcula**
- Los descuentos se agregan como líneas negativas al ticket (el total se recalcula desde ítems persistidos)

**Criterio de aceptación (lado POS) — ✅ LOS 9 SE CUMPLEN:**
- El botón "Cliente" permite identificar por teléfono y ver beneficios
- El paso de entrega ofrece Imprimir / WhatsApp / Email
- Si el CRM está caído, la venta continúa sin beneficios
- Si Notificaciones está caído, la venta continúa con impresión
- El POS nunca importa `Order` ni escribe `customers`/`notification_outbox` (Guard E-15)
- El POS nunca define ni persiste la política de lealtad

> **Evidencia:** [`FICHA_F8_CRM_NOTIFICACIONES.md`](../../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F8_CRM_NOTIFICACIONES.md)

---

> [!NOTE]
> **NOTA DE COHERENCIA — RENUMERACIÓN RESUELTA EN F8.0 (actualizada 30 Sep 2026).**
> La [`PROPUESTA_CRM_Y_NOTIFICACIONES_DEL_NUEVO_POS.md`](./PROPUESTA_CRM_Y_NOTIFICACIONES_DEL_NUEVO_POS.md)
> se redactó cuando el plan iba en **17 contratos** y **73 reglas**. Al abrir la Fase 8, el POS
> ya tenía **25 contratos** y **81 reglas** (RN-74..81 ocupadas por visión, auditoría y zona
> horaria). La renumeración se ejecutó en **F8.0** (`e594de0`):
>
> | Elemento | Número en la propuesta | Número real declarado en F8.0 | Motivo |
> |---|---|---|---|
> | Contrato `clientes.beneficios_para_ticket` | #18 | **#26** ✅ | Los contratos 18–23 ya existían (F3.2 atómico + F5.0 cuentas) |
> | Contrato `notificaciones.encolar_ticket` | #19 | **#27** ✅ | Idem |
> | Reglas de lealtad/notificaciones | RN-74..RN-85 | **RN-82..RN-93** ✅ | RN-74..81 ya estaban declaradas en [`rules/registry.py`](../../NUEVO-POS/apps/api/rules/registry.py:596) |
>
> **Regla para el futuro:** los números de contrato y de regla se asignan **al momento de
> declararlos**, nunca al momento de proponerlos. La propuesta es un documento de intención;
> el `registry.py` es la fuente de verdad.

---

### Fase 9 — Rescate de UX del viejo POS (y afinado de la UI por defecto) — ✅ CERRADA (30 Sep 2026)
> **Se rescatan tres piezas de UX del viejo POS antes de congelar la UI por defecto del módulo nuevo.**

> [!IMPORTANT]
> **ESTADO: CERRADA.** Los 8 criterios de aceptación se cumplen, los 7 guardianes están
> limpios y todos los tests están en verde. Evidencia completa en
> [`FICHA_F9_UX_RESCATE.md`](../../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F9_UX_RESCATE.md).
> Sub-fases: F9.0.1 (modal de salida) · F9.0.2 (teclado numérico) · F9.0.3 (OfflineBanner) · F9.0.4 (cierre).

> [!NOTE]
> Esta fase **no bloquea** las fases 1–8. Es una **micro-fase de afinado operativo**: no añade
> capacidad de negocio nueva, solo rescata UX que el usuario ya pagó y valoró.

**Contexto:** el usuario invirtió mucho en la UX del viejo POS (`apps/pos/` del ERP). Antes de
definir la UI por defecto del POS nuevo, pidió comparar ambas UX y rescatar lo que valga la pena
**sin mayor problema**. La comparación archivo por archivo arrojó que **el POS nuevo ya heredó
~80% de la UX del viejo** (§6.8 cumplido). Quedaban tres piezas concretas de bajo riesgo.

**Lo que SÍ se construye en esta fase (3 piezas, 100% frontend):**

| # | Pieza | Origen (viejo POS) | Destino (nuevo POS) | Test |
|---|-------|--------------------|---------------------|------|
| 1 | Modal de salida "Cuenta sin enviar" | `RetailVisionPOS.jsx` (modal inline) | `ExitAccountModal.jsx` | `ExitAccountModal.f9_0_1.test.jsx` |
| 2 | Teclado numérico + cambio en vivo | `CheckoutScreen.jsx` (teclado) | `TecladoNumerico.jsx` | `TecladoNumerico.f9_0_2.test.jsx` |
| 3 | `OfflineBanner` (solo estado de red) | `POSOverlays.jsx` (`OfflineBanner` con `pendingCount`) | `POSOverlays.jsx` (`OfflineBanner` sin conteo) | `POSOverlays.f9_0_3.test.jsx` |

**Lo que NO se construye en esta fase (explícito):**

| Elemento | Por qué NO | Dónde va |
|---|---|---|
| Pagos mixtos (efectivo + tarjeta) | Toca el contrato de cobro y las RN | **F9.1** (plan propio) |
| Cola local (offline-first) | Contradice "el servidor es la única verdad"; reintroduce el riesgo de la cicatriz de $453; toca caja/folios/multi-terminal | Fase propia si el dolor de red es real y medido (p. ej. "F10") |
| Conteo de pendientes en el banner | No existe cola local que lo respalde (verificado, REGLA DURA 2) | No aplica |
| Cambio de colores/temas | El nuevo POS usa tokens; el viejo usaba colores hardcodeados | No aplica |
| Revertir el layout responsivo | El viejo era desktop-only; el nuevo es fluido | No aplica |

> **DECISIÓN ARQUITECTÓNICA (F9.0.3) — No se construye cola local.** Se verificó contra el código
> real (REGLA DURA 2) que el POS nuevo **no tiene cola local**: persiste directo contra la API en
> cada acción ([`useCart.anadirLinea`](../../NUEVO-POS/apps/pos/src/hooks/useCart.js:122) →
> `cliente.anadirItem`). Lo único en `localStorage` son preferencias de UI (`pos.tema`,
> `pos.ordenTerminales`), no operaciones. Por tanto, el `OfflineBanner` muestra **solo el estado de
> red**, sin conteo. Construir una cola local sería un motor offline-first que contradice el modelo
> "el servidor es la única fuente de verdad", reintroduce el riesgo de la cicatriz de $453 (cobrar
> sin red) y toca caja, folios y multi-terminal. Si algún día el dolor de red es real y medido, se
> abrirá una **fase propia**, no un parche de UX.

**Criterio de aceptación — ✅ LOS 8 SE CUMPLEN:**
- Las 3 piezas están implementadas y cableadas en la pantalla real
- Cada pieza tiene su test verde (30/30)
- `npm run ci` verde (lint 0 errores + todos los tests + los 7 guards limpios)
- La ficha de cierre existe y está completa
- §7 del Plan Maestro actualizado
- Commits + push en ambos repos
- **Ningún** cambio en backend, contratos ni reglas de negocio
- **Ningún** color hardcodeado nuevo (se usan los tokens del tema)

> **Evidencia:** [`FICHA_F9_UX_RESCATE.md`](../../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F9_UX_RESCATE.md)

---

### Fase 9.1 — Pagos mixtos (efectivo + tarjeta + transferencia) — ✅ CERRADA (30 Sep 2026)

> [!IMPORTANT]
> **ESTADO: CERRADA.** Los 10 criterios de aceptación se cumplen, los 7 guardianes están
> limpios y todos los tests están en verde. Evidencia completa en
> [`FICHA_F9_1_PAGOS_MIXTOS.md`](../../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F9_1_PAGOS_MIXTOS.md).
> Sub-fases: F9.1.0 (contrato + RN-94/RN-95) · F9.1.1 (arqueo lee N pagos) · F9.1.2 (servicio + hook) ·
> F9.1.3 (UI de abonos) · F9.1.4 (impresión + cierre).

**Contexto:** el viejo POS permitía cobrar un ticket con **varios métodos a la vez** (pago mixto).
El POS nuevo solo aceptaba un método por ticket. Esta micro-fase cierra esa brecha **sin romper**
el cobro viejo (retrocompatibilidad total).

**Lo que SÍ se construyó:**

| # | Pieza | Archivo | Test |
|---|-------|---------|------|
| 1 | Contrato `pagos[]` + `TicketSalida.payment_details` | `schemas.py` | `test_f9_1_pagos.py` (18) |
| 2 | RN-94 (suma cuadra total, sin tolerancia) + RN-95 (métodos válidos) | `rules/registry.py` | `test_f9_1_pagos.py` |
| 3 | Normalización del cobro viejo → `pagos[]` | `routers/pos.py` | `test_f9_1_pagos.py` |
| 4 | **Arqueo lee N pagos y suma SOLO el efectivo (RN-53)** | `routers/cash.py` | `test_f9_1_arqueo.py` (4) |
| 5 | Servicio + hook de cobro (`construirPaymentDetails`) | `checkoutService.js` | `checkoutService.f9_1_2.test.jsx` (22) |
| 6 | UI de abonos (agregar/editar/quitar) | `CheckoutScreen.jsx` | `CheckoutScreen.f9_1_3.test.jsx` (10) |
| 7 | Desglose de pagos en el ticket impreso | `ticketGenerator.js` | `ticketGenerator.f9_1_4.test.jsx` (8) |

**La corrección crítica (F9.1.1):** el arqueo sumaba el **total** del ticket al esperado en
efectivo, aunque parte se hubiera pagado con tarjeta. Ahora lee los N pagos y suma **solo los
abonos en efectivo** (RN-53). Un ticket $40 efectivo + $60 tarjeta sube el esperado **solo $40**.

**La lección de integración (F9.1.4a):** el cobro mixto **persistía** `payment_details` en la
base, pero `TicketSalida` **no lo exponía** — el dato nunca llegaba al cliente, así que el ticket
impreso jamás podría mostrar el desglose. Es la **misma clase de defecto** que el `GestorDeCaja`
huérfano (F4.5): *el componente existe y pasa su test, pero el usuario no puede llegar a él*.
Confirma §10.6.1: **el paso de INTEGRACIÓN también es una compuerta.**

**Criterio de aceptación — ✅ LOS 10 SE CUMPLEN:**
- Un pago mixto que cuadra el total se acepta; cobrar de menos/más se rechaza con 400
- Sin tolerancia de punto flotante (0.01 de diferencia falla) — dinero en `Decimal` (DT-02)
- Un método inválido en cualquier abono se rechaza (RN-95)
- El arqueo suma SOLO el efectivo del mixto (RN-53)
- Retrocompatibilidad total con el cobro viejo de un solo método
- La UI permite agregar/editar/quitar abonos hasta cuadrar
- El ticket impreso muestra el desglose de pagos
- `TicketSalida` expone `payment_details`
- `npm run ci` verde (lint 0 errores + todos los tests + los 7 guards limpios)
- Ficha de cierre + §7 del Plan Maestro + commits/push en ambos repos

> **Evidencia:** [`FICHA_F9_1_PAGOS_MIXTOS.md`](../../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F9_1_PAGOS_MIXTOS.md)

---

### Fase 10 — Auditoría de Paridad (viejo POS vs. nuevo POS) — ✅ CERRADA (30 Sep 2026)

> [!IMPORTANT]
> **ESTADO: CERRADA.** La auditoría de paridad se ejecutó **antes** de declarar el POS
> por concluido y de redactar la documentación final (§11). Evidencia completa en
> [`FICHA_F10_PARIDAD.md`](../../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F10_PARIDAD.md).
> Sub-fases: F10.0 (inventario crudo) · F10.1 (triaje) · F10.2 (cierre de brechas) · F10.3 (cierre).

**Contexto — por qué existe esta fase:** el dueño detectó, al revisar el Gestor de Terminales,
que el botón **"Copiar URL"** del viejo POS **no existía** en el nuevo. Ese hallazgo disparó una
pregunta legítima: *si se omitió una cosa, ¿cuántas más se omitieron?* En vez de declarar el POS
terminado y escribir la documentación sobre un sistema incompleto, se ejecutó una **auditoría de
paridad sistemática** contra el viejo POS.

**El diagnóstico del caso "Copiar URL":** no era un problema de "falta el botón pero la lógica
está". Se verificó el código y **ni el botón ni la lógica** existían en el nuevo POS. Fue una
**OMISIÓN completa** — la misma clase de defecto que el `GestorDeCaja` huérfano (F4.5) y que
`payment_details` no expuesto (F9.1.4a).

**Las 4 categorías de paridad:**

| Categoría | Significado |
|---|---|
| **PORTADA** | Existe en el viejo y en el nuevo, y funciona. |
| **OMITIDA** | Existe en el viejo, **no** en el nuevo. Es la brecha a cerrar. |
| **HUÉRFANA** | La lógica existe en el nuevo, pero **sin punto de entrada**. |
| **DESCARTADA** | Se decidió **no** portarla, con una razón documentada. |

**El triaje (F10.1) — 11 brechas verificadas:**

| # | Brecha | Categoría | Decisión |
|---|--------|-----------|----------|
| B-01 | Botón "Copiar URL" en el Gestor de Terminales | OMITIDA | **PORTAR** (cerrada en F10.2) |
| V-01 | Zero-Auto-Restore de sesión (localStorage) | DESCARTADA | No aplica: el nuevo POS no persiste sesión en localStorage |
| V-02 | `getProductEmoji` | PORTADA | Reubicada al backend (`producto.icono`) |
| V-03 | `handleImageUpload` de terminales | DESCARTADA | Diseño distinto: `PRESET_ICONS` |
| V-04 | `loadTerminalsConfig` | PORTADA | `fetchTerminalConfig()` |
| V-05 | `DEFAULT_TERMINALS` | PORTADA | `TERM-01..TERM-06` |
| V-06 | `ForceLogoutModal` | DESCARTADA | Reemplazada por el heartbeat con TTL (`useTerminalLocking`) |
| V-07 | `OfflineBanner` con `pendingCount` | DESCARTADA | Intencional: el nuevo POS **no** tiene cola local (decisión v1.1) |
| V-08 | `useVisitDraft` | DESCARTADA | Fuera de alcance (pertenece a Grandeza) |
| V-09 | `calcularDenominaciones` | DESCARTADA | No aplica: el nuevo usa `BILLETES_RAPIDOS` |
| V-10 | `calcularPuntosAGanar` | DESCARTADA | La lealtad es del CRM, no del POS |

**Resultado:** 1 brecha portada (B-01), 4 ya portadas, 6 descartadas con razón. **El POS nuevo
está en paridad funcional con el viejo** en todo lo que importa.

**La brecha cerrada (F10.2 — B-01):** se portó la **integración** (el botón) y se reescribió la
**implementación** (Clipboard API con fallback a `execCommand`, siguiendo §6.8). Test dedicado
`TerminalSelector.f10_2.test.jsx` (4/4 verde). `npm run ci` verde.

**La lección (F10.3):** ver §10.6.1 — *"el componente existe y pasa su test" ≠ "el conjunto está
completo"*.

> **Evidencia:** [`FICHA_F10_PARIDAD.md`](../../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F10_PARIDAD.md)

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
| **IA (voz/visión)** | Consumir el **Centro de IA** por contrato (DT-07) | El POS no contiene el motor de IA. Cuando el Centro de IA exista, el POS no se toca: solo cambia quién implementa el contrato. |
| **CRM / Notificaciones** | Consumir por contrato (Fase 8) | El POS consume beneficios, no los define. Degradación elegante: nunca bloquea la venta. |
| **Valores transversales** | Consumir de **Vista General** por contexto (DT-06) | Zona horaria, moneda y sucursal se declaran una sola vez. El POS los consume, no los define. |

---

## 8.1 EL POS ES EL PRIMER MÓDULO DE UN ERP RECONSTRUIDO

> **Aclaración explícita del dueño (29 Sep 2026).** Se documenta aquí para que no exista
> posibilidad de confusión cuando se construyan los demás módulos.

**La visión completa:**

1. **Esto no es un proyecto del POS.** Es el **primer módulo** de una reconstrucción del ERP
   completo. El método que se usa aquí (diagnóstico → plan por partes → sub-fases con gate →
   ficha → commit) se **replicará** en cada módulo del ERP.
2. **El POS se integra con los demás módulos del ERP**, siempre **por contrato**:
   - **Centro de IA** (voz, visión, OCR, NLU) — ver **DT-07**.
   - **CRM / Notificaciones** (clientes, lealtad, envío de tickets) — ver **Fase 8**.
   - **Vista General** (zona horaria, moneda, sucursal) — ver **DT-06**.
   - **Estadísticas** (contexto diario, resúmenes de venta) — ver **§8.2** y **F10.4**.
   > **Corrección importante (v2.3):** estos módulos **NO están vacíos**. Existen y funcionan
   > en el ERP viejo, pero con **arquitectura parchada**. Ver **§8.2** para el hecho completo.
3. **Lo que sí podemos hacer ahora** es **dejar el POS preparado** para que, cuando cada módulo
   sea rehecho (uno a uno, por el dueño), la integración sea limpia y no haya que reescribir el POS. Eso significa:
   - **Declarar los contratos** que faltan (empezando por los de IA en F7.0).
   - **No inventar** el motor de IA, ni el CRM, ni el selector de valores transversales dentro del POS.
   - **Acoplar por contrato** a los módulos viejos que ya existen (p. ej. Estadísticas en F10.4),
     sin heredar su arquitectura parchada.
   - **Respetar la degradación elegante**: si un módulo del ERP cae, el POS sigue vendiendo.

**La regla que lo resume:**

> **El POS es hermano de los demás módulos del ERP, no su padre. Consume por contrato; nunca los contiene.**

---

## 8.2 EL ERP VIEJO EXISTE, PERO SU ARQUITECTURA ES PARCHADA — SE REHARÁ MÓDULO POR MÓDULO

> **Aclaración explícita del dueño (30 Sep 2026).** Se documenta aquí, junto a §8.1, porque
> corrige el supuesto más peligroso que se podría hacer al leer este Plan Maestro.

**El hecho, sin adornos:**

1. **El ERP que corre el POS viejo NO está vacío.** Existe, funciona, y tiene módulos reales
   (Estadísticas, CRM, Almacenes, RRHH, Auditoría, Reparto Grandeza, etc.). **No hay que
   inventarlos desde cero.**
2. **Pero ese ERP se construyó SOBRE LA MARCHA**, módulo por módulo, según las necesidades
   operativas del día a día, **agregando y modificando cosas** sin una arquitectura planificada
   de antemano. Es, en palabras del dueño, una **"arquitectura parchada"**.
3. **Consecuencia directa:** cada módulo viejo arrastra los mismos vicios que el POS viejo
   arrastraba (acoplamiento, tablas compartidas, lógica duplicada, ausencia de contratos,
   ausencia de tests). **No se puede asumir que un módulo viejo es "la versión buena" solo
   porque existe y funciona.**
4. **El trabajo del dueño es explícito y secuencial:** construir la **versión mejorada de cada
   módulo** (con el mismo método de este Plan Maestro: diagnóstico → plan por partes → sub-fases
   con gate → ficha → commit) e **irlos integrando UNO A UNO** con el POS nuevo.

**Lo que esto cambia en la forma de trabajar:**

| Antes (supuesto incorrecto) | Ahora (hecho documentado) |
|---|---|
| "Los módulos del ERP aún no existen; el POS se prepara para cuando existan." | "Los módulos existen, pero están parchados; el POS se integra con ellos **por contrato** y cada uno se rehará a su tiempo." |
| "Integrar = esperar a que el módulo nazca." | "Integrar = **acoplar por contrato** al módulo viejo hoy, y **rehacerlo** cuando le toque su fase." |
| "Si el módulo viejo funciona, sirve tal cual." | "Que funcione no significa que sirva: hay que **profesionalizarlo y acoplarlo**." |

**La regla que lo resume:**

> **El ERP viejo es el ORÁCULO, no el MODELO.** Se le consulta qué debe hacer cada módulo
> (su comportamiento, sus reglas, su UX heredada); **no** se le copia su arquitectura parchada.
> Cada módulo se rehará con el método del POS, uno a uno, y se acoplará por contrato.

**Corolario operativo (evita dos errores opuestos):**

- **Error A — "hay que inventar el módulo":** falso. El módulo existe; hay que **profesionalizarlo
  y acoplarlo**, no construirlo de cero.
- **Error B — "el módulo viejo ya sirve":** falso. Su arquitectura es parchada; **no se hereda
  su implementación, solo su comportamiento** (misma lógica que §6.8: *la integración se hereda,
  la implementación se reescribe*).

**Aplicación inmediata (F10.4):** el "Contexto diario post-corte" (B-02) es el primer caso de
este patrón. El módulo **Estadísticas ya existe** (backend `apps/api/modules/analytics/` +
frontend `apps/analytics/` + tabla `daily_contexts` + endpoints `GET/PUT /analytics/context`).
Por lo tanto **no se inventa nada**: el POS nuevo **se acopla por contrato** al endpoint que
Estadísticas ya expone. "Profesionalizar y acoplar Estadísticas" al nuevo ERP es una **fase
propia (F11)**, no un parche dentro del cierre del POS.

---

## 9. RESULTADO ESPERADO

Cuando las 8 fases estén completas, al abrir `localhost:5100/` verás:

1. **Landing** → Selector de terminales (tarjetas con estado libre/mío/ajeno)
2. **Login** → El cajero se identifica
3. **POS** → Catálogo + ticket + checkout (idéntico al viejo pero limpio)
4. **Gestor de caja** → Abrir/cerrar turno, cortes
5. **Pizarrón** → Cuentas abiertas en paralelo
6. **Voz** → "Agrega dos baguettes" → aparecen en el ticket
7. **Cámara** → Reconoce productos en la charola
8. **PDF** → Exporta catálogo para cobro manual sin sistema
9. **Temas** → Elige entre 3 estilos visuales
10. **Cliente** → "Es la señora María" → se aplican sus beneficios automáticamente
11. **WhatsApp** → "Le envío su ticket por WhatsApp" → el cliente lo recibe en su celular
12. **Lealtad** → "Tiene 150 puntos acumulados" → puede canjearlos por descuento

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

### 10.6 El principio "de adentro hacia afuera" — diferencia intencional con DeepSeek

> **Origen:** PLAN_DE_CONSTRUCCION, §0.2

**Cómo lo proponía DeepSeek (por capas horizontales):**

DeepSeek diseñó un orden de construcción **por capas**, donde se completa una capa entera del edificio antes de subir a la siguiente:

```
F1. TODAS las tablas de datos (17 tablas)          ← cimiento completo
F2. TODOS los contratos entre módulos (17)         ← frontera completa
F3. TODAS las 81 reglas de negocio con tests       ← comportamiento completo
F4. TODOS los tests guardianes                     ← blindaje completo
F5. TODAS las 26 interfaces                        ← superficie completa
F6. Consolidación central                          ← techo
```

**Ventaja de este enfoque:** garantiza que nunca construyes una pared sin cimiento. Es académicamente puro.

**Desventaja:** hasta completar F5 (la 5ª capa), no tienes NADA usable. No puedes enseñarle una pantalla funcional al dueño del negocio hasta que hayas construido las 17 tablas, los 17 contratos y las 81 reglas. Eso puede tardar semanas sin mostrar progreso visible.

---

**Cómo lo hacemos nosotros (por funciones verticales):**

Nuestro plan construye **rebanadas verticales** — cada fase entrega una función completa de piso a techo (dato + contrato + regla + test + interfaz):

```
Fase 1: Terminal Selector   → tabla terminal_locks + endpoint + hook + UI
Fase 2: Sesión               → tabla sessions + servicio + hook + UI
Fase 3: POS Completo         → tablas tickets/items + 9 hooks + 12 endpoints + UI
Fase 4: Gestor de Caja       → tablas cash + servicio + UI
...
```

**Ventaja:** al terminar la Fase 1, ya puedes abrir el POS y ver las terminales. Al terminar la Fase 3, ya puedes cobrar. El dueño ve progreso real en cada fase, puede probar, puede opinar.

**Desventaja:** podrías construir una tabla sin respetar los estándares (UUID, UTC, Numeric). Por eso existe la §10.3 y la §10.4 como guardias.

---

**La garantía:** dentro de cada fase, respetamos el orden de DeepSeek:

```
Cada fase internamente sigue:
  1. Endpoint (dato + contrato)    ← cimiento de la rebanada
  2. Hook (regla de negocio)       ← comportamiento de la rebanada
  3. Test (guardián)               ← blindaje de la rebanada
  4. Componente (interfaz)         ← superficie de la rebanada
```

> **En resumen:** DeepSeek proponía construir TODOS los cimientos, luego TODAS las paredes, luego TODOS los techos. Nosotros construimos una habitación completa a la vez (cimiento + pared + techo), pero cada habitación respeta el mismo orden interno. El resultado final es el mismo edificio — la diferencia es que el nuestro se puede ir probando habitación por habitación.

---

#### 10.6.1 LA LECCIÓN DE LA MICRO-FASE F4.5 — el paso de INTEGRACIÓN también es una compuerta

> **Origen:** Micro-fase correctiva F4.5 "Montaje del Gestor de Caja" (30 Sep 2026).
> **Ficha:** `FICHA_F4_5_MONTAJE_CAJA.md`.

El principio "de adentro hacia afuera" tiene un **riesgo latente** que se
materializó en la Fase 4 y que esta micro-fase cierra:

**El hecho:** el `GestorDeCaja.jsx` (467 líneas) se construyó, se probó y pasó su
compuerta en verde. El backend de caja (`cash.py`, 6 endpoints) y el servicio
(`cashService.js`) también. **Pero nadie lo montó en la pantalla.** El componente
quedó **huérfano**: existía, estaba probado, y **ningún usuario podía llegar a
él**.

**El impacto:** RN-49 exige una sesión de caja `OPEN` para cobrar. Sin punto de
entrada para abrir el turno, **el POS no podía cobrar**. Un sistema con todas sus
piezas verdes era, sin embargo, funcionalmente inoperante.

**La causa raíz:** cada sub-fase pasó su compuerta **en aislamiento**. La
compuerta de F4.3 verificaba "el componente funciona"; la de F4.4 verificaba "el
corte se genera". **Ninguna compuerta verificaba "el usuario puede llegar al
componente".** El paso de INTEGRACIÓN no estaba declarado como compuerta.

**La lección (regla nueva):**

> *"el componente existe y pasa su test" ≠ "el usuario puede llegar a él".*
>
> Toda sub-fase que construye un **componente de superficie** (una pantalla, un
> panel, un overlay) debe declarar explícitamente **su punto de entrada** — el
> botón, el gesto o la ruta que lo hace alcanzable — y **probarlo**. La
> integración no es un detalle de cierre: es una compuerta más.

**Cómo se previene en adelante:**

1. En el plan de cada fase, la sub-fase de superficie debe responder por escrito:
   *"¿desde dónde llega el usuario a este componente?"*.
2. El test de integración debe montar la **pantalla real** (no el componente
   aislado) y verificar que el punto de entrada existe y abre el componente.
3. Al cerrar una fase, revisar que **ningún componente construido quede sin
   punto de entrada** (grep de imports vs. renders).

**Corolario operativo:** una guarda nueva en la ruta crítica (como la de F4.5.3,
que exige turno abierto para cobrar) **obliga** a revisar todos los tests que
tocan esa ruta. En F4.5 esto rompió dos tests existentes (`f8_6` y `f3_cierre`)
que cobraban sin declarar el contrato de caja; se corrigieron sembrando el turno
`OPEN`. La guarda era correcta; lo que faltaba era que los tests declararan el
contrato que la pantalla ahora consume.

#### 10.6.2 LA LECCIÓN DE LA FASE 10 — la COMPLETITUD del conjunto también es una compuerta

> **Origen:** Fase 10 "Auditoría de Paridad" (30 Sep 2026).
> **Ficha:** `FICHA_F10_PARIDAD.md`.

La lección de §10.6.1 ("el componente existe y pasa su test" ≠ "el usuario puede
llegar a él") tenía una **segunda mitad** que se materializó en la Fase 10:

> *"el componente existe y pasa su test" ≠ "el conjunto está completo".*

**El hecho:** el dueño, revisando el Gestor de Terminales, notó que el botón
**"Copiar URL"** del viejo POS **no existía** en el nuevo. Se verificó el código:
**ni el botón ni la lógica** estaban. No era un componente huérfano — era una
**pieza que nunca se construyó**, porque ninguna sub-fase la había pedido
explícitamente.

**La causa raíz — la misma clase de defecto, tercera instancia:**

| # | Instancia | Fase | Qué pasó |
|---|-----------|------|----------|
| 1 | `GestorDeCaja` huérfano | F4.5 | El componente existía y pasaba su test, pero **nadie podía llegar a él**. |
| 2 | `payment_details` no expuesto | F9.1.4a | El dato se persistía, pero **no se exponía** en la salida. |
| 3 | "Copiar URL" omitido | F10 | La pieza existía en el viejo POS, pero **nunca se construyó** en el nuevo. |

Las tres comparten la raíz: **"de adentro hacia afuera" verifica cada pieza en
aislamiento, pero no verifica la COMPLETITUD del conjunto contra el viejo POS.**
Cada sub-fase pasa su compuerta; ninguna compuerta compara el inventario completo.

**La lección (regla nueva):**

> *"el componente existe y pasa su test" ≠ "el conjunto está completo".*
>
> Antes de declarar un módulo terminado, se debe ejecutar una **auditoría de
> paridad** contra el sistema de referencia: inventariar **todo** lo que el viejo
> hacía, clasificar cada pieza (PORTADA / OMITIDA / HUÉRFANA / DESCARTADA) y
> **cerrar cada OMITIDA o justificarla por escrito**. La completitud no se asume:
> se audita.

**Cómo se previene en adelante:**

1. Al cerrar un módulo, ejecutar una **auditoría de paridad** contra el sistema
   de referencia (el viejo POS, el ERP, el contrato). No basta con que "todo lo
   construido pase": hay que verificar que **nada de lo esperado falte**.
2. Cada pieza del inventario se clasifica en una de las 4 categorías, y cada
   **OMITIDA** se cierra o se documenta como **DESCARTADA** con su razón.
3. La auditoría se ejecuta **antes** de la documentación final: no se documenta
   un sistema sin verificar que está completo.

**Corolario:** la documentación final (§11) es el **último** entregable, no el
primero. Se escribe sobre un sistema **verificado completo**, no sobre uno que
"parece" completo porque todas sus piezas verdes pasaron sus tests.

#### 10.6.3 LA LECCIÓN DE LA FASE 10.4 — el inventario de componentes no ve las integraciones

> **Origen:** Fase 10.4 "Contexto diario post-corte" (30 Sep 2026).
> **Ficha:** `FICHA_F10_4_CONTEXTO_DIARIO.md`.

La auditoría de paridad de F10 inventarió **componentes** (¿existe el botón?
¿existe el modal?). Pero el dueño, al **probar** la auditoría, encontró una pieza
que el inventario no veía: el **"Contexto diario post-corte"** (clima, día atípico,
notas) que el viejo POS escribía en la tabla `daily_contexts` del módulo de
Estadísticas.

**El hecho:** no era un componente del POS. Era una **escritura del POS hacia la
tabla de OTRO módulo**. El inventario de componentes del POS no la contenía, porque
no es un componente del POS — es un **puente entre módulos**.

**La lección (regla nueva):**

> *"el inventario de componentes no ve las integraciones."*

> La auditoría de paridad debe inventariar no solo los **componentes**, sino también
> **los flujos de datos entre módulos** (escrituras del POS hacia el ERP). Un POS
> puede tener todos sus componentes y aun así faltarle un **puente entre módulos**.

**Cómo se previene en adelante:**

1. Al inventariar la paridad, incluir una columna de **"escrituras hacia otros
   módulos"** (no solo pantallas y botones).
2. Cada escritura del POS hacia el ERP se declara como **contrato** (frontera A-02),
   aunque su implementación viva en el otro módulo.
3. Si el endpoint del otro módulo aún no está profesionalizado, el contrato se
   declara con `estado_hoy="Deuda"` y se acopla en la fase del módulo destino.

#### 10.6.4 LA LECCIÓN DE LA FASE 10.5 — el inventario de componentes no ve los FLUJOS DE DATOS

> **Origen:** Fase 10.5 "Paridad de datos de caja" (30 Sep 2026).
> **Ficha:** `FICHA_F10_5_PARIDAD_DE_DATOS_DE_CAJA.md`.

La lección de §10.6.3 ("el inventario no ve las integraciones") tenía una **tercera
mitad** que se materializó en F10.5, a partir de la **analogía del trasplante de
corazón** del dueño: *"el nuevo POS debe recibir la misma 'presión sanguínea'
(datos) que el viejo, o el cuerpo (ERP) rechazará el órgano."*

**El hecho:** el viejo `CashSummaryResponse` devolvía **8 campos**; el nuevo
`ResumenTurnoSalida` (contrato 12) devolvía **2** (`esperado`, `movimientos`).
**7 campos faltaban.** El componente existía, pasaba su test y aun así **consumía
menos datos** que su equivalente viejo. Y el nombre del cajero (`usuario_nombre`)
tampoco viajaba al abrir el turno (contrato 10).

**La causa raíz — la misma clase de defecto, quinta instancia:**

| # | Instancia | Fase | Qué pasó |
|---|-----------|------|----------|
| 1 | `GestorDeCaja` huérfano | F4.5 | El componente existía, pero **nadie llegaba a él**. |
| 2 | `payment_details` no expuesto | F9.1.4a | El dato se persistía, pero **no se exponía**. |
| 3 | "Copiar URL" omitido | F10 | La pieza existía en el viejo, pero **nunca se construyó**. |
| 4 | "Contexto diario" ausente | F10.4 | La **integración POS→ERP** no se portó. |
| 5 | Resumen de caja incompleto | F10.5 | El **flujo de datos** del contrato estaba incompleto. |

**La lección (regla nueva):**

> *"el inventario de componentes no ve los FLUJOS DE DATOS."*

> Un componente puede existir, pasar su test y aun así **consumir menos datos** que
> su equivalente viejo. La auditoría de paridad debe comparar **el payload de cada
> contrato** (entrada y salida) contra el del módulo viejo, **campo por campo**. Un
> contrato "que funciona" puede estar **incompleto**.

**Cómo se previene en adelante:**

1. Al inventariar la paridad, incluir una columna de **"campos del payload"** por
   cada contrato (entrada y salida), comparando viejo vs nuevo.
2. Todo campo que el viejo exponía y el nuevo no, se cierra o se documenta como
   **DESCARTADA** con su razón.
3. La comparación de payloads se ejecuta **antes** de la documentación final, junto
   con la auditoría de componentes (§10.6.2) y de integraciones (§10.6.3).

**Corolario:** la paridad tiene **tres dimensiones**, no una: **componentes**
(§10.6.2), **integraciones** (§10.6.3) y **flujos de datos** (§10.6.4). Un módulo
solo está completo cuando las tres están auditadas.

### 10.7 Documentos de referencia obligatoria por fase

| Antes de construir... | Consultar... |
|---|---|
| **Cualquier cosa** | [PROMPT_DEL_ARQUITECTO_DEL_NUEVO_POS.md](./06-prompt-del-arquitecto/PROMPT_DEL_ARQUITECTO_DEL_NUEVO_POS.md) (personalidad + 16 estándares de calidad) |
| **Cualquier cosa** | [DIRECTRICES_TRANSVERSALES_DEL_ERP.md](./DIRECTRICES_TRANSVERSALES_DEL_ERP.md) (DT-01 a DT-06: tiempo, dinero, identidad) |
| Cualquier pantalla | [ESPECIFICACION_DE_INTERFACES_POS.md](./ESPECIFICACION_DE_INTERFACES_POS.md) (las 26 fichas) |
| Cualquier componente | [ESPECIFICACION_RESPONSIVA_Y_ERGONOMIA_TACTIL.md](./ESPECIFICACION_RESPONSIVA_Y_ERGONOMIA_TACTIL.md) (R-01 a R-04) |
| Cualquier regla de negocio | [ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md](./ESPECIFICACIONES%20DEL%20PROYECTO/ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md) (RN-01 a RN-81) |
| Cualquier tabla/endpoint | [MODELO_DE_DATOS_DEL_NUEVO_POS.md](./MODELO_DE_DATOS_DEL_NUEVO_POS.md) |
| Cualquier frontera entre módulos | [CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md](./CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md) |
| Integración con CRM/Notificaciones | [PROPUESTA_CRM_Y_NOTIFICACIONES_DEL_NUEVO_POS.md](./PROPUESTA_CRM_Y_NOTIFICACIONES_DEL_NUEVO_POS.md) |

### 10.8 Los 16 estándares de calidad del constructor (PROMPT DEL ARQUITECTO)

> **Origen:** [PROMPT_DEL_ARQUITECTO_DEL_NUEVO_POS.md](./06-prompt-del-arquitecto/PROMPT_DEL_ARQUITECTO_DEL_NUEVO_POS.md)
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
1. Tocar el ERP viejo
2. Entregar código basura (provisional, placeholders, console.log)
3. Escribir `try/except pass` en ruta crítica
4. Usar `Float` para dinero o `DateTime` naive
5. Usar el folio como identidad
6. Leer tablas de otro módulo
7. Avanzar de fase con pruebas fallando
8. Decir "ya funciona" sin mostrar evidencia
9. **Asumir sin verificar** — nombrar una tabla/campo/contrato/endpoint sin comprobar que existe, afirmar qué hace una función sin leerla, o contar campos/reglas/estados sin abrirlos y contarlos (REGLA DURA 2)

---

## 11. LA DOCUMENTACIÓN FINAL DEL NUEVO POS (entregable obligatorio)

> **Origen:** Decisión del dueño, 29 Sep 2026.
> **Estado:** PENDIENTE — se ejecuta al cerrar la última fase (Fase 8).
> **Regla dura:** esta sección NO es opcional. El proyecto NO se considera terminado hasta que exista la documentación final.

### 11.1 Qué se debe producir

Al terminar las 8 fases, se redacta **UNA documentación nueva, única y actualizada del POS nuevo**, que **fusiona dos fuentes**:

| Fuente | Qué aporta | Dónde vive |
|---|---|---|
| **Documentación del POS viejo** | La **intención**: qué debía hacer el sistema, las 81 reglas de negocio, las cicatrices, el cementerio de bugs, las especificaciones funcionales | `ERP-R-DE-RICO` → `ESPECIFICACIONES DEL PROYECTO/` |
| **Documentación generada durante la construcción** | La **realidad**: qué se construyó de verdad, con qué evidencia, qué se dejó fuera y por qué | `NUEVO-POS` → `docs/05-plan-de-construccion/FICHA_F*.md` + `PLANOS-ARQUITECTONICOS` |

**El resultado NO es "pegar los dos documentos".** Es un documento nuevo que integra ambas capas y las reconcilia.

### 11.2 El principio rector: el POR QUÉ de cada decisión

> [!IMPORTANT]
> **Cada decisión de diseño debe explicar de dónde viene.** No basta con decir "se hizo así". Hay que decir **por qué necesidad**, **por qué problema**, **por qué circunstancia** surgió esa decisión.

Toda decisión documentada debe responder estas preguntas:

1. **¿Qué problema resolvía?** (la necesidad concreta)
2. **¿En qué circunstancia surgió?** (¿fue un bug en producción? ¿una limitación técnica? ¿una lección de la batalla?)
3. **¿Qué alternativas se descartaron y por qué?**
4. **¿Cómo se verifica hoy que sigue siendo correcta?** (el test que la protege)

**Ejemplo del nivel de detalle exigido:**

> ❌ **Mal:** "El endpoint devuelve 5 campos."
>
> ✅ **Bien:** "El endpoint devuelve exactamente 5 campos porque en el POS viejo, cuando el pizarrón pedía las cuentas abiertas con todas sus líneas, la respuesta tardaba varios segundos con 8+ cuentas y congelaba la pantalla del cajero (incidente documentado). La solución fue exponer solo la proyección mínima (Regla 15) y leer las líneas solo al recuperar una cuenta concreta. Se verifica con `test_respuesta_ligera_max_5_campos`."

### 11.3 Qué NO se desecha (prohibido tirar)

> [!CAUTION]
> **Nada de lo siguiente se borra ni se resume "para ahorrar espacio".** Es la memoria del proyecto y su valor principal.

- **El cementerio de bugs** — cada bug resuelto, con su síntoma, su causa raíz y su blindaje actual.
- **Las cicatrices** — los incidentes de producción (T5/CAJA, v6.1 $453, etc.) y qué regla nació de cada uno.
- **Las 81 reglas de negocio (RN-01 a RN-81)** — con su enunciado, su origen y su test.
- **Las 6 prohibiciones absolutas** — con el caso real que las originó.
- **Las 10 reglas arquitectónicas derivadas de la batalla** — con su historia.
- **Los 16 estándares de calidad (E-01 a E-16)** — con su criterio de verificación.
- **Las decisiones descartadas** — lo que se probó y NO funcionó, y por qué. (Evita que una IA futura lo reintente.)

### 11.4 El objetivo final: blindar contra la repetición de errores

> [!IMPORTANT]
> **El propósito de esta documentación es que NINGUNA IA que programe sobre este POS vuelva a cometer los errores del pasado.**

Para lograrlo, la documentación final debe ser **legible por una IA sin contexto previo**. Eso significa:

- Cada regla dice **qué prohíbe**, **por qué existe** y **cómo se detecta** si se viola.
- Cada decisión dice **qué se descartó** y **por qué**, para que nadie lo reintente.
- Cada cicatriz dice **qué pasó**, **cuánto costó** y **qué la previene hoy**.
- Los guardianes automáticos (los greps de CI) están documentados como la **aplicación viva** de esas reglas.

**Criterio de éxito:** una IA nueva, leyendo solo esta documentación, debe poder:
1. Entender la arquitectura sin leer todo el código.
2. Saber qué está prohibido y por qué.
3. No reintentar soluciones que ya fallaron.
4. Saber dónde está el test que prueba cada afirmación.

### 11.5 Estructura propuesta de la documentación final

| Tomo | Contenido | Fuente principal |
|---|---|---|
| **I. Visión y arquitectura** | Objetivo, las 6 prohibiciones, las 10 reglas de batalla, los 16 estándares, el principio "de adentro hacia afuera" | Plan Maestro §1–§5, §10 |
| **II. Las 81 reglas de negocio** | RN-01 a RN-81, cada una con origen + test | Especificación funcional vieja + `rules/registry.py` |
| **III. El cementerio de bugs y las cicatrices** | Cada bug/incidente: síntoma, causa, blindaje | Documentación vieja + fichas de fase |
| **IV. Contratos y fronteras** | Los 23 contratos, con firma, garantías y errores | `contracts/registry.py` |
| **V. Superficie e interfaces** | Las 24 interfaces, los 6 flujos, la paleta canónica | `superficie/registry.py` |
| **VI. El acta de obra** | Las fichas de cada sub-fase: qué se construyó, con qué evidencia | `docs/05-plan-de-construccion/FICHA_F*.md` |
| **VII. Guía para la próxima IA** | Cómo leer esta documentación, qué está prohibido, dónde está cada test | Síntesis de todo lo anterior |

### 11.6 Cuándo se ejecuta

- **No antes.** Se redacta **al cerrar la Fase 8**, cuando ya existe el acta de obra completa.
- **Durante la construcción**, cada ficha de fase ya va acumulando el material crudo (decisiones, defectos hallados, evidencia). Esa es la materia prima del Tomo VI.
- **Al final**, se hace la síntesis y la reconciliación con la documentación del POS viejo.

> **En una frase:** el POS viejo nos dio el plano y las cicatrices; la construcción nos dio el acta de obra. Al final, ambos se funden en un solo documento que explica **qué es el POS nuevo, por qué es así, y cómo evitar repetir los errores que lo hicieron necesario.**

---

> [!IMPORTANT]
> **Este plan (v1.4) integra lo valioso de los 3 planes anteriores, las 5 decisiones del dueño sobre CRM, los 16 estándares de calidad del constructor, y el entregable obligatorio de la documentación final (§11). Los documentos originales de planes quedan en la carpeta PLANES DESCONTINUADOS.**


