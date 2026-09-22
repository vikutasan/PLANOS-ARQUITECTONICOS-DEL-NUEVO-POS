# ACTA DE RECONCILIACIÓN — IA LOCAL Y CAPTURA MULTIMODAL

> **Estado:** ✅ **APLICADA** (22 Sep 2026). Aprobada por el dueño; los cambios C-1 a C-9 se aplicaron a [`ESPECIFICACION_IA_LOCAL_Y_MULTIMODAL.md`](ESPECIFICACION_IA_LOCAL_Y_MULTIMODAL.md:1) y el ancla dual se registró en [`README.md`](../README.md:1) y [`GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md`](../ESPECIFICACIONES%20DEL%20PROYECTO/GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md:1).
> **Fecha:** 22 Sep 2026
> **Autoridad:** Este documento **no** sustituye a [`ESPECIFICACION_IA_LOCAL_Y_MULTIMODAL.md`](ESPECIFICACION_IA_LOCAL_Y_MULTIMODAL.md:1). Lo **audita** y propone su corrección. Si hay contradicción, prevalece el Documento 0 ([`CONTEXTO_SISTEMA_IA.md`](../../ESPECIFICACIONES%20DEL%20PROYECTO/CONTEXTO_SISTEMA_IA.md:1)).
> **Alcance:** Reconciliar el plano de IA con lo que **realmente se construyó** en el ERP en operación entre el commit `5802f45` (V23, ancla del plano) y el commit `c0c66fe` (v26.1, realidad actual).

---

## 0. POR QUÉ EXISTE ESTE DOCUMENTO

El repo de planos nació por una razón concreta: **el POS que corre es frágil por acumulación**. Creció capa sobre capa, parche sobre deuda. La solución **no** es refactorizarlo (cirugía a corazón abierto sobre producción), sino **dibujar los planos de un POS nuevo que parezca que nació bien desde el principio**.

Ese plano está **anclado al commit `5802f45` (V23)**. La ingeniería inversa se hizo sobre `fe9f6ed` (tag `v22-estable-fe9f6ed`).

**El problema:** entre ese ancla y hoy, el ERP en operación **siguió avanzando**. En particular, se integró al ERP real toda la funcionalidad de IA (visión + voz + entrenamiento), que el plano describía como "pendiente de ejecución". El plano de IA **quedó desfasado**.

**Este acta reconcilia ese desfase.** Pero con una regla que no se negocia:

> **No se copia lo que se hizo al plano. Se DECIDE, punto por punto, qué es buen diseño (va al plano) y qué fue una concesión al ERP viejo (no va, o va marcado como deuda a resolver en el NUEVO POS).**

---

## 1. LA REGLA DE ORO DE ESTA RECONCILIACIÓN

> **El plano describe el POS que QUEREMOS, no el ERP que TENEMOS.**

Copiar el estado actual al plano sería **contaminar el plano con la deuda** — exactamente el Riesgo 2 que el propio [`PLANO ARQUITECTONICO PARA EL NUEVO POS.md`](../PLANO%20ARQUITECTONICO%20PARA%20EL%20NUEVO%20POS.md:1) §0.2 advierte.

Por eso, cada desfase se clasifica en **una de tres categorías**:

| Categoría | Significado | Qué se hace en el plano |
|---|---|---|
| **✅ DISEÑO** | Lo construido es buen diseño, coherente con el plano | Se **incorpora** al plano como "ya definido" |
| **⚠️ CONCESIÓN** | Se construyó así por limitaciones del ERP viejo | **NO** entra al plano. Se registra como deuda a resolver en el NUEVO POS |
| **❌ PENDIENTE** | El plano lo pedía y **no** se construyó | Se mantiene como pendiente. **No** se borra del plano |

---

## 2. EVIDENCIA: QUÉ SE CONSTRUYÓ REALMENTE

**Commits de referencia (repo `ERP-R-DE-RICO-CON-POS-SIMPLIFICADO`):**

| Commit | Qué entregó |
|---|---|
| `8abe54a` | Centro de IA (3 pestañas: Estado, Visión, Voz) + motor de IA local + pipeline de entrenamiento YOLO |
| `c0c66fe` | Fusión documental: `DOCUMENTACION_CENTRO_IA.md` (1199 líneas) absorbe el antiguo doc de entrenamiento |

**Documento fuente de la realidad:** [`DOCUMENTACION_CENTRO_IA.md`](../../ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_CENTRO_IA.md:1) (en el repo del ERP).

**Código real (repo del ERP):**

| Componente | Archivo | Qué hace hoy |
|---|---|---|
| Gateway (fallback 503) | [`service.py`](../../apps/api/modules/ai/service.py:1) | `_ia_habilitada()`, `_lanzar_no_disponible()`, traducción de errores |
| Endpoints IA | [`router.py`](../../apps/api/modules/ai/router.py:1) | `/status`, `/vision/detect`, `/voice/transcribe`, `/voice/parse-intent`, `/vision/dataset-summary`, `/vision/train/status`, `/vision/train` |
| Motor de visión | [`vision.py`](../../ai-local/app/engines/vision.py:1) | YOLOv8 (detección/conteo) |
| Motor de voz | [`nlu.py`](../../ai-local/app/engines/nlu.py:1) | Whisper (transcripción) + Ollama (intención) |
| Orquestador de entrenamiento | [`training.py`](../../ai-local/app/engines/training.py:1) | Subproceso YOLO + hot-reload del modelo |
| Pipeline YOLO | [`train_bakery.py`](../../ai-local/train/train_bakery.py:1) | `descubrir_dataset` → `construir_dataset_yolo` → `entrenar` → `publicar_modelo` |
| UI de anotación | [`AnnotationCanvas.jsx`](../../apps/pos/components/AnnotationCanvas.jsx:1) | Dibujo de bounding boxes |
| UI de entrenamiento | [`VisionTrainingUI.jsx`](../../apps/pos/VisionTrainingUI.jsx:1) | Captura + anotación + lanzar entrenamiento |
| UI de voz (POS) | [`useVoiceCart.js`](../../apps/pos/hooks/useVoiceCart.js:1) | Dictado → intención → propuesta de carrito |
| UI de voz (almacén) | [`WarehouseManagerUI.jsx`](../../apps/inventory/WarehouseManagerUI.jsx:1) | Dictado → propuesta de entrada |

---

## 3. AUDITORÍA DECISIÓN POR DECISIÓN

### D-1 — La IA Local es una capacidad del sistema

**Lo que dice el plano:** la IA es una capacidad declarada, con la misma jerarquía que Almacenes o POS. Regla de oro: *"El módulo de almacenes NUNCA debe romperse porque la IA no esté."*

**Lo que hay hoy:**
- ✅ El gateway existe y está aislado en `apps/api/modules/ai/`.
- ✅ `_ia_habilitada()` lee el interruptor maestro (default `false`).
- ✅ `_lanzar_no_disponible()` lanza el 503 estándar.
- ✅ El ERP **nunca** importa `torch`, `whisper` ni `ultralytics` (viven en `ai-local/`, contenedor separado).

**Desfase detectado (⚠️ CONCESIÓN):**
- El plano cita `AI_LOCAL_ENABLED` (§1.2). El código real usa **`AI_HABILITADA`**.
- El plano cita `_ia_habilitada()` en la línea 32; el código real lo tiene en la línea **46**.

**Clasificación:** ✅ **DISEÑO** (la capacidad está bien construida). El desfase es **de nomenclatura**, no de arquitectura.

**Acción propuesta:** corregir el nombre de la variable en el plano a `AI_HABILITADA` (o declarar que el nombre canónico es el del código). **No** se cambia el código.

---

### D-2 — Flexibilidad de topología (3 modos)

**Lo que dice el plano:** 3 modos (M1 local por sucursal, M2 local central, M3 nube). El ERP solo conoce **una URL** (`AI_LOCAL_URL`). Variables: `AI_LOCAL_ENABLED`, `AI_LOCAL_URL`, `AI_LOCAL_TIMEOUT`, `AI_LOCAL_MODE`.

**Lo que hay hoy:**
- ✅ El ERP solo conoce una URL base del motor (`_url_motor_ia()`).
- ✅ `AI_LOCAL_TIMEOUT` existe (`_timeout_motor_ia()`, default 30s).
- ❌ **`AI_LOCAL_MODE` NO está implementado.** No existe en el código.
- ⚠️ El interruptor maestro se llama `AI_HABILITADA`, no `AI_LOCAL_ENABLED`.

**Clasificación:**
- La **flexibilidad por URL** (el corazón de D-2) es ✅ **DISEÑO** — está bien resuelta.
- `AI_LOCAL_MODE` es ❌ **PENDIENTE** — el plano lo pedía como "etiqueta informativa"; no se construyó. **Se mantiene en el plano como pendiente** (es barato y útil para diagnóstico/UI).

**Acción propuesta:** mantener `AI_LOCAL_MODE` como pendiente en el plano. Corregir `AI_LOCAL_ENABLED` → `AI_HABILITADA`.

---

### D-3 — Recomendación para conteo de panes

**Lo que dice el plano:** separar **identificación** (P1: ORB, ya existe) de **conteo** (P2: YOLOv8-nano, a construir). Criterio: mínimo **300 imágenes anotadas por tipo de pan**.

**Lo que hay hoy:**
- ✅ P1 (ORB) sigue existiendo: [`predict_vision()`](../../apps/api/modules/pos/service.py:1011).
- ✅ P2 (YOLOv8) **se construyó**: [`vision.py`](../../ai-local/app/engines/vision.py:1) detecta y cuenta.
- ✅ La arquitectura híbrida (ORB → YOLO → propuesta → operador confirma) está implementada.
- ✅ El resultado nace con `confirmado: false` ([`mapVisionDetectionsToProposals()`](../../apps/inventory/utils/warehouseMappers.js:220)).
- ❌ **El dataset NO tiene 300 imágenes.** Tiene **8 imágenes de 1 SKU** (el mismo prototipo que el plano diagnosticó).

**Clasificación:**
- La **arquitectura de conteo** es ✅ **DISEÑO** — se construyó como el plano pedía.
- El **criterio de 300 imágenes** es ❌ **PENDIENTE** — es un criterio de **datos**, no de código. El código está listo; el dataset no.

**Acción propuesta:** mantener el criterio de 300 imágenes como pendiente. **No** se relaja el criterio para "hacerlo pasar". El plano tiene razón: sin datos, el conteo no es confiable.

---

### D-4 — Rediseño del módulo "Entrenamiento IA"

**Lo que dice el plano:** 4 etapas (E1 Captura, E2 Anotación, E3 Entrenamiento, E4 Evaluación). 4 reglas: dataset versionado; sin mAP no hay promoción; entrenamiento es un **job asíncrono**; anotación humana.

**Lo que hay hoy:**

| Etapa | Estado real | Clasificación |
|---|---|---|
| **E1 Captura** | ✅ Existe ([`VisionTrainingUI.jsx`](../../apps/pos/VisionTrainingUI.jsx:1)) | ✅ DISEÑO |
| **E2 Anotación** | ✅ Existe ([`AnnotationCanvas.jsx`](../../apps/pos/components/AnnotationCanvas.jsx:1)) | ✅ DISEÑO |
| **E3 Entrenamiento** | ✅ Existe ([`training.py`](../../ai-local/app/engines/training.py:1) + [`train_bakery.py`](../../ai-local/train/train_bakery.py:1)) | ✅ DISEÑO |
| **E4 Evaluación (mAP)** | ❌ **NO existe.** El pipeline publica `best.pt` sin medir mAP | ❌ PENDIENTE |

**Reglas del módulo:**

| Regla del plano | Estado real | Clasificación |
|---|---|---|
| Dataset versionado (fecha + autor + métricas) | ❌ No hay versionado formal | ❌ PENDIENTE |
| Sin mAP no hay promoción | ❌ Se publica `best.pt` sin mAP | ❌ PENDIENTE |
| Entrenamiento es un **job asíncrono** | ❌ Es **síncrono** (timeout 1h) | ⚠️ CONCESIÓN |
| Anotación humana | ✅ Es humana | ✅ DISEÑO |

**Análisis crítico de la concesión (⚠️):**
El entrenamiento síncrono (esperar hasta 1h en un endpoint) es una **concesión al ERP viejo**: era lo más simple de cablear sin infraestructura de colas. **El plano tiene razón**: debe ser un job asíncrono. Esta concesión **NO debe copiarse al plano**. En el NUEVO POS, el entrenamiento es un job con estado consultable.

**Acción propuesta:**
1. Mantener E4 (Evaluación/mAP) como **pendiente crítico** en el plano.
2. Mantener el versionado del dataset como pendiente.
3. Mantener "sin mAP no hay promoción" como regla **inviolable** del plano.
4. **Reafirmar** que el entrenamiento es un job asíncrono (el plano ya lo dice; la implementación síncrona es deuda del ERP viejo, no diseño del nuevo).

---

### D-5 — Se mantiene el fallback 503

**Lo que dice el plano:** el 503 `IA_NO_DISPONIBLE` es respuesta de primera clase. Nunca 500. Nunca bloquea el POS.

**Lo que hay hoy:**
- ✅ `CODIGO_IA_NO_DISPONIBLE = "IA_NO_DISPONIBLE"` existe.
- ✅ La matriz de degradación está implementada (motor caído → 503; timeout → 503; respuesta malformada → 503).
- ✅ La traducción de errores está documentada en [`DOCUMENTACION_CENTRO_IA.md`](../../ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_CENTRO_IA.md:1) §15.4.

**Desfase detectado:** el plano cita `AI_LOCAL_ENABLED` en la matriz (§5.2). El código usa `AI_HABILITADA`.

**Clasificación:** ✅ **DISEÑO** — el fallback está perfectamente implementado. Solo hay desfase de nomenclatura.

**Acción propuesta:** corregir el nombre de la variable. **Nada más.**

---

### D-6 — Doble integración (ERP en operación + NUEVO POS)

**Lo que dice el plano:** integrar al ERP **sin destruir el POS**. Fases I1-I5. Regla de oro: **ninguna fase puede tocar `pos/service.py` sin autorización explícita**.

**Lo que hay hoy:**
- ✅ I1 (gateway existe): hecho.
- ✅ I2 (motor en contenedor separado): hecho ([`docker-compose.ai.yml`](../../docker-compose.ai.yml:1)).
- ✅ I3 (cablear URL): hecho.
- ✅ I4 (reemplazar `VoiceAgentService.js` mock): hecho ([`useVoiceCart.js`](../../apps/pos/hooks/useVoiceCart.js:1) usa el gateway real).
- ✅ I5 (rediseñar "Entrenamiento IA"): hecho (Centro de IA).
- ⚠️ **`pos/service.py` SÍ se tocó**: se le añadieron `upload_training_images`, `list_training_dataset`, `save_annotations`, `predict_vision`. **Esto viola la regla de oro de D-6 §6.2.**

**Análisis crítico (⚠️ CONCESIÓN):**
Tocar `pos/service.py` fue una **concesión**: el módulo de visión del POS necesitaba endpoints de dataset, y se colocaron ahí por cercanía. **El plano tiene razón**: en el NUEVO POS, el dataset y la anotación viven en el **módulo de IA**, no en el POS. El POS solo **consume** el contrato.

**Clasificación:** ⚠️ **CONCESIÓN** — no se copia al plano. En el NUEVO POS, el POS no conoce el dataset.

**Acción propuesta:** registrar como deuda explícita: *"En el NUEVO POS, los endpoints de dataset/anotación viven en el módulo de IA, no en el POS."*

---

### A-1 — "Manos libres" es el dispositivo, no el método

**Lo que dice el plano:** el sistema es **multimodal por diseño**. Voz, touch, teclado, ratón, visión. Regla: *"Ningún flujo puede depender de un solo canal de entrada."*

**Lo que hay hoy:**
- ✅ La voz es **un** canal: existe [`useVoiceCart.js`](../../apps/pos/hooks/useVoiceCart.js:1) (POS) y la voz en [`WarehouseManagerUI.jsx`](../../apps/inventory/WarehouseManagerUI.jsx:1) (almacén).
- ✅ El touch/teclado/ratón siguen funcionando (el POS no depende de la voz).
- ✅ La visión es otro canal ([`VisionScanner.jsx`](../../apps/pos/VisionScanner.jsx:1)).

**Clasificación:** ✅ **DISEÑO** — la multimodalidad se respetó. La voz es una aceleración, no un requisito.

**Acción propuesta:** ninguna. El plano ya describe correctamente lo construido.

---

## 4. RESUMEN DE LA AUDITORÍA

| Decisión | Veredicto | Acción sobre el plano |
|---|---|---|
| **D-1** | ✅ DISEÑO | Corregir `AI_LOCAL_ENABLED` → `AI_HABILITADA` |
| **D-2** | ✅ DISEÑO + ❌ PENDIENTE | Corregir variable; mantener `AI_LOCAL_MODE` como pendiente |
| **D-3** | ✅ DISEÑO + ❌ PENDIENTE | Mantener criterio de 300 imágenes como pendiente |
| **D-4** | ✅ DISEÑO + ❌ PENDIENTE + ⚠️ CONCESIÓN | Mantener E4/mAP y versionado como pendientes; reafirmar job asíncrono |
| **D-5** | ✅ DISEÑO | Corregir variable |
| **D-6** | ⚠️ CONCESIÓN | Registrar deuda: dataset/anotación no viven en el POS |
| **A-1** | ✅ DISEÑO | Ninguna |

**Conteo:**
- ✅ **DISEÑO** (se incorpora/confirma): D-1, D-2 (parcial), D-3 (parcial), D-4 (parcial), D-5, A-1
- ⚠️ **CONCESIÓN** (no va al plano, va como deuda): D-4 (entrenamiento síncrono), D-6 (tocar `pos/service.py`)
- ❌ **PENDIENTE** (se mantiene en el plano): D-2 (`AI_LOCAL_MODE`), D-3 (300 imágenes), D-4 (E4/mAP + versionado)

---

## 5. LAS 3 DEUDAS REALES (LO QUE FALTA DE VERDAD)

De toda la auditoría, solo **3 cosas** son incumplimientos reales del plano (no desfases de nomenclatura):

| # | Deuda | Decisión | Por qué importa | Dónde se resuelve |
|---|---|---|---|---|
| **DEUDA-IA-01** | **No hay E4 (Evaluación/mAP)** | D-4 §4.2 | Sin métricas, no se sabe si el modelo mejoró o empeoró | NUEVO POS (módulo de IA) |
| **DEUDA-IA-02** | **El entrenamiento es síncrono** | D-4 §4.3 regla 3 | Bloquea un worker hasta 1h; no escala | NUEVO POS (job asíncrono) |
| **DEUDA-IA-03** | **Se promueve sin mAP** | D-4 §4.3 regla 2 | Se publica `best.pt` sin verificar calidad | NUEVO POS (guardia de promoción) |

**Nota:** la deuda D-6 (tocar `pos/service.py`) es una **concesión consciente y autorizada** en su momento, no un incumplimiento. Se registra como deuda de diseño para el NUEVO POS, no como error.

---

## 6. LO QUE **NO** SE HACE EN ESTA RECONCILIACIÓN

1. **No se toca el ERP.** Ni una línea. (Regla dura.)
2. **No se copia el estado actual al plano.** El plano describe el POS que queremos.
3. **No se relajan los criterios** (300 imágenes, mAP) para "hacerlos pasar".
4. **No se borra del plano** lo que no se construyó. Se marca como pendiente.
5. **No se convierte una concesión en diseño** solo porque ya funciona.

---

## 7. CAMBIOS PROPUESTOS AL PLANO (PARA APROBACIÓN)

Si el dueño aprueba esta acta, se aplicarán **estos cambios quirúrgicos** a [`ESPECIFICACION_IA_LOCAL_Y_MULTIMODAL.md`](ESPECIFICACION_IA_LOCAL_Y_MULTIMODAL.md:1):

| # | Sección | Cambio propuesto | Tipo |
|---|---|---|---|
| **C-1** | §1.2, §2.3, §5.2 | `AI_LOCAL_ENABLED` → `AI_HABILITADA` (nombre real del código) | Corrección |
| **C-2** | §1.2 | Actualizar referencia de línea `service.py:32` → `service.py:46` | Corrección |
| **C-3** | §0 (tabla de decisiones) | Añadir columna "Estado de implementación" (✅/⚠️/❌) | Adición |
| **C-4** | §4.2 | Marcar E4 como **PENDIENTE** (no construido) | Anotación |
| **C-5** | §4.3 | Añadir nota: "la implementación actual es síncrona; es una concesión al ERP viejo, no diseño del NUEVO POS" | Anotación |
| **C-6** | §6.2 | Añadir nota: "`pos/service.py` fue tocado; en el NUEVO POS el dataset vive en el módulo de IA" | Anotación |
| **C-7** | §8 (matriz de trazabilidad) | Añadir columna "Estado real (v26.1)" | Adición |
| **C-8** | §12 (nuevo) | Añadir sección "Deudas registradas" con DEUDA-IA-01/02/03 | Adición |
| **C-9** | Cabecera | Actualizar el ancla: de `5802f45` (V23) a `c0c66fe` (v26.1) | Corrección |

**Además**, se propone actualizar el ancla en los documentos que la citan:
- [`GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md`](../ESPECIFICACIONES%20DEL%20PROYECTO/GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md:358) §7 (checklist: "HEAD `5802f45`" → `c0c66fe`)
- [`README.md`](../README.md:1) (sección de anclaje)
- [`GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md`](../ESPECIFICACIONES%20DEL%20PROYECTO/GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md:370) §8 (declaración de cierre)

> **Nota sobre el ancla:** el ancla `5802f45` (V23) sigue siendo válida para **el resto del plano** (POS, almacenes, RRHH, etc.), que no cambió. Solo la **parte de IA** avanzó. Por eso se propone un **ancla dual**: `5802f45` para el núcleo, `c0c66fe` para IA. Esto evita invalidar todo el plano por un solo módulo.

---

## 8. PREGUNTAS PARA EL DUEÑO (ANTES DE EJECUTAR)

> **Resolución (22 Sep 2026):** el dueño aprobó las 3 propuestas.
> 1. ✅ **Ancla dual** aprobada (`5802f45` núcleo + `c0c66fe` IA).
> 2. ✅ **Deudas** aprobadas como pendientes del NUEVO POS (DEUDA-IA-01/02/03/04).
> 3. ✅ **9 cambios quirúrgicos** (C-1 a C-9) aprobados y aplicados.

---

## 9. QUÉ PASA DESPUÉS (SI SE APRUEBA)

1. Se aplican C-1 a C-9 a [`ESPECIFICACION_IA_LOCAL_Y_MULTIMODAL.md`](ESPECIFICACION_IA_LOCAL_Y_MULTIMODAL.md:1).
2. Se actualiza el ancla en los 3 documentos citados.
3. Se hace `commit` + `push` **en el repo de planos** (nunca en el del ERP).
4. Se marca esta acta como **"Aplicada"** (deja de ser propuesta).

---

**Fin del acta. Propuesta pendiente de aprobación.**
