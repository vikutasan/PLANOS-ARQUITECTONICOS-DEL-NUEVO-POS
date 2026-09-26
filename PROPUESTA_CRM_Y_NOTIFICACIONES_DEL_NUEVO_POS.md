# PROPUESTA — CRM Y NOTIFICACIONES DEL NUEVO POS

**Documento 13 — Los dos módulos que faltaban en el plano**

> **Propósito de este documento.** El [`CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md`](./CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md)
> define **17 contratos** entre el POS y los módulos del ERP. Ninguno de esos 17 contratos
> cubre dos necesidades que el negocio ya expresó:
>
> 1. **Saber a quién se le está cobrando** (público general vs. cliente registrado) y
>    **aplicarle sus beneficios** (precio preferente, descuento, puntos de lealtad).
> 2. **Entregar el ticket por un canal distinto al papel** (WhatsApp o correo electrónico).
>
> Este documento **propone** cómo integrar esas dos necesidades como **dos módulos
> monolíticos del ERP** que hablan con el POS **mediante contrato**, sin que el POS lea
> jamás una tabla ajena. Extiende la matriz del Documento 9 de **17 a 19 contratos**.
>
> **Estado.** **Propuesta.** No es un documento maestro aprobado todavía. Se somete a
> revisión del dueño del negocio antes de incorporarse al índice del [`README.md`](./README.md).
>
> **Anclaje.** Anclado al commit `5802f45` (V23) del ERP actual, igual que el resto del
> plano. La ingeniería inversa de los acoplamientos citados se hizo sobre `fe9f6ed`
> (tag `v22-estable-fe9f6ed`).

---

## SECCIÓN 0 — EL PROBLEMA QUE ESTE DOCUMENTO RESUELVE

### 0.1 Lo que el negocio pidió (textual)

> *"Deseo que el nuevo POS al momento de cobrar una cuenta me dé la opción de imprimir el
> ticket (ya lo hace) o enviarlo por WhatsApp (previa anotación del teléfono del cliente)
> o por e-mail (previa anotación del correo del cliente)."*

> *"Llegará un momento en que tendremos una base de datos de clientes que nos será útil
> para administrar tarjetas de lealtad, enviar tickets, programar promociones y descuentos."*

### 0.2 Lo que el plano actual NO cubre

| Necesidad | ¿Existe en el plano? | Evidencia |
|-----------|----------------------|-----------|
| Identificar al cliente al cobrar | ❌ **No** | `tickets` no tiene `customer_phone` ni `customer_email` ([`MODELO_DE_DATOS_DEL_NUEVO_POS.md`](./MODELO_DE_DATOS_DEL_NUEVO_POS.md)) |
| Base de datos de clientes del POS | ❌ **No** | `Clientes` no aparece en la SECCIÓN 5 del [`PLANO ARQUITECTONICO PARA EL NUEVO POS.md`](./PLANO%20ARQUITECTONICO%20PARA%20EL%20NUEVO%20POS.md:858) |
| Enviar ticket por WhatsApp / correo | ❌ **No** | No hay contrato de notificaciones en la matriz del Documento 9 |
| Tarjetas de lealtad / puntos | ❌ **No** | No hay ledger de lealtad en el modelo de datos |
| Promociones y descuentos configurables | ❌ **No** | No hay módulo de promociones |
| WhatsApp como canal | ⚠️ **Parcial** | Solo existe para **reparto** ([`GrandezaOrderRequestsTab.jsx`](../../apps/pos/GrandezaOrderRequestsTab.jsx:231)), no para tickets |
| Normalización de teléfono | ⚠️ **Precedente** | [`_normalizar_telefono()`](../../apps/api/modules/grandeza/service.py:68) — se reutiliza el criterio |

### 0.3 La decisión de arquitectura

**No es un módulo. Son dos.** El negocio lo planteó como "un módulo de clientes", pero
son **dos responsabilidades distintas** con **ciclos de vida distintos**:

| Módulo | Responsabilidad | Dueño de qué tablas |
|--------|-----------------|---------------------|
| **Clientes (CRM)** | Identidad del cliente, lealtad, promociones, beneficios | `customers`, `loyalty_ledger`, `promotions`, `customer_benefits` |
| **Notificaciones** | Entrega de mensajes por canal (WhatsApp, correo, papel) | `notification_outbox`, `notification_log`, `channel_config` |

**Por qué separarlos.** El CRM **decide** *qué* beneficio aplica y *a quién* se le manda.
Notificaciones **ejecuta** *cómo* se entrega. Si mañana se agrega un canal nuevo (SMS,
Telegram), **solo cambia Notificaciones**; el CRM y el POS no se tocan. Si mañana cambia
la regla de lealtad, **solo cambia el CRM**; Notificaciones y el POS no se tocan.

**Acoplamiento entre ellos:** Notificaciones **consume** del CRM el destinatario
(`customers.contacto`), pero **no** lee su tabla: pide por contrato
`clientes.contacto_para_notificar`.

---

## SECCIÓN 1 — LOS DOS MÓDULOS COMO MONOLITO MODULAR

### 1.1 Qué significa "monolito modular" aquí

El ERP es **un solo despliegue** (un proceso, una base de datos por sucursal). "Modular"
significa que **cada módulo es dueño de sus tablas** y **nadie más las escribe**. La
comunicación entre módulos es **por contrato** (llamada de operación), nunca por lectura
directa de tabla.

```
┌─────────────────────────────────────────────────────────────┐
│                    ERP (un despliegue)                       │
│                                                              │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐  │
│  │  POS     │   │ CLIENTES │   │NOTIFICA- │   │  CAJA    │  │
│  │          │   │  (CRM)   │   │ CIONES   │   │          │  │
│  │ tickets  │   │customers │   │ outbox   │   │ sessions │  │
│  │ items    │   │ loyalty  │   │ log      │   │ movements│  │
│  │ payments │   │promotions│   │ channels │   │          │  │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘   └────┬─────┘  │
│       │              │              │              │         │
│       └──────────────┴──────────────┴──────────────┘         │
│                    CONTRATOS (operaciones)                    │
│              NADIE lee la tabla de otro módulo                │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Los 3 principios (heredados del Documento 9)

| # | Principio | Aplicado a estos módulos |
|---|-----------|--------------------------|
| **P-01** | El dueño de la tabla es el único que la escribe | Solo Clientes escribe `customers`. Solo Notificaciones escribe `notification_outbox`. |
| **P-02** | El consumidor pide por operación, no por tabla | El POS pide "¿este teléfono tiene beneficios?", no "dame la tabla `customers`". |
| **P-03** | El contrato es estable; la tabla es libre | Clientes puede refactorizar `loyalty_ledger` sin romper al POS. |

### 1.3 El módulo Clientes (CRM) — alcance

**Dueño de 4 tablas:**

| Tabla | Qué guarda | Patrón |
|-------|-----------|--------|
| `customers` | Identidad: teléfono (clave), nombre, correo, RFC, fecha de alta | Entidad |
| `loyalty_ledger` | **Ledger inmutable** de movimientos de puntos (gana / canjea / expira) | Ledger (Regla de Oro #10) |
| `promotions` | Reglas de promoción: vigencia, condición, beneficio, alcance | Configuración |
| `customer_benefits` | **Caché derivado** del saldo de puntos y del nivel de lealtad | Caché (se recalcula del ledger) |

**Regla dura del CRM:** el saldo de puntos **nunca** se hace `UPDATE`. Se **asienta** en
`loyalty_ledger` y `customer_benefits` es una **proyección** recalculable. Igual que
`movimientos_inventario` → `stock_almacen` en Almacenes.

**Lo que el CRM NO hace:** no imprime, no envía WhatsApp, no envía correo. Solo **decide**
el beneficio y **expone** el contacto.

### 1.4 El módulo Notificaciones — alcance

**Dueño de 3 tablas:**

| Tabla | Qué guarda | Patrón |
|-------|-----------|--------|
| `notification_outbox` | Cola de mensajes a enviar (evento, canal, destinatario, payload, estado) | **Outbox** (Regla de Oro #7) |
| `notification_log` | Bitácora de envíos: qué se envió, cuándo, por qué canal, resultado | Bitácora |
| `channel_config` | Credenciales y plantillas por canal (WhatsApp Business, SMTP) | Configuración |

**Regla dura de Notificaciones:** el envío **nunca bloquea la venta**. El POS **encola**
en `notification_outbox` dentro de la **misma transacción** del ticket; el **worker**
envía después. Si WhatsApp está caído, el ticket ya se cobró y el mensaje queda en cola.

**Idempotencia:** cada mensaje lleva `evento_id`. `uq_notification_evento_canal
(evento_id, canal)` impide el doble envío si el POS reintenta.

---

## SECCIÓN 2 — LOS CONTRATOS NUEVOS (18 y 19)

Se añaden **2 contratos** a la matriz del Documento 9. Se numeran **18** y **19** para
continuar la secuencia existente (1–17).

### 2.1 CONTRATO 18 — `clientes.beneficios_para_ticket`

```
CONTRATO clientes.beneficios_para_ticket
  Consumidor:   POS
  Proveedor:    Clientes (CRM)
  Acoplamiento actual:  NO EXISTE. Hoy el POS no sabe quién es el cliente.
                        →  apps/pos/components/CheckoutScreen.jsx (no hay campo de cliente)
  Problema:     El POS cobra a ciegas. No puede aplicar precio preferente, descuento
                ni acumular puntos porque no sabe a quién le cobra.
  Contrato nuevo:  El POS envía el identificador del cliente (teléfono normalizado) y el
                   carrito (lista de {product_id, qty, unit_price}); el CRM devuelve los
                   beneficios aplicables y el saldo de puntos.
                   Entrada:  { telefono, items: [{product_id, qty, unit_price}] }
                   Salida:   { customer_id, nombre, nivel,
                               descuentos: [{ tipo, valor, aplica_a, motivo }],
                               puntos_a_ganar, puntos_disponibles,
                               puede_canjear: bool }
  Garantías:    - El CRM es el ÚNICO que decide el beneficio. El POS no calcula descuentos.
                - La respuesta es determinista para el mismo carrito y el mismo día.
                - Si el cliente no existe, devuelve `customer_id: null` y cero beneficios
                  (NO es un error: es "público general").
  Errores:      - 404 si el teléfono está mal formado (no normalizable).
                - 503 si el CRM no responde → el POS cobra SIN beneficios (degradación
                  elegante) y lo registra en el ticket como `beneficios_no_aplicados`.
```

**Por qué este contrato NO bloquea la venta.** Si el CRM cae, el POS cobra a precio de
lista. El beneficio se pierde, pero la venta no. Es la misma filosofía que Almacenes:
solo Productos y Seguridad pueden bloquear (ver Documento 9, línea 731).

### 2.2 CONTRATO 19 — `notificaciones.encolar_ticket`

```
CONTRATO notificaciones.encolar_ticket
  Consumidor:   POS
  Proveedor:    Notificaciones
  Acoplamiento actual:  NO EXISTE. Hoy el ticket solo se imprime.
                        →  apps/pos/utils/ticketGenerator.js (solo genera HTML de impresión)
  Problema:     El negocio quiere enviar el ticket por WhatsApp o correo, pero no hay
                canal ni cola. Enviar de forma síncrona bloquearía el cobro.
  Contrato nuevo:  El POS encola el ticket para uno o más canales. La operación es
                   TRANSACCIONAL con el guardado del ticket (mismo commit).
                   Entrada:  { evento_id, ticket_uuid, canales: [WHATSAPP|EMAIL],
                               destinatario: { telefono?, email? },
                               payload: { folio, total, items, fecha } }
                   Salida:   { encolado: true, mensajes: [{ canal, estado: PENDIENTE }] }
  Garantías:    - La operación NUNCA falla por culpa del canal externo: solo escribe en
                  la cola. El envío real lo hace el worker.
                - Idempotente por `evento_id`: reintentar no duplica el mensaje
                  (uq_notification_evento_canal).
                - El payload es una PROYECCIÓN del ticket, no la tabla `tickets`.
  Errores:      - 422 si el destinatario no tiene el dato del canal pedido
                  (ej. canal EMAIL sin correo).
                - 503 si la cola no está disponible → el POS NO revierte el cobro;
                  muestra "ticket impreso, envío pendiente" y reintenta por outbox.
```

**Por qué este contrato es Outbox.** El POS **no espera** a que WhatsApp entregue. Escribe
en la cola y sigue. Es el mismo patrón que `almacenes.consumir_por_venta` (Documento 9,
§2.2): el evento se persiste en la misma transacción del ticket y el efecto ocurre después.

### 2.3 La matriz extendida (17 → 19)

| # | Contrato | Consumidor | Proveedor | Reemplaza a | Estado hoy |
|---|----------|-----------|-----------|-------------|------------|
| 1 | `catalogo.productos_para_venta` | POS | Catálogo | Lectura de `products` | Ya existe (parcial) |
| 2 | `almacenes.consumir_por_venta` | POS | Almacenes | Escritura de `products.stock` | **Deuda** |
| 3 | `almacenes.disponibilidad` | POS | Almacenes | Lectura de `products.stock` | **Deuda** |
| 4 | `produccion.disponible_para_vender` | POS | Producción | Lectura de `doughs` | **Deuda** |
| 5 | `pos.eventos_auditables` | Auditoría | POS | (ya existe) | Cicatriz |
| 6 | `pos.resumen_de_venta` | Estadísticas | POS | Lectura de `tickets` | **Deuda** |
| 7 | `seguridad.identidad_del_empleado` | POS | Seguridad | Lectura de `employees` | **Deuda** |
| 8 | `seguridad.validar_pin` | POS | Seguridad | Lectura de `employees` | **Deuda** |
| 9 | `caja.sesion_activa` | POS | Caja | (ya existe) | **Ya existe** |
| 10 | `caja.abrir_turno` | POS | Caja | (ya existe) | **Ya existe** |
| 11 | `caja.registrar_movimiento` | POS | Caja | (ya existe) | **Ya existe** |
| 12 | `caja.resumen_del_turno` | POS | Caja | (ya existe) | **Ya existe** |
| 13 | `caja.cerrar_turno` | POS | Caja | (ya existe) | **Ya existe** |
| 14 | `caja.reporte_diario` | POS / Estadísticas | Caja | (ya existe) | **Ya existe** |
| 15 | `pedidos.registrar_desde_ticket` | POS | Pedidos | Escritura directa de `orders` | **Deuda** |
| 16 | `pedidos.pedido_del_ticket` | POS | Pedidos | Lectura de `orders` | **Deuda** |
| 17 | `vision.reconocer_producto` | POS | Visión | Motor interno del POS | **Deuda** |
| **18** | **`clientes.beneficios_para_ticket`** | **POS** | **Clientes (CRM)** | **Nada (no existía)** | **Nuevo** |
| **19** | **`notificaciones.encolar_ticket`** | **POS** | **Notificaciones** | **Nada (no existía)** | **Nuevo** |

**Lectura de la matriz extendida.** De 19 contratos: **2 son nuevos** (este documento),
**1 es cicatriz**, **1 ya existe parcial**, **6 ya existen completos** y **9 son deuda**.

**Contratos internos entre los dos módulos nuevos** (no aparecen en la matriz del POS
porque el POS no participa):

| Contrato interno | Consumidor | Proveedor | Para qué |
|------------------|-----------|-----------|----------|
| `clientes.contacto_para_notificar` | Notificaciones | Clientes | Obtener el teléfono/correo del destinatario sin leer `customers` |
| `clientes.registrar_canje` | POS | Clientes | Confirmar el canje de puntos (reserva → confirmación) |

---

## SECCIÓN 3 — EL FLUJO EN EL POS

### 3.1 Principio rector: la identificación NO precede a la carga

**Corrección importante.** No es cierto que el cajero deba decidir "público general vs.
registrado" **antes** de cargar los ítems. Eso contradice **RN-08** (cada operación de
ítem es atómica e inmediata) y **RN-10** (el total siempre se recalcula desde los ítems
persistidos). El POS debe permitir cargar primero y **decidir el cliente en cualquier
momento antes de finalizar**.

**Diseño:** un botón **"Cliente"** en el [`POSHeader`](../PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS/ESPECIFICACION_DE_INTERFACES_POS.md:105)
(FICHA 01, `RetailVisionPOS`). Por defecto el ticket es **"Público General"**. El cajero
puede pulsarlo en cualquier momento; si lo pulsa, se abre un panel de identificación.

### 3.2 Los 3 caminos de identificación

| Camino | Cuándo | Cómo | Resultado |
|--------|--------|------|-----------|
| **A — Público General** | Cliente no registrado o no quiere dar datos | No se pulsa nada | `customer_id: null`. Sin beneficios. Ticket solo impreso. |
| **B — Cliente registrado** | El cliente da su teléfono | Botón "Cliente" → teclear teléfono → buscar | Se llama `clientes.beneficios_para_ticket`. Se aplican descuentos. |
| **C — Alta rápida** | Cliente nuevo que quiere registrarse | Botón "Cliente" → "Nuevo" → nombre + teléfono | Se crea en `customers` y se aplica camino B. |

**El teléfono es la clave.** Se normaliza con el criterio de
[`_normalizar_telefono()`](../../apps/api/modules/grandeza/service.py:68). El correo es
**opcional** y solo se pide si el cliente quiere el ticket por e-mail.

### 3.3 El flujo de cobro completo (paso a paso)

```
1. El cajero carga ítems en RetailVisionPOS (RN-08: cada ítem se persiste al agregarlo).
   → El ticket nace como "Público General".

2. [Opcional] El cajero pulsa "Cliente" en el POSHeader.
   → Camino A: no hace nada.
   → Camino B/C: identifica al cliente.

3. Si hay cliente, el POS llama clientes.beneficios_para_ticket(carrito).
   → El CRM devuelve descuentos + puntos.
   → El POS AGREGA los descuentos como LÍNEAS NEGATIVAS al ticket.
   → RN-10: el total se recalcula desde los ítems persistidos (incluidas las negativas).

4. El cajero pulsa "COBRAR" → se abre CheckoutScreen (FICHA 08).
   → Elige método(s) de pago, teclea montos, "FINALIZAR VENTA".

5. Al finalizar, en UNA SOLA TRANSACCIÓN:
   a. Se guarda el ticket (estado PAID).
   b. Se llama almacenes.consumir_por_venta (outbox).
   c. Si hay cliente: se llama clientes.registrar_canje (si canjeó puntos).
   d. Se llama notificaciones.encolar_ticket con los canales elegidos.

6. El POS muestra el resultado:
   → "Ticket impreso" (siempre, salvo que el negocio lo desactive).
   → "Enviado a WhatsApp" / "Enviado a e-mail" (o "en cola" si el canal está lento).

7. El worker de Notificaciones envía en segundo plano.
   → Si falla, reintenta. El ticket YA está cobrado (RN-03: PAID es inmutable).
```

### 3.4 Dónde se eligen los canales de envío

En el [`CheckoutScreen`](../PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS/ESPECIFICACION_DE_INTERFACES_POS.md:439)
(FICHA 08), **después** de "FINALIZAR VENTA", aparece un paso de **entrega del ticket**:

```
┌─────────────────────────────────────────────┐
│  VENTA COMPLETADA — Folio V0042             │
│                                             │
│  ¿Cómo entregamos el ticket?                │
│                                             │
│  [ 🖨️  IMPRIMIR ]  ← siempre disponible     │
│  [ 📱 WHATSAPP  ]  ← si hay teléfono        │
│  [ ✉️  E-MAIL   ]  ← si hay correo          │
│                                             │
│  [ OMITIR ]                                 │
└─────────────────────────────────────────────┘
```

**Regla dura:** "IMPRIMIR" **siempre** está disponible y es el comportamiento por defecto
(CA-19: el ticket impreso debe ser idéntico al POS actual). WhatsApp y e-mail son
**adicionales**, nunca sustitutos.

### 3.5 Cómo se aplican los beneficios (dos tipos)

| Tipo | Cuándo se aplica | Cómo | Ejemplo |
|------|------------------|------|---------|
| **Tipo A — Beneficio de precio** | **ANTES** de cobrar, sobre el carrito | Línea negativa en el ticket | "10% en pan dulce", "precio B2B" |
| **Tipo B — Beneficio acumulable** | **DESPUÉS** de cobrar, como evento | Asiento en `loyalty_ledger` | "1 punto por cada $10" |

**Por qué la distinción.** El Tipo A **cambia el total** y por eso debe estar en el ticket
antes de PAID (RN-11: el precio se congela al agregar). El Tipo B **no cambia el total**;
es un evento posterior que se asienta en el ledger.

**El canje de puntos (reserva → confirmación).** Si el cliente canjea puntos:

1. **Antes de cobrar:** el POS llama `clientes.beneficios_para_ticket` y ve
   `puntos_disponibles`. Si el cajero decide canjear, el POS **reserva** los puntos.
2. **Al cobrar:** en la transacción del ticket, el POS llama `clientes.registrar_canje`.
3. **Si el cobro falla:** la reserva se libera (no se descuentan puntos).

Esto evita el bug clásico de "desconté puntos pero la venta no se completó".

---

## SECCIÓN 4 — DÓNDE SE CONFIGURAN LAS PROMOCIONES

### 4.1 La decisión: en el módulo Clientes, no en Vista General ni en el POS

| Candidato | ¿Por qué NO? |
|-----------|--------------|
| **Vista General** | Es el módulo de **configuración transversal** (zona horaria, moneda, sucursal). No es un módulo de negocio. Ver [`DIRECTRICES_TRANSVERSALES_DEL_ERP.md`](./DIRECTRICES_TRANSVERSALES_DEL_ERP.md) (DT-06). |
| **POS** | El POS **consume** beneficios, no los **define**. Si el POS los definiera, rompería P-01 (el dueño de la tabla es el único que la escribe). |
| **Clientes (CRM)** | ✅ **Correcto.** Las promociones son una propiedad del CRM: dependen de la identidad del cliente y de su historial. |

### 4.2 Las dos superficies del módulo Clientes

El módulo Clientes tiene **una entrada en el sidebar** (en `allModules` de
[`ExperimentCenterUI.jsx`](../../apps/ExperimentCenterUI.jsx:343)) con **dos pestañas**:

| Pestaña | Para qué | Quién la usa |
|---------|----------|--------------|
| **Clientes** | Ver/editar clientes, ver su saldo de puntos, su historial de compras | Operación diaria |
| **Promociones** | Crear/editar promociones, definir vigencia, condición y beneficio | Configuración (dueño) |

**Por qué una sola entrada con dos pestañas.** Son el mismo dominio (el cliente y lo que
se le ofrece). Separarlas en dos entradas del sidebar fragmentaría el modelo mental.

### 4.3 El ciclo de vida de una promoción

```
BORRADOR → ACTIVA → (VENCIDA | PAUSADA)
```

- **BORRADOR:** se está configurando, no aplica.
- **ACTIVA:** dentro de vigencia, el CRM la considera en `beneficios_para_ticket`.
- **VENCIDA:** pasó la fecha de fin. No se borra (histórico).
- **PAUSADA:** el dueño la desactiva temporalmente sin perder la configuración.

**Regla dura:** una promoción **nunca se borra**, solo se vence o se pausa. Así el ticket
histórico puede explicar por qué aplicó un descuento (auditoría).

---

## SECCIÓN 5 — EL FLUJO DE NOTIFICACIONES (WHATSAPP / E-MAIL)

### 5.1 El patrón Outbox en detalle

```
POS                          Notificaciones                    Canal externo
 │                                  │                                │
 │ 1. encolar_ticket(evento_id)     │                                │
 │─────────────────────────────────>│                                │
 │    (misma transacción del ticket)│                                │
 │                                  │ 2. escribe en notification_outbox
 │ <────────────────────────────────│    estado = PENDIENTE          │
 │    { encolado: true }            │                                │
 │                                  │                                │
 │ 3. El POS sigue. La venta cerró. │                                │
 │                                  │ 4. El worker toma la cola      │
 │                                  │───────────────────────────────>│
 │                                  │    (WhatsApp Business API /    │
 │                                  │     SMTP)                      │
 │                                  │ <──────────────────────────────│
 │                                  │ 5. actualiza notification_log  │
 │                                  │    estado = ENVIADO | FALLIDO  │
```

### 5.2 Por qué Outbox y no envío síncrono

| Envío síncrono (malo) | Outbox (bueno) |
|-----------------------|----------------|
| El cobro espera a WhatsApp | El cobro no espera a nadie |
| Si WhatsApp cae, el cobro falla | Si WhatsApp cae, el ticket ya se cobró |
| No hay reintento | Reintento automático |
| No hay bitácora | `notification_log` registra todo |

**Regla de Oro #7:** "No hay `try/except pass` en la ruta crítica. Outbox transaccional."

### 5.3 Idempotencia

Cada mensaje lleva `evento_id` (generado por el POS). La tabla
`notification_outbox` tiene `uq_notification_evento_canal (evento_id, canal) UNIQUE`.
Si el POS reintenta el encolado (por un timeout de red), el segundo intento **no duplica**
el mensaje. Es el mismo patrón que `uq_movimiento_evento_item` en Almacenes
([`MODELO_DE_DATOS_DEL_NUEVO_POS.md`](./MODELO_DE_DATOS_DEL_NUEVO_POS.md:391)).

### 5.4 Las plantillas de mensaje

| Canal | Plantilla | Variables |
|-------|-----------|-----------|
| **WhatsApp** | Texto plano con formato WhatsApp (`*negrita*`) | folio, fecha, total, items, negocio |
| **E-mail** | HTML (reutiliza el HTML de [`ticketGenerator.js`](../../apps/pos/utils/ticketGenerator.js)) | Las mismas + logo |

**Precedente reutilizable:** [`buildMatrixWhatsAppText()`](../../apps/pos/GrandezaOrderRequestsTab.jsx:199)
ya construye texto de WhatsApp para reparto. Se reutiliza el criterio de formato, no el código.

### 5.5 El destinatario

El POS **no** lee `customers`. Pide el contacto por contrato:

```
CONTRATO clientes.contacto_para_notificar
  Consumidor:   Notificaciones (y el POS, para mostrar el dato en pantalla)
  Proveedor:    Clientes (CRM)
  Contrato nuevo:  Entrada: { customer_id } o { telefono }
                   Salida:  { nombre, telefono, email }
  Garantías:    - Devuelve solo los campos de contacto, nunca la ficha completa.
                - Si el cliente no tiene correo, `email: null` (no es error).
  Errores:      - 404 si el cliente no existe.
```

**Regla dura:** el POS **muestra** el teléfono/correo en pantalla (para que el cajero
confirme), pero **no lo guarda** en `tickets`. El ticket guarda `customer_id`; el contacto
vive en el CRM. Esto evita la doble fuente de verdad.

---

## SECCIÓN 6 — IMPACTO EN LOS DOCUMENTOS EXISTENTES

Este documento **propone cambios** a 4 documentos maestros. Ninguno se modifica hasta que
el dueño apruebe esta propuesta.

| Documento | Cambio propuesto |
|-----------|------------------|
| [`README.md`](./README.md) | Añadir el Documento 13 al índice (pasa de 13 a 14 documentos maestros). |
| [`CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md`](./CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md) | Extender la matriz de 17 a 19 contratos. Añadir las SECCIONES de los contratos 18 y 19. |
| [`MODELO_DE_DATOS_DEL_NUEVO_POS.md`](./MODELO_DE_DATOS_DEL_NUEVO_POS.md) | Añadir **8 tablas nuevas** (5 del CRM + 3 de Notificaciones). Pasa de 17 a **25 tablas**. |
| [`ESPECIFICACION_DE_INTERFACES_POS.md`](./ESPECIFICACION_DE_INTERFACES_POS.md) | Añadir el botón "Cliente" al `POSHeader` (FICHA 01) y el paso de entrega de ticket al `CheckoutScreen` (FICHA 08). Pasa de 26 a 28 interfaces. |
| [`PLANO ARQUITECTONICO PARA EL NUEVO POS.md`](./PLANO%20ARQUITECTONICO%20PARA%20EL%20NUEVO%20POS.md) | Añadir Clientes y Notificaciones a la SECCIÓN 5 (módulos del ERP fuera del alcance del POS). |

### 6.1 Ubicación en las fases de construcción

| Fase | Qué se construye de estos módulos |
|------|-----------------------------------|
| **F0 — Andamiaje** | Nada. Los módulos no existen todavía. |
| **F1 — Cimiento de datos** | Las 8 tablas nuevas (5 CRM + 3 Notificaciones). |
| **F2 — Frontera** | Los contratos 18 y 19 + los 2 internos. |
| **F3 — Comportamiento** | Las reglas de lealtad, promociones y outbox + sus tests. |
| **F4 — Guardianes** | Verificar que el POS no lee `customers` ni `notification_outbox`. |
| **F5 — Superficie** | Las 2 pestañas del CRM + el paso de entrega del ticket. |
| **F6 — Consolidación** | El CRM y Notificaciones reportan al central (resúmenes, no tablas). |

**Nota sobre el orden.** Estos dos módulos **no bloquean** la construcción del POS. El POS
puede construirse completo (F0–F6) sin ellos y funcionar como hoy. El CRM y Notificaciones
son una **capa posterior** que se añade cuando el negocio esté listo para operar lealtad.

---

## SECCIÓN 7 — LAS REGLAS DE NEGOCIO NUEVAS

Se proponen **12 reglas de negocio** nuevas (RN-74 a RN-85), continuando la numeración del
[`PLANO ARQUITECTONICO PARA EL NUEVO POS.md`](./PLANO%20ARQUITECTONICO%20PARA%20EL%20NUEVO%20POS.md)
(que llega hasta **RN-73**, según su tabla de totales: `RN-01 a RN-73 | 73`). Las reglas
RN-80 a RN-85 se derivan de las **decisiones del dueño** (Sección 9).

> **Nota de corrección (autocrítica).** La primera versión de este documento numeró estas
> reglas como RN-82 a RN-93, asumiendo erróneamente que el plano llegaba hasta RN-81. El
> plano llega hasta **RN-73**. Se renumera a **RN-74 a RN-85** para que la secuencia sea
> continua y no deje huecos.

### 7.1 Reglas de identidad y flujo (RN-74 a RN-76)

| # | Regla | Por qué |
|---|-------|---------|
| **RN-74** | El ticket nace como "Público General". La identificación del cliente es **opcional** y **posterior** a la carga de ítems. | Respeta RN-08 (operaciones atómicas) y no obliga al cajero a un paso previo. |
| **RN-75** | El teléfono normalizado es la **clave** del cliente. El correo es opcional. | Un solo identificador fuerte; el correo solo se pide si se quiere e-mail. |
| **RN-76** | Los beneficios de **precio** se aplican como **líneas negativas** en el ticket, antes de PAID. Los beneficios **acumulables** se asientan en el ledger, después de PAID. | Separa lo que cambia el total de lo que no. |

### 7.2 Reglas del ledger de lealtad (RN-77, RN-80, RN-81)

| # | Regla | Por qué |
|---|-------|---------|
| **RN-77** | El saldo de puntos **nunca** se hace `UPDATE`. Se asienta en `loyalty_ledger`; `customer_benefits` es una proyección recalculable. | Regla de Oro #10 (ledger inmutable). |
| **RN-80** | **La política de lealtad es configurable desde el CRM y cambiable a voluntad.** El modo (puntos / niveles / ambos), la tasa de acumulación y la regla de expiración se **declaran** en la tabla `loyalty_policy`, no en el código. | Decisión del dueño (Q-1/Q-2). Permite cambiar la política sin redeployar. |
| **RN-81** | **Cada asiento del `loyalty_ledger` guarda la política vigente al momento del asiento** (versión de `loyalty_policy`). Un cambio de política **no** recalcula los puntos ya acumulados. | Evita que un cambio de política altere retroactivamente el saldo del cliente. |

### 7.3 Reglas del canje (RN-82, RN-83)

| # | Regla | Por qué |
|---|-------|---------|
| **RN-82** | **El canje de puntos requiere autorización de un gerente** (PIN de supervisor, contrato `seguridad.validar_pin` #8) **y es a petición del cliente**. El cajero no puede canjear por iniciativa propia. | Decisión del dueño (Q-4). Protege el saldo del cliente. |
| **RN-83** | **El POS avisa proactivamente al cajero cuando el cliente identificado tiene un beneficio canjeable disponible.** El aviso es en pantalla, no un envío. | Decisión del dueño (Q-4): "previo aviso del cajero de que tiene un beneficio disponible". |

### 7.4 Reglas de notificación (RN-78, RN-79, RN-84)

| # | Regla | Por qué |
|---|-------|---------|
| **RN-78** | El envío de ticket por WhatsApp/correo es **Outbox**: se encola en la transacción del ticket y se envía después. **Nunca** bloquea el cobro. | Regla de Oro #7 (outbox transaccional). |
| **RN-79** | El ticket impreso **siempre** está disponible y es el comportamiento por defecto. WhatsApp y e-mail son **adicionales**, nunca sustitutos. | CA-19 (ticket impreso idéntico al POS actual). |
| **RN-84** | **El cajero elige el canal de entrega cada vez, pero el sistema lo asiste:** si el cliente ya está identificado, el paso de entrega **precarga** el teléfono/correo desde el CRM y el cajero solo confirma o cambia el canal. **No se vuelve a teclear el contacto.** | Decisión del dueño (Q-3). Evita recaptura y errores. |

### 7.5 Regla del alta de cliente (RN-85)

| # | Regla | Por qué |
|---|-------|---------|
| **RN-85** | **El cajero puede dar de alta un cliente libremente**, pero el alta rápida **busca primero por teléfono normalizado**; si el cliente ya existe, ofrece "¿es este cliente?" en vez de crear un duplicado. | Decisión del dueño (Q-5) + salvaguarda contra duplicados. |

---

## SECCIÓN 8 — LO QUE ESTE DOCUMENTO **NO** HACE

Para evitar malentendidos, se declara explícitamente el alcance negativo:

1. **No modifica el ERP instalado.** Cero líneas de código de producción. Es una propuesta
   de plano, no una obra.
2. **No implementa nada.** Define contratos, tablas y flujos; la construcción es F1–F5.
3. **No decide las reglas de lealtad del negocio.** Define *cómo* se aplicarían; *cuántos*
   puntos vale cada peso, *qué* promociones existen y *cuándo* vencen lo decide el dueño.
4. **No elige proveedor de WhatsApp.** El contrato `notificaciones.encolar_ticket` es
   agnóstico: funciona con WhatsApp Business API, Twilio o cualquier otro. Eso se decide
   en `channel_config`.
5. **No toca el módulo de Reparto (Grandeza).** Su WhatsApp existente
   ([`GrandezaOrderRequestsTab.jsx`](../../apps/pos/GrandezaOrderRequestsTab.jsx:231)) es
   para reparto, no para tickets. Se reutiliza el *criterio*, no el código.

---

## SECCIÓN 9 — DECISIONES DEL DUEÑO (RESUELTAS)

Las 5 preguntas abiertas fueron **respondidas por el dueño del negocio**. Se documentan aquí
con su consecuencia arquitectónica.

| # | Pregunta | **Decisión del dueño** | Consecuencia en el plano |
|---|----------|------------------------|--------------------------|
| **Q-1** | ¿Lealtad por puntos o niveles? | **Configurable desde el CRM y cambiable a voluntad.** | Nueva tabla `loyalty_policy` que declara el modo (puntos / niveles / ambos). No es código: es dato. |
| **Q-2** | ¿Los puntos expiran? | **Configurable desde el CRM y cambiable a voluntad.** | La regla de expiración vive en `loyalty_policy`. El `loyalty_ledger` guarda la política vigente por asiento (RN-81). |
| **Q-3** | ¿Envío automático o elección del cajero? | **El cajero elige cada vez, pero el sistema lo asiste:** si el cliente ya está identificado, no se vuelve a capturar el teléfono/correo. | El paso de entrega **precarga** el contacto desde el CRM (contrato 18 ya lo trae). RN-84. |
| **Q-4** | ¿Canjear o solo acumular? | **Acumular siempre. Canjear con autorización del gerente y a petición del cliente, previo aviso del cajero de que tiene un beneficio disponible.** | El canje usa `seguridad.validar_pin` (#8). El POS avisa proactivamente del beneficio canjeable. RN-82 y RN-83. |
| **Q-5** | ¿Alta de cliente requiere supervisor? | **El cajero puede dar de alta libremente.** | El alta rápida busca duplicado por teléfono normalizado antes de crear. RN-85. |

### 9.1 Las dos piezas nuevas que introdujo la decisión Q-4

La respuesta a Q-4 añadió **dos elementos** que la propuesta original no contemplaba:

1. **Autorización de gerente para canjear.** El canje deja de ser una acción libre del
   cajero. Reutiliza el contrato **`seguridad.validar_pin` (#8)** que ya existe en la matriz
   del Documento 9. No se necesita un contrato nuevo: se reutiliza uno existente.

2. **Aviso proactivo del beneficio disponible.** Cuando el cliente identificado tiene un
   beneficio canjeable, el POS **avisa en pantalla** al cajero. Esto es una **notificación
   de UI**, no un envío por canal. No toca el módulo de Notificaciones.

### 9.2 La pieza nueva que introdujo la decisión Q-1/Q-2

La respuesta "configurable y cambiable a voluntad" convierte la política de lealtad en
**dato de configuración**. Consecuencia: la tabla `loyalty_policy` se suma a las 4 tablas
del CRM (Sección 1.3), que pasan de 4 a **5**. El total de tablas nuevas pasa de 7 a **8**
(5 CRM + 3 Notificaciones).

**Riesgo advertido y mitigado.** Cambiar la política a mitad de operación podría alterar
retroactivamente el saldo del cliente. Se mitiga con **RN-81**: cada asiento del ledger
guarda la versión de política vigente al momento del asiento. Un cambio de política aplica
**hacia adelante**, nunca hacia atrás.

---

## SECCIÓN 10 — CIERRE

**Lo que este documento deja claro:**

1. La necesidad del negocio se resuelve con **dos módulos**, no uno: **Clientes (CRM)** y
   **Notificaciones**. Cada uno con sus tablas, sus contratos y su ciclo de vida.
2. El POS **sigue sin leer tablas ajenas**. Pide beneficios por
   `clientes.beneficios_para_ticket` y encola tickets por `notificaciones.encolar_ticket`.
3. La matriz de contratos pasa de **17 a 19**. Los dos nuevos son **Outbox** y **no
   bloquean la venta**.
4. La identificación del cliente es **opcional y posterior** a la carga de ítems, respetando
   RN-08 y RN-10.
5. Las promociones se configuran en el **módulo Clientes**, no en Vista General ni en el POS.
6. El envío por WhatsApp/e-mail **nunca** bloquea el cobro (patrón Outbox, Regla de Oro #7).
7. La **política de lealtad es dato, no código**: se configura desde el CRM y se cambia a
   voluntad (RN-80), sin tocar el POS ni desplegar código nuevo.
8. El **canje de beneficios requiere autorización de gerente** (reutiliza
   `seguridad.validar_pin` #8) y es **a petición del cliente**, previo aviso proactivo del
   cajero (RN-82 y RN-83).
9. El **alta de cliente es libre para el cajero**, con búsqueda de duplicado por teléfono
   normalizado antes de crear (RN-85).
10. El **contacto se captura una sola vez**: si el cliente ya está identificado, el paso de
    entrega precarga teléfono/correo desde el CRM y no vuelve a pedirlos (RN-84).

### 10.1 Estado de la propuesta

| Aspecto | Estado |
|---------|--------|
| Preguntas abiertas (Sección 9) | **Resueltas por el dueño** (Q-1 a Q-5). |
| Reglas de negocio nuevas | **12** (RN-74 a RN-85), en la Sección 7. |
| Tablas nuevas | **8** (5 CRM + 3 Notificaciones). |
| Contratos nuevos | **2** (#18 y #19); la matriz pasa de 17 a 19. |
| Documentos maestros a actualizar | **4** (Sección 6). |
| Código del ERP tocado | **Cero.** Regla Dura respetada. |

### 10.2 Lo que sigue

Si el dueño aprueba esta propuesta, el siguiente paso es:

1. **Incorporarla al índice del [`README.md`](./README.md)** como **Documento 13**.
2. **Propagar los cambios** a los 4 documentos afectados (Sección 6): `CONTRATOS` (17→19),
   `MODELO_DE_DATOS` (17→25 tablas), `ESPECIFICACION_DE_INTERFACES_POS` (26→28) y el
   `PLANO ARQUITECTONICO` (fases F0–F6).
3. **Escribir la especificación funcional** del módulo Clientes, igual que se hizo con
   Vista General (Documento 12), ahora que las 5 decisiones están resueltas.

---

**Fin del Documento 13 — Propuesta.**