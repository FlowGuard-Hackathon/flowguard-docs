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
