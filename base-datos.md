# Base de datos y persistencia

Base de datos oficial: **Supabase / PostgreSQL**. Este documento es la fuente de verdad para nombres de tablas, columnas y estados; [contratos-api.md](contratos-api.md) debe mantenerse coherente con lo descrito aquí.

> **Aviso (2026-10-17):** la migración se renombró al español según el [glosario](glosario.md) y **no se ha vuelto a ejecutar**. Las verificaciones descritas aquí se hicieron con los nombres anteriores en inglés, así que la versión actual queda **pendiente de verificar** al aplicarla en el proyecto Supabase de desarrollo (Dev 1). No se usa Docker ni otra herramienta fuera del stack documentado.

> **Estado de verificación de la migración:** la migración de referencia ([`supabase/migrations/001_esquema_inicial.sql`](https://github.com/FlowGuard-Hackathon/flowguard-supabase/blob/main/migrations/001_esquema_inicial.sql)), incluidas las cuatro funciones de § 5 y § 2.4, se ejecutó contra un PostgreSQL 16 genérico en contenedores Docker desechables, en tres rondas (una por cada auditoría de esta especificación). La ronda más reciente encontró y corrigió un error real (`RETURNING` ambiguo en `fn_finalizar_intento_investigacion`, por colisión de nombres entre columnas de tabla y salidas de la función) y, tras corregirlo, confirmó con datos reales: creación atómica del primer intento sin ventana `NO_INICIADA` huérfana; el bloqueo de `fn_programar_intento_investigacion` contra intentos duplicados; el token de reclamación de `fn_finalizar_intento_investigacion` rechazando a un worker que no es el dueño del intento; la propagación correcta a `ejecuciones` solo cuando el intento sigue vigente; el reconciliador de § 5.6 recuperando una ejecución `NO_INICIADA` simulada; el *reaper* de § 5.3 recuperando un intento `EN_CURSO` abandonado y propagando correctamente el resultado (a diferencia de la versión con dos `UPDATE` sueltos de la auditoría anterior); y el contador de *rate limiting* de § 2.4, bajo 20 solicitudes realmente concurrentes contra un límite de 10, permitiendo exactamente 10 y rechazando exactamente 10. **No** se ha ejecutado contra un proyecto Supabase real (que añade roles y extensiones propias de la plataforma) — ese paso queda para el repositorio de implementación.

## 1. Modelo de datos

| Tabla | Responsabilidad |
|---|---|
| `definiciones_flujo` | Definición del flujo/escenario disponible |
| `reglas_politica` | Configuración versionada de cada regla |
| `saldos_presupuesto` | Presupuesto simulado por categoría/ámbito |
| `solicitudes_pago` | Datos de la solicitud de pago tal como fue validada |
| `claves_idempotencia` | Control de unicidad e idempotencia de solicitudes de creación |
| `ejecuciones` | Ejecución: decisión, estado técnico y versión de política aplicada |
| `evaluaciones_reglas` | Resultado de cada una de las cinco reglas para una ejecución |
| `registros_evidencia` | Evidencias consultables que respaldan cada resultado de regla |
| `intentos_investigacion` | Historial de cada intento de investigación de IA |
| `informes_agente` | Informe estructurado producido por un intento de investigación |
| `contadores_limite_solicitudes` | Contador compartido de solicitudes por cliente/endpoint (§ 2.4 de [seguridad.md](seguridad.md#24-límite-de-solicitudes-rate-limiting)) |

No se agregan tablas de propósito empresarial (roles multiempresa, auditoría genérica, colas de eventos) que el MVP no necesita. `claves_idempotencia` e `intentos_investigacion` se agregan porque son indispensables para implementar correctamente la idempotencia (sección 5 de [contratos-api.md](contratos-api.md)) y el historial de reintentos de investigación (sección 7), que de otro modo no tendrían dónde persistirse.

### 1.1. `definiciones_flujo`

```sql
CREATE TABLE definiciones_flujo (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  nombre          TEXT NOT NULL,
  escenario_id   TEXT NOT NULL,
  version       TEXT NOT NULL,
  activa     BOOLEAN NOT NULL DEFAULT TRUE,
  creado_en    TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (escenario_id, version)
);

CREATE INDEX idx_definiciones_flujo_escenario_activa
  ON definiciones_flujo (escenario_id)
  WHERE activa;
```

### 1.2. `reglas_politica`

```sql
CREATE TABLE reglas_politica (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  regla_id       TEXT NOT NULL,            -- 'PAG-001' ... 'PAG-005'
  version_politica TEXT NOT NULL,           -- 'politica-pagos-v1'
  configuracion_json   JSONB NOT NULL,
  habilitada       BOOLEAN NOT NULL DEFAULT TRUE,
  creado_en    TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (regla_id, version_politica)
);

CREATE INDEX idx_reglas_politica_version ON reglas_politica (version_politica);
```

Conservar todas las versiones históricas (no se eliminan filas): una ejecución pasada debe poder reconstruirse con la configuración vigente en el momento en que se evaluó, mediante `ejecuciones.version_politica`.

### 1.3. `saldos_presupuesto`

```sql
CREATE TABLE saldos_presupuesto (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  presupuesto_id         TEXT NOT NULL UNIQUE,   -- 'PRE-OPERACIONES-01'
  disponible_centavos   BIGINT NOT NULL CHECK (disponible_centavos >= 0),
  version           INTEGER NOT NULL DEFAULT 1,
  actualizado_en        TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

`version` se usa para control de concurrencia optimista (`UPDATE ... WHERE presupuesto_id = $1 AND version = $2`) si el simulador llega a descontar presupuesto de forma concurrente. Para el MVP, el presupuesto se simula y se consulta como saldo disponible; no se mueve dinero real.

### 1.4. `solicitudes_pago`

```sql
CREATE TABLE solicitudes_pago (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pago_id      TEXT NOT NULL UNIQUE,     -- 'PAG-2026-001'
  monto_centavos    BIGINT NOT NULL CHECK (monto_centavos > 0),
  moneda        TEXT NOT NULL CHECK (moneda IN ('MXN')),
  categoria        TEXT NOT NULL,
  descripcion     TEXT,
  solicitante_id    TEXT NOT NULL,
  presupuesto_id       TEXT NOT NULL REFERENCES saldos_presupuesto (presupuesto_id),
  creado_en      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_solicitudes_pago_solicitante ON solicitudes_pago (solicitante_id);
```

`monto_centavos` es la cantidad en centavos enteros (ver [contratos-api.md § 2.1](contratos-api.md#21-importes)). Los adjuntos (`adjuntos`) se resuelven contra el contexto confiable del backend en tiempo de validación y se referencian desde `registros_evidencia`, no se duplican aquí.

### 1.5. `claves_idempotencia`

```sql
CREATE TABLE claves_idempotencia (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clave_idempotencia   TEXT NOT NULL,
  escenario_id       TEXT NOT NULL,
  hash_solicitud      TEXT NOT NULL,          -- hash determinista del contenido normalizado
  estado_ejecucion  TEXT NOT NULL DEFAULT 'RECIBIDA'
                     CHECK (estado_ejecucion IN
                       ('RECIBIDA','VALIDANDO','EVALUANDO','PERSISTIDA','FALLIDA')),
  ejecucion_id      UUID REFERENCES ejecuciones (id),
  creado_en        TIMESTAMPTZ NOT NULL DEFAULT now(),
  actualizado_en        TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (clave_idempotencia, escenario_id)
);
```

Esta tabla es la autoridad de idempotencia: se inserta una fila **antes** de evaluar las reglas, dentro de la misma transacción que reserva el intento. Una segunda solicitud concurrente con la misma clave falla por violación de `UNIQUE (clave_idempotencia, escenario_id)`, y el backend decide la respuesta (`200` con el resultado existente, o `409 IDEMPOTENCIA_EN_PROGRESO` / `409 CLAVE_IDEMPOTENCIA_REUTILIZADA`) según `hash_solicitud` y `estado_ejecucion`. Ver [contratos-api.md § 2.2](contratos-api.md#22-idempotencia).

### 1.6. `ejecuciones`

```sql
CREATE TABLE ejecuciones (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  codigo_ejecucion        TEXT NOT NULL UNIQUE,    -- 'EJE-2026-0001', visible en la API
  solicitud_pago_id    UUID NOT NULL REFERENCES solicitudes_pago (id),
  version_motor        TEXT NOT NULL,
  version_politica        TEXT NOT NULL,
  decision              TEXT NOT NULL CHECK (decision IN ('PERMITIR','BLOQUEAR')),
  estado_investigacion  TEXT NOT NULL DEFAULT 'NO_INICIADA'
                         CHECK (estado_investigacion IN
                           ('NO_INICIADA','PENDIENTE','EN_CURSO','COMPLETADA','PARCIAL','FALLIDA')),
  numero_ultimo_intento INTEGER NOT NULL DEFAULT 0,
  creado_en            TIMESTAMPTZ NOT NULL DEFAULT now(),
  actualizado_en            TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ejecuciones_creado_en ON ejecuciones (creado_en DESC);
CREATE INDEX idx_ejecuciones_decision ON ejecuciones (decision);
CREATE INDEX idx_ejecuciones_estado_investigacion ON ejecuciones (estado_investigacion);
```

**`ejecuciones` no tiene columna `estado_ejecucion`.** En la primera versión de este documento sí la tenía, con el mismo conjunto cerrado que `claves_idempotencia.estado_ejecucion`; era redundante y, peor, engañosa: como una fila en `ejecuciones` solo se crea dentro de la transacción que confirma la persistencia (paso 5 de § 3), esa columna **nunca podía observarse con un valor distinto de `'PERSISTIDA'`** en una fila real — `RECIBIDA`, `VALIDANDO`, `EVALUANDO` y `FALLIDA` jamás ocurrían ahí. Se eliminó para no sugerir una máquina de estados que la tabla no implementa.

- **Fuente de verdad de `decision`:** esta tabla, columna `decision`. Se escribe una sola vez, en el `INSERT`, y ninguna sentencia `UPDATE` de la aplicación la toca jamás.
- **Fuente de verdad del estado de la *creación* de la ejecución** (`RECIBIDA` → `VALIDANDO` → `EVALUANDO` → `PERSISTIDA`/`FALLIDA`): la tabla `claves_idempotencia` (§ 1.5), no `ejecuciones`. La existencia misma de una fila en `ejecuciones` ya *es* el hecho de que ese pipeline llegó a `PERSISTIDA`; no hace falta una columna que lo repita. El contrato de API (ver [contratos-api.md § 0](contratos-api.md#0-modelo-de-estados-resumen)) sigue exponiendo un campo `estado_ejecucion` en las respuestas JSON, pero es un valor **constante** (`"PERSISTIDA"`) calculado al servir la respuesta, no una columna almacenada — se documenta así explícitamente para no prometer transiciones que no existen en el dato.
- **Fuente de verdad del estado de *cada intento* de investigación:** `intentos_investigacion` (§ 1.9).
- **Fuente de verdad del estado *vigente* de la investigación** (el que ve el frontend sin tener que calcular un `MAX(numero_intento)` en cada lectura): la columna `estado_investigacion` de esta tabla, **derivada** de `intentos_investigacion` y mantenida al día mediante la regla de *fencing* de § 2.3. `numero_ultimo_intento` se guarda junto a ella por la misma razón: permite a cualquier `UPDATE` que intente escribir el resultado de un intento verificar, en una sola condición `WHERE`, si sigue siendo el intento más reciente antes de escribir. Si `estado_investigacion`/`numero_ultimo_intento` llegaran a corromperse por un bug, se reconstruyen de forma determinista con:

  ```sql
  SELECT numero_intento, estado
  FROM intentos_investigacion
  WHERE ejecucion_id = $1
  ORDER BY numero_intento DESC
  LIMIT 1;
  ```

### 1.7. `evaluaciones_reglas`

```sql
CREATE TABLE evaluaciones_reglas (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  ejecucion_id    UUID NOT NULL REFERENCES ejecuciones (id) ON DELETE CASCADE,
  regla_id         TEXT NOT NULL,
  estado          TEXT NOT NULL CHECK (estado IN ('CUMPLE','NO_CUMPLE','INDETERMINADO')),
  observado_json   JSONB NOT NULL,
  esperado_json   JSONB NOT NULL,
  mensaje         TEXT,
  severidad        TEXT NOT NULL DEFAULT 'BLOQUEANTE' CHECK (severidad IN ('BLOQUEANTE','INFORMATIVA')),
  creado_en      TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (ejecucion_id, regla_id)
);

CREATE INDEX idx_evaluaciones_reglas_ejecucion ON evaluaciones_reglas (ejecucion_id);
```

La restricción `UNIQUE (ejecucion_id, regla_id)` (ya prevista en la versión anterior de este documento) impide que una misma regla quede evaluada dos veces para la misma ejecución.

### 1.8. `registros_evidencia`

```sql
CREATE TABLE registros_evidencia (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  ref_evidencia    TEXT NOT NULL,           -- 'EV-0001', visible en la API
  ejecucion_id    UUID NOT NULL REFERENCES ejecuciones (id) ON DELETE CASCADE,
  tipo_evidencia   TEXT NOT NULL,           -- 'REGISTRO_APROBACION', 'REGISTRO_FACTURA', ...
  origen          TEXT NOT NULL,           -- de qué parte del contexto confiable proviene
  contenido_json    JSONB NOT NULL,
  creado_en      TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (ejecucion_id, ref_evidencia)
);

CREATE INDEX idx_registros_evidencia_ejecucion ON registros_evidencia (ejecucion_id);
```

### 1.9. `intentos_investigacion`

```sql
CREATE TABLE intentos_investigacion (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  codigo_intento      TEXT NOT NULL UNIQUE,     -- 'INT-2026-0002', visible en la API
  ejecucion_id      UUID NOT NULL REFERENCES ejecuciones (id) ON DELETE CASCADE,
  numero_intento    INTEGER NOT NULL,         -- 1, 2, 3... por ejecucion_id
  estado            TEXT NOT NULL CHECK (estado IN
                       ('PENDIENTE','EN_CURSO','COMPLETADA','PARCIAL','FALLIDA')),
  llamadas_herramientas_usadas   INTEGER NOT NULL DEFAULT 0,
  codigo_error        TEXT,                     -- código del catálogo de errores si FALLIDA
  reclamado_por        TEXT,                     -- identificador del proceso worker que lo tomó
  reclamado_en        TIMESTAMPTZ,              -- cuándo pasó a EN_CURSO; base para detectar abandono
  iniciado_en        TIMESTAMPTZ NOT NULL DEFAULT now(),
  completado_en      TIMESTAMPTZ,
  UNIQUE (ejecucion_id, numero_intento)
);

CREATE INDEX idx_intentos_investigacion_ejecucion ON intentos_investigacion (ejecucion_id);

-- Cola de trabajo: el worker escanea solo esta franja, nunca toda la tabla.
CREATE INDEX idx_intentos_investigacion_pendientes
  ON intentos_investigacion (iniciado_en)
  WHERE estado = 'PENDIENTE';

-- Detección de intentos abandonados (ver § 5.3).
CREATE INDEX idx_intentos_investigacion_en_curso_reclamados
  ON intentos_investigacion (reclamado_en)
  WHERE estado = 'EN_CURSO';
```

Cada llamada a `POST /api/v1/ejecuciones/:id/investigar` que resulta permitida (ver tabla de comportamiento en [contratos-api.md § 7](contratos-api.md#7-post-apiv1executionsidinvestigate)) crea una fila nueva con `numero_intento` incrementado, calculado y bloqueado dentro de la misma transacción que valida que no haya otro intento `PENDIENTE`/`EN_CURSO` (ver § 5.1). `ejecuciones.estado_investigacion`/`numero_ultimo_intento` se actualizan junto con cada transición de esta tabla, nunca por separado.

### 1.10. `informes_agente`

```sql
CREATE TABLE informes_agente (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  intento_id      UUID NOT NULL REFERENCES intentos_investigacion (id) ON DELETE CASCADE,
  proveedor        TEXT NOT NULL DEFAULT 'nebius',
  modelo           TEXT NOT NULL,
  version_prompt  TEXT NOT NULL,
  informe_json     JSONB NOT NULL,           -- contrato de contratos-api.md § 10
  creado_en      TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (intento_id)
);
```

Solo existe una fila de `informes_agente` por intento, y solo para intentos cuyo resultado final es `COMPLETADA` o `PARCIAL`. Un intento `FALLIDA` no produce fila en `informes_agente`; su información queda en `intentos_investigacion.codigo_error`.

## 2. Modelo de estados unificado

Tres campos independientes, cada uno con su propio conjunto cerrado de valores. Ningún valor se reutiliza entre campos con un significado distinto.

### 2.1. `decision`

| Valor | Significado |
|---|---|
| `PERMITIR` | Las cinco reglas pasaron |
| `BLOQUEAR` | Al menos una regla falló o quedó indeterminada |

Se asigna **una sola vez**, en el momento en que el motor evalúa la solicitud, y nunca cambia después. Ningún otro componente (incluida la IA) puede modificarlo.

### 2.2. `estado_ejecucion`

Esta máquina de estados vive exclusivamente en `claves_idempotencia.estado_ejecucion` (§ 1.5); `ejecuciones` no tiene esta columna (ver § 1.6 para la justificación).

| Valor | Significado | ¿Terminal? |
|---|---|---|
| `RECIBIDA` | La solicitud llegó y se reservó la clave de idempotencia | No |
| `VALIDANDO` | Se están validando el contrato y el contexto confiable | No |
| `EVALUANDO` | El motor está evaluando las cinco reglas | No |
| `PERSISTIDA` | La ejecución, la decisión y las cinco evaluaciones quedaron guardadas en una transacción confirmada; `claves_idempotencia.ejecucion_id` ya apunta a la fila de `ejecuciones` | **Sí** |
| `FALLIDA` | Un error técnico impidió completar el pipeline antes de persistir | No — admite reintento con la misma clave (ver [contratos-api.md § 2.2, Caso D](contratos-api.md#22-idempotencia)) |

**Transiciones permitidas:** `RECIBIDA → VALIDANDO → EVALUANDO → PERSISTIDA`, o una salida a `FALLIDA` desde cualquiera de los tres primeros estados. Desde `FALLIDA`, una nueva solicitud con la misma clave y el mismo `hash_solicitud` puede reiniciar el ciclo (`FALLIDA → RECIBIDA`); es la única transición que sale de un estado no exitoso. **Prohibido:** saltar directamente a `PERSISTIDA` sin pasar por `EVALUANDO`; transicionar desde `PERSISTIDA` hacia cualquier otro valor (es terminal en sentido estricto); reintentar automáticamente desde `FALLIDA` sin que llegue una nueva solicitud del cliente.

**Quién cambia este campo:** exclusivamente el orquestador de ejecuciones del backend (`src/servidor/ejecuciones/`). Ni el frontend ni el agente de IA pueden escribirlo.

**Qué queda persistido si falla:** como `ejecuciones` solo recibe una fila al confirmar la transacción (paso 5 de § 3), un fallo antes de ese punto **no** deja una fila en `ejecuciones`. El único rastro persistido es la fila de `claves_idempotencia` con `estado_ejecucion = 'FALLIDA'`. El frontend recibe el error correspondiente del catálogo (ver [catalogo-errores.md](catalogo-errores.md)) y debe tratarlo como "la operación no se evaluó", nunca como una decisión de negocio.

### 2.3. `estado_investigacion`

| Valor | Significado | ¿Terminal? |
|---|---|---|
| `NO_INICIADA` | Valor por defecto interno antes de que `fn_programar_intento_investigacion` cree el primer intento dentro de la misma transacción ([§ 3](#3-persistencia-segura)); **no debería observarse en una fila confirmada bajo operación normal**. Si aparece, el reconciliador de [§ 5.6](#56-reconciliador-de-ejecuciones-no_iniciada-defensa-en-profundidad) lo recupera | No |
| `PENDIENTE` | Se creó un intento y está en cola para ejecutarse | No |
| `EN_CURSO` | El agente está consultando herramientas y generando el informe | No |
| `COMPLETADA` | El intento más reciente produjo un informe completo y validado | Sí (sin reintento automático) |
| `PARCIAL` | El intento más reciente produjo un informe útil pero incompleto | No — admite reintento manual |
| `FALLIDA` | El intento más reciente no produjo un informe utilizable | No — admite reintento manual |

**Transiciones permitidas:** `NO_INICIADA → PENDIENTE → EN_CURSO → {COMPLETADA | PARCIAL | FALLIDA}`. Desde `FALLIDA` o `PARCIAL`, una nueva llamada a `POST /investigar` crea un intento adicional y vuelve a `PENDIENTE → EN_CURSO → ...`. Un caso especial de `EN_CURSO → FALLIDA` ocurre sin que el agente responda: el *reaper* descrito en § 5.3 lo fuerza cuando un intento queda abandonado. **Prohibido:** reintentar automáticamente desde `COMPLETADA` (ver tabla de comportamiento en [contratos-api.md § 7](contratos-api.md#7-post-apiv1executionsidinvestigate)); transicionar sin pasar por `PENDIENTE`/`EN_CURSO`; que dos intentos del mismo `ejecucion_id` estén en `PENDIENTE`/`EN_CURSO` a la vez (impedido por el bloqueo de § 5.1).

**Quién cambia este campo:** exclusivamente `fn_programar_intento_investigacion` (al crear el primer intento o un reintento, invocada desde la transacción de creación, desde `POST /investigar`, o desde el reconciliador) y `fn_finalizar_intento_investigacion`/`fn_recuperar_intentos_abandonados` (al resolverlo o recuperarlo) — ver § 5. Nunca el modelo de Nebius directamente ni el frontend. El frontend solo puede *solicitar* una transición llamando a `POST /investigar`; el backend decide si es válida.

**Cómo se representan los errores:** un intento `FALLIDA` registra `codigo_error` en `intentos_investigacion` (código del catálogo: `TIEMPO_AGOTADO_PROVEEDOR`, `RESPUESTA_INVALIDA_PROVEEDOR`, `REFERENCIA_EVIDENCIA_INVALIDA`, `PROCESO_PERDIDO`, etc. — ver [catalogo-errores.md](catalogo-errores.md)). `decision` no se ve afectada en ningún caso.

**Qué respuesta recibe el frontend:** siempre `decision` (inmutable) del registro persistido en `ejecuciones`, un `estado_ejecucion` constante `"PERSISTIDA"` (ver § 2.2), más el `estado_investigacion` vigente derivado del intento más reciente. Nunca una respuesta que combine una decisión no persistida con un estado de investigación avanzado.

**Un intento antiguo nunca sobrescribe un reintento más reciente** (requisito explícito del hackathon): toda escritura de resultado sobre `intentos_investigacion` y toda propagación hacia `ejecuciones.estado_investigacion` está condicionada por `numero_intento`/`numero_ultimo_intento`, no solo por el `id` del intento. El mecanismo exacto está en § 5.4.

## 3. Persistencia segura

La versión anterior de este documento dividía esto en dos transacciones (persistir la decisión, y luego programar la investigación por separado), con el argumento de que un fallo al programar la investigación no debía revertir la decisión. Esa separación dejaba una ventana real: si la segunda transacción fallaba, la ejecución quedaba en `estado_investigacion = 'NO_INICIADA'` **sin ningún trabajo pendiente que un worker pudiera reclamar**, y el *reaper* de § 5.3 no la detectaba porque solo vigila intentos que llegaron a `EN_CURSO` — un `NO_INICIADA` sin intento asociado es invisible para él. Esta sección corrige eso fusionando la creación del primer intento en la misma transacción que persiste la decisión (§ 3, abajo), y añade un reconciliador como defensa adicional para cualquier fila que, por una razón no prevista (migración de datos, bug futuro), quede en ese estado igualmente (§ 5.6).

La secuencia para `POST /api/v1/ejecuciones` es ahora **una sola transacción** para todo lo que debe quedar atómico junto con la decisión:

1. Validar la solicitud y la identidad (`claves_idempotencia.estado_ejecucion` conceptual: `VALIDANDO`).
2. Reservar/reutilizar la clave de idempotencia en `claves_idempotencia` según el algoritmo de [contratos-api.md § 2.2](contratos-api.md#22-idempotencia) (`hash_solicitud`, `estado_ejecucion = 'RECIBIDA'`, o reactivación desde `FALLIDA`).
3. Obtener el contexto confiable (presupuesto, permisos, adjuntos).
4. Evaluar las cinco reglas (`estado_ejecucion` conceptual: `EVALUANDO`).
5. **En una única transacción:** insertar `solicitudes_pago`; insertar `ejecuciones` (`estado_investigacion = 'NO_INICIADA'`, `numero_ultimo_intento = 0`); insertar las cinco filas de `evaluaciones_reglas` y las filas de `registros_evidencia` correspondientes; actualizar `claves_idempotencia.estado_ejecucion = 'PERSISTIDA'` y `claves_idempotencia.ejecucion_id`; **y, como última sentencia de la misma transacción, invocar `fn_programar_intento_investigacion(ejecucion_id)`** (§ 5.1) para crear el primer intento (`numero_intento = 1`, `estado = 'PENDIENTE'`) y dejar `ejecuciones.estado_investigacion = 'PENDIENTE'`, `numero_ultimo_intento = 1`.
6. Confirmar la transacción.

Si cualquier paso de la transacción falla — incluida la creación del primer intento — **todo** se revierte: no queda fila en `ejecuciones` ni en `intentos_investigacion`, solo `claves_idempotencia.estado_ejecucion = 'FALLIDA'`. La API responde `500 FALLO_PERSISTENCIA` y nunca comunica una autorización satisfactoria. Esto es deliberado y es la razón por la que fusionar ambas operaciones es seguro: crear la fila `PENDIENTE` en `intentos_investigacion` es trabajo puramente de base de datos (ningún `INSERT` llama a Nebius ni a ningún servicio externo), así que no hay ningún motivo legítimo para que tenga éxito la persistencia de la decisión y falle por separado la creación del intento — si la base de datos está disponible para lo primero, lo está para lo segundo, en la misma sentencia.

**Qué significa esto para `estado_investigacion = 'NO_INICIADA'`:** bajo operación normal, ninguna ejecución `PERSISTIDA` debería observarse jamás en `NO_INICIADA` — el estado pasa a existir únicamente como un valor transitorio interno a la transacción, nunca confirmado de forma visible sin su `PENDIENTE` correspondiente. Se conserva en el `CHECK` de la columna (§ 1.6) por compatibilidad hacia atrás y como valor de entrada por defecto dentro de la función, no porque se espere observarlo en una fila confirmada. Si aparece en una fila real, es la señal inequívoca de un bug o de datos migrados desde una versión anterior de este esquema — exactamente el caso que cubre el reconciliador de § 5.6.

## 4. Conservación y eliminación de registros

Para el MVP no se implementa purga automática: todas las filas de `ejecuciones`, `evaluaciones_reglas`, `registros_evidencia`, `intentos_investigacion` y `informes_agente` se conservan indefinidamente, porque son la base de la trazabilidad exigida en [seguridad.md § 2.3](seguridad.md#23-evidencia-y-trazabilidad). Si en el futuro se requiere una política de retención, debe definirse por separado y no mezclarse con esta especificación del MVP.

## 5. Concurrencia y recuperación del mecanismo asíncrono de investigación

Esta sección da las sentencias exactas para las garantías exigidas al mecanismo asíncrono elegido (justificación arquitectónica completa en [arquitectura.md § 7](arquitectura.md#7-mecanismo-de-investigación-asíncrona)): un worker persistente que reclama trabajos de `intentos_investigacion` mediante bloqueo a nivel de fila, con recuperación tras reinicio porque el estado de cada intento vive en PostgreSQL, no en memoria de proceso.

### 5.1. Crear un intento sin duplicar investigaciones en curso

Esta lógica vive en **una sola función PL/pgSQL**, `fn_programar_intento_investigacion`, para que la transacción A ([§ 3](#3-persistencia-segura)), `POST /investigar` y el reconciliador ([§ 5.6](#56-reconciliador-de-ejecuciones-no_iniciada-defensa-en-profundidad)) compartan exactamente el mismo algoritmo en vez de reimplementarlo tres veces:

```sql
CREATE OR REPLACE FUNCTION fn_programar_intento_investigacion(p_ejecucion_id UUID)
RETURNS TABLE(resultado TEXT, intento_id UUID, numero_intento INTEGER)
LANGUAGE plpgsql AS $$
DECLARE
  v_estado TEXT;
  v_ultimo INTEGER;
  v_intento_id UUID;
  v_siguiente_intento INTEGER;
BEGIN
  SELECT estado_investigacion, numero_ultimo_intento
  INTO v_estado, v_ultimo
  FROM ejecuciones
  WHERE id = p_ejecucion_id
  FOR UPDATE;

  IF NOT FOUND THEN
    RETURN QUERY SELECT 'EJECUCION_NO_ENCONTRADA', NULL::UUID, NULL::INTEGER;
    RETURN;
  END IF;

  IF v_estado IN ('PENDIENTE', 'EN_CURSO') THEN
    RETURN QUERY SELECT 'EN_PROGRESO', NULL::UUID, NULL::INTEGER;
    RETURN;
  END IF;

  IF v_estado = 'COMPLETADA' THEN
    RETURN QUERY SELECT 'YA_COMPLETADA', NULL::UUID, NULL::INTEGER;
    RETURN;
  END IF;

  -- v_estado aquí solo puede ser NO_INICIADA, FALLIDA o PARCIAL: los tres son elegibles.
  v_siguiente_intento := v_ultimo + 1;

  INSERT INTO intentos_investigacion (codigo_intento, ejecucion_id, numero_intento, estado)
  VALUES ('INV-ADJ-' || lpad(v_siguiente_intento::TEXT, 4, '0') || '-' || substr(p_ejecucion_id::TEXT, 1, 8),
          p_ejecucion_id, v_siguiente_intento, 'PENDIENTE')
  RETURNING id INTO v_intento_id;

  UPDATE ejecuciones
  SET estado_investigacion = 'PENDIENTE', numero_ultimo_intento = v_siguiente_intento
  WHERE id = p_ejecucion_id;

  RETURN QUERY SELECT 'CREADO', v_intento_id, v_siguiente_intento;
END;
$$;
```

**Condiciones de elegibilidad** (la razón por la que se puede llamar sin riesgo desde tres lugares distintos): un `ejecucion_id` es elegible para un nuevo intento si y solo si su `estado_investigacion` actual es `NO_INICIADA`, `FALLIDA` o `PARCIAL`. `PENDIENTE`/`EN_CURSO` devuelven `EN_PROGRESO` (nada se crea) y `COMPLETADA` devuelve `YA_COMPLETADA` (nada se crea). El llamador traduce `resultado` a la respuesta HTTP correspondiente (`CREADO` → `202`/`201` según el caso; `EN_PROGRESO` → `409 INVESTIGACION_EN_CURSO`; `YA_COMPLETADA` → `409 INVESTIGACION_YA_COMPLETADA`; `EJECUCION_NO_ENCONTRADA` → `404`).

**Garantía de concurrencia:** el `SELECT ... FOR UPDATE` sobre `ejecuciones`, dentro de la función, serializa cualquier llamada concurrente para el mismo `ejecucion_id` — venga de `POST /investigar`, de otra invocación de `POST /investigar`, o del reconciliador compitiendo por la misma fila. La segunda llamada espera a que la primera libere el bloqueo (al terminar la función, dentro de la misma transacción del llamador) y entonces relee un `estado_investigacion` ya actualizado, devolviendo `EN_PROGRESO` en vez de crear un segundo intento. Esto resuelve de raíz tanto "dos intentos de investigación simultáneos" como "el worker, el reconciliador o un usuario programan la misma investigación a la vez": los tres llamadores pasan por el mismo bloqueo de fila, sin importar quién llegó primero.

**Por qué es una función y no una transacción ensamblada en el backend:** al ejecutarse como una sola sentencia `SELECT * FROM fn_programar_intento_investigacion($1)`, PostgreSQL garantiza que si cualquier paso interno lanza una excepción, **todos** sus efectos (el `SELECT FOR UPDATE`, el `INSERT`, el `UPDATE`) se revierten como una unidad, sin que el backend tenga que coordinar manualmente un `BEGIN`/`COMMIT` alrededor de varias llamadas de red. Es compatible con Supabase sin cambios: se invoca igual desde una conexión directa de `pg`/`postgres.js` que desde `supabase.rpc('fn_programar_intento_investigacion', { p_ejecucion_id })`.

### 5.2. El worker reclama un trabajo (`PENDIENTE → EN_CURSO`)

```sql
UPDATE intentos_investigacion
SET estado = 'EN_CURSO', reclamado_por = $proceso_id, reclamado_en = now()
WHERE id = (
  SELECT id FROM intentos_investigacion
  WHERE estado = 'PENDIENTE'
  ORDER BY iniciado_en
  FOR UPDATE SKIP LOCKED
  LIMIT 1
)
RETURNING id, ejecucion_id, numero_intento;
```

`FOR UPDATE SKIP LOCKED` es la razón por la que varias instancias del worker (o varios *ticks* del mismo worker) pueden ejecutar esta misma consulta en paralelo sin reclamar el mismo trabajo dos veces: la fila que ya está bloqueada por otra transacción se omite en vez de esperar. Inmediatamente después, en otra sentencia, se propaga a `ejecuciones`:

```sql
UPDATE ejecuciones
SET estado_investigacion = 'EN_CURSO'
WHERE id = $ejecucion_id AND estado_investigacion = 'PENDIENTE';
```

La cláusula `AND estado_investigacion = 'PENDIENTE'` es la primera aplicación de *fencing*: si por cualquier motivo `ejecuciones` ya no refleja `PENDIENTE` (por ejemplo, el reaper ya actuó), esta actualización no tiene efecto y el worker debe revisar el estado antes de continuar.

**`reclamado_por` debe ser un identificador fresco por arranque del proceso** (por ejemplo, un UUID generado una vez al iniciar el worker, no simplemente el hostname), nunca un valor fijo reutilizado entre reinicios. La razón es la base del token de reclamación que usa `fn_finalizar_intento_investigacion` (§ 5.4): si un worker muere a mitad de un intento y se reinicia, su nueva instancia debe tener un `reclamado_por` distinto al de la instancia anterior, para que nunca pueda "reclamar como propio" — y finalizar — un trabajo que en realidad quedó abandonado y le corresponde resolver al *reaper* (§ 5.3), no a una coincidencia accidental de identificador.

### 5.3. El *reaper*: recuperar trabajos abandonados

Un intento puede quedar en `EN_CURSO` sin resolverse si el proceso worker muere, se reinicia o pierde conexión a mitad de una llamada a Nebius. La versión anterior de esta sección ejecutaba la recuperación como dos sentencias `UPDATE` sueltas desde la aplicación, con el mismo defecto que se corrigió en § 5.4: si el proceso que ejecuta el reaper muere **entre** la primera y la segunda sentencia, `intentos_investigacion` queda `FALLIDA` pero `ejecuciones.estado_investigacion` se queda `EN_CURSO` para siempre — ningún reaper futuro vuelve a encontrar esa fila, porque ya no está en `EN_CURSO`. Por eso esto también es una función atómica:

```sql
CREATE OR REPLACE FUNCTION fn_recuperar_intentos_abandonados(p_umbral_abandono_ms BIGINT)
RETURNS TABLE(intento_id UUID, ejecucion_id UUID, numero_intento INTEGER)
LANGUAGE plpgsql AS $$
DECLARE
  r RECORD;
BEGIN
  FOR r IN
    UPDATE intentos_investigacion
    SET estado = 'FALLIDA', codigo_error = 'PROCESO_PERDIDO', completado_en = now()
    WHERE estado = 'EN_CURSO'
      AND reclamado_en < now() - (p_umbral_abandono_ms || ' milliseconds')::interval
    RETURNING id, intentos_investigacion.ejecucion_id, intentos_investigacion.numero_intento
  LOOP
    UPDATE ejecuciones
    SET estado_investigacion = 'FALLIDA'
    WHERE id = r.ejecucion_id
      AND numero_ultimo_intento = r.numero_intento
      AND estado_investigacion = 'EN_CURSO';

    intento_id := r.id;
    ejecucion_id := r.ejecucion_id;
    numero_intento := r.numero_intento;
    RETURN NEXT;
  END LOOP;
END;
$$;
```

Toda la función corre como una sola transacción: si el proceso que la invocó muere a mitad de la ejecución (p. ej. el propio worker, que también hace de reaper, se cae justo durante este ciclo), PostgreSQL revierte **todo** lo que alcanzó a hacer — ninguna fila queda marcada `FALLIDA` en `intentos_investigacion` sin su propagación correspondiente a `ejecuciones`. La próxima vez que cualquier instancia del worker ejecute `SELECT * FROM fn_recuperar_intentos_abandonados($1)`, las mismas filas (que siguen en `EN_CURSO` porque la transacción anterior no confirmó nada) vuelven a ser candidatas, sin ninguna inconsistencia acumulada.

`p_umbral_abandono_ms` se calcula como un múltiplo de `AGENTE_TIEMPO_LIMITE_MS` más un margen fijo (recomendado: `2 × AGENTE_TIEMPO_LIMITE_MS + 10000`), para no marcar como abandonado un intento que simplemente está cerca del límite normal. Un intento recuperado así queda en `FALLIDA` con `codigo_error = 'PROCESO_PERDIDO'` (ver [catalogo-errores.md](catalogo-errores.md)) y es reintentable mediante `POST /investigar` como cualquier otro `FALLIDA`.

### 5.4. *Fencing*: finalización atómica de un intento

La versión anterior de esta sección describía el *fencing* como dos sentencias `UPDATE` separadas que el worker ejecutaba una tras otra desde la aplicación. Eso deja una ventana real: si el worker falla **entre** la primera y la segunda sentencia (por ejemplo, se cae justo después de marcar el intento `COMPLETADA` pero antes de insertar el informe o de propagar el estado a `ejecuciones`), el intento queda en un estado terminal sin su informe, o `ejecuciones.estado_investigacion` no llega a reflejarlo — una inconsistencia real que una "transacción" ensamblada a partir de llamadas HTTP independientes no evita, porque agrupar llamadas de red en una transacción de aplicación no les da atomicidad real en PostgreSQL si el proceso muere a mitad de camino entre una llamada y la siguiente.

La solución es la misma que en § 5.1: una **única función PL/pgSQL** que hace todo el trabajo de finalización como una sola sentencia, con un token de reclamación explícito (`reclamado_por`) para verificar que quien finaliza es efectivamente quien reclamó el trabajo:

```sql
CREATE OR REPLACE FUNCTION fn_finalizar_intento_investigacion(
  p_intento_id      UUID,
  p_reclamado_por      TEXT,          -- token de reclamación: debe coincidir con intentos_investigacion.reclamado_por
  p_estado_final    TEXT,          -- 'COMPLETADA' | 'PARCIAL' | 'FALLIDA'
  p_llamadas_herramientas_usadas INTEGER,
  p_codigo_error      TEXT DEFAULT NULL,
  p_proveedor        TEXT DEFAULT 'nebius',
  p_modelo           TEXT DEFAULT NULL,
  p_version_prompt  TEXT DEFAULT NULL,
  p_informe_json     JSONB DEFAULT NULL
)
RETURNS TABLE(aplicado BOOLEAN, propagado BOOLEAN, ejecucion_id UUID, numero_intento INTEGER)
LANGUAGE plpgsql AS $$
DECLARE
  v_ejecucion_id UUID;
  v_numero_intento INTEGER;
  v_filas INTEGER;
BEGIN
  IF p_estado_final NOT IN ('COMPLETADA', 'PARCIAL', 'FALLIDA') THEN
    RAISE EXCEPTION 'estado terminal inválido: %', p_estado_final;
  END IF;

  -- Paso 1: reclamar la finalización, solo si el intento sigue EN_CURSO y pertenece a este worker.
  -- Las columnas se califican con el nombre de la tabla porque coinciden con
  -- los nombres de salida de la función (RETURNS TABLE) — sin calificar,
  -- PostgreSQL rechaza el RETURNING por ambiguo (se detectó exactamente este
  -- error al probar la función contra PostgreSQL real; ver nota de
  -- verificación al inicio de este documento).
  UPDATE intentos_investigacion
  SET estado = p_estado_final,
      llamadas_herramientas_usadas = p_llamadas_herramientas_usadas,
      codigo_error = p_codigo_error,
      completado_en = now()
  WHERE id = p_intento_id
    AND estado = 'EN_CURSO'
    AND reclamado_por = p_reclamado_por
  RETURNING intentos_investigacion.ejecucion_id, intentos_investigacion.numero_intento
  INTO v_ejecucion_id, v_numero_intento;

  IF NOT FOUND THEN
    -- El reaper ya lo recuperó, otro worker lo reclamó, o el intento ya no existe: no se hace nada más.
    RETURN QUERY SELECT FALSE, FALSE, NULL::UUID, NULL::INTEGER;
    RETURN;
  END IF;

  -- Paso 2: persistir el informe, solo si el resultado es COMPLETADA o PARCIAL.
  IF p_estado_final IN ('COMPLETADA', 'PARCIAL') THEN
    INSERT INTO informes_agente (intento_id, proveedor, modelo, version_prompt, informe_json)
    VALUES (p_intento_id, p_proveedor, p_modelo, p_version_prompt, p_informe_json);
  END IF;

  -- Paso 3: propagar a ejecuciones solo si este intento sigue siendo el más reciente (fencing).
  UPDATE ejecuciones
  SET estado_investigacion = p_estado_final
  WHERE id = v_ejecucion_id AND numero_ultimo_intento = v_numero_intento;

  GET DIAGNOSTICS v_filas = ROW_COUNT;

  RETURN QUERY SELECT TRUE, (v_filas > 0), v_ejecucion_id, v_numero_intento;
END;
$$;
```

**Cómo satisface cada requisito:**

- **Verifica que el intento siga perteneciendo al worker autorizado y en `EN_CURSO`:** la cláusula `WHERE id = p_intento_id AND estado = 'EN_CURSO' AND reclamado_por = p_reclamado_por` del paso 1. `p_reclamado_por` es el mismo valor que el worker usó para reclamar el trabajo en § 5.2 — actúa como un *token de reclamación*: debe generarse de nuevo (no reutilizar un valor fijo como el hostname) cada vez que arranca un proceso worker, precisamente para que, si un worker muere y se reinicia, la nueva instancia **no** herede accidentalmente la propiedad de un trabajo que la instancia anterior había reclamado antes de morir — esa recuperación es responsabilidad exclusiva del *reaper* (§ 5.3), nunca de una coincidencia de identificador.
- **Verifica que el intento siga siendo el más reciente:** la cláusula `AND numero_ultimo_intento = v_numero_intento` del paso 3 — igual que en la versión anterior, pero ahora dentro de la misma función.
- **Resultado tardío de Nebius sobre un intento ya recuperado por el *reaper*:** si el *reaper* (§ 5.3) ya puso `estado = 'FALLIDA'` en ese intento, el paso 1 de esta función no encuentra ninguna fila que actualizar (`estado = 'EN_CURSO'` ya no se cumple) y devuelve `aplicado = FALSE` sin ejecutar los pasos 2 y 3. El worker que recibe `aplicado = FALSE` descarta el resultado (lo registra en logs, nunca lo reintenta como si fuera un error nuevo) y no muestra ningún error al cliente — la ejecución ya refleja correctamente `FALLIDA` por abandono.
- **Nada queda parcialmente confirmado:** los tres pasos corren dentro del cuerpo de una sola función invocada como una sola sentencia SQL. Si el paso 2 (`INSERT INTO informes_agente`) lanza una excepción — por ejemplo, una violación de una restricción `NOT NULL` en `modelo` —, PostgreSQL revierte automáticamente **todo** lo que la función alcanzó a hacer, incluido el `UPDATE` del paso 1, porque una excepción no capturada dentro de una función aborta la transacción que la invoca. El backend nunca ve un intento marcado `COMPLETADA` sin su informe.
- **No requiere agrupar llamadas HTTP:** el backend hace una sola llamada (`SELECT * FROM fn_finalizar_intento_investigacion(...)` o el `.rpc()` equivalente de Supabase), no una secuencia de peticiones envueltas en una transacción de aplicación — la atomicidad existe en PostgreSQL, no en el código que la invoca.
- **Historial sin sobrescritura:** un intento antiguo que termina tarde (`aplicado = TRUE` porque todavía estaba `EN_CURSO`, pero `propagado = FALSE` porque ya no es el más reciente) queda correctamente registrado en `intentos_investigacion`/`informes_agente` para el historial, sin pisar el `estado_investigacion` vigente de `ejecuciones`. Esto solo puede ocurrir si ese intento ya había terminado antes en `FALLIDA`/`PARCIAL` y alguien pidió un reintento (dado el bloqueo de § 5.1), así que es un caso legítimo, no un error.

### 5.5. Dónde corre el worker (supuesto de despliegue)

El mecanismo anterior asume que existe al menos un proceso Node.js de larga duración ejecutando un bucle de sondeo (`setInterval`/bucle `async` con espera) que repite § 5.2–5.3 cada `AGENTE_INTERVALO_SONDEO_MS`. Ver la justificación de esta elección, el supuesto de hosting concreto y la alternativa para plataformas estrictamente *serverless* en [arquitectura.md § 7](arquitectura.md#7-mecanismo-de-investigación-asíncrona).

### 5.6. Reconciliador de ejecuciones `NO_INICIADA` (defensa en profundidad)

Con el cambio de § 3, una ejecución `PERSISTIDA` nunca debería quedar en `estado_investigacion = 'NO_INICIADA'` bajo operación normal: el primer intento se crea en la misma transacción que la decisión. Este reconciliador existe para el residuo de riesgo que esa garantía no cubre — principalmente, filas heredadas de una versión anterior de este esquema (antes de este cambio) o un bug futuro que reintroduzca el problema — no como el mecanismo principal de prevención.

**Algoritmo**, ejecutado en el mismo bucle del worker (§ 5.5), en un ciclo propio y mucho menos frecuente que el sondeo de `PENDIENTE` (recomendado: cada `RECONCILIADOR_INTERVALO_MS`, por defecto 60000 — un orden de magnitud más espaciado que `AGENTE_INTERVALO_SONDEO_MS`, porque en el caso normal esta consulta no debe encontrar ninguna fila):

```sql
SELECT id
FROM ejecuciones
WHERE estado_investigacion = 'NO_INICIADA'
  AND creado_en < now() - interval '30 seconds'
ORDER BY creado_en
LIMIT 20;
```

**Condición de elegibilidad:** `estado_investigacion = 'NO_INICIADA'` y `creado_en` anterior a un margen de gracia (30 segundos recomendado) — el margen evita que el reconciliador compita innecesariamente con la transacción de creación en el instante en que todavía está en curso bajo carga alta; no es una ventana de inconsistencia esperada, es solo para no disparar falsos positivos bajo latencia normal.

Por cada fila candidata, el reconciliador llama a la **misma función** que todo lo demás, sin lógica propia de creación:

```sql
SELECT * FROM fn_programar_intento_investigacion($ejecucion_id);
```

Como esa función ya es idempotente y segura ante concurrencia (§ 5.1, mediante `SELECT ... FOR UPDATE`), no hace falta ninguna coordinación adicional entre el reconciliador y una llamada simultánea a `POST /investigar` sobre la misma fila: cualquiera que llegue primero gana el bloqueo, crea el intento, y el otro recibe `EN_PROGRESO` sin duplicar nada. Esto responde directamente al requisito de "evitar intentos duplicados cuando el worker, el reconciliador o un usuario intentan programar la misma investigación simultáneamente": los tres comparten el mismo punto de entrada atómico.

**Por qué un reconciliador por sondeo y no un *outbox* transaccional separado:** un *outbox* transaccional clásico necesitaría una tabla intermedia y un proceso relator que la lea y publique el trabajo "real" — exactamente la responsabilidad que `intentos_investigacion` ya cumple. Separar eso en una tabla de *outbox* adicional sería introducir una pieza más sin necesidad: aquí la prevención (§ 3, crear el intento en la misma transacción que la decisión) ya cubre el caso normal con garantías más fuertes que cualquier *outbox* asíncrono, y el reconciliador cubre el residuo. Para un hackathon de 15 días, dos piezas (prevención atómica + reconciliador de baja frecuencia) sobre la infraestructura que ya existe es más simple y más confiable que construir y operar un patrón *outbox* completo para un caso que, bien prevenido, debería ocurrir cero veces.

## Referencias

- Contratos JSON que se persisten en estas tablas: [contratos-api.md](contratos-api.md)
- Reglas que generan `evaluaciones_reglas`: [reglas.md](reglas.md)
- Controles de acceso a estas tablas (RLS, roles de servicio): [seguridad.md](seguridad.md)
- Mecanismo asíncrono: justificación, hosting y alternativa serverless: [arquitectura.md § 7](arquitectura.md#7-mecanismo-de-investigación-asíncrona)
- Migración SQL de referencia: [`supabase/migrations/001_esquema_inicial.sql`](https://github.com/FlowGuard-Hackathon/flowguard-supabase/blob/main/migrations/001_esquema_inicial.sql)
