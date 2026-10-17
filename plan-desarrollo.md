# Plan de desarrollo de 15 días

Este documento es la vista ejecutiva del plan de 15 días. El desglose operativo día a día, por responsable, con pruebas y riesgos, ya existe en [checklist-implementacion.md](checklist-implementacion.md) y en [arquitectura.md § Plan de trabajo de 15 días](arquitectura.md#plan-de-trabajo-de-15-días-resumen); este documento no lo duplica, sino que lo organiza en los cinco bloques solicitados por el equipo y enlaza el plan individual de cada puesto.

El plan individual completo de cada puesto vive en `desarrolladores/desarrollador-N/plan-trabajo.md` (`Dev N` = `Puesto N`; ver [README.md](README.md#distribución-del-equipo-cinco-puestos)). El calendario, los cinco bloques y los hitos no cambian con la reorganización del equipo en cinco puestos.

## Bloque 1 — Días 1–3: fundamentos, estructura y contratos

| Campo | Contenido |
|---|---|
| Objetivo | Congelar los contratos compartidos (JSON/Zod) y el modelo de estados unificado; levantar la estructura de carpetas de cada repositorio; dejar el motor de reglas, el frontend y el adaptador de Nebius funcionando contra mocks |
| Hitos | Contratos aprobados por escrito por los cinco puestos; migración inicial aplicada en Supabase de desarrollo; motor de reglas pasa los 10 casos obligatorios |
| Entregables | `pago.schema.ts`, `ejecucion.schema.ts`, `motor.schema.ts`, `informe-evidencias.schema.ts`; `supabase/migrations/001_esquema_inicial.sql`; `evaluarPago` con las cinco reglas; UI con datos mock; adaptador Nebius con respuesta mock; fixtures en `mocks/` |
| Dependencias | Todo el equipo depende de que los contratos se cierren el día 1; Dev 2 y Dev 3 dependen de las fixtures de Dev 5; Dev 1 valida los contratos `pago`, `ejecucion` y `motor` que propone Dev 5 |
| Criterio de aceptación | Ver fila por fila en [checklist-implementacion.md § Bloque 1](checklist-implementacion.md#bloque-1--días-13-congelación-de-contratos-y-estructura-del-proyecto) |
| Estado | Pendiente |

## Bloque 2 — Días 4–6: primer flujo funcional

| Campo | Contenido |
|---|---|
| Objetivo | Lograr el primer recorrido end-to-end del flujo de aprobación de pagos, con la IA todavía en modo mock |
| Hitos | `POST /api/v1/ejecuciones` persiste ejecución, decisión y las cinco evaluaciones; simulador conectado a la API real; worker asíncrono reclamando y resolviendo intentos de investigación |
| Entregables | Endpoint de ejecuciones con idempotencia; herramientas de lectura del agente (`obtener_ejecucion`, `obtener_resultados_reglas`, `obtener_evidencia`, `obtener_contexto_pago`, `obtener_detalle_regla`); worker + reaper + reconciliador; rate limiting aplicado |
| Dependencias | Dev 1 depende de la migración y el motor de reglas del Bloque 1; Dev 2 depende de que `POST /ejecuciones` esté disponible; Dev 3 depende de la migración aplicada |
| Criterio de aceptación | Hito obligatorio del día 6: crear una ejecución, evaluar las cinco reglas, persistir la decisión y mostrarla en el panel, con la IA en mock — ver [checklist-implementacion.md § Bloque 2](checklist-implementacion.md#bloque-2--días-46-primer-recorrido-end-to-end-con-ia-simulada) |
| Estado | Pendiente |

## Bloque 3 — Días 7–9: integración del agente de IA

| Campo | Contenido |
|---|---|
| Objetivo | Completar la checklist de compatibilidad con Nebius y sustituir el adaptador mock por la integración real, manteniendo la separación entre decisión (reglas) e investigación (IA) |
| Hitos | Checklist de Nebius confirmada; orquestación real `POST /ejecuciones` → `POST /investigar`; centro de evidencias conectado al informe real; pruebas E2E de fallos de proveedor |
| Entregables | Integración real con tool calling y validación de informes contra el esquema; `GET /ejecuciones/:id/evidencias` mostrando datos reales en la UI |
| Dependencias | Dev 3 depende de credenciales y documentación oficial de Nebius; Dev 1 depende del hito del día 6; Dev 2 depende del contrato del informe del agente |
| Criterio de aceptación | Hito obligatorio del día 9: una ejecución bloqueada produce un informe real o un `estado_investigacion = FALLIDA` explícito y correctamente comunicado — ver [checklist-implementacion.md § Bloque 3](checklist-implementacion.md#bloque-3--días-79-integración-real-de-nebius) |
| Estado | Pendiente |

## Bloque 4 — Días 10–12: seguridad y pruebas

| Campo | Contenido |
|---|---|
| Objetivo | Reforzar consistencia de datos, autorización e idempotencia bajo concurrencia; pulir estados de error en la UI; ajustar límites del agente a valores reales observados; ejecutar la regresión completa |
| Hitos | MVP end-to-end estable incluso si Nebius falla |
| Entregables | Pruebas de concurrencia de idempotencia; UI con estados de carga/error/detalle pulidos; `AGENTE_MAX_LLAMADAS_HERRAMIENTAS`/`AGENTE_TIEMPO_LIMITE_MS` calibrados; suite completa (`vitest` + `playwright`) ejecutada por los propietarios de cada módulo, con coordinación y registro de defectos transversales por Dev 5 |
| Dependencias | Todo el bloque depende del hito del día 9 |
| Criterio de aceptación | Hito obligatorio del día 12: se fuerza un fallo de Nebius en el ensayo y la decisión persistida no cambia — ver [checklist-implementacion.md § Bloque 4](checklist-implementacion.md#bloque-4--días-1012-resiliencia-y-manejo-de-fallos) |
| Estado | Pendiente |

## Bloque 5 — Días 13–15: estabilización, documentación y demo

| Campo | Contenido |
|---|---|
| Objetivo | Congelar funcionalidades, estabilizar migraciones y UI, verificar el comportamiento ante errores de Nebius, y ensayar la demo final |
| Hitos | Versión estable, pruebas críticas aprobadas, README reproducible, demo ensayada |
| Entregables | Migración verificada en entorno de demo limpio; ajustes visuales finales; plan de contingencia de la demo ensayado; guion de demo cronometrado |
| Dependencias | Todo el equipo depende del hito del día 12; los PRs posteriores al día 13 solo deben tocar bugs o documentación |
| Criterio de aceptación | Hito obligatorio del día 15: versión estable, pruebas críticas aprobadas, README reproducible y demo ensayada — ver [checklist-implementacion.md § Bloque 5](checklist-implementacion.md#bloque-5--días-1315-congelación-y-entrega) y [guion-demo.md](guion-demo.md) |
| Estado | Pendiente |

## Referencias

- Desglose día a día, responsable, prueba y riesgo de cada entregable: [checklist-implementacion.md](checklist-implementacion.md)
- Resumen arquitectónico de los cinco bloques: [arquitectura.md § Plan de trabajo de 15 días](arquitectura.md#plan-de-trabajo-de-15-días-resumen)
- Plan individual de cada puesto: `desarrolladores/desarrollador-1/plan-trabajo.md`, `desarrolladores/desarrollador-2/plan-trabajo.md`, `desarrolladores/desarrollador-3/plan-trabajo.md`, `desarrolladores/desarrollador-4/plan-trabajo.md` (plan de CI/CD **propuesto, pendiente de aprobación**), `desarrolladores/desarrollador-5/plan-trabajo.md` (plan flexible, sin calendario fijo)
- Criterios para considerar cerrado un bloque o una tarea: [definicion-de-terminado.md](definicion-de-terminado.md)
- Cómo colaboran los cinco puestos entre bloques: [integracion-y-dependencias.md](integracion-y-dependencias.md)
