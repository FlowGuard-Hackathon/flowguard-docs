# Integración y dependencias entre developers

Este documento explica cómo colaboran los cuatro developers de FlowGuard, qué contratos comparten, qué pueden construir en paralelo, qué necesitan de los demás, y cómo integran sus cambios. Complementa — no sustituye — [api-contracts.md](api-contracts.md), [database.md](database.md) y [architecture.md](architecture.md), que son la fuente de verdad técnica.

## Roles y repositorios

| Developer | Responsabilidad | Repositorio(s) principal(es) |
|---|---|---|
| Developer 1 | Backend, motor de reglas, base de datos y DevOps | [`flowguard-src`](https://github.com/FlowGuard-Hackathon/flowguard-src), [`flowguard-supabase`](https://github.com/FlowGuard-Hackathon/flowguard-supabase) |
| Developer 2 | Frontend y UX | [`flowguard-app`](https://github.com/FlowGuard-Hackathon/flowguard-app) |
| Developer 3 | Agente de IA y Nebius | [`flowguard-src`](https://github.com/FlowGuard-Hackathon/flowguard-src) (módulo del agente) |
| Developer 4 | Contratos, mocks, pruebas e integración | [`flowguard-mocks`](https://github.com/FlowGuard-Hackathon/flowguard-mocks), [`flowguard-tests`](https://github.com/FlowGuard-Hackathon/flowguard-tests) |

El repositorio [`flowguard`](https://github.com/FlowGuard-Hackathon/flowguard) integra a todos los anteriores vía `git subtree` y es responsabilidad de todo el equipo mantenerlo sincronizado.

## Contratos compartidos (no se negocian sin acuerdo del equipo)

Estos contratos son la superficie de integración entre los cuatro developers. Cambiarlos después del día 1 requiere acuerdo explícito de todo el equipo, porque rompen a los demás en silencio:

- **Esquemas Zod**: `payment.schema.ts`, `execution.schema.ts`, `engine.schema.ts`, `evidence-report.schema.ts` — definidos en [api-contracts.md](api-contracts.md).
- **Modelo de estados unificado** (`decision`, `investigation_status`, etc.) — [database.md § 2](database.md#2-modelo-de-estados-unificado).
- **Las cinco reglas de negocio** (`PAY-001` a `PAY-005`) y sus resultados posibles (`PASS`/`FAIL`/`INDETERMINATE`) — [rules.md](rules.md).
- **Contrato del informe del agente** — [api-contracts.md § 10](api-contracts.md#10-contrato-del-informe-del-agente).
- **Catálogo de errores** — [error-catalog.md](error-catalog.md).

Propietario de la congelación: Developer 4, con aprobación de Developer 1 y Developer 3 (ver [implementation-checklist.md § Bloque 1](implementation-checklist.md#bloque-1--días-13-congelación-de-contratos-y-estructura-del-proyecto)).

## Qué puede avanzar en paralelo

Gracias a los contratos congelados el día 1 y a las fixtures de Developer 4, los cuatro developers pueden trabajar en paralelo desde el Bloque 1 sin bloquearse entre sí:

- **Developer 1** construye el motor de reglas y la migración de base de datos sin depender de nadie (no dependen de servicios externos).
- **Developer 2** construye la UI completa contra los mocks de Developer 4 (`mocks/payment-approved.json`, `mocks/payment-blocked.json`), sin esperar a que la API real exista.
- **Developer 3** construye el adaptador de Nebius con una respuesta mock intercambiable por variable de entorno, sin esperar el acceso real al proveedor.
- **Developer 4** define los contratos, publica las fixtures y prepara la suite de pruebas de contrato desde el día 1.

## Qué necesita cada developer de los demás

| Developer | Necesita de... | Qué necesita | Cuándo |
|---|---|---|---|
| Developer 1 | Developer 4 | Contratos congelados y estructura de carpetas acordada | Día 1 |
| Developer 2 | Developer 4 | Fixtures de mocks (éxito, bloqueo, fallo) | Día 1–3 |
| Developer 2 | Developer 1 | `POST /api/v1/executions` real disponible | Día 4–6 |
| Developer 2 | Developer 3 | Contrato del informe del agente y `GET /executions/:id/evidence` con datos reales | Día 7–9 |
| Developer 3 | Developer 4 | Contrato del informe del agente congelado | Día 1 |
| Developer 3 | Developer 1 | Migración aplicada y `POST /executions` disponible, para que las herramientas de lectura tengan datos reales contra los que operar | Día 4–6 |
| Developer 4 | Developer 1, Developer 3 | Confirmación de que no faltan campos en los contratos antes de congelarlos | Día 1 |
| Todo el equipo | Developer 1 | Decisión de plataforma de hosting (proceso persistente vs. serverless), porque determina el diseño del worker asíncrono | A más tardar día 3 |
| Todo el equipo | Developer 3 | Checklist de compatibilidad con Nebius confirmada, o decisión explícita de seguir en modo mock | A más tardar día 7 |

Ver también la tabla de dependencias entre hitos principales en [implementation-checklist.md § Dependencias entre hitos principales](implementation-checklist.md#dependencias-entre-hitos-principales).

## Cómo se integran los cambios

1. Cada developer trabaja en su propio repositorio (ver tabla de roles), respetando los contratos congelados.
2. Los cambios que afecten a un contrato compartido se proponen primero como actualización a la documentación en `flowguard-docs` (este repositorio), vía Pull Request, antes de implementarse en el código.
3. Las fixtures de `flowguard-mocks` son la referencia de integración entre Developer 2 y Developer 3 mientras el backend o el proveedor de IA no están disponibles.
4. La integración final combina todos los repositorios en [`flowguard`](https://github.com/FlowGuard-Hackathon/flowguard) vía `git subtree`.
5. Ningún cambio se considera integrado solo porque compile en su propio repositorio: debe pasar las pruebas de contrato de `flowguard-tests` (Developer 4) antes de integrarse.

## Separación de responsabilidades (no negociable)

La regla de diseño que ningún developer puede romper al integrar su trabajo: **el motor de reglas decide, el backend persiste, la IA investiga y el frontend presenta.** La IA (Developer 3) solo tiene herramientas de lectura; no puede autorizar pagos, modificar reglas ni ejecutar operaciones de escritura (ver [security.md § 1.1](security.md#11-responsabilidad)).

## Referencias

- Contratos técnicos completos: [api-contracts.md](api-contracts.md)
- Modelo de datos y de estados: [database.md](database.md)
- Reglas de negocio: [rules.md](rules.md)
- Plan de 15 días: [development-plan.md](development-plan.md)
- Criterios de cierre de una tarea: [definition-of-done.md](definition-of-done.md)
