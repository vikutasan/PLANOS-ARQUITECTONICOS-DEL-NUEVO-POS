# 🧩 PLAN DE ABORDAJE — MICRO-FASE F4.5 "MONTAJE DEL GESTOR DE CAJA"

> **Versión:** 1.0
> **Fecha:** 30 Sep 2026
> **Autor:** Arquitecto del Nuevo POS
> **Estado:** PROPUESTO — pendiente de aprobación
> **Naturaleza:** Micro-fase **CORRECTIVA** (no es una fase nueva del Plan Maestro;
> cierra un hueco de INTEGRACIÓN detectado en la Fase 4).

---

## 1. EL HALLAZGO (verificado contra el código real — REGLA DURA 2)

El usuario reportó: *"acabo de ver que no está el botón de habilitar caja y no sé
si esté la lógica de habilitar caja"*.

**Diagnóstico verificado (no asumido):**

| Pieza | ¿Existe? | Evidencia |
|---|---|---|
| Backend de caja (6 endpoints) | ✅ SÍ | `cash.py` — 371 líneas, contratos 9–14 |
| Servicio de caja (frontend) | ✅ SÍ | `cashService.js` — 6 operaciones |
| Componente Gestor de Caja | ✅ SÍ | `GestorDeCaja.jsx` — 467 líneas, 3 estados |
| Test del componente | ✅ SÍ | `GestorDeCaja.test.jsx` — pasa en verde |
| **Punto de entrada (botón)** | ❌ **NO** | `POSHeader.jsx` no tiene botón de caja |
| **Montaje en la pantalla** | ❌ **NO** | `RetailVisionPOS.jsx` no importa ni renderiza `GestorDeCaja` |
| **Ruta dedicada** | ❌ **NO** | `App.jsx` solo tiene `/` y `/pos` |

**Conclusión:** `GestorDeCaja.jsx` es un **componente huérfano**. Existe, está
probado, pero **ningún usuario puede llegar a él**. Es exactamente el riesgo que
el propio Plan Maestro advirtió en §10.6 ("de adentro hacia afuera"): cada
sub-fase pasó su compuerta en aislamiento, pero **el paso de INTEGRACIÓN se
olvidó**.

**Impacto real (severidad ALTA):** RN-49 exige una sesión de caja `OPEN` para
cobrar (`_sesion_caja_activa_o_400` en `pos.py`). Sin punto de entrada, el
operador **no puede abrir el turno**, y por lo tanto **el POS no puede cobrar**.
El sistema está funcionalmente incompleto para producción.

**Lección registrada:** *"el componente existe y pasa su test" ≠ "el usuario
puede llegar a él"*. Esta micro-fase cierra esa brecha y deja el patrón
documentado para que no se repita.

---

## 2. ALCANCE (5 puntos, acotado y verificable)

### F4.5.1 — Punto de entrada: botón "Caja" en el header
- Añadir un botón **"Caja"** al grupo de acciones de la derecha de
  `POSHeader.jsx` (el `<div className="flex items-center gap-2">` de la línea 107).
- Props nuevas: `onAbrirCaja` (callback) y `cajaAbierta` (booleano, para
  resaltar el botón cuando hay turno abierto — mismo patrón que
  `pedidoProgramado` y `clienteIdentificado`).
- Sigue el patrón visual existente: `min-h-tactil min-w-tactil`, `aria-label`,
  `title`, resaltado con `bg-acento` cuando `cajaAbierta`.
- **Gate:** el botón es visible y clickeable; dispara `onAbrirCaja`.

### F4.5.2 — Montaje del `GestorDeCaja` en `RetailVisionPOS`
- Importar `GestorDeCaja` en `RetailVisionPOS.jsx`.
- Estado nuevo: `const [cajaAbierta, setCajaAbierta] = useState(false)`.
- Renderizar el gestor como **overlay** (mismo patrón que `temaAbierto`,
  `vozAbierta`, `clienteAbierto`), pasando:
  - `terminalId={terminalEfectiva}`
  - `usuarioId={sesion?.employee_id || null}`
  - `onCerrar={() => setCajaAbierta(false)}`
- Cablear `onAbrirCaja={() => setCajaAbierta(true)}` al `POSHeader`.
- **Gate:** al pulsar "Caja", el gestor se abre; al cerrar, se oculta.

### F4.5.3 — Guarda de cobro (aviso, no solo 400)
- Hoy, si no hay turno abierto, el cobro falla con un `400` del backend
  (RN-49). El operador ve un error críptico.
- Añadir una **guarda proactiva** en el flujo de cobro de `RetailVisionPOS`:
  si no hay turno de caja abierto, mostrar un aviso claro
  ("Abre la caja antes de cobrar") y ofrecer abrir el gestor, en vez de dejar
  que el `fetch` falle.
- **Gate:** intentar cobrar sin turno muestra el aviso y no dispara el cobro.

### F4.5.4 — Test de integración
- Test nuevo `RetailVisionPOS.f4_5.test.jsx` que verifique:
  1. El botón "Caja" existe en el header.
  2. Al pulsarlo, `GestorDeCaja` se monta (aparece su contenido).
  3. Al cerrar, desaparece.
  4. Cobrar sin turno muestra el aviso (no llama a `cobrarTicket`).
- **Gate:** test en verde.

### F4.5.5 — Cierre documental
- Ficha `FICHA_F4_5_MONTAJE_CAJA.md` con la evidencia.
- Actualizar §10.6 del Plan Maestro con la lección ("el paso de integración
  también es una compuerta").
- Registrar el hash real del commit en la ficha (patrón ficha-hash).
- **Gate:** CI completo en verde + push.

---

## 3. LO QUE **NO** SE HACE (anti-alcance)

- ❌ **No** se reescribe `GestorDeCaja.jsx` (ya funciona y está probado).
- ❌ **No** se toca el backend `cash.py` (los 6 endpoints ya existen).
- ❌ **No** se crea una ruta nueva en `App.jsx` (el gestor es un overlay, no
  una pantalla; coherente con el resto de paneles del POS).
- ❌ **No** se implementan pagos mixtos (eso es F9.1, fase separada).
- ❌ **No** se añade cola local ni modo offline-first (decisión arquitectónica
  ya tomada: el servidor es la única fuente de verdad).

---

## 4. ORDEN DE EJECUCIÓN Y DEPENDENCIAS

```
F4.5.1 (botón)  ──►  F4.5.2 (montaje)  ──►  F4.5.3 (guarda)  ──►  F4.5.4 (test)  ──►  F4.5.5 (cierre)
```

- F4.5.1 y F4.5.2 son **inseparables** (el botón sin montaje no sirve; el
  montaje sin botón no es alcanzable). Se ejecutan juntos.
- F4.5.3 depende de que el gestor esté montado (para poder ofrecer abrirlo).
- F4.5.4 valida todo lo anterior.
- F4.5.5 cierra.

---

## 5. RIESGOS Y MITIGACIONES

| Riesgo | Probabilidad | Mitigación |
|---|---|---|
| El overlay tapa el carrito y confunde | Baja | Mismo patrón que los otros overlays; cierre explícito |
| `usuarioId` nulo si no hay sesión | Media | El gestor ya maneja `usuarioId`; se pasa `sesion?.employee_id || null` |
| Romper tests existentes de `RetailVisionPOS` | Media | Añadir props opcionales con default; no cambiar firmas existentes |
| Olvidar registrar el hash | Baja | F4.5.5 lo exige como compuerta |

---

## 6. CRITERIOS DE ACEPTACIÓN (la compuerta de la micro-fase)

1. ✅ Existe un botón "Caja" visible en el header del POS.
2. ✅ Al pulsarlo, se abre el `GestorDeCaja` con la terminal y el usuario reales.
3. ✅ El gestor permite abrir turno, registrar movimientos y cerrar turno
   (funcionalidad ya existente, ahora alcanzable).
4. ✅ Cobrar sin turno abierto muestra un aviso claro (no un 400 críptico).
5. ✅ Test de integración en verde.
6. ✅ CI completo (`npm run ci`) en verde.
7. ✅ Ficha escrita + §10.6 del Plan Maestro actualizado + hash registrado.
8. ✅ Ambos repos commiteados y pusheados.

---

## 7. BITÁCORA DE CAMBIOS

| Versión | Fecha | Cambio |
|---|---|---|
| 1.0 | 30 Sep 2026 | Redacción inicial. Micro-fase correctiva F4.5 para montar el `GestorDeCaja` huérfano (hallazgo verificado contra el código real). |
