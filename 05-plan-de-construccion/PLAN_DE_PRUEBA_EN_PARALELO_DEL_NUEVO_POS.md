# PLAN DE PRUEBA EN PARALELO DEL NUEVO POS

> **Estado:** PROPUESTA — pendiente de aprobación.
> **Fecha:** 2026-09-26.
> **Autor:** Arquitecto (modo Debug).
> **Contexto:** El ERP está instalado y **vendiendo ahora mismo**. El POS nuevo
> tiene F0–F6 cerradas (contratos, reglas, guardianes, superficie, consolidación),
> pero **no tiene frontend ejecutable todavía**.

---

## 1. La intención original del usuario

> "En este instante el ERP y por ende el POS viejo están operando, aunque es un
> día de venta tranquilo. ¿Podemos armar un plan para **desacoplar
> momentáneamente el módulo POS actual**, ponerlo a un lado y **acoplar el nuevo
> POS momentáneamente** para probarlo, y si algo sale mal **volver a desacoplar
> el nuevo POS**, ponerlo a un lado y **reacoplar el viejo POS funcional**?"

La intención es legítima y el instinto es correcto: **probar el POS nuevo en
condiciones reales antes de comprometerse**. Lo que sigue documenta por qué el
mecanismo propuesto (swap en caliente) no es viable, y cuál es la vía que sí lo es.

---

## 2. Los obstáculos insalvables

Estos obstáculos **no son opiniones**: se verificaron leyendo el código del ERP y
del POS nuevo.

### O-01 — El POS viejo no es un módulo desmontable; es código dentro del ERP

El POS viejo no es un servicio ni un paquete aislado. Es un conjunto de
componentes React **importados directamente por el shell del ERP**:

- [`main.jsx`](../../ERP-R-DE-RICO/main.jsx:69) monta un único `<ExperimentCenterUI />`.
- [`ExperimentCenterUI.jsx`](../../ERP-R-DE-RICO/apps/ExperimentCenterUI.jsx:22) hace
  `import { RetailVisionPOS } from './pos/RetailVisionPOS'`.
- [`ExperimentCenterUI.jsx`](../../ERP-R-DE-RICO/apps/ExperimentCenterUI.jsx:684) lo
  renderiza cuando `activeModule === 'pos_retail'`.

**Consecuencia:** "desacoplar el POS viejo" significa **editar el ERP** (quitar
imports, quitar el render, tocar el shell). No hay una pieza que se saque sin
cirugía.

### O-02 — El POS nuevo no tiene frontend ejecutable

[`../NUEVO-POS/apps/pos/`](../NUEVO-POS/apps/pos) contiene **solo `.gitkeep`**.
Lo que existe del POS nuevo es **Python** (contratos, reglas, guardianes,
superficie, consolidación) en [`../NUEVO-POS/apps/api/`](../NUEVO-POS/apps/api).

**Consecuencia:** no hay nada que "acoplar" del lado de la UI. El swap no tiene
segundo extremo.

### O-03 — El API del ERP es un monolito de un solo contenedor

El API del ERP es **un solo contenedor** (`rderico-api-dev`, puerto 5001) que
sirve **todos** los módulos: POS, almacenes, RRHH, grandeza, etc.
([`docker-compose.yml`](../../ERP-R-DE-RICO/docker-compose.yml:15)).

**Consecuencia:** no hay un backend exclusivo del POS que se pueda intercambiar.
Apagar el API del POS = apagar el ERP entero.

### O-04 — El POS nuevo tiene su propia base de datos, separada por diseño

El POS nuevo usa `nuevo_pos` en su propio contenedor
([`../NUEVO-POS/docker-compose.yml`](../NUEVO-POS/docker-compose.yml:18)),
**separada** de la del ERP. El POS viejo lee y escribe la BD del ERP.

**Consecuencia:** no basta "repuntar" el frontend. Los dos POS tienen esquemas
distintos (17 tablas nuevas vs. el esquema del ERP). Compartir la BD del ERP
violaría el desacople por diseño que el plano exige.

### O-05 — La regla dura prohíbe tocar el ERP

[`../NUEVO-POS/README.md`](../NUEVO-POS/README.md:11) declara:

> **NO SE TOCA EL ERP INSTALADO Y CORRIENDO.**
> **NO SE TOCA NINGUNO DE SUS MÓDULOS.**
> **NO SE MODIFICA NI UNA LÍNEA DEL POS ACTUAL.**

**Consecuencia:** los obstáculos O-01 y O-03 exigen editar el ERP. La regla dura
lo prohíbe. **El swap en caliente es, por definición, una violación de la regla
dura.**

### O-06 — El ERP está vendiendo ahora

Aunque sea un día tranquilo, hay una caja abierta y clientes en el mostrador. Un
swap en caliente del módulo POS, sin el POS nuevo construido ni probado, es la
forma más segura de tumbar la venta.

---

## 3. Por qué el swap en caliente no se puede ejecutar

| Lo que asume el swap | La realidad | Obstáculo |
|---|---|---|
| "Desacoplar el POS viejo" | Son ~30 archivos `.jsx` importados por el shell del ERP. Sacarlos = editar el ERP. | O-01, O-05 |
| "Acoplar el POS nuevo" | El POS nuevo no tiene UI. No hay componente que montar. | O-02 |
| "Si falla, volver a acoplar el viejo" | Volver atrás exige **re-editar el ERP** otra vez. | O-01, O-05 |
| "Es momentáneo, no pasa nada" | El ERP está vendiendo. Un swap en caliente arriesga la caja. | O-06 |

**Conclusión:** el swap en caliente no es un plan riesgoso; es un plan
**imposible** sin violar la regla dura. No se puede "probar y volver" cuando
probar exige tocar lo que está prohibido tocar.

---

## 4. La vía que sí respeta la regla dura: prueba en paralelo aislada

El POS nuevo se levanta **al lado** del ERP, en su propio puerto y su propia BD,
sin que el ERP se entere. El ERP sigue vendiendo en 5000/5001. El POS nuevo se
prueba con **datos de prueba** (nunca con datos reales de venta).

### Principio rector

> **El POS nuevo no toca al ERP. El ERP no sabe que el POS nuevo existe.**
> Se prueban en paralelo, en espacios separados, con datos separados.

### Arquitectura de la prueba en paralelo

```
┌─────────────────────────────────────────────────────────────┐
│  HOST (la máquina que ya corre el ERP)                       │
│                                                              │
│  ┌──────────────────────┐      ┌──────────────────────┐     │
│  │  ERP (INTOCABLE)     │      │  POS NUEVO (prueba)  │     │
│  │                      │      │                      │     │
│  │  pos:5000            │      │  pos:5100            │     │
│  │  api:5001            │      │  api:5101            │     │
│  │  db:5433 (ERP)       │      │  db:5434 (nuevo_pos) │     │
│  │                      │      │                      │     │
│  │  BD: erp_rderico     │      │  BD: nuevo_pos       │     │
│  │  Datos: REALES       │      │  Datos: DE PRUEBA    │     │
│  └──────────────────────┘      └──────────────────────┘     │
│         ▲                              ▲                     │
│         │                              │                     │
│    Caja real                     Prueba aislada              │
│    (no se toca)                  (se puede romper)           │
└─────────────────────────────────────────────────────────────┘
```

**Puntos de separación (los 4 que garantizan el aislamiento):**

| Recurso | ERP (intocable) | POS nuevo (prueba) |
|---|---|---|
| Puerto frontend | 5000 | **5100** |
| Puerto API | 5001 | **5101** |
| Puerto BD | 5433 | **5434** |
| Base de datos | `erp_rderico` | `nuevo_pos` |
| Datos | Reales | **De prueba (seed)** |
| Contenedores | `rderico-*` | `nuevo_pos_*` |

Ninguno de estos valores colisiona con el ERP. Los contenedores del POS nuevo
tienen nombres propios (`nuevo_pos_db`, `nuevo_pos_api`) y no comparten red ni
volumen con el ERP.

---

## 5. El plan, por etapas

### Etapa P0 — Preparación (sin tocar el ERP)

**Objetivo:** dejar el POS nuevo listo para levantarse aislado.

| Paso | Acción | Verificación |
|---|---|---|
| P0.1 | Confirmar que el ERP está en verde y anotar su HEAD | `git -C ../ERP-R-DE-RICO rev-parse HEAD` |
| P0.2 | Confirmar que el árbol del ERP está limpio | `git -C ../ERP-R-DE-RICO status --short` (vacío) |
| P0.3 | Ajustar los puertos del POS nuevo a 5100/5101/5434 | `docker compose config` sin colisión |
| P0.4 | Verificar que la BD del POS nuevo es `nuevo_pos`, separada | `docker compose config` |
| P0.5 | Correr la suite completa del POS nuevo (192 tests) | `docker compose exec api pytest -q` → 192 passed |
| P0.6 | Correr la puerta F0 del POS nuevo | `npm run ci` → PUERTA F0 EN VERDE |

**Puerta P0:** el POS nuevo pasa sus 192 tests y su CI, con puertos que no
colisionan con el ERP. **El ERP no se tocó.**

### Etapa P1 — Levantar el POS nuevo en paralelo

**Objetivo:** el POS nuevo corre aislado, con datos de prueba.

| Paso | Acción | Verificación |
|---|---|---|
| P1.1 | Levantar la BD del POS nuevo | `docker compose up -d db` (contenedor `nuevo_pos_db`) |
| P1.2 | Aplicar las migraciones (17 tablas) | `docker compose run --rm api alembic upgrade head` |
| P1.3 | Sembrar datos de prueba (productos, categorías, un empleado) | script de seed (a crear) |
| P1.4 | Levantar el API del POS nuevo | `docker compose up -d api` (contenedor `nuevo_pos_api`) |
| P1.5 | Verificar que el ERP sigue vendiendo | abrir `http://<host>:5000` → POS viejo responde |

**Puerta P1:** el POS nuevo tiene su BD con datos de prueba y su API arriba; el
ERP sigue operando en 5000/5001. **Los dos coexisten sin tocarse.**

### Etapa P2 — Construir el frontend mínimo del POS nuevo

**Objetivo:** tener algo que probar. **Esta etapa es la más grande y hoy no
existe.**

El POS nuevo no tiene UI. Para probarlo hay que construir, como mínimo, el flujo
E.1 (venta directa) de la superficie F5:

| Paso | Acción | Base |
|---|---|---|
| P2.1 | Montar el esqueleto React + Vite del POS nuevo | stack del plano (React 18 + Vite + Tailwind) |
| P2.2 | Implementar `RetailVisionPOS` (pantalla raíz) | [`superficie/registry.py`](../NUEVO-POS/apps/api/superficie/registry.py:185) interfaz 1 |
| P2.3 | Implementar `ProductGrid`, `ProductCard`, `CategoryBar` | interfaces 19–21 del registro |
| P2.4 | Implementar `SalesReceipt` y `CheckoutScreen` | interfaces 8 y 15 del registro |
| P2.5 | Conectar el frontend al API del POS nuevo (5101) | `CONFIG.API_BASE_URL` propio |
| P2.6 | Aplicar los 3 modos y la paleta canónica | reglas R-01..R-04 de F5 |

**Puerta P2:** el flujo E.1 (venta directa) funciona de punta a punta contra la
BD de prueba del POS nuevo. **Sigue sin tocar el ERP.**

### Etapa P3 — Prueba en paralelo

**Objetivo:** operar el POS nuevo como si fuera real, con datos de prueba, al
lado del ERP.

| Paso | Acción | Verificación |
|---|---|---|
| P3.1 | Abrir el POS nuevo en `http://<host>:5100` | carga la pantalla raíz |
| P3.2 | Hacer una venta de prueba de punta a punta | ticket PAID en la BD `nuevo_pos` |
| P3.3 | Verificar que la venta **no** aparece en el ERP | consultar el ERP → no hay ese ticket |
| P3.4 | Verificar que el ERP sigue vendiendo en paralelo | caja real intacta |
| P3.5 | Probar el corte de caja del POS nuevo | sesión cerrada en `nuevo_pos` |
| P3.6 | Probar la consolidación (F6) contra un central de prueba | outbox → payload → central |

**Puerta P3:** el POS nuevo completa un ciclo de venta + corte + consolidación
con datos de prueba, **sin que el ERP se entere**. Si algo se rompe, se rompe
solo el POS nuevo; el ERP sigue vendiendo.

### Etapa P4 — Decisión de corte (acto futuro, no de este plan)

**Objetivo:** decidir si el POS nuevo está listo para reemplazar al viejo.

Esta etapa **no se ejecuta ahora**. Se ejecuta cuando P3 esté verde y el POS
nuevo tenga paridad funcional completa (los 6 flujos E.1–E.6). El corte real es
un **acto de despliegue planificado**, con ventana de mantenimiento, respaldo
previo y plan de reversa — no un swap en caliente.

---

## 6. Plan de reversa (por qué aquí sí existe)

En el swap en caliente, la reversa era "re-editar el ERP" (violando la regla
dura). En la prueba en paralelo, la reversa es trivial y **no toca el ERP**:

| Si algo sale mal en… | Reversa | ¿Toca el ERP? |
|---|---|---|
| P1 (levantar POS nuevo) | `docker compose down` del POS nuevo | **No** |
| P2 (construir frontend) | `git checkout` del POS nuevo | **No** |
| P3 (prueba en paralelo) | `docker compose down` del POS nuevo | **No** |
| P4 (corte real) | Plan de reversa dedicado (respaldo + ventana) | Sí, pero planificado |

**La clave:** en P1–P3, apagar el POS nuevo es un `docker compose down`. El ERP
nunca supo que existía. No hay nada que "reacoplar" porque nunca se desacopló
nada.

---

## 7. Qué opino (evaluación honesta)

**Tu instinto es correcto; el mecanismo no.** Quieres probar antes de
comprometerte, y eso es exactamente lo que hay que hacer. Pero el swap en
caliente confunde "probar" con "reemplazar":

- **Probar** = levantar el nuevo al lado, con datos de prueba, sin tocar el viejo.
- **Reemplazar** = apagar el viejo y encender el nuevo, con datos reales.

El swap en caliente intenta **reemplazar para probar**, que es lo peor de ambos
mundos: arriesga la venta real y viola la regla dura, sin ganar nada que la
prueba en paralelo no dé.

**Lo que sí conviene hacer, en orden:**

1. **Ahora:** levantar el POS nuevo en paralelo (P0–P1). Es barato, no toca el
   ERP, y valida que el andamiaje funciona en la máquina real.
2. **Después:** construir el frontend mínimo (P2). Es el trabajo grande y hoy no
   existe. Sin esto, no hay nada que probar.
3. **Luego:** probar el flujo completo con datos de prueba (P3).
4. **Al final:** decidir el corte real (P4), con ventana y respaldo.

**El obstáculo real no es el ERP; es que el POS nuevo no tiene UI.** El swap en
caliente es una solución a un problema que no tenemos (el ERP no estorba para
probar). El problema que sí tenemos es que falta construir la superficie.

**Recomendación:** aprobar P0–P1 ahora (barato, seguro, valida el andamiaje) y
tratar P2 como el siguiente bloque de construcción del POS nuevo. El corte real
(P4) se decide cuando P3 esté verde.

---

## 8. Resumen ejecutivo

| Pregunta | Respuesta |
|---|---|
| ¿Se puede hacer el swap en caliente? | **No.** Viola la regla dura y arriesga la venta. |
| ¿Por qué? | El POS viejo no es desmontable sin editar el ERP (O-01), el POS nuevo no tiene UI (O-02), el API es monolito (O-03), las BD están separadas (O-04), la regla dura lo prohíbe (O-05), el ERP está vendiendo (O-06). |
| ¿Qué sí se puede? | **Prueba en paralelo aislada:** POS nuevo en su propio stack (API sin puerto publicado, BD en 5432), BD `nuevo_pos`, datos de prueba, cero contacto con el ERP. |
| ¿Cuál es el obstáculo real? | El POS nuevo **no tiene frontend**. Hay que construirlo (P2). |
| ¿Cuál es el plan de reversa? | `docker compose down` del POS nuevo. El ERP nunca se tocó. |
| ¿Qué propongo? | Aprobar P0–P1 ahora; tratar P2 como el siguiente bloque de construcción; decidir el corte real (P4) cuando P3 esté verde. |

---

## 9. Trazabilidad

| Afirmación | Fuente verificada |
|---|---|
| El POS viejo se importa en el shell del ERP | [`ExperimentCenterUI.jsx`](../../ERP-R-DE-RICO/apps/ExperimentCenterUI.jsx:22) |
| El POS viejo se renderiza por `activeModule` | [`ExperimentCenterUI.jsx`](../../ERP-R-DE-RICO/apps/ExperimentCenterUI.jsx:684) |
| El API del ERP es un monolito de un contenedor | [`docker-compose.yml`](../../ERP-R-DE-RICO/docker-compose.yml:15) |
| El POS nuevo tiene BD separada | [`../NUEVO-POS/docker-compose.yml`](../NUEVO-POS/docker-compose.yml:18) |
| El POS nuevo no tiene frontend | [`../NUEVO-POS/apps/pos/`](../NUEVO-POS/apps/pos) (solo `.gitkeep`) |
| La regla dura prohíbe tocar el ERP | [`../NUEVO-POS/README.md`](../NUEVO-POS/README.md:11) |
| El POS nuevo tiene 192 tests en verde | suite F1–F6 del POS nuevo |
| La superficie F5 tiene 24 interfaces | [`superficie/registry.py`](../NUEVO-POS/apps/api/superficie/registry.py:1) |

---

## 10. Registro de ejecución (P0–P1)

> **Estado:** P0 y P1 **EJECUTADAS Y VERIFICADAS** el 2026-09-26.
> **Regla dura respetada:** el ERP no se tocó en ningún momento.

### 10.1 P0 — Preparación

| Paso | Acción | Resultado verificado |
|---|---|---|
| P0.1 | Anotar el HEAD del ERP | `b0bc297fbe0ccbe7a9f127e9d2fac983df0383b5`, rama `feat/responsive-productos` |
| P0.2 | Confirmar árbol limpio | `git status --short` vacío (sin cambios) |
| P0.3 | Revisar colisión de puertos | **No hay colisión.** ERP: 5000/5001/5433. POS nuevo: 5432. No se cambió nada. |
| P0.4 | Confirmar aislamiento de la BD | `nuevo_pos` en red `nuevo-pos_default`, volumen `nuevo-pos_nuevo_pos_data` |
| P0.5 | Correr la suite completa del POS nuevo | `docker compose exec api pytest -q` → **192 passed in 1.71s** |
| P0.6 | Correr la puerta F0 del POS nuevo | `npm run ci` → **PUERTA F0 EN VERDE** (lint 0 errores / 81 archivos, 6 tests OK, 7 greps limpios) |

### 10.2 P1 — Levantar el POS nuevo en paralelo

| Paso | Acción | Resultado verificado |
|---|---|---|
| P1.1 | BD arriba | `nuevo_pos_db` — `Up 10 hours (healthy)`, `0.0.0.0:5432->5432/tcp` |
| P1.2 | Migraciones aplicadas | 18 tablas (17 de dominio + `alembic_version`) |
| P1.3 | Sembrar datos de prueba | [`scripts/seed_demo.py`](../NUEVO-POS/apps/api/scripts/seed_demo.py:1) → 3 categorías, 6 productos, 1 sesión de terminal. **Idempotente** (2ª corrida: 0 productos nuevos). |
| P1.4 | API arriba | `nuevo_pos_api` — `Up 10 hours` (sin puerto publicado) |
| P1.5 | Verificar que el ERP sigue vendiendo | HEAD intacto `b0bc297`, árbol limpio, contenedores `Up 28 hours` (pos 5000, api 5001, db 5433) |

### 10.3 Corrección al plan original

El plan original (P0.3) asumía que había que **cambiar los puertos del POS nuevo a
5100/5101/5434**. La verificación con `docker ps` demostró que **no hay colisión**:
el ERP usa 5000/5001/5433 y el POS nuevo usa 5432. Cambiar puertos habría sido
trabajo innecesario. **No se cambió nada.** Esta corrección se documenta aquí
porque el plan es un instrumento vivo, no un dogma.

### 10.4 Estado del entorno tras P0–P1

```
nuevo_pos_api       Up 10 hours
nuevo_pos_db        Up 10 hours (healthy)   0.0.0.0:5432->5432/tcp
rderico-ia-local    Up 28 hours (healthy)   127.0.0.1:9000->9000/tcp
rderico-api-dev     Up 28 hours             0.0.0.0:5001->3001/tcp
rderico-ia-ollama   Up 28 hours             11434/tcp
rderico-pos-dev     Up 28 hours             0.0.0.0:5000->3000/tcp
rderico-db-dev      Up 28 hours             0.0.0.0:5433->5432/tcp
```

**Conclusión:** el POS nuevo corre en paralelo, aislado, con datos de prueba, y el
ERP sigue vendiendo sin haber sido tocado. El siguiente bloque es **P2 — construir
el frontend mínimo**, que es el único obstáculo real que queda.
