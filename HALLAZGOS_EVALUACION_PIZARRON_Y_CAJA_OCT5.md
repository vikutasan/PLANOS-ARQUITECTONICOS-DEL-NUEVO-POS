# HALLAZGOS — Evaluación del Pizarrón y Gestor de Caja (5 Oct 2026)

**Sesión:** Antigravity — Evaluación funcional viejo POS vs nuevo POS.
**Commits:** `ed4d0a5` (pizarrón) + `d9bd311` (gestor de caja).
**Repositorio:** `github.com/vikutasan/NUEVO-POS`.

---

## 1. Pizarrón de Cuentas Abiertas (OpenAccountsCorkboard)

### D1 — ⛔ CRÍTICA (CORREGIDA): CAJA debe ver TODAS las cuentas

**Hallazgo:** El viejo POS mostraba TODAS las cuentas OPEN cuando la terminal
era CAJA (header decía "X TOTALES"). El nuevo POS siempre filtraba por terminal
(RN-31), impidiendo que la CAJA viera cuentas de otras terminales para cobrarlas.

**Decisión:** PORTAR. La CAJA es el punto de cobro central.

**Implementación:**
- `client.js`: `listarCuentasAbiertas()` soporta terminal_id opcional.
- `openAccountsService.js`: nueva función `listarTodasLasCuentasAbiertas()`.
- `useOpenAccounts.js`: opción `todasLasTerminales` para modo CAJA.
- `RetailVisionPOS.jsx`: pasa `cajaHabilitada={Boolean(turnoCaja)}`.
- `POSHeader.jsx`: muestra "X totales" (CAJA) vs "X mías" (terminal).

### D3 — Click en post-it + botón (MEJORADA)

**Hallazgo:** El viejo POS hacía todo el post-it clickeable; el nuevo tenía solo
un botón "Recuperar". El nuevo era mejor ergonomía pero faltaba el click amplio.

**Decisión:** COMBINAR: click en todo el post-it + botón como refuerzo visual +
accesibilidad por teclado (Enter/Space).

### D4 — Botón Refrescar manual (AGREGADA)

**Hallazgo:** El viejo POS tenía polling cada 5s. El nuevo cargaba una vez.

**Decisión:** NO portar el polling (Prohibición #1: sin timers). Agregar un
botón 🔄 manual en el encabezado del pizarrón.

### D5 — Paridad visual del post-it (PORTADA)

**Hallazgo:** El nuevo POS tenía post-its genéricos (sin aspect-square, sin folio
grande, sin esquina doblada, sin hover lift).

**Decisión:** PORTAR el estilo del viejo POS: `aspect-square`, folio `#XXX` en
tamaño 5xl, hover con lift (-translate-y-2), esquina doblada CSS, "VACÍO"
estilizado, "CLIENTE LOCAL" visible, pin con brillo interior.

### D6 — Save status badge (DESCARTADA correctamente)

**Hallazgo:** El viejo POS mostraba "⚠️ SIN GUARDAR" en el botón del pizarrón.

**Decisión:** NO portar. El nuevo POS usa persistencia atómica por ítem (cada
operación es un POST individual). No hay cola local que pueda fallar.

---

## 2. Gestor de Caja (GestorDeCaja)

### B1 — Default de tipo de movimiento (CORREGIDA)

**Hallazgo:** El viejo POS defaulteaba a `SALIDA`; el nuevo a `ENTRADA`.

**Decisión:** Cambiar a `SALIDA`. En una panadería, las salidas de efectivo
(cambio, compras menores de insumos) son más frecuentes que las entradas.

### B2 — Confirmación de 2 pasos al abrir turno (CORREGIDA)

**Hallazgo:** El viejo POS pedía confirmación: "¿Confirmar fondo de $X?".
El nuevo abría el turno directamente.

**Decisión:** PORTAR. El fondo inicial afecta todo el arqueo del turno. Un error
aquí descuadra 8 horas de operación.

**Implementación:**
- Nuevo estado `confirmandoFondo`.
- `alPedirConfirmacionFondo()` valida y muestra modal.
- `alAbrirTurno()` solo se ejecuta al confirmar.
- Modal con `role="alertdialog"` y tokens DT-09.

### B3 — Confirmación de 2 pasos al cerrar turno (CORREGIDA)

**Hallazgo:** El viejo POS pedía confirmación en 2 pasos: "¿Confirmar cantidades?"
→ "¿Cerrar turno?". El nuevo cerraba directamente.

**Decisión:** PORTAR. El cierre es IRREVERSIBLE.

**Implementación:**
- Nuevo estado `confirmandoCierre`.
- `alPedirConfirmacionCierre()` muestra modal con ⚠️ "Esta acción es irreversible".
- El modal muestra el descuadre en vivo con color semántico (verde/rojo).
- Botón "Sí, cerrar turno" en `bg-peligro` (rojo) para reforzar la gravedad.
- `alCerrarTurno()` solo se ejecuta al confirmar.

### B4 — Botón "Iniciar Nuevo Turno" (CORREGIDA)

**Hallazgo:** El viejo POS ofrecía un botón "Iniciar Nuevo Turno" tras cerrar.
El nuevo requería salir y reentrar al gestor.

**Decisión:** PORTAR. Hace el flujo de relevo de turno más fluido.

**Implementación:**
- `alNuevoTurno()` resetea TODO el estado al modo SIN_TURNO.
- El botón reemplaza a "Cancelar / Cerrar turno" cuando `diferencia` ya existe.
- Usa `data-testid="nuevo-turno"` para verificabilidad.

### B5 — Polling de resumen cada 10s (NO CORREGIDA)

**Hallazgo:** El viejo POS refrescaba el resumen cada 10s con `setInterval`.

**Decisión:** NO portar. La Prohibición #1 de la nueva arquitectura prohíbe
timers y auto-guardado. El refresco manual al agregar/eliminar movimientos es
suficiente para la operación normal.

### Nota sobre PIN eliminado (NO es brecha)

El viejo POS tenía un sistema de validación de PIN independiente del login del
ERP. El nuevo POS correctamente eliminó esta duplicidad: el usuario ya está
identificado por el login del ERP. El PIN del viejo era un vestigio de cuando
el POS no tenía integración con el sistema de perfiles.

---

## 3. Cumplimiento DT-09 de ambos componentes

| Regla | Pizarrón | Gestor de Caja |
|-------|----------|----------------|
| R-01 (sin anchos fijos) | ✅ `max-w-[1100px]` | ✅ `max-w-[1100px]` |
| R-02 (mínimo 10px) | ✅ `text-[10px]` min | ✅ `text-sm` min |
| R-03 (3 modos) | ✅ `grid-cols-1/sm:2/lg:3/xl:4` | ✅ `grid-cols-1/lg:2` |
| R-04 (≥44px) | ✅ `min-h-tactil` | ✅ `min-h-tactil` |
| Tokens | ✅ `bg-madera-panel`, `text-crema` | ✅ `bg-fondo-panel`, `text-crema-ticket` |
