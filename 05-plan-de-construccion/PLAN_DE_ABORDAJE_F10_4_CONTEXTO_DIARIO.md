# 📋 PLAN DE ABORDAJE — FASE 10.4 "Contexto diario post-corte" (brecha B-02)

> **Fecha:** 30 Sep 2026
> **Versión:** 1.0
> **Autor:** Antigravity + Víctor (dueño de R de Rico)
> **Estado:** PROPUESTO — pendiente de aprobación
> **Origen:** Hallazgo del dueño al **poner a prueba la auditoría F10** (30 Sep 2026).
> **Padre:** `PLAN_DE_ABORDAJE_FASE_10_PARIDAD.md` (F10.0–F10.3 cerradas)

---

## 1. EL HALLAZGO (por qué existe esta sub-fase)

Al probar la auditoría de paridad F10, el dueño recordó una funcionalidad del viejo POS:

> *"en el viejo pos se había establecido que al momento de sacar el corte de caja apareciera
> un modal que preguntara cómo había estado el día, si lluvioso, si caluroso, etc. y dejaba
> poner alguna nota. Esto alimentaba al módulo de estadísticas para toma de decisiones de
> producción."*

**Verificación (REGLA DURA 2 — "verificar, no asumir"):** el hallazgo es **EXACTO**. La
funcionalidad existe completa en el viejo POS y está **AUSENTE** en el nuevo.

### 1.1 Evidencia en el viejo POS (frontend)

| Elemento | Ubicación | Qué hace |
|---|---|---|
| Estado del modal | [`GestorDeCaja.jsx:101-106`](../../../apps/pos/components/GestorDeCaja.jsx) | `showDailyContext`, `dailyContextSaved`, `ctxWeather`, `ctxAtypical`, `ctxNotes` |
| Disparo | [`GestorDeCaja.jsx:351`](../../../apps/pos/components/GestorDeCaja.jsx) | `handleConfirmarCierre` → `setShowDailyContext(true)` tras cerrar |
| Escritura | [`GestorDeCaja.jsx:363-385`](../../../apps/pos/components/GestorDeCaja.jsx) | `PUT ${API}/analytics/context?target_date=...` con `{target_date, is_atypical, weather_condition, notes}` |
| UI del modal | [`GestorDeCaja.jsx:712-777`](../../../apps/pos/components/GestorDeCaja.jsx) | "📝 ¿Cómo estuvo el día?" + 6 botones de clima + toggle ⚠️ Atípico + notas + Guardar/Omitir |

**Los 6 climas** ([`GestorDeCaja.jsx:717-724`](../../../apps/pos/components/GestorDeCaja.jsx)):
`SOLEADO ☀️`, `NUBLADO 🌤️`, `LLUVIA 🌧️`, `TORMENTA ⛈️`, `MUCHO_CALOR 🥵`, `FRIO ❄️`.

### 1.2 Evidencia en el viejo POS (backend — módulo Estadísticas)

| Elemento | Ubicación | Qué hace |
|---|---|---|
| Endpoints | [`apps/api/modules/analytics/router.py:13-21`](../../../apps/api/modules/analytics/router.py) | `GET /analytics/context` y `PUT /analytics/context` |
| Tabla | [`apps/api/modules/analytics/models.py:5-16`](../../../apps/api/modules/analytics/models.py) | `DailyContext`: `target_date` (único), `is_atypical`, `weather_condition`, `holiday_name`, `notes`, `temperature_c` |
| Servicio | [`apps/api/modules/analytics/service.py`](../../../apps/api/modules/analytics/service.py) | `get_or_create_daily_context`, `update_daily_context`, filtro `exclude_atypical` |
| Registro | [`apps/api/main.py:391`](../../../apps/api/main.py) | `analytics_router` montado en `/api/v1` |
| Frontend | [`apps/analytics/EstadisticasVentasUI.jsx`](../../../apps/analytics/EstadisticasVentasUI.jsx) | El módulo que **consume** `daily_context` |

### 1.3 Evidencia en el nuevo POS

- **NO existe** el modal de contexto diario en [`GestorDeCaja.jsx`](../NUEVO-POS/apps/pos/src/GestorDeCaja.jsx).
- `alCerrarTurno` ([`GestorDeCaja.jsx:175-191`](../NUEVO-POS/apps/pos/src/GestorDeCaja.jsx)) cierra el turno y **no dispara ningún modal**.
- **NO existe** módulo `analytics` en el backend del nuevo POS.

---

## 2. LA CAUSA RAÍZ (la lección que F10 debe aprender)

**El inventario F10.0 revisó los COMPONENTES del POS viejo, pero NO las INTEGRACIONES
POS→ERP.** El "Contexto diario" no es un componente del POS: es una **escritura del POS
hacia la tabla de otro módulo** (Estadísticas).

Es la **CUARTA instancia** de la misma clase de falla:

| # | Instancia | Fase | Naturaleza |
|---|---|---|---|
| 1 | `GestorDeCaja` huérfano | F4.5 | Componente sin punto de entrada |
| 2 | `payment_details` no expuesto | F9.1.4a | Dato persistido pero no expuesto |
| 3 | "Copiar URL" ausente | F10 / B-01 | Botón sin lógica ni UI |
| 4 | **"Contexto diario" ausente** | **F10.4 / B-02** | **Integración POS→ERP no portada** |

**La lección nueva (§10.6.3):** *"el inventario de componentes no ve las integraciones."*
Un POS puede tener todos sus componentes y aun así faltarle un **puente entre módulos**.

---

## 3. EL HECHO ARQUITECTÓNICO DECISIVO (corrección del supuesto)

> **El módulo Estadísticas YA EXISTE.** No hay que inventarlo.

El Plan Maestro §8.1 asumía que los módulos del ERP "aún no existen". El dueño corrigió
(30 Sep 2026): **existen y funcionan, pero con arquitectura parchada** (ver **§8.2** del
Plan Maestro). El trabajo del dueño es **rehacer cada módulo e integrarlos uno a uno**.

**Consecuencia para F10.4:** el POS nuevo **se acopla por contrato** al endpoint que
Estadísticas **ya expone** (`PUT /analytics/context`). No se inventa nada.

**La frontera A-02 se respeta así:**
- El POS **NO crea** la tabla `daily_contexts` (pertenece a Estadísticas).
- El POS **escribe vía el contrato** que Estadísticas expone.
- El POS es **cliente** de ese contrato; Estadísticas es **dueño** de la tabla.

---

## 4. ALCANCE (decidido por el dueño — Opción 4b)

**F10.4 cierra SOLO la brecha del POS.** Incluye:

1. **El modal post-corte** (`DailyContextModal`) — la UX heredada del viejo POS.
2. **El servicio** que llama al contrato existente de Estadísticas (`PUT /analytics/context`).
3. **La integración** en `GestorDeCaja.jsx` (se abre al cerrar el turno).
4. **El contrato declarado** en el registro del nuevo POS (POS → Estadísticas).
5. **Test + CI verde + ficha.**

**F10.4 NO incluye** (queda para **F11**):
- Profesionalizar/acoplar el módulo Estadísticas al nuevo ERP.
- Mover `apps/analytics/` y `apps/api/modules/analytics/` al NUEVO-POS.
- Cualquier cambio en la tabla `daily_contexts`.

> **Nota de honestidad:** como el módulo Estadísticas aún vive en el ERP viejo, el endpoint
> `PUT /analytics/context` **existe pero no está montado en el backend del nuevo POS**. El
> contrato se declara con `estado_hoy="Deuda"` (igual que el contrato 6 hoy). El modal
> funciona, arma el payload y lo envía; si el endpoint no responde, **degrada en silencio**
> (no bloquea al cajero — misma regla que el viejo POS).

---

## 5. DISEÑO TÉCNICO

### 5.1 El contrato nuevo (registro del nuevo POS)

Se declara un contrato nuevo en [`../NUEVO-POS/apps/api/contracts/registry.py`](../NUEVO-POS/apps/api/contracts/registry.py):

```python
Contrato(
    numero=28,  # siguiente número libre
    nombre="pos.contexto_diario",
    consumidor="Estadísticas",
    proveedor="POS",
    operacion="PUT /analytics/context?target_date={fecha}",
    entrada={
        "target_date": "Date (hora local)",
        "is_atypical": "Boolean",
        "weather_condition": "String NULL",
        "notes": "String NULL",
    },
    salida={"contexto": "DailyContextRead"},
    garantias=(
        "El POS ESCRIBE el contexto del día; Estadísticas es dueño de la tabla.",
        "Usa la fecha LOCAL (America/Mexico_City), no UTC.",
        "Si el endpoint no responde, el POS NO bloquea al cajero (degradación elegante).",
    ),
    errores=("400 si el formato de fecha es inválido.",),
    estado_hoy="Deuda",  # el endpoint vive en el ERP viejo; se acopla en F11
),
```

### 5.2 El servicio (`dailyContextService.js`)

Nuevo archivo `../NUEVO-POS/apps/pos/src/services/dailyContextService.js`:

- `guardarContexto({ target_date, is_atypical, weather_condition, notes })` → `{outcome, reason, data}`.
- Sigue el **patrón de servicio del proyecto**: nunca lanza, devuelve `{outcome, reason, data}`.
- Calcula la fecha local con `Intl.DateTimeFormat('en-CA', { timeZone: 'America/Mexico_City' })`.
- **Degradación elegante:** si falla, devuelve `{outcome: 'error', reason: 'sin_conexion'}` y el
  modal se cierra igual (no bloquea al cajero).

### 5.3 El modal (`DailyContextModal.jsx`)

Nuevo archivo `../NUEVO-POS/apps/pos/src/components/DailyContextModal.jsx`:

- Props: `{ abierto, onGuardar, onOmitir }`.
- UI heredada del viejo POS:
  - Título "📝 ¿Cómo estuvo el día?".
  - 6 botones de clima (SOLEADO/NUBLADO/LLUVIA/TORMENTA/MUCHO_CALOR/FRIO) con emoji.
  - Toggle "⚠️ Día atípico".
  - Campo de notas (texto libre).
  - Botones "✓ Guardar" y "Omitir →".
- **No bloqueante:** "Omitir" cierra sin guardar.
- Contenedor fluido (R-01), táctil (min-h-tactil), sin timers (prohibición #1).

### 5.4 La integración en `GestorDeCaja.jsx`

- Nuevo estado: `const [mostrarContexto, setMostrarContexto] = useState(false)`.
- En `alCerrarTurno`, tras `setDiferencia(r.data)` (línea 190), añadir `setMostrarContexto(true)`.
- Renderizar `<DailyContextModal abierto={mostrarContexto} ... />` al final del componente.
- Al guardar u omitir → `setMostrarContexto(false)`.

---

## 6. SUB-FASES

| Sub-fase | Qué hace | Gate |
|---|---|---|
| **F10.4.0** | Escribir este plan y presentarlo | Aprobación del dueño |
| **F10.4.1** | Declarar el contrato `pos.contexto_diario` en el registro | Test del registro (contrato existe, `estado_hoy="Deuda"`) |
| **F10.4.2** | Crear `dailyContextService.js` + `DailyContextModal.jsx` | Tests unitarios del servicio y del modal |
| **F10.4.3** | Integrar el modal en `GestorDeCaja.jsx` | Test de integración (el modal se abre al cerrar) |
| **F10.4.4** | CI verde + ficha `FICHA_F10_4_CONTEXTO_DIARIO.md` + actualizar `FICHA_F10_PARIDAD.md` (B-02) | `npm run ci` verde |
| **F10.4.5** | Commit + push (NUEVO-POS y PLANOS) | Commits en `origin/main` |

---

## 7. CRITERIOS DE ACEPTACIÓN

1. El contrato `pos.contexto_diario` existe en el registro con `estado_hoy="Deuda"`.
2. `dailyContextService.guardarContexto()` devuelve `{outcome, reason, data}` y **nunca lanza**.
3. El servicio usa la **fecha local** (America/Mexico_City), no UTC.
4. `DailyContextModal` renderiza los 6 climas, el toggle atípico y las notas.
5. El modal se abre **al cerrar el turno** en `GestorDeCaja`.
6. "Omitir" cierra el modal **sin guardar** y **sin bloquear**.
7. Si el endpoint falla, el modal se cierra igual (degradación elegante).
8. `npm run ci` verde.
9. `FICHA_F10_4_CONTEXTO_DIARIO.md` escrita con evidencia.
10. `FICHA_F10_PARIDAD.md` actualizada: B-02 pasa de "OMITIDA" a "PORTADA (con deuda de acople)".

---

## 8. LO QUE ESTA SUB-FASE DEJA EXPLÍCITO (deuda trazable)

- **Deuda de acople:** el endpoint `PUT /analytics/context` vive en el ERP viejo. El POS nuevo
  lo declara por contrato; **F11** lo acopla cuando se profesionalice Estadísticas.
- **Deuda de módulo:** "profesionalizar y acoplar Estadísticas al nuevo ERP" es **F11**, con su
  propio plan, su auditoría de paridad y su ficha.

> **Regla que se respeta:** *"El ERP viejo es el ORÁCULO, no el MODELO."* Se hereda el
> **comportamiento** (el modal, el contrato), no la **arquitectura parchada**.

---

## 9. REFERENCIAS

- Plan Maestro §8.2 — "El ERP viejo existe, pero su arquitectura es parchada".
- Plan Maestro §10.6.1 — "el paso de INTEGRACIÓN también es una compuerta".
- Plan Maestro §10.6.2 — "la COMPLETITUD del conjunto también es una compuerta".
- `PLAN_DE_ABORDAJE_FASE_10_PARIDAD.md` — F10.0–F10.3.
- `FICHA_F10_PARIDAD.md` — brecha B-02.
- Viejo POS: [`GestorDeCaja.jsx`](../../../apps/pos/components/GestorDeCaja.jsx), [`analytics/router.py`](../../../apps/api/modules/analytics/router.py).
