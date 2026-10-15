# Plan de trabajo — Developer 4 (Contratos, mocks, pruebas e integración)

Responsabilidades: contratos compartidos, fixtures de escenarios, pruebas unitarias, de integración y end-to-end, validación de compatibilidad, documentación de ejecución, control de calidad y preparación de la demo.

Este plan es la vista diaria del rol de Developer 4 dentro de [development-plan.md](../../development-plan.md) y [implementation-checklist.md](../../implementation-checklist.md), que siguen siendo la fuente de verdad sobre pruebas y criterios técnicos exactos. Todos los estados inician en `Pendiente`: no se marcan como completados hasta que exista evidencia verificable, según [definition-of-done.md](../../definition-of-done.md).

| Día | Objetivo | Tareas | Entregable | Dependencias | Criterio de aceptación | Estado |
|---|---|---|---|---|---|---|
| 1 | Contratos congelados y estructura del proyecto | Proponer y cerrar los esquemas Zod y el modelo de estados unificado con aprobación de Dev 1 y Dev 3; crear la estructura de carpetas (`app/`, `src/`, `supabase/`, `mocks/`, `tests/`) | Contratos aprobados por escrito; estructura de carpetas creada | Acuerdo de todo el equipo sobre [api-contracts.md](../../api-contracts.md) y [database.md § 2](../../database.md#2-modelo-de-estados-unificado) | Los cuatro developers confirman por escrito que no necesitan campos adicionales | Pendiente |
| 2 | Fixtures de éxito y bloqueo | Publicar `mocks/payment-approved.json` y `mocks/payment-blocked.json` | Fixtures publicadas | Contratos congelados (día 1) | Cada fixture valida contra su esquema Zod correspondiente | Pendiente |
| 3 | Fixture de fallo de proveedor | Publicar `mocks/investigation-failed.json`; prueba de contrato automatizada sobre los tres fixtures | Fixture de fallo publicada; prueba de contrato pasando | Contratos congelados (día 1) | Los tres fixtures validan contra su esquema correspondiente | Pendiente |
| 4 | Pruebas de integración de ejecuciones | Preparar pruebas de integración para `POST /api/v1/executions` ($8,500 → `ALLOW`, $15,000 sin aprobación → `BLOCK`) | Suite de pruebas de integración | `POST /executions` de Dev 1 | Ambos casos producen el resultado documentado en [testing.md § 2](../../testing.md#2-pruebas-de-integración) | Pendiente |
| 5 | Prueba de idempotencia | Diseñar y ejecutar la prueba de idempotencia bajo la misma `idempotency_key` | Prueba de idempotencia pasando | `idempotency_keys` de Dev 1 | Repetir la misma clave con el mismo payload no crea una segunda fila | Pendiente |
| 6 | Verificación del hito del bloque 2 | Coordinar y verificar el ensayo interno del hito obligatorio del día 6 | Hito del día 6 verificado y registrado | Todo lo anterior del bloque 2 | Demo interna reproduce el hito sin errores | Pendiente |
| 7 | Revisión de la checklist de Nebius | Revisión cruzada de la checklist de compatibilidad de Dev 3 contra la fuente oficial citada | Checklist revisada y aprobada o devuelta con observaciones | Checklist de Dev 3 ([security.md § 1.8](../../security.md#18-lista-de-verificación-de-compatibilidad-con-nebius)) | Cada punto de la checklist tiene una respuesta confirmada, no un supuesto | Pendiente |
| 8 | Pruebas E2E de fallos de proveedor | Diseñar y ejecutar pruebas de timeout, JSON inválido y referencia inválida junto con Dev 3 | Suite E2E de fallos de proveedor | Integración real o mock controlado de fallos | Los tres casos producen `investigation_status = FAILED`/`PARTIAL` sin alterar `decision` | Pendiente |
| 9 | Verificación del hito del bloque 3 | Coordinar y verificar el ensayo interno del hito obligatorio del día 9 | Hito del día 9 verificado y registrado | Todo lo anterior del bloque 3 | Demo interna reproduce ambos caminos (éxito y fallo del proveedor) | Pendiente |
| 10 | Regresión completa | Ejecutar la suite completa (`vitest` + `playwright test`) y registrar defectos encontrados | Reporte de regresión con defectos y severidad | Todo lo anterior | Todas las pruebas de [testing.md](../../testing.md) pasan o tienen un defecto registrado con severidad | Pendiente |
| 11 | Seguimiento de defectos | Dar seguimiento a los defectos registrados el día 10; re-ejecutar pruebas corregidas | Defectos cerrados o priorizados | Reporte de regresión (día 10) | Los defectos de severidad alta quedan cerrados o explícitamente aceptados como riesgo | Pendiente |
| 12 | Verificación del hito del bloque 4 | Coordinar y verificar el ensayo con fallo forzado de Nebius | Hito del día 12 verificado y registrado | Todo lo anterior del bloque 4 | La decisión persistida no cambia cuando se fuerza un fallo de Nebius en el ensayo | Pendiente |
| 13 | Revisión de la congelación de funcionalidades | Revisar que los PRs posteriores al día 13 solo toquen bugs o documentación | Revisión de PRs registrada | Hito del día 12 | Ningún PR revisado agrega funcionalidad nueva | Pendiente |
| 14 | QA final de demo | Apoyar la verificación final del modelo y la UI antes del ensayo de demo | Checklist de QA final completada | Congelación de funcionalidades | No quedan defectos de severidad alta abiertos antes del ensayo de demo | Pendiente |
| 15 | Entrega final y demo | Ensayar el guion de demo cronometrado con datos reiniciables; revisión final de la documentación | Guion de demo ensayado; documentación final revisada | Todo lo anterior | El guion de tres minutos se ejecuta dentro del tiempo con los datos de [demo-script.md § 1](../../demo-script.md#1-preparación) | Pendiente |

## Referencias

- [implementation-checklist.md](../../implementation-checklist.md) — detalle técnico completo por fila
- [testing.md](../../testing.md) — plan de pruebas completo
- [integration-and-dependencies.md](../../integration-and-dependencies.md) — qué necesita Developer 4 de los demás
