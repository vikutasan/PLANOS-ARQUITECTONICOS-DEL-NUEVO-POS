# METODOLOGÍA DE INGENIERÍA INVERSA Y DISEÑO
## Cómo extraer, documentar y rediseñar cualquier módulo del ERP sin tocar el que corre

> **Este documento es el manual del método.** No describe el POS: describe **cómo hicimos
> el POS**, para que el mismo procedimiento se pueda aplicar, sin improvisar, a cada uno
> de los módulos restantes del ERP (Almacenes, Productos, RRHH, Estadísticas, Auditoría,
> Heladería, Reparto, Monitoreo de Red, Pedidos, Perfiles).
>
> **Regla dura:** el ERP instalado y corriendo **no se toca**. Este método produce
> **documentos**, no código de producción.

---

## SECCIÓN 0 — POR QUÉ EXISTE ESTE MÉTODO

### 0.1 El problema que resuelve

Un ERP que lleva años en producción acumula dos cosas mezcladas:

1. **Conocimiento del negocio** — reglas que son verdad y que costaron dinero aprender.
2. **Accidentes técnicos** — atajos, acoplamientos y parches que resolvieron un incendio
   pero que hoy son deuda.

El peligro al "modernizar" es **borrar ambas cosas a la vez**: limpiar la arquitectura y,
sin darte cuenta, borrar también la protección que evitaba un bug real.

### 0.2 La distinción que sostiene todo el método

| Concepto | Definición | Qué se hace con ella |
|----------|-----------|----------------------|
| **Cicatriz** | Guarda nacida de un bug real que ya se pagó | **Se conserva.** Se porta con su test |
| **Deuda** | Atajo que oculta un fallo o acopla módulos | **Se elimina.** Se sustituye por un patrón correcto |
| **Regla de negocio** | Comportamiento que el negocio exige | **Se documenta y se porta** |
| **Accidente** | Decisión técnica sin valor de negocio | **Se descarta** en el rediseño |

**Confundir cicatriz con deuda es el error más caro de este trabajo.** El método existe
para que esa distinción se haga **con evidencia** (código + línea), no con opinión.

### 0.3 El principio de anclaje

Todo hallazgo debe estar **anclado a evidencia verificable**: `archivo:línea`, commit, o
test. Una afirmación sin ancla es una opinión, y las opiniones no se portan.

---

## SECCIÓN 1 — LAS 6 FASES DEL MÉTODO

El método tiene 6 fases secuenciales. Cada fase produce **un artefacto** y tiene un
**criterio de salida** que debe cumplirse antes de pasar a la siguiente.

```
FASE 1  INGENIERÍA INVERSA      →  ESPECIFICACION_FUNCIONAL_<MODULO>.md
                                →  ESPECIFICACION_DE_INTERFACES_<MODULO>.md
FASE 2  DISEÑO DEL NUEVO        →  PLANO_<MODULO>.md
FASE 3  PLAN DE ACCIÓN          →  PLAN_ACCION_<MODULO>.md
FASE 4  MODELO DE DESPLIEGUE    →  (reutiliza el modelo global hub-and-spoke)
FASE 5  PUNTO DE ENTRADA        →  GUIA_<MODULO>.md
FASE 6  AUTOCRÍTICA             →  AUTOCRITICA_<MODULO>.md
```

> **Nota sobre la Fase 1.** La Fase 1 produce **dos** artefactos: la especificación
> funcional (qué hace el módulo, backend) y la especificación de interfaces (cómo se ve
> y cómo se toca, frontend). Un módulo sin su mapa de interfaces es un módulo a medio
> documentar: quien lo reconstruya sabrá qué calcular, pero no cómo debe sentirse.

---

## SECCIÓN 2 — FASE 1: INGENIERÍA INVERSA

**Objetivo:** entender y documentar **qué hace hoy** el módulo, con evidencia.

### 2.1 Paso 1.1 — Inventario de superficie

Lista todos los archivos del módulo antes de leerlos. **Un módulo tiene dos caras: el
backend (`.py`) y el frontend (`.jsx`).** Inventaría ambas.

```
apps/api/modules/<modulo>/          ← BACKEND (qué hace)
  ├── models.py      → el modelo de datos (qué se guarda)
  ├── schemas.py     → los contratos de entrada/salida (qué entra y sale)
  ├── service.py     → la lógica de negocio (qué se hace)
  ├── router.py      → los endpoints (qué se expone)
  └── <otros>.py     → helpers específicos (locks, auditoría, sync…)

apps/<modulo>/                      ← FRONTEND (cómo se ve y se toca)
  ├── <Modulo>UI.jsx          → pantallas raíz
  ├── components/*.jsx        → modales, paneles, overlays, composición
  └── <Modulo>Template.jsx    → plantillas de impresión (si aplica)
```

**Criterio de salida:** sabes cuántos archivos hay (backend **y** frontend) y qué rol
cumple cada uno.

### 2.2 Paso 1.2 — Lectura completa del código

Lee **todo** el módulo, no solo lo que parece importante. Los bugs viven en los bordes.

**Qué buscar mientras lees:**

| Señal en el código | Qué significa | Cómo se clasifica |
|--------------------|---------------|-------------------|
| `try/except pass` | Fallo silencioso | **Deuda** |
| `SELECT`/`JOIN` a tabla de otro módulo | Acoplamiento por BD | **Deuda** (AC-XX) |
| Guarda condicional "rara" con comentario | Cicatriz de un bug | **Cicatriz** |
| Columna `version` + comparación | Bloqueo optimista | **Cicatriz** |
| `utcnow()` / conversión de zona horaria | Regla transversal | **Regla** |
| Secuencia / `nextval` / folio | Identidad vs presentación | **Regla** |
| `if` de negocio (descuentos, límites) | Regla de negocio | **Regla** (RN-XX) |

### 2.3 Paso 1.3 — Extraer las reglas de negocio (RN-XX)

Numera las reglas con el prefijo **RN-** y agrúpalas por categoría (C.1, C.2, …).

**Formato obligatorio de cada regla:**

```
RN-XX — <título corto>
  Qué: <comportamiento observable>
  Dónde: <archivo>:<línea>
  Por qué: <razón de negocio o incidente>
  Test que la cubre: <archivo de test>::<nombre del test>  (o "SIN TEST" si no existe)
```

**Regla de oro de esta fase:** si una regla no tiene test, se marca **SIN TEST**. Eso es
un hallazgo, no un detalle.

### 2.4 Paso 1.4 — Extraer las funcionalidades (F-XX)

Numera las funcionalidades con el prefijo **F-**. Una funcionalidad es una capacidad
observable desde fuera (ej. "reservar ticket", "calcular corte de caja").

**Formato:**

```
F-XX — <nombre de la funcionalidad>
  Entrada: <qué recibe>
  Proceso: <qué hace>
  Salida: <qué devuelve / qué persiste>
  Endpoints: <rutas HTTP>
  Reglas que aplica: RN-XX, RN-YY
```

### 2.5 Paso 1.5 — Documentar el modelo de datos

Para cada tabla del módulo:

```
Tabla: <nombre>
  Columnas clave: <nombre> <tipo> <restricción>
  Identidad: <PK actual>  ← ¿es entero autoincremental o UUID?
  Relaciones: <FK a qué tabla>
  Notas: <índices, unicidad, JSON embebido>
```

**Pregunta crítica:** ¿la PK es un entero autoincremental? Si sí, **anótalo**: será un
problema al replicar el módulo en varias sucursales (colisión de identidades).

### 2.6 Paso 1.6 — Documentar los flujos

Dibuja los flujos principales como secuencias de pasos. Para cada flujo:

```
Flujo: <nombre>
  1. <actor> hace <acción> → <endpoint>
  2. El servicio <qué hace>
  3. Se persiste <qué>
  4. Se emite <evento / outbox>
  Puntos de fallo: <dónde puede romperse>
```

### 2.7 Paso 1.7 — Catalogar los hallazgos

Clasifica **todo** lo que encontraste en 4 categorías con prefijos fijos:

| Prefijo | Categoría | Significado |
|---------|-----------|-------------|
| **DEUDA-XX** | Deuda técnica | Atajo que hay que eliminar |
| **AC-XX** | Acoplamiento | El módulo lee/escribe algo que no es suyo |
| **DB-XX** | Debilidad de diseño | Decisión que no escala o es frágil |
| **RC-XX** | Riesgo de concurrencia | Condición de carrera o lock faltante |

### 2.8 Paso 1.8 — Inventariar las interfaces (frontend)

**El backend dice qué calcula el módulo; las interfaces dicen cómo se ve y cómo se toca.**
Si solo documentas el backend, quien reconstruya el módulo sabrá qué debe calcular, pero
no cómo debe sentirse en el mostrador. La ergonomía vive en los `.jsx`, no en los `.py`.

**Qué hacer:**

1. **Inventariar** cada interfaz del módulo y clasificarla por tipo:

   | Tipo | Qué es | Ejemplo |
   |------|--------|---------|
   | **Pantalla raíz** | La vista completa que orquesta todo | `RetailVisionPOS` |
   | **Modal** | Diálogo que bloquea y pide una decisión | `CheckoutScreen` |
   | **Panel / overlay** | Zona fija o aviso global | `SalesReceipt`, `POSHeader` |
   | **Composición** | Pieza reutilizable dentro de una pantalla | `ProductGrid`, `ProductCard` |
   | **Impresión** | Plantilla de ancho fijo (ticket, corte) | `TicketTemplate` |

2. **Documentar cada interfaz con una ficha de 7 puntos:**

   ```
   FICHA XX — <NombreDelComponente>
     1. Propósito        → qué problema resuelve en el mostrador
     2. Estructura visual → las zonas (header, cuerpo, panel lateral, acciones)
     3. Controles        → cada botón, campo y tecla, con su handler
     4. Estados          → vacío, cargando, error, éxito, offline
     5. Navegación       → cómo se entra, cómo se sale, qué la abre
     6. Modo responsivo  → Mostrador / Compacto / Móvil (según R-01 a R-04)
     7. Anclaje al código → <archivo>:<línea>
   ```

3. **Anclar cada ficha a `archivo:línea`** (igual que las reglas RN-XX). Una ficha sin
   ancla es una opinión, y las opiniones no se portan.

4. **Registrar las observaciones de ergonomía** (O-XX) que surjan — **sin aplicarlas**.
   Son notas para el módulo nuevo, no correcciones al módulo actual.

**Artefacto:** `ESPECIFICACION_DE_INTERFACES_<MODULO>.md` (o un anexo de interfaces dentro
de `ESPECIFICACION_FUNCIONAL_<MODULO>.md` si el módulo tiene pocas interfaces).

**Criterio de salida de la Fase 1:** existe `ESPECIFICACION_FUNCIONAL_<MODULO>.md` con
las reglas (RN-XX), las funcionalidades (F-XX), el modelo de datos, los flujos y los
hallazgos catalogados — **todo anclado a `archivo:línea`** — **y** existe
`ESPECIFICACION_DE_INTERFACES_<MODULO>.md` con una ficha de 7 puntos por cada interfaz
del frontend.

---

## SECCIÓN 3 — FASE 2: DISEÑO DEL NUEVO MÓDULO

**Objetivo:** definir **qué debe ser** el módulo nuevo, sin repetir los accidentes.

### 3.1 Paso 2.1 — Declarar la opinión sobre el enfoque

Antes de diseñar, escribe tu **opinión honesta** sobre el enfoque actual: qué está bien,
qué está mal y por qué. Esto fija el criterio de las decisiones siguientes.

### 3.2 Paso 2.2 — Decidir el destino de cada hallazgo

Para **cada** hallazgo de la Fase 1, decide explícitamente:

| Hallazgo | Decisión | Justificación |
|----------|----------|---------------|
| DEUDA-01 | **Eliminar** | Se sustituye por <patrón> |
| AC-03 | **Eliminar** | Se sustituye por contrato |
| DB-02 | **Eliminar** | Se sustituye por UUID |
| RC-01 | **Conservar como cicatriz** | Es una guarda válida |

**Ningún hallazgo puede quedar sin decisión.** Si no decides, el accidente se reproduce.

### 3.3 Paso 2.3 — Definir los contratos entre módulos

El módulo nuevo **no lee tablas ajenas**. Define contratos explícitos:

```
Contrato: <nombre>
  Proveedor: <módulo que expone>
  Consumidor: <módulo que consume>
  Entrada: <esquema>
  Salida: <esquema>
  Errores: <códigos y significados>
  Garantías: <idempotencia, orden, atomicidad>
```

### 3.4 Paso 2.4 — Definir el modelo de datos nuevo

Aplica las reglas transversales del sistema:

- **Identidad:** UUID v4 como PK global. El entero autoincremental solo como folio de
  display, local a la sucursal.
- **Tiempo:** todo timestamp en UTC. La conversión a hora local es de presentación.
- **Inventario:** ledger inmutable. El stock se **deriva**, nunca se sobrescribe.

### 3.5 Paso 2.5 — Definir la estructura del repositorio nuevo

Propón la estructura de carpetas del módulo en el proyecto nuevo, coherente con el
resto del ERP.

**Criterio de salida de la Fase 2:** existe `PLANO_<MODULO>.md` con la opinión, el
destino de cada hallazgo, los contratos, el modelo de datos y la estructura.

---

## SECCIÓN 4 — FASE 3: PLAN DE ACCIÓN

**Objetivo:** traducir el diseño en **acciones concretas y verificables**.

### 4.1 Paso 3.1 — Definir las acciones arquitectónicas (A-XX)

Cada acción debe ser **ejecutable** y **verificable**. Formato:

```
A-XX — <título>
  Qué: <la acción>
  Por qué: <el hallazgo que resuelve>
  Cómo se verifica: <test, comando, o criterio observable>
  Criterio de aceptación: <condición binaria de "hecho">
```

### 4.2 Paso 3.2 — Construir la matriz de trazabilidad

Mapea **cada hallazgo** a la acción que lo resuelve:

| Hallazgo | Acción | Estado |
|----------|--------|--------|
| DEUDA-04 | A-04 | Pendiente |
| AC-01 | A-02 | Pendiente |
| DB-02 | A-05 | Pendiente |

**Si un hallazgo no tiene acción, el plan está incompleto.**

### 4.3 Paso 3.3 — Definir el orden de ejecución

Las acciones tienen dependencias. Ordénalas y explica por qué ese orden.

### 4.4 Las 5 acciones canónicas (reutilizables)

Estas 5 acciones se repiten en casi todos los módulos. Adáptalas, no las reinventes:

| Acción | Qué garantiza |
|--------|---------------|
| **A-01** | Portar las reglas **con sus tests** (no perder las cicatrices) |
| **A-02** | Frontera por contratos (el módulo no lee tablas ajenas) |
| **A-03** | Test guardián por regla crítica (el CI falla si se viola) |
| **A-04** | Outbox transaccional (elimina `try/except pass`) |
| **A-05** | Identidad (UUID) ≠ Presentación (folio) |

**Criterio de salida de la Fase 3:** existe `PLAN_ACCION_<MODULO>.md` con las acciones,
la matriz de trazabilidad completa y el orden de ejecución.

---

## SECCIÓN 5 — FASE 4: MODELO DE DESPLIEGUE

**Objetivo:** definir **dónde vive** el módulo y cómo se consolida.

### 5.1 No reinventar la topología

La topología es **global**, no por módulo. El módulo nuevo hereda el modelo
**hub-and-spoke** ya definido en [`CONTEXTO_SISTEMA_IA.md`](ESPECIFICACIONES DEL PROYECTO/CONTEXTO_SISTEMA_IA.md:112) §3.3:

- Cada sucursal corre un **ERP completo** con su propia base de datos.
- El **servidor central** consolida, no es transaccional.
- La sincronización es **al cierre del día** (default 23:30, configurable).
- La identidad global es **UUID v4**; el folio es local.

### 5.2 Lo que sí es por módulo

Para cada módulo, define:

- **Qué datos** de este módulo se consolidan al central.
- **Con qué frecuencia** (respetando el default de cierre del día).
- **Qué conflictos** pueden surgir y cómo se resuelven (el servidor de sucursal gana
  sobre su propio dominio; los catálogos son propiedad corporativa).

**Criterio de salida de la Fase 4:** el módulo tiene declarado su contrato de
consolidación, coherente con el modelo global.

---

## SECCIÓN 6 — FASE 5: PUNTO DE ENTRADA

**Objetivo:** que cualquier colaborador nuevo pueda empezar en **10 minutos**.

### 6.1 El documento de entrada

Escribe `GUIA_<MODULO>.md` con esta estructura fija:

1. **Bienvenida** — qué es este módulo, en una frase.
2. **La regla dura** — no se toca el ERP.
3. **Los documentos maestros** — qué leer y en qué orden.
4. **Las reglas de oro** — el resumen ejecutivo.
5. **Los errores que no debes cometer** — anti-patrones.
6. **Glosario** — para no perderse.
7. **Checklist de incorporación** — marcable.

### 6.2 La regla de los dos sistemas de numeración

**Advertencia obligatoria en toda guía:** existen dos sistemas de numeración y **no son
intercambiables**:

| Sistema | Rango | Dónde vive | Qué es |
|---------|-------|-----------|--------|
| **RN-XX** | RN-01 a RN-NN | Especificación funcional §C | Reglas de negocio (funcional) |
| **Regla N** | Regla 1 a Regla N | Documentación del módulo §4 | Reglas de oro (técnica, cicatrices) |

**Criterio de salida de la Fase 5:** existe `GUIA_<MODULO>.md` que permite empezar sin
leer los otros documentos primero.

---

## SECCIÓN 7 — FASE 6: AUTOCRÍTICA

**Objetivo:** verificar que lo escrito es **excelente**, no solo que existe.

### 7.1 El contraste obligatorio

Relee **las fuentes autoritativas** y contrasta contra lo que escribiste:

1. [`CONTEXTO_SISTEMA_IA.md`](ESPECIFICACIONES DEL PROYECTO/CONTEXTO_SISTEMA_IA.md:1) — la doctrina inamovible.
2. [`DOCUMENTACION_MODULO_<MODULO>.md`](ESPECIFICACIONES DEL PROYECTO/DOCUMENTACION_MODULO_POS.md:1) — la documentación del módulo.
3. **El código real** — la verdad última.

### 7.2 Las 3 preguntas de la autocrítica

| Pregunta | Qué busca |
|----------|-----------|
| **¿Es redundante?** | ¿Repito algo que ya está en una fuente autoritativa sin declararlo? |
| **¿Es ambiguo?** | ¿Uso términos que se pueden confundir (ej. RN-XX vs Regla N)? |
| **¿Es incompleto?** | ¿Omití algo que la fuente sí documenta (ej. las reglas de oro)? |

### 7.3 El formato del informe

```
D-1 — <defecto>            (severidad: ALTA/MEDIA/BAJA)
  Qué: <descripción>
  Evidencia: <dónde está el problema>
  Corrección: C-X

C-1 — <corrección>
  Documento afectado: <archivo>
  Cambio: <qué se añade/modifica>
```

### 7.4 Verificación final

- **ERP intacto:** `git status --short` vacío; HEAD sin cambios.
- **UTF-8:** sin BOM, cero caracteres de reemplazo.
- **Trazabilidad:** toda afirmación anclada a evidencia.

**Criterio de salida de la Fase 6:** existe `AUTOCRITICA_<MODULO>.md` con los defectos
encontrados y las correcciones aplicadas.

---

## SECCIÓN 8 — CHECKLIST MAESTRA (COPIAR Y PEGAR POR MÓDULO)

```
MÓDULO: ____________________    FECHA: __________

FASE 1 — INGENIERÍA INVERSA
[ ] Inventario de superficie (archivos y roles)
[ ] Lectura completa del código
[ ] Reglas de negocio extraídas (RN-XX) con archivo:línea
[ ] Funcionalidades extraídas (F-XX)
[ ] Modelo de datos documentado (¿PK es UUID o entero?)
[ ] Flujos documentados con puntos de fallo
[ ] Hallazgos catalogados (DEUDA / AC / DB / RC)
[ ] Interfaces inventariadas (pantallas, modales, paneles, overlays, composición, impresión)
[ ] Artefacto: ESPECIFICACION_FUNCIONAL_<MODULO>.md
[ ] Artefacto: ESPECIFICACION_DE_INTERFACES_<MODULO>.md (ficha de 7 puntos por interfaz)

FASE 2 — DISEÑO
[ ] Opinión sobre el enfoque actual
[ ] Destino decidido para CADA hallazgo (eliminar/conservar)
[ ] Contratos entre módulos definidos
[ ] Modelo de datos nuevo (UUID, UTC, ledger)
[ ] Estructura del repositorio nuevo
[ ] Modo responsivo de cada interfaz verificado contra R-01 a R-04
[ ] Artefacto: PLANO_<MODULO>.md

FASE 3 — PLAN DE ACCIÓN
[ ] Acciones arquitectónicas (A-XX) definidas y verificables
[ ] Matriz de trazabilidad completa (hallazgo → acción)
[ ] Orden de ejecución justificado
[ ] Artefacto: PLAN_ACCION_<MODULO>.md

FASE 4 — DESPLIEGUE
[ ] Datos a consolidar identificados
[ ] Frecuencia declarada (default: cierre del día 23:30)
[ ] Resolución de conflictos declarada
[ ] Coherencia con hub-and-spoke verificada

FASE 5 — PUNTO DE ENTRADA
[ ] GUIA_<MODULO>.md con las 7 secciones fijas
[ ] Advertencia de los dos sistemas de numeración incluida
[ ] Checklist de incorporación incluida

FASE 6 — AUTOCRÍTICA
[ ] Contraste contra las 3 fuentes autoritativas
[ ] Las 3 preguntas respondidas (redundante/ambiguo/incompleto)
[ ] Defectos documentados (D-XX) y correcciones aplicadas (C-XX)
[ ] ERP intacto verificado (git status vacío)
[ ] UTF-8 verificado (sin BOM, sin caracteres de reemplazo)
[ ] Artefacto: AUTOCRITICA_<MODULO>.md

MIGRACIÓN
[ ] Documentos copiados al repo PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS
[ ] Commit + push al repo de planos
[ ] Archivos .md borrados del repo del ERP
[ ] ERP verificado 100% limpio
```

---

## SECCIÓN 9 — LOS ERRORES QUE NO DEBES COMETER

| # | Error | Consecuencia |
|---|-------|--------------|
| **1** | Confundir cicatriz con deuda | Borras la protección y reproduces el bug |
| **2** | Escribir una regla sin ancla (`archivo:línea`) | Es una opinión, no un hallazgo |
| **3** | Dejar un hallazgo sin decisión | El accidente se reproduce en el módulo nuevo |
| **4** | Presentar como "nuevo" algo que ya está en la doctrina | Redundancia no declarada (defecto D-1) |
| **5** | Usar RN-XX y Regla N como sinónimos | Ambigüedad que confunde al colaborador |
| **6** | Omitir las reglas de oro del módulo | El colaborador no conoce las cicatrices |
| **7** | Tocar el ERP "para arreglar algo rapidito" | Rompes la caja de una panadería en horario pico |
| **8** | Dejar documentos del proyecto nuevo en el repo del ERP | Contaminas el repositorio de producción |

---

## SECCIÓN 10 — GLOSARIO DEL MÉTODO

| Término | Significado |
|---------|-------------|
| **Cicatriz** | Guarda nacida de un bug real. **No se quita.** |
| **Deuda** | Atajo que oculta un fallo. **Se elimina.** |
| **Ancla** | Evidencia verificable (`archivo:línea`, commit, test). |
| **RN-XX** | Regla de negocio numerada (funcional). |
| **F-XX** | Funcionalidad numerada. |
| **A-XX** | Acción arquitectónica. |
| **DEUDA-XX** | Deuda técnica catalogada. |
| **AC-XX** | Acoplamiento innecesario. |
| **DB-XX** | Debilidad de diseño. |
| **RC-XX** | Riesgo de concurrencia. |
| **D-XX** | Defecto encontrado en la autocrítica. |
| **C-XX** | Corrección aplicada. |
| **Regla N** | Regla de oro del módulo (técnica, cicatriz). |
| **Hub-and-spoke** | Topología: ERP completo por sucursal + central consolidador. |

---

## SECCIÓN 11 — DECLARACIÓN DE LA REGLA DURA

> **NO SE TOCA EL ERP INSTALADO Y CORRIENDO.**
> **NO SE TOCA NINGUNO DE SUS MÓDULOS.**
> **NO SE MODIFICA NI UNA LÍNEA DEL CÓDIGO DE PRODUCCIÓN.**

Este método produce **documentos** en el repositorio
`PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS`. El ERP permanece intacto y operando.

---

*Metodología de ingeniería inversa y diseño. Versión 1.0.
Derivada del trabajo realizado sobre el módulo POS (v22, commit `fe9f6ed`).
Aplicable a: Almacenes, Productos, RRHH, Estadísticas, Auditoría, Heladería,
Reparto, Monitoreo de Red, Pedidos, Perfiles.*
