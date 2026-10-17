# Plan de pruebas y criterios de aceptación

## 0. Qué se ejecutó realmente y qué queda solo especificado

Este documento es, en su mayoría, una **especificación** de pruebas para que el equipo las implemente con Vitest/Playwright — no hay código de aplicación en este repositorio que las ejecute. La única excepción son las garantías a nivel de base de datos del mecanismo asíncrono y del *rate limiting*, que sí se **ejecutaron de verdad** contra un PostgreSQL 16 real en un contenedor Docker desechable, durante la tercera auditoría de esta especificación (ver la nota de verificación al inicio de [database.md](database.md) y el comentario en la cabecera de `supabase/migrations/001_initial_schema.sql`). Concretamente, se ejecutaron con datos reales y se confirmó el resultado indicado:

| Escenario | Resultado confirmado con datos reales |
|---|---|
| Restricción `UNIQUE (idempotency_key, scenario_id)` | Rechaza una clave duplicada |
| Creación atómica del primer intento junto con la decisión | Sin ventana `NOT_STARTED` huérfana |
| `fn_schedule_investigation_attempt` llamada dos veces para la misma ejecución | La segunda devuelve `IN_PROGRESS`, no crea un segundo intento |
| `fn_finalize_investigation_attempt` invocada con un `claimed_by` que no es el dueño del intento | `applied = false`, no modifica nada |
| `fn_finalize_investigation_attempt` invocada por el dueño correcto | Aplica, persiste el informe y propaga el estado a `executions` |
| `fn_finalize_investigation_attempt` invocada dos veces sobre el mismo intento ya finalizado | La segunda vez `applied = false`; no se duplica el informe en `agent_reports` |
| Reconciliador (`fn_schedule_investigation_attempt` sobre una ejecución `NOT_STARTED` simulada) | Recupera la ejecución y crea su primer intento sin duplicar |
| *Reaper* (`fn_reap_abandoned_attempts`) sobre un intento `RUNNING` abandonado | Lo marca `FAILED`/`WORKER_LOST` y propaga correctamente a `executions.investigation_status` |
| `fn_check_rate_limit` bajo 20 solicitudes realmente concurrentes contra un límite de 10 | Exactamente 10 permitidas, 10 rechazadas — sin evasión por condición de carrera |

Todo lo demás en este documento — motor de reglas, herramientas del agente, endpoints HTTP, frontend, RLS end-to-end, Basic Auth de la demo — está **especificado** aquí con el detalle suficiente para implementarse, pero no se ha ejecutado, porque requiere el código de aplicación que todavía no existe en este repositorio documental.

## Responsables de las pruebas

Esta tabla indica **quién mantiene cada tipo de prueba** con la distribución de cinco puestos. Es una asignación de responsabilidad, no evidencia de ejecución: salvo las pruebas marcadas como ejecutadas en la sección 0, todo lo especificado en este documento sigue siendo *planificado*. `flowguard-tests` solo contiene hoy un README y un `.gitignore`: **todavía no existe una suite funcional**.

| Tipo de prueba o trabajo | Responsable |
|---|---|
| Motor de reglas, persistencia e integración del backend | Puesto 1 (Dev 1) |
| Interfaz, formularios y flujos E2E de usuario | Puesto 2 (Dev 2) |
| Agente, proveedor, respuestas estructuradas y fallos de IA | Puesto 3 (Dev 3) |
| Contratos compartidos, fixtures y compatibilidad transversal | Puesto 5 (Dev 5) |
| Coordinación de la suite compartida `flowguard-tests` | Puesto 5 (Dev 5) |
| Automatización de la ejecución de las pruebas existentes (CI) | Puesto 4 (Dev 4) |
| Regresión completa entre módulos | Responsabilidad conjunta de los propietarios de los módulos; coordinación del puesto 5 |
| Integración y pruebas que atraviesan varios módulos | Responsabilidad compartida de los propietarios involucrados; coordinación del puesto 5 |
| Registro de defectos transversales | Puesto 5, con corrección por el propietario técnico del módulo afectado |

El puesto 5 coordina la suite transversal y mantiene los contratos y fixtures, pero **no** implementa todas las pruebas de todos los módulos: los propietarios técnicos siguen siendo responsables de las pruebas funcionales de los suyos. El puesto 4 no es responsable funcional de QA: automatiza la ejecución y comunica los fallos del pipeline, y el propietario corrige los defectos funcionales. Los hitos de los días 6, 9, 12 y 15 se verifican de forma cruzada (ver [definition-of-done.md](definition-of-done.md#criterios-por-hito-de-bloque)).

## 1. Pruebas unitarias

### Motor de reglas

- Importes por debajo, iguales y por encima del umbral.
- Categorías permitidas y no permitidas.
- Solicitante y aprobador idénticos.
- Factura verificada, ausente e indeterminada.
- Presupuesto suficiente, insuficiente e indeterminado.
- Reproducción de una misma decisión con la misma versión de política.

### Servicio de IA

- Respuesta válida.
- Respuesta JSON inválida.
- Referencia de evidencia inexistente.
- Exceso de llamadas a herramientas.
- Timeout.
- Error del proveedor.
- Informe parcial.
- Intento de alterar la decisión.

## 2. Pruebas de integración

- Crear una ejecución y persistirla.
- Verificar que se guardan las cinco evaluaciones.
- Verificar que el bloqueo se guarda antes de investigar.
- Recuperar evidencias desde herramientas.
- Persistir un informe validado.
- Rechazar informes inconsistentes.
- Reintentar una investigación sin modificar la decisión.
- Evitar duplicados por idempotencia.

### Idempotencia

Cada prueba indica el estado inicial relevante de `idempotency_keys`/`executions`, la operación ejecutada, el resultado esperado y la invariante que no debe romperse.

| Prueba | Estado inicial | Operación | Resultado esperado | Invariante que no debe romperse |
|---|---|---|---|---|
| Caso A — repetición idéntica | Ya existe una ejecución `PERSISTED` creada con `idempotency_key = K` y contenido `H` | `POST /executions` con `idempotency_key = K` y el mismo contenido (`request_hash = H`) | `200 OK` con la ejecución existente; no se evalúan las reglas de nuevo | No se crea una segunda fila en `executions`; `rule_evaluations` no cambia |
| Caso B — clave reutilizada con otro contenido | Igual que arriba | `POST /executions` con `idempotency_key = K` y contenido distinto (`request_hash ≠ H`) | `409 IDEMPOTENCY_KEY_REUSED` | La ejecución existente no se modifica; no se crea una ejecución nueva |
| Caso C — solicitudes concurrentes idénticas | Ninguna fila previa para `idempotency_key = K` | Dos `POST /executions` enviados en paralelo (promesas concurrentes en la prueba) con `idempotency_key = K` y contenido idéntico | Exactamente una recibe `201 Created`; la otra recibe `200 OK` con el mismo resultado o `409 IDEMPOTENCY_IN_PROGRESS` seguido de un reintento exitoso | Nunca existen dos filas en `executions` para la misma `idempotency_key`; la restricción `UNIQUE (idempotency_key, scenario_id)` rechaza la segunda inserción a nivel de base de datos, no solo a nivel de aplicación |
| Caso D — reintento tras fallo de persistencia | `idempotency_keys.execution_status = 'FAILED'` para `K` (simulado forzando un error en la transacción A) | `POST /executions` con `idempotency_key = K` y el mismo contenido | Se reevalúan las reglas desde cero; `201 Created` si tiene éxito esta vez | No se trata como Caso A (no hay ejecución previa que devolver); un `request_hash` distinto en este reintento sigue siendo Caso B, no libera la clave |
| Caso E — creación repetida tras investigación fallida | Ejecución `PERSISTED` existente con `investigation_status = FAILED` | `POST /executions` con la misma `idempotency_key` y contenido | `200 OK` con la ejecución existente, incluyendo su `investigation_status = FAILED` actual; **no** se crea un nuevo `investigation_attempts` | La creación nunca dispara una investigación nueva; solo `POST /investigate` puede hacerlo |
| Hash insensible al orden de propiedades | — | Calcular `request_hash` de dos payloads semánticamente idénticos pero con las propiedades JSON en distinto orden | Mismo `request_hash` en ambos casos | La función de normalización ordena las claves antes de serializar |
| Hash insensible a campos opcionales omitidos vs. explícitos | — | Calcular `request_hash` de un payload que omite un campo opcional y otro que lo envía explícitamente con su valor por defecto | Mismo `request_hash` en ambos casos | Los valores por defecto se aplican antes de hashear |

### Fallos de persistencia, del proveedor y del mecanismo asíncrono

| Prueba | Estado inicial | Operación | Resultado esperado | Invariante que no debe romperse |
|---|---|---|---|---|
| Fallo antes de persistir la decisión | Transacción A configurada para fallar (mock de error de base de datos) | `POST /executions` | `500 PERSISTENCE_FAILURE`; no se inicia ninguna llamada al servicio de IA | No queda fila en `executions`; `idempotency_keys.execution_status = 'FAILED'` |
| Timeout de Nebius | Intento `RUNNING`, mock de Nebius configurado para no responder dentro de `AGENT_TIMEOUT_MS` | El worker procesa el intento | `investigation_attempts.status = FAILED`, `error_code = PROVIDER_TIMEOUT` | `decision` no se toca; no se reintenta automáticamente |
| Respuesta JSON inválida del proveedor | Intento `RUNNING`, mock de Nebius responde un JSON que no valida contra el esquema | El worker procesa la respuesta | `investigation_attempts.status = FAILED`, `error_code = PROVIDER_INVALID_RESPONSE` | No se persiste ningún `agent_reports` con datos inventados |
| Referencia de evidencia inexistente | Intento `RUNNING`, mock de Nebius cita un `evidence_ref` que no existe para esa ejecución | El worker valida el informe | El hallazgo con la referencia inválida se descarta; si no queda nada verificable, `FAILED` con `error_code = EVIDENCE_REFERENCE_INVALID` | Ningún hallazgo sin evidencia real llega a mostrarse como válido |
| Informe parcial válido | Intento `RUNNING`, mock de Nebius responde hallazgos parciales pero verificables | El worker valida el informe | `investigation_attempts.status = PARTIAL`, fila en `agent_reports`, `limitations` no vacío | `decision` no se toca |
| **Reinicio o abandono de un trabajo de investigación** [✅ ejecutada] | Fila en `investigation_attempts` con `status = RUNNING` y `claimed_at` más antiguo que el umbral de abandono (simulando un worker caído a mitad de proceso) | Se ejecuta `fn_reap_abandoned_attempts` ([database.md § 5.3](database.md#53-el-reaper-recuperar-trabajos-abandonados)) | La fila pasa a `status = FAILED`, `error_code = WORKER_LOST`; `executions.investigation_status` se actualiza a `FAILED` en la misma llamada atómica | `decision` no se toca; la recuperación no requiere intervención manual; confirmado con datos reales (ver § 0) |
| **Decisión persistida, pero falla la creación del primer intento** | Simular que `fn_schedule_investigation_attempt`, llamada dentro de la misma transacción que persiste la ejecución, lanza una excepción (p. ej. forzando un error de restricción) | `POST /executions` | La transacción completa se revierte: **tampoco** queda la fila de `executions` — no es posible tener la decisión persistida sin su primer intento, por diseño ([database.md § 3](database.md#3-persistencia-segura)) | No existe el escenario "decisión persistida sin investigación programada" bajo operación normal; si aparece una fila así, solo puede venir de datos heredados de un esquema anterior |
| **El reconciliador recupera una ejecución `NOT_STARTED` sin crear duplicados** [✅ ejecutada] | Fila de `executions` con `investigation_status = 'NOT_STARTED'` insertada directamente (simulando datos heredados de un esquema anterior a esta auditoría) y `created_at` anterior al margen de gracia | Ciclo del reconciliador: `SELECT` de elegibles + `fn_schedule_investigation_attempt` ([database.md § 5.6](database.md#56-reconciliador-de-ejecuciones-not_started-defensa-en-profundidad)) | Se crea el intento 1, `investigation_status` pasa a `PENDING` | Ejecutar el reconciliador dos veces seguidas sobre la misma fila la segunda vez no encuentra nada elegible (ya no está `NOT_STARTED`); confirmado con datos reales (ver § 0) |
| **Dos procesos intentan programar simultáneamente la misma investigación** [✅ ejecutada como caso equivalente] | Ejecución con `investigation_status = NOT_STARTED` | Dos llamadas concurrentes a `fn_schedule_investigation_attempt` para la misma ejecución — sin importar si vienen de `POST /investigate`, de la transacción de creación o del reconciliador | Exactamente una recibe `CREATED`; la otra recibe `IN_PROGRESS` | Nunca hay dos intentos `PENDING`/`RUNNING` simultáneos para la misma ejecución; el caso de dos `POST /investigate` concurrentes se probó secuencialmente con datos reales (ver § 0), pendiente de repetir con concurrencia real como la prueba del rate limiter |
| **El worker se reinicia mientras hay trabajos `PENDING`** | Fila en `investigation_attempts` con `status = PENDING` creada antes de "reiniciar" el worker de prueba | Se reinicia el proceso worker (nueva instancia del loop de sondeo, nuevo `claimed_by`) | El trabajo `PENDING` se reclama y procesa con normalidad en el primer ciclo tras el reinicio | Ningún trabajo `PENDING` queda huérfano indefinidamente; no depende de memoria del proceso anterior |
| **Un intento antiguo que termina después de un reintento** [✅ ejecutada] | Intento 1 marcado `FAILED` por el *reaper*; intento 2 creado y en `RUNNING` | Un worker con el `claimed_by` del intento 1 (que seguía procesando en segundo plano) llama a `fn_finalize_investigation_attempt` para el intento 1 | `applied = false` (el intento 1 ya no está `RUNNING`); se descarta y se registra en logs | `executions.investigation_status` sigue reflejando el intento 2, nunca se sobrescribe con el resultado tardío del intento 1; confirmado con datos reales (ver § 0) |
| **La persistencia del informe falla durante la finalización** | Intento `RUNNING`, `claimed_by` correcto, pero `report_json` viola una restricción de `agent_reports` (p. ej. `model` nulo cuando la columna lo exige) | `fn_finalize_investigation_attempt(..., 'COMPLETED', ...)` | La función entera revierte: el intento permanece `RUNNING` (el `UPDATE` del paso 1 también se revierte), no queda fila en `agent_reports` | No existe un estado donde el intento diga `COMPLETED` sin que exista su informe — la atomicidad de la función lo impide por construcción |
| **La actualización del estado de la ejecución falla durante la finalización** | Intento `RUNNING` cuyo `attempt_number` ya no es el más reciente de `executions.latest_attempt_number` (otro intento más nuevo se creó mientras tanto) | `fn_finalize_investigation_attempt(..., 'COMPLETED', ...)` | `applied = true`, `propagated = false`: el intento se finaliza y su informe se persiste igual (para el historial), pero `executions.investigation_status` no se toca | El intento superado nunca pisa el estado vigente; ver el mismo mecanismo probado en la fila anterior |
| **Dos solicitudes concurrentes de finalización no generan informes duplicados** [✅ ejecutada] | Intento ya finalizado (`COMPLETED`, con su fila en `agent_reports`) | Una segunda llamada a `fn_finalize_investigation_attempt` para el mismo `attempt_id` | `applied = false`; `agent_reports` sigue con exactamente una fila para ese intento | La condición `status = 'RUNNING'` del paso 1 hace que la segunda llamada no reclame nada que reclamar; confirmado con datos reales (ver § 0) |
| Reintento manual sin modificar la decisión | Ejecución `BLOCK` con `investigation_status = FAILED` | `POST /investigate` | Nuevo `investigation_attempts` con `attempt_number` incrementado | `decision = BLOCK` y los cinco `rule_evaluations` originales permanecen idénticos byte a byte |
| Investigación ya completada | `investigation_status = COMPLETED` | `POST /investigate` | `409 INVESTIGATION_ALREADY_COMPLETED`; no se crea un intento nuevo | El informe `COMPLETED` existente no se modifica |

### Rate limiting

| Prueba | Estado inicial | Operación | Resultado esperado | Invariante que no debe romperse |
|---|---|---|---|---|
| **El límite funciona y no se evade con concurrencia** [✅ ejecutada] | `rate_limit_counters` vacía para la clave de prueba | 20 llamadas realmente concurrentes a `fn_check_rate_limit(key, 60, 10)` (procesos separados, no la misma conexión) | Exactamente 10 `allowed = true`, exactamente 10 `allowed = false` | Confirmado con datos reales: 20 llamadas concurrentes produjeron exactamente ese resultado (ver § 0 y la nota de verificación en [database.md](database.md)) |
| **Un error del sistema de rate limiting se comporta según la política definida** | `rate_limit_counters`/la conexión a PostgreSQL no disponible durante la verificación | `POST /api/v1/executions` | La solicitud se permite (*fail-open*) y se registra un `warn` en logs, según la política explícita de [security.md § 2.4](security.md#24-límite-de-solicitudes-rate-limiting) | Un fallo del almacén de límites nunca bloquea tráfico legítimo de la demo (decisión documentada, no un descuido) |
| **El rate limiting se mantiene efectivo con varias instancias** | Dos procesos backend simulados (dos conexiones distintas a la misma base), misma `rate_key` | Cada instancia procesa la mitad de 20 solicitudes concurrentes contra un límite de 10 | El total permitido across ambas instancias sigue siendo exactamente 10, no 20 | El contador vive en PostgreSQL, no en memoria de cada instancia — ya verificado en la prueba de concurrencia anterior, que es equivalente a "varias instancias" porque cada conexión psql es independiente |

### Transiciones de estado

- Verificar cada transición permitida de `idempotency_keys.execution_status` (`RECEIVED → VALIDATING → EVALUATING → PERSISTED`, la salida a `FAILED` desde cualquiera de los tres primeros, y `FAILED → RECEIVED` al reintentar con el mismo contenido).
- Verificar que una transición prohibida (p. ej. de `PERSISTED` a cualquier otro valor, o saltar directo a `PERSISTED` sin `EVALUATING`) es rechazada por el código del backend, no solo por la restricción `CHECK` de la base de datos.
- Verificar cada transición permitida de `investigation_status` (`NOT_STARTED → PENDING → RUNNING → {COMPLETED | PARTIAL | FAILED}`, el reingreso a `PENDING` desde `FAILED`/`PARTIAL`, y `RUNNING → FAILED` forzado por el *reaper*).
- Verificar que no existe ningún camino de código que escriba `executions.decision` más de una vez para la misma ejecución.
- **Fallos de actualización de estados:** simular que la actualización condicional de `executions.investigation_status` (la que lleva `AND latest_attempt_number = $attempt_number`, [database.md § 5.4](database.md#54-fencing-finalización-atómica-de-un-intento)) afecta 0 filas porque ya fue superada por un intento más nuevo: el worker debe continuar sin lanzar un error no controlado, y el resultado del intento superado debe quedar solo en `investigation_attempts`, nunca propagarse.

### Evidencia, autorización y RLS

- Validación de referencias de evidencia: un informe que cite un `evidence_ref` o `rule_id` inexistente para esa ejecución debe ser rechazado (ver [security.md § 1.7](security.md#17-validación-de-informes)) con `error_code = EVIDENCE_REFERENCE_INVALID`.
- Autorización y permisos: un intento de enviar `approval_status`, roles o permisos de aprobador directamente en el payload de `POST /executions` debe ser ignorado por el backend; la regla `PAY-001`/`PAY-003` debe evaluarse solo contra el contexto confiable.
- RLS: con la clave `anon`, un intento de `SELECT`/`INSERT` directo contra cualquiera de las once tablas (incluida `rate_limit_counters`) debe ser denegado por Supabase (ver [`supabase/migrations/001_initial_schema.sql`](https://github.com/FlowGuard-Hackathon/flowguard-supabase/blob/main/migrations/001_initial_schema.sql)); solo el rol de servicio usado por el backend debe poder leer/escribir. Las funciones PL/pgSQL (§ 5 de [database.md](database.md)) se ejecutan con los permisos de quien las invoca por defecto (`SECURITY INVOKER`, el comportamiento estándar de PostgreSQL): el backend debe invocarlas con el rol de servicio, igual que cualquier otra operación — esta es una verificación pendiente de confirmar específicamente contra un proyecto Supabase real (ver riesgo abierto en [implementation-checklist.md](implementation-checklist.md)).

### Conversión monetaria

- `amount` con más de 2 decimales debe rechazarse con `VALIDATION_ERROR`, no redondearse silenciosamente.
- `amount` negativo, cero, `NaN` o fuera del rango admitido (`0.01`–`10,000,000.00` MXN) debe rechazarse.
- La conversión de `amount` (pesos) a `amount_minor` (centavos enteros) debe producir el mismo resultado sin importar el módulo que la invoque, verificado contra la única función centralizada (`toMinorUnits`).

### Recuperación del historial

- `GET /api/v1/executions` con cada combinación de filtros (`decision`, `investigation_status`, rango de fechas) debe devolver solo las filas que corresponden. No existe un filtro `execution_status` (ver [api-contracts.md § 4](api-contracts.md#4-get-apiv1executions)).
- `GET /api/v1/executions/:id` debe reflejar el `investigation_status` más reciente y el conteo correcto de `attempts` después de varios reintentos.

### Acceso no autorizado a la demo pública

- Con `DEMO_BASIC_AUTH_USER`/`DEMO_BASIC_AUTH_PASSWORD` configurados (ver [security.md § 2.0](security.md#20-estrategia-de-seguridad-por-entorno)), una solicitud a cualquier ruta (incluidas `/api/v1/rules` y `/api/v1/health`) sin credenciales válidas debe ser rechazada por el middleware antes de llegar al route handler.
- Verificar que las credenciales de Basic Auth nunca aparecen en el HTML ni en los bundles de JavaScript servidos al navegador (inspección del build de producción).
- Verificar que `429 RATE_LIMITED` se activa ante un volumen de solicitudes anómalo durante la ventana de la demo, incluso con credenciales válidas.
- Responsables: Dev 1 (middleware y credenciales) y Dev 2 (verifica el acceso desde la interfaz); el funcionamiento del acceso se comprueba durante el ensayo de la demo. Las credenciales nunca se almacenan en Git ni se imprimen en logs.

### Coherencia entre base de datos y contratos

- Cada campo de las respuestas JSON documentadas en [api-contracts.md](api-contracts.md) tiene una columna o valor derivado correspondiente en [database.md](database.md); una prueba de contrato recorre ambos documentos y falla si encuentra un campo sin origen claro (p. ej. ejecutada como parte de la regresión de documentación, no como prueba de runtime).
- El esquema Zod de cada contrato valida contra al menos un ejemplo generado a partir de una fila real de cada tabla relevante.

## 3. Pruebas end-to-end

1. Pago de $8,500: resultado `ALLOW`.
2. Pago de $15,000 sin aprobación: resultado `BLOCK`.
3. Pago con categoría no permitida: resultado `BLOCK`.
4. Pago con factura no verificada: resultado `BLOCK`.
5. Pago con presupuesto insuficiente: resultado `BLOCK`.
6. Nebius no disponible: `decision` y `execution_status = PERSISTED` se mantienen; `investigation_status` queda en `FAILED`.
7. Informe parcial: limitaciones visibles en la interfaz, `investigation_status = PARTIAL`.
8. Repetición de una solicitud con la misma `idempotency_key`: no genera duplicados no deseados; el usuario ve la misma ejecución.
9. Reintento manual de una investigación fallida desde el centro de evidencias: nuevo intento visible en el historial, `decision` sin cambios.

## 4. Criterios de salida

- Las cinco reglas se evalúan de forma determinista.
- Todas las decisiones tienen versión de política.
- Los resultados quedan persistidos.
- El agente no dispone de herramientas de escritura.
- Las referencias de evidencia se validan en el backend.
- La caída de Nebius no cambia la decisión.
- El frontend muestra resultados del backend.
- Los escenarios principales pasan las pruebas end-to-end.
- El repositorio se puede instalar siguiendo el README.
- La demo puede repetirse con datos conocidos.

## Definition of Done

Una tarea no se considera terminada únicamente porque su código compile. Debe cumplir los criterios aplicables:

- Implementación funcional.
- Contrato respetado.
- Pruebas aprobadas.
- Errores tratados.
- Sin secretos en el repositorio.
- Documentación actualizada cuando corresponda.
- Pull request revisado.
- Integración compatible con la rama principal.

## Referencias

- Las cinco reglas evaluadas: [rules.md](rules.md)
- Contratos usados en fixtures de prueba: [api-contracts.md](api-contracts.md)
- Comportamiento esperado ante fallos de Nebius/DB: [architecture.md](architecture.md#5-resiliencia-escalabilidad-y-aislamiento-de-fallos)
- Mecanismo asíncrono, *fencing* y recuperación (base de las pruebas de concurrencia): [database.md § 5](database.md#5-concurrencia-y-recuperación-del-mecanismo-asíncrono-de-investigación)
- Catálogo de errores y códigos citados en esta tabla: [error-catalog.md](error-catalog.md)
