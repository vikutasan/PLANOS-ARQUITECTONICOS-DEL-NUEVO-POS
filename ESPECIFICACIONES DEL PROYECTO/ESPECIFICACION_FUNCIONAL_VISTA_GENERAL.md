# ESPECIFICACIÓN FUNCIONAL — VISTA GENERAL

> **Documento:** 12 (nuevo)
> **Módulo:** Vista General (el "hub" de declaración de valores transversales)
> **Método:** Ingeniería inversa + diseño (FASE 1 de la metodología)
> **Anclaje:** ERP `ERP-R-DE-RICO-CON-POS-SIMPLIFICADO`, commit `5802f45` (V23)
> **Estado:** ✅ FASE 1 completada (22 Sep 2026)
> **Regla dura:** este documento es un artefacto de análisis. **No se ha modificado ni se modificará el ERP en producción.** Todo trabajo derivado (el nuevo POS) se realizará en un proyecto y repositorio separados.

---

## SECCIÓN A — PROPÓSITO Y MÉTODO

### A.1 Propósito

Vista General es el **módulo donde se declaran los valores que afectan a todos los módulos del ERP**. No es un módulo de negocio: es el **panel de configuración transversal** y la **pantalla de bienvenida** del sistema.

Su rol ya estaba decidido en [`DIRECTRICES_TRANSVERSALES_DEL_ERP.md`](../DIRECTRICES_TRANSVERSALES_DEL_ERP.md:283) (DT-06) y en [`PLANO ARQUITECTONICO PARA EL NUEVO POS.md`](../PLANO%20ARQUITECTONICO%20PARA%20EL%20NUEVO%20POS.md:867). Lo que faltaba era su **especificación funcional completa** (pantallas, campos, validaciones, permisos). Este documento la escribe.

**La pregunta que responde:** *¿Qué hace hoy Vista General, y qué debe hacer el Vista General del nuevo POS?*

### A.2 Método

1. **Lectura del código** (fuente de verdad): [`apps/ExperimentCenterUI.jsx`](../../apps/ExperimentCenterUI.jsx:1) (frontend, 622 líneas), [`apps/api/modules/settings/router.py`](../../apps/api/modules/settings/router.py:1), [`apps/api/modules/settings/service.py`](../../apps/api/modules/settings/service.py:1), [`apps/api/modules/settings/models.py`](../../apps/api/modules/settings/models.py:1), [`apps/api/core/timezone.py`](../../apps/api/core/timezone.py:1), [`apps/shared/TimezoneContext.jsx`](../../apps/shared/TimezoneContext.jsx:1).
2. **Abstracción**: se ignoró la estructura del código (nombres de componentes, hooks) y se describió el **comportamiento observable**: qué se ve, qué se edita, qué se persiste, qué se propaga.
3. **Extracción de reglas**: cada condicional, guarda, umbral y transición se tradujo a una regla de negocio numerada (VG-xx).
4. **Anclaje**: cada afirmación lleva `archivo:línea` verificable.

### A.3 Advertencia de nomenclatura

Vista General **no es** el "Centro de Experimentación". El archivo se llama `ExperimentCenterUI.jsx` por razones históricas (nació como demo del ecosistema), pero su módulo por defecto es `overview` y **es** la Vista General. El nombre del archivo es una **deuda de nomenclatura** (ver Hallazgo DB-VG-01).

---

## SECCIÓN B — INVENTARIO DE INTERFACES

Vista General tiene **4 interfaces** (1 pantalla raíz + 2 modales + 1 franja embebida):

| # | Interfaz | Tipo | Ancla |
|---|----------|------|-------|
| VG-I-01 | **Pantalla raíz** (reloj + calendario + semana) | Pantalla raíz | [`ExperimentCenterUI.jsx:374-407`](../../apps/ExperimentCenterUI.jsx:374) |
| VG-I-02 | **Franja de identidad del negocio** | Franja embebida | [`ExperimentCenterUI.jsx:386-395`](../../apps/ExperimentCenterUI.jsx:386) |
| VG-I-03 | **Modal "Información del Negocio"** | Modal (6 campos) | [`ExperimentCenterUI.jsx:409-473`](../../apps/ExperimentCenterUI.jsx:409) |
| VG-I-04 | **Modal de advertencia de cambio de zona horaria** | Modal de confirmación | [`ExperimentCenterUI.jsx:475-503`](../../apps/ExperimentCenterUI.jsx:475) |

---

## SECCIÓN C — REGLAS DE NEGOCIO (VG-XX)

### C.1 Identidad y navegación

- **VG-01** — Vista General es el **módulo por defecto** al iniciar sesión, salvo que la URL declare `?terminal=DRIVER` (abre Reparto Grandeza) o `?module=heladeria` (abre Heladería). Ancla: [`ExperimentCenterUI.jsx:74-76`](../../apps/ExperimentCenterUI.jsx:74).
- **VG-02** — Vista General **no tiene botón de salida propio**: vive dentro del shell del ERP (el sidebar). Ancla: [`ExperimentCenterUI.jsx:374`](../../apps/ExperimentCenterUI.jsx:374).
- **VG-03** — La navegación hacia otro módulo se **intercepta** si el POS tiene cuentas sin guardar (`window.requestPOSExit`). Ancla: [`ExperimentCenterUI.jsx:236-243`](../../apps/ExperimentCenterUI.jsx:236).

### C.2 Reloj y calendario

- **VG-04** — El reloj se actualiza **cada segundo** (`setInterval` de 1000 ms). Ancla: [`ExperimentCenterUI.jsx:100-103`](../../apps/ExperimentCenterUI.jsx:100).
- **VG-05** — La hora y la fecha se formatean con la **zona horaria del negocio** (`timezone` del `TimezoneContext`), no con la del navegador. Ancla: [`ExperimentCenterUI.jsx:375-377`](../../apps/ExperimentCenterUI.jsx:375).
- **VG-06** — El formato de fecha es `es-MX` con día de la semana, año, mes y día. Ancla: [`ExperimentCenterUI.jsx:376`](../../apps/ExperimentCenterUI.jsx:376).
- **VG-07** — El formato de hora es `es-MX`, 2 dígitos, 24 horas (`hour12: false`), con `fontVariantNumeric: 'tabular-nums'` para que los dígitos no "bailen". Ancla: [`ExperimentCenterUI.jsx:377`](../../apps/ExperimentCenterUI.jsx:377) y [`400`](../../apps/ExperimentCenterUI.jsx:400).
- **VG-08** — El **número de semana** se calcula como `ceil((díaDelAño + díaDeLaSemanaDel1Enero + 1) / 7)`. Ancla: [`ExperimentCenterUI.jsx:378-381`](../../apps/ExperimentCenterUI.jsx:378).
- **VG-09** — El número de semana se muestra en mayúsculas con tracking amplio y color naranja (`text-orange-500`). Ancla: [`ExperimentCenterUI.jsx:404`](../../apps/ExperimentCenterUI.jsx:404).

### C.3 Identidad del negocio (franja negra)

- **VG-10** — La franja muestra **6 datos**: nombre del negocio, nombre de la sucursal, dirección, teléfono, y (en el modal) moneda y zona horaria. Ancla: [`ExperimentCenterUI.jsx:387-391`](../../apps/ExperimentCenterUI.jsx:387).
- **VG-11** — El nombre del negocio se muestra en `text-8xl font-black uppercase`. Ancla: [`ExperimentCenterUI.jsx:387`](../../apps/ExperimentCenterUI.jsx:387).
- **VG-12** — El teléfono se muestra en naranja (`text-orange-400`). Ancla: [`ExperimentCenterUI.jsx:391`](../../apps/ExperimentCenterUI.jsx:391).
- **VG-13** — El botón "Editar" **solo aparece** si el usuario tiene el permiso `editar_info_negocio` o `all === 'full'`. Ancla: [`ExperimentCenterUI.jsx:392`](../../apps/ExperimentCenterUI.jsx:392).

### C.4 Carga y persistencia de datos

- **VG-14** — Los datos del negocio se cargan de `GET /settings/` y se filtran a **6 claves**: `business_name`, `branch_name`, `business_address`, `business_phone`, `business_currency`, `business_currency_symbol`. Ancla: [`ExperimentCenterUI.jsx:114-122`](../../apps/ExperimentCenterUI.jsx:114).
- **VG-15** — Si la carga falla, hay **degradación elegante**: se conservan los valores por defecto del estado inicial (`R de Rico`, `Sucursal San Pablo`, `MXN`, `$`). Ancla: [`ExperimentCenterUI.jsx:106`](../../apps/ExperimentCenterUI.jsx:106) y [`125`](../../apps/ExperimentCenterUI.jsx:125).
- **VG-16** — El guardado hace **un PATCH por cada clave** modificada (`PATCH /settings/{key}`), no un PUT masivo. Ancla: [`ExperimentCenterUI.jsx:132-138`](../../apps/ExperimentCenterUI.jsx:132).
- **VG-17** — Tras guardar, el estado local se actualiza **sin recargar** (`setBizInfo(prev => ({...prev, ...bizForm}))`). Ancla: [`ExperimentCenterUI.jsx:139`](../../apps/ExperimentCenterUI.jsx:139).
- **VG-18** — Si el guardado falla, se muestra `alert('Error al guardar')`. Ancla: [`ExperimentCenterUI.jsx:141`](../../apps/ExperimentCenterUI.jsx:141).

### C.5 Moneda (DT-02 + DT-06, V23)

- **VG-19** — El selector de moneda ofrece **3 opciones**: MXN (`$`), USD (`US$`), EUR (`€`). Ancla: [`ExperimentCenterUI.jsx:437-439`](../../apps/ExperimentCenterUI.jsx:437).
- **VG-20** — Al cambiar la moneda, se actualizan **dos claves a la vez**: `business_currency` (código ISO) y `business_currency_symbol` (símbolo). Ancla: [`ExperimentCenterUI.jsx:432-435`](../../apps/ExperimentCenterUI.jsx:432).
- **VG-21** — El selector **declara, no convierte**: el texto de ayuda dice explícitamente *"Solo declara la moneda para formateo en UI. NO convierte montos existentes."* Ancla: [`ExperimentCenterUI.jsx:441`](../../apps/ExperimentCenterUI.jsx:441).
- **VG-22** — La moneda se persiste en `system_settings` con categoría `business` e `input_type` `text`. Ancla: [`service.py:181-194`](../../apps/api/modules/settings/service.py:181).
- **VG-23** — El endpoint `GET /settings/currency` devuelve `{currency, symbol}` con fallback `MXN`/`$` si la clave no existe. Ancla: [`router.py:28-46`](../../apps/api/modules/settings/router.py:28).

### C.6 Zona horaria (DT-01 + DT-06)

- **VG-24** — El selector de zona horaria ofrece **12 opciones** (4 de México + 4 de Latinoamérica + 3 de EEUU + 1 de España). Ancla: [`ExperimentCenterUI.jsx:452-463`](../../apps/ExperimentCenterUI.jsx:452).
- **VG-25** — Al cambiar la zona horaria, **NO se guarda de inmediato**: se abre un **modal de advertencia** (`tzWarning`). Ancla: [`ExperimentCenterUI.jsx:445-450`](../../apps/ExperimentCenterUI.jsx:445).
- **VG-26** — El modal de advertencia lista **4 impactos**: horas en todo el ERP, check-in/check-out, reportes y estadísticas, y la regla de día de negocio (5 AM). Ancla: [`ExperimentCenterUI.jsx:485-488`](../../apps/ExperimentCenterUI.jsx:485).
- **VG-27** — El modal muestra el cambio solicitado con el valor viejo tachado en rojo y el nuevo en verde. Ancla: [`ExperimentCenterUI.jsx:491-493`](../../apps/ExperimentCenterUI.jsx:491).
- **VG-28** — El modal aclara: *"Los datos existentes no se modifican."* Ancla: [`ExperimentCenterUI.jsx:495`](../../apps/ExperimentCenterUI.jsx:495).
- **VG-29** — Solo al **confirmar** se aplica el cambio al formulario (`setBizForm`). Ancla: [`ExperimentCenterUI.jsx:499`](../../apps/ExperimentCenterUI.jsx:499).
- **VG-30** — El endpoint `GET /settings/timezone` devuelve `{timezone, offset_hours}`. Ancla: [`router.py:16-23`](../../apps/api/modules/settings/router.py:16).
- **VG-31** — **ORDEN CRÍTICO de rutas**: `/timezone` y `/currency` deben declararse **antes** de `/{key}`, o FastAPI interpreta "timezone"/"currency" como una clave y devuelve 404. Ancla: [`router.py:14-15`](../../apps/api/modules/settings/router.py:14) y [`25-27`](../../apps/api/modules/settings/router.py:25).

### C.7 Permisos

- **VG-32** — El acceso a Vista General **no requiere permiso especial**: es el módulo por defecto. Ancla: [`ExperimentCenterUI.jsx:76`](../../apps/ExperimentCenterUI.jsx:76).
- **VG-33** — La **edición** de la info del negocio requiere `editar_info_negocio` o `all === 'full'`. Ancla: [`ExperimentCenterUI.jsx:392`](../../apps/ExperimentCenterUI.jsx:392).
- **VG-34** — El filtrado de módulos visibles usa permisos granulares si existen; si no, cae a la lógica de roles legacy (`mod.access.includes(userRole)`). Ancla: [`ExperimentCenterUI.jsx:203-215`](../../apps/ExperimentCenterUI.jsx:203).

---

## SECCIÓN D — MODELO DE DATOS

### D.1 Tabla `system_settings`

| Columna | Tipo | Restricción | Nota |
|---------|------|-------------|------|
| `id` | Integer | PK | — |
| `key` | String | único | La clave de configuración |
| `value` | String | — | El valor (siempre texto; los JSON se serializan) |
| `description` | String | — | Descripción legible |
| `category` | String | NOT NULL | `business`, `polling`, `security`, `heladeria`, `general`… |
| `input_type` | String | NOT NULL | `text`, `number`, `json`… |

Ancla: [`models.py:4-12`](../../apps/api/modules/settings/models.py:4).

### D.2 Las 6 claves de Vista General

| Clave | Valor por defecto | Categoría | input_type |
|-------|-------------------|-----------|------------|
| `business_name` | `R de Rico` | business | text |
| `branch_name` | `Sucursal San Pablo` | business | text |
| `business_address` | (vacío) | business | text |
| `business_phone` | (vacío) | business | text |
| `business_currency` | `MXN` | business | text |
| `business_currency_symbol` | `$` | business | text |
| `business_timezone` | `America/Mexico_City` | business | text |

Ancla: [`ExperimentCenterUI.jsx:106`](../../apps/ExperimentCenterUI.jsx:106) y [`service.py:181-194`](../../apps/api/modules/settings/service.py:181).

### D.3 Reparación de filas históricas (BUG 4)

- **VG-35** — `seed_settings` repara filas con `category` o `input_type` en NULL, porque **un solo registro NULL hacía que `GET /settings/` devolviera HTTP 500** (ResponseValidationError), rompiendo la carga de Vista General. Ancla: [`service.py:204-218`](../../apps/api/modules/settings/service.py:204).

---

## SECCIÓN E — FLUJOS

### E.1 Flujo: cargar Vista General

```
1. El usuario inicia sesión → activeModule = 'overview' (VG-01).
2. useEffect dispara GET /settings/ (VG-14).
3. Se filtran las 6 claves de negocio (VG-14).
4. Si hay datos, se fusionan con los defaults (VG-15).
5. Si falla, se conservan los defaults (degradación elegante, VG-15).
6. El reloj arranca su setInterval de 1 s (VG-04).
7. Se renderiza la franja negra + el reloj + la semana (VG-10 a VG-12).
```

### E.2 Flujo: editar la info del negocio

```
1. El usuario con permiso pulsa "Editar" (VG-13).
2. Se abre el modal con bizForm = copia de bizInfo (VG-16).
3. El usuario edita los campos.
4. Si cambia la zona horaria → modal de advertencia (VG-25 a VG-29).
5. Si cambia la moneda → se actualizan código + símbolo juntos (VG-20).
6. Al pulsar "Guardar" → un PATCH por clave (VG-16).
7. El estado local se actualiza sin recargar (VG-17).
8. Si falla → alert (VG-18).
```

### E.3 Flujo: cambio de zona horaria (el flujo crítico)

```
1. El usuario cambia el <select> de zona horaria.
2. NO se guarda: se abre tzWarning = { oldTz, newTz } (VG-25).
3. El modal lista los 4 impactos (VG-26).
4. Si cancela → tzWarning = null, no pasa nada (VG-29).
5. Si confirma → bizForm.business_timezone = newTz (VG-29).
6. El guardado real ocurre al pulsar "Guardar" en el modal principal.
```

---

## SECCIÓN F — HALLAZGOS CATALOGADOS

| ID | Tipo | Hallazgo | Ancla |
|----|------|----------|-------|
| **DB-VG-01** | Debilidad de diseño | El archivo se llama `ExperimentCenterUI.jsx` pero es la Vista General. Deuda de nomenclatura. | [`ExperimentCenterUI.jsx:68`](../../apps/ExperimentCenterUI.jsx:68) |
| **DB-VG-02** | Debilidad de diseño | El guardado hace **N PATCH secuenciales** sin transacción: si el 3.º falla, los 2 primeros ya se guardaron. No hay rollback. | [`ExperimentCenterUI.jsx:132-138`](../../apps/ExperimentCenterUI.jsx:132) |
| **DB-VG-03** | Debilidad de diseño | El error de guardado usa `alert()` nativo, no un toast del sistema. | [`ExperimentCenterUI.jsx:141`](../../apps/ExperimentCenterUI.jsx:141) |
| **DB-VG-04** | Debilidad de diseño | El cálculo del número de semana es una aproximación propia, no ISO 8601. Puede diferir del estándar. | [`ExperimentCenterUI.jsx:378-381`](../../apps/ExperimentCenterUI.jsx:378) |
| **DB-VG-05** | Debilidad de diseño | El `catch` de la carga usa `/* degradacion elegante */` vacío: un fallo de red es invisible para el operador. | [`ExperimentCenterUI.jsx:125`](../../apps/ExperimentCenterUI.jsx:125) |
| **AC-VG-01** | Acoplamiento | Vista General conoce el `CONFIG.API_BASE_URL` y arma las URLs a mano. Debería usar un servicio. | [`ExperimentCenterUI.jsx:114`](../../apps/ExperimentCenterUI.jsx:114) |
| **AC-VG-02** | Acoplamiento | El modal de moneda hardcodea el mapeo código→símbolo en el `onChange`. Debería ser una tabla declarativa. | [`ExperimentCenterUI.jsx:434`](../../apps/ExperimentCenterUI.jsx:434) |
| **DEUDA-VG-01** | Deuda | No existe `MoneyContext.jsx`: la moneda se declara pero **ningún módulo la consume todavía**. Es la deuda de V24. | [`DIRECTRICES_TRANSVERSALES_DEL_ERP.md:118`](../DIRECTRICES_TRANSVERSALES_DEL_ERP.md:118) |

---

## SECCIÓN G — DISEÑO DEL VISTA GENERAL DEL NUEVO POS

### G.1 Lo que se conserva idéntico (cicatrices)

| Cicatriz | Por qué se conserva |
|----------|---------------------|
| **Modal de advertencia de zona horaria** | Nació de un bug real: cambiar la TZ sin avisar rompía reportes históricos. Es una guarda, no un adorno. |
| **"Declarar, no convertir"** | La moneda declara el símbolo; nunca reescribe montos. Evita corromper datos históricos. |
| **Degradación elegante** | Si `GET /settings/` falla, la Vista General sigue mostrando algo. El ERP no se cae por un fallo de red. |
| **Reparación de filas NULL (BUG 4)** | Un solo NULL rompía todo el endpoint. La reparación es una cicatriz de producción. |
| **Orden crítico de rutas** | `/timezone` y `/currency` antes de `/{key}`. Es una cicatriz de FastAPI. |

### G.2 Lo que se corrige (deuda)

| Deuda | Corrección en el nuevo POS |
|-------|---------------------------|
| **DB-VG-02** (N PATCH sin transacción) | Un solo `PUT /settings/business` que guarda las 6 claves en **una transacción**. |
| **DB-VG-03** (`alert()`) | Toast del sistema, consistente con el resto del ERP. |
| **DB-VG-05** (`catch` vacío) | Registrar el fallo y mostrar un aviso discreto ("no se pudo cargar la config"). |
| **AC-VG-01** (URLs a mano) | Un `settingsService.js` centraliza las llamadas. |
| **AC-VG-02** (mapeo hardcodeado) | Una tabla declarativa `CURRENCIES = [{code, symbol, label}]`. |
| **DEUDA-VG-01** (sin `MoneyContext`) | Crear `MoneyContext.jsx` espejo de `TimezoneContext.jsx`, y migrar los ~80 `toFixed(2)`. |
| **DB-VG-01** (nombre del archivo) | Renombrar a `VistaGeneralUI.jsx` en el proyecto nuevo. |

### G.3 Lo que se añade (nuevo)

| Añadido | Justificación |
|---------|---------------|
| **Sucursal (`sucursal_id`)** | DT-06 declara 3 valores transversales: zona horaria, moneda y **sucursal**. Hoy la sucursal es solo texto (`branch_name`); debe ser una FK a la tabla `sucursales` (O-18 del Documento 8). |
| **Vista de solo lectura para no-admin** | Hoy el botón "Editar" se oculta; en el nuevo POS conviene mostrar los valores en modo lectura explícito. |
| **Validación de campos** | Hoy no hay validación (se puede guardar un nombre vacío). El nuevo POS valida nombre y sucursal no vacíos. |

### G.4 Contrato con los demás módulos

Vista General **escribe** `system_settings`; los demás módulos **leen** por contexto:

```
Vista General ──escribe──> system_settings
                                │
                                ├──> GET /settings/timezone ──> TimezoneContext ──> todos los módulos
                                └──> GET /settings/currency ──> MoneyContext (V24) ──> todos los módulos
```

**Regla dura (DT-06):** ningún módulo define ni sobrescribe estos valores. Solo los consumen.

---

## SECCIÓN H — CRITERIOS DE ACEPTACIÓN

| # | Criterio | Verificación |
|---|----------|--------------|
| **VG-CA-01** | La hora se muestra en la zona del negocio, no la del navegador | Cambiar la TZ del SO y verificar que la hora no cambia |
| **VG-CA-02** | Cambiar la TZ abre el modal de advertencia | Interacción manual |
| **VG-CA-03** | Cancelar el modal no cambia nada | Interacción manual |
| **VG-CA-04** | Cambiar la moneda actualiza código y símbolo juntos | Inspeccionar `system_settings` |
| **VG-CA-05** | El guardado es transaccional (todo o nada) | Simular fallo en la 3.ª clave |
| **VG-CA-06** | Si `GET /settings/` falla, la Vista General sigue usable | Cortar la red y recargar |
| **VG-CA-07** | Un usuario sin `editar_info_negocio` no ve el botón "Editar" | Login con rol restringido |
| **VG-CA-08** | `GET /settings/currency` devuelve `{currency, symbol}` | `curl` al endpoint |
| **VG-CA-09** | `GET /settings/timezone` devuelve `{timezone, offset_hours}` | `curl` al endpoint |
| **VG-CA-10** | Ningún módulo define valores transversales por su cuenta | Búsqueda estática de `business_timezone`/`business_currency` fuera de Vista General |

---

## SECCIÓN I — TRAZABILIDAD

| Documento | Relación |
|-----------|----------|
| [`DIRECTRICES_TRANSVERSALES_DEL_ERP.md`](../DIRECTRICES_TRANSVERSALES_DEL_ERP.md:283) | DT-06 declara el rol de Vista General. Este documento lo especifica. |
| [`PLANO ARQUITECTONICO PARA EL NUEVO POS.md`](../PLANO%20ARQUITECTONICO%20PARA%20EL%20NUEVO%20POS.md:867) | §3.3 declara los 3 valores transversales. |
| [`MODELO_DE_DATOS_DEL_NUEVO_POS.md`](../MODELO_DE_DATOS_DEL_NUEVO_POS.md:541) | O-18 (tabla `sucursales`) y O-19 (`business_currency`). |
| [`CRITERIOS_DE_ACEPTACION_DEL_NUEVO_POS.md`](../CRITERIOS_DE_ACEPTACION_DEL_NUEVO_POS.md:291) | CA-21 (dinero por un solo formateador). |
| [`METODOLOGIA_DE_INGENIERIA_INVERSA_Y_DISENO.md`](../METODOLOGIA_DE_INGENIERIA_INVERSA_Y_DISENO.md:93) | Las 6 fases. Este documento es la FASE 1 de Vista General. |

---

## SECCIÓN J — LO QUE ESTE DOCUMENTO **NO** RESUELVE

| Pendiente | Etapa | Descripción |
|-----------|-------|-------------|
| `MoneyContext.jsx` | V24 | El contexto global de dinero no existe. Este documento lo declara como deuda, no lo implementa. |
| Migración de los ~80 `toFixed(2)` | V24 | Alcance de V24, no de esta FASE 1. |
| Tabla `sucursales` | Etapa 3 | O-18 del Documento 8. Se declara aquí, se construye en el modelo de datos. |
| Especificación de interfaces (ficha de 7 puntos) | FASE 2 | Este documento es FASE 1 (funcional). La ficha visual/táctil de las 4 interfaces es FASE 2. |

---

**Especificación funcional de Vista General. Versión 1.0.**
Derivada del trabajo de ingeniería inversa sobre `ExperimentCenterUI.jsx` (commit `5802f45`, V23).
FASE 1 de la metodología. El ERP permanece intacto y operando.
