# Plan de trabajo — Developer 3 (Agente de IA, LLMs, Nebius Token Factory y modelos NVIDIA)

Responsabilidades: contrato del agente, proveedor simulado, integración con Nebius Token Factory y con el modelo NVIDIA elegido, investigación y explicación de evidencias, validación de respuestas, gestión de fallos y herramientas de solo lectura; propiedad de `src/server/agent/`, `src/server/tools/` y `src/contracts/evidence-report.schema.ts`.

Este plan es la vista diaria del rol de Developer 3 dentro de [development-plan.md](../../development-plan.md) y [implementation-checklist.md](../../implementation-checklist.md), que siguen siendo la fuente de verdad sobre pruebas y criterios técnicos exactos. Todos los estados inician en `Pendiente`: no se marcan como completados hasta que exista evidencia verificable, según [definition-of-done.md](../../definition-of-done.md).

| Día | Objetivo | Tareas | Entregable | Dependencias | Criterio de aceptación | Estado |
|---|---|---|---|---|---|---|
| 1 | Contrato del informe del agente | Participar en la definición del contrato del informe junto con Dev 4 | Contrato acordado en [api-contracts.md § 10](../../api-contracts.md#10-contrato-del-informe-del-agente) y reflejado en `src/contracts/evidence-report.schema.ts` | Contratos congelados (día 1) | El contrato cubre los campos que necesita el adaptador | Completado |
| 2 | Adaptador de Nebius con mock | Construir el adaptador (`src/server/agent/`) con una respuesta mock intercambiable por variable de entorno | Adaptador funcional en modo mock (`MockAgentProvider`, selección por `AGENT_PROVIDER`) | Contrato del informe (día 1) | El adaptador produce un informe válido contra el esquema, usando el mock | Completado |
| 3 | Validación del adaptador | Prueba unitaria que valida el mock contra el esquema Zod; coordinar con Dev 5 la fixture `investigation-failed.json` | Prueba unitaria pasando (`mockProvider.test.ts`, 2/2) | Adaptador con mock (día 2) | La prueba unitaria confirma que el mock cumple el esquema | En progreso* |
| 4 | Herramientas de lectura (esqueleto) | Implementar `get_execution`, `get_rule_results`, `get_evidence`, `get_payment_context`, `get_rule_details` contra datos mock | Firmas creadas en `src/server/tools/` (lanzan `NotImplementedError` hasta tener datos mock o reales) | Contrato del informe | Cada herramienta devuelve datos consistentes con el mock | Pendiente |
| 5 | Herramientas contra datos reales | Conectar las herramientas de lectura a la base real una vez disponible `POST /executions` | Herramientas devolviendo datos reales | Migración aplicada, `POST /executions` (Dev 1) | Cada herramienta devuelve datos reales de una ejecución de prueba | Pendiente |
| 6 | Pruebas unitarias por herramienta y hito del bloque 2 | Completar prueba unitaria por cada herramienta; participar en el ensayo del hito obligatorio | Suite de pruebas por herramienta; hito del día 6 demostrado | Herramientas contra datos reales (día 5) | Demo interna reproduce el hito sin errores, con IA en modo mock | Pendiente |
| 7 | Checklist de compatibilidad con Nebius | Completar y documentar la checklist de [security.md § 1.8](../../security.md#18-lista-de-verificación-de-compatibilidad-con-nebius) | Checklist completada con respuestas confirmadas, no supuestos | Acceso a documentación oficial y credenciales del hackathon | Cada punto de la lista tiene una respuesta confirmada; revisión cruzada de Dev 5 (propuesto, pendiente de confirmar; no revisas tu propio trabajo) | Pendiente |
| 8 | Integración real de Nebius | Implementar tool calling real y validación de informes; con Dev 5, diseñar los escenarios E2E de fallos de proveedor (timeout, JSON inválido, referencia inválida) | Adaptador real funcionando | Checklist de Nebius confirmada (día 7) | Un informe real pasa la validación de [security.md § 1.7](../../security.md#17-validación-de-informes) | Pendiente |
| 9 | Validación con proveedor real y hito del bloque 3 | Ejecutar prueba de integración con la API real de Nebius (cuenta de pruebas) y documentar la evidencia verificable de las llamadas reales y del modelo NVIDIA usado; participar en el ensayo del hito obligatorio | Integración validada; hito del día 9 demostrado | Integración real (día 8) | Una ejecución bloqueada produce un informe real o `investigation_status = FAILED` explícito y correctamente comunicado | Pendiente |
| 10 | Calibración de límites | Ajustar `AGENT_MAX_TOOL_CALLS` y `AGENT_TIMEOUT_MS` a valores reales observados | Parámetros calibrados y documentados | Integración real de Nebius (o decisión documentada de seguir en mock) | Los valores reflejan latencias reales medidas, no solo el valor por defecto | Pendiente |
| 11 | Pruebas de fallos de proveedor | Diseñar y ejecutar pruebas de timeout, JSON inválido y referencia inválida (tú eres el propietario de estas pruebas; Dev 5 coordina escenarios y fixtures, y Dev 4 automatiza su ejecución en CI) | Casos de fallo cubiertos | Integración real o mock controlado de fallos | Los tres casos producen `investigation_status = FAILED`/`PARTIAL` sin alterar `decision` | Pendiente |
| 12 | Hito obligatorio del bloque 4 | Participar en el ensayo con fallo forzado de Nebius | Comportamiento ante fallo confirmado | Todo lo anterior del bloque | La decisión persistida no cambia cuando se fuerza un fallo de Nebius | Pendiente |
| 13 | Congelación de funcionalidades | Solo correcciones de bugs o documentación a partir de este día | PRs únicamente de corrección | Hito del día 12 | Ningún PR posterior agrega funcionalidad nueva | Pendiente |
| 14 | Verificación final del modelo | Verificar el comportamiento ante errores de Nebius para la demo; ensayar el escenario de fallo | Plan de contingencia ejecutable | Congelación de funcionalidades | El plan de [demo-script.md § 3](../../demo-script.md#3-plan-de-contingencia-de-la-demo) se ejecuta sin improvisar | Pendiente |
| 15 | Entrega final | Apoyar el ensayo cronometrado de la demo | Agente estable para la demo | Todo lo anterior | Checklist de [implementation-checklist.md](../../implementation-checklist.md) completa para las filas de Developer 3 | Pendiente |

\* Día 3: la prueba unitaria del mock ya pasa; falta que Dev 5 publique la fixture `investigation-failed.json` en `flowguard-mocks` para cerrar el día por completo.

> **Nota de reorganización (cinco puestos):** las menciones a `Dev 4` en los días ya completados (por ejemplo, el día 1) se refieren a la distribución anterior, en la que ese puesto coordinaba los contratos; no se reescribe el historial. A partir de la reorganización, los contratos compartidos y los mocks son del puesto 5 y el puesto 4 es CI/CD. El contrato del informe del agente (`evidence-report.schema.ts`) es y sigue siendo propiedad de este puesto.

## Progreso registrado

- 2026-10-16 — Días 1 a 3 avanzados: esqueleto del agente inicializado en [flowguard-src PR #1](https://github.com/FlowGuard-Hackathon/flowguard-src/pull/1) (fusionado a `main`). Detalle día a día en [daily-log.md](daily-log.md); decisiones técnicas en [notes.md](notes.md).

## Referencias

- [implementation-checklist.md](../../implementation-checklist.md) — detalle técnico completo por fila
- [security.md](../../security.md) — ciclo del agente, herramientas y validación de informes
- [integration-and-dependencies.md](../../integration-and-dependencies.md) — qué necesita Developer 3 de los demás
