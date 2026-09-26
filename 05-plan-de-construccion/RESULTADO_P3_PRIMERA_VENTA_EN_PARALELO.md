# RESULTADO DE P3 — Primera venta de prueba en paralelo

**Fecha:** 2026-09-26
**Etapa del plan:** P3 — Prueba en paralelo
**Estado:** ✅ CERRADA
**Regla dura respetada:** NO SE TOCÓ EL ERP INSTALADO Y CORRIENDO.

---

## 1. Objetivo de P3

Levantar el frontend del POS nuevo en el puerto **5100** y hacer la **primera venta de
prueba end-to-end** contra su **propia API** (puerto 5101) y su **propia base de datos**
(`nuevo_pos`), **sin tocar el ERP**.

---

## 2. Lo que se hizo (paso a paso)

| Paso | Acción | Resultado |
|------|--------|-----------|
| P3.1 | Verificar el ERP antes de empezar | HEAD `b0bc297`, árbol limpio, `rderico-*` Up 30 h |
| P3.2 | Revisar [`vite.config.js`](../../NUEVO-POS/apps/pos/vite.config.js) | puerto 5100, `strictPort: true`, `host: true` |
| P3.3 | `npm install` + `npm run dev` en `apps/pos` | 129 paquetes; Vite v5.4.21 en `http://localhost:5100/` |
| P3.4 | Verificar frontend + API | app shell servido; `/health` ok; catálogo 6 productos / 3 categorías |
| P3.5 | Primera venta end-to-end | folio **V0003**, total **34.00**, **PAID** |
| P3.6 | Verificar el ERP después de la prueba | HEAD `b0bc297`, árbol limpio, `rderico-*` Up 30 h |

---

## 3. Evidencia de la primera venta

**Productos elegidos del catálogo propio del POS nuevo:**

- Concha de vainilla (`PAN-003`) — 8.00 × 2 = 16.00
- Café americano (`BEB-001`) — 18.00 × 1 = 18.00

**Llamadas a la API del POS nuevo (`http://localhost:5101`):**

```
POST /pos/tickets            -> 201  folio: V0003  status: OPEN  total: 34.00  version: 0
  lineas: 2x 8.00 = 16.00 | 1x 18.00 = 18.00
POST /pos/tickets/{id}/pay   -> 200  folio: V0003  status: PAID  version: 1
Re-pago (RN-23)              -> 400  regla: RN-23  "No se puede modificar un ticket PAID"
```

**Estado de la base de datos aislada `nuevo_pos`:**

```
 account_num | status | total | version
-------------+--------+-------+---------
 V0001       | PAID   | 25.00 |       1
 V0002       | OPEN   |  4.00 |       0
 V0003       | PAID   | 34.00 |       1
```

Los folios V0001 y V0002 son de las pruebas de P2.7; **V0003 es la primera venta de P3**.

---

## 4. Aislamiento verificado (lo más importante)

| Recurso | ERP (intocable) | POS nuevo (en prueba) |
|---------|-----------------|------------------------|
| Frontend | `rderico-pos-dev` (5000) | Vite (5100) |
| API | `rderico-api-dev` (5001) | `nuevo_pos_api` (5101) |
| Base de datos | `rderico-db-dev` (5433) | `nuevo_pos_db` (5432) |
| Proyecto Compose | `ERP-R-DE-RICO` | `NUEVO-POS` |

- El ERP permaneció **Up 30 horas** durante toda la prueba.
- El árbol de trabajo del ERP quedó **limpio** (sin cambios).
- El commit del ERP no cambió: `b0bc297fbe0ccbe7a9f127e9d2fac983df0383b5`.
- La venta V0003 se escribió **solo** en la base de datos `nuevo_pos`.
- **No se ejecutó ningún comando de Compose dentro de `ERP-R-DE-RICO`.**

---

## 5. Reglas de negocio ejercitadas en vivo

| Regla | Qué se probó | Resultado |
|-------|--------------|-----------|
| RN-10 | Formato de folio `V0003` | ✅ |
| RN-14 | Ciclo de vida OPEN → PAID | ✅ |
| RN-18 | Precio congelado en la línea | ✅ |
| RN-19 | Subtotal de línea (2×8.00=16.00; 1×18.00=18.00) | ✅ |
| RN-16 | Total = suma de subtotales (34.00) | ✅ |
| RN-23 | No modificar un ticket PAID | ✅ 400 |
| RN-27 | Incrementar versión al cobrar (0 → 1) | ✅ |

---

## 6. Conclusión

**P3 queda cerrada.** El POS nuevo:

1. Sirve su frontend en el puerto **5100**.
2. Habla con su **propia API** en el puerto **5101**.
3. Escribe en su **propia base de datos** (`nuevo_pos`).
4. Completa el flujo **E.1 (venta directa)** de punta a punta.
5. **No tocó el ERP** en ningún momento.

La puerta queda abierta para **P4 — Decisión de corte**, que ya no es una tarea técnica
sino una decisión del negocio sobre cuándo migrar la operación real.
