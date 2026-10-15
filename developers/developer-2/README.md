# Developer 2 — Frontend y UX

Página de inicio del espacio de trabajo de Developer 2.

## Archivos de esta carpeta

- [work-plan.md](work-plan.md) — plan completo de los 15 días
- [daily-log.md](daily-log.md) — bitácora diaria
- [notes.md](notes.md) — cuaderno técnico personal
- [blockers.md](blockers.md) — bloqueos registrados

## Módulo que posee

La aplicación Next.js de FlowGuard: el formulario de pagos simulados, el dashboard de evaluaciones, el historial y detalle de pagos, la visualización de reglas, evidencias, bloqueos y errores, y la integración con la API desde el frontend. Repositorio principal: [`flowguard-app`](https://github.com/FlowGuard-Hackathon/flowguard-app).

## Responsabilidades

- Construir el simulador de pagos y conectarlo a la API real.
- Construir el dashboard de ejecuciones, el historial y la vista de detalle.
- Construir el centro de evidencias que muestra reglas, evidencias e informe del agente sin datos inventados.
- Representar correctamente todos los estados de interfaz (carga, error, bloqueo) descritos en [architecture.md § 6.4](../../architecture.md#64-estados-de-interfaz).
- Mantener pruebas de interfaz (Playwright) contra mocks y contra la API real.

## Cómo coordinarse con el resto del equipo

- Desde el día 1 trabajas contra las fixtures de Developer 4 (`mocks/payment-approved.json`, `mocks/payment-blocked.json`), sin esperar a que la API real exista.
- Dependes de Developer 1 para que `POST /api/v1/executions` y los demás endpoints estén disponibles antes de conectar el simulador real.
- Dependes de Developer 3 para el contrato del informe del agente y para que `GET /executions/:id/evidence` devuelva datos reales.
- Ver el detalle completo de dependencias en [integration-and-dependencies.md](../../integration-and-dependencies.md).

## Reglas de actualización y colaboración

1. Actualiza tu bitácora ([daily-log.md](daily-log.md)) al terminar tu jornada.
2. Tus notas ([notes.md](notes.md)) deben incluir contexto suficiente para que otro integrante las entienda sin preguntar.
3. Registra y comunica los bloqueos importantes en [blockers.md](blockers.md).
4. Las decisiones que afecten a la arquitectura o a varios módulos también deben reflejarse en la documentación general (`development-plan.md`, `integration-and-dependencies.md`, o el documento técnico correspondiente).
5. Una tarea no se considera completada solo porque esté anotada como terminada: debe existir evidencia verificable, según [definition-of-done.md](../../definition-of-done.md).
6. Los archivos de esta carpeta pueden modificarse mediante Pull Requests pequeños.
7. Los documentos generales de la raíz del repositorio son compartidos y no deben contradecir los contratos técnicos existentes.

Estos archivos Markdown son compartidos: los cuatro integrantes pueden proponer cambios si tienen permisos de escritura en GitHub. La carpeta organiza el trabajo, no establece permisos privados.
