# Plantilla de extracción — Referencias de UI (Pinterest)

> **Para qué sirve:** convertir una imagen de referencia (Pinterest, Dribbble, Behance, screenshot propio) en una **ficha de tokens** que el ingeniero pueda traducir a CSS/Tailwind sin ambigüedad.
>
> **Regla de oro:** una ficha por referencia. No mezclar dos imágenes en una ficha.
>
> **Contexto de disciplina:** el modelo de IA que implementa **no puede ver imágenes**. Todo lo que no quede escrito en esta ficha, no existe para el implementador. Si algo es "obvio" en la imagen, escríbelo igual.

---

## 0. Cómo llenar esta plantilla

1. Copia el bloque **FICHA** de abajo, una vez por cada referencia (3–5 en total).
2. Rellena cada campo. Si un campo no aplica, escribe `N/A` (no lo dejes vacío).
3. Para los colores, si no tienes un color picker a mano, usa una app de "eyedropper" (o el selector de color de Windows: `Win + Shift + C` en PowerToys, o cualquier extensión del navegador). **Aproximado está bien** — yo ajusto el contraste después.
4. Al terminar, pásame el archivo completo (o el texto) y yo lo traduzco a tokens.

### Criterios de selección de referencias (importante)

- **Mismo "mundo":** retail, punto de venta, food service, kiosco táctil, autoservicio. Evita mezclar banca + videojuego + e-commerce.
- **Que se parezca al problema real:** pantalla de venta con grilla de productos + ticket lateral + botón de cobro. Si la referencia es una landing page, sirve menos.
- **3–5 referencias máximo.** Más cantidad no mejora la señal; la diluye.
- **Anota la URL** de cada una (para poder volver a ella).

---

## FICHA (copiar una vez por referencia)

```
=== REFERENCIA #__ ===

[IDENTIFICACIÓN]
- URL / fuente:
- Qué es (app, POS, kiosco, web, concepto):
- Por qué la elegí (1 frase):

[CAPA 1 — FONDO]
- Tipo: plano | textura | degradado | imagen
- Color(es) aproximado(s) en hex: #______  (si es degradado, de #______ a #______)
- ¿Es claro u oscuro?
- ¿Hay textura visible? (madera, papel, ruido, ninguna):

[CAPA 2 — SUPERFICIE / PANEL]
- ¿El panel se separa del fondo por...? color | borde | sombra | ninguno
- Color del panel en hex: #______
- ¿Tiene borde? grosor y color: ______
- ¿Tiene sombra? suave | marcada | ninguna
- Radio de esquina del panel (aprox. en px o "muy redondeado / poco redondeado"):

[CAPA 3 — ACENTO]
- ¿Cuántos colores de acento usa? (1, 2, 3...):
- Acento principal en hex: #______
- Acento secundario en hex: #______ (si aplica)
- ¿Dónde aplica el acento? (marca con X)
    [ ] botón primario (cobrar / confirmar)
    [ ] precio
    [ ] categoría activa
    [ ] encabezado / logo
    [ ] iconos
    [ ] otro: ______
- Color de "peligro" / cancelar en hex: #______ (si se ve)

[CAPA 4 — TIPOGRAFÍA]
- Estilo: geométrica | humanista | serif | monoespaciada | no identificable
- ¿Mayúsculas o minúsculas en los títulos?
- ¿El precio es más pesado que el nombre del producto? sí | no
- ¿Cuántos tamaños distintos se ven? (aprox.):
- Nombre de la fuente si la reconoces (o "no sé"):

[CAPA 5 — FORMA]
- Radios: muy redondeados (burbuja) | medios | casi rectos (técnico)
- Botones: rellenos | con borde (outline) | mixtos
- ¿Las tarjetas de producto tienen imagen grande o pequeña?
- ¿Hay bordes visibles en las tarjetas? sí | no

[CAPA 6 — DENSIDAD]
- ¿Aireada (pocas cosas, mucho espacio) o densa (muchas cosas, poco espacio)?
- ¿Cuántos productos caben "a la vista" aproximadamente?
- ¿El ticket/carrito es lateral, inferior, o modal?
- ¿Se ve un teclado numérico en pantalla? sí | no

[NOTAS LIBRES]
- Lo que más me gusta de esta referencia:
- Lo que NO quiero copiar:
- ¿Se parece a alguna de las 3 candidatas (A Panadería Cálida / B Mostrador Nocturno / C Verde Mercado)? ¿A cuál y por qué?
```

---

## 1. Qué haré yo con estas fichas

Cuando me pases las fichas, produciré:

1. **Una tabla comparativa** de las 3–5 referencias contra las 3 candidatas ya escritas y contra la canónica actual (`#c1d72e` / `#0a0a0a` / `#1a1a1a` / `#fdfbf7` / `#ef4444`).
2. **Una paleta sintetizada** (o la confirmación de una candidata existente) con los 5 tokens canónicos + radios + tipografía.
3. **El cálculo de contraste WCAG** de cada par texto/fondo (mínimo 4.5:1 para texto normal, 3:1 para texto grande y elementos de UI).
4. **La traducción a las 3 fuentes de verdad** que hoy están duplicadas:
   - `apps/api/superficie/registry.py` → `PALETA_CANONICA`
   - `apps/pos/src/index.css` → `:root`
   - `apps/pos/tailwind.config.js` → `theme.extend.colors`
5. **La propuesta de Fase A** (colapsar las 3 fuentes en 1, sin tocar el ERP).

---

## 2. Restricciones técnicas que la paleta final DEBE cumplir

Estas vienen de las reglas duras ya aprobadas (Doc 6 §3 y Doc 7 §2.1). No son negociables:

| # | Restricción | Por qué |
|---|-------------|---------|
| R1 | Contraste texto/fondo ≥ 4.5:1 | Legibilidad en pantalla de mostrador con luz de tienda |
| R2 | Contraste acento/fondo ≥ 3:1 | El acento debe verse como elemento de UI, no como decoración |
| R3 | El acento debe funcionar sobre fondo oscuro Y sobre panel | El POS es oscuro por defecto |
| R4 | El color de peligro debe ser distinguible del acento | Cancelar no puede confundirse con cobrar |
| R5 | Máximo 3 colores institucionales configurables | Decisión del usuario (evitar el "collage") |
| R6 | La tipografía debe tener fallback del sistema | Si la fuente no carga, la UI no se rompe |

---

## 3. Estado

- **Creado:** 26 Sep 2026
- **Propósito:** insumo para la decisión de paleta canónica (Fase A del branding transversal)
- **Depende de:** [`PROPUESTA_PALETA_CANONICA_V2.md`](./PROPUESTA_PALETA_CANONICA_V2.md) (3 candidatas) y [`PROPUESTA_BRANDING_TRANSVERSAL_DESDE_VISTA_GENERAL.md`](./PROPUESTA_BRANDING_TRANSVERSAL_DESDE_VISTA_GENERAL.md) (modelo de 3 capas)
- **Siguiente paso:** el usuario llena 3–5 fichas y las entrega
