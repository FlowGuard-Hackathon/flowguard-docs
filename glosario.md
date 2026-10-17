# Glosario de identificadores

Fuente única de verdad para nombrar **todo** en FlowGuard: endpoints, campos JSON, tablas, columnas, funciones SQL, estados, códigos de error, variables de entorno, herramientas del agente, archivos, carpetas y código TypeScript. Si un documento o archivo usa un nombre distinto al de este glosario, manda el glosario.

Convenciones relacionadas: [convencion-commits.md](convencion-commits.md) y [convencion-ramas.md](convencion-ramas.md).

## Decisión

| Campo | Contenido |
|---|---|
| Fecha | 2026-10-17 |
| Decide | Quien coordina el proyecto (puesto 3) |
| Decisión | Todos los identificadores y mensajes del sistema pasan al español, incluidos los contratos de la API, la base de datos y los códigos de error |
| Motivo | Equipo hispanohablante; se decide antes de que exista código del backend o de la base de datos, cuando el costo del cambio es mínimo |
| Contexto | Hasta esta fecha los contratos estaban congelados en inglés. Este cambio reemplaza esos nombres; **no** cambia la arquitectura, el stack, las reglas de negocio, el modelo de datos ni el comportamiento de ningún componente: solo los nombres |
| Escalabilidad | Los textos visibles (mensajes al usuario y al modelo) se centralizan en catálogos de mensajes con claves estables (`mensajes/es.ts`); agregar otro idioma consiste en añadir otro catálogo, sin tocar la lógica |
| Estado | **Migración en curso.** Los documentos y el código que aún usan nombres en inglés se renombran por repositorio, en el orden de la sección final |

## Reglas de nomenclatura

| Dónde | Formato | Ejemplo |
|---|---|---|
| Campos JSON, tablas, columnas, funciones SQL | `snake_case` | `ejecucion_id`, `evaluaciones_reglas` |
| Enums, estados y códigos de error | `MAYUSCULAS_CON_GUION_BAJO` | `BLOQUEAR`, `EJECUCION_NO_ENCONTRADA` |
| Variables de entorno | `MAYUSCULAS_CON_GUION_BAJO` | `AGENTE_TIEMPO_LIMITE_MS` |
| Variables y funciones TypeScript | `camelCase` | `crearHerramientasLectura` |
| Tipos, clases e interfaces TypeScript | `PascalCase` | `FuenteDatosHerramientas` |
| Constantes TypeScript | `MAYUSCULAS_CON_GUION_BAJO` | `SEMILLA_SIMULADA` |
| Archivos y carpetas | `kebab-case` (documentos) o el formato del identificador que contienen | `contratos-api.md`, `informe-evidencias.schema.ts` |
| URLs | minúsculas, `kebab-case` | `/api/v1/ejecuciones` |

- **Sin acentos ni ñ en ningún identificador.** Se escribe `ejecucion`, `contrasena`, `tamano`. Los acentos y la ñ dan problemas en URLs, SQL, variables de entorno y algunas herramientas. Los **textos** para personas sí llevan acentos.
- **Palabras clave en orden natural del español:** `ejecucion_id` (no `id_ejecucion`), `monto_centavos`, `estado_investigacion`.
- **Plural para colecciones, singular para un elemento:** `reglas` (arreglo), `regla_id` (uno).

### Excepciones: lo que no se traduce

| Qué | Motivo |
|---|---|
| Nombres propios: Nebius, NVIDIA, Supabase, PostgreSQL, Next.js, React, Zod, Vitest, Playwright | Son marcas |
| `id`, `json`, `url`, `api`, `hash`, `prompt`, `mxn` cuando forman parte de un identificador | Términos técnicos universales |
| Moneda `MXN`, métodos HTTP (`GET`, `POST`), palabras reservadas de SQL | Estándares |
| Prefijo `/api/v1` | Convención de versionado de la API |
| Carpeta `app/` de Next.js y archivos `page.tsx`, `layout.tsx`, `route.ts`, `next.config.ts` | Los exige el framework |
| Prefijo `NEXT_PUBLIC_` | Next.js lo exige para exponer una variable al navegador |
| Scripts `npm test`, `npm run dev`, `npm run build`, `npm run start`, `npm run typecheck` | Convención de npm y de las herramientas |
| Tipos de commit (`feat`, `fix`…) | Estándar Conventional Commits |
| Nombres de los repositorios (`flowguard-*`) | Renombrarlos rompería enlaces, clones e integraciones |

## Endpoints

| Antes | Ahora |
|---|---|
| `POST /api/v1/executions` | `POST /api/v1/ejecuciones` |
| `GET /api/v1/executions` | `GET /api/v1/ejecuciones` |
| `GET /api/v1/executions/:id` | `GET /api/v1/ejecuciones/:id` |
| `GET /api/v1/executions/:id/evidence` | `GET /api/v1/ejecuciones/:id/evidencias` |
| `POST /api/v1/executions/:id/investigate` | `POST /api/v1/ejecuciones/:id/investigar` |
| `GET /api/v1/rules` | `GET /api/v1/reglas` |
| `GET /api/v1/health` | `GET /api/v1/salud` |

## Campos JSON de la API

| Antes | Ahora | | Antes | Ahora |
|---|---|---|---|---|
| `action` | `accion` | | `investigation` | `investigacion` |
| `agent` | `agente` | | `investigation_status` | `estado_investigacion` |
| `agent_can_override` | `agente_puede_anular` | | `invoice_verified` | `factura_verificada` |
| `agent_report` | `informe_agente` | | `items` | `elementos` |
| `allowed_categories` | `categorias_permitidas` | | `last_attempt_id` | `ultimo_intento_id` |
| `amount` | `monto` | | `last_updated_at` | `actualizado_en` |
| `amount_minor` | `monto_centavos` | | `limitations` | `limitaciones` |
| `amount_minor_lte_available` | `monto_no_supera_disponible` | | `may_advance` | `puede_avanzar` |
| `amount_mxn` | `monto_mxn` | | `message` | `mensaje` |
| `approval_required` | `aprobacion_requerida` | | `model` | `modelo` |
| `approval_status` | `estado_aprobacion` | | `name` | `nombre` |
| `approval_threshold_minor` | `umbral_aprobacion_centavos` | | `observed` | `observado` |
| `approver_id` | `aprobador_id` | | `page` | `pagina` |
| `approver_required_for_amount` | `aprobador_requerido_por_monto` | | `page_size` | `tamano_pagina` |
| `attachment_id` | `adjunto_id` | | `passed` | `cumplidas` |
| `attachments` | `adjuntos` | | `payload` | `contenido` |
| `attempt_id` | `intento_id` | | `payment` | `pago` |
| `attempts` | `intentos` | | `payment_id` | `pago_id` |
| `available_budget_minor` | `presupuesto_disponible_centavos` | | `policy_version` | `version_politica` |
| `budget_id` | `presupuesto_id` | | `prompt_version` | `version_prompt` |
| `category` | `categoria` | | `provider` | `proveedor` |
| `claim` | `afirmacion` | | `recommended_actions` | `acciones_recomendadas` |
| `code` | `codigo` | | `reference` | `referencia` |
| `config` | `configuracion` | | `report` | `informe` |
| `context` | `contexto` | | `request_id` | `solicitud_id` |
| `created_at` | `creado_en` | | `requested_by` | `solicitado_por` |
| `currency` | `moneda` | | `requester_id` | `solicitante_id` |
| `database` | `base_datos` | | `requires_human_action` | `requiere_accion_humana` |
| `decision` | `decision` | | `requires_investigation` | `requiere_investigacion` |
| `decision_source` | `origen_decision` | | `rule_id` | `regla_id` |
| `decision_summary` | `resumen_decision` | | `rules` | `reglas` |
| `dependencies` | `dependencias` | | `scenario_id` | `escenario_id` |
| `description` | `descripcion` | | `schema_version` | `version_esquema` |
| `details` | `detalles` | | `source` | `origen` |
| `enabled` | `habilitada` | | `status` | `estado` |
| `engine_version` | `version_motor` | | `summary` | `resumen` |
| `error` | `error` | | `total_items` | `total_elementos` |
| `event` | `evento` | | `total_rules` | `total_reglas` |
| `evidence_records` | `registros_evidencia` | | `type` | `tipo` |
| `evidence_ref` | `ref_evidencia` | | `user_id` | `usuario_id` |
| `evidence_refs` | `refs_evidencia` | | `verified_authorization` | `autorizacion_verificada` |
| `evidence_type` | `tipo_evidencia` | | `execution_controls` | `controles_ejecucion` |
| `execution_id` | `ejecucion_id` | | `execution_status` | `estado_ejecucion` |
| `expected` | `esperado` | | `explanation` | `explicacion` |
| `failed` | `no_cumplidas` | | `indeterminate` | `indeterminadas` |
| `field` | `campo` | | `finding_id` | `hallazgo_id` |
| `findings` | `hallazgos` | | `headline` | `titular` |
| `idempotency_key` | `clave_idempotencia` | | `nebius` | `nebius` |

## Valores cerrados (enums)

| Concepto | Antes | Ahora |
|---|---|---|
| Decisión | `ALLOW`, `BLOCK` | `PERMITIR`, `BLOQUEAR` |
| Resultado de regla | `PASS`, `FAIL`, `INDETERMINATE` | `CUMPLE`, `NO_CUMPLE`, `INDETERMINADO` |
| Severidad de regla | `BLOCKING`, `INFO` | `BLOQUEANTE`, `INFORMATIVA` |
| Estado de ejecución | `RECEIVED`, `VALIDATING`, `EVALUATING`, `PERSISTED`, `FAILED` | `RECIBIDA`, `VALIDANDO`, `EVALUANDO`, `PERSISTIDA`, `FALLIDA` |
| Estado de investigación y de intento | `NOT_STARTED`, `PENDING`, `RUNNING`, `COMPLETED`, `PARTIAL`, `FAILED` | `NO_INICIADA`, `PENDIENTE`, `EN_CURSO`, `COMPLETADA`, `PARCIAL`, `FALLIDA` |
| Resultado de programar un intento | `CREATED`, `IN_PROGRESS`, `ALREADY_COMPLETED` | `CREADO`, `EN_PROGRESO`, `YA_COMPLETADA` |
| Estado de aprobación | `APPROVED`, `NOT_APPROVED` | `APROBADO`, `NO_APROBADO` |
| Origen de la decisión | `RULE_ENGINE` | `MOTOR_REGLAS` |
| Tipo de evento | `PAYMENT_REQUESTED` | `PAGO_SOLICITADO` |
| Tipo de adjunto | `INVOICE` | `FACTURA` |
| Categorías | `OPERATIONS`, `SOFTWARE`, `TRAVEL` | `OPERACIONES`, `SOFTWARE`, `VIAJES` |
| Tipos de evidencia | `APPROVAL_RECORD`, `INVOICE_RECORD`, `APPROVAL_POLICY`, `CATEGORY_RECORD`, `BUDGET_BALANCE` | `REGISTRO_APROBACION`, `REGISTRO_FACTURA`, `POLITICA_APROBACION`, `REGISTRO_CATEGORIA`, `SALDO_PRESUPUESTO` |
| Moneda | `MXN` | `MXN` (sin cambio) |

## Reglas de negocio e identificadores de ejemplo

| Antes | Ahora |
|---|---|
| `PAY-001` … `PAY-005` | `PAG-001` … `PAG-005` |
| Escenario `payment-approval-v1` | `aprobacion-pago-v1` |
| Política `payment-policy-v1` | `politica-pagos-v1` |
| Ejecución `EXE-2026-0001` | `EJE-2026-0001` |
| Intento `INV-ATT-0001` | `INT-2026-0001` |
| Evidencia `EV-0001` | `EV-0001` (sin cambio) |
| Pago `PAY-2026-001` | `PAG-2026-001` |
| Usuario `USR-001` | `USU-001` |
| Adjunto `ATT-001` | `ADJ-001` |
| Factura `INV-2026-001` | `FAC-2026-001` |
| Presupuesto `BUD-OPERATIONS-01` | `PRE-OPERACIONES-01` |

## Códigos de error

| Antes | Ahora |
|---|---|
| `VALIDATION_ERROR` | `ERROR_VALIDACION` |
| `SEMANTIC_VALIDATION_ERROR` | `ERROR_VALIDACION_SEMANTICA` |
| `BUDGET_NOT_FOUND` | `PRESUPUESTO_NO_ENCONTRADO` |
| `ATTACHMENT_NOT_FOUND` | `ADJUNTO_NO_ENCONTRADO` |
| `EXECUTION_NOT_FOUND` | `EJECUCION_NO_ENCONTRADA` |
| `UNAUTHENTICATED` | `NO_AUTENTICADO` |
| `FORBIDDEN` | `PROHIBIDO` |
| `IDEMPOTENCY_KEY_REUSED` | `CLAVE_IDEMPOTENCIA_REUTILIZADA` |
| `IDEMPOTENCY_IN_PROGRESS` | `IDEMPOTENCIA_EN_PROGRESO` |
| `INVESTIGATION_IN_PROGRESS` | `INVESTIGACION_EN_CURSO` |
| `INVESTIGATION_ALREADY_COMPLETED` | `INVESTIGACION_YA_COMPLETADA` |
| `RATE_LIMITED` | `LIMITE_SOLICITUDES_EXCEDIDO` |
| `PERSISTENCE_FAILURE` | `FALLO_PERSISTENCIA` |
| `DEPENDENCY_UNAVAILABLE` | `DEPENDENCIA_NO_DISPONIBLE` |
| `PROVIDER_TIMEOUT` | `TIEMPO_AGOTADO_PROVEEDOR` |
| `PROVIDER_INVALID_RESPONSE` | `RESPUESTA_INVALIDA_PROVEEDOR` |
| `EVIDENCE_REFERENCE_INVALID` | `REFERENCIA_EVIDENCIA_INVALIDA` |
| `AGENT_TOOL_LIMIT_EXCEEDED` | `LIMITE_HERRAMIENTAS_AGENTE_EXCEDIDO` |
| `WORKER_LOST` | `PROCESO_PERDIDO` |
| `INTERNAL_ERROR` | `ERROR_INTERNO` |

## Base de datos

### Tablas

| Antes | Ahora |
|---|---|
| `flow_definitions` | `definiciones_flujo` |
| `policy_rules` | `reglas_politica` |
| `budget_balances` | `saldos_presupuesto` |
| `payment_requests` | `solicitudes_pago` |
| `idempotency_keys` | `claves_idempotencia` |
| `executions` | `ejecuciones` |
| `rule_evaluations` | `evaluaciones_reglas` |
| `evidence_records` | `registros_evidencia` |
| `investigation_attempts` | `intentos_investigacion` |
| `agent_reports` | `informes_agente` |
| `rate_limit_counters` | `contadores_limite_solicitudes` |

### Columnas

Las columnas que coinciden con un campo JSON usan el mismo nombre de la tabla de campos (por ejemplo `execution_id` → `ejecucion_id`, `created_at` → `creado_en`). Las propias de la base de datos:

| Antes | Ahora | | Antes | Ahora |
|---|---|---|---|---|
| `is_active` | `activa` | | `attempt_code` | `codigo_intento` |
| `version` | `version` | | `attempt_number` | `numero_intento` |
| `config_json` | `configuracion_json` | | `tool_calls_used` | `llamadas_herramientas_usadas` |
| `available_minor` | `disponible_centavos` | | `error_code` | `codigo_error` |
| `updated_at` | `actualizado_en` | | `claimed_by` | `reclamado_por` |
| `request_hash` | `hash_solicitud` | | `claimed_at` | `reclamado_en` |
| `execution_code` | `codigo_ejecucion` | | `started_at` | `iniciado_en` |
| `payment_request_id` | `solicitud_pago_id` | | `completed_at` | `completado_en` |
| `latest_attempt_number` | `numero_ultimo_intento` | | `report_json` | `informe_json` |
| `observed_json` | `observado_json` | | `rate_key` | `clave_limite` |
| `expected_json` | `esperado_json` | | `window_start` | `inicio_ventana` |
| `payload_json` | `contenido_json` | | `request_count` | `conteo_solicitudes` |
| `severity` | `severidad` | | `requester_id` | `solicitante_id` |

### Funciones PL/pgSQL

| Antes | Ahora |
|---|---|
| `fn_schedule_investigation_attempt` | `fn_programar_intento_investigacion` |
| `fn_finalize_investigation_attempt` | `fn_finalizar_intento_investigacion` |
| `fn_reap_abandoned_attempts` | `fn_recuperar_intentos_abandonados` |
| `fn_check_rate_limit` | `fn_verificar_limite_solicitudes` |

Los parámetros (`p_*`) y variables internas (`v_*`) de estas funciones se renombran con el mismo criterio, conservando los prefijos `p_` y `v_`.

## Variables de entorno

| Antes | Ahora |
|---|---|
| `AGENT_PROVIDER` | `AGENTE_PROVEEDOR` |
| `NEBIUS_API_KEY` | `NEBIUS_CLAVE_API` |
| `NEBIUS_BASE_URL` | `NEBIUS_URL_BASE` |
| `NEBIUS_MODEL` | `NEBIUS_MODELO` |
| `AGENT_MAX_TOOL_CALLS` | `AGENTE_MAX_LLAMADAS_HERRAMIENTAS` |
| `AGENT_TIMEOUT_MS` | `AGENTE_TIEMPO_LIMITE_MS` |
| `AGENT_WORKER_CONCURRENCY` | `AGENTE_CONCURRENCIA_PROCESOS` |
| `AGENT_WORKER_POLL_INTERVAL_MS` | `AGENTE_INTERVALO_SONDEO_MS` |
| `RECONCILER_INTERVAL_MS` | `RECONCILIADOR_INTERVALO_MS` |
| `SUPABASE_SERVICE_ROLE_KEY` | `SUPABASE_CLAVE_ROL_SERVICIO` |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | `NEXT_PUBLIC_SUPABASE_CLAVE_ANONIMA` |
| `RATE_LIMIT_ENABLED` | `LIMITE_SOLICITUDES_ACTIVO` |
| `TRUSTED_PROXY_HEADER` | `ENCABEZADO_PROXY_CONFIABLE` |
| `DEMO_BASIC_AUTH_USER` | `DEMO_AUTH_BASICA_USUARIO` |
| `DEMO_BASIC_AUTH_PASSWORD` | `DEMO_AUTH_BASICA_CONTRASENA` |

Valor de `AGENTE_PROVEEDOR`: `mock` pasa a `simulado`; `nebius` se mantiene.

## Herramientas del agente (las que pide el LLM)

| Antes | Ahora |
|---|---|
| `get_execution` | `obtener_ejecucion` |
| `get_rule_details` | `obtener_detalle_regla` |
| `get_rule_results` | `obtener_resultados_reglas` |
| `get_evidence` | `obtener_evidencia` |
| `get_payment_context` | `obtener_contexto_pago` |

## Código TypeScript (`flowguard-src`)

### Carpetas y archivos

| Antes | Ahora |
|---|---|
| `src/contracts/` | `src/contratos/` |
| `src/contracts/payment.schema.ts` | `src/contratos/pago.schema.ts` |
| `src/contracts/execution.schema.ts` | `src/contratos/ejecucion.schema.ts` |
| `src/contracts/engine.schema.ts` | `src/contratos/motor.schema.ts` |
| `src/contracts/evidence-report.schema.ts` | `src/contratos/informe-evidencias.schema.ts` |
| `src/server/` | `src/servidor/` |
| `src/server/agent/` | `src/servidor/agente/` |
| `src/server/tools/` | `src/servidor/herramientas/` |
| `src/server/rules/` | `src/servidor/reglas/` |
| `src/server/executions/` | `src/servidor/ejecuciones/` |
| `src/server/persistence/` | `src/servidor/persistencia/` |
| `src/shared/` | `src/compartido/` |
| `src/messages/` (nuevo) | `src/mensajes/` con `es.ts` |

### Nombres principales

| Antes | Ahora |
|---|---|
| `evaluatePayment` | `evaluarPago` |
| `EngineDecision` | `DecisionMotor` |
| `trustedContext`, `policySnapshot` | `contextoConfiable`, `instantaneaPolitica` |
| `toMinorUnits` | `aCentavos` |
| `AgentProvider` | `ProveedorAgente` |
| `MockAgentProvider` | `ProveedorAgenteSimulado` |
| `NebiusAgentProvider` | `ProveedorAgenteNebius` |
| `InvestigationInput`, `InvestigationOutcome` | `EntradaInvestigacion`, `ResultadoInvestigacion` |
| `validateReport` | `validarInforme` |
| `EvidenceReportSchema`, `EvidenceReport`, `Finding` | `EsquemaInformeEvidencias`, `InformeEvidencias`, `Hallazgo` |
| `AGENT_SYSTEM_PROMPT`, `AGENT_PROMPT_VERSION` | `PROMPT_SISTEMA_AGENTE`, `VERSION_PROMPT_AGENTE` |
| `createReadTools` | `crearHerramientasLectura` |
| `ToolDataSource` | `FuenteDatosHerramientas` |
| `InMemoryToolDataSource` | `FuenteDatosEnMemoria` |
| `SIMULATED_SEED` | `SEMILLA_SIMULADA` |
| `ToolResult`, `ToolErrorCode` | `ResultadoHerramienta`, `CodigoErrorHerramienta` |

Cualquier nombre nuevo que no esté en esta tabla se forma con las reglas de nomenclatura de arriba y, si se vuelve parte de un contrato, se agrega aquí en el mismo PR.

## `flowguard-app`

| Antes | Ahora |
|---|---|
| `app/dashboard/` | `app/panel/` |
| `app/simulator/` | `app/simulador/` |
| `app/evidence/` | `app/evidencias/` |
| `app/api/v1/executions/`, `rules/`, `health/` | `app/api/v1/ejecuciones/`, `reglas/`, `salud/` |

## Documentación (`flowguard-docs`)

| Antes | Ahora |
|---|---|
| `architecture.md` | `arquitectura.md` |
| `api-contracts.md` | `contratos-api.md` |
| `database.md` | `base-datos.md` |
| `security.md` | `seguridad.md` |
| `rules.md` | `reglas.md` |
| `error-catalog.md` | `catalogo-errores.md` |
| `testing.md` | `pruebas.md` |
| `implementation-checklist.md` | `checklist-implementacion.md` |
| `demo-script.md` | `guion-demo.md` |
| `development-plan.md` | `plan-desarrollo.md` |
| `integration-and-dependencies.md` | `integracion-y-dependencias.md` |
| `definition-of-done.md` | `definicion-de-terminado.md` |
| `developers/developer-N/` | `desarrolladores/desarrollador-N/` |
| `work-plan.md`, `daily-log.md`, `notes.md`, `blockers.md` | `plan-trabajo.md`, `bitacora.md`, `notas.md`, `bloqueos.md` |

## Orden de la migración

Cada paso es un PR independiente con commits `refactor(...)!` o `docs(...)!` según [convencion-commits.md](convencion-commits.md):

1. `flowguard-docs`: este glosario y las convenciones (este PR).
2. `flowguard-docs`: renombrado de documentos y de todos los identificadores dentro de ellos.
3. `flowguard-supabase`: migración SQL con tablas, columnas, funciones y valores en español. **La verificación previa contra PostgreSQL real deja de aplicar** y debe repetirse antes de volver a marcarla como verificada.
4. `flowguard-src`: carpetas, contratos Zod, agente, herramientas y catálogo `src/mensajes/es.ts`.
5. `flowguard-app`: rutas de páginas y de la API.
6. `flowguard-mocks`, `flowguard-tests` y `flowguard`: READMEs y referencias.
