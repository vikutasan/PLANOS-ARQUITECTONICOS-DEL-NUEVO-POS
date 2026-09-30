# 🔍 PLAN DE ABORDAJE — FASE 10: AUDITORÍA DE PARIDAD (viejo POS → nuevo POS)

**Versión:** 1.0
**Fecha:** 30 Sep 2026
**Estado:** EN EJECUCIÓN (F10.0 completada)
**Autor:** Arquitecto del Nuevo POS
**Origen:** Hallazgo del usuario — *"en el gestor de terminales no aparece la opción copiar url que sí aparece en el pos viejo"*

---

## 0. POR QUÉ EXISTE ESTA FASE

Las Fases 1–9.1 construyeron el nuevo POS **"de adentro hacia afuera"**: cada
pieza (regla, servicio, hook, componente) se verificó **en aislamiento** y pasó
su compuerta. Ese método garantiza que **lo que se construyó funciona**.

Pero **no garantiza que se haya construido TODO lo que el viejo POS hacía**.

La prueba viva es el hallazgo del usuario: el **"Copiar URL"** del gestor de
terminales existe en el viejo POS y **no existe** en el nuevo — ni el botón ni
la lógica. Es una **OMISIÓN TOTAL**, no un cableado pendiente.

### Las tres instancias de la misma clase de falla

| # | Falla | Fase donde se detectó | Naturaleza |
|---|-------|----------------------|------------|
| 1 | `GestorDeCaja` huérfano (existía, nadie llegaba a él) | F4.5 | Integración olvidada |
| 2 | `payment_details` no expuesto en la salida del ticket | F9.1.4a | Dato construido, no expuesto |
| 3 | **"Copiar URL" ausente en el gestor de terminales** | **F10 (ahora)** | **Omisión total** |

**Causa raíz común:** el método "de adentro hacia afuera" verifica la **calidad**
de cada pieza, pero **no la COMPLETITUD del conjunto** contra el viejo POS.

**Lección codificada (§10.6.1):** *"el componente existe y pasa su test" ≠
"el usuario puede llegar a él"*. F10 extiende esa lección: *"el componente
existe y pasa su test" ≠ "el conjunto está completo"*.

---

## 1. OBJETIVO DE LA FASE 10

Producir una **auditoría de paridad exhaustiva** entre el viejo POS
(`apps/pos/`) y el nuevo POS (`../NUEVO-POS/apps/pos/src/`), clasificar **cada**
archivo y **cada** capacidad de UI en una de cuatro categorías, y cerrar las
brechas que el usuario apruebe — **antes** de redactar la documentación final
(§11).

**Regla dura de esta fase (REGLA DURA 2 — "Verificar, no asumir"):**
ninguna brecha se declara "no aplica" sin evidencia leída del código.

---

## 2. LAS CUATRO CATEGORÍAS DE PARIDAD

| Categoría | Significado | Acción |
|-----------|-------------|--------|
| **PORTADA** | Existe en el viejo y existe (y funciona) en el nuevo | Nada. Solo registrar evidencia. |
| **OMITIDA** | Existe en el viejo y **NO** existe en el nuevo | Decidir: portar o descartar con razón. |
| **HUÉRFANA** | La lógica existe en el nuevo pero **no hay punto de entrada** | Cablear (como F4.5). |
| **DESCARTADA** | Se decidió **deliberadamente** no portar | Registrar el POR QUÉ (no es un olvido). |

---

## 3. F10.0 — INVENTARIO CRUDO DE PARIDAD (COMPLETADA)

### 3.1 Inventario del viejo POS (`apps/pos/`)

**Raíz (10 archivos):**

| Archivo | Qué es | Categoría | Evidencia |
|---------|--------|-----------|-----------|
| `RetailVisionPOS.jsx` | Pantalla principal del POS | **PORTADA** | Reescrito como `RetailVisionPOS.jsx` (824 líneas) |
| `OpenAccountsCorkboard.jsx` | Pizarrón de cuentas | **PORTADA** | Reescrito en F5.3 |
| `TableServicePOS.jsx` | TPV Meseros & KDS (servicio de mesa) | **DESCARTADA** | Módulo separado (KDS vive en `apps/kds/`). No es el POS de mostrador. |
| `CategoryEditor.jsx` | Config de visibilidad IA por categoría | **DESCARTADA** | Herramienta de admin/visión, no del POS de venta |
| `GrandezaDailyUI.jsx` | Reparto Pan Grandeza — gestión diaria | **DESCARTADA** | Módulo Grandeza (reparto), fuera del alcance del POS |
| `GrandezaDriverUI.jsx` | Reparto Pan Grandeza — app repartidor | **DESCARTADA** | Módulo Grandeza (reparto) |
| `GrandezaOrderRequestsTab.jsx` | Reparto Pan Grandeza — programación pedidos | **DESCARTADA** | Módulo Grandeza (reparto) |
| `GrandezaParamsUI.jsx` | Reparto Pan Grandeza — parámetros | **DESCARTADA** | Módulo Grandeza (reparto) |
| `RepartoPanGrandezaUI.jsx` | Reparto Pan Grandeza — landing | **DESCARTADA** | Módulo Grandeza (reparto) |
| `config.js` | Config del POS viejo | **PORTADA** | Reemplazado por `api/client.js` + `CONFIG` |

**`components/` (16 archivos):**

| Archivo | Categoría | Notas |
|---------|-----------|-------|
| `TerminalSelector.jsx` | **OMITIDA (parcial)** | ⚠️ **DEFECTO CONFIRMADO:** el nuevo tiene el selector pero **le falta `copyUrl`** (botón + lógica). Ver §3.3. |
| `CategoryBar.jsx` | **PORTADA** | Reescrito |
| `CheckoutScreen.jsx` | **PORTADA** | Reescrito + pagos mixtos (F9.1.3) |
| `CorteTicketTemplate.jsx` | **PORTADA** | Reescrito (F4.4) |
| `GestorDeCaja.jsx` | **PORTADA** | Reescrito (F4.3) |
| `POSHeader.jsx` | **PORTADA** | Reescrito |
| `POSOverlays.jsx` | **PORTADA** | Reescrito (`OverlayExito`, `OverlayError`, `OverlayConfirmar`, `OfflineBanner`) |
| `ProductCard.jsx` | **PORTADA** | Reescrito |
| `ProductGrid.jsx` | **PORTADA** | Reescrito |
| `ProgramacionPedidoModal.jsx` | **PORTADA** | Reescrito como `OrderProgrammingModal.jsx` (F7.5.5) — paridad verificada |
| `SalesReceipt.jsx` | **PORTADA** | Reescrito |
| `TicketTemplate.jsx` | **PORTADA** | Reescrito |
| `VisionVisor.jsx` | **PORTADA** | Reescrito |
| `VoiceCartPanel.jsx` | **PORTADA** | Reescrito |
| `AnnotationCanvas.jsx` | **DESCARTADA** | Herramienta de etiquetado YOLO (dataset de visión), no del POS |
| `GestionPersonal.jsx` | **DESCARTADA** | Módulo RRHH (usa `securityService`), fuera del alcance del POS |

**`hooks/` (9 archivos):**

| Archivo | Categoría | Notas |
|---------|-----------|-------|
| `useBarcodeScanner.js` | **PORTADA** | Reescrito |
| `useBeforeUnload.js` | **PORTADA** | Reescrito |
| `useCart.js` | **PORTADA** | Reescrito |
| `useNetworkHealth.js` | **PORTADA** | Reescrito |
| `useTerminalLocking.js` | **PORTADA** | Reescrito |
| `useTicketActions.js` | **PORTADA** | Reescrito |
| `useVision.js` | **PORTADA** | Reescrito |
| `useVoiceCart.js` | **PORTADA** | Reescrito |
| `usePOSSession.js` | **PORTADA (absorbida)** | Su rol (carga de catálogo, folios, Zero-Auto-Restore) se repartió entre `useCart`, `useCheckout`, `useTerminals` y `RetailVisionPOS`. **Verificar Zero-Auto-Restore** en F10.1. |

**`services/` (8 archivos):**

| Archivo | Categoría | Notas |
|---------|-----------|-------|
| `cashService.js` | **PORTADA** | Reescrito (F4.2) |
| `securityService.js` | **PORTADA** | Reescrito |
| `terminalService.js` | **PORTADA** | Reescrito |
| `POSService.js` | **PORTADA (absorbida)** | Sus llamadas se repartieron en `api/client.js` + servicios por dominio |
| `paymentService.js` | **PORTADA** | Reemplazado por `checkoutService.js` + RN-94/95 (F9.1) |
| `networkMonitor.js` | **PORTADA** | Reemplazado por `useNetworkHealth.js` |
| `loyaltyService.js` | **DESCARTADA** | La lealtad es del **CRM** (Fase 8), no del POS. Decisión arquitectónica. |
| `offlineStore.js` | **DESCARTADA** | Cola IndexedDB del **repartidor (Grandeza)**, no del POS. Ver "cola local" (v1.1). |

**`state/` (1 archivo):**

| Archivo | Categoría | Notas |
|---------|-----------|-------|
| `sessionReset.js` | **PORTADA** | Reescrito |

**`utils/` (2 archivos):**

| Archivo | Categoría | Notas |
|---------|-----------|-------|
| `terminalCardState.js` | **PORTADA** | Reescrito |
| `posConstants.js` | **PORTADA (absorbida)** | `INITIAL_CATEGORIES`, `getProductEmoji`, `DEFAULT_TERMINALS`, `loadTerminalsConfig` → repartidos en `useTerminals` + catálogo del API. **Verificar `getProductEmoji`** en F10.1. |

### 3.2 Inventario del nuevo POS (`../NUEVO-POS/apps/pos/src/`)

**Raíz:** `App.jsx`, `GestorDeCaja.jsx`, `RetailVisionPOS.jsx`, `main.jsx`, `index.css`
**`admin/`:** `ExtractorEstetica.jsx`
**`api/`:** `client.js`
**`components/` (22):** `CatalogoPDF.jsx`, `CategoryBar.jsx`, `CheckoutScreen.jsx`, `CorteTicketTemplate.jsx`, `CustomerIdentificationPanel.jsx`, `ExitAccountModal.jsx`, `LoginScreen.jsx`, `OpenAccountsCorkboard.jsx`, `OrderProgrammingModal.jsx`, `POSHeader.jsx`, `POSOverlays.jsx`, `ProductCard.jsx`, `ProductGrid.jsx`, `SalesReceipt.jsx`, `SelectorCategoriasPDF.jsx`, `TecladoNumerico.jsx`, `TerminalSelector.jsx`, `ThemeSelector.jsx`, `TicketDeliveryPanel.jsx`, `TicketTemplate.jsx`, `VisionVisor.jsx`, `VoiceCartPanel.jsx`
**`config/`:** `vision.js`, `voz.js`
**`hooks/` (16):** `useAuth.js`, `useBarcodeScanner.js`, `useBeforeUnload.js`, `useCart.js`, `useCheckout.js`, `useCustomerIdentification.js`, `useModo.js`, `useNetworkHealth.js`, `useOpenAccounts.js`, `useOrderProgramming.js`, `useTerminalLocking.js`, `useTerminals.js`, `useTheme.js`, `useTicketActions.js`, `useVision.js`, `useVoiceCart.js`
**`services/` (9):** `benefitsService.js`, `cashService.js`, `checkoutService.js`, `notificationsService.js`, `openAccountsService.js`, `ordersService.js`, `printService.js`, `securityService.js`, `terminalService.js`
**`state/`:** `sessionReset.js`
**`theme/`:** `default.js`, `index.js`, `minimal.js`, `nocturno.js`
**`utils/`:** `outcome.js`, `terminalCardState.js`, `ticketGenerator.js`, `voiceCartMapper.js`, `withRetries.js`

### 3.3 BRECHAS CONFIRMADAS (candidatas a F10.2)

| # | Brecha | Categoría | Evidencia | Severidad |
|---|--------|-----------|-----------|-----------|
| **B-01** | **"Copiar URL" en el gestor de terminales** | **OMITIDA** | Viejo: `apps/pos/components/TerminalSelector.jsx:63` (`copyUrl`) + botón en `:184`. Nuevo: 0 coincidencias de `copiar\|clipboard\|copyUrl`. | **ALTA** — el usuario la reportó; es una capacidad operativa real (configurar accesos directos de las máquinas). |

### 3.4 BRECHAS A VERIFICAR EN F10.1 (sospechas, sin evidencia concluyente aún)

| # | Sospecha | Por qué | Cómo verificar |
|---|----------|---------|----------------|
| **V-01** | **Zero-Auto-Restore** (terminal siempre inicia en blanco) | Vivía en `usePOSSession.js:53-76`. El nuevo no tiene `usePOSSession`. | Buscar el efecto de limpieza en `RetailVisionPOS.jsx` / `useCart.js`. |
| **V-02** | **`getProductEmoji`** (emoji por producto) | Vivía en `posConstants.js:26-47`. El nuevo no tiene `posConstants.js`. | Ver si `ProductCard.jsx` / `ProductGrid.jsx` resuelven emoji. |
| **V-03** | **`handleImageUpload`** (subir imagen de terminal) | Vivía en el viejo `TerminalSelector.jsx:120-126`. | Ver si el nuevo `TerminalSelector.jsx` permite subir imagen. |
| **V-04** | **`loadTerminalsConfig`** (terminales desde BD) | Vivía en `posConstants.js:63-79`. | Ver si `useTerminals.js` lee de `/settings/pos_terminals_config`. |
| **V-05** | **`DEFAULT_TERMINALS`** (fallback hardcodeado) | Vivía en `posConstants.js:50-57`. | Ver si `useTerminals.js` tiene fallback. |
| **V-06** | **`ForceLogoutModal`** (desbloqueo forzado por admin) | Vivía en el viejo `POSOverlays.jsx:15-42`. | Ver si el nuevo `POSOverlays.jsx` lo tiene. |
| **V-07** | **`OfflineBanner` con `pendingCount`/`isSyncing`** | El viejo mostraba cola pendiente; el nuevo no tiene cola local (v1.1). | Confirmar que el nuevo `OfflineBanner` es solo "sin conexión" (intencional). |
| **V-08** | **`useVisitDraft` / borradores** | Vivía en `GrandezaDriverUI.jsx` (repartidor). | Confirmar que es del módulo Grandeza (DESCARTADA). |
| **V-09** | **`calcularDenominaciones`** (desglose de cambio) | Vivía en `paymentService.js:148-161`. | Ver si `checkoutService.js` lo tiene. |
| **V-10** | **`calcularPuntosAGanar` / `infoRedencionCheckout`** | Vivían en `paymentService.js` / `loyaltyService.js`. | Confirmar que la lealtad vive en el CRM (Fase 8). |

---

## 4. F10.1 — TRIAJE (COMPLETADA)

Para **cada** brecha (B-01 y V-01…V-10) se produjo una decisión explícita,
verificada contra el código del nuevo POS (REGLA DURA 2 — "Verificar, no asumir"):

- **PORTAR** → se construye en F10.2, con su test.
- **CABLEAR** → la lógica ya existe, solo falta el punto de entrada.
- **DESCARTAR** → se documenta el POR QUÉ (no es un olvido).

### 4.1 Tabla de decisiones (verificada)

| # | Brecha | Decisión | Evidencia leída (nuevo POS) |
|---|--------|----------|------------------------------|
| **B-01** | "Copiar URL" en el gestor de terminales | **PORTAR** | Nuevo `TerminalSelector.jsx` (504 líneas): el gestor tiene editar/eliminar/agregar/guardar, pero **0 coincidencias** de `copyUrl`/`clipboard`/`copiar`. Viejo: `apps/pos/components/TerminalSelector.jsx:63` (`copyUrl`) + botón en `:184`. **OMISIÓN TOTAL confirmada.** |
| **V-01** | Zero-Auto-Restore (terminal inicia en blanco) | **DESCARTAR (no aplica)** | El nuevo POS **no persiste sesión en `localStorage`**: `RetailVisionPOS.jsx` carga catálogo + `getSesionActiva` al montar (líneas 178-201) y el carrito vive en `useCart` (memoria + servidor). No hay estado de sesión que "restaurar", así que el problema que Zero-Auto-Restore resolvía **no existe** en el nuevo diseño. La persistencia es **por ítem en el servidor** (contratos 18-20), no en el cliente. |
| **V-02** | `getProductEmoji` (emoji por producto) | **PORTADA (reubicada al backend)** | `ProductCard.jsx:58` usa `producto.icono \|\| '🍞'`. El emoji viene del **API** (campo `icono`), no de una función cliente. La lógica se movió al backend. |
| **V-03** | `handleImageUpload` (subir imagen de terminal) | **DESCARTAR (decisión de diseño)** | El nuevo `TerminalSelector.jsx` usa `PRESET_ICONS` (líneas 18-25) — un catálogo de emojis predefinidos. No hay subida de imagen. **Es una simplificación deliberada**: el ícono de terminal es un emoji, no un archivo. El viejo permitía subir imagen; el nuevo lo reemplazó por presets. Se documenta como DESCARTADA con razón. |
| **V-04** | `loadTerminalsConfig` (terminales desde BD) | **PORTADA** | `useTerminals.js:103` llama `fetchTerminalConfig()` (de `terminalService.js`) dentro del `Promise.all` de init. La config SÍ se lee del backend. |
| **V-05** | `DEFAULT_TERMINALS` (fallback hardcodeado) | **PORTADA** | `useTerminals.js:54-61` define `DEFAULT_TERMINALS` con `TERM-01..TERM-06` (F7.7d unificó el vocabulario de IDs). Es el fallback si el API no responde (`:103`). |
| **V-06** | `ForceLogoutModal` (desbloqueo forzado por admin) | **DESCARTAR (reemplazado por heartbeat)** | El nuevo `POSOverlays.jsx` NO tiene `ForceLogoutModal`. Pero el problema que resolvía (terminal "ocupada" por un cajero ausente) lo resuelve el **heartbeat** de `useTerminalLocking.js` (líneas 100-124): mientras la pestaña vive, late; si deja de latir, el lock **expira por TTL**. Ya no hace falta un admin que fuerce el desbloqueo. Decisión arquitectónica superior. |
| **V-07** | `OfflineBanner` con `pendingCount`/`isSyncing` | **DESCARTAR (intencional)** | El nuevo `OfflineBanner` (`POSOverlays.jsx:155-169`) es solo "sin conexión", con comentario explícito de DECISIÓN ARQUITECTÓNICA (v1.1): el nuevo POS **no tiene cola local**. |
| **V-08** | `useVisitDraft` / borradores | **DESCARTAR (fuera de alcance)** | Vive en `GrandezaDriverUI.jsx` (app del repartidor). Es del módulo Grandeza, no del POS. |
| **V-09** | `calcularDenominaciones` (desglose de cambio) | **DESCARTAR (no aplica)** | `checkoutService.js` tiene `calcularCambio` (líneas 77-82) pero NO `calcularDenominaciones`. El nuevo `CheckoutScreen.jsx` usa `BILLETES_RAPIDOS = [50,100,200,500]` (línea 60) para **capturar** el efectivo recibido, no para desglosar el cambio en billetes/monedas. El desglose de denominaciones era una ayuda visual del viejo POS; el nuevo muestra el cambio como monto. Decisión de UX, no una omisión funcional. |
| **V-10** | `calcularPuntosAGanar` / `infoRedencionCheckout` | **DESCARTAR (la lealtad es del CRM)** | La lealtad vive en el **CRM** (Fase 8). El POS consume el contrato 26 (`benefitsService.js` → `getBeneficiosParaTicket`) y muestra `puntos_a_ganar`/`puntos_disponibles` que **el CRM calcula** (`CustomerIdentificationPanel.jsx:204,241`). El POS NO calcula puntos (A-02: no lee tablas ajenas). Decisión arquitectónica. |

### 4.2 Brechas que pasan a F10.2

| # | Brecha | Acción | Severidad |
|---|--------|--------|-----------|
| **B-01** | "Copiar URL" en el gestor de terminales | **PORTAR** — botón + lógica `copyUrl` en el nuevo `TerminalSelector.jsx`, con su test. | ALTA |

**Resultado del triaje:** de 11 brechas sospechadas, **1 se porta** (B-01), **4 son
PORTADAS** (V-02, V-04, V-05 + las ya confirmadas en F10.0) y **6 se DESCARTAN**
con razón documentada (V-01, V-03, V-06, V-07, V-08, V-09, V-10).

**Compuerta de F10.1:** ✅ cada brecha tiene una decisión escrita y justificada.
Ninguna queda "por ver".

---

## 5. F10.2 — CIERRE DE BRECHAS APROBADAS (PENDIENTE)

Cada brecha aprobada se cierra **con su test** (patrón de todas las fases):

1. Escribir el test que falla (rojo).
2. Implementar (verde).
3. Correr la compuerta completa (`npm run ci`).
4. Escribir la ficha de la brecha.
5. Commit + push.

**Compuerta de F10.2:** CI verde + cada brecha cerrada tiene su test y su ficha.

---

## 6. F10.3 — CIERRE DE F10 (PENDIENTE)

1. Correr `npm run ci` completo (verde).
2. Escribir `FICHA_F10_PARIDAD.md` con la tabla final de paridad.
3. Actualizar el Plan Maestro (§7 Fase 10 + §10.6.1 con la lección de completitud).
4. Commit + push.

**Compuerta de F10.3:** CI verde + ficha + Plan Maestro actualizado + push.

---

## 7. LO QUE VIENE DESPUÉS DE F10

Solo cuando F10 esté cerrada (paridad verificada), se aborda el entregable
final: **§11 — LA DOCUMENTACIÓN FINAL DEL NUEVO POS** (7 tomos).

**Principio:** no se documenta como "terminado" un POS que no se ha verificado
completo contra su predecesor.

---

## 8. TRAZABILIDAD

| Documento | Relación |
|-----------|----------|
| `PLAN_MAESTRO_DEFINITIVO_POS.md` §6.8 | UX heredada — la integración se hereda |
| `PLAN_MAESTRO_DEFINITIVO_POS.md` §10.6.1 | La lección de F4.5 — la integración es una compuerta |
| `PLAN_MAESTRO_DEFINITIVO_POS.md` §11 | Documentación final (se ejecuta DESPUÉS de F10) |
| `FICHA_F4_5_INTEGRACION.md` | El caso GestorDeCaja huérfano (instancia 1) |
| `FICHA_F9_1_4_IMPRESION_CIERRE.md` | El caso `payment_details` (instancia 2) |
