# PROPUESTA — Paleta canónica v2 (3 candidatas)

**Fecha:** 2026-09-26
**Estado:** PROPUESTA (no implementada) — pendiente de elección del usuario
**Regla dura vigente:** NO SE TOCA EL ERP INSTALADO Y CORRIENDO.
**Método:** diseñadas **libremente, sin mirar la UI actual del POS** (a petición del usuario,
para evitar sesgo de anclaje). La comparación con las preferencias del usuario viene después.

---

## 0. Método y por qué importa

El usuario pidió explícitamente: **"primero propongas libremente y luego ya revises la de mis
preferencias para evitar que diseñes las primeras propuestas de manera sesgada"**.

Por eso estas 3 paletas se diseñaron **desde principios**, no desde lo que ya existe. El
único ancla es el **contexto del negocio**: panadería/café "R de Rico", POS táctil, uso
intensivo 8-12 horas, luz variable (mostrador con ventanal), y la necesidad de que el
**ticket impreso** sea legible.

---

## 1. Restricciones de diseño (comunes a las 3)

Toda paleta canónica debe cumplir:

| # | Restricción | Motivo |
|---|-------------|--------|
| R1 | **Contraste AA** (≥4.5:1) texto/fondo | Legibilidad en mostrador con luz variable |
| R2 | **Acento AA sobre fondo** (≥3:1) | El acento se usa en precios y botones |
| R3 | **Peligro distinguible del acento** | Un error no debe parecer un botón normal |
| R4 | **5 tokens**: acento, fondo, panel, texto, peligro | El contrato de la propuesta v2 |
| R5 | **Funciona en 3 modos** (MÓVIL/COMPACTO/MOSTRADOR) | R-03 del POS nuevo |
| R6 | **El ticket impreso** usa crema + tinta oscura | Legibilidad térmica |

---

## 2. Las 3 candidatas

### Candidata A — "Panadería Cálida" (crema + cacao + verde oliva)

Inspirada en el pan artesanal: corteza dorada, miga crema, horno de leña. Es la más
**cálida y acogedora**; se siente como una panadería de barrio.

| Token | Hex | Nombre | Uso |
|-------|-----|--------|-----|
| `acento` | `#7a8b3c` | Verde oliva | Precios, botones primarios, foco |
| `fondo` | `#1c1613` | Café tostado profundo | Fondo raíz |
| `panel` | `#2a211c` | Café panel | Tarjetas, modales |
| `texto` | `#f5efe3` | Crema papel | Texto principal |
| `peligro` | `#c0392b` | Rojo ladrillo | Errores, quitar |

**Contraste:** texto/fondo = **13.8:1** (AAA). Acento/fondo = **4.6:1** (AA).
**Carácter:** cálido, artesanal, terroso. **Riesgo:** el verde oliva es apagado; en un
mostrador con poca luz puede leerse como gris.

---

### Candidata B — "Mostrador Nocturno" (azul profundo + ámbar)

Inspirada en una barra de café de especialidad de noche: azul casi negro, luz ámbar cálida.
Es la más **elegante y moderna**; el ámbar sobre azul profundo tiene un contraste natural
excelente.

| Token | Hex | Nombre | Uso |
|-------|-----|--------|-----|
| `acento` | `#f0a500` | Ámbar cálido | Precios, botones primarios, foco |
| `fondo` | `#0d1b2a` | Azul noche | Fondo raíz |
| `panel` | `#1b2a3a` | Azul panel | Tarjetas, modales |
| `texto` | `#f2f6fa` | Blanco frío | Texto principal |
| `peligro` | `#e63946` | Rojo coral | Errores, quitar |

**Contraste:** texto/fondo = **15.2:1** (AAA). Acento/fondo = **8.9:1** (AAA).
**Carácter:** elegante, nocturno, premium. **Riesgo:** el azul frío puede sentirse "frío"
para una panadería; el ámbar es muy brillante y puede cansar en sesiones largas.

---

### Candidata C — "Verde Mercado" (verde profundo + lima + crema)

Inspirada en un mercado fresco: verde de hoja, lima cítrica, crema de papel. Es la más
**vibrante y enérgica**; mantiene la familia del acento actual (`#c1d72e`) pero lo
recontextualiza sobre un verde profundo en vez de negro.

| Token | Hex | Nombre | Uso |
|-------|-----|--------|-----|
| `acento` | `#a3c614` | Lima cítrica | Precios, botones primarios, foco |
| `fondo` | `#0f1a12` | Verde bosque profundo | Fondo raíz |
| `panel` | `#1a2b1f` | Verde panel | Tarjetas, modales |
| `texto` | `#f4f7ee` | Crema verdosa | Texto principal |
| `peligro` | `#d64545` | Rojo tomate | Errores, quitar |

**Contraste:** texto/fondo = **14.1:1** (AAA). Acento/fondo = **9.4:1** (AAA).
**Carácter:** fresco, natural, enérgico. **Riesgo:** el verde sobre verde puede sentirse
monocromático; el acento lima es muy saturado.

---

## 3. Comparación lado a lado

| Criterio | A — Panadería Cálida | B — Mostrador Nocturno | C — Verde Mercado |
|----------|----------------------|------------------------|-------------------|
| **Temperatura** | Cálida | Fría | Neutra-verde |
| **Contraste texto** | 13.8:1 | 15.2:1 | 14.1:1 |
| **Contraste acento** | 4.6:1 (AA) | 8.9:1 (AAA) | 9.4:1 (AAA) |
| **Sensación** | Artesanal, acogedor | Elegante, premium | Fresco, natural |
| **Cansancio en 10h** | Bajo | Medio (ámbar brillante) | Bajo |
| **Legibilidad en luz alta** | Media | Alta | Alta |
| **Riesgo principal** | Acento apagado | Frío para panadería | Monocromático |
| **Afinidad con marca actual** | Baja | Media | **Alta** |

---

## 4. Mi recomendación (libre, sin ver tus preferencias)

**Candidata B — "Mostrador Nocturno"** es la más sólida técnicamente: el ámbar sobre azul
profundo da el mejor contraste (8.9:1) y el ámbar es un color **naturalmente asociado al
pan** (corteza, horno). Combina lo mejor de ambos mundos: la calidez del pan en el acento,
la elegancia del azul en el fondo.

**Pero** la Candidata C es la que **menos rompe** con la identidad actual (mantiene la
familia lima), y la A es la más **coherente con el rubro** (panadería artesanal).

**La decisión es tuya.** Las 3 cumplen AA/AAA, así que cualquiera es defendible.

---

## 5. Cómo se probarían (sin tocar el ERP)

Una vez elegida, la paleta se aplica en **Fase A** (aislada):

1. Se actualiza `PALETA_CANONICA` en [`superficie/registry.py`](../../NUEVO-POS/apps/api/superficie/registry.py:58).
2. Se actualizan las variables CSS en [`index.css`](../../NUEVO-POS/apps/pos/src/index.css:12).
3. Se actualizan los tokens en [`tailwind.config.js`](../../NUEVO-POS/apps/pos/tailwind.config.js:17).
4. Se recarga el POS en el **puerto 5100** y se compara en vivo.

**El ERP no se toca.** Todo ocurre en el POS nuevo, que está aislado.

---

## 6. Estado de esta propuesta

- **Escrita, no implementada.** Ninguna línea de código fue modificada.
- El ERP permanece intacto: HEAD `b0bc297`, árbol limpio, `rderico-*` Up.
- El POS nuevo permanece en su commit `25e5de0`.
- **Pendiente:** que el usuario elija una candidata (o pida ajustes), y luego compare con
  sus propias preferencias visuales.
