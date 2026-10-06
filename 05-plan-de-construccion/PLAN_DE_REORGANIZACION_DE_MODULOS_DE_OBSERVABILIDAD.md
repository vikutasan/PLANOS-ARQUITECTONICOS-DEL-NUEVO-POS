# 🧭 PLAN DE REORGANIZACIÓN DE MÓDULOS DE OBSERVABILIDAD Y CONFIGURACIÓN

> **Fecha:** 6 Oct 2026
> **Versión:** 1.0
> **Autor:** Antigravity + Víctor (dueño de R de Rico)
> **Estado:** APROBADO por el dueño — decisión de arquitectura registrada, ejecución diferida por módulo
> **Documento padre:** [`PLAN_MAESTRO_DEFINITIVO_POS.md`](../PLAN_MAESTRO_DEFINITIVO_POS.md:1) §8.1 (el POS es el primer módulo de un ERP reconstruido) + §8.2 (el ERP viejo se rehará módulo por módulo)
> **Precedente directo:** [`PLAN_DE_ABORDAJE_FASE_13_POR_PARTES.md`](PLAN_DE_ABORDAJE_FASE_13_POR_PARTES.md:1) (Auditoría y Control)
> **Documentación del ERP viejo (el oráculo):**
> - [`DOCUMENTACION_MODULO_MONITOREO_DE_RED.md`](../../ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_MODULO_MONITOREO_DE_RED.md:1)
> - [`DOCUMENTACION_VISTA_GENERAL.md`](../../ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_VISTA_GENERAL.md:1)
> - [`DOCUMENTACION_MODULO_AUDITORIA_Y_CONTROL.md`](../../ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_MODULO_AUDITORIA_Y_CONTROL.md:1)

> [!IMPORTANT]
> **Este documento es un artefacto de diseño.** No contiene código de producción.
> El código va en `NUEVO-POS`; los planos van en `PLANOS-ARQUITECTONICOS` (Plan Maestro §1).

> [!CAUTION]
> **LECTURA OBLIGATORIA ANTES DE CUALQUIER OTRA COSA: §2 (la decisión) y §6 (por qué NO se ejecuta ahora).**
> Este plan **NO se ejecuta parcheando el ERP viejo**. Registra una decisión de arquitectura que se
> materializa **cuando cada módulo se reconstruya** en el ERP nuevo (§8.2). Ejecutarlo hoy sería
> añadir un parche más al ERP que está en proceso de reemplazo.

---

## 0. PROPÓSITO DE ESTE DOCUMENTO

El ERP viejo tiene cuatro módulos cuyas responsabilidades se **solapan parcialmente** y cuyos nombres
ya no describen con precisión lo que hacen:

1. **Ajustes del Sistema** (`SystemSettingsUI.jsx`) — mezcla configuración de negocio con parámetros de ingeniería.
2. **Visión General** (`ExperimentCenterUI.jsx`, sección overview) — configuración de negocio.
3. **Monitoreo de Red** (`NetworkMonitorUI.jsx`) — observación de la infraestructura LAN.
4. **Auditoría y Control** (`AuditoriaControlUI.jsx`) — observación de la operación de negocio.

Este documento:

1. **Registra la decisión del dueño** sobre el reparto de responsabilidades (§2).
2. **Justifica** cada asignación con evidencia del repositorio (§3).
3. **Define la frontera** entre observabilidad de infraestructura y de negocio (§4).
4. **Resuelve el caso del "control de polling"** → "monitoreo de polling" (§5).
5. **Delimita** por qué la ejecución se difiere por módulo (§6).
6. **Se autocritica** contra la evidencia (§7).

---

## 1. DIAGNÓSTICO — EL SOLAPAMIENTO ACTUAL

### 1.1 Qué hace hoy cada módulo (evidencia)

| Módulo | Archivo gobernado | Qué hace hoy | Evidencia |
|---|---|---|---|
| **Ajustes del Sistema** | [`apps/settings/SystemSettingsUI.jsx`](../../apps/settings/SystemSettingsUI.jsx:7) | Lista TODOS los `system_settings` agrupados por categoría; expone `polling_ms`, `ttl_m`, `heartbeat_interval_ms` con explicaciones de impacto | [`SystemSettingsUI.jsx:51`](../../apps/settings/SystemSettingsUI.jsx:51) (`getImpactExplanation`) |
| **Visión General** | [`apps/ExperimentCenterUI.jsx`](../../apps/ExperimentCenterUI.jsx:1) (sección overview) | Encabezado institucional + reloj + modal de edición de 6 campos de negocio | [`DOCUMENTACION_VISTA_GENERAL.md:44`](../../ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_VISTA_GENERAL.md:44) |
| **Monitoreo de Red** | [`apps/network/NetworkMonitorUI.jsx`](../../apps/network/NetworkMonitorUI.jsx:1) | Latencia, 6 estados de terminal, historial de incidentes (`network_incidents`) | [`DOCUMENTACION_MODULO_MONITOREO_DE_RED.md:22`](../../ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_MODULO_MONITOREO_DE_RED.md:22) |
| **Auditoría y Control** | [`apps/AuditoriaControlUI.jsx`](../../apps/AuditoriaControlUI.jsx:1) | Tickets (quién capturó/cobró), cortes de caja, reporte diario consolidado | [`DOCUMENTACION_MODULO_AUDITORIA_Y_CONTROL.md:10`](../../ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_MODULO_AUDITORIA_Y_CONTROL.md:10) |

### 1.2 El solapamiento real

El problema no es que los módulos se dupliquen, sino que **Ajustes del Sistema mezcla dos dominios
distintos** en una sola pantalla:

- **Configuración de NEGOCIO** (nombre, dirección, moneda, zona horaria del negocio) — que **también**
  vive en Visión General.
- **Configuración de INGENIERÍA** (`polling_ms`, `ttl_m`, `heartbeat_interval_ms`) — que es tuning
  de infraestructura.

Además, el **"control de polling"** (configurar el polling) es un concepto del **POS viejo**. En el
POS nuevo, el estado de terminal se resuelve por **lock con TTL + heartbeat**
([`useTerminalLocking.js`](../../NUEVO-POS/apps/pos/src/hooks/useTerminalLocking.js:35)), no por un
polling configurable por el usuario. Lo que **sigue siendo vigente** es **observar** que la red y los
latidos funcionan — y eso es exactamente lo que hace Monitoreo de Red.

> **Conclusión del diagnóstico:** el solapamiento se resuelve **separando por dominio** (negocio vs.
> ingeniería vs. observación), no fusionando módulos. La decisión del dueño (§2) hace precisamente eso.

---

## 2. LA DECISIÓN (aprobada por el dueño)

> **DECISIÓN DE REORGANIZACIÓN — 6 Oct 2026:**
> Se conservan los **cuatro módulos**, cada uno con un dominio único y sin solapamiento:
>
> | Módulo | Dominio | Responsabilidad |
> |---|---|---|
> | **Ajustes del Sistema** | Configuración de **ingeniería** | Parámetros técnicos del sistema |
> | **Visión General** | Configuración de **negocio** | Datos institucionales de la sucursal |
> | **Monitoreo de Red** | **Observar** la infraestructura | Latencia, estados, incidentes |
> | **Auditoría y Control** | **Observar** la operación | Tickets, cortes, reporte diario, eventos auditables |
>
> Y se renombra la sección **"Control de polling" → "Monitoreo de polling"**, trasladando la
> responsabilidad de **configurar** el polling (ingeniería) a **observar** el polling (red).

### 2.1 El reparto, en una frase

> **Negocio → Visión General. Ingeniería → Ajustes del Sistema. Observar la red → Monitoreo de Red.
> Observar la operación → Auditoría y Control.**

---

## 3. JUSTIFICACIÓN DE CADA ASIGNACIÓN

### 3.1 Ajustes del Sistema = ingeniería

**Por qué:** los parámetros `polling_ms`, `ttl_m`, `heartbeat_interval_ms` son **decisiones de diseño
del POS**, no de operación. Un dueño de panadería no debe ajustar milisegundos de polling a mano.

**Evidencia:** [`SystemSettingsUI.jsx:51`](../../apps/settings/SystemSettingsUI.jsx:51) ya advierte del
riesgo ("Polling muy agresivo. Aumentará significativamente la carga del servidor"). Esa advertencia
es la prueba de que el parámetro es de ingeniería: si el usuario puede romper el sistema ajustándolo,
no debería estar en una pantalla de operación.

**Regla:** Ajustes del Sistema conserva **solo** parámetros técnicos. La configuración de negocio se
retira de aquí (ya vive en Visión General).

### 3.2 Visión General = negocio

**Por qué:** es la pantalla de inicio del ERP, la que ve **cualquier empleado** al entrar
([`DOCUMENTACION_VISTA_GENERAL.md:11`](../../ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_VISTA_GENERAL.md:11)).
Su contenido debe ser información institucional (nombre, dirección, teléfono, moneda, zona horaria),
no parámetros técnicos.

**Evidencia:** ya tiene el modal de 6 campos de negocio y el permiso `editar_info_negocio`
([`DOCUMENTACION_VISTA_GENERAL.md:44`](../../ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_VISTA_GENERAL.md:44)).

**Regla:** Visión General **absorbe** la configuración de negocio que hoy está duplicada en Ajustes
del Sistema. **NO absorbe** parámetros de ingeniería.

### 3.3 Monitoreo de Red = observar la infraestructura

**Por qué:** ya es el módulo técnico de observación de red. Su misión documentada es responder tres
preguntas operativas: ¿el servidor está vivo?, ¿qué terminales están ocupadas?, ¿hubo caídas?
([`DOCUMENTACION_MODULO_MONITOREO_DE_RED.md:22`](../../ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_MODULO_MONITOREO_DE_RED.md:22)).

**Evidencia:** es 100% pasivo respecto al POS (Regla de Oro #4,
[`DOCUMENTACION_MODULO_MONITOREO_DE_RED.md:445`](../../ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_MODULO_MONITOREO_DE_RED.md:445)).

**Regla:** Monitoreo de Red **asume** la observación de red (ya la tiene) y **absorbe** el setting
`network_tz_offset_hours`, que hoy vive en Ajustes del Sistema pero es un parámetro **del módulo de
red** ([`DOCUMENTACION_MODULO_MONITOREO_DE_RED.md:582`](../../ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_MODULO_MONITOREO_DE_RED.md:582) §14.3).

### 3.4 Auditoría y Control = observar la operación

**Por qué:** es el centro de monitoreo operativo donde la gerencia rastrea la trazabilidad de cada
centavo ([`DOCUMENTACION_MODULO_AUDITORIA_Y_CONTROL.md:10`](../../ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_MODULO_AUDITORIA_Y_CONTROL.md:10)).

**Evidencia:** ya cubre tickets, cortes de caja y reporte diario consolidado.

**Regla:** Auditoría y Control **absorbe** la superficie de consulta de los eventos auditables del POS
(el contrato 5, `GET /pos/auditable-events`, construido en F13.1). El **panel** de consulta de
auditoría vive **aquí**, no en el POS (§4.2).

---

## 4. LA FRONTERA: INFRAESTRUCTURA vs. NEGOCIO

### 4.1 Dos observabilidades distintas, complementarias

| | Monitoreo de Red | Auditoría y Control |
|---|---|---|
| **Observa** | La infraestructura (¿la red funciona?) | La operación (¿quién hizo qué?) |
| **Evidencia** | Técnica (latencia, incidentes) | Financiera (tickets, cortes, diferencias) |
| **Pregunta** | "¿Por qué se cayó la T2?" | "¿Quién cobró el ticket de $2,550?" |
| **Tabla** | `network_incidents` | `tickets`, `cash_sessions`, `pos_audit_log` |

**La doc del ERP ya los declara complementarios, no fusionados:**
> *"Los incidentes de red son evidencia operativa complementaria a la auditoría de operaciones
> sensibles."* — [`DOCUMENTACION_MODULO_MONITOREO_DE_RED.md:481`](../../ESPECIFICACIONES%20DEL%20PROYECTO/DOCUMENTACION_MODULO_MONITOREO_DE_RED.md:481) §11.3

### 4.2 Corolario para F13: dónde vive el panel de auditoría

Esto **cierra el círculo de F13.2**:

- El contrato 5 declara como consumidor a **Auditoría**, no al POS
  ([`contracts/registry.py:218`](../../NUEVO-POS/apps/api/contracts/registry.py:218)).
- El `pos_audit_log` (F13.0) vive en la base del POS porque el POS **escribe** la evidencia
  (P-01: el dueño escribe su tabla).
- El módulo de Auditoría **lee** esa evidencia por contrato (P-02: el consumidor pregunta por operación).
- Por lo tanto: **F13.2a (el `auditService.js`) sí procede** — es la capa de servicio reutilizable.
  **F13.2b (el panel dentro del POS) NO procede** — el panel pertenece al módulo de Auditoría y Control.

> **Regla:** el POS **produce** la evidencia; Auditoría la **consume**. No confundir "el log vive en
> el POS" con "el panel vive en el POS".

---

## 5. EL CASO DEL "CONTROL DE POLLING" → "MONITOREO DE POLLING"

### 5.1 La distinción de vocabulario

| Término | Verbo | Dominio | Módulo |
|---|---|---|---|
| **Control de polling** | *Configurar* el polling | Ingeniería | Ajustes del Sistema |
| **Monitoreo de polling** | *Observar* el polling | Observación | Monitoreo de Red |

### 5.2 Por qué el renombre es correcto

El cambio de nombre captura el traslado de **configurar** a **observar**:

- El **ajuste** de `polling_ms`/`ttl_m`/`heartbeat_interval_ms` se queda en **Ajustes del Sistema**
  (ingeniería).
- La **observación** de que la red y los latidos funcionan se va a **Monitoreo de Red**.

### 5.3 Advertencia: no reintroducir la idea de "ajustar el polling"

En el POS nuevo, el polling configurable **ya casi no existe** — el estado de terminal se resuelve por
lock+TTL+heartbeat ([`useTerminalLocking.js`](../../NUEVO-POS/apps/pos/src/hooks/useTerminalLocking.js:35)).
"Monitoreo de polling" en la práctica será **"monitoreo de la salud de la red y de los latidos"**, que
es lo que Monitoreo de Red ya hace. El nombre es correcto; solo hay que asegurar que **no reintroduzca**
la idea de que el usuario *ajusta* el polling desde ahí.

---

## 6. POR QUÉ NO SE EJECUTA AHORA (ejecución diferida por módulo)

### 6.1 El principio §8.2

> *"El ERP viejo existe, pero su arquitectura es parchada — se rehará módulo por módulo."*
> — [`PLAN_MAESTRO_DEFINITIVO_POS.md:786`](../PLAN_MAESTRO_DEFINITIVO_POS.md:786)

Los cuatro módulos afectados viven en el **ERP viejo** (`apps/settings/`, `apps/network/`,
`apps/AuditoriaControlUI.jsx`, `apps/ExperimentCenterUI.jsx`). Parchearlos ahora sería **añadir un
parche más** al ERP que está en proceso de reemplazo.

### 6.2 La regla de ejecución

> **REGLA DE EJECUCIÓN:** esta reorganización se materializa **cuando cada módulo se reconstruya** en
> el ERP nuevo. La decisión queda **registrada como arquitectura** (este documento), no como un parche.

| Módulo | Cuándo se ejecuta la reorganización |
|---|---|
| **Auditoría y Control** | Cuando se reconstruya el módulo (absorbe el panel de eventos auditables de F13) |
| **Monitoreo de Red** | Cuando se reconstruya (absorbe `network_tz_offset_hours`; renombra "Control"→"Monitoreo" de polling) |
| **Visión General** | Cuando se reconstruya (absorbe la config de negocio duplicada) |
| **Ajustes del Sistema** | Cuando se reconstruya (retira la config de negocio; conserva solo ingeniería) |

### 6.3 Lo que SÍ se hace ahora

- **Registrar la decisión** en este documento (hecho).
- **Cerrar F13** con el alcance correcto: F13.2a sí, F13.2b no (§4.2).
- **NO tocar** el ERP viejo.

---

## 7. AUTOCRÍTICA

### 7.1 ¿Es esta reorganización un "parche más"?

**Riesgo detectado:** reorganizar módulos del ERP viejo podría ser exactamente el tipo de parche que
§8.2 advierte. **Mitigación:** el plan **no se ejecuta ahora** (§6). Solo registra la decisión para
que, al reconstruir cada módulo, se construya ya con el reparto correcto. Es documentación de
arquitectura, no un parche.

### 7.2 ¿Se está inflando el alcance? (§10.6.2)

**Riesgo detectado:** crear un plan de reorganización podría inflar el trabajo de F13. **Mitigación:**
este plan es **independiente de F13**. F13 cierra con F13.2a (el servicio). La reorganización es una
decisión de arquitectura del ERP, no una sub-fase de F13.

### 7.3 ¿Falta algún módulo?

**Verificación:** se revisaron los módulos que tocan configuración u observación. Los cuatro
identificados (Ajustes, Visión General, Monitoreo de Red, Auditoría) son los únicos con solapamiento.
Otros módulos (POS, Almacenes, RRHH, etc.) tienen dominios claros y no se ven afectados.

### 7.4 ¿El renombre "Control → Monitoreo" rompe algo?

**Verificación:** el renombre es de **sección de UI**, no de contrato ni de tabla. No afecta
`system_settings` ni `network_incidents`. Es seguro.

---

## 8. RESUMEN DE LA DECISIÓN

| Módulo | Dominio | Cambio respecto a hoy |
|---|---|---|
| **Ajustes del Sistema** | Ingeniería | Retira la config de negocio (ya en Visión General) |
| **Visión General** | Negocio | Absorbe la config de negocio duplicada |
| **Monitoreo de Red** | Observar red | Absorbe `network_tz_offset_hours`; renombra "Control"→"Monitoreo" de polling |
| **Auditoría y Control** | Observar operación | Absorbe el panel de eventos auditables (contrato 5, F13) |

**Ejecución:** diferida por módulo (§6). **Registrada como arquitectura, no como parche.**

---

## 9. QUÉ NO ENTRA EN ESTE PLAN

- ❌ **NO** se toca el ERP viejo ahora.
- ❌ **NO** se fusionan Monitoreo de Red y Auditoría (son complementarios, §4.1).
- ❌ **NO** se mete configuración de ingeniería en Visión General.
- ❌ **NO** se construye el panel de auditoría dentro del POS (pertenece a Auditoría, §4.2).
- ❌ **NO** se reintroduce el polling configurable por el usuario (§5.3).
