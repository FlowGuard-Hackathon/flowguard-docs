# Developer 1 (Puesto 1) — Backend, motor determinista y base de datos

Página de inicio del espacio de trabajo de Developer 1 (Puesto 1). La persona que ocupará este puesto aún no está asignada.

## Archivos de esta carpeta

- [work-plan.md](work-plan.md) — plan completo de los 15 días
- [daily-log.md](daily-log.md) — bitácora diaria
- [notes.md](notes.md) — cuaderno técnico personal
- [blockers.md](blockers.md) — bloqueos registrados

## Módulo que posee

Backend de FlowGuard: API con Next.js Route Handlers, validación con Zod, el motor determinista de las cinco reglas de negocio, la persistencia en Supabase/PostgreSQL, la idempotencia, el procesamiento asíncrono del worker de investigación, y la seguridad y manejo de errores del backend. Repositorios principales: [`flowguard-src`](https://github.com/FlowGuard-Hackathon/flowguard-src) (todo el backend **excepto** `src/server/agent/`, `src/server/tools/` y `src/contracts/evidence-report.schema.ts`, que son del puesto 3) y [`flowguard-supabase`](https://github.com/FlowGuard-Hackathon/flowguard-supabase).

## Responsabilidades

- Implementar y mantener el motor de reglas (`evaluatePayment`) y las cinco reglas `PAY-001` a `PAY-005`.
- Diseñar e implementar el esquema de base de datos y aplicar las migraciones.
- Implementar los endpoints de la API (`POST /executions`, `GET /executions`, etc.) según [api-contracts.md](../../api-contracts.md).
- Implementar el worker asíncrono, el reaper y el reconciliador de ejecuciones.
- Aplicar rate limiting y los controles de seguridad del backend descritos en [security.md](../../security.md).
- Validar los contratos compartidos `payment`, `execution` y `engine` que propone el puesto 5.
- Mantener las pruebas del motor de reglas, la persistencia y la integración del backend.
- Implementar el middleware de Basic Auth de la demo y generar y distribuir las credenciales por un canal apropiado, sin guardarlas en Git ni imprimirlas en logs ([security.md § 2.0](../../security.md#20-estrategia-de-seguridad-por-entorno)).
- Coordinar la integración técnica del backend en el repositorio final `flowguard`, siguiendo la estrategia `git subtree` ya documentada.
- Colaborar con el puesto 4 para que los pipelines de CI ejecuten las verificaciones pertinentes del backend, y ser responsable técnico de la ejecución y configuración propia del backend.

## Cómo coordinarse con el resto del equipo

- Developer 2 depende de que los endpoints de la API estén disponibles y respeten el contrato congelado — cualquier cambio de contrato se documenta primero en este repositorio.
- Developer 3 depende de la migración aplicada para que las herramientas de lectura del agente tengan datos reales contra los que operar. Tú mantienes el acceso a datos; Developer 3 define qué información necesita el agente. Acuerden las interfaces antes de implementar (ver [integration-and-dependencies.md](../../integration-and-dependencies.md)).
- Developer 5 propone los contratos `payment`, `execution` y `engine` para que los valides, y mantiene las fixtures que necesita el backend.
- Developer 4 automatiza en CI las verificaciones de tu módulo; las pruebas del backend las mantienes tú. Los hitos de los días 6, 9, 12 y 15 se verifican de forma cruzada, no por una sola persona.
- Ver el detalle completo de dependencias en [integration-and-dependencies.md](../../integration-and-dependencies.md).

## Límites

No delegues al LLM la decisión final de autorización, no cambies las reglas de negocio ni los contratos compartidos por tu cuenta, y no asumas el trabajo del frontend o del agente de IA.

## Reglas de actualización y colaboración

1. Actualiza tu bitácora ([daily-log.md](daily-log.md)) al terminar tu jornada.
2. Tus notas ([notes.md](notes.md)) deben incluir contexto suficiente para que otro integrante las entienda sin preguntar.
3. Registra y comunica los bloqueos importantes en [blockers.md](blockers.md).
4. Las decisiones que afecten a la arquitectura o a varios módulos también deben reflejarse en la documentación general (`development-plan.md`, `integration-and-dependencies.md`, o el documento técnico correspondiente).
5. Una tarea no se considera completada solo porque esté anotada como terminada: debe existir evidencia verificable, según [definition-of-done.md](../../definition-of-done.md).
6. Los archivos de esta carpeta pueden modificarse mediante Pull Requests pequeños.
7. Los documentos generales de la raíz del repositorio son compartidos y no deben contradecir los contratos técnicos existentes.

Estos archivos Markdown son compartidos: los cinco puestos pueden proponer cambios si tienen permisos de escritura en GitHub. La carpeta organiza el trabajo, no establece permisos privados.
