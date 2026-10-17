# Integración y dependencias entre puestos

Este documento explica cómo colaboran los cinco puestos de FlowGuard, qué contratos comparten, qué pueden construir en paralelo, qué necesitan de los demás, y cómo integran sus cambios. Complementa — no sustituye — [api-contracts.md](api-contracts.md), [database.md](database.md) y [architecture.md](architecture.md), que son la fuente de verdad técnica.

> **Nomenclatura:** `Dev N` / `Developer N` es el nombre corto del `Puesto N` (ver [README.md](README.md#distribución-del-equipo-cinco-puestos)). Los puestos 1, 2, 4 y 5 no tienen persona asignada; el puesto 3 está reservado para quien coordina el proyecto.

## Roles y repositorios

| Puesto | Responsabilidad | Repositorio(s) principal(es) |
|---|---|---|
| Puesto 1 (Developer 1) | Backend, motor de reglas determinista y base de datos | [`flowguard-src`](https://github.com/FlowGuard-Hackathon/flowguard-src) (salvo las rutas del puesto 3), [`flowguard-supabase`](https://github.com/FlowGuard-Hackathon/flowguard-supabase) |
| Puesto 2 (Developer 2) | Frontend y experiencia de usuario | [`flowguard-app`](https://github.com/FlowGuard-Hackathon/flowguard-app) |
| Puesto 3 (Developer 3) | Agente de IA, LLMs, Nebius Token Factory y modelos NVIDIA | [`flowguard-src`](https://github.com/FlowGuard-Hackathon/flowguard-src) (`src/server/agent/`, `src/server/tools/`, `src/contracts/evidence-report.schema.ts`) |
| Puesto 4 (Developer 4) | CI/CD exclusivamente | Workflows de GitHub Actions en los repositorios que correspondan (por determinar según la estructura real) |
| Puesto 5 (Developer 5) | Contratos compartidos, mocks y apoyo general | [`flowguard-mocks`](https://github.com/FlowGuard-Hackathon/flowguard-mocks), documentación de contratos en [`flowguard-docs`](https://github.com/FlowGuard-Hackathon/flowguard-docs), coordinación de la suite compartida en [`flowguard-tests`](https://github.com/FlowGuard-Hackathon/flowguard-tests) |

El repositorio [`flowguard`](https://github.com/FlowGuard-Hackathon/flowguard) integra a todos los anteriores vía `git subtree`, conforme a la estrategia ya documentada. El puesto 1 coordina la integración técnica del backend en ese repositorio; el puesto 4 automatiza las verificaciones que lo hacen reproducible; los demás puestos aportan su módulo y lo mantienen sincronizado.

## Contratos compartidos (no se negocian sin acuerdo del equipo)

Estos contratos son la superficie de integración entre los cinco puestos. Cambiarlos después del día 1 requiere acuerdo explícito de todo el equipo, porque rompen a los demás en silencio:

- **Esquemas Zod**: `payment.schema.ts`, `execution.schema.ts`, `engine.schema.ts`, `evidence-report.schema.ts` — definidos en [api-contracts.md](api-contracts.md).
- **Modelo de estados unificado** (`decision`, `investigation_status`, etc.) — [database.md § 2](database.md#2-modelo-de-estados-unificado).
- **Las cinco reglas de negocio** (`PAY-001` a `PAY-005`) y sus resultados posibles (`PASS`/`FAIL`/`INDETERMINATE`) — [rules.md](rules.md).
- **Contrato del informe del agente** — [api-contracts.md § 10](api-contracts.md#10-contrato-del-informe-del-agente).
- **Catálogo de errores** — [error-catalog.md](error-catalog.md).

Quién propone y quién valida cada contrato:

| Contrato | Lo propone y mantiene | Lo valida o aprueba |
|---|---|---|
| `payment.schema.ts`, `execution.schema.ts`, `engine.schema.ts` | Puesto 5 | Puesto 1 |
| `evidence-report.schema.ts` y contrato del informe del agente | Puesto 3 (propietario) | Puesto 5 coordina la compatibilidad; los demás puestos afectados confirman que no necesitan campos adicionales |
| Documentación de contratos y ejemplos en `flowguard-docs` | Puesto 5 | Responsables de los módulos afectados |

El puesto 5 coordina la congelación del día 1 y registra las propuestas de modificación, pero **no cambia unilateralmente** ningún contrato aprobado ni la semántica de las reglas de negocio (ver [implementation-checklist.md § Bloque 1](implementation-checklist.md#bloque-1--días-13-congelación-de-contratos-y-estructura-del-proyecto)). Mantener la documentación de un contrato no otorga autoridad exclusiva para modificarlo.

## Qué puede avanzar en paralelo

Gracias a los contratos congelados el día 1 y a las fixtures del puesto 5, los cinco puestos pueden trabajar en paralelo desde el Bloque 1 sin bloquearse entre sí:

- **Developer 1** construye el motor de reglas y la migración de base de datos sin depender de nadie (no dependen de servicios externos).
- **Developer 2** construye la UI completa contra los mocks de Developer 5 (`mocks/payment-approved.json`, `mocks/payment-blocked.json`), sin esperar a que la API real exista.
- **Developer 3** construye el adaptador de Nebius con una respuesta mock intercambiable por variable de entorno, sin esperar el acceso real al proveedor.
- **Developer 4** prepara la configuración de CI/CD con lo que ya exista en los repositorios, e incorpora comprobaciones a medida que los propietarios publican scripts (ver el plan propuesto en [developers/developer-4/work-plan.md](developers/developer-4/work-plan.md)).
- **Developer 5** define y mantiene los contratos acordados, publica las fixtures y prepara las pruebas de contrato de la suite compartida desde el día 1.

## Qué necesita cada puesto de los demás

| Puesto | Necesita de... | Qué necesita | Cuándo |
|---|---|---|---|
| Developer 1 | Developer 5 | Propuesta de los contratos `payment`, `execution` y `engine` para validarla, y estructura de carpetas acordada | Día 1 |
| Developer 2 | Developer 5 | Fixtures de mocks (éxito, bloqueo, fallo) | Día 1–3 |
| Developer 2 | Developer 1 | `POST /api/v1/executions` real disponible | Día 4–6 |
| Developer 2 | Developer 3 | Contrato del informe del agente y `GET /executions/:id/evidence` con datos reales | Día 7–9 |
| Developer 3 | Developer 5 | Coordinación de compatibilidad del contrato del informe del agente y fixture `investigation-failed.json` | Día 1–3 |
| Developer 3 | Developer 1 | Migración aplicada y `POST /executions` disponible, para que las herramientas de lectura tengan datos reales contra los que operar | Día 4–6 |
| Developer 5 | Developer 1, Developer 3 | Confirmación de que no faltan campos en los contratos antes de congelarlos | Día 1 |
| Developer 4 | Developer 1, 2, 3, 5 | Scripts de instalación, lint, tipos, pruebas y build de cada módulo, y fixtures, para poder automatizar su ejecución | A medida que existan (por bloque) |
| Developer 2 | Developer 1 | Credenciales de la demo distribuidas por un canal apropiado, para verificar el acceso desde la interfaz | Día 13 |
| Todo el equipo | Developer 1 | Decisión de plataforma de hosting (proceso persistente vs. serverless), porque determina el diseño del worker asíncrono | A más tardar día 3 |
| Todo el equipo | Developer 3 | Checklist de compatibilidad con Nebius confirmada, o decisión explícita de seguir en modo mock | A más tardar día 7 |

Ver también la tabla de dependencias entre hitos principales en [implementation-checklist.md § Dependencias entre hitos principales](implementation-checklist.md#dependencias-entre-hitos-principales).

## Coordinación entre Developer 1 y Developer 3 en `flowguard-src`

Las herramientas de lectura del agente (`src/server/tools/`) pueden depender de servicios, consultas y persistencia mantenidos por el puesto 1 (`src/server/persistence/`).

- El puesto 3 define qué información necesita el agente y qué resultados espera recibir.
- El puesto 1 mantiene la lógica de acceso a datos y los servicios de backend correspondientes.
- Ambos acuerdan las interfaces antes de implementar una integración que afecte a los dos módulos.
- El puesto 3 no crea mecanismos alternativos de acceso a datos para eludir la arquitectura, y el puesto 1 no cambia unilateralmente los contratos de las herramientas del agente.

## Cómo se integran los cambios

1. Cada puesto trabaja en su propio repositorio (ver tabla de roles), respetando los contratos congelados.
2. Los cambios que afecten a un contrato compartido se proponen primero como actualización a la documentación en `flowguard-docs` (este repositorio), vía Pull Request, antes de implementarse en el código.
3. Las fixtures de `flowguard-mocks` son la referencia de integración entre Developer 2 y Developer 3 mientras el backend o el proveedor de IA no están disponibles.
4. La integración final combina todos los repositorios en [`flowguard`](https://github.com/FlowGuard-Hackathon/flowguard) vía `git subtree`.
5. Ningún cambio se considera integrado solo porque compile en su propio repositorio: debe pasar las pruebas de contrato de `flowguard-tests` (coordinadas por Developer 5 y ejecutadas por la CI de Developer 4 cuando existan) antes de integrarse.
6. Nadie trabaja directamente sobre `main`: cada cambio pasa por una rama y un Pull Request revisado por los responsables de los módulos afectados.

## Procedimiento de apoyo entre puestos

### Solicitud de ayuda

El integrante que se atasca proporciona, en la medida en que aplique:

- Módulo y archivo afectados.
- Comportamiento esperado y comportamiento observado.
- Error o resultado obtenido.
- Pasos para reproducirlo.
- Acciones que ya intentó.
- Ayuda concreta que necesita, urgencia y tareas bloqueadas.

### Intervención del puesto 5

1. Revisa el problema y confirma que puede reproducirse, o indica qué información falta.
2. Acuerda con el responsable del módulo el alcance de la intervención.
3. Investiga sin modificar innecesariamente el módulo.
4. Propone o implementa una solución coordinada con el responsable.
5. Ejecuta las verificaciones pertinentes.
6. Documenta el resultado en su bitácora y, si queda un riesgo, lo registra en los bloqueos.
7. Queda disponible para la siguiente necesidad.

El responsable original mantiene la propiedad de su módulo y valida la solución. Si la solución afecta a varios módulos o contratos, intervienen los responsables afectados antes de integrar el cambio. El puesto 5 no debe convertirse en el único integrante capaz de resolver problemas de integración.

### Prioridad de las solicitudes

1. Bloqueos que impiden ejecutar o probar el producto.
2. Bloqueos que detienen a varios integrantes.
3. Defectos que impiden demostrar el flujo principal.
4. Tareas que amenazan una dependencia crítica o una fecha límite.
5. Mejoras menores o limpieza.

Si dos bloqueos tienen la misma prioridad, se decide con los responsables afectados.

## CI/CD y verificación cruzada

- **CI/CD (puesto 4):** configura y mantiene GitHub Actions, y automatiza la ejecución de lo que mantienen los propietarios. Comunica los fallos del pipeline y colabora en su diagnóstico, pero el propietario del módulo corrige los defectos funcionales. El plan es **propuesto, pendiente de aprobación**: [developers/developer-4/work-plan.md](developers/developer-4/work-plan.md).
- **Quién mantiene cada tipo de prueba:** ver [testing.md § Responsables de las pruebas](testing.md#responsables-de-las-pruebas).
- **Hitos de los días 6, 9, 12 y 15:** siguen exigiendo una verificación mediante ensayo con evidencia, ahora como verificación cruzada entre puestos. Ver [definition-of-done.md](definition-of-done.md#criterios-por-hito-de-bloque).

## Separación de responsabilidades (no negociable)

La regla de diseño que ningún puesto puede romper al integrar su trabajo: **el motor de reglas decide, el backend persiste, la IA investiga y el frontend presenta.** La IA (Developer 3) solo tiene herramientas de lectura; no puede autorizar pagos, modificar reglas ni ejecutar operaciones de escritura (ver [security.md § 1.1](security.md#11-responsabilidad)).

## Referencias

- Contratos técnicos completos: [api-contracts.md](api-contracts.md)
- Modelo de datos y de estados: [database.md](database.md)
- Reglas de negocio: [rules.md](rules.md)
- Plan de 15 días: [development-plan.md](development-plan.md)
- Criterios de cierre de una tarea: [definition-of-done.md](definition-of-done.md)
