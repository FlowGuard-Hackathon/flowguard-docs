# Cuaderno técnico — Developer 3

Notas personales de trabajo. Cada nota debe incluir fecha, contexto y estado, para que otro integrante del equipo pueda entenderla sin preguntar. No se registran secretos, tokens, contraseñas ni credenciales aquí ni en ningún otro archivo de este repositorio.

## Decisiones técnicas

<!-- Formato sugerido por nota:
### [fecha] Título de la decisión
- Contexto:
- Decisión tomada:
- Alternativas consideradas:
- Estado: Propuesta | Aprobada | Revertida
-->

### [2026-10-16] Esqueleto inicial del módulo del agente en flowguard-src

- Contexto: inicio del Bloque 1 (días 1-3) del plan. Había que arrancar el adaptador de Nebius sin depender todavía de credenciales reales ni de la persistencia de Dev 1.
- Decisión tomada: inicializar `flowguard-src` con TypeScript + Zod + Vitest, y construir primero el proveedor **mock** (`MockAgentProvider`) en vez de empezar directo con `NebiusAgentProvider`, siguiendo la secuencia del plan (día 2 = mock, día 7-9 = integración real).
- Alternativas consideradas: ninguna — el plan ya fija el mock como paso previo obligatorio antes de la integración real (ver [implementation-checklist.md, Bloque 1](../../implementation-checklist.md#bloque-1--días-13-congelación-de-contratos-y-estructura-del-proyecto)).
- Estado: Aprobada (PR fusionado).
- Referencia: [flowguard-src PR #1](https://github.com/FlowGuard-Hackathon/flowguard-src/pull/1).

### [2026-10-16] Las 5 herramientas de lectura quedan como stub, no como mock de datos

- Contexto: las herramientas (`get_execution`, `get_rule_results`, `get_evidence`, `get_payment_context`, `get_rule_details`) necesitan datos reales de `src/server/persistence/`, que todavía no existe (es de Dev 1).
- Decisión tomada: cada herramienta lanza `NotImplementedError` explícito en vez de devolver datos inventados, para no simular un comportamiento que no es real.
- Alternativas consideradas: devolver datos mock por herramienta — descartada porque el plan (día 4-5) indica implementarlas "contra datos mock" solo como paso intermedio documentado, no como sustituto silencioso de la persistencia real; se prefirió dejar explícito el bloqueo.
- Estado: Aprobada.

## Hallazgos

<!-- Formato sugerido:
### [fecha] Título del hallazgo
- Contexto:
- Qué se encontró:
- Impacto:
- Estado: Confirmado | En investigación
-->

## Ideas pendientes de validar

<!-- Formato sugerido:
### [fecha] Idea
- Contexto:
- Por qué podría valer la pena:
- Qué falta para validarla:
- Estado: Pendiente de validar
-->

## Comandos útiles

<!-- Formato sugerido:
### [fecha] Descripción del comando
```
comando aquí
```
- Contexto / cuándo usarlo:
-->

## Referencias a archivos o Pull Requests

<!-- Formato sugerido:
### [fecha] Referencia
- Archivo o PR:
- Por qué es relevante:
-->

### [2026-10-16] flowguard-src PR #1 — esqueleto del agente

- Archivo o PR: [flowguard-src#1](https://github.com/FlowGuard-Hackathon/flowguard-src/pull/1) (rama `dev3/agent-skeleton`, fusionada a `main`)
- Por qué es relevante: es el punto de partida de todo mi trabajo. Contiene:
  - `src/contracts/evidence-report.schema.ts` — contrato Zod del informe, según [api-contracts.md §10](../../api-contracts.md#10-contrato-del-informe-del-agente).
  - `src/server/agent/mockProvider.ts` — proveedor mock funcional (probado con `npm test`).
  - `src/server/agent/nebiusProvider.ts` — stub, bloqueado hasta completar la checklist del día 7.
  - `src/server/agent/validateReport.ts`, `prompt.ts`, `index.ts` (selección por `AGENT_PROVIDER`).
  - `src/server/tools/` — firmas de las 5 herramientas de lectura, pendientes de `src/server/persistence/` de Dev 1.
  - `mockProvider.test.ts` — prueba unitaria exigida en el día 3 del plan, pasando.

## Preguntas para el equipo

<!-- Formato sugerido:
### [fecha] Pregunta
- Contexto:
- Pregunta:
- Estado: Abierta | Respondida (anotar la respuesta y quién la dio)
-->
