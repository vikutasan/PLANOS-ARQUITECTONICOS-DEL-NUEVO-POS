# Requerimientos UX y Flujos Heredados Pendientes

**Fecha de registro:** 7 de Octubre de 2026
**Contexto:** Tras la corrección de errores críticos en los cobros mixtos (Fase 9.1), el dueño del producto identificó flujos visuales y de interacción del POS viejo que resultan más ergonómicos y deben preservarse en la nueva arquitectura.

## 1. Ergonomía del Teclado de Pagos ("Abonar pago")
**Problema actual:** En el Nuevo POS, el botón de `+ Agregar pago (...)` está ubicado debajo de la lista de abonos y de los inputs de montos, lo cual resulta menos orgánico que en el POS original donde el flujo visual tecleo → aceptación es inmediato.
**Requisito:** 
- Reposicionar o rediseñar el botón de confirmación de abono dentro de `CheckoutScreen.jsx` (y/o `TecladoNumerico.jsx`) para que acompañe el flujo del teclado en pantalla (p. ej. integrado cerca del botón de "Borrar" o en la zona inferior del teclado).
- **Restricción (DT-09):** La solución debe ser responsiva y mantener los estándares táctiles (mínimo 44px de área clickeable) definidos en el Nuevo POS.

## 2. Separación Visual de Tarjeta (Crédito vs Débito)
**Problema actual:** En la interfaz actual del `CheckoutScreen.jsx`, hay un solo botón "Tarjeta" que asume "Débito" por defecto para el backend. 
**Requisito:**
- Separar visualmente el botón "Tarjeta" en dos botones distintos: **Tarjeta de Crédito** y **Tarjeta de Débito**. 
- Esto respeta la distinción que existía en el POS viejo y que el backend actual del Nuevo POS ya soporta perfectamente en su estructura de datos (`METODOS_VALIDOS: ['DEBITO', 'CREDITO']`).

## 3. Pantalla de Confirmación de Pedidos (Revisión Previa)
**Problema actual:** Al capturar un pedido en el Nuevo POS, se llenan los datos del cliente y fecha y se pasa directo a cobrar.
**Requisito:**
- Restaurar la "Pantalla intermedia de Confirmación" que existía en el POS viejo. 
- Antes de que el operador pueda pagar, el sistema debe mostrarle un resumen claro de todo lo que implica el pedido (productos, totales, datos de cliente, fecha de entrega) para que pueda **confirmar los detalles con el cliente** verbalmente antes de cobrar e imprimir.
- **Implementación sugerida:** Puede integrarse como un estado previo dentro de `CheckoutScreen` o como un modal independiente intermedio que se intercepte al darle "Cobrar" cuando el ticket tiene naturaleza de pedido. 

---
*Nota para IAs futuras: Estos requerimientos solo atañen a la capa de UI (React/Tailwind) y la orquestación del flujo. No requieren cambios en la capa de datos del backend ni contravienen la persistencia atómica.*
