# 📚 DOCUMENTACIÓN FINAL DEL NUEVO POS — Índice Maestro

> **Qué es esto.** La documentación final del POS nuevo de "R de Rico". No es un manual de usuario; es el **acta de construcción** del edificio: qué se construyó, por qué, cómo se comunica, cómo se ve y cómo continuar sin romper nada.
>
> **Para quién es.** Para la próxima IA (o el próximo humano) que retome el proyecto. Se lee **una vez, de arriba hacia abajo**, antes de tocar una sola línea.

---

## 1. Los 7 tomos

| # | Tomo | Qué contiene | Cuándo leerlo |
|---|------|--------------|---------------|
| I | [Visión y arquitectura](./TOMO_I_VISION_Y_ARQUITECTURA.md) | El *qué* y el *por qué* del edificio. Objetivo central, reglas duras, prohibiciones, estándares, frontera por contratos. | Primero. Da el contexto. |
| II | [Las 95 reglas de negocio](./TOMO_II_LAS_95_REGLAS_DE_NEGOCIO.md) | El *comportamiento* esperado. RN-01 a RN-95, con su categoría y su test. | Segundo. Define el comportamiento. |
| III | [Cementerio de bugs y cicatrices](./TOMO_III_CEMENTERIO_DE_BUGS_Y_CICATRICES.md) | El *dolor* que produjo cada regla. 18 incidentes, 21 reglas de oro, 5 cicatrices, 6 prohibiciones, 11 reglas de batalla. | Tercero. Explica el por qué. |
| IV | [Contratos y fronteras](./TOMO_IV_CONTRATOS_Y_FRONTERAS.md) | El *cómo* se comunican los módulos. Los 31 contratos, DT-07, el patrón Outbox, la puerta F2. | Cuarto. Define las fronteras. |
| V | [Superficie e interfaces](./TOMO_V_SUPERFICIE_E_INTERFACES.md) | El *cómo se ve* y se toca. Las 24 interfaces, la paleta canónica, las 4 reglas duras, los 3 modos de layout, los 6 flujos. | Quinto. Define la superficie. |
| VI | [Acta de obra](./TOMO_VI_ACTA_DE_OBRA.md) | El *cómo se construyó*. Las 84 fichas, las 14 fases (F0–F13), cada una con su puerta de salida. | Sexto. Define la construcción. |
| VII | [Guía para la próxima IA](./TOMO_VII_GUIA_PARA_LA_PROXIMA_IA.md) | El *cómo continuar sin romper nada*. Reglas que nunca romper, prohibiciones, cicatrices, protocolo de trabajo, trampas del entorno. | Séptimo. Es la carta de navegación. |

---

## 2. El orden de lectura

**Lee los tomos en orden, una sola vez, antes de empezar.** No es de consulta; es de **inmersión**.

```
I → II → III → IV → V → VI → VII
```

**Por qué el orden importa:** el Tomo VII (la guía) parece una lista arbitraria de prohibiciones si se lee primero. Leído al final, cada prohibición es una **cicatriz** con su historia.

---

## 3. Los números reales del proyecto

**Memorízalos. Los planos viejos mienten.**

| Concepto | Número real | Nota |
|----------|-------------|------|
| Contratos | **31** | No 29. Los planos viejos decían 29. |
| Interfaces | **24** | No 26. Los planos viejos decían 26. |
| Reglas de negocio | **95** | RN-01 a RN-95. La tupla se llama `LAS_81_REGLAS` (residuo histórico). |
| Fichas | **84** | Repartidas en 14 fases (F0–F13). |
| Fases | **14** | F0 a F13. |
| Incidentes documentados | **18** | En el Tomo III. |
| Reglas de oro | **21** | En el Tomo III. |
| Cicatrices | **5** | En el Tomo III. |
| Prohibiciones absolutas | **6** | En el Tomo III. |
| Reglas de batalla | **11** | En el Tomo III. |
| Greps de CI | **5** | Desde el día 1. |

**La fuente de verdad es el código, no los planos viejos:**
- Contratos: [`NUEVO-POS/apps/api/contracts/registry.py`](../NUEVO-POS/apps/api/contracts/registry.py)
- Interfaces: [`NUEVO-POS/apps/api/superficie/registry.py`](../NUEVO-POS/apps/api/superficie/registry.py)
- Reglas: [`NUEVO-POS/apps/api/rules/registry.py`](../NUEVO-POS/apps/api/rules/registry.py)

---

## 4. Las 3 reglas que nunca debes romper

1. **El POS es el primer módulo de un ERP reconstruido.** Cada decisión sienta precedente. Cuando dudes, elige **correcto**, no **rápido**.
2. **El ERP viejo es el ORÁCULO, no el modelo.** Te dice **qué** debe hacer el negocio; **no** te dice **cómo** implementarlo.
3. **Un fallo ajeno nunca bloquea una venta (DT-07).** El POS degrada y cobra igual.

---

## 5. Las 6 prohibiciones absolutas

1. Prohibido leer tablas ajenas. (A-02 / O-23)
2. Prohibido exponer una tabla en un contrato. (`tabla_expuesta` siempre `None`)
3. Prohibido enviar directamente. (Usa el Outbox: RN-85, RN-86)
4. Prohibido silenciar una excepción en la ruta crítica. (E-05)
5. Prohibido hardcodear un offset de zona horaria. (RN-81)
6. Prohibido avanzar de fase con la puerta en rojo.

---

## 6. Las 14 fases y sus puertas

| Fase | Nombre | Puerta de salida |
|------|--------|------------------|
| F0 | Andamiaje | CI en verde con 0 tests + 5 greps activos |
| F1 | Cimiento de datos | Migraciones aplican y revierten limpias |
| F2 | Frontera (contratos) | Test de arquitectura: 0 imports ajenos |
| F3 | Comportamiento (reglas) | Matriz `regla → test` completa |
| F4 | Guardianes | CI falla si se viola una regla crítica |
| F5 | Superficie (interfaces) | Paridad funcional con el POS actual |
| F6 | Consolidación (central) | Sync de cierre de día verificada |
| F7 | Voz + Visión IA + Temas | Los 3 modos de topología IA operan sin bloquear la venta |
| F8 | CRM y Notificaciones | Beneficios + envío por Outbox sin bloquear la venta |
| F9 | Rescate de UX + Pagos mixtos | UX equivalente + pagos mixtos cuadran |
| F10 | Auditoría de Paridad | La completitud del conjunto está verificada |
| F11 | Verificación Plan vs. Realidad | El plan y la realidad coinciden |
| F12 | Paridad con el viejo POS | Cada funcionalidad portada tiene paridad + test |
| F13 | Auditoría y Control | Cada escritura queda registrada y consultable |

**La regla de avance:** ninguna fase empieza sin que la **puerta** de la anterior esté en verde.

---

## 7. Criterios de aceptación de la documentación final

La documentación final se considera **completa** cuando:

- [x] **Tomo I** — Visión y arquitectura (§1–§16).
- [x] **Tomo II** — Las 95 reglas de negocio (§0–§21).
- [x] **Tomo III** — Cementerio de bugs y cicatrices (§0–§12).
- [x] **Tomo IV** — Contratos y fronteras (§0–§9, 31 contratos).
- [x] **Tomo V** — Superficie e interfaces (§0–§11, 24 interfaces).
- [x] **Tomo VI** — Acta de obra (§0–§18 + Cierre, 84 fichas).
- [x] **Tomo VII** — Guía para la próxima IA (§0–§12 + Cierre).
- [x] **README índice maestro** — este documento.
- [ ] **Commit/push** en el repo de planos.

**El principio rector:** el **POR QUÉ** de cada decisión. No basta con documentar **qué** se hizo; hay que documentar **por qué**, para que la próxima IA no lo deshaga "porque no entendía".

---

## 8. Los 3 repositorios

| Repo | Qué es | Permiso |
|------|--------|---------|
| `NUEVO-POS` | El código del POS nuevo. | Lectura/escritura. |
| `PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS` | Los planos y esta documentación. | Lectura/escritura. |
| `ERP-R-DE-RICO` | El ERP viejo (el oráculo). | **SOLO LECTURA.** |

**No mezcles commits entre repos.** Cada repo tiene su propio `git`.

---

## 9. El objetivo final

El objetivo **no** es "terminar el POS". El objetivo es **blindar contra la repetición de errores**.

Cada regla, cada guardián, cada cicatriz, cada puerta existe por una razón: **que el bug que ya pagamos no vuelva a ocurrir**. Esta documentación es el **sistema inmunológico** del proyecto.

**La medida del éxito:** que la próxima IA pueda retomar el proyecto **sin repetir ninguno de los 18 incidentes** del Tomo III.

---

## 10. Por dónde empezar

Si eres la próxima IA y acabas de llegar:

1. **Lee los 7 tomos, en orden.** No empieces a codificar antes.
2. **Verifica el estado real:** `git log` en ambos repos, levanta el stack Docker, corre la CI completa.
3. **Identifica la fase actual** en el Acta de Obra (Tomo VI).
4. **Trabaja de adentro hacia afuera.**
5. **Verifica la puerta ANTES de avanzar.**
6. **Documenta cada desviación.**

**La última advertencia:** la tentación de "avanzar dejando pendiente" es exactamente lo que produjo los 18 incidentes. La puerta no es burocracia; es la **cicatriz** que impide repetir el bug.
