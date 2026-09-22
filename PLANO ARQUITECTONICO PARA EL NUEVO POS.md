# PLANO ARQUITECTÓNICO PARA EL NUEVO POS

> **Documento fundacional del proyecto `PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS`.**
> Este documento NO describe el POS actual. Describe el POS que **debería existir**,
> diseñado desde un plano, para ser replicado en futuras sucursales.

---

## REGLA DURA (INVIOLABLE)

> **NO SE TOCA EL ERP INSTALADO Y CORRIENDO.**
> **NO SE TOCA NINGUNO DE SUS MÓDULOS.**
> **NO SE MODIFICA NI UNA LÍNEA DEL POS ACTUAL.**

Este es un **trabajo alterno**, en un **proyecto nuevo**, en un **repositorio nuevo**:
`https://github.com/vikutasan/PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS`

El ERP actual (`ERP-R-DE-RICO-CON-POS-SIMPLIFICADO`) permanece **intacto y operando**.
Este repositorio contiene **solo documentos**: planos, especificaciones, contratos y
decisiones de arquitectura. **Cero código de producción.** Cero dependencias. Cero
riesgo para el negocio en marcha.

**Corolario:** el POS actual es la **fuente de verdad funcional** (lo que hace), pero
**no** es la fuente de verdad estructural (cómo está construido). Copiamos su
*comportamiento*, no su *deuda*.

---

## SECCIÓN 0 — OPINIÓN SOBRE EL ENFOQUE

### 0.1 Veredicto: **el enfoque es correcto, y el momento es el correcto**

La analogía del edificio es exacta. El POS actual es un edificio funcional al que se le
fueron añadiendo habitaciones, tuberías e instalaciones sobre la marcha. Funciona. Pero
cada nueva habitación tuvo que conectarse a las tuberías que ya existían, y esas tuberías
no fueron diseñadas para esa carga.

**Tres razones por las que este enfoque es el correcto:**

**1. El costo de replicar la deuda es multiplicativo, no aditivo.**
Si replicamos el POS actual tal cual en 5 sucursales, no tendremos 5 POS: tendremos
**5 copias de la misma deuda**, y cada una divergirá. En 2 años, un bug arreglado en la
sucursal A no estará en la B. Un plano compartido convierte 5 mantenimientos en 1.

**2. El POS actual ya nos dio el regalo más caro: los requisitos.**
El activo más valioso del POS actual no es su código — es que **ya sabemos qué hace
falta**. Sabemos que existe el DRAFT GUARD, que existe el bloqueo optimista, que existe
el reciclaje de folios, que existe el outbox a Almacenes. Esa lista de reglas de negocio
tardó años en descubrirse. Un arquitecto que llega hoy **no tiene que adivinar los
requisitos**: los puede leer del edificio existente. Eso es una ventaja enorme.

**3. Extraer la lógica de negocio primero es el orden correcto.**
No se puede diseñar la cimentación sin saber cuánto peso va a soportar. La lógica de
negocio **es** ese peso. Extraerla primero —sin mirar el código de interfaz— nos da el
"programa arquitectónico" (qué necesita el edificio) antes de dibujar los planos
(cómo se construye).

### 0.2 Los tres riesgos de este enfoque (y cómo los mitigamos)

**Riesgo 1 — El plano se vuelve ficción.**
Un plano que describe un POS ideal que nunca se construye es un documento muerto.
*Mitigación:* cada regla de negocio extraída debe estar **trazada a su origen** en el
POS actual (archivo + línea). El plano es verificable contra la realidad.

**Riesgo 2 — El plano se contamina con la deuda.**
Es tentador copiar decisiones del POS actual que son *accidentes históricos*, no
*requisitos*. Ejemplo: el `db.expire_all()` de `add_item_to_ticket` no es una regla de
negocio, es un parche. Si lo copiamos al plano, replicamos el parche.
*Mitigación:* separar estrictamente **REGLAS DE NEGOCIO** (el *qué*) de **DECISIONES
TÉCNICAS** (el *cómo*). Este documento solo contiene lo primero.

**Riesgo 3 — El POS actual cambia mientras planificamos.**
El ERP en producción seguirá evolucionando. El plano podría quedar desactualizado.
*Mitigación:* el plano se ancla a un **commit congelado** del POS actual (la ingeniería
inversa se hizo sobre `fe9f6ed`, tag `v22-estable-fe9f6ed`; el ERP hoy corre en `5802f45`,
V23). Cualquier divergencia posterior es una decisión consciente, no un accidente.

### 0.3 Lo que este enfoque NO es

- **No es una reescritura del POS actual.** El POS actual sigue vivo y no se toca.
- **No es un refactor.** No hay "mover código". Hay "escribir un plano nuevo".
- **No es un rediseño de la interfaz.** Este documento no menciona UI, componentes,
  ni pantallas. Solo reglas de negocio.
- **No es un compromiso de migración.** Es un plano. La decisión de construir sobre él
  es posterior y separada.

### 0.4 Por qué "monolito modular con contratos" es la forma correcta

El POS no es un sistema aislado: **cobra** (necesita productos), **descuenta inventario**
(necesita almacenes), **registra quién hizo qué** (necesita auditoría), **reporta ventas**
(necesita estadísticas), **crea pedidos** (necesita producción/reparto).

Un monolito modular con **contratos explícitos** es la forma correcta porque:

- **Un solo despliegue** (una sucursal = una instancia, no 6 microservicios que
  coordinar). Crítico para una panadería, donde la simplicidad operativa vale más que
  la elegancia distribuida.
- **Fronteras claras** (cada módulo expone un contrato, no una tabla). Esto es lo que
  el POS actual **no** tiene: hoy el POS lee directamente `Product`, `Order`,
  `WarehouseEvent` y `Employee` de otros módulos. Eso es acoplamiento por base de datos.
- **Replicable** (el contrato es el mismo en todas las sucursales; solo cambia la
  configuración).

---

## SECCIÓN 1 — LOGICA DEL NEGOCIO

> **Esta sección describe QUÉ hace el módulo POS, sin mencionar código de interfaz.**
> Cada regla está trazada a su origen en el POS actual (commit `fe9f6ed`; ver §A.4 de la
> Especificación Funcional para los cambios posteriores que tocan al POS).
> Las reglas se numeran `RN-XX` (Regla de Negocio) para poder referenciarlas en los
> contratos de la Sección 2.

### 1.1 Contexto del negocio

El POS es el punto donde **el dinero entra al negocio**. Su responsabilidad es
transformar una **intención de compra** (un cliente quiere N productos) en un
**hecho contable inmutable** (un ticket pagado, con folio único, con trazabilidad de
quién lo capturó y quién lo cobró).

El POS opera en un entorno hostil:
- **Múltiples terminales concurrentes** (varias cajas cobrando al mismo tiempo).
- **Red inestable** (el navegador puede cerrarse en cualquier momento).
- **Personal rotativo** (el cajero que captura no siempre es el que cobra).
- **Dos canales de venta** (Panadería y Heladería) que comparten infraestructura.

### 1.2 Entidades del dominio (el "qué", no el "cómo")

| Entidad | Qué representa | Ciclo de vida |
|---|---|---|
| **Sesión de Terminal** | El turno de una caja física | Se abre, está activa, se cierra |
| **Ticket** | Una cuenta de venta | Nace DRAFT → OPEN → PAID (o CANCELLED) |
| **Ítem de Ticket** | Una línea de la cuenta | Se agrega, se modifica, se elimina |
| **Candado de Terminal** | El derecho exclusivo de un cajero sobre una caja | Se toma, se renueva, se libera, expira |
| **Evento de Almacén** | El aviso de que se vendió algo (para descontar inventario) | Se emite al pagar |
| **Pedido (Order)** | La proyección del ticket hacia producción/reparto | Se crea/actualiza al pagar o al enviar al pizarrón |

### 1.3 REGLAS DE NEGOCIO — Ciclo de vida del Ticket

**RN-01 — Todo ticket nace como DRAFT.**
Un ticket recién creado (al agregar el primer ítem o al reservar folio) es un
**borrador privado**. No es visible para otras terminales. Representa una intención
de compra aún no confirmada.
*Origen: `service.py:385` (add_item), `service.py:806` (reserve).*

**RN-02 — Un DRAFT solo se vuelve visible al enviarse al pizarrón (→ OPEN).**
El paso de DRAFT a OPEN es una **acción explícita del cajero**. Solo entonces la cuenta
aparece en el pizarrón compartido y puede ser recuperada por otra terminal.
*Origen: `service.py:539` (get_open_tickets filtra solo OPEN).*

**RN-03 — Un ticket PAID es inmutable.**
Una vez pagado, un ticket **no puede modificarse**: no se le agregan ítems, no se le
cambian cantidades, no se le eliminan ítems, no se le cambia el total. Cualquier intento
se rechaza con un error de negocio explícito.
*Origen: `service.py:105-109`, `service.py:391-392`, `service.py:455-456`, `service.py:501-502`.*

**RN-04 — Un ticket solo puede pagarse una vez.**
El estado PAID es terminal. No existe "re-cobro". Si el folio ya está pagado, el sistema
lo dice por su nombre de folio.
*Origen: `service.py:106-109`.*

**RN-05 — El folio es único y se asigna atómicamente.**
Dos terminales que pidan folio en el mismo instante **nunca** reciben el mismo número.
El folio tiene el formato `V####` (V + 4 dígitos, con ceros a la izquierda).
*Origen: `service.py:782-820` (secuencia PostgreSQL `ticket_folio_seq`).*

**RN-06 — El folio se genera por secuencia de base de datos, no por conteo.**
El folio no se calcula contando tickets existentes (eso sería una condición de carrera).
Se obtiene de una secuencia atómica del motor de base de datos. Si por alguna razón
hubiera una colisión, se reintenta hasta 3 veces avanzando la secuencia.
*Origen: `service.py:793-820`.*

**RN-07 — El folio se inicializa desde el ticket más alto existente.**
Al crear la secuencia por primera vez, arranca en (folio más alto + 1), para no
reutilizar folios históricos.
*Origen: `service.py:761-775`.*

### 1.4 REGLAS DE NEGOCIO — Captura de ítems (operaciones atómicas)

**RN-08 — Cada operación de ítem es atómica e inmediata.**
Agregar, modificar cantidad o eliminar un ítem se persiste **inmediatamente**, no al
final. Si el navegador se cierra a mitad de la venta, lo capturado hasta ese momento
está a salvo.
*Origen: `service.py:351-533` (las tres operaciones hacen commit propio).*

**RN-09 — Agregar un producto ya presente incrementa su cantidad.**
No se crean líneas duplicadas del mismo producto. Si el producto ya está en la cuenta,
se suma la cantidad y se recalcula el subtotal.
*Origen: `service.py:413-415`.*

**RN-10 — El total del ticket siempre se recalcula desde los ítems persistidos.**
El total nunca se "arrastra" ni se confía en el valor que envía el cliente. Se recalcula
sumando los subtotales reales de los ítems en la base de datos, después de cada operación.
*Origen: `service.py:429-433`, `service.py:478-482`, `service.py:523-527`.*

**RN-11 — El subtotal de un ítem es cantidad × precio unitario.**
El precio unitario se **congela** al momento de agregar el producto. Si el precio del
catálogo cambia después, el ítem ya capturado conserva su precio original.
*Origen: `service.py:415`, `service.py:417-423`, `service.py:474`.*

**RN-12 — No se puede vender un producto inexistente.**
Si el producto no existe en el catálogo, la operación se rechaza.
*Origen: `service.py:359-360`.*

**RN-13 — No se puede vender un producto inactivo.**
Si el producto existe pero está desactivado, la operación se rechaza. Un producto
inactivo no debe poder entrar a una cuenta nueva.
*Origen: `service.py:361-362`.*

**RN-14 — No se puede operar sobre un ticket inexistente.**
Modificar cantidad o eliminar un ítem de un ticket que no existe se rechaza.
*Origen: `service.py:453-454`, `service.py:499-500`.*

**RN-15 — No se puede modificar/eliminar un ítem que no está en el ticket.**
Si el producto no está en la cuenta, la operación se rechaza.
*Origen: `service.py:470-471`, `service.py:516-517`.*

**RN-16 — Toda operación de ítem requiere una sesión de terminal activa.**
Si la sesión de la terminal no existe o está inactiva, no se puede crear un ticket nuevo.
*Origen: `service.py:376-378`, `service.py:174-176`.*

### 1.5 REGLAS DE NEGOCIO — Concurrencia (bloqueo optimista)

**RN-17 — Toda cuenta tiene un número de versión.**
Cada ticket lleva un contador de versión que se incrementa en **cada** modificación.
*Origen: `service.py:434`, `service.py:483`, `service.py:528`, `service.py:168`.*

**RN-18 — El cliente debe declarar la versión sobre la que opera.**
Si el cliente envía una versión y no coincide con la versión actual del servidor, la
operación se rechaza con un conflicto. Esto evita que dos cajeros sobrescriban el trabajo
del otro.
*Origen: `service.py:395-400`, `service.py:459-460`, `service.py:505-506`, `service.py:129-135`.*

**RN-19 — No existe bypass de versión.**
Cualquier conflicto de versión es legítimo y se rechaza. El cliente debe recuperar la
versión fresca del servidor y reintentar. No hay "forzar".
*Origen: `service.py:126-128` (comentario explícito: "Sin bypass").*

**RN-20 — Toda operación sobre un ticket bloquea la fila.**
Antes de modificar un ticket, se bloquea su fila para evitar que dos operaciones
concurrentes se pisen.
*Origen: `service.py:100`, `service.py:369`, `service.py:450`, `service.py:496`.*

**RN-21 — El reciclaje de folios usa bloqueo con salto.**
Al buscar un ticket vacío para reciclar, se usa un bloqueo que **salta** las filas ya
bloqueadas por otra terminal, para que dos terminales nunca reciclen el mismo ticket.
*Origen: `service.py:668` (`with_for_update(skip_locked=True)`).*

### 1.6 REGLAS DE NEGOCIO — DRAFT GUARD (protección de borradores)

**RN-22 — Un DRAFT solo puede cobrarse desde la terminal que lo creó.**
Si una terminal intenta cobrar (→ PAID) un borrador creado por **otra** terminal, la
operación se rechaza. El borrador es privado de su terminal.
*Origen: `service.py:113-125`.*

**RN-23 — Para cobrar un borrador ajeno, primero hay que enviarlo al pizarrón.**
La otra terminal debe primero convertir el DRAFT en OPEN (acción explícita), y solo
entonces puede cobrarlo. El mensaje de error lo dice explícitamente.
*Origen: `service.py:121-125`.*

**RN-24 — La comparación de terminales es tolerante a mayúsculas y espacios.**
La identidad de la terminal se normaliza (mayúsculas, sin espacios) antes de comparar,
para evitar falsos rechazos por diferencias de formato.
*Origen: `service.py:118-119`.*

### 1.7 REGLAS DE NEGOCIO — Reciclaje de folios

**RN-25 — Un ticket vacío reciente puede reciclarse.**
Al reservar una cuenta, si existe un ticket vacío (sin ítems, total cero) creado hace
**menos de 5 minutos**, se reutiliza en vez de generar un folio nuevo.
*Origen: `service.py:634-642`, `service.py:659`.*

**RN-26 — El reciclaje solo aplica al mismo terminal y al mismo canal.**
Un ticket de Heladería **nunca** es reciclado por el POS de Panadería, y viceversa.
*Origen: `service.py:663-667`.*

**RN-27 — El reciclaje solo aplica a tickets DRAFT.**
Un ticket OPEN no se recicla (ya fue enviado al pizarrón, es visible).
*Origen: `service.py:664`.*

**RN-28 — El límite de 5 minutos evita colisiones con folios ya pagados.**
Tickets viejos se ignoran deliberadamente para no reutilizar un folio que ya pudo haber
sido impreso o cobrado.
*Origen: `service.py:652-653`.*

### 1.8 REGLAS DE NEGOCIO — Recolección de basura (GC)

**RN-29 — Los tickets vacíos huérfanos se eliminan tras 1 hora.**
Un ticket OPEN o DRAFT sin ítems y con total cero, creado hace más de 1 hora, se elimina.
*Origen: `service.py:692-705`.*

**RN-30 — Los borradores con ítems expiran según configuración (default 1 día).**
Un DRAFT con ítems que supera el TTL configurado (`pos_draft_ttl_days`, default 1 día)
se marca como **CANCELLED** (no se borra), para preservar trazabilidad.
*Origen: `service.py:707-735`.*

**RN-31 — El GC está limitado a una ejecución por minuto.**
Para no degradar el rendimiento en hora pico, la limpieza no se ejecuta más de una vez
por minuto, sin importar cuántas reservas ocurran.
*Origen: `service.py:687-689`.*

**RN-32 — El GC nunca interrumpe el POS.**
Si la limpieza falla por cualquier razón, se registra el error y el POS continúa.
*Origen: `service.py:745-746`.*

**RN-33 — El TTL de borradores es configurable.**
El negocio puede ajustar cuántos días vive un borrador antes de expirar.
*Origen: `service.py:710-715`.*

### 1.9 REGLAS DE NEGOCIO — Ocupación de terminales (candados)

**RN-34 — Una terminal solo puede ser ocupada por un cajero a la vez.**
Si la terminal ya está ocupada por otra persona, el intento de tomarla falla.
*Origen: `occupancy.py:56-62`.*

**RN-35 — El mismo cajero puede renovar su propio candado.**
Si el cajero que intenta tomar la terminal ya es el dueño, se renueva el candado (no falla).
*Origen: `occupancy.py:57-61`.*

**RN-36 — Los candados expiran por inactividad (TTL, default 15 minutos).**
Un candado cuyo timestamp supere el TTL se libera automáticamente. Esto evita que una
terminal quede bloqueada para siempre si el cajero se va sin cerrar sesión.
*Origen: `occupancy.py:20-29`, `occupancy.py:48`.*

**RN-37 — Solo el dueño puede liberar su candado.**
Un cajero no puede liberar el candado de otro cajero.
*Origen: `occupancy.py:83-88`.*

**RN-38 — Liberar una terminal libre es exitoso (idempotente).**
Si la terminal ya estaba libre, la operación se considera exitosa.
*Origen: `occupancy.py:89`.*

**RN-39 — Un administrador puede forzar la liberación.**
Existe una operación privilegiada para liberar una terminal ocupada, sin ser el dueño.
*Origen: `occupancy.py:92-100`.*

**RN-40 — El latido (heartbeat) renueva el candado.**
El cliente envía señales periódicas para mantener vivo su candado mientras trabaja.
*Origen: `occupancy.py:103-123`.*

**RN-41 — El latido también purga candados expirados de cualquier terminal.**
Aprovecha la señal periódica para limpiar candados muertos de otras terminales.
*Origen: `occupancy.py:108-109`.*

**RN-42 — Los candados son persistentes, no en memoria.**
Un reinicio del servidor **no** libera los candados. Esto elimina las "expulsiones
fantasma" (un cajero que pierde su terminal porque el servidor se reinició).
*Origen: `occupancy.py:1-8` (docstring explícito).*

### 1.10 REGLAS DE NEGOCIO — Guardado de emergencia (ZERO-LOSS)

**RN-43 — El guardado de emergencia nunca falla hacia el cliente.**
Cuando el navegador se cierra, envía un guardado de emergencia. Este endpoint **siempre**
responde éxito, para no bloquear el cierre del navegador.
*Origen: `router.py:441-443`, `router.py:495-497`.*

**RN-44 — Un guardado de emergencia sin folio o sin ítems se ignora.**
Si el payload está incompleto, no se intenta guardar nada.
*Origen: `router.py:453-455`.*

**RN-45 — El guardado de emergencia busca la sesión de la terminal indicada.**
No toma una sesión arbitraria: busca la sesión activa de **la terminal** que envía el
guardado.
*Origen: `router.py:459-468`.*

**RN-46 — Si no encuentra la sesión de la terminal, usa un respaldo.**
Para no perder el ticket de emergencia, si no hay sesión para esa terminal, usa cualquier
sesión activa. Es un compromiso consciente: mejor un ticket con terminal aproximada que
un ticket perdido.
*Origen: `router.py:470-477`.*

**RN-47 — El guardado de emergencia crea el ticket como OPEN.**
No como DRAFT: el guardado de emergencia es una acción explícita de "guardar esto",
así que la cuenta debe quedar visible.
*Origen: `router.py:488`.*

**RN-48 — El guardado de emergencia es idempotente.**
Si se envía dos veces el mismo folio, no se duplica la cuenta.
*Origen: comportamiento de `create_ticket` (upsert por `account_num`).*

### 1.11 REGLAS DE NEGOCIO — Puente hacia Almacenes (outbox)

**RN-49 — Al pagar un ticket, se emite un evento de almacén.**
El POS notifica al módulo de Almacenes que se vendieron ciertos productos, para que
descuente el inventario.
*Origen: `service.py:43-54`.*

**RN-50 — El evento de almacén se emite ANTES del commit, para garantizar atomicidad.**
Si el ticket se guarda, el evento también. No puede haber un ticket pagado sin su evento.
*Origen: `service.py:44` (comentario explícito).*

**RN-51 — El evento de almacén solo se emite para tickets PAID.**
Un ticket OPEN o DRAFT no descuenta inventario (aún no es una venta).
*Origen: `service.py:46`.*

**RN-52 — Un fallo en el puente de almacén NUNCA interrumpe el POS.**
Si el módulo de Almacenes falla, el POS sigue cobrando. El inventario es importante,
pero **cobrar es más importante**.
*Origen: `service.py:45`, `service.py:53-54`.*

**RN-53 — El evento de almacén lleva el SKU y la cantidad de cada producto.**
El contrato con Almacenes es: lista de `{sku, qty}`.
*Origen: `service.py:50`.*

> **NOTA CRÍTICA (deuda conocida D28):** en el POS actual, este puente está **roto**:
> el evento nunca se inserta porque el SKU no está cargado y el error se silencia.
> **El plano debe especificar el contrato correcto**, no replicar el bug. Ver §1.16.

### 1.12 REGLAS DE NEGOCIO — Puente hacia Producción (pedidos)

**RN-54 — Un ticket de tipo PEDIDO se proyecta como Order.**
Cuando un ticket es de tipo PEDIDO y está OPEN o PAID, se crea o actualiza un registro
en el módulo de Producción/Reparto.
*Origen: `service.py:58-60`, `service.py:306-344`.*

**RN-55 — El estado del pedido se deriva del estado del ticket.**
Ticket PAID → Order PAGADO. Ticket OPEN → Order TENTATIVO.
*Origen: `service.py:316`.*

**RN-56 — Un pedido sin tipo de entrega asume PICKUP.**
Si no se especifica, el pedido se asume para recoger en tienda.
*Origen: `service.py:321`, `service.py:334`.*

**RN-57 — Un pedido sin empaque asume PROPIO.**
Si no se especifica, el empaque es del negocio.
*Origen: `service.py:325`, `service.py:339`.*

**RN-58 — La proyección del pedido es idempotente.**
Si el pedido ya existe, se actualiza; si no, se crea. Nunca se duplica.
*Origen: `service.py:311-329`.*

**RN-59 — Un ticket de VENTA_DIRECTA no genera pedido.**
Solo los PEDIDO se proyectan a Producción.
*Origen: `service.py:59`.*

### 1.13 REGLAS DE NEGOCIO — Trazabilidad y auditoría

**RN-60 — Todo ticket registra quién lo capturó.**
El cajero que crea la cuenta queda registrado desde el primer instante.
*Origen: `service.py:186`, `service.py:383`, `service.py:436-437`, `service.py:639-641`.*

**RN-61 — Todo ticket pagado registra quién lo cobró.**
El cajero que cobra queda registrado, y puede ser **distinto** del que capturó.
*Origen: `service.py:158-159`, `service.py:187`.*

**RN-62 — La terminal original de un ticket NUNCA se sobrescribe.**
Si otra terminal (la caja) cobra un ticket, se registra quién cobró, pero la terminal
de origen se preserva. Esto permite auditar "este ticket nació en T3 y se cobró en Caja".
*Origen: `service.py:164-165` (comentario explícito).*

**RN-63 — Un ticket no pagado muestra "PENDIENTE" como cobrador.**
En los estados OPEN y DRAFT, el campo de cobrador se presenta como pendiente, no como
vacío.
*Origen: `service.py:299-302`.*

**RN-64 — Un ticket sin capturista se atribuye a "SISTEMA".**
Si no hay capturista registrado, se atribuye al sistema (no queda vacío).
*Origen: `service.py:297`.*

**RN-65 — Un ticket sin cobrador (pero pagado) se atribuye a "SISTEMA/AUTO".**
Cubre el caso de cobros automáticos.
*Origen: `service.py:302`.*

**RN-66 — La fecha de un ticket se interpreta en la zona horaria del negocio.**
Al filtrar tickets por fecha, los límites del día se calculan en la zona horaria
configurada del negocio, no en UTC ni en la zona del servidor.
*Origen: `service.py:577-582`.*

### 1.14 REGLAS DE NEGOCIO — Canales (Panadería / Heladería)

**RN-67 — Todo ticket pertenece a un canal.**
El canal es PANADERIA o HELADERIA. Los tickets históricos sin canal se tratan como
PANADERIA.
*Origen: `service.py:625`, `service.py:667`.*

**RN-68 — El canal se escribe en el mismo INSERT, nunca después.**
Un ticket **nunca** existe con canal nulo. Esto elimina la ventana en la que un ticket
de Heladería aparecía en el POS de Panadería.
*Origen: `service.py:787-789` (comentario explícito).*

**RN-69 — El canal es parte de la identidad del reciclaje.**
Ver RN-26.

### 1.15 REGLAS DE NEGOCIO — Visión (conteo automático)

**RN-70 — El conteo por visión es un asistente, no una autoridad.**
El sistema intenta reconocer productos por cámara, pero si falla, el operador continúa
en modo manual. La visión nunca bloquea la venta.
*Origen: `router.py:505-510`.*

**RN-71 — Un fallo del motor de visión devuelve una respuesta vacía, no un error.**
Si OpenCV no está disponible o la imagen es inválida, se devuelve "sin detecciones"
en vez de propagar un error.
*Origen: `router.py:512-520`, `service.py:873-874`.*

**RN-72 — El reconocimiento usa un umbral de confianza mínimo.**
Para evitar falsos positivos, solo se reporta una detección si supera un umbral.
*Origen: `service.py:924`.*

**RN-73 — El entrenamiento de visión es por SKU.**
Las imágenes de entrenamiento se organizan por SKU de producto.
*Origen: `service.py:831-833`.*

### 1.16 DEUDAS CONOCIDAS DEL POS ACTUAL (que el plano NO debe replicar)

> Estas son **fallas verificadas empíricamente** en el POS actual (v22). El plano
> arquitectónico debe especificar el comportamiento **correcto**, no el actual.

**DEUDA-01 (D28) — El puente a Almacenes está muerto.**
El evento de almacén nunca se inserta porque el SKU del producto no está cargado en
memoria al momento de construir el evento, y el error se silencia con un `try/except`.
**Consecuencia de negocio:** el inventario no se descuenta automáticamente al vender.
**El plano debe especificar:** el contrato de outbox debe construirse con datos
explícitamente cargados, y el fallo debe ser **observable** (log + métrica), no silencioso.

**DEUDA-02 (D29) — El respaldo de sesión elige arbitrariamente.**
Si no se encuentra la sesión de la terminal, el respaldo toma "la primera sesión activa"
sin orden definido. Con varias cajas activas, puede asociar el ticket a la terminal
equivocada.
**El plano debe especificar:** el respaldo debe ser determinista (orden explícito) y
**registrar** que se usó un respaldo, para que sea auditable.

**DEUDA-03 (D27) — La sesión de datos se expira en bloque tras cada operación.**
Tras cada operación de ítem, se expiran **todos** los objetos de la sesión de datos.
Esto es un parche de rendimiento que contamina el diseño y causa errores sutiles.
**El plano debe especificar:** las operaciones deben devolver respuestas ligeras
explícitas, sin expirar el estado global de la sesión.

**DEUDA-04 — Acoplamiento por base de datos con otros módulos.**
El POS lee directamente tablas de Productos, Pedidos, Almacenes y Empleados.
**El plano debe especificar:** contratos explícitos entre módulos (ver Sección 2).

**DEUDA-05 — El POS contiene lógica de visión (OpenCV) en su servicio.**
El reconocimiento de imágenes no es una responsabilidad del punto de venta.
**El plano debe especificar:** la visión es un módulo aparte, consumido por contrato.

### 1.17 Resumen cuantitativo de la lógica extraída

| Categoría | Reglas | Cantidad |
|---|---|---|
| Ciclo de vida del Ticket | RN-01 a RN-07 | 7 |
| Captura de ítems | RN-08 a RN-16 | 9 |
| Concurrencia (bloqueo optimista) | RN-17 a RN-21 | 5 |
| DRAFT GUARD | RN-22 a RN-24 | 3 |
| Reciclaje de folios | RN-25 a RN-28 | 4 |
| Recolección de basura (GC) | RN-29 a RN-33 | 5 |
| Ocupación de terminales (candados) | RN-34 a RN-42 | 9 |
| Guardado de emergencia (ZERO-LOSS) | RN-43 a RN-48 | 6 |
| Puente hacia Almacenes (outbox) | RN-49 a RN-53 | 5 |
| Puente hacia Producción (pedidos) | RN-54 a RN-59 | 6 |
| Trazabilidad y auditoría | RN-60 a RN-66 | 7 |
| Canales (Panadería / Heladería) | RN-67 a RN-69 | 3 |
| Visión (conteo automático) | RN-70 a RN-73 | 4 |
| **TOTAL** | **RN-01 a RN-73** | **73** |

**Lectura del cuadro:** el POS actual contiene **73 reglas de negocio verificables**.
Ninguna de ellas es "código de interfaz". Todas son comportamiento del dominio.
Estas 73 reglas son el **programa arquitectónico** del nuevo POS: el nuevo POS debe
satisfacer las 73, pero puede hacerlo con una estructura distinta (y mejor).

**Deudas conocidas:** 5 (DEUDA-01 a DEUDA-05). El nuevo POS debe satisfacer las 73
reglas **sin** heredar las 5 deudas.

---

## SECCIÓN 2 — CONTRATOS ENTRE MÓDULOS

> **Esta sección define las fronteras del nuevo POS.** El POS actual lee tablas de otros
> módulos (acoplamiento por base de datos, DEUDA-04). El nuevo POS **no** lee tablas
> ajenas: **pide** a través de contratos explícitos.
>
> Un contrato es una promesa: "si me pides esto, te respondo esto otro, con estas
> garantías". El contrato no dice **cómo** el otro módulo lo hace por dentro.

### 2.1 Principios de los contratos

**C-01 — El POS nunca lee una tabla que no sea suya.**
El POS es dueño de: Sesión de Terminal, Ticket, Ítem de Ticket, Candado de Terminal.
Todo lo demás (productos, inventario, pedidos, empleados, auditoría) se obtiene por
contrato.

**C-02 — Todo contrato es explícito y versionado.**
Cada contrato tiene un nombre, una versión, una entrada y una salida. Si cambia, cambia
de versión. El POS declara qué versión consume.

**C-03 — Un contrato puede fallar, y el POS debe saber qué hacer.**
Cada contrato declara su comportamiento ante fallo: ¿bloquea la venta? ¿degrada?
¿continúa? (Ver RN-52: el puente a Almacenes **nunca** bloquea la venta.)

**C-04 — Los contratos son sincrónicos para lo crítico y asincrónicos para lo derivado.**
- **Sincrónico** (el POS espera la respuesta): validar producto, validar empleado.
- **Asincrónico / outbox** (el POS no espera): descontar inventario, proyectar pedido,
  registrar auditoría. Si el otro módulo está caído, el POS sigue cobrando.

### 2.2 Contrato con **Productos** (Catálogo)

**Necesidad del POS:** saber si un producto existe, si está activo, su precio y su SKU.

| Aspecto | Definición |
|---|---|
| **Nombre** | `catalogo.producto.consultar` |
| **Tipo** | Sincrónico (bloquea la venta si falla) |
| **Entrada** | `producto_id` (o `sku`) |
| **Salida** | `{ id, sku, nombre, precio_unitario, activo, canal }` |
| **Garantías** | El precio devuelto es el vigente al momento de la consulta. |
| **Fallo** | Si el producto no existe → error de negocio (RN-12). Si está inactivo → error de negocio (RN-13). Si el módulo está caído → **el POS no puede cobrar** (no se puede vender a ciegas). |

**Reglas que dependen de este contrato:** RN-11 (congelar precio), RN-12, RN-13.

**Nota de diseño:** el POS **congela** el precio en el ítem (RN-11). El contrato entrega
el precio vigente; el POS lo copia al ítem. Si el precio cambia después, el ítem ya
capturado no cambia.

### 2.3 Contrato con **Almacenes** (Inventario)

**Necesidad del POS:** avisar que se vendió algo, para descontar inventario.

| Aspecto | Definición |
|---|---|
| **Nombre** | `almacenes.movimiento.registrar` |
| **Tipo** | Asincrónico / outbox (NO bloquea la venta) |
| **Entrada** | `{ ticket_folio, canal, items: [{ sku, qty }], ocurrido_en }` |
| **Salida** | Acuse de recibo (o nada, si es fire-and-forget) |
| **Garantías** | Si el ticket se guarda, el evento se registra (atomicidad, RN-50). |
| **Fallo** | **NUNCA interrumpe el POS** (RN-52). El fallo debe ser **observable** (log + métrica), no silencioso (corrige DEUDA-01). |

**Reglas que dependen de este contrato:** RN-49 a RN-53.

**Corrección de DEUDA-01:** el contrato exige que el `sku` esté **explícitamente
cargado** antes de construir el evento. El POS actual lo construye con un SKU vacío y
silencia el error. El nuevo POS debe: (a) cargar el SKU por contrato con Productos,
(b) si falta, **registrar el fallo** y no silenciarlo.

### 2.4 Contrato con **Producción / Reparto** (Pedidos)

**Necesidad del POS:** proyectar un ticket de tipo PEDIDO como un pedido de producción.

| Aspecto | Definición |
|---|---|
| **Nombre** | `produccion.pedido.proyectar` |
| **Tipo** | Asincrónico / outbox (NO bloquea la venta) |
| **Entrada** | `{ ticket_folio, estado, tipo_entrega, empaque, items, cliente }` |
| **Salida** | Acuse de recibo |
| **Garantías** | La proyección es **idempotente** (RN-58): el mismo folio no duplica el pedido. |
| **Fallo** | No interrumpe el POS. Se reintenta. |

**Reglas que dependen de este contrato:** RN-54 a RN-59.

**Mapeo de estados (RN-55):** Ticket PAID → Pedido PAGADO; Ticket OPEN → Pedido TENTATIVO.

**Defaults (RN-56, RN-57):** sin tipo de entrega → PICKUP; sin empaque → PROPIO.

### 2.5 Contrato con **Auditoría**

**Necesidad del POS:** registrar quién hizo qué, para trazabilidad.

| Aspecto | Definición |
|---|---|
| **Nombre** | `auditoria.evento.registrar` |
| **Tipo** | Asincrónico / outbox (NO bloquea la venta) |
| **Entrada** | `{ actor_id, terminal_id, accion, entidad, entidad_id, payload, ocurrido_en }` |
| **Salida** | Acuse de recibo |
| **Garantías** | Todo evento es inmutable y con marca de tiempo en UTC. |
| **Fallo** | No interrumpe el POS. |

**Reglas que dependen de este contrato:** RN-60 a RN-66.

**Nota:** la auditoría **no** es la fuente de verdad del ticket. El ticket guarda sus
propios campos de trazabilidad (capturista, cobrador, terminal de origen). La auditoría
es el registro histórico de eventos.

### 2.6 Contrato con **Estadísticas**

**Necesidad del POS:** reportar ventas para dashboards.

| Aspecto | Definición |
|---|---|
| **Nombre** | `estadisticas.venta.registrar` |
| **Tipo** | Asincrónico / outbox |
| **Entrada** | `{ ticket_folio, canal, total, items, cobrado_en, terminal_id }` |
| **Salida** | Acuse de recibo |
| **Garantías** | Solo se reportan tickets PAID (una venta es un hecho, no una intención). |
| **Fallo** | No interrumpe el POS. |

**Regla clave:** Estadísticas **consume** el hecho de venta; no lo produce. El POS es
la fuente de verdad de la venta.

### 2.7 Contrato con **Seguridad / Empleados**

**Necesidad del POS:** saber quién es el cajero y qué permisos tiene.

| Aspecto | Definición |
|---|---|
| **Nombre** | `seguridad.empleado.consultar` |
| **Tipo** | Sincrónico (bloquea la operación si falla) |
| **Entrada** | `empleado_id` |
| **Salida** | `{ id, nombre, activo, permisos: [...] }` |
| **Garantías** | El empleado devuelto está activo. |
| **Fallo** | Si el empleado no existe o está inactivo → error de negocio. Si el módulo está caído → el POS no puede autenticar. |

**Reglas que dependen de este contrato:** RN-16 (sesión activa), RN-60, RN-61.

### 2.8 Contrato con **Visión** (módulo aparte)

**Necesidad del POS:** asistir al cajero con conteo automático por cámara.

| Aspecto | Definición |
|---|---|
| **Nombre** | `vision.deteccion.predecir` |
| **Tipo** | Sincrónico, pero **degradable** (nunca bloquea la venta) |
| **Entrada** | `{ imagen, terminal_id }` |
| **Salida** | `{ detecciones: [{ sku, confianza, bbox }] }` |
| **Garantías** | Solo se reportan detecciones que superan el umbral de confianza (RN-72). |
| **Fallo** | Devuelve lista vacía, **no** un error (RN-71). El operador continúa en manual (RN-70). |

**Corrección de DEUDA-05:** la visión **no** vive dentro del POS. Es un módulo aparte
que el POS consume por contrato. El POS no importa OpenCV.

### 2.9 Matriz de dependencias del nuevo POS

| Módulo | Tipo de contrato | ¿Bloquea la venta? | Reglas |
|---|---|---|---|
| Productos | Sincrónico | **Sí** | RN-11, RN-12, RN-13 |
| Almacenes | Outbox | No | RN-49 a RN-53 |
| Producción/Reparto | Outbox | No | RN-54 a RN-59 |
| Auditoría | Outbox | No | RN-60 a RN-66 |
| Estadísticas | Outbox | No | (derivado) |
| Seguridad/Empleados | Sincrónico | **Sí** | RN-16, RN-60, RN-61 |
| Visión | Sincrónico degradable | No | RN-70 a RN-73 |

**Principio rector:** solo **dos** contratos pueden bloquear la venta (Productos y
Seguridad). Todo lo demás es derivado y tolerante a fallos. Esto es lo que hace que el
POS sea robusto: **cobrar es lo único que no puede fallar.**

---

## SECCIÓN 3 — PLAN DE LA PRIMERA ETAPA: EXTRAER LA LÓGICA DEL NEGOCIO

> **Objetivo de la etapa:** producir el documento de reglas de negocio (Sección 1 de
> este plano) de forma **completa, verificable y trazable**, sin tocar el ERP.
>
> **Resultado de la etapa:** las 73 reglas (RN-01 a RN-73) extraídas, trazadas a su
> origen, y las 5 deudas documentadas. **Esta etapa ya está ejecutada en este documento**
> (Sección 1). El plan siguiente describe cómo se hizo y cómo se verifica.

### 3.1 Alcance de la etapa

**Dentro del alcance:**
- Leer el módulo POS actual (solo lectura, commit congelado `fe9f6ed`).
- Identificar y numerar cada regla de negocio.
- Trazar cada regla a su origen (archivo + línea).
- Separar reglas de negocio de decisiones técnicas.
- Documentar las deudas conocidas.
- Definir los contratos entre módulos (Sección 2).

**Fuera del alcance (etapas posteriores):**
- Diseñar el modelo de datos del nuevo POS.
- Diseñar la estructura de carpetas del nuevo POS.
- Escribir código del nuevo POS.
- Decidir la tecnología del nuevo POS.
- Migrar datos.

### 3.2 Método de extracción (5 pasos)

**Paso 1 — Congelar la fuente.**
Anclar la lectura al commit `fe9f6ed` (tag `v22-estable-fe9f6ed`). Cualquier lectura
posterior es sobre una versión distinta y debe re-trazarse. (El ERP hoy corre en `5802f45`,
V23; los cambios V22/V23 que tocan al POS están en §A.4 de la Especificación Funcional.)

**Paso 2 — Inventariar los puntos de entrada.**
Listar todas las operaciones que el POS expone (crear sesión, reservar, agregar ítem,
modificar, eliminar, cobrar, bloquear terminal, guardado de emergencia, visión). Cada
punto de entrada esconde una o más reglas.

**Paso 3 — Extraer reglas por operación.**
Para cada operación, responder: ¿qué valida? ¿qué garantiza? ¿qué rechaza? ¿qué
proyecta hacia otros módulos? Cada respuesta es una regla candidata.

**Paso 4 — Clasificar cada regla.**
- ¿Es **regla de negocio** (el *qué*)? → va a la Sección 1.
- ¿Es **decisión técnica** (el *cómo*)? → **no** va al plano (es un accidente histórico).
- ¿Es **deuda** (una falla)? → va a §1.16, marcada como "no replicar".

**Paso 5 — Trazar y verificar.**
Cada regla debe tener su origen (archivo + línea). Una regla sin origen es una
invención; una regla con origen es un hecho verificable.

### 3.3 Criterios de aceptación de la etapa

| # | Criterio | Cómo se verifica |
|---|---|---|
| A1 | Toda operación del POS está cubierta por al menos una regla | Cruzar la lista de endpoints con las reglas |
| A2 | Toda regla tiene origen (archivo + línea) | Revisar que ninguna diga "origen: desconocido" |
| A3 | Ninguna regla menciona código de interfaz | Buscar términos de UI en la Sección 1 |
| A4 | Las deudas están documentadas y marcadas "no replicar" | Revisar §1.16 |
| A5 | Los contratos cubren todas las dependencias externas | Cruzar §2.9 con las reglas de puente |
| A6 | El documento no toca el ERP | `git status` del ERP limpio |

### 3.4 Entregables de la etapa

1. **Este documento** (`PLANO ARQUITECTONICO PARA EL NUEVO POS.md`), con:
   - Sección 0: opinión sobre el enfoque.
   - Sección 1: las 73 reglas de negocio + 5 deudas.
   - Sección 2: los contratos entre módulos.
   - Sección 3: este plan.
2. **Respaldo en el repositorio nuevo:**
   `https://github.com/vikutasan/PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS`

### 3.5 Riesgos de la etapa y mitigaciones

| Riesgo | Mitigación |
|---|---|
| Extraer una regla que en realidad es un bug | Trazar a origen + marcar deudas explícitamente |
| Omitir una regla | Cruzar endpoints contra reglas (criterio A1) |
| Contaminar el plano con decisiones técnicas | Clasificación estricta (Paso 4) |
| Desactualizar el plano si el POS cambia | Anclar a commit congelado (Paso 1) |

### 3.6 Qué sigue (etapas posteriores, NO en esta etapa)

- **Etapa 2:** modelo de datos del nuevo POS (entidades, relaciones, invariantes).
- **Etapa 3:** estructura de módulos y fronteras internas.
- **Etapa 4:** diseño de los contratos en detalle (esquemas de entrada/salida).
- **Etapa 5:** plan de construcción (orden de implementación).
- **Etapa 6:** plan de replicación por sucursal (configuración vs. código).

---

## SECCIÓN 4 — ESTRUCTURA DEL REPOSITORIO NUEVO

> El repositorio `PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS` contiene **solo documentos**.
> Cero código de producción. Cero dependencias. Cero riesgo para el negocio en marcha.

```
PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS/
├── README.md                                  # Qué es este repo y su regla dura
├── PLANO ARQUITECTONICO PARA EL NUEVO POS.md  # Este documento (fundacional)
├── 01-logica-del-negocio/
│   ├── reglas-de-negocio.md                   # Las 73 reglas (RN-01 a RN-73)
│   └── deudas-conocidas.md                    # Las 5 deudas (DEUDA-01 a DEUDA-05)
├── 02-contratos/
│   ├── contrato-productos.md
│   ├── contrato-almacenes.md
│   ├── contrato-produccion.md
│   ├── contrato-auditoria.md
│   ├── contrato-estadisticas.md
│   ├── contrato-seguridad.md
│   └── contrato-vision.md
├── 03-modelo-de-datos/                        # (Etapa 2 — pendiente)
├── 04-estructura-de-modulos/                  # (Etapa 3 — pendiente)
└── 05-plan-de-construccion/                   # (Etapa 5 — pendiente)
```

**Regla del repositorio:** ningún archivo de este repositorio puede importar, referenciar
o depender del ERP actual. Es un repositorio de **planos**, no de **obras**.

---

## SECCIÓN 5 — MÓDULOS DEL ERP FUERA DEL ALCANCE DEL POS

> Este documento es el plano **del POS**. Pero el POS no es el único módulo del ERP.
> Hay módulos que **no** son el POS, que el POS **consume**, y cuyo rol ya está decidido
> aunque todavía no se reconstruyan.

Esta sección existe para que la IA constructora **no invente** lo que ya está decidido.
Un módulo que no se menciona es un módulo que se improvisa.

### 5.1 Vista General (módulo de configuración transversal)

**Qué es:** el módulo donde se **declaran** los valores que afectan a todos los módulos del ERP.

**Qué NO es:**
- No es una pantalla del POS.
- No es un módulo de negocio (no vende, no cobra, no mueve inventario).
- No es un dashboard de solo lectura.

**Su responsabilidad (decidida):**

| Valor transversal | Dónde se guarda | Quién lo consume |
|---|---|---|
| Zona horaria (`business_timezone`) | `system_settings` | Todo el ERP, vía `TimezoneContext` |
| Moneda (`business_currency`) | `system_settings` | Todo el ERP, vía `MoneyContext` |
| Sucursal (`sucursal_id`) | `system_settings` | Todo el ERP |

**Su rol está anclado en:** [`DIRECTRICES_TRANSVERSALES_DEL_ERP.md`](./DIRECTRICES_TRANSVERSALES_DEL_ERP.md) — DT-06 (Configuración del Negocio).

**Su estado:** ✅ **FASE 1 completada** (22 Sep 2026). Su especificación funcional completa (pantallas, campos, validaciones, permisos) vive en [`ESPECIFICACION_FUNCIONAL_VISTA_GENERAL.md`](./ESPECIFICACIONES%20DEL%20PROYECTO/ESPECIFICACION_FUNCIONAL_VISTA_GENERAL.md): 4 interfaces, 35 reglas (VG-01 a VG-35), 8 hallazgos catalogados y 10 criterios de aceptación. Lo que ya estaba decidido —y por eso se declaró aquí— es **su rol transversal**: es el único lugar donde se declaran los valores que todos los módulos consumen.

**Por qué se declara ahora y no después:** si no se declara, la IA constructora que arme el POS (o Caja, o Almacenes) va a inventar un selector de zona horaria o de moneda dentro de su propio módulo. Eso es exactamente el error que produjo las 5 implementaciones de tiempo y los 80 formateos de dinero del ERP actual. Declarar el rol ahora cierra esa puerta antes de que se abra.

### 5.2 La simetría completa

```
system_settings  →  Vista General  →  contexto global  →  cada módulo
   (guarda)           (declara)        (distribuye)        (consume)
```

- **`system_settings`** — la tabla donde vive el valor. Documentada en el Documento 8.
- **Vista General** — la interfaz donde el humano lo elige. **Declarada aquí.**
- **Contexto global** — `TimezoneContext` (existe), `MoneyContext` (por crear). Documentado en el compendio.
- **Cada módulo** — consume el contexto; nunca define el valor. Verificado por DT-06.

### 5.3 Otros módulos fuera del alcance (por decidir)

Los siguientes módulos existen en el ERP actual pero **su rol transversal aún no se ha decidido**. Se listan para que no se improvisen, no para especificarlos:

| Módulo | Rol transversal | Estado |
|---|---|---|
| Vista General | Configuración del negocio | ✅ Decidido (DT-06) |
| Auditoría | Rastro de operaciones | ✅ Decidido (DT-05) |
| Almacenes | Ledger de inventario | ✅ Decidido (DT-04) |
| Seguridad / Perfiles | Identidad y permisos | ⏳ Por decidir |
| Estadísticas | Reportería | ⏳ Por decidir |
| Visión | Reconocimiento de imágenes | ⏳ Por decidir |

**Regla:** un módulo de esta tabla no se especifica hasta que se reconstruya. Pero su **rol transversal**, si ya está decidido, se declara aquí para que ningún otro módulo lo invada.

---

## SECCIÓN 6 — DECLARACIÓN DE LA REGLA DURA (REPETIDA AL CIERRE)

> **NO SE TOCA EL ERP INSTALADO Y CORRIENDO.**
> **NO SE TOCA NINGUNO DE SUS MÓDULOS.**
> **NO SE MODIFICA NI UNA LÍNEA DEL POS ACTUAL.**

Este documento se produjo **leyendo** el POS actual (solo lectura), anclado al commit
`fe9f6ed` (tag `v22-estable-fe9f6ed`). El ERP permanece intacto y operando; su HEAD es
`5802f45` (V23) al 22 Sep 2026.

El POS actual es la **fuente de verdad funcional** (lo que hace). Este plano es la
**fuente de verdad estructural** (cómo debería construirse). Copiamos su
*comportamiento*, no su *deuda*.

---

*Documento fundacional. Versión 1.1. Anclado al commit `5802f45` (V23) del ERP actual; ingeniería inversa sobre `fe9f6ed`.*