# 📋 PLAN DE ABORDAJE — FASE 8: CRM Y NOTIFICACIONES (lado POS)

**Versión:** 2.0
**Fecha:** 30 Sep 2026
**Autor:** Arquitecto del Nuevo POS
**Estado:** Propuesta — pendiente de aprobación del dueño
**Precedente:** `PLAN_MAESTRO_DEFINITIVO_POS.md` §7 (Fase 8) + `PROPUESTA_CRM_Y_NOTIFICACIONES_DEL_NUEVO_POS.md` (Documento 13)
**Regla que gobierna este plan:** REGLA DURA 2 — *"Verificar, no asumir"*
**Autocrítica que origina esta versión:** §9 (5 defectos, 2 de ellos críticos)
**Cambios vs v1.0:** corregidos D-1 (dónde vive el paso de entrega), D-2 (props del header), D-3 (archivos de test reales), D-4 (declarar ≠ implementar), D-5 (radio de impacto de los tests de conteo)

---

## §0. VERIFICACIÓN PREVIA (REGLA DURA 2)

> **Antes de escribir una sola línea de este plan se verificó el estado real del código.**
> No se asumió nada. Cada número de este documento proviene de una lectura del repositorio.

### §0.1 Lo que se verificó y lo que se encontró

| Afirmación a verificar | Método | Resultado verificado |
|---|---|---|
| ¿Cuántos contratos existen hoy? | Lectura de [`contracts/registry.py`](../NUEVO-POS/apps/api/contracts/registry.py:631) | **25** — el último es `#25 ia.interpretar_intencion` |
| ¿Cuál es el siguiente contrato libre? | Conteo del registro | **#26** y **#27** |
| ¿Cuántas reglas existen hoy? | Lectura de [`rules/registry.py`](../NUEVO-POS/apps/api/rules/registry.py:766) | **81** — la última es `RN-81` |
| ¿Cuál es la siguiente regla libre? | Conteo del registro | **RN-82** en adelante |
| ¿Existe código de CRM/Notificaciones en el POS nuevo? | Búsqueda `customer\|notification\|loyalty\|benefit\|promotion` en `apps/api` y `apps/pos/src` | **NO EXISTE.** Cero coincidencias reales |
| ¿Existe la tabla `system_settings`? | Lectura de [`models/__init__.py`](../NUEVO-POS/apps/api/models/__init__.py:1) | **SÍ** — creada en F7.5.1a (18 modelos) |
| ¿Existe el documento de la propuesta CRM? | `list_files` del repo de planos | **SÍ** — 649 líneas, 10 secciones, Q-1..Q-5 resueltas |
| ¿Qué tests afirman el conteo de contratos? | Búsqueda `len(CONTRATOS) == 25` en `tests/` | **DOS archivos**: [`test_f2_frontera.py:150`](../NUEVO-POS/apps/api/tests/test_f2_frontera.py:150) y [`test_f7_contratos_ia.py:67`](../NUEVO-POS/apps/api/tests/test_f7_contratos_ia.py:67) |
| ¿Qué tests afirman el conteo de reglas? | Búsqueda `81` en `tests/` | **TRES aserciones** en [`test_f3_comportamiento.py`](../NUEVO-POS/apps/api/tests/test_f3_comportamiento.py:30): matriz=81, números=RN-01..RN-81 sin huecos, `listar_reglas()`=81 |
| ¿Dónde vive el paso post-cobro hoy? | Lectura de [`RetailVisionPOS.jsx`](../NUEVO-POS/apps/pos/src/RetailVisionPOS.jsx:80) | En el **screen**, no en `CheckoutScreen` (que es el modal de pago y se cierra al cobrar) |
| ¿El header tiene botones de acción? | Lectura de [`POSHeader.jsx`](../NUEVO-POS/apps/pos/src/components/POSHeader.jsx:105) | **SÍ** — 📌 pedido, 🎨 tema, 🎤 voz (líneas 105-140) |

### §0.2 El hallazgo crítico: la propuesta tiene DOS numeraciones obsoletas, no una

El Plan Maestro ya advierte (§7, nota de coherencia) que la propuesta se redactó cuando el plano
iba en **17 contratos y 73 reglas**, y que los números deben renumerarse. Pero al verificar el
documento se encontró un **segundo error, más sutil**:

> **§7 de la propuesta contiene una "nota de corrección (autocrítica)" que afirma:**
> *"La primera versión de este documento numeró estas reglas como RN-82 a RN-93, asumiendo
> erróneamente que el plano llegaba hasta RN-81. **El plano llega hasta RN-73.** Se renumera a
> RN-74 a RN-85."*

**Esa corrección es HOY falsa.** El plano **sí** llega hasta RN-81 (verificado en
[`rules/registry.py:766`](../NUEVO-POS/apps/api/rules/registry.py:766)). La propuesta se
"corrigió" hacia atrás, a un número que ya era obsoleto cuando se escribió la corrección.

**Consecuencia:** la numeración correcta de las reglas nuevas es **RN-82 a RN-93** — que es,
irónicamente, la numeración que la propuesta **descartó** por considerarla errónea. La
"corrección" fue el error.

| Elemento | Propuesta original | "Corrección" de la propuesta | **Número real verificado** |
|---|---|---|---|
| Contrato CRM | #18 | (sin corregir) | **#26** |
| Contrato Notificaciones | #19 | (sin corregir) | **#27** |
| Reglas nuevas | RN-74..RN-85 | RN-74..RN-85 | **RN-82..RN-93** |

> **Lección (refuerza REGLA DURA 2):** una corrección no verificada contra el código es tan
> peligrosa como el error original. La propuesta corrigió un número leyendo otro documento
> desactualizado, no el `registry.py`. **El `registry.py` es la única fuente de verdad.**

### §0.3 Lo que la propuesta SÍ acierta (y se conserva)

| Acierto | Por qué se conserva |
|---|---|
| Dos módulos, no uno (CRM + Notificaciones) | Responsabilidades y ciclos de vida distintos |
| El POS nunca lee tablas ajenas (P-01/P-02/P-03) | Es la frontera por contratos (A-02) |
| Degradación elegante: nunca bloquea la venta | Coherente con DT-07 |
| Outbox transaccional para notificaciones | Regla de Oro #7 |
| Identificación del cliente **posterior** a la carga | Respeta RN-08 y RN-10 |
| La política de lealtad es **dato, no código** | Coherente con DT-06 |

---

## §1. ALCANCE REAL DE LA FASE 8 (lado POS)

### §1.1 Lo que el Plan Maestro manda construir

> **Cita textual** ([`PLAN_MAESTRO:475`](../PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS/PLAN_MAESTRO_DEFINITIVO_POS.md:475)):
> *"Son **2 componentes** y **2 llamadas a contrato**. Nada más."*

| # | Componente | Qué hace | Contrato que consume |
|---|---|---|---|
| 1 | `CustomerIdentificationPanel.jsx` | Botón "👤 Cliente" en el header + panel para teclear teléfono | **#26** `clientes.beneficios_para_ticket` |
| 2 | `TicketDeliveryPanel.jsx` | Paso post-cobro: 🖨️ Imprimir / 📱 WhatsApp / ✉️ Email / Omitir | **#27** `notificaciones.encolar_ticket` |

### §1.2 Lo que NO se construye (es de otro módulo)

| Componente | Pertenece a | Módulo |
|---|---|---|
| Pantalla de gestión de clientes | ❌ No es del POS | Clientes (CRM) |
| Pantalla de promociones | ❌ No es del POS | Clientes (CRM) |
| Tablas `customers`, `loyalty_ledger`, `promotions`, `customer_benefits`, `loyalty_policy` | ❌ No es del POS | Clientes (CRM) |
| Worker de WhatsApp/SMTP | ❌ No es del POS | Notificaciones |
| Tablas `notification_outbox`, `notification_log`, `channel_config` | ❌ No es del POS | Notificaciones |

### §1.3 La pregunta de alcance que este plan debe responder

El Plan Maestro dice "2 componentes y 2 llamadas a contrato". Pero **los contratos #26 y #27 no
tienen proveedor todavía** — el CRM y Notificaciones no existen. Esto plantea una decisión:

| Opción | Qué implica | Riesgo |
|---|---|---|
| **A — POS solo, con degradación** | El POS construye los 2 componentes y llama a los contratos. Como el proveedor no existe, la llamada falla y **degrada** (precio de lista / solo impresión). | Bajo. Es exactamente lo que el Plan Maestro pide. |
| **B — POS + stubs de los módulos** | Además del POS, se construyen endpoints mínimos que simulan el CRM y Notificaciones. | Alto. Construye código de módulos ajenos, viola P-01. |
| **C — Esperar a que existan los módulos** | No se construye nada hasta que CRM y Notificaciones existan. | Bloquea la Fase 8 indefinidamente. |

**Recomendación: Opción A.** Es la que el Plan Maestro manda, respeta la frontera por contratos
(el POS consume, no define) y aprovecha la degradación elegante ya probada en F7.5 (DT-07).

> **Nota:** la Opción A implica que, hasta que el CRM exista, el botón "Cliente" siempre
> responderá "sin beneficios" y el panel de entrega solo ofrecerá "Imprimir". **Eso es correcto
> y esperado** — es la degradación funcionando. Cuando los módulos existan, el POS no se toca.

---

## §2. LOS CONTRATOS #26 Y #27 (renumerados y verificados)

### §2.1 CONTRATO #26 — `clientes.beneficios_para_ticket`

```
CONTRATO 26 — clientes.beneficios_para_ticket
  Consumidor:   POS
  Proveedor:    Clientes (CRM)
  Operación:    POST /customers/benefits-for-ticket
  Entrada:      { telefono: String,
                  items: [{ product_id: String, qty: Integer, unit_price: String }] }
  Salida:       { customer_id: String NULL,
                  nombre: String NULL,
                  nivel: String NULL,
                  descuentos: [{ tipo: String, valor: String, aplica_a: String, motivo: String }],
                  puntos_a_ganar: Integer,
                  puntos_disponibles: Integer,
                  puede_canjear: Boolean }
  Garantías:    - El CRM es el ÚNICO que decide el beneficio. El POS no calcula descuentos.
                - Determinista para el mismo carrito y el mismo día.
                - Si el cliente no existe: `customer_id: null` y cero beneficios (200, NO error).
  Errores:      - 404 si el teléfono no es normalizable.
                - 503 si el CRM no responde → el POS cobra SIN beneficios (degradación elegante).
  Estado hoy:   DECLARADO (F8.0). Proveedor: NO EXISTE todavía.
```

> **Nota DT-02:** `unit_price` y `valor` viajan como **String** en el cable (dinero decimal),
> coherente con la letra chica de DT-02 reforzada en el commit `f0222f7`. El POS los coerciona
> con `Number()` en la frontera, igual que [`ProductCard.jsx`](../NUEVO-POS/apps/pos/src/components/ProductCard.jsx:30).

### §2.2 CONTRATO #27 — `notificaciones.encolar_ticket`

```
CONTRATO 27 — notificaciones.encolar_ticket
  Consumidor:   POS
  Proveedor:    Notificaciones
  Operación:    POST /notifications/enqueue-ticket
  Entrada:      { evento_id: String,
                  ticket_uuid: String,
                  canales: [String],   # WHATSAPP | EMAIL
                  destinatario: { telefono: String NULL, email: String NULL },
                  payload: { folio: String, total: String,
                             items: [...], fecha: String } }
  Salida:       { encolado: Boolean,
                  mensajes: [{ canal: String, estado: String }] }
  Garantías:    - NUNCA falla por culpa del canal externo: solo escribe en la cola.
                - Idempotente por `evento_id` (uq_notification_evento_canal).
                - El payload es una PROYECCIÓN del ticket, no la tabla `tickets`.
  Errores:      - 422 si el destinatario no tiene el dato del canal pedido.
                - 503 si la cola no está disponible → el POS NO revierte el cobro.
  Estado hoy:   DECLARADO (F8.0). Proveedor: NO EXISTE todavía.
```

### §2.3 Las reglas nuevas (RN-82 a RN-93) — numeración verificada

> **Corrección respecto a la propuesta:** la propuesta numeró estas reglas como RN-74..RN-85.
> Verificado contra [`rules/registry.py`](../NUEVO-POS/apps/api/rules/registry.py:766), el plano
> llega hasta **RN-81**. Por lo tanto las nuevas son **RN-82 a RN-93**.

| # | Regla | Origen | ¿Se implementa en F8? |
|---|---|---|---|
| **RN-82** | El ticket nace como "Público General". La identificación es opcional y posterior a la carga. | Propuesta RN-74 | ✅ Sí (POS) |
| **RN-83** | El teléfono normalizado es la clave del cliente. El correo es opcional. | Propuesta RN-75 | ✅ Sí (POS) |
| **RN-84** | Los beneficios de precio se aplican como líneas negativas antes de PAID; los acumulables se asientan después. | Propuesta RN-76 | ✅ Sí (POS) |
| **RN-85** | El saldo de puntos nunca se hace UPDATE; se asienta en `loyalty_ledger`. | Propuesta RN-77 | ❌ No (CRM) — solo se declara |
| **RN-86** | El envío por WhatsApp/correo es Outbox: se encola en la transacción del ticket. Nunca bloquea el cobro. | Propuesta RN-78 | ✅ Sí (POS) |
| **RN-87** | El ticket impreso siempre está disponible y es el comportamiento por defecto. | Propuesta RN-79 | ✅ Sí (POS) |
| **RN-88** | La política de lealtad es configurable desde el CRM (tabla `loyalty_policy`), no en el código. | Propuesta RN-80 | ❌ No (CRM) — solo se declara |
| **RN-89** | Cada asiento del ledger guarda la política vigente al momento del asiento. | Propuesta RN-81 | ❌ No (CRM) — solo se declara |
| **RN-90** | El canje de puntos requiere autorización de gerente (contrato #8) y es a petición del cliente. | Propuesta RN-82 | ❌ No (CRM) — solo se declara |
| **RN-91** | El POS avisa proactivamente cuando el cliente tiene un beneficio canjeable. | Propuesta RN-83 | ✅ Sí (POS) |
| **RN-92** | El cajero elige el canal cada vez, pero el sistema precarga el contacto desde el CRM. | Propuesta RN-84 | ✅ Sí (POS) |
| **RN-93** | El alta de cliente es libre, pero busca duplicado por teléfono normalizado antes de crear. | Propuesta RN-85 | ❌ No (CRM) — solo se declara |

> **Aclaración de alcance (corrige el defecto D-4 de la v1.0):** las 12 reglas se **DECLARAN**
> en el `registry.py` (para reservar el número y que la matriz quede completa). De ellas,
> **7 se implementan** en el POS (RN-82, RN-83, RN-84, RN-86, RN-87, RN-91, RN-92) y **5 se
> declaran sin implementar** porque pertenecen al CRM (RN-85, RN-88, RN-89, RN-90, RN-93).
> Declarar ≠ implementar. La matriz regla→test exige que **toda** regla declarada tenga un
> nombre de test; por eso las 5 del CRM se declaran con un test que verifica su **enunciado**
> (no su comportamiento), igual que se hizo con las reglas de fases futuras.

---

## §3. LAS SUB-FASES (de adentro hacia afuera)

> **Principio rector (§10.6 del Plan Maestro):** cada pieza se construye con su puerta (gate)
> **antes** de cablearla. Primero el contrato, luego el servicio, luego el hook, luego el
> componente, y **al final** el cableado end-to-end.

### §3.1 F8.0 — Declarar los contratos #26 y #27 + las reglas RN-82..RN-93

| Aspecto | Detalle |
|---|---|
| **Qué** | Añadir los 2 contratos a [`contracts/registry.py`](../NUEVO-POS/apps/api/contracts/registry.py:657) y las 12 reglas a [`rules/registry.py`](../NUEVO-POS/apps/api/rules/registry.py:767) |
| **Por qué primero** | El contrato es la frontera. Sin contrato declarado, el POS no tiene qué consumir. |
| **Archivos de producción** | `contracts/registry.py` (25→27), `rules/registry.py` (81→93) |
| **Archivos de test a actualizar (radio de impacto verificado)** | **5 aserciones en 3 archivos**: [`test_f2_frontera.py:150`](../NUEVO-POS/apps/api/tests/test_f2_frontera.py:150) (25→27), [`test_f2_frontera.py:218`](../NUEVO-POS/apps/api/tests/test_f2_frontera.py:218) (25→27), [`test_f7_contratos_ia.py:67`](../NUEVO-POS/apps/api/tests/test_f7_contratos_ia.py:67) (25→27), [`test_f3_comportamiento.py:30`](../NUEVO-POS/apps/api/tests/test_f3_comportamiento.py:30) (matriz 81→93), [`test_f3_comportamiento.py:36`](../NUEVO-POS/apps/api/tests/test_f3_comportamiento.py:36) (rango RN-01..RN-81 → RN-01..RN-93), [`test_f3_comportamiento.py:64`](../NUEVO-POS/apps/api/tests/test_f3_comportamiento.py:64) (`listar_reglas()` 81→93) |
| **Gate** | Los 3 archivos de test pasan con los conteos nuevos (27 contratos, 93 reglas) |
| **Riesgo** | Bajo en producción; **medio en tests** (el test de "sin huecos" es estricto: exige el rango completo RN-01..RN-93) |
| **Verificación** | `grep` de que `listar_contratos()` devuelve 27 y `listar_reglas()` devuelve 93 |

> **Nota (defecto D-5 corregido):** la v1.0 decía "el test de contratos pasa de 25 a 27" en
> singular. La verificación encontró **dos** archivos que afirman el conteo de contratos y
> **tres** aserciones que afirman el de reglas. El radio de impacto es mayor de lo que la v1.0
> declaraba. Se documenta aquí para que la ejecución no se sorprenda.

### §3.2 F8.1 — Servicio de beneficios (`benefitsService.js`)

| Aspecto | Detalle |
|---|---|
| **Qué** | Crear `apps/pos/src/services/benefitsService.js` que llama al contrato #26 |
| **Patrón** | Idéntico a [`openAccountsService.js`](../NUEVO-POS/apps/pos/src/services/openAccountsService.js:1): devuelve `{outcome, reason, data}`, usa `withRetries`, nunca lanza |
| **Degradación** | 503 → `outcome: 'error'`, `reason: 'crm_no_disponible'`. El POS cobra a precio de lista. |
| **Gate** | `benefitsService.f8_1.test.jsx` — 5 criterios: éxito, 404, 503, teléfono vacío (guarda local), nunca lanza |
| **Riesgo** | Bajo. Es un clon del patrón ya probado en F5.1. |

### §3.3 F8.2 — Servicio de notificaciones (`notificationsService.js`)

| Aspecto | Detalle |
|---|---|
| **Qué** | Crear `apps/pos/src/services/notificationsService.js` que llama al contrato #27 |
| **Patrón** | Idéntico a `benefitsService.js` |
| **Degradación** | 503 → `reason: 'cola_no_disponible'`. El ticket ya se cobró; el envío queda pendiente. |
| **Gate** | `notificationsService.f8_2.test.jsx` — 5 criterios: éxito, 422 (canal sin dato), 503, evento_id vacío, nunca lanza |
| **Riesgo** | Bajo. |

### §3.4 F8.3 — Hook `useCustomerIdentification.js`

| Aspecto | Detalle |
|---|---|
| **Qué** | Hook que gestiona el estado de identificación del cliente (teléfono, beneficios, cargando, error) |
| **Patrón** | Idéntico a [`useOpenAccounts.js`](../NUEVO-POS/apps/pos/src/hooks/useOpenAccounts.js:52): inyectable para tests |
| **Expone** | `{ cliente, beneficios, cargando, error, identificar, limpiar }` |
| **Gate** | `useCustomerIdentification.f8_3.test.jsx` — 6 criterios: identificar OK, degradación, limpiar, teléfono vacío, no llama con vacío, inyectable |
| **Riesgo** | Bajo. |

### §3.5 F8.4 — Componente `CustomerIdentificationPanel.jsx`

| Aspecto | Detalle |
|---|---|
| **Qué** | Panel modal para teclear el teléfono + mostrar beneficios |
| **UX heredada (§6.8)** | Se hereda la **integración** del viejo POS (botón en el header), se reescribe la **implementación** |
| **Selectores** | `aria-label="Identificación del cliente"` (dialog), `id="input-telefono-cliente"`, `id="btn-buscar-cliente"`, `aria-label="Cerrar"` |
| **Estados** | Sin identificar / cargando / identificado con beneficios / identificado sin beneficios / CRM caído |
| **Gate** | `CustomerIdentificationPanel.f8_4.test.jsx` — 7 criterios |
| **Riesgo** | Medio. Es UI nueva, pero sigue el patrón de [`OrderProgrammingModal.jsx`](../NUEVO-POS/apps/pos/src/components/OrderProgrammingModal.jsx:134). |

### §3.6 F8.5 — Componente `TicketDeliveryPanel.jsx`

| Aspecto | Detalle |
|---|---|
| **Qué** | Paso post-cobro: 🖨️ Imprimir / 📱 WhatsApp / ✉️ Email / Omitir |
| **Dónde vive (defecto D-1 corregido)** | Es un **overlay del screen** ([`RetailVisionPOS.jsx`](../NUEVO-POS/apps/pos/src/RetailVisionPOS.jsx:80)), NO un paso dentro de [`CheckoutScreen.jsx`](../NUEVO-POS/apps/pos/src/components/CheckoutScreen.jsx:46). `CheckoutScreen` es el modal de **pago** y se cierra al cobrar; el paso de entrega aparece **después**, cuando `cobrar()` ya resolvió. Se renderiza junto a `OverlayExito`, con el mismo patrón de overlay. |
| **Regla dura** | Imprimir **siempre** disponible (RN-87). WhatsApp/Email son adicionales. |
| **Precarga (RN-92)** | Si el cliente está identificado, precarga teléfono/correo desde el CRM. No se vuelve a teclear. |
| **Selectores** | `aria-label="Entrega del ticket"` (dialog), `id="btn-imprimir"`, `id="btn-whatsapp"`, `id="btn-email"`, `id="btn-omitir"` |
| **Gate** | `TicketDeliveryPanel.f8_5.test.jsx` — 8 criterios |
| **Riesgo** | Medio. |

### §3.7 F8.6 — Cableado end-to-end en `RetailVisionPOS`

| Aspecto | Detalle |
|---|---|
| **Qué** | Botón "👤 Cliente" en [`POSHeader.jsx`](../NUEVO-POS/apps/pos/src/components/POSHeader.jsx:47) + render del panel + render del paso de entrega tras el cobro |
| **Props nuevas del header (defecto D-2 corregido)** | `POSHeader` recibe **2 props nuevas**: `onAbrirCliente` (abre el panel) y `clienteIdentificado` (resalta el botón, igual que `pedidoProgramado`). `RetailVisionPOS` debe pasarlas. |
| **Punto delicado** | El paso de entrega aparece **después** de `cobrar()`. Si el cobro falla, no aparece. |
| **Gate** | `RetailVisionPOS.f8_6.test.jsx` — 9 criterios (incluye: el POS nunca importa `Order` ni escribe `customers`) |
| **Riesgo** | Medio-alto. Es el cableado; aquí es donde F7.5a encontró el bug `ticket` vs `ticketId`. |

### §3.8 F8.7 — Cierre: CI completo + ficha + commit + push

| Aspecto | Detalle |
|---|---|
| **Qué** | `npm run ci` verde + `FICHA_F8_CRM_NOTIFICACIONES.md` + actualizar §7 del Plan Maestro + commit + push |
| **Gate** | Los 7 guards limpios + todos los tests verdes |
| **Riesgo** | Bajo. |

---

## §4. LA PUERTA DE LA FASE 8 (criterios de aceptación)

La Fase 8 se considera cerrada cuando **los 9 criterios** se cumplen:

| # | Criterio | Cómo se verifica |
|---|---|---|
| 1 | Los contratos #26 y #27 están declarados | `listar_contratos()` devuelve 27 |
| 2 | Las reglas RN-82..RN-93 están declaradas | `listar_reglas()` devuelve 93 |
| 3 | El botón "Cliente" permite identificar por teléfono | Gate F8.4 |
| 4 | El panel muestra los beneficios que el CRM devuelve | Gate F8.4 |
| 5 | El paso de entrega ofrece Imprimir / WhatsApp / Email | Gate F8.5 |
| 6 | **Si el CRM está caído, la venta continúa sin beneficios** | Gate F8.1 + F8.6 |
| 7 | **Si Notificaciones está caído, la venta continúa con impresión** | Gate F8.2 + F8.6 |
| 8 | **El POS nunca importa `Order` ni escribe `customers`/`notification_outbox`** | Guard de frontera (E-15) |
| 9 | El POS nunca define ni persiste la política de lealtad | Gate F8.6 |

---

## §5. TRAZABILIDAD (regla → sub-fase → test)

| Origen | Sub-fase | Test |
|---|---|---|
| Contrato #26 | F8.0 + F8.1 | `test_f2_frontera` (27) + `benefitsService.f8_1` |
| Contrato #27 | F8.0 + F8.2 | `test_f2_frontera` (27) + `notificationsService.f8_2` |
| RN-82, RN-83 | F8.3 + F8.4 | `useCustomerIdentification.f8_3` + `CustomerIdentificationPanel.f8_4` |
| RN-84 | F8.4 + F8.6 | `CustomerIdentificationPanel.f8_4` + `RetailVisionPOS.f8_6` |
| RN-86, RN-87 | F8.5 + F8.6 | `TicketDeliveryPanel.f8_5` + `RetailVisionPOS.f8_6` |
| RN-91 | F8.4 | `CustomerIdentificationPanel.f8_4` |
| RN-92 | F8.5 | `TicketDeliveryPanel.f8_5` |
| RN-85, RN-88, RN-89, RN-90, RN-93 | F8.0 (solo declaración) | `test_f3_comportamiento` (enunciado, no comportamiento) |
| DT-02 (dinero String) | F8.1 + F8.2 | `benefitsService.f8_1` + `notificationsService.f8_2` |
| DT-07 (degradación) | F8.1 + F8.2 + F8.6 | Los 3 gates |
| §6.8 (UX heredada) | F8.4 + F8.5 | Los 2 gates de componente |
| A-02 (frontera) | F8.6 | Guard E-15 |

---

## §6. LO QUE ESTA FASE **NO** HACE

1. **No construye el CRM.** Ni sus tablas, ni sus pantallas, ni su lógica de lealtad.
2. **No construye Notificaciones.** Ni el worker, ni la cola, ni la integración con WhatsApp.
3. **No define la política de lealtad.** Eso es del CRM (RN-88).
4. **No toca el ERP instalado.** Cero líneas de código de producción del ERP viejo.
5. **No bloquea la venta jamás.** Es la regla dura de toda la fase.

---

## §7. RIESGOS Y MITIGACIONES

| Riesgo | Probabilidad | Mitigación |
|---|---|---|
| El bug `ticket` vs `ticketId` de F7.5a se repite en el cableado | Media | El gate F8.6 monta la pantalla REAL y verifica el render |
| El POS importa `Order` por descuido | Baja | Guard E-15 + criterio 8 de la puerta |
| La degradación no se prueba (solo el camino feliz) | Media | Cada gate tiene un criterio explícito de "proveedor caído" |
| Se construye código de módulos ajenos | Baja | §1.2 lo prohíbe explícitamente |
| La numeración vuelve a quedar obsoleta | Baja | §0.2 documenta el hallazgo; el `registry.py` es la fuente de verdad |
| **El test "sin huecos" de reglas rompe al añadir RN-82..RN-93** | **Alta** | §3.1 lista las 3 aserciones exactas a actualizar (defecto D-5) |
| **El paso de entrega se cablea dentro de `CheckoutScreen` por error** | **Media** | §3.6 fija que vive en el screen, no en el modal de pago (defecto D-1) |

---

## §8. REGISTRO DE CAMBIOS

| Versión | Fecha | Cambio |
|---|---|---|
| 1.0 | 30 Sep 2026 | Versión inicial. Verificación previa (§0) con el hallazgo de la doble numeración obsoleta. |
| 2.0 | 30 Sep 2026 | Autocrítica verificada contra el código (§9). Corregidos 5 defectos: D-1 (el paso de entrega vive en el screen, no en `CheckoutScreen`), D-2 (2 props nuevas del header), D-3 (archivos de test reales citados), D-4 (declarar ≠ implementar: 7 se implementan, 5 solo se declaran), D-5 (radio de impacto: 2 archivos de test de contratos + 3 aserciones de reglas). |

---

## §9. AUTOCRÍTICA DE LA v1.0 (verificada contra el código)

> **Método:** cada afirmación de la v1.0 se contrastó con el repositorio. No se asumió nada.
> Se encontraron **5 defectos**, 2 de ellos críticos. Todos están corregidos en esta v2.0.

### §9.1 D-1 (CRÍTICA) — El paso de entrega NO vive en `CheckoutScreen`

| | |
|---|---|
| **Lo que decía la v1.0** | §3.6: *"Paso post-cobro… se hereda la integración del `CheckoutScreen`"* |
| **Lo que dice el código** | [`CheckoutScreen.jsx:46`](../NUEVO-POS/apps/pos/src/components/CheckoutScreen.jsx:46) es el modal de **pago** (`onConfirmar`/`onCancelar`); se cierra al cobrar. El estado post-cobro (éxito) vive en el **screen** ([`RetailVisionPOS.jsx`](../NUEVO-POS/apps/pos/src/RetailVisionPOS.jsx:80)), junto a `OverlayExito`. |
| **Por qué es crítico** | Si el paso de entrega se cablea dentro de `CheckoutScreen`, el panel nunca aparecería (el modal ya se cerró) o aparecería antes de cobrar (violando RN-86: el encolado va en la transacción del ticket). |
| **Corrección** | §3.6 ahora fija que el panel es un **overlay del screen**, renderizado tras `cobrar()`. |

### §9.2 D-2 — Faltaban las 2 props nuevas del header

| | |
|---|---|
| **Lo que decía la v1.0** | §3.7: *"Botón '👤 Cliente' en `POSHeader.jsx`"* — sin decir cómo. |
| **Lo que dice el código** | [`POSHeader.jsx:47-60`](../NUEVO-POS/apps/pos/src/components/POSHeader.jsx:47) recibe props explícitas; el botón de pedido usa `onAbrirPedido` + `pedidoProgramado` (líneas 58-59, 105-117). |
| **Corrección** | §3.7 declara las 2 props nuevas: `onAbrirCliente` y `clienteIdentificado`, y que `RetailVisionPOS` debe pasarlas. |

### §9.3 D-3 — Los archivos de test no estaban citados

| | |
|---|---|
| **Lo que decía la v1.0** | §3.1: *"el test de contratos pasa de 25 a 27"* — sin citar el archivo. |
| **Lo que dice el código** | El conteo de contratos se afirma en **2 archivos** ([`test_f2_frontera.py:150`](../NUEVO-POS/apps/api/tests/test_f2_frontera.py:150), [`test_f7_contratos_ia.py:67`](../NUEVO-POS/apps/api/tests/test_f7_contratos_ia.py:67)). |
| **Corrección** | §3.1 cita los archivos y líneas exactos. |

### §9.4 D-4 — "Declarar" y "implementar" se confundían

| | |
|---|---|
| **Lo que decía la v1.0** | §2.3: *"solo las que tocan el POS se implementan… se declaran aquí para reservar el número"* — pero §4 criterio 2 exigía `listar_reglas()` = 93, es decir, **todas** declaradas. |
| **Por qué es un defecto** | La matriz regla→test exige que **toda** regla declarada tenga un nombre de test ([`test_f3_comportamiento.py:44`](../NUEVO-POS/apps/api/tests/test_f3_comportamiento.py:44)). Declarar 12 y testear 7 rompería la matriz. |
| **Corrección** | §2.3 ahora distingue explícitamente: **7 se implementan** (con test de comportamiento) y **5 se declaran** (con test de enunciado). §5 lo refleja en la trazabilidad. |

### §9.5 D-5 (CRÍTICA) — El radio de impacto de los tests de conteo era mayor

| | |
|---|---|
| **Lo que decía la v1.0** | §3.1: *"el test de reglas de 81 a 93"* — en singular. |
| **Lo que dice el código** | [`test_f3_comportamiento.py`](../NUEVO-POS/apps/api/tests/test_f3_comportamiento.py:30) tiene **3 aserciones** que rompen: matriz=81 (línea 30), rango `RN-01..RN-81` **sin huecos** (línea 36), `listar_reglas()`=81 (línea 64). La del rango es la más estricta: exige la lista completa. |
| **Por qué es crítico** | Sin actualizar las 3, el CI queda rojo y la fase no cierra. La v1.0 habría subestimado el trabajo. |
| **Corrección** | §3.1 lista las 3 aserciones exactas; §7 añade el riesgo con probabilidad **Alta**. |

### §9.6 Lo que la autocrítica NO encontró (y por qué)

| Afirmación de la v1.0 | Veredicto |
|---|---|
| Los contratos son #26 y #27 | ✅ Verificado contra `registry.py` |
| Las reglas son RN-82..RN-93 | ✅ Verificado contra `registry.py` |
| No existe código de CRM/Notificaciones | ✅ Verificado por búsqueda |
| El patrón de servicio es el de `openAccountsService.js` | ✅ Verificado (líneas 26-65) |
| La Opción A (POS solo, con degradación) es la correcta | ✅ Coherente con el Plan Maestro §7 |
| El hallazgo de la doble numeración obsoleta | ✅ Verificado (la propuesta se autocorrigió mal) |

---

**Fin del plan v2.0 — pendiente de aprobación del dueño.**
