# Plan de Implementación — UI por Módulo del Nuevo POS

> **Documento padre (arquitectura):** [`PROPUESTA_APARIENCIA_POR_MODULO_V4.md`](./PROPUESTA_APARIENCIA_POR_MODULO_V4.md) — se conserva como **anexo de arquitectura** (el "por qué").
> **Documento complementario (IA):** [`ESPECIFICACION_EXTRACTOR_ESTETICA_VISUAL.md`](./ESPECIFICACION_EXTRACTOR_ESTETICA_VISUAL.md) — la especificación del Extractor de Estética Visual (Fase 1.5).
> **Este documento es el "cómo":** el plan de ejecución para darle al nuevo POS una **UI por defecto** y **dos opcionales**, con el enfoque de configuración general de UIs del nuevo ERP.
> **Regla dura vigente:** NO SE TOCA EL ERP INSTALADO Y CORRIENDO. NO SE TOCA EL POS QUE CORRE.
> **Alcance de este plan:** SOLO el nuevo POS (`../NUEVO-POS/`). Los demás módulos del ERP son **referencia futura**.
> **Creado:** 28 Sep 2026. **Corregido:** 28 Sep 2026 (el default visual **no está diseñado**; se añadió la Fase D). **Actualizado:** 28 Sep 2026 (se integró la Fase 1.5 — Extractor de Estética Visual, idea del dueño).

---

## 0. La aclaración del dueño (esto manda)

El dueño precisó el diseño. Estas 4 frases son la ley de este plan:

1. **La identidad del negocio se configura desde el módulo que se llamará Visión General.**
2. **El tema por defecto de cada módulo vive en el módulo.** (En este caso, el nuevo POS.)
3. **El módulo da (o puede NO dar) la opción de seleccionar si deseamos cambiar el tema, y cuando la da, presenta dos opciones más.**
4. **El default visual del POS todavía no está diseñado, y el dueño quiere diseñarlo.** (Aclarado 28 Sep 2026.)

### 0.1 Lo que esto significa, traducido

| Frase del dueño | Consecuencia de diseño |
|---|---|
| "Se configura desde Visión General" | Visión General es el **único** lugar donde se edita la identidad (logo, nombre, colores, tipografía). Ningún otro módulo la edita. |
| "El tema por defecto vive en el módulo" | El **default** es un archivo **dentro** de `apps/pos/theme/`. No vive en Visión General ni en un catálogo global. |
| "El módulo da (o puede NO dar) la opción de cambiar" | El **selector de tema es opcional y propiedad del módulo**. Un módulo puede decidir no ofrecer cambio de tema. |
| "Presenta dos opciones más" | Cuando el módulo sí ofrece cambio, muestra **2 opciones adicionales** al default → **máximo 3 temas** (1 default + 2 opcionales). |
| "El default no está diseñado y quiero diseñarlo" | Se añade la **Fase D (diseño)** **antes** de la Fase 0. El default **no se hereda**: se diseña y se aprueba. |

### 0.2 La regla de oro del selector

> **El módulo decide si ofrece o no el selector. Visión General solo configura la identidad, no los temas.**

Esto separa dos responsabilidades que la V4 mezclaba:

- **Visión General** → identidad del negocio (logo, nombre, colores, tipografía). **No** elige temas.
- **El módulo** → su tema por defecto + (opcionalmente) su selector con 2 opciones más.

### 0.3 ¿Se puede hacer esto sin molestar al ERP que está vendiendo? (SÍ)

**Pregunta del dueño (28 Sep 2026):** *"¿Podremos hacer esto sin molestar ni interrumpir al POS del ERP que corre actualmente y que está en hora rush?"*

**Respuesta: SÍ, con una condición.** El trabajo de este plan es **100% aislado** del ERP. La evidencia está en el código, no en una promesa:

| Aislamiento | Evidencia | Por qué no molesta al ERP |
|---|---|---|
| **Base de datos separada** | El POS nuevo usa `nuevo_pos` en el contenedor `nuevo_pos_db` ([`docker-compose.yml`](../../NUEVO-POS/docker-compose.yml:27)). El ERP usa `rderico-db-dev` en el puerto 5433 ([`docker-compose.yml`](docker-compose.yml:4)). | Son dos Postgres distintos. Escribir en uno no toca al otro. |
| **Puertos distintos** | POS nuevo: frontend 5100, API 5101. ERP: POS 5000, API 5001, BD 5433. | Cero colisión de puertos. |
| **API separada** | El POS nuevo tiene su propio contenedor `nuevo_pos_api` ([`docker-compose.yml`](../../NUEVO-POS/docker-compose.yml:44)). El ERP tiene `rderico-api-dev` ([`docker-compose.yml`](docker-compose.yml:19)). | Apagar el API del POS nuevo no apaga el ERP. |
| **Frontend separado** | El POS nuevo es un proyecto Vite propio en `../NUEVO-POS/apps/pos/`. El POS viejo es código dentro del ERP. | Son dos carpetas distintas. |
| **Config propia** | [`config.js`](../../NUEVO-POS/apps/shared/config.js:4) declara: *"`CONFIG.API_BASE_URL` es PROPIO del POS nuevo. NO se reutiliza el del ERP."* | El POS nuevo nunca apunta al API del ERP. |

**La condición (importante):** el aislamiento es real **siempre que se respete la regla dura**. El riesgo no es técnico, es humano:

> **NUNCA ejecutar `docker compose down` en la carpeta del ERP.** Ese comando apagaría el ERP entero. Todos los comandos de este plan se ejecutan **dentro de `../NUEVO-POS/`**, nunca en la raíz del ERP.

**Regla operativa para hora rush:**

| Se puede hacer en hora rush | NO se puede hacer en hora rush |
|---|---|
| Editar archivos del POS nuevo (Tailwind, CSS, componentes). | Tocar cualquier archivo del ERP. |
| Levantar el POS nuevo en 5100/5101. | Apagar o reiniciar contenedores del ERP. |
| Correr tests del POS nuevo. | Compartir la BD del ERP. |
| Diseñar y probar temas. | Publicar el POS nuevo en el puerto del ERP (5000). |

**Conclusión:** el diseño del default visual (Fase D) y todo el plan se pueden hacer **en paralelo, en hora rush, sin tocar el ERP**. El único cuidado es no ejecutar comandos en la carpeta equivocada.

### 0.4 Qué podrás diseñar tú (en palabras simples)

**La idea:** el POS tiene **6 colores, 3 formas y 2 tipografías**. Eso es todo lo que define su apariencia. Si tú controlas esos 11 valores, controlas el 100% del look. No hay más.

**Los 6 colores que podrás elegir:**

| # | Qué pinta | Dónde se ve | Ejemplo de hoy |
|---|---|---|---|
| 1 | **Acento** | Precios, botón COBRAR, categoría activa, logo | Verde lima `#c1d72e` |
| 2 | **Fondo profundo** | El fondo de toda la pantalla | Casi negro `#0a0a0a` |
| 3 | **Fondo profundo alterno** | El header y las zonas de imagen | Negro `#080808` |
| 4 | **Fondo panel** | Las tarjetas de producto y los botones | Gris oscuro `#1a1a1a` |
| 5 | **Crema ticket** | El panel del ticket y el texto principal | Blanco cálido `#fdfbf7` |
| 6 | **Peligro** | El botón de quitar (✕) y los errores | Rojo `#ef4444` |

**Las 3 formas que podrás elegir:**

| # | Qué pinta | Ejemplo de hoy |
|---|---|---|
| 1 | **Radio grande** | Tarjetas de producto y botones | 35px |
| 2 | **Radio medio** | El panel del ticket | 40px |
| 3 | **Radio máximo** | Elementos redondos | 50px |

**Las 2 tipografías que podrás elegir:**

| # | Qué pinta | Ejemplo de hoy |
|---|---|---|
| 1 | **Tipografía de la UI** | Todo el POS | Inter |
| 2 | **Tipografía del ticket** | Los números del ticket | Monoespaciada |

**Lo que esto significa en la práctica:** tú dices "quiero un POS claro con acento azul, esquinas suaves y tipografía redondeada" y eso se traduce a 11 valores. El POS entero cambia. **No hay que tocar 24 componentes uno por uno.**

**Las expectativas (lo que el diseño debe cumplir):**

1. **Buen gusto:** los 6 colores deben convivir sin chocar. Un acento que resalte sin gritar.
2. **Legible:** el texto debe leerse sin esfuerzo. Contraste mínimo 4.5:1 (WCAG AA).
3. **Táctil:** los botones deben ser fáciles de tocar con el dedo (44×44px mínimo). Ya está resuelto.
4. **Coherente en los 3 modos:** debe verse bien en MOSTRADOR (pantalla grande), COMPACTO (tablet) y MÓVIL (celular).
5. **Que no rompa el cobro:** el diseño no puede hacer que un botón desaparezca o que un precio no se lea.

**Lo que NO podrás cambiar (y por qué):**

- **La estructura:** dónde va el ticket, dónde van los productos. Eso es funcional, no visual.
- **Los 3 modos:** MOSTRADOR, COMPACTO y MÓVIL son fijos (regla R-03).
- **El target táctil de 44px:** es una regla dura (R-04), no un gusto.

### 0.5 Dónde se cambia el tema y cómo se ve (la UX)

**La respuesta corta:** el selector de tema vive en el **header del POS**, arriba a la derecha, junto al indicador de terminal y sesión. Es un botón discreto que abre un panel con las 3 opciones.

**El header actual** ([`RetailVisionPOS.jsx`](../../NUEVO-POS/apps/pos/src/RetailVisionPOS.jsx:162)) ya tiene el lugar perfecto:

```
┌──────────────────────────────────────────────────────────────┐
│  R de Rico  POS Nuevo        [TERM-01] [Sesión abierta] [🎨] │
└──────────────────────────────────────────────────────────────┘
                                                          ↑
                                              aquí va el selector
```

**Cómo se ve la UX, paso a paso:**

1. **El botón:** un ícono de paleta (🎨) o un círculo con el color de acento actual. Discreto, en el header.
2. **Al tocarlo:** se abre un panel pequeño (no una pantalla completa) con **3 tarjetas de vista previa**:
   - **Default** (el que tú diseñes) — marcado como "Actual".
   - **Opción 1** — con su nombre.
   - **Opción 2** — con su nombre.
3. **Cada tarjeta** muestra una miniatura: un cuadrito con los colores del tema, para que se vea sin aplicarlo.
4. **Al elegir una:** el POS cambia **al instante**, sin recargar. El panel se cierra.
5. **Al recargar la página:** el POS recuerda tu elección.

**La regla de oro (tu aclaración):** el selector es **opcional y del módulo**. Si algún día decides que el POS **no** debe ofrecer cambio de tema, se apaga con una sola línea (`ofreceSelector: false`) y el botón desaparece. El POS se queda con su default y nadie puede cambiarlo.

**Dónde NO va el selector:**

- **No va en Visión General.** Visión General configura la **identidad del negocio** (logo, nombre, colores corporativos), no los temas de cada módulo.
- **No es global.** Cambiar el tema del POS no cambia el tema de otros módulos.

**Quién puede cambiar el tema:** por defecto, cualquiera que use el POS. Si quieres restringirlo (solo supervisores, por ejemplo), se puede añadir un permiso. Eso se decide en la Fase 4.

---

### 0.6 ¿El tema elegido es persistente o se decide a diario?

**Pregunta del dueño (28 Sep 2026):** *"Una vez establecido un tema opcional, ¿esta configuración será persistente hasta nuevo cambio o habrá que decidirla a diario?"*

**Respuesta: PERSISTENTE.** Se elige una vez y se queda hasta que alguien la cambie. **No se decide a diario.**

**Cómo funciona la persistencia (3 niveles):**

| Nivel | Qué guarda | Dónde | Cuándo se usa |
|---|---|---|---|
| **1. Memoria del navegador** | El tema elegido en **esa terminal**. | `localStorage` del navegador. | Al recargar la página, el POS recuerda el tema. |
| **2. Preferencia por terminal** | El tema elegido para **esa caja específica**. | Base de datos del POS nuevo (`nuevo_pos`). | Si se limpia el navegador, el POS recupera el tema de la terminal. |
| **3. Default del módulo** | El tema de fábrica. | `apps/pos/theme/default.js`. | Si nunca se ha elegido nada, se usa este. |

**El orden de decisión (de mayor a menor prioridad):**

1. ¿Hay una elección guardada en **esta terminal**? → Se usa esa.
2. ¿No hay? → Se usa el **default del módulo**.

**Lo que esto significa en la práctica:**

- **Día 1:** el POS arranca con el default que tú diseñes.
- **Día 2:** el cajero cambia a "Opción 1". El POS la guarda.
- **Día 3 en adelante:** el POS arranca **siempre** con "Opción 1". Nadie tiene que volver a elegir.
- **Día N:** alguien cambia a "Opción 2". El POS la guarda. Y así hasta el siguiente cambio.

**Preguntas que esto resuelve:**

| Pregunta | Respuesta |
|---|---|
| ¿Hay que elegir el tema cada día? | **No.** Se elige una vez. |
| ¿Se pierde al recargar la página? | **No.** Se guarda en el navegador. |
| ¿Se pierde al reiniciar la computadora? | **No.** El navegador lo conserva. |
| ¿Se pierde si se limpia el navegador? | **No**, si se guarda también por terminal (nivel 2). |
| ¿Cada caja puede tener su propio tema? | **Sí.** Es por terminal, no global. |
| ¿El tema de una caja afecta a otra? | **No.** Cada terminal es independiente. |

**DECISIÓN DEL DUEÑO (28 Sep 2026): OPCIÓN B — niveles 1 + 2.** La preferencia se guarda **en el navegador (nivel 1) Y por terminal en la base de datos (nivel 2)**. Si se limpia el navegador de una caja, el POS recupera el tema de la terminal. Es la opción robusta y la que da la experiencia persistente real.

**Cómo se implementa la Opción B (2 capas):**

| Capa | Qué guarda | Dónde | Rol |
|---|---|---|---|
| **Capa rápida (nivel 1)** | El tema de **esta terminal**. | `localStorage` del navegador. | Lectura instantánea al arrancar (sin esperar a la red). |
| **Capa durable (nivel 2)** | El tema de **esta caja**. | Tabla `preferencias_terminal` en `nuevo_pos`. | Respaldo: si se limpia el navegador, se recupera de aquí. |

**Orden de arranque (Opción B):**

1. Leer `localStorage` (capa rápida). Si hay tema → se aplica **de inmediato** (sin parpadeo).
2. En paralelo, consultar la terminal en la base de datos (capa durable).
3. Si la base de datos tiene un tema **distinto** al del navegador → gana la base de datos (es la fuente de verdad de la caja) y se re-sincroniza el `localStorage`.
4. Si no hay nada en ninguna capa → se usa el **default del módulo** (nivel 3).

**Al cambiar el tema:** se escribe en **las dos capas** (navegador + terminal) para que queden sincronizadas.

**Lo que NO pasa:** el tema **no** se resetea cada día, **no** se resetea al cerrar sesión, y **no** se resetea al cobrar. Es una preferencia estable, como el volumen de un equipo.

**Nota técnica:** hoy el POS nuevo **no tiene ningún mecanismo de persistencia** (se verificó: no hay `localStorage` en el código) **ni endpoint de preferencias por terminal**. La Opción B se construye en la **Fase 4**: tarea 4.3 (capa rápida, `localStorage`) y tarea 4.5 (capa durable, endpoint + tabla `preferencias_terminal`).

---

## 1. Las 3 decisiones que la V4 dejó abiertas (resueltas aquí)

### Decisión 1 — Mecanismo: tokens Tailwind apuntando a variables CSS

**El problema real (defecto crítico de la V4):** los componentes usan clases Tailwind con hex compilado en build-time ([`tailwind.config.js`](../../NUEVO-POS/apps/pos/tailwind.config.js:19): `acento: '#c1d72e'`). Cambiar una variable CSS en runtime **no cambia nada**, porque la clase nunca lee la variable.

**La decisión:** los tokens de Tailwind **apuntan a variables CSS**. Así la clase `bg-acento` compila a `background-color: var(--acento)`, y cambiar `--acento` en runtime **sí** cambia la UI.

```js
// apps/pos/tailwind.config.js (DESPUÉS)
colors: {
  acento: 'rgb(var(--acento) / <alpha-value>)',
  'fondo-profundo': 'rgb(var(--fondo-profundo) / <alpha-value>)',
  'fondo-profundo-alt': 'rgb(var(--fondo-profundo-alt) / <alpha-value>)',
  'fondo-panel': 'rgb(var(--fondo-panel) / <alpha-value>)',
  'crema-ticket': 'rgb(var(--crema-ticket) / <alpha-value>)',
  peligro: 'rgb(var(--peligro) / <alpha-value>)',
},
```

**Por qué `rgb(var(--x) / <alpha-value>)` y no `var(--x)` directo:** Tailwind necesita el canal separado para que funcionen las opacidades (`bg-acento/40`, `text-crema-ticket/50`). Con `var(--x)` directo, las opacidades se rompen. Esta es la forma canónica de Tailwind para tokens dinámicos.

**Consecuencia:** las variables se declaran como **canales RGB** (`--acento: 193 215 46`), no como hex. El hex se conserva solo como referencia en la paleta canónica.

**Costo real (corrige la subestimación de la V4):** no son "~150 líneas de motor". Es:
- Reescribir `tailwind.config.js` (1 archivo).
- Reescribir `index.css` para declarar canales RGB (1 archivo).
- **Verificar** que los 24 componentes sigan viéndose igual (no hay que reescribir cada componente si usan los nombres de token; solo cambia cómo se resuelven).

**Lo que NO hay que hacer:** reescribir los 24 componentes uno por uno. Los componentes ya usan **nombres de token** (`bg-acento`, `text-crema-ticket`), no hex sueltos. Al cambiar la resolución del token, los componentes siguen funcionando sin tocarlos. **Este es el hallazgo que salva el plan.**

### Decisión 2 — Vocabulario único de tokens

Se adopta el vocabulario **canónico que ya existe** en [`PALETA_CANONICA`](../../NUEVO-POS/apps/api/superficie/registry.py:58). La V4 inventaba nombres nuevos; se descartan.

| Token (único, canónico) | Canal RGB por defecto | Hex de referencia |
|---|---|---|
| `--acento` | `193 215 46` | `#c1d72e` |
| `--fondo-profundo` | `10 10 10` | `#0a0a0a` |
| `--fondo-profundo-alt` | `8 8 8` | `#080808` |
| `--fondo-panel` | `26 26 26` | `#1a1a1a` |
| `--crema-ticket` | `253 251 247` | `#fdfbf7` |
| `--peligro` | `239 68 68` | `#ef4444` |

**Regla:** no se inventan nombres nuevos. Si un tema necesita un color, usa uno de estos 6 tokens.

### Decisión 3 — El default es un DISEÑO PENDIENTE del dueño (corregido 28 Sep 2026)

> **Corrección importante.** La versión anterior de este plan decía "el default es la paleta canónica actual". **Eso era un error de suposición.** El dueño aclaró: *"no he tenido oportunidad de ver siquiera el default visual del POS y de hecho sí me gustaría diseñar este default visual del POS."*

**El default visual del POS todavía NO está diseñado.** Lo que existe hoy es un **punto de partida funcional**, no un default aprobado. La paleta canónica (`#c1d72e`, `#0a0a0a`, …) es el **piso técnico** que el POS usa hoy para funcionar, pero **no es la decisión visual final**.

**La distinción que faltaba:**

| Concepto | Qué es | Quién lo decide |
|---|---|---|
| **Paleta canónica** | El piso técnico. Los 6 colores que el POS usa hoy para no romperse. | Ya existe en [`PALETA_CANONICA`](../../NUEVO-POS/apps/api/superficie/registry.py:58). |
| **Default visual del POS** | El diseño final de la apariencia por defecto. | **El dueño. Está pendiente.** |

**Consecuencia para el plan:** se añade una **Fase de Diseño (Fase D)** **antes** de la Fase 0. El mecanismo (Fase 0) se construye igual, pero el **valor** del default se decide en la Fase D, no se hereda por defecto.

**Lo que NO cambia:** el default sigue viviendo **dentro del módulo** (`apps/pos/theme/canonico.js`), y sigue siendo el que la puerta F5 valida. Lo que cambia es que su contenido se **diseña**, no se asume.

**Regla nueva:** mientras el dueño no apruebe el diseño del default, la Fase 0 usa la paleta canónica **como valor provisional** para poder probar el mecanismo. El diseño aprobado la reemplaza después.

---

## 2. Reconciliación con la puerta F5 (defecto crítico de la V4)

La puerta F5 verifica que **la paleta canónica se respete** ([`registry.py`](../../NUEVO-POS/apps/api/superficie/registry.py:6)). La V4 proponía un tema "Cálido" con acento `#7a8b3c`, que **viola** esa verificación.

**La reconciliación:**

| Concepto | Regla |
|---|---|
| **Paleta canónica** | Es el **piso técnico**. Los 6 colores que el POS usa hoy para funcionar. |
| **Default visual** | Es el **diseño aprobado en la Fase D**. Puede coincidir con la canónica o no. |
| **Tema opcional** | Puede cambiar el acento, pero **debe pasar el contraste WCAG AA** y **debe declararse como tema**, no como paleta. |
| **Puerta F5** | Verifica que el **default** respete el piso canónico. Los temas opcionales se validan con el **test de contraste del motor**, no con F5. |

**Regla nueva:** la puerta F5 valida el **default** (que debe respetar el piso canónico). El motor valida los **temas opcionales**. Son dos validaciones distintas y no se pisan.

**Pregunta abierta para el dueño:** si el default diseñado en la Fase D se aleja de la paleta canónica, hay que decidir si F5 se relaja o si el default debe respetar la canónica. **Esta decisión se toma en la Fase D, no antes.**

---

## 3. El contrato del módulo (con selector opcional)

El contrato ahora declara **si el módulo ofrece selector o no**.

```js
// apps/pos/theme/index.js
export const TEMA_DEL_MODULO = {
  modulo: 'pos',

  // El default vive AQUÍ, en el módulo. Es el DISEÑADO en la Fase D.
  default: 'default',

  // ¿El módulo ofrece cambiar de tema? Si es false, no hay selector.
  ofreceSelector: true,

  // Los temas disponibles. Si ofreceSelector=false, solo hay 1 (el default).
  permitidos: ['default', 'nocturno', 'minimal'],

  temas: {
    default:  () => import('./default.js'),
    nocturno: () => import('./nocturno.js'),
    minimal:  () => import('../../../../packages/theme-engine/temas-compartidos/minimal.js'),
  },
};
```

### 3.1 Las dos formas del contrato

| Forma | `ofreceSelector` | `permitidos` | Cuándo se usa |
|---|---|---|---|
| **Módulo sin selector** | `false` | `['default']` (solo el default) | El módulo no quiere que el usuario cambie su tema. |
| **Módulo con selector** | `true` | `['default', 'nocturno', 'minimal']` (1 + 2) | El módulo ofrece 2 opciones más. |

**Regla dura:** si `ofreceSelector` es `false`, `permitidos` **debe** tener exactamente 1 elemento (el default). El test lo verifica.

### 3.2 Los 6 tests de contrato (antes eran 5)

1. **El default existe en permitidos.** `permitidos.includes(default)` es verdadero.
2. **Hay entre 1 y 3 permitidos.** El default siempre está; los opcionales son hasta 2.
3. **Cada permitido tiene su archivo.** Todo nombre en `permitidos` existe como llave en `temas`.
4. **Cada tema pasa el contraste.** `validarContraste(tema)` no lanza error.
5. **El módulo no importa el motor de otro módulo.** Ningún import apunta a `apps/<otro>/theme/`.
6. **Coherencia del selector (NUEVO).** Si `ofreceSelector === false`, entonces `permitidos.length === 1`. Si `ofreceSelector === true`, entonces `permitidos.length` está entre 2 y 3.

---

## 4. Dónde vive cada pieza (corregido)

```
NUEVO-POS/
├── packages/
│   └── theme-engine/                    ← EL MOTOR (uno solo, compartido)
│       ├── index.js                     ← aplicarTema, resolverTema, validarContraste
│       ├── contraste.js                 ← cálculo de contraste WCAG
│       └── temas-compartidos/           ← temas que varios módulos reusan
│           └── minimal.js
│
└── apps/
    └── pos/
        ├── tailwind.config.js           ← tokens → var(--x) (DECISIÓN 1)
        ├── src/
        │   ├── index.css                ← declara los canales RGB del default
        │   └── theme/
        │       ├── index.js             ← TEMA_DEL_MODULO (el contrato)
        │       ├── default.js           ← DEFAULT (el DISEÑADO en la Fase D)
        │       ├── nocturno.js          ← opcional 1
        │       └── minimal.js           ← opcional 2 (reusa el compartido)
        └── src/components/
            └── ThemeSelector.jsx        ← el selector (solo si ofreceSelector=true)
```

**Regla de oro:** un módulo **nunca** importa el motor de otro módulo. Todos importan `packages/theme-engine/`.

---

## 5. El motor compartido (3 funciones)

### 5.1 `aplicarTema(tema, contenedor)`

Escribe los **canales RGB** como variables CSS en el **contenedor del módulo** (no en `:root`, para evitar contaminación cruzada).

```js
// packages/theme-engine/index.js
export function aplicarTema(tema, contenedor = document.documentElement) {
  const canales = {
    '--acento':            tema.acento,
    '--fondo-profundo':    tema.fondoProfundo,
    '--fondo-profundo-alt': tema.fondoProfundoAlt,
    '--fondo-panel':       tema.fondoPanel,
    '--crema-ticket':      tema.cremaTicket,
    '--peligro':           tema.peligro,
  };
  for (const [nombre, valor] of Object.entries(canales)) {
    contenedor.style.setProperty(nombre, valor); // valor = "193 215 46"
  }
}
```

### 5.2 `resolverTema(modulo, eleccionDelUsuario, identidad)`

Sigue el orden de las 3 capas: tema del módulo → identidad encima → canónica como piso.

### 5.3 `validarContraste(tema)`

Valida texto/fondo ≥ 4.5:1 y acento/fondo ≥ 3:1. Se aplica **al resultado final** (`fusionarIdentidad(tema, identidad)`), no solo al tema.

---

## 6. Las fases (con archivos, tareas, criterios y reversa)

### Fase D — DISEÑO del default visual del POS (NUEVA, va primero)

**Objetivo:** que el dueño **vea y diseñe** la apariencia por defecto del POS. Hoy no la ha visto; lo que existe es un punto de partida funcional, no un default aprobado.

**Punto de partida (lo que hay hoy, para que el dueño lo vea):**

| Elemento | Valor actual (provisional) | Archivo |
|---|---|---|
| Acento | `#c1d72e` (verde lima) | [`tailwind.config.js`](../../NUEVO-POS/apps/pos/tailwind.config.js:19) |
| Fondo profundo | `#0a0a0a` (casi negro) | [`index.css`](../../NUEVO-POS/apps/pos/src/index.css:15) |
| Fondo panel | `#1a1a1a` (gris oscuro) | [`index.css`](../../NUEVO-POS/apps/pos/src/index.css:17) |
| Crema ticket | `#fdfbf7` (blanco cálido) | [`index.css`](../../NUEVO-POS/apps/pos/src/index.css:18) |
| Peligro | `#ef4444` (rojo) | [`index.css`](../../NUEVO-POS/apps/pos/src/index.css:19) |
| Radios | 35px / 40px / 50px | [`tailwind.config.js`](../../NUEVO-POS/apps/pos/tailwind.config.js:28) |
| Tipografía | Inter (UI) + monoespaciada (ticket) | [`tailwind.config.js`](../../NUEVO-POS/apps/pos/tailwind.config.js:34) |
| Target táctil | 44×44px mínimo | [`index.css`](../../NUEVO-POS/apps/pos/src/index.css:41) |

**Cómo se ve hoy, en palabras:** fondo casi negro, tarjetas en gris oscuro con esquinas muy redondeadas (35px), acento verde lima para precios y botones activos, texto crema. Es un look "oscuro con acento lima".

**Dos vías para diseñar el default (se pueden combinar):**

| Vía | Cómo funciona | Cuándo usarla |
|---|---|---|
| **A — Manual** (la original) | El dueño define dirección, valores, tipografía y radios a mano. El contraste se calcula manualmente. | Cuando el dueño tiene una visión clara y específica de los valores. |
| **B — Extractor de Estética Visual** (nueva, 28 Sep 2026) | El dueño sube una imagen de Pinterest/Dribbble al Extractor (Fase 1.5). La IA extrae los 11 tokens, valida contraste automáticamente y sugiere ajustes. El dueño revisa, calibra y aprueba. | Cuando el dueño tiene una imagen de referencia pero no los valores exactos. **Reduce drásticamente el esfuerzo.** |

> **Especificación completa de la Vía B:** [`ESPECIFICACION_EXTRACTOR_ESTETICA_VISUAL.md`](./ESPECIFICACION_EXTRACTOR_ESTETICA_VISUAL.md)

| # | Tarea | Vía A (manual) | Vía B (extractor) | Criterio de aceptación |
|---|---|---|---|---|
| D.1 | El dueño **ve** el POS actual corriendo | Captura o sesión en vivo | Igual | El dueño confirma que ya vio el default actual. |
| D.2 | El dueño define la **dirección visual** del default | Nota de dirección (oscuro/claro, acento, densidad) | **Sube 1-3 imágenes de referencia al Extractor** | Hay una dirección (escrita o extraída), no una suposición. |
| D.3 | Definir los **6 valores** del default | Tabla de los 6 tokens (escrita a mano) | **La IA extrae los valores; el dueño ajusta** | Los 6 valores están escritos y aprobados. |
| D.4 | Definir **tipografía y radios** del default | Nota | **La IA sugiere del catálogo; el dueño confirma** | Aprobados. |
| D.5 | Calcular el **contraste** del default | Tabla WCAG manual | **`validarContraste()` lo calcula automáticamente** | Texto/fondo ≥ 4.5:1, acento/fondo ≥ 3:1. |
| D.6 | Aprobar el default | Firma del dueño | Igual (la aprobación es siempre humana) | El default queda aprobado antes de la Fase 0. |

**Reversa:** no aplica (es diseño, no código).

**Nota:** la Fase 0 puede empezar **en paralelo** con la Fase D usando la paleta canónica como valor provisional, para no bloquear el trabajo técnico. Pero el **valor final** del default sale de la Fase D.

**Dependencia de la Vía B:** para usar el Extractor, deben estar cerradas la Fase 0 y la Fase 1 (motor mínimo) y construida la Fase 1.5 (el extractor). Si la Vía B no está lista, se usa la Vía A (manual). **Las dos vías producen el mismo resultado:** un `default.js` con los 11 tokens aprobados.

### Fase 0 — Decisión de mecanismo (BLOQUEANTE)

**Objetivo:** confirmar que los tokens Tailwind apuntan a variables CSS y que el POS se ve **idéntico** a hoy.

| # | Tarea | Archivo | Criterio de aceptación |
|---|---|---|---|
| 0.1 | Convertir los 6 tokens a `rgb(var(--x) / <alpha-value>)` | [`tailwind.config.js`](../../NUEVO-POS/apps/pos/tailwind.config.js:17) | El build compila sin error. |
| 0.2 | Declarar los 6 canales RGB en `:root` | [`index.css`](../../NUEVO-POS/apps/pos/src/index.css:12) | `--acento: 193 215 46` (canal, no hex). |
| 0.3 | Verificar visualmente los 24 componentes | — | **Cero cambios visuales.** El POS se ve igual que antes. |
| 0.4 | Verificar opacidades (`bg-acento/40`, `text-crema-ticket/50`) | — | Las opacidades siguen funcionando. |

**Reversa:** revertir los 2 archivos. El POS vuelve al estado actual.

**Riesgo:** si algún componente usa un hex suelto (no un token), ese punto no cambiará. **Mitigación:** buscar hex sueltos con `search_files` antes de cerrar la fase.

### Fase 1 — El motor mínimo

**Objetivo:** un motor que funcione de verdad contra un componente real.

| # | Tarea | Archivo | Criterio de aceptación |
|---|---|---|---|
| 1.1 | Crear `aplicarTema` | `packages/theme-engine/index.js` | Escribe los 6 canales en un contenedor. |
| 1.2 | Crear `validarContraste` | `packages/theme-engine/contraste.js` | Lanza error si texto/fondo < 4.5:1. |
| 1.3 | Crear `resolverTema` | `packages/theme-engine/index.js` | Devuelve el tema fusionado con la identidad. |
| 1.4 | Probar contra `ProductCard` | — | Cambiar `--acento` cambia el precio en pantalla. |
| 1.5 | Crear `normalizar_a_canales_rgb` | `packages/theme-engine/index.js` | Convierte `#7a8b3c` → `122 139 60`. (Necesario para Fase 1.5) |
| 1.6 | Crear `mapear_a_catalogo` | `packages/theme-engine/index.js` | Mapea fuente sugerida al catálogo cerrado de 5 fuentes. (Necesario para Fase 1.5) |

**Reversa:** el motor no se importa desde ningún componente todavía. Borrar la carpeta.

### Fase 1.5 — Extractor de Estética Visual (NUEVA — idea del dueño, 28 Sep 2026)

**Objetivo:** que el módulo de IA del ERP permita subir una imagen de referencia (Pinterest, Dribbble, captura propia) y extraiga automáticamente los 11 tokens de apariencia para crear un tema.

> **Especificación completa:** [`ESPECIFICACION_EXTRACTOR_ESTETICA_VISUAL.md`](./ESPECIFICACION_EXTRACTOR_ESTETICA_VISUAL.md)

**Los 2 problemas que resuelve:**

1. **Diseñar temas es difícil si no eres diseñador.** Con el extractor, subes una imagen de una UI que te guste → la IA extrae los valores → se calibran y se aplican.
2. **Si al admin no le gustan los 3 temas prearmados, no tiene salida.** Con el extractor, sube una imagen → genera un tema nuevo → lo asigna.

**Las 5 reglas del extractor (EX-01 a EX-05):**

- EX-01: La IA da dirección, no valores exactos. Se calibran con `validarContraste()`.
- EX-02: Las tipografías se mapean al catálogo cerrado de 5 fuentes.
- EX-03: El módulo sigue ofreciendo máximo 3 temas al cajero.
- EX-04: Ningún tema se aplica sin pasar `validarContraste()`.
- EX-05: El extractor requiere `AI_HABILITADA = true`. Si no hay IA, el proceso manual sigue disponible.

| # | Tarea | Archivo | Criterio de aceptación |
|---|---|---|---|
| 1.5.1 | Crear migración de `temas_generados` | `apps/api/migrations/versions/` | Tabla con UUID, timezone, versión (C-01, C-02, C-04). |
| 1.5.2 | Crear `POST /ia/extraer-estetica` | `apps/api/routers/ia.py` | Recibe imagen + módulo, devuelve 11 tokens + contraste en JSON. |
| 1.5.3 | Integrar con gateway de IA | `apps/api/routers/ia.py` | Usa `AI_LOCAL_URL`. Si IA no disponible → 503. |
| 1.5.4 | Crear pantalla `ExtractorEstetica.jsx` | `apps/ia/src/ExtractorEstetica.jsx` | Dropzone + resultado + ajustes + nombre + asignación. |
| 1.5.5 | Crear `ThemePreview.jsx` | `apps/ia/src/components/ThemePreview.jsx` | Miniatura del POS con tokens extraídos para vista previa. |
| 1.5.6 | Crear `POST /ia/guardar-tema` | `apps/api/routers/ia.py` | Escribe el `.js` del tema y actualiza `TEMA_DEL_MODULO`. |
| 1.5.7 | Test: extraer + validar contraste | `apps/api/tests/test_extractor.py` | Resultado pasa `validarContraste()` y tiene 11 tokens. |
| 1.5.8 | Test: IA no disponible → 503 limpio | `apps/api/tests/test_extractor.py` | `AI_HABILITADA=false` → 503; el módulo no se rompe. |

**Reversa:** borrar endpoint, pantalla y tabla. Los temas ya inyectados se quedan como archivos `.js` independientes. El POS sigue funcionando con temas estáticos.

**Dependencias:** Fase 0 cerrada + Fase 1 cerrada + gateway de IA operativo (`AI_HABILITADA=true`).

### Fase 2 — El tema default (el DISEÑADO en la Fase D)

**Objetivo:** el default vive en el módulo y su contenido es el que el dueño aprobó en la Fase D.

| # | Tarea | Archivo | Criterio de aceptación |
|---|---|---|---|
| 2.1 | Crear `default.js` con los valores de la Fase D | `apps/pos/src/theme/default.js` | Los 6 canales = los aprobados en la Fase D. |
| 2.2 | Crear el contrato | `apps/pos/src/theme/index.js` | `ofreceSelector: false`, `permitidos: ['default']`. |
| 2.3 | Aplicar el default al montar | [`main.jsx`](../../NUEVO-POS/apps/pos/src/main.jsx:15) | El POS se ve como el diseño aprobado. |
| 2.4 | Escribir los 6 tests de contrato | `apps/pos/src/theme/theme.test.js` | Los 6 pasan. |

**Reversa:** quitar el import en `main.jsx`. El POS vuelve a leer `index.css`.

**Nota:** si la Fase D aún no está aprobada, esta fase usa la paleta canónica como **valor provisional** y se marca como tal. No se cierra la Fase 2 hasta que el default esté aprobado.

### Fase 3 — Los 2 temas opcionales

**Objetivo:** dos temas más, con contraste calculado.

| # | Tarea | Archivo | Criterio de aceptación |
|---|---|---|---|
| 3.1 | Crear `nocturno.js` | `apps/pos/src/theme/nocturno.js` | Pasa `validarContraste`. |
| 3.2 | Crear `minimal.js` | `packages/theme-engine/temas-compartidos/minimal.js` | Pasa `validarContraste`. |
| 3.3 | Calcular y documentar el contraste de los 3 | — | Tabla de contraste en el doc. |
| 3.4 | Activar `ofreceSelector: true` | `apps/pos/src/theme/index.js` | `permitidos` tiene 3. |

**Reversa:** volver a `ofreceSelector: false`. Los 2 temas quedan en el repo pero no se ofrecen.

### Fase 4 — El selector (opcional, propiedad del módulo)

**Objetivo:** el módulo ofrece cambiar de tema y presenta 2 opciones más.

| # | Tarea | Archivo | Criterio de aceptación |
|---|---|---|---|
| 4.1 | Crear `ThemeSelector.jsx` | `apps/pos/src/components/ThemeSelector.jsx` | Muestra el default + 2 opciones. |
| 4.2 | Montarlo solo si `ofreceSelector` | [`RetailVisionPOS.jsx`](../../NUEVO-POS/apps/pos/src/RetailVisionPOS.jsx:28) | Si `false`, no aparece. |
| 4.3 | **Capa rápida (nivel 1):** persistir en `localStorage` | `apps/pos/src/theme/persistencia.js` | Al recargar, se mantiene el tema elegido. |
| 4.4 | Probar los 3 temas en los 3 modos | — | MOSTRADOR, COMPACTO y MÓVIL se ven bien. |
| 4.5 | **Capa durable (nivel 2):** endpoint `GET/PUT /pos/preferencias/{terminal_id}` | `apps/api/routers/` | Devuelve y guarda el tema de la terminal. |
| 4.6 | Crear la tabla `preferencias_terminal` | migración en `apps/api/` | Guarda `terminal_id` + `tema` + `actualizado_en`. |
| 4.7 | Sincronizar las 2 capas al arrancar y al cambiar | `apps/pos/src/theme/persistencia.js` | Si difieren, gana la base de datos y se re-sincroniza el navegador. |
| 4.8 | Test de persistencia (Opción B) | `apps/pos/src/theme/persistencia.test.js` | Limpiar `localStorage` → el tema se recupera de la terminal. |

**Decisión aplicada:** **Opción B (niveles 1 + 2)** — decidida por el dueño el 28 Sep 2026 (ver §0.6).

**Reversa:** quitar el `<ThemeSelector />`. El POS vuelve al default. La tabla `preferencias_terminal` queda pero no se consulta.

### Fase 5 — El enfoque de configuración general del ERP (referencia futura)

**Objetivo:** dejar preparado el contrato para cuando Visión General exista.

| # | Tarea | Archivo | Criterio de aceptación |
|---|---|---|---|
| 5.1 | Definir `GET /branding` (lectura) | `apps/api/routers/` | Devuelve logo, nombre, colores, tipografía. |
| 5.2 | Definir `PUT /branding` (escritura) | `apps/api/routers/` | **Solo Visión General** lo llama. |
| 5.3 | Definir el permiso de edición | — | Solo quien tenga el permiso edita la identidad. |
| 5.4 | Documentar la propagación | — | Cómo se enteran los módulos abiertos del cambio. |

**Nota:** esta fase **no toca el ERP**. Es el contrato del POS nuevo, preparado para cuando Visión General se construya.

---

## 7. Tabla de contraste de los 3 temas (a calcular en Fase D y Fase 3)

| Tema | Texto/Fondo | Acento/Fondo | ¿Pasa AA? |
|---|---|---|---|
| **Default** (a diseñar en Fase D) | ⏳ (sale de la Fase D) | ⏳ (sale de la Fase D) | ⏳ (calcular tras el diseño) |
| **Opción 1** | ⏳ (calcular en Fase 3) | ⏳ (calcular en Fase 3) | ⏳ (calcular) |
| **Opción 2** | ⏳ (calcular en Fase 3) | ⏳ (calcular en Fase 3) | ⏳ (calcular) |

**Regla:** ningún tema se activa sin la tabla calculada. Afirmar "pasa el contraste" sin calcularlo es lo que la V4 hacía mal. **El default no se hereda: se diseña (Fase D) y luego se calcula su contraste.**

**Nota sobre la paleta canónica:** los valores actuales (`#fdfbf7` / `#0a0a0a`, acento `#c1d72e`) son el **piso técnico provisional**. Si el dueño diseña un default distinto en la Fase D, la tabla se recalcula con los valores nuevos.

---

## 8. Lo que este plan NO hace

- **No toca el ERP.** Ni el instalado ni el que corre.
- **No toca el POS que corre.** Solo el POS nuevo.
- **No asume el default visual.** El default **se diseña** en la Fase D; la paleta canónica es solo el piso técnico provisional.
- **No reescribe los 24 componentes.** Solo cambia cómo se resuelven los tokens.
- **No crea temas para 12 módulos.** Solo el POS. Los demás, cuando existan.
- **No pone el selector en Visión General.** El selector es del módulo.

---

## 9. Estado

- [x] Revisión autocrítica de la V4 contra el código real.
- [x] Las 3 decisiones resueltas (mecanismo, vocabulario, default).
- [x] Reconciliación con la puerta F5.
- [x] El contrato con selector opcional.
- [x] Las 6 fases con archivos, criterios y reversa.
- [x] Corrección: el default visual **no está diseñado**; se añadió la Fase D.
- [x] **Fase 1.5 — Extractor de Estética Visual integrado al plan** (idea del dueño, 28 Sep 2026). Especificación en [`ESPECIFICACION_EXTRACTOR_ESTETICA_VISUAL.md`](./ESPECIFICACION_EXTRACTOR_ESTETICA_VISUAL.md).
- [x] Fase D actualizada con **dos vías** (manual y extractor). Ambas producen el mismo resultado.
- [ ] **Fase D — el dueño ve y diseña el default visual del POS** (Vía A manual o Vía B extractor).
- [ ] Aprobación del dueño para empezar la Fase 0.
- [ ] Ejecución de la Fase 0 (bloqueante).
- [ ] Ejecución de la Fase 1 (motor mínimo).
- [ ] Ejecución de la Fase 1.5 (extractor de estética visual).

---

**Fin del plan de implementación.**
