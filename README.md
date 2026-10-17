# FlowGuard — Documentación técnica

**Versión:** 1.0
**Fecha:** 15 de octubre de 2026
**Tipo de documento:** Especificación técnica previa a la implementación
**Equipo:** 5 puestos (ver [Distribución del equipo](#distribución-del-equipo-cinco-puestos))
**Duración:** 15 días
**Alcance:** MVP funcional para un proceso de aprobación de pagos simulados.

## Resumen del proyecto

FlowGuard es un agente de IA que supervisa procesos automatizados, detecta infracciones de reglas, verifica acciones y evita que los errores avancen, proporcionando evidencia verificable sobre lo ocurrido.

El MVP implementa un único proceso: aprobación de solicitudes de pago. Por ejemplo, una solicitud de $15,000 MXN requiere autorización de un supervisor porque supera el umbral configurado de $10,000 MXN.

La solución demuestra cuatro capacidades:

1. Evaluar reglas de negocio mediante código determinista.
2. Bloquear una operación cuando no cumple las condiciones obligatorias.
3. Investigar la infracción mediante un agente conectado a Nebius Token Factory.
4. Presentar evidencia persistida, explicación y una acción correctiva propuesta.

El sistema **no** realiza transferencias bancarias reales.

La regla principal de diseño es innegociable: **el motor de reglas decide, el backend persiste, la IA investiga y el frontend presenta.**

## Índice de documentos

> **Identificadores en español (2026-10-17):** todos los nombres del sistema (API, base de datos, códigos de error, variables de entorno, código y documentos) pasan al español según [glosario.md](glosario.md). La migración se hace por repositorio; mientras tanto, si un documento usa un nombre en inglés, manda el glosario.

| Documento | Contenido |
|---|---|
| [glosario.md](glosario.md) | Fuente única de nombres: endpoints, campos, tablas, estados, códigos de error, variables de entorno, archivos y código |
| [convencion-commits.md](convencion-commits.md) | Formato de los mensajes de commit (`feat`, `fix`, `docs`…) en español |
| [convencion-ramas.md](convencion-ramas.md) | Formato de los nombres de rama (`tipo/devN-descripcion`) y reglas de uso |
| [architecture.md](architecture.md) | Decisiones arquitectónicas, stack, estructura del repositorio, flujo de datos, resiliencia y resumen del plan de 15 días |
| [api-contracts.md](api-contracts.md) | Modelo de estados, contratos JSON por endpoint, idempotencia, conversión monetaria, formato de error |
| [rules.md](rules.md) | Motor de reglas deterministas, las cinco reglas de negocio, casos de prueba obligatorios |
| [database.md](database.md) | Modelo de datos completo (DDL por tabla), modelo de estados unificado, persistencia segura |
| [security.md](security.md) | Servicio de IA (herramientas, ciclo, prompt, validación), estrategia de seguridad por entorno, checklist de Nebius |
| [error-catalog.md](error-catalog.md) | Catálogo cerrado de códigos de error, su efecto sobre `decision` y comportamiento esperado del frontend |
| [testing.md](testing.md) | Plan de pruebas (unitarias, integración, E2E) y criterios de aceptación |
| [implementation-checklist.md](implementation-checklist.md) | Checklist día a día: entregable, responsable, dependencias, criterio de aceptación, prueba, riesgos |
| [demo-script.md](demo-script.md) | Guion cronometrado de la demo del hackathon y plan de contingencia |
| [development-plan.md](development-plan.md) | Vista ejecutiva del plan de 15 días por bloques, con enlaces al plan individual de cada developer |
| [integration-and-dependencies.md](integration-and-dependencies.md) | Cómo colaboran los cinco puestos, qué contratos comparten y qué necesitan entre sí |
| [definition-of-done.md](definition-of-done.md) | Checklist operativa para considerar terminada una tarea o un hito de bloque |

## Distribución del equipo (cinco puestos)

El equipo se organiza en exactamente cinco puestos. **`Dev N` / `Developer N` es el nombre corto del `Puesto N`** (por ejemplo, `Dev 4` = `Puesto 4`); la nomenclatura `Dev N` se conserva en todos los documentos. Las menciones históricas de `Dev 4` en este repositorio se refieren al puesto 4 *anterior* (contratos, mocks y pruebas) solo cuando el texto lo indica expresamente; las tareas que antes tenía ese puesto fueron reasignadas fila por fila (ver [implementation-checklist.md](implementation-checklist.md) y [developers/developer-4/work-plan.md](developers/developer-4/work-plan.md)).

Los puestos 1, 2, 4 y 5 **no tienen todavía una persona asignada**. El puesto 3 está reservado para quien coordina el proyecto. Estos nombres (`Developer 1` a `Developer 5`) se usan hasta que se conozcan los usuarios reales de GitHub de cada integrante.

| Puesto | Responsabilidad | Carpeta |
|---|---|---|
| Puesto 1 (Dev 1) | Backend, motor de reglas determinista y base de datos | [developers/developer-1/](developers/developer-1/README.md) |
| Puesto 2 (Dev 2) | Frontend y experiencia de usuario | [developers/developer-2/](developers/developer-2/README.md) |
| Puesto 3 (Dev 3) | Agente de IA, LLMs, Nebius Token Factory y modelos NVIDIA | [developers/developer-3/](developers/developer-3/README.md) |
| Puesto 4 (Dev 4) | CI/CD exclusivamente | [developers/developer-4/](developers/developer-4/README.md) |
| Puesto 5 (Dev 5) | Contratos compartidos, mocks y apoyo general | [developers/developer-5/](developers/developer-5/README.md) |

Cada puesto tiene su propia carpeta en `developers/` con su plan de trabajo, bitácora diaria, notas técnicas y bloqueos.

### Límites de cada puesto

| Puesto | Lo que NO hace |
|---|---|
| 1 | No delega al LLM la decisión de autorización; no cambia reglas de negocio ni contratos compartidos por su cuenta |
| 2 | No implementa una segunda versión de las reglas ni autoriza pagos desde el frontend; no inventa endpoints ni esquemas; no expone credenciales en el navegador |
| 3 | El agente no autoriza pagos, no cambia reglas, no escribe en la base de datos mediante herramientas y no sustituye al motor determinista |
| 4 | No es responsable funcional de QA, de los contratos, de los fixtures, de los módulos de aplicación ni de `flowguard-tests`; solo automatiza la ejecución de lo que otros mantienen |
| 5 | No cambia contratos aprobados por su cuenta; no asume CI/CD, despliegues ni todo el QA funcional; el apoyo general no debe desplazar de forma sistemática los contratos y mocks |

### Liderazgo y coordinación

Quien ocupa el puesto 3 también coordina las prioridades generales del proyecto, pero eso **no** lo convierte en responsable de todas las tareas de entrega. Cada puesto mantiene la responsabilidad técnica de su módulo, y los cambios que afecten a varios módulos se acuerdan con los responsables afectados. Los cambios de arquitectura quedan fuera de esta organización y requieren un proceso de decisión separado.

### Apoyo entre puestos

Cuando un integrante se atasca, solicita apoyo indicando el módulo y archivo afectados, el comportamiento esperado y el observado, el error, los pasos para reproducirlo, lo que ya intentó y la urgencia. El puesto 5 acuerda el alcance con el responsable del módulo, investiga o implementa junto con él, y el responsable original valida la solución. El responsable siempre conserva la propiedad de su módulo. Detalle en [integration-and-dependencies.md](integration-and-dependencies.md).

## Reglas de actualización y colaboración

1. Cada puesto actualiza su bitácora (`daily-log.md`) al terminar su jornada.
2. Las notas (`notes.md`) deben incluir contexto suficiente para que otro integrante las entienda.
3. Los bloqueos importantes deben registrarse en `blockers.md` y comunicarse al equipo.
4. Las decisiones que afecten a la arquitectura o a varios módulos también deben reflejarse en la documentación general (este README, `development-plan.md`, `integration-and-dependencies.md`, o el documento técnico correspondiente).
5. No se considera completada una tarea solo porque esté anotada como terminada: debe existir evidencia verificable (ver [definition-of-done.md](definition-of-done.md)).
6. Los archivos individuales pueden modificarse mediante Pull Requests pequeños.
7. Los documentos generales son compartidos y no deben contradecir los contratos técnicos existentes.

Los archivos Markdown de este repositorio son compartidos: los cinco puestos pueden proponer cambios si tienen permisos de escritura en GitHub. Las carpetas de `developers/` organizan el trabajo, no establecen permisos privados.

## Otros repositorios de FlowGuard-Hackathon

Este repositorio (`flowguard-docs`) es solo documental. El proyecto completo vive repartido en varios repositorios independientes, cada uno con su propio responsable de mantenimiento. Estas asignaciones indican responsabilidad de mantenimiento, no exclusividad de acceso: cualquier integrante puede revisar o ayudar en un módulo.

| Repositorio | Contenido | Responsable principal | Colaboración |
|---|---|---|---|
| [`flowguard-docs`](https://github.com/FlowGuard-Hackathon/flowguard-docs) | Documentación técnica y organizativa (este repositorio) | Compartida por área: cada puesto mantiene la documentación de su especialidad | Dev 5 coordina la documentación de contratos; Dev 5 ayuda cuando se solicita |
| [`flowguard-supabase`](https://github.com/FlowGuard-Hackathon/flowguard-supabase) | Migración SQL de referencia (`migrations/001_initial_schema.sql`) correspondiente a [database.md](database.md) | Dev 1 | Dev 4 automatiza las verificaciones pertinentes; Dev 5 ayuda con escenarios y contratos |
| [`flowguard-src`](https://github.com/FlowGuard-Hackathon/flowguard-src) | Contratos compartidos, motor de reglas, persistencia, agente de IA | Dev 1 (backend, reglas y persistencia) y Dev 3 (agente y herramientas), por rutas: ver tabla siguiente | Dev 5 coordina los contratos compartidos; Dev 4 configura CI |
| [`flowguard-app`](https://github.com/FlowGuard-Hackathon/flowguard-app) | Frontend Next.js y Route Handlers de la API | Dev 2 | Dev 4 configura sus verificaciones de CI; Dev 5 aporta mocks y coordinación de contratos |
| [`flowguard-mocks`](https://github.com/FlowGuard-Hackathon/flowguard-mocks) | Fixtures de demostración | Dev 5 | Dev 1, 2 y 3 solicitan y validan los escenarios que necesitan |
| [`flowguard-tests`](https://github.com/FlowGuard-Hackathon/flowguard-tests) | Pruebas unitarias, de integración, de contrato y E2E | Cada propietario mantiene las pruebas funcionales de su módulo; Dev 5 coordina la suite compartida | Dev 4 configura CI para ejecutarlas |
| [`flowguard`](https://github.com/FlowGuard-Hackathon/flowguard) | Repositorio final combinado (vía `git subtree`), con todos los anteriores integrados como carpetas | Dev 1 coordina la integración técnica del backend | Dev 4 automatiza verificaciones; los demás contribuyen según su módulo |

### Propiedad por rutas en `flowguard-src`

| Ruta | Propietario |
|---|---|
| `src/server/agent/`, `src/server/tools/`, `src/contracts/evidence-report.schema.ts` | Dev 3 |
| El resto del backend: `src/server/rules/`, `src/server/executions/`, `src/server/persistence/` y los demás módulos del servidor | Dev 1 |
| `src/contracts/{payment,execution,engine}.schema.ts` | Dev 5 los propone y mantiene; Dev 1 los valida |

Las rutas definen propiedad de mantenimiento, no permiso para trabajar sin coordinación: los cambios que cruzan una frontera se acuerdan antes con el otro propietario. `src/contracts/` sigue siendo una zona compartida (ver [architecture.md § 3.1](architecture.md#31-propiedad-de-carpetas)).

`.env.example` con las variables de entorno de referencia vive en el repositorio `flowguard` (el combinado), no en este.
