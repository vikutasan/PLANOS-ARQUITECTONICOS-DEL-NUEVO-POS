# PLAN DE ABORDAJE — F7.5 (Integración de la Fase 7 en la pantalla real)

> **Micro-fase de integración.** No construye funciones nuevas: **cablea** las tres
> piezas que la Fase 7 ya construyó y probó, para que el operador pueda usarlas.
>
> **Versión:** 1.0 — 29 Sep 2026
> **Estado:** propuesto (pendiente de aprobación del dueño)
> **Predecesor:** Fase 7 cerrada (F7.0–F7.4, commit `76d1c76`)
> **Sucesor:** Fase 8 (CRM + Notificaciones, lado POS)

---

## §1. EL HALLAZGO QUE MOTIVA ESTA MICRO-FASE

Al cerrar la Fase 7, las tres funciones (temas, voz, visión) quedaron **construidas y
con su gate en verde**, pero **no están conectadas a la pantalla real del POS**.

### §1.1 Evidencia (verificada en el código, no en la ficha)

| Entregable | Archivo construido | Gate | ¿Cableado en la pantalla real? |
|---|---|---|---|
| Temas | [`useTheme.js`](../../NUEVO-POS/apps/pos/src/hooks/useTheme.js) + [`ThemeSelector.jsx`](../../NUEVO-POS/apps/pos/src/components/ThemeSelector.jsx) | ✅ 25 tests | ⚠️ **Parcial** — `useTheme()` se llama en [`App.jsx`](../../NUEVO-POS/apps/pos/src/App.jsx:35), pero **`ThemeSelector` no se renderiza en ninguna parte** |
| Voz | [`useVoiceCart.js`](../../NUEVO-POS/apps/pos/src/hooks/useVoiceCart.js) + [`VoiceCartPanel.jsx`](../../NUEVO-POS/apps/pos/src/components/VoiceCartPanel.jsx) | ✅ 47 tests | ❌ **No** — [`RetailVisionPOS.jsx`](../../NUEVO-POS/apps/pos/src/RetailVisionPOS.jsx:37) no lo importa |
| Visión | [`useVision.js`](../../NUEVO-POS/apps/pos/src/hooks/useVision.js) + [`VisionVisor.jsx`](../../NUEVO-POS/apps/pos/src/components/VisionVisor.jsx) | ✅ 26 tests | ❌ **No** — [`RetailVisionPOS.jsx`](../../NUEVO-POS/apps/pos/src/RetailVisionPOS.jsx:37) no lo importa |

**Prueba directa:** el bloque de imports de [`RetailVisionPOS.jsx`](../../NUEVO-POS/apps/pos/src/RetailVisionPOS.jsx:37)
contiene **solo** los componentes de la Fase 3 (`CategoryBar`, `ProductGrid`,
`SalesReceipt`, `CheckoutScreen`, `POSHeader`, `POSOverlays`). Ningún componente de la
Fase 7 aparece. Y [`POSHeader.jsx`](../../NUEVO-POS/apps/pos/src/components/POSHeader.jsx:38)
no tiene ningún botón para abrir tema, voz o visión.

### §1.2 Diagnóstico honesto

**No es un defecto de la Fase 7.** La Fase 7 se cerró en el punto "piezas construidas y
probadas", y la integración quedó fuera de su alcance declarado. Es **el mismo patrón que
usamos en la Fase 3**: primero se construyen las piezas con su gate (F3.3: hooks), y
**después** se cablean end-to-end en la pantalla real (F3.4 / cierre F3). Lo llamamos
"de adentro hacia afuera".

**Pero sin este paso, la Fase 7 no entrega valor al operador.** Tres funciones que nadie
puede abrir son arquitectura sin puerta de entrada. Esta micro-fase es esa puerta.

---

## §2. ALCANCE

### §2.1 Lo que SÍ se construye

| Sub-paso | Qué hace | Archivos |
|---|---|---|
| **F7.5.0** | Botones 🎨 Tema · 🎤 Voz · 📷 Visión en el header (R-04: `min-h-tactil`) | [`POSHeader.jsx`](../../NUEVO-POS/apps/pos/src/components/POSHeader.jsx) |
| **F7.5.1** | Cablear `ThemeSelector` en el header (ya existe `useTheme` en App) | [`POSHeader.jsx`](../../NUEVO-POS/apps/pos/src/components/POSHeader.jsx) + [`RetailVisionPOS.jsx`](../../NUEVO-POS/apps/pos/src/RetailVisionPOS.jsx) |
| **F7.5.2** | Cablear `VoiceCartPanel` + `useVoiceCart`; la confirmación llama a `carrito.anadirLinea` | [`RetailVisionPOS.jsx`](../../NUEVO-POS/apps/pos/src/RetailVisionPOS.jsx) |
| **F7.5.3** | Cablear `VisionVisor` + `useVision`; el "➕ Agregar" llama a `carrito.anadirLinea` | [`RetailVisionPOS.jsx`](../../NUEVO-POS/apps/pos/src/RetailVisionPOS.jsx) |
| **F7.5.4** | Gate de integración + ficha + commit + push | 2 archivos nuevos |

### §2.2 Lo que NO se construye (y por qué)

| Elemento | Por qué NO |
|---|---|
| Funciones nuevas de IA | Ya existen. Esta micro-fase solo las conecta. |
| Cambios en los hooks (`useTheme`, `useVoiceCart`, `useVision`) | Ya están probados. **No se tocan** (evita regresiones). |
| Cambios en los contratos (17, 24, 25) | Ya están declarados. **No se tocan.** |
| `ProgramacionPedidoModal.jsx` (pedidos programados) | Sigue diferido (§12.3 del plan F7). Su alcance no está definido con el dueño. |
| Fase 8 (CRM/Notificaciones) | Es la fase siguiente, no esta micro-fase. |

---

## §3. PRINCIPIOS QUE GOBIERNAN ESTA MICRO-FASE

1. **La IA propone, el operador confirma (H-5).** El cableado NO puede romper esta regla.
   `VoiceCartPanel` y `VisionVisor` solo llaman a `carrito.anadirLinea` **después** de una
   acción explícita del operador (botón "Agregar al carrito" / "➕ Agregar").
2. **La venta nunca se bloquea (DT-07).** Si la voz o la visión no están disponibles, el
   POS sigue vendiendo en modo manual. Los paneles ya muestran el aviso ámbar; el cableado
   solo los monta.
3. **No se toca lo que ya está probado.** Los hooks y los componentes de la Fase 7 no se
   modifican. Solo se **usan** desde la pantalla.
4. **R-03 (3 modos) y R-04 (target ≥44px).** Los botones nuevos del header respetan el
   target táctil y son visibles en mostrador/compacto/móvil.
5. **R-01 (fluido).** Nada de anchos fijos en px.

---

## §4. DISEÑO TÉCNICO

### §4.1 F7.5.0 — Botones en el header

[`POSHeader.jsx`](../../NUEVO-POS/apps/pos/src/components/POSHeader.jsx:84) tiene una zona
DERECHA (`<div className="flex items-center gap-2">`) con: sesión, red y modo. Se añaden
**tres botones** antes del indicador de sesión:

```jsx
{/* Acciones de IA (F7.5) */}
<button
  type="button"
  onClick={onAbrirTema}
  className="min-h-tactil min-w-tactil bg-fondo-profundo border border-white/5 rounded-xl px-3 flex items-center gap-2 hover:bg-fondo-panel transition-all"
  title="Cambiar tema"
  aria-label="Cambiar tema"
>
  <span aria-hidden="true">🎨</span>
</button>
<button
  type="button"
  onClick={onAbrirVoz}
  className="min-h-tactil min-w-tactil bg-fondo-profundo border border-white/5 rounded-xl px-3 flex items-center gap-2 hover:bg-fondo-panel transition-all"
  title="Dictado por voz"
  aria-label="Dictado por voz"
>
  <span aria-hidden="true">🎤</span>
</button>
<button
  type="button"
  onClick={onAbrirVision}
  className="min-h-tactil min-w-tactil bg-fondo-profundo border border-white/5 rounded-xl px-3 flex items-center gap-2 hover:bg-fondo-panel transition-all"
  title="Visión cenital"
  aria-label="Visión cenital"
>
  <span aria-hidden="true">📷</span>
</button>
```

**Props nuevas de `POSHeader`:** `onAbrirTema`, `onAbrirVoz`, `onAbrirVision` (todas
opcionales; si no se pasan, los botones no rompen — se usa `?.()`).

### §4.2 F7.5.1 — Cablear `ThemeSelector`

`useTheme` ya vive en [`App.jsx`](../../NUEVO-POS/apps/pos/src/App.jsx:35). El problema es
que `App.jsx` es el contenedor de **selección de terminal**, no la pantalla del POS. Hay
dos opciones:

- **Opción A (recomendada):** llamar a `useTheme()` también dentro de
  [`RetailVisionPOS.jsx`](../../NUEVO-POS/apps/pos/src/RetailVisionPOS.jsx) y renderizar
  `ThemeSelector` en un panel desplegable desde el botón 🎨. El hook es idempotente
  (aplica el mismo tema dos veces no daña).
- **Opción B:** pasar `tema`/`cambiarTema` desde `App.jsx` como props. Más acoplamiento.

**Se elige la Opción A** por menor acoplamiento y porque el hook ya está diseñado para
llamarse desde cualquier punto del árbol.

El panel de tema es un overlay simple (R-03) que contiene `<ThemeSelector>`:

```jsx
{temaAbierto ? (
  <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/70 backdrop-blur-sm p-4">
    <div className="bg-zinc-900 border border-white/10 rounded-3xl shadow-2xl w-full max-w-md p-6 space-y-4">
      <h2 className="text-xl font-black uppercase tracking-widest text-white">🎨 Tema</h2>
      <ThemeSelector
        tema={tema.tema}
        temas={tema.temas}
        ofreceSelector={tema.ofreceSelector}
        onCambiarTema={tema.cambiarTema}
      />
      <button
        type="button"
        onClick={() => setTemaAbierto(false)}
        className="w-full min-h-tactil bg-fondo-panel text-crema-ticket rounded-xl"
      >
        Cerrar
      </button>
    </div>
  </div>
) : null}
```

### §4.3 F7.5.2 — Cablear `VoiceCartPanel`

En [`RetailVisionPOS.jsx`](../../NUEVO-POS/apps/pos/src/RetailVisionPOS.jsx):

```jsx
const voz = useVoiceCart(productos);
const [vozAbierta, setVozAbierta] = useState(false);
```

El panel se monta cuando `vozAbierta` es true. **El punto crítico es `onApply`:** la
confirmación del operador es lo que agrega al carrito:

```jsx
{vozAbierta ? (
  <VoiceCartPanel
    grabando={voz.grabando}
    transcribiendo={voz.transcribiendo}
    texto={voz.texto}
    propuesta={voz.propuesta}
    disponible={voz.disponible}
    error={voz.error}
    fase={voz.fase}
    nivel={voz.nivel}
    productos={productos}
    onToggleRecording={voz.alternar}
    onEditLine={voz.editarLinea}
    onRemoveLine={voz.quitarLinea}
    onToggleConfirm={voz.alternarConfirmacion}
    onApply={() => {
      // La IA propone; el operador confirma. Solo aquí se toca el carrito.
      const lineas = voz.propuesta?.lineas || [];
      lineas.forEach((l) => {
        if (l.resuelto && l.producto) {
          carrito.anadirLinea({ producto: l.producto, cantidad: l.cantidad });
        }
      });
      voz.reset();
      setVozAbierta(false);
    }}
    onCancel={() => {
      voz.reset();
      setVozAbierta(false);
    }}
  />
) : null}
```

**Nota:** se usa `carrito.anadirLinea` (el mismo punto de entrada que usa el grid de
productos), de modo que la persistencia atómica por ítem (contratos 18–20) siga
operando igual. **No se inventa un camino nuevo al carrito.**

### §4.4 F7.5.3 — Cablear `VisionVisor`

```jsx
const vision = useVision(productos);
const [visionAbierta, setVisionAbierta] = useState(false);
```

```jsx
{visionAbierta ? (
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
    onAgregar={(s) => {
      // La IA sugiere; el operador decide. Solo aquí se toca el carrito.
      if (s.resuelto && s.producto) {
        carrito.anadirLinea({ producto: s.producto, cantidad: 1 });
      }
    }}
    onLimpiar={vision.limpiarSugerencias}
    onCerrar={() => {
      vision.detener();
      setVisionAbierta(false);
    }}
  />
) : null}
```

**Nota:** al cerrar el visor se llama a `vision.detener()` para apagar la cámara (no dejar
el LED encendido). El hook ya libera la cámara al desmontar, pero cerrar el visor sin
desmontar el hook requiere el `detener()` explícito.

### §4.5 F7.5.4 — Gate de integración

Un test que monta la **pantalla real** ([`RetailVisionPOS.jsx`](../../NUEVO-POS/apps/pos/src/RetailVisionPOS.jsx))
con la API simulada y verifica:

| Criterio | Qué verifica |
|---|---|
| 1 | El header muestra los 3 botones (🎨, 🎤, 📷) |
| 2 | Tocar 🎨 abre el panel de tema con los 3 temas |
| 3 | Tocar 🎤 abre el panel de voz |
| 4 | Tocar 📷 abre el visor de visión |
| 5 | Confirmar una propuesta de voz agrega la línea al carrito (vía `anadirLinea`) |
| 6 | Agregar una sugerencia de visión agrega la línea al carrito |
| 7 | Si la voz no está disponible, el panel muestra el aviso y el POS sigue usable |
| 8 | Si la visión no está disponible, el visor muestra el aviso y el POS sigue usable |
| 9 | Cerrar los paneles no rompe la pantalla |
| 10 | Los botones del header tienen `min-h-tactil` (R-04) |
| 11 | Los paneles son overlays fluidos, sin ancho fijo en px (R-03) |

---

## §5. ORDEN DE EJECUCIÓN

```
F7.5.0 (botones header)
   │
   ├── F7.5.1 (tema)     ─┐
   ├── F7.5.2 (voz)       ├── F7.5.4 (gate + ficha + commit)
   └── F7.5.3 (visión)   ─┘
```

F7.5.1, F7.5.2 y F7.5.3 son independientes entre sí (tocan zonas distintas del archivo).
Se ejecutan en orden para evitar conflictos de edición, pero no hay dependencia lógica.

---

## §6. CRITERIOS DE APROBACIÓN

La micro-fase se considera cerrada cuando:

1. Los 3 botones aparecen en el header y abren sus paneles.
2. Confirmar voz agrega al carrito **solo tras** la acción del operador.
3. Agregar visión agrega al carrito **solo tras** la acción del operador.
4. Si la IA no está disponible, el POS sigue vendiendo en modo manual.
5. El gate de integración (11 criterios) está en verde.
6. `npm run ci` está en verde (lint + tests + guardianes).
7. Cero regresiones en los tests existentes.
8. La ficha `FICHA_F7_5_INTEGRACION.md` documenta el qué, el por qué y la evidencia.

---

## §7. RIESGOS Y MITIGACIONES

| Riesgo | Probabilidad | Mitigación |
|---|---|---|
| Doble llamada a `useTheme` (App + RetailVisionPOS) causa parpadeo | Baja | El hook es idempotente; aplica el mismo tema. Si parpadea, se pasa por props (Opción B). |
| `carrito.anadirLinea` con un producto no resuelto | Media | Se filtra con `if (l.resuelto && l.producto)` antes de llamar. |
| El visor deja la cámara encendida al cerrar | Media | `onCerrar` llama a `vision.detener()`. |
| Los botones del header rompen el layout en móvil | Baja | `min-w-tactil` + `gap-2`; se verifica en el gate (criterio 10). |
| Regresión en los tests de la Fase 3 | Baja | No se tocan los hooks ni los componentes de F3; solo se añaden imports y JSX. |

---

## §8. TRAZABILIDAD

| Regla / Directriz | Cómo la cumple F7.5 |
|---|---|
| **H-5** (la IA propone, el operador confirma) | `onApply` y `onAgregar` solo actúan tras la acción del operador |
| **DT-07** (IA por contrato, nunca bloquea) | Los paneles muestran el aviso ámbar; el POS sigue vendiendo |
| **DT-08** (visión cenital) | El visor se monta como flujo persistente desde el botón 📷 |
| **R-01** (fluido) | Overlays con `w-full max-w-*`, sin px fijos |
| **R-03** (3 modos) | Los paneles son overlays; los botones visibles en los 3 modos |
| **R-04** (target ≥44px) | Botones del header y controles de los paneles con `min-h-tactil` |
| **A-02** (frontera por contratos) | El cableado usa los hooks, que ya consumen los contratos 17/24/25 |

---

## §9. RESUMEN EJECUTIVO

| Sub-paso | Entregable | Archivos | Riesgo |
|---|---|---|---|
| **F7.5.0** | Botones 🎨 🎤 📷 en el header | 1 | Bajo |
| **F7.5.1** | Panel de tema con `ThemeSelector` | 1 | Bajo |
| **F7.5.2** | Panel de voz con `VoiceCartPanel` | 1 | Bajo |
| **F7.5.3** | Visor con `VisionVisor` | 1 | Bajo |
| **F7.5.4** | Gate de integración + ficha | 2 | Bajo |

**Total:** 2 archivos modificados ([`POSHeader.jsx`](../../NUEVO-POS/apps/pos/src/components/POSHeader.jsx),
[`RetailVisionPOS.jsx`](../../NUEVO-POS/apps/pos/src/RetailVisionPOS.jsx)) + 2 archivos
nuevos (gate + ficha). **Cero dependencias nuevas. Cero cambios en hooks o contratos.**

**El POS termina la F7.5 con:** los tres entregables de la Fase 7 **accesibles desde la
pantalla real**, respetando la regla de oro (la IA propone, el operador confirma) y la
degradación elegante (si la IA cae, el POS sigue vendiendo).

---

## §10. POR QUÉ ESTA MICRO-FASE ES EL PASO NATURAL

1. **Cierra el ciclo de la Fase 7.** Sin ella, tres funciones probadas no tienen puerta
   de entrada. Con ella, el operador las usa.
2. **Es de bajo riesgo.** No se toca nada probado; solo se conecta.
3. **Es corta.** 2 archivos modificados + 2 nuevos.
4. **Desbloquea la Fase 8.** Una vez que la Fase 7 entrega valor real, tiene sentido abrir
   la Fase 8 (CRM/Notificaciones) con la renumeración ya documentada (contratos #26/#27,
   RN-82..93).

**Alternativas descartadas:**

- **Saltar a la Fase 8:** construiríamos dos componentes más que tampoco se cablearían.
  Acumularíamos deuda de integración.
- **F7.5 de pedidos programados:** su alcance no está definido con el dueño (H-7).
- **Probar en piso antes de cablear:** no se puede probar lo que el operador no puede abrir.
