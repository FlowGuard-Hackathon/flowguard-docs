# Developer 3 — Agente de IA y Nebius

Página de inicio del espacio de trabajo de Developer 3.

## Archivos de esta carpeta

- [work-plan.md](work-plan.md) — plan completo de los 15 días
- [daily-log.md](daily-log.md) — bitácora diaria
- [notes.md](notes.md) — cuaderno técnico personal
- [blockers.md](blockers.md) — bloqueos registrados

## Módulo que posee

El agente de IA de FlowGuard: el contrato del informe del agente, el proveedor simulado (mock) y la integración real con Nebius Token Factory, las herramientas de solo lectura, el ciclo de investigación, la validación de respuestas y la gestión de fallos del proveedor. Módulo dentro de [`flowguard-src`](https://github.com/FlowGuard-Hackathon/flowguard-src).

## Responsabilidades

- Definir y mantener el contrato del informe del agente ([api-contracts.md § 10](../../api-contracts.md#10-contrato-del-informe-del-agente)).
- Construir el adaptador de Nebius, primero en modo mock y después con la integración real.
- Implementar las herramientas de lectura (`get_execution`, `get_rule_results`, `get_evidence`, `get_payment_context`, `get_rule_details`).
- Completar la checklist de compatibilidad con Nebius ([security.md § 1.8](../../security.md#18-lista-de-verificación-de-compatibilidad-con-nebius)).
- Validar los informes del agente y gestionar timeouts, JSON inválido y referencias inválidas sin alterar la `decision` ya persistida.

## Cómo coordinarse con el resto del equipo

- Dependes de Developer 4 para el contrato del informe del agente congelado desde el día 1.
- Dependes de Developer 1 para que la migración esté aplicada y `POST /executions` esté disponible, de forma que las herramientas de lectura tengan datos reales contra los que operar.
- Developer 2 depende de tu contrato del informe y de `GET /executions/:id/evidence` para construir el centro de evidencias.
- La IA investiga y explica, pero **no puede** autorizar pagos, modificar reglas ni ejecutar operaciones de escritura — es una regla de diseño no negociable.
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
