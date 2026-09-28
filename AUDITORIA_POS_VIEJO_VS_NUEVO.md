# 🔍 Auditoría CORREGIDA: Módulo "Punto de Venta IA" vs POS Nuevo

> **Fecha:** 28 Sep 2026  
> **Método:** Se leyeron los imports de `RetailVisionPOS.jsx` del ERP viejo (líneas 1-36) para determinar EXACTAMENTE qué archivos pertenecen al módulo "Punto de Venta IA". Se EXCLUYERON módulos ajenos (Grandeza, TableService, VisionTraining, CategoryEditor).

---

## Definición del módulo "Punto de Venta IA"

En el ERP viejo, el sidebar (`ExperimentCenterUI.jsx`, línea 344) define:
```js
{ id: 'pos_retail', name: 'Punto de Venta IA', ... }
```
Que renderiza `RetailVisionPOS` (línea 686). Todo lo que `RetailVisionPOS.jsx` importa = el módulo completo.

---

## Archivos que SÍ son del módulo POS (confirmados por imports)

### Pantalla raíz (1 archivo)

| Archivo | Tamaño | ¿Existe en Nuevo? | Estado |
|---|---|---|---|
| `RetailVisionPOS.jsx` | 40 KB | ⚠️ 11 KB (simplificado) | Parcial |

### Componentes (12 archivos)

| # | Componente | Tamaño | ¿Existe en Nuevo? |
|---|---|---|---|
| 1 | `ProductGrid.jsx` | 2.7 KB | ✅ Sí |
| 2 | `SalesReceipt.jsx` | 15.9 KB | ⚠️ Parcial (6.3 KB, sin edición de qty) |
| 3 | `CheckoutScreen.jsx` | 28.7 KB | ⚠️ Parcial (5.5 KB) |
| 4 | `POSHeader.jsx` | 12.3 KB | ⚠️ Parcial (estética OK, sin funciones) |
| 5 | `TerminalSelector.jsx` | 31.3 KB | ❌ NO |
| 6 | `GestorDeCaja.jsx` | 56.3 KB | ❌ NO |
| 7 | `VoiceCartPanel.jsx` | 14.7 KB | ❌ NO |
| 8 | `VisionVisor.jsx` | 4.4 KB | ❌ NO |
| 9 | `ProgramacionPedidoModal.jsx` | 21.6 KB | ❌ NO |
| 10 | `POSOverlays.jsx` | 4.0 KB | ❌ NO |
| 11 | `ProductCard.jsx`* | 3.4 KB | ✅ Sí |
| 12 | `CategoryBar.jsx`* | 1.7 KB | ✅ Sí |

*ProductCard y CategoryBar se usan vía ProductGrid, no son imports directos de RetailVisionPOS.

> [!NOTE]
> `AnnotationCanvas.jsx` (10.7 KB), `TicketTemplate.jsx` (5 KB), `CorteTicketTemplate.jsx` (9.8 KB), y `GestionPersonal.jsx` (14.5 KB) viven en la carpeta pero **NO los importa RetailVisionPOS.jsx**. Podrían ser usados por GestorDeCaja o TerminalSelector internamente, pero no son imports directos del módulo POS.

### Hooks (9 archivos)

| # | Hook | Tamaño | ¿Existe en Nuevo? |
|---|---|---|---|
| 1 | `useCart.js` | 5.9 KB | ❌ (lógica inline en RetailVisionPOS) |
| 2 | `useVision.js` | 0.4 KB | ❌ NO |
| 3 | `useTerminalLocking.js` | 7.8 KB | ❌ NO |
| 4 | `useBeforeUnload.js` | 3.9 KB | ❌ NO |
| 5 | `useBarcodeScanner.js` | 1.2 KB | ❌ NO |
| 6 | `useNetworkHealth.js` | 3.2 KB | ❌ NO |
| 7 | `usePOSSession.js` | 7.8 KB | ❌ NO |
| 8 | `useTicketActions.js` | 23.5 KB | ❌ NO |
| 9 | `useVoiceCart.js` | 16.7 KB | ❌ NO |

En el nuevo POS solo existe: `useModo.js` (1.5 KB) — que ni siquiera existe en el viejo.

### Servicios (2 imports directos)

| # | Servicio | Tamaño | ¿Existe en Nuevo? |
|---|---|---|---|
| 1 | `POSService.js` | 10.3 KB | ❌ NO |
| 2 | `cashService.js` | 4.1 KB | ❌ NO |

### Utils (3 imports directos)

| # | Utilidad | Tamaño | ¿Existe en Nuevo? |
|---|---|---|---|
| 1 | `ticketGenerator.js` | 12.9 KB | ❌ NO |
| 2 | `voiceCartMapper.js` | 9.4 KB | ❌ NO |
| 3 | `withRetries.js` | 2.0 KB | ❌ NO |

### State (1 import directo)

| # | Módulo | Tamaño | ¿Existe en Nuevo? |
|---|---|---|---|
| 1 | `sessionReset.js` | 6.9 KB | ❌ NO |

### Config (1 archivo)

| # | Config | Tamaño | ¿Existe en Nuevo? |
|---|---|---|---|
| 1 | `config.js` | 1.2 KB | ✅ Sí (como `shared/config.js`) |

### Pantalla extra importada por RetailVisionPOS

| # | Pantalla | Tamaño | ¿Existe en Nuevo? |
|---|---|---|---|
| 1 | `OpenAccountsCorkboard.jsx` | 8.4 KB | ❌ NO |

---

## Archivos que NO son del módulo POS (EXCLUIDOS)

Estos archivos viven en `apps/pos/` pero **NO los importa RetailVisionPOS.jsx**:

| Archivo | Tamaño | Pertenece a |
|---|---|---|
| `GrandezaParamsUI.jsx` | 106 KB | Módulo Grandeza |
| `GrandezaDriverUI.jsx` | 108 KB | Módulo Grandeza |
| `GrandezaDailyUI.jsx` | 56 KB | Módulo Grandeza |
| `GrandezaOrderRequestsTab.jsx` | 60 KB | Módulo Grandeza |
| `RepartoPanGrandezaUI.jsx` | 11.7 KB | Módulo Grandeza |
| `VisionScanner.jsx` | 11.9 KB | Módulo Centro IA |
| `VisionTrainingUI.jsx` | 33 KB | Módulo Centro IA |
| `TableServicePOS.jsx` | 10.8 KB | Módulo Mesas |
| `CategoryEditor.jsx` | 5.6 KB | Módulo Gestión Productos |

---

## Resumen numérico CORREGIDO

| Categoría | Viejo (archivos) | Nuevo (archivos) | Cobertura |
|---|---|---|---|
| Pantalla raíz | 1 (+1 sub) | 1 (parcial) | ~30% |
| Componentes | 12 | 5 (3 parciales) | ~25% |
| Hooks | 9 | 1 (nuevo, no del viejo) | ~0% |
| Servicios | 2 | 0 | 0% |
| Utils | 3 | 0 | 0% |
| State | 1 | 0 | 0% |
| Config | 1 | 1 | 100% |
| **TOTAL** | **29 archivos** | **8 archivos** | **~20%** |

### En KB de código funcional:

| | Viejo | Nuevo | Faltante |
|---|---|---|---|
| **Total módulo POS** | ~310 KB | ~35 KB | **~275 KB** |

---

## Lo que falta construir (SOLO del módulo POS):

1. **TerminalSelector** — la landing page (seleccionar terminal)
2. **GestorDeCaja** — cortes, arqueos, turnos de caja
3. **VoiceCartPanel + useVoiceCart** — agregar productos por voz
4. **VisionVisor + useVision** — cámara de reconocimiento IA
5. **ProgramacionPedidoModal** — pedidos programados
6. **OpenAccountsCorkboard** — pizarrón de cuentas abiertas
7. **CheckoutScreen** completo — efectivo, tarjeta, cambio, folio
8. **POSOverlays** — modales de confirmación
9. **Todos los hooks** (9) — carrito, sesión, terminal lock, voz, etc.
10. **Todos los servicios** (2) — POSService, cashService
11. **Todas las utils** (3) — tickets, voz, reintentos
12. **State** (1) — session reset
