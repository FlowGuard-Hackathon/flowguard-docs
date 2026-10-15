# Developer 4 — Contratos, mocks, pruebas e integración

Página de inicio del espacio de trabajo de Developer 4.

## Archivos de esta carpeta

- [work-plan.md](work-plan.md) — plan completo de los 15 días
- [daily-log.md](daily-log.md) — bitácora diaria
- [notes.md](notes.md) — cuaderno técnico personal
- [blockers.md](blockers.md) — bloqueos registrados

## Módulo que posee

Los contratos compartidos y la calidad del proyecto: los esquemas Zod, las fixtures de escenarios, las pruebas unitarias, de integración y end-to-end, la validación de compatibilidad entre módulos, la documentación de ejecución, el control de calidad y la preparación de la demo. Repositorios principales: [`flowguard-mocks`](https://github.com/FlowGuard-Hackathon/flowguard-mocks) y [`flowguard-tests`](https://github.com/FlowGuard-Hackathon/flowguard-tests).

## Responsabilidades

- Proponer y cerrar los contratos compartidos (`payment.schema.ts`, `execution.schema.ts`, `engine.schema.ts`, `evidence-report.schema.ts`) con aprobación de Developer 1 y Developer 3.
- Publicar y mantener las fixtures de escenarios en `mocks/`.
- Construir y mantener la suite de pruebas unitarias, de integración y E2E en `flowguard-tests`.
- Verificar los hitos obligatorios de cada bloque mediante ensayos internos.
- Coordinar la regresión completa y el registro de defectos.
- Preparar y ensayar el guion de demo final.

## Cómo coordinarse con el resto del equipo

- Coordinas la congelación de contratos el día 1: necesitas confirmación de Developer 1 y Developer 3 de que los contratos cubren lo que necesitan antes de cerrarlos.
- Developer 2 depende de tus fixtures desde el día 1 para construir la UI sin esperar a la API real.
- Verificas, junto con cada developer, que los hitos obligatorios de cada bloque (días 6, 9, 12 y 15) se cumplen con evidencia real, no solo con la anotación de "completado".
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
