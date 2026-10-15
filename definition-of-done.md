# Definition of Done

Este documento amplía, a nivel de todo el equipo y de cada tarea individual, los "Criterios de salida" y la "Definition of Done" ya definidos en [testing.md § 4](testing.md#4-criterios-de-salida). No los sustituye: si hay cualquier discrepancia, [testing.md](testing.md) es la fuente de verdad técnica sobre pruebas, y este documento es la checklist operativa que cada developer aplica antes de marcar una tarea como terminada.

## Principio general

Una tarea no se considera terminada porque esté anotada como terminada en un `work-plan.md` o en una bitácora. Debe existir evidencia verificable: una prueba que pasa, una respuesta real de la API, una captura de la UI, o una revisión de otro developer.

## Criterios obligatorios por tarea

Antes de marcar una tarea como `Completado` en cualquier `work-plan.md` o `daily-log.md`, debe cumplirse todo lo aplicable:

- [ ] **Código revisado**: el cambio pasó por Pull Request, no se integró directamente a la rama principal del repositorio correspondiente.
- [ ] **Contrato respetado**: si la tarea toca un contrato compartido (esquema Zod, modelo de estados, catálogo de errores), el cambio no contradice [api-contracts.md](api-contracts.md), [database.md](database.md) ni [error-catalog.md](error-catalog.md). Si el contrato necesita cambiar, el cambio se documenta primero en `flowguard-docs`.
- [ ] **Pruebas ejecutadas**: existe al menos la prueba mínima indicada en la fila correspondiente de [implementation-checklist.md](implementation-checklist.md) (unitaria, de integración o E2E, según [testing.md](testing.md)), y se ejecutó realmente — no se asume que "debería pasar".
- [ ] **Comportamiento verificable**: el resultado puede reproducirse por otro developer (por ejemplo, un endpoint real responde lo esperado, o la UI muestra el estado documentado), no solo describirse.
- [ ] **Errores tratados**: los casos de error relevantes (ver [error-catalog.md](error-catalog.md)) no rompen el flujo ni cambian la `decision` ya persistida.
- [ ] **Sin secretos en el repositorio**: no se incluyen tokens, contraseñas ni credenciales de Nebius o Supabase en ningún archivo versionado, incluidas notas (`notes.md`) y bitácoras (`daily-log.md`).
- [ ] **Documentación actualizada**: si la tarea afecta a la arquitectura, a un contrato o a más de un módulo, se refleja en la documentación general (`development-plan.md`, `integration-and-dependencies.md`, o el documento técnico correspondiente en `flowguard-docs`), no solo en la nota personal del developer.
- [ ] **Integración compatible**: el cambio no rompe el trabajo de otro developer que depende del mismo contrato (ver [integration-and-dependencies.md](integration-and-dependencies.md)).

## Criterios por hito de bloque

Cada uno de los cinco hitos obligatorios del plan de 15 días (ver [development-plan.md](development-plan.md)) tiene, además, su propio criterio de aceptación específico, ya definido en [implementation-checklist.md](implementation-checklist.md):

| Bloque | Hito obligatorio |
|---|---|
| 1 (días 1–3) | Contratos congelados y confirmados por escrito por los cuatro developers |
| 2 (días 4–6) | Crear una ejecución, evaluar las cinco reglas, persistir la decisión y mostrarla en el panel, con la IA en modo mock |
| 3 (días 7–9) | Una ejecución bloqueada produce un informe real o un `investigation_status = FAILED` explícito y correctamente comunicado |
| 4 (días 10–12) | El MVP funciona de extremo a extremo y conserva la decisión cuando falla Nebius |
| 5 (días 13–15) | Versión estable, pruebas críticas aprobadas, README reproducible y demo ensayada |

Un hito de bloque no se considera cumplido solo porque cada tarea individual esté marcada como completada: se verifica mediante un ensayo interno (ver la columna "Prueba que lo demuestra" de cada bloque en [implementation-checklist.md](implementation-checklist.md)), normalmente verificado por Developer 4.

## Lo que NO cuenta como "terminado"

- Una tarea marcada `Completado` sin una prueba o revisión que lo respalde.
- Un endpoint que "debería funcionar" pero no se probó contra una base de datos real.
- Una nota en `notes.md` describiendo una decisión técnica que no se reflejó en la documentación general cuando afecta a otros developers.
- Un bloqueo resuelto sin registrar su resolución en `blockers.md`.

## Referencias

- Criterios de salida técnicos y Definition of Done original: [testing.md § 4](testing.md#4-criterios-de-salida)
- Plan de pruebas completo: [testing.md](testing.md)
- Catálogo de errores: [error-catalog.md](error-catalog.md)
- Cómo se integran los cambios entre developers: [integration-and-dependencies.md](integration-and-dependencies.md)
