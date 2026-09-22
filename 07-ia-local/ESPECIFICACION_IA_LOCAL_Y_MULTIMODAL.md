# ESPECIFICACIÓN — IA LOCAL Y CAPTURA MULTIMODAL

> **Estado:** Decisión arquitectónica tomada. **Parcialmente implementada** (ver §0.1).
> **Fecha:** 22 Sep 2026
> **Autoridad:** Este documento **hereda** de [`CONTEXTO_SISTEMA_IA.md`](../../ESPECIFICACIONES%20DEL%20PROYECTO/CONTEXTO_SISTEMA_IA.md:1) (Documento 0) y de [`SPEC_AI_GATEWAY_TRANSVERSAL.md`](../../docs/SPEC_AI_GATEWAY_TRANSVERSAL.md:1). Si hay contradicción, **prevalece el Documento 0**.
> **Alcance:** Aplica al **ERP en operación** (integración sin destruir el POS) y al **NUEVO POS** (arquitectura nativa).
> **Ancla:** `5802f45` (V23) para el núcleo del plano; `c0c66fe` (v26.1) para la parte de IA. Ver [`ACTA_DE_RECONCILIACION_IA.md`](ACTA_DE_RECONCILIACION_IA.md:1).

---

## 0.1 ESTADO DE IMPLEMENTACIÓN (RECONCILIACIÓN v26.1)

> **Nota de reconciliación:** entre el ancla `5802f45` (V23) y el commit `c0c66fe` (v26.1), la IA **se construyó parcialmente** en el ERP en operación. Esta sección registra qué está hecho y qué no. El detalle completo está en [`ACTA_DE_RECONCILIACION_IA.md`](ACTA_DE_RECONCILIACION_IA.md:1).

| Decisión | Estado real (v26.1) | Nota |
|---|---|---|
| **D-1** | ✅ Implementada | Desfase de nombre: el código usa `AI_HABILITADA`, no `AI_LOCAL_ENABLED` |
| **D-2** | ✅ Parcial | Flexibilidad por URL implementada; `AI_LOCAL_MODE` **no** existe |
| **D-3** | ✅ Parcial | Arquitectura de conteo construida; faltan las 300 imágenes por tipo de pan |
| **D-4** | ✅ Parcial | E1-E3 construidas; **E4 (mAP) no existe**; entrenamiento **síncrono** (concesión) |
| **D-5** | ✅ Implementada | Fallback 503 completo |
| **D-6** | ⚠️ Concesión | Se tocó `pos/service.py` (viola la regla de oro §6.2) |
| **A-1** | ✅ Implementada | Multimodalidad respetada |

**Deudas registradas:** ver §12.

---

## 0. RESUMEN EJECUTIVO

El dueño del negocio tomó **6 decisiones** y dio **1 aclaración** sobre la incorporación de Inteligencia Artificial al sistema. Este documento las convierte en un contrato verificable.

| # | Decisión | Estado |
|---|---|---|
| D-1 | Se acepta la IA Local como capacidad del sistema, con **habilitación prioritaria** | ✅ Aceptada |
| D-2 | El sistema debe permitir **elegir** entre IA local por sucursal, IA local central o IA en la nube | ✅ Aceptada |
| D-3 | Se adopta la recomendación técnica para **conteo de panes** | ✅ Aceptada |
| D-4 | El módulo "Entrenamiento IA" se **rediseña** como módulo de anotación | ✅ Aceptada |
| D-5 | El fallback **503 `IA_NO_DISPONIBLE`** se mantiene y se documenta | ✅ Aceptada |
| D-6 | Se integra al **ERP en operación** (sin romper el POS) **y** al **NUEVO POS** | ✅ Aceptada |
| A-1 | "Manos libres" es el **dispositivo**, no el único método de entrada | ✅ Aclarada |

---

## 1. DECISIÓN D-1 — La IA Local es una capacidad del sistema

### 1.1 Qué significa

La IA deja de ser un experimento y se convierte en una **capacidad declarada** del ERP, con la misma jerarquía que el módulo de Almacenes o el POS.

### 1.2 Regla de oro heredada (no negociable)

> *"El módulo de almacenes NUNCA debe romperse porque la IA no esté."*
> — [`service.py`](../../apps/api/modules/ai/service.py:3)

Esta regla ya está implementada en el código: [`_ia_habilitada()`](../../apps/api/modules/ai/service.py:46) lee `AI_HABILITADA` (default `false`) y [`_lanzar_no_disponible()`](../../apps/api/modules/ai/service.py:69) lanza el 503 estándar.

> **Nota de reconciliación (C-1):** el nombre canónico de la variable es **`AI_HABILITADA`** (así está en el código real). Versiones anteriores de este documento citaban `AI_LOCAL_ENABLED`, que **no existe** en el código.

### 1.3 Habilitación prioritaria

La habilitación se prioriza, pero **no se salta el orden de construcción**. La IA Local se habilita **después** de que el POS esté blindado, porque:

1. La IA **propone**; el operador **confirma** (human-in-the-loop, §5.2 del spec del gateway).
2. Si el POS se cae por culpa de la IA, se viola la regla de oro.
3. El motor de IA **nunca** corre en el mismo proceso que el API del ERP (§6.1 del spec del gateway).

---

## 2. DECISIÓN D-2 — Flexibilidad de topología (3 modos)

### 2.1 Los 3 modos soportados

| Modo | Descripción | Cuándo usarlo |
|---|---|---|
| **M1 — Local por sucursal** | Un contenedor de IA en cada sucursal, junto al ERP local | Sucursales con hardware propio y conectividad intermitente |
| **M2 — Local central** | Un contenedor de IA en la matriz, sirviendo a todas las sucursales | Sucursales delgadas (sin GPU), red estable |
| **M3 — Nube** | Un servicio de IA externo (API de terceros) | Picos de demanda, modelos grandes, sin hardware local |

### 2.2 El contrato que hace posible la flexibilidad

La flexibilidad **no se logra con código condicional**. Se logra porque **el ERP solo conoce una URL**:

```python
# apps/api/modules/ai/service.py:42
def _url_motor_ia() -> str:
    """URL base del motor de IA Local (Whisper + LLM). Vacio si no se configuro."""
    return os.getenv("AI_LOCAL_URL", "").strip()
```

**El ERP no sabe —ni debe saber— si esa URL apunta a:**
- `http://ia-sucursal-01:9000` (M1)
- `http://ia-matriz:9000` (M2)
- `https://api.proveedor-ia.com/v1` (M3)

### 2.3 Variables de entorno (contrato de configuración)

| Variable | Default | Descripción | Estado real (v26.1) |
|---|---|---|---|
| `AI_HABILITADA` | `false` | Interruptor maestro. Apagado = modo manual puro | ✅ Implementada |
| `AI_LOCAL_URL` | `""` (vacío) | URL base del motor. Define el modo (M1/M2/M3) | ✅ Implementada |
| `AI_LOCAL_TIMEOUT` | `30` | Segundos antes de degradar a 503 | ✅ Implementada |
| `AI_LOCAL_MODE` | `local` | Etiqueta informativa: `local` \| `central` \| `cloud` | ❌ **No implementada** (pendiente) |

> **Regla:** `AI_LOCAL_MODE` es **solo informativo** (para diagnóstico y UI). El comportamiento **nunca** debe ramificar según este valor. Si el código hace `if mode == "cloud"`, es un error de diseño.

> **Nota de reconciliación (C-1):** el interruptor maestro se llama **`AI_HABILITADA`** en el código real, no `AI_LOCAL_ENABLED`. `AI_LOCAL_MODE` **no existe todavía**; se mantiene en el plano como pendiente (es barato y útil para diagnóstico/UI).

### 2.4 Criterio de aceptación D-2

- [ ] Cambiar de M1 a M2 a M3 se logra **solo** cambiando `AI_LOCAL_URL` (cero cambios de código)
- [ ] `GET /api/v1/ai/status` reporta el modo activo sin exponer secretos
- [ ] Existe un test que arranca el gateway con las 3 URLs y verifica que el contrato es idéntico

---

## 3. DECISIÓN D-3 — Recomendación para conteo de panes

### 3.1 El problema real (diagnóstico)

El código actual **no cuenta panes**. [`predict_vision()`](../../apps/api/modules/pos/service.py:854) es un **identificador de SKU**:

```python
# apps/api/modules/pos/service.py (resumen del comportamiento real)
orb = cv2.ORB_create(nfeatures=500)
# ... BFMatcher, distance < 45 ...
score = sku_max_matches / 40.0
if score > 0.35:  # identifica el SKU
    return qty=1   # ← SIEMPRE devuelve 1
```

**Conclusión:** el motor actual responde *"¿qué producto es?"*, nunca *"¿cuántos hay?"*. Son **dos problemas de visión distintos**.

### 3.2 Recomendación: separar los dos problemas

| Problema | Pregunta | Técnica recomendada | Por qué |
|---|---|---|---|
| **P1 — Identificación** | ¿Qué producto es? | **ORB actual** (ya funciona) o embeddings CLIP | Barato, sin GPU, ya está en producción |
| **P2 — Conteo** | ¿Cuántos hay? | **Detección de objetos** (YOLOv8-nano / RT-DETR) | Cuenta instancias, no clasifica |

### 3.3 Recomendación concreta para P2 (conteo de panes)

**Opción recomendada: YOLOv8-nano entrenado con dataset propio.**

| Criterio | YOLOv8-nano | RT-DETR | Conteo por densidad |
|---|---|---|---|
| Precisión en objetos apilados | Alta | Muy alta | Media |
| Velocidad en CPU | ~30 FPS | ~5 FPS | ~10 FPS |
| Requiere GPU | No (opcional) | Recomendable | No |
| Facilidad de entrenamiento | Alta (Ultralytics) | Media | Baja |
| **Veredicto** | ✅ **Elegida** | Sobredimensionada | No aplica |

**Justificación:**
1. Los panes son objetos **homogéneos y bien delimitados** — el caso ideal para detección.
2. YOLOv8-nano corre en **CPU** a velocidad aceptable (no exige GPU en sucursal).
3. Ultralytics tiene el flujo de entrenamiento más simple (anotar → `yolo train` → exportar).
4. Si el conteo falla, el operador **siempre** puede corregir a mano (human-in-the-loop).

### 3.4 Arquitectura híbrida recomendada

```
Cámara → [1] ORB (¿qué producto?) → [2] YOLO (¿cuántos?) → [3] Propuesta → [4] Operador confirma
              ↑ ya existe              ↑ a construir          ↑ confirmado: false
```

**Regla:** el resultado de [2] **nunca** se registra solo. Nace con `confirmado: false` (contrato ya blindado en [`mapVisionDetectionsToProposals()`](../../apps/inventory/utils/warehouseMappers.js:206)).

### 3.5 Criterio de aceptación D-3

- [ ] El conteo devuelve `qty > 1` (el motor actual siempre devuelve 1)
- [ ] Con confianza < 0.7, la UI resalta en ámbar y exige revisión
- [ ] El dataset de conteo tiene **mínimo 300 imágenes anotadas** por tipo de pan
- [ ] El POS sigue cobrando si el motor de conteo está caído (503 → manual)

---

## 4. DECISIÓN D-4 — Rediseño del módulo "Entrenamiento IA"

### 4.1 Diagnóstico del módulo actual

[`VisionTrainingUI.jsx`](../../apps/pos/VisionTrainingUI.jsx:15) es un **recolector de fotos**, no un entrenador:

| Lo que hace | Lo que NO hace |
|---|---|
| Seleccionar producto | Entrenar un modelo |
| Capturar 20 fotos | Anotar (bounding boxes) |
| Subir a `static/training/` | Versionar datasets |
| — | Medir precisión |

**Evidencia del estado real:** el dataset tiene **1 SKU (`10`) con 8 fotos**. Es un prototipo, no un sistema de entrenamiento.

### 4.2 Rediseño propuesto: 4 etapas

| Etapa | Nombre | Responsabilidad | Estado real (v26.1) |
|---|---|---|---|
| **E1** | **Captura** | Tomar fotos (ya existe, se conserva) | ✅ Construida |
| **E2** | **Anotación** | Dibujar bounding boxes sobre cada foto (NUEVO) | ✅ Construida |
| **E3** | **Entrenamiento** | Lanzar `yolo train` con el dataset anotado (NUEVO) | ✅ Construida |
| **E4** | **Evaluación** | Medir precisión (mAP) y promover el modelo (NUEVO) | ❌ **NO construida** (pendiente) |

> **Nota de reconciliación (C-4):** E1, E2 y E3 **se construyeron** en el ERP en operación (commits `8abe54a` y `c0c66fe`). **E4 (Evaluación/mAP) NO existe**: el pipeline publica `best.pt` sin medir precisión. Se mantiene como **pendiente crítico** (ver DEUDA-IA-01 en §12).

### 4.3 Reglas del módulo rediseñado

1. **El dataset es un activo versionado.** Cada versión tiene fecha, autor y métricas.
2. **Un modelo no se promueve sin métricas.** Si no hay mAP, no hay despliegue.
3. **El entrenamiento corre fuera del API del ERP.** Es un job, no un endpoint síncrono.
4. **La anotación es humana.** No se auto-etiqueta con el modelo que se quiere mejorar (sesgo circular).

> **Nota de reconciliación (C-5):** la implementación actual **viola las reglas 1, 2 y 3**:
> - **Regla 1:** no hay versionado formal del dataset.
> - **Regla 2:** se publica `best.pt` **sin mAP** (ver DEUDA-IA-03 en §12).
> - **Regla 3:** el entrenamiento es **síncrono** (timeout de 1h), no un job asíncrono. **Esto es una concesión al ERP viejo**, no diseño del NUEVO POS. El plano tiene razón: en el NUEVO POS debe ser un job con estado consultable (ver DEUDA-IA-02 en §12).
>
> La **regla 4 sí se cumple**: la anotación es humana.

### 4.4 Criterio de aceptación D-4

- [ ] Existe una UI de anotación con bounding boxes
- [ ] El dataset se versiona (fecha + autor + métricas)
- [ ] El entrenamiento es un job asíncrono, no bloquea el API
- [ ] Un modelo sin mAP registrado **no** puede activarse

---

## 5. DECISIÓN D-5 — Se mantiene el fallback 503

### 5.1 El contrato actual (se conserva tal cual)

```python
# apps/api/modules/ai/service.py:24-29
CODIGO_IA_NO_DISPONIBLE = "IA_NO_DISPONIBLE"
MENSAJE_IA_NO_DISPONIBLE = (
    "El motor de IA Local no esta disponible. Continue en modo manual."
)
```

### 5.2 Matriz de degradación (heredada, se mantiene)

| Escenario | Comportamiento esperado |
|---|---|
| IA apagada (`AI_HABILITADA=false`) | 503 → toast → operador usa entrada manual |
| IA encendida pero motor caído | `httpx` lanza excepción → capturar → 503 → toast |
| Timeout del motor (>30s) | `httpx.TimeoutException` → 503 → toast |
| Respuesta malformada | `ValidationError` → 503 → toast |
| Confianza baja (<0.7) | 200 con `confianza` baja → UI resalta en ámbar → operador revisa |

### 5.3 Regla de oro (se anota y se conserva)

> **El 503 es una respuesta de primera clase, no un error.**
> Un 503 significa *"sigue trabajando a mano"*, no *"algo se rompió"*.
> **Nunca** se debe convertir en un 500. **Nunca** debe bloquear el POS.

### 5.4 Criterio de aceptación D-5

- [ ] Con el motor caído, los endpoints devuelven **503** (no 500)
- [ ] La UI muestra un toast informativo, no un error rojo
- [ ] El POS sigue cobrando sin degradación medible
- [ ] Existe un test que apaga el motor y verifica el 503

---

## 6. DECISIÓN D-6 — Doble integración (ERP en operación + NUEVO POS)

### 6.1 Principio rector

> **Integrar al ERP en operación SIN destruir la funcionalidad y operatividad del POS.**

### 6.2 Estrategia de integración al ERP en operación

| Fase | Acción | Riesgo | Mitigación |
|---|---|---|---|
| **I1** | El gateway ya existe (`/api/v1/ai`) | Ninguno | Ya está en producción con stubs |
| **I2** | Levantar el motor en contenedor separado | Bajo | `mem_limit`; el ERP nunca importa `torch` |
| **I3** | Cablear `AI_LOCAL_URL` | Bajo | Si falla, 503 → manual |
| **I4** | Reemplazar `VoiceAgentService.js` (mock) | Medio | Hacerlo endpoint por endpoint, con tests |
| **I5** | Rediseñar "Entrenamiento IA" | Medio | Es un módulo aislado, no toca el POS |

**Regla de oro:** ninguna fase de integración puede tocar [`apps/api/modules/pos/service.py`](../../apps/api/modules/pos/service.py:1) sin autorización explícita.

> **Nota de reconciliación (C-6):** en la implementación real (v26.1), **`pos/service.py` SÍ se tocó**: se le añadieron `upload_training_images`, `list_training_dataset`, `save_annotations` y `predict_vision`. Fue una **concesión** (los endpoints de dataset se colocaron en el POS por cercanía). **El plano tiene razón:** en el NUEVO POS, el dataset y la anotación viven en el **módulo de IA**, no en el POS. El POS solo **consume** el contrato. Ver DEUDA-IA-04 en §12.

### 6.3 Estrategia de integración al NUEVO POS

En el NUEVO POS, la IA **nace integrada** desde el diseño:

- El **contrato** (`/api/v1/ai/*`) es parte de la Frontera (F2).
- El **fallback 503** es parte de los Guardianes (F4).
- La **captura multimodal** es parte de la Superficie (F5).
- El **motor** vive en un contenedor separado (nunca en el proceso del API).

### 6.4 Criterio de aceptación D-6

- [ ] El ERP en operación cobra normalmente con la IA encendida y apagada
- [ ] El NUEVO POS declara la IA en su Frontera (F2) y sus Guardianes (F4)
- [ ] Cero cambios en `pos/service.py` sin autorización
- [ ] Los tests del POS siguen en verde con la IA encendida

---

## 7. ACLARACIÓN A-1 — "Manos libres" es el dispositivo, no el método

### 7.1 La aclaración del dueño

> *"Con manos libres me refería al **dispositivo**, pero el sistema debe permitir usar la voz y también los demás métodos de entrada como el display touch o teclado o ratón."*

### 7.2 La consecuencia arquitectónica

**El sistema es multimodal por diseño.** La voz es **un** canal, no **el** canal.

| Canal de entrada | Dispositivo típico | Uso principal |
|---|---|---|
| **Voz** | Diadema / micrófono manos libres | Operador con las manos ocupadas (amasando, empacando) |
| **Touch** | Display táctil | Operador frente al POS |
| **Teclado** | Teclado físico | Captura rápida de códigos / cantidades |
| **Ratón** | Ratón / trackpad | Navegación en escritorio |
| **Visión** | Cámara | Conteo e identificación de producto |

### 7.3 Regla de diseño (crítica)

> **Ningún flujo puede depender de un solo canal de entrada.**
> Si la voz falla, el touch debe funcionar. Si el touch falla, el teclado debe funcionar.
> **La voz es una aceleración, nunca un requisito.**

### 7.4 Criterio de aceptación A-1

- [ ] Todo flujo accesible por voz es accesible por touch y por teclado
- [ ] Apagar el micrófono no bloquea ninguna operación
- [ ] La UI no asume que existe un dispositivo manos libres
- [ ] Existe un test que ejecuta el flujo completo sin voz

---

## 8. MATRIZ DE TRAZABILIDAD

| Decisión | Documento que la rige | Código que la implementa | Fase del NUEVO POS | Estado real (v26.1) |
|---|---|---|---|---|
| D-1 | Este documento + Documento 0 | [`service.py`](../../apps/api/modules/ai/service.py:46) | F2 (Frontera) | ✅ Implementada |
| D-2 | Este documento §2 | [`_url_motor_ia()`](../../apps/api/modules/ai/service.py:80) | F2 (Frontera) | ✅ Parcial (`AI_LOCAL_MODE` pendiente) |
| D-3 | Este documento §3 | [`predict_vision()`](../../apps/api/modules/pos/service.py:1011) | F3 (Comportamiento) | ✅ Parcial (faltan 300 imágenes) |
| D-4 | Este documento §4 | [`VisionTrainingUI.jsx`](../../apps/pos/VisionTrainingUI.jsx:1) | F5 (Superficie) | ✅ Parcial (E4 pendiente) |
| D-5 | [`SPEC_AI_GATEWAY_TRANSVERSAL.md`](../../docs/SPEC_AI_GATEWAY_TRANSVERSAL.md:152) | [`_lanzar_no_disponible()`](../../apps/api/modules/ai/service.py:69) | F4 (Guardianes) | ✅ Implementada |
| D-6 | Este documento §6 | Todo el módulo `ai/` | F2 + F4 + F5 | ⚠️ Concesión (`pos/service.py` tocado) |
| A-1 | Este documento §7 | Frontend multimodal | F5 (Superficie) | ✅ Implementada |

---

## 9. LO QUE ESTE DOCUMENTO **NO** AUTORIZA

1. **No autoriza** tocar [`apps/api/modules/pos/service.py`](../../apps/api/modules/pos/service.py:1) sin permiso explícito.
2. **No autoriza** cargar modelos de IA en el proceso del API del ERP.
3. **No autoriza** registrar stock automáticamente sin confirmación humana.
4. **No autoriza** ramificar el comportamiento según `AI_LOCAL_MODE`.
5. **No autoriza** depender de un solo canal de entrada.
6. **No autoriza** promover un modelo sin métricas (mAP) registradas.

---

## 10. ORDEN DE EJECUCIÓN RECOMENDADO

| # | Paso | Depende de | Riesgo |
|---|---|---|---|
| 1 | Levantar el motor de IA en contenedor separado (M1) | — | Bajo |
| 2 | Cablear `AI_LOCAL_URL` y verificar `/status` | Paso 1 | Bajo |
| 3 | Implementar `transcribir_voz` (Whisper) | Paso 2 | Medio |
| 4 | Implementar `interpretar_intencion` (LLM) | Paso 3 | Medio |
| 5 | Reemplazar `VoiceAgentService.js` (mock) | Paso 4 | Medio |
| 6 | Rediseñar "Entrenamiento IA" (anotación) | — | Medio |
| 7 | Entrenar YOLOv8-nano para conteo | Paso 6 | Alto |
| 8 | Implementar `detectar_vision` (conteo) | Paso 7 | Alto |
| 9 | Verificar multimodalidad (voz + touch + teclado) | Paso 5 | Bajo |

> **Nota:** los pasos 1-5 (voz) y 6-8 (visión) son **independientes** y pueden ejecutarse en paralelo.

---

## 11. REFERENCIAS

- [`CONTEXTO_SISTEMA_IA.md`](../../ESPECIFICACIONES%20DEL%20PROYECTO/CONTEXTO_SISTEMA_IA.md:1) — Documento 0 (autoridad máxima)
- [`SPEC_AI_GATEWAY_TRANSVERSAL.md`](../../docs/SPEC_AI_GATEWAY_TRANSVERSAL.md:1) — Spec del gateway
- [`service.py`](../../apps/api/modules/ai/service.py:1) — Capa de fallback (stubs)
- [`schemas.py`](../../apps/api/modules/ai/schemas.py:1) — Contratos Pydantic
- [`router.py`](../../apps/api/modules/ai/router.py:1) — Endpoints
- [`predict_vision()`](../../apps/api/modules/pos/service.py:854) — Motor ORB actual
- [`VisionTrainingUI.jsx`](../../apps/pos/VisionTrainingUI.jsx:15) — Recolector a rediseñar
- [`VoiceAgentService.js`](../../apps/voice-agent/VoiceAgentService.js:1) — Mock a reemplazar
- [`PLAN_DE_CONSTRUCCION_DEL_NUEVO_POS.md`](../05-plan-de-construccion/PLAN_DE_CONSTRUCCION_DEL_NUEVO_POS.md:1) — Plan de construcción
- [`PROMPT_DEL_ARQUITECTO_DEL_NUEVO_POS.md`](../06-prompt-del-arquitecto/PROMPT_DEL_ARQUITECTO_DEL_NUEVO_POS.md:1) — Contrato del arquitecto

---

## 12. DEUDAS REGISTRADAS (RECONCILIACIÓN v26.1)

> **Origen:** [`ACTA_DE_RECONCILIACION_IA.md`](ACTA_DE_RECONCILIACION_IA.md:1). Estas son las deudas **reales** (incumplimientos del plano), no desfases de nomenclatura. **No se resuelven en el ERP en operación**: se resuelven en el NUEVO POS.

| # | Deuda | Decisión | Por qué importa | Dónde se resuelve |
|---|---|---|---|---|
| **DEUDA-IA-01** | No existe E4 (Evaluación/mAP) | D-4 §4.2 | Sin métricas, no se sabe si el modelo mejoró o empeoró | NUEVO POS (módulo de IA) |
| **DEUDA-IA-02** | El entrenamiento es síncrono | D-4 §4.3 regla 3 | Bloquea un worker hasta 1h; no escala | NUEVO POS (job asíncrono) |
| **DEUDA-IA-03** | Se promueve sin mAP | D-4 §4.3 regla 2 | Se publica `best.pt` sin verificar calidad | NUEVO POS (guardia de promoción) |
| **DEUDA-IA-04** | El dataset vive en el POS | D-6 §6.2 | El POS no debe conocer el dataset; solo el contrato | NUEVO POS (mover al módulo de IA) |

**Nota:** DEUDA-IA-04 es una **concesión consciente y autorizada** en su momento, no un error. Se registra como deuda de diseño para el NUEVO POS.

---

**Fin del documento.**
