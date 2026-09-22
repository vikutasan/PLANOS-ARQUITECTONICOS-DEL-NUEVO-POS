# ESPECIFICACIÓN RESPONSIVA Y ERGONOMÍA TÁCTIL

> **Documento maestro #6 del Nuevo POS**
> **Estado:** Vigente — regla dura del proyecto
> **Aplica a:** TODOS los módulos del ERP (POS, Almacenes, Productos, RRHH, Estadísticas, Auditoría, Heladería, Reparto, Monitoreo de Red, Pedidos, Perfiles, KDS, Producción).
> **Se convierte en:** Acción canónica **A-06** del [`PLAN_ACCION_ARQUITECTONICO_NUEVO_POS.md`](./ESPECIFICACIONES%20DEL%20PROYECTO/PLAN_ACCION_ARQUITECTONICO_NUEVO_POS.md)

---

## §0. Por qué existe este documento

El ERP actual es **responsivo a medias, de forma inconsistente**. La auditoría del código real (v22, commit `fe9f6ed`; el ERP hoy corre en `5802f45`, V23) reveló una asimetría clara:

| Zona del sistema | Estado responsivo | Evidencia |
|---|---|---|
| Módulos de gestión (Almacenes, RRHH, Estadísticas, Auditoría, Producción) | **Bueno** | Usan breakpoints Tailwind (`md:`, `lg:`, `xl:`) y patrones mobile-first |
| POS transaccional (cobro, ticket, caja, mesas, visión) | **Malo** | Anchos fijos en píxeles (`w-[420px]`, `w-[1100px]`, `w-[800px]`…) |

El POS transaccional es precisamente la parte que **más se usa** y la que **más se va a replicar** en cada sucursal. Si nace con anchos fijos, cada sucursal con tablet distinta requerirá una "adaptación" manual — y eso mata la economía del despliegue multi-sucursal.

**La decisión de este documento:** el Nuevo POS nace responsivo **desde el primer componente**, sin excepciones, y sin degradar la ergonomía del mostrador que ya está probada en producción.

---

## §1. La distinción fundamental: ESTÉTICA vs. LAYOUT

Este es el concepto que resuelve el conflicto aparente entre *"quiero conservar la estética"* y *"quiero que sea responsivo"*. **No son lo mismo.**

### §1.1 ESTÉTICA — INTOCABLE (se preserva 100%)

Todo lo que da identidad visual al POS **no se toca**. Se replica idéntico:

| Elemento estético | Valor actual | Se preserva |
|---|---|---|
| Fondo de papel del ticket | `#fdfbf7` | ✅ Sí |
| Tipografía del ticket | `font-mono` (monospace de recibo) | ✅ Sí |
| Bordes serrados del ticket | `absolute top-[-14px]` + `height: 14px` | ✅ Sí |
| Sombras profundas | `shadow-2xl`, `shadow-[0_0_100px_rgba(...)]` | ✅ Sí |
| Esquinas ultra-redondeadas | `rounded-[35px]`, `rounded-[50px]` | ✅ Sí |
| Paleta de acento | `#c1d72e` (lima), `orange-500`, `amber-400` | ✅ Sí |
| Modo oscuro base | `bg-[#1a1a1a]`, `bg-black/80` | ✅ Sí |
| Animaciones | `animate-in fade-in`, `zoom-in-95`, `duration-300` | ✅ Sí |
| Iconografía | Emojis + iconos actuales | ✅ Sí |

**Regla:** si el cambio propuesto altera *cómo se ve* el POS en la tablet del mostrador (≥1024px), **está prohibido**.

### §1.2 LAYOUT — PARAMETRIZABLE (se vuelve fluido)

Todo lo que define *cuánto mide* y *cómo se acomoda* **se parametriza**:

| Elemento de layout | Valor actual (rígido) | Valor nuevo (fluido) |
|---|---|---|
| Ancho del ticket | `w-[420px]` | `w-full max-w-[420px]` |
| Ancho del modal de cobro | `w-[1100px]` / `w-[800px]` | `w-full max-w-[1100px]` |
| Columna lateral de pedido | `w-[320px] flex-shrink-0` | `w-full lg:w-[320px] lg:flex-shrink-0` |
| Numpad de PIN | `w-[280px]` | `w-full sm:w-[280px]` |
| Teclado táctil de caja | `w-[380px]` | `w-full lg:w-[380px]` |
| Comanda lateral de mesas | `w-[400px]` | `w-full lg:w-[400px]` |
| Panel de entrenamiento de visión | `w-[450px]` | `w-full lg:w-[450px]` |

**Regla:** en el modo Mostrador (≥1024px), el resultado visual es **idéntico** al actual. El `max-w-[420px]` con `w-full` produce exactamente 420px cuando hay espacio. La diferencia solo aparece cuando **no** hay espacio — y ahí es donde el sistema se adapta en vez de romperse.

---

## §2. Los 3 modos de layout

El Nuevo POS define **tres modos explícitos**, no un continuum difuso. Cada modo tiene un propósito de negocio distinto.

| Modo | Breakpoint | Dispositivo típico | Usuario | Propósito |
|---|---|---|---|---|
| **MOSTRADOR** | `≥ 1024px` | Tablet fija 10"–13" horizontal, monitor de caja | Cajero | Cobro rápido, teclado táctil grande, ticket siempre visible |
| **COMPACTO** | `768px – 1023px` | Tablet chica 7"–8", tablet vertical | Cajero / Gerente | Cobro en mostrador reducido, ticket colapsable |
| **MÓVIL** | `< 768px` | Celular del dueño, celular del repartidor | Dueño / Repartidor | Consulta, autorización, reparto — **no** cobro intensivo |

### §2.1 Tabla de breakpoints (Tailwind)

| Modo | Prefijo Tailwind | Rango | Regla de layout |
|---|---|---|---|
| MÓVIL | (base, sin prefijo) | `< 768px` | 1 columna, ticket en drawer, acciones en barra inferior |
| COMPACTO | `md:` | `768px – 1023px` | 2 columnas, ticket lateral colapsable |
| MOSTRADOR | `lg:` y `xl:` | `≥ 1024px` | Layout actual completo, sin cambios visuales |

### §2.2 Reglas por modo

**MODO MOSTRADOR (≥1024px) — el modo de referencia**
- Es el layout actual del POS v22. **No se modifica nada visualmente.**
- Ticket siempre visible a la derecha (`max-w-[420px]`).
- Teclado táctil siempre visible.
- Modal de cobro centrado a `max-w-[1100px]`.
- Todos los targets táctiles ≥ 44×44px.

**MODO COMPACTO (768–1023px) — el modo de la tablet chica**
- El ticket pasa a **panel colapsable** (botón "Ver ticket" que abre un drawer lateral).
- El teclado táctil se reduce a `w-full` bajo el contenido.
- El modal de cobro ocupa `w-full` con `max-w-[800px]`.
- La columna lateral de pedido (`w-[320px]`) se apila debajo del contenido principal.
- Se preservan colores, tipografía y sombras.

**MODO MÓVIL (<768px) — el modo de consulta**
- **No es un modo de cobro intensivo.** Es para que el dueño consulte ventas, autorice descuentos, vea el reparto.
- 1 columna, scroll vertical.
- Acciones primarias en **barra inferior fija** (zona del pulgar).
- El ticket se abre en pantalla completa (no lateral).
- Targets táctiles ≥ 44×44px, con separación mínima de 8px.

---

## §3. Las 4 reglas duras (R-01 a R-04)

Estas reglas son **obligatorias** para todo componente nuevo del Nuevo POS. Un componente que las viole **no se acepta** en revisión.

### R-01 — CERO anchos absolutos en contenedores raíz

**Prohibido** en el contenedor raíz de cualquier pantalla o modal:
```
w-[420px]   w-[800px]   w-[1100px]   w-[320px]   w-[280px]   w-[380px]   w-[400px]   w-[450px]
```

**Obligatorio:**
```
w-full max-w-[<valor>]        // contenedores centrados
w-full lg:w-[<valor>]         // columnas laterales
w-full sm:w-[<valor>]         // elementos que crecen en móvil
```

**Excepción permitida:** `min-w-[...]` en tablas con scroll horizontal (`overflow-x-auto`), porque ahí el ancho mínimo es intencional y el scroll es la solución. Ejemplo válido: `min-w-[600px]` dentro de `overflow-x-auto`.

**Excepción permitida:** `max-h-[...]` en listas con scroll vertical (`overflow-y-auto`). El alto máximo es una decisión de ergonomía, no de layout.

### R-02 — Tipografía que escala, no que se fija

**Prohibido** como tamaño base de texto legible:
```
text-[7px]   text-[8px]   text-[9px]
```

**Obligatorio:**
```
text-[10px] md:text-xs        // etiquetas secundarias
text-xs md:text-sm            // texto de cuerpo
text-sm md:text-base          // texto principal
text-3xl md:text-4xl          // números grandes (cantidad, importe)
```

**Excepción permitida:** las plantillas de **impresión** (`TicketTemplate.jsx`, `CorteTicketTemplate.jsx`) pueden usar `text-[7px]`/`text-[8px]` porque el ancho del papel térmico es fijo (58mm/80mm) y no hay pantalla que adaptar. **Estas plantillas NO son responsivas y no deben serlo.**

**Excepción permitida:** etiquetas decorativas dentro de un componente ya escalado (ej. el `text-[8px]` de "Importe a Recibir" sobre un número `text-4xl`), siempre que el contenedor padre escale.

### R-03 — Los 3 modos son explícitos, no implícitos

Cada pantalla debe declarar **explícitamente** cómo se comporta en los 3 modos. No se acepta "que se acomode solo".

**Patrón obligatorio:**
```jsx
// Contenedor raíz de pantalla
<div className="w-full max-w-[1100px] mx-auto
                flex flex-col
                lg:flex-row lg:gap-6">

  {/* Contenido principal */}
  <div className="w-full lg:flex-1">...</div>

  {/* Columna lateral (ticket, comanda, teclado) */}
  <div className="w-full lg:w-[420px] lg:flex-shrink-0">...</div>
</div>
```

**Patrón obligatorio para acciones en móvil:**
```jsx
{/* Barra inferior fija solo en móvil */}
<div className="fixed bottom-0 left-0 right-0 p-4
                bg-black/90 backdrop-blur-xl
                lg:hidden">
  <button className="w-full h-14 ...">Cobrar</button>
</div>
```

### R-04 — Targets táctiles de 44×44px mínimo

Todo elemento interactivo (botón, celda de numpad, ítem de lista, checkbox) debe tener un área táctil de **al menos 44×44px** en los 3 modos.

**Prohibido:**
```jsx
<button className="text-[9px] px-1 py-0.5">[ Quitar ]</button>   // ❌ ~20×14px
```

**Obligatorio:**
```jsx
<button className="min-h-[44px] min-w-[44px] px-3 py-2 text-xs">Quitar</button>   // ✅
```

**Excepción permitida:** elementos puramente decorativos o de solo lectura (badges, etiquetas de estado) no son targets táctiles.

---

## §4. Inventario de contenedores a parametrizar

Auditoría del código real (v22, commit `fe9f6ed`; el ERP hoy corre en `5802f45`, V23). Estos son los contenedores que **deben** nacer fluidos en el Nuevo POS.

### §4.1 POS transaccional — CRÍTICOS (bloquean el cobro)

| # | Archivo | Línea | Actual | Nuevo | Modo afectado |
|---|---|---|---|---|---|
| 1 | [`SalesReceipt.jsx`](../../apps/pos/components/SalesReceipt.jsx:41) | 41 | `w-[420px]` | `w-full max-w-[420px]` | COMPACTO, MÓVIL |
| 2 | [`CheckoutScreen.jsx`](../../apps/pos/components/CheckoutScreen.jsx:132) | 132 | `w-[1100px]` | `w-full max-w-[1100px]` | COMPACTO, MÓVIL |
| 3 | [`CheckoutScreen.jsx`](../../apps/pos/components/CheckoutScreen.jsx:133) | 133 | `w-[800px]` | `w-full max-w-[800px]` | COMPACTO, MÓVIL |
| 4 | [`CheckoutScreen.jsx`](../../apps/pos/components/CheckoutScreen.jsx:322) | 322 | `w-[320px] flex-shrink-0` | `w-full lg:w-[320px] lg:flex-shrink-0` | COMPACTO, MÓVIL |
| 5 | [`GestionPersonal.jsx`](../../apps/pos/components/GestionPersonal.jsx:123) | 123 | `w-[800px] h-[600px]` | `w-full max-w-[800px] h-full max-h-[600px]` | COMPACTO, MÓVIL |
| 6 | [`GestionPersonal.jsx`](../../apps/pos/components/GestionPersonal.jsx:241) | 241 | `w-[280px]` | `w-full sm:w-[280px]` | MÓVIL |
| 7 | [`GestorDeCaja.jsx`](../../apps/pos/components/GestorDeCaja.jsx:853) | 853 | `w-[380px]` | `w-full lg:w-[380px]` | COMPACTO, MÓVIL |
| 8 | [`TableServicePOS.jsx`](../../apps/pos/TableServicePOS.jsx:134) | 134 | `w-[400px]` | `w-full lg:w-[400px]` | COMPACTO, MÓVIL |
| 9 | [`VisionTrainingUI.jsx`](../../apps/pos/VisionTrainingUI.jsx:80) | 80 | `w-[450px]` | `w-full lg:w-[450px]` | COMPACTO, MÓVIL |
| 10 | [`SalesReceipt.jsx`](../../apps/pos/components/SalesReceipt.jsx:206) | 206 | `w-[400px]` (modal cantidad) | `w-full max-w-[400px]` | COMPACTO, MÓVIL |

### §4.2 POS — tipografía diminuta (R-02)

| Archivo | Línea | Actual | Contexto | Acción |
|---|---|---|---|---|
| [`SalesReceipt.jsx`](../../apps/pos/components/SalesReceipt.jsx:87) | 87 | `text-[9px]` | Botón "[ Quitar ]" | Subir a `text-xs` + `min-h-[44px]` (R-04) |
| [`SalesReceipt.jsx`](../../apps/pos/components/SalesReceipt.jsx:128) | 128 | `text-[8px]` | Etiqueta secundaria | Subir a `text-[10px] md:text-xs` |
| [`SalesReceipt.jsx`](../../apps/pos/components/SalesReceipt.jsx:182) | 182 | `text-[8px]` | Etiqueta secundaria | Subir a `text-[10px] md:text-xs` |
| [`CheckoutScreen.jsx`](../../apps/pos/components/CheckoutScreen.jsx:194) | 194 | `text-[8px]` | "Importe a Recibir" | Subir a `text-[10px] md:text-xs` |
| [`GestionPersonal.jsx`](../../apps/pos/components/GestionPersonal.jsx:135) | 135 | `text-[8px]` | Etiqueta | Subir a `text-[10px] md:text-xs` |
| [`GestionPersonal.jsx`](../../apps/pos/components/GestionPersonal.jsx:214) | 214 | `text-[8px]` | Etiqueta | Subir a `text-[10px] md:text-xs` |
| [`GestionPersonal.jsx`](../../apps/pos/components/GestionPersonal.jsx:257) | 257 | `text-[8px]` | Etiqueta | Subir a `text-[10px] md:text-xs` |
| [`TableServicePOS.jsx`](../../apps/pos/TableServicePOS.jsx:135) | 135 | `text-[10px]` | "Orden Actual" | Mantener (ya es legible) |

### §4.3 Módulos de gestión — anchos fijos a revisar

Estos módulos ya tienen buen responsivo general, pero conservan contenedores rígidos puntuales:

| Archivo | Línea | Actual | Acción |
|---|---|---|---|
| [`AuditoriaControlUI.jsx`](../../apps/AuditoriaControlUI.jsx:231) | 231 | `w-[400px]` | `w-full max-w-[400px]` |
| [`RecursosHumanosUI.jsx`](../../apps/hr/RecursosHumanosUI.jsx:280) | 280–351 | `w-[130px]` (×13) | `w-full sm:w-[130px]` |
| [`KDSComponent.jsx`](../../apps/kds/KDSComponent.jsx:31) | 31 | `min-w-[300px]` | Revisar: si es columna de KDS, `min-w` es válido con scroll |
| [`DoughManagerUI.jsx`](../../apps/production/DoughManagerUI.jsx:1801) | 1801 | `min-w-[400px]` | Revisar: probablemente válido con scroll |
| [`ProcesoProduccionMasaUI.jsx`](../../apps/production/ProcesoProduccionMasaUI.jsx:822) | 822 | `max-w-[1400px]` | Válido (`max-w` ya es fluido) |
| [`ProcesoProduccionMasaUI.jsx`](../../apps/production/ProcesoProduccionMasaUI.jsx:917) | 917 | `max-w-[500px]` | Válido |
| [`ProductStatsView.jsx`](../../apps/production/ProductStatsView.jsx:478) | 478 | `max-w-[1800px]` | Válido |
| [`EstadisticasVentasUI.jsx`](../../apps/EstadisticasVentasUI.jsx:449) | 449 | `max-w-[1600px]` | Válido |

> **Nota:** los `max-w-[...]` **no son un problema** — ya son fluidos. El problema son los `w-[...]` fijos. La auditoría los separa para no generar trabajo innecesario.

### §4.4 Plantillas de impresión — EXENTAS

| Archivo | Motivo de exención |
|---|---|
| [`TicketTemplate.jsx`](../../apps/pos/components/TicketTemplate.jsx:7) | Papel térmico 58mm/80mm — ancho físico fijo |
| [`CorteTicketTemplate.jsx`](../../apps/pos/components/CorteTicketTemplate.jsx:65) | Papel térmico — ancho físico fijo |

**Estas plantillas NO se tocan.** Su `text-[7px]`/`text-[8px]` es correcto porque el destino es papel, no pantalla.

---

## §5. Estándar táctil de 44×44px

### §5.1 Por qué 44px

44×44px es el mínimo recomendado por Apple HIG y Google Material Design para un target táctil fiable con el dedo. En un POS de panadería, donde el cajero toca la pantalla **cientos de veces por turno** con prisa, un target de 20px produce errores de cobro — y un error de cobro es dinero perdido o un cliente enojado.

### §5.2 Elementos que NO cumplen hoy (a corregir en el Nuevo POS)

| Elemento | Archivo | Tamaño actual | Corrección |
|---|---|---|---|
| Botón "[ Quitar ]" del ticket | [`SalesReceipt.jsx:87`](../../apps/pos/components/SalesReceipt.jsx:87) | ~20×14px (`text-[9px]`) | `min-h-[44px] min-w-[44px]` |
| Etiquetas de estado | Varios | `text-[8px]` | Solo lectura — exentas |
| Celdas de numpad de PIN | [`GestionPersonal.jsx:241`](../../apps/pos/components/GestionPersonal.jsx:241) | Revisar | `min-h-[44px]` |

### §5.3 Regla de separación

Entre dos targets táctiles adyacentes debe haber **mínimo 8px** de separación, para evitar toques accidentales en el vecino.

---

## §6. Regla de oro

> ### **"La ergonomía del mostrador es intocable.**
> ### **El responsive se logra AGREGANDO modos, no DEGRADANDO el modo existente."**

**Interpretación operativa:**

1. El modo MOSTRADOR (≥1024px) es la **referencia de verdad**. Si un cambio altera cómo se ve el POS en la tablet del mostrador, **el cambio está mal**.
2. El modo COMPACTO y el modo MÓVIL son **adiciones**, no sustituciones. Se implementan con prefijos Tailwind (`md:`, `lg:`) que **no afectan** el layout base.
3. Nunca se "simplifica" el mostrador para que quepa en móvil. Se **agrega** un layout móvil distinto.
4. La estética (§1.1) viaja intacta en los 3 modos. Solo el layout (§1.2) cambia.

---

## §7. Checklist de aceptación de un componente responsivo

Antes de aceptar cualquier componente del Nuevo POS, verificar:

- [ ] **R-01:** El contenedor raíz no tiene `w-[...px]` fijo. Usa `w-full max-w-[...]` o `w-full lg:w-[...]`.
- [ ] **R-02:** No hay `text-[7px]`, `text-[8px]` ni `text-[9px]` como tamaño base legible (excepto plantillas de impresión).
- [ ] **R-03:** El componente declara explícitamente su comportamiento en los 3 modos (MÓVIL base, `md:` COMPACTO, `lg:` MOSTRADOR).
- [ ] **R-04:** Todo elemento interactivo mide ≥ 44×44px, con ≥ 8px de separación entre targets adyacentes.
- [ ] **Estética:** Colores, tipografía, sombras, radios y animaciones son idénticos al POS v22.
- [ ] **Mostrador intacto:** En una pantalla ≥1024px, el resultado visual es indistinguible del POS actual.
- [ ] **Sin scroll horizontal:** En ningún modo aparece scroll horizontal no intencional (excepto tablas con `overflow-x-auto` explícito).
- [ ] **Probado en 3 anchos:** 375px (móvil), 820px (tablet chica), 1280px (mostrador).

---

## §8. Glosario

| Término | Definición |
|---|---|
| **Modo MOSTRADOR** | Layout de referencia del POS, ≥1024px. Intocable. |
| **Modo COMPACTO** | Layout para tablet chica, 768–1023px. Ticket colapsable. |
| **Modo MÓVIL** | Layout de consulta, <768px. No es modo de cobro intensivo. |
| **Estética** | Colores, tipografía, sombras, radios, animaciones. Intocable. |
| **Layout** | Anchos, columnas, posición de elementos. Parametrizable. |
| **Target táctil** | Área interactiva de un elemento. Mínimo 44×44px. |
| **R-01…R-04** | Las 4 reglas duras de responsividad de este documento. |
| **A-06** | Acción canónica que materializa este documento en el plan de acción. |

---

## §9. Declaración de regla dura

> **Este documento es una regla dura del proyecto.**
>
> Todo componente del Nuevo POS debe cumplir R-01 a R-04. Un componente que las viole no se acepta en revisión, sin importar cuán bien se vea en el mostrador.
>
> **La estética del POS v22 se preserva al 100%. La ergonomía del mostrador es intocable. El responsive se logra agregando modos, no degradando el modo existente.**
>
> **El ERP en producción NO se toca.** Este documento describe cómo debe nacer el Nuevo POS, no cómo modificar el actual.

---

*Documento maestro #6 — Nuevo POS*
*Generado a partir de la auditoría del código real v22 (commit `fe9f6ed`; el ERP hoy corre en `5802f45`, V23)*
*28 contenedores de ancho fijo catalogados en `apps/pos/`, 81 en el ERP completo*
