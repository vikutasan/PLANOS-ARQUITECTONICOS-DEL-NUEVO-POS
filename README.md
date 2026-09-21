# PLANOS ARQUITECTÓNICOS DEL NUEVO POS

> **Repositorio de planos. No de obras.**

Este repositorio contiene los **planos arquitectónicos** del nuevo POS del negocio
(panadería "R de Rico"), diseñados para ser replicados en futuras sucursales.

---

## REGLA DURA (INVIOLABLE)

> **NO SE TOCA EL ERP INSTALADO Y CORRIENDO.**
> **NO SE TOCA NINGUNO DE SUS MÓDULOS.**
> **NO SE MODIFICA NI UNA LÍNEA DEL POS ACTUAL.**

Este es un **trabajo alterno**, en un **proyecto nuevo**, en un **repositorio nuevo**.
El ERP actual (`ERP-R-DE-RICO-CON-POS-SIMPLIFICADO`) permanece **intacto y operando**.

Este repositorio contiene **solo documentos**: planos, especificaciones, contratos y
decisiones de arquitectura. **Cero código de producción.** Cero dependencias. Cero
riesgo para el negocio en marcha.

---

## ⚠️ EMPIEZA AQUÍ

Si vas a meter mano en este proyecto, **lee primero la guía maestra**:

> ### 📘 [GUÍA MAESTRA PARA COLABORADORES](./ESPECIFICACIONES%20DEL%20PROYECTO/GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md)
>
> Es el **punto de entrada único**. Te dice qué leer, en qué orden, y qué reglas
> nunca romper. Si solo vas a leer un documento, lee ese.

Si lo que quieres es **aplicar este mismo método a otro módulo del ERP** (Almacenes,
Productos, RRHH, Estadísticas, Auditoría, Heladería, Reparto, Monitoreo de Red,
Pedidos, Perfiles), lee la metodología:

> ### 🧭 [METODOLOGÍA DE INGENIERÍA INVERSA Y DISEÑO](./METODOLOGIA_DE_INGENIERIA_INVERSA_Y_DISENO.md)
>
> Es el **manual de obra reutilizable**: las 6 fases (ingeniería inversa → diseño →
> plan de acción → despliegue → punto de entrada → autocrítica), la checklist maestra
> por módulo, y los 8 errores que no se deben cometer. El POS fue el primer módulo
> construido con este método; los demás se construyen igual.

Y si vas a construir **cualquier pantalla del Nuevo POS**, lee antes la especificación
responsiva. Es una **regla dura**: ningún componente se acepta si no la cumple.

> ### 📱 [ESPECIFICACIÓN RESPONSIVA Y ERGONOMÍA TÁCTIL](./ESPECIFICACION_RESPONSIVA_Y_ERGONOMIA_TACTIL.md)
>
> Define los **3 modos de layout** (Mostrador / Compacto / Móvil), las **4 reglas duras**
> (R-01 a R-04), el **estándar táctil de 44×44px**, y el inventario de contenedores de
> ancho fijo a parametrizar. **La estética del POS v22 se preserva al 100%; la ergonomía
> del mostrador es intocable.**

Y si vas a reconstruir **cualquier interfaz del POS** (pantalla, modal, panel, overlay,
composición o plantilla de impresión), lee antes el mapa visual y táctil. Documenta las
**26 interfaces reales** con una ficha de 7 puntos cada una.

> ### 🖥️ [ESPECIFICACIÓN DE INTERFACES DEL POS](./ESPECIFICACION_DE_INTERFACES_POS.md)
>
> El **mapa visual y táctil** del Punto de Venta. Documenta las 26 interfaces (7 pantallas
> raíz + 6 modales + 5 paneles/overlays + 4 composición + 2 impresión) con una ficha de
> 7 puntos: propósito, estructura visual, controles, estados, navegación, modo responsivo
> y anclaje al código. Cierra el círculo: el backend dice **qué calcula**, este documento
> dice **cómo se ve y cómo se toca**.

Y si vas a **arreglar el ERP módulo por módulo**, lee antes el compendio de directrices
transversales. Son las reglas que **no pertenecen a un módulo**: pertenecen a todos.

> ### 🧱 [DIRECTRICES TRANSVERSALES DEL ERP](./DIRECTRICES_TRANSVERSALES_DEL_ERP.md)
>
> El **compendio de reglas que atraviesan todos los módulos**: Tiempo (DT-01), Dinero
> (DT-02), Identidad (DT-03), Inventario (DT-04) y Auditoría (DT-05). Cada directriz trae
> su regla, su ancla al código, su verificación y una **matriz de cumplimiento por módulo**.
> Una directriz transversal no se "aplica" a un módulo: se **verifica** contra un módulo.

---

## ¿Qué es este repositorio?

El POS actual es un edificio funcional al que se le fueron añadiendo habitaciones,
tuberías e instalaciones sobre la marcha. Funciona. Pero cada nueva habitación tuvo que
conectarse a las tuberías que ya existían, y esas tuberías no fueron diseñadas para esa
carga.

Este repositorio es el **plano** que un arquitecto dibujaría para las **futuras
sucursales**: la misma funcionalidad del edificio actual, pero con las cimentaciones,
las instalaciones y las tuberías puestas de forma elegante desde un principio.

**No es para demoler el edificio actual.** Es para no repetir sus accidentes en los
edificios nuevos.

---

## La topología (hub-and-spoke)

**No habrá una operación multisucursal.** Lo que se hará es:

> **Instalar un ERP completo (con el POS incluido como uno de sus módulos) en cada
> sucursal, y cada ERP enviará data a un servidor central del corporativo.**

```
        ┌──────────────────────────────────┐
        │   SERVIDOR CENTRAL CORPORATIVO   │
        │   (consolidación / reportería)   │
        │   NO transaccional               │
        └──────────────────────────────────┘
              ▲          ▲          ▲
    ┌─────────┘          │          └─────────┐
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│ SUCURSAL A  │   │ SUCURSAL B  │   │ SUCURSAL C  │
│ ERP COMPLETO│   │ ERP COMPLETO│   │ ERP COMPLETO│
│  ├─ POS     │   │  ├─ POS     │   │  ├─ POS     │
│  └─ BD PROPIA│  │  └─ BD PROPIA│  │  └─ BD PROPIA│
└─────────────┘   └─────────────┘   └─────────────┘
```

**Las 3 implicaciones clave:**

1. **Cada sucursal es independiente.** Tiene su propio ERP y su propia base de datos.
2. **El central no es transaccional.** Si se apaga, las sucursales siguen vendiendo.
3. **El folio `V####` es local y es correcto así.** La identidad real es el UUID.

---

## Contenido (los 12 documentos maestros)

Todos los documentos viven en la carpeta [`ESPECIFICACIONES DEL PROYECTO/`](./ESPECIFICACIONES%20DEL%20PROYECTO/),
salvo el plano fundacional y este README, que están en la raíz.

| # | Documento | Responde | Estado |
|---|-----------|----------|--------|
| **0** | [`GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md`](./ESPECIFICACIONES%20DEL%20PROYECTO/GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md) | **¿Por dónde empiezo?** | ✅ Punto de entrada |
| **1** | [`PLANO ARQUITECTONICO PARA EL NUEVO POS.md`](./PLANO%20ARQUITECTONICO%20PARA%20EL%20NUEVO%20POS.md) | **¿Qué debe ser el nuevo POS?** | ✅ Fundacional |
| **2** | [`ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md`](./ESPECIFICACIONES%20DEL%20PROYECTO/ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md) | **¿Qué hace hoy el POS actual?** | ✅ Ingeniería inversa |
| **3** | [`PLAN_ACCION_ARQUITECTONICO_NUEVO_POS.md`](./ESPECIFICACIONES%20DEL%20PROYECTO/PLAN_ACCION_ARQUITECTONICO_NUEVO_POS.md) | **¿Qué acciones concretas ejecutar?** | ✅ 5 acciones |
| **4** | [`MODELO_DESPLIEGUE_Y_CONSOLIDACION_CENTRAL.md`](./ESPECIFICACIONES%20DEL%20PROYECTO/MODELO_DESPLIEGUE_Y_CONSOLIDACION_CENTRAL.md) | **¿Dónde vive y cómo se consolida?** | ✅ Topología |
| **5** | [`METODOLOGIA_DE_INGENIERIA_INVERSA_Y_DISENO.md`](./METODOLOGIA_DE_INGENIERIA_INVERSA_Y_DISENO.md) | **¿Cómo aplico esto a otro módulo?** | ✅ Manual de obra |
| **6** | [`ESPECIFICACION_RESPONSIVA_Y_ERGONOMIA_TACTIL.md`](./ESPECIFICACION_RESPONSIVA_Y_ERGONOMIA_TACTIL.md) | **¿Cómo debe verse y tocarse en cualquier pantalla?** | ✅ Regla dura (A-06) |
| **7** | [`ESPECIFICACION_DE_INTERFACES_POS.md`](./ESPECIFICACION_DE_INTERFACES_POS.md) | **¿Cómo se ve y se toca cada una de las 26 interfaces?** | ✅ Mapa visual y táctil |
| **8** | [`MODELO_DE_DATOS_DEL_NUEVO_POS.md`](./MODELO_DE_DATOS_DEL_NUEVO_POS.md) | **¿Cómo se guardan los datos?** | ✅ Cimiento (17 tablas) |
| **9** | [`CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md`](./CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md) | **¿Cómo se comunican los módulos sin leer tablas ajenas?** | ✅ 17 contratos |
| **10** | [`CRITERIOS_DE_ACEPTACION_DEL_NUEVO_POS.md`](./CRITERIOS_DE_ACEPTACION_DEL_NUEVO_POS.md) | **¿Cuándo se considera terminado?** | ✅ 21 criterios |
| **11** | [`DIRECTRICES_TRANSVERSALES_DEL_ERP.md`](./DIRECTRICES_TRANSVERSALES_DEL_ERP.md) | **¿Qué reglas unifican a todos los módulos?** | ✅ 5 directrices (DT-01 a DT-05) |

### Documentos de soporte (no maestros)

| Documento | Responde | Estado |
|-----------|----------|--------|
| [`AUTOCRITICA_DE_LOS_DOCUMENTOS_DEL_NUEVO_POS.md`](./ESPECIFICACIONES%20DEL%20PROYECTO/AUTOCRITICA_DE_LOS_DOCUMENTOS_DEL_NUEVO_POS.md) | **¿Qué corregimos de nuestros propios documentos?** | ✅ 3 defectos, 6 correcciones |

### Orden de lectura recomendado

```
0. GUÍA MAESTRA (empieza aquí)  → contexto y reglas
        ↓
1. PLANO (qué debe ser)         → el diseño
        ↓
2. ESPECIFICACIÓN (qué hace)    → la fuente de verdad funcional
        ↓
3. PLAN DE ACCIÓN (cómo)        → las acciones obligatorias
        ↓
4. DESPLIEGUE (dónde)           → la topología
        ↓
5. METODOLOGÍA (cómo replicar)  → el manual de obra para los demás módulos
        ↓
6. RESPONSIVA (cómo se ve/toca) → la regla dura de toda pantalla
        ↓
7. INTERFACES (cómo se ve cada una) → el mapa de las 26 interfaces
        ↓
8. MODELO DE DATOS (cómo se guarda) → el cimiento del edificio nuevo
        ↓
9. CONTRATOS (cómo se comunican)    → las fronteras entre módulos
        ↓
10. ACEPTACIÓN (cuándo está listo)  → la definición de "obra terminada"
        ↓
11. DIRECTRICES (qué unifica todo)  → las reglas transversales + matriz por módulo
```

---

## Las 11 reglas de oro

| # | Regla |
|---|-------|
| **1** | **No se toca el ERP.** Nunca. Ni una línea. |
| **2** | **El POS es un módulo del ERP**, no un sistema aparte. |
| **3** | **Cada sucursal tiene su propia BD.** No hay multi-tenant. |
| **4** | **El folio es local; el UUID es global.** Nunca uses el folio como identidad. |
| **5** | **El POS no lee tablas ajenas.** Solo contratos. |
| **6** | **Ninguna regla migra sin su test.** La unidad es regla + test. |
| **7** | **No hay `try/except pass` en la ruta crítica.** Outbox transaccional. |
| **8** | **El central no es transaccional.** Si cae, las sucursales siguen. |
| **9** | **Todo timestamp se guarda en UTC.** Se muestra en hora local. |
| **10** | **El inventario es un ledger inmutable.** Nunca `UPDATE stock`. |
| **11** | **El dinero se presenta por un solo camino.** Un formateador; el selector de moneda **declara**, nunca convierte. |

---

## Estructura actual del repositorio

```
PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS/
├── README.md                                       # Este archivo
├── METODOLOGIA_DE_INGENIERIA_INVERSA_Y_DISENO.md   # Manual de obra (reutilizable)
├── ESPECIFICACION_RESPONSIVA_Y_ERGONOMIA_TACTIL.md # Regla dura de toda pantalla
├── ESPECIFICACION_DE_INTERFACES_POS.md             # Mapa visual y táctil (26 interfaces)
├── MODELO_DE_DATOS_DEL_NUEVO_POS.md                # Cimiento: 17 tablas, UUID/UTC/ledger
├── CONTRATOS_ENTRE_MODULOS_DEL_NUEVO_POS.md        # Fronteras: 17 contratos entre módulos
├── CRITERIOS_DE_ACEPTACION_DEL_NUEVO_POS.md        # Definición de "obra terminada" (21 criterios)
├── DIRECTRICES_TRANSVERSALES_DEL_ERP.md            # Reglas transversales (DT-01 a DT-05) + matriz
├── PLANO ARQUITECTONICO PARA EL NUEVO POS.md       # Documento fundacional
└── ESPECIFICACIONES DEL PROYECTO/
    ├── GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md      # Punto de entrada
    ├── ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md # Ingeniería inversa
    ├── PLAN_ACCION_ARQUITECTONICO_NUEVO_POS.md           # Las 5 acciones
    ├── MODELO_DESPLIEGUE_Y_CONSOLIDACION_CENTRAL.md      # Topología hub-and-spoke
    └── AUTOCRITICA_DE_LOS_DOCUMENTOS_DEL_NUEVO_POS.md    # Autocrítica
```

### Estructura propuesta (etapas posteriores)

```
├── 01-logica-del-negocio/
│   ├── reglas-de-negocio.md                        # Las 81 reglas (RN-01 a RN-81)
│   └── deudas-conocidas.md                         # Las 5 deudas (DEUDA-01 a DEUDA-05)
├── 02-contratos/                                   # ✅ Cubierto por el Documento 9
│   ├── contrato-productos.md
│   ├── contrato-almacenes.md
│   ├── contrato-produccion.md
│   ├── contrato-auditoria.md
│   ├── contrato-estadisticas.md
│   ├── contrato-seguridad.md
│   └── contrato-vision.md
├── 03-modelo-de-datos/                             # ✅ Cubierto por el Documento 8
├── 04-estructura-de-modulos/                       # (Etapa 3 — pendiente)
└── 05-plan-de-construccion/                        # (Etapa 5 — pendiente)
```

---

## Anclaje

El plano está anclado al commit `fe9f6ed` (tag `v22-estable-fe9f6ed`) del ERP actual.
Cualquier divergencia posterior del ERP es una decisión consciente, no un accidente.

---

## Principio rector

El POS actual es la **fuente de verdad funcional** (lo que hace). Este repositorio es la
**fuente de verdad estructural** (cómo debería construirse).

**Copiamos su comportamiento, no su deuda.**
