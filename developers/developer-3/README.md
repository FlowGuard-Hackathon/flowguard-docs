# Developer 3 (Puesto 3) — Agente de IA, LLMs, Nebius Token Factory y modelos NVIDIA

Página de inicio del espacio de trabajo de Developer 3 (Puesto 3). Este puesto está reservado para quien coordina el proyecto y permanece dedicado al agente de IA y a los LLMs.

## Archivos de esta carpeta

- [work-plan.md](work-plan.md) — plan completo de los 15 días
- [daily-log.md](daily-log.md) — bitácora diaria
- [notes.md](notes.md) — cuaderno técnico personal
- [blockers.md](blockers.md) — bloqueos registrados

## Módulo que posee

El agente de IA de FlowGuard: el contrato del informe del agente, el proveedor simulado (mock) y la integración real con Nebius Token Factory, las herramientas de solo lectura, el ciclo de investigación, la validación de respuestas y la gestión de fallos del proveedor. Módulo dentro de [`flowguard-src`](https://github.com/FlowGuard-Hackathon/flowguard-src).

**Rutas de propiedad en `flowguard-src`:** `src/server/agent/`, `src/server/tools/` y `src/contracts/evidence-report.schema.ts`. El resto del backend es del puesto 1. Las rutas definen propiedad de mantenimiento, no permiso para trabajar sin coordinación. El puesto 5 puede ayudar con la compatibilidad de los contratos compartidos, pero no modifica `evidence-report.schema.ts` por su cuenta ni se apropia de él.

## Responsabilidades

- Definir y mantener el contrato del informe del agente ([api-contracts.md § 10](../../api-contracts.md#10-contrato-del-informe-del-agente)).
- Integrar el modelo NVIDIA de código abierto compatible con los requisitos del hackathon y documentar, con evidencia verificable, las llamadas reales a Nebius Token Factory y el uso de ese modelo.
- Mantener los prompts y las pruebas del agente y del proveedor (respuestas válidas, JSON inválido, referencias inválidas, timeouts y errores del proveedor), y verificar que el sistema siga funcionando con el proveedor simulado cuando no haya credenciales reales.
- Construir el adaptador de Nebius, primero en modo mock y después con la integración real.
- Implementar las herramientas de lectura (`get_execution`, `get_rule_results`, `get_evidence`, `get_payment_context`, `get_rule_details`).
- Completar la checklist de compatibilidad con Nebius ([security.md § 1.8](../../security.md#18-lista-de-verificación-de-compatibilidad-con-nebius)).
- Validar los informes del agente y gestionar timeouts, JSON inválido y referencias inválidas sin alterar la `decision` ya persistida.

## Cómo coordinarse con el resto del equipo

- Dependes de Developer 5 para la coordinación de compatibilidad del contrato del informe del agente, que se congela el día 1, y para la fixture `investigation-failed.json`. Tú sigues siendo el propietario del esquema.
- Dependes de Developer 1 para que la migración esté aplicada y `POST /executions` esté disponible, de forma que las herramientas de lectura tengan datos reales contra los que operar. Tú defines qué información necesita el agente; Developer 1 mantiene el acceso a datos. No creas mecanismos alternativos de acceso a datos para eludir la arquitectura, y Developer 1 no cambia unilateralmente los contratos de tus herramientas.
- Developer 4 automatiza en CI las verificaciones del agente, pero las pruebas del agente las mantienes tú. No expongas credenciales en pipelines ni en logs.
- Los hitos de los días 6, 9, 12 y 15 se verifican de forma cruzada; no apruebas en solitario los componentes que desarrollaste.
- Además de tu rol técnico, coordinas las prioridades generales del proyecto. Eso **no** te hace responsable de todas las tareas de entrega: la demo, el README final, el video y las evidencias se reparten entre los puestos (ver [definition-of-done.md](../../definition-of-done.md#requisitos-del-hackathon-y-evidencias-de-cumplimiento)).
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

Estos archivos Markdown son compartidos: los cinco puestos pueden proponer cambios si tienen permisos de escritura en GitHub. La carpeta organiza el trabajo, no establece permisos privados.
