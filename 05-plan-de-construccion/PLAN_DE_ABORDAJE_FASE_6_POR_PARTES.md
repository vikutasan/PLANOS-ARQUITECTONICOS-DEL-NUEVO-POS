# PLAN DE ABORDAJE — FASE 6 POR PARTES
## Impresión Térmica Unificada + Exportación de Catálogo en PDF

> **Versión:** 2.1
> **Fecha:** 2026-09-29
> **Autor:** Arquitecto del POS nuevo
> **Estado:** PROPUESTA — pendiente de aprobación del dueño
> **Cambios v2.0 → v2.1:** se elimina por completo el Entregable C ("Documento de respaldo / contingencia"), que **no fue pedido por el dueño** y fue una invención del arquitecto. El alcance queda estrictamente en **dos entregables**: (A) impresión térmica unificada y (B) exportación de catálogo en PDF con selector de categorías. Se ajustan las sub-fases (de 6 a 5) y se limpia toda la complejidad asociada al entregable eliminado.
> **Referencias:**
> - [`PLAN_MAESTRO_DEFINITIVO_POS.md`](../PLAN_MAESTRO_DEFINITIVO_POS.md) §Fase 6 + §6.2
> - [`PROMPT_DEL_ARQUITECTO_DEL_NUEVO_POS.md`](../06-prompt-del-arquitecto/PROMPT_DEL_ARQUITECTO_DEL_NUEVO_POS.md) §7.4 (estándares) + §10.8 (E-01 a E-16)
> - [`PLAN_DE_ABORDAJE_FASE_5_POR_PARTES.md`](PLAN_DE_ABORDAJE_FASE_5_POR_PARTES.md) (precedente de método)
> - POS viejo: `ticketGenerator.js` (266 líneas), `useTicketActions.js` (líneas 75–114), `GestorDeCaja.jsx` (líneas 152–181), `CorteTicketTemplate.jsx` (190 líneas)
> - POS viejo: `guia_configuracion_impresion.md` (impresión silenciosa con `--kiosk-printing`)

---

## §1. ACLARACIÓN DE ALCANCE — DOS ENTREGABLES DISTINTOS

El Plan Maestro agrupa bajo "Fase 6" cosas que **tienen propósitos opuestos**. La confusión entre ellas fue el primer defecto detectado en la autocrítica (§3, D-1). Este plan las separa explícitamente en **dos** entregables:

| | **A — Impresión térmica** | **B — Catálogo PDF** |
|---|---|---|
| **Para qué** | Entregar el comprobante al cliente en el momento | Enviar la carta al proveedor de impresiones |
| **Destino** | Impresora térmica de 80 mm, en el mostrador | Correo electrónico → proveedor externo |
| **Papel** | Rollo térmico 80 mm (ancho físico fijo) | El que el dueño decida (carta, oficio, A4, couché) |
| **Color** | Monocromo (la térmica no imprime color) | A color o B/N — **lo decide el proveedor** |
| **Quién imprime** | El POS, directo | Un tercero, desde el archivo que se le envía |
| **Frecuencia** | Cada venta / cada corte | Cuando cambian precios o productos |
| **Tecnología** | HTML + `iframe.contentWindow.print()` | **jsPDF** → archivo `.pdf` real, descargable |
| **Dependencias nuevas** | Cero | `jspdf` |

**Por qué A y B usan tecnologías distintas (decisión técnica, no preferencia):**

- El ticket térmico **no necesita un archivo PDF** — necesita mandar bytes a una impresora. El patrón `iframe.print()` ya está probado en producción en el POS viejo (`useTicketActions.js`, líneas 75–114) y no requiere dependencias.
- La carta **sí necesita un archivo `.pdf` de verdad**, porque el dueño lo va a **adjuntar a un correo**. `window.print()` abre el diálogo del navegador y obliga a "Guardar como PDF" a mano — no produce un archivo que el POS pueda descargar. Para el caso de uso real, se necesita **jsPDF**.

> **Nota sobre el color:** el POS entrega un PDF con las imágenes a color. El proveedor decide si imprime a color o en B/N. El POS **no** decide el color — eso es exactamente lo que el dueño pidió.

---

## §2. DIAGNÓSTICO — EVIDENCIA, NO OPINIÓN (E-14)

### §2.1 Qué dice el plano

**§Fase 6** declara 3 archivos:
- `TicketTemplate.jsx` — plantilla de venta
- `ticketGenerator.js` — generador de HTML para impresión
- `CatalogoPDF.jsx` — ✨ NUEVO: PDF del grid de productos

**§6.2** detalla el PDF: botón "📄 Exportar PDF" en la barra de categorías; contenido = grid (imagen + nombre + precio); agrupado por categoría; formato carta/oficio; precios vigentes al momento.

### §2.2 Qué existe HOY en el código

| Elemento | Estado real | Evidencia |
|---|---|---|
| Código de impresión en NUEVO-POS | **NO EXISTE** | `search_files` de `print\|jsPDF\|window.open` → 0 resultados en `apps/pos/src` |
| `TicketTemplate.jsx` | **NO EXISTE** | `list_files` de `components/` no lo lista |
| `ticketGenerator.js` | **NO EXISTE** | `list_files` de `utils/` solo tiene `outcome.js`, `terminalCardState.js`, `withRetries.js`, `utilidades.test.js` |
| `CatalogoPDF.jsx` | **NO EXISTE** | `list_files` de `components/` no lo lista |
| `CorteTicketTemplate.jsx` | **YA EXISTE** (F4.4) | 215 líneas; su cabecera dice *"SOLO RENDERIZA. NO imprime. La impresión física real es Fase 6"* |
| `SalesReceipt.jsx` | **YA EXISTE** (F3.4) | 169 líneas; es el ticket en pantalla, no la plantilla térmica |
| Librería PDF en `package.json` | **NO HAY** | solo `react`, `react-dom`, `react-router-dom` |
| Referencia en el POS viejo | **`ticketGenerator.js`** (266 líneas) | genera HTML térmico 80mm; patrón `iframe.contentWindow.print()` |
| Referencia PDF en el POS viejo | **NO HAY CatalogoPDF** | solo `pdfRenderer.js` (jsPDF, pero para displays de heladería, no el catálogo POS) |

### §2.3 Hallazgos que condicionan el plan

**H-1 — `TicketTemplate.jsx` es una interfaz DECLARADA que no existe.**
El registro de superficie la declara como **interfaz 23**:
```
numero=23, nombre="TicketTemplate", tipo="Impresión",
anclaje="apps/pos/components/TicketTemplate.jsx",
contenedor_raiz="w-[58mm] font-mono",
paleta=("#fdfbf7",), exenta_responsiva=True
```
El test de la puerta F5 (`test_f5_superficie.py`) verifica que cada interfaz tenga anclaje al código — pero **hoy pasa** porque solo comprueba que el *nombre* esté en el registro, no que el archivo exista. Al construir Fase 6, el archivo debe respetar **exactamente** el contrato declarado.

**H-2 — El guard R-01 prohíbe `w-[...px]` en `apps/pos/`, pero las plantillas usan `w-[58mm]`/`w-[80mm]`.**
El regex del guard es `/(?<!max-)(?<!min-)\bw-\[\d+px\]/` — solo dispara con **`px`**, no con **`mm`**. Por eso `CorteTicketTemplate.jsx` (que usa `w-[80mm]`) pasa hoy. `TicketTemplate.jsx` con `w-[58mm]` también pasará. Esto es correcto por diseño (R-02: las plantillas de impresión están exentas), pero **debe quedar documentado** para que nadie "arregle" el guard y rompa las plantillas.

**H-3 — El PDF de catálogo NO tiene precedente en el POS viejo.**
Es genuinamente nuevo. Se construye con jsPDF (decisión §1). El POS viejo tiene jsPDF en `apps/heladeria/utils/pdfRenderer.js` — sirve como referencia de API, no como código a portar.

**H-4 — El botón "📄 Exportar PDF" va en `CategoryBar.jsx` (interfaz 21).**
Hoy recibe `{ categorias, categoriaActiva, onSeleccionar }`. Añadir el botón exige una prop nueva (`onExportarPDF`) y tocar la interfaz 21 — que ya está construida y con test. Es un cambio de contrato de componente, y debe preservar los 3 modos (R-03) y el target táctil ≥44px (R-04).

**H-5 — El dueño quiere ELEGIR qué categorías incluir antes de exportar.**
Corrección de alcance dada por el dueño durante el diagnóstico. El PDF no es "todo o nada": necesita un **paso de selección previo**. Esto añade un componente (modal de selección) que el Plan Maestro no contemplaba. El propósito es evitar imprimir categorías ocultas o de prueba.

**H-6 — El POS viejo imprime el ticket y el corte con DOS arquitecturas distintas.**
Evidencia:
- **Ticket de venta:** `generateTicketHTML()` devuelve un **string HTML completo** (con su propio `<style>`), y `handlePrintTicket` lo escribe en el iframe con `doc.write(html)`. El HTML es autosuficiente: no depende de Tailwind ni del CSS de la app.
- **Corte de caja:** `handlePrintCorte` (líneas 152–181) **NO usa un string HTML**. Toma `cortePrintRef.current.innerHTML` (un componente React ya renderizado en un `<div>` oculto) y le inyecta **Tailwind por CDN** (`cdn.jsdelivr.net/npm/tailwindcss@2.2.19`). Es decir: el corte **depende de una CDN externa** para verse bien.

Esta asimetría es un **defecto del POS viejo**, no un patrón a copiar. Consecuencias reales:
1. Si no hay internet, el corte se imprime **sin estilos** (Tailwind no carga).
2. El corte depende de que React ya haya renderizado el `<div>` oculto.
3. El ticket y el corte se imprimen por caminos distintos → dos lugares donde arreglar bugs.

**H-7 — El POS viejo tiene impresión silenciosa documentada (`--kiosk-printing`).**
`guia_configuracion_impresion.md` documenta que el navegador se lanza con la bandera `--kiosk-printing`, lo que hace que `iframe.contentWindow.print()` **no muestre diálogo** y saque el papel directo. Esto es una **configuración del equipo**, no código. El POS nuevo debe **preservar esta compatibilidad** (el patrón `iframe.print()` es el que respeta `--kiosk-printing`).

---

## §3. AUTOCRÍTICA — CONTRA EL CÓDIGO REAL DEL POS VIEJO

Antes de escribir la v1.0 se hallaron 5 defectos (D-1 a D-5). Al revisar la v1.0 **contra el código real del POS viejo** (no contra la idea que tenía de él), se hallaron 4 defectos nuevos (D-6 a D-9). Y al revisar la v2.0, el dueño detectó un defecto grave de alcance (D-10). Los 10 se listan aquí.

### Defectos de la v1.0 (ya corregidos)

**D-1 (real) — Se trataba `TicketTemplate.jsx` como "archivo nuevo libre".**
Falso: es la **interfaz 23 declarada** con contrato exacto (`w-[58mm] font-mono`, paleta `#fdfbf7`, exenta R-01).
**Corrección:** la plantilla cumple el contrato declarado al pie de la letra, y el gate verifica el anclaje real.

**D-2 (real) — Se iba a añadir jsPDF sin justificarlo.**
El Plan Maestro dice "PDF" pero no dice "con qué". Añadir dependencias a un POS que funciona offline es una decisión arquitectónica, no un detalle.
**Corrección:** decisión explícita y documentada (§1). jsPDF **solo** para la carta; cero dependencias para la térmica.

**D-3 (real) — No se había delimitado el alcance de "impresión".**
El Plan Maestro mezcla 3 cosas: (1) plantilla térmica de venta, (2) generador de HTML, (3) PDF de catálogo. Además, `CorteTicketTemplate.jsx` ya existe pero **nadie lo imprime** — falta el disparador.
**Corrección:** sub-fases + declarar que el corte también se imprime en Fase 6.

**D-4 (real) — No se había verificado si el POS viejo ya resolvía el PDF.**
Verificado: **no lo resuelve**. Pero sí tiene el patrón de impresión térmica completo (266 líneas) que **debe portarse, no reinventarse** (A-01: las reglas se portan con su test).
**Corrección:** portar `ticketGenerator.js` adaptándolo al contrato `{outcome, reason}` y a los 5 campos ligeros.

**D-5 (real, detectado por el dueño) — Se asumía "todo el catálogo" sin opción de filtrar.**
El dueño corrigió: quiere **elegir qué categorías incluir** antes de exportar.
**Corrección:** añadir un modal de selección de categorías (F6.3) antes de generar el PDF.

### Defectos hallados en la revisión v2.0 (contra el código real)

**D-6 (real, grave) — La v1.0 asumía que el corte se imprimía con un string HTML, igual que el ticket.**
**Falso.** Evidencia (H-6): el POS viejo imprime el corte tomando `innerHTML` de un componente React + Tailwind por CDN. La v1.0 proponía `generarCorteHTML(corte)` como función pura — eso **no replica** lo que hacía el POS viejo, y peor: **hereda el defecto de la CDN** si se copia el patrón, o **rompe la paridad** si se inventa otro.
**Corrección (decisión arquitectónica, §4):** el POS nuevo **NO copia la asimetría**. Se unifica: **tanto el ticket como el corte se generan como string HTML autosuficiente** (con su `<style>` embebido, sin CDN). Esto es una **mejora deliberada** sobre el POS viejo, documentada como tal. El corte gana: funciona sin internet, no depende de React renderizado, y usa el mismo camino de impresión que el ticket.

**D-7 (real) — La v1.0 no mencionaba `--kiosk-printing`.**
Evidencia (H-7): la impresión silenciosa del POS viejo depende de una bandera del navegador, documentada en una guía operativa. Si el POS nuevo no preserva el patrón `iframe.print()`, **rompe la impresión silenciosa** que el personal ya tiene configurada.
**Corrección:** el plan declara explícitamente que el patrón `iframe.print()` se preserva **por compatibilidad con `--kiosk-printing`**, y la ficha de cierre incluye la guía operativa actualizada.

**D-8 (real) — La v1.0 no verificaba la numeración de interfaces del corte.**
Evidencia: el registro declara `CorteTicketTemplate` como **interfaz 24**, pero el encabezado del archivo `CorteTicketTemplate.jsx` (F4.4) dice *"interfaz 15"*. Hay una **discrepancia de numeración** entre el comentario del archivo y el registro. No rompe nada (el test solo verifica el nombre), pero es deuda documental.
**Corrección:** F6.1 corrige el comentario del archivo para que diga "interfaz 24", y el gate verifica la coherencia.

**D-9 (real) — La v1.0 no distinguía "exportar PDF" de "documento de respaldo".**
El arquitecto interpretó que la carta física impresa debía ser un **artefacto de contingencia** y añadió un Entregable C completo (con fecha de generación, precios congelados, leyenda de respaldo, instrucción de uso). **Eso no fue pedido por el dueño.**
**Corrección (v2.1):** se **elimina por completo** el Entregable C y toda su complejidad. El alcance queda en dos entregables (A y B). Ver D-10.

### Defecto detectado por el dueño en la revisión v2.0

**D-10 (real, grave, detectado por el dueño) — El arquitecto inventó un Entregable C ("Documento de respaldo / contingencia") que nunca se pidió.**
La v2.0 presentaba **tres** entregables, siendo el tercero una invención del arquitecto: un "documento de respaldo para contingencia" con requisitos propios (fecha visible, precios congelados, leyenda, instrucción operativa) y una sub-fase dedicada (F6.4). El dueño lo rechazó explícitamente: *"Te inventaste un complejo 'documento de respaldo o contingencia' (Entregable C) que jamás te pedí."*
**Corrección (v2.1):** se elimina el Entregable C, su sub-fase (F6.4) y todas sus referencias. El plan queda con **dos entregables (A y B)** y **cinco sub-fases (F6.0 a F6.4)**. Esto es un recordatorio de la regla E-14 ("evidencia, no opinión") aplicada al **alcance**: no se añaden requisitos que el dueño no pidió.

---

## §4. DECISIÓN ARQUITECTÓNICA CENTRAL — UN SOLO CAMINO DE IMPRESIÓN

Esta es la decisión más importante de la fase, y nace de D-6.

### Lo que hacía el POS viejo (asimetría)

```
TICKET:  generateTicketHTML() → string HTML autosuficiente → doc.write() → print()
CORTE:   <div ref> con React + Tailwind CDN → innerHTML → doc.write() → print()
```

Dos caminos distintos. El corte depende de internet (CDN) y de React renderizado.

### Lo que hará el POS nuevo (unificado)

```
TICKET:  generarTicketHTML(ticket) → string HTML autosuficiente → printService → print()
CORTE:   generarCorteHTML(corte)   → string HTML autosuficiente → printService → print()
```

**Un solo camino.** Ambos son strings HTML autosuficientes (con `<style>` embebido, sin CDN, sin dependencia de React). Ambos pasan por el mismo `printService.js`.

### Por qué es una mejora y no una desviación

| Aspecto | POS viejo | POS nuevo | ¿Mejor? |
|---|---|---|---|
| Ticket sin internet | ✅ funciona | ✅ funciona | Igual |
| Corte sin internet | ❌ **se imprime sin estilos** | ✅ funciona | **Mejor** |
| Corte depende de React renderizado | ❌ sí | ✅ no | **Mejor** |
| Lugares donde arreglar un bug de impresión | 2 | 1 | **Mejor** |
| Compatible con `--kiosk-printing` | ✅ | ✅ | Igual |
| Testeable sin navegador | ❌ (corte) | ✅ (ambos) | **Mejor** |

**Justificación (A-01):** las reglas de negocio se portan con su test, pero los **defectos** no se portan. La asimetría del corte es un defecto (dependencia de CDN), no una regla. Se corrige, y se documenta el porqué.

**Consecuencia para `CorteTicketTemplate.jsx` (F4.4):** el componente React **se conserva** (sirve para mostrar el corte en pantalla, dentro de `GestorDeCaja`). Lo que se añade es `generarCorteHTML()` para la **impresión**. El componente y el generador comparten la misma estructura visual, pero el generador es un string puro. Esto respeta la separación "render en pantalla" vs "render para imprimir" que ya existe entre `SalesReceipt` (pantalla) y `TicketTemplate` (impresión).

---

## §5. PRINCIPIOS QUE GOBIERNAN ESTA FASE

1. **De adentro hacia afuera (vertical slices).** Cada sub-fase entrega algo que funciona de punta a punta, no una capa horizontal.
2. **Evidencia, no opinión (E-14).** Cada afirmación del diagnóstico está anclada a un archivo y una línea. **Aplica también al alcance:** no se añaden requisitos que el dueño no pidió (D-10).
3. **Cero dependencias para lo que no las necesita.** La térmica no añade nada. La carta añade solo jsPDF.
4. **Las plantillas de impresión están exentas de R-01/R-02.** Su ancho es físico (mm), no responsivo.
5. **El contrato `{outcome, reason}` aplica a toda función que pueda fallar.** El generador de PDF puede fallar (imagen rota, canvas bloqueado) → devuelve `{outcome, reason}`.
6. **Ninguna función supera 20 líneas ni 3 niveles de anidación (E-16).**
7. **Los tests son la puerta.** Cada sub-fase cierra con su gate en verde + suite completa + guards.
8. **Un solo camino de impresión (§4).** Ticket y corte comparten el mismo generador de string y el mismo disparador.
9. **Autosuficiencia total.** Ningún documento de impresión depende de CDN, internet o React renderizado.
10. **Los defectos no se portan; las reglas sí (A-01).** La asimetría del corte se corrige, no se copia.

---

## §6. SUB-FASE 6.0 — `ticketGenerator.js` (utilidad pura)

### §6.1 Qué construye

`apps/pos/src/utils/ticketGenerator.js` — funciones puras que generan el HTML térmico de 80 mm desde los datos del ticket y del corte. Portadas del POS viejo (`apps/pos/utils/ticketGenerator.js`, 266 líneas), adaptadas y **unificadas** (§4).

### §6.2 Decisiones de diseño

- **Funciones puras, sin DOM.** Reciben datos, devuelven un string HTML. Testeables sin navegador.
- **Adaptación al POS nuevo:** el POS viejo usa `ticketData.items`, `payment_details`, `captured_by_name`, etc. El POS nuevo usa los **5 campos ligeros** del contrato 21 (`id`, `account_num`, `status`, `total`, `version`) + las líneas del ticket. El generador se adapta a la forma nueva.
- **Dos funciones exportadas:** `generarTicketHTML(ticket)` y `generarCorteHTML(corte)`.
- **`generarCorteHTML` es NUEVA (no existe en el POS viejo como string).** Replica la estructura visual de `CorteTicketTemplate.jsx` (F4.4) pero como string autosuficiente. Corrige D-6.
- **Autosuficiencia:** el `<style>` va embebido en el string. **Cero CDN.** Corrige D-6.
- **Sin `console.log` (E-05), sin TODO sin formato (E-15).**

### §6.3 Archivos

| Archivo | Acción |
|---|---|
| `apps/pos/src/utils/ticketGenerator.js` | CREAR |
| `apps/pos/src/utils/ticketGenerator.f6_0.test.jsx` | CREAR (gate) |

### §6.4 Criterios de la puerta (gate)

1. `generarTicketHTML` devuelve un string que contiene `<html>` y `@page { size: 80mm`.
2. El HTML incluye el número de cuenta (`account_num`) y el total.
3. Cada línea del ticket aparece con cantidad, nombre y precio.
4. `generarCorteHTML` devuelve un string con el fondo inicial, los movimientos y el conteo final.
5. Un ticket vacío no rompe: devuelve HTML válido con "sin líneas".
6. **Ninguna de las dos funciones contiene `http://` ni `https://` (autosuficiencia, D-6).**
7. **Ninguna de las dos funciones toca el DOM (no usa `document`).**

### §6.5 Gate de cierre

```
npx vitest run src/utils/ticketGenerator.f6_0.test.jsx   → verde
npm run test                                              → verde
node scripts/guards.mjs                                   → 7/7 verde
```

---

## §7. SUB-FASE 6.1 — `TicketTemplate.jsx` (interfaz 23)

### §7.1 Qué construye

`apps/pos/src/components/TicketTemplate.jsx` — plantilla React del ticket de venta. **Solo renderiza.** Cumple el contrato declarado de la interfaz 23.

### §7.2 Decisiones de diseño

- **Contrato del registro de superficie (H-1):** `contenedor_raiz="w-[58mm] font-mono"`, paleta `#fdfbf7`, `exenta_responsiva=True`. Se respeta al pie de la letra.
- **Solo renderiza, no imprime.** Igual que `CorteTicketTemplate.jsx` (F4.4). El disparador de impresión es F6.2.
- **Sigue el patrón de `CorteTicketTemplate.jsx`:** `formatearPrecio`, `formatearFecha`, componente `Fila`, props explícitas.
- **R-01 exenta:** usa `w-[58mm]` (mm, no px) → el guard no dispara (H-2).
- **Corrección de deuda documental (D-8):** se corrige el comentario de `CorteTicketTemplate.jsx` para que diga "interfaz 24" (no 15).

### §7.3 Archivos

| Archivo | Acción |
|---|---|
| `apps/pos/src/components/TicketTemplate.jsx` | CREAR |
| `apps/pos/src/components/TicketTemplate.f6_1.test.jsx` | CREAR (gate) |
| `apps/pos/src/components/CorteTicketTemplate.jsx` | MODIFICAR (corregir comentario "interfaz 15" → "interfaz 24", D-8) |

### §7.4 Criterios de la puerta (gate)

1. Renderiza el encabezado con "R DE RICO" y "Ticket de Venta".
2. Renderiza el número de cuenta y la fecha.
3. Renderiza cada línea con cantidad, nombre y precio.
4. Renderiza el total.
5. El contenedor raíz usa `w-[58mm]` y `font-mono`.
6. La plantilla NO llama a `window.print` (solo renderiza).
7. Un ticket vacío muestra "Ticket vacío" sin romper.
8. El comentario de `CorteTicketTemplate.jsx` dice "interfaz 24" (D-8).

### §7.5 Gate de cierre

```
npx vitest run src/components/TicketTemplate.f6_1.test.jsx   → verde
npm run test                                                  → verde
node scripts/guards.mjs                                       → 7/7 verde
```

---

## §8. SUB-FASE 6.2 — `printService.js` (el disparador único)

### §8.1 Qué construye

`apps/pos/src/services/printService.js` — el disparador de impresión térmica. Porta el patrón `iframe.contentWindow.print()` del POS viejo, envuelto en `{outcome, reason}`. **Un solo servicio para ticket y corte** (§4).

### §8.2 Decisiones de diseño

- **Patrón del POS viejo (D-4):** crear un `<iframe>` oculto, escribir el HTML, esperar, `focus()` + `print()`, y remover el iframe. Ya probado en producción.
- **Compatibilidad con `--kiosk-printing` (D-7):** se preserva el patrón exacto (`iframe.contentWindow.print()`) porque es el que respeta la bandera de impresión silenciosa que el personal ya tiene configurada. **No se usa `window.print()`** (rompería la impresión silenciosa).
- **Contrato `{outcome, reason}`:** `imprimirTicket(ticket)` e `imprimirCorte(corte)` devuelven `{outcome, reason}`. Nunca lanzan.
- **Inyectable para tests:** el servicio recibe `documento` (por defecto `document`) para poder testear sin navegador real.
- **Cablea los dos casos:** venta (F6.1) y corte (`CorteTicketTemplate.jsx`, que hoy no se imprime).
- **Recibe el HTML ya generado** (de F6.0), no lo genera él. Separación de responsabilidades: F6.0 genera, F6.2 imprime.

### §8.3 Archivos

| Archivo | Acción |
|---|---|
| `apps/pos/src/services/printService.js` | CREAR |
| `apps/pos/src/services/printService.f6_2.test.jsx` | CREAR (gate) |

### §8.4 Criterios de la puerta (gate)

1. `imprimirTicket` devuelve `{outcome: 'ok'}` cuando el iframe se crea y se imprime.
2. `imprimirTicket` devuelve `{outcome: 'fallo', reason: ...}` si el documento no está disponible.
3. El iframe se remueve del DOM tras imprimir.
4. `imprimirCorte` usa el HTML del corte, no el del ticket.
5. El servicio NO lanza excepciones (contrato `{outcome, reason}`).
6. El servicio usa `documento` inyectado, no `document` global directo.
7. **El servicio usa `iframe.contentWindow.print()`, NO `window.print()` (compatibilidad `--kiosk-printing`, D-7).**

### §8.5 Gate de cierre

```
npx vitest run src/services/printService.f6_2.test.jsx   → verde
npm run test                                              → verde
node scripts/guards.mjs                                   → 7/7 verde
```

---

## §9. SUB-FASE 6.3 — `CatalogoPDF.jsx` + selector de categorías

### §9.1 Qué construye

- `apps/pos/src/components/CatalogoPDF.jsx` — genera el PDF del catálogo con jsPDF.
- `apps/pos/src/components/SelectorCategoriasPDF.jsx` — modal para elegir qué categorías incluir (H-5).
- Botón "📄 Exportar PDF" en `CategoryBar.jsx` (interfaz 21).

### §9.2 Decisiones de diseño

- **jsPDF (decisión §1):** genera un archivo `.pdf` real, descargable, adjuntable a un correo.
- **Selección de categorías (H-5):** el modal lista las categorías con checkboxes; por defecto todas marcadas. El usuario desmarca las que no quiere (para evitar imprimir categorías ocultas o de prueba).
- **Contenido del PDF (confirmado por el dueño):** logo + nombre del negocio + fecha, y cada categoría seleccionada con sus productos (imagen + nombre + precio).
- **Multipágina:** si el catálogo no cabe en una página, jsPDF añade páginas automáticamente.
- **Imágenes:** se cargan desde `producto.image_url`; si una imagen falla, se dibuja el ícono/emoji como respaldo (degradación elegante).
- **Contrato `{outcome, reason}`:** `generarCatalogoPDF(categorias)` devuelve `{outcome, reason}`.
- **El botón respeta R-03 (3 modos) y R-04 (target ≥44px).**

### §9.3 Archivos

| Archivo | Acción |
|---|---|
| `apps/pos/src/components/CatalogoPDF.jsx` | CREAR |
| `apps/pos/src/components/SelectorCategoriasPDF.jsx` | CREAR |
| `apps/pos/src/components/CategoryBar.jsx` | MODIFICAR (añadir botón + prop) |
| `apps/pos/src/components/CatalogoPDF.f6_3.test.jsx` | CREAR (gate) |
| `apps/pos/package.json` | MODIFICAR (añadir `jspdf`) |

### §9.4 Criterios de la puerta (gate)

1. `generarCatalogoPDF` devuelve `{outcome: 'ok'}` con categorías válidas.
2. El PDF incluye el nombre del negocio y la fecha.
3. Cada categoría seleccionada aparece con su nombre.
4. Cada producto aparece con nombre y precio.
5. Una categoría sin productos no rompe la generación.
6. `generarCatalogoPDF([])` devuelve `{outcome: 'fallo', reason: ...}` (no genera PDF vacío).
7. El modal de selección lista todas las categorías con checkbox.
8. El botón "📄 Exportar PDF" existe en `CategoryBar` y dispara el modal.
9. El botón respeta R-03 (visible en los 3 modos) y R-04 (target ≥44px).
10. `CategoryBar` sigue funcionando con sus props originales (no rompe el test de F3.4).

### §9.5 Gate de cierre

```
npx vitest run src/components/CatalogoPDF.f6_3.test.jsx   → verde
npm run test                                               → verde
node scripts/guards.mjs                                    → 7/7 verde
```

---

## §10. SUB-FASE 6.4 — Cierre de Fase 6

### §10.1 Qué construye

`docs/05-plan-de-construccion/FICHA_F6_CIERRE.md` — la ficha de cierre que consolida las 4 sub-fases, con la evidencia de cada gate en verde.

### §10.2 Contenido de la ficha

1. **Resumen de las 4 sub-fases** (F6.0 a F6.3) con su commit.
2. **Tabla de paridad con el POS viejo:** qué se portó igual (ticket), qué se mejoró (corte sin CDN), qué es nuevo (carta PDF).
3. **Guía operativa de impresión actualizada:** la bandera `--kiosk-printing` (H-7), cómo configurarla, y qué hacer si no está.
4. **Los 9 defectos (D-1 a D-9) y cómo se corrigieron.**
5. **Evidencia de los gates:** salida de `npm run test` y `node scripts/guards.mjs`.

### §10.3 Archivos

| Archivo | Acción |
|---|---|
| `docs/05-plan-de-construccion/FICHA_F6_CIERRE.md` | CREAR |

### §10.4 Gate de cierre

```
npm run test                → verde (suite completa)
node scripts/guards.mjs     → 7/7 verde
```

---

## §11. RIESGOS Y MITIGACIONES

| Riesgo | Probabilidad | Impacto | Mitigación |
|---|---|---|---|
| jsPDF infla el bundle del POS | Media | Bajo | Se importa **solo** en `CatalogoPDF.jsx` (lazy import si el bundle crece) |
| Las imágenes del catálogo no cargan (CORS) | Media | Medio | Degradación elegante: si la imagen falla, se dibuja el ícono/emoji (F6.3, criterio 5) |
| `--kiosk-printing` no está configurada en el equipo | Media | Alto | La ficha de cierre incluye la guía operativa (F6.4) |
| El corte impreso difiere del mostrado en pantalla | Baja | Medio | `generarCorteHTML` replica la estructura de `CorteTicketTemplate.jsx`; el gate lo verifica |
| El guard R-01 se "arregla" y rompe las plantillas | Baja | Alto | H-2 queda documentado en el plan y en la ficha de cierre |

---

## §12. ORDEN DE EJECUCIÓN Y CRITERIO DE APROBACIÓN

### §12.1 Orden

```
F6.0  ticketGenerator.js          (utilidad pura, sin UI)
  ↓
F6.1  TicketTemplate.jsx          (interfaz 23, solo renderiza)
  ↓
F6.2  printService.js             (disparador único, iframe.print)
  ↓
F6.3  CatalogoPDF.jsx + selector  (carta PDF, jsPDF)
  ↓
F6.4  FICHA_F6_CIERRE.md          (cierre de fase)
```

Cada sub-fase cierra con: gate en verde + suite completa + guards 7/7 + ficha + commit + push. **No se abre la siguiente sin cerrar la anterior.**

### §12.2 Criterio de aprobación

Este plan se considera aprobado cuando el dueño confirma:

1. Que la **separación en 2 entregables** (§1) refleja lo que pidió.
2. Que la **unificación del camino de impresión** (§4) es aceptable como mejora sobre el POS viejo.
3. Que el **orden de ejecución** (§12.1) es el correcto.
4. Que el **selector de categorías** (§9) cubre lo que necesita para excluir categorías ocultas o de prueba.

### §12.3 Lo que este plan NO hace (delimitación explícita)

- **No** implementa envío de correo. El PDF se descarga; el dueño lo adjunta manualmente.
- **No** implementa impresión directa a la impresora térmica desde el navegador sin diálogo (eso depende de `--kiosk-printing`, que es configuración del equipo, no código).
- **No** toca el POS viejo (`ERP-R-DE-RICO`). Solo lo lee como referencia.
- **No** modifica los contratos del backend. Fase 6 es 100% frontend.
- **No** añade dependencias más allá de `jspdf`.
- **No** incluye ningún "documento de respaldo / contingencia" (Entregable C eliminado en v2.1, D-9).
