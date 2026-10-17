# Plan de trabajo — Developer 2 (Frontend y experiencia de usuario)

Responsabilidades: aplicación Next.js, formulario de pagos simulados, dashboard de evaluaciones, historial y detalle de pagos, visualización de reglas, evidencias, bloqueos y errores, integración con la API, pruebas de interfaz y E2E de sus flujos, y verificación del acceso a la demo desde la interfaz.

Este plan es la vista diaria del rol de Developer 2 dentro de [development-plan.md](../../development-plan.md) y [implementation-checklist.md](../../implementation-checklist.md), que siguen siendo la fuente de verdad sobre pruebas y criterios técnicos exactos. Todos los estados inician en `Pendiente`: no se marcan como completados hasta que exista evidencia verificable, según [definition-of-done.md](../../definition-of-done.md).

| Día | Objetivo | Tareas | Entregable | Dependencias | Criterio de aceptación | Estado |
|---|---|---|---|---|---|---|
| 1 | Base de la aplicación | Levantar `flowguard-app` (Next.js/React); layout y navegación base | Esqueleto de la app desplegable localmente | Contratos congelados (día 1) | La app arranca sin errores | Pendiente |
| 2 | Simulador de pagos con mocks | Construir el formulario de pagos simulados consumiendo mocks | Formulario funcional contra datos mock | Fixtures de Dev 5 | El formulario renderiza los campos del contrato de pago | Pendiente |
| 3 | Dashboard con mocks | Construir el panel de ejecuciones con `mocks/payment-approved.json` y `mocks/payment-blocked.json` | Dashboard renderizando los tres mocks sin llamar a la API real | Fixtures de Dev 5 | Revisión visual + prueba de render con Playwright apuntando a mocks | Pendiente |
| 4 | Simulador conectado a la API real | Conectar el simulador a `POST /api/v1/executions` | Simulador mostrando la decisión real del backend | `POST /executions` de Dev 1 | Ejecutar $8,500 y $15,000 desde la UI refleja la decisión real | Pendiente |
| 5 | Historial de pagos | Implementar la vista de historial usando `GET /api/v1/executions` | Historial funcional con datos reales | Simulador conectado (día 4) | El historial lista ejecuciones reales | Pendiente |
| 6 | Detalle de ejecución y hito del bloque 2 | Implementar la vista de detalle (`GET /executions/:id`); participar en el ensayo del hito obligatorio | Detalle de ejecución funcional; hito del día 6 demostrado | `GET /executions/:id` disponible | Demo interna reproduce el hito sin errores | Pendiente |
| 7 | Diseño del centro de evidencias | Diseñar la vista de evidencias contra el contrato del informe del agente, usando mock si Nebius aún no está listo | Vista de evidencias con datos mock | Contrato del informe (Dev 3/Dev 5) | La vista respeta el esquema del informe | Pendiente |
| 8 | Centro de evidencias conectado | Conectar la vista a `GET /executions/:id/evidence` | Centro de evidencias con datos reales cuando estén disponibles | `GET /executions/:id/evidence`, contrato del informe | Un `BLOCK` real muestra reglas, evidencias e informe sin datos inventados | Pendiente |
| 9 | Validación del centro de evidencias y hito del bloque 3 | Verificar que no se muestran datos inventados cuando falla el proveedor | Centro de evidencias validado | Integración real o mock controlado de fallos | Prueba E2E del centro de evidencias pasa | Pendiente |
| 10 | Estados de carga y error | Pulir estados de carga, error y detalle, incluyendo historial de intentos de investigación (`investigation.attempts`) | UI con todos los estados representados | `GET /executions/:id` con `investigation.attempts` | Cada estado de [architecture.md § 6.4](../../architecture.md#64-estados-de-interfaz) tiene representación visual verificable | Pendiente |
| 11 | Revisión visual y snapshots | Revisión visual completa; pruebas de snapshot | Suite de snapshots actualizada | Estados pulidos (día 10) | Revisión visual + prueba de snapshot pasa | Pendiente |
| 12 | Resiliencia visual ante fallo de Nebius y hito del bloque 4 | Verificar que la UI se mantiene consistente cuando se fuerza un fallo de Nebius | UI estable ante fallo de proveedor | Hito del día 9 | La UI no contradice una decisión persistida cuando el proveedor falla | Pendiente |
| 13 | Congelación de funcionalidades | Solo correcciones de bugs o documentación a partir de este día | PRs únicamente de corrección | Hito del día 12 | Ningún PR posterior agrega funcionalidad nueva | Pendiente |
| 14 | Ajustes visuales finales | Pulir detalles visuales finales; recorrido manual del guion de demo; si la demo es pública, verificar el acceso desde la interfaz con las credenciales entregadas por Dev 1 | UI lista para demo | Congelación de funcionalidades | La UI no presenta estados inconsistentes durante el guion de demo | Pendiente |
| 15 | Entrega final | Apoyar el ensayo cronometrado de la demo | Frontend estable para la demo | Todo lo anterior | Checklist de [implementation-checklist.md](../../implementation-checklist.md) completa para las filas de Developer 2 | Pendiente |

## Referencias

- [implementation-checklist.md](../../implementation-checklist.md) — detalle técnico completo por fila
- [api-contracts.md](../../api-contracts.md) — contratos de API que consume el frontend
- [integration-and-dependencies.md](../../integration-and-dependencies.md) — qué necesita Developer 2 de los demás
