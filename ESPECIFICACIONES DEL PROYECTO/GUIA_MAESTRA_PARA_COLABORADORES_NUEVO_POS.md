# GUÍA MAESTRA PARA COLABORADORES — NUEVO POS
## Punto de entrada único: qué leer, en qué orden, y qué reglas nunca romper

> **Este es el primer documento que debe leer cualquier persona que vaya a meter mano
> en el nuevo proyecto.** Si solo vas a leer un documento, lee este. Al final sabrás
> qué leer después, en qué orden, y qué reglas son inviolables.
>
> **Regla dura:** no se toca el ERP. Este documento es un artefacto de diseño.

---

## SECCIÓN 0 — BIENVENIDA: QUÉ ES ESTE PROYECTO

### 0.1 En una frase

> **Estamos dibujando el plano del POS que construiremos en las futuras sucursales,
> sin tocar el POS que hoy está vendiendo.**

### 0.2 La analogía del edificio

El POS actual es un **edificio funcional** al que se le fueron añadiendo habitaciones,
tuberías e instalaciones sobre la marcha. **Funciona.** Pero cada nueva habitación tuvo
que conectarse a las tuberías que ya existían, y esas tuberías no fueron diseñadas para
esa carga.

Este proyecto es el **plano** que un arquitecto dibujaría para los **edificios nuevos**:
la misma funcionalidad, pero con las cimentaciones y las instalaciones puestas de forma
elegante desde un principio.

**No es para demoler el edificio actual. Es para no repetir sus accidentes en los nuevos.**

### 0.3 La topología (lo que cambió y hay que entender)

**No habrá una operación multisucursal.** Lo que se hará es:

> **Instalar un ERP completo (con el POS incluido como uno de sus módulos) en cada
> sucursal, y cada ERP enviará data a un servidor central del corporativo.**

Esto es **hub-and-spoke** (concentrador y radios):

```
        ┌──────────────────────────────────┐
        │   SERVIDOR CENTRAL CORPORATIVO   │
        │   (consolidación / reportería)   │
        │   NO transaccional               │
        └──────────────────────────────────┘
              ▲          ▲          ▲
    ┌─────────┘          │          └─────────┐
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│ SUCURSAL A  │   │ SUCURSAL B  │   │ SUCURSAL C  │
│ ERP COMPLETO│   │ ERP COMPLETO│   │ ERP COMPLETO│
│  ├─ POS     │   │  ├─ POS     │   │  ├─ POS     │
│  └─ BD PROPIA│  │  └─ BD PROPIA│  │  └─ BD PROPIA│
└─────────────┘   └─────────────┘   └─────────────┘
```

**Las 3 cosas que esto implica y que debes memorizar:**

1. **Cada sucursal es independiente.** Tiene su propio ERP y su propia base de datos.
2. **El central no es transaccional.** Si se apaga, las sucursales siguen vendiendo.
3. **El folio `V####` es local y es correcto así.** No es un bug. La identidad real es el UUID.

---

## SECCIÓN 1 — LA REGLA DURA (LO PRIMERO QUE DEBES GRABARTE)

> ## ⛔ NO SE TOCA EL ERP INSTALADO Y CORRIENDO.
> ## ⛔ NO SE TOCA NINGUNO DE SUS MÓDULOS.
> ## ⛔ NO SE MODIFICA NI UNA LÍNEA DEL POS ACTUAL.

### 1.1 Qué significa en la práctica

| ✅ SÍ puedes | ❌ NO puedes |
|--------------|--------------|
| Leer el código del POS actual (solo lectura) | Modificar cualquier archivo del ERP |
| Escribir documentos en el repo de planos | Hacer commit en el repo del ERP |
| Proponer cambios en los planos | "Arreglar rapidito" algo en el ERP |
| Crear archivos nuevos en el repo de planos | Importar código del ERP al repo de planos |

### 1.2 Por qué esta regla es inviolable

El ERP está **vendiendo todos los días**. Un cambio mal hecho no rompe un test:
**rompe la caja de una panadería en horario de atención**. El costo de un error no es
un bug, es **dinero y clientes perdidos**.

Por eso el trabajo se hace **al lado**, en un **proyecto nuevo**, en un **repositorio nuevo**.

### 1.3 Cómo se verifica que se respetó

```bash
# En el repo del ERP:
git status --short      # Solo deben aparecer archivos .md sin trackear
git log -1 --format="%h"  # Debe seguir siendo fe9f6ed
```

Si el HEAD cambió o hay archivos de código modificados, **la regla dura se violó**.

---

## SECCIÓN 2 — LOS 5 DOCUMENTOS MAESTROS (QUÉ ES CADA UNO)

El proyecto tiene **5 documentos maestros**. Cada uno responde una pregunta distinta.
**No son redundantes: son capas.**

> **⚠️ CORRECCIÓN C-6 (autocrítica v1.1):** la versión 1.0 de esta guía listaba solo
> 4 documentos y **omitía el más autoritativo de todos**: [`CONTEXTO_SISTEMA_IA.md`](ESPECIFICACIONES DEL PROYECTO/CONTEXTO_SISTEMA_IA.md:1).
> Ese documento es el **system prompt del sistema** y su §3.3 ya documentaba la topología
> hub-and-spoke como decisión "inamovible". Se añade como **Documento 0**.

| # | Documento | Responde la pregunta | Cuándo leerlo |
|---|-----------|---------------------|---------------|
| **0** | [`CONTEXTO_SISTEMA_IA.md`](ESPECIFICACIONES DEL PROYECTO/CONTEXTO_SISTEMA_IA.md:1) | **¿Cuál es la doctrina inamovible del sistema?** | **Primero de todos.** Es la máxima autoridad |
| **1** | [`PLANO ARQUITECTONICO PARA EL NUEVO POS.md`](PLANO ARQUITECTONICO PARA EL NUEVO POS.md:1) | **¿Qué debe ser el nuevo POS?** | Segundo, para entender el diseño |
| **2** | [`ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md`](ESPECIFICACIONES DEL PROYECTO/ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md:1) | **¿Qué hace hoy el POS actual?** | Tercero, para conocer la fuente de verdad |
| **3** | [`PLAN_ACCION_ARQUITECTONICO_NUEVO_POS.md`](ESPECIFICACIONES DEL PROYECTO/PLAN_ACCION_ARQUITECTONICO_NUEVO_POS.md:1) | **¿Qué acciones concretas ejecutar?** | Cuarto, para saber cómo no perder las cicatrices |
| **4** | [`MODELO_DESPLIEGUE_Y_CONSOLIDACION_CENTRAL.md`](ESPECIFICACIONES DEL PROYECTO/MODELO_DESPLIEGUE_Y_CONSOLIDACION_CENTRAL.md:1) | **¿Dónde vive y cómo se consolida?** | Quinto, para entender la topología |

### 2.0 Documento 0 — El Contexto del Sistema (máxima autoridad)

**Qué contiene:** la doctrina completa del sistema. En particular, su **§3.3** documenta
la topología hub-and-spoke como decisión **"inamovible"**: los 3 niveles de conectividad
(§3.3.2), la frecuencia de sincronización al cierre del día (§3.3.3), la regla de
**UUID v4 como clave primaria global** (§3.3.4), la resolución de conflictos (§3.3.5),
la infraestructura del servidor local (§3.3.6) y las tecnologías excluidas (§3.3.8).

**Por qué es el primero:** si este documento contradice a cualquier otro, **manda este**.
Los demás documentos son **derivados** de él, no al revés.

### 2.1 Documento 1 — El Plano (fundacional)

**Qué contiene:** la opinión sobre el enfoque, las **81 reglas de negocio** (RN-01 a RN-81),
los **contratos entre módulos**, el plan de la primera etapa y la estructura del repositorio.

**Qué NO contiene:** código, UI, decisiones técnicas accidentales.

**Es la fuente de verdad estructural:** cómo *debería* construirse.

### 2.2 Documento 2 — La Especificación Funcional (ingeniería inversa)

**Qué contiene:** las **33 funcionalidades** (F-01 a F-33), las **81 reglas** con su origen
(archivo + línea), el **modelo de datos** (qué entra, se procesa y se guarda), los **6 flujos**,
y los **29 hallazgos** (5 deudas + 10 acoplamientos + 10 debilidades + 4 riesgos de concurrencia).

**Es la fuente de verdad funcional:** qué *hace* el POS actual.

### 2.3 Documento 3 — El Plan de Acción (las 5 acciones)

**Qué contiene:** las **5 acciones arquitectónicas** (A-01 a A-05), la **matriz de trazabilidad**
que mapea los 29 hallazgos a su acción, el orden de ejecución y los criterios de aceptación.

| Acción | Qué garantiza |
|--------|---------------|
| **A-01** | Portar las reglas **con sus tests** (no perder las cicatrices) |
| **A-02** | Frontera por contratos (el POS no lee tablas ajenas) |
| **A-03** | Test guardián por regla crítica (CI falla si se viola) |
| **A-04** | Outbox transaccional (elimina `try/except pass`) |
| **A-05** | Identidad (UUID) ≠ Presentación (folio) |

**Es la capa de ejecución:** cómo *no perder* lo que el POS actual ya aprendió.

### 2.4 Documento 4 — El Modelo de Despliegue (topología)

**Qué contiene:** la topología hub-and-spoke, la identidad de sucursal (`branch_id`),
el contrato de consolidación (qué envía cada ERP al central), el servidor central,
el proceso de despliegue por sucursal y los criterios de "terminado".

**Es la capa de despliegue:** dónde *vive* y cómo se *consolida*.

---

## SECCIÓN 3 — EL ORDEN DE LECTURA RECOMENDADO

### 3.1 Si tienes 10 minutos (lo mínimo indispensable)

1. **Este documento** (ya lo estás leyendo).
2. **Sección 1 de este documento** (la regla dura).
3. **Sección 0.3 de este documento** (la topología).

Con eso ya sabes lo esencial: no tocar el ERP, y cómo se despliega.

### 3.2 Si tienes 1 hora (para empezar a trabajar)

1. **Este documento** completo.
2. [`PLANO ARQUITECTONICO PARA EL NUEVO POS.md`](PLANO ARQUITECTONICO PARA EL NUEVO POS.md:1) — Sección 0 (opinión) y Sección 1 (reglas).
3. [`MODELO_DESPLIEGUE_Y_CONSOLIDACION_CENTRAL.md`](ESPECIFICACIONES DEL PROYECTO/MODELO_DESPLIEGUE_Y_CONSOLIDACION_CENTRAL.md:1) — Secciones 0, 1 y 2.

### 3.3 Si vas a escribir código (lectura completa)

1. **Este documento** completo.
2. **Documento 1 (Plano)** completo — es la fuente de verdad estructural.
3. **Documento 2 (Especificación)** — para conocer el comportamiento a replicar.
4. **Documento 3 (Plan de Acción)** completo — las 5 acciones son obligatorias.
5. **Documento 4 (Despliegue)** — para entender dónde corre tu código.

### 3.4 El orden lógico (por qué este orden)

```
1. GUÍA MAESTRA (este doc)     → contexto y reglas
        ↓
2. PLANO (qué debe ser)        → el diseño
        ↓
3. ESPECIFICACIÓN (qué hace)   → la fuente de verdad funcional
        ↓
4. PLAN DE ACCIÓN (cómo)       → las acciones obligatorias
        ↓
5. DESPLIEGUE (dónde)          → la topología
```

**Principio:** primero el **contexto**, luego el **diseño**, luego la **realidad**,
luego la **ejecución**, luego el **despliegue**.

---

## SECCIÓN 4 — LAS 10 REGLAS DE ORO (RESUMEN EJECUTIVO)

Si no recuerdas nada más, recuerda estas 10 reglas. Todas están detalladas en los
5 documentos maestros.

| # | Regla | Documento que la detalla |
|---|-------|--------------------------|
| **1** | **No se toca el ERP.** Nunca. Ni una línea. | Todos |
| **2** | **El POS es un módulo del ERP**, no un sistema aparte. | Despliegue §0.3 |
| **3** | **Cada sucursal tiene su propia BD.** No hay multi-tenant. | Despliegue §1.5 |
| **4** | **El folio es local; el UUID es global.** Nunca uses el folio como identidad. | Plan de Acción A-05 |
| **5** | **El POS no lee tablas ajenas.** Solo contratos. | Plan de Acción A-02 |
| **6** | **Ninguna regla migra sin su test.** La unidad es regla + test. | Plan de Acción A-01 |
| **7** | **No hay `try/except pass` en la ruta crítica.** Outbox transaccional. | Plan de Acción A-04 |
| **8** | **El central no es transaccional.** Si cae, las sucursales siguen. | Despliegue §3.2 |
| **9** | **Todo timestamp se guarda en UTC.** Se muestra en hora local. | Plano RN-78 a RN-80 |
| **10** | **El inventario es un ledger inmutable.** Nunca `UPDATE stock`. | Plano RN-61 a RN-66 |

### 4.1 ⚠️ DOS SISTEMAS DE NUMERACIÓN DISTINTOS (NO CONFUNDIRLOS)

> **⚠️ CORRECCIÓN C-3 (autocrítica v1.1):** la versión 1.0 de esta guía usaba "RN-XX"
> y "Regla N" de forma ambigua. **Son dos sistemas distintos y no intercambiables.**

| Sistema | Rango | Dónde vive | Qué es |
|---------|-------|-----------|--------|
| **RN-XX** | RN-01 a RN-81 | [`ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md`](ESPECIFICACIONES DEL PROYECTO/ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md:1) §C | **81 reglas de negocio** extraídas por ingeniería inversa, en 15 categorías (C.1 a C.15) |
| **Regla N** | Regla 1 a Regla 21 | [`DOCUMENTACION_MODULO_POS.md`](ESPECIFICACIONES DEL PROYECTO/DOCUMENTACION_MODULO_POS.md:1) §4 | **21 reglas de oro del POS** nacidas de incidentes reales (cicatrices) |

**Cómo recordarlo:**
- **RN-XX** = *qué hace el negocio* (funcional, numeración larga, 81 reglas).
- **Regla N** = *qué no debe romperse en el código* (técnica, numeración corta, 21 reglas).

Cuando alguien diga "la regla 19", pregunta: **¿RN-19 o Regla 19?** No son la misma.

---

## SECCIÓN 5 — LOS ERRORES QUE NO DEBES COMETER

### 5.0 ⚠️ LAS 21 REGLAS DE ORO DEL POS (LAS CICATRICES CODIFICADAS)

> **⚠️ CORRECCIÓN C-4 (autocrítica v1.1):** la versión 1.0 de esta guía **omitía por
> completo** las 21 reglas de oro documentadas en [`DOCUMENTACION_MODULO_POS.md`](ESPECIFICACIONES DEL PROYECTO/DOCUMENTACION_MODULO_POS.md:1) §4.
> Son la destilación de incidentes reales. **Cada una nació de un bug que ya se pagó.**

Las 9 más críticas (las que un colaborador nuevo debe memorizar antes de tocar código):

| Regla | Qué prohíbe / exige | Incidente que la originó |
|-------|---------------------|--------------------------|
| **Regla 5** | El POS no lee tablas de otro módulo; solo contratos | Acoplamiento por BD (AC-01 a AC-10) |
| **Regla 7** | No `try/except pass` en la ruta crítica | Fallos invisibles (DEUDA-04) |
| **Regla 11** | El folio es local; la identidad es el UUID | Colisión de folios entre sucursales (DB-02) |
| **Regla 14** | DRAFT GUARD: una terminal no pisa el ticket de otra | Dos cajeros editando la misma venta |
| **Regla 15** | Anti-degradación: un payload incompleto (>50% menos) no borra líneas | Pérdida silenciosa de ventas |
| **Regla 16** | Bloqueo optimista (`version`): no sobrescribir cambios concurrentes | Pérdida del último cambio |
| **Regla 19** | Toda ruta de cierre aplica `buildResetPatch()` (contrato de reset) | Estado sucio entre terminales |
| **Regla 20** | Guardián de simetría: las 4 rutas de salida aplican el mismo patch | Rutas divergentes (v20) |
| **Regla 21** | Guardián del `useEffect` de re-sincronización de refs | Refs obsoletas tras cambio de terminal (v21) |

**Las 21 reglas completas** están en [`DOCUMENTACION_MODULO_POS.md`](ESPECIFICACIONES DEL PROYECTO/DOCUMENTACION_MODULO_POS.md:1) §4.
**No las resumas de memoria: léelas del documento fuente.**

### 5.1 El error más peligroso: limpiar las cicatrices

El mayor peligro de limpiar la arquitectura es **limpiar también las cicatrices**.

Ejemplos de cicatrices que **NO** debes quitar "porque con contratos limpios ya no hacen falta":

| Cicatriz | Por qué existe | Qué pasa si la quitas |
|----------|----------------|----------------------|
| **DRAFT GUARD** | Evita que dos terminales editen el mismo ticket | Dos cajeros pisan la misma venta |
| **Anti-degradación (>50%)** | Evita que un payload incompleto borre líneas | Se pierden ventas silenciosamente |
| **Bloqueo optimista (`version`)** | Evita sobrescribir cambios concurrentes | Se pierde el último cambio |
| **Reciclaje de folios** | Evita huecos en la numeración | Reportes con folios saltados |
| **Idempotencia de emergencia** | Evita duplicar tickets al reconectar | Ventas duplicadas |

**Cada guarda rara es la cicatriz de un bug real que ya se pagó.** Quitarla reproduce
el bug en 6 meses.

### 5.2 Los 5 anti-patrones prohibidos

| # | Anti-patrón | Por qué está prohibido |
|---|-------------|------------------------|
| **1** | `try/except pass` en la ruta crítica | El fallo se vuelve invisible (DEUDA-04) |
| **2** | Leer tablas de otro módulo | Acoplamiento por BD (AC-01 a AC-10) |
| **3** | Usar el folio como clave | Colisión entre sucursales (DB-02) |
| **4** | Hardcodear el offset de zona horaria (`+6h`) | Viola RN-81 (DB-04) |
| **5** | `UPDATE stock` en vez de insertar en el ledger | Rompe la trazabilidad (RN-61 a RN-66) |

### 5.3 El error de proceso: trabajar en el repo equivocado

| Repo | Qué se hace ahí |
|------|-----------------|
| `ERP-R-DE-RICO-CON-POS-SIMPLIFICADO` | **NADA.** Solo lectura. |
| `PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS` | **TODO.** Documentos y, en su momento, el código nuevo. |

Si te encuentras haciendo `git commit` en el repo del ERP, **detente**: estás violando
la regla dura.

---

## SECCIÓN 6 — GLOSARIO (PARA NO PERDERSE)

| Término | Significado |
|---------|-------------|
| **ERP** | Sistema de planificación de recursos empresariales. El sistema completo del negocio. |
| **POS** | Punto de venta. En este proyecto, **un módulo** del ERP. |
| **Sucursal (spoke)** | Una instalación completa del ERP en una ubicación física. |
| **Central (hub)** | El servidor corporativo que recibe y consolida datos. No es transaccional. |
| **Hub-and-spoke** | Topología de concentrador y radios: muchos emisores, un receptor. |
| **`branch_id`** | Identificador de sucursal. Es **configuración**, no dato de negocio. |
| **UUID** | Identificador único universal. La **identidad real** de cada registro. |
| **Folio (`V####`)** | Número visible del ticket. **Local** a cada sucursal. Solo presentación. |
| **Outbox** | Patrón: el evento se guarda en la misma transacción y se envía después. |
| **Ledger** | Registro inmutable de movimientos. El stock se **deriva**, no se sobrescribe. |
| **RN-XX** | Regla de Negocio numerada (RN-01 a RN-81). |
| **F-XX** | Funcionalidad numerada (F-01 a F-33). |
| **A-XX** | Acción arquitectónica (A-01 a A-05). |
| **DEUDA-XX** | Deuda técnica conocida (DEUDA-01 a DEUDA-05). |
| **AC-XX** | Acoplamiento innecesario (AC-01 a AC-10). |
| **DB-XX** | Debilidad de diseño (DB-01 a DB-10). |
| **RC-XX** | Riesgo de concurrencia (RC-01 a RC-04). |
| **DRAFT GUARD** | Guarda que impide que dos terminales editen el mismo ticket. |
| **Cicatriz** | Guarda nacida de un bug real. **No se quita.** |

---

## SECCIÓN 7 — CHECKLIST DE INCORPORACIÓN

Marca cada punto cuando lo hayas completado. Si terminas con todos marcados,
estás listo para trabajar en el proyecto.

- [ ] Leí este documento completo.
- [ ] Entendí la **regla dura** (no se toca el ERP).
- [ ] Entendí la **topología** (hub-and-spoke: ERP por sucursal + central).
- [ ] Sé que **el POS es un módulo**, no un sistema aparte.
- [ ] Sé que **cada sucursal tiene su propia BD** (no multi-tenant).
- [ ] Sé que **el folio es local y el UUID es global**.
- [ ] Leí el **Documento 1 (Plano)** — al menos las Secciones 0 y 1.
- [ ] Leí el **Documento 2 (Especificación)** — al menos las Secciones B y C.
- [ ] Leí el **Documento 3 (Plan de Acción)** — las 5 acciones.
- [ ] Leí el **Documento 4 (Despliegue)** — al menos las Secciones 0, 1 y 2.
- [ ] Memorizo las **10 reglas de oro** (Sección 4).
- [ ] Sé que **no debo limpiar las cicatrices** (Sección 5.1).
- [ ] Verifiqué que el **ERP sigue intacto** (`git status` + HEAD `fe9f6ed`).
- [ ] Sé en qué **repositorio** debo trabajar (Sección 5.3).

---

## SECCIÓN 8 — DECLARACIÓN DE LA REGLA DURA (REPETIDA AL CIERRE)

> **NO SE TOCA EL ERP INSTALADO Y CORRIENDO.**
> **NO SE TOCA NINGUNO DE SUS MÓDULOS.**
> **NO SE MODIFICA NI UNA LÍNEA DEL POS ACTUAL.**

Este documento es el **punto de entrada** al proyecto. No contiene código de producción.
El ERP permanece intacto y operando, anclado al commit `fe9f6ed` (tag `v22-estable-fe9f6ed`).

---

*Guía maestra de incorporación. Versión 1.0. Anclada al commit `fe9f6ed`.
Topología: hub-and-spoke (ERP completo por sucursal + servidor central corporativo).*
