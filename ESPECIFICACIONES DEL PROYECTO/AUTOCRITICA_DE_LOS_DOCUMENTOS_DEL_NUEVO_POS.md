# AUTOCRÍTICA DE LOS DOCUMENTOS DEL NUEVO POS
## Contraste de los 5 documentos propios contra las fuentes autoritativas y el código real

> **Propósito:** verificar, con rigor y sin indulgencia, que lo que escribí es **fiel** a las
> fuentes autoritativas ([`DOCUMENTACION_MODULO_POS.md`](ESPECIFICACIONES DEL PROYECTO/DOCUMENTACION_MODULO_POS.md:1),
> [`CONTEXTO_SISTEMA_IA.md`](ESPECIFICACIONES DEL PROYECTO/CONTEXTO_SISTEMA_IA.md:1)) y al **código real** del POS.
> **Regla dura:** no se toca el ERP. Este documento es un artefacto de diseño.
> **Anclaje:** commit `5802f45` (V23); ingeniería inversa sobre `fe9f6ed` (tag `v22-estable-fe9f6ed`).

---

## SECCIÓN 0 — VEREDICTO GENERAL

**El veredicto es: los documentos son sólidos en sustancia, pero contienen 3 defectos
materiales que deben corregirse.** No son errores de invención (nada de lo que escribí
contradice al código), sino **errores de omisión y de encuadre**:

| # | Defecto | Gravedad | Documento afectado |
|---|---------|----------|--------------------|
| **D-1** | **Redundancia no declarada**: `CONTEXTO_SISTEMA_IA.md` §3.3 **ya documenta** la topología hub-and-spoke. Mi `MODELO_DESPLIEGUE...` no lo reconoce y parece "descubrir" algo que ya existía. | **ALTA** | `MODELO_DESPLIEGUE_Y_CONSOLIDACION_CENTRAL.md` |
| **D-2** | **Confusión de nomenclatura**: mezclo "Regla N" (las 21 reglas de `DOCUMENTACION_MODULO_POS.md`) con "RN-XX" (las 81 reglas de la especificación) sin aclarar que son **dos sistemas distintos**. | **MEDIA** | `GUIA_MAESTRA...` §4, §6 |
| **D-3** | **Omisión de las 21 Reglas de Oro del POS**: mis documentos citan las 81 RN pero **no mencionan** las 21 Reglas (Regla 1-21) que son la capa de blindaje del POS actual. Un colaborador que lea solo mis documentos no sabrá que existen. | **MEDIA** | `GUIA_MAESTRA...`, `MODELO_DESPLIEGUE...` |

**Lo que está BIEN y verifiqué contra el código:**

- ✅ Las 81 RN y su numeración (RN-61 a RN-66 = ledger; RN-78 a RN-80 = TZ; RN-81 = transversal) — **confirmado** en [`ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md`](ESPECIFICACIONES DEL PROYECTO/ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md:209).
- ✅ El DRAFT GUARD (Regla 7 / RN-31 a RN-36) — **confirmado** en [`service.py:111-125`](apps/api/modules/pos/service.py:111).
- ✅ El bloqueo optimista `version` (RN-25 a RN-30) — **confirmado** en [`service.py:129-135`](apps/api/modules/pos/service.py:129) y [`models.py:37`](apps/api/modules/pos/models.py:37).
- ✅ El folio atómico por secuencia PostgreSQL — **confirmado** en [`service.py:748-820`](apps/api/modules/pos/service.py:748).
- ✅ El outbox POS→Almacenes con `try/except pass` (DEUDA-04) — **confirmado** en [`service.py:46-54`](apps/api/modules/pos/service.py:46).
- ✅ La respuesta ligera de 5 campos (Regla 15) — **confirmado** en [`service.py:270-290`](apps/api/modules/pos/service.py:270).
- ✅ El TTL de 15 min de los candados — **confirmado** en [`occupancy.py:20-29`](apps/api/modules/pos/occupancy.py:20).
- ✅ La inmutabilidad de `terminal_id` (Regla 14) — **confirmado** en [`service.py:164-165`](apps/api/modules/pos/service.py:164).

---

## SECCIÓN 1 — DEFECTO D-1: LA REDUNDANCIA NO DECLARADA (GRAVEDAD ALTA)

### 1.1 El hallazgo

Al releer [`CONTEXTO_SISTEMA_IA.md`](ESPECIFICACIONES DEL PROYECTO/CONTEXTO_SISTEMA_IA.md:113), encontré que
**la Sección 3.3 ya documenta la topología hub-and-spoke completa**, y lo hace con más detalle
del que yo supuse:

| Subsección de `CONTEXTO_SISTEMA_IA.md` | Contenido | ¿Lo cubrí yo? |
|----------------------------------------|-----------|---------------|
| §3.3.1 Topología General | Diagrama Servidor Corporativo + Sucursal A/B | Sí, pero **sin citarlo** |
| §3.3.2 Tres Niveles de Conectividad | Nivel 1 (LAN), Nivel 2 (IndexedDB), Nivel 3 (tablets) | **NO** |
| §3.3.3 Sincronización Sucursal → Corporativo | Frecuencia: cierre del día, default `23:30` | Parcial (dije "5–15 min") |
| §3.3.4 Identificadores Únicos Globales | **UUID v4 como PK; enteros solo como folios de display** | Sí, pero **sin citarlo** |
| §3.3.5 Resolución de Conflictos | Sucursal gana en su dominio; catálogo es del corporativo; tabla `sync_conflictos` | **NO** |
| §3.3.6 Infraestructura del Servidor Local | Mini PC, Cloudflare Tunnels, `config.js` | **NO** |
| §3.3.8 Tecnologías Excluidas | CRDT, Kafka, WebSockets | **NO** |

### 1.2 Por qué es un defecto

Mi [`MODELO_DESPLIEGUE_Y_CONSOLIDACION_CENTRAL.md`](ESPECIFICACIONES DEL PROYECTO/MODELO_DESPLIEGUE_Y_CONSOLIDACION_CENTRAL.md:1)
está escrito como si la topología hub-and-spoke fuera **una decisión nueva** que el negocio
acaba de tomar. La Sección 0 dice: *"Lo que se creía (y quedó descartado)"* y presenta el
hub-and-spoke como la resolución de una duda.

**Pero no era una duda: ya estaba escrito en el system prompt.** El documento §3.3.4 dice
literalmente:

> *"Toda entidad creada en cualquier nodo usa **UUID v4** como clave primaria. Los enteros
> autoincrementales solo se usan como folios de display, locales a cada sucursal."*

Esto es **exactamente** lo que yo presenté como A-05 y como Sección 1 de mi modelo de despliegue.
No lo inventé: lo **redescubrí** sin darme cuenta de que ya era doctrina establecida.

### 1.3 La consecuencia práctica

Un colaborador que lea mi documento podría creer que:
1. La topología es una **propuesta** a validar, cuando es una **decisión inamovible** (§3.3 dice: *"Es una decisión de diseño inamovible"*).
2. El UUID es una **recomendación mía**, cuando es una **regla crítica** del system prompt.
3. La frecuencia de sync es **5–15 min** (lo que yo sugerí), cuando el default establecido es **23:30 (cierre del día)**.

### 1.4 La corrección

Debo añadir a mi documento una **nota de trazabilidad** que reconozca explícitamente que
`CONTEXTO_SISTEMA_IA.md` §3.3 es la **fuente autoritativa** y que mi documento es una
**expansión operativa** de ella, no un descubrimiento. Y debo **alinear la frecuencia de sync**
con el default establecido (23:30), no con mi sugerencia de 5–15 min.

---

## SECCIÓN 2 — DEFECTO D-2: LA CONFUSIÓN DE NOMENCLATURA (GRAVEDAD MEDIA)

### 2.1 El hallazgo

Existen **dos sistemas de numeración distintos** que yo mezclo sin aclararlo:

| Sistema | Dónde vive | Qué es | Cuántas |
|---------|-----------|--------|---------|
| **RN-XX** | [`ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md`](ESPECIFICACIONES DEL PROYECTO/ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md:120) §C | Reglas de negocio extraídas por ingeniería inversa | **81** (RN-01 a RN-81) |
| **Regla N** | [`DOCUMENTACION_MODULO_POS.md`](ESPECIFICACIONES DEL PROYECTO/DOCUMENTACION_MODULO_POS.md:572) §4 | Reglas de oro operativas del POS (blindaje) | **21** (Regla 1 a Regla 21) |

**Son capas distintas:**
- Las **RN-XX** describen **qué hace** el negocio (el "qué").
- Las **Regla N** describen **cómo se blinda** el POS contra bugs (el "cómo no romperlo").

### 2.2 Por qué es un defecto

Mi [`GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md`](ESPECIFICACIONES DEL PROYECTO/GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md:114)
dice: *"las **81 reglas de negocio** (RN-01 a RN-81)"* y en el glosario define `RN-XX` como
"Regla de Negocio numerada". **Nunca menciona las 21 Reglas de Oro.** Un colaborador que lea
solo mi guía no sabrá que existe la capa de blindaje (Regla 19 `buildResetPatch()`, Regla 20
guardián de simetría, Regla 21 guardián del `useEffect`), que es **la capa más frágil y más
importante de preservar**.

### 2.3 La corrección

Añadir a la guía una **tabla de los dos sistemas de numeración** y una mención explícita de
las 21 Reglas de Oro como capa de blindaje, con referencia a `DOCUMENTACION_MODULO_POS.md` §4.

---

## SECCIÓN 3 — DEFECTO D-3: LA OMISIÓN DE LAS 21 REGLAS DE ORO (GRAVEDAD MEDIA)

### 3.1 El hallazgo

[`DOCUMENTACION_MODULO_POS.md`](ESPECIFICACIONES DEL PROYECTO/DOCUMENTACION_MODULO_POS.md:572) §4 contiene
**21 Reglas de Oro** que son el resultado de **21 incidentes reales** (el "cementerio de bugs").
Estas reglas son la memoria del sistema. Las más críticas:

| Regla | Qué blinda | Incidente que la originó |
|-------|-----------|--------------------------|
| **Regla 5** | DRAFT vs OPEN (visibilidad en pizarrón) | Persistencia atómica por ítem |
| **Regla 7** | DRAFT GUARD (backend) | Robo accidental de tickets |
| **Regla 11** | Reciclaje de folios (máx. 5 min) | Huecos en numeración |
| **Regla 14** | Inmutabilidad de `terminal_id` | Terminal fantasma |
| **Regla 15** | Respuesta ligera (5 campos) | Degradación en hora rush |
| **Regla 16** | Retries simétricos (3 ops atómicas) | Asimetría de reintentos |
| **Regla 19** | `buildResetPatch()` (limpieza única) | Cuentas perdidas v7.0.3 |
| **Regla 20** | Guardián de simetría (4 rutas) | Asimetría v17 |
| **Regla 21** | Guardián del `useEffect` de refs | Acoplamiento implícito v21 |

### 3.2 Por qué es un defecto

Mi guía menciona "las cicatrices" (Sección 5.1) y da 5 ejemplos, pero **no nombra las 21 Reglas
ni las ancla a su documento fuente**. El riesgo es que un colaborador "limpie" una guarda sin
saber que tiene un número de regla y un incidente detrás.

### 3.3 La corrección

Añadir a la guía una sección que liste las **21 Reglas de Oro** (al menos las 9 críticas de la
tabla anterior) con su número y su incidente, y que remita a `DOCUMENTACION_MODULO_POS.md` §4
como fuente autoritativa.

---

## SECCIÓN 4 — VERIFICACIÓN CONTRA EL CÓDIGO (LO QUE CONFIRMÉ)

Para que la autocrítica sea honesta, verifiqué cada afirmación técnica de mis documentos
contra el código real. **Todas resultaron correctas:**

| Afirmación en mis documentos | Archivo verificado | Resultado |
|------------------------------|--------------------|-----------|
| El DRAFT GUARD impide cobrar un DRAFT de otra terminal | [`service.py:111-125`](apps/api/modules/pos/service.py:111) | ✅ Confirmado |
| El bloqueo optimista usa `version` y devuelve 409 | [`service.py:129-135`](apps/api/modules/pos/service.py:129) | ✅ Confirmado |
| El folio se genera con secuencia PostgreSQL atómica | [`service.py:748-820`](apps/api/modules/pos/service.py:748) | ✅ Confirmado |
| El outbox POS→Almacenes usa `try/except pass` | [`service.py:46-54`](apps/api/modules/pos/service.py:46) | ✅ Confirmado |
| La respuesta ligera tiene 5 campos | [`service.py:270-290`](apps/api/modules/pos/service.py:270) | ✅ Confirmado |
| Los candados tienen TTL de 15 min | [`occupancy.py:20-29`](apps/api/modules/pos/occupancy.py:20) | ✅ Confirmado |
| `terminal_id` nunca se sobrescribe | [`service.py:164-165`](apps/api/modules/pos/service.py:164) | ✅ Confirmado |
| El anti-downgrade detecta caídas >50% | [`service.py:220-232`](apps/api/modules/pos/service.py:220) | ✅ Confirmado |
| El ledger de inventario es inmutable (RN-61) | [`ESPECIFICACION...md:211`](ESPECIFICACIONES DEL PROYECTO/ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md:211) | ✅ Confirmado |
| El offset de TZ hardcodeado viola RN-81 (DB-04) | [`ESPECIFICACION...md:524`](ESPECIFICACIONES DEL PROYECTO/ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md:524) | ✅ Confirmado |

**Conclusión de la verificación:** **no hay ni una sola afirmación técnica falsa** en mis
documentos. Los defectos son de **encuadre y de omisión**, no de exactitud.

---

## SECCIÓN 5 — LO QUE ESTABA BIEN Y NO DEBO CAMBIAR

Para no "corregir" lo que ya estaba correcto, dejo constancia de lo que **debe permanecer**:

1. **La regla dura** (no tocar el ERP) — correcta y bien repetida.
2. **La distinción `branch_id` ≠ `tenant_id`** — correcta y valiosa (el system prompt §19
   planea `tenant_id` para SaaS; mi distinción aclara que el hub-and-spoke **no** lo necesita).
3. **El contrato de consolidación** (P1-P6, payload JSON, outbox) — correcto y **no** está en
   el system prompt con ese nivel de detalle. **Esto sí es aportación nueva y valiosa.**
4. **Los criterios de aceptación** (S1-S7, H1-H6) — correctos y no están en las fuentes.
5. **El orden de lectura y el checklist de incorporación** — correctos y útiles.
6. **La analogía del edificio** — correcta y pedagógica.

**Principio de la corrección:** **añadir trazabilidad y alinear, no reescribir.** Los documentos
no se tiran: se **anclan** a sus fuentes.

---

## SECCIÓN 6 — LAS CORRECCIONES (APLICADAS)

> **Estado (22 Sep 2026):** las 6 correcciones fueron **aplicadas** en los documentos destino.
> Se registra aquí la evidencia para cerrar el ciclo de la autocrítica.

| # | Corrección | Documento | Tipo | Estado | Evidencia |
|---|-----------|-----------|------|--------|-----------|
| **C-1** | Añadir nota de trazabilidad: `CONTEXTO_SISTEMA_IA.md` §3.3 es la fuente autoritativa de la topología | `MODELO_DESPLIEGUE...` §0 | Añadir | ✅ Aplicada | Bloque "⚠️ NOTA DE TRAZABILIDAD (autocrítica v1.1)" al inicio del documento |
| **C-2** | Alinear la frecuencia de sync al default establecido (23:30, cierre del día) | `MODELO_DESPLIEGUE...` §2.3 | Corregir | ✅ Aplicada | Tabla de dominios con "Al cierre del día (default `23:30`)" + bloque de alineación §3.3.3 |
| **C-3** | Añadir la tabla de los dos sistemas de numeración (RN-XX vs Regla N) | `GUIA_MAESTRA...` §4/§6 | Añadir | ✅ Aplicada | §4.1 "DOS SISTEMAS DE NUMERACIÓN DISTINTOS" + glosario `RN-XX` |
| **C-4** | Añadir las 21 Reglas de Oro (las 9 críticas) con su incidente | `GUIA_MAESTRA...` §5 | Añadir | ✅ Aplicada | §5.0 "LAS 21 REGLAS DE ORO DEL POS" con tabla de 9 reglas + incidente |
| **C-5** | Mencionar los 3 niveles de conectividad (§3.3.2) y la resolución de conflictos (§3.3.5) | `MODELO_DESPLIEGUE...` §2 | Añadir | ✅ Aplicada | §2.7 "Los 3 niveles de conectividad" + §2.8 "La resolución de conflictos" |
| **C-6** | Añadir `CONTEXTO_SISTEMA_IA.md` a la lista de documentos maestros | `GUIA_MAESTRA...` §2 | Añadir | ✅ Aplicada | §2 "Documento 0" + §2.0 "El Contexto del Sistema (máxima autoridad)" |

---

## SECCIÓN 7 — DECLARACIÓN DE LA REGLA DURA

> **NO SE TOCA EL ERP INSTALADO Y CORRIENDO.**
> **NO SE TOCA NINGUNO DE SUS MÓDULOS.**
> **NO SE MODIFICA NI UNA LÍNEA DEL POS ACTUAL.**

Este documento es un artefacto de diseño. No contiene código de producción.
El ERP permanece intacto y operando; su HEAD es `5802f45` (V23) al 22 Sep 2026. La ingeniería
inversa se hizo sobre `fe9f6ed` (tag `v22-estable-fe9f6ed`).

---

*Informe de autocrítica. Versión 1.2. Anclado al commit `5802f45` (V23); ingeniería inversa sobre `fe9f6ed`.
3 defectos materiales encontrados (1 alto, 2 medios). 0 afirmaciones técnicas falsas.
6 correcciones propuestas — **6 aplicadas** (ver Sección 6).*
