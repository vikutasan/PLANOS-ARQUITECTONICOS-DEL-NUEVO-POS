# PROPUESTA V4 — Apariencia por Módulo (Motor Compartido + Temas por Módulo + Identidad Encima)

> **ANEXO DE ARQUITECTURA (corrección 28 Sep 2026).** Este documento se conserva como el **"por qué"** de la arquitectura de temas. El **"cómo"** (el plan de ejecución) vive en [`PLAN_DE_IMPLEMENTACION_UI_POR_MODULO.md`](./PLAN_DE_IMPLEMENTACION_UI_POR_MODULO.md), que resuelve las 3 decisiones que aquí quedaron abiertas (mecanismo, vocabulario y default) y corrige los defectos detectados en la revisión autocrítica contra el código real del POS.
>
> **No ejecutar nada de este documento sin leer primero el plan de implementación.** Los puntos de este anexo que contradigan al plan, se resuelven a favor del plan.

**Estado:** propuesta para revisión del dueño. No se ha escrito una sola línea de código de esta propuesta.
**Alcance:** SOLO el POS nuevo que estamos construyendo (`../NUEVO-POS/`).
**Regla dura vigente:** NO SE TOCA EL ERP INSTALADO Y CORRIENDO. NO SE TOCA EL POS QUE CORRE.
**Naturaleza del documento (corrección 28 Sep 2026):** los 7 puntos de aplicación de la identidad que se mencionan aquí son **referencia de planificación** para cuando esos módulos del ERP se reconstruyan. **Hoy no se toca ninguno.** El trabajo actual es solo **explorar cómo el POS nuevo interactuará** con esos módulos que aún no se construyen, y **dejar todo preparado** para cuando existan.
**Reemplaza a:** `PROPUESTA_APARIENCIA_POR_MODULO_V3.md` (en lo que respecta a "3 temas por módulo").
**Se apoya en:** `PROPUESTA_BRANDING_TRANSVERSAL_DESDE_VISTA_GENERAL.md` (v2) y `ADDENDUM_PUNTOS_DE_APLICACION_DEL_BRANDING.md`.

---

## 0. Qué cambia respecto a la v2 y al addendum

| Punto | v2 + Addendum | V4 (esta propuesta) |
|---|---|---|
| ¿Quién define los temas? | Un catálogo global único (4 temas para todo el ERP). | **Cada módulo define sus propios temas.** El catálogo global solo guarda los temas *compartidos* (los que varios módulos quieren reusar). |
| ¿Cuántos temas ve el usuario? | 4 temas globales. | **De 1 a 3 temas por módulo** (1 predeterminado obligatorio + hasta 2 opcionales). |
| ¿Dónde vive el motor? | Implícito, sin lugar declarado. | **`packages/theme-engine/`** — un solo motor compartido, fuera de todo módulo. |
| ¿Dónde viven los temas? | En el catálogo global. | **Dentro de la carpeta de cada módulo** (`apps/<modulo>/theme/`). |
| ¿La identidad del negocio? | Capa 3 encima del tema. | **Igual** — capa encima, configurada en Vista General, guardada en la BD. |
| ¿Elección por módulo o global? | Global. | **Por módulo, con predeterminado.** El usuario puede cambiar el tema de un módulo sin afectar a los demás. |

Lo que **NO** cambia: el modelo de 3 capas (`UI final = CANÓNICA ⊕ TEMA ⊕ IDENTIDAD`), los 3 estados del logo, y los 2 avisos visibles.

**Corrección importante (27 Sep 2026):** los puntos de aplicación de la identidad **ya no se definen desde la óptica del POS**. La identidad aplica en **7 lugares muy específicos del ERP** (ver [`ADDENDUM_PUNTOS_DE_APLICACION_DEL_BRANDING.md`](./ADDENDUM_PUNTOS_DE_APLICACION_DEL_BRANDING.md)), y **6 de esos 7 no tocan el nuevo POS**. Los colores corporativos son **de 1 a 3**, no forzosamente 3.

**Corrección v4 (28 Sep 2026):** los 7 puntos son **referencia de planificación**, no trabajo actual. **Hoy no se toca el ERP ni el POS que corre.** Solo se explora cómo el POS nuevo interactuará con esos módulos que aún no se construyen, dejando todo preparado para cuando existan. La Fase C **no toca el ERP** (ver §10).

---

## 1. La decisión de arquitectura, en palabras simples

### 1.1 La imprenta y los libros

Imagina una imprenta. La imprenta tiene **una sola máquina** que sabe imprimir. Esa máquina no pertenece a ningún libro: es de la imprenta.

Cada **libro** (cada módulo del ERP) trae su propio diseño: su portada, sus colores, su tipografía. El libro le dice a la máquina "imprímeme con mi diseño". La máquina obedece.

- **La máquina** = el motor de temas. Vive en `packages/theme-engine/`. Es **una sola** para todo el sistema.
- **El diseño de cada libro** = los 3 temas de cada módulo. Viven **dentro de la carpeta del módulo**.
- **El dueño del negocio** = encima de todo. Pone su logo, sus colores y su tipografía. La máquina los respeta por encima del diseño del libro.

### 1.2 Por qué el motor NO puede vivir en Vista General

Esta es la corrección más importante de esta propuesta.

Si el motor viviera dentro del módulo **Vista General**, pasaría esto:

1. Vista General se cae (un error, un deploy a medias, lo que sea).
2. **Todos** los módulos se quedan sin motor.
3. **Todos** los módulos se quedan sin tema, sin colores, sin nada.
4. Un fallo en un módulo tumba la apariencia de los otros 11.

Eso es exactamente lo que queremos evitar. El motor es **infraestructura compartida**, no una función de Vista General.

**Vista General es un módulo más.** Su trabajo es *configurar* la identidad del negocio (guardar el logo, los colores, la tipografía en la BD). Pero **no es dueño del motor**. Si Vista General se cae, la identidad no se puede *cambiar*, pero la apariencia que ya estaba guardada **sigue funcionando** en todos los módulos.

### 1.3 Dónde vive cada pieza

```
NUEVO-POS/
├── packages/
│   └── theme-engine/              ← EL MOTOR (uno solo, compartido)
│       ├── index.js               ← aplicarTema, resolverTema, validarContraste
│       ├── contraste.js           ← cálculo de contraste WCAG
│       └── temas-compartidos/     ← temas que varios módulos reusan
│           ├── minimal.js
│           └── clasico.js
│
└── apps/
    ├── pos/
    │   └── theme/                 ← LOS TEMAS DEL MÓDULO POS
    │       ├── index.js           ← TEMA_DEL_MODULO (el contrato)
    │       ├── calido.js          ← predeterminado
    │       ├── nocturno.js        ← opcional 1
    │       └── minimal.js         ← opcional 2 (reusa el compartido)
    │
    ├── estadisticas/
    │   └── theme/                 ← LOS TEMAS DEL MÓDULO ESTADÍSTICAS
    │       ├── index.js
    │       ├── tablero.js         ← predeterminado
    │       ├── compacto.js        ← opcional 1
    │       └── clasico.js         ← opcional 2 (reusa el compartido)
    │
    └── vista-general/
        └── theme/                 ← LOS TEMAS DE VISTA GENERAL
            ├── index.js
            ├── kiosco.js          ← predeterminado
            ├── nocturno.js        ← opcional 1
            └── minimal.js         ← opcional 2
```

**Regla de oro:** un módulo **nunca** importa el motor de otro módulo. Todos importan `packages/theme-engine/`. Si un módulo necesita un tema que otro ya hizo, lo toma de `temas-compartidos/`, no de la carpeta del otro módulo.

---

## 2. El contrato del módulo

Cada módulo declara sus temas con una sola estructura. Es el "formulario" que el módulo llena para decirle al motor qué tiene.

```js
// apps/pos/theme/index.js
export const TEMA_DEL_MODULO = {
  modulo: 'pos',
  default: 'calido',
  permitidos: ['calido', 'nocturno', 'minimal'],
  temas: {
    calido:   () => import('./calido.js'),
    nocturno: () => import('./nocturno.js'),
    minimal:  () => import('../../../../packages/theme-engine/temas-compartidos/minimal.js'),
  },
};
```

### 2.1 Qué significa cada campo

| Campo | Qué es | Regla |
|---|---|---|
| `modulo` | El nombre del módulo. | Debe coincidir con el nombre de la carpeta. |
| `default` | El tema con el que nace el módulo. | **Obligatorio.** Debe estar en `permitidos`. |
| `permitidos` | Los temas que el usuario puede elegir. | **De 1 a 3.** El default + hasta 2 opcionales. |
| `temas` | El mapa de nombre → archivo del tema. | Cada nombre en `permitidos` debe existir aquí. |

### 2.2 La forma de un tema

Un tema es un objeto plano de tokens. Nada de lógica, nada de condicionales. Solo valores.

```js
// apps/pos/theme/calido.js
export default {
  nombre: 'Cálido',
  fondo:        '#1c1613',
  panel:        '#2a211c',
  borde:        '#3a2f28',
  texto:        '#f5efe3',
  textoSuave:   '#b8a99a',
  acento:       '#7a8b3c',
  peligro:      '#c0392b',
  radio:        '35px',
  densidad:     'aireada',
  fuenteCuerpo: 'Inter',
};
```

### 2.3 Los 5 tests de contrato

Cada módulo debe pasar estos 5 tests. Si uno falla, el módulo no se puede construir.

1. **El default existe en permitidos.** `permitidos.includes(default)` es verdadero.
2. **Hay entre 1 y 3 permitidos.** El default siempre está; los opcionales son hasta 2.
3. **Cada permitido tiene su archivo.** Todo nombre en `permitidos` existe como llave en `temas`.
4. **Cada tema pasa el contraste.** `validarContraste(tema)` no lanza error.
5. **El módulo no importa el motor de otro módulo.** Ningún import apunta a `apps/<otro>/theme/`.

Estos tests son la garantía de que un módulo nuevo no puede romper a los demás.

---

## 3. El motor compartido

El motor tiene **3 funciones**. Nada más. Es deliberadamente pequeño para que no se convierta en un monstruo.

### 3.1 `aplicarTema(tema)`

Toma un tema y lo escribe como variables CSS en el `:root` (o en el contenedor del módulo). A partir de ahí, todo el CSS del módulo lee esas variables.

```js
// packages/theme-engine/index.js
export function aplicarTema(tema, raiz = document.documentElement) {
  const variables = {
    '--fondo':        tema.fondo,
    '--panel':        tema.panel,
    '--borde':        tema.borde,
    '--texto':        tema.texto,
    '--texto-suave':  tema.textoSuave,
    '--acento':       tema.acento,
    '--peligro':      tema.peligro,
    '--radio':        tema.radio,
    '--fuente-cuerpo': tema.fuenteCuerpo,
  };
  for (const [nombre, valor] of Object.entries(variables)) {
    raiz.style.setProperty(nombre, valor);
  }
}
```

### 3.2 `resolverTema(modulo, eleccionDelUsuario, identidadDelNegocio)`

Es la función que decide **qué tema se aplica al final**. Sigue el orden de las 3 capas.

```js
// packages/theme-engine/index.js
export async function resolverTema(modulo, eleccionDelUsuario, identidad) {
  // CAPA 1: el tema del módulo (o su default si no hay elección)
  const nombreTema = elegirTemaValido(modulo, eleccionDelUsuario);
  const tema = await cargarTema(modulo, nombreTema);

  // CAPA 2: la identidad del negocio encima (logo, colores, tipografía)
  const conIdentidad = fusionarIdentidad(tema, identidad);

  // CAPA 3: la canónica como piso (lo que falte, se toma de aquí)
  return fusionarCanonica(conIdentidad);
}
```

**El orden importa:** el tema del módulo manda en fondos y superficies; la identidad del negocio manda en los 5 puntos de color y los 3 de tipografía; la canónica llena lo que nadie definió.

### 3.3 `validarContraste(tema)`

Revisa que el texto se lea sobre el fondo y que el acento se distinga. Si algo no pasa, lanza un error con el detalle.

```js
// packages/theme-engine/contraste.js
export function validarContraste(tema) {
  const problemas = [];
  if (contraste(tema.texto, tema.fondo) < 4.5) {
    problemas.push(`texto sobre fondo: ${contraste(tema.texto, tema.fondo).toFixed(2)}:1 (mínimo 4.5:1)`);
  }
  if (contraste(tema.acento, tema.fondo) < 3) {
    problemas.push(`acento sobre fondo: ${contraste(tema.acento, tema.fondo).toFixed(2)}:1 (mínimo 3:1)`);
  }
  if (contraste(tema.peligro, tema.acento) < 1.5) {
    problemas.push('peligro y acento son demasiado parecidos');
  }
  if (problemas.length > 0) {
    throw new Error(`Tema "${tema.nombre}" no pasa contraste:\n- ${problemas.join('\n- ')}`);
  }
}
```

---

## 4. La identidad del negocio (encima de todo)

La identidad **no es un tema**. Es una capa que se pone encima del tema que el módulo haya elegido.

### 4.1 Dónde se configura y dónde se guarda

- **Se configura** en el módulo **Vista General** (es el módulo de configuración del negocio).
- **Se guarda** en la base de datos, en la tabla `system_settings` que ya existe.
- **Se lee** desde cualquier módulo, con `GET /branding`.

### 4.2 Dónde APLICA la identidad (corregido 27 Sep 2026)

**La identidad no aplica en todo el ERP.** Aplica en **7 lugares muy específicos**, definidos en [`ADDENDUM_PUNTOS_DE_APLICACION_DEL_BRANDING.md`](./ADDENDUM_PUNTOS_DE_APLICACION_DEL_BRANDING.md). Fuera de esos 7, manda el tema.

| # | Punto de aplicación | Logo | Nombre | Colores | ¿Toca el nuevo POS? |
|---|---------------------|------|--------|---------|---------------------|
| 1 | Pantalla de logueo al ERP | ✅ | ✅ | ✅ | ❌ No |
| 2 | Barra selectora de módulos (lateral izq.) | ✅ | ✅ | ✅ | ❌ No |
| 3 | Encabezado de Vista General | ✅ | ✅ | ✅ | ❌ No |
| 4 | Tickets y cortes de caja | ✅ | ✅ | ❌ No | ⚠️ Parcial (solo formato) |
| 5 | Documentos oficiales (impresión/exportación) | ✅ | ✅ | ✅ (solo encabezado) | ❌ No |
| 6 | Icono de acceso de la app del ERP | ✅ | ❌ No | ✅ | ❌ No |
| 7 | Pantalla de carga de la app del ERP | ✅ | ✅ | ✅ | ❌ No |

**Consecuencia para esta propuesta:** los 7 puntos son del **ERP**, no del POS nuevo. Son **referencia de planificación**: describen **dónde irá** la identidad cuando esos módulos se reconstruyan. **Hoy no se implementa ninguno.** La Fase C **no toca el ERP** (ver §10). Las fases A, B y D siguen aisladas al POS nuevo.

### 4.3 Las 5 llaves nuevas

| Llave | Qué guarda | Ejemplo |
|---|---|---|
| `business_logo` | La ruta o el base64 del logo. | `/assets/logo.png` |
| `business_name` | El nombre de marca. | `"R de Rico"` |
| `business_colors` | **De 1 a 3** colores institucionales (JSON). | `["#7a8b3c", "#2f6b3f"]` |
| `business_font` | La tipografía corporativa. | `"Montserrat"` |
| `business_themes` | El mapa de temas por módulo (JSON). | `{"pos": "nocturno"}` |

**Nota 1:** los colores son **de 1 a 3**, no forzosamente 3. El sistema deriva el resto por contraste.

**Nota 2:** los temas van en **una sola llave JSON** (`business_themes`), no en una llave por módulo. Así el usuario puede poner el POS en "Nocturno" y Estadísticas en "Tablero" sin que uno afecte al otro, y el esquema no crece a mano.

### 4.4 El contrato `GET /branding`

```json
{
  "logo": "/assets/logo.png",
  "name": "R de Rico",
  "colors": ["#7a8b3c", "#2f6b3f"],
  "font": "Montserrat",
  "themes": {
    "pos": "nocturno",
    "estadisticas": "tablero",
    "vista-general": "kiosco"
  },
  "resolved": true
}
```

El campo `resolved` dice si la identidad se pudo leer completa. Si es `false`, el frontend muestra el aviso (ver sección 6).

---

## 5. Los 4 escenarios de caída

Esta es la parte que responde a la pregunta: *"¿y si algo se cae?"*

### Escenario 1 — Vista General se cae

- **Qué pasa:** no se puede *cambiar* la identidad (el formulario no carga).
- **Qué NO pasa:** la apariencia **sigue funcionando**. Los módulos ya tienen su tema guardado y su identidad cacheada.
- **Qué ve el usuario:** nada raro. Todo se ve igual que antes.
- **Por qué:** el motor no vive en Vista General. La identidad ya está en la BD y en caché.

### Escenario 2 — La base de datos se cae

- **Qué pasa:** no se puede leer la identidad ni el tema elegido.
- **Qué hace el sistema:** cada módulo **adopta su tema predeterminado** (el `default` del contrato).
- **Qué ve el usuario:** el aviso "Usando apariencia por defecto".
- **Por qué:** el `default` está en el código del módulo, no en la BD. Siempre está disponible.

### Escenario 3 — Un tema no carga (archivo roto, red lenta)

- **Qué pasa:** el `import()` del tema falla.
- **Qué hace el sistema:** cae al **tema predeterminado del módulo**.
- **Qué ve el usuario:** el aviso "No se pudo cargar el tema elegido; se usó el predeterminado".
- **Por qué:** el default siempre está en el bundle del módulo.

### Escenario 4 — Un tema no pasa el contraste

- **Qué pasa:** el tema tiene colores que no se leen.
- **Qué hace el sistema:** cae a la **paleta canónica** y avisa.
- **Qué ve el usuario:** el aviso "Se usó el color más cercano que sí se lee".
- **Por qué:** nunca mostramos texto ilegible. La legibilidad gana sobre la estética.

**Resumen:** en los 4 escenarios, el sistema **nunca se queda sin apariencia**. Siempre hay un piso.

---

## 6. Los 2 avisos visibles

Los avisos son discretos pero visibles. No bloquean el trabajo.

### Aviso A — "Usando apariencia por defecto"

Aparece cuando la identidad o el tema no se pudieron leer (escenarios 1, 2 y 3).
Texto: *"No se pudo cargar tu apariencia. Estás viendo la apariencia por defecto."*
Dónde: una franja delgada arriba del módulo, se puede cerrar.

### Aviso B — "Se usó el color más cercano"

Aparece cuando un color de la identidad no pasa el contraste (escenario 4).
Texto: *"Uno de tus colores no se lee bien sobre el fondo. Usamos el más cercano que sí se lee."*
Dónde: junto al punto de color afectado, en Vista General.

---

## 7. Elección por módulo vs. elección global

**Decisión: por módulo, con predeterminado.**

### 7.1 Por qué por módulo

Cada módulo tiene una vocación distinta. El POS se usa de pie, con prisa, con las manos ocupadas. Estadísticas se usa sentado, con calma, leyendo gráficas. Vista General es un kiosco que se ve de lejos.

Forzar un solo tema para los tres sería como ponerle el mismo uniforme al panadero, al contador y al vigilante. Cada uno necesita lo suyo.

### 7.2 Por qué con predeterminado

El usuario no debería tener que elegir tema para 12 módulos el primer día. Cada módulo **nace con un tema que ya funciona**. Si el usuario no toca nada, todo se ve bien. Si quiere cambiar, puede.

### 7.3 Cómo se ve en la interfaz

En Vista General, una sección "Apariencia por módulo":

```
Apariencia por módulo

  POS                 [ Cálido ▾ ]   (predeterminado)
  Estadísticas        [ Tablero ▾ ]  (predeterminado)
  Vista General       [ Kiosco ▾ ]   (predeterminado)
  ...
```

Cada módulo muestra su tema actual. El desplegable ofrece los 3 del módulo. El predeterminado viene marcado.

---

## 8. Los 3 temas del módulo POS (el primero que construimos)

Como el POS es el primer módulo que estamos construyendo, aquí están sus 3 temas concretos.

### 8.1 Cálido (predeterminado)

Pensado para una panadería. Tonos tierra, luz suave, se siente acogedor.

| Token | Valor |
|---|---|
| fondo | `#1c1613` |
| panel | `#2a211c` |
| borde | `#3a2f28` |
| texto | `#f5efe3` |
| textoSuave | `#b8a99a` |
| acento | `#7a8b3c` |
| peligro | `#c0392b` |
| radio | `35px` |
| densidad | `aireada` |

### 8.2 Nocturno (opcional 1)

Pensado para turnos de noche o locales con poca luz. Azul profundo, acento dorado.

| Token | Valor |
|---|---|
| fondo | `#0d1b2a` |
| panel | `#1b2a3a` |
| borde | `#2a3d52` |
| texto | `#f2f6fa` |
| textoSuave | `#9fb3c8` |
| acento | `#f0a500` |
| peligro | `#e63946` |
| radio | `35px` |
| densidad | `aireada` |

### 8.3 Minimal (opcional 2)

Pensado para quien quiere lo esencial sin adornos. Grises neutros, acento verde.

| Token | Valor |
|---|---|
| fondo | `#111111` |
| panel | `#1c1c1c` |
| borde | `#2e2e2e` |
| texto | `#fafafa` |
| textoSuave | `#a3a3a3` |
| acento | `#a3c614` |
| peligro | `#d64545` |
| radio | `12px` |
| densidad | `compacta` |

**Nota:** "Minimal" vive en `temas-compartidos/` porque varios módulos lo van a querer. Los otros dos son propios del POS.

---

## 9. Costo y mitigaciones

### 9.1 El costo real

| Concepto | Costo |
|---|---|
| El motor compartido | Se escribe **una vez**. ~150 líneas. |
| El contrato del módulo | Se escribe **una vez por módulo**. ~15 líneas. |
| Los temas por módulo | **De 1 a 3 por módulo** (1 predeterminado + hasta 2 opcionales). Este es el costo que crece. |
| Los 5 tests de contrato | Se escriben **una vez**, se reusan por módulo. |

Con 12 módulos: 12 contratos + 36 temas. Es trabajo, pero es trabajo **acotado y repetible**.

### 9.2 Las mitigaciones

1. **`temas-compartidos/`** — los temas que varios módulos quieren se escriben una sola vez.
2. **El contrato es idéntico** — copiar la estructura de un módulo a otro es trivial.
3. **Los tests son genéricos** — el mismo test corre para todos los módulos.
4. **Se construye por módulo** — no hay que hacer los 36 temas de golpe. Se hace el del POS ahora, los demás cuando se construyan.

### 9.3 Lo que NO se hace

- No se crean 36 temas de golpe. Se crean cuando se construye cada módulo.
- No se inventan temas "por si acaso". Cada tema tiene que tener una razón.
- **No se toca el ERP. No se toca el POS que corre.** Los 7 puntos son referencia futura (ver Fase C).

---

## 10. Impacto en las fases (corregido v4 — 28 Sep 2026)

| Fase | Qué se hace | Toca el ERP? | Toca el POS nuevo? |
|---|---|---|---|
| **A — Motor** | Crear `packages/theme-engine/` con las 3 funciones y los tests. | **No** | Sí |
| **B — POS** | Crear `apps/pos/theme/` con los temas y el contrato. Aplicar en el POS nuevo. | **No** | Sí |
| **C — Identidad (referencia futura)** | **Explorar** cómo el POS nuevo interactuará con los módulos que aún no se construyen, y **dejar preparado** el contrato `GET /branding` y las 5 llaves. Los **7 puntos de aplicación en el ERP** quedan como **referencia de planificación** para cuando esos módulos se reconstruyan. | **No — es referencia futura** | Sí (solo el POS nuevo) |
| **D — Los demás módulos** | Repetir el patrón al construir cada módulo. | **No** | Sí |

**Cambio clave respecto a la versión anterior:** la Fase C **NO toca el ERP hoy**. Los 7 puntos de aplicación son **referencia de planificación** para cuando esos módulos se reconstruyan. Por eso **no requiere ventana de mantenimiento**: no hay nada que mantener porque no se toca nada. Lo único que se hace hoy en la Fase C es **explorar la interacción** y **dejar todo preparado** (contrato, llaves, motor). Las fases A, B y D siguen aisladas al POS nuevo.

---

## 11. Estado

- [x] Decisión de arquitectura: motor compartido + temas por módulo + identidad encima.
- [x] Decisión de elección: por módulo, con predeterminado.
- [x] Los 3 temas del POS definidos (Cálido, Nocturno, Minimal).
- [x] Los 4 escenarios de caída resueltos.
- [x] Los 5 tests de contrato definidos.
- [x] **Corrección 27 Sep 2026:** puntos de aplicación de la identidad alineados con el addendum (7 puntos del ERP, no del POS).
- [x] **Corrección 27 Sep 2026:** colores corporativos de 1 a 3 (no forzosamente 3).
- [x] **Corrección 27 Sep 2026:** temas de 1 a 3 por módulo (no "exactamente 3").
- [x] **Corrección 27 Sep 2026:** una sola llave JSON `business_themes` (no una por módulo).
- [x] **Mejoras incorporadas:** contraste post-identidad, tema en contenedor del módulo, default importado estáticamente.
- [x] **Corrección v4 (28 Sep 2026):** los 7 puntos son **referencia de planificación**, no trabajo actual. La regla dura (**NO se toca el ERP ni el POS que corre**) se mantiene intacta. La Fase C vuelve a **"No toca el ERP"**.
- [ ] Revisión del dueño.
- [ ] Aprobación para empezar la Fase A.

---

**Fin de la propuesta V4.**
