# FlowGuard — Documentación técnica

**Versión:** 1.0
**Fecha:** 15 de octubre de 2026
**Tipo de documento:** Especificación técnica previa a la implementación
**Equipo:** 4 developers
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

| Documento | Contenido |
|---|---|
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
| [integration-and-dependencies.md](integration-and-dependencies.md) | Cómo colaboran los cuatro developers, qué contratos comparten y qué necesitan entre sí |
| [definition-of-done.md](definition-of-done.md) | Checklist operativa para considerar terminada una tarea o un hito de bloque |

## Espacio de trabajo de cada developer

Cada developer tiene su propia carpeta en `developers/` con su plan completo de 15 días, bitácora diaria, notas técnicas y bloqueos:

| Developer | Responsabilidad | Carpeta |
|---|---|---|
| Developer 1 | Backend, motor de reglas, base de datos y DevOps | [developers/developer-1/](developers/developer-1/README.md) |
| Developer 2 | Frontend y UX | [developers/developer-2/](developers/developer-2/README.md) |
| Developer 3 | Agente de IA y Nebius | [developers/developer-3/](developers/developer-3/README.md) |
| Developer 4 | Contratos, mocks, pruebas e integración | [developers/developer-4/](developers/developer-4/README.md) |

Estos nombres (`Developer 1` a `Developer 4`) se usan hasta que se conozcan los usuarios reales de GitHub de cada integrante.

## Reglas de actualización y colaboración

1. Cada developer actualiza su bitácora (`daily-log.md`) al terminar su jornada.
2. Las notas (`notes.md`) deben incluir contexto suficiente para que otro integrante las entienda.
3. Los bloqueos importantes deben registrarse en `blockers.md` y comunicarse al equipo.
4. Las decisiones que afecten a la arquitectura o a varios módulos también deben reflejarse en la documentación general (este README, `development-plan.md`, `integration-and-dependencies.md`, o el documento técnico correspondiente).
5. No se considera completada una tarea solo porque esté anotada como terminada: debe existir evidencia verificable (ver [definition-of-done.md](definition-of-done.md)).
6. Los archivos individuales pueden modificarse mediante Pull Requests pequeños.
7. Los documentos generales son compartidos y no deben contradecir los contratos técnicos existentes.

Los archivos Markdown de este repositorio son compartidos: los cuatro integrantes pueden proponer cambios si tienen permisos de escritura en GitHub. Las carpetas de `developers/` organizan el trabajo, no establecen permisos privados.

## Otros repositorios de FlowGuard-Hackathon

Este repositorio (`flowguard-docs`) es solo documental. El proyecto completo vive repartido en varios repositorios independientes, cada uno con su propio responsable:

| Repositorio | Contenido | Responsable |
|---|---|---|
| [`flowguard-supabase`](https://github.com/FlowGuard-Hackathon/flowguard-supabase) | Migración SQL de referencia (`migrations/001_initial_schema.sql`) correspondiente a [database.md](database.md) | Dev 1 |
| [`flowguard-src`](https://github.com/FlowGuard-Hackathon/flowguard-src) | Contratos compartidos, motor de reglas, persistencia, agente de IA | Dev 1 / Dev 3 |
| [`flowguard-app`](https://github.com/FlowGuard-Hackathon/flowguard-app) | Frontend Next.js y Route Handlers de la API | Dev 2 |
| [`flowguard-mocks`](https://github.com/FlowGuard-Hackathon/flowguard-mocks) | Fixtures de demostración | Dev 4 |
| [`flowguard-tests`](https://github.com/FlowGuard-Hackathon/flowguard-tests) | Pruebas unitarias, de integración, de contrato y E2E | Dev 4 |
| [`flowguard`](https://github.com/FlowGuard-Hackathon/flowguard) | Repositorio final combinado (vía `git subtree`), con todos los anteriores integrados como carpetas | Todo el equipo |

`.env.example` con las variables de entorno de referencia vive en el repositorio `flowguard` (el combinado), no en este.
