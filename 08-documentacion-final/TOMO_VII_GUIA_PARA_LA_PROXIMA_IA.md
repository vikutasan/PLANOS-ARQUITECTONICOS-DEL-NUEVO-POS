# TOMO VII — GUÍA PARA LA PRÓXIMA IA

> **Qué es este tomo.** No es un manual de usuario ni una descripción del sistema. Es una **carta de navegación** para la IA (o el humano) que retome este proyecto. Aquí está lo que **debes leer antes de tocar una sola línea**, lo que **debes hacer** y, sobre todo, lo que **NO debes hacer** — porque ya se hizo y costó caro.

---

## §0. Cómo leer este tomo

Este tomo se lee **de arriba hacia abajo, una sola vez, antes de empezar**. No es de consulta; es de **inmersión**. Si lo lees por partes, perderás el hilo de por qué cada regla existe.

El orden de lectura de **toda** la documentación final es:

1. **Tomo I — Visión y arquitectura.** El *qué* y el *por qué* del edificio.
2. **Tomo II — Las 95 reglas de negocio.** El *comportamiento* esperado.
3. **Tomo III — Cementerio de bugs y cicatrices.** El *dolor* que produjo cada regla.
4. **Tomo IV — Contratos y fronteras.** El *cómo* se comunican los módulos.
5. **Tomo V — Superficie e interfaces.** El *cómo se ve* y se toca.
6. **Tomo VI — Acta de obra.** El *cómo se construyó*, fase por fase.
7. **Tomo VII — Esta guía.** El *cómo continuar sin romper nada*.

**Regla de oro de la lectura:** no leas el Tomo VII primero. Sin el contexto de los Tomos I–VI, esta guía parecerá una lista arbitraria de prohibiciones. Con el contexto, verás que cada prohibición es una **cicatriz**.

---

## §1. El estado actual del proyecto

**Lo que está construido y cerrado:**

- **Fases 1–11.0** — completas.
- **Fase 12 (F12.1–F12.19)** — completa y pusheada. Es la paridad con el viejo POS.
- **Fase 13 (F13.0, F13.1, F13.2a)** — completa y pusheada. Es la auditoría y el control.
- **F13.2b** — **NO procede.** El panel de auditoría pertenece al módulo de Auditoría y Control del ERP, no al POS.
- **Reorganización de módulos** — completa y pusheada (`26f3bfa`).
- **Documentación final (F11)** — los 7 tomos.

**Los números reales (memorízalos, porque los planos viejos mienten):**

| Concepto | Número real | Nota |
|----------|-------------|------|
| Contratos | **31** | No 29. Los planos viejos decían 29. |
| Interfaces | **24** | No 26. Los planos viejos decían 26. |
| Reglas de negocio | **95** | RN-01 a RN-95. |
| Fichas | **84** | Repartidas en 14 fases (F0–F13). |
| Fases | **14** | F0 a F13. |

**El residuo de nombres:** la tupla de reglas se llama `LAS_81_REGLAS` pero **contiene 95 reglas**. Esto es un **residuo histórico** (cuando había 81). **NO lo renombres** sin actualizar todos los imports; el nombre es feo pero inofensivo. Está documentado en el Tomo II.

---

## §2. Las 3 reglas que nunca debes romper

Si solo recuerdas tres cosas de este tomo, que sean estas:

### Regla 1 — El POS es el primer módulo de un ERP reconstruido

No estás construyendo "un POS". Estás construyendo **el primer módulo** de un ERP que se rehará módulo por módulo. Cada decisión que tomes en el POS **sienta precedente** para los demás módulos. Si el POS lee una tabla ajena, el módulo de Estadísticas también lo hará. Si el POS silencia una excepción, el módulo de Almacenes también lo hará.

**Consecuencia práctica:** cuando dudes entre "rápido y sucio" y "correcto y un poco más lento", elige **correcto**. El POS es el **modelo**.

### Regla 2 — El ERP viejo es el ORÁCULO, no el modelo

El ERP viejo (`ERP-R-DE-RICO`) existe y funciona. Pero su arquitectura es **parchada**. Es un **oráculo**: te dice **qué** debe hacer el negocio (las reglas, los flujos, los casos borde). **NO** te dice **cómo** implementarlo.

**Consecuencia práctica:** lee el viejo ERP para **entender el negocio**, nunca para **copiar el código**. La UX se hereda en su **integración**; la implementación se **reescribe**.

### Regla 3 — Un fallo ajeno nunca bloquea una venta (DT-07)

Un fallo de CRM, Notificaciones, IA o Estadísticas **jamás** bloquea una venta. El POS **degrada** ("sin beneficios", "sin envío") y **cobra igual**.

**Consecuencia práctica:** todo lo que no sea el núcleo transaccional (ticket, pago, caja) debe estar **detrás de una frontera** que absorba su fallo. Si tu código nuevo puede tumbar una venta cuando falla algo externo, **está mal**.

---

## §3. Las 6 prohibiciones absolutas

Estas prohibiciones nacieron del **cementerio de bugs** (Tomo III). Cada una costó un incidente real.

1. **Prohibido leer tablas ajenas.** El consumidor pide por **operación** (contrato), nunca lee la tabla del proveedor. (A-02 / O-23)
2. **Prohibido exponer una tabla en un contrato.** El campo `tabla_expuesta` de todo `Contrato` es **siempre** `None`. Existe para afirmarlo en el test de la puerta F2.
3. **Prohibido enviar directamente.** El POS **encola** en el Outbox dentro de la transacción; el worker envía después. (Regla de Oro #7, RN-85, RN-86)
4. **Prohibido silenciar una excepción en la ruta crítica.** Un `except: pass` en el camino de la venta es un bug esperando a ocurrir. (E-05)
5. **Prohibido hardcodear un offset de zona horaria.** Se usa la utilidad central; se **declara**, no se **convierte**. (RN-81, DT-02, DT-06)
6. **Prohibido avanzar de fase con la puerta en rojo.** Si la puerta falla, se corrige la fase actual; no se avanza "dejando pendiente".

---

## §4. Las 10 reglas de batalla

Estas son las reglas que **guían el día a día** de la construcción. No son prohibiciones; son **hábitos**.

1. **De adentro hacia afuera.** Primero el núcleo (datos, reglas), luego los guardianes, luego la superficie. Nunca al revés.
2. **Cada regla de negocio se porta CON su test.** (A-01) No hay regla sin prueba.
3. **La integración es una compuerta.** Un componente puede existir y no estar cableado. Verifica el cableado. (Lección F4.5)
4. **La completitud es una compuerta.** El conjunto debe estar completo, no solo cada pieza. (Lección F10)
5. **La paridad se verifica, no se declara.** Audita (F10), construye (F12), exige el test (A-01).
6. **El contrato es estable; la tabla es libre.** Puedes cambiar la tabla sin romper a nadie; no puedes cambiar el contrato sin avisar.
7. **Un módulo es dueño de sus tablas.** Nadie más escribe en ellas. (P-01)
8. **El consumidor pregunta por operación.** Nunca por tabla. (P-02)
9. **La IA es asistiva, nunca bloqueante.** Si la visión falla, el cajero cobra manual. (RN-74)
10. **La UX se hereda en su integración, se reescribe en su implementación.** Copia el flujo, no el código.

---

## §5. Las 5 cicatrices que debes conservar

Una **cicatriz** es un guardián nacido de un bug real ya pagado. Se **conserva** y se **porta con su test**. No la elimines "porque ya no pasa".

1. **Draft Guard** — impide que una terminal escriba el draft de otra. (RN-31, RN-32, RN-33)
2. **Anti-degradación** — rechaza una operación que reduce el total más del 50%. (RN-37)
3. **Bloqueo optimista** — un `version` obsoleto responde 409 y no escribe. (RN-25, RN-26)
4. **Reciclaje de folios** — el folio se recicla correctamente al cancelar. (RN-10)
5. **Idempotencia de emergencia** — 2× POST del mismo `item_id` deja el ticket en el mismo estado.

**La diferencia entre cicatriz y deuda:**
- **Cicatriz** = guardián nacido de un bug real → **SE CONSERVA**.
- **Deuda** = atajo que esconde un fallo o acopla módulos → **SE ELIMINA**.

---

## §6. Las 5 lecciones de integración

Estas lecciones son las **más caras** del proyecto. Nacieron de fases que "parecían terminadas" y no lo estaban.

1. **F4.5 — el paso de INTEGRACIÓN también es una compuerta.** Un componente puede estar bien hecho y **no estar cableado**. El inventario de componentes no lo ve.
2. **F10 — la COMPLETITUD del conjunto también es una compuerta.** Cada pieza puede estar bien y el **conjunto** estar incompleto.
3. **F10.4 — el inventario de componentes no ve las integraciones.** Un componente existe pero no está conectado.
4. **F10.5 — el inventario de componentes no ve los flujos de datos.** Los datos fluyen mal aunque cada componente funcione.
5. **F10.6 — el inventario de componentes no ve la paridad de operación.** La operación del cajero difiere aunque los datos cuadren.

**Consecuencia práctica:** nunca declares una fase terminada basándote solo en "los archivos existen". Verifica **integración**, **flujos** y **operación**.

---

## §7. El protocolo de trabajo

Cuando retomes el proyecto, sigue este protocolo **en orden**:

### Paso 1 — Lee la documentación final completa
Los 7 tomos, en orden. No empieces a codificar antes.

### Paso 2 — Verifica el estado real
- `git log --oneline -20` en ambos repos.
- Levanta el stack Docker (`nuevo_pos_api` en 5101, `nuevo_pos_db` en 5432).
- Corre la CI completa: tests de backend, tests de frontend, y los 7 guards.

### Paso 3 — Identifica la fase actual
Mira el Acta de Obra (Tomo VI). ¿En qué fase estás? ¿Cuál es su **puerta**?

### Paso 4 — Trabaja de adentro hacia afuera
Si la fase es de datos, empieza por los datos. Si es de superficie, empieza por el contrato.

### Paso 5 — Verifica la puerta ANTES de avanzar
No avances hasta que la puerta esté **en verde**. Si falla, corrige la fase actual.

### Paso 6 — Documenta la desviación
Si algo del plan no coincide con la realidad, **documéntalo**. No lo escondas.

---

## §8. Las trampas específicas del entorno

Estas son trampas **técnicas** que ya nos mordieron. No las repitas.

### Trampa 1 — El API no recarga solo
El contenedor `nuevo_pos_api` corre **sin** `--reload`. Si cambias código de backend, **debes** hacer `docker restart nuevo_pos_api`. No esperes a que recargue; no lo hará.

### Trampa 2 — El frontend corre en el host, no en Docker
El servidor de Vite corre en el **host** (puerto 5100), no en Docker. No busques el frontend en un contenedor.

### Trampa 3 — Hay 3 repos, no 1
- `NUEVO-POS` — el código.
- `PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS` — los planos.
- `ERP-R-DE-RICO` — el ERP viejo (**SOLO LECTURA**).

No mezcles commits entre repos. Cada repo tiene su propio `git`.

### Trampa 4 — El ERP viejo es SOLO LECTURA
Nunca escribas en `ERP-R-DE-RICO`. Es el oráculo; se consulta, no se modifica.

### Trampa 5 — Los planos viejos mienten en los números
Los planos viejos dicen 29 contratos y 26 interfaces. Los números reales son **31** y **24**. Confía en el **código** (`registry.py`), no en los planos viejos.

### Trampa 6 — El nombre `LAS_81_REGLAS` contiene 95
No te confundas. El nombre es un residuo. La tupla tiene 95 reglas.

---

## §9. Los 5 greps de CI

Desde el día 1, la CI corre **5 greps** que fallan el build si se violan. Conócelos:

1. **Grep de imports ajenos** — falla si un módulo importa de otro módulo directamente (debe usar el contrato).
2. **Grep de tablas ajenas** — falla si un módulo lee/escribe una tabla que no es suya.
3. **Grep de envío directo** — falla si el POS envía sin pasar por el Outbox.
4. **Grep de excepciones silenciadas** — falla si hay un `except: pass` en la ruta crítica.
5. **Grep de offsets hardcodeados** — falla si hay un offset de zona horaria hardcodeado.

**Consecuencia práctica:** si tu cambio rompe un grep, **no lo desactives**. Arregla el código.

---

## §10. La matriz de trazabilidad

Cada regla de negocio tiene **un test** que la verifica. La matriz `regla → test` está en el Tomo II y se verifica en la puerta de F3.

**Consecuencia práctica:** si agregas una regla de negocio, **agrega su test**. Si agregas un test, **referencia su regla**. La matriz debe estar **completa** siempre.

---

## §11. Qué NO hacer (el resumen ejecutivo)

Si solo lees una sección de este tomo, que sea esta:

- **NO** leas tablas ajenas. Pide por operación.
- **NO** expongas una tabla en un contrato.
- **NO** envíes directamente. Usa el Outbox.
- **NO** silencies excepciones en la ruta crítica.
- **NO** hardcodees offsets de zona horaria.
- **NO** avances de fase con la puerta en rojo.
- **NO** copies código del ERP viejo. Copia el **flujo**.
- **NO** escribas en `ERP-R-DE-RICO`.
- **NO** mezcles commits entre los 3 repos.
- **NO** confíes en los números de los planos viejos.
- **NO** renombres `LAS_81_REGLAS` sin actualizar todos los imports.
- **NO** elimines una cicatriz "porque ya no pasa".
- **NO** declares una fase terminada sin verificar integración, flujos y operación.
- **NO** dejes que un fallo ajeno bloquee una venta.

---

## §12. El objetivo final

El objetivo **no** es "terminar el POS". El objetivo es **blindar contra la repetición de errores**.

Cada regla, cada guardián, cada cicatriz, cada puerta existe por una razón: **que el bug que ya pagamos no vuelva a ocurrir**. La documentación final no es un adorno; es el **sistema inmunológico** del proyecto.

**La medida del éxito:** que la próxima IA (o el próximo humano) pueda retomar el proyecto **sin repetir ninguno de los 18 incidentes** del Tomo III.

---

## Cierre del Tomo VII

Has llegado al final de la documentación final. Si leíste los 7 tomos, ahora sabes:

- **Qué** es el POS (Tomo I).
- **Cómo** se comporta (Tomo II).
- **Por qué** cada regla existe (Tomo III).
- **Cómo** se comunican los módulos (Tomo IV).
- **Cómo** se ve y se toca (Tomo V).
- **Cómo** se construyó (Tomo VI).
- **Cómo** continuar sin romper nada (Tomo VII).

**La última advertencia:** la tentación de "avanzar dejando pendiente" es exactamente lo que produjo los 18 incidentes. La puerta no es burocracia; es la **cicatriz** que impide repetir el bug.

**Construye de adentro hacia afuera. Verifica cada puerta. Documenta cada desviación. Y nunca, nunca dejes que un fallo ajeno bloquee una venta.**
