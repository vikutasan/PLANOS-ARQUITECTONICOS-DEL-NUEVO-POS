# 📋 PLAN DE ABORDAJE — FASE 10.5: PARIDAD DE DATOS DE CAJA

> **Documento:** `PLAN_DE_ABORDAJE_F10_5_PARIDAD_DE_DATOS_DE_CAJA.md`
> **Versión:** 1.0
> **Fecha:** 30 de Septiembre de 2026
> **Autor:** Antigravity + Víctor
> **Estado:** PROPUESTO — pendiente de aprobación
> **Depende de:** F10.4 (Contexto diario post-corte) — se ejecuta DESPUÉS
> **Cierra:** Brechas #1 (nombre del cajero) y #2 (desglose del resumen) del análisis de la analogía del trasplante

---

## 1. EL HALLAZGO

### 1.1 La analogía del trasplante de corazón

El usuario planteó la situación con una analogía precisa:

> *"Imaginemos que tenemos un paciente (el ERP viejo) cuyo corazón (el POS viejo) tiene las venas tapadas y es débil... los médicos construyeron un nuevo corazón de alta tecnología (el nuevo POS) para trasplantárselo. Para que no sea rechazado por su cuerpo deben asegurarse que haga las funciones vitales con extrema eficiencia... los doctores saben que correrá sangre a determinada presión. Por eso me preocupa que los modales de apertura de caja y gestión de caja sean tan distintos: al hacerlos tan distintos, el receptor los rechace o no se integren bien."*

La analogía es correcta y separa dos cosas que hay que distinguir con precisión:

| Tipo de diferencia | ¿Es riesgo? | Por qué |
|---|---|---|
| **Diferencia de IMPLEMENTACIÓN** (cómo se ve el modal, qué botones tiene, cómo se navega) | **NO** | Es intencional. El nuevo POS reescribe la implementación. §6.8: *"la integración se hereda, la implementación se reescribe."* |
| **Diferencia de INTEGRACIÓN** (qué DATOS entran y salen del módulo de caja) | **SÍ** | Si el nuevo POS no produce/consume los mismos datos que el viejo, el resto del ERP (Estadísticas, Auditoría, Reportes) **no puede leerlo**. Eso es el "rechazo del órgano". |

### 1.2 La evidencia decisiva (verificada, no asumida)

Se comparó **campo por campo** el contrato de resumen de caja del viejo ERP contra el del nuevo POS.

**Viejo ERP — `CashSummaryResponse`** (`apps/api/modules/cash/schemas.py:55-64`):

```python
class CashSummaryResponse(BaseModel):
    efectivo_esperado: float
    total_credito: float
    total_debito: float
    total_ventas: float
    num_transacciones: int
    fondo_inicial: float
    total_entradas: float
    total_salidas: float
```

**Nuevo POS — `ResumenTurnoSalida`** (`../NUEVO-POS/apps/api/schemas.py:353-361`):

```python
class ResumenTurnoSalida(BaseModel):
    esperado: Decimal
    movimientos: list[MovimientoResumen] = Field(default_factory=list)
```

### 1.3 La tabla de paridad de datos

| Campo del viejo ERP | ¿Existe en el nuevo POS? | Destino |
|---|---|---|
| `efectivo_esperado` | ✅ SÍ → `esperado` | PORTADO |
| `total_credito` | ❌ NO | **BRECHA #2** |
| `total_debito` | ❌ NO | **BRECHA #2** |
| `total_ventas` | ❌ NO | **BRECHA #2** |
| `num_transacciones` | ❌ NO | **BRECHA #2** |
| `fondo_inicial` | ❌ NO | **BRECHA #2** |
| `total_entradas` | ❌ NO | **BRECHA #2** |
| `total_salidas` | ❌ NO | **BRECHA #2** |

**7 de 8 campos del resumen financiero NO se exponen.** El nuevo POS calcula internamente casi todos ellos y luego **los tira a la basura**.

### 1.4 El agravante: el dato YA ESTÁ CALCULADO

En `../NUEVO-POS/apps/api/routers/cash.py:88-126`, la función `_ventas_en_efectivo` ya hace la clasificación completa:

```python
clasificado = rn58_clasificacion_alimenta_resumen(pagos)
return clasificado.get("EFECTIVO", Decimal("0.00"))
```

`rn58_clasificacion_alimenta_resumen` devuelve `{EFECTIVO, CREDITO, DEBITO, TRANSFERENCIA}` — **los cuatro métodos clasificados**. Pero la función solo devuelve `EFECTIVO` y **descarta** crédito, débito y transferencia.

Lo mismo con los movimientos: `_sumar_movimientos` (líneas 142-152) ya calcula `(entradas, salidas)` — y el resumen solo usa ambos para calcular `esperado`, sin exponerlos.

**El dato existe. El cálculo existe. Solo no se publica.** Esto es exactamente la misma clase de falla que `payment_details` (F9.1.4a): *el dato se produce pero no se expone en el contrato*.

### 1.5 La brecha #1: el nombre del cajero

En `../NUEVO-POS/apps/api/routers/cash.py:219-225`, al abrir el turno:

```python
sesion = CashSession(
    terminal_id=entrada.terminal_id,
    employee_id=entrada.usuario_id,
    employee_name=str(entrada.usuario_id),   # ← el nombre es el ID
    opening_float=fondo,
    status="OPEN",
)
```

El viejo POS envía `employee_name` real (`cashService.abrirSesion({terminal_id, employee_id, employee_name, opening_float})`). El nuevo POS **no recibe el nombre** en `AbrirTurnoEntrada` y lo rellena con el UUID del usuario. El corte impreso y el reporte diario mostrarán un UUID en lugar del nombre del cajero.

---

## 2. LA CAUSA RAÍZ

### 2.1 La cuarta instancia de la misma clase de falla

Esta es la **cuarta vez** que aparece la misma clase de falla en el proyecto:

| # | Instancia | Fase | Qué pasó |
|---|---|---|---|
| 1 | `GestorDeCaja` huérfano | F4.5 | El componente existía y pasaba su test, pero nadie podía llegar a él |
| 2 | `payment_details` no expuesto | F9.1.4a | El dato se guardaba pero no se exponía en el contrato |
| 3 | "Copiar URL" ausente | F10 / B-01 | La función existía en el viejo POS, no se portó |
| 4 | "Contexto diario" ausente | F10.4 / B-02 | El modal existía en el viejo POS, no se portó |
| **5** | **Resumen de caja incompleto** | **F10.5** | **El dato se calcula pero no se expone en el contrato** |

### 2.2 La lección que F10.5 añade

Las lecciones ya codificadas en el Plan Maestro:

- **§10.6.1 (F4.5):** *"el componente existe y pasa su test" ≠ "el usuario puede llegar a él"*.
- **§10.6.2 (F10):** *"el componente existe y pasa su test" ≠ "el conjunto está completo"*.
- **§10.6.3 (F10.4):** *"el inventario de componentes no ve las integraciones"*.

**La lección de F10.5:**

> **§10.6.4 — "el inventario de componentes no ve los FLUJOS DE DATOS."**
> Un componente puede estar portado, integrado y pasar su test, y aun así **producir menos datos de los que el resto del ERP espera consumir**. La paridad no es solo de componentes: es también de **contratos de datos**. Hay que auditar campo por campo lo que el viejo POS producía y el nuevo produce.

### 2.3 Por qué F10 no lo detectó

F10 auditó **componentes** (¿existe el modal? ¿existe el botón? ¿existe el flujo?). No auditó **contratos de datos** (¿el resumen expone los mismos campos?). El inventario de componentes no ve los flujos de datos — exactamente la lección §10.6.4.

---

## 3. EL HECHO ARQUITECTÓNICO DECISIVO

### 3.1 El consumidor de estos datos YA EXISTE

No es hipotético. El módulo de **Estadísticas** del viejo ERP ya consume estos datos:

- **Backend:** `apps/api/modules/analytics/` (`router.py`, `models.py`, `service.py`).
- **Frontend:** `apps/analytics/` (`EstadisticasVentasUI.jsx`, `ProductStatsView.jsx`).
- **Tabla:** `daily_contexts` (con `target_date` único, `is_atypical`, `weather_condition`, `holiday_name`, `notes`, `temperature_c`).
- **Endpoints:** `GET/PUT /api/v1/analytics/context`, `GET /analytics/rankings`, `GET /analytics/product-daily-sales`.

El contrato **6** (`pos.resumen_de_venta`) ya declara: **Consumidor = Estadísticas, Proveedor = POS**. Es decir, la frontera ya reconoce que Estadísticas lee del POS. Si el POS no expone el desglose, Estadísticas no puede alimentarse.

### 3.2 La frontera por contratos (A-02) exige la paridad

La regla A-02 prohíbe que un módulo lea las tablas de otro. Estadísticas **no puede** leer `cash_sessions` ni `cash_movements` directamente. Solo puede leer lo que el POS **exponga por contrato**. Si el contrato 12 expone 2 campos, Estadísticas solo ve 2 campos. **La paridad de datos es una condición de la frontera, no un lujo.**

---

## 4. ALCANCE

### 4.1 Qué SÍ entra en F10.5

1. **Brecha #2 — Desglose del resumen:** ampliar `ResumenTurnoSalida` (contrato 12) para exponer los 7 campos faltantes: `total_credito`, `total_debito`, `total_ventas`, `num_transacciones`, `fondo_inicial`, `total_entradas`, `total_salidas`.
2. **Brecha #1 — Nombre del cajero:** añadir `usuario_nombre` a `AbrirTurnoEntrada` (contrato 10) y persistirlo en `employee_name` en lugar del UUID.
3. **Refactor de `_ventas_en_efectivo`:** renombrar a `_clasificar_ventas_del_turno` y devolver el diccionario completo `{EFECTIVO, CREDITO, DEBITO, TRANSFERENCIA}` en vez de solo el efectivo.
4. **Actualizar el registro de contratos** (contratos 10 y 12) con los campos nuevos.
5. **Actualizar el frontend** (`cashService.js` + `GestorDeCaja.jsx`) para consumir y mostrar el desglose.
6. **Tests de paridad** que verifiquen campo por campo que el nuevo contrato expone lo mismo que el viejo.
7. **Ficha F10.5** + actualización de `FICHA_F10_PARIDAD.md`.

### 4.2 Qué NO entra en F10.5 (frontera explícita)

- **NO** se toca el módulo de Estadísticas (eso es F11: "profesionalizar/acoplar Estadísticas").
- **NO** se crea la tabla `daily_contexts` (pertenece a Estadísticas — decisión 4b de F10.4).
- **NO** se cambia la UX del modal de caja (la implementación se reescribe; solo se añade la visualización del desglose).
- **NO** se portan campos que el viejo POS tampoco exponía.
- **NO** se toca el cálculo de `esperado` (RN-53 ya es correcto).

### 4.3 Principio rector

> **El ERP viejo es el ORÁCULO, no el MODELO (§8.2).**
> No copiamos su código (es parchado). Copiamos su **CONTRATO DE DATOS**: qué información producía, para que el nuevo POS sea un órgano compatible. La implementación es nueva; la interfaz de datos es heredada.

---

## 5. DISEÑO TÉCNICO

### 5.1 Contrato 12 — `caja.resumen_del_turno` (ampliado)

**Antes:**
```python
class ResumenTurnoSalida(BaseModel):
    esperado: Decimal
    movimientos: list[MovimientoResumen] = Field(default_factory=list)
```

**Después:**
```python
class ResumenTurnoSalida(BaseModel):
    """Salida del contrato 12: la PROYECCIÓN del turno, no la tabla.

    Paridad de datos con el viejo ERP (F10.5): expone el desglose completo
    que el módulo de Estadísticas consume (contrato 6). `esperado` es el
    efectivo que debería haber en la caja (RN-53).
    """

    esperado: Decimal
    fondo_inicial: Decimal
    total_entradas: Decimal
    total_salidas: Decimal
    total_credito: Decimal
    total_debito: Decimal
    total_ventas: Decimal
    num_transacciones: int
    movimientos: list[MovimientoResumen] = Field(default_factory=list)
```

### 5.2 Contrato 10 — `caja.abrir_turno` (ampliado)

**Antes:**
```python
class AbrirTurnoEntrada(BaseModel):
    terminal_id: str
    usuario_id: str
    monto_inicial: Decimal
```

**Después:**
```python
class AbrirTurnoEntrada(BaseModel):
    terminal_id: str
    usuario_id: str
    usuario_nombre: str | None = None   # F10.5: nombre real del cajero
    monto_inicial: Decimal
```

Y en `routers/cash.py:219-225`:
```python
sesion = CashSession(
    terminal_id=entrada.terminal_id,
    employee_id=entrada.usuario_id,
    employee_name=entrada.usuario_nombre or str(entrada.usuario_id),
    opening_float=fondo,
    status="OPEN",
)
```

### 5.3 Refactor de la clasificación de ventas

**Antes** (`routers/cash.py:88-126`): `_ventas_en_efectivo` devuelve solo el efectivo.

**Después:** `_clasificar_ventas_del_turno` devuelve el diccionario completo:

```python
async def _clasificar_ventas_del_turno(
    db: AsyncSession, cash_session_id: UUID
) -> dict[str, Decimal]:
    """Clasifica las ventas del turno por método (RN-57, RN-58).

    F10.5: devuelve el desglose COMPLETO {EFECTIVO, CREDITO, DEBITO,
    TRANSFERENCIA} — no solo el efectivo. El resumen (contrato 12) y el
    reporte diario (contrato 14) consumen este desglose.
    """
    # ... (misma lógica de lectura de pagos que hoy) ...
    return rn58_clasificacion_alimenta_resumen(pagos)
```

Y el resumen:
```python
clasificado = await _clasificar_ventas_del_turno(db, cash_session_id)
ventas_efectivo = clasificado.get("EFECTIVO", Decimal("0.00"))
esperado = rn53_efectivo_esperado(
    Decimal(str(sesion.opening_float)), entradas, salidas, ventas_efectivo
)
total_ventas = sum(clasificado.values())
return ResumenTurnoSalida(
    esperado=esperado,
    fondo_inicial=Decimal(str(sesion.opening_float)),
    total_entradas=entradas,
    total_salidas=salidas,
    total_credito=clasificado.get("CREDITO", Decimal("0.00")),
    total_debito=clasificado.get("DEBITO", Decimal("0.00")),
    total_ventas=total_ventas,
    num_transacciones=len(tickets),
    movimientos=[...],
)
```

### 5.4 Frontend — `cashService.js`

El servicio ya devuelve `{outcome, reason, data}`. No cambia su firma; solo el `data` de `obtenerResumen` ahora trae más campos. **Cero cambios estructurales** — solo se enriquece el payload.

### 5.5 Frontend — `GestorDeCaja.jsx`

En el estado `CERRADO` (líneas 372-463), el bloque de arqueo hoy muestra `esperado`, `capturado` y `descuadre`. Se añade un **desglose de paridad** (fondo, entradas, salidas, crédito, débito, total de ventas, número de transacciones) — el mismo que el viejo POS mostraba en su sección "Balance".

### 5.6 Registro de contratos

Actualizar en `../NUEVO-POS/apps/api/contracts/registry.py`:
- **Contrato 10** (líneas 284-295): añadir `usuario_nombre` a la entrada.
- **Contrato 12** (líneas 313-327): ampliar la salida con los 7 campos nuevos.

---

## 6. SUB-FASES

### F10.5.0 — Escribir este plan y presentarlo
- **Entregable:** este documento.
- **Compuerta:** aprobación del usuario.

### F10.5.1 — Ampliar los contratos 10 y 12 (backend)
- Añadir `usuario_nombre` a `AbrirTurnoEntrada`.
- Ampliar `ResumenTurnoSalida` con los 7 campos.
- Refactorizar `_ventas_en_efectivo` → `_clasificar_ventas_del_turno`.
- Actualizar `routers/cash.py` (abrir_turno + resumen_del_turno).
- Actualizar el registro de contratos.
- **Compuerta:** `pytest` verde con los tests nuevos.

### F10.5.2 — Tests de paridad de datos
- Test que verifica que `ResumenTurnoSalida` expone los 8 campos del viejo `CashSummaryResponse`.
- Test que verifica que `employee_name` guarda el nombre real, no el UUID.
- Test que verifica que el desglose clasifica correctamente un turno con pagos mixtos.
- **Compuerta:** los 3 tests pasan.

### F10.5.3 — Frontend: consumir y mostrar el desglose
- `cashService.js`: sin cambios estructurales (el payload se enriquece).
- `GestorDeCaja.jsx`: añadir el bloque de desglose en el estado CERRADO.
- **Compuerta:** test de componente verde.

### F10.5.4 — CI verde + ficha F10.5
- `npm run ci` verde.
- Escribir `FICHA_F10_5_PARIDAD_DE_DATOS_DE_CAJA.md`.
- Actualizar `FICHA_F10_PARIDAD.md` (brechas #1 y #2 → CERRADAS).
- **Compuerta:** CI verde + ficha escrita.

### F10.5.5 — Commit + push
- Commit en `NUEVO-POS` (código + tests + ficha).
- Commit en `PLANOS` (este plan + actualización del Plan Maestro §10.6.4).
- Registrar el hash real en la ficha (patrón ficha-hash).
- **Compuerta:** push confirmado.

---

## 7. CRITERIOS DE ACEPTACIÓN

1. `ResumenTurnoSalida` expone los 8 campos del viejo `CashSummaryResponse` (paridad campo por campo).
2. `AbrirTurnoEntrada` acepta `usuario_nombre` y `employee_name` guarda el nombre real.
3. `_clasificar_ventas_del_turno` devuelve el desglose completo de los 4 métodos.
4. Un turno con pagos mixtos ($40 efectivo + $60 tarjeta) reporta `total_credito`/`total_debito` correctos y `esperado` solo con el efectivo.
5. El frontend muestra el desglose en el estado CERRADO.
6. Los 3 tests de paridad pasan.
7. `npm run ci` verde.
8. El registro de contratos refleja los campos nuevos.
9. `FICHA_F10_PARIDAD.md` marca las brechas #1 y #2 como CERRADAS.
10. El Plan Maestro incorpora §10.6.4 (la lección de los flujos de datos).

---

## 8. DEUDA TRAZABLE

| Deuda | Dueño | Fase que la cierra |
|---|---|---|
| El módulo de Estadísticas no está acoplado al nuevo POS | F11 | F11 |
| La tabla `daily_contexts` no existe en el nuevo POS | F11 | F11 |
| El reporte diario (contrato 14) no expone el desglose por método | F10.5 (opcional) / F11 | F10.5 o F11 |

---

## 9. REFERENCIAS

- **§6.8 (Plan Maestro):** la UX heredada — la integración se hereda, la implementación se reescribe.
- **§8.2 (Plan Maestro):** el ERP viejo es el ORÁCULO, no el MODELO.
- **§10.6.1 (F4.5):** el paso de INTEGRACIÓN también es una compuerta.
- **§10.6.2 (F10):** la COMPLETITUD del conjunto también es una compuerta.
- **§10.6.3 (F10.4):** el inventario de componentes no ve las integraciones.
- **§10.6.4 (F10.5, este plan):** el inventario de componentes no ve los FLUJOS DE DATOS.
- **Contrato 6:** `pos.resumen_de_venta` — Consumidor: Estadísticas, Proveedor: POS.
- **Contrato 10:** `caja.abrir_turno` — entrada `{terminal_id, usuario_id, monto_inicial}`.
- **Contrato 12:** `caja.resumen_del_turno` — salida `{esperado, movimientos}`.
- **Viejo ERP:** `apps/api/modules/cash/schemas.py:55-64` (`CashSummaryResponse`), `apps/api/modules/cash/service.py:96-122` (`calcular_resumen`).
- **Nuevo POS:** `../NUEVO-POS/apps/api/schemas.py:353-361` (`ResumenTurnoSalida`), `../NUEVO-POS/apps/api/routers/cash.py:88-126` (`_ventas_en_efectivo`), `:274-299` (resumen), `:187-230` (abrir_turno).
- **RN-53:** `rn53_efectivo_esperado(fondo, entradas, salidas, ventas_efectivo)`.
- **RN-57:** `rn57_clasificar_por_metodo(metodo)`.
- **RN-58:** `rn58_clasificacion_alimenta_resumen(pagos)`.
- **RN-60:** `rn60_resumen_distingue_esperado_y_contado(esperado, contado)`.
- **DT-02:** el dinero viaja como String en el cable.
