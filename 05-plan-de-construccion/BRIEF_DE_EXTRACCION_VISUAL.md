# BRIEF DE EXTRACCIÓN VISUAL — Para diseñar el default del POS

> **Creado:** 28 Sep 2026.
> **Propósito:** guiar la extracción de los **11 valores** del default visual del POS nuevo a partir de una **imagen de referencia** (Pinterest, Dribbble, captura, etc.).
> **Se usa en:** **Fase D** de [`PLAN_DE_IMPLEMENTACION_UI_POR_MODULO.md`](./PLAN_DE_IMPLEMENTACION_UI_POR_MODULO.md:407).
> **Alcance:** SOLO el POS nuevo (`../NUEVO-POS/`). No toca el ERP.

---

## 1. Veredicto: ¿sirve la idea?

**Sí, sirve — y es la mejor forma de comunicar "gusto".** Una imagen transmite en 2 segundos lo que un párrafo no logra en 2 páginas. Pero hay **3 correcciones** que hay que entender antes de usarla:

| # | Corrección | Por qué |
|---|---|---|
| 1 | **La imagen da DIRECCIÓN, no VALORES exactos.** | Una IA que mira un JPEG **estima** los colores; no los lee. El JPEG comprime, la pantalla cambia el brillo y el perfil de color altera los tonos. Los hex que devuelva son **aproximados**. |
| 2 | **Hay que pedir una extracción ESTRUCTURADA.** | Si le pides "descríbeme el estilo", te da poesía. Si le pides "dame estos 11 valores en una tabla", te da algo usable. |
| 3 | **Lo extraído se CALIBRA y se VALIDA.** | Los valores que traigas pasan por la **Fase D** (contraste WCAG AA) y por el **piso técnico** (los 6 tokens deben existir). No se copian a ciegas. |

**En una frase:** la imagen es el **norte**; los valores exactos los **calibramos** aquí. La IA te da el borrador; nosotros lo hacemos correcto.

---

## 2. Los 11 valores que necesito (obligatorios)

Estos son **todos** los valores que definen el look del POS. Si controlas estos 11, controlas el 100%.

### A. Los 6 colores

| # | Token | Qué es | Ejemplo actual (piso técnico) |
|---|---|---|---|
| 1 | `acento` | El color "de marca" que resalta (botones, totales, activo). | `#c1d72e` (verde-lima) |
| 2 | `fondo-profundo` | El fondo principal (el más oscuro). | `#0a0a0a` |
| 3 | `fondo-profundo-alt` | Variante del fondo (franjas, header, separadores). | `#080808` |
| 4 | `fondo-panel` | Superficie de tarjetas y paneles. | `#1a1a1a` |
| 5 | `crema-ticket` | El color claro del texto y del ticket. | `#fdfbf7` |
| 6 | `peligro` | El rojo de alerta / cancelar / error. | `#ef4444` |

### B. Los 3 radios de borde

| # | Token | Qué es | Ejemplo actual |
|---|---|---|---|
| 7 | `radio-pequeno` | Esquinas de elementos pequeños (chips, inputs). | `35px` |
| 8 | `radio-medio` | Esquinas de tarjetas y botones. | `40px` |
| 9 | `radio-grande` | Esquinas de paneles grandes. | `50px` |

### C. Las 2 tipografías

| # | Token | Qué es | Ejemplo actual |
|---|---|---|---|
| 10 | `fuente-ui` | La tipografía de la interfaz (botones, menús, precios). | `Inter` |
| 11 | `fuente-ticket` | La tipografía del ticket (monoespaciada, para imprimir). | `ui-monospace` |

---

## 3. Extras que ayudan (opcionales, pero valen oro)

No son obligatorios, pero si la IA los saca, el diseño queda mucho mejor:

| Extra | Qué pedir |
|---|---|
| **Sombras** | ¿Las tarjetas tienen sombra? ¿Suave, dura o ninguna? |
| **Bordes** | ¿Hay bordes visibles? ¿De qué grosor? ¿De qué color? |
| **Densidad** | ¿El diseño es compacto (mucha info) o aireado (mucho espacio)? |
| **Iconos** | ¿Son de línea, rellenos, de color? |
| **Imágenes de producto** | ¿Fondo blanco, transparente, con esquinas redondeadas? |
| **Sensación (3 adjetivos)** | Ej: "moderno, cálido, minimalista". |

---

## 4. Lo que NO sirve pedirle a una IA mirando una imagen

| No pidas | Por qué |
|---|---|
| **Hex exactos** | Los **estima** desde los píxeles. Sirven como punto de partida, no como verdad. |
| **Nombres exactos de fuentes** | Adivina. "Parece Poppins" no es "es Poppins". |
| **Medidas exactas en px** | No puede medir con precisión desde una foto. |
| **"Copia este diseño tal cual"** | Riesgo legal (diseños con derechos) y riesgo técnico (no encaja con tu POS). |

**Regla:** la IA da **dirección + aproximación**. La exactitud la damos aquí con la calibración y el contraste.

---

## 5. Prompt listo para pegar en Gemini (Antigravity)

Copia esto y adjunta la imagen:

```
Actúa como un diseñador de UI senior especializado en sistemas de diseño.

Te adjunto una imagen de referencia de una interfaz de Punto de Venta (POS).
NO quiero una descripción poética. Quiero una EXTRACCIÓN ESTRUCTURADA
de los valores que necesito para replicar su estética en un POS oscuro.

Devuélveme EXACTAMENTE esto, en tablas:

1. LOS 6 COLORES (en HEX aproximado, y dime tu nivel de confianza 0-100%):
   - acento (el color de marca que resalta)
   - fondo-profundo (el fondo principal)
   - fondo-profundo-alt (variante del fondo para franjas/header)
   - fondo-panel (superficie de tarjetas)
   - crema-ticket (el color claro del texto)
   - peligro (el rojo de alerta)

2. LOS 3 RADIOS DE BORDE (en px aproximados):
   - radio-pequeno, radio-medio, radio-grande

3. LAS 2 TIPOGRAFÍAS:
   - fuente-ui (la de la interfaz)
   - fuente-ticket (la monoespaciada del ticket)
   Para cada una: nombre probable + familia genérica (sans-serif, mono, etc.)

4. EXTRAS:
   - Sombras: ¿suaves, duras o ninguna?
   - Bordes: ¿visibles? ¿grosor?
   - Densidad: ¿compacta o aireada?
   - Iconos: ¿línea, relleno o color?
   - 3 adjetivos que describan la sensación.

5. ADVERTENCIA HONESTA:
   Dime explícitamente qué valores NO puedes saber con certeza desde una
   imagen y cuáles son solo una aproximación.

Formato: tablas markdown. Sin relleno. Sin introducción.
```

---

## 6. Qué hago yo con lo que traigas

Cuando me pases la tabla que devuelva Gemini:

1. **Normalizo** los valores al formato del POS (canales RGB, no hex, para que Tailwind funcione).
2. **Verifico** que los 6 tokens existan y sean válidos.
3. **Calculo el contraste WCAG AA** (texto/fondo ≥ 4.5:1, acento/fondo ≥ 3:1).
4. **Ajusto** lo que no pase el contraste (te aviso qué cambié y por qué).
5. **Escribo** el resultado en la tabla de la **Fase D** (tareas D.3, D.4, D.5).
6. **Dejo listo** el `default.js` para la Fase 2.

---

## 7. Advertencia legal (breve)

Pinterest es un tablero de **referencias**, no una tienda de diseños. Usa la imagen como **inspiración de dirección** (paleta, mood, densidad), **no como copia 1:1**. Copiar un diseño ajeno tal cual puede tener problemas de derechos. Inspirarse en una paleta y un estilo es práctica normal de diseño.

---

## 8. Resumen

- **¿Sirve la idea?** Sí. Es la mejor forma de comunicar gusto.
- **¿Qué corrijo?** La imagen da dirección; los valores exactos se calibran aquí.
- **¿Qué pido?** Los 11 valores en una tabla estructurada (prompt de §5).
- **¿Qué hago con eso?** Normalizo, valido contraste, ajusto y lo dejo listo para la Fase D.
- **¿Toca el ERP?** No. Solo el POS nuevo.
