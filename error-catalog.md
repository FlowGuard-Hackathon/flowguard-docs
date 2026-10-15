# Catálogo de errores

Todos los errores de la API siguen el formato uniforme definido en [api-contracts.md § 1](api-contracts.md#1-formato-uniforme-de-error). Esta tabla es la lista cerrada de códigos admitidos para el MVP; un código nuevo debe agregarse aquí antes de usarse en el backend.

**Distinción importante:** un *error técnico* (esta tabla) es distinto de una *infracción de negocio*. Una regla que falla (`PAY-00X: FAIL`) no es un error: es un resultado válido del motor, representado con HTTP `200`/`201` y `decision: "BLOCK"`. Los códigos de esta tabla representan fallos del sistema (validación, infraestructura, proveedor externo), no decisiones de negocio.

| Código | HTTP | Descripción | Causa probable | ¿Reintentable? | Comportamiento esperado del frontend | Efecto sobre `decision` | ¿Se registra en logs? |
|---|---|---|---|---|---|---|---|
| `VALIDATION_ERROR` | 400 | El payload no cumple el esquema o las reglas de formato (tipo, rango, campos obligatorios) | Cliente envía datos malformados | No (hay que corregir el payload) | Mostrar el campo inválido tomado de `details.field` | No se evalúa; no existe decisión | Sí, nivel `warn` |
| `SEMANTIC_VALIDATION_ERROR` | 422 | El payload es sintácticamente válido pero semánticamente inconsistente (p. ej. moneda no admitida) | Cliente envía una combinación de valores no soportada | No | Mostrar mensaje específico | No se evalúa; no existe decisión | Sí, nivel `warn` |
| `BUDGET_NOT_FOUND` | 404 | El `budget_id` referenciado no existe en el contexto confiable | Fixture/demo mal configurada o `budget_id` inválido | No | Pedir al usuario elegir un presupuesto válido | No se evalúa | Sí, nivel `warn` |
| `ATTACHMENT_NOT_FOUND` | 404 | La referencia de adjunto no existe en el contexto confiable | Fixture/demo mal configurada | No | Pedir una referencia de factura válida | No se evalúa | Sí, nivel `warn` |
| `EXECUTION_NOT_FOUND` | 404 | El `execution_id` no existe | Enlace o id incorrecto | No | Mostrar "ejecución no encontrada" | N/A | No necesariamente |
| `UNAUTHENTICATED` | 401 | Falta identidad válida (cuando el entorno la exige, ver [security.md](security.md)) | Sesión ausente o expirada | Sí, tras reautenticar | Redirigir a login | No se evalúa | Sí, nivel `warn` |
| `FORBIDDEN` | 403 | La identidad no tiene permiso para la operación | Rol insuficiente | No | Mostrar "sin permisos" | No se evalúa | Sí, nivel `warn` |
| `IDEMPOTENCY_KEY_REUSED` | 409 | La misma `idempotency_key` se usó con un payload de contenido distinto (`request_hash` no coincide) — se verifica **antes** que el estado de progreso, así que tiene precedencia sobre `IDEMPOTENCY_IN_PROGRESS` | Error del cliente o reintento incorrecto | No (requiere una clave nueva) | Mostrar conflicto; sugerir generar una nueva clave | No se evalúa en esta llamada | Sí, nivel `warn` |
| `IDEMPOTENCY_IN_PROGRESS` | 409 | Ya existe una solicitud con la misma `idempotency_key` y el mismo contenido todavía procesándose (`execution_status` en `RECEIVED`/`VALIDATING`/`EVALUATING`) | Doble clic o reintento muy rápido del cliente | Sí — **reintentar la misma solicitud `POST /executions`** (idéntica) con backoff; no existe un endpoint separado de consulta previo a que exista `execution_id` | Reintentar automáticamente con backoff; no mostrar error al usuario en el primer reintento | Pendiente (el reintento eventualmente recibe `200`/`201` o un error técnico) | Sí, nivel `info` |
| `INVESTIGATION_IN_PROGRESS` | 409 | Ya hay un intento de investigación `PENDING`/`RUNNING` | Doble llamada a `POST /investigate` | Sí, consultando el estado actual | No disparar un nuevo intento; mostrar el actual | Sin efecto (la decisión ya estaba persistida) | Sí, nivel `info` |
| `INVESTIGATION_ALREADY_COMPLETED` | 409 | Se pidió investigar una ejecución con `investigation_status = COMPLETED` | Cliente reintenta innecesariamente | No | Mostrar el informe ya existente | Sin efecto | Sí, nivel `info` |
| `RATE_LIMITED` | 429 | Se superó el límite de solicitudes para el `rate_key` y grupo de endpoint correspondientes (ver límites concretos y cálculo de `rate_key` en [security.md § 2.4](security.md#24-límite-de-solicitudes-rate-limiting)) | Abuso o pico de tráfico | Sí, respetando la cabecera `Retry-After` devuelta | Mostrar "demasiadas solicitudes, intenta de nuevo en N s" usando `Retry-After`/`details.window_seconds` | No se evalúa | Sí, nivel `warn` |
| `PERSISTENCE_FAILURE` | 500 | La transacción de persistencia de la ejecución falló | Base de datos no disponible o error inesperado | Sí, reenviando con la misma `idempotency_key` | Mostrar error genérico; permitir reintentar | **No crea decisión**; nunca se infiere `ALLOW` | Sí, nivel `error` |
| `DEPENDENCY_UNAVAILABLE` | 503 | Una dependencia necesaria (base de datos u otra) no está disponible | Caída de infraestructura | Sí, con backoff | Mostrar estado de servicio degradado | No se evalúa | Sí, nivel `error` |
| `PROVIDER_TIMEOUT` | — (no es respuesta HTTP al cliente; se registra en `investigation_attempts.error_code`) | Nebius no respondió dentro de `AGENT_TIMEOUT_MS` | Latencia o caída del proveedor | Sí, vía `POST /investigate` | Mostrar `investigation_status: FAILED` manteniendo la `decision` persistida | **Ninguno** | Sí, nivel `error` |
| `PROVIDER_INVALID_RESPONSE` | — (idem) | Nebius devolvió un JSON que no cumple el esquema del informe | Error del modelo o cambio de formato no soportado | Sí, vía `POST /investigate` | Igual que arriba | **Ninguno** | Sí, nivel `error` |
| `EVIDENCE_REFERENCE_INVALID` | — (idem) | El informe referencia un `evidence_ref` o `rule_id` que no existe para esa ejecución | El modelo alucinó una referencia | Sí, vía `POST /investigate` | Igual que arriba; el hallazgo específico se descarta, no se inventa | **Ninguno** | Sí, nivel `error` |
| `AGENT_TOOL_LIMIT_EXCEEDED` | — (idem) | Se alcanzó `AGENT_MAX_TOOL_CALLS` sin completar el informe | Investigación compleja o bucle evitado por el límite | Sí, vía `POST /investigate` | Mostrar `investigation_status: PARTIAL` o `FAILED` según si hay informe parcial válido | **Ninguno** | Sí, nivel `warn` |
| `WORKER_LOST` | — (idem) | El intento quedó en `RUNNING` sin resolverse más allá del umbral de abandono y el *reaper* lo recuperó forzándolo a `FAILED` | Reinicio o caída del proceso worker mientras procesaba el intento | Sí, vía `POST /investigate` | Igual que los demás `FAILED`; informar que fue una recuperación automática, no un rechazo del informe | **Ninguno** | Sí, nivel `error` |
| `INTERNAL_ERROR` | 500 | Error no categorizado | Excepción no controlada | Depende del caso | Mostrar error genérico | Depende del punto donde ocurrió (ver `PERSISTENCE_FAILURE` si fue en persistencia) | Sí, nivel `error` |

## Notas de implementación

- Los códigos que no son respuesta HTTP directa (`PROVIDER_TIMEOUT`, `PROVIDER_INVALID_RESPONSE`, `EVIDENCE_REFERENCE_INVALID`, `AGENT_TOOL_LIMIT_EXCEEDED`) se exponen al cliente indirectamente a través de `investigation_status` y, si se necesita el detalle, mediante `investigation_attempts.error_code` mostrado en el centro de evidencias.
- Ningún código de esta tabla, por sí solo, puede convertir una ejecución `BLOCK` en `ALLOW`, ni viceversa. `decision` solo se escribe una vez, en el paso 5 de [database.md § 3](database.md#3-persistencia-segura).
- `details` en la respuesta de error nunca incluye el cuerpo crudo de la respuesta de Nebius, ni SQL, ni trazas de pila.

## Referencias

- Formato de error y dónde se usa cada código en cada endpoint: [api-contracts.md](api-contracts.md)
- Estados persistidos afectados: [database.md § 2](database.md#2-modelo-de-estados-unificado)
- Casos de prueba de cada error: [testing.md](testing.md)
