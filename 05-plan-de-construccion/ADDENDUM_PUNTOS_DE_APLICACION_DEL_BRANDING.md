# Addendum — Puntos de aplicación del branding

> **Documento padre:** [`PROPUESTA_BRANDING_TRANSVERSAL_DESDE_VISTA_GENERAL.md`](./PROPUESTA_BRANDING_TRANSVERSAL_DESDE_VISTA_GENERAL.md) (v2, commit `fe255bf`)
> **Complementa a:** [`PROPUESTA_PALETA_CANONICA_V2.md`](./PROPUESTA_PALETA_CANONICA_V2.md) (3 candidatas, commit `962cc64`)
> **Creado:** 26 Sep 2026
> **Motivo:** la v2 definió **qué** capas existen (IDENTIDAD / TEMA / CANÓNICA) pero dejó implícito **dónde** aplica cada una. Este addendum cierra esa ambigüedad con listas cerradas.

---

## 0. El problema que resuelve este addendum

La propuesta v2 decía "la identidad se aplica encima del tema". Eso es correcto pero insuficiente: sin una **lista cerrada de puntos de aplicación**, cada desarrollador decide dónde poner el color corporativo, y el resultado es el "collage" que ya advertimos (un acento aquí, otro allá, sin jerarquía).

**Regla de este addendum:** cada capa tiene una **lista cerrada** de puntos donde aplica. Fuera de esa lista, la capa **no aplica**. Lo que no está en la lista, lo manda el tema.

---

## 1. Jerarquía de capas (el "piso" y las "excepciones")

```
┌─────────────────────────────────────────────────────────┐
│  CAPA 3 — CANÓNICA                                      │  ← fallback de TODO
│  (piso del sistema: si nada más existe, esto se ve)     │
└─────────────────────────────────────────────────────────┘
                          ▲
┌─────────────────────────────────────────────────────────┐
│  CAPA 2 — TEMA                                          │  ← domina el 90%
│  (fondos, superficies, bordes, sombras, radios,         │
│   densidad, estilos de modal y de botón)                │
└─────────────────────────────────────────────────────────┘
                          ▲
┌─────────────────────────────────────────────────────────┐
│  CAPA 1 — IDENTIDAD                                     │  ← excepciones puntuales
│  (logo + tipografía + 2-3 colores institucionales)      │
│  aplica SOLO en los puntos listados abajo               │
└─────────────────────────────────────────────────────────┘
```

**Fórmula de resolución:**

```
UI final = CANÓNICA ⊕ TEMA ⊕ IDENTIDAD
```

Donde `⊕` significa "sobrescribe solo en los puntos declarados". La IDENTIDAD nunca sobrescribe un punto que no esté en su lista.

---

## 2. Punto 1 — LOGOTIPO

### 2.1 Decisión

El logotipo **solo aparece donde el usuario lo establezca**. Pero si la comunicación con Vista General se pierde, la UI **no puede quedar sin marca**. Por eso hay **3 estados explícitos**, evaluados en orden:

| Orden | Estado | Condición | Qué se muestra |
|-------|--------|-----------|----------------|
| 1 | **Logo del negocio** | `business_logo` existe Y la URL carga | La imagen del negocio |
| 2 | **Wordmark del sistema** | `business_logo` no existe, o la URL falla | Texto "R de Rico" con la fuente canónica |
| 3 | **Nunca vacío** | — | (el estado 2 siempre está disponible) |

### 2.2 Por qué el fallback es un WORDMARK y no un SVG remoto

**Decisión: el fallback es un wordmark tipográfico, no una imagen.**

Razón técnica: si el fallback fuera otra imagen remota, y la red se cae (que es exactamente el escenario que dispara el fallback), el fallback también fallaría. Un wordmark es **texto renderizado con una fuente que ya está en el bundle** — no puede fallar por red.

```
Estado 1: <img src={business_logo} onError={() => setEstado(2)} />
Estado 2: <span className="font-pos text-acento">R de Rico</span>
```

El SVG local empaquetado queda como **mejora opcional** (Fase D), no como fallback primario.

### 2.3 Puntos de aplicación del logo (lista cerrada)

| # | Punto | ¿Aplica? | Notas |
|---|-------|----------|-------|
| L1 | Encabezado de la pantalla raíz | ✅ Sí | Es el único punto obligatorio |
| L2 | Encabezado del ticket impreso | ✅ Sí | Solo si el tema lo permite |
| L3 | Pantalla de login | ✅ Sí | Refuerza identidad en el arranque |
| L4 | Modales | ❌ No | El modal no lleva logo; lleva título |
| L5 | Botones | ❌ No | Nunca |
| L6 | Fondos | ❌ No | Nunca |
| L7 | Favicon / manifest PWA | ✅ Sí | Ya existe el patrón en el ERP (`aplicarBrandingAlManifest`) |

**Regla:** el logo **no se propaga solo**. Si un punto no está en esta tabla, no lleva logo.

---

## 3. Punto 2 — COLORES CORPORATIVOS

### 3.1 Decisión

Los colores corporativos (2–3, máximo 3) aplican en **exactamente 5 puntos**. Ni uno más.

### 3.2 Los 5 puntos de aplicación (lista cerrada)

| # | Punto | Qué se pinta | Qué NO se pinta |
|---|-------|--------------|-----------------|
| C1 | **Encabezado** | La franja/barra superior de marca | El texto del encabezado (lo manda el tema) |
| C2 | **Botón primario** | El fondo del botón de cobrar/confirmar | El texto del botón (contraste calculado) |
| C3 | **Precio** | El número del precio | La etiqueta "Total" (esa es del tema) |
| C4 | **Categoría activa** | El chip de la categoría seleccionada | Los chips inactivos (esos son del tema) |
| C5 | **Ticket impreso** | El encabezado del recibo | El cuerpo del recibo (papel crema del tema) |

### 3.3 Lo que el color corporativo NUNCA toca

- Fondos de pantalla → **tema**
- Superficies / paneles → **tema**
- Bordes → **tema**
- Sombras → **tema**
- Estados hover / focus / disabled → **tema**
- Texto de cuerpo → **tema**
- Iconos → **tema** (salvo que el icono sea el logo)

### 3.4 Regla de contraste (no negociable)

Cada uno de los 5 puntos debe cumplir:

| Punto | Contraste mínimo | Por qué |
|-------|------------------|---------|
| C1 Encabezado | 3:1 contra el fondo | Es un bloque de UI |
| C2 Botón primario | 4.5:1 (texto del botón vs. fondo del botón) | Es texto sobre color |
| C3 Precio | 4.5:1 contra el panel | Es texto crítico |
| C4 Categoría activa | 3:1 contra el fondo | Es un estado de UI |
| C5 Ticket impreso | 4.5:1 en papel | Legibilidad impresa |

Si un color corporativo no cumple, el sistema **avisa** (ver §6) y ofrece el color canónico más cercano que sí cumpla.

---

## 4. Punto 3 — TIPOGRAFÍA

### 4.1 Decisión

La tipografía corporativa aplica en **exactamente 3 puntos**. El resto usa la fuente del tema.

### 4.2 Los 3 puntos de aplicación (lista cerrada)

| # | Punto | Qué usa la fuente corporativa |
|---|-------|-------------------------------|
| T1 | **Encabezado / marca** | El nombre del negocio y el wordmark |
| T2 | **Títulos de sección** | Los `<h1>`/`<h2>` de las pantallas |
| T3 | **Ticket impreso** | El encabezado del recibo |

### 4.3 Lo que la tipografía corporativa NUNCA toca

- Nombres de producto → **tema**
- Precios → **tema**
- Botones → **tema**
- Etiquetas y chips → **tema**
- Cuerpo de texto → **tema**

**Razón:** legibilidad y rendimiento. Cargar una fuente corporativa para todo el cuerpo de la UI multiplica el peso de la página y arriesga el FOUT (flash of unstyled text) en la pantalla de mostrador, que es la más crítica.

### 4.4 Cadena de fallback (3 niveles, nunca un hueco)

```
Fuente corporativa (si carga)
    ↓ falla
Fuente del tema (Inter / Roboto / Montserrat / Nunito / Poppins)
    ↓ falla
Fallback del sistema (system-ui, -apple-system, sans-serif)
```

Esto ya está implementado parcialmente en [`tailwind.config.js`](../../../NUEVO-POS/apps/pos/tailwind.config.js:32): `fontFamily: { pos: ['Inter', 'system-ui', 'sans-serif'] }`. La cadena de 3 niveles solo agrega el nivel corporativo al frente.

---

## 5. Punto 4 — TEMAS

### 5.1 Decisión

El tema **domina todo lo demás**. Es el "piso" visual del sistema.

### 5.2 Qué controla el tema (lista abierta — es el dueño del resto)

| Aspecto | El tema decide |
|---------|----------------|
| Fondos | Color, textura, degradado |
| Superficies / paneles | Color, borde, sombra |
| Bordes | Grosor, color, radio |
| Sombras | Suavidad, profundidad |
| Radios | 35px / 40px / 50px (canónicos) o los del tema |
| Densidad | Aireada vs. compacta |
| Estilos de modal | Fondo, borde, radio, sombra |
| Estilos de botón | Relleno vs. outline, radio, sombra |
| Estados | hover, focus, disabled, active |
| Tipografía de cuerpo | La fuente del tema |

### 5.3 Los 4 temas del catálogo (de la v2)

| Tema | Carácter | Fondo | Uso |
|------|----------|-------|-----|
| **Clásico** | Neutro, oscuro | `#0a0a0a` | Default (= POS actual) |
| **Madera** | Cálido, texturizado | `wood_bg.jpg` | Panadería / artesanal |
| **Minimal** | Claro, limpio | Claro | Oficina / mostrador moderno |
| **Neón** | Oscuro, vibrante | `#0a0a0a` + acentos | Nocturno / bar |

### 5.4 Relación tema ↔ identidad

El tema **cede** en los 5 puntos de color (§3.2) y los 3 de tipografía (§4.2). En todo lo demás, el tema manda. Si la identidad no define un color, el tema provee el suyo.

---

## 6. El aviso visible (coherente con la decisión #4 del usuario)

El usuario pidió "aviso" cuando la personalización no se puede aplicar. Este addendum define **2 casos de aviso**:

| Caso | Cuándo | Qué dice el aviso |
|------|--------|-------------------|
| **A — Vista General caída** | No se pudo leer `business_*` | "Usando apariencia por defecto. La personalización no está disponible." |
| **B — Contraste insuficiente** | Un color corporativo no cumple §3.4 | "El color institucional no cumple el contraste mínimo. Se usó el más cercano." |

El aviso es **visible pero no bloqueante**: no impide vender. Es un banner discreto, no un modal.

---

## 7. Resumen ejecutivo (una tabla)

| Capa | Dueño | Puntos de aplicación | Fallback |
|------|-------|----------------------|----------|
| **IDENTIDAD — Logo** | Vista General (`business_logo`) | L1 encabezado, L2 ticket, L3 login, L7 PWA | Wordmark "R de Rico" |
| **IDENTIDAD — Color** | Vista General (`business_colors`) | C1 encabezado, C2 botón primario, C3 precio, C4 categoría activa, C5 ticket | Color canónico más cercano |
| **IDENTIDAD — Tipografía** | Vista General (`business_font`) | T1 marca, T2 títulos, T3 ticket | Fuente del tema → sistema |
| **TEMA** | Vista General (`business_theme`) | Todo lo demás | Tema Clásico |
| **CANÓNICA** | Backend (código) | Piso de todo | — (es el piso) |

---

## 8. Impacto en las fases (de la v2)

| Fase | Qué agrega este addendum |
|------|--------------------------|
| **A** | Colapsar las 3 fuentes de paleta en 1. Sin cambios de puntos de aplicación todavía. |
| **B** | Las 4 claves (`business_logo`, `business_colors`, `business_theme`, `business_font`) con seed. |
| **C** | Vista General edita las 4 claves. Aquí se implementan los 5 puntos de color y los 3 de tipografía. |
| **D** | Texturas, SVG local del logo, temas adicionales. |

---

## 9. Estado

- **Creado:** 26 Sep 2026
- **Estado:** propuesta — pendiente de aprobación del usuario
- **Depende de:** [`PROPUESTA_BRANDING_TRANSVERSAL_DESDE_VISTA_GENERAL.md`](./PROPUESTA_BRANDING_TRANSVERSAL_DESDE_VISTA_GENERAL.md) y [`PROPUESTA_PALETA_CANONICA_V2.md`](./PROPUESTA_PALETA_CANONICA_V2.md)
- **Siguiente paso:** el usuario aprueba el addendum; luego llegan las fichas de Pinterest para fijar la paleta canónica
                                                                                                                                                                                                                                                                                                                    