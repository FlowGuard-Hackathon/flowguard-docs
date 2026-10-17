# Plan de trabajo — Developer 4 (CI/CD)

Responsabilidad exclusiva: integración continua y despliegue continuo. Configura y mantiene GitHub Actions, y automatiza la ejecución de las verificaciones que mantienen los propietarios de cada módulo.

> **Estado de este plan: Propuesto — pendiente de aprobación.** Se ha alineado con los cinco bloques del calendario vigente ([development-plan.md](../../development-plan.md)). No introduce fechas nuevas ni reemplaza el calendario aprobado. Todos los estados inician en `Propuesto` o `Pendiente`: nada se marca como completado hasta que exista evidencia verificable, según [definition-of-done.md](../../definition-of-done.md).

## Qué NO hace este puesto

- No es responsable funcional de QA, de los contratos, de los fixtures, de los módulos de aplicación ni de `flowguard-tests`. Los propietarios mantienen sus pruebas; este puesto automatiza su ejecución.
- Puede comunicar fallos del pipeline y colaborar en su diagnóstico, pero el propietario técnico corrige los defectos funcionales.
- No introduce plataformas de CI/CD ni infraestructura nueva: [architecture.md](../../architecture.md) ya fija GitHub (ramas, revisiones y CI) como solución.
- No consume credenciales reales ni despliega cambios productivos como parte de la reorganización documental.

## Estado real inspeccionado

Verificado en `main` de los siete repositorios el 2026-10-17:

- **Ningún repositorio tiene workflows de GitHub Actions** (`.github/` no existe en ninguno).
- `flowguard-src` es el único con scripts ejecutables: `npm test` (`vitest run`) y `npm run typecheck` (`tsc --noEmit`). No hay scripts de `lint` ni de `build`.
- `flowguard-app`, `flowguard-tests`, `flowguard-mocks`, `flowguard-supabase` y `flowguard` no tienen aún scripts que automatizar (README y, en el caso de `flowguard-supabase`, una migración SQL).
- Observación de instalación local: `vitest@5.0.5` declara requerir Node `^22.12.0 || ^24.0.0 || >=26.0.0` (advertencia `EBADENGINE` con Node 20). La versión de Node del workflow debe confirmarse con los propietarios de `flowguard-src`; `package.json` no declara `engines`.

Por eso este plan **no declara trabajo realizado**: depende de que los propietarios publiquen sus scripts.

## Plan propuesto por bloque

| Bloque | Días del bloque (ya existentes) | Actividad propuesta | Evidencia esperada | Depende de | Estado |
|---|---|---|---|---|---|
| 1 | 1–3 | Inspeccionar el estado real de repositorios, scripts y workflows; preparar una configuración base de GitHub Actions (instalación, `typecheck` y `test`) para `flowguard-src`, con la versión de Node confirmada con sus propietarios | Workflow que ejecuta los scripts existentes de `flowguard-src` y su resultado visible en un PR | Aprobación de este plan; confirmación de la versión de Node por Dev 1 y Dev 3 | Propuesto |
| 2 | 4–6 | Incorporar lint, comprobación de tipos, pruebas y build de `flowguard-app` y del backend a medida que sus propietarios publiquen los scripts; ejecutar las pruebas de contrato de la suite compartida cuando Dev 5 las publique | Workflows por repositorio con resultados reproducibles | Scripts publicados por Dev 1, Dev 2 y Dev 5 | Propuesto |
| 3 | 7–9 | Automatizar la ejecución de las pruebas de integración y E2E mantenidas por los propietarios; configurar los secretos necesarios como GitHub Secrets (nunca en el repositorio ni en logs); las pruebas contra el proveedor real de Nebius solo se ejecutan de forma controlada y aprobada por Dev 3 | Resultados de integración y E2E como evidencia complementaria del hito del día 9 | Pruebas de los propietarios; decisión de Dev 3 sobre el uso de credenciales reales en CI | Propuesto |
| 4 | 10–12 | Estabilizar los pipelines; aportar resultados reproducibles de la regresión conjunta como evidencia complementaria del hito del día 12; documentar cómo se ejecutan y diagnostican los pipelines | Documentación operativa de los workflows; reporte de resultados de CI | Regresión conjunta de los propietarios | Propuesto |
| 5 | 13–15 | Aportar la evidencia de pipeline para la congelación y el hito del día 15; ayudar a que la integración del repositorio `flowguard` (`git subtree`) sea reproducible; automatizar el despliegue solo conforme a la infraestructura ya acordada | Pipeline verificable sobre la rama de entrega; instrucciones de reproducción | Decisión de plataforma de hosting (Dev 1, día 3); congelación de funcionalidades | Propuesto |

Si alguna actividad no encaja en un bloque, se registra en [blockers.md](blockers.md) o se plantea al equipo; **no se altera el calendario por iniciativa propia**.

## Coordinación

- **Dev 1:** comprobaciones y despliegue del backend; verificaciones de la migración solo en el entorno de prueba autorizado.
- **Dev 2:** comprobaciones del frontend.
- **Dev 3:** verificaciones del agente y su proveedor, sin exponer credenciales.
- **Dev 5:** contratos y mocks necesarios para validar las integraciones; apoyo cuando un pipeline falle por causas transversales.

## Tareas del plan anterior de Developer 4 y su nuevo responsable

El plan anterior de este puesto (contratos, mocks, pruebas, QA e integración) fue reasignado fila por fila. Ninguna tarea desaparece: cambia quién la ejecuta o verifica. Los criterios de aceptación originales se conservan en [implementation-checklist.md](../../implementation-checklist.md).

| Día | Tarea anterior | Responsable nuevo | Nota |
|---|---|---|---|
| 1 | Proponer y cerrar los esquemas Zod y el modelo de estados; crear la estructura de carpetas | Dev 5 propone `payment`, `execution` y `engine` y coordina la congelación; Dev 1 los valida; `evidence-report.schema.ts` es de Dev 3. Estructura de carpetas: cada propietario en su repositorio, y Dev 1 coordina el repositorio integrado `flowguard` | Los cinco puestos confirman por escrito |
| 2 | Publicar `payment-approved.json` y `payment-blocked.json` | Dev 5 | |
| 3 | Publicar `investigation-failed.json`; prueba de contrato automatizada sobre los tres fixtures | Dev 5 (fixture y prueba de contrato) | Dev 4 ejecuta la prueba en CI cuando el script exista |
| 4 | Pruebas de integración de `POST /executions` | Dev 1 | Pruebas del backend |
| 5 | Prueba de idempotencia | Dev 1 | |
| 6 | Verificar el ensayo del hito del bloque 2 | Verificación cruzada entre los módulos involucrados, con evidencia | Dev 4 aporta resultados de CI; Dev 5 coordina lo transversal |
| 7 | Revisión cruzada de la checklist de Nebius | Dev 5 (propuesto, pendiente de confirmar) | Dev 3 no revisa su propio trabajo |
| 8 | Pruebas E2E de fallos de proveedor | Dev 3 (agente y proveedor), con Dev 2 para la UI; Dev 5 coordina escenarios | Dev 4 automatiza la ejecución |
| 9 | Verificar el ensayo del hito del bloque 3 | Verificación cruzada de la integración y de los escenarios de fallo del proveedor | |
| 10 | Regresión completa | Propietarios de cada módulo (regresión conjunta); coordinación de Dev 5 | Dev 4 automatiza la ejecución |
| 11 | Seguimiento de defectos | Dev 5 registra los defectos transversales; el propietario del módulo afectado los corrige | |
| 12 | Verificar el ensayo del hito del bloque 4 | Regresión conjunta, registro de defectos y evaluación de pendientes | Dev 4 aporta resultados de CI |
| 13 | Revisar que los PRs posteriores solo toquen bugs o documentación | Propietarios de los módulos afectados (revisión cruzada de PRs) | Dev 4 aporta el resultado de CI |
| 14 | QA final de demo | Verificación conjunta: Dev 2 (interfaz y experiencia de demo), Dev 3 (modelo y manejo de errores de Nebius), Dev 5 (registro de defectos) | |
| 15 | Ensayar el guion de demo con datos reiniciables; revisión final de la documentación | Dev 2 (experiencia de demo y guion), Dev 1 (datos reiniciables), Dev 5 (fixtures y consistencia documental); verificación cruzada del ensayo final | |

## Referencias

- [implementation-checklist.md](../../implementation-checklist.md) — detalle técnico completo por fila
- [testing.md](../../testing.md) — plan de pruebas y responsables por tipo de prueba
- [integration-and-dependencies.md](../../integration-and-dependencies.md) — qué necesita Developer 4 de los demás
