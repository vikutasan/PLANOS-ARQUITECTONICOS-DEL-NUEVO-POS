# HALLAZGOS DE LA AUDITORÍA DE BRECHAS — 5 Oct 2026

> **Autor:** Auditoría cruzada POS viejo vs POS nuevo (Antigravity + revisión humana)
> **Fecha:** 5 Oct 2026
> **Método:** Comparación archivo por archivo del ERP viejo (`ERP-R-DE-RICO/apps/pos/`)
> contra el POS nuevo (`NUEVO-POS/apps/pos/src/`), contrastando contra las decisiones
> arquitectónicas documentadas en este repositorio.
>
> **REGLA:** Cualquier IA o desarrollador que trabaje en el POS nuevo DEBE leer este
> documento antes de "rescatar" funcionalidad del POS viejo. No todo lo que falta
> es una omisión — algunas ausencias son MEJORAS deliberadas.

---

## 1. CONTEXTO

El POS nuevo se construyó aplicando ingeniería inversa sobre el POS viejo (anclado al
commit `fe9f6ed`, tag `v22-estable-fe9f6ed`). Durante la construcción (fases F0–F12),
Deepseek portó ~85% de la funcionalidad. Sin embargo, se identificaron 4 brechas
aparentes. Esta auditoría evalúa si cada brecha debe cerrarse o si la ausencia es
una consecuencia correcta del rediseño.

---

## 2. LAS 4 BRECHAS EVALUADAS

### 2.1 ESTADO FINAL

| ID | Brecha | Veredicto | Acción |
|----|--------|-----------|--------|
| **B1** | `useBeforeUnload` no cableado | ✅ CERRADA (5 Oct 2026) — Hook mejorado con `confirmar` + cableado en `RetailVisionPOS.jsx` (2 instancias: emergency save + lock release) | Completo |
| **B2** | `usePOSSession` no existe | ✅ AUSENCIA CORRECTA — superada por D-12 | **NO PORTAR** |
| **B3** | `sessionReset` sin tests guardianes | ✅ CERRADA (5 Oct 2026) — `FORBIDDEN_KEYS` añadido + 18 tests guardianes en `sessionReset.guardian.test.js` (4 guardianes × N tests) | Completo |
| **B4** | `SalesReceipt` parcial | ✅ YA RESUELTA — paridad funcional confirmada | Ninguna |

---

## 3. B1 — `useBeforeUnload`: BRECHA DE CABLEADO

### 3.1 Hallazgo

El hook [`useBeforeUnload.js`](../NUEVO-POS/apps/pos/src/hooks/useBeforeUnload.js) está
correctamente implementado y es **arquitectónicamente superior** al viejo:

| Aspecto | Viejo | Nuevo |
|---------|-------|-------|
| Diseño | Hardcodea endpoints + payload inline | Genérico: `{ url, obtenerPayload, activo, enviar }` |
| Testabilidad | No testeable (hardcoded `navigator.sendBeacon`) | Inyectable: `enviar` es un parámetro |
| Stale closures | Usa refs manuales | Usa `payloadRef` (prohibición #3) |

**PERO:** al buscar `useBeforeUnload` en `RetailVisionPOS.jsx` se encontraron **0 resultados**.
El hook existe, tiene tests (`hooks.test.jsx`), pero la pantalla **nunca lo monta**.

### 3.2 Qué falta (exactamente)

1. **Montar `useBeforeUnload` en `RetailVisionPOS.jsx`** con:
   - `url`: el endpoint de persistencia de emergencia
   - `obtenerPayload`: función que lee el carrito y terminal de los refs
   - `activo`: `carrito.lineas.length > 0`

2. **Añadir `e.preventDefault` + `e.returnValue`** al hook como opción configurable
   (`confirmar: true`). Esto muestra el diálogo nativo "¿seguro que desea salir?".
   La decisión de mostrarlo la toma el CONSUMIDOR (la pantalla), no el hook.

3. **Montar un segundo `useBeforeUnload`** para liberar el lock de terminal:
   - `url`: endpoint de unlock
   - `obtenerPayload`: función que devuelve `{ occupier_id }`
   - `activo`: `locking.esDueno`

### 3.3 Esfuerzo: ~2.5h

---

## 4. B2 — `usePOSSession`: AUSENCIA CORRECTA (NO PORTAR)

> [!CAUTION]
> **ESTA ES LA SECCIÓN MÁS IMPORTANTE DE ESTE DOCUMENTO.**
>
> Cualquier IA que vea que `usePOSSession` "falta" en el POS nuevo puede sentirse
> tentada a portarlo. **NO LO HAGA.** La ausencia es una MEJORA, no una omisión.

### 4.1 Qué hacía `usePOSSession` en el viejo POS

El hook vivía en [`ERP-R-DE-RICO/apps/pos/hooks/usePOSSession.js`](../ERP-R-DE-RICO/apps/pos/hooks/usePOSSession.js) (7.8K, 175 líneas) y tenía 4 responsabilidades:

1. **Carga de catálogo** — categorías + productos desde el servidor
2. **Pre-reserva de folio** — `posService.reserveTicket(terminalId)` creaba un ticket DRAFT *antes* de agregar productos
3. **Zero-Auto-Restore** — limpiaba localStorage al entrar a terminal
4. **Persistencia de metadatos** — guardaba `account_num`, `orderType`, `orderData` en localStorage

### 4.2 Por qué existía en el viejo POS

El viejo POS usaba **auto-save periódico por BLOB completo** (`setInterval` cada N
segundos guardaba `{ account_num, items[], status, terminal_id }`). Este diseño
necesitaba un `account_num` ANTES de que existieran ítems, porque:

- Sin folio, no había dónde guardar el BLOB del auto-save
- El auto-save era la ÚNICA red de seguridad contra cortes de luz
- El localStorage era backup del backup

### 4.3 Por qué el nuevo POS NO lo necesita (decisión D-12)

El nuevo POS usa **persistencia atómica por ítem** (contratos 18-20). Cada
`anadirLinea()` es una operación HTTP idempotente con su `item_id`. No hay auto-save.
No hay BLOB. No hay localStorage.

La decisión D-12 (documentada en `RetailVisionPOS.jsx:183-186` y `314-342`) establece:

```
El ticket OPEN nace al agregar el PRIMER ítem.
No hay pre-reserva. No hay DRAFT vacío.
```

Esto se implementa con `asegurarTicket()`:

```js
// RetailVisionPOS.jsx:328-342
const asegurarTicket = useCallback(async (bloque = null) => {
    if (ticketIdRef.current) return ticketIdRef.current;  // ya existe → reusar
    const creado = await acciones.crearTicket([], bloque); // no existe → crear
    // ...
}, [acciones]);
```

### 4.4 Qué pasaría si se portara `usePOSSession`

| Consecuencia | Gravedad |
|-------------|----------|
| Se crearían tickets DRAFT vacíos que nunca se usan → basura en la DB | 🔴 Alta |
| Se reintroduciría la race condition de folio (v4.3: "ANTI-RACE: sync refs ANTES del setState") | 💀 Crítica |
| Se reintroduciría la dependencia de localStorage (eliminada por diseño) | 🔴 Alta |
| Se contradiría el contrato 18 (ítem atómico idempotente) | 💀 Crítica |
| Se duplicaría la carga de catálogo (ya existe en RetailVisionPOS.jsx:210-233) | 🟡 Media |

### 4.5 Dónde vive ahora cada responsabilidad del viejo `usePOSSession`

| Responsabilidad vieja | Dónde está en el nuevo POS |
|-----------------------|---------------------------|
| Carga de catálogo | `RetailVisionPOS.jsx:210-233` — efecto directo con `api.getCatalogo()` |
| Pre-reserva de folio | **ELIMINADA** — reemplazada por `asegurarTicket()` (D-12) |
| Zero-Auto-Restore | **INNECESARIA** — no hay localStorage; el servidor es la fuente de verdad |
| Persistencia de metadatos | **INNECESARIA** — no hay localStorage; toda persistencia es atómica |

### 4.6 Decisión (FINAL, no negociable)

**`usePOSSession` NO se porta. NO se reimplementa. NO se recrea "mejorado".**

La pre-reserva de folio es un patrón del paradigma BLOB que el nuevo POS abandonó.
Recrearla bajo cualquier nombre sería una regresión arquitectónica.

---

## 5. B3 — `sessionReset`: FALTAN TESTS, NO CÓDIGO

### 5.1 Hallazgo

La implementación de [`sessionReset.js`](../NUEVO-POS/apps/pos/src/state/sessionReset.js)
(2.9K) es **superior** a la del viejo (6.9K):

| Aspecto | Viejo | Nuevo |
|---------|-------|-------|
| Claves | 12 hardcodeadas al componente | `VALOR_INICIAL` genérico + extensible |
| Refs | No maneja refs (el caller sincroniza) | Maneja refs nativamente (`{ current }`) |
| Extensibilidad | Editar 3 archivos para agregar una clave | Declarar en el llamador |

Lo que **falta** son los tests guardianes que existían en el viejo:

| Test viejo | Qué protegía |
|------------|-------------|
| `architecture.test.js` (31.6K) | Que las 4 rutas de salida usen `buildResetPatch()` |
| `sessionReset.test.js` (3.2K) | Unitarios del patch |
| `sessionReset.asymmetry.test.js` (5.1K) | Detecta asimetría entre rutas |
| `sessionReset.equivalence.test.js` (9.5K) | Equivalencia del patch en todas las rutas |

### 5.2 Qué se debe crear

1. **Test de simetría:** Verificar que TODAS las rutas de salida de
   `RetailVisionPOS.jsx` llaman a `aplicarReset()` o `resetearSesion()`.

2. **Test de claves prohibidas:** Crear un `FORBIDDEN_KEYS` en `sessionReset.js` y
   verificar que `VALOR_INICIAL` nunca incluya claves con ciclo de vida propio
   (las líneas del carrito se limpian vía `clearCart`, no vía reset).

3. **Test de cobertura de refs:** Si alguien agrega un `useRef` nuevo al componente
   y olvida incluirlo en `VALOR_INICIAL`, el test debe fallar.

### 5.3 Esfuerzo: ~2.5h

---

## 6. B4 — `SalesReceipt`: YA RESUELTA

### 6.1 Hallazgo

El nuevo [`SalesReceipt.jsx`](../NUEVO-POS/apps/pos/src/components/SalesReceipt.jsx)
(8.2K) tiene paridad funcional con el viejo (15.9K):

| Feature | Viejo | Nuevo | Evidencia |
|---------|-------|-------|-----------|
| Edición de cantidad (+/−) | ✅ | ✅ | Props `onIncrementar`, `onDecrementar` |
| Quitar línea | ✅ | ✅ | Prop `onQuitar` |
| Banner persistente | ✅ | ✅ | Prop `banner: { tipo, mensaje }` + JSDoc "no auto-ocultable" |
| Terminal ID | ✅ | ✅ | Prop `terminalId` |
| Botón COBRAR | ✅ | ✅ | Prop `onCobrar` + `cobrando` |

La diferencia de tamaño (15.9K → 8.2K) se explica por:
- Tailwind CSS (clases utilitarias) vs CSS inline/custom
- Theme engine (variables CSS) vs colores hardcodeados
- Composición (delega más al caller) vs monolito

### 6.2 Decisión

**No hay trabajo pendiente.** La brecha era un falso positivo causado por comparar
tamaños de archivo sin leer el código.

---

## 7. RESUMEN DE ACCIONES

| Prioridad | Acción | Esfuerzo | Archivo(s) |
|-----------|--------|----------|------------|
| 🔴 Alta | Cablear `useBeforeUnload` en `RetailVisionPOS.jsx` | ~1.5h | `RetailVisionPOS.jsx` |
| 🔴 Alta | Añadir opción `confirmar` a `useBeforeUnload` | ~0.5h | `useBeforeUnload.js` |
| 🟡 Media | Montar segundo beacon para unlock de terminal | ~0.5h | `RetailVisionPOS.jsx` |
| 🟡 Media | Añadir `FORBIDDEN_KEYS` a `sessionReset.js` | ~0.5h | `sessionReset.js` |
| 🟡 Media | Crear 3 tests guardianes de simetría | ~2.0h | `sessionReset.test.js` (nuevo) |
| ❌ | ~~Portar `usePOSSession`~~ | — | **NO HACER** |
| ❌ | ~~Completar `SalesReceipt`~~ | — | **YA RESUELTO** |
| | | **~5h total** | |

---

## 8. ADVERTENCIAS PARA IAs FUTURAS

### ❌ NO portar `usePOSSession`

Si ves que el POS nuevo no tiene `usePOSSession` y piensas "hay que portarlo":

1. Lee la sección 4 de este documento.
2. Lee `RetailVisionPOS.jsx:183-186` (decisión D-12).
3. Lee `RetailVisionPOS.jsx:314-342` (`asegurarTicket`).
4. Entiende que la pre-reserva de folios era un parche del paradigma BLOB.
5. Entiende que el nuevo POS usa persistencia atómica por ítem (contratos 18-20).
6. **Concluye que la ausencia es correcta.**

### ❌ NO recrear auto-save periódico

El viejo POS tenía `setInterval` para guardar el carrito cada N segundos. El nuevo
POS NO tiene auto-save porque cada operación (`anadirLinea`, `cambiarCantidad`,
`quitarLinea`) es atómica e idempotente. Si falla, `withRetries` reintenta. No hay
nada que "auto-guardar" porque cada cambio ya se persiste en el acto.

### ❌ NO usar localStorage para estado del POS

El viejo POS usaba `localStorage` como backup. El nuevo POS tiene al **servidor como
fuente única de verdad**. Si la pestaña muere, el ticket OPEN sobrevive en la DB y se
recupera desde el pizarrón de cuentas abiertas (contrato 23). No hay localStorage.

### ✅ SÍ cablearlo cuando el hook exista pero esté huérfano

Si un hook existe, tiene tests, pero la pantalla no lo usa → es una brecha de CABLEADO,
no de implementación. La solución es montar el hook, no reescribirlo.

---

## 9. MEJORAS DE UX APLICADAS (5 Oct 2026)

Adicionalmente a las brechas de funcionalidad, se aplicaron 3 mejoras de UX:

### 9.1 Eliminación de "MODO: MOSTRADOR"

**Qué era:** Un `<span>` en el header que mostraba el breakpoint responsivo activo
("MOSTRADOR", "COMPACTO", "MÓVIL"). Era un artefacto de desarrollo.

**Por qué se quitó:** No es accionable (el modo cambia solo por ancho de pantalla),
es redundante (el cajero ya VE el layout), y ocupaba espacio en el header.

**Archivo:** `POSHeader.jsx` — líneas 318-322 eliminadas.

### 9.2 Reubicación del selector de tema (🎨)

**Antes:** Botón en el header del POS + overlay en `RetailVisionPOS.jsx`.

**Después:** Botón en la landing (`TerminalSelector.jsx`), junto al botón "⚙️ Gestor
de Terminales", formando un "rincón de configuración".

**Justificación:** El tema es una PREFERENCIA, no una acción transaccional. Su lugar
natural es la pantalla de configuración (landing), no el header donde se vende.

**Archivos:**
- `POSHeader.jsx` — botón 🎨 eliminado
- `RetailVisionPOS.jsx` — estado `temaAbierto` y overlay eliminados
- `TerminalSelector.jsx` — botón + overlay añadidos

### 9.3 Indicador de red con latencia y semáforo de 3 estados

**Antes (nuevo POS):** Span suelto en el header → "En línea" / "Sin red" (binario).

**Después:** Sub-línea DENTRO del botón de terminal → "● RED OK 9ms" / "RED LENTA 540ms"
/ "SIN RED" (semáforo de 3 colores: verde/amarillo/rojo).

**Justificación:**
- Fusiona lo mejor de ambos POS: diagnóstico con latencia del viejo + diseño inyectable
  del nuevo.
- Asocia visualmente la salud de la red con la terminal activa.
- El estado `slow` (>500ms) avisa PREVENTIVAMENTE antes de la caída.
- 2 fallos consecutivos para `down` (evita falsos positivos).

**Archivos:**
- `useNetworkHealth.js` — reescrito con `performance.now()`, semáforo 3 estados,
  etiqueta/color pre-formateados.
- `POSHeader.jsx` — indicador movido al botón de terminal (sub-línea).
- `RetailVisionPOS.jsx` — pasa `red.estado`/`red.etiqueta`/`red.color` en vez de
  `enLinea`.

### 9.4 Eliminación de "Sin sesión" / "Sesión abierta"

**Qué era:** Un badge en el header que mostraba si había sesión de terminal activa.

**Por qué se quitó:** No es accionable (el cajero no puede abrir/cerrar sesión desde
ahí), es redundante (si llegaste al POS = tienes sesión), y el viejo POS nunca tuvo
esta leyenda.

**Nota:** El estado `sesion` sigue vivo internamente — los hooks `useTerminalLocking`,
`useBeforeUnload` y `GestorDeCaja` lo necesitan.

### 9.5 Reorganización de botones del header (Voz → Caja → Pizarrón)

**Antes:** Caja → Pizarrón → 🎤 (iconito suelto).

**Después:** Voz → Caja → Pizarrón. Los 3 con el mismo patrón visual heredado del
viejo POS: rectángulo con título grande (18px) + estado/acción (14px acento).

**Archivos:** `POSHeader.jsx` — sección derecha reescrita.

---

## 10. AUDITORÍA DE LÓGICA DE TERMINALES (5 Oct 2026)

### 10.1 Contexto

Se evaluó si la lógica de bloqueo/desbloqueo de terminales del nuevo POS tiene
debilidades respecto al viejo POS. Se identificaron 3 diferencias potenciales y se
auditaron contra la documentación arquitectónica.

### 10.2 Las 3 diferencias evaluadas

| ID | Diferencia | Nuevo POS | Viejo POS |
|----|-----------|-----------|-----------|
| **D1** | Detección de robo de lock | No la tiene | Polling cada 15s + `lockWarning` + `ForceLogoutModal` |
| **D2** | Cleanup al desmontar | Solo `useBeforeUnload` | `useEffect(return cleanup, [])` + `useBeforeUnload` |
| **D3** | Intervalos configurables | Constantes hardcodeadas | Leídos del backend (`/settings`) |

### 10.3 VEREDICTO: NINGUNA SE PORTA

#### D1 — Detección de robo de lock → ❌ NO PORTAR

**Evidencia documental:**
- **PLAN_MAESTRO (línea 706):** `V-06 ForceLogoutModal` → **DESCARTADA** —
  "Reemplazada por el heartbeat con TTL (`useTerminalLocking`)".
- **RN-36:** Los candados expiran por inactividad (TTL, 15 min). El mecanismo de
  recuperación es el TTL, no un polling de vigilancia.
- **DT-05.1:** "Exclusividad garantizada por candado de latido activo". La garantía
  es el heartbeat, no el polling de seguridad.
- **DT-05.2.2:** "Cero Falsos Desbloqueos — prohibido usar objetos volátiles en deps
  de useEffect para unlock". El polling de seguridad del viejo POS era exactamente lo
  que causaba el bug.

**Bug del viejo POS que esto causaba:**
El polling de seguridad (`checkMyLock` cada 15s) causó el **bug anti-ping-pong**
documentado en `useTerminalLocking.js` del viejo POS (líneas 15, 96):
1. El polling detectaba que el lock se perdió (por blip de red o TTL transitorio).
2. El cajero se re-conectaba automáticamente.
3. Esto generaba un ciclo lock/unlock que desestabilizaba las terminales.
4. La solución fue marcar `ANTI-PING-PONG: Ya no intenta re-adquirir candados`.
5. Finalmente se **descartó** toda la feature (V-06) en el nuevo POS.

**Conclusión:** No hay "robo" de lock. Hay expiración natural (TTL). Si un admin usa
`force_unlock` (RN-39), la terminal del cajero original simplemente deja de recibir
heartbeats exitosos y el lock expira. El nuevo POS ya maneja esto correctamente.

#### D2 — Cleanup al desmontar → ⚠️ YA CUBIERTO

El nuevo POS tiene `useBeforeUnload` cableado (B1 cerrada) que hace `liberarLock()`
con `sendBeacon` al cierre de pestaña. Además, el TTL (RN-36) cubre los escenarios
donde `beforeunload` no dispara (crash, fuerza bruta). Agregar un `useEffect` cleanup
sería un refuerzo de cortesía, no una brecha funcional.

**Importante:** Si una IA futura decide agregar un `useEffect` cleanup, DEBE respetar:
- **DT-05.2.2:** Solo best-effort unlock. NUNCA re-adquirir locks.
- **Prohibición #3:** Usar refs, no estado cerrado.
- **Anti-ping-pong:** No reintentar si el unlock falla.

#### D3 — Intervalos configurables → ❌ NO PORTAR (ahora)

Deseable a futuro para tuning en producción, pero no es una brecha funcional. Los
valores hardcodeados son razonables:
- Heartbeat: 10s (viejo: 20s configurable)
- Network health: 10s (viejo: 15s configurable)
- Terminal polling: 15s (viejo: 5s configurable)

### 10.4 ADVERTENCIA PARA FUTURAS IAs

> **⛔ PROHIBIDO portar `ForceLogoutModal`, `lockWarning`, o cualquier polling de
> seguridad que verifique "¿sigo siendo el dueño del lock?".**
>
> Esta feature fue la causa del **bug anti-ping-pong** en el viejo POS y fue
> **DESCARTADA explícitamente** (V-06 en el Plan Maestro). El mecanismo de
> recuperación del nuevo POS es el **TTL** (RN-36): si el lock se pierde, expira
> naturalmente. No hay nada que detectar ni nada que alertar al cajero.
>
> **NO confundir con:** el `useBeforeUnload` (B1), que sí libera el lock como
> cortesía al cerrar la pestaña. Eso es un RELEASE proactivo, no un POLLING
> de vigilancia.

### 10.5 Resumen de la lógica de terminales del nuevo POS

```
LANDING                           POS ACTIVO
───────                           ──────────
useTerminals                      useTerminalLocking
  ├── poll /status (15s)            ├── heartbeat (10s) → renueva TTL
  ├── resolveCardState()            ├── tomarLock()     → POST /lock
  │   └── free | mine | occupied    ├── liberarLock()   → POST /unlock
  └── selectTerminal()              └── bloqueada/dueno (estado local)
       └── POST /lock                         │
                                    useBeforeUnload (B1)
                                      └── sendBeacon /unlock
                                           (al cerrar pestaña)

                    BACKEND
                    ───────
                    TerminalLock (PostgreSQL, no RAM)
                      ├── TTL = 15 min (RN-36)
                      ├── Solo dueño libera (RN-37)
                      ├── Admin force_unlock (RN-39)
                      └── Heartbeat purga expirados (RN-41)
```

---

## 11. EVALUACIÓN: ¿RESTRINGIR NAVEGACIÓN ENTRE TERMINALES POR PERFIL? (5 Oct 2026)

### 11.1 La duda planteada por el dueño del proyecto

> "Estoy considerando crear un permiso llamado 'navegación entre terminales del POS
> de panadería' en el gestor de perfiles, para controlar qué perfil puede cambiar
> libremente de terminal (tanto en la landing como con 'Cambiar Estación' dentro
> del POS). La finalidad sería:
>
> 1. Saber DÓNDE está físicamente la persona que captura.
> 2. Deslindar responsabilidades sobre CÓMO se capturó una cuenta."

### 11.2 Análisis: ¿qué datos ya tiene el sistema?

El modelo de datos del nuevo POS (`MODELO_DE_DATOS_DEL_NUEVO_POS.md`, tabla `tickets`,
líneas 87-93) ya registra POR CADA TICKET:

| Campo | Ejemplo | Qué responde |
|-------|---------|-------------|
| `terminal_id` | `TERM-04` | **¿DÓNDE?** — ubicación física de la máquina |
| `captured_by_id` | UUID de Victor | **¿QUIÉN capturó?** |
| `cashed_by_id` | UUID de María | **¿QUIÉN cobró?** |
| `session_id` | FK a `terminal_sessions` | **¿EN QUÉ TURNO?** |
| `cash_session_id` | FK a `cash_sessions` | **¿EN QUÉ TURNO DE CAJA?** |
| `created_at` | `2026-10-05 10:32:00 UTC` | **¿CUÁNDO?** |

### 11.3 Respuesta a cada objetivo

#### Objetivo 1: "Saber dónde está físicamente la persona"

**YA SE SABE.** Cada ticket registra `terminal_id` + `captured_by_id`. Si Victor
captura un ticket en TERM-04, la base de datos dice: "Victor, en la máquina 4, a
las 10:32am". Si Victor se cambia a TERM-05, los tickets nuevos dicen TERM-05. Los
anteriores siguen diciendo TERM-04. La trazabilidad es completa sin restringir nada.

#### Objetivo 2: "Deslindar responsabilidades"

**NO TIENE IMPACTO ADICIONAL.** La responsabilidad se deslinda con **QUIÉN**
(`captured_by_id`), no con **DÓNDE** (`terminal_id`). Ejemplo:

> "Este ticket tiene un descuento no autorizado."
> - ¿Quién lo hizo? → `captured_by_id` = Victor. **Suficiente.**
> - ¿En qué máquina? → `terminal_id` = TERM-04. **Complementario, no determinante.**

Restringir la navegación no cambia quién es responsable. Victor es responsable de lo
que Victor capture, esté en TERM-04 o en TERM-05.

### 11.4 Conclusión: NO IMPLEMENTAR

**Se descarta esta feature por 5 razones:**

1. **Costo alto, ganancia nula.** Crear un permiso, validarlo en landing, validarlo
   en "Cambiar Estación", manejar edge cases (¿terminal asignada caída?) — todo esto
   para obtener un dato que ya existe en el ticket.

2. **Rigidez operativa.** Si TERM-04 se descompone a mitad de turno, el cajero
   necesita cambiarse a TERM-05 rápidamente. Con restricciones, tiene que buscar un
   admin. Eso es tiempo muerto en hora pico.

3. **Los shortcuts de Chrome ya hacen el trabajo.** Cada máquina tiene su acceso
   directo (T1, T2, T3...) con `?terminal=X` en la URL. Es la solución correcta y
   natural: no necesita ser forzada con permisos.

4. **El lock ya impide conflictos.** RN-34: una terminal solo puede ser ocupada por
   un cajero a la vez. No hay riesgo de duplicidad.

5. **Antecedente del viejo POS.** El viejo POS tiene `canSwitch` en el header
   (línea 58) que permite ocultar "Cambiar Estación" por permiso, pero en la práctica
   nunca se usó para restringir — la rigidez causaba más problemas que los que resolvía.

### 11.5 Alternativa recomendada

Si se desea una vista de "¿quién estuvo en qué terminal y cuándo?", la solución
correcta es un **dashboard de auditoría** que consulte los datos que ya existen en
`terminal_sessions` + `tickets` (campos `terminal_id`, `captured_by_id`, `created_at`).
Esto da la foto completa sin restringir la operación.

### 11.6 ADVERTENCIA PARA FUTURAS IAs

> **⛔ NO crear permisos de "navegación entre terminales" ni restringir el selector
> de terminales ni el botón "Cambiar Estación" por perfil de usuario.**
>
> Esta feature fue evaluada el 5 Oct 2026 con el dueño del proyecto y se concluyó
> que NO aporta valor: los datos de trazabilidad (`terminal_id`, `captured_by_id`,
> `cashed_by_id`) ya existen en cada ticket. Restringir la navegación solo añade
> rigidez operativa sin beneficio de auditoría.
>
> **SI el dueño en el futuro cambia de opinión**, debe re-evaluar esta sección
> explícitamente. No implementar silenciosamente.

---

*Documento generado el 5 Oct 2026. Anchored al estado del repo `NUEVO-POS` en esa fecha.*

