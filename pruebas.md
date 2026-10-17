# Plan de pruebas y criterios de aceptación

## 0. Qué se ejecutó realmente y qué queda solo especificado

Este documento es, en su mayoría, una **especificación** de pruebas para que el equipo las implemente con Vitest/Playwright — no hay código de aplicación en este repositorio que las ejecute. La única excepción son las garantías a nivel de base de datos del mecanismo asíncrono y del *rate limiting*, que sí se **ejecutaron de verdad** contra un PostgreSQL 16 real en un contenedor Docker desechable, durante la tercera auditoría de esta especificación (ver la nota de verificación al inicio de [base-datos.md](base-datos.md) y el comentario en la cabecera de `supabase/migrations/001_esquema_inicial.sql`). Concretamente, se ejecutaron con datos reales y se confirmó el resultado indicado:

| Escenario | Resultado confirmado con datos reales |
|---|---|
| Restricción `UNIQUE (clave_idempotencia, escenario_id)` | Rechaza una clave duplicada |
| Creación atómica del primer intento junto con la decisión | Sin ventana `NO_INICIADA` huérfana |
| `fn_programar_intento_investigacion` llamada dos veces para la misma ejecución | La segunda devuelve `EN_PROGRESO`, no crea un segundo intento |
| `fn_finalizar_intento_investigacion` invocada con un `reclamado_por` que no es el dueño del intento | `aplicado = false`, no modifica nada |
| `fn_finalizar_intento_investigacion` invocada por el dueño correcto | Aplica, persiste el informe y propaga el estado a `ejecuciones` |
| `fn_finalizar_intento_investigacion` invocada dos veces sobre el mismo intento ya finalizado | La segunda vez `aplicado = false`; no se duplica el informe en `informes_agente` |
| Reconciliador (`fn_programar_intento_investigacion` sobre una ejecución `NO_INICIADA` simulada) | Recupera la ejecución y crea su primer intento sin duplicar |
| *Reaper* (`fn_recuperar_intentos_abandonados`) sobre un intento `EN_CURSO` abandonado | Lo marca `FALLIDA`/`PROCESO_PERDIDO` y propaga correctamente a `ejecuciones.estado_investigacion` |
| `fn_verificar_limite_solicitudes` bajo 20 solicitudes realmente concurrentes contra un límite de 10 | Exactamente 10 permitidas, 10 rechazadas — sin evasión por condición de carrera |

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

El puesto 5 coordina la suite transversal y mantiene los contratos y fixtures, pero **no** implementa todas las pruebas de todos los módulos: los propietarios técnicos siguen siendo responsables de las pruebas funcionales de los suyos. El puesto 4 no es responsable funcional de QA: automatiza la ejecución y comunica los fallos del pipeline, y el propietario corrige los defectos funcionales. Los hitos de los días 6, 9, 12 y 15 se verifican de forma cruzada (ver [definicion-de-terminado.md](definicion-de-terminado.md#criterios-por-hito-de-bloque)).

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

Cada prueba indica el estado inicial relevante de `claves_idempotencia`/`ejecuciones`, la operación ejecutada, el resultado esperado y la invariante que no debe romperse.

| Prueba | Estado inicial | Operación | Resultado esperado | Invariante que no debe romperse |
|---|---|---|---|---|
| Caso A — repetición idéntica | Ya existe una ejecución `PERSISTIDA` creada con `clave_idempotencia = K` y contenido `H` | `POST /ejecuciones` con `clave_idempotencia = K` y el mismo contenido (`hash_solicitud = H`) | `200 OK` con la ejecución existente; no se evalúan las reglas de nuevo | No se crea una segunda fila en `ejecuciones`; `evaluaciones_reglas` no cambia |
| Caso B — clave reutilizada con otro contenido | Igual que arriba | `POST /ejecuciones` con `clave_idempotencia = K` y contenido distinto (`hash_solicitud ≠ H`) | `409 CLAVE_IDEMPOTENCIA_REUTILIZADA` | La ejecución existente no se modifica; no se crea una ejecución nueva |
| Caso C — solicitudes concurrentes idénticas | Ninguna fila previa para `clave_idempotencia = K` | Dos `POST /ejecuciones` enviados en paralelo (promesas concurrentes en la prueba) con `clave_idempotencia = K` y contenido idéntico | Exactamente una recibe `201 Created`; la otra recibe `200 OK` con el mismo resultado o `409 IDEMPOTENCIA_EN_PROGRESO` seguido de un reintento exitoso | Nunca existen dos filas en `ejecuciones` para la misma `clave_idempotencia`; la restricción `UNIQUE (clave_idempotencia, escenario_id)` rechaza la segunda inserción a nivel de base de datos, no solo a nivel de aplicación |
| Caso D — reintento tras fallo de persistencia | `claves_idempotencia.estado_ejecucion = 'FALLIDA'` para `K` (simulado forzando un error en la transacción A) | `POST /ejecuciones` con `clave_idempotencia = K` y el mismo contenido | Se reevalúan las reglas desde cero; `201 Created` si tiene éxito esta vez | No se trata como Caso A (no hay ejecución previa que devolver); un `hash_solicitud` distinto en este reintento sigue siendo Caso B, no libera la clave |
| Caso E — creación repetida tras investigación fallida | Ejecución `PERSISTIDA` existente con `estado_investigacion = FALLIDA` | `POST /ejecuciones` con la misma `clave_idempotencia` y contenido | `200 OK` con la ejecución existente, incluyendo su `estado_investigacion = FALLIDA` actual; **no** se crea un nuevo `intentos_investigacion` | La creación nunca dispara una investigación nueva; solo `POST /investigar` puede hacerlo |
| Hash insensible al orden de propiedades | — | Calcular `hash_solicitud` de dos payloads semánticamente idénticos pero con las propiedades JSON en distinto orden | Mismo `hash_solicitud` en ambos casos | La función de normalización ordena las claves antes de serializar |
| Hash insensible a campos opcionales omitidos vs. explícitos | — | Calcular `hash_solicitud` de un payload que omite un campo opcional y otro que lo envía explícitamente con su valor por defecto | Mismo `hash_solicitud` en ambos casos | Los valores por defecto se aplican antes de hashear |

### Fallos de persistencia, del proveedor y del mecanismo asíncrono

| Prueba | Estado inicial | Operación | Resultado esperado | Invariante que no debe romperse |
|---|---|---|---|---|
| Fallo antes de persistir la decisión | Transacción A configurada para fallar (mock de error de base de datos) | `POST /ejecuciones` | `500 FALLO_PERSISTENCIA`; no se inicia ninguna llamada al servicio de IA | No queda fila en `ejecuciones`; `claves_idempotencia.estado_ejecucion = 'FALLIDA'` |
| Timeout de Nebius | Intento `EN_CURSO`, mock de Nebius configurado para no responder dentro de `AGENTE_TIEMPO_LIMITE_MS` | El worker procesa el intento | `intentos_investigacion.estado = FALLIDA`, `codigo_error = TIEMPO_AGOTADO_PROVEEDOR` | `decision` no se toca; no se reintenta automáticamente |
| Respuesta JSON inválida del proveedor | Intento `EN_CURSO`, mock de Nebius responde un JSON que no valida contra el esquema | El worker procesa la respuesta | `intentos_investigacion.estado = FALLIDA`, `codigo_error = RESPUESTA_INVALIDA_PROVEEDOR` | No se persiste ningún `informes_agente` con datos inventados |
| Referencia de evidencia inexistente | Intento `EN_CURSO`, mock de Nebius cita un `ref_evidencia` que no existe para esa ejecución | El worker valida el informe | El hallazgo con la referencia inválida se descarta; si no queda nada verificable, `FALLIDA` con `codigo_error = REFERENCIA_EVIDENCIA_INVALIDA` | Ningún hallazgo sin evidencia real llega a mostrarse como válido |
| Informe parcial válido | Intento `EN_CURSO`, mock de Nebius responde hallazgos parciales pero verificables | El worker valida el informe | `intentos_investigacion.estado = PARCIAL`, fila en `informes_agente`, `limitaciones` no vacío | `decision` no se toca |
| **Reinicio o abandono de un trabajo de investigación** [✅ ejecutada] | Fila en `intentos_investigacion` con `estado = EN_CURSO` y `reclamado_en` más antiguo que el umbral de abandono (simulando un worker caído a mitad de proceso) | Se ejecuta `fn_recuperar_intentos_abandonados` ([base-datos.md § 5.3](base-datos.md#53-el-reaper-recuperar-trabajos-abandonados)) | La fila pasa a `estado = FALLIDA`, `codigo_error = PROCESO_PERDIDO`; `ejecuciones.estado_investigacion` se actualiza a `FALLIDA` en la misma llamada atómica | `decision` no se toca; la recuperación no requiere intervención manual; confirmado con datos reales (ver § 0) |
| **Decisión persistida, pero falla la creación del primer intento** | Simular que `fn_programar_intento_investigacion`, llamada dentro de la misma transacción que persiste la ejecución, lanza una excepción (p. ej. forzando un error de restricción) | `POST /ejecuciones` | La transacción completa se revierte: **tampoco** queda la fila de `ejecuciones` — no es posible tener la decisión persistida sin su primer intento, por diseño ([base-datos.md § 3](base-datos.md#3-persistencia-segura)) | No existe el escenario "decisión persistida sin investigación programada" bajo operación normal; si aparece una fila así, solo puede venir de datos heredados de un esquema anterior |
| **El reconciliador recupera una ejecución `NO_INICIADA` sin crear duplicados** [✅ ejecutada] | Fila de `ejecuciones` con `estado_investigacion = 'NO_INICIADA'` insertada directamente (simulando datos heredados de un esquema anterior a esta auditoría) y `creado_en` anterior al margen de gracia | Ciclo del reconciliador: `SELECT` de elegibles + `fn_programar_intento_investigacion` ([base-datos.md § 5.6](base-datos.md#56-reconciliador-de-ejecuciones-not_started-defensa-en-profundidad)) | Se crea el intento 1, `estado_investigacion` pasa a `PENDIENTE` | Ejecutar el reconciliador dos veces seguidas sobre la misma fila la segunda vez no encuentra nada elegible (ya no está `NO_INICIADA`); confirmado con datos reales (ver § 0) |
| **Dos procesos intentan programar simultáneamente la misma investigación** [✅ ejecutada como caso equivalente] | Ejecución con `estado_investigacion = NO_INICIADA` | Dos llamadas concurrentes a `fn_programar_intento_investigacion` para la misma ejecución — sin importar si vienen de `POST /investigar`, de la transacción de creación o del reconciliador | Exactamente una recibe `CREADO`; la otra recibe `EN_PROGRESO` | Nunca hay dos intentos `PENDIENTE`/`EN_CURSO` simultáneos para la misma ejecución; el caso de dos `POST /investigar` concurrentes se probó secuencialmente con datos reales (ver § 0), pendiente de repetir con concurrencia real como la prueba del rate limiter |
| **El worker se reinicia mientras hay trabajos `PENDIENTE`** | Fila en `intentos_investigacion` con `estado = PENDIENTE` creada antes de "reiniciar" el worker de prueba | Se reinicia el proceso worker (nueva instancia del loop de sondeo, nuevo `reclamado_por`) | El trabajo `PENDIENTE` se reclama y procesa con normalidad en el primer ciclo tras el reinicio | Ningún trabajo `PENDIENTE` queda huérfano indefinidamente; no depende de memoria del proceso anterior |
| **Un intento antiguo que termina después de un reintento** [✅ ejecutada] | Intento 1 marcado `FALLIDA` por el *reaper*; intento 2 creado y en `EN_CURSO` | Un worker con el `reclamado_por` del intento 1 (que seguía procesando en segundo plano) llama a `fn_finalizar_intento_investigacion` para el intento 1 | `aplicado = false` (el intento 1 ya no está `EN_CURSO`); se descarta y se registra en logs | `ejecuciones.estado_investigacion` sigue reflejando el intento 2, nunca se sobrescribe con el resultado tardío del intento 1; confirmado con datos reales (ver § 0) |
| **La persistencia del informe falla durante la finalización** | Intento `EN_CURSO`, `reclamado_por` correcto, pero `informe_json` viola una restricción de `informes_agente` (p. ej. `modelo` nulo cuando la columna lo exige) | `fn_finalizar_intento_investigacion(..., 'COMPLETADA', ...)` | La función entera revierte: el intento permanece `EN_CURSO` (el `UPDATE` del paso 1 también se revierte), no queda fila en `informes_agente` | No existe un estado donde el intento diga `COMPLETADA` sin que exista su informe — la atomicidad de la función lo impide por construcción |
| **La actualización del estado de la ejecución falla durante la finalización** | Intento `EN_CURSO` cuyo `numero_intento` ya no es el más reciente de `ejecuciones.numero_ultimo_intento` (otro intento más nuevo se creó mientras tanto) | `fn_finalizar_intento_investigacion(..., 'COMPLETADA', ...)` | `aplicado = true`, `propagado = false`: el intento se finaliza y su informe se persiste igual (para el historial), pero `ejecuciones.estado_investigacion` no se toca | El intento superado nunca pisa el estado vigente; ver el mismo mecanismo probado en la fila anterior |
| **Dos solicitudes concurrentes de finalización no generan informes duplicados** [✅ ejecutada] | Intento ya finalizado (`COMPLETADA`, con su fila en `informes_agente`) | Una segunda llamada a `fn_finalizar_intento_investigacion` para el mismo `intento_id` | `aplicado = false`; `informes_agente` sigue con exactamente una fila para ese intento | La condición `estado = 'EN_CURSO'` del paso 1 hace que la segunda llamada no reclame nada que reclamar; confirmado con datos reales (ver § 0) |
| Reintento manual sin modificar la decisión | Ejecución `BLOQUEAR` con `estado_investigacion = FALLIDA` | `POST /investigar` | Nuevo `intentos_investigacion` con `numero_intento` incrementado | `decision = BLOQUEAR` y los cinco `evaluaciones_reglas` originales permanecen idénticos byte a byte |
| Investigación ya completada | `estado_investigacion = COMPLETADA` | `POST /investigar` | `409 INVESTIGACION_YA_COMPLETADA`; no se crea un intento nuevo | El informe `COMPLETADA` existente no se modifica |

### Rate limiting

| Prueba | Estado inicial | Operación | Resultado esperado | Invariante que no debe romperse |
|---|---|---|---|---|
| **El límite funciona y no se evade con concurrencia** [✅ ejecutada] | `contadores_limite_solicitudes` vacía para la clave de prueba | 20 llamadas realmente concurrentes a `fn_verificar_limite_solicitudes(key, 60, 10)` (procesos separados, no la misma conexión) | Exactamente 10 `permitido = true`, exactamente 10 `permitido = false` | Confirmado con datos reales: 20 llamadas concurrentes produjeron exactamente ese resultado (ver § 0 y la nota de verificación en [base-datos.md](base-datos.md)) |
| **Un error del sistema de rate limiting se comporta según la política definida** | `contadores_limite_solicitudes`/la conexión a PostgreSQL no disponible durante la verificación | `POST /api/v1/ejecuciones` | La solicitud se permite (*fail-open*) y se registra un `warn` en logs, según la política explícita de [seguridad.md § 2.4](seguridad.md#24-límite-de-solicitudes-rate-limiting) | Un fallo del almacén de límites nunca bloquea tráfico legítimo de la demo (decisión documentada, no un descuido) |
| **El rate limiting se mantiene efectivo con varias instancias** | Dos procesos backend simulados (dos conexiones distintas a la misma base), misma `clave_limite` | Cada instancia procesa la mitad de 20 solicitudes concurrentes contra un límite de 10 | El total permitido across ambas instancias sigue siendo exactamente 10, no 20 | El contador vive en PostgreSQL, no en memoria de cada instancia — ya verificado en la prueba de concurrencia anterior, que es equivalente a "varias instancias" porque cada conexión psql es independiente |

### Transiciones de estado

- Verificar cada transición permitida de `claves_idempotencia.estado_ejecucion` (`RECIBIDA → VALIDANDO → EVALUANDO → PERSISTIDA`, la salida a `FALLIDA` desde cualquiera de los tres primeros, y `FALLIDA → RECIBIDA` al reintentar con el mismo contenido).
- Verificar que una transición prohibida (p. ej. de `PERSISTIDA` a cualquier otro valor, o saltar directo a `PERSISTIDA` sin `EVALUANDO`) es rechazada por el código del backend, no solo por la restricción `CHECK` de la base de datos.
- Verificar cada transición permitida de `estado_investigacion` (`NO_INICIADA → PENDIENTE → EN_CURSO → {COMPLETADA | PARCIAL | FALLIDA}`, el reingreso a `PENDIENTE` desde `FALLIDA`/`PARCIAL`, y `EN_CURSO → FALLIDA` forzado por el *reaper*).
- Verificar que no existe ningún camino de código que escriba `ejecuciones.decision` más de una vez para la misma ejecución.
- **Fallos de actualización de estados:** simular que la actualización condicional de `ejecuciones.estado_investigacion` (la que lleva `AND numero_ultimo_intento = $numero_intento`, [base-datos.md § 5.4](base-datos.md#54-fencing-finalización-atómica-de-un-intento)) afecta 0 filas porque ya fue superada por un intento más nuevo: el worker debe continuar sin lanzar un error no controlado, y el resultado del intento superado debe quedar solo en `intentos_investigacion`, nunca propagarse.

### Evidencia, autorización y RLS

- Validación de referencias de evidencia: un informe que cite un `ref_evidencia` o `regla_id` inexistente para esa ejecución debe ser rechazado (ver [seguridad.md § 1.7](seguridad.md#17-validación-de-informes)) con `codigo_error = REFERENCIA_EVIDENCIA_INVALIDA`.
- Autorización y permisos: un intento de enviar `estado_aprobacion`, roles o permisos de aprobador directamente en el payload de `POST /ejecuciones` debe ser ignorado por el backend; la regla `PAG-001`/`PAG-003` debe evaluarse solo contra el contexto confiable.
- RLS: con la clave `anon`, un intento de `SELECT`/`INSERT` directo contra cualquiera de las once tablas (incluida `contadores_limite_solicitudes`) debe ser denegado por Supabase (ver [`supabase/migrations/001_esquema_inicial.sql`](https://github.com/FlowGuard-Hackathon/flowguard-supabase/blob/main/migrations/001_esquema_inicial.sql)); solo el rol de servicio usado por el backend debe poder leer/escribir. Las funciones PL/pgSQL (§ 5 de [base-datos.md](base-datos.md)) se ejecutan con los permisos de quien las invoca por defecto (`SECURITY INVOKER`, el comportamiento estándar de PostgreSQL): el backend debe invocarlas con el rol de servicio, igual que cualquier otra operación — esta es una verificación pendiente de confirmar específicamente contra un proyecto Supabase real (ver riesgo abierto en [checklist-implementacion.md](checklist-implementacion.md)).

### Conversión monetaria

- `monto` con más de 2 decimales debe rechazarse con `ERROR_VALIDACION`, no redondearse silenciosamente.
- `monto` negativo, cero, `NaN` o fuera del rango admitido (`0.01`–`10,000,000.00` MXN) debe rechazarse.
- La conversión de `monto` (pesos) a `monto_centavos` (centavos enteros) debe producir el mismo resultado sin importar el módulo que la invoque, verificado contra la única función centralizada (`aCentavos`).

### Recuperación del historial

- `GET /api/v1/ejecuciones` con cada combinación de filtros (`decision`, `estado_investigacion`, rango de fechas) debe devolver solo las filas que corresponden. No existe un filtro `estado_ejecucion` (ver [contratos-api.md § 4](contratos-api.md#4-get-apiv1executions)).
- `GET /api/v1/ejecuciones/:id` debe reflejar el `estado_investigacion` más reciente y el conteo correcto de `intentos` después de varios reintentos.

### Acceso no autorizado a la demo pública

- Con `DEMO_AUTH_BASICA_USUARIO`/`DEMO_AUTH_BASICA_CONTRASENA` configurados (ver [seguridad.md § 2.0](seguridad.md#20-estrategia-de-seguridad-por-entorno)), una solicitud a cualquier ruta (incluidas `/api/v1/reglas` y `/api/v1/salud`) sin credenciales válidas debe ser rechazada por el middleware antes de llegar al route handler.
- Verificar que las credenciales de Basic Auth nunca aparecen en el HTML ni en los bundles de JavaScript servidos al navegador (inspección del build de producción).
- Verificar que `429 LIMITE_SOLICITUDES_EXCEDIDO` se activa ante un volumen de solicitudes anómalo durante la ventana de la demo, incluso con credenciales válidas.
- Responsables: Dev 1 (middleware y credenciales) y Dev 2 (verifica el acceso desde la interfaz); el funcionamiento del acceso se comprueba durante el ensayo de la demo. Las credenciales nunca se almacenan en Git ni se imprimen en logs.

### Coherencia entre base de datos y contratos

- Cada campo de las respuestas JSON documentadas en [contratos-api.md](contratos-api.md) tiene una columna o valor derivado correspondiente en [base-datos.md](base-datos.md); una prueba de contrato recorre ambos documentos y falla si encuentra un campo sin origen claro (p. ej. ejecutada como parte de la regresión de documentación, no como prueba de runtime).
- El esquema Zod de cada contrato valida contra al menos un ejemplo generado a partir de una fila real de cada tabla relevante.

## 3. Pruebas end-to-end

1. Pago de $8,500: resultado `PERMITIR`.
2. Pago de $15,000 sin aprobación: resultado `BLOQUEAR`.
3. Pago con categoría no permitida: resultado `BLOQUEAR`.
4. Pago con factura no verificada: resultado `BLOQUEAR`.
5. Pago con presupuesto insuficiente: resultado `BLOQUEAR`.
6. Nebius no disponible: `decision` y `estado_ejecucion = PERSISTIDA` se mantienen; `estado_investigacion` queda en `FALLIDA`.
7. Informe parcial: limitaciones visibles en la interfaz, `estado_investigacion = PARCIAL`.
8. Repetición de una solicitud con la misma `clave_idempotencia`: no genera duplicados no deseados; el usuario ve la misma ejecución.
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

- Las cinco reglas evaluadas: [reglas.md](reglas.md)
- Contratos usados en fixtures de prueba: [contratos-api.md](contratos-api.md)
- Comportamiento esperado ante fallos de Nebius/DB: [arquitectura.md](arquitectura.md#5-resiliencia-escalabilidad-y-aislamiento-de-fallos)
- Mecanismo asíncrono, *fencing* y recuperación (base de las pruebas de concurrencia): [base-datos.md § 5](base-datos.md#5-concurrencia-y-recuperación-del-mecanismo-asíncrono-de-investigación)
- Catálogo de errores y códigos citados en esta tabla: [catalogo-errores.md](catalogo-errores.md)
