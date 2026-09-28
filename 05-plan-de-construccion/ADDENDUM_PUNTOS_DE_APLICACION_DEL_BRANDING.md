# Addendum — Puntos de aplicación del branding (v3 — referencia de planificación)

> **Documento padre:** [`PROPUESTA_BRANDING_TRANSVERSAL_DESDE_VISTA_GENERAL.md`](./PROPUESTA_BRANDING_TRANSVERSAL_DESDE_VISTA_GENERAL.md) (v2, commit `fe255bf`)
> **Complementa a:** [`PROPUESTA_PALETA_CANONICA_V2.md`](./PROPUESTA_PALETA_CANONICA_V2.md) (3 candidatas, commit `962cc64`)
> **Creado:** 26 Sep 2026
> **Corregido:** 27 Sep 2026 — se reemplaza la lista de puntos de aplicación por la instrucción directa del dueño.
> **Corregido v3:** 28 Sep 2026 — se aclara que los 7 puntos son **referencia de planificación**, no trabajo actual. La regla dura se mantiene intacta.
> **Motivo de la corrección v3:** el dueño precisó que **NO se toca el ERP ni el POS que corre**. Los 7 puntos describen **dónde irá la identidad cuando esos módulos se reconstruyan**, no dónde hay que ir a tocar código hoy. Este documento es un **plano de referencia**, no una orden de trabajo.

---

## 0. La corrección, en una frase

La v1 de este addendum respondía a la pregunta *"¿dónde se ve la identidad en el POS?"*. La pregunta correcta era *"¿dónde irá la identidad en el ERP cuando se reconstruya?"*. Son dos preguntas distintas y la respuesta cambia por completo.

### 0.1 La regla dura se mantiene intacta (esto es lo más importante)

> **NO SE TOCA EL ERP INSTALADO Y CORRIENDO. NO SE TOCA EL POS QUE CORRE.**

Este addendum **no autoriza tocar nada**. Es un **documento de planificación**. Los 7 puntos que se listan abajo son **referencia de dónde irá la identidad** cuando esos módulos del ERP se reconstruyan. Hoy **no se toca ninguno**.

### 0.2 Qué se hace hoy y qué no

| Hoy (POS nuevo) | Mañana (cuando se reconstruyan los módulos) |
|---|---|
| Se explora cómo el POS nuevo **interactuará** con los módulos que aún no se construyen. | Se implementan los 7 puntos de aplicación en los módulos reconstruidos. |
| Se deja **todo preparado** (contratos, llaves, motor) para cuando esos módulos existan. | Se conecta la identidad a los puntos ya definidos. |
| **No se toca el ERP. No se toca el POS que corre.** | Se toca **solo lo reconstruido**, nunca lo que corre hoy. |

**Regla dura que gobierna este addendum:** la identidad del negocio (logotipo, nombre de marca y colores corporativos) aplicará **solo en los 7 lugares del ERP listados abajo**, **cuando esos módulos se reconstruyan**. Fuera de esos 7, la identidad **no aplica**. Y de esos 7, **6 no tocarán el nuevo POS que estamos construyendo**.

---

## 1. Los 7 puntos de aplicación (lista cerrada — instrucción del dueño)

| # | Punto | Logotipo | Nombre de marca | Colores corporativos | ¿Toca el nuevo POS? |
|---|-------|----------|-----------------|----------------------|---------------------|
| **1** | Pantalla de logueo al ERP (donde se escribe la clave de acceso) | ✅ Sí | ✅ Sí | ✅ Sí | ❌ **No** |
| **2** | Parte superior de la barra selectora de módulos (lateral izquierdo del ERP) | ✅ Sí | ✅ Sí | ✅ Sí | ❌ **No** |
| **3** | Encabezado del módulo Vista General (sitio desde donde se configura la identidad) | ✅ Sí | ✅ Sí | ✅ Sí | ❌ **No** |
| **4** | Tickets y cortes de caja | ✅ Sí | ✅ Sí | ❌ **No** (solo logo y nombre) | ⚠️ **Parcial** (ver §5) |
| **5** | Documentos oficiales que genere el ERP (impresión o exportación) | ✅ Sí | ✅ Sí | ✅ Sí (solo en el encabezado del documento) | ❌ **No** |
| **6** | Icono de acceso de la app del ERP (launcher / icono en el escritorio o dispositivo) | ✅ Sí | ❌ No | ✅ Sí | ❌ **No** |
| **7** | Pantalla de carga de la app del ERP (splash / arranque) | ✅ Sí | ✅ Sí | ✅ Sí | ❌ **No** |

**Lectura de la tabla:** los puntos 1, 2, 3, 5, 6 y 7 son **100% del ERP** y no tocan el nuevo POS. El punto 4 (tickets y cortes) es el único que tiene relación con el POS, y esa relación se resuelve en §5.

**Naturaleza de esta tabla (importante):** es una **referencia de planificación**. Describe **dónde irá la identidad cuando esos módulos del ERP se reconstruyan**. Hoy **no se toca ninguno de los 7 puntos**. El trabajo actual es solo **explorar cómo el POS nuevo interactuará** con esos módulos que aún no se construyen, y **dejar todo preparado** para cuando existan.

---

## 2. Punto 1 — Pantalla de logueo al ERP

### 2.1 Qué aplica

| Elemento | ¿Aplica? | Dónde exactamente |
|----------|----------|-------------------|
| Logotipo | ✅ Sí | En el bloque de marca del formulario de acceso |
| Nombre de marca | ✅ Sí | Junto al logotipo, con la tipografía corporativa |
| Colores corporativos | ✅ Sí | En el botón de "Entrar" y en la franja de marca del formulario |

### 2.2 Qué NO aplica

- El fondo de la pantalla → **tema del ERP**
- Los campos de texto (usuario, clave) → **tema del ERP**
- Los mensajes de error → **tema del ERP**

### 2.3 Alcance

**Este punto NO toca el nuevo POS.** Es la pantalla de acceso del ERP instalado.

**Estado hoy: NO SE IMPLEMENTA.** Es referencia de planificación. Se implementará **cuando el módulo de acceso del ERP se reconstruya**, nunca sobre el ERP que corre hoy.

---

## 3. Punto 2 — Barra selectora de módulos (lateral izquierdo)

### 3.1 Qué aplica

| Elemento | ¿Aplica? | Dónde exactamente |
|----------|----------|-------------------|
| Logotipo | ✅ Sí | En la **parte superior** de la barra lateral, arriba de la lista de módulos |
| Nombre de marca | ✅ Sí | Debajo o junto al logotipo, en el encabezado de la barra |
| Colores corporativos | ✅ Sí | En el fondo del encabezado de la barra y en el indicador del módulo activo |

### 3.2 Qué NO aplica

- Los iconos y nombres de los módulos → **tema del ERP**
- El fondo del cuerpo de la barra → **tema del ERP**
- Los estados hover / focus de cada módulo → **tema del ERP**

### 3.3 Alcance

**Este punto NO toca el nuevo POS.** Es la barra de navegación del ERP instalado.

**Estado hoy: NO SE IMPLEMENTA.** Es referencia de planificación. Se implementará **cuando la barra de navegación del ERP se reconstruya**, nunca sobre el ERP que corre hoy.

---

## 4. Punto 3 — Encabezado del módulo Vista General

### 4.1 Qué aplica

| Elemento | ¿Aplica? | Dónde exactamente |
|----------|----------|-------------------|
| Logotipo | ✅ Sí | En el encabezado del módulo Vista General |
| Nombre de marca | ✅ Sí | En el encabezado, con la tipografía corporativa |
| Colores corporativos | ✅ Sí | En el encabezado del módulo y en los controles de configuración de la identidad |

### 4.2 Por qué este punto es especial

Vista General es **el sitio desde donde se configura esta identidad de negocio**. Es el único punto de los 7 donde la identidad se **edita** además de **mostrarse**. Los otros 6 puntos solo la **muestran**.

Esto significa que Vista General tiene dos responsabilidades distintas:

1. **Mostrar** la identidad (como los otros puntos).
2. **Editar** la identidad (logotipo, nombre de marca, colores corporativos) y guardarla en la BD.

### 4.3 Qué NO aplica

- El cuerpo del formulario de configuración → **tema del ERP**
- Las tablas y listas de datos → **tema del ERP**

### 4.4 Alcance

**Este punto NO toca el nuevo POS.** Es el módulo Vista General del ERP instalado.

**Estado hoy: NO SE IMPLEMENTA.** Es referencia de planificación. Se implementará **cuando Vista General se reconstruya**, nunca sobre el ERP que corre hoy.

---

## 5. Punto 4 — Tickets y cortes de caja

### 5.1 Qué aplica

| Elemento | ¿Aplica? | Dónde exactamente |
|----------|----------|-------------------|
| Logotipo | ✅ Sí | En el encabezado del ticket y del corte |
| Nombre de marca | ✅ Sí | En el encabezado, junto al logotipo |
| Colores corporativos | ❌ **No** | **Solo logotipo y nombre de marca. Sin colores.** |

### 5.2 La regla explícita del dueño

> **"En los tickets y cortes de caja (solo logotipo y nombre de marca sin colores corporativos)."**

Esta es una decisión deliberada y tiene sentido técnico: los tickets se imprimen en papel térmico, donde el color no se reproduce (sale en escala de grises o se pierde). Aplicar color corporativo a un ticket térmico es trabajo perdido y puede reducir la legibilidad. **Solo logo y nombre.**

### 5.3 Alcance — el único punto con relación al POS

Este es el único de los 7 puntos que tiene relación con el nuevo POS, y la relación se resuelve así:

- **El formato del ticket es el mismo** en el ERP y en el nuevo POS (ya está en los criterios de aceptación: *"El ticket impreso es idéntico al del POS actual"*).
- **La identidad que se imprime** (logo + nombre) se lee de la misma fuente de configuración.
- **El nuevo POS no se toca para implementar este punto** en el ERP. Lo que se comparte es el **formato del ticket**, no el código del ERP.

**Precisión importante:** el nuevo POS ya tiene su propio generador de tickets. Este punto del addendum describe el ticket **del ERP**. La coherencia entre ambos tickets es un criterio de aceptación del POS nuevo, no una modificación del ERP.

---

## 6. Punto 5 — Documentos oficiales del ERP (impresión o exportación)

### 6.1 Qué aplica

| Elemento | ¿Aplica? | Dónde exactamente |
|----------|----------|-------------------|
| Logotipo | ✅ Sí | En el encabezado del documento |
| Nombre de marca | ✅ Sí | En el encabezado, junto al logotipo |
| Colores corporativos | ✅ Sí | **Solo en el encabezado del documento** |

### 6.2 Qué documentos entran

Cualquier documento oficial que el ERP genere, ya sea por **impresión** o por **exportación**:

- Reportes impresos
- Estados de cuenta
- Órdenes de compra / pedidos
- Cualquier exportación (PDF, Excel, etc.) que lleve encabezado de marca

### 6.3 La regla explícita del dueño

> **"En los documentos oficiales que genere el ERP ya sea por impresión o por exportación (acá sí se podría incluir color en el encabezado de dichos documentos)."**

**El color se limita al encabezado.** El cuerpo del documento usa el tema del ERP. Esto evita que un documento oficial se vuelva ilegible por una combinación de colores.

### 6.4 Alcance

**Este punto NO toca el nuevo POS.** Son documentos del ERP instalado.

**Estado hoy: NO SE IMPLEMENTA.** Es referencia de planificación. Se implementará **cuando el generador de documentos del ERP se reconstruya**, nunca sobre el ERP que corre hoy.

---

## 7. Punto 6 — Icono de acceso de la app del ERP

### 7.1 Qué aplica

| Elemento | ¿Aplica? | Dónde exactamente |
|----------|----------|-------------------|
| Logotipo | ✅ Sí | Como icono de la app (launcher / escritorio / dispositivo) |
| Nombre de marca | ❌ No | El icono no lleva texto de marca |
| Colores corporativos | ✅ Sí | Como fondo o acento del icono |

### 7.2 Qué es este punto

Es el **icono con el que se abre la app del ERP**: el icono en el escritorio, en la barra de tareas, o en la pantalla de inicio del dispositivo. Es la primera impresión de la marca antes de abrir la aplicación.

### 7.3 Alcance

**Este punto NO toca el nuevo POS.** Es el icono de acceso del ERP instalado.

**Estado hoy: NO SE IMPLEMENTA.** Es referencia de planificación. Se implementará **cuando el empaquetado de la app del ERP se reconstruya**, nunca sobre el ERP que corre hoy.

---

## 8. Punto 7 — Pantalla de carga de la app del ERP

### 8.1 Qué aplica

| Elemento | ¿Aplica? | Dónde exactamente |
|----------|----------|-------------------|
| Logotipo | ✅ Sí | Centrado en la pantalla de carga (splash) |
| Nombre de marca | ✅ Sí | Debajo del logotipo |
| Colores corporativos | ✅ Sí | En el fondo de la pantalla de carga |

### 8.2 Qué es este punto

Es la **pantalla que se muestra mientras la app del ERP arranca** (splash screen). Es el momento en que el usuario espera y ve la marca.

### 8.3 Alcance

**Este punto NO toca el nuevo POS.** Es la pantalla de arranque del ERP instalado.

**Estado hoy: NO SE IMPLEMENTA.** Es referencia de planificación. Se implementará **cuando el empaquetado de la app del ERP se reconstruya**, nunca sobre el ERP que corre hoy.

---

## 9. Lo que la identidad NUNCA toca (en el ERP)

Fuera de los 7 puntos de §1, la identidad **no aplica**. Lo manda el tema del ERP:

- Fondos de pantalla y de módulos
- Superficies / paneles / tarjetas
- Bordes, sombras y radios
- Estados hover / focus / disabled / active
- Texto de cuerpo, tablas, listas y formularios
- Iconos de módulos y de acciones
- Modales y diálogos

**Regla:** si un punto no está en la tabla de §1, no lleva identidad.

---

## 10. Los 3 estados del logotipo (sin cambios)

El logotipo se evalúa en orden, y **nunca queda vacío**:

| Orden | Estado | Condición | Qué se muestra |
|-------|--------|-----------|----------------|
| 1 | **Logo del negocio** | `business_logo` existe Y la URL carga | La imagen del negocio |
| 2 | **Wordmark del sistema** | `business_logo` no existe, o la URL falla | Texto "R de Rico" con la fuente canónica |
| 3 | **Nunca vacío** | — | (el estado 2 siempre está disponible) |

**Por qué el fallback es un wordmark y no una imagen:** si el fallback fuera otra imagen remota, y la red se cae (que es exactamente el escenario que dispara el fallback), el fallback también fallaría. Un wordmark es **texto renderizado con una fuente que ya está en el bundle** — no puede fallar por red.

```
Estado 1: <img src={business_logo} onError={() => setEstado(2)} />
Estado 2: <span className="font-marca text-acento">R de Rico</span>
```

---

## 11. Los colores corporativos: de 1 a 3 (no forzosamente 3)

### 11.1 La precisión del dueño

> **"La precisión que puede ser de 1 a 3 colores corporativos, no forzosamente 3."**

### 11.2 La regla

El negocio elige **entre 1 y 3** colores corporativos de la paleta curada. No está obligado a elegir 3.

| Cantidad | Caso de uso | Cómo se deriva el resto |
|----------|-------------|-------------------------|
| **1 color** | Negocio minimalista o con una sola marca | El sistema deriva los demás por contraste |
| **2 colores** | Caso más común (primario + secundario) | El sistema deriva los demás por contraste |
| **3 colores** | Negocio con paleta completa | El sistema deriva los demás por contraste |

**En los 3 casos, el sistema deriva automáticamente** el fondo, el panel, el texto y el peligro por contraste. El negocio nunca elige un fondo ilegible porque no elige el fondo: lo deriva el sistema.

### 11.3 Implicación en la validación

La validación de contraste (§12.1) se aplica **al conjunto de 1 a 3 colores que el negocio haya elegido**, no a un conjunto fijo de 3. Si elige 1, se valida ese 1. Si elige 3, se validan los 3.

---

## 12. Mejoras incorporadas a este addendum

Estas mejoras se agregan a la propuesta original para cerrar huecos detectados en la revisión técnica.

### 12.1 Validar contraste DESPUÉS de fusionar la identidad

**Problema:** validar el contraste del tema aislado no garantiza que la UI final se lea, porque la identidad se aplica **encima** del tema. Un acento corporativo claro sobre un fondo claro pasa el test del tema pero rompe la UI final.

**Regla nueva:** el contraste se valida sobre el **resultado final** (`fusionarIdentidad(tema, identidad)`), no solo sobre el tema. Si el resultado no pasa WCAG AA, se descarta el color corporativo y se usa el canónico más cercano, **con aviso visible**.

### 12.2 Una sola llave JSON para los temas, no una llave por módulo

**Problema:** una llave por módulo (`business_theme_pos`, `business_theme_estadisticas`, ...) crece a mano y se desincroniza.

**Regla nueva:** una sola llave `business_themes` con un mapa JSON:

```json
{
  "pos": "nocturno",
  "estadisticas": "tablero",
  "vista-general": "kiosco"
}
```

Es más fácil de leer, de escribir y de migrar. El contrato `GET /branding` ya devuelve un objeto `themes`, así que la BD refleja eso.

### 12.3 Los temas: mínimo 1, máximo 3 (no "exactamente 3")

**Problema:** obligar a exactamente 3 temas por módulo fuerza a inventar temas que nadie pidió, sobre todo en módulos de configuración como Vista General.

**Regla nueva:** cada módulo declara **entre 1 y 3** temas, con el predeterminado obligatorio. El test valida el rango, no el número exacto.

### 12.4 El tema se aplica al contenedor del módulo, no al `:root` global

**Problema:** si cada módulo escribe su tema en el `:root` global, al navegar entre módulos hay un parpadeo con el tema viejo.

**Regla nueva:** el tema se aplica al **contenedor del módulo**, no al `:root`. Así no hay contaminación cruzada ni parpadeo entre módulos.

### 12.5 El tema predeterminado se importa estáticamente

**Problema:** si el default se carga con `import()` dinámico, la primera pintura ocurre sin tema y luego salta (FOUC).

**Regla nueva:** el tema predeterminado se importa **estáticamente** (siempre en el bundle). Solo los temas opcionales usan `import()` dinámico.

---

## 13. El aviso visible (2 casos)

| Caso | Cuándo | Qué dice el aviso |
|------|--------|-------------------|
| **A — Vista General caída** | No se pudo leer `business_*` | "Usando apariencia por defecto. La personalización no está disponible." |
| **B — Contraste insuficiente** | Un color corporativo no cumple el contraste mínimo | "El color institucional no cumple el contraste mínimo. Se usó el más cercano." |

El aviso es **visible pero no bloqueante**: no impide trabajar. Es un banner discreto, no un modal.

---

## 14. Resumen ejecutivo (una tabla)

| # | Punto de aplicación | Logo | Nombre | Colores | ¿Toca el nuevo POS? | Dónde se implementará (referencia futura) |
|---|---------------------|------|--------|---------|---------------------|---------------------|
| 1 | Pantalla de logueo al ERP | ✅ | ✅ | ✅ | ❌ No | ERP reconstruido (no hoy) |
| 2 | Barra selectora de módulos (lateral izq.) | ✅ | ✅ | ✅ | ❌ No | ERP reconstruido (no hoy) |
| 3 | Encabezado de Vista General | ✅ | ✅ | ✅ | ❌ No | ERP reconstruido (no hoy) |
| 4 | Tickets y cortes de caja | ✅ | ✅ | ❌ No | ⚠️ Parcial (solo formato) | ERP reconstruido + criterio del POS |
| 5 | Documentos oficiales (impresión/exportación) | ✅ | ✅ | ✅ (solo encabezado) | ❌ No | ERP reconstruido (no hoy) |
| 6 | Icono de acceso de la app del ERP | ✅ | ❌ No | ✅ | ❌ No | ERP reconstruido (no hoy) |
| 7 | Pantalla de carga de la app del ERP | ✅ | ✅ | ✅ | ❌ No | ERP reconstruido (no hoy) |

> **Recordatorio:** "ERP reconstruido (no hoy)" significa que **hoy no se toca nada**. La columna describe **dónde irá** la identidad cuando esos módulos se reconstruyan. Ninguna fila autoriza trabajo sobre el ERP que corre.

**Llaves de configuración (corregidas):**

| Llave | Tipo | Valor por defecto | Descripción |
|-------|------|-------------------|-------------|
| `business_logo` | text (URL/ruta) | `""` | Ruta del logotipo. Vacío = wordmark "R de Rico". |
| `business_name` | text | `"R de Rico"` | Nombre de marca. Ya existe. |
| `business_colors` | json | `[]` | **De 1 a 3** colores institucionales de la paleta curada. Vacío = canónica. |
| `business_font` | text | `"inter"` | Una de: `inter`, `roboto`, `montserrat`, `nunito`, `poppins`. |
| `business_themes` | json | `{}` | Mapa `{ modulo: tema }`. Una sola llave, no una por módulo. |

---

## 15. Impacto en las fases (corregido v3)

| Fase | Qué agrega | ¿Toca el ERP? | ¿Toca el nuevo POS? |
|------|-----------|---------------|---------------------|
| **A** | Colapsar las 3 fuentes de paleta en 1. Motor de temas compartido. | **No** | Sí (solo el POS nuevo) |
| **B** | Las 5 llaves (`business_logo`, `business_name`, `business_colors`, `business_font`, `business_themes`) con seed. | **No** | Sí (BD del POS nuevo) |
| **C** | Los 7 puntos de aplicación en el ERP (logueo, barra lateral, Vista General, tickets, documentos, icono de app, pantalla de carga). | **No — es referencia futura** | No |
| **D** | Texturas, SVG local del logo, temas adicionales. | **No** | Sí (solo el POS nuevo) |

**Cambio clave respecto a la v2:** la Fase C **NO toca el ERP hoy**. Los 7 puntos son **referencia de planificación** para cuando esos módulos se reconstruyan. Por eso **no requiere ventana de mantenimiento**: no hay nada que mantener porque no se toca nada. Las fases A, B y D siguen siendo aisladas al POS nuevo.

**Lo que sí se hace hoy (Fase C, versión real):** solo **explorar cómo el POS nuevo interactuará** con esos módulos que aún no se construyen, y **dejar todo preparado** (contratos, llaves, motor) para cuando existan. Nada más.

---

## 16. Estado

- **Creado:** 26 Sep 2026
- **Corregido:** 27 Sep 2026 — puntos de aplicación reemplazados por la instrucción directa del dueño (7 puntos, no 5; se agregan icono de acceso y pantalla de carga). Colores corporativos: de 1 a 3, no forzosamente 3.
- **Corregido v3:** 28 Sep 2026 — se aclara que los 7 puntos son **referencia de planificación**, no trabajo actual. La regla dura (**NO se toca el ERP ni el POS que corre**) se mantiene intacta. La Fase C vuelve a **"No toca el ERP"**.
- **Estado:** propuesta — pendiente de aprobación del dueño.
- **Depende de:** [`PROPUESTA_BRANDING_TRANSVERSAL_DESDE_VISTA_GENERAL.md`](./PROPUESTA_BRANDING_TRANSVERSAL_DESDE_VISTA_GENERAL.md) y [`PROPUESTA_PALETA_CANONICA_V2.md`](./PROPUESTA_PALETA_CANONICA_V2.md)
- **Pendiente de reconciliar:** [`PROPUESTA_APARIENCIA_POR_MODULO_V4.md`](./PROPUESTA_APARIENCIA_POR_MODULO_V4.md) debe reflejar también que los 7 puntos son referencia futura y que la Fase C no toca el ERP.
- **Siguiente paso:** el dueño aprueba este addendum corregido; luego se alinea la V4 y llegan las fichas de Pinterest para fijar la paleta canónica.
