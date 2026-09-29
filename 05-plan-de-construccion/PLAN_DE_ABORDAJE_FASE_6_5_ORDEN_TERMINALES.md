# PLAN DE ABORDAJE — FASE 6.5 (Orden de despliegue de terminales)

> **Tipo:** Micro-fase correctiva (extensión de la Fase 6, ya cerrada)
> **Versión:** v1.0
> **Estado:** Propuesto — pendiente de aprobación del dueño
> **Fecha:** 2026-09-29

---

## §1. ORIGEN Y ALCANCE

### §1.1 De dónde sale

Al cerrar la Fase 6, el dueño detectó una capacidad que **olvidó integrar al gestor de
terminales**: poder definir si la numeración de las terminales se muestra de
**izquierda a derecha** o de **derecha a izquierda**, mediante un selector.

### §1.2 Qué es (y qué NO es)

**Es:** una preferencia de **despliegue visual**. El array de terminales se invierte
al renderizar el grid. Los IDs (`T1`, `T2`, …) **no cambian**.

**NO es:** una renumeración. Los IDs de terminal están ligados a `TerminalSession`,
`TerminalLock`, `Ticket.terminal_id` y `CashSession`. Renumerar rompería el historial
de sesiones y los tickets ya cobrados, y violaría **RN-12**
(`rn12_terminal_id_inmutable`). Esta opción se descartó explícitamente.

### §1.3 Decisión del dueño

El dueño eligió la **Opción A — invertir el orden de despliegue (visual)**, sin tocar
los IDs.

---

## §2. DIAGNÓSTICO DEL CÓDIGO REAL

| Pieza | Archivo | Estado actual |
|---|---|---|
| Hook de negocio | `apps/pos/src/hooks/useTerminals.js` | `terminals` = `[{id, name, icon}]`; `addTerminal(position)` ya acepta `'start'`/`'end'` |
| Servicio | `apps/pos/src/services/terminalService.js` | `saveTerminalConfig(terminals)` envía `{ terminals }` |
| UI | `apps/pos/src/components/TerminalSelector.jsx` | Gestor (líneas 214-344) edita nombre/icono; grid (líneas 373-451) renderiza `terminals.map(...)` |
| Backend | `apps/api` | **No existe** endpoint `/pos/terminals/config` (0 coincidencias) |

### §2.1 Hallazgo clave

**El orden de las terminales ya es el orden del array.** El grid hace
`terminals.map(...)` y renderiza en ese orden. Por lo tanto, la capacidad de invertir
la numeración **ya existe implícitamente**: solo falta exponerla con un control.

**Consecuencia:** la complejidad es **baja**. No hay que tocar el motor de render,
solo añadir un control que reordene el array al desplegarlo.

---

## §3. DISEÑO

### §3.1 Modelo de estado

Se añade al hook `useTerminals` una preferencia:

```js
const [ordenTerminales, setOrdenTerminales] = useState('izq-der'); // 'izq-der' | 'der-izq'
```

Y un array derivado para el render:

```js
const terminalesDesplegadas = useMemo(
  () => (ordenTerminales === 'der-izq' ? [...terminals].reverse() : terminals),
  [terminals, ordenTerminales]
);
```

**Regla dura:** `terminals` (el array fuente) **nunca** se muta al invertir. Solo se
invierte una **copia** para el despliegue. Esto garantiza que `saveConfig()` siga
persistiendo el orden canónico y que los IDs no se toquen.

### §3.2 Persistencia

El backend **no tiene** endpoint de configuración de terminales. Por lo tanto, la
preferencia se guarda en **`localStorage`** bajo la clave
`pos.ordenTerminales`:

- Al montar: se lee de `localStorage` (con fallback a `'izq-der'`).
- Al cambiar: se escribe en `localStorage`.

**Nota:** cuando exista el endpoint `/pos/terminals/config`, esta preferencia se
migrará a la configuración persistida del servidor. Se documenta como deuda técnica
explícita.

### §3.3 UI

En el **gestor de terminales** (modo `showManager`), junto a los botones
"💾 Guardar cambios" y "← Volver", se añade un selector de dos estados:

```
Orden de despliegue:  [ → Izquierda a derecha ]  [ ← Derecha a izquierda ]
```

- Dos botones tipo "toggle" (el activo resaltado en `#ea580c`).
- Al pulsar, se actualiza `ordenTerminales` y se persiste en `localStorage`.
- El grid del gestor **y** el grid principal respetan el orden elegido.

### §3.4 Estética

Se respeta la estética del POS viejo ya presente en `TerminalSelector.jsx`:
estilos inline, fondo madera, acento `#ea580c`, esquinas redondeadas. **No** se
introduce Tailwind en este componente (usa estilos inline por diseño).

---

## §4. SUB-FASES

| Sub-fase | Qué construye | Archivos | Gate |
|---|---|---|---|
| **F6.5.0** | Estado `ordenTerminales` + `terminalesDesplegadas` + persistencia en `localStorage` + `invertirOrden()` en el hook | `useTerminals.js` | Tests del hook |
| **F6.5.1** | Selector de orden en el gestor + aplicar `terminalesDesplegadas` a ambos grids | `TerminalSelector.jsx` | Tests del componente |
| **F6.5.2** | Ficha de la micro-fase + commit + push | `FICHA_F6_5_ORDEN_TERMINALES.md` | Suite completa + guards |

**Orden:** F6.5.0 → F6.5.1 → F6.5.2. Cada sub-fase cierra con gate en verde + suite
completa + guards 7/7 + commit + push. **No se abre la siguiente sin cerrar la
anterior.**

---

## §5. CRITERIOS DEL GATE

El gate `useTerminals.f6_5.test.jsx` debe verificar:

1. El valor por defecto de `ordenTerminales` es `'izq-der'`.
2. `terminalesDesplegadas` devuelve el array en orden canónico por defecto.
3. Al llamar `invertirOrden()`, `terminalesDesplegadas` queda invertido.
4. **`terminals` (el array fuente) NO se muta** al invertir (regla dura §3.1).
5. La preferencia se persiste en `localStorage` al invertir.
6. La preferencia se **lee** de `localStorage` al montar.
7. Un valor inválido en `localStorage` cae al default `'izq-der'`.
8. `saveConfig()` sigue enviando el array **canónico** (no el invertido).

El gate `TerminalSelector.f6_5.test.jsx` debe verificar:

9. El selector de orden se renderiza en el gestor.
10. Al pulsar "Derecha a izquierda", el grid se invierte visualmente.
11. Los IDs de las tarjetas **no cambian** al invertir (siguen siendo `T1`, `T2`, …).

---

## §6. RIESGOS Y MITIGACIONES

| Riesgo | Probabilidad | Impacto | Mitigación |
|---|---|---|---|
| Invertir muta el array fuente y corrompe `saveConfig()` | Media | Alto | Se invierte una **copia** (`[...terminals].reverse()`); el gate lo verifica (criterio 4 y 8) |
| `localStorage` no disponible (modo privado) | Baja | Bajo | Lectura/escritura envueltas en `try/catch`; fallback al default |
| El backend añade el endpoint y hay dos fuentes de verdad | Media | Medio | Deuda técnica documentada en §3.2; migración planificada |
| El selector rompe la estética del gestor | Baja | Bajo | Se usan los mismos estilos inline ya presentes |

---

## §7. LO QUE ESTE PLAN NO HACE (delimitación explícita)

- **No** renumera las terminales ni cambia sus IDs (Opción B descartada, §1.2).
- **No** toca el backend ni añade endpoints.
- **No** modifica `TerminalSession`, `TerminalLock`, `Ticket` ni `CashSession`.
- **No** añade dependencias.
- **No** introduce Tailwind en `TerminalSelector.jsx` (usa estilos inline por diseño).
- **No** reabre la Fase 6: es una **micro-fase correctiva** posterior a su cierre.

---

## §8. CRITERIO DE APROBACIÓN

Este plan se considera aprobado cuando el dueño confirma:

1. Que la **Opción A** (inversión visual, sin tocar IDs) es lo que quiere.
2. Que la **persistencia en `localStorage`** es aceptable mientras no exista el
   endpoint de configuración.
3. Que el **orden de ejecución** (§4) es el correcto.
