# MODELO DE DESPLIEGUE Y CONSOLIDACIÓN CENTRAL — NUEVO POS
## Topología hub-and-spoke: ERP completo por sucursal + servidor central corporativo

> **Documento complementario** al [`PLANO ARQUITECTONICO PARA EL NUEVO POS.md`](PLANO ARQUITECTONICO PARA EL NUEVO POS.md:1)
> y al [`PLAN_ACCION_ARQUITECTONICO_NUEVO_POS.md`](PLAN_ACCION_ARQUITECTONICO_NUEVO_POS.md:1).
> El plano describe **qué debe ser** el nuevo POS. El plan de acción describe **qué acciones**
> incorporar. Este documento describe **dónde vive** y **cómo se consolida**.
> **Regla dura:** no se toca el ERP. Este documento es un artefacto de diseño.

---

### ⚠️ NOTA DE TRAZABILIDAD (autocrítica v1.1)

**La topología hub-and-spoke NO es una decisión nueva de este documento.** Ya está
establecida como **decisión de diseño inamovible** en
[`CONTEXTO_SISTEMA_IA.md`](ESPECIFICACIONES DEL PROYECTO/CONTEXTO_SISTEMA_IA.md:113) **§3.3
"ARQUITECTURA DE RESILIENCIA — DISEÑO HUB AND SPOKE"**, que es la **fuente autoritativa**.

Este documento **no descubre** la topología: la **expande operativamente**. La fuente
autoritativa ya fija:

- **§3.3.1 Topología General** → este documento la expande (Sección 0).
- **§3.3.2 Tres Niveles de Conectividad** → este documento la referencia (Sección 2.7).
- **§3.3.3 Sync al cierre del día (default 23:30)** → este documento la respeta (Sección 2.3).
- **§3.3.4 UUID v4 como PK; folio local** → este documento la extiende al central (Sección 1).
- **§3.3.5 Resolución de conflictos** → este documento la referencia (Sección 2.8).
- **§3.3.6 Infraestructura del servidor local** → este documento la referencia (Sección 4).

**Lo que este documento SÍ aporta como nuevo** (no está en la fuente autoritativa):
el **contrato de consolidación** (Sección 2: principios P1-P6, payload JSON, outbox de
consolidación), el **modelo de datos del central** (Sección 3.4) y los **criterios de
aceptación** (Sección 5: S1-S7, H1-H6).

---

## SECCIÓN 0 — LA ACLARACIÓN QUE REFRAMEÓ LA ARQUITECTURA

### 0.1 Lo que se creía (y quedó descartado)

Los documentos previos dejaron abierta la pregunta de la operación multisucursal. Se contemplaba
una de dos formas:

- **Multisucursal con BD compartida** (multi-tenant): un solo ERP, una sola base de datos,
  un campo `branch_id` en cada tabla, y el POS filtrando por sucursal.
- **Multisucursal con BD por sucursal pero operación unificada**: varias bases, pero un solo
  sistema lógico que las ve como una sola operación.

**Ambas quedaron descartadas.**

### 0.2 Lo que se hará (la decisión del negocio)

> **No habrá una operación multisucursal.**
> **Lo que se hará es instalar un ERP (con el POS incluido como uno de sus módulos) en cada
> sucursal, y cada ERP a su vez enviará data a un servidor central del corporativo.**

Esto es una topología **hub-and-spoke** (concentrador y radios), también llamada
**consolidación central** o **data warehousing operativo**:

```
                    ┌──────────────────────────────────┐
                    │   SERVIDOR CENTRAL CORPORATIVO   │
                    │   (consolidación / reportería)   │
                    │   NO transaccional               │
                    └──────────────────────────────────┘
                          ▲          ▲          ▲
                          │          │          │
              ┌───────────┘          │          └───────────┐
              │                      │                      │
    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
    │  SUCURSAL A     │    │  SUCURSAL B     │    │  SUCURSAL C     │
    │  ERP COMPLETO   │    │  ERP COMPLETO   │    │  ERP COMPLETO   │
    │  ├─ POS         │    │  ├─ POS         │    │  ├─ POS         │
    │  ├─ Almacenes   │    │  ├─ Almacenes   │    │  ├─ Almacenes   │
    │  ├─ Caja        │    │  ├─ Caja        │    │  ├─ Caja        │
    │  ├─ Pedidos     │    │  ├─ Pedidos     │    │  ├─ Pedidos     │
    │  └─ ...         │    │  └─ ...         │    │  └─ ...         │
    │  BD PROPIA      │    │  BD PROPIA      │    │  BD PROPIA      │
    └─────────────────┘    └─────────────────┘    └─────────────────┘
```

### 0.3 Las 5 consecuencias arquitectónicas

| # | Consecuencia | Efecto en el diseño |
|---|--------------|---------------------|
| **C1** | Cada sucursal es un **ERP completo e independiente** | El POS es un **módulo** dentro de ese ERP, no un sistema aparte. La frontera POS↔ERP se define **una vez** y se replica idéntica en cada sucursal. |
| **C2** | Cada sucursal tiene **su propia base de datos** | No hay `branch_id` en las tablas transaccionales. La BD local es la **fuente de verdad** de la operación de esa sucursal. |
| **C3** | El **folio `V####` local es correcto por diseño** | Cada sucursal numera sus tickets desde 1. No es un bug: es la consecuencia de la independencia. Pero el central **necesita** distinguir el origen. |
| **C4** | El **servidor central NO es transaccional** | No cobra, no vende, no abre caja. Solo **recibe, consolida y reporta**. Si el central cae, las sucursales siguen operando. |
| **C5** | La **replicación es de datos, no de esquema** | Se replica el **mismo ERP** (mismo código, misma versión) en cada sucursal. Lo que viaja al central es **data consolidada**, no la BD completa. |

### 0.4 Lo que esto resuelve de las decisiones pendientes

En el análisis previo quedaron 3 decisiones abiertas. Esta aclaración resuelve dos:

| Decisión pendiente | Estado | Resolución |
|--------------------|--------|------------|
| (a) ¿BD compartida o propia? | **RESUELTA** | **Propia.** Cada sucursal tiene su BD. No hay multi-tenant. |
| (b) ¿Reemplazar el POS actual o solo sucursales nuevas? | **RESUELTA** | Cada sucursal recibe un **ERP completo** (con POS como módulo). El POS actual se reemplaza por el módulo POS del nuevo ERP, sucursal por sucursal. |
| (c) ¿Qué es "terminado"? | **PENDIENTE** | Se define en la Sección 5 de este documento (criterios de aceptación). |

---

## SECCIÓN 1 — LA IDENTIDAD DE SUCURSAL (branch_id)

### 1.1 El problema que resuelve

Con C3 (folio local correcto por diseño), el servidor central recibiría de cada sucursal un
ticket `V0001`. Sin un identificador de origen, **tres sucursales producen tres `V0001`
indistinguibles**. El central no puede consolidar.

### 1.2 La solución: tres capas de identidad

| Capa | Identificador | Alcance | Quién lo asigna | Para qué sirve |
|------|---------------|---------|-----------------|----------------|
| **Global** | `UUID v4` | Mundial | El ERP de la sucursal al crear el registro | Identidad única e irrepetible. Es la **clave real**. |
| **Sucursal** | `branch_id` | Corporativo | El corporativo al instalar el ERP | Distinguir el origen. Es **configuración**, no código. |
| **Presentación** | `folio` (`V####`) | Local | El ERP de la sucursal | Mostrar al cliente y al cajero. Es **cosmético**. |

### 1.3 La regla de oro de la identidad

> **Ninguna regla de negocio usa el folio como identidad.**
> **Ninguna consolidación usa el folio como clave.**
> **El folio es para humanos; el UUID es para máquinas.**

Esto ya está establecido en **A-05** del plan de acción. Este documento lo **extiende al central**:

- La clave de consolidación es `(branch_id, uuid)` — nunca `(branch_id, folio)`.
- El folio viaja al central **solo como dato descriptivo** (para que un reporte muestre
  "Ticket V0042 de la Sucursal Centro").
- Si dos sucursales tienen el mismo folio, **no colisionan** porque el UUID es distinto.

### 1.4 Dónde vive el `branch_id`

| Ubicación | Cómo se define | Cuándo se lee |
|-----------|----------------|---------------|
| **Configuración del ERP** | Variable de entorno / archivo de config de la instalación | Al arrancar el ERP |
| **Metadatos de cada registro enviado al central** | Se adjunta en el momento de la exportación | En cada envío al central |
| **Tablas transaccionales locales** | **NO se almacena** (la BD local es de una sola sucursal) | — |

**Principio:** el `branch_id` es **configuración de despliegue**, no un dato de negocio.
La BD local no lo necesita porque toda ella pertenece a una sola sucursal. Solo se adjunta
**al salir** hacia el central.

### 1.5 El `branch_id` no es un `tenant_id`

| Aspecto | `tenant_id` (multi-tenant) | `branch_id` (hub-and-spoke) |
|---------|---------------------------|----------------------------|
| Dónde vive | En **cada fila** de cada tabla | En la **configuración** del ERP |
| Quién filtra | Cada consulta filtra por él | Nadie: la BD es de una sucursal |
| Si falta | Fuga de datos entre inquilinos | El envío al central no se puede enrutar |
| Riesgo | Alto (seguridad) | Bajo (solo enrutamiento) |

**Conclusión:** el `branch_id` es mucho más barato y seguro que un `tenant_id`, porque
**no contamina el modelo de datos transaccional**. Es la ventaja principal de esta topología.

---

## SECCIÓN 2 — EL CONTRATO DE CONSOLIDACIÓN

### 2.1 Qué es

El **contrato de consolidación** es el acuerdo entre cada ERP de sucursal (spoke) y el
servidor central (hub) sobre **qué datos viajan, en qué formato, con qué frecuencia y con
qué garantías**.

Es un contrato **unidireccional**: la sucursal **envía**, el central **recibe**. El central
**nunca** escribe en la sucursal.

### 2.2 Principios del contrato

| # | Principio | Implicación |
|---|-----------|-------------|
| **P1** | **Unidireccional** | Sucursal → Central. Nunca al revés. |
| **P2** | **Asíncrono** | La sucursal no espera al central para operar. Si el central cae, la sucursal sigue. |
| **P3** | **Idempotente** | Reenviar el mismo dato no duplica. La clave `(branch_id, uuid)` lo garantiza. |
| **P4** | **Reanudable** | Si el envío falla, se reintenta desde el último punto confirmado. |
| **P5** | **Auditable** | Todo envío deja rastro: qué se envió, cuándo, con qué resultado. |
| **P6** | **Sin acoplamiento de esquema** | El central no lee la BD de la sucursal. Recibe un **payload** definido. |

### 2.3 Qué se consolida (los 4 dominios)

No todo se envía. Se envía lo que el corporativo necesita para **decidir**:

| Dominio | Qué viaja | Frecuencia | Por qué |
|---------|-----------|-----------|---------|
| **Ventas** | Tickets cerrados (PAID), con líneas, totales, forma de pago, terminal, cajero, timestamps UTC | **Al cierre del día** (default `23:30`, configurable en `SystemSetting`) | Es el dato que el corporativo más consulta |
| **Caja** | Sesiones cerradas, movimientos, diferencias de arqueo | **Al cierre del día** (junto con ventas) | Control de efectivo y detección de faltantes |
| **Inventario** | Movimientos de inventario (ledger), no el stock | **Al cierre del día** (junto con ventas) | El stock se **recalcula** en el central; el ledger es la verdad |
| **Catálogo** | Altas/bajas/cambios de productos y precios | **Al cierre del día** (o manual desde el panel) | El corporativo necesita ver el catálogo vigente por sucursal |

> **⚠️ ALINEACIÓN CON LA FUENTE AUTORITATIVA (autocrítica v1.1):**
> La frecuencia **no es una sugerencia de este documento**. [`CONTEXTO_SISTEMA_IA.md`](ESPECIFICACIONES DEL PROYECTO/CONTEXTO_SISTEMA_IA.md:179)
> **§3.3.3** establece: *"Frecuencia: Automática al cierre del día. Hora configurable en
> `SystemSetting` (default: `23:30`). También puede lanzarse manualmente desde el panel de
> administración."* Este documento **respeta** ese default. La versión 1.0 de este documento
> sugería "5–15 min", lo cual **contradecía** la fuente autoritativa y quedó corregido.
>
> **Manejo de fallos (fuente autoritativa §3.3.3):** *"Si la sync falla, la operación del día
> siguiente no se ve afectada. Los datos se acumulan y se envían en la próxima sync exitosa."*
> Esto es exactamente el principio P2 (asíncrono) y P4 (reanudable) de la Sección 2.2.

**Lo que NO se consolida (en esta etapa):**

- Sesiones de terminal y candados (son efímeros, locales).
- Borradores (DRAFT) y tickets abiertos (aún no son venta).
- Datos de visión (imágenes de entrenamiento, predicciones).
- Configuración de hardware (impresoras, cámaras).

### 2.4 El formato del payload

El payload es **JSON versionado**, con un sobre común y un cuerpo por dominio:

```json
{
  "schema_version": "1.0",
  "branch_id": "SUC-CENTRO-01",
  "erp_version": "1.0.0",
  "sent_at_utc": "2026-09-21T21:43:39Z",
  "batch_id": "b7e2c1a0-...",
  "domain": "ventas",
  "records": [
    {
      "uuid": "3f2a...",
      "folio": "V0042",
      "status": "PAID",
      "total": 245.50,
      "payment_method": "EFECTIVO",
      "terminal_id": "T1",
      "captured_by_id": 7,
      "created_at_utc": "2026-09-21T18:12:03Z",
      "closed_at_utc": "2026-09-21T18:14:22Z",
      "items": [
        { "sku": "PAN-001", "quantity": 3, "unit_price": 15.00, "subtotal": 45.00 }
      ]
    }
  ]
}
```

**Reglas del payload:**

1. **`schema_version`** permite evolucionar sin romper al central.
2. **`branch_id`** identifica el origen (Sección 1).
3. **`erp_version`** permite detectar sucursales desactualizadas.
4. **`sent_at_utc`** es el momento del envío (no del hecho).
5. **`batch_id`** permite idempotencia y reanudación.
6. **`uuid`** es la clave real; **`folio`** es descriptivo.
7. **Todos los timestamps en UTC** (RN-81 se respeta también en el central).

### 2.5 El mecanismo de envío: outbox de consolidación

El envío **no se hace en el request del usuario**. Se hace con el **mismo patrón de outbox**
que ya establece **A-04** para Almacenes:

```
[Transacción del ticket]
  ├─ INSERT ticket
  ├─ INSERT ticket_items
  └─ INSERT outbox_consolidacion (dominio='ventas', uuid=ticket.uuid, payload=...)
       └─ COMMIT  ← el ticket y su intención de envío son atómicos

[Job dedicado de consolidación]  (fuera del request)
  ├─ SELECT outbox_consolidacion WHERE enviado=false
  ├─ POST al central (con reintentos y backoff)
  ├─ Si OK → UPDATE enviado=true, enviado_at_utc=...
  └─ Si falla → deja enviado=false; el próximo ciclo reintenta
```

**Ventajas:**

- El cajero **nunca espera** al central.
- Si el central está caído, los datos **se acumulan localmente** y se envían al volver.
- El fallo es **observable** (registros `enviado=false` con antigüedad), no silencioso.
- **No hay `try/except pass`** (la deuda DEUDA-04 no se replica).

### 2.6 Garantías del contrato

| Garantía | Cómo se logra |
|----------|---------------|
| **At-least-once** | El job reintenta hasta confirmar. |
| **Idempotencia** | El central deduplica por `(branch_id, uuid)`. |
| **Orden por registro** | Cada registro es independiente; no hay orden global requerido. |
| **Reanudación** | `batch_id` + `enviado=false` permiten retomar. |
| **Trazabilidad** | Cada envío deja `enviado_at_utc` y `response_code`. |

### 2.7 Los 3 niveles de conectividad (fuente autoritativa §3.3.2)

El contrato de consolidación es la **capa superior** de una arquitectura de resiliencia de
3 niveles que ya está definida en [`CONTEXTO_SISTEMA_IA.md`](ESPECIFICACIONES DEL PROYECTO/CONTEXTO_SISTEMA_IA.md:153)
**§3.3.2**. Este documento **no la redefine**: la referencia.

| Nivel | Condición | Comportamiento | Indicador UI |
|-------|-----------|----------------|--------------|
| **Nivel 1 — Normal** | Terminal conectada por **cable Ethernet (LAN)** al servidor local | Todas las operaciones en tiempo real contra PostgreSQL local | `● Conectado` (verde) |
| **Nivel 2 — Degradado** | El servidor local está caído o el cable se desconectó | Opera 100% desde **IndexedDB** local; cada operación va a una **cola de sincronización** con UUID y timestamp | `● Offline — N operaciones pendientes` (amarillo) |
| **Nivel 3 — Tablets de reparto** | Tablet en ruta, fuera de la red | Opera 100% offline; descarga su paquete de trabajo al salir y sincroniza al volver al WiFi | `🚚 En Ruta` con contador |

**Relación con la consolidación:** el Nivel 2 y el Nivel 3 son **locales** (sucursal ↔ terminal).
La consolidación (sucursal → central) es una **cuarta capa**, asíncrona y de cierre de día.
**No se confunden:** la cola de sincronización del Nivel 2 es hacia el servidor local; el outbox
de consolidación de la Sección 2.5 es hacia el central.

### 2.8 La resolución de conflictos (fuente autoritativa §3.3.5)

La fuente autoritativa establece la regla de propiedad de datos:

> *"El servidor de sucursal gana sobre su propio dominio."*

| Dato | Propietario | Implicación para la consolidación |
|------|-------------|-----------------------------------|
| **Ventas, producción, inventario** | La **sucursal** | El central **no los modifica retroactivamente**. Solo los consolida. |
| **Catálogo, precios, configuración global** | El **corporativo** | Las sucursales los **reciben**, no los originan. (Nota: en esta etapa el catálogo viaja sucursal → central como dato descriptivo; la propiedad corporativa se materializa en la Etapa 6.) |
| **Conflicto genuino** | — | Se registra en la tabla `sync_conflictos` para **revisión manual**. **Nunca** se resuelve automáticamente con lógica silenciosa. |

**Principio heredado:** *"Un dato perdido sin aviso es peor que un conflicto visible."*
Esto refuerza P5 (auditable) de la Sección 2.2 y prohíbe el `try/except pass` (DEUDA-04).

---

## SECCIÓN 3 — EL SERVIDOR CENTRAL (EL HUB)

### 3.1 Qué es y qué no es

| El central **ES** | El central **NO ES** |
|-------------------|----------------------|
| Un **repositorio de datos consolidados** | Un sistema transaccional |
| Un **motor de reportería** corporativa | Un punto de cobro |
| Un **comparador** entre sucursales | Un ERP |
| Un **detector de anomalías** (faltantes, desviaciones) | Un sistema del que dependa la operación |
| Un **archivo histórico** | Un backup de las sucursales |

### 3.2 El principio de no-dependencia

> **Si el servidor central se apaga, las sucursales siguen vendiendo, cobrando y operando
> con normalidad.**

Esto es **no negociable**. El central es un **consumidor pasivo**. Su caída degrada la
**visibilidad corporativa**, no la **operación**. Esto se logra porque:

1. El envío es asíncrono (P2).
2. La sucursal acumula localmente si el central no responde.
3. Ninguna operación de la sucursal **espera** una respuesta del central.

### 3.3 Qué hace el central con los datos

| Función | Descripción |
|---------|-------------|
| **Consolidar** | Unificar las ventas de todas las sucursales en una vista única. |
| **Comparar** | Ranking de sucursales, productos más vendidos por región, etc. |
| **Detectar anomalías** | Diferencias de arqueo recurrentes, caídas de venta, precios desalineados. |
| **Reportar** | Reportes corporativos (diarios, semanales, mensuales). |
| **Auditar** | Verificar que cada sucursal respeta las reglas (p. ej., precios del catálogo). |

### 3.4 El modelo de datos del central (conceptual)

El central **no replica** el modelo de la sucursal. Tiene su **propio modelo**, optimizado
para consulta:

| Entidad central | Origen | Clave |
|-----------------|--------|-------|
| `VentaConsolidada` | Tickets PAID de todas las sucursales | `(branch_id, uuid)` |
| `LineaVentaConsolidada` | Líneas de esos tickets | `(branch_id, uuid, sku)` |
| `SesionCajaConsolidada` | Sesiones cerradas | `(branch_id, uuid)` |
| `MovimientoInventarioConsolidado` | Ledger de inventario | `(branch_id, uuid)` |
| `CatalogoConsolidado` | Catálogo vigente por sucursal | `(branch_id, sku)` |
| `Sucursal` | Registro de sucursales | `branch_id` |
| `EnvioConsolidacion` | Bitácora de envíos recibidos | `batch_id` |

**Nota:** el central **recalcula** el stock a partir del ledger; no lo recibe calculado.
Esto evita que un error de cálculo local se propague al corporativo.

---

## SECCIÓN 4 — EL DESPLIEGUE POR SUCURSAL

### 4.1 Qué se replica y qué se configura

| Se **replica** (idéntico en todas) | Se **configura** (distinto por sucursal) |
|------------------------------------|------------------------------------------|
| El código del ERP (misma versión) | `branch_id` |
| El esquema de la BD | Nombre/dirección de la sucursal |
| Las 81 reglas de negocio | Credenciales de conexión al central |
| Los contratos entre módulos | Terminales y su configuración |
| Los tests (incluidos los guardianes) | Catálogo inicial y precios locales |
| El outbox de consolidación | Zona horaria (si difiere) |

**Principio:** **el código es uno; la configuración es muchas.** Esto es lo que hace
sostenible la replicación: un solo artefacto, N configuraciones.

### 4.2 El proceso de instalación de una sucursal

| Paso | Acción | Verificación |
|------|--------|--------------|
| 1 | Instalar el ERP (misma versión que las demás) | `erp_version` coincide |
| 2 | Configurar `branch_id` y datos de la sucursal | El ERP arranca con el `branch_id` correcto |
| 3 | Configurar la conexión al central | El job de consolidación puede autenticarse |
| 4 | Cargar el catálogo inicial | Los productos aparecen en el POS |
| 5 | Configurar terminales | Las terminales se ven en el selector |
| 6 | Prueba de humo: venta de prueba | El ticket llega al central con el `branch_id` correcto |
| 7 | Capacitar al personal | — |

### 4.3 La versión del ERP como contrato

Cada sucursal reporta su `erp_version` en cada envío. Esto permite:

- **Detectar desactualización:** si una sucursal reporta `1.0.0` y el resto `1.1.0`, el
  corporativo lo ve.
- **Planificar actualizaciones:** se actualiza sucursal por sucursal, no todas a la vez.
- **Compatibilidad de payload:** el `schema_version` permite que un central nuevo entienda
  a una sucursal vieja (y viceversa).

### 4.4 El orden de despliegue (sucursal piloto primero)

| Fase | Alcance | Objetivo |
|------|---------|----------|
| **F1** | 1 sucursal piloto | Validar el ERP completo + la consolidación en condiciones reales |
| **F2** | 2–3 sucursales | Validar que la consolidación escala y que las sucursales son independientes |
| **F3** | Resto de sucursales | Despliegue masivo con el proceso ya probado |

**Regla:** ninguna sucursal se despliega sin que la anterior haya cerrado su ciclo completo
(operación + consolidación verificada).

---

## SECCIÓN 5 — CRITERIOS DE ACEPTACIÓN (QUÉ ES "TERMINADO")

### 5.1 Criterios de una sucursal

| # | Criterio | Cómo se verifica |
|---|----------|------------------|
| **S1** | El ERP completo opera con el POS como módulo | Venta real de punta a punta |
| **S2** | Las 81 reglas se cumplen | Suite de tests en verde (incluidos guardianes) |
| **S3** | El POS no lee tablas ajenas | Test de arquitectura en verde |
| **S4** | No hay `try/except pass` en la ruta crítica | Búsqueda automatizada en CI |
| **S5** | El folio es local y el UUID es global | Revisión de la matriz de trazabilidad |
| **S6** | La sucursal opera con el central caído | Prueba de desconexión del central |
| **S7** | Los datos llegan al central con `branch_id` correcto | Consulta en el central |

### 5.2 Criterios del central

| # | Criterio | Cómo se verifica |
|---|----------|------------------|
| **H1** | Recibe datos de N sucursales sin colisión | Dos sucursales con el mismo folio coexisten |
| **H2** | Es idempotente | Reenviar un batch no duplica |
| **H3** | Es reanudable | Un envío interrumpido se retoma |
| **H4** | No es transaccional | El central no tiene endpoints de venta/cobro |
| **H5** | Consolida y reporta | Reporte corporativo con datos de todas las sucursales |
| **H6** | Detecta sucursales desactualizadas | Alerta si `erp_version` difiere |

### 5.3 Criterio global (el que cierra el proyecto)

> **Una sucursal puede operar un día completo, con el central apagado, y al reconectarse
> el central recibe todos sus datos sin pérdida ni duplicación.**

Este es el criterio que demuestra que la topología hub-and-spoke funciona: **independencia
operativa + consolidación garantizada**.

---

## SECCIÓN 6 — LO QUE ESTE MODELO **NO** RESUELVE (Y QUEDA PENDIENTE)

| Pendiente | Etapa | Descripción |
|-----------|-------|-------------|
| **Modelo de datos del nuevo POS** | Etapa 2 | Entidades, relaciones, invariantes (con UUID y folio separados) |
| **Esquemas de los contratos** | Etapa 4 | Entrada/salida/errores de los 10 contratos entre módulos |
| **Esquema detallado del payload de consolidación** | Etapa 4 | Versión 1.0 completa, con validación |
| **Estructura de módulos del ERP** | Etapa 3 | Cómo se organiza el ERP con el POS como módulo |
| **Plan de construcción** | Etapa 5 | Orden de implementación |
| **Tecnología del central** | Etapa 6 | Motor de reportería, almacenamiento, autenticación |
| **Seguridad del canal** | Etapa 6 | Autenticación sucursal↔central, cifrado en tránsito |
| **Retención y purga** | Etapa 6 | Cuánto tiempo guarda el central; qué se purga localmente |

---

## SECCIÓN 7 — DECLARACIÓN DE LA REGLA DURA

> **NO SE TOCA EL ERP INSTALADO Y CORRIENDO.**
> **NO SE TOCA NINGUNO DE SUS MÓDULOS.**
> **NO SE MODIFICA NI UNA LÍNEA DEL POS ACTUAL.**

Este documento es un artefacto de diseño derivado de la ingeniería inversa del POS actual
(commit `fe9f6ed`, tag `v22-estable-fe9f6ed`) y de la aclaración del negocio sobre la
topología de despliegue. No contiene código de producción. El ERP permanece intacto y
operando; su HEAD es `5802f45` (V23) al 22 Sep 2026.

---

*Documento complementario al plano fundacional y al plan de acción. Versión 1.1.
Anclado al commit `5802f45` (V23); ingeniería inversa sobre `fe9f6ed`. Topología: hub-and-spoke
(ERP completo por sucursal + servidor central corporativo de consolidación).*
