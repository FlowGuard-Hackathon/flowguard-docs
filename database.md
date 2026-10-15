# Base de datos y persistencia

Base de datos oficial: **Supabase / PostgreSQL**. Este documento es la fuente de verdad para nombres de tablas, columnas y estados; [api-contracts.md](api-contracts.md) debe mantenerse coherente con lo descrito aquí.

> **Estado de verificación de la migración:** la migración de referencia ([`supabase/migrations/001_initial_schema.sql`](https://github.com/FlowGuard-Hackathon/flowguard-supabase/blob/main/migrations/001_initial_schema.sql)), incluidas las cuatro funciones de § 5 y § 2.4, se ejecutó contra un PostgreSQL 16 genérico en contenedores Docker desechables, en tres rondas (una por cada auditoría de esta especificación). La ronda más reciente encontró y corrigió un error real (`RETURNING` ambiguo en `fn_finalize_investigation_attempt`, por colisión de nombres entre columnas de tabla y salidas de la función) y, tras corregirlo, confirmó con datos reales: creación atómica del primer intento sin ventana `NOT_STARTED` huérfana; el bloqueo de `fn_schedule_investigation_attempt` contra intentos duplicados; el token de reclamación de `fn_finalize_investigation_attempt` rechazando a un worker que no es el dueño del intento; la propagación correcta a `executions` solo cuando el intento sigue vigente; el reconciliador de § 5.6 recuperando una ejecución `NOT_STARTED` simulada; el *reaper* de § 5.3 recuperando un intento `RUNNING` abandonado y propagando correctamente el resultado (a diferencia de la versión con dos `UPDATE` sueltos de la auditoría anterior); y el contador de *rate limiting* de § 2.4, bajo 20 solicitudes realmente concurrentes contra un límite de 10, permitiendo exactamente 10 y rechazando exactamente 10. **No** se ha ejecutado contra un proyecto Supabase real (que añade roles y extensiones propias de la plataforma) — ese paso queda para el repositorio de implementación.

## 1. Modelo de datos

| Tabla | Responsabilidad |
|---|---|
| `flow_definitions` | Definición del flujo/escenario disponible |
| `policy_rules` | Configuración versionada de cada regla |
| `budget_balances` | Presupuesto simulado por categoría/ámbito |
| `payment_requests` | Datos de la solicitud de pago tal como fue validada |
| `idempotency_keys` | Control de unicidad e idempotencia de solicitudes de creación |
| `executions` | Ejecución: decisión, estado técnico y versión de política aplicada |
| `rule_evaluations` | Resultado de cada una de las cinco reglas para una ejecución |
| `evidence_records` | Evidencias consultables que respaldan cada resultado de regla |
| `investigation_attempts` | Historial de cada intento de investigación de IA |
| `agent_reports` | Informe estructurado producido por un intento de investigación |
| `rate_limit_counters` | Contador compartido de solicitudes por cliente/endpoint (§ 2.4 de [security.md](security.md#24-límite-de-solicitudes-rate-limiting)) |

No se agregan tablas de propósito empresarial (roles multiempresa, auditoría genérica, colas de eventos) que el MVP no necesita. `idempotency_keys` e `investigation_attempts` se agregan porque son indispensables para implementar correctamente la idempotencia (sección 5 de [api-contracts.md](api-contracts.md)) y el historial de reintentos de investigación (sección 7), que de otro modo no tendrían dónde persistirse.

### 1.1. `flow_definitions`

```sql
CREATE TABLE flow_definitions (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name          TEXT NOT NULL,
  scenario_id   TEXT NOT NULL,
  version       TEXT NOT NULL,
  is_active     BOOLEAN NOT NULL DEFAULT TRUE,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (scenario_id, version)
);

CREATE INDEX idx_flow_definitions_scenario_active
  ON flow_definitions (scenario_id)
  WHERE is_active;
```

### 1.2. `policy_rules`

```sql
CREATE TABLE policy_rules (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  rule_id       TEXT NOT NULL,            -- 'PAY-001' ... 'PAY-005'
  policy_version TEXT NOT NULL,           -- 'payment-policy-v1'
  config_json   JSONB NOT NULL,
  enabled       BOOLEAN NOT NULL DEFAULT TRUE,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (rule_id, policy_version)
);

CREATE INDEX idx_policy_rules_version ON policy_rules (policy_version);
```

Conservar todas las versiones históricas (no se eliminan filas): una ejecución pasada debe poder reconstruirse con la configuración vigente en el momento en que se evaluó, mediante `executions.policy_version`.

### 1.3. `budget_balances`

```sql
CREATE TABLE budget_balances (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  budget_id         TEXT NOT NULL UNIQUE,   -- 'BUD-OPERATIONS-01'
  available_minor   BIGINT NOT NULL CHECK (available_minor >= 0),
  version           INTEGER NOT NULL DEFAULT 1,
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

`version` se usa para control de concurrencia optimista (`UPDATE ... WHERE budget_id = $1 AND version = $2`) si el simulador llega a descontar presupuesto de forma concurrente. Para el MVP, el presupuesto se simula y se consulta como saldo disponible; no se mueve dinero real.

### 1.4. `payment_requests`

```sql
CREATE TABLE payment_requests (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  payment_id      TEXT NOT NULL UNIQUE,     -- 'PAY-2026-001'
  amount_minor    BIGINT NOT NULL CHECK (amount_minor > 0),
  currency        TEXT NOT NULL CHECK (currency IN ('MXN')),
  category        TEXT NOT NULL,
  description     TEXT,
  requester_id    TEXT NOT NULL,
  budget_id       TEXT NOT NULL REFERENCES budget_balances (budget_id),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payment_requests_requester ON payment_requests (requester_id);
```

`amount_minor` es la cantidad en centavos enteros (ver [api-contracts.md § 2.1](api-contracts.md#21-importes)). Los adjuntos (`attachments`) se resuelven contra el contexto confiable del backend en tiempo de validación y se referencian desde `evidence_records`, no se duplican aquí.

### 1.5. `idempotency_keys`

```sql
CREATE TABLE idempotency_keys (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  idempotency_key   TEXT NOT NULL,
  scenario_id       TEXT NOT NULL,
  request_hash      TEXT NOT NULL,          -- hash determinista del payload normalizado
  execution_status  TEXT NOT NULL DEFAULT 'RECEIVED'
                     CHECK (execution_status IN
                       ('RECEIVED','VALIDATING','EVALUATING','PERSISTED','FAILED')),
  execution_id      UUID REFERENCES executions (id),
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (idempotency_key, scenario_id)
);
```

Esta tabla es la autoridad de idempotencia: se inserta una fila **antes** de evaluar las reglas, dentro de la misma transacción que reserva el intento. Una segunda solicitud concurrente con la misma clave falla por violación de `UNIQUE (idempotency_key, scenario_id)`, y el backend decide la respuesta (`200` con el resultado existente, o `409 IDEMPOTENCY_IN_PROGRESS` / `409 IDEMPOTENCY_KEY_REUSED`) según `request_hash` y `execution_status`. Ver [api-contracts.md § 2.2](api-contracts.md#22-idempotencia).

### 1.6. `executions`

```sql
CREATE TABLE executions (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  execution_code        TEXT NOT NULL UNIQUE,    -- 'EXE-2026-0001', visible en la API
  payment_request_id    UUID NOT NULL REFERENCES payment_requests (id),
  engine_version        TEXT NOT NULL,
  policy_version        TEXT NOT NULL,
  decision              TEXT NOT NULL CHECK (decision IN ('ALLOW','BLOCK')),
  investigation_status  TEXT NOT NULL DEFAULT 'NOT_STARTED'
                         CHECK (investigation_status IN
                           ('NOT_STARTED','PENDING','RUNNING','COMPLETED','PARTIAL','FAILED')),
  latest_attempt_number INTEGER NOT NULL DEFAULT 0,
  created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_executions_created_at ON executions (created_at DESC);
CREATE INDEX idx_executions_decision ON executions (decision);
CREATE INDEX idx_executions_investigation_status ON executions (investigation_status);
```

**`executions` no tiene columna `execution_status`.** En la primera versión de este documento sí la tenía, con el mismo conjunto cerrado que `idempotency_keys.execution_status`; era redundante y, peor, engañosa: como una fila en `executions` solo se crea dentro de la transacción que confirma la persistencia (paso 5 de § 3), esa columna **nunca podía observarse con un valor distinto de `'PERSISTED'`** en una fila real — `RECEIVED`, `VALIDATING`, `EVALUATING` y `FAILED` jamás ocurrían ahí. Se eliminó para no sugerir una máquina de estados que la tabla no implementa.

- **Fuente de verdad de `decision`:** esta tabla, columna `decision`. Se escribe una sola vez, en el `INSERT`, y ninguna sentencia `UPDATE` de la aplicación la toca jamás.
- **Fuente de verdad del estado de la *creación* de la ejecución** (`RECEIVED` → `VALIDATING` → `EVALUATING` → `PERSISTED`/`FAILED`): la tabla `idempotency_keys` (§ 1.5), no `executions`. La existencia misma de una fila en `executions` ya *es* el hecho de que ese pipeline llegó a `PERSISTED`; no hace falta una columna que lo repita. El contrato de API (ver [api-contracts.md § 0](api-contracts.md#0-modelo-de-estados-resumen)) sigue exponiendo un campo `execution_status` en las respuestas JSON, pero es un valor **constante** (`"PERSISTED"`) calculado al servir la respuesta, no una columna almacenada — se documenta así explícitamente para no prometer transiciones que no existen en el dato.
- **Fuente de verdad del estado de *cada intento* de investigación:** `investigation_attempts` (§ 1.9).
- **Fuente de verdad del estado *vigente* de la investigación** (el que ve el frontend sin tener que calcular un `MAX(attempt_number)` en cada lectura): la columna `investigation_status` de esta tabla, **derivada** de `investigation_attempts` y mantenida al día mediante la regla de *fencing* de § 2.3. `latest_attempt_number` se guarda junto a ella por la misma razón: permite a cualquier `UPDATE` que intente escribir el resultado de un intento verificar, en una sola condición `WHERE`, si sigue siendo el intento más reciente antes de escribir. Si `investigation_status`/`latest_attempt_number` llegaran a corromperse por un bug, se reconstruyen de forma determinista con:

  ```sql
  SELECT attempt_number, status
  FROM investigation_attempts
  WHERE execution_id = $1
  ORDER BY attempt_number DESC
  LIMIT 1;
  ```

### 1.7. `rule_evaluations`

```sql
CREATE TABLE rule_evaluations (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  execution_id    UUID NOT NULL REFERENCES executions (id) ON DELETE CASCADE,
  rule_id         TEXT NOT NULL,
  status          TEXT NOT NULL CHECK (status IN ('PASS','FAIL','INDETERMINATE')),
  observed_json   JSONB NOT NULL,
  expected_json   JSONB NOT NULL,
  message         TEXT,
  severity        TEXT NOT NULL DEFAULT 'BLOCKING' CHECK (severity IN ('BLOCKING','INFO')),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (execution_id, rule_id)
);

CREATE INDEX idx_rule_evaluations_execution ON rule_evaluations (execution_id);
```

La restricción `UNIQUE (execution_id, rule_id)` (ya prevista en la versión anterior de este documento) impide que una misma regla quede evaluada dos veces para la misma ejecución.

### 1.8. `evidence_records`

```sql
CREATE TABLE evidence_records (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  evidence_ref    TEXT NOT NULL,           -- 'EV-0001', visible en la API
  execution_id    UUID NOT NULL REFERENCES executions (id) ON DELETE CASCADE,
  evidence_type   TEXT NOT NULL,           -- 'APPROVAL_RECORD', 'INVOICE_RECORD', ...
  source          TEXT NOT NULL,           -- de qué parte del contexto confiable proviene
  payload_json    JSONB NOT NULL,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (execution_id, evidence_ref)
);

CREATE INDEX idx_evidence_records_execution ON evidence_records (execution_id);
```

### 1.9. `investigation_attempts`

```sql
CREATE TABLE investigation_attempts (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  attempt_code      TEXT NOT NULL UNIQUE,     -- 'INV-ATT-0002', visible en la API
  execution_id      UUID NOT NULL REFERENCES executions (id) ON DELETE CASCADE,
  attempt_number    INTEGER NOT NULL,         -- 1, 2, 3... por execution_id
  status            TEXT NOT NULL CHECK (status IN
                       ('PENDING','RUNNING','COMPLETED','PARTIAL','FAILED')),
  tool_calls_used   INTEGER NOT NULL DEFAULT 0,
  error_code        TEXT,                     -- código del catálogo de errores si FAILED
  claimed_by        TEXT,                     -- identificador del proceso worker que lo tomó
  claimed_at        TIMESTAMPTZ,              -- cuándo pasó a RUNNING; base para detectar abandono
  started_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  completed_at      TIMESTAMPTZ,
  UNIQUE (execution_id, attempt_number)
);

CREATE INDEX idx_investigation_attempts_execution ON investigation_attempts (execution_id);

-- Cola de trabajo: el worker escanea solo esta franja, nunca toda la tabla.
CREATE INDEX idx_investigation_attempts_pending
  ON investigation_attempts (started_at)
  WHERE status = 'PENDING';

-- Detección de intentos abandonados (ver § 5.3).
CREATE INDEX idx_investigation_attempts_running_claimed
  ON investigation_attempts (claimed_at)
  WHERE status = 'RUNNING';
```

Cada llamada a `POST /api/v1/executions/:id/investigate` que resulta permitida (ver tabla de comportamiento en [api-contracts.md § 7](api-contracts.md#7-post-apiv1executionsidinvestigate)) crea una fila nueva con `attempt_number` incrementado, calculado y bloqueado dentro de la misma transacción que valida que no haya otro intento `PENDING`/`RUNNING` (ver § 5.1). `executions.investigation_status`/`latest_attempt_number` se actualizan junto con cada transición de esta tabla, nunca por separado.

### 1.10. `agent_reports`

```sql
CREATE TABLE agent_reports (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  attempt_id      UUID NOT NULL REFERENCES investigation_attempts (id) ON DELETE CASCADE,
  provider        TEXT NOT NULL DEFAULT 'nebius',
  model           TEXT NOT NULL,
  prompt_version  TEXT NOT NULL,
  report_json     JSONB NOT NULL,           -- contrato de api-contracts.md § 10
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (attempt_id)
);
```

Solo existe una fila de `agent_reports` por intento, y solo para intentos cuyo resultado final es `COMPLETED` o `PARTIAL`. Un intento `FAILED` no produce fila en `agent_reports`; su información queda en `investigation_attempts.error_code`.

## 2. Modelo de estados unificado

Tres campos independientes, cada uno con su propio conjunto cerrado de valores. Ningún valor se reutiliza entre campos con un significado distinto.

### 2.1. `decision`

| Valor | Significado |
|---|---|
| `ALLOW` | Las cinco reglas pasaron |
| `BLOCK` | Al menos una regla falló o quedó indeterminada |

Se asigna **una sola vez**, en el momento en que el motor evalúa la solicitud, y nunca cambia después. Ningún otro componente (incluida la IA) puede modificarlo.

### 2.2. `execution_status`

Esta máquina de estados vive exclusivamente en `idempotency_keys.execution_status` (§ 1.5); `executions` no tiene esta columna (ver § 1.6 para la justificación).

| Valor | Significado | ¿Terminal? |
|---|---|---|
| `RECEIVED` | La solicitud llegó y se reservó la clave de idempotencia | No |
| `VALIDATING` | Se están validando el contrato y el contexto confiable | No |
| `EVALUATING` | El motor está evaluando las cinco reglas | No |
| `PERSISTED` | La ejecución, la decisión y las cinco evaluaciones quedaron guardadas en una transacción confirmada; `idempotency_keys.execution_id` ya apunta a la fila de `executions` | **Sí** |
| `FAILED` | Un error técnico impidió completar el pipeline antes de persistir | No — admite reintento con la misma clave (ver [api-contracts.md § 2.2, Caso D](api-contracts.md#22-idempotencia)) |

**Transiciones permitidas:** `RECEIVED → VALIDATING → EVALUATING → PERSISTED`, o una salida a `FAILED` desde cualquiera de los tres primeros estados. Desde `FAILED`, una nueva solicitud con la misma clave y el mismo `request_hash` puede reiniciar el ciclo (`FAILED → RECEIVED`); es la única transición que sale de un estado no exitoso. **Prohibido:** saltar directamente a `PERSISTED` sin pasar por `EVALUATING`; transicionar desde `PERSISTED` hacia cualquier otro valor (es terminal en sentido estricto); reintentar automáticamente desde `FAILED` sin que llegue una nueva solicitud del cliente.

**Quién cambia este campo:** exclusivamente el orquestador de ejecuciones del backend (`src/server/executions/`). Ni el frontend ni el agente de IA pueden escribirlo.

**Qué queda persistido si falla:** como `executions` solo recibe una fila al confirmar la transacción (paso 5 de § 3), un fallo antes de ese punto **no** deja una fila en `executions`. El único rastro persistido es la fila de `idempotency_keys` con `execution_status = 'FAILED'`. El frontend recibe el error correspondiente del catálogo (ver [error-catalog.md](error-catalog.md)) y debe tratarlo como "la operación no se evaluó", nunca como una decisión de negocio.

### 2.3. `investigation_status`

| Valor | Significado | ¿Terminal? |
|---|---|---|
| `NOT_STARTED` | Valor por defecto interno antes de que `fn_schedule_investigation_attempt` cree el primer intento dentro de la misma transacción ([§ 3](#3-persistencia-segura)); **no debería observarse en una fila confirmada bajo operación normal**. Si aparece, el reconciliador de [§ 5.6](#56-reconciliador-de-ejecuciones-not_started-defensa-en-profundidad) lo recupera | No |
| `PENDING` | Se creó un intento y está en cola para ejecutarse | No |
| `RUNNING` | El agente está consultando herramientas y generando el informe | No |
| `COMPLETED` | El intento más reciente produjo un informe completo y validado | Sí (sin reintento automático) |
| `PARTIAL` | El intento más reciente produjo un informe útil pero incompleto | No — admite reintento manual |
| `FAILED` | El intento más reciente no produjo un informe utilizable | No — admite reintento manual |

**Transiciones permitidas:** `NOT_STARTED → PENDING → RUNNING → {COMPLETED | PARTIAL | FAILED}`. Desde `FAILED` o `PARTIAL`, una nueva llamada a `POST /investigate` crea un intento adicional y vuelve a `PENDING → RUNNING → ...`. Un caso especial de `RUNNING → FAILED` ocurre sin que el agente responda: el *reaper* descrito en § 5.3 lo fuerza cuando un intento queda abandonado. **Prohibido:** reintentar automáticamente desde `COMPLETED` (ver tabla de comportamiento en [api-contracts.md § 7](api-contracts.md#7-post-apiv1executionsidinvestigate)); transicionar sin pasar por `PENDING`/`RUNNING`; que dos intentos del mismo `execution_id` estén en `PENDING`/`RUNNING` a la vez (impedido por el bloqueo de § 5.1).

**Quién cambia este campo:** exclusivamente `fn_schedule_investigation_attempt` (al crear el primer intento o un reintento, invocada desde la transacción de creación, desde `POST /investigate`, o desde el reconciliador) y `fn_finalize_investigation_attempt`/`fn_reap_abandoned_attempts` (al resolverlo o recuperarlo) — ver § 5. Nunca el modelo de Nebius directamente ni el frontend. El frontend solo puede *solicitar* una transición llamando a `POST /investigate`; el backend decide si es válida.

**Cómo se representan los errores:** un intento `FAILED` registra `error_code` en `investigation_attempts` (código del catálogo: `PROVIDER_TIMEOUT`, `PROVIDER_INVALID_RESPONSE`, `EVIDENCE_REFERENCE_INVALID`, `WORKER_LOST`, etc. — ver [error-catalog.md](error-catalog.md)). `decision` no se ve afectada en ningún caso.

**Qué respuesta recibe el frontend:** siempre `decision` (inmutable) del registro persistido en `executions`, un `execution_status` constante `"PERSISTED"` (ver § 2.2), más el `investigation_status` vigente derivado del intento más reciente. Nunca una respuesta que combine una decisión no persistida con un estado de investigación avanzado.

**Un intento antiguo nunca sobrescribe un reintento más reciente** (requisito explícito del hackathon): toda escritura de resultado sobre `investigation_attempts` y toda propagación hacia `executions.investigation_status` está condicionada por `attempt_number`/`latest_attempt_number`, no solo por el `id` del intento. El mecanismo exacto está en § 5.4.

## 3. Persistencia segura

La versión anterior de este documento dividía esto en dos transacciones (persistir la decisión, y luego programar la investigación por separado), con el argumento de que un fallo al programar la investigación no debía revertir la decisión. Esa separación dejaba una ventana real: si la segunda transacción fallaba, la ejecución quedaba en `investigation_status = 'NOT_STARTED'` **sin ningún trabajo pendiente que un worker pudiera reclamar**, y el *reaper* de § 5.3 no la detectaba porque solo vigila intentos que llegaron a `RUNNING` — un `NOT_STARTED` sin intento asociado es invisible para él. Esta sección corrige eso fusionando la creación del primer intento en la misma transacción que persiste la decisión (§ 3, abajo), y añade un reconciliador como defensa adicional para cualquier fila que, por una razón no prevista (migración de datos, bug futuro), quede en ese estado igualmente (§ 5.6).

La secuencia para `POST /api/v1/executions` es ahora **una sola transacción** para todo lo que debe quedar atómico junto con la decisión:

1. Validar la solicitud y la identidad (`idempotency_keys.execution_status` conceptual: `VALIDATING`).
2. Reservar/reutilizar la clave de idempotencia en `idempotency_keys` según el algoritmo de [api-contracts.md § 2.2](api-contracts.md#22-idempotencia) (`request_hash`, `execution_status = 'RECEIVED'`, o reactivación desde `FAILED`).
3. Obtener el contexto confiable (presupuesto, permisos, adjuntos).
4. Evaluar las cinco reglas (`execution_status` conceptual: `EVALUATING`).
5. **En una única transacción:** insertar `payment_requests`; insertar `executions` (`investigation_status = 'NOT_STARTED'`, `latest_attempt_number = 0`); insertar las cinco filas de `rule_evaluations` y las filas de `evidence_records` correspondientes; actualizar `idempotency_keys.execution_status = 'PERSISTED'` y `idempotency_keys.execution_id`; **y, como última sentencia de la misma transacción, invocar `fn_schedule_investigation_attempt(execution_id)`** (§ 5.1) para crear el primer intento (`attempt_number = 1`, `status = 'PENDING'`) y dejar `executions.investigation_status = 'PENDING'`, `latest_attempt_number = 1`.
6. Confirmar la transacción.

Si cualquier paso de la transacción falla — incluida la creación del primer intento — **todo** se revierte: no queda fila en `executions` ni en `investigation_attempts`, solo `idempotency_keys.execution_status = 'FAILED'`. La API responde `500 PERSISTENCE_FAILURE` y nunca comunica una autorización satisfactoria. Esto es deliberado y es la razón por la que fusionar ambas operaciones es seguro: crear la fila `PENDING` en `investigation_attempts` es trabajo puramente de base de datos (ningún `INSERT` llama a Nebius ni a ningún servicio externo), así que no hay ningún motivo legítimo para que tenga éxito la persistencia de la decisión y falle por separado la creación del intento — si la base de datos está disponible para lo primero, lo está para lo segundo, en la misma sentencia.

**Qué significa esto para `investigation_status = 'NOT_STARTED'`:** bajo operación normal, ninguna ejecución `PERSISTED` debería observarse jamás en `NOT_STARTED` — el estado pasa a existir únicamente como un valor transitorio interno a la transacción, nunca confirmado de forma visible sin su `PENDING` correspondiente. Se conserva en el `CHECK` de la columna (§ 1.6) por compatibilidad hacia atrás y como valor de entrada por defecto dentro de la función, no porque se espere observarlo en una fila confirmada. Si aparece en una fila real, es la señal inequívoca de un bug o de datos migrados desde una versión anterior de este esquema — exactamente el caso que cubre el reconciliador de § 5.6.

## 4. Conservación y eliminación de registros

Para el MVP no se implementa purga automática: todas las filas de `executions`, `rule_evaluations`, `evidence_records`, `investigation_attempts` y `agent_reports` se conservan indefinidamente, porque son la base de la trazabilidad exigida en [security.md § 2.3](security.md#23-evidencia-y-trazabilidad). Si en el futuro se requiere una política de retención, debe definirse por separado y no mezclarse con esta especificación del MVP.

## 5. Concurrencia y recuperación del mecanismo asíncrono de investigación

Esta sección da las sentencias exactas para las garantías exigidas al mecanismo asíncrono elegido (justificación arquitectónica completa en [architecture.md § 7](architecture.md#7-mecanismo-de-investigación-asíncrona)): un worker persistente que reclama trabajos de `investigation_attempts` mediante bloqueo a nivel de fila, con recuperación tras reinicio porque el estado de cada intento vive en PostgreSQL, no en memoria de proceso.

### 5.1. Crear un intento sin duplicar investigaciones en curso

Esta lógica vive en **una sola función PL/pgSQL**, `fn_schedule_investigation_attempt`, para que la transacción A ([§ 3](#3-persistencia-segura)), `POST /investigate` y el reconciliador ([§ 5.6](#56-reconciliador-de-ejecuciones-not_started-defensa-en-profundidad)) compartan exactamente el mismo algoritmo en vez de reimplementarlo tres veces:

```sql
CREATE OR REPLACE FUNCTION fn_schedule_investigation_attempt(p_execution_id UUID)
RETURNS TABLE(outcome TEXT, attempt_id UUID, attempt_number INTEGER)
LANGUAGE plpgsql AS $$
DECLARE
  v_status TEXT;
  v_latest INTEGER;
  v_attempt_id UUID;
  v_next_attempt INTEGER;
BEGIN
  SELECT investigation_status, latest_attempt_number
  INTO v_status, v_latest
  FROM executions
  WHERE id = p_execution_id
  FOR UPDATE;

  IF NOT FOUND THEN
    RETURN QUERY SELECT 'EXECUTION_NOT_FOUND', NULL::UUID, NULL::INTEGER;
    RETURN;
  END IF;

  IF v_status IN ('PENDING', 'RUNNING') THEN
    RETURN QUERY SELECT 'IN_PROGRESS', NULL::UUID, NULL::INTEGER;
    RETURN;
  END IF;

  IF v_status = 'COMPLETED' THEN
    RETURN QUERY SELECT 'ALREADY_COMPLETED', NULL::UUID, NULL::INTEGER;
    RETURN;
  END IF;

  -- v_status aquí solo puede ser NOT_STARTED, FAILED o PARTIAL: los tres son elegibles.
  v_next_attempt := v_latest + 1;

  INSERT INTO investigation_attempts (attempt_code, execution_id, attempt_number, status)
  VALUES ('INV-ATT-' || lpad(v_next_attempt::TEXT, 4, '0') || '-' || substr(p_execution_id::TEXT, 1, 8),
          p_execution_id, v_next_attempt, 'PENDING')
  RETURNING id INTO v_attempt_id;

  UPDATE executions
  SET investigation_status = 'PENDING', latest_attempt_number = v_next_attempt
  WHERE id = p_execution_id;

  RETURN QUERY SELECT 'CREATED', v_attempt_id, v_next_attempt;
END;
$$;
```

**Condiciones de elegibilidad** (la razón por la que se puede llamar sin riesgo desde tres lugares distintos): un `execution_id` es elegible para un nuevo intento si y solo si su `investigation_status` actual es `NOT_STARTED`, `FAILED` o `PARTIAL`. `PENDING`/`RUNNING` devuelven `IN_PROGRESS` (nada se crea) y `COMPLETED` devuelve `ALREADY_COMPLETED` (nada se crea). El llamador traduce `outcome` a la respuesta HTTP correspondiente (`CREATED` → `202`/`201` según el caso; `IN_PROGRESS` → `409 INVESTIGATION_IN_PROGRESS`; `ALREADY_COMPLETED` → `409 INVESTIGATION_ALREADY_COMPLETED`; `EXECUTION_NOT_FOUND` → `404`).

**Garantía de concurrencia:** el `SELECT ... FOR UPDATE` sobre `executions`, dentro de la función, serializa cualquier llamada concurrente para el mismo `execution_id` — venga de `POST /investigate`, de otra invocación de `POST /investigate`, o del reconciliador compitiendo por la misma fila. La segunda llamada espera a que la primera libere el bloqueo (al terminar la función, dentro de la misma transacción del llamador) y entonces relee un `investigation_status` ya actualizado, devolviendo `IN_PROGRESS` en vez de crear un segundo intento. Esto resuelve de raíz tanto "dos intentos de investigación simultáneos" como "el worker, el reconciliador o un usuario programan la misma investigación a la vez": los tres llamadores pasan por el mismo bloqueo de fila, sin importar quién llegó primero.

**Por qué es una función y no una transacción ensamblada en el backend:** al ejecutarse como una sola sentencia `SELECT * FROM fn_schedule_investigation_attempt($1)`, PostgreSQL garantiza que si cualquier paso interno lanza una excepción, **todos** sus efectos (el `SELECT FOR UPDATE`, el `INSERT`, el `UPDATE`) se revierten como una unidad, sin que el backend tenga que coordinar manualmente un `BEGIN`/`COMMIT` alrededor de varias llamadas de red. Es compatible con Supabase sin cambios: se invoca igual desde una conexión directa de `pg`/`postgres.js` que desde `supabase.rpc('fn_schedule_investigation_attempt', { p_execution_id })`.

### 5.2. El worker reclama un trabajo (`PENDING → RUNNING`)

```sql
UPDATE investigation_attempts
SET status = 'RUNNING', claimed_by = $worker_id, claimed_at = now()
WHERE id = (
  SELECT id FROM investigation_attempts
  WHERE status = 'PENDING'
  ORDER BY started_at
  FOR UPDATE SKIP LOCKED
  LIMIT 1
)
RETURNING id, execution_id, attempt_number;
```

`FOR UPDATE SKIP LOCKED` es la razón por la que varias instancias del worker (o varios *ticks* del mismo worker) pueden ejecutar esta misma consulta en paralelo sin reclamar el mismo trabajo dos veces: la fila que ya está bloqueada por otra transacción se omite en vez de esperar. Inmediatamente después, en otra sentencia, se propaga a `executions`:

```sql
UPDATE executions
SET investigation_status = 'RUNNING'
WHERE id = $execution_id AND investigation_status = 'PENDING';
```

La cláusula `AND investigation_status = 'PENDING'` es la primera aplicación de *fencing*: si por cualquier motivo `executions` ya no refleja `PENDING` (por ejemplo, el reaper ya actuó), esta actualización no tiene efecto y el worker debe revisar el estado antes de continuar.

**`claimed_by` debe ser un identificador fresco por arranque del proceso** (por ejemplo, un UUID generado una vez al iniciar el worker, no simplemente el hostname), nunca un valor fijo reutilizado entre reinicios. La razón es la base del token de reclamación que usa `fn_finalize_investigation_attempt` (§ 5.4): si un worker muere a mitad de un intento y se reinicia, su nueva instancia debe tener un `claimed_by` distinto al de la instancia anterior, para que nunca pueda "reclamar como propio" — y finalizar — un trabajo que en realidad quedó abandonado y le corresponde resolver al *reaper* (§ 5.3), no a una coincidencia accidental de identificador.

### 5.3. El *reaper*: recuperar trabajos abandonados

Un intento puede quedar en `RUNNING` sin resolverse si el proceso worker muere, se reinicia o pierde conexión a mitad de una llamada a Nebius. La versión anterior de esta sección ejecutaba la recuperación como dos sentencias `UPDATE` sueltas desde la aplicación, con el mismo defecto que se corrigió en § 5.4: si el proceso que ejecuta el reaper muere **entre** la primera y la segunda sentencia, `investigation_attempts` queda `FAILED` pero `executions.investigation_status` se queda `RUNNING` para siempre — ningún reaper futuro vuelve a encontrar esa fila, porque ya no está en `RUNNING`. Por eso esto también es una función atómica:

```sql
CREATE OR REPLACE FUNCTION fn_reap_abandoned_attempts(p_stale_threshold_ms BIGINT)
RETURNS TABLE(attempt_id UUID, execution_id UUID, attempt_number INTEGER)
LANGUAGE plpgsql AS $$
DECLARE
  r RECORD;
BEGIN
  FOR r IN
    UPDATE investigation_attempts
    SET status = 'FAILED', error_code = 'WORKER_LOST', completed_at = now()
    WHERE status = 'RUNNING'
      AND claimed_at < now() - (p_stale_threshold_ms || ' milliseconds')::interval
    RETURNING id, investigation_attempts.execution_id, investigation_attempts.attempt_number
  LOOP
    UPDATE executions
    SET investigation_status = 'FAILED'
    WHERE id = r.execution_id
      AND latest_attempt_number = r.attempt_number
      AND investigation_status = 'RUNNING';

    attempt_id := r.id;
    execution_id := r.execution_id;
    attempt_number := r.attempt_number;
    RETURN NEXT;
  END LOOP;
END;
$$;
```

Toda la función corre como una sola transacción: si el proceso que la invocó muere a mitad de la ejecución (p. ej. el propio worker, que también hace de reaper, se cae justo durante este ciclo), PostgreSQL revierte **todo** lo que alcanzó a hacer — ninguna fila queda marcada `FAILED` en `investigation_attempts` sin su propagación correspondiente a `executions`. La próxima vez que cualquier instancia del worker ejecute `SELECT * FROM fn_reap_abandoned_attempts($1)`, las mismas filas (que siguen en `RUNNING` porque la transacción anterior no confirmó nada) vuelven a ser candidatas, sin ninguna inconsistencia acumulada.

`p_stale_threshold_ms` se calcula como un múltiplo de `AGENT_TIMEOUT_MS` más un margen fijo (recomendado: `2 × AGENT_TIMEOUT_MS + 10000`), para no marcar como abandonado un intento que simplemente está cerca del límite normal. Un intento recuperado así queda en `FAILED` con `error_code = 'WORKER_LOST'` (ver [error-catalog.md](error-catalog.md)) y es reintentable mediante `POST /investigate` como cualquier otro `FAILED`.

### 5.4. *Fencing*: finalización atómica de un intento

La versión anterior de esta sección describía el *fencing* como dos sentencias `UPDATE` separadas que el worker ejecutaba una tras otra desde la aplicación. Eso deja una ventana real: si el worker falla **entre** la primera y la segunda sentencia (por ejemplo, se cae justo después de marcar el intento `COMPLETED` pero antes de insertar el informe o de propagar el estado a `executions`), el intento queda en un estado terminal sin su informe, o `executions.investigation_status` no llega a reflejarlo — una inconsistencia real que una "transacción" ensamblada a partir de llamadas HTTP independientes no evita, porque agrupar llamadas de red en una transacción de aplicación no les da atomicidad real en PostgreSQL si el proceso muere a mitad de camino entre una llamada y la siguiente.

La solución es la misma que en § 5.1: una **única función PL/pgSQL** que hace todo el trabajo de finalización como una sola sentencia, con un token de reclamación explícito (`claimed_by`) para verificar que quien finaliza es efectivamente quien reclamó el trabajo:

```sql
CREATE OR REPLACE FUNCTION fn_finalize_investigation_attempt(
  p_attempt_id      UUID,
  p_claimed_by      TEXT,          -- token de reclamación: debe coincidir con investigation_attempts.claimed_by
  p_final_status    TEXT,          -- 'COMPLETED' | 'PARTIAL' | 'FAILED'
  p_tool_calls_used INTEGER,
  p_error_code      TEXT DEFAULT NULL,
  p_provider        TEXT DEFAULT 'nebius',
  p_model           TEXT DEFAULT NULL,
  p_prompt_version  TEXT DEFAULT NULL,
  p_report_json     JSONB DEFAULT NULL
)
RETURNS TABLE(applied BOOLEAN, propagated BOOLEAN, execution_id UUID, attempt_number INTEGER)
LANGUAGE plpgsql AS $$
DECLARE
  v_execution_id UUID;
  v_attempt_number INTEGER;
  v_rows INTEGER;
BEGIN
  IF p_final_status NOT IN ('COMPLETED', 'PARTIAL', 'FAILED') THEN
    RAISE EXCEPTION 'estado terminal inválido: %', p_final_status;
  END IF;

  -- Paso 1: reclamar la finalización, solo si el intento sigue RUNNING y pertenece a este worker.
  -- Las columnas se califican con el nombre de la tabla porque coinciden con
  -- los nombres de salida de la función (RETURNS TABLE) — sin calificar,
  -- PostgreSQL rechaza el RETURNING por ambiguo (se detectó exactamente este
  -- error al probar la función contra PostgreSQL real; ver nota de
  -- verificación al inicio de este documento).
  UPDATE investigation_attempts
  SET status = p_final_status,
      tool_calls_used = p_tool_calls_used,
      error_code = p_error_code,
      completed_at = now()
  WHERE id = p_attempt_id
    AND status = 'RUNNING'
    AND claimed_by = p_claimed_by
  RETURNING investigation_attempts.execution_id, investigation_attempts.attempt_number
  INTO v_execution_id, v_attempt_number;

  IF NOT FOUND THEN
    -- El reaper ya lo recuperó, otro worker lo reclamó, o el intento ya no existe: no se hace nada más.
    RETURN QUERY SELECT FALSE, FALSE, NULL::UUID, NULL::INTEGER;
    RETURN;
  END IF;

  -- Paso 2: persistir el informe, solo si el resultado es COMPLETED o PARTIAL.
  IF p_final_status IN ('COMPLETED', 'PARTIAL') THEN
    INSERT INTO agent_reports (attempt_id, provider, model, prompt_version, report_json)
    VALUES (p_attempt_id, p_provider, p_model, p_prompt_version, p_report_json);
  END IF;

  -- Paso 3: propagar a executions solo si este intento sigue siendo el más reciente (fencing).
  UPDATE executions
  SET investigation_status = p_final_status
  WHERE id = v_execution_id AND latest_attempt_number = v_attempt_number;

  GET DIAGNOSTICS v_rows = ROW_COUNT;

  RETURN QUERY SELECT TRUE, (v_rows > 0), v_execution_id, v_attempt_number;
END;
$$;
```

**Cómo satisface cada requisito:**

- **Verifica que el intento siga perteneciendo al worker autorizado y en `RUNNING`:** la cláusula `WHERE id = p_attempt_id AND status = 'RUNNING' AND claimed_by = p_claimed_by` del paso 1. `p_claimed_by` es el mismo valor que el worker usó para reclamar el trabajo en § 5.2 — actúa como un *token de reclamación*: debe generarse de nuevo (no reutilizar un valor fijo como el hostname) cada vez que arranca un proceso worker, precisamente para que, si un worker muere y se reinicia, la nueva instancia **no** herede accidentalmente la propiedad de un trabajo que la instancia anterior había reclamado antes de morir — esa recuperación es responsabilidad exclusiva del *reaper* (§ 5.3), nunca de una coincidencia de identificador.
- **Verifica que el intento siga siendo el más reciente:** la cláusula `AND latest_attempt_number = v_attempt_number` del paso 3 — igual que en la versión anterior, pero ahora dentro de la misma función.
- **Resultado tardío de Nebius sobre un intento ya recuperado por el *reaper*:** si el *reaper* (§ 5.3) ya puso `status = 'FAILED'` en ese intento, el paso 1 de esta función no encuentra ninguna fila que actualizar (`status = 'RUNNING'` ya no se cumple) y devuelve `applied = FALSE` sin ejecutar los pasos 2 y 3. El worker que recibe `applied = FALSE` descarta el resultado (lo registra en logs, nunca lo reintenta como si fuera un error nuevo) y no muestra ningún error al cliente — la ejecución ya refleja correctamente `FAILED` por abandono.
- **Nada queda parcialmente confirmado:** los tres pasos corren dentro del cuerpo de una sola función invocada como una sola sentencia SQL. Si el paso 2 (`INSERT INTO agent_reports`) lanza una excepción — por ejemplo, una violación de una restricción `NOT NULL` en `model` —, PostgreSQL revierte automáticamente **todo** lo que la función alcanzó a hacer, incluido el `UPDATE` del paso 1, porque una excepción no capturada dentro de una función aborta la transacción que la invoca. El backend nunca ve un intento marcado `COMPLETED` sin su informe.
- **No requiere agrupar llamadas HTTP:** el backend hace una sola llamada (`SELECT * FROM fn_finalize_investigation_attempt(...)` o el `.rpc()` equivalente de Supabase), no una secuencia de peticiones envueltas en una transacción de aplicación — la atomicidad existe en PostgreSQL, no en el código que la invoca.
- **Historial sin sobrescritura:** un intento antiguo que termina tarde (`applied = TRUE` porque todavía estaba `RUNNING`, pero `propagated = FALSE` porque ya no es el más reciente) queda correctamente registrado en `investigation_attempts`/`agent_reports` para el historial, sin pisar el `investigation_status` vigente de `executions`. Esto solo puede ocurrir si ese intento ya había terminado antes en `FAILED`/`PARTIAL` y alguien pidió un reintento (dado el bloqueo de § 5.1), así que es un caso legítimo, no un error.

### 5.5. Dónde corre el worker (supuesto de despliegue)

El mecanismo anterior asume que existe al menos un proceso Node.js de larga duración ejecutando un bucle de sondeo (`setInterval`/bucle `async` con espera) que repite § 5.2–5.3 cada `AGENT_WORKER_POLL_INTERVAL_MS`. Ver la justificación de esta elección, el supuesto de hosting concreto y la alternativa para plataformas estrictamente *serverless* en [architecture.md § 7](architecture.md#7-mecanismo-de-investigación-asíncrona).

### 5.6. Reconciliador de ejecuciones `NOT_STARTED` (defensa en profundidad)

Con el cambio de § 3, una ejecución `PERSISTED` nunca debería quedar en `investigation_status = 'NOT_STARTED'` bajo operación normal: el primer intento se crea en la misma transacción que la decisión. Este reconciliador existe para el residuo de riesgo que esa garantía no cubre — principalmente, filas heredadas de una versión anterior de este esquema (antes de este cambio) o un bug futuro que reintroduzca el problema — no como el mecanismo principal de prevención.

**Algoritmo**, ejecutado en el mismo bucle del worker (§ 5.5), en un ciclo propio y mucho menos frecuente que el sondeo de `PENDING` (recomendado: cada `RECONCILER_INTERVAL_MS`, por defecto 60000 — un orden de magnitud más espaciado que `AGENT_WORKER_POLL_INTERVAL_MS`, porque en el caso normal esta consulta no debe encontrar ninguna fila):

```sql
SELECT id
FROM executions
WHERE investigation_status = 'NOT_STARTED'
  AND created_at < now() - interval '30 seconds'
ORDER BY created_at
LIMIT 20;
```

**Condición de elegibilidad:** `investigation_status = 'NOT_STARTED'` y `created_at` anterior a un margen de gracia (30 segundos recomendado) — el margen evita que el reconciliador compita innecesariamente con la transacción de creación en el instante en que todavía está en curso bajo carga alta; no es una ventana de inconsistencia esperada, es solo para no disparar falsos positivos bajo latencia normal.

Por cada fila candidata, el reconciliador llama a la **misma función** que todo lo demás, sin lógica propia de creación:

```sql
SELECT * FROM fn_schedule_investigation_attempt($execution_id);
```

Como esa función ya es idempotente y segura ante concurrencia (§ 5.1, mediante `SELECT ... FOR UPDATE`), no hace falta ninguna coordinación adicional entre el reconciliador y una llamada simultánea a `POST /investigate` sobre la misma fila: cualquiera que llegue primero gana el bloqueo, crea el intento, y el otro recibe `IN_PROGRESS` sin duplicar nada. Esto responde directamente al requisito de "evitar intentos duplicados cuando el worker, el reconciliador o un usuario intentan programar la misma investigación simultáneamente": los tres comparten el mismo punto de entrada atómico.

**Por qué un reconciliador por sondeo y no un *outbox* transaccional separado:** un *outbox* transaccional clásico necesitaría una tabla intermedia y un proceso relator que la lea y publique el trabajo "real" — exactamente la responsabilidad que `investigation_attempts` ya cumple. Separar eso en una tabla de *outbox* adicional sería introducir una pieza más sin necesidad: aquí la prevención (§ 3, crear el intento en la misma transacción que la decisión) ya cubre el caso normal con garantías más fuertes que cualquier *outbox* asíncrono, y el reconciliador cubre el residuo. Para un hackathon de 15 días, dos piezas (prevención atómica + reconciliador de baja frecuencia) sobre la infraestructura que ya existe es más simple y más confiable que construir y operar un patrón *outbox* completo para un caso que, bien prevenido, debería ocurrir cero veces.

## Referencias

- Contratos JSON que se persisten en estas tablas: [api-contracts.md](api-contracts.md)
- Reglas que generan `rule_evaluations`: [rules.md](rules.md)
- Controles de acceso a estas tablas (RLS, roles de servicio): [security.md](security.md)
- Mecanismo asíncrono: justificación, hosting y alternativa serverless: [architecture.md § 7](architecture.md#7-mecanismo-de-investigación-asíncrona)
- Migración SQL de referencia: [`supabase/migrations/001_initial_schema.sql`](https://github.com/FlowGuard-Hackathon/flowguard-supabase/blob/main/migrations/001_initial_schema.sql)
