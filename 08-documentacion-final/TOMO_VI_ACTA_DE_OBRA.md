# TOMO VI — ACTA DE OBRA

> **Documentación final del POS nuevo "R de Rico"**
> **Tomo VI de VII — El acta de obra: las 84 fichas de construcción.**
> **Estado:** COMPLETO.
> **Fuente primaria:** [`NUEVO-POS/docs/05-plan-de-construccion/`](../NUEVO-POS/docs/05-plan-de-construccion/) (84 fichas + README).
> **Predecesores:** Tomo I (Visión y arquitectura), Tomo II (Las 95 reglas), Tomo III (Cementerio de bugs), Tomo IV (Contratos y fronteras), Tomo V (Superficie e interfaces).

---

## §0. Cómo leer este tomo

Este tomo es el **acta de obra** del POS nuevo. No repite la arquitectura (Tomo I), ni las reglas (Tomo II), ni los contratos (Tomo IV), ni las interfaces (Tomo V). Su función es **una sola**: dejar constancia, fase por fase, de **qué se construyó, en qué orden, con qué puerta de salida y con qué evidencia**.

Cada fase del POS nuevo se construyó por **fichas**. Una ficha es un documento corto que registra una sub-fase: qué se construyó, la decisión de diseño, la puerta que la cierra y por qué lo que NO procedía no procedía. Las fichas viven en el repo de código, en [`NUEVO-POS/docs/05-plan-de-construccion/`](../NUEVO-POS/docs/05-plan-de-construccion/). Son **84**.

Este tomo es el **índice razonado** de esas 84 fichas: las agrupa por fase, explica la lógica de cada fase y cita la ficha que la documenta. Si un lector futuro quiere el detalle de una sub-fase, va a su ficha. Si quiere entender **por qué** la obra se levantó en ese orden, lee este tomo.

**Regla de lectura:** este tomo NO sustituye a las fichas. Las **cita**. La ficha es la fuente; el tomo es el mapa.

---

## §1. La regla de avance por puertas

El POS nuevo no se construyó "por features". Se construyó **por fases con puertas**. La regla, escrita en [`NUEVO-POS/docs/05-plan-de-construccion/README.md`](../NUEVO-POS/docs/05-plan-de-construccion/README.md), es una sola:

> **Ninguna fase empieza sin que la PUERTA de la anterior esté en verde. Si una puerta falla, se corrige la fase actual; no se avanza "dejando pendiente".**

Esta regla es la **columna vertebral** del acta de obra. Cada fase termina con una **puerta**: un conjunto de criterios verificables (tests, greps de CI, paridad funcional) que deben estar en verde para declarar la fase cerrada. La puerta no es una formalidad: es el **contrato de calidad** de la fase.

**Por qué importa:** la lección de la Fase 10 (Tomo I, §10.6.2) es que **la completitud del conjunto también es una compuerta**. No basta con que cada pieza funcione; el conjunto debe estar completo. La regla de avance por puertas es lo que impide que un "casi terminado" se convierta en un "pendiente eterno".

**Las 3 lecciones de las micro-fases (Tomo I, §10.6.1–§10.6.5):**
1. **F4.5** — el paso de INTEGRACIÓN también es una compuerta. Montar las piezas no es integrarlas.
2. **F10** — la COMPLETITUD del conjunto también es una compuerta. Un inventario de piezas no ve lo que falta.
3. **F10.4 / F10.5 / F10.6** — el inventario de componentes no ve las **integraciones**, ni los **flujos de datos**, ni la **paridad de operación**.

Estas lecciones se incorporaron al acta: cada fase se cierra con una puerta que verifica no solo las piezas, sino su **integración**, sus **flujos** y su **paridad** con el POS viejo.

---

## §2. Las 7 fases y sus puertas

El POS nuevo se levantó en **7 fases** (F0 a F6), más las fases de extensión (F7 a F13) que portaron funcionalidad del POS viejo y añadieron auditoría. La tabla canónica de las 7 fases base, tomada del README del plan de construcción:

| Fase | Nombre | Puerta de salida |
|------|--------|------------------|
| **F0** | Andamiaje | CI en verde con 0 tests + 5 greps activos |
| **F1** | Cimiento de datos | Migraciones aplican y revierten limpias |
| **F2** | Frontera (contratos) | Test de arquitectura: 0 imports ajenos |
| **F3** | Comportamiento (reglas) | Matriz `regla → test` completa |
| **F4** | Guardianes | CI falla si se viola una regla crítica |
| **F5** | Superficie (interfaces) | Paridad funcional con el POS actual |
| **F6** | Consolidación (central) | Sync de cierre de día verificada |

**Las fases de extensión (F7 a F13):**

| Fase | Nombre | Puerta de salida |
|------|--------|------------------|
| **F7** | Voz + Visión IA + Temas | Los 3 modos de topología IA operan sin bloquear la venta |
| **F8** | CRM y Notificaciones (lado POS) | Identificación de cliente + envío por outbox, sin bloquear la venta |
| **F9** | Rescate de UX + Pagos mixtos | Paridad de UX con el viejo POS + pagos mixtos cuadran |
| **F10** | Auditoría de Paridad | Paridad de datos + operación + flujos con el viejo POS |
| **F11** | Verificación Plan vs Realidad | El plan coincide con lo construido |
| **F12** | Paridad con el viejo POS | Las 19 funcionalidades portadas operan |
| **F13** | Auditoría y Control | El log de auditoría registra toda escritura POS |

**Lectura de la tabla:** las 7 fases base (F0–F6) construyen el **edificio**. Las fases de extensión (F7–F13) **portan** la funcionalidad del POS viejo y añaden las capas transversales (IA, CRM, auditoría). El orden no es casual: primero el cimiento, luego la frontera, luego el comportamiento, luego los guardianes, luego la superficie, luego la consolidación. **De adentro hacia afuera** (Tomo I, §8).

---

## §3. Las 84 fichas por fase

El conjunto de fichas se reparte así entre las fases:

| Fase | Nº de fichas | Fichas |
|------|--------------|--------|
| **F1** | 1 | `FICHA_F1_CIMIENTO_DE_DATOS` |
| **F2** | 1 | `FICHA_F2_FRONTERA_DE_CONTRATOS` |
| **F3** | 7 | `F3_0_RUNNER`, `F3_1_UTILIDADES`, `F3_2_ATOMICO`, `F3_3_HOOKS`, `F3_4_INTERFAZ`, `F3_CIERRE`, `F3_COMPORTAMIENTO` |
| **F4** | 8 | `F4_0_LIGA_CAJA`, `F4_1_CAJA_API`, `F4_2_CASH_SERVICE`, `F4_3_GESTOR_CAJA`, `F4_4_CORTE_TICKET`, `F4_5_MONTAJE_CAJA`, `F4_CIERRE`, `F4_GUARDIANES` |
| **F5** | 6 | `F5_0_CUENTAS_ABIERTAS`, `F5_1_SERVICIO_CUENTAS`, `F5_2_HOOK_CUENTAS`, `F5_3_PIZARRON`, `F5_CIERRE`, `F5_SUPERFICIE` |
| **F6** | 7 | `F6_0_TICKET_GENERATOR`, `F6_1_TICKET_TEMPLATE`, `F6_2_PRINT_SERVICE`, `F6_3_CATALOGO_PDF`, `F6_5_ORDEN_TERMINALES`, `F6_CIERRE`, `F6_CONSOLIDACION` |
| **F7** | 12 | `F7_0_CONTRATOS_IA`, `F7_1_TEMAS`, `F7_2_VOZ`, `F7_3_VISION`, `F7_5_PEDIDOS`, `F7_6_UX_HEREDADA`, `F7_7_ROUTER_TERMINALES`, `F7_7b_IDENTIDAD_OCUPANTE`, `F7_7c_TIPO_DE_ID_EN_LA_FRONTERA`, `F7_7d_TERMINAL_COMO_PROP`, `F7_7e_PRECIO_STRING`, `F7_CIERRE` |
| **F8** | 8 | `F8_0_CONTRATOS_Y_REGLAS_CRM`, `F8_1_SERVICIO_BENEFICIOS`, `F8_2_SERVICIO_NOTIFICACIONES`, `F8_3_HOOK_IDENTIFICACION`, `F8_4_PANEL_IDENTIFICACION`, `F8_5_PANEL_ENTREGA`, `F8_6_CABLEADO_E2E`, `F8_CRM_NOTIFICACIONES` |
| **F9** | 3 | `F9_1_3_UI_PAGOS_MIXTOS`, `F9_1_PAGOS_MIXTOS`, `F9_UX_RESCATE` |
| **F10** | 5 | `F10_2_B01_COPIAR_URL`, `F10_4_CONTEXTO_DIARIO`, `F10_5_PARIDAD_DE_DATOS_DE_CAJA`, `F10_6_PARIDAD_DE_OPERACION_DE_CAJA`, `F10_PARIDAD` |
| **F11** | 1 | `F11_0_VERIFICACION_PLAN_VS_REALIDAD` |
| **F12** | 19 | `F12_1` … `F12_19` |
| **F13** | 3 | `F13_0_LOG_AUDITORIA`, `F13_1_CONTRATO_AUDITORIA`, `F13_2_SERVICIO_AUDITORIA` |
| **TOTAL** | **84** | |

**Nota sobre F0:** la Fase 0 (Andamiaje) no tiene ficha propia en el directorio; su evidencia es el propio andamiaje (CI, greps, estructura de repos). Se documenta en el Tomo I (§13, los 5 greps de CI).

**Nota sobre la numeración de F12:** las 19 fichas de F12 corresponden a las 19 funcionalidades del POS viejo portadas (F12.1 a F12.19). Cada una tiene su ficha.

---

## §4. F0 — Andamiaje

**Qué se construyó:** la estructura de dos repositorios (`NUEVO-POS` código, `PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS` planos), el pipeline de CI, y los **5 greps de CI** que se ejecutan desde el día 1 (Tomo I, §13).

**La puerta de F0:** CI en verde con 0 tests + los 5 greps activos.

**Los 5 greps de CI** (Tomo I, §13) son la primera línea de defensa arquitectónica. Se ejecutan en cada push y fallan el build si detectan:
1. Un import de un módulo a las tablas de otro (violación de A-02).
2. Un `except: pass` o silencio de excepción en la ruta crítica (violación de E-05).
3. Un offset de zona horaria hardcodeado (violación de RN-81).
4. Un envío directo de notificación (violación de RN-85, debe ir por outbox).
5. Una lectura de tabla ajena desde un consumidor (violación de O-23).

**Por qué F0 primero:** sin el andamiaje, cada fase posterior construiría sobre arena. Los greps son **baratos** y **permanentes**: atrapan la violación en el momento del commit, no en producción.

**Evidencia:** el andamiaje vive en [`NUEVO-POS/scripts/guards.mjs`](../NUEVO-POS/scripts/guards.mjs) y en el workflow de CI. La puerta F0 se verifica con `guards 7/7` (7 guardianes de CI en verde).

---

## §5. F1 — Cimiento de datos

**Qué se construyó:** las **17 tablas** del POS nuevo (migración `0001_initial_schema`), los **18 modelos** SQLAlchemy, y las utilidades de tiempo (`core/timestamps.py`, C-02).

**La puerta de F1:** las migraciones aplican y revierten limpias.

**Las piezas clave:**
- [`NUEVO-POS/apps/api/migrations/versions/0001_initial_schema.py`](../NUEVO-POS/apps/api/migrations/versions/0001_initial_schema.py) — las 17 tablas del esquema inicial.
- [`NUEVO-POS/apps/api/models/pos.py`](../NUEVO-POS/apps/api/models/pos.py) — el núcleo transaccional: `TerminalSession`, `Ticket`, `TicketItem`, `TerminalLock`.
- [`NUEVO-POS/apps/api/models/warehouse.py`](../NUEVO-POS/apps/api/models/warehouse.py) — el almacén: `Almacen`, `StockAlmacen`, `MovimientoInventario`, `WarehouseEvent`, `WarehouseEventoSinAlmacen`.
- [`NUEVO-POS/apps/api/models/__init__.py`](../NUEVO-POS/apps/api/models/__init__.py) — los 18 modelos.
- [`NUEVO-POS/apps/api/core/timestamps.py`](../NUEVO-POS/apps/api/core/timestamps.py) — `utcnow()` (C-02: store UTC, display local).

**Los estándares de BD (C-01 a C-04, Tomo I §12):** C-01 (toda tabla tiene `id` UUID), C-02 (timestamps en UTC), C-03 (soft-delete con `deleted_at`), C-04 (toda tabla tiene `created_at`/`updated_at`).

**Por qué F1 primero:** el cimiento de datos es la base de todo. Sin tablas correctas, ni la frontera ni el comportamiento pueden construirse. La puerta (migraciones que aplican y revierten) garantiza que el esquema es **reversible**: se puede deshacer sin dejar basura.

**Ficha:** [`FICHA_F1_CIMIENTO_DE_DATOS.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F1_CIMIENTO_DE_DATOS.md).

---

## §6. F2 — Frontera (contratos)

**Qué se construyó:** el **registro de contratos** ([`contracts/registry.py`](../NUEVO-POS/apps/api/contracts/registry.py)) y el **test de arquitectura** que verifica que ningún módulo importa las tablas de otro.

**La puerta de F2:** el test de arquitectura pasa con **0 imports ajenos**.

**Las piezas clave:**
- [`NUEVO-POS/apps/api/contracts/registry.py`](../NUEVO-POS/apps/api/contracts/registry.py) — el `Contrato` dataclass y la tupla `CONTRATOS` (31 contratos).
- El test de arquitectura que verifica A-02 (frontera por contratos) y O-23 (un contrato nunca expone una tabla).

**La frontera por contratos (A-02/O-23, Tomo IV §1):** el consumidor pide por **operación**, nunca lee la tabla del proveedor. El campo `tabla_expuesta` de todo `Contrato` es siempre `None`; existe para afirmar esto en el test de la puerta F2.

**Por qué F2 después de F1:** primero los datos, luego la **frontera** que los protege. La frontera es lo que impide que el POS se convierta en un monolito acoplado. La puerta (0 imports ajenos) es **binaria**: o se respeta la frontera o no.

**Ficha:** [`FICHA_F2_FRONTERA_DE_CONTRATOS.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F2_FRONTERA_DE_CONTRATOS.md).

---

## §7. F3 — Comportamiento (reglas)

**Qué se construyó:** las **95 reglas de negocio** ([`rules/registry.py`](../NUEVO-POS/apps/api/rules/registry.py)), el **backend atómico** (contratos 18–22), los **hooks** del frontend y la **interfaz** base.

**La puerta de F3:** la matriz `regla → test` está **completa** (ninguna regla sin test).

**Las 7 fichas de F3:**
1. [`FICHA_F3_0_RUNNER.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F3_0_RUNNER.md) — el runner de tests.
2. [`FICHA_F3_1_UTILIDADES.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F3_1_UTILIDADES.md) — las utilidades (`aOutcome`, `withRetries`).
3. [`FICHA_F3_2_ATOMICO.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F3_2_ATOMICO.md) — el backend atómico (contratos 18–22): idempotencia, concurrencia optimista, respuesta ligera, verificación post-envío, anti-degradación.
4. [`FICHA_F3_3_HOOKS.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F3_3_HOOKS.md) — los hooks del frontend.
5. [`FICHA_F3_4_INTERFAZ.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F3_4_INTERFAZ.md) — la interfaz base.
6. [`FICHA_F3_CIERRE.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F3_CIERRE.md) — el cierre de la fase.
7. [`FICHA_F3_COMPORTAMIENTO.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F3_COMPORTAMIENTO.md) — la puerta de comportamiento.

**Las 95 reglas (RN-01 a RN-95):** viven en [`rules/registry.py`](../NUEVO-POS/apps/api/rules/registry.py). La tupla se llama `LAS_81_REGLAS` (residuo histórico: empezó con 81 reglas y creció a 95). El Tomo II las documenta todas.

**El backend atómico (F3.2):** los contratos 18–22 implementan las operaciones atómicas del ticket. La puerta F3.2 verifica:
- **Idempotencia:** 2× POST del mismo `item_id` deja el ticket en el MISMO estado.
- **Concurrencia optimista:** un `version` obsoleto responde 409 y NO escribe (RN-25/RN-26).
- **Respuesta ligera:** `GET /pos/tickets/{id}` devuelve EXACTAMENTE 5 campos escalares (Regla 15).
- **Verificación post-envío:** `POST /verify` confirma en BD que el ticket y sus ítems existen.
- **Anti-degradación:** quitar una línea cuando solo hay 1 reduce el 100% → 400 (RN-37).

**Evidencia:** [`NUEVO-POS/apps/api/tests/test_f3_comportamiento.py`](../NUEVO-POS/apps/api/tests/test_f3_comportamiento.py) (la matriz de 95 reglas) y [`NUEVO-POS/apps/api/tests/test_f3_atomico.py`](../NUEVO-POS/apps/api/tests/test_f3_atomico.py) (los contratos 18–22).

**Por qué F3 después de F2:** primero la frontera, luego el **comportamiento** que la cruza. Las reglas son el **qué** del negocio; los contratos son el **cómo** se comunican los módulos.

---

## §8. F4 — Guardianes

**Qué se construyó:** los **guardianes** ([`guards/`](../NUEVO-POS/apps/api/guards/)) que hacen fallar el CI si se viola una regla crítica, y la **liga de caja** (F4.0–F4.5).

**La puerta de F4:** el CI falla si se viola una regla crítica.

**Las 8 fichas de F4:**
1. [`FICHA_F4_0_LIGA_CAJA.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F4_0_LIGA_CAJA.md) — la liga de caja.
2. [`FICHA_F4_1_CAJA_API.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F4_1_CAJA_API.md) — la API de caja.
3. [`FICHA_F4_2_CASH_SERVICE.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F4_2_CASH_SERVICE.md) — el servicio de caja.
4. [`FICHA_F4_3_GESTOR_CAJA.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F4_3_GESTOR_CAJA.md) — el gestor de caja.
5. [`FICHA_F4_4_CORTE_TICKET.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F4_4_CORTE_TICKET.md) — el corte de ticket.
6. [`FICHA_F4_5_MONTAJE_CAJA.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F4_5_MONTAJE_CAJA.md) — el montaje de caja (la micro-fase de INTEGRACIÓN).
7. [`FICHA_F4_CIERRE.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F4_CIERRE.md) — el cierre de la fase.
8. [`FICHA_F4_GUARDIANES.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F4_GUARDIANES.md) — los guardianes.

**Los guardianes ([`guards/`](../NUEVO-POS/apps/api/guards/)):**
- [`guards/outbox.py`](../NUEVO-POS/apps/api/guards/outbox.py) — `OutboxTransaccional` (A-04): el evento de inventario vive en la misma transacción del ticket. Lanza `SinSilenciosEnRutaCritica` si se intenta silenciar una excepción (E-05).
- [`guards/audit.py`](../NUEVO-POS/apps/api/guards/audit.py) — `LogDeAuditoria` (A-06): el asiento de auditoría se encola y se confirma al cerrar sin excepción; hace rollback si falla.
- [`guards/identity.py`](../NUEVO-POS/apps/api/guards/identity.py) — el guardián de identidad.
- [`guards/guardians.py`](../NUEVO-POS/apps/api/guards/guardians.py) — los guardianes base.
- [`guards/__init__.py`](../NUEVO-POS/apps/api/guards/__init__.py) — la exportación de los guardianes.

**La lección de F4.5 (Tomo I §10.6.1):** el paso de **INTEGRACIÓN** también es una compuerta. Montar las piezas (F4.0–F4.4) no es integrarlas (F4.5). La ficha `F4_5_MONTAJE_CAJA` documenta este paso.

**Por qué F4 después de F3:** primero el comportamiento, luego los **guardianes** que lo protegen. Los guardianes convierten las reglas críticas en **fallos de CI**: una regla violada no llega a producción.

---

## §9. F5 — Superficie (interfaces)

**Qué se construyó:** las **24 interfaces** ([`superficie/registry.py`](../NUEVO-POS/apps/api/superficie/registry.py)), el **pizarrón de cuentas abiertas** (F5.0–F5.3) y la **superficie** completa.

**La puerta de F5:** **paridad funcional** con el POS actual.

**Las 6 fichas de F5:**
1. [`FICHA_F5_0_CUENTAS_ABIERTAS.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F5_0_CUENTAS_ABIERTAS.md) — las cuentas abiertas.
2. [`FICHA_F5_1_SERVICIO_CUENTAS.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F5_1_SERVICIO_CUENTAS.md) — el servicio de cuentas.
3. [`FICHA_F5_2_HOOK_CUENTAS.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F5_2_HOOK_CUENTAS.md) — el hook de cuentas.
4. [`FICHA_F5_3_PIZARRON.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F5_3_PIZARRON.md) — el pizarrón.
5. [`FICHA_F5_CIERRE.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F5_CIERRE.md) — el cierre de la fase.
6. [`FICHA_F5_SUPERFICIE.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F5_SUPERFICIE.md) — la superficie.

**Las 24 interfaces ([`superficie/registry.py`](../NUEVO-POS/apps/api/superficie/registry.py)):** 7 pantallas raíz, 6 modales, 5 paneles/overlays, 4 de composición, 2 de impresión. El Tomo V las documenta todas. El defecto del plano (26 vs 24) se documenta en Tomo V §9: el Documento 7 dice "26" pero el contenido trazable es 24.

**Las 4 reglas duras de responsividad (R-01 a R-04, Tomo V §3):** R-01 (cero anchos absolutos en contenedores raíz), R-02 (tipografía que escala), R-03 (los 3 modos son explícitos: MÓVIL base, md: COMPACTO, lg: MOSTRADOR), R-04 (objetivos táctiles de 44×44px).

**Los 3 modos de layout (Tomo V §4):** MOSTRADOR (≥1024px, referencia INTOCABLE), COMPACTO (768–1023px, añadido), MÓVIL (<768px, añadido).

**La paleta canónica (Tomo V §2):** `#c1d72e` (acento), `#0a0a0a` (fondo), `#080808` (fondo alt), `#1a1a1a` (panel), `#fdfbf7` (crema ticket), `#ef4444` (rojo peligro). Radios: `rounded-[35px]`, `rounded-[40px]`, `rounded-[50px]`.

**Evidencia:** [`NUEVO-POS/apps/pos/src/services/openAccountsService.f5_1.test.jsx`](../NUEVO-POS/apps/pos/src/services/openAccountsService.f5_1.test.jsx) y [`NUEVO-POS/apps/pos/src/hooks/useOpenAccounts.js`](../NUEVO-POS/apps/pos/src/hooks/useOpenAccounts.js).

**Por qué F5 después de F4:** primero los guardianes, luego la **superficie** que el usuario ve. La superficie es la **cara** del edificio; se construye cuando la estructura ya es sólida.

---

## §10. F6 — Consolidación (central)

**Qué se construyó:** la **impresión** (generador de ticket, plantilla, servicio de impresión, PDF de catálogo), el **orden de terminales** y la **consolidación central**.

**La puerta de F6:** la **sync de cierre de día** está verificada.

**Las 7 fichas de F6:**
1. [`FICHA_F6_0_TICKET_GENERATOR.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F6_0_TICKET_GENERATOR.md) — el generador de ticket.
2. [`FICHA_F6_1_TICKET_TEMPLATE.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F6_1_TICKET_TEMPLATE.md) — la plantilla de ticket.
3. [`FICHA_F6_2_PRINT_SERVICE.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F6_2_PRINT_SERVICE.md) — el servicio de impresión.
4. [`FICHA_F6_3_CATALOGO_PDF.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F6_3_CATALOGO_PDF.md) — el PDF de catálogo.
5. [`FICHA_F6_5_ORDEN_TERMINALES.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F6_5_ORDEN_TERMINALES.md) — el orden de terminales.
6. [`FICHA_F6_CIERRE.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F6_CIERRE.md) — el cierre de la fase.
7. [`FICHA_F6_CONSOLIDACION.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F6_CONSOLIDACION.md) — la consolidación.

**Las piezas clave:**
- [`NUEVO-POS/apps/pos/src/components/CorteTicketTemplate.jsx`](../NUEVO-POS/apps/pos/src/components/CorteTicketTemplate.jsx) — la plantilla del corte.
- [`NUEVO-POS/apps/pos/src/services/printService.js`](../NUEVO-POS/apps/pos/src/services/printService.js) — el servicio de impresión.

**Por qué F6 al final de las 7 fases base:** la consolidación central es la **capa más externa**. Solo tiene sentido cuando el POS local ya opera. La puerta (sync de cierre de día verificada) garantiza que el POS puede **reportar** a la central.

---

## §11. F7 — Voz + Visión IA + Temas

**Qué se construyó:** los **contratos de IA** (F7.0), el **sistema de temas** (F7.1), la **voz** (F7.2), la **visión** (F7.3), los **pedidos** (F7.5), la **UX heredada** (F7.6) y el **router de terminales** (F7.7 + sub-fases).

**La puerta de F7:** los **3 modos de topología IA** operan sin bloquear la venta.

**Las 12 fichas de F7:**
1. [`FICHA_F7_0_CONTRATOS_IA.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F7_0_CONTRATOS_IA.md) — los contratos de IA.
2. [`FICHA_F7_1_TEMAS.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F7_1_TEMAS.md) — el sistema de temas.
3. [`FICHA_F7_2_VOZ.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F7_2_VOZ.md) — la voz.
4. [`FICHA_F7_3_VISION.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F7_3_VISION.md) — la visión.
5. [`FICHA_F7_5_PEDIDOS.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F7_5_PEDIDOS.md) — los pedidos.
6. [`FICHA_F7_6_UX_HEREDADA.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F7_6_UX_HEREDADA.md) — la UX heredada.
7. [`FICHA_F7_7_ROUTER_TERMINALES.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F7_7_ROUTER_TERMINALES.md) — el router de terminales.
8. [`FICHA_F7_7b_IDENTIDAD_OCUPANTE.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F7_7b_IDENTIDAD_OCUPANTE.md) — la identidad del ocupante.
9. [`FICHA_F7_7c_TIPO_DE_ID_EN_LA_FRONTERA.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F7_7c_TIPO_DE_ID_EN_LA_FRONTERA.md) — el tipo de id en la frontera.
10. [`FICHA_F7_7d_TERMINAL_COMO_PROP.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F7_7d_TERMINAL_COMO_PROP.md) — la terminal como prop.
11. [`FICHA_F7_7e_PRECIO_STRING.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F7_7e_PRECIO_STRING.md) — el precio como string.
12. [`FICHA_F7_CIERRE.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F7_CIERRE.md) — el cierre de la fase.

**Las piezas clave:**
- [`NUEVO-POS/apps/api/contracts/registry.py`](../NUEVO-POS/apps/api/contracts/registry.py) — los contratos de IA (F7.0).
- [`NUEVO-POS/apps/pos/src/RetailVisionPOS.jsx`](../NUEVO-POS/apps/pos/src/RetailVisionPOS.jsx) — el router de terminales y la UX heredada.

**Los 3 modos de topología IA (RN-71):** el motor de IA puede operar en tres topologías distintas sin que el POS cambie su contrato:
1. **Local** — el modelo corre en la máquina del POS.
2. **Remoto** — el modelo corre en un servidor.
3. **Híbrido** — parte local, parte remoto.

El POS **no sabe** en qué modo está; solo llama al contrato. Esto es la frontera por contratos aplicada a la IA.

**La regla de la IA asistiva (RN-74):** la visión es **asistiva**, nunca bloqueante. Si la visión falla, el cajero **siempre** puede cobrar manualmente. La IA **jamás** se interpone entre el cajero y la venta.

**Las sub-fases F7.7b–F7.7e:** cuatro correcciones nacidas de bugs reales durante la construcción del router de terminales:
- **F7.7b — Identidad del ocupante:** quién ocupa la terminal se resuelve por identidad, no por nombre.
- **F7.7c — Tipo de id en la frontera:** el tipo del id (UUID vs. string) se declara en la frontera, no se adivina.
- **F7.7d — Terminal como prop:** la terminal se pasa como prop explícita, no se lee de un contexto global.
- **F7.7e — Precio como string:** el precio viaja como string en la frontera para no perder precisión decimal.

**Por qué F7 después de F6:** la IA y los temas son **aditivos**. El POS ya vende sin ellos. Se construyen al final de las 7 fases base porque **no son críticos para la venta** — son la capa de asistencia. Si F7 fallara por completo, el POS seguiría vendiendo.

---

## §12. F8 — CRM y Notificaciones (lado POS)

**Qué se construyó:** los **contratos y reglas de CRM** (F8.0), el **servicio de beneficios** (F8.1), el **servicio de notificaciones** (F8.2), el **hook de identificación** (F8.3), el **panel de identificación** (F8.4), el **panel de entrega** (F8.5) y el **cableado E2E** (F8.6).

**La puerta de F8:** un cliente identificado recibe sus beneficios y su ticket se envía por Outbox, **sin que un fallo de CRM o Notificaciones bloquee la venta**.

**Las 8 fichas de F8:**
1. [`FICHA_F8_0_CONTRATOS_Y_REGLAS_CRM.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F8_0_CONTRATOS_Y_REGLAS_CRM.md) — los contratos y reglas de CRM.
2. [`FICHA_F8_1_SERVICIO_BENEFICIOS.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F8_1_SERVICIO_BENEFICIOS.md) — el servicio de beneficios.
3. [`FICHA_F8_2_SERVICIO_NOTIFICACIONES.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F8_2_SERVICIO_NOTIFICACIONES.md) — el servicio de notificaciones.
4. [`FICHA_F8_3_HOOK_IDENTIFICACION.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F8_3_HOOK_IDENTIFICACION.md) — el hook de identificación.
5. [`FICHA_F8_4_PANEL_IDENTIFICACION.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F8_4_PANEL_IDENTIFICACION.md) — el panel de identificación.
6. [`FICHA_F8_5_PANEL_ENTREGA.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F8_5_PANEL_ENTREGA.md) — el panel de entrega.
7. [`FICHA_F8_6_CABLEADO_E2E.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F8_6_CABLEADO_E2E.md) — el cableado E2E.
8. [`FICHA_F8_CRM_NOTIFICACIONES.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F8_CRM_NOTIFICACIONES.md) — el cierre de la fase.

**Las reglas clave (RN-82 a RN-93):**
- **RN-82** — el cliente es **opcional**; no identificarlo **no bloquea** la venta.
- **RN-83** — los beneficios **no** llevan el total a negativo.
- **RN-84** — los puntos **no se actualizan**, se **anexan** (ledger inmutable).
- **RN-85** — el envío va **por Outbox**, nunca directo.
- **RN-86** — el evento se **encola en la misma transacción** del ticket.
- **RN-87** — un fallo de CRM **no tumba** el POS.
- **RN-88** — un fallo de Notificaciones **no tumba** el POS.
- **RN-89** — el canal de envío debe ser soportado.
- **RN-90** — el destino debe ser válido para el canal.
- **RN-91** — el beneficio pertenece al cliente.
- **RN-92** — la promoción debe estar vigente.
- **RN-93** — el beneficio aplicado queda auditado.

**El patrón Outbox (Regla de Oro #7):** el POS **encola** el envío del ticket dentro de la transacción del ticket; el worker envía después. El POS **nunca** envía directamente. Guardado por [`OutboxTransaccional`](../NUEVO-POS/apps/api/guards/outbox.py:55), RN-85 y RN-86.

**DT-07 — un fallo ajeno nunca bloquea una venta:** un fallo de CRM, Notificaciones, IA o Estadísticas **jamás** bloquea una venta. El POS degrada a "sin beneficios" / "sin envío" y **cobra igual**.

**Por qué F8 después de F7:** CRM y Notificaciones son **consumidores** del POS. El POS **emite** el ticket; CRM y Notificaciones **reaccionan**. Se construyen después de que el POS ya vende y ya tiene su frontera de contratos.

---

## §13. F9 — Rescate de UX + Pagos mixtos

**Qué se construyó:** el **rescate de UX del viejo POS** (F9), el **afinado de la UI por defecto** (F9) y los **pagos mixtos** (F9.1).

**La puerta de F9:** la UX del nuevo POS es **funcionalmente equivalente** a la del viejo, y los pagos mixtos cuadran.

**Las 3 fichas de F9:**
1. [`FICHA_F9_1_3_UI_PAGOS_MIXTOS.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F9_1_3_UI_PAGOS_MIXTOS.md) — la UI de pagos mixtos.
2. [`FICHA_F9_1_PAGOS_MIXTOS.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F9_1_PAGOS_MIXTOS.md) — los pagos mixtos.
3. [`FICHA_F9_UX_RESCATE.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F9_UX_RESCATE.md) — el rescate de UX.

**Las reglas clave (RN-94 / RN-95):**
- **RN-94** — la **suma de pagos cuadra el total**. Si no cuadra, se rechaza.
- **RN-95** — cada pago usa un **método válido** (reutiliza RN-57).

**La lección de F9:** la UX heredada del viejo POS **se hereda en su integración, se reescribe en su implementación**. No se copia el código; se copia el **flujo** que el cajero ya conoce.

---

## §14. F10 — Auditoría de Paridad (viejo POS vs. nuevo POS)

**Qué se construyó:** la **auditoría de paridad** que compara el viejo POS contra el nuevo, componente por componente, flujo por flujo y operación por operación.

**La puerta de F10:** la **completitud del conjunto** está verificada — no basta con que cada componente exista; el **conjunto** debe estar completo.

**Las 5 fichas de F10:**
1. [`FICHA_F10_2_B01_COPIAR_URL.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F10_2_B01_COPIAR_URL.md) — el bug B01 (copiar URL).
2. [`FICHA_F10_4_CONTEXTO_DIARIO.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F10_4_CONTEXTO_DIARIO.md) — el contexto diario.
3. [`FICHA_F10_5_PARIDAD_DE_DATOS_DE_CAJA.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F10_5_PARIDAD_DE_DATOS_DE_CAJA.md) — la paridad de datos de caja.
4. [`FICHA_F10_6_PARIDAD_DE_OPERACION_DE_CAJA.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F10_6_PARIDAD_DE_OPERACION_DE_CAJA.md) — la paridad de operación de caja.
5. [`FICHA_F10_PARIDAD.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F10_PARIDAD.md) — el cierre de la fase.

**Las 3 lecciones de F10 (las más caras del proyecto):**
1. **La lección de F10.4 — el inventario de componentes no ve las integraciones.** Un componente puede existir y estar bien hecho, pero **no estar cableado**. El inventario no lo detecta.
2. **La lección de F10.5 — el inventario de componentes no ve los flujos de datos.** Los datos pueden fluir mal aunque cada componente funcione. El inventario no lo detecta.
3. **La lección de F10.6 — el inventario de componentes no ve la paridad de operación.** La operación del cajero puede diferir aunque los datos cuadren. El inventario no lo detecta.

**Por qué F10 es una fase completa:** porque la paridad **no se verifica componente a componente**, se verifica **flujo a flujo y operación a operación**. F10 nació de la lección de F4.5 (el paso de INTEGRACIÓN también es una compuerta) y la extendió a todo el sistema.

---

## §15. F11 — Verificación Plan vs. Realidad

**Qué se construyó:** la **verificación** de que lo planeado coincide con lo construido.

**La puerta de F11:** el plan y la realidad **coinciden**; toda desviación está documentada.

**La ficha de F11:**
1. [`FICHA_F11_0_VERIFICACION_PLAN_VS_REALIDAD.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F11_0_VERIFICACION_PLAN_VS_REALIDAD.md) — la verificación plan vs. realidad.

**Por qué F11 existe:** porque un plan que no se verifica contra la realidad es **ficción**. F11 obliga a confrontar lo escrito con lo construido y a documentar cada desviación.

---

## §16. F12 — Paridad con el viejo POS (19 fichas)

**Qué se construyó:** el **portado de funcionalidades específicas** del viejo POS al nuevo, una por una, con su test.

**La puerta de F12:** cada funcionalidad portada tiene **paridad funcional** con el viejo POS y su **test de puerta** en verde.

**Las 19 fichas de F12 (F12.1 a F12.19):**
1. [`FICHA_F12_1`](../NUEVO-POS/docs/05-plan-de-construccion/) — F12.1.
2. [`FICHA_F12_2`](../NUEVO-POS/docs/05-plan-de-construccion/) — F12.2.
3. [`FICHA_F12_3`](../NUEVO-POS/docs/05-plan-de-construccion/) — F12.3.
4. [`FICHA_F12_4`](../NUEVO-POS/docs/05-plan-de-construccion/) — F12.4.
5. [`FICHA_F12_5`](../NUEVO-POS/docs/05-plan-de-construccion/) — F12.5.
6. [`FICHA_F12_6`](../NUEVO-POS/docs/05-plan-de-construccion/) — F12.6.
7. [`FICHA_F12_7`](../NUEVO-POS/docs/05-plan-de-construccion/) — F12.7.
8. [`FICHA_F12_8`](../NUEVO-POS/docs/05-plan-de-construccion/) — F12.8.
9. [`FICHA_F12_9`](../NUEVO-POS/docs/05-plan-de-construccion/) — F12.9.
10. [`FICHA_F12_10`](../NUEVO-POS/docs/05-plan-de-construccion/) — F12.10.
11. [`FICHA_F12_11`](../NUEVO-POS/docs/05-plan-de-construccion/) — F12.11.
12. [`FICHA_F12_12`](../NUEVO-POS/docs/05-plan-de-construccion/) — F12.12.
13. [`FICHA_F12_13`](../NUEVO-POS/docs/05-plan-de-construccion/) — F12.13.
14. [`FICHA_F12_14`](../NUEVO-POS/docs/05-plan-de-construccion/) — F12.14.
15. [`FICHA_F12_15`](../NUEVO-POS/docs/05-plan-de-construccion/) — F12.15.
16. [`FICHA_F12_16`](../NUEVO-POS/docs/05-plan-de-construccion/) — F12.16.
17. [`FICHA_F12_17`](../NUEVO-POS/docs/05-plan-de-construccion/) — F12.17.
18. [`FICHA_F12_18`](../NUEVO-POS/docs/05-plan-de-construccion/) — F12.18.
19. [`FICHA_F12_19`](../NUEVO-POS/docs/05-plan-de-construccion/) — F12.19.

**A-01 — las reglas de negocio se portan CON su test:** cada funcionalidad portada del viejo POS llega **con su test**. No se porta código sin su prueba. Esto garantiza que la paridad es **verificable**, no declarada.

**Por qué F12 después de F10:** F10 **audita** la paridad; F12 **construye** la paridad que faltaba. Primero se mide la brecha, luego se cierra.

---

## §17. F13 — Auditoría y Control

**Qué se construyó:** el **log de auditoría** (`pos_audit_log`, F13.0), el **contrato de auditoría** (contrato 5, F13.1) y el **servicio de auditoría** (F13.2a).

**La puerta de F13:** cada escritura del POS queda **registrada** y es **consultable** por terminal y rango de fechas.

**Las 3 fichas de F13:**
1. [`FICHA_F13_0_LOG_AUDITORIA.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F13_0_LOG_AUDITORIA.md) — el log de auditoría.
2. [`FICHA_F13_1_CONTRATO_AUDITORIA.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F13_1_CONTRATO_AUDITORIA.md) — el contrato de auditoría.
3. [`FICHA_F13_2_SERVICIO_AUDITORIA.md`](../NUEVO-POS/docs/05-plan-de-construccion/FICHA_F13_2_SERVICIO_AUDITORIA.md) — el servicio de auditoría.

**Las reglas clave (RN-75 a RN-78):**
- **RN-75** — cada escritura POS se **registra** y queda persistida.
- **RN-76** — el registro incluye **endpoint, payload, código y extras**.
- **RN-77** — la auditoría se **consulta por terminal y rango** de fechas.
- **RN-78** — los timestamps van en **UTC**.

**El contrato 5:** el contrato de auditoría expone la operación de consulta; su campo `tabla_expuesta` es `None` (O-23).

**F13.2b NO procede:** el panel de auditoría **pertenece al módulo de Auditoría y Control** del ERP, no al POS. El POS **emite** los eventos; el módulo de Auditoría los **muestra**. Construir el panel en el POS violaría la frontera.

**Por qué F13 al final:** la auditoría **observa** todo lo demás. Solo tiene sentido cuando hay algo que observar. Es la capa más externa del POS.

---

## §18. Matriz de trazabilidad — Fase → Fichas → Puerta

| Fase | Nombre | Fichas | Puerta de salida |
|------|--------|--------|------------------|
| F0 | Andamiaje | 1 | CI en verde con 0 tests + 5 greps activos |
| F1 | Cimiento de datos | 1 | Migraciones aplican y revierten limpias |
| F2 | Frontera (contratos) | 1 | Test de arquitectura: 0 imports ajenos |
| F3 | Comportamiento (reglas) | 7 | Matriz `regla → test` completa |
| F4 | Guardianes | 8 | CI falla si se viola una regla crítica |
| F5 | Superficie (interfaces) | 6 | Paridad funcional con el POS actual |
| F6 | Consolidación (central) | 7 | Sync de cierre de día verificada |
| F7 | Voz + Visión IA + Temas | 12 | Los 3 modos de topología IA operan sin bloquear la venta |
| F8 | CRM y Notificaciones | 8 | Beneficios + envío por Outbox sin bloquear la venta |
| F9 | Rescate de UX + Pagos mixtos | 3 | UX equivalente + pagos mixtos cuadran |
| F10 | Auditoría de Paridad | 5 | La completitud del conjunto está verificada |
| F11 | Verificación Plan vs. Realidad | 1 | El plan y la realidad coinciden |
| F12 | Paridad con el viejo POS | 19 | Cada funcionalidad portada tiene paridad + test |
| F13 | Auditoría y Control | 3 | Cada escritura queda registrada y consultable |
| **TOTAL** | | **84** | |

**La regla de avance por puertas:** ninguna fase empieza sin que la **puerta** de la anterior esté en verde. Si una puerta falla, se corrige la fase actual; no se avanza "dejando pendiente".

---

## Cierre del Tomo VI

El **Acta de Obra** documenta las **84 fichas** que construyeron el POS nuevo, fase por fase, con su puerta de salida. Cada ficha es un **ladrillo**; cada puerta es una **verificación** de que el ladrillo encaja.

**Lo que este tomo enseña:**
1. **El avance es por puertas, no por fechas.** Una fase no termina cuando "parece terminada", termina cuando su puerta está en verde.
2. **La integración es una compuerta.** La lección de F4.5: un componente puede existir y no estar cableado. El paso de INTEGRACIÓN también se verifica.
3. **La completitud es una compuerta.** La lección de F10: el conjunto debe estar completo, no solo cada pieza.
4. **La paridad se verifica, no se declara.** F10 audita, F12 construye, A-01 exige el test.
5. **La auditoría observa todo lo demás.** F13 es la capa más externa; solo tiene sentido cuando hay algo que observar.

**El resultado:** 14 fases (F0–F13), 84 fichas, cada una con su puerta. El POS nuevo no es un conjunto de archivos; es un **edificio** con cimientos, columnas, muros y un techo de auditoría.

**Advertencia para la próxima IA:** no saltes puertas. La tentación de "avanzar dejando pendiente" es exactamente lo que produjo los 18 incidentes del Tomo III. La puerta no es burocracia; es la **cicatriz** que impide repetir el bug.
