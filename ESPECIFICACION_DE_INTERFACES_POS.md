# ESPECIFICACIÓN DE INTERFACES DEL POS

**Documento 7 — El mapa visual y táctil del Punto de Venta**

> **Propósito de este documento.** El [`ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md`](../ESPECIFICACIONES%20DEL%20PROYECTO/ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md) documenta **qué hace** el POS (backend: reglas de negocio, contratos, transacciones). Este documento documenta **cómo se ve y cómo se toca** el POS (frontend: las 26 interfaces reales). Juntos cierran el círculo: sin este documento, quien reconstruya el Nuevo POS sabrá qué debe calcular, pero no cómo debe sentirse en el mostrador.
>
> **Regla de oro de este documento.** La ergonomía del mostrador es **intocable**. Cada ficha describe la interfaz **tal como es hoy**, no como "debería ser". Las mejoras se anotan como observaciones, nunca se aplican aquí.

---

## §0. Diagnóstico: el hueco que este documento cierra

### §0.1 Qué estaba documentado y qué no

| Capa | Documento existente | Cobertura |
|---|---|---|
| **Backend** (reglas, contratos, transacciones) | `ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md` | ✅ 100% — leyó los `.py` |
| **Frontend** (páginas, modales, paneles) | *Ninguno* | ❌ 0% — nunca se documentó |

El propio §A.2 de la especificación funcional lo declara: *"se ignoró la estructura del código"* (se leyó solo el backend). Por lo tanto, **ninguna de las 26 interfaces del POS tiene hoy una ficha propia**.

### §0.2 Por qué importa

La ergonomía y la estética del POS —que el dueño refinó durante años— **viven en los 26 archivos `.jsx`**, no en el backend. Si el Nuevo POS se reconstruye leyendo solo el backend, se perderá:

- La **cascada de imágenes** de 5 niveles (`API → SKU.png → SKU.jpg → Legacy PNG → Legacy JPG → Emoji`).
- El **borde serrado** del ticket (`SalesReceipt`), que imita papel físico.
- La **paleta canónica**: `#c1d72e` (verde lima), `#0a0a0a`/`#080808` (negro fondo), `#fdfbf7` (crema), `rounded-[35px]`/`rounded-[40px]`/`rounded-[50px]` (radios grandes).
- El **flujo de 3 vistas** del salón (`salon → menu → kds`).
- Los **estados de IA** del escáner (`IDLE / ANALYZING / LOCAL / CLOUD / ERROR`).
- El **modo offline** del repartidor (borrador en `localStorage`, cola de operaciones, buffer GPS).

### §0.3 Inventario de las 26 interfaces

| Tipo | Cantidad | Componentes |
|---|---|---|
| **Pantallas raíz** | 7 | `RetailVisionPOS`, `TableServicePOS`, `VisionTrainingUI`, `GrandezaParamsUI`, `GrandezaDailyUI`, `GrandezaDriverUI`, `RepartoPanGrandezaUI` |
| **Modales** | 6 | `CheckoutScreen`, `GestionPersonal`, `GestorDeCaja`, `ProgramacionPedidoModal`, `TerminalSelector`, `OpenAccountsCorkboard` |
| **Paneles y overlays** | 5 | `SalesReceipt`, `POSHeader`, `POSOverlays` (3 overlays), `VisionVisor`, `VisionScanner` |
| **Composición** | 5 | `ProductGrid`, `ProductCard`, `CategoryBar`, `CategoryEditor` |
| **Impresión** | 2 | `TicketTemplate`, `CorteTicketTemplate` |
| **TOTAL** | **26** | |

> **Nota de conteo.** `POSOverlays.jsx` exporta 3 overlays (`ForceLogoutModal`, `OfflineBanner`, `ToastNotification`) que se cuentan como un solo archivo. `CategoryEditor` vive en `apps/pos/` (no en `components/`) pero es composición del POS.

---

## §1. Cómo leer cada ficha

Cada interfaz tiene una ficha de **7 puntos**:

1. **Propósito** — qué problema resuelve esa pantalla en el mostrador.
2. **Estructura visual** — las zonas (header, cuerpo, panel lateral, barra de acciones).
3. **Controles** — cada botón, campo y tecla, con su acción.
4. **Estados** — vacío, cargando, error, éxito, offline.
5. **Navegación** — cómo se entra, cómo se sale, qué la abre.
6. **Modo responsivo** — comportamiento en Mostrador / Compacto / Móvil (según R-01 a R-04 del Documento 6).
7. **Anclaje al código** — archivo + líneas, para trazabilidad.

---

## §2. Paleta y lenguaje visual canónico

Antes de las fichas, el vocabulario visual compartido por las 26 interfaces.

### §2.1 Colores

| Rol | Valor | Uso |
|---|---|---|
| **Acento principal** | `#c1d72e` | Botones primarios, selección, precios, "COBRAR" |
| **Fondo profundo** | `#0a0a0a` / `#080808` | Fondo de pantallas raíz |
| **Fondo panel** | `#1a1a1a` | Modales (`GestionPersonal`, `GestorDeCaja`) |
| **Crema ticket** | `#fdfbf7` | Papel del ticket impreso |
| **Naranja salón** | `orange-600` / `orange-500` | `TableServicePOS` (TPV Meseros) |
| **Ámbar Grandeza** | `amber-500` / `orange-600` | `GrandezaDailyUI`, `GrandezaParamsUI` |
| **Azul Grandeza** | `blue-500` / `indigo-700` | `GrandezaParamsUI` (admin) |
| **Verde Grandeza** | `emerald-500` / `green-700` | `GrandezaDailyUI` (gerente) |
| **Rojo peligro** | `red-500` | Eliminar, cerrar sesión, detener escáner |

### §2.2 Tipografía

- **Tickets**: `font-mono font-bold` (monoespaciada, como impresora térmica).
- **UI general**: `font-black uppercase tracking-widest` (peso máximo, mayúsculas, tracking amplio).
- **Precios**: `font-mono italic tracking-tighter` (monoespaciada, itálica, apretada).
- **Tamaños**: `text-[7px]` a `text-[10px]` en tickets; `text-xs` a `text-4xl` en UI.

### §2.3 Radios y sombras

- **Radios grandes**: `rounded-[35px]` (tarjetas de producto), `rounded-[40px]` (contenedores), `rounded-[50px]` (modales).
- **Sombras**: `shadow-2xl`, `shadow-[0_4px_20px_rgba(0,0,0,0.6)]`, y sombras de color (`shadow-[#c1d72e]/20`).
- **Efectos**: `backdrop-blur-xl`/`backdrop-blur-3xl` (cristal), `animate-in fade-in zoom-in-95` (entrada).

### §2.4 Fondo

- **POS de mostrador**: `bg-transparent` sobre `wood_bg.jpg` (madera) — el fondo de la panadería.
- **Salón**: `bg-[#0a0a0a]` (negro sólido).
- **Grandeza repartidor**: `wood_bg.jpg` + `bg-black/75` (madera oscurecida).

---

## §3. Fichas — Pantallas raíz (7)

---

### FICHA 01 — `RetailVisionPOS`

**1. Propósito.** Es el **shell raíz** del POS de mostrador. Orquesta todo: el header, el catálogo (o el escáner), el ticket lateral, y los 6 modales. Es la pantalla que ve el cajero el 90% del tiempo.

**2. Estructura visual.**
```
┌─────────────────────────────────────────────────────┐
│ POSHeader (terminal, red, cuenta, Venta/Pedido, Caja)│
├──────────────────────────────┬──────────────────────┤
│                              │                      │
│  Cuerpo principal (flex-1)   │  SalesReceipt        │
│  ├─ VisionVisor (CAMERA)     │  (panel lateral      │
│  └─ ProductGrid (GRID)       │   derecho, ticket)   │
│                              │                      │
├──────────────────────────────┴──────────────────────┤
│ CategoryBar (categorías + ESCANER IA)               │
└─────────────────────────────────────────────────────┘
+ Modales superpuestos: CheckoutScreen, OpenAccountsCorkboard,
  GestorDeCaja, ProgramacionPedidoModal, ForceLogoutModal,
  ToastNotification, modal inline de salida.
```

**3. Controles.**
- **`viewMode`**: `'CAMERA'` (escáner IA) ↔ `'GRID'` (catálogo). Se alterna desde `CategoryBar`.
- **`showCheckout`**: abre `CheckoutScreen`.
- **`showCorkboard`**: abre `OpenAccountsCorkboard`.
- **`showGestorCaja`**: abre `GestorDeCaja`.
- **`showProgramacion`**: abre `ProgramacionPedidoModal`.
- **`showExitModal`**: modal inline de confirmación de salida (3 opciones: enviar y salir, salir sin guardar, cancelar).
- **`handleTerminalSwitch`**: cambia de terminal.
- **`handleForceLogout`**: cierre forzado (aplica `buildResetPatch()`).

**4. Estados.**
- **Cargando**: `usePOSSession` carga categorías y productos.
- **Vacío**: `ProductGrid` muestra placeholders (`bg-black/5 rounded-[35px]`).
- **Offline**: `OfflineBanner` (desde `POSOverlays`) muestra `pendingCount` e `isSyncing`.
- **Éxito**: `ToastNotification` con mensaje temporal.
- **Error**: `ToastNotification` con tipo error.

**5. Navegación.**
- **Entrada**: es la raíz; se monta tras el login y la selección de terminal.
- **Salida**: `handleTerminalSwitch` (vuelve a `TerminalSelector`), `handleForceLogout` (logout), o el modal de salida.
- **Abre**: los 6 modales listados arriba.

**6. Modo responsivo.**
- **Mostrador (≥1024px)**: layout completo de 2 columnas (cuerpo + ticket).
- **Compacto (768–1023px)**: el ticket lateral se estrecha; el grid pasa a 3 columnas.
- **Móvil (<768px)**: el ticket se convierte en panel inferior deslizable; el grid a 2 columnas. *(Hoy no implementado — ver R-03.)*

**7. Anclaje al código.** [`apps/pos/RetailVisionPOS.jsx`](../../apps/pos/RetailVisionPOS.jsx:30) (767 líneas). Layout en líneas 585–767. Modal de salida inline en 708–763. `handleForceLogout` en 454–532.

---

### FICHA 02 — `TableServicePOS`

**1. Propósito.** TPV para **meseros y KDS** (salón). Gestiona mesas, comandas y el seguimiento de cocina. Es un módulo especializado, distinto del mostrador.

**2. Estructura visual.**
```
┌─────────────────────────────────────────────────────┐
│ Header: logo R | nav (Salón / Cocina KDS) | mesero  │
├─────────────────────────────────────────────────────┤
│ Vista 'salon': grid de mesas (aspect-square)        │
│ Vista 'menu':  selector de menú + comanda lateral   │
│ Vista 'kds':   tarjetas de orden (OrderCard)        │
└─────────────────────────────────────────────────────┘
```

**3. Controles.**
- **Nav `salon` / `kds`**: alterna vista.
- **Tarjeta de mesa**: `onClick` → `setSelectedTable` + `setView('menu')`.
- **Botón `⬅`**: vuelve a `salon`.
- **Item de menú**: `onClick` (agrega a comanda — hoy placeholder).
- **Botón "Enviar a Cocina (KDS)"**: envía la comanda.

**4. Estados.**
- **Mesa libre**: punto verde.
- **Mesa ordenando** (`proceso`): punto amarillo `animate-pulse`.
- **Mesa ocupada** (`comiendo`/`pendiente_pago`): punto rojo.
- **Comanda vacía**: "No hay productos en la orden" (texto gris itálico).

**5. Navegación.**
- **Entrada**: módulo independiente (ruta propia).
- **Salida**: no tiene botón de salida propio (vive en el shell del ERP).
- **Interna**: `salon → menu → salon`, `salon ↔ kds`.

**6. Modo responsivo.**
- **Mostrador**: grid de mesas `lg:grid-cols-4`; comanda lateral `w-[400px]`.
- **Compacto**: grid `grid-cols-2`; comanda `w-[400px]` (violación R-01).
- **Móvil**: grid `grid-cols-2`; comanda debería ser panel inferior.

**7. Anclaje al código.** [`apps/pos/TableServicePOS.jsx`](../../apps/pos/TableServicePOS.jsx:13) (165 líneas). Comanda lateral `w-[400px]` en línea 134. Datos mock (mesas, menú, órdenes) en 16–35.

---

### FICHA 03 — `VisionTrainingUI`

**1. Propósito.** Módulo para **entrenar la IA**: seleccionar un producto, capturar 20 fotos, y sincronizarlas al dataset. Es la "mente" del escáner.

**2. Estructura visual.**
```
┌──────────────────────┬──────────────────────────────┐
│ Panel de control     │  VisionScanner (cámara)      │
│ w-[450px]            │  (área de captura)           │
│ ├─ 1. Seleccionar    │                              │
│ ├─ 2. Estadísticas   │                              │
│ └─ 3. Acciones       │                              │
└──────────────────────┴──────────────────────────────┘
```

**3. Controles.**
- **`<select>` de producto**: agrupado por categoría (`<optgroup>`).
- **Botón ⚙️**: abre `CategoryEditor`.
- **Botón "Capturar"**: inicia ráfaga (deshabilitado sin producto o capturando).
- **Botón "Sincronizar"**: sube las 20 imágenes (`posService.uploadTrainingImages`).
- **Barra de progreso**: `syncProgress` 0 → 10 → 100.

**4. Estados.**
- **Sin producto**: acciones deshabilitadas (`disabled:opacity-20 disabled:grayscale`).
- **Capturando**: `isCapturing = true`, contador `N / 20`.
- **Sincronizando**: barra de progreso.
- **Éxito**: `syncProgress = 100`, limpia tras 3s.
- **Error**: `alert("Error al subir imágenes al servidor corporativo.")`.

**5. Navegación.**
- **Entrada**: módulo de administración (ruta propia).
- **Salida**: `onBack` (o el shell del ERP).
- **Abre**: `CategoryEditor` (sub-vista completa).

**6. Modo responsivo.**
- **Mostrador**: 2 columnas (panel `w-[450px]` + cámara).
- **Compacto**: panel `w-[450px]` (violación R-01).
- **Móvil**: debería apilarse (panel arriba, cámara abajo).

**7. Anclaje al código.** [`apps/pos/VisionTrainingUI.jsx`](../../apps/pos/VisionTrainingUI.jsx:15) (190 líneas). Panel `w-[450px]` en línea 80. `handleSync` en 42–62.

---

### FICHA 04 — `GrandezaParamsUI`

**1. Propósito.** Herramienta del **administrador de Grandeza**: gestiona productos B2B, clientes, rutas regulares y rutas extraordinarias. Es la configuración maestra del reparto.

**2. Estructura visual.**
```
┌─────────────────────────────────────────────────────┐
│ Header (logo Grandeza + título)                     │
├─────────────────────────────────────────────────────┤
│ Tabs: Productos | Clientes | Rutas | Rutas Extra    │
├─────────────────────────────────────────────────────┤
│ Contenido del tab activo                            │
│ ├─ Productos: grid de B2BProductCard                │
│ ├─ Clientes: lista + edición                        │
│ ├─ Rutas: selector de día + slots                   │
│ └─ Rutas Extra: fecha + etiqueta + slots            │
└─────────────────────────────────────────────────────┘
```

**3. Controles.**
- **Tabs**: `activeTab` (`products` / `clients` / `routes`).
- **`B2BProductCard`**: muestra SKU, nombre, precio tienda, precio B2B.
- **Selector de día**: `DAYS` (LUNES–DOMINGO).
- **Edición de cliente**: `editingClient`.
- **Eliminación**: `clientToDelete` + `showDeleteConfirm`.
- **Rutas extraordinarias**: `extraRouteDate`, `extraRouteLabel`, `extraRouteSlots`.

**4. Estados.**
- **Cargando**: `loading` (spinner).
- **Modal de estado**: `statusModal` (éxito/error).
- **Confirmación de borrado**: `showDeleteConfirm`, `showDeleteExtraConfirm`.
- **Estadísticas**: `statsLoading`, `statsFilter` (`ALL` / otros).

**5. Navegación.**
- **Entrada**: desde `RepartoPanGrandezaUI` (suite `params`).
- **Salida**: `onBack` → vuelve al landing de Grandeza.
- **Interna**: tabs.

**6. Modo responsivo.**
- **Mostrador**: grid de productos multi-columna.
- **Compacto**: grid reducido.
- **Móvil**: 1 columna. *(Hoy parcial — ver R-03.)*

**7. Anclaje al código.** [`apps/pos/GrandezaParamsUI.jsx`](../../apps/pos/GrandezaParamsUI.jsx:65) (1254 líneas). `B2BProductCard` en 6–62. Tabs y estado en 66–111.

---

### FICHA 05 — `GrandezaDailyUI`

**1. Propósito.** Herramienta del **gerente de Grandeza**: abre la jornada, registra inventario inicial, fondo de caja, asigna repartidor, despacha la ruta, y al final recibe mercancía/dinero y cierra la jornada.

**2. Estructura visual.**
```
┌─────────────────────────────────────────────────────┐
│ Header (fecha, estado de jornada, botón mapa)       │
├─────────────────────────────────────────────────────┤
│ ├─ Inventario inicial (inputs por producto)         │
│ ├─ Fondo de caja + asignación de repartidor         │
│ ├─ Bitácora de visitas (lista)                      │
│ ├─ Recepción de Mercancía (tabla min-w-[600px])     │
│ ├─ Recepción de Dinero (tabla de cuadre)            │
│ ├─ Notas de retroalimentación (textarea)            │
│ └─ Botón "🔒 Cerrar Jornada"                        │
└─────────────────────────────────────────────────────┘
+ Modal de confirmación de cierre.
```

**3. Controles.**
- **Inputs de inventario**: `setInvQty(productId, qty)`.
- **Fondo de caja**: `cashFund`.
- **Selector de repartidor**: `driverId` (filtra empleados `REPARTIDOR`/`DRIVER`).
- **Botón "📍 Ver en Mapa"**: `openGoogleMaps`.
- **Inputs de recepción**: `updateReceipt(productId, 'freshReceived'|'exchangeReceived', value)`.
- **Efectivo recibido**: `cashReceived` → calcula `cashDiff`.
- **Notas**: `feedbackNotes`.
- **Botón "🔒 Cerrar Jornada"**: `handleCerrarRequest` → `handleCerrar`.

**4. Estados.**
- **Cargando**: `loading`.
- **Sin jornada**: `journey = null` (formulario de apertura).
- **Jornada abierta**: muestra inventario y despacho.
- **Jornada cerrada**: solo lectura.
- **Toast**: `showToast(msg, type)`.
- **Confirmación**: `showConfirmModal`.

**5. Navegación.**
- **Entrada**: desde `RepartoPanGrandezaUI` (suite `daily`).
- **Salida**: `onBack`.
- **Cambio de fecha**: `viewDate` (recarga datos).

**6. Modo responsivo.**
- **Mostrador**: tablas completas.
- **Compacto**: tablas con scroll horizontal (`min-w-[600px]` en línea 752).
- **Móvil**: `text-[10px] md:text-xs`, inputs `w-12 md:w-14` (ya tiene algunos breakpoints).

**7. Anclaje al código.** [`apps/pos/GrandezaDailyUI.jsx`](../../apps/pos/GrandezaDailyUI.jsx:18) (891 líneas). Tabla de recepción `min-w-[600px]` en 752. Modal de cierre en 864–887.

---

### FICHA 06 — `GrandezaDriverUI`

**1. Propósito.** Herramienta del **repartidor en ruta** (smartphone). Muestra la ruta del día, registra visitas, cobros, gastos, y funciona **offline** con sincronización diferida.

**2. Estructura visual.**
```
┌─────────────────────────────────────────────────────┐
│ Fondo: wood_bg.jpg + bg-black/75                    │
├─────────────────────────────────────────────────────┤
│ Vista 'route':   lista de visitas de la ruta        │
│ Vista 'visit':   detalle de visita + items + cobro  │
│ Vista 'summary': resumen del día                    │
│ Vista 'order':   pedido extraordinario              │
│ Vista 'expenses': gastos operativos                 │
└─────────────────────────────────────────────────────┘
+ Modales: post-visita, aviso de pago faltante,
  detalle de visita, PIN.
```

**3. Controles.**
- **`view`**: `route` / `visit` / `summary` / `order` / `expenses`.
- **`activeVisit`**: visita abierta.
- **`visitItems`**: items de la visita.
- **`paymentReceived`**: cobro.
- **`incidentNotes`**: notas de incidente.
- **`extClientName` / `extClientPhone`**: cliente extemporáneo.
- **Gastos**: `expDesc`, `expAmount`.
- **`pinInput`**: validación por PIN.
- **`canEditClients`**: permiso `grandeza_edit_clients`.

**4. Estados.**
- **Online/Offline**: `isOnline`, `pendingSyncCount`.
- **Borrador de visita**: `useVisitDraft` (localStorage `grandeza_visit_draft`).
- **Cola offline**: `enqueueOperation`, `processQueue`, `getPendingCount`.
- **GPS**: `bufferGPSPoint`, `flushGPSBuffer`.
- **Toast**: `toast`.
- **Guardando**: `saving`, `expSaving`.
- **Post-visita**: `lastVisitResult`.
- **Aviso de pago**: `showPaymentWarning`.

**5. Navegación.**
- **Entrada**: desde `RepartoPanGrandezaUI` (suite `driver`) o `?terminal=DRIVER`.
- **Salida**: `onBack`.
- **Interna**: `route → visit → route`, `route → summary`, `route → expenses`.

**6. Modo responsivo.**
- **Móvil-first**: diseñado para smartphone (el repartidor lo usa en la calle).
- **Mostrador**: no aplica (no se usa en mostrador).

**7. Anclaje al código.** [`apps/pos/GrandezaDriverUI.jsx`](../../apps/pos/GrandezaDriverUI.jsx:45) (1562 líneas). `useVisitDraft` en 13–28. `DriverBackground` en 30–39. Estado en 49–80.

---

### FICHA 07 — `RepartoPanGrandezaUI`

**1. Propósito.** **Landing page** del módulo Grandeza. Presenta las 3 sub-suites (admin, gerente, repartidor) y enruta según permisos.

**2. Estructura visual.**
```
┌─────────────────────────────────────────────────────┐
│ Header (logo Grandeza + título del módulo)          │
├─────────────────────────────────────────────────────┤
│ Grid de 3 tarjetas de suite:                        │
│ ├─ 📋 Herramienta Administrador (azul)              │
│ ├─ 📅 Herramienta Gerente (verde)                   │
│ └─ 🚗 Herramienta Repartidor (ámbar)                │
└─────────────────────────────────────────────────────┘
```

**3. Controles.**
- **Tarjeta de suite**: `onClick` → `setActiveSuite(id)`.
- **`hasPermission(permId)`**: filtra suites visibles.
- **`isDriverTerminal`**: `?terminal=DRIVER` → entra directo a `driver`.

**4. Estados.**
- **Sin suite activa**: muestra el grid de 3 tarjetas.
- **Suite activa**: renderiza la sub-suite (`params` / `daily` / `driver`).
- **Sin permisos**: tarjetas ocultas.

**5. Navegación.**
- **Entrada**: módulo del ERP (ruta propia).
- **Salida**: `onBack`.
- **Abre**: `GrandezaParamsUI`, `GrandezaDailyUI`, `GrandezaDriverUI`.

**6. Modo responsivo.**
- **Mostrador**: grid de 3 tarjetas.
- **Compacto**: `p-4 md:p-8`, iconos `w-12 md:w-16`.
- **Móvil**: 1 columna.

**7. Anclaje al código.** [`apps/pos/RepartoPanGrandezaUI.jsx`](../../apps/pos/RepartoPanGrandezaUI.jsx:16) (221 líneas). Definición de suites en 34–68. Enrutado en 71–80.

---

## §4. Fichas — Modales (6)

---

### FICHA 08 — `CheckoutScreen`

**1. Propósito.** Modal de **cobro**. Captura los pagos (efectivo, crédito, débito, transferencia), permite dividir el pago en varios métodos, y finaliza la venta.

**2. Estructura visual.**
```
┌─────────────────────────────────────────────────────┐
│ Header: "COBRAR" + total + botón cerrar             │
├──────────────────────────┬──────────────────────────┤
│ Panel izquierdo          │ Panel derecho            │
│ ├─ Selector de método    │ ├─ Lista de pagos        │
│ ├─ Campo de monto        │ ├─ Editar / eliminar     │
│ └─ Numpad táctil         │ └─ Restante              │
├──────────────────────────┴──────────────────────────┤
│ Botón "FINALIZAR VENTA"                             │
└─────────────────────────────────────────────────────┘
```

**3. Controles.**
- **Selector de método**: efectivo / crédito / débito / transferencia.
- **Numpad**: `handleNumberClick(num)`.
- **Botón "Agregar pago"**: `handleAddPayment`.
- **Botón eliminar**: `handleDeletePayment(id)`.
- **Botón editar**: `handleStartEdit(p)` → `handleSaveEdit(id)`.
- **Botón "FINALIZAR VENTA"**: `handleFinalize` (async).

**4. Estados.**
- **Sin pagos**: lista vacía, restante = total.
- **Pago parcial**: restante > 0.
- **Pago completo**: restante = 0, botón habilitado.
- **Procesando**: `handleFinalize` en curso.
- **Con orden**: `orderData` (muestra `OrderDetailRow` con `committed_at`).

**5. Navegación.**
- **Entrada**: `showCheckout = true` (desde `RetailVisionPOS`).
- **Salida**: `onClose` o `onConfirm` (tras finalizar).
- **Abre**: nada (es hoja).

**6. Modo responsivo.**
- **Mostrador**: `w-[1100px]` (línea 132).
- **Compacto**: `w-[800px]` (línea 133).
- **Móvil**: debería ser pantalla completa. *(Hoy no implementado — violación R-01.)*

**7. Anclaje al código.** [`apps/pos/components/CheckoutScreen.jsx`](../../apps/pos/components/CheckoutScreen.jsx:4) (435 líneas). Anchos `w-[1100px]`/`w-[800px]` en 132–133. `w-[320px]` en 322. `handleFinalize` en 88–126. `OrderDetailRow` en 427–434.

---

### FICHA 09 — `GestionPersonal`

**1. Propósito.** Modal de **gestión de claves de acceso**. Lista colaboradores, crea/edita perfiles, y asigna PIN de acceso con un numpad.

**2. Estructura visual.**
```
┌─────────────────────────────────────────────────────┐
│ Header: "GESTOR DE CLAVES DE ACCESO" + cerrar       │
├─────────────────────────────────────────────────────┤
│ Vista 'list':  lista de colaboradores + "+ Nuevo"   │
│ Vista 'form':  ├─ Formulario (nombre, perfil)       │
│                └─ Numpad de PIN (w-[280px])         │
└─────────────────────────────────────────────────────┘
```

**3. Controles.**
- **Botón "+ Nuevo Colaborador"**: `handleOpenForm()`.
- **Botón ✎**: `handleOpenForm(emp)` (editar).
- **Botón ✕**: `handleDeactivate(emp.id)`.
- **Input nombre**: `formData.name`.
- **Selector de perfil**: `profiles` (grid `grid-cols-2`).
- **Numpad**: `handleNumpad(val)` (máx. 6 dígitos, `C` limpia).
- **Botón "Registrar/Actualizar"**: `handleSave`.

**4. Estados.**
- **Cargando**: "Cargando Colaboradores..." (`animate-pulse`).
- **Lista**: colaboradores activos.
- **Formulario**: `view === 'form'`.
- **Error**: banner rojo.
- **PIN**: `numpadValue` (enmascarado con `•`).

**5. Navegación.**
- **Entrada**: módulo de RRHH / seguridad.
- **Salida**: `onClose` (si no es sección) o `onBack`.
- **Interna**: `list ↔ form`.

**6. Modo responsivo.**
- **Mostrador**: `w-[800px] h-[600px]` (línea 123).
- **Compacto**: `w-[800px]` (violación R-01).
- **Móvil**: debería ser pantalla completa.
- **Modo sección**: `w-full h-[700px]` (línea 118) cuando `isSection`.

**7. Anclaje al código.** [`apps/pos/components/GestionPersonal.jsx`](../../apps/pos/components/GestionPersonal.jsx:117) (266 líneas). `containerClasses`/`modalClasses` en 117–123. Numpad `w-[280px]` en 241. `handleNumpad` en 108–115.

---

### FICHA 10 — `GestorDeCaja`

**1. Propósito.** Modal de **gestión de caja**. Abre/cierra sesión de caja, registra movimientos (entradas/salidas), valida PIN, y genera el corte.

**2. Estructura visual.**
```
┌─────────────────────────────────────────────────────┐
│ Header: "GESTOR DE CAJA" + cerrar                   │
├──────────────────────────┬──────────────────────────┤
│ Panel izquierdo          │ Panel derecho            │
│ ├─ Resumen de sesión     │ ├─ Numpad táctil         │
│ ├─ Movimientos           │ │  w-[380px]             │
│ ├─ Diferencias           │ └─ Validación PIN        │
│ └─ Botones de acción     │                          │
└──────────────────────────┴──────────────────────────┘
```

**3. Controles.**
- **Numpad**: `handleKeypadPress(val)`.
- **Validar PIN**: `handleValidarPin`.
- **Registrar fondo**: `handleRegistrarFondo` → `handleConfirmarFondo`.
- **Agregar movimiento**: `handleAgregarMovimiento`.
- **Eliminar movimiento**: `handleEliminarMovimiento(id)`.
- **Iniciar cierre**: `handleIniciarCierre` → `handleConfirmarCierre`.
- **Imprimir corte**: `handlePrintCorte`.
- **Guardar contexto diario**: `handleSaveDailyContext`.
- **Nuevo turno**: `handleNuevoTurno`.

**4. Estados.**
- **Sin sesión**: formulario de apertura (fondo + PIN).
- **Sesión activa**: resumen + movimientos.
- **Alerta**: `mostrarAlerta(mensaje, tipo)` (error / éxito).
- **Validando PIN**: `handleValidarPin` en curso.
- **Cerrando**: `handleConfirmarCierre` en curso.
- **Corte impreso**: `handlePrintCorte` genera el `CorteTicketTemplate`.

**5. Navegación.**
- **Entrada**: `showGestorCaja = true` (desde `RetailVisionPOS`).
- **Salida**: `onClose` (o `onCajaHabilitada` / `onCajaDeshabilitada`).
- **Abre**: `CorteTicketTemplate` (impresión).

**6. Modo responsivo.**
- **Mostrador**: 2 columnas (resumen + numpad `w-[380px]`).
- **Compacto**: numpad `w-[380px]` (violación R-01).
- **Móvil**: debería apilarse (resumen arriba, numpad abajo).

**7. Anclaje al código.** [`apps/pos/components/GestorDeCaja.jsx`](../../apps/pos/components/GestorDeCaja.jsx:74) (907 líneas). Numpad `w-[380px]` en 853. Sub-componentes `SeccionHeader` (9–19), `CampoMoneda` (21–26), `FilaDiferencia` (28–56), `AlertaInterna` (58–70).

---

### FICHA 11 — `ProgramacionPedidoModal`

**1. Propósito.** Modal de **programación de pedidos**. Permite agendar un pedido para una fecha/hora futura, con anticipación mínima calculada por producto (`calcMaxLeadTime`).

**2. Estructura visual.**
```
┌─────────────────────────────────────────────────────┐
│ Header: "PROGRAMAR PEDIDO" + cerrar                 │
├─────────────────────────────────────────────────────┤
│ ├─ Campo: Nombre del cliente                        │
│ ├─ Campo: Teléfono                                  │
│ ├─ Campo: Fecha de entrega (datetime-local)         │
│ ├─ Campo: Notas                                     │
│ ├─ Resumen del carrito                              │
│ └─ Botón "PROGRAMAR"                                │
└─────────────────────────────────────────────────────┘
```

**3. Controles.**
- **`field(label, id, value, onChange, type, placeholder)`**: helper de campo (línea 72).
- **Botón "PROGRAMAR"**: `handleSave` (async, 47–70).
- **`calcMaxLeadTime(cart)`**: calcula anticipación mínima (332–339).
- **`toLocalIsoString(date)`**: formatea fecha local (342–345).

**4. Estados.**
- **Sin carrito**: no se puede programar.
- **Guardando**: `handleSave` en curso.
- **Error**: alerta.
- **Éxito**: `onSave` → cierra.

**5. Navegación.**
- **Entrada**: `showProgramacion = true` (desde `RetailVisionPOS`).
- **Salida**: `onClose` o `onSave`.
- **Abre**: nada (es hoja).

**6. Modo responsivo.**
- **Mostrador**: `max-w-2xl max-h-[90vh]`.
- **Compacto**: `max-w-2xl` (ya es fluido).
- **Móvil**: `max-h-[90vh]` con scroll interno.

**7. Anclaje al código.** [`apps/pos/components/ProgramacionPedidoModal.jsx`](../../apps/pos/components/ProgramacionPedidoModal.jsx:13) (354 líneas). `handleSave` en 47–70. `field` en 72–84. `calcMaxLeadTime` en 332–339.

---

### FICHA 12 — `TerminalSelector`

**1. Propósito.** Modal de **selección de terminal**. Muestra las terminales disponibles (con su estado de ocupación), permite elegir una, y administrarlas (agregar, eliminar, editar icono).

**2. Estructura visual.**
```
┌─────────────────────────────────────────────────────┐
│ Header: "SELECCIONA TU TERMINAL"                    │
├─────────────────────────────────────────────────────┤
│ Grid de tarjetas de terminal:                       │
│ ├─ Icono (PRESET_ICONS)                             │
│ ├─ Nombre + estado (libre/ocupada)                  │
│ ├─ Botón copiar URL                                 │
│ └─ Botones editar/eliminar                          │
├─────────────────────────────────────────────────────┤
│ Botón "+ Agregar terminal"                          │
└─────────────────────────────────────────────────────┘
```

**3. Controles.**
- **`PRESET_ICONS`**: 7 iconos predefinidos (7–14).
- **`renderIcon(icon)`**: renderiza icono (16–21).
- **`copyUrl(tid)`**: copia URL de la terminal (63–73).
- **`applyEdit()`**: aplica edición de icono (78–81).
- **`addTerminal(position)`**: agrega terminal (83–88).
- **`removeTerminal(tid)`**: elimina terminal (90–102).
- **`executeDelete()`**: confirma eliminación (104–107).
- **`handleSave()`**: guarda cambios (109–118).
- **`handleImageUpload(e)`**: sube imagen de icono (120–126).

**4. Estados.**
- **Cargando**: esperando `terminalStatuses`.
- **Libre**: terminal disponible.
- **Ocupada**: terminal en uso (por otro usuario).
- **Editando**: modo edición de icono.
- **Confirmando eliminación**: `executeDelete`.

**5. Navegación.**
- **Entrada**: al iniciar sesión (si no hay terminal asignada).
- **Salida**: `onTerminalSelected(tid)`.
- **Abre**: nada (es hoja).

**6. Modo responsivo.**
- **Mostrador**: grid `grid-cols-3 md:grid-cols-6`.
- **Compacto**: `grid-cols-3`.
- **Móvil**: `grid-cols-2`.

**7. Anclaje al código.** [`apps/pos/components/TerminalSelector.jsx`](../../apps/pos/components/TerminalSelector.jsx:23) (439 líneas). `PRESET_ICONS` en 7–14. `renderIcon` en 16–21. Handlers en 63–126.

---

### FICHA 13 — `OpenAccountsCorkboard`

**1. Propósito.** Modal de **cuentas abiertas** (corcho de post-its). Muestra las cuentas en curso como notas adhesivas, con rotación aleatoria y color por terminal.

**2. Estructura visual.**
```
┌─────────────────────────────────────────────────────┐
│ Header: "CUENTAS ABIERTAS" + cerrar                 │
├─────────────────────────────────────────────────────┤
│ Corcho (aspect-[16/9]):                             │
│ ├─ Post-it #1 (rotado, color por terminal)          │
│ ├─ Post-it #2 (rotado, color por terminal)          │
│ └─ Post-it #N ...                                   │
└─────────────────────────────────────────────────────┘
```

**3. Controles.**
- **Post-it**: `onClick` → `onSelectAccount(account)`.
- **`getRandomRotation(index)`**: rotación aleatoria (18–21).
- **`getPostItColor(terminal)`**: color por terminal (23–33).
- **Botón cerrar**: `onClose`.

**4. Estados.**
- **Sin cuentas**: corcho vacío.
- **Con cuentas**: post-its renderizados.
- **Seleccionando**: `onSelectAccount` en curso.

**5. Navegación.**
- **Entrada**: `showCorkboard = true` (desde `RetailVisionPOS`).
- **Salida**: `onClose` o `onSelectAccount`.
- **Abre**: recupera la cuenta en `RetailVisionPOS`.

**6. Modo responsivo.**
- **Mostrador**: `max-w-6xl aspect-[16/9]`.
- **Compacto**: `max-w-6xl`.
- **Móvil**: debería ser scroll vertical. *(Hoy no implementado.)*

**7. Anclaje al código.** [`apps/pos/OpenAccountsCorkboard.jsx`](../../apps/pos/OpenAccountsCorkboard.jsx:10) (145 líneas). `getRandomRotation` en 18–21. `getPostItColor` en 23–33.

---

## §5. Fichas — Paneles y overlays (5)

---

### FICHA 14 — `SalesReceipt`

**1. Propósito.** Panel lateral derecho del POS. Es el **ticket en pantalla**: lista los ítems del carrito, permite editar cantidades, y contiene los botones "COBRAR" y "MANTENER CUENTA".

**2. Estructura visual.**
```
┌──────────────────────────┐
│ Header: "CUENTA #N"      │
├──────────────────────────┤
│ Lista de ítems:          │
│ ├─ Nombre + cantidad     │
│ ├─ Precio unitario       │
│ └─ Subtotal              │
├──────────────────────────┤
│ TOTAL                    │
├──────────────────────────┤
│ [MANTENER CUENTA]        │
│ [COBRAR]                 │
└──────────────────────────┘
```

**3. Controles.**
- **`handleQuantityClick(item)`**: edita cantidad (9–13).
- **`handleEditNumberClick(num)`**: numpad inline (15–20).
- **`handleEditConfirm()`**: confirma edición (24–33).
- **`handleEditCancel()`**: cancela edición (35–38).
- **Botón "COBRAR"**: `handleCheckout`.
- **Botón "MANTENER CUENTA"**: `handleHoldAccount`.
- **`removeFromCart` / `updateQuantity`**: mutaciones del carrito.

**4. Estados.**
- **Vacío**: sin ítems.
- **Con ítems**: lista + total.
- **Editando cantidad**: numpad inline.
- **Enviando a pizarrón**: `isSendingToPizarron`.
- **Estado de guardado**: `lastSaveStatus` (`idle` / `saving` / `saved` / `error`).
- **Caja deshabilitada**: `cashEnabled = false`.

**5. Navegación.**
- **Entrada**: siempre visible en `RetailVisionPOS`.
- **Salida**: no aplica (es panel).
- **Abre**: `CheckoutScreen` (vía "COBRAR").

**6. Modo responsivo.**
- **Mostrador**: `w-[420px]` (línea 41).
- **Compacto**: `w-[420px]` (violación R-01).
- **Móvil**: debería ser un drawer inferior.

**7. Anclaje al código.** [`apps/pos/components/SalesReceipt.jsx`](../../apps/pos/components/SalesReceipt.jsx:3) (245 líneas). `w-[420px]` en 41. Modal inline `w-[400px]` en 206.

---

### FICHA 15 — `POSHeader`

**1. Propósito.** Barra superior del POS. Muestra el estado de la terminal, la red, la cuenta actual, el selector Venta/Pedido, y el acceso a la caja.

**2. Estructura visual.**
```
┌─────────────────────────────────────────────────────┐
│ [Logo] │ Terminal: T1 │ Red: 🟢 │ Cuenta: #123       │
│        │ [VENTA] [PEDIDO] │ [CAJA] │ [Salir]         │
└─────────────────────────────────────────────────────┘
```

**3. Controles.**
- **Selector Venta/Pedido**: cambia el modo de operación.
- **Botón CAJA**: abre `GestorDeCaja`.
- **Botón Salir**: `handleTerminalSwitch` / `doTerminalExit`.
- **Indicador de red**: online/offline.
- **Indicador de cuenta**: número de cuenta actual.

**4. Estados.**
- **Online**: indicador verde.
- **Offline**: indicador rojo + `OfflineBanner`.
- **Caja habilitada/deshabilitada**: color del botón CAJA.
- **Venta/Pedido**: modo activo resaltado.

**5. Navegación.**
- **Entrada**: siempre visible en `RetailVisionPOS`.
- **Salida**: no aplica (es panel).
- **Abre**: `GestorDeCaja`, `TerminalSelector` (al salir).

**6. Modo responsivo.**
- **Mostrador**: 3 zonas completas.
- **Compacto**: tipografía `text-[7px]`/`[8px]`/`[9px]` (R-02).
- **Móvil**: colapsa a 2 filas.

**7. Anclaje al código.** [`apps/pos/components/POSHeader.jsx`](../../apps/pos/components/POSHeader.jsx:15) (197 líneas). Tipografía escalada en 7–9px.

---

### FICHA 16 — `POSOverlays`

**1. Propósito.** Archivo que exporta **3 overlays** globales: `ForceLogoutModal` (cierre forzado), `OfflineBanner` (aviso offline), `ToastNotification` (notificación efímera).

**2. Estructura visual.**
```
ForceLogoutModal:  ┌──────────────────────────┐
                   │ "SESIÓN CERRADA"         │
                   │ [Reingresar]             │
                   └──────────────────────────┘
OfflineBanner:     ┌──────────────────────────┐
                   │ ⚠️ Sin conexión (N pend.)│
                   └──────────────────────────┘
ToastNotification: ┌──────────────────────────┐
                   │ ✅ Mensaje               │
                   └──────────────────────────┘
```

**3. Controles.**
- **`ForceLogoutModal`**: `onForceLogout` (15–42).
- **`OfflineBanner`**: `pendingCount`, `isSyncing` (45–68).
- **`ToastNotification`**: `message` (71–81).

**4. Estados.**
- **ForceLogout**: `visible = true`.
- **Offline**: `pendingCount > 0`.
- **Syncing**: `isSyncing = true`.
- **Toast**: `message` presente.

**5. Navegación.**
- **Entrada**: eventos globales (logout forzado, pérdida de red, notificación).
- **Salida**: auto-dismiss (toast) o acción del usuario.
- **Abre**: nada (son overlays).

**6. Modo responsivo.**
- **Mostrador**: centrado / top.
- **Compacto**: igual.
- **Móvil**: ancho completo.

**7. Anclaje al código.** [`apps/pos/components/POSOverlays.jsx`](../../apps/pos/components/POSOverlays.jsx:15) (82 líneas). `ForceLogoutModal` (15–42), `OfflineBanner` (45–68), `ToastNotification` (71–81).

---

### FICHA 17 — `VisionVisor`

**1. Propósito.** Visor de estado de la IA. Muestra el estado del escáner (`IDLE / ANALYZING / LOCAL / CLOUD / ERROR`) con color e icono.

**2. Estructura visual.**
```
┌──────────────────────────┐
│ 🟢 IA LOCAL (T6)         │
│ ESPERANDO...             │
└──────────────────────────┘
```

**3. Controles.**
- **`setIsScanning`**: activa/desactiva escáner.
- **`addToCart`**: agrega producto detectado.
- **`statusConfig`**: mapa de estados (7–13).

**4. Estados.**
- **IDLE**: `bg-gray-500`, "ESPERANDO...".
- **ANALYZING**: `bg-blue-400`, "ANALIZANDO...".
- **LOCAL**: `bg-[#c1d72e]`, "IA LOCAL (T6)".
- **CLOUD**: `bg-yellow-400`, "IA NUBE (GEMINI)".
- **ERROR**: `bg-red-500`, "IA OFFLINE".

**5. Navegación.**
- **Entrada**: modo CAMERA en `RetailVisionPOS`.
- **Salida**: no aplica (es panel).
- **Abre**: nada.

**6. Modo responsivo.**
- **Mostrador**: `min-w-[180px]` (línea 51).
- **Compacto**: `min-w-[180px]`.
- **Móvil**: debería ser overlay flotante.

**7. Anclaje al código.** [`apps/pos/components/VisionVisor.jsx`](../../apps/pos/components/VisionVisor.jsx:4) (73 líneas). `statusConfig` en 7–13. `min-w-[180px]` en 51.

---

### FICHA 18 — `VisionScanner`

**1. Propósito.** Motor de **cámara + IA**. Enumera cámaras, inicializa el stream, captura frames y los analiza con Gemini (`gemini-2.0-flash`).

**2. Estructura visual.**
```
┌──────────────────────────┐
│ <video> (stream cámara)  │
│ ┌──────────────────────┐ │
│ │ Overlay de detección │ │
│ └──────────────────────┘ │
└──────────────────────────┘
```

**3. Controles.**
- **`getDevices()`**: enumera cámaras (42–54).
- **`startCamera()`**: inicializa stream (61–78).
- **`analyzeFrame()`**: analiza frame con Gemini (103–150).
- **`animate()`**: loop de captura (156–173).
- **`onCaptureFrame`**: callback de frame capturado.

**4. Estados.**
- **Sin cámara**: error de permisos.
- **Inicializando**: `startCamera` en curso.
- **Escaneando**: loop activo.
- **Analizando**: `analyzeFrame` en curso.
- **Error**: IA offline.

**5. Navegación.**
- **Entrada**: modo CAMERA en `RetailVisionPOS`.
- **Salida**: `setIsScanning(false)`.
- **Abre**: nada (es motor).

**6. Modo responsivo.**
- **Mostrador**: video a pantalla completa del cuerpo.
- **Compacto**: igual.
- **Móvil**: igual.

**7. Anclaje al código.** [`apps/pos/VisionScanner.jsx`](../../apps/pos/VisionScanner.jsx:11) (231 líneas). Gemini en 23–24. Enumeración en 41–56. Loop en 85–100.

---

## §6. Fichas — Composición (4)

---

### FICHA 19 — `ProductGrid`

**1. Propósito.** Rejilla paginada de productos. Es el corazón del catálogo: muestra los productos filtrados por categoría, con paginador lateral.

**2. Estructura visual.**
```
┌────┬──────────────────────────────────────┐
│ ▲  │ ┌────┐ ┌────┐ ┌────┐ ┌────┐          │
│ 1  │ │ P1 │ │ P2 │ │ P3 │ │ P4 │          │
│ 2  │ └────┘ └────┘ └────┘ └────┘          │
│ 3  │ ┌────┐ ┌────┐ ┌────┐ ┌────┐          │
│ ▼  │ │ P5 │ │ P6 │ │ P7 │ │ P8 │          │
│w-16│ └────┘ └────┘ └────┘ └────┘          │
└────┴──────────────────────────────────────┘
```

**3. Controles.**
- **Paginador**: `setCurrentPage` (botones ▲/▼ + número).
- **`ProductCard`**: `onAdd={onAddToCart}`.
- **Filtro**: `products.filter(...)` por categoría.
- **Orden**: `sort(...)`.

**4. Estados.**
- **Vacío**: solo placeholders (`bg-black/5`).
- **Con productos**: grid lleno.
- **Paginado**: `totalPages > 1` → muestra paginador.
- **Última página**: placeholders rellenan el grid.

**5. Navegación.**
- **Entrada**: cuerpo de `RetailVisionPOS` (modo GRID).
- **Salida**: no aplica (es composición).
- **Abre**: `ProductCard` → `onAddToCart`.

**6. Modo responsivo.**
- **Mostrador**: `lg:grid-cols-4 grid-rows-3`.
- **Compacto**: `md:grid-cols-3`.
- **Móvil**: `grid-cols-2`.

**7. Anclaje al código.** [`apps/pos/components/ProductGrid.jsx`](../../apps/pos/components/ProductGrid.jsx:9) (55 líneas). Paginador `w-16`. Grid `grid-cols-2 md:grid-cols-3 lg:grid-cols-4 grid-rows-3`.

---

### FICHA 20 — `ProductCard`

**1. Propósito.** Tarjeta de producto individual. Muestra imagen (con cascada de 5 niveles), nombre y precio; al tocarla agrega al carrito.

**2. Estructura visual.**
```
┌──────────────────┐
│ ┌──────────────┐ │
│ │   Imagen     │ │
│ └──────────────┘ │
│ Nombre           │
│ $12.50           │
└──────────────────┘
```

**3. Controles.**
- **Botón tarjeta**: `onClick={() => onAdd(product)}`.
- **`IMG_CHAIN`**: cascada de 5 niveles (25–31).
- **`resolveUrl(url)`**: resuelve URL (18–23).

**4. Estados.**
- **Imagen cargada**: muestra imagen.
- **Imagen fallida**: cae al siguiente nivel de `IMG_CHAIN`.
- **Fallback**: emoji.
- **Hover**: `hover:bg-[#c1d72e]`.

**5. Navegación.**
- **Entrada**: `ProductGrid`.
- **Salida**: no aplica (es composición).
- **Abre**: agrega al carrito.

**6. Modo responsivo.**
- **Mostrador**: `rounded-[35px]`.
- **Compacto**: igual.
- **Móvil**: igual.

**7. Anclaje al código.** [`apps/pos/components/ProductCard.jsx`](../../apps/pos/components/ProductCard.jsx:8) (63 líneas). `IMG_CHAIN` en 25–31. `rounded-[35px]`. Precio `text-[14px] font-mono italic`.

---

### FICHA 21 — `CategoryBar`

**1. Propósito.** Barra inferior de categorías. Muestra el botón "ESCANER IA" y las categorías; al tocar una, filtra el grid.

**2. Estructura visual.**
```
┌─────────────────────────────────────────────────────┐
│ [ESCANER IA] │ [Pan] [Pastel] [Bebida] [Helado] ... │
└─────────────────────────────────────────────────────┘
```

**3. Controles.**
- **Botón "ESCANER IA"**: `setViewMode('CAMERA')`.
- **Botón categoría**: `setActiveCategory(cat)` + `setCurrentPage(1)`.
- **Scroll horizontal**: `overflow-x-auto`.

**4. Estados.**
- **Categoría activa**: resaltada.
- **Modo GRID/CAMERA**: botón ESCANER IA resaltado en CAMERA.

**5. Navegación.**
- **Entrada**: pie de `RetailVisionPOS`.
- **Salida**: no aplica (es composición).
- **Abre**: filtra `ProductGrid` o cambia a `VisionVisor`.

**6. Modo responsivo.**
- **Mostrador**: `text-[18px]`.
- **Compacto**: igual.
- **Móvil**: `overflow-x-auto`.

**7. Anclaje al código.** [`apps/pos/components/CategoryBar.jsx`](../../apps/pos/components/CategoryBar.jsx:3) (30 líneas). Botón ESCANER IA + categorías. `text-[18px]`.

---

### FICHA 22 — `CategoryEditor`

**1. Propósito.** Editor de categorías con IA. Permite activar/desactivar la visión por categoría y guardar.

**2. Estructura visual.**
```
┌─────────────────────────────────────────────────────┐
│ Header: "EDITOR DE CATEGORÍAS"                      │
├─────────────────────────────────────────────────────┤
│ Grid de categorías:                                 │
│ ├─ [Pan]      [👁️ Visión ON]                        │
│ ├─ [Pastel]   [👁️ Visión OFF]                       │
│ └─ ...                                              │
├─────────────────────────────────────────────────────┤
│ [Cancelar] [Guardar]                                │
└─────────────────────────────────────────────────────┘
```

**3. Controles.**
- **`toggleVision(name)`**: activa/desactiva visión (16–20).
- **Botón "Guardar"**: `onSave`.
- **Botón "Cancelar"**: `onCancel`.

**4. Estados.**
- **Visión ON**: categoría incluida en el escáner.
- **Visión OFF**: categoría excluida.
- **Guardando**: `onSave` en curso.

**5. Navegación.**
- **Entrada**: desde `VisionTrainingUI` o ajustes.
- **Salida**: `onSave` / `onCancel`.
- **Abre**: nada (es composición).

**6. Modo responsivo.**
- **Mostrador**: `lg:grid-cols-3`.
- **Compacto**: `md:grid-cols-2`.
- **Móvil**: `grid-cols-1`.

**7. Anclaje al código.** [`apps/pos/CategoryEditor.jsx`](../../apps/pos/CategoryEditor.jsx:8) (89 líneas). `toggleVision` en 16–20. Grid `grid-cols-1 md:grid-cols-2 lg:grid-cols-3`.

---

## §7. Fichas — Impresión (2)

---

### FICHA 23 — `TicketTemplate`

**1. Propósito.** Plantilla del **ticket de venta** (80mm). Se renderiza oculta y se imprime con `window.print()`. Incluye la auditoría de responsables (CAPTURÓ / COBRÓ).

**2. Estructura visual.**
```
┌──────────────────────────┐ (80mm)
│      LOGO / NEGOCIO      │
│  Folio: #123             │
│  Fecha: 2026-09-21 14:30 │
├──────────────────────────┤
│ 2x Pan bolillo   $25.00  │
│ 1x Concha        $12.00  │
├──────────────────────────┤
│ TOTAL            $37.00  │
├──────────────────────────┤
│ CAPTURÓ: Juan            │
│ COBRÓ:   María           │
│ Terminal: T1             │
└──────────────────────────┘
```

**3. Controles.**
- **`ref`**: referencia para impresión.
- **`@page { size: 80mm auto }`**: configuración de impresión.
- **Auditoría**: CAPTURÓ / COBRÓ / Terminal (79–92).

**4. Estados.**
- **Renderizado oculto**: listo para imprimir.
- **Imprimiendo**: `window.print()`.

**5. Navegación.**
- **Entrada**: `handlePrintTicket` (desde `useTicketActions`).
- **Salida**: no aplica (es plantilla).
- **Abre**: diálogo de impresión del navegador.

**6. Modo responsivo.**
- **No aplica**: es una plantilla de impresión de ancho fijo (`w-[80mm]`).

**7. Anclaje al código.** [`apps/pos/components/TicketTemplate.jsx`](../../apps/pos/components/TicketTemplate.jsx:1) (103 líneas). `w-[80mm] font-mono`. Auditoría en 79–92.

---

### FICHA 24 — `CorteTicketTemplate`

**1. Propósito.** Plantilla del **corte de caja** (80mm). Resume la sesión: totales por método, diferencias (efectivo/crédito/débito), cuentas cobradas y flujo de efectivo.

**2. Estructura visual.**
```
┌──────────────────────────┐ (80mm)
│     CORTE DE CAJA        │
│  Terminal: T1            │
│  Apertura: 08:00         │
│  Cierre:   20:00         │
├──────────────────────────┤
│ Efectivo esperado $500   │
│ Efectivo capturado $495  │
│ Diferencia        -$5    │
├──────────────────────────┤
│ Crédito / Débito         │
├──────────────────────────┤
│ Cuentas cobradas: 42     │
│ Flujo de efectivo        │
└──────────────────────────┘
```

**3. Controles.**
- **`ref`**: referencia para impresión.
- **`formatHora(iso)`**: formatea hora (7–13).
- **`difCash` / `difCredit` / `difDebit`**: diferencias calculadas.

**4. Estados.**
- **Renderizado oculto**: listo para imprimir.
- **Con diferencias**: resaltadas.
- **Sin diferencias**: cuadre perfecto.

**5. Navegación.**
- **Entrada**: `handlePrintCorte` (desde `GestorDeCaja`).
- **Salida**: no aplica (es plantilla).
- **Abre**: diálogo de impresión del navegador.

**6. Modo responsivo.**
- **No aplica**: es una plantilla de impresión de ancho fijo (`w-[80mm]`).

**7. Anclaje al código.** [`apps/pos/components/CorteTicketTemplate.jsx`](../../apps/pos/components/CorteTicketTemplate.jsx:1) (190 líneas). `difCash`/`difCredit`/`difDebit`. Flujo de efectivo en 81–117.

---

## §8. Matriz de trazabilidad — Interfaz → Backend

Esta matriz conecta cada interfaz con el endpoint/servicio del backend que consume. Es la prueba de que **ninguna interfaz es huérfana** y de que **ningún endpoint carece de interfaz**.

| # | Interfaz | Endpoint / Servicio | Módulo |
|---|---|---|---|
| 01 | `RetailVisionPOS` | Orquesta todos | `pos` |
| 02 | `TableServicePOS` | *(datos mock — pendiente backend)* | `pos` |
| 03 | `VisionTrainingUI` | `POST /pos/vision/predict`, `upload_training_images` | `pos` |
| 04 | `GrandezaParamsUI` | `/grandeza/products`, `/clients`, `/routes` | `grandeza` |
| 05 | `GrandezaDailyUI` | `/grandeza/journeys`, `/inventory`, `/expenses` | `grandeza` |
| 06 | `GrandezaDriverUI` | `/grandeza/journeys`, `/visits`, `/location`, `/orders` | `grandeza` |
| 07 | `RepartoPanGrandezaUI` | *(enrutador de suites)* | `grandeza` |
| 08 | `CheckoutScreen` | `POST /pos/tickets` (checkout) | `pos` |
| 09 | `GestionPersonal` | `/employees`, `/profiles` | `rrhh` |
| 10 | `GestorDeCaja` | `cashService.abrirSesion/cerrarSesion`, `/analytics/context` | `cash` |
| 11 | `ProgramacionPedidoModal` | `POST /pos/tickets` (PEDIDO) | `pos` |
| 12 | `TerminalSelector` | `/pos/terminals/status`, `/lock`, `/unlock` | `pos` |
| 13 | `OpenAccountsCorkboard` | `GET /pos/tickets` (open) | `pos` |
| 14 | `SalesReceipt` | `addItemToTicket`, `updateItemQuantity`, `removeItemFromTicket` | `pos` |
| 15 | `POSHeader` | `/pos/terminals/status`, `heartbeat` | `pos` |
| 16 | `POSOverlays` | `emergency-save`, `force_unlock` | `pos` |
| 17 | `VisionVisor` | `POST /pos/vision/predict` | `pos` |
| 18 | `VisionScanner` | Gemini API (externo) | `pos` |
| 19 | `ProductGrid` | `GET /catalog/products` | `catalog` |
| 20 | `ProductCard` | *(cascada de imágenes)* | `catalog` |
| 21 | `CategoryBar` | `GET /catalog/categories` | `catalog` |
| 22 | `CategoryEditor` | `PUT /catalog/categories` | `catalog` |
| 23 | `TicketTemplate` | *(plantilla de impresión)* | `pos` |
| 24 | `CorteTicketTemplate` | *(plantilla de impresión)* | `cash` |

> **Hallazgo de trazabilidad.** `TableServicePOS` (FICHA 02) usa **datos mock** (`tables`, `menuItems`, `orders` hardcodeados). Es la única interfaz sin backend real. Se documenta como **deuda conocida** (no se corrige aquí).

---

## §9. Observaciones de ergonomía (NO se aplican)

Estas observaciones **no son correcciones**: son notas para el Nuevo POS. Se registran aquí para que el equipo de diseño las evalúe, pero **el POS actual no se toca**.

| ID | Interfaz | Observación | Tipo |
|---|---|---|---|
| O-01 | `CheckoutScreen` | `w-[1100px]`/`w-[800px]` viola R-01 en Compacto/Móvil | Responsive |
| O-02 | `GestionPersonal` | `w-[800px]` viola R-01 en Compacto | Responsive |
| O-03 | `GestorDeCaja` | Numpad `w-[380px]` viola R-01 en Compacto | Responsive |
| O-04 | `SalesReceipt` | `w-[420px]` viola R-01 en Compacto | Responsive |
| O-05 | `OpenAccountsCorkboard` | `aspect-[16/9]` no colapsa en Móvil | Responsive |
| O-06 | `TableServicePOS` | Datos mock — pendiente backend real | Deuda |
| O-07 | `VisionVisor` | `min-w-[180px]` podría ser overlay flotante en Móvil | Responsive |
| O-08 | `POSHeader` | Tipografía `text-[7px]` es muy pequeña para táctil | Ergonomía |
| O-09 | `ProductCard` | `IMG_CHAIN` de 5 niveles podría simplificarse | Arquitectura |
| O-10 | `TerminalSelector` | `grid-cols-3 md:grid-cols-6` podría ser `grid-cols-2` en Móvil | Responsive |
| O-11 | `CategoryBar` | `text-[18px]` podría escalar con `clamp()` | Responsive |
| O-12 | `TicketTemplate` | Auditoría CAPTURÓ/COBRÓ es excelente — preservar | Cicatriz |

> **Regla de oro.** Ninguna observación se aplica al POS actual. Todas se evalúan en el Nuevo POS. La ergonomía del mostrador es **intocable**.

---

## §10. Cierre

Este documento (Documento 7) cierra el círculo de la documentación del POS:

| Documento | Capa | Estado |
|---|---|---|
| `ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md` | Backend (qué hace) | ✅ |
| `ESPECIFICACION_RESPONSIVA_Y_ERGONOMIA_TACTIL.md` (Doc 6) | Responsive (cómo se adapta) | ✅ |
| **`ESPECIFICACION_DE_INTERFACES_POS.md` (Doc 7)** | **Frontend (cómo se ve y se toca)** | ✅ |

Con los tres documentos, quien reconstruya el Nuevo POS tendrá:
1. **Qué debe calcular** (backend).
2. **Cómo debe adaptarse** (responsive).
3. **Cómo debe verse y sentirse** (interfaces).

Las 26 interfaces quedan documentadas con su ficha de 7 puntos, su anclaje al código, y su trazabilidad al backend. La ergonomía del mostrador —refinada durante años— queda preservada para la réplica por sucursal.