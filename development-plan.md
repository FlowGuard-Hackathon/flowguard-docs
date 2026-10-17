# Plan de desarrollo de 15 días

Este documento es la vista ejecutiva del plan de 15 días. El desglose operativo día a día, por responsable, con pruebas y riesgos, ya existe en [implementation-checklist.md](implementation-checklist.md) y en [architecture.md § Plan de trabajo de 15 días](architecture.md#plan-de-trabajo-de-15-días-resumen); este documento no lo duplica, sino que lo organiza en los cinco bloques solicitados por el equipo y enlaza el plan individual de cada puesto.

El plan individual completo de cada puesto vive en `developers/developer-N/work-plan.md` (`Dev N` = `Puesto N`; ver [README.md](README.md#distribución-del-equipo-cinco-puestos)). El calendario, los cinco bloques y los hitos no cambian con la reorganización del equipo en cinco puestos.

## Bloque 1 — Días 1–3: fundamentos, estructura y contratos

| Campo | Contenido |
|---|---|
| Objetivo | Congelar los contratos compartidos (JSON/Zod) y el modelo de estados unificado; levantar la estructura de carpetas de cada repositorio; dejar el motor de reglas, el frontend y el adaptador de Nebius funcionando contra mocks |
| Hitos | Contratos aprobados por escrito por los cinco puestos; migración inicial aplicada en Supabase de desarrollo; motor de reglas pasa los 10 casos obligatorios |
| Entregables | `payment.schema.ts`, `execution.schema.ts`, `engine.schema.ts`, `evidence-report.schema.ts`; `supabase/migrations/001_initial_schema.sql`; `evaluatePayment` con las cinco reglas; UI con datos mock; adaptador Nebius con respuesta mock; fixtures en `mocks/` |
| Dependencias | Todo el equipo depende de que los contratos se cierren el día 1; Dev 2 y Dev 3 dependen de las fixtures de Dev 5; Dev 1 valida los contratos `payment`, `execution` y `engine` que propone Dev 5 |
| Criterio de aceptación | Ver fila por fila en [implementation-checklist.md § Bloque 1](implementation-checklist.md#bloque-1--días-13-congelación-de-contratos-y-estructura-del-proyecto) |
| Estado | Pendiente |

## Bloque 2 — Días 4–6: primer flujo funcional

| Campo | Contenido |
|---|---|
| Objetivo | Lograr el primer recorrido end-to-end del flujo de aprobación de pagos, con la IA todavía en modo mock |
| Hitos | `POST /api/v1/executions` persiste ejecución, decisión y las cinco evaluaciones; simulador conectado a la API real; worker asíncrono reclamando y resolviendo intentos de investigación |
| Entregables | Endpoint de ejecuciones con idempotencia; herramientas de lectura del agente (`get_execution`, `get_rule_results`, `get_evidence`, `get_payment_context`, `get_rule_details`); worker + reaper + reconciliador; rate limiting aplicado |
| Dependencias | Dev 1 depende de la migración y el motor de reglas del Bloque 1; Dev 2 depende de que `POST /executions` esté disponible; Dev 3 depende de la migración aplicada |
| Criterio de aceptación | Hito obligatorio del día 6: crear una ejecución, evaluar las cinco reglas, persistir la decisión y mostrarla en el panel, con la IA en mock — ver [implementation-checklist.md § Bloque 2](implementation-checklist.md#bloque-2--días-46-primer-recorrido-end-to-end-con-ia-simulada) |
| Estado | Pendiente |

## Bloque 3 — Días 7–9: integración del agente de IA

| Campo | Contenido |
|---|---|
| Objetivo | Completar la checklist de compatibilidad con Nebius y sustituir el adaptador mock por la integración real, manteniendo la separación entre decisión (reglas) e investigación (IA) |
| Hitos | Checklist de Nebius confirmada; orquestación real `POST /executions` → `POST /investigate`; centro de evidencias conectado al informe real; pruebas E2E de fallos de proveedor |
| Entregables | Integración real con tool calling y validación de informes contra el esquema; `GET /executions/:id/evidence` mostrando datos reales en la UI |
| Dependencias | Dev 3 depende de credenciales y documentación oficial de Nebius; Dev 1 depende del hito del día 6; Dev 2 depende del contrato del informe del agente |
| Criterio de aceptación | Hito obligatorio del día 9: una ejecución bloqueada produce un informe real o un `investigation_status = FAILED` explícito y correctamente comunicado — ver [implementation-checklist.md § Bloque 3](implementation-checklist.md#bloque-3--días-79-integración-real-de-nebius) |
| Estado | Pendiente |

## Bloque 4 — Días 10–12: seguridad y pruebas

| Campo | Contenido |
|---|---|
| Objetivo | Reforzar consistencia de datos, autorización e idempotencia bajo concurrencia; pulir estados de error en la UI; ajustar límites del agente a valores reales observados; ejecutar la regresión completa |
| Hitos | MVP end-to-end estable incluso si Nebius falla |
| Entregables | Pruebas de concurrencia de idempotencia; UI con estados de carga/error/detalle pulidos; `AGENT_MAX_TOOL_CALLS`/`AGENT_TIMEOUT_MS` calibrados; suite completa (`vitest` + `playwright`) ejecutada por los propietarios de cada módulo, con coordinación y registro de defectos transversales por Dev 5 |
| Dependencias | Todo el bloque depende del hito del día 9 |
| Criterio de aceptación | Hito obligatorio del día 12: se fuerza un fallo de Nebius en el ensayo y la decisión persistida no cambia — ver [implementation-checklist.md § Bloque 4](implementation-checklist.md#bloque-4--días-1012-resiliencia-y-manejo-de-fallos) |
| Estado | Pendiente |

## Bloque 5 — Días 13–15: estabilización, documentación y demo

| Campo | Contenido |
|---|---|
| Objetivo | Congelar funcionalidades, estabilizar migraciones y UI, verificar el comportamiento ante errores de Nebius, y ensayar la demo final |
| Hitos | Versión estable, pruebas críticas aprobadas, README reproducible, demo ensayada |
| Entregables | Migración verificada en entorno de demo limpio; ajustes visuales finales; plan de contingencia de la demo ensayado; guion de demo cronometrado |
| Dependencias | Todo el equipo depende del hito del día 12; los PRs posteriores al día 13 solo deben tocar bugs o documentación |
| Criterio de aceptación | Hito obligatorio del día 15: versión estable, pruebas críticas aprobadas, README reproducible y demo ensayada — ver [implementation-checklist.md § Bloque 5](implementation-checklist.md#bloque-5--días-1315-congelación-y-entrega) y [demo-script.md](demo-script.md) |
| Estado | Pendiente |

## Referencias

- Desglose día a día, responsable, prueba y riesgo de cada entregable: [implementation-checklist.md](implementation-checklist.md)
- Resumen arquitectónico de los cinco bloques: [architecture.md § Plan de trabajo de 15 días](architecture.md#plan-de-trabajo-de-15-días-resumen)
- Plan individual de cada puesto: `developers/developer-1/work-plan.md`, `developers/developer-2/work-plan.md`, `developers/developer-3/work-plan.md`, `developers/developer-4/work-plan.md` (plan de CI/CD **propuesto, pendiente de aprobación**), `developers/developer-5/work-plan.md` (plan flexible, sin calendario fijo)
- Criterios para considerar cerrado un bloque o una tarea: [definition-of-done.md](definition-of-done.md)
- Cómo colaboran los cinco puestos entre bloques: [integration-and-dependencies.md](integration-and-dependencies.md)
