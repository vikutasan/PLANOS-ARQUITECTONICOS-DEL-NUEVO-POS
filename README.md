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

> ### 📘 [GUÍA MAESTRA PARA COLABORADORES](./GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md)
>
> Es el **punto de entrada único**. Te dice qué leer, en qué orden, y qué reglas
> nunca romper. Si solo vas a leer un documento, lee ese.

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

## Contenido (los 5 documentos maestros)

| # | Documento | Responde | Estado |
|---|-----------|----------|--------|
| **0** | [`GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md`](./GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md) | **¿Por dónde empiezo?** | ✅ Punto de entrada |
| **1** | [`PLANO ARQUITECTONICO PARA EL NUEVO POS.md`](./PLANO%20ARQUITECTONICO%20PARA%20EL%20NUEVO%20POS.md) | **¿Qué debe ser el nuevo POS?** | ✅ Fundacional |
| **2** | [`ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md`](./ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md) | **¿Qué hace hoy el POS actual?** | ✅ Ingeniería inversa |
| **3** | [`PLAN_ACCION_ARQUITECTONICO_NUEVO_POS.md`](./PLAN_ACCION_ARQUITECTONICO_NUEVO_POS.md) | **¿Qué acciones concretas ejecutar?** | ✅ 5 acciones |
| **4** | [`MODELO_DESPLIEGUE_Y_CONSOLIDACION_CENTRAL.md`](./MODELO_DESPLIEGUE_Y_CONSOLIDACION_CENTRAL.md) | **¿Dónde vive y cómo se consolida?** | ✅ Topología |

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
```

---

## Las 10 reglas de oro

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

---

## Estructura propuesta (etapas posteriores)

```
PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS/
├── README.md                                       # Este archivo
├── GUIA_MAESTRA_PARA_COLABORADORES_NUEVO_POS.md    # Punto de entrada
├── PLANO ARQUITECTONICO PARA EL NUEVO POS.md       # Documento fundacional
├── ESPECIFICACION_FUNCIONAL_POS_INGENIERIA_INVERSA.md  # Ingeniería inversa
├── PLAN_ACCION_ARQUITECTONICO_NUEVO_POS.md         # Las 5 acciones
├── MODELO_DESPLIEGUE_Y_CONSOLIDACION_CENTRAL.md    # Topología hub-and-spoke
├── 01-logica-del-negocio/
│   ├── reglas-de-negocio.md                        # Las 81 reglas (RN-01 a RN-81)
│   └── deudas-conocidas.md                         # Las 5 deudas (DEUDA-01 a DEUDA-05)
├── 02-contratos/
│   ├── contrato-productos.md
│   ├── contrato-almacenes.md
│   ├── contrato-produccion.md
│   ├── contrato-auditoria.md
│   ├── contrato-estadisticas.md
│   ├── contrato-seguridad.md
│   └── contrato-vision.md
├── 03-modelo-de-datos/                             # (Etapa 2 — pendiente)
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
