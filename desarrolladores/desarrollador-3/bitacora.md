# Bitácora diaria — Developer 3

Copia la plantilla de abajo para cada día de trabajo. No se rellenan avances inventados: cada sección se completa el día que realmente ocurre el trabajo.

## Cómo usar esta bitácora

- Actualízala al terminar tu jornada, según la regla 1 de [README.md](../../README.md#reglas-de-actualización-y-colaboración).
- Si una sección no aplica ese día, déjala vacía o escribe "N/A"; no inventes contenido para rellenarla.
- Los bloqueos reales van en [blockers.md](bloqueos.md), no aquí.

---

## Día 1 — 2026-10-16

- Día del plan: 1
- Fecha: 2026-10-16
- Estado general: Completado

### Trabajo realizado
- Revisión del contrato del informe del agente ya definido en [api-contracts.md §10](../../contratos-api.md#10-contrato-del-informe-del-agente); no se necesitaron campos adicionales para el adaptador de Nebius.
- Inicialización del repositorio `flowguard-src` (package.json, tsconfig.json, vitest, .gitignore, .env.example).

### Entregables
- `flowguard-src`: estructura de carpetas `src/contracts/`, `src/server/agent/`, `src/server/tools/` creada.
- `src/contracts/evidence-report.schema.ts` — contrato Zod del informe del agente.

### Pruebas ejecutadas y resultados
- N/A (todavía no había código ejecutable este día; la prueba llega el día 3).

### Trabajo pendiente
- Adaptador mock (día 2).

### Dependencias de otros developers
- Ninguna bloqueante: el contrato del informe ya estaba congelado por Dev 4 en `flowguard-docs`.

### Próximo objetivo
- Construir `MockAgentProvider`.

---

## Día 2 — 2026-10-16

- Día del plan: 2
- Fecha: 2026-10-16
- Estado general: Completado

### Trabajo realizado
- Implementación de `MockAgentProvider` (`src/server/agent/mockProvider.ts`): genera un informe `COMPLETED` cuando la decisión es `ALLOW`, y `PARTIAL` con un `finding` real cuando hay una regla en `FAIL`.
- Implementación de `validateReport.ts`, `prompt.ts` (prompt exacto de [security.md §1.6](../../seguridad.md#16-prompt-del-agente)) y `index.ts` con selección de proveedor por variable de entorno `AGENT_PROVIDER`.
- Stub de `nebiusProvider.ts` con error explícito que recuerda la dependencia de la checklist del día 7.

### Entregables
- Adaptador de Nebius en modo mock, intercambiable por variable de entorno, según exige el plan.

### Pruebas ejecutadas y resultados
- `npm run typecheck` sin errores.

### Trabajo pendiente
- Prueba unitaria que valide el mock contra el esquema (día 3).

### Dependencias de otros developers
- Ninguna.

### Próximo objetivo
- Escribir y correr la prueba unitaria del mock.

---

## Día 3 — 2026-10-16

- Día del plan: 3
- Fecha: 2026-10-16
- Estado general: Completado

### Trabajo realizado
- Prueba unitaria (`src/server/agent/mockProvider.test.ts`) que valida el informe del mock contra el esquema Zod, tanto para `ALLOW` como para `BLOCK`.
- Esqueleto de las cinco herramientas de lectura (`get_execution`, `get_rule_results`, `get_evidence`, `get_payment_context`, `get_rule_details`) en `src/server/tools/`, cada una lanzando `NotImplementedError` hasta que exista la persistencia real de Dev 1.
- Detectadas y corregidas 6 vulnerabilidades (2 críticas) en dependencias de Vitest al instalar; resuelto actualizando a Vitest 5 (`npm audit` quedó en 0 vulnerabilidades).
- Apertura y fusión de [flowguard-src PR #1](https://github.com/FlowGuard-Hackathon/flowguard-src/pull/1).

### Entregables
- `npm test` ejecuta y pasa 2 pruebas.
- `npm run typecheck` sin errores.
- PR del esqueleto del agente fusionado a `main` en `flowguard-src`.

### Pruebas ejecutadas y resultados
- `npm test` → 2/2 pruebas pasan (mock válido contra el esquema para `ALLOW` y para `BLOCK`).
- `npm run typecheck` → sin errores.
- `npm audit` → 0 vulnerabilidades (tras actualizar Vitest).

### Trabajo pendiente
- Conectar las herramientas de lectura a datos reales (día 4-5), pendiente de que Dev 1 publique `src/server/persistence/`.
- Checklist de compatibilidad con Nebius (día 7) — pendiente de credenciales/documentación oficial, ver [blockers.md](bloqueos.md) si se registra como bloqueo formal.

### Dependencias de otros developers
- Dev 1: `src/server/persistence/` y `POST /api/v1/executions`, necesarios para que las herramientas de lectura dejen de ser stub.

### Próximo objetivo
- Día 4: iniciar `get_execution`, `get_rule_results`, etc. contra datos mock mientras no esté la persistencia real.

---

## Día 4 — 2026-10-17

- Día del plan: 4
- Fecha: 2026-10-17
- Estado general: En progreso (código terminado, sin publicar)

### Trabajo realizado
- **Herramientas de lectura (puesto 3):** las 5 herramientas (`obtener_ejecucion`, `obtener_resultados_reglas`, `obtener_evidencia`, `obtener_contexto_pago`, `obtener_detalle_regla`) implementadas sobre la interfaz `FuenteDatosHerramientas`, que Dev 1 implementará con la persistencia real. Parámetros validados con Zod (`.strict()`), errores del catálogo cerrado y un mock en memoria con datos tomados de los ejemplos de `contratos-api.md`. Incluye una guía para Dev 1 en `src/servidor/herramientas/README.md` con 4 preguntas abiertas (ver [notas.md](notas.md#preguntas-para-el-equipo)).
- **Coordinación:** reorganización del equipo en cinco puestos (PRs #4 a #7 de `flowguard-docs`, y #2 de `flowguard-src`, `flowguard-tests`, `flowguard-mocks` y `flowguard`).
- **Esqueletos:** frontend Next.js (`flowguard-app` #2) y carpetas y variables de entorno del backend (`flowguard-src` #3).
- **Guías operativas:** comandos con resultado esperado en los READMEs de `flowguard-app` (#3) y `flowguard-src` (#4).
- **Decisión de nombres:** todos los identificadores pasan al español. Se crearon el [glosario](../../glosario.md) y las convenciones de [commits](../../convencion-commits.md) y [ramas](../../convencion-ramas.md) (`flowguard-docs` #8). También se hizo el renombrado completo de `flowguard-docs`, pendiente de PR.

### Entregables
- 13 PRs fusionados el 2026-10-17 en 6 repositorios.
- Rama local `dev3/read-tools` (`flowguard-src`) con las herramientas, sin publicar.

### Pruebas ejecutadas y resultados
- `flowguard-src`: `npm test` → 19/19 pruebas pasan (2 del proveedor simulado + 17 de herramientas); `npm run typecheck` sin errores. Ejecutado con Node 20.20: Vitest 5 avisa `EBADENGINE` (pide Node 22.12+), pero las pruebas pasan.
- `flowguard-app`: `npm run typecheck` y `npm run build` pasan; `/`, `/dashboard`, `/simulator` y `/evidence` respondieron 200 en un arranque de prueba (nombres anteriores al glosario).
- Las herramientas solo se probaron contra el mock: **no hay validación contra Supabase real**.

### Trabajo pendiente
- Adaptar las herramientas al glosario, renombrar la rama a `feat/dev3-herramientas-lectura` y abrir el PR.
- Pasos 3 a 6 del renombrado (`flowguard-supabase`, `flowguard-src`, `flowguard-app`, resto de repos).
- Cerrar el día 3: fixture `investigacion-fallida.json` (Dev 5).

### Dependencias de otros desarrolladores
- Dev 1: implementación real de `FuenteDatosHerramientas` y respuestas a las 4 preguntas abiertas.
- Dev 5: fixtures oficiales para alinear los datos simulados.
- Los puestos 1, 2, 4 y 5 siguen sin persona asignada.

### Próximo objetivo
- Publicar las herramientas ya en español y seguir con el día 5 cuando exista persistencia.

### Actualización (mismo día, cierre)
- Las herramientas se adaptaron al glosario y se fusionaron en `flowguard-src` #5 (dos commits: `refactor(src)!` y `feat(herramientas)`). Estado del día 4: **Completado**. `npm test` 19/19 y `typecheck` sin errores tras el renombrado.
- `flowguard-docs` #9: documentación renombrada al español.
- Pasos 3, 5 y 6 del glosario preparados: migración SQL de `flowguard-supabase` en español (marcada como pendiente de verificar en Supabase), rutas en español en `flowguard-app` (`typecheck`, `build` y `/panel`, `/simulador`, `/evidencias` respondiendo 200) y READMEs de `flowguard-mocks`, `flowguard-tests` y `flowguard`.
- Decisión de coordinación: no se agregan tecnologías fuera del stack (por ejemplo Docker) porque el resto del equipo tiene poca experiencia.

### Dependencias de otros developers
-

### Próximo objetivo
-

---

## Día 5 — [fecha]

- Día del plan:
- Fecha:
- Estado general:

### Trabajo realizado
-

### Entregables
-

### Pruebas ejecutadas y resultados
-

### Trabajo pendiente
-

### Dependencias de otros developers
-

### Próximo objetivo
-

---

## Día 6 — [fecha]

- Día del plan:
- Fecha:
- Estado general:

### Trabajo realizado
-

### Entregables
-

### Pruebas ejecutadas y resultados
-

### Trabajo pendiente
-

### Dependencias de otros developers
-

### Próximo objetivo
-

---

## Día 7 — [fecha]

- Día del plan:
- Fecha:
- Estado general:

### Trabajo realizado
-

### Entregables
-

### Pruebas ejecutadas y resultados
-

### Trabajo pendiente
-

### Dependencias de otros developers
-

### Próximo objetivo
-

---

## Día 8 — [fecha]

- Día del plan:
- Fecha:
- Estado general:

### Trabajo realizado
-

### Entregables
-

### Pruebas ejecutadas y resultados
-

### Trabajo pendiente
-

### Dependencias de otros developers
-

### Próximo objetivo
-

---

## Día 9 — [fecha]

- Día del plan:
- Fecha:
- Estado general:

### Trabajo realizado
-

### Entregables
-

### Pruebas ejecutadas y resultados
-

### Trabajo pendiente
-

### Dependencias de otros developers
-

### Próximo objetivo
-

---

## Día 10 — [fecha]

- Día del plan:
- Fecha:
- Estado general:

### Trabajo realizado
-

### Entregables
-

### Pruebas ejecutadas y resultados
-

### Trabajo pendiente
-

### Dependencias de otros developers
-

### Próximo objetivo
-

---

## Día 11 — [fecha]

- Día del plan:
- Fecha:
- Estado general:

### Trabajo realizado
-

### Entregables
-

### Pruebas ejecutadas y resultados
-

### Trabajo pendiente
-

### Dependencias de otros developers
-

### Próximo objetivo
-

---

## Día 12 — [fecha]

- Día del plan:
- Fecha:
- Estado general:

### Trabajo realizado
-

### Entregables
-

### Pruebas ejecutadas y resultados
-

### Trabajo pendiente
-

### Dependencias de otros developers
-

### Próximo objetivo
-

---

## Día 13 — [fecha]

- Día del plan:
- Fecha:
- Estado general:

### Trabajo realizado
-

### Entregables
-

### Pruebas ejecutadas y resultados
-

### Trabajo pendiente
-

### Dependencias de otros developers
-

### Próximo objetivo
-

---

## Día 14 — [fecha]

- Día del plan:
- Fecha:
- Estado general:

### Trabajo realizado
-

### Entregables
-

### Pruebas ejecutadas y resultados
-

### Trabajo pendiente
-

### Dependencias de otros developers
-

### Próximo objetivo
-

---

## Día 15 — [fecha]

- Día del plan:
- Fecha:
- Estado general:

### Trabajo realizado
-

### Entregables
-

### Pruebas ejecutadas y resultados
-

### Trabajo pendiente
-

### Dependencias de otros developers
-

### Próximo objetivo
-
