# PROPUESTA — Branding transversal configurable desde Vista General

**Fecha:** 2026-09-26
**Estado:** PROPUESTA (no implementada)
**Regla dura vigente:** NO SE TOCA EL ERP INSTALADO Y CORRIENDO.
**Autor:** sesión de construcción del POS nuevo (etapa P3 cerrada)

---

## 1. La idea, en una frase

Que el módulo **Vista General** del ERP deje de configurar solo *datos* del negocio
(nombre, dirección, teléfono, zona horaria, moneda) y pase a configurar también su
**identidad visual** (logotipo, paleta de colores, texturas, tipografía), y que el
**POS nuevo respete esa identidad** en vez de tener su propio diseño hardcodeado.

---

## 2. Hallazgo clave: el 70% ya existe

Esta propuesta **no parte de cero**. El ERP ya tiene el patrón exacto, aplicado a medias.
Todo lo siguiente fue verificado leyendo el código, no supuesto:

| Pieza existente | Archivo | Qué demuestra |
|-----------------|---------|---------------|
| Canal genérico de settings | [`apps/api/modules/settings/router.py`](../../ERP-R-DE-RICO/apps/api/modules/settings/router.py:48) | `GET/PATCH /settings/{key}` ya es un sistema clave-valor genérico |
| Branding leído desde settings | [`apps/inventory/services/pwaRuntime.js`](../../ERP-R-DE-RICO/apps/inventory/services/pwaRuntime.js:156) | `derivarBrandingDesdeSettings()` ya lee `business_name` y `theme_color` |
| Manifest PWA reescrito en caliente | [`pwaRuntime.js`](../../ERP-R-DE-RICO/apps/inventory/services/pwaRuntime.js:175) | `aplicarBrandingAlManifest()` ya prueba el concepto "la UI se viste desde settings" |
| Vista General ya edita info del negocio | [`apps/ExperimentCenterUI.jsx`](../../ERP-R-DE-RICO/apps/ExperimentCenterUI.jsx:228) | `bizInfo` + `saveBizInfo()` ya persisten 6 claves con `PATCH /settings/{key}` |
| Precedente de "tema" por módulo | [`apps/api/modules/settings/service.py`](../../ERP-R-DE-RICO/apps/api/modules/settings/service.py:126) | `heladeria_display_precios_config` ya guarda `"theme":"LIGHT"` como JSON |
| Precedente de branding editable | [`service.py`](../../ERP-R-DE-RICO/apps/api/modules/settings/service.py:157) | `heladeria_branding` ya guarda nombre y eslogan editables |
| Paleta canónica declarada | [`apps/api/superficie/registry.py`](../../NUEVO-POS/apps/api/superficie/registry.py:58) | El POS nuevo ya trata el diseño como **dato declarado**, no como código disperso |

**Conclusión del hallazgo:** lo que se propone es **unificar tres esfuerzos dispersos**
(`theme_color`, `heladeria_branding`, `PALETA_CANONICA`) en **un solo sistema de branding
transversal**. No es una ocurrencia suelta: es la conclusión natural de un patrón ya iniciado.

---

## 3. Lo que falta

| Pieza | Estado hoy | Qué falta |
|-------|-----------|-----------|
| Logotipo | ❌ no existe `business_logo` | clave nueva + carga de archivo + fallback |
| Paleta de colores | ⚠️ solo `theme_color` (1 color) | paleta completa (acento, fondo, panel, texto, peligro) |
| Texturas / fondos | ❌ hardcodeadas | clave `business_texture` (asset) |
| Tipografía | ❌ hardcodeada | clave `business_font` |
| **Que el POS nuevo lo respete** | ❌ **no existe** | **esto es lo nuevo y lo importante** |

---

## 4. Por qué encaja con el POS nuevo

El POS nuevo ya tiene:

1. Una **paleta canónica declarada** en [`superficie/registry.py`](../../NUEVO-POS/apps/api/superficie/registry.py:58)
   (`PALETA_CANONICA`).
2. Una **regla dura R-03** que exige 3 modos explícitos.
3. Un principio ya documentado: **"Declarar, no convertir"**
   ([`DOCUMENTACION_VISTA_GENERAL.md`](../../ERP-R-DE-RICO/ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_VISTA_GENERAL.md:367)).

Eso significa que el POS nuevo puede leer el branding y aplicarlo como **variables CSS**
(`--acento`, `--fondo`, `--panel`, `--texto`, `--peligro`), donde:

- La **paleta canónica** pasa de ser una constante a ser el **valor por defecto**.
- El **ERP y el POS nuevo comparten la misma fuente de verdad** (la tabla `system_settings`).

---

## 5. Los 3 riesgos reales

### Riesgo 1 — El ERP está corriendo y vendiendo

Tocar [`ExperimentCenterUI.jsx`](../../ERP-R-DE-RICO/apps/ExperimentCenterUI.jsx:228) o
[`settings/service.py`](../../ERP-R-DE-RICO/apps/api/modules/settings/service.py:26) es
**tocar el ERP**, lo que viola la regla dura.

**Mitigación:** construir primero el lado del POS nuevo (aislado). La edición en Vista
General se deja para una fase posterior **con ventana de mantenimiento acordada**.

### Riesgo 2 — El POS nuevo no ve la tabla `system_settings` del ERP

Hoy el POS nuevo tiene **su propia base de datos** (`nuevo_pos`). No ve la del ERP.

**Dos caminos:**

- **(A) Contrato de solo lectura.** El POS nuevo pide el branding al ERP vía un endpoint.
  Encaja con la arquitectura de contratos de F2. Limpio, pero acopla.
- **(B) Replicar las claves de branding** en la BD del POS nuevo y sincronizarlas.
  Más aislado, pero duplica.

**Recomendación:** **(A) ahora** (contrato de lectura, durante la prueba en paralelo) y
**(B) en el corte** (cuando el POS nuevo sea el dueño de sus datos).

### Riesgo 3 — "Paleta de texturas" es lo más caro

Un color es un dato; una textura es un **asset** (imagen). Implica carga de archivo,
almacenamiento, validación de peso y un **fallback obligatorio**.

**Mitigación:** dejar texturas y tipografía para el final. Colores y logo dan el 90% del
impacto visual con el 20% del trabajo.

---

## 6. Contratos propuestos (borrador, sin implementar)

### 6.1 Claves nuevas en `system_settings`

| Clave | Tipo | Valor por defecto | Descripción |
|-------|------|-------------------|-------------|
| `business_logo` | text (URL/ruta) | `""` | Ruta del logotipo. Vacío = usar el logo estático actual. |
| `business_palette` | json | `{}` | Paleta completa. Vacío = usar `PALETA_CANONICA`. |
| `business_texture` | text (URL/ruta) | `""` | Fondo/textura. Vacío = fondo sólido canónico. |
| `business_font` | text | `""` | Familia tipográfica. Vacío = tipografía canónica. |

### 6.2 Forma del JSON de paleta

```json
{
  "acento": "#c1d72e",
  "fondo": "#0a0a0a",
  "panel": "#1a1a1a",
  "texto": "#fdfbf7",
  "peligro": "#ef4444"
}
```

**Regla de fusión:** el JSON del negocio se **fusiona sobre** `PALETA_CANONICA`. Las claves
ausentes o inválidas caen al valor canónico. **Nunca** se aplica una paleta incompleta.

### 6.3 Contrato de lectura (camino A)

```
GET /branding
-> 200 {
     "logo": "…" | null,
     "palette": { … } | null,
     "texture": "…" | null,
     "font": "…" | null,
     "source": "settings" | "default"
   }
```

- **Garantía:** si el ERP no responde, el POS nuevo **sirve igual** con `PALETA_CANONICA`
  (mismo principio de degradación elegante que ya usa
  [`pwaRuntime.js`](../../ERP-R-DE-RICO/apps/inventory/services/pwaRuntime.js:145)).
- **Nunca** devuelve un error que rompa el arranque del POS.

---

## 7. Plan por fases

### Fase A — POS nuevo lee branding y lo aplica (AISLADA, cero riesgo para el ERP)

- El POS nuevo consume `GET /branding` (o su equivalente local) y lo aplica como
  variables CSS.
- Si no hay datos, usa `PALETA_CANONICA`.
- **No se toca el ERP.** Se puede probar en vivo en el puerto 5100.

### Fase B — Claves de branding en la BD del POS nuevo (AISLADA)

- Agregar `business_logo` y `business_palette` a la BD `nuevo_pos`, con su seed.
- Permite probar el efecto completo sin depender del ERP.

### Fase C — Edición en Vista General del ERP (CON VENTANA ACORDADA)

- Añadir los campos de logo y paleta al modal de Vista General.
- Reutilizar el permiso `editar_info_negocio` que **ya existe**
  ([`ExperimentCenterUI.jsx`](../../ERP-R-DE-RICO/apps/ExperimentCenterUI.jsx:560)).
- **Requiere confirmación explícita del usuario antes de tocar el ERP.**

### Fase D — Texturas y tipografía (OPCIONAL)

- Solo si las fases A–C demuestran valor.
- Es la fase más cara (assets, almacenamiento, validación).

---

## 8. Veredicto

**Viable: sí, totalmente.** Y no es una idea aislada: es la **unificación** de un patrón que
ya existe en tres lugares del ERP y del POS nuevo.

**Único cambio al planteamiento original:** el **orden**. No empezar por el modal del ERP
(que está corriendo y vendiendo), sino por el POS nuevo (que está aislado y es donde se
quiere que el branding se respete). Así el ERP no se toca hasta que el usuario decida abrir
la ventana de mantenimiento.

---

## 9. Estado de esta propuesta

- **Escrita, no implementada.** Ninguna línea de código fue modificada.
- El ERP permanece intacto: HEAD `b0bc297`, árbol limpio, `rderico-*` Up.
- El POS nuevo permanece en su commit `25e5de0`.
- La decisión de avanzar a la Fase A corresponde al usuario.
