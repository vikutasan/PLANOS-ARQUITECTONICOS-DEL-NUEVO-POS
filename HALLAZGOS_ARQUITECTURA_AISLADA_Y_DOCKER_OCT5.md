# Hallazgos de Arquitectura: Plan de Prueba en Paralelo y Aislamiento del Nuevo POS

**Fecha:** 5 de Octubre de 2026
**Contexto:** El usuario reportó que en el Nuevo POS no podía capturar cuentas, enviar pedidos al pizarrón ni abrir el gestor de caja. Su hipótesis era que faltaba integrar los módulos "Gestor de Productos" y "Vista General" del ERP.

Este documento clarifica para futuras IAs cómo funciona la arquitectura de aislamiento del Nuevo POS y cómo se ejecuta localmente junto al ERP sin colisionar.

## 1. El Mito de la Integración Bloqueante
**Es FALSO que el Nuevo POS requiera estar integrado o conectado al ERP monolítico para funcionar.** 
Bajo el paradigma del "Plan de Prueba en Paralelo", el Nuevo POS fue diseñado con **Autonomía Estricta**. 
- Tiene su **propio backend** en Python (FastAPI).
- Tiene su **propia base de datos** aislada (`nuevo_pos`).
- Las integraciones con el ERP (como CRM o Configuraciones) operan por contratos HTTP asíncronos y están diseñadas con **Degradación Elegante (DT-07)**: si el ERP no responde, el POS asume valores seguros (ej. 100% de pago mínimo, sin beneficios CRM) pero **NUNCA bloquea la venta ni crashea**.

## 2. Topología de Puertos y Contenedores (Docker)
El entorno de desarrollo ejecuta el Nuevo POS y el ERP en paralelo sin ningún conflicto de puertos ni de bases de datos.

### El ERP (R de Rico Viejo)
- **Frontend:** Puerto `5000` (`rderico-pos-dev`)
- **Backend:** Puerto `5001` (`rderico-api-dev`)
- **Base de Datos:** Puerto `5433` -> `5432` (`rderico-db-dev`)

### El Nuevo POS
- **Frontend:** Puerto `5100` (Vite)
- **Backend (API):** Puerto `5101` (`nuevo_pos_api` - FastAPI)
- **Base de Datos:** Puerto `5432` (`nuevo_pos_db` - PostgreSQL dedicada)

*Nota:* Ambos coexisten usando `docker-compose`. La base de datos del POS usa una imagen separada en el puerto 5432 expuesto al host, por lo que su cadena de conexión es `postgresql+asyncpg://pos:pos@localhost:5432/nuevo_pos`.

## 3. ¿Por qué el POS se veía "vacío"?
Si el UI del POS arranca pero no se ven productos ni se puede abrir la caja, **no es un error de integración**. Es porque la base de datos `nuevo_pos` está vacía (no tiene catálogo ni turnos abiertos).

### Procedimiento para Poblar y Activar el POS
Para encender el backend y sembrar los datos de prueba, se debe usar el contenedor del API (`nuevo_pos_api`) ejecutando las migraciones (Alembic) y el script semilla (`seed_demo.py`):

```bash
# 1. Correr las migraciones para crear las tablas
docker exec nuevo_pos_api alembic upgrade head

# 2. Inyectar catálogo y abrir el turno en la terminal TERM-01
docker exec nuevo_pos_api python scripts/seed_demo.py
```

El script `seed_demo.py` crea 3 categorías, 6 productos (Bolillo, Concha, Café, etc.) y un turno/sesión activo para la terminal `TERM-01`. **Esto habilita inmediatamente la capacidad de cargar cuentas, cobrar e interactuar con el Pizarrón.**

## 4. Regla Dura para IAs Futuras
Si un usuario reporta que el POS nuevo no carga productos, no permite cobrar o muestra errores de conexión:
1. **NO intentes "integrarlo" al ERP viejo** leyendo la base de datos del ERP. Está estrictamente prohibido por la regla **A-02 (Frontera de Módulos)**.
2. Verifica que el contenedor `nuevo_pos_api` esté corriendo en el puerto `5101`.
3. Verifica que el script `seed_demo.py` haya sido ejecutado.
4. Recuerda que los NIPs de prueba y los folios de terminal deben existir en la tabla `terminal_sessions` de la base de datos `nuevo_pos`.
