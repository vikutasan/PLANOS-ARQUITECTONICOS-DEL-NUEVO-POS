# PLAN DE CORRECCIÓN — F7.6 (UX heredada del viejo POS)

> **Micro-fase de corrección.** No construye funciones nuevas ni reabre la Fase 7.
> **Corrige el cableado** que la F7.5 dejó mal: la integración de los tres
> entregables de la Fase 7 debe **respetar la UX que el viejo POS ya definió**.
>
> **Versión:** 2.0 — 29 Sep 2026
> **Estado:** aprobado por el dueño (pendiente de ejecución)
> **Predecesor:** F7.5 (cableado inicial, incorrecto en la forma de integrar)
> **Sucesor:** Fase 8 (CRM + Notificaciones, lado POS)

---

## §1. EL HALLAZGO QUE MOTIVA ESTA CORRECCIÓN

La F7.5 cableó los tres entregables de la Fase 7 en la pantalla real, pero **lo hizo
inventando una integración nueva** (tres overlays abiertos desde tres botones del
header). El dueño advirtió que **el viejo POS ya tiene definida la UX** de cómo se
integran estos componentes.

### §1.1 Evidencia (verificada en el código del viejo POS)

| Entregable | Cómo lo integra el viejo POS | Cómo lo cableó la F7.5 (incorrecto) |
|---|---|---|
| **Visión** | Es un **modo de vista** (`viewMode === 'CAMERA'`): **reemplaza el cuerpo** (grid ↔ visor). Se conmuta desde la `CategoryBar` con el botón "ESCANER IA". | Un **overlay** abierto desde un botón del header. |
| **Voz** | Un **botón en el header** (`#btn-dictado-voz`) con `disabled={!voiceAvailable}`, que abre un **overlay** (`showVoicePanel`). | Un botón sin gate de disponibilidad. |
| **Tema** | **No existe** en el viejo POS. | Un overlay abierto desde un botón del header. |

**Prueba directa (viejo POS):**
- `apps/pos/RetailVisionPOS.jsx:667` — `viewMode === 'CAMERA' ? <VisionVisor .../> : <ProductGrid .../>`.
- `apps/pos/RetailVisionPOS.jsx:723` — `showVoicePanel && <VoiceCartPanel .../>`.
- `apps/pos/components/POSHeader.jsx:140` — botón `#btn-dictado-voz` con `disabled={!voiceAvailable}`.
- `apps/pos/components/CategoryBar.jsx:9` — botón "ESCANER IA" que hace `setViewMode('CAMERA')`; cada categoría hace `setViewMode('GRID')`.

### §1.2 El principio que se adopta

> **Cuando un componente ya existe en el viejo POS, su INTEGRACIÓN se hereda;
> solo su IMPLEMENTACIÓN se reescribe.**

El viejo POS es la **fuente de verdad de la integración** (dónde vive cada cosa y cómo
se abre). El nuevo POS es la **fuente de verdad de la implementación** (los hooks, los
contratos, la degradación elegante, los tests).

### §1.3 Diagnóstico honesto

La F7.5 **no fue un error de código**: fue un error de **suposición**. Se asumió que
"integrar" significaba "poner un botón que abra un panel", sin verificar que el viejo
POS ya tenía una gramática de interacción establecida. Esta micro-fase corrige esa
suposición **antes** de escribir el gate, para no blindar con tests una UX equivocada.

---

## §2. ALCANCE

### §2.1 Lo que SÍ se corrige

| Sub-paso | Qué corrige | Archivo |
|---|---|---|
| **F7.6.0** | `CategoryBar`: añadir `viewMode` + `onCambiarVista`; el botón "Escáner IA" conmuta a `CAMERA`; cada categoría conmuta a `GRID`. | `apps/pos/src/components/CategoryBar.jsx` |
| **F7.6.1** | `POSHeader`: la voz pasa a `disabled={!vozDisponible}`; **se retira el botón de visión** (la visión ya no se abre desde el header). | `apps/pos/src/components/POSHeader.jsx` |
| **F7.6.2** | `RetailVisionPOS`: `viewMode` como estado; el **cuerpo alterna grid ↔ visor**; la voz conserva su gate; el tema sigue como overlay nuevo. | `apps/pos/src/RetailVisionPOS.jsx` |
| **F7.6.3** | Gate de integración (11 criterios) + ficha + commit + push. | 2 archivos nuevos |
| **F7.6.4** | Documentar la UX heredada (Plan F7 §12.6 + Plan Maestro §6.8). | 2 documentos |

### §2.2 Lo que NO se toca (y por qué)

| Elemento | Por qué NO |
|---|---|
| `VisionVisor.jsx` | Ya funciona como overlay a pantalla completa. **No se le añade una prop `variante`**: el visor se monta **dentro del cuerpo** cuando `viewMode === 'CAMERA'`, y su propio `fixed inset-0` lo cubre. Cero cambios al componente. |
| `useVision.js` / `useVoiceCart.js` / `useTheme.js` | Los hooks están correctos y con gate en verde. Solo cambia **dónde se montan**, no cómo funcionan. |
| La Fase 7 (F7.0–F7.4) | **No se reabre.** La corrección es de cableado, no de construcción. |
| El gate de la Fase 7 | Sigue vigente. La F7.6 añade su propio gate de integración. |

---

## §3. DISEÑO TÉCNICO DE LA CORRECCIÓN

### §3.1 `CategoryBar` — el conmutador de vista (F7.6.0)

**Antes (F7.5):** el botón "Escáner IA" llamaba a `onSeleccionar(null)`.

**Después (F7.6.0):** se añaden dos props (`viewMode`, `onCambiarVista`) y el botón
"Escáner IA" conmuta la vista a `CAMERA`; cada categoría conmuta a `GRID`:

```jsx
export default function CategoryBar({
  categorias,
  categoriaActiva,
  onSeleccionar,
  onExportarPDF = null,
  viewMode = 'GRID',        // F7.6.0 — 'GRID' | 'CAMERA'
  onCambiarVista,           // F7.6.0 — (modo) => void
}) {
  // Botón ESCÁNER IA → conmuta a CAMERA
  onClick={() => onCambiarVista?.('CAMERA')}
  // Botón de categoría → conmuta a GRID + selecciona
  onClick={() => { onSeleccionar(cat.id); onCambiarVista?.('GRID'); }}
}
```

**Regla de compatibilidad:** si `onCambiarVista` no se pasa, el componente sigue
funcionando como antes (los botones solo seleccionan). Así el gate de la Fase 3
(que monta `CategoryBar` sin `viewMode`) **no se rompe**.

### §3.2 `POSHeader` — la voz con gate, sin botón de visión (F7.6.1)

**Antes (F7.5):** tres botones (tema, voz, visión) sin gate.

**Después (F7.6.1):**
- Se **retira el botón de visión** (la visión se abre desde la `CategoryBar`, no del header).
- El botón de voz recibe `disabled={!vozDisponible}` y un estilo atenuado cuando no está
  disponible (igual que el viejo POS).
- El botón de tema se conserva: es un overlay nuevo, no existía en el viejo POS.

```jsx
// F7.6.1 — la voz respeta la disponibilidad (como el viejo POS)
<button
  type="button"
  onClick={() => onAbrirVoz?.()}
  disabled={!vozDisponible}
  className={`... ${vozDisponible ? '...' : 'opacity-40 cursor-not-allowed'}`}
  title={vozDisponible ? 'Dictado por voz' : 'Dictado por voz no disponible'}
>
  <span aria-hidden="true">🎤</span>
</button>
```

### §3.3 `RetailVisionPOS` — el cuerpo alterna grid ↔ visor (F7.6.2)

**Antes (F7.5):** `visionAbierta` como booleano; el visor era un overlay al final.

**Después (F7.6.2):**
- Se reemplaza `visionAbierta` por `viewMode` (`'GRID' | 'CAMERA'`).
- El **cuerpo** alterna: `viewMode === 'CAMERA' ? <VisionVisor/> : <ProductGrid/>`.
- La voz conserva su overlay con gate (`voz.disponible`).
- El tema sigue como overlay nuevo.

```jsx
// F7.6.2 — el cuerpo alterna grid ↔ visor (UX heredada del viejo POS)
{viewMode === 'CAMERA' ? (
  <VisionVisor
    activo={vision.activo}
    analizando={vision.analizando}
    sugerencias={vision.sugerencias}
    disponible={vision.disponible}
    error={vision.error}
    umbral={vision.umbral}
    videoRef={vision.videoRef}
    canvasRef={vision.canvasRef}
    onToggle={vision.alternar}
    onAgregar={async (s) => { /* asegurarTicket + carrito.anadirLinea */ }}
    onLimpiar={vision.limpiarSugerencias}
    onCerrar={() => { vision.detener(); setViewMode('GRID'); }}
  />
) : (
  <ProductGrid productos={productosVisibles} onAgregar={agregarProducto} />
)}
```

**Nota sobre el `onCerrar`:** el visor se cierra volviendo a `GRID` (no a un booleano).
El botón "×" del visor y el botón "Escáner IA" de la `CategoryBar` son las dos formas
de entrar/salir del modo cámara.

### §3.4 El contrato del carrito se conserva

La confirmación de voz y el "Agregar" de visión siguen llamando a
`carrito.anadirLinea({ product_id, name, quantity, unit_price })` **previo
`asegurarTicket()`**. Es el MISMO camino que el grid, así que la persistencia atómica
por ítem (contratos 18–20) sigue operando igual. **La corrección de UX no toca la
persistencia.**

---

## §4. ORDEN DE EJECUCIÓN

| # | Sub-paso | Depende de | Resultado verificable |
|---|---|---|---|
| 1 | **F7.6.0** — `CategoryBar` con `viewMode`/`onCambiarVista` | — | El botón "Escáner IA" conmuta a `CAMERA`; las categorías a `GRID`. |
| 2 | **F7.6.1** — `POSHeader` voz con gate, sin botón de visión | — | La voz se deshabilita si `!vozDisponible`; no hay botón de visión. |
| 3 | **F7.6.2** — `RetailVisionPOS` con `viewMode` | F7.6.0, F7.6.1 | El cuerpo alterna grid ↔ visor; la voz abre con gate; el tema abre. |
| 4 | **F7.6.3** — Gate + ficha + commit + push | F7.6.0–F7.6.2 | 11 criterios en verde; CI verde; commit empujado. |
| 5 | **F7.6.4** — Documentar la UX heredada | F7.6.3 | Plan F7 §12.6 + Plan Maestro §6.8 escritos. |

---

## §5. GATE DE INTEGRACIÓN — 11 CRITERIOS

El gate vive en `RetailVisionPOS.f7_6.test.jsx` y verifica, sobre la pantalla real:

1. **La `CategoryBar` expone el conmutador de vista.** El botón "Escáner IA" llama a
   `onCambiarVista('CAMERA')`.
2. **Cada categoría conmuta a `GRID`.** Al pulsar una categoría se llama a
   `onSeleccionar(id)` **y** a `onCambiarVista('GRID')`.
3. **El cuerpo alterna grid ↔ visor.** Con `viewMode='CAMERA'` se monta `VisionVisor`;
   con `viewMode='GRID'` se monta `ProductGrid`.
4. **El visor NO es un overlay suelto.** No existe un estado `visionAbierta` que lo
   monte al final; vive dentro del cuerpo.
5. **La voz se abre desde el header con gate.** El botón de voz está `disabled` cuando
   `voz.disponible === false`.
6. **La voz SÍ se abre cuando está disponible.** Con `voz.disponible === true`, pulsar
   el botón de voz monta `VoiceCartPanel`.
7. **El header NO tiene botón de visión.** No existe un botón de visión en el header.
8. **El tema se abre como overlay nuevo.** El botón de tema monta `ThemeSelector`.
9. **La confirmación de voz toca el carrito por el camino atómico.** `onApply` llama a
   `asegurarTicket()` y luego a `carrito.anadirLinea` con la firma
   `{product_id, name, quantity, unit_price}`.
10. **El "Agregar" de visión toca el carrito por el mismo camino.** `onAgregar` llama a
    `asegurarTicket()` y luego a `carrito.anadirLinea`.
11. **La degradación elegante se conserva.** Si el Centro de IA no responde, el POS
    sigue vendiendo en modo manual (el grid sigue montado cuando `viewMode='GRID'`).

---

## §6. RIESGOS Y MITIGACIONES

| Riesgo | Mitigación |
|---|---|
| Romper el gate de la Fase 3 (que monta `CategoryBar` sin `viewMode`) | Las props nuevas son **opcionales** con default `'GRID'`; sin `onCambiarVista` el componente se comporta como antes. |
| El visor, al ser `fixed inset-0`, tape el ticket lateral | Es el comportamiento heredado del viejo POS (el visor cubre el cuerpo). El operador sale con "×" o con una categoría. |
| Duplicar el estado de visión (`visionAbierta` + `viewMode`) | Se **elimina** `visionAbierta`; `viewMode` es la única fuente de verdad. |
| El botón de voz deshabilitado confunde al operador | Se añade `title` explicativo ("Dictado por voz no disponible"), igual que el viejo POS. |

---

## §7. TRAZABILIDAD

| Regla / Directriz | Cómo la respeta la F7.6 |
|---|---|
| **H-5 (Regla de Oro)** | La IA propone; el operador confirma. Solo `onApply`/`onAgregar` tocan el carrito. |
| **RN-74 (visión asistiva)** | El visor sugiere; nunca bloquea la venta manual (el grid sigue disponible). |
| **DT-07 (Centro de IA único)** | Los tres hooks consumen contratos; degradan con elegancia. |
| **DT-08 (visión cenital)** | El visor es persistente (modo "escáner de charola"), no se cierra por producto. |
| **A-02 (frontera por contratos)** | El POS no lee tablas ajenas; consume contratos 17/24/25. |
| **R-03 (3 modos)** | El visor usa `max-w-3xl`/`max-h-[80vh]`; el cuerpo alterna en los 3 modos. |
| **R-04 (target táctil)** | Todos los controles nuevos tienen `min-h-tactil`. |
| **UX heredada** | La integración replica la del viejo POS (visión = modo de vista; voz = overlay con gate). |

---

## §8. RESUMEN EJECUTIVO

- **Qué es:** una micro-fase de **corrección de cableado**, no de construcción.
- **Qué corrige:** la visión pasa de overlay a **modo de vista** (grid ↔ visor); la voz
  gana su **gate de disponibilidad**; el botón de visión sale del header.
- **Qué conserva:** el tema como overlay nuevo; los hooks intactos; la persistencia
  atómica por ítem intacta; el gate de la Fase 7 intacto.
- **Qué produce:** 3 archivos corregidos + 1 gate (11 criterios) + 1 ficha + 2 documentos
  actualizados.
- **Cero dependencias nuevas.**

---

## §9. POR QUÉ ES EL PASO NATURAL

1. **Cierra el hueco que la F7.5 abrió mal.** Sin esta corrección, el gate de la F7.5
   blindaría con tests una UX que el dueño ya rechazó.
2. **Respeta la fuente de verdad.** El viejo POS ya resolvió esta integración; ignorarlo
   sería repetir el error que el proyecto entero busca evitar.
3. **Deja la Fase 7 realmente utilizable.** Con la visión como modo de vista, el operador
   ve la cámara cenital **en el cuerpo**, como en el viejo POS, no en un modal encima.
4. **Prepara la Fase 8 sin deuda.** La Fase 8 (CRM + Notificaciones) se apoyará en esta
   gramática de interacción ya estabilizada.
