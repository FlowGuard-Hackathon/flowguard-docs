# Bitácora diaria — Developer 3

Copia la plantilla de abajo para cada día de trabajo. No se rellenan avances inventados: cada sección se completa el día que realmente ocurre el trabajo.

## Cómo usar esta bitácora

- Actualízala al terminar tu jornada, según la regla 1 de [README.md](../../README.md#reglas-de-actualización-y-colaboración).
- Si una sección no aplica ese día, déjala vacía o escribe "N/A"; no inventes contenido para rellenarla.
- Los bloqueos reales van en [blockers.md](blockers.md), no aquí.

---

## Día 1 — 2026-10-16

- Día del plan: 1
- Fecha: 2026-10-16
- Estado general: Completado

### Trabajo realizado
- Revisión del contrato del informe del agente ya definido en [api-contracts.md §10](../../api-contracts.md#10-contrato-del-informe-del-agente); no se necesitaron campos adicionales para el adaptador de Nebius.
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
- Implementación de `validateReport.ts`, `prompt.ts` (prompt exacto de [security.md §1.6](../../security.md#16-prompt-del-agente)) y `index.ts` con selección de proveedor por variable de entorno `AGENT_PROVIDER`.
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
- Checklist de compatibilidad con Nebius (día 7) — pendiente de credenciales/documentación oficial, ver [blockers.md](blockers.md) si se registra como bloqueo formal.

### Dependencias de otros developers
- Dev 1: `src/server/persistence/` y `POST /api/v1/executions`, necesarios para que las herramientas de lectura dejen de ser stub.

### Próximo objetivo
- Día 4: iniciar `get_execution`, `get_rule_results`, etc. contra datos mock mientras no esté la persistencia real.

---

## Día 4 — [fecha]

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
