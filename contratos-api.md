# Contratos de datos y API

Los contratos JSON son la interfaz oficial entre los módulos. Los ejemplos de este documento son fixtures de demostración; los valores reales deben proceder del backend.

> Este documento asume el modelo de estados unificado descrito en [base-datos.md § 2](base-datos.md#2-modelo-de-estados-unificado). Si hay alguna discrepancia entre ambos documentos, **base-datos.md es la fuente de verdad** para nombres y transiciones de estado, porque describe lo que efectivamente se persiste.

## 0. Modelo de estados (resumen)

FlowGuard distingue tres conceptos independientes que nunca deben colapsarse en un solo campo:

| Campo | Significa | Quién lo cambia |
|---|---|---|
| `decision` | Resultado de negocio de las reglas: `PERMITIR` o `BLOQUEAR` | Exclusivamente el motor de reglas, una sola vez por ejecución |
| `estado_ejecucion` | Progreso técnico del pipeline de evaluación/persistencia | Exclusivamente el backend (orquestador de ejecuciones) |
| `estado_investigacion` | Progreso técnico del trabajo de IA | Exclusivamente el backend (orquestador de investigación y worker asíncrono), nunca el modelo directamente |

> **Nota sobre `estado_ejecucion` en las respuestas de este documento:** toda ejecución que la API puede devolver ya está persistida (ver [base-datos.md § 1.6](base-datos.md#16-executions): una fila de `ejecuciones` solo existe una vez confirmada la transacción). Por eso `estado_ejecucion` aparece siempre como el valor constante `"PERSISTIDA"` en cualquier respuesta exitosa — no es una columna que cambie en la base, es un campo calculado al servir la respuesta. El resto del ciclo de vida (`RECIBIDA`/`VALIDANDO`/`EVALUANDO`/`FALLIDA`) ocurre antes de que exista una ejecución que consultar, y solo es observable en el momento mismo de la solicitud fallida, vía el código de error correspondiente (ver [catalogo-errores.md](catalogo-errores.md)), nunca a través de un `GET` posterior.

Valores cerrados:

- `decision`: `PERMITIR`, `BLOQUEAR`
- `estado_ejecucion`: `RECIBIDA`, `VALIDANDO`, `EVALUANDO`, `PERSISTIDA`, `FALLIDA`
- `estado_investigacion`: `NO_INICIADA`, `PENDIENTE`, `EN_CURSO`, `COMPLETADA`, `PARCIAL`, `FALLIDA`

`COMPLETADA` nunca se usa para `estado_ejecucion` ni para `decision`; solo existe como valor de `estado_investigacion`. `PERMITIR`/`BLOQUEAR` nunca se usan como estados técnicos. El detalle completo de transiciones está en [base-datos.md § 2](base-datos.md#2-modelo-de-estados-unificado).

## 1. Formato uniforme de error

Toda respuesta de error (4xx/5xx) sigue este esquema, sin excepción:

```json
{
  "error": {
    "codigo": "ERROR_VALIDACION",
    "mensaje": "El campo pago.monto debe ser positivo y finito.",
    "solicitud_id": "req_01HZXYZ...",
    "detalles": {
      "campo": "pago.monto"
    }
  }
}
```

- `codigo`: identificador estable en mayúsculas, ver el catálogo completo en [catalogo-errores.md](catalogo-errores.md).
- `mensaje`: texto apto para mostrar o registrar, sin datos sensibles.
- `solicitud_id`: identificador de correlación generado por el backend en cada solicitud; debe incluirse en logs.
- `detalles`: objeto opcional, seguro de exponer. Nunca incluye stack traces, SQL, secretos ni respuestas crudas de Nebius.

Un error técnico posterior a la persistencia de la decisión **nunca** cambia `decision`. Ver [catalogo-errores.md](catalogo-errores.md) para el efecto de cada código sobre `decision`.

## 2. Conversión e idempotencia (reglas transversales)

### 2.1. Importes

- El contrato público (`pago.monto`) se expresa en **pesos mexicanos (MXN)**, con hasta 2 decimales.
- PostgreSQL almacena los importes como enteros en centavos (`monto_centavos`, tipo `BIGINT`), nunca como `float`/`numeric` de punto flotante binario.
- La conversión se realiza mediante una única función centralizada, por ejemplo `aCentavos(montoMXN: number): bigint` en `src/compartido/dinero.ts`. Ningún otro módulo implementa su propia conversión.
- Política: se **rechaza** (no se redondea silenciosamente) cualquier importe con más de 2 decimales. No se admiten importes `NaN`, `Infinity`, negativos o iguales a cero.
- Límite práctico para el MVP: `0.01` a `10,000,000.00` MXN. Un importe fuera de este rango se rechaza con `ERROR_VALIDACION`.

### 2.2. Idempotencia

`evento.clave_idempotencia` es obligatoria en `POST /api/v1/ejecuciones`. **Ámbito de unicidad:** `(clave_idempotencia, escenario_id)`, único en la tabla `claves_idempotencia` (ver [base-datos.md § 1.5](base-datos.md#15-idempotency_keys)). Esto permite reutilizar la misma clave en distintos escenarios de demo sin colisionar.

#### Hash del contenido (`hash_solicitud`)

El backend calcula un hash determinista del contenido semántico del request para distinguir una repetición legítima de una reutilización indebida de la clave:

1. **Campos incluidos:** todo el cuerpo del request **excepto** `evento.clave_idempotencia` (incluye `version_esquema`, `escenario_id`, `evento.tipo`, y todo el objeto `pago` y `contexto`). `clave_idempotencia` no participa en el hash porque es la clave de búsqueda misma; incluirla lo haría circular.
2. **Normalización antes de calcular el hash:**
   - Se aplican primero los valores por defecto de los campos opcionales (de modo que omitir un campo opcional produzca el mismo resultado que enviarlo explícitamente con su valor por defecto).
   - Los importes se normalizan a su representación canónica en centavos enteros (`monto_centavos`, ver § 2.1) antes de hashear, no al texto decimal recibido — así `"monto": 15000` y, si se permitiera, `"monto": 15000.00` producen el mismo hash.
   - Las claves de cada objeto se ordenan alfabéticamente de forma recursiva antes de serializar, para que el orden de las propiedades en el JSON recibido no afecte el resultado.
   - El resultado se serializa con `JSON.stringify` sobre la estructura ya canonicalizada y se calcula `SHA-256` en hexadecimal.
3. La función de normalización y hash es única y centralizada (junto a `aCentavos`, en `src/compartido/dinero.ts` o un módulo hermano), igual que la conversión monetaria: ningún otro módulo reimplementa esta lógica.

#### Casos cubiertos

**Caso A — misma clave, mismo contenido (mismo `hash_solicitud`):**
La API no vuelve a evaluar las reglas. Devuelve la ejecución ya existente. Si la fila ya existía antes de esta solicitud → `200 OK`. Si esta misma solicitud es la que efectivamente ganó la carrera y la creó → `201 Created`. El cuerpo de la respuesta es idéntico en ambos casos; solo cambia el código HTTP, según si esta llamada fue la creadora o no.

**Caso B — misma clave, contenido diferente (`hash_solicitud` distinto):**
`409 Conflict`, código `CLAVE_IDEMPOTENCIA_REUTILIZADA`. Se verifica el hash **antes** que cualquier otra condición (incluida si el intento original sigue en curso): una clave reutilizada con otro contenido siempre es `409 CLAVE_IDEMPOTENCIA_REUTILIZADA`, nunca `IDEMPOTENCIA_EN_PROGRESO`. No se crea una ejecución nueva ni se modifica la existente.

**Caso C — solicitud concurrente con la misma clave y mismo contenido, todavía procesándose** (`claves_idempotencia.estado_ejecucion` en `RECIBIDA`/`VALIDANDO`/`EVALUANDO`): `409 Conflict`, código `IDEMPOTENCIA_EN_PROGRESO`. **Política de cliente definida:** reintentar la misma solicitud `POST /ejecuciones` (idéntica, con la misma clave y el mismo contenido) con *backoff*; no existe un endpoint separado para consultar el estado de una clave de idempotencia antes de que exista un `ejecucion_id`. Cada reintento vuelve a evaluar los tres casos (A/B/C) contra el estado más reciente: cuando la solicitud original termine, el reintento recibirá `200` con el resultado (Caso A) o, si terminó en error técnico, podrá disparar una nueva creación (Caso D).

**Caso D — la primera solicitud falló antes de persistir** (`claves_idempotencia.estado_ejecucion = 'FALLIDA'`): una nueva solicitud con la misma clave y el mismo `hash_solicitud` **reinicia el ciclo** (`FALLIDA → RECIBIDA`, ver [base-datos.md § 2.2](base-datos.md#22-execution_status)) y evalúa las reglas de nuevo como si fuera la primera vez; no se trata como Caso A porque no hay ninguna ejecución previa que devolver. Responde `201 Created` si tiene éxito esta vez, o vuelve a `500 FALLO_PERSISTENCIA`/queda en `FALLIDA` si falla de nuevo. Una nueva solicitud con la misma clave pero `hash_solicitud` **distinto** del que falló sigue siendo Caso B (`409 CLAVE_IDEMPOTENCIA_REUTILIZADA`): un fallo previo no "libera" la clave para usarse con otro contenido.

**Caso E — la investigación de una ejecución ya persistida falló; se repite la creación:** si el cliente reenvía `POST /ejecuciones` con la misma clave y el mismo contenido de una ejecución cuya investigación está en `FALLIDA`/`PARCIAL`, el Caso A aplica normalmente (devuelve la ejecución existente, con su `estado_investigacion` actual). **La creación nunca vuelve a programar ni a duplicar la investigación**: el primer y único intento automático ocurre una vez, inmediatamente después de la persistencia original (transacción B de [base-datos.md § 3](base-datos.md#3-persistencia-segura)). Reintentar la investigación requiere una llamada explícita a `POST /ejecuciones/:id/investigar` (§ 7); `POST /ejecuciones` nunca la dispara de nuevo.

#### Restricción de base de datos

`UNIQUE (clave_idempotencia, escenario_id)` en `claves_idempotencia`, con el registro insertado (o reactivado desde `FALLIDA`, Caso D) dentro de la misma transacción que reserva el intento, antes de evaluar las reglas. Esto evita duplicados bajo solicitudes concurrentes: la segunda transacción que intenta insertar la misma clave falla por violación de unicidad (o, en el caso D, compite por el mismo `UPDATE` condicional) y el backend la traduce al caso correspondiente (B o C según el hash).

## 3. `POST /api/v1/ejecuciones`

**Objetivo:** crear y evaluar una ejecución de forma síncrona respecto a la decisión (las reglas se evalúan y persisten antes de responder); la investigación de IA se programa después de confirmar la persistencia y puede completarse de forma asíncrona.

**Autenticación:** requerida cuando el endpoint se expone fuera de un entorno de demo controlado. Ver [seguridad.md § 2](seguridad.md#2-estrategia-de-seguridad-por-entorno).

**Parámetros de ruta/query:** ninguno.

### Solicitud

```json
{
  "version_esquema": "1.0",
  "escenario_id": "aprobacion-pago-v1",
  "evento": {
    "tipo": "PAGO_SOLICITADO",
    "clave_idempotencia": "demo-pago-001"
  },
  "pago": {
    "pago_id": "PAG-2026-001",
    "monto": 15000,
    "moneda": "MXN",
    "categoria": "OPERACIONES",
    "descripcion": "Compra de equipo de oficina",
    "solicitado_por": {
      "usuario_id": "USU-001"
    },
    "adjuntos": [
      {
        "adjunto_id": "ADJ-001",
        "tipo": "FACTURA",
        "referencia": "FAC-2026-001"
      }
    ]
  },
  "contexto": {
    "presupuesto_id": "PRE-OPERACIONES-01"
  }
}
```

### Validación del backend

- `monto`: positivo, finito, máximo 2 decimales, dentro del rango admitido (ver § 2.1).
- `moneda`: pertenece al conjunto admitido (`MXN` para el MVP).
- `categoria`: valor reconocido por la política activa (ver [reglas.md](reglas.md)).
- `solicitado_por.usuario_id`: corresponde a un usuario existente en el contexto confiable.
- `adjuntos[].adjunto_id` / `referencia`: deben existir en el contexto confiable; el backend los resuelve, no confía en el `tipo` enviado por el cliente.
- `contexto.presupuesto_id`: debe existir y estar asociado a la categoría correspondiente.
- El cliente **no** puede establecer `estado_aprobacion`, roles, permisos de aprobador ni el resultado de verificación de factura: esos datos se resuelven exclusivamente contra el contexto confiable del backend, no contra el payload.
- `evento.clave_idempotencia`: ver § 2.2.

### Respuesta — éxito (`201 Created`, o `200 OK` si la idempotencia devuelve una ejecución existente)

```json
{
  "version_esquema": "1.0",
  "ejecucion_id": "EJE-2026-0001",
  "version_motor": "1.0.0",
  "version_politica": "politica-pagos-v1",
  "decision": "BLOQUEAR",
  "estado_ejecucion": "PERSISTIDA",
  "estado_investigacion": "PENDIENTE",
  "resumen": {
    "total_reglas": 5,
    "cumplidas": 4,
    "no_cumplidas": 1,
    "indeterminadas": 0
  },
  "reglas": [
    {
      "regla_id": "PAG-001",
      "estado": "NO_CUMPLE",
      "observado": { "monto_mxn": 15000, "estado_aprobacion": "NO_APROBADO" },
      "esperado": { "aprobacion_requerida": true, "estado_aprobacion": "APROBADO" },
      "refs_evidencia": ["EV-0001", "EV-0002"]
    },
    {
      "regla_id": "PAG-002",
      "estado": "CUMPLE",
      "observado": { "categoria": "OPERACIONES" },
      "esperado": { "categorias_permitidas": ["OPERACIONES", "SOFTWARE", "VIAJES"] },
      "refs_evidencia": ["EV-0003"]
    },
    {
      "regla_id": "PAG-003",
      "estado": "CUMPLE",
      "observado": { "solicitante_id": "USU-001", "aprobador_id": null },
      "esperado": { "aprobador_requerido_por_monto": false },
      "refs_evidencia": []
    },
    {
      "regla_id": "PAG-004",
      "estado": "CUMPLE",
      "observado": { "factura_verificada": true },
      "esperado": { "factura_verificada": true },
      "refs_evidencia": ["EV-0004"]
    },
    {
      "regla_id": "PAG-005",
      "estado": "CUMPLE",
      "observado": { "monto_centavos": 1500000, "presupuesto_disponible_centavos": 2500000 },
      "esperado": { "monto_no_supera_disponible": true },
      "refs_evidencia": ["EV-0005"]
    }
  ],
  "controles_ejecucion": {
    "puede_avanzar": false,
    "requiere_investigacion": true
  }
}
```

> El array `reglas` siempre contiene el resultado de las **cinco** reglas, en el orden `PAG-001`…`PAG-005`, no solo las que fallan.

### Errores relevantes

`400` `ERROR_VALIDACION` · `404` `PRESUPUESTO_NO_ENCONTRADO` / `ADJUNTO_NO_ENCONTRADO` · `409` `CLAVE_IDEMPOTENCIA_REUTILIZADA` / `IDEMPOTENCIA_EN_PROGRESO` · `422` `ERROR_VALIDACION_SEMANTICA` · `429` `LIMITE_SOLICITUDES_EXCEDIDO` · `500` `FALLO_PERSISTENCIA` / `ERROR_INTERNO` · `503` `DEPENDENCIA_NO_DISPONIBLE`.

Si la persistencia de la decisión falla (paso 4 de [base-datos.md § 3](base-datos.md#3-persistencia-segura)), la API responde `500 FALLO_PERSISTENCIA` y **no** crea una ejecución ni sugiere que el pago avanzó.

## 4. `GET /api/v1/ejecuciones`

**Objetivo:** listar el historial de ejecuciones con filtros y paginación.

**Query params:**

| Parámetro | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `decision` | `PERMITIR` \| `BLOQUEAR` | No | Filtra por decisión |
| `estado_investigacion` | enum | No | Filtra por estado de investigación |
| `desde` / `hasta` | ISO 8601 | No | Rango de fechas por `creado_en` |
| `pagina` | entero ≥ 1 | No (default 1) | Página |
| `tamano_pagina` | entero 1–100 | No (default 20) | Tamaño de página |

No existe un filtro `estado_ejecucion`: toda fila listada aquí es, por definición, una ejecución persistida (ver § 0), así que ese campo nunca varía dentro de este listado.

### Respuesta (`200 OK`)

```json
{
  "version_esquema": "1.0",
  "elementos": [
    {
      "ejecucion_id": "EJE-2026-0001",
      "pago_id": "PAG-2026-001",
      "monto": 15000,
      "moneda": "MXN",
      "solicitado_por": "USU-001",
      "decision": "BLOQUEAR",
      "estado_ejecucion": "PERSISTIDA",
      "estado_investigacion": "PARCIAL",
      "creado_en": "2026-10-15T18:00:00Z"
    }
  ],
  "pagina": 1,
  "tamano_pagina": 20,
  "total_elementos": 1
}
```

**Autenticación:** requerida en despliegue público; opcional en demo local controlada (ver [seguridad.md](seguridad.md)).

**Errores relevantes:** `400` `ERROR_VALIDACION` (parámetros de filtro inválidos) · `401`/`403` según entorno.

## 5. `GET /api/v1/ejecuciones/:id`

**Objetivo:** consultar el detalle completo de una ejecución, incluida la última evaluación de reglas y el estado de investigación vigente.

**Parámetros de ruta:** `id` (`ejecucion_id`).

### Respuesta (`200 OK`)

Mismo cuerpo que la respuesta de `POST /api/v1/ejecuciones` (§ 3), más:

```json
{
  "investigacion": {
    "estado": "PARCIAL",
    "intentos": 2,
    "ultimo_intento_id": "INT-2026-0002",
    "actualizado_en": "2026-10-15T18:04:12Z"
  }
}
```

**Errores relevantes:** `404` `EJECUCION_NO_ENCONTRADA`.

## 6. `GET /api/v1/ejecuciones/:id/evidencias`

**Objetivo:** consultar las evidencias asociadas a una ejecución y, si existe, el informe del agente.

### Respuesta (`200 OK`)

```json
{
  "version_esquema": "1.0",
  "ejecucion_id": "EJE-2026-0001",
  "decision": "BLOQUEAR",
  "reglas": [ "...mismo arreglo que en POST /ejecuciones..." ],
  "registros_evidencia": [
    {
      "ref_evidencia": "EV-0001",
      "tipo_evidencia": "REGISTRO_APROBACION",
      "origen": "contexto_confiable.aprobaciones",
      "contenido": { "estado_aprobacion": "NO_APROBADO" }
    }
  ],
  "informe_agente": {
    "intento_id": "INT-2026-0002",
    "estado": "PARCIAL",
    "informe": { "...contrato del informe del agente, ver § 7..." }
  }
}
```

Si no existe ningún intento de investigación todavía, `informe_agente` es `null` y el detalle de `investigacion` en `GET /ejecuciones/:id` muestra `estado: "NO_INICIADA"`.

**Errores relevantes:** `404` `EJECUCION_NO_ENCONTRADA`.

## 7. `POST /api/v1/ejecuciones/:id/investigar`

**Objetivo:** iniciar la primera investigación o reintentar una investigación en estado `FALLIDA` o `PARCIAL`. **Nunca** recalcula ni modifica `decision`, `estado_ejecucion`, los datos originales de la ejecución, ni los `evaluaciones_reglas` ya persistidos.

**Parámetros de ruta:** `id` (`ejecucion_id`).

**Request:** cuerpo vacío o `{}`.

### Comportamiento según el estado actual

| `estado_investigacion` actual | Efecto de la llamada |
|---|---|
| `NO_INICIADA` | Crea el primer intento, pasa a `PENDIENTE` → `EN_CURSO` |
| `PENDIENTE` o `EN_CURSO` | `409 Conflict`, código `INVESTIGACION_EN_CURSO`; no crea un intento duplicado |
| `FALLIDA` | Crea un nuevo intento (incrementa el historial), pasa a `PENDIENTE` → `EN_CURSO` |
| `PARCIAL` | Crea un nuevo intento para intentar completar la investigación, pasa a `PENDIENTE` → `EN_CURSO` |
| `COMPLETADA` | `409 Conflict`, código `INVESTIGACION_YA_COMPLETADA`; un informe completo no se vuelve a generar automáticamente |

Cada intento queda registrado en `intentos_investigacion` (ver [base-datos.md](base-datos.md)); el historial completo es consultable, aunque la API solo expone por defecto el último intento en `GET /ejecuciones/:id`.

**Implementación:** este endpoint es una llamada directa a `fn_programar_intento_investigacion(ejecucion_id)` ([base-datos.md § 5.1](base-datos.md#51-crear-un-intento-sin-duplicar-investigaciones-en-curso)) — la misma función que usa la transacción de creación para el primer intento y el reconciliador de ejecuciones abandonadas ([base-datos.md § 5.6](base-datos.md#56-reconciliador-de-ejecuciones-not_started-defensa-en-profundidad)). El backend traduce el `resultado` de la función directamente: `CREADO` → `202` (tabla de arriba, filas `NO_INICIADA`/`FALLIDA`/`PARCIAL`), `EN_PROGRESO` → `409 INVESTIGACION_EN_CURSO`, `YA_COMPLETADA` → `409 INVESTIGACION_YA_COMPLETADA`, `EJECUCION_NO_ENCONTRADA` → `404`.

**Concurrencia:** la verificación de la tabla anterior y la creación del nuevo intento ocurren dentro de esa misma función, que bloquea la fila de la ejecución (`SELECT ... FOR UPDATE`), de modo que dos llamadas concurrentes a este endpoint para la misma ejecución — o una llamada concurrente con el reconciliador — nunca crean dos intentos activos a la vez: la segunda siempre ve el `estado_investigacion` ya actualizado por la primera y recibe `EN_PROGRESO`.

**Resultados tardíos:** si un intento se marca `FALLIDA` por abandono (recuperado por el *reaper*, [base-datos.md § 5.3](base-datos.md#53-el-reaper-recuperar-trabajos-abandonados)) y la llamada original a Nebius responde más tarde, esa respuesta tardía se descarta internamente (ver [base-datos.md § 5.4](base-datos.md#54-fencing-finalización-atómica-de-un-intento)) y nunca llega a ser visible por la API: el cliente que consulte `GET /ejecuciones/:id` en ese momento ya ve `FALLIDA` con `codigo_error = PROCESO_PERDIDO`, y puede reintentar con una nueva llamada a este endpoint.

### Respuesta (`202 Accepted`)

```json
{
  "version_esquema": "1.0",
  "ejecucion_id": "EJE-2026-0001",
  "estado_investigacion": "PENDIENTE",
  "intento_id": "INT-2026-0003"
}
```

**Errores relevantes:** `404` `EJECUCION_NO_ENCONTRADA` · `409` `INVESTIGACION_EN_CURSO` / `INVESTIGACION_YA_COMPLETADA`.

## 8. `GET /api/v1/reglas`

**Objetivo:** consultar las reglas activas y su versión de política vigente.

### Respuesta (`200 OK`)

```json
{
  "version_esquema": "1.0",
  "version_politica": "politica-pagos-v1",
  "reglas": [
    {
      "regla_id": "PAG-001",
      "nombre": "Límite de autorización",
      "habilitada": true,
      "configuracion": { "umbral_aprobacion_centavos": 1000000 }
    }
  ]
}
```

**Autenticación:** lectura pública permitida en demo; en despliegue público, requerida como el resto de endpoints.

## 9. `GET /api/v1/salud`

**Objetivo:** comprobar disponibilidad de la API y, opcionalmente, de sus dependencias críticas.

### Respuesta (`200 OK`)

```json
{
  "estado": "OK",
  "dependencias": {
    "base_datos": "OK",
    "nebius": "DESCONOCIDO"
  }
}
```

`nebius` se reporta como `DESCONOCIDO` cuando el health check no prueba activamente el proveedor (recomendado para el MVP, ya que el agente no es crítico para servir el resto de la API). Si en algún momento se agrega una comprobación activa, debe tener su propio timeout corto y no bloquear la respuesta del endpoint.

## 10. Contrato del informe del agente

```json
{
  "version_esquema": "1.0",
  "ejecucion_id": "EJE-2026-0001",
  "intento_id": "INT-2026-0002",
  "agente": {
    "proveedor": "nebius",
    "modelo": "MODELO_CONFIGURADO",
    "version_prompt": "flowguard-investigator-v1"
  },
  "investigacion": {
    "estado": "PARCIAL",
    "titular": "Pago bloqueado por falta de autorización",
    "explicacion": "El importe supera el límite configurado y no existe una aprobación válida registrada.",
    "hallazgos": [
      {
        "hallazgo_id": "FND-001",
        "regla_id": "PAG-001",
        "afirmacion": "El pago requiere aprobación del supervisor.",
        "refs_evidencia": ["EV-0001", "EV-0002"]
      }
    ],
    "acciones_recomendadas": [
      {
        "accion": "SOLICITAR_APROBACION_SUPERVISOR",
        "requiere_accion_humana": true
      }
    ]
  },
  "resumen_decision": {
    "decision": "BLOQUEAR",
    "origen_decision": "MOTOR_REGLAS",
    "agente_puede_anular": false
  },
  "limitaciones": [
    "No se completaron todas las consultas complementarias."
  ]
}
```

`investigacion.estado` en este contrato usa el mismo conjunto cerrado que `estado_investigacion` (§ 0): `COMPLETADA`, `PARCIAL` o `FALLIDA` (un intento `FALLIDA` puede no producir este objeto en absoluto; ver [seguridad.md § 1.7](seguridad.md#17-validación-de-informes)).

## Referencias

- Modelo de estados persistido, tablas e índices: [base-datos.md](base-datos.md)
- Reglas de negocio aplicadas por el motor: [reglas.md](reglas.md)
- Controles de validación y autorización por entorno: [seguridad.md](seguridad.md)
- Catálogo completo de errores: [catalogo-errores.md](catalogo-errores.md)
