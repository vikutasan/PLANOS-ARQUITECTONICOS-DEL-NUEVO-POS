# 🏗️ PLAN MAESTRO DEFINITIVO — POS Nuevo "R de Rico"

> **Fecha:** 28 Sep 2026  
> **Versión del plan:** 1.2  
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

### Fase 8 — CRM + Notificaciones (Clientes, Lealtad, WhatsApp/Email)
> **Saber a quién le vendes. Premiar al que vuelve. Entregar el ticket como el cliente quiera.**

> [!NOTE]
> Esta fase **no bloquea** las fases 1–7. El POS funciona completo sin ella. Se construye cuando el negocio esté listo para operar lealtad.

**Especificación completa:** [PROPUESTA_CRM_Y_NOTIFICACIONES_DEL_NUEVO_POS.md](https://github.com/vikutasan/PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS/blob/main/PROPUESTA_CRM_Y_NOTIFICACIONES_DEL_NUEVO_POS.md) (40 KB, diseño completo con contratos, flujos, tablas, reglas)

**Módulo CRM (Clientes) — archivos a construir:**
- `CustomerIdentificationPanel.jsx` — panel de identificación (teléfono → buscar / alta rápida)
- `CustomerCRMHub.jsx` — 2 pestañas: Clientes + Promociones (entrada en sidebar del ERP futuro)
- `LoyaltyLedgerView.jsx` — historial de puntos del cliente
- `PromotionEditor.jsx` — crear/editar promociones (BORRADOR → ACTIVA → VENCIDA/PAUSADA)
- `customerService.js` — API del CRM (contratos 18 + internos)
- Backend: 5 tablas (`customers`, `loyalty_ledger`, `promotions`, `customer_benefits`, `customer_history`)

**Módulo Notificaciones — archivos a construir:**
- `TicketDeliveryPanel.jsx` — paso post-cobro (Imprimir / WhatsApp / Email / Omitir)
- `notificationService.js` — API de notificaciones (contrato 19)
- `notificationWorker.py` — worker que procesa la cola (WhatsApp Business API / SMTP)
- Backend: 3 tablas (`notification_outbox`, `notification_log`, `channel_config`)

**Lecciones integradas:**
- Outbox transaccional: el mensaje se encola en la MISMA transacción del ticket (Regla de Oro #7)
- NUNCA bloquea la venta: si el CRM o WhatsApp caen, la venta continúa
- Ledger inmutable para puntos: se asientan, nunca se hacen UPDATE (como inventario)
- Idempotencia por `evento_id` + constraint UNIQUE
- Canje de puntos: reserva → confirmación (si el cobro falla, la reserva se libera)
- Promociones nunca se borran (auditoría histórica)

**Criterio de aceptación:**
- Puedes cobrar identificando al cliente por teléfono y ver sus beneficios aplicados
- Puedes enviar ticket por WhatsApp o email después de cobrar
- Si el CRM está caído, la venta continúa a precio de lista
- Si WhatsApp está caído, el ticket queda en cola y se envía cuando se recupere

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

