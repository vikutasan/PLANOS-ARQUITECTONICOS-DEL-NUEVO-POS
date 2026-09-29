# PLAN DE ABORDAJE — FASE 7 POR PARTES
## Selector de Temas (UI) + Voz + Visión IA — todo por contrato con el Centro de IA

> **Versión:** 1.1
> **Fecha:** 2026-09-29 (v1.1: ajuste por la aclaración del hardware de visión — DT-08)
> **Autor:** Arquitecto del POS nuevo
> **Estado:** APROBADO — F7.0, F7.1 y F7.2 ejecutadas; F7.3 ajustada por DT-08
> **Referencias:**
> - [`PLAN_MAESTRO_DEFINITIVO_POS.md`](../PLAN_MAESTRO_DEFINITIVO_POS.md) §Fase 7 + §8 + §8.1
> - [`DIRECTRICES_TRANSVERSALES_DEL_ERP.md`](../DIRECTRICES_TRANSVERSALES_DEL_ERP.md) **DT-07** (IA) + **DT-08** (visión cenital) + DT-06 (Vista General)
> - [`PROMPT_DEL_ARQUITECTO_DEL_NUEVO_POS.md`](../06-prompt-del-arquitecto/PROMPT_DEL_ARQUITECTO_DEL_NUEVO_POS.md) §7.4 (estándares) + §10.8 (E-01 a E-16)
> - [`PLAN_DE_ABORDAJE_FASE_6_POR_PARTES.md`](PLAN_DE_ABORDAJE_FASE_6_POR_PARTES.md) (precedente de método)
> - POS viejo: `apps/pos/hooks/useVoiceCart.js` (409 líneas), `apps/pos/utils/voiceCartMapper.js` (236 líneas), `apps/pos/VisionScanner.jsx` (231 líneas), `apps/ai/utils/aiCenterConstants.js`
> - POS viejo: `ESPECIFICACIONES DEL PROYECTO/DOCUMENTACION_CENTRO_IA.md` (v27.4, 1672 líneas)

---

## §1. ACLARACIÓN DE ALCANCE — TRES ENTREGABLES, UNO DE ELLOS NO ES DEL POS

La Fase 7 agrupa bajo un mismo título cosas que **pertenecen a módulos distintos**. Confundirlas fue el primer riesgo detectado en la autocrítica (§3, D-1). Este plan las separa explícitamente en **tres** entregables:

| | **A — Selector de Temas** | **B — Voz** | **C — Visión** |
|---|---|---|---|
| **De quién es** | ✅ **Del POS** (UI propia) | ⚠️ **Del Centro de IA** (el POS solo consume) | ⚠️ **Del Centro de IA** (el POS solo consume) |
| **Qué construye el POS** | El selector visual + el cableado del motor de temas | El panel + el hook que **llama al contrato** | El visor de cámara que **llama al contrato** |
| **Qué NO construye el POS** | — | El motor Whisper, el NLU Ollama | El motor YOLO, el entrenamiento |
| **Dependencia nueva** | Cero (el motor ya existe en `packages/`) | Cero (solo `fetch` al contrato) | Cero (solo `fetch` al contrato) |
| **Riesgo si el proveedor no existe** | Ninguno (el motor es local) | Degrada a modo manual | Degrada a modo manual |
| **Sub-fase** | **F7.1** | **F7.2** | **F7.3** |

**Por qué A es del POS y B/C no (decisión arquitectónica, no preferencia):**

- El **motor de temas** ya existe y vive en `packages/theme-engine/` — es infraestructura compartida, no propiedad de ningún módulo. El POS **sí** es dueño de su selector (la UI que el operador toca). Ver [`packages/theme-engine/index.js`](../../NUEVO-POS/packages/theme-engine/index.js:1).
- La **voz y la visión** son capacidades del **Centro de IA** (DT-07). El POS **nunca** importa `whisper`, `ultralytics` ni `torch`. Solo declara contratos y los consume. Si el Centro de IA no está construido, el POS sigue vendiendo.

> **Nota sobre el estado del Centro de IA:** hoy solo existe como **documentación** del ERP viejo (`DOCUMENTACION_CENTRO_IA.md`, v27.4). No hay código en `apps/ai/` de este repo. Eso es exactamente el escenario que el dueño describió ("aún no está construido"). Por eso **F7.0 declara los contratos primero** y F7.2/F7.3 trabajan contra un **stub** hasta que el Centro de IA real exista.

---

## §2. DIAGNÓSTICO — EVIDENCIA, NO OPINIÓN (E-14)

### §2.1 Qué dice el plano

**§Fase 7** declara 7 archivos:
- `VoiceCartPanel.jsx` — panel de voz
- `useVoiceCart.js` — hook de voz (consume `ia.transcribir_voz` + `ia.interpretar_intencion`)
- `voiceCartMapper.js` — mapeo de intents a productos
- `VisionVisor.jsx` — visor de cámara IA (consume contrato 17)
- `useVision.js` — hook de visión
- `ProgramacionPedidoModal.jsx` — pedidos programados
- `ThemeSelector.jsx` — ✨ selector visual de temas

**§8** ya declara la decisión: *"IA (voz/visión): consumir el Centro de IA por contrato (DT-07)"*.

### §2.2 Qué existe HOY en el código (NUEVO-POS)

| Elemento | Estado real | Evidencia |
|---|---|---|
| Motor de temas | **YA EXISTE** | [`packages/theme-engine/index.js`](../../NUEVO-POS/packages/theme-engine/index.js:1) — 300 líneas, 5 funciones públicas |
| Contrato `TEMA_DEL_MODULO` del POS | **YA EXISTE** | [`apps/pos/src/theme/index.js`](../../NUEVO-POS/apps/pos/src/theme/index.js:23) — 3 temas: default, nocturno, minimal |
| Temas concretos | **YA EXISTEN** | `apps/pos/src/theme/{default,nocturno,minimal}.js` |
| **Cableado del motor al POS** | **NO EXISTE** | `search_files` de `resolverTema\|aplicarTema` en `apps/pos/src` → **0 llamadas reales** (solo el test y el comentario del contrato) |
| `ThemeSelector.jsx` | **NO EXISTE** | `search_files` de `ThemeSelector` en `apps/` → 0 resultados |
| Código de voz en el POS nuevo | **NO EXISTE** | `search_files` de `voz\|voice\|Whisper\|getUserMedia` en `apps/pos/src` → 0 resultados |
| Código de visión en el POS nuevo | **NO EXISTE** | `search_files` de `vision\|camera` en `apps/pos/src` → 0 resultados |
| Contrato 17 (`vision.reconocer_producto`) | **YA DECLARADO** | [`registry.py`](../../NUEVO-POS/apps/api/contracts/registry.py:400) — proveedor `"Centro de IA"`, `estado_hoy="Deuda"` |
| Contratos de voz/NLU | **NO DECLARADOS** | `registry.py` solo tiene 23 contratos; no hay `ia.transcribir_voz` ni `ia.interpretar_intencion` |
| RN-71..74 (reglas de visión) | **YA DECLARADAS** | [`rules/registry.py`](../../NUEVO-POS/apps/api/rules/registry.py:577) — motor ORB, umbral 0.35, etiquetado por SKU, visión asistiva |

### §2.3 Qué existe en el POS VIEJO (referencia a portar)

| Activo del viejo | Líneas | Qué hace | Veredicto |
|---|---|---|---|
| `apps/pos/hooks/useVoiceCart.js` | 409 | Pipeline completo: MediaRecorder → `POST /ai/voice/transcribe` (Whisper) → `POST /ai/voice/parse-intent` (Ollama) → propuesta editable. Captura continua con `AudioContext` + `AnalyserNode` (RMS). | **Portar la lógica, corregir el contrato** |
| `apps/pos/utils/voiceCartMapper.js` | 236 | Mapea intents a propuestas de carrito. Allowlist: **solo `AGREGAR_ITEM`**. Umbral de confianza 0.7. | **Portar tal cual** (es lógica pura, testeable) |
| `apps/pos/VisionScanner.jsx` | 231 | Cámara + análisis **híbrido: servidor local → Gemini (nube)**. Usa `@google/generative-ai`. | **Portar la cámara, ELIMINAR Gemini** |
| `apps/ai/utils/aiCenterConstants.js` | — | `VOZ_CONFIG` (umbral RMS, silencio, tiempos). Fuente única compartida POS ↔ Centro de IA. | **Portar a `apps/pos/src/config/`** |

### §2.4 Hallazgos que condicionan el plan

**H-1 — El motor de temas existe pero NO está cableado. El POS hoy no aplica ningún tema.**
Evidencia: `resolverTema`/`aplicarTema` no se llaman desde ningún componente. El contrato `TEMA_DEL_MODULO` y los 3 temas existen, pero nadie los usa. **F7.1 no es "crear el selector": es "cablear el motor + crear el selector".** Esto es más grande de lo que el Plan Maestro sugiere.

**H-2 — El POS viejo llama a `/ai/voice/transcribe` y `/ai/voice/parse-intent` SIN contrato declarado.**
Evidencia: [`useVoiceCart.js`](apps/pos/hooks/useVoiceCart.js:17) líneas 17-18. Esos dos endpoints no están en el registro de contratos. Consumirlos así viola la Regla Dura **A-02** (frontera por contratos). **F7.0 debe declararlos ANTES de portar el hook.**

**H-3 — El POS viejo usa Gemini (nube) para visión. Eso viola DT-07.**
Evidencia: [`VisionScanner.jsx`](apps/pos/VisionScanner.jsx:2) importa `@google/generative-ai` y usa `gemini-2.0-flash`. DT-07 dice que el motor de IA es **local** (`ai-local/`) y que ningún módulo importa dependencias de IA. **La visión del POS nuevo NO porta Gemini.** Solo porta la captura de cámara y consume el contrato 17 (que apunta al Centro de IA local).

**H-4 — El contrato 17 ya existe pero su proveedor es "Deuda".**
Evidencia: [`registry.py`](../../NUEVO-POS/apps/api/contracts/registry.py:400) — `estado_hoy="Deuda"`. El contrato está declarado (bueno) pero no implementado (esperado: el Centro de IA no existe). F7.3 consume un contrato que devolverá 503 hasta que el Centro de IA exista. **La degradación elegante es obligatoria, no opcional.**

**H-5 — La regla de oro del POS viejo es "la IA PROPONE, el operador CONFIRMA".**
Evidencia: [`useVoiceCart.js`](apps/pos/hooks/useVoiceCart.js:29) línea 29 y [`voiceCartMapper.js`](apps/pos/utils/voiceCartMapper.js:8) línea 8. El hook **nunca** toca el carrito; solo produce una `propuesta`. Esta regla es **no negociable** y debe preservarse en el POS nuevo.

**H-6 — El POS viejo restringe la voz a UNA sola intención: `AGREGAR_ITEM`.**
Evidencia: [`voiceCartMapper.js`](apps/pos/utils/voiceCartMapper.js:47) línea 47 — `POS_ALLOWED_INTENTS = new Set([AGREGAR_ITEM])`. Cobrar, cancelar y quitar por voz están **prohibidos por diseño** (son acciones destructivas/fiscales). Debe preservarse.

**H-7 — `ProgramacionPedidoModal.jsx` (pedidos programados) NO tiene precedente claro en el POS viejo.**
El Plan Maestro lo lista en Fase 7, pero no hay un componente equivalente identificado. **Se propone DIFERIRLO** a una fase posterior (ver §3, D-4), porque no es voz ni visión ni temas, y su alcance no está definido.

---

## §3. AUTOCRÍTICA — CONTRA EL CÓDIGO REAL

Antes de escribir la v1.0 se hallaron 4 defectos (D-1 a D-4). Se listan aquí.

**D-1 (real) — Se trataba "voz" y "visión" como si fueran del POS.**
Falso: son del Centro de IA (DT-07). El POS solo construye los puntos de contacto.
**Corrección:** §1 separa los 3 entregables y marca B/C como "del Centro de IA".

**D-2 (real) — Se asumía que el motor de temas ya estaba cableado.**
Falso: `resolverTema`/`aplicarTema` no se llaman desde ningún componente (H-1). El POS hoy no aplica ningún tema.
**Corrección:** F7.1 incluye el **cableado** del motor, no solo el selector.

**D-3 (real) — Se iba a portar `VisionScanner.jsx` tal cual, con Gemini.**
Falso: Gemini es una dependencia de nube que viola DT-07 (H-3).
**Corrección:** F7.3 porta **solo la captura de cámara** y consume el contrato 17. Cero dependencias de nube.

**D-4 (real) — `ProgramacionPedidoModal.jsx` no tiene alcance definido.**
El Plan Maestro lo lista pero no hay precedente claro (H-7). Meterlo en Fase 7 diluye el foco.
**Corrección:** se **difiere** explícitamente a una fase posterior (§12.3). Fase 7 se enfoca en temas + voz + visión.

---

## §4. LA FRONTERA POR CONTRATOS — EL CORAZÓN DE LA FASE

Esta fase es la primera que **cruza la frontera hacia otro módulo del ERP**. Por eso F7.0 es obligatoria y va primero.

```
┌─────────────────────────────────────────────────────────────────┐
│  apps/pos/  (el POS — lo que construimos)                        │
│                                                                  │
│   ThemeSelector.jsx ──► packages/theme-engine/  (local, ya existe)│
│                                                                  │
│   useVoiceCart.js ──┐                                            │
│   useVision.js ─────┼──► fetch a CONTRATOS ──┐                   │
│                     │                        │                   │
└─────────────────────┼────────────────────────┼───────────────────┘
                      │                        │
                      │  (frontera A-02)       │
                      ▼                        ▼
        ┌──────────────────────────────────────────────┐
        │  apps/api/  (el Gateway)                      │
        │  · ia.transcribir_voz      → 503 si no hay IA │
        │  · ia.interpretar_intencion→ 503 si no hay IA │
        │  · vision.reconocer_producto (contrato 17)    │
        └──────────────────────┬───────────────────────┘
                               │
                               ▼
        ┌──────────────────────────────────────────────┐
        │  Centro de IA  (apps/ai/ — AÚN NO EXISTE)     │
        │  · Whisper (voz) · Ollama (NLU) · YOLO (visión)│
        └──────────────────────────────────────────────┘
```

**Regla de oro (DT-07):** un fallo del motor de IA **NUNCA** bloquea una venta. El Gateway traduce cualquier fallo a **503 `IA_NO_DISPONIBLE`**. El POS degrada a modo manual.

---

## §5. SUB-FASE 7.0 — Declarar la frontera POS ↔ Centro de IA (PRIMERO, sin UI)

### §5.1 Qué construye

Tres contratos en [`apps/api/contracts/registry.py`](../../NUEVO-POS/apps/api/contracts/registry.py:1) + su test de puerta. **Cero UI.**

| Contrato | Consumidor | Proveedor | Operación |
|---|---|---|---|
| `vision.reconocer_producto` (17) | POS | Centro de IA | `POST /vision/predict` — **ya existe**, solo se confirma |
| `ia.transcribir_voz` (24) | POS | Centro de IA | `POST /ai/voice/transcribe` |
| `ia.interpretar_intencion` (25) | POS | Centro de IA | `POST /ai/voice/parse-intent` |

### §5.2 Decisiones de diseño

- **El registro pasa de 23 a 25 contratos.** El test `test_criterio2_hay_exactamente_23_contratos` debe actualizarse a 25 (es un cambio esperado y documentado).
- **La degradación se declara en el contrato:** ambos contratos de voz documentan `503 IA_NO_DISPONIBLE` en su campo `errores`.
- **El contrato 17 ya está correcto** (proveedor `"Centro de IA"`); solo se verifica.
- **No se implementa el endpoint.** F7.0 es **solo declaración** (el registro es la fuente de verdad). La implementación es del Centro de IA.

### §5.3 Archivos

| Archivo | Acción |
|---|---|
| `apps/api/contracts/registry.py` | MODIFICAR (añadir contratos 24 y 25) |
| `apps/api/tests/test_f2_frontera.py` | MODIFICAR (23 → 25 contratos) |
| `apps/api/tests/test_f7_contratos_ia.py` | CREAR (gate) |

### §5.4 Criterios de la puerta (gate)

1. El registro tiene **exactamente 25 contratos**.
2. `ia.transcribir_voz` está declarado con proveedor `"Centro de IA"`.
3. `ia.interpretar_intencion` está declarado con proveedor `"Centro de IA"`.
4. Ambos contratos documentan `503 IA_NO_DISPONIBLE` en `errores`.
5. El contrato 17 sigue con proveedor `"Centro de IA"`.
6. Ningún contrato expone una tabla (criterio 3 de F2 sigue verde).
7. El POS sigue siendo proveedor en sus propios contratos (criterio 3 de F2).

### §5.5 Gate de cierre

```
cd apps/api && pytest tests/test_f7_contratos_ia.py   → verde
npm run test                                           → verde
node scripts/guards.mjs                                → 7/7 verde
```

---

## §6. SUB-FASE 7.1 — Cableado del motor de temas + `ThemeSelector.jsx`

### §6.1 Qué construye

- `apps/pos/src/hooks/useTheme.js` — hook que resuelve y aplica el tema (cablea el motor).
- `apps/pos/src/components/ThemeSelector.jsx` — selector visual (interfaz nueva).
- Cableado en `App.jsx` o `RetailVisionPOS.jsx` para aplicar el tema al montar.

### §6.2 Decisiones de diseño

- **El motor NO se toca.** `packages/theme-engine/` ya tiene las 5 funciones. F7.1 solo las **llama**.
- **Persistencia:** la elección del usuario se guarda en `localStorage` (clave `pos.tema`), igual que `pos.ordenTerminales` (F6.5).
- **`resolverTema(modulo, eleccion, identidad)`:** se llama con `TEMA_DEL_MODULO`, la elección guardada, e `identidad=null` (Vista General aún no existe; DT-06).
- **`aplicarTema(tema, contenedor)`:** se aplica al contenedor raíz del POS, no a `:root` (evita contaminación cruzada entre módulos).
- **Degradación:** si un tema no carga, `resolverTema` ya cae al default del módulo (línea 128-133 del motor). El hook no necesita manejar eso.
- **El selector respeta R-03 (3 modos) y R-04 (target ≥44px).**
- **`ofreceSelector: true`** ya está en el contrato; el selector se muestra solo si es true.

### §6.3 Archivos

| Archivo | Acción |
|---|---|
| `apps/pos/src/hooks/useTheme.js` | CREAR |
| `apps/pos/src/components/ThemeSelector.jsx` | CREAR |
| `apps/pos/src/App.jsx` | MODIFICAR (aplicar tema al montar) |
| `apps/pos/src/hooks/useTheme.f7_1.test.jsx` | CREAR (gate) |
| `apps/pos/src/components/ThemeSelector.f7_1.test.jsx` | CREAR (gate) |

### §6.4 Criterios de la puerta (gate)

1. `useTheme` llama a `resolverTema` con el `TEMA_DEL_MODULO` del POS.
2. `useTheme` llama a `aplicarTema` con el tema resuelto.
3. La elección del usuario persiste en `localStorage` (`pos.tema`).
4. Al montar, el POS aplica el tema guardado (o el default).
5. `ThemeSelector` lista los 3 temas permitidos (`default`, `nocturno`, `minimal`).
6. `ThemeSelector` NO se muestra si `ofreceSelector` es false.
7. Cambiar de tema aplica el nuevo tema sin recargar.
8. El selector respeta R-03 (visible en los 3 modos) y R-04 (target ≥44px).
9. Si `localStorage` tiene un tema inválido, se cae al default (no rompe).

### §6.5 Gate de cierre

```
npx vitest run src/hooks/useTheme.f7_1.test.jsx          → verde
npx vitest run src/components/ThemeSelector.f7_1.test.jsx → verde
npm run test                                              → verde
node scripts/guards.mjs                                   → 7/7 verde
```

---

## §7. SUB-FASE 7.2 — Voz (`useVoiceCart.js` + `VoiceCartPanel.jsx` + `voiceCartMapper.js`)

### §7.1 Qué construye

- `apps/pos/src/utils/voiceCartMapper.js` — porta el mapper del viejo (lógica pura).
- `apps/pos/src/config/voz.js` — porta `VOZ_CONFIG` del viejo.
- `apps/pos/src/hooks/useVoiceCart.js` — hook que consume los contratos 24 y 25.
- `apps/pos/src/components/VoiceCartPanel.jsx` — panel de voz (propuesta editable).

### §7.2 Decisiones de diseño

- **Consume los contratos 24 y 25** (declarados en F7.0), NO endpoints sueltos. Esto corrige H-2.
- **Regla de oro (H-5):** el hook **nunca** toca el carrito. Solo produce una `propuesta` que la UI muestra para que el operador confirme.
- **Allowlist (H-6):** solo `AGREGAR_ITEM`. Cobrar/cancelar/quitar por voz están prohibidos.
- **Umbral de confianza 0.7:** por debajo, la propuesta se marca `revisar: true` (resaltado ámbar).
- **Degradación elegante:** si el navegador no soporta audio o el contrato devuelve 503, `disponible=false` y el POS sigue en modo manual.
- **El mapper es lógica pura** → se porta con su test (A-01: cada regla con su test).
- **`VOZ_CONFIG` se centraliza** en `apps/pos/src/config/voz.js` (el POS nuevo no importa de `apps/ai/`; esa carpeta no existe aquí).

### §7.3 Archivos

| Archivo | Acción |
|---|---|
| `apps/pos/src/utils/voiceCartMapper.js` | CREAR (porta del viejo) |
| `apps/pos/src/utils/voiceCartMapper.f7_2.test.jsx` | CREAR (gate) |
| `apps/pos/src/config/voz.js` | CREAR (porta `VOZ_CONFIG`) |
| `apps/pos/src/hooks/useVoiceCart.js` | CREAR |
| `apps/pos/src/hooks/useVoiceCart.f7_2.test.jsx` | CREAR (gate) |
| `apps/pos/src/components/VoiceCartPanel.jsx` | CREAR |
| `apps/pos/src/components/VoiceCartPanel.f7_2.test.jsx` | CREAR (gate) |

### §7.4 Criterios de la puerta (gate)

1. `voiceCartMapper` resuelve un producto por SKU exacto.
2. `voiceCartMapper` resuelve por nombre normalizado (sin acentos).
3. `voiceCartMapper` marca `resuelto: false` si no encuentra el producto.
4. `voiceCartMapper` marca `revisar: true` si la confianza < 0.7.
5. `voiceCartMapper` degrada cualquier intent fuera de la allowlist a `DESCONOCIDA`.
6. `useVoiceCart` llama al contrato `ia.transcribir_voz` (no a un endpoint suelto).
7. `useVoiceCart` llama al contrato `ia.interpretar_intencion`.
8. `useVoiceCart` expone `disponible=false` si el contrato devuelve 503.
9. `useVoiceCart` **NUNCA** modifica el carrito (solo produce `propuesta`).
10. `VoiceCartPanel` muestra la propuesta y exige confirmación explícita.
11. `VoiceCartPanel` respeta R-03 (3 modos) y R-04 (target ≥44px).

### §7.5 Gate de cierre

```
npx vitest run src/utils/voiceCartMapper.f7_2.test.jsx    → verde
npx vitest run src/hooks/useVoiceCart.f7_2.test.jsx       → verde
npx vitest run src/components/VoiceCartPanel.f7_2.test.jsx → verde
npm run test                                               → verde
node scripts/guards.mjs                                    → 7/7 verde
```

---

## §8. SUB-FASE 7.3 — Visión (`useVision.js` + `VisionVisor.jsx`)

> [!IMPORTANT]
> **Ajuste por la aclaración del hardware (29 Sep 2026).** El dueño confirmó que la visión opera
> sobre una **cámara cenital con iluminación dedicada** (ver **DT-08**). Esto cambia el diseño de
> esta sub-fase en tres puntos: (1) el umbral 0.35 es de **calibración cenital**, configurable;
> (2) el contrato 17 declara el **`modo_captura`**; (3) el visor es un **flujo persistente**
> ("escáner de charola"), no una captura bajo demanda.

### §8.1 Qué construye

- `apps/pos/src/hooks/useVision.js` — hook que captura frames y consume el contrato 17.
- `apps/pos/src/components/VisionVisor.jsx` — visor de cámara + sugerencias.

### §8.2 Decisiones de diseño

- **Consume el contrato 17** (`vision.reconocer_producto`), NO Gemini (corrige H-3).
- **Cero dependencias de nube.** Solo `getUserMedia` (nativo del navegador).
- **Visión asistiva (RN-74):** la visión **sugiere**, nunca bloquea la venta manual.
- **Umbral 0.35 (RN-72) = calibración cenital, configurable (DT-08):** por debajo, la detección se descarta. El valor 0.35 es el calibrado para el montaje cenital con iluminación controlada; el hook lo lee de configuración (con 0.35 como valor por defecto), no lo hardcodea.
- **`modo_captura` en el contrato 17 (DT-08):** el hook envía `modo_captura: 'cenital'` en la llamada al contrato. El contrato distingue `cenital` de `manual`.
- **Flujo persistente, no bajo demanda (DT-08):** el visor cenital permanece abierto durante la venta (modo "escáner de charola"). El operador coloca los productos y el sistema los reconoce sin apuntar.
- **Etiquetado por SKU (RN-73):** la detección se resuelve contra el catálogo por SKU.
- **Degradación elegante:** si el contrato devuelve 503 (Centro de IA no disponible), el visor muestra "IA no disponible" y el POS sigue en modo manual.
- **La cámara es opcional:** el visor se abre cuando el operador lo decide, pero una vez abierto **permanece** (no se cierra por producto).

### §8.3 Archivos

| Archivo | Acción |
|---|---|
| `apps/pos/src/hooks/useVision.js` | CREAR |
| `apps/pos/src/hooks/useVision.f7_3.test.jsx` | CREAR (gate) |
| `apps/pos/src/components/VisionVisor.jsx` | CREAR |
| `apps/pos/src/components/VisionVisor.f7_3.test.jsx` | CREAR (gate) |

### §8.4 Criterios de la puerta (gate)

1. `useVision` llama al contrato 17 (`vision.reconocer_producto`).
2. `useVision` NO importa `@google/generative-ai` ni ninguna dependencia de nube.
3. `useVision` expone `disponible=false` si el contrato devuelve 503.
4. `useVision` descarta detecciones con confianza < 0.35 (RN-72).
5. `useVision` resuelve la detección contra el catálogo por SKU (RN-73).
6. `VisionVisor` sugiere productos pero **NO** los agrega al carrito automáticamente (RN-74).
7. `VisionVisor` muestra "IA no disponible" si el contrato falla.
8. `VisionVisor` respeta R-03 (3 modos) y R-04 (target ≥44px).
9. `useVision` envía `modo_captura: 'cenital'` en la llamada al contrato 17 (DT-08).
10. `useVision` lee el umbral de configuración (no lo hardcodea); 0.35 es el valor por defecto (DT-08).
11. `VisionVisor` mantiene el visor abierto entre detecciones (flujo persistente, no bajo demanda) (DT-08).

### §8.5 Gate de cierre

```
npx vitest run src/hooks/useVision.f7_3.test.jsx       → verde
npx vitest run src/components/VisionVisor.f7_3.test.jsx → verde
npm run test                                            → verde
node scripts/guards.mjs                                 → 7/7 verde
```

---

## §9. SUB-FASE 7.4 — Cierre de Fase 7

### §9.1 Qué construye

`docs/05-plan-de-construccion/FICHA_F7_CIERRE.md` — la ficha de cierre que consolida las 4 sub-fases.

### §9.2 Contenido de la ficha

1. **Resumen de las 4 sub-fases** (F7.0 a F7.3) con su commit.
2. **Tabla de paridad con el POS viejo:** qué se portó igual (mapper de voz), qué se mejoró (visión sin Gemini), qué es nuevo (selector de temas cableado).
3. **Estado de la frontera POS ↔ Centro de IA:** qué contratos se declararon, qué falta del lado del Centro de IA.
4. **Los 4 defectos (D-1 a D-4) y cómo se corrigieron.**
5. **Evidencia de los gates:** salida de `npm run test` y `node scripts/guards.mjs`.

### §9.3 Archivos

| Archivo | Acción |
|---|---|
| `docs/05-plan-de-construccion/FICHA_F7_CIERRE.md` | CREAR |

### §9.4 Gate de cierre

```
npm run test                → verde (suite completa)
node scripts/guards.mjs     → 7/7 verde
```

---

## §10. RIESGOS Y MITIGACIONES

| Riesgo | Probabilidad | Impacto | Mitigación |
|---|---|---|---|
| El Centro de IA no existe → los contratos devuelven 503 | **Alta** | Bajo | Degradación elegante obligatoria (F7.2/F7.3). El POS sigue vendiendo. |
| El test de F2 se rompe al pasar de 23 a 25 contratos | Alta | Bajo | Se actualiza el test en F7.0 (cambio esperado y documentado). |
| El motor de temas contamina otros módulos | Baja | Medio | `aplicarTema` se aplica al contenedor del POS, no a `:root`. |
| `getUserMedia` requiere HTTPS | Media | Medio | Documentar en la ficha; el visor degrada si no hay permiso. |
| El mapper de voz no resuelve nombres con acentos | Media | Bajo | El mapper normaliza (NFD + strip diacríticos); el gate lo verifica. |
| Se cuela una dependencia de nube (Gemini) en visión | Baja | **Alto** | El gate de F7.3 verifica que NO se importe `@google/generative-ai`. |
| `ProgramacionPedidoModal.jsx` se cuela sin alcance | Media | Bajo | Se difiere explícitamente (§12.3). |
| El umbral 0.35 se trata como universal y no se puede calibrar | Media | Medio | **DT-08:** el umbral es de calibración cenital y configurable; el gate de F7.3 verifica que se lea de configuración (criterio 10). |
| El visor se implementa como captura bajo demanda (no persistente) | Media | Bajo | **DT-08:** el gate de F7.3 verifica el flujo persistente (criterio 11). |
| El contrato 17 no declara el `modo_captura` | Media | Bajo | **DT-08:** el gate de F7.3 verifica que el hook envíe `modo_captura: 'cenital'` (criterio 9). |

---

## §11. ORDEN DE EJECUCIÓN Y CRITERIO DE APROBACIÓN

### §11.1 Orden

```
F7.0  contratos 24 y 25 + test        (frontera, sin UI)
  ↓
F7.1  useTheme + ThemeSelector        (temas, menor riesgo)
  ↓
F7.2  voiceCartMapper + useVoiceCart  (voz, consume contratos 24/25)
  ↓
F7.3  useVision + VisionVisor         (visión, consume contrato 17)
  ↓
F7.4  FICHA_F7_CIERRE.md              (cierre de fase)
```

Cada sub-fase cierra con: gate en verde + suite completa + guards 7/7 + ficha + commit + push. **No se abre la siguiente sin cerrar la anterior.**

### §11.2 Criterio de aprobación

Este plan se considera aprobado cuando el dueño confirma:

1. Que la **separación en 3 entregables** (§1) refleja la aclaración arquitectónica (temas = del POS; voz/visión = del Centro de IA).
2. Que **F7.0 va primero** (declarar contratos antes de tocar UI).
3. Que el **orden de ejecución** (§11.1) es el correcto.
4. Que **diferir `ProgramacionPedidoModal.jsx`** (§12.3) es aceptable.

### §11.3 Lo que este plan NO hace (delimitación explícita)

- **No** construye el Centro de IA. Eso es otro módulo del ERP, con su propio plan.
- **No** implementa los motores de IA (Whisper, Ollama, YOLO). Solo declara contratos y los consume.
- **No** porta Gemini ni ninguna dependencia de nube (corrige H-3).
- **No** implementa `ProgramacionPedidoModal.jsx` (diferido, §12.3).
- **No** toca el POS viejo (`ERP-R-DE-RICO`). Solo lo lee como referencia.
- **No** añade dependencias nuevas. El motor de temas ya existe; la voz y la visión usan `fetch` nativo.
- **No** modifica el motor de temas (`packages/theme-engine/`). Solo lo cablea.

---

## §12. NOTAS DE CIERRE

### §12.1 Por qué F7.0 es la sub-fase más importante

Es la única que **no produce UI** y, sin embargo, es la que hace posible todo lo demás. Sin F7.0, el POS hablaría con el Centro de IA por **convención implícita** (como el POS viejo, H-2), violando la Regla Dura A-02. Con F7.0, cuando el Centro de IA se construya, el POS **no se toca**: solo cambia quién implementa el contrato.

### §12.2 El patrón de degradación (DT-07) aplicado a las 3 sub-fases

| Sub-fase | Si el proveedor falla | Qué ve el operador |
|---|---|---|
| F7.1 (temas) | El tema no carga | El motor cae al default del módulo (ya implementado) |
| F7.2 (voz) | Contrato 24/25 devuelve 503 | `disponible=false`; el POS sigue en modo manual |
| F7.3 (visión) | Contrato 17 devuelve 503 | "IA no disponible"; el POS sigue en modo manual |

En los tres casos, **la venta nunca se bloquea**. Esa es la regla de oro de DT-07.

### §12.3 Lo diferido (y por qué)

**`ProgramacionPedidoModal.jsx` (pedidos programados)** se difiere a una fase posterior. Razones:
1. No es voz, ni visión, ni temas — no encaja en el hilo de Fase 7.
2. No tiene precedente claro en el POS viejo (H-7): su alcance no está definido.
3. Meterlo diluye el foco de una fase que ya tiene 3 entregables distintos.

Se propone tratarlo en una **Fase 7.5** o en la **Fase 8** (junto con CRM/Notificaciones), cuando su alcance se defina con el dueño.

### §12.4 Trazabilidad regla → sub-fase

| Regla / Directriz | Sub-fase que la cumple |
|---|---|
| **DT-07** (IA por contrato) | F7.0 (declara) + F7.2/F7.3 (consumen) |
| **DT-08** (visión cenital) | F7.3 (criterios 9, 10 y 11 del gate) |
| **A-02** (frontera por contratos) | F7.0 (cierra la brecha de H-2) |
| **RN-71** (motor ORB) | F7.3 (el motor vive en el Centro de IA) |
| **RN-72** (umbral 0.35) | F7.3 (criterio 4 del gate) |
| **RN-73** (etiquetado por SKU) | F7.3 (criterio 5 del gate) |
| **RN-74** (visión asistiva) | F7.3 (criterio 6 del gate) |
| **R-03** (3 modos) | F7.1, F7.2, F7.3 (criterios de gate) |
| **R-04** (target ≥44px) | F7.1, F7.2, F7.3 (criterios de gate) |
| **A-01** (regla con su test) | F7.2 (el mapper se porta con su test) |
| **DT-06** (Vista General) | F7.1 (identidad=null hasta que exista) |

### §12.5 La aclaración del hardware de visión (DT-08) y por qué cambia F7.3

El dueño aclaró (29 Sep 2026) que la visión del POS opera sobre una **cámara cenital con iluminación
dedicada** que elimina las sombras sobre el mostrador. Esto **no es un detalle de hardware**: cambia
el análisis de riesgo de la visión.

**Antes de la aclaración**, la visión se veía como el entregable de mayor riesgo ("adorno"): un
operador apuntando con una webcam a cada producto es **más lento** que teclear. **Después de la
aclaración**, la visión es un **"escáner de charola"**: punto de vista fijo, iluminación controlada,
el operador coloca los productos y el sistema los reconoce sin apuntar. Ese es el caso de uso
**fuerte** de la visión, y **sí agiliza** la captura.

**Los 3 ajustes concretos a F7.3 (ya incorporados en §8):**

1. **El umbral 0.35 es de calibración cenital, configurable (DT-08).** No es un valor universal. El
   hook lo lee de configuración (0.35 por defecto), no lo hardcodea. El Centro de IA lo calibra.
2. **El contrato 17 declara el `modo_captura` (`cenital` | `manual`).** El hook envía
   `modo_captura: 'cenital'`. Así el Centro de IA aplica la calibración correcta.
3. **El visor es un flujo persistente, no bajo demanda.** Permanece abierto durante la venta (modo
   "escáner de charola"), no se abre y cierra por producto.

**Lo que NO cambia:** la regla de oro (la visión sugiere, nunca decide — RN-74), la degradación
elegante (si el Centro de IA cae, el POS sigue vendiendo), y la frontera por contrato (A-02).

---

## §13. RESUMEN EJECUTIVO

| Sub-fase | Entregable | Archivos | Riesgo | Depende de |
|---|---|---|---|---|
| **F7.0** | Contratos 24 y 25 + test | 3 | Bajo | — |
| **F7.1** | Cableado de temas + `ThemeSelector` | 5 | Bajo | F7.0 |
| **F7.2** | Voz (mapper + hook + panel) | 7 | Medio | F7.0 |
| **F7.3** | Visión cenital (hook + visor persistente) | 4 | Medio | F7.0 |
| **F7.4** | Ficha de cierre | 1 | Bajo | F7.1–F7.3 |

**Total:** 20 archivos, 4 sub-fases de construcción + 1 de cierre. **Cero dependencias nuevas.**

**El POS termina la Fase 7 con:** un selector de temas funcional, un panel de voz que propone (nunca ejecuta), y un visor de visión cenital que sugiere (nunca bloquea) — los tres **preparados por contrato** para integrarse con el Centro de IA cuando exista.

**Cambio de versión 1.0 → 1.1:** la sub-fase F7.3 se ajustó por la aclaración del hardware de visión
(cámara cenital con iluminación dedicada). Ver **DT-08** y §12.5.