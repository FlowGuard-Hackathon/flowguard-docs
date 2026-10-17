# Developer 5 (Puesto 5) — Contratos, mocks y apoyo general

Página de inicio del espacio de trabajo de Developer 5 (Puesto 5). La persona que ocupará este puesto aún no está asignada.

## Archivos de esta carpeta

- [work-plan.md](work-plan.md) — plan flexible basado en contratos pendientes, fixtures necesarios y bloqueos reales (sin calendario fijo de quince días)
- [daily-log.md](daily-log.md) — bitácora diaria
- [notes.md](notes.md) — cuaderno técnico personal
- [blockers.md](blockers.md) — bloqueos registrados

## Objetivo del puesto

Este puesto tiene dos responsabilidades base y una transversal:

1. **Contratos compartidos:** mantener su documentación y ejemplos, y coordinar la compatibilidad entre frontend, backend y agente.
2. **Mocks y datos simulados:** crear y mantener fixtures reproducibles en `flowguard-mocks`.
3. **Apoyo general:** ayudar temporalmente a quien tenga un bloqueo técnico, sin abandonar las dos responsabilidades base.

No tiene propiedad permanente sobre el backend, el frontend, el agente de IA ni la CI/CD.

## Contratos compartidos

- Propone y mantiene `payment.schema.ts`, `execution.schema.ts` y `engine.schema.ts`; el puesto 1 los valida.
- `src/contracts/evidence-report.schema.ts` es propiedad del puesto 3. Este puesto no lo modifica sin coordinación y aprobación del puesto 3 y de los demás responsables afectados.
- Registra las propuestas de modificación y coordina su aprobación con los responsables afectados. No cambia endpoints, campos, estados, esquemas ni formatos de respuesta por su cuenta, no decide la semántica de las reglas de negocio y no sustituye decisiones de arquitectura.
- Mantener la documentación de un contrato no otorga autoridad exclusiva para cambiarlo: las decisiones técnicas corresponden a los responsables de los módulos afectados.

## Mocks y datos simulados

- Repositorio principal: [`flowguard-mocks`](https://github.com/FlowGuard-Hackathon/flowguard-mocks).
- Datos reproducibles de solicitudes de pago, facturas, presupuestos y autorizaciones; escenarios válidos, inválidos e incompletos; fixtures coherentes con los contratos aprobados.
- Coordina con Dev 1 los escenarios del backend, con Dev 2 los datos para la interfaz y con Dev 3 los datos para probar el agente, y entrega fixtures a Dev 4 para que los pipelines ejecuten las verificaciones.
- Los mocks son datos simulados, no evidencias reales; no sirven para afirmar que una integración real con Nebius o Supabase fue validada.

## Suite compartida y defectos transversales

- Coordina la suite compartida de [`flowguard-tests`](https://github.com/FlowGuard-Hackathon/flowguard-tests) y mantiene las pruebas de contrato y compatibilidad. **No** implementa todas las pruebas de todos los módulos: cada propietario mantiene las de su módulo (ver [testing.md](../../testing.md#responsables-de-las-pruebas)).
- Registra los defectos transversales con pasos para reproducirlos; el propietario del módulo afectado los corrige.

## Apoyo general: cómo solicitarlo

Quien se atasca indica: módulo y archivo afectados, comportamiento esperado y observado, error obtenido, pasos para reproducirlo, acciones ya intentadas, ayuda concreta que necesita, y urgencia y tareas bloqueadas. El puesto 5 acuerda el alcance con el responsable, investiga o implementa junto con él, ejecuta las verificaciones y deja el resultado documentado en su bitácora. El responsable original valida la solución. Detalle: [work-plan.md](work-plan.md#2-apoyo-general-proceso-por-solicitud) y [integration-and-dependencies.md](../../integration-and-dependencies.md#procedimiento-de-apoyo-entre-puestos).

## Límites de autoridad

- No cambia unilateralmente contratos aprobados.
- No asume CI/CD, despliegues, DevOps ni la propiedad exclusiva de todo el QA funcional.
- No modifica código de otro módulo sin coordinar el alcance con su propietario ni se apropia de él.
- El apoyo general no debe desplazar sistemáticamente el trabajo de contratos y mocks.
- No aprueba en solitario los hitos de los días 6, 9, 12 y 15: se verifican de forma cruzada.

## Repositorios en los que puede colaborar

Principales: `flowguard-mocks` y la documentación compartida en `flowguard-docs`. Puede ayudar en cualquiera de los demás, lo que no le otorga propiedad permanente sobre ellos.

## Cómo registrar y cerrar bloqueos

Los bloqueos reales se registran en [blockers.md](blockers.md) (nunca bloqueos ficticios). Un bloqueo se cierra cuando el responsable del módulo valida la solución y se anota la fecha y el resultado. Lo que se resuelve en una sola intervención queda en [daily-log.md](daily-log.md).

## Reglas de actualización y colaboración

1. Actualiza tu bitácora ([daily-log.md](daily-log.md)) al terminar tu jornada.
2. Tus notas ([notes.md](notes.md)) deben incluir contexto suficiente para que otro integrante las entienda sin preguntar.
3. Registra y comunica los bloqueos importantes en [blockers.md](blockers.md).
4. Las decisiones que afecten a la arquitectura o a varios módulos también deben reflejarse en la documentación general (`development-plan.md`, `integration-and-dependencies.md`, o el documento técnico correspondiente).
5. Una tarea no se considera completada solo porque esté anotada como terminada: debe existir evidencia verificable, según [definition-of-done.md](../../definition-of-done.md).
6. Los archivos de esta carpeta pueden modificarse mediante Pull Requests pequeños.
7. Los documentos generales de la raíz del repositorio son compartidos y no deben contradecir los contratos técnicos existentes.

Estos archivos Markdown son compartidos: los cinco puestos pueden proponer cambios si tienen permisos de escritura en GitHub. La carpeta organiza el trabajo, no establece permisos privados.
