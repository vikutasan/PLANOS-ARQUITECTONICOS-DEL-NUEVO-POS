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

## Contenido

| Documento | Descripción |
|---|---|
| [`PLANO ARQUITECTONICO PARA EL NUEVO POS.md`](./PLANO%20ARQUITECTONICO%20PARA%20EL%20NUEVO%20POS.md) | Documento fundacional: opinión del enfoque, 73 reglas de negocio, contratos entre módulos y plan de la primera etapa. |

---

## Estructura propuesta (etapas posteriores)

```
PLANOS-ARQUITECTONICOS-DEL-NUEVO-POS/
├── README.md                                  # Este archivo
├── PLANO ARQUITECTONICO PARA EL NUEVO POS.md  # Documento fundacional
├── 01-logica-del-negocio/
│   ├── reglas-de-negocio.md                   # Las 73 reglas (RN-01 a RN-73)
│   └── deudas-conocidas.md                    # Las 5 deudas (DEUDA-01 a DEUDA-05)
├── 02-contratos/
│   ├── contrato-productos.md
│   ├── contrato-almacenes.md
│   ├── contrato-produccion.md
│   ├── contrato-auditoria.md
│   ├── contrato-estadisticas.md
│   ├── contrato-seguridad.md
│   └── contrato-vision.md
├── 03-modelo-de-datos/                        # (Etapa 2 — pendiente)
├── 04-estructura-de-modulos/                  # (Etapa 3 — pendiente)
└── 05-plan-de-construccion/                   # (Etapa 5 — pendiente)
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
