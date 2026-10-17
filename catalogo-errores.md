# Catálogo de errores

Todos los errores de la API siguen el formato uniforme definido en [contratos-api.md § 1](contratos-api.md#1-formato-uniforme-de-error). Esta tabla es la lista cerrada de códigos admitidos para el MVP; un código nuevo debe agregarse aquí antes de usarse en el backend.

**Distinción importante:** un *error técnico* (esta tabla) es distinto de una *infracción de negocio*. Una regla que falla (`PAG-00X: NO_CUMPLE`) no es un error: es un resultado válido del motor, representado con HTTP `200`/`201` y `decision: "BLOQUEAR"`. Los códigos de esta tabla representan fallos del sistema (validación, infraestructura, proveedor externo), no decisiones de negocio.

| Código | HTTP | Descripción | Causa probable | ¿Reintentable? | Comportamiento esperado del frontend | Efecto sobre `decision` | ¿Se registra en logs? |
|---|---|---|---|---|---|---|---|
| `ERROR_VALIDACION` | 400 | El payload no cumple el esquema o las reglas de formato (tipo, rango, campos obligatorios) | Cliente envía datos malformados | No (hay que corregir el payload) | Mostrar el campo inválido tomado de `detalles.campo` | No se evalúa; no existe decisión | Sí, nivel `warn` |
| `ERROR_VALIDACION_SEMANTICA` | 422 | El payload es sintácticamente válido pero semánticamente inconsistente (p. ej. moneda no admitida) | Cliente envía una combinación de valores no soportada | No | Mostrar mensaje específico | No se evalúa; no existe decisión | Sí, nivel `warn` |
| `PRESUPUESTO_NO_ENCONTRADO` | 404 | El `presupuesto_id` referenciado no existe en el contexto confiable | Fixture/demo mal configurada o `presupuesto_id` inválido | No | Pedir al usuario elegir un presupuesto válido | No se evalúa | Sí, nivel `warn` |
| `ADJUNTO_NO_ENCONTRADO` | 404 | La referencia de adjunto no existe en el contexto confiable | Fixture/demo mal configurada | No | Pedir una referencia de factura válida | No se evalúa | Sí, nivel `warn` |
| `EJECUCION_NO_ENCONTRADA` | 404 | El `ejecucion_id` no existe | Enlace o id incorrecto | No | Mostrar "ejecución no encontrada" | N/A | No necesariamente |
| `NO_AUTENTICADO` | 401 | Falta identidad válida (cuando el entorno la exige, ver [seguridad.md](seguridad.md)) | Sesión ausente o expirada | Sí, tras reautenticar | Redirigir a login | No se evalúa | Sí, nivel `warn` |
| `PROHIBIDO` | 403 | La identidad no tiene permiso para la operación | Rol insuficiente | No | Mostrar "sin permisos" | No se evalúa | Sí, nivel `warn` |
| `CLAVE_IDEMPOTENCIA_REUTILIZADA` | 409 | La misma `clave_idempotencia` se usó con un payload de contenido distinto (`hash_solicitud` no coincide) — se verifica **antes** que el estado de progreso, así que tiene precedencia sobre `IDEMPOTENCIA_EN_PROGRESO` | Error del cliente o reintento incorrecto | No (requiere una clave nueva) | Mostrar conflicto; sugerir generar una nueva clave | No se evalúa en esta llamada | Sí, nivel `warn` |
| `IDEMPOTENCIA_EN_PROGRESO` | 409 | Ya existe una solicitud con la misma `clave_idempotencia` y el mismo contenido todavía procesándose (`estado_ejecucion` en `RECIBIDA`/`VALIDANDO`/`EVALUANDO`) | Doble clic o reintento muy rápido del cliente | Sí — **reintentar la misma solicitud `POST /ejecuciones`** (idéntica) con backoff; no existe un endpoint separado de consulta previo a que exista `ejecucion_id` | Reintentar automáticamente con backoff; no mostrar error al usuario en el primer reintento | Pendiente (el reintento eventualmente recibe `200`/`201` o un error técnico) | Sí, nivel `info` |
| `INVESTIGACION_EN_CURSO` | 409 | Ya hay un intento de investigación `PENDIENTE`/`EN_CURSO` | Doble llamada a `POST /investigar` | Sí, consultando el estado actual | No disparar un nuevo intento; mostrar el actual | Sin efecto (la decisión ya estaba persistida) | Sí, nivel `info` |
| `INVESTIGACION_YA_COMPLETADA` | 409 | Se pidió investigar una ejecución con `estado_investigacion = COMPLETADA` | Cliente reintenta innecesariamente | No | Mostrar el informe ya existente | Sin efecto | Sí, nivel `info` |
| `LIMITE_SOLICITUDES_EXCEDIDO` | 429 | Se superó el límite de solicitudes para el `clave_limite` y grupo de endpoint correspondientes (ver límites concretos y cálculo de `clave_limite` en [seguridad.md § 2.4](seguridad.md#24-límite-de-solicitudes-rate-limiting)) | Abuso o pico de tráfico | Sí, respetando la cabecera `Retry-After` devuelta | Mostrar "demasiadas solicitudes, intenta de nuevo en N s" usando `Retry-After`/`detalles.ventana_segundos` | No se evalúa | Sí, nivel `warn` |
| `FALLO_PERSISTENCIA` | 500 | La transacción de persistencia de la ejecución falló | Base de datos no disponible o error inesperado | Sí, reenviando con la misma `clave_idempotencia` | Mostrar error genérico; permitir reintentar | **No crea decisión**; nunca se infiere `PERMITIR` | Sí, nivel `error` |
| `DEPENDENCIA_NO_DISPONIBLE` | 503 | Una dependencia necesaria (base de datos u otra) no está disponible | Caída de infraestructura | Sí, con backoff | Mostrar estado de servicio degradado | No se evalúa | Sí, nivel `error` |
| `TIEMPO_AGOTADO_PROVEEDOR` | — (no es respuesta HTTP al cliente; se registra en `intentos_investigacion.codigo_error`) | Nebius no respondió dentro de `AGENTE_TIEMPO_LIMITE_MS` | Latencia o caída del proveedor | Sí, vía `POST /investigar` | Mostrar `estado_investigacion: FALLIDA` manteniendo la `decision` persistida | **Ninguno** | Sí, nivel `error` |
| `RESPUESTA_INVALIDA_PROVEEDOR` | — (idem) | Nebius devolvió un JSON que no cumple el esquema del informe | Error del modelo o cambio de formato no soportado | Sí, vía `POST /investigar` | Igual que arriba | **Ninguno** | Sí, nivel `error` |
| `REFERENCIA_EVIDENCIA_INVALIDA` | — (idem) | El informe referencia un `ref_evidencia` o `regla_id` que no existe para esa ejecución | El modelo alucinó una referencia | Sí, vía `POST /investigar` | Igual que arriba; el hallazgo específico se descarta, no se inventa | **Ninguno** | Sí, nivel `error` |
| `LIMITE_HERRAMIENTAS_AGENTE_EXCEDIDO` | — (idem) | Se alcanzó `AGENTE_MAX_LLAMADAS_HERRAMIENTAS` sin completar el informe | Investigación compleja o bucle evitado por el límite | Sí, vía `POST /investigar` | Mostrar `estado_investigacion: PARCIAL` o `FALLIDA` según si hay informe parcial válido | **Ninguno** | Sí, nivel `warn` |
| `PROCESO_PERDIDO` | — (idem) | El intento quedó en `EN_CURSO` sin resolverse más allá del umbral de abandono y el *reaper* lo recuperó forzándolo a `FALLIDA` | Reinicio o caída del proceso worker mientras procesaba el intento | Sí, vía `POST /investigar` | Igual que los demás `FALLIDA`; informar que fue una recuperación automática, no un rechazo del informe | **Ninguno** | Sí, nivel `error` |
| `ERROR_INTERNO` | 500 | Error no categorizado | Excepción no controlada | Depende del caso | Mostrar error genérico | Depende del punto donde ocurrió (ver `FALLO_PERSISTENCIA` si fue en persistencia) | Sí, nivel `error` |

## Notas de implementación

- Los códigos que no son respuesta HTTP directa (`TIEMPO_AGOTADO_PROVEEDOR`, `RESPUESTA_INVALIDA_PROVEEDOR`, `REFERENCIA_EVIDENCIA_INVALIDA`, `LIMITE_HERRAMIENTAS_AGENTE_EXCEDIDO`) se exponen al cliente indirectamente a través de `estado_investigacion` y, si se necesita el detalle, mediante `intentos_investigacion.codigo_error` mostrado en el centro de evidencias.
- Ningún código de esta tabla, por sí solo, puede convertir una ejecución `BLOQUEAR` en `PERMITIR`, ni viceversa. `decision` solo se escribe una vez, en el paso 5 de [base-datos.md § 3](base-datos.md#3-persistencia-segura).
- `detalles` en la respuesta de error nunca incluye el cuerpo crudo de la respuesta de Nebius, ni SQL, ni trazas de pila.

## Referencias

- Formato de error y dónde se usa cada código en cada endpoint: [contratos-api.md](contratos-api.md)
- Estados persistidos afectados: [base-datos.md § 2](base-datos.md#2-modelo-de-estados-unificado)
- Casos de prueba de cada error: [pruebas.md](pruebas.md)
