# Contratos de datos y API

Los contratos JSON son la interfaz oficial entre los módulos. Los ejemplos de este documento son fixtures de demostración; los valores reales deben proceder del backend.

> Este documento asume el modelo de estados unificado descrito en [database.md § 2](database.md#2-modelo-de-estados-unificado). Si hay alguna discrepancia entre ambos documentos, **database.md es la fuente de verdad** para nombres y transiciones de estado, porque describe lo que efectivamente se persiste.

## 0. Modelo de estados (resumen)

FlowGuard distingue tres conceptos independientes que nunca deben colapsarse en un solo campo:

| Campo | Significa | Quién lo cambia |
|---|---|---|
| `decision` | Resultado de negocio de las reglas: `ALLOW` o `BLOCK` | Exclusivamente el motor de reglas, una sola vez por ejecución |
| `execution_status` | Progreso técnico del pipeline de evaluación/persistencia | Exclusivamente el backend (orquestador de ejecuciones) |
| `investigation_status` | Progreso técnico del trabajo de IA | Exclusivamente el backend (orquestador de investigación y worker asíncrono), nunca el modelo directamente |

> **Nota sobre `execution_status` en las respuestas de este documento:** toda ejecución que la API puede devolver ya está persistida (ver [database.md § 1.6](database.md#16-executions): una fila de `executions` solo existe una vez confirmada la transacción). Por eso `execution_status` aparece siempre como el valor constante `"PERSISTED"` en cualquier respuesta exitosa — no es una columna que cambie en la base, es un campo calculado al servir la respuesta. El resto del ciclo de vida (`RECEIVED`/`VALIDATING`/`EVALUATING`/`FAILED`) ocurre antes de que exista una ejecución que consultar, y solo es observable en el momento mismo de la solicitud fallida, vía el código de error correspondiente (ver [error-catalog.md](error-catalog.md)), nunca a través de un `GET` posterior.

Valores cerrados:

- `decision`: `ALLOW`, `BLOCK`
- `execution_status`: `RECEIVED`, `VALIDATING`, `EVALUATING`, `PERSISTED`, `FAILED`
- `investigation_status`: `NOT_STARTED`, `PENDING`, `RUNNING`, `COMPLETED`, `PARTIAL`, `FAILED`

`COMPLETED` nunca se usa para `execution_status` ni para `decision`; solo existe como valor de `investigation_status`. `ALLOW`/`BLOCK` nunca se usan como estados técnicos. El detalle completo de transiciones está en [database.md § 2](database.md#2-modelo-de-estados-unificado).

## 1. Formato uniforme de error

Toda respuesta de error (4xx/5xx) sigue este esquema, sin excepción:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "El campo payment.amount debe ser positivo y finito.",
    "request_id": "req_01HZXYZ...",
    "details": {
      "field": "payment.amount"
    }
  }
}
```

- `code`: identificador estable en mayúsculas, ver el catálogo completo en [error-catalog.md](error-catalog.md).
- `message`: texto apto para mostrar o registrar, sin datos sensibles.
- `request_id`: identificador de correlación generado por el backend en cada solicitud; debe incluirse en logs.
- `details`: objeto opcional, seguro de exponer. Nunca incluye stack traces, SQL, secretos ni respuestas crudas de Nebius.

Un error técnico posterior a la persistencia de la decisión **nunca** cambia `decision`. Ver [error-catalog.md](error-catalog.md) para el efecto de cada código sobre `decision`.

## 2. Conversión e idempotencia (reglas transversales)

### 2.1. Importes

- El contrato público (`payment.amount`) se expresa en **pesos mexicanos (MXN)**, con hasta 2 decimales.
- PostgreSQL almacena los importes como enteros en centavos (`amount_minor`, tipo `BIGINT`), nunca como `float`/`numeric` de punto flotante binario.
- La conversión se realiza mediante una única función centralizada, por ejemplo `toMinorUnits(amountMXN: number): bigint` en `src/shared/money.ts`. Ningún otro módulo implementa su propia conversión.
- Política: se **rechaza** (no se redondea silenciosamente) cualquier importe con más de 2 decimales. No se admiten importes `NaN`, `Infinity`, negativos o iguales a cero.
- Límite práctico para el MVP: `0.01` a `10,000,000.00` MXN. Un importe fuera de este rango se rechaza con `VALIDATION_ERROR`.

### 2.2. Idempotencia

`event.idempotency_key` es obligatoria en `POST /api/v1/executions`. **Ámbito de unicidad:** `(idempotency_key, scenario_id)`, único en la tabla `idempotency_keys` (ver [database.md § 1.5](database.md#15-idempotency_keys)). Esto permite reutilizar la misma clave en distintos escenarios de demo sin colisionar.

#### Hash del contenido (`request_hash`)

El backend calcula un hash determinista del contenido semántico del request para distinguir una repetición legítima de una reutilización indebida de la clave:

1. **Campos incluidos:** todo el cuerpo del request **excepto** `event.idempotency_key` (incluye `schema_version`, `scenario_id`, `event.type`, y todo el objeto `payment` y `context`). `idempotency_key` no participa en el hash porque es la clave de búsqueda misma; incluirla lo haría circular.
2. **Normalización antes de calcular el hash:**
   - Se aplican primero los valores por defecto de los campos opcionales (de modo que omitir un campo opcional produzca el mismo resultado que enviarlo explícitamente con su valor por defecto).
   - Los importes se normalizan a su representación canónica en centavos enteros (`amount_minor`, ver § 2.1) antes de hashear, no al texto decimal recibido — así `"amount": 15000` y, si se permitiera, `"amount": 15000.00` producen el mismo hash.
   - Las claves de cada objeto se ordenan alfabéticamente de forma recursiva antes de serializar, para que el orden de las propiedades en el JSON recibido no afecte el resultado.
   - El resultado se serializa con `JSON.stringify` sobre la estructura ya canonicalizada y se calcula `SHA-256` en hexadecimal.
3. La función de normalización y hash es única y centralizada (junto a `toMinorUnits`, en `src/shared/money.ts` o un módulo hermano), igual que la conversión monetaria: ningún otro módulo reimplementa esta lógica.

#### Casos cubiertos

**Caso A — misma clave, mismo contenido (mismo `request_hash`):**
La API no vuelve a evaluar las reglas. Devuelve la ejecución ya existente. Si la fila ya existía antes de esta solicitud → `200 OK`. Si esta misma solicitud es la que efectivamente ganó la carrera y la creó → `201 Created`. El cuerpo de la respuesta es idéntico en ambos casos; solo cambia el código HTTP, según si esta llamada fue la creadora o no.

**Caso B — misma clave, contenido diferente (`request_hash` distinto):**
`409 Conflict`, código `IDEMPOTENCY_KEY_REUSED`. Se verifica el hash **antes** que cualquier otra condición (incluida si el intento original sigue en curso): una clave reutilizada con otro contenido siempre es `409 IDEMPOTENCY_KEY_REUSED`, nunca `IDEMPOTENCY_IN_PROGRESS`. No se crea una ejecución nueva ni se modifica la existente.

**Caso C — solicitud concurrente con la misma clave y mismo contenido, todavía procesándose** (`idempotency_keys.execution_status` en `RECEIVED`/`VALIDATING`/`EVALUATING`): `409 Conflict`, código `IDEMPOTENCY_IN_PROGRESS`. **Política de cliente definida:** reintentar la misma solicitud `POST /executions` (idéntica, con la misma clave y el mismo contenido) con *backoff*; no existe un endpoint separado para consultar el estado de una clave de idempotencia antes de que exista un `execution_id`. Cada reintento vuelve a evaluar los tres casos (A/B/C) contra el estado más reciente: cuando la solicitud original termine, el reintento recibirá `200` con el resultado (Caso A) o, si terminó en error técnico, podrá disparar una nueva creación (Caso D).

**Caso D — la primera solicitud falló antes de persistir** (`idempotency_keys.execution_status = 'FAILED'`): una nueva solicitud con la misma clave y el mismo `request_hash` **reinicia el ciclo** (`FAILED → RECEIVED`, ver [database.md § 2.2](database.md#22-execution_status)) y evalúa las reglas de nuevo como si fuera la primera vez; no se trata como Caso A porque no hay ninguna ejecución previa que devolver. Responde `201 Created` si tiene éxito esta vez, o vuelve a `500 PERSISTENCE_FAILURE`/queda en `FAILED` si falla de nuevo. Una nueva solicitud con la misma clave pero `request_hash` **distinto** del que falló sigue siendo Caso B (`409 IDEMPOTENCY_KEY_REUSED`): un fallo previo no "libera" la clave para usarse con otro contenido.

**Caso E — la investigación de una ejecución ya persistida falló; se repite la creación:** si el cliente reenvía `POST /executions` con la misma clave y el mismo contenido de una ejecución cuya investigación está en `FAILED`/`PARTIAL`, el Caso A aplica normalmente (devuelve la ejecución existente, con su `investigation_status` actual). **La creación nunca vuelve a programar ni a duplicar la investigación**: el primer y único intento automático ocurre una vez, inmediatamente después de la persistencia original (transacción B de [database.md § 3](database.md#3-persistencia-segura)). Reintentar la investigación requiere una llamada explícita a `POST /executions/:id/investigate` (§ 7); `POST /executions` nunca la dispara de nuevo.

#### Restricción de base de datos

`UNIQUE (idempotency_key, scenario_id)` en `idempotency_keys`, con el registro insertado (o reactivado desde `FAILED`, Caso D) dentro de la misma transacción que reserva el intento, antes de evaluar las reglas. Esto evita duplicados bajo solicitudes concurrentes: la segunda transacción que intenta insertar la misma clave falla por violación de unicidad (o, en el caso D, compite por el mismo `UPDATE` condicional) y el backend la traduce al caso correspondiente (B o C según el hash).

## 3. `POST /api/v1/executions`

**Objetivo:** crear y evaluar una ejecución de forma síncrona respecto a la decisión (las reglas se evalúan y persisten antes de responder); la investigación de IA se programa después de confirmar la persistencia y puede completarse de forma asíncrona.

**Autenticación:** requerida cuando el endpoint se expone fuera de un entorno de demo controlado. Ver [security.md § 2](security.md#2-estrategia-de-seguridad-por-entorno).

**Parámetros de ruta/query:** ninguno.

### Request

```json
{
  "schema_version": "1.0",
  "scenario_id": "payment-approval-v1",
  "event": {
    "type": "PAYMENT_REQUESTED",
    "idempotency_key": "demo-payment-001"
  },
  "payment": {
    "payment_id": "PAY-2026-001",
    "amount": 15000,
    "currency": "MXN",
    "category": "OPERATIONS",
    "description": "Compra de equipo de oficina",
    "requested_by": {
      "user_id": "USR-001"
    },
    "attachments": [
      {
        "attachment_id": "ATT-001",
        "type": "INVOICE",
        "reference": "INV-2026-001"
      }
    ]
  },
  "context": {
    "budget_id": "BUD-OPERATIONS-01"
  }
}
```

### Validación del backend

- `amount`: positivo, finito, máximo 2 decimales, dentro del rango admitido (ver § 2.1).
- `currency`: pertenece al conjunto admitido (`MXN` para el MVP).
- `category`: valor reconocido por la política activa (ver [rules.md](rules.md)).
- `requested_by.user_id`: corresponde a un usuario existente en el contexto confiable.
- `attachments[].attachment_id` / `reference`: deben existir en el contexto confiable; el backend los resuelve, no confía en el `type` enviado por el cliente.
- `context.budget_id`: debe existir y estar asociado a la categoría correspondiente.
- El cliente **no** puede establecer `approval_status`, roles, permisos de aprobador ni el resultado de verificación de factura: esos datos se resuelven exclusivamente contra el contexto confiable del backend, no contra el payload.
- `event.idempotency_key`: ver § 2.2.

### Response — éxito (`201 Created`, o `200 OK` si la idempotencia devuelve una ejecución existente)

```json
{
  "schema_version": "1.0",
  "execution_id": "EXE-2026-0001",
  "engine_version": "1.0.0",
  "policy_version": "payment-policy-v1",
  "decision": "BLOCK",
  "execution_status": "PERSISTED",
  "investigation_status": "PENDING",
  "summary": {
    "total_rules": 5,
    "passed": 4,
    "failed": 1,
    "indeterminate": 0
  },
  "rules": [
    {
      "rule_id": "PAY-001",
      "status": "FAIL",
      "observed": { "amount_mxn": 15000, "approval_status": "NOT_APPROVED" },
      "expected": { "approval_required": true, "approval_status": "APPROVED" },
      "evidence_refs": ["EV-0001", "EV-0002"]
    },
    {
      "rule_id": "PAY-002",
      "status": "PASS",
      "observed": { "category": "OPERATIONS" },
      "expected": { "allowed_categories": ["OPERATIONS", "SOFTWARE", "TRAVEL"] },
      "evidence_refs": ["EV-0003"]
    },
    {
      "rule_id": "PAY-003",
      "status": "PASS",
      "observed": { "requester_id": "USR-001", "approver_id": null },
      "expected": { "approver_required_for_amount": false },
      "evidence_refs": []
    },
    {
      "rule_id": "PAY-004",
      "status": "PASS",
      "observed": { "invoice_verified": true },
      "expected": { "invoice_verified": true },
      "evidence_refs": ["EV-0004"]
    },
    {
      "rule_id": "PAY-005",
      "status": "PASS",
      "observed": { "amount_minor": 1500000, "available_budget_minor": 2500000 },
      "expected": { "amount_minor_lte_available": true },
      "evidence_refs": ["EV-0005"]
    }
  ],
  "execution_controls": {
    "may_advance": false,
    "requires_investigation": true
  }
}
```

> El array `rules` siempre contiene el resultado de las **cinco** reglas, en el orden `PAY-001`…`PAY-005`, no solo las que fallan.

### Errores relevantes

`400` `VALIDATION_ERROR` · `404` `BUDGET_NOT_FOUND` / `ATTACHMENT_NOT_FOUND` · `409` `IDEMPOTENCY_KEY_REUSED` / `IDEMPOTENCY_IN_PROGRESS` · `422` `SEMANTIC_VALIDATION_ERROR` · `429` `RATE_LIMITED` · `500` `PERSISTENCE_FAILURE` / `INTERNAL_ERROR` · `503` `DEPENDENCY_UNAVAILABLE`.

Si la persistencia de la decisión falla (paso 4 de [database.md § 3](database.md#3-persistencia-segura)), la API responde `500 PERSISTENCE_FAILURE` y **no** crea una ejecución ni sugiere que el pago avanzó.

## 4. `GET /api/v1/executions`

**Objetivo:** listar el historial de ejecuciones con filtros y paginación.

**Query params:**

| Parámetro | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `decision` | `ALLOW` \| `BLOCK` | No | Filtra por decisión |
| `investigation_status` | enum | No | Filtra por estado de investigación |
| `from` / `to` | ISO 8601 | No | Rango de fechas por `created_at` |
| `page` | entero ≥ 1 | No (default 1) | Página |
| `page_size` | entero 1–100 | No (default 20) | Tamaño de página |

No existe un filtro `execution_status`: toda fila listada aquí es, por definición, una ejecución persistida (ver § 0), así que ese campo nunca varía dentro de este listado.

### Response (`200 OK`)

```json
{
  "schema_version": "1.0",
  "items": [
    {
      "execution_id": "EXE-2026-0001",
      "payment_id": "PAY-2026-001",
      "amount": 15000,
      "currency": "MXN",
      "requested_by": "USR-001",
      "decision": "BLOCK",
      "execution_status": "PERSISTED",
      "investigation_status": "PARTIAL",
      "created_at": "2026-10-15T18:00:00Z"
    }
  ],
  "page": 1,
  "page_size": 20,
  "total_items": 1
}
```

**Autenticación:** requerida en despliegue público; opcional en demo local controlada (ver [security.md](security.md)).

**Errores relevantes:** `400` `VALIDATION_ERROR` (parámetros de filtro inválidos) · `401`/`403` según entorno.

## 5. `GET /api/v1/executions/:id`

**Objetivo:** consultar el detalle completo de una ejecución, incluida la última evaluación de reglas y el estado de investigación vigente.

**Parámetros de ruta:** `id` (`execution_id`).

### Response (`200 OK`)

Mismo cuerpo que la respuesta de `POST /api/v1/executions` (§ 3), más:

```json
{
  "investigation": {
    "status": "PARTIAL",
    "attempts": 2,
    "last_attempt_id": "INV-ATT-0002",
    "last_updated_at": "2026-10-15T18:04:12Z"
  }
}
```

**Errores relevantes:** `404` `EXECUTION_NOT_FOUND`.

## 6. `GET /api/v1/executions/:id/evidence`

**Objetivo:** consultar las evidencias asociadas a una ejecución y, si existe, el informe del agente.

### Response (`200 OK`)

```json
{
  "schema_version": "1.0",
  "execution_id": "EXE-2026-0001",
  "decision": "BLOCK",
  "rules": [ "...mismo arreglo que en POST /executions..." ],
  "evidence_records": [
    {
      "evidence_ref": "EV-0001",
      "evidence_type": "APPROVAL_RECORD",
      "source": "trusted_context.approvals",
      "payload": { "approval_status": "NOT_APPROVED" }
    }
  ],
  "agent_report": {
    "attempt_id": "INV-ATT-0002",
    "status": "PARTIAL",
    "report": { "...contrato del informe del agente, ver § 7..." }
  }
}
```

Si no existe ningún intento de investigación todavía, `agent_report` es `null` y el detalle de `investigation` en `GET /executions/:id` muestra `status: "NOT_STARTED"`.

**Errores relevantes:** `404` `EXECUTION_NOT_FOUND`.

## 7. `POST /api/v1/executions/:id/investigate`

**Objetivo:** iniciar la primera investigación o reintentar una investigación en estado `FAILED` o `PARTIAL`. **Nunca** recalcula ni modifica `decision`, `execution_status`, los datos originales de la ejecución, ni los `rule_evaluations` ya persistidos.

**Parámetros de ruta:** `id` (`execution_id`).

**Request:** cuerpo vacío o `{}`.

### Comportamiento según el estado actual

| `investigation_status` actual | Efecto de la llamada |
|---|---|
| `NOT_STARTED` | Crea el primer intento, pasa a `PENDING` → `RUNNING` |
| `PENDING` o `RUNNING` | `409 Conflict`, código `INVESTIGATION_IN_PROGRESS`; no crea un intento duplicado |
| `FAILED` | Crea un nuevo intento (incrementa el historial), pasa a `PENDING` → `RUNNING` |
| `PARTIAL` | Crea un nuevo intento para intentar completar la investigación, pasa a `PENDING` → `RUNNING` |
| `COMPLETED` | `409 Conflict`, código `INVESTIGATION_ALREADY_COMPLETED`; un informe completo no se vuelve a generar automáticamente |

Cada intento queda registrado en `investigation_attempts` (ver [database.md](database.md)); el historial completo es consultable, aunque la API solo expone por defecto el último intento en `GET /executions/:id`.

**Implementación:** este endpoint es una llamada directa a `fn_schedule_investigation_attempt(execution_id)` ([database.md § 5.1](database.md#51-crear-un-intento-sin-duplicar-investigaciones-en-curso)) — la misma función que usa la transacción de creación para el primer intento y el reconciliador de ejecuciones abandonadas ([database.md § 5.6](database.md#56-reconciliador-de-ejecuciones-not_started-defensa-en-profundidad)). El backend traduce el `outcome` de la función directamente: `CREATED` → `202` (tabla de arriba, filas `NOT_STARTED`/`FAILED`/`PARTIAL`), `IN_PROGRESS` → `409 INVESTIGATION_IN_PROGRESS`, `ALREADY_COMPLETED` → `409 INVESTIGATION_ALREADY_COMPLETED`, `EXECUTION_NOT_FOUND` → `404`.

**Concurrencia:** la verificación de la tabla anterior y la creación del nuevo intento ocurren dentro de esa misma función, que bloquea la fila de la ejecución (`SELECT ... FOR UPDATE`), de modo que dos llamadas concurrentes a este endpoint para la misma ejecución — o una llamada concurrente con el reconciliador — nunca crean dos intentos activos a la vez: la segunda siempre ve el `investigation_status` ya actualizado por la primera y recibe `IN_PROGRESS`.

**Resultados tardíos:** si un intento se marca `FAILED` por abandono (recuperado por el *reaper*, [database.md § 5.3](database.md#53-el-reaper-recuperar-trabajos-abandonados)) y la llamada original a Nebius responde más tarde, esa respuesta tardía se descarta internamente (ver [database.md § 5.4](database.md#54-fencing-finalización-atómica-de-un-intento)) y nunca llega a ser visible por la API: el cliente que consulte `GET /executions/:id` en ese momento ya ve `FAILED` con `error_code = WORKER_LOST`, y puede reintentar con una nueva llamada a este endpoint.

### Response (`202 Accepted`)

```json
{
  "schema_version": "1.0",
  "execution_id": "EXE-2026-0001",
  "investigation_status": "PENDING",
  "attempt_id": "INV-ATT-0003"
}
```

**Errores relevantes:** `404` `EXECUTION_NOT_FOUND` · `409` `INVESTIGATION_IN_PROGRESS` / `INVESTIGATION_ALREADY_COMPLETED`.

## 8. `GET /api/v1/rules`

**Objetivo:** consultar las reglas activas y su versión de política vigente.

### Response (`200 OK`)

```json
{
  "schema_version": "1.0",
  "policy_version": "payment-policy-v1",
  "rules": [
    {
      "rule_id": "PAY-001",
      "name": "Límite de autorización",
      "enabled": true,
      "config": { "approval_threshold_minor": 1000000 }
    }
  ]
}
```

**Autenticación:** lectura pública permitida en demo; en despliegue público, requerida como el resto de endpoints.

## 9. `GET /api/v1/health`

**Objetivo:** comprobar disponibilidad de la API y, opcionalmente, de sus dependencias críticas.

### Response (`200 OK`)

```json
{
  "status": "OK",
  "dependencies": {
    "database": "OK",
    "nebius": "UNKNOWN"
  }
}
```

`nebius` se reporta como `UNKNOWN` cuando el health check no prueba activamente el proveedor (recomendado para el MVP, ya que el agente no es crítico para servir el resto de la API). Si en algún momento se agrega una comprobación activa, debe tener su propio timeout corto y no bloquear la respuesta del endpoint.

## 10. Contrato del informe del agente

```json
{
  "schema_version": "1.0",
  "execution_id": "EXE-2026-0001",
  "attempt_id": "INV-ATT-0002",
  "agent": {
    "provider": "nebius",
    "model": "CONFIGURED_MODEL",
    "prompt_version": "flowguard-investigator-v1"
  },
  "investigation": {
    "status": "PARTIAL",
    "headline": "Pago bloqueado por falta de autorización",
    "explanation": "El importe supera el límite configurado y no existe una aprobación válida registrada.",
    "findings": [
      {
        "finding_id": "FND-001",
        "rule_id": "PAY-001",
        "claim": "El pago requiere aprobación del supervisor.",
        "evidence_refs": ["EV-0001", "EV-0002"]
      }
    ],
    "recommended_actions": [
      {
        "action": "REQUEST_SUPERVISOR_APPROVAL",
        "requires_human_action": true
      }
    ]
  },
  "decision_summary": {
    "decision": "BLOCK",
    "decision_source": "RULE_ENGINE",
    "agent_can_override": false
  },
  "limitations": [
    "No se completaron todas las consultas complementarias."
  ]
}
```

`investigation.status` en este contrato usa el mismo conjunto cerrado que `investigation_status` (§ 0): `COMPLETED`, `PARTIAL` o `FAILED` (un intento `FAILED` puede no producir este objeto en absoluto; ver [security.md § 1.7](security.md#17-validación-de-informes)).

## Referencias

- Modelo de estados persistido, tablas e índices: [database.md](database.md)
- Reglas de negocio aplicadas por el motor: [rules.md](rules.md)
- Controles de validación y autorización por entorno: [security.md](security.md)
- Catálogo completo de errores: [error-catalog.md](error-catalog.md)
