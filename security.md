# Servicio de IA, seguridad y límites de confianza

> Los valores de `investigation_status` usados en este documento (`NOT_STARTED`, `PENDING`, `RUNNING`, `COMPLETED`, `PARTIAL`, `FAILED`) siguen el modelo de estados unificado de [database.md § 2](database.md#2-modelo-de-estados-unificado). `execution_status` y `decision` nunca se modifican desde el flujo descrito aquí.

## Parte 1 — Servicio de IA y ciclo de investigación

### 1.1. Responsabilidad

El agente investiga una ejecución concreta. **No** toma decisiones de negocio, **no** modifica registros y **no** ejecuta acciones correctivas.

Su entrada mínima es:

```ts
type InvestigationInput = {
  executionId: string;
  decision: "ALLOW" | "BLOCK";
  violations: RuleResult[];
};
```

Su salida es un informe estructurado, validado y con referencias a evidencias existentes (ver [api-contracts.md](api-contracts.md#3-contrato-del-informe-del-agente)).

### 1.2. Herramientas de lectura

| Herramienta | Parámetros | Uso |
|---|---|---|
| `get_execution` | `execution_id` | Estado y decisión persistida |
| `get_rule_details` | `rule_id` | Política y umbral aplicable |
| `get_rule_results` | `execution_id` | Resultados de las cinco reglas |
| `get_evidence` | `execution_id`, `evidence_ref` | Recuperar el registro de evidencia |
| `get_payment_context` | `execution_id` | Importe, categoría, presupuesto y autorización verificados |

Las herramientas se implementan en el backend. Nebius solo solicita una herramienta; el servidor valida los argumentos y ejecuta la consulta con permisos limitados.

### 1.3. Ciclo de investigación

1. Recibir el identificador de ejecución y la decisión persistida.
2. Identificar qué infracciones requieren explicación.
3. Solicitar herramientas de lectura.
4. Validar las respuestas y recuperar evidencias.
5. Generar un informe estructurado.
6. Validar las referencias y guardar el informe sin modificar la decisión.

El servicio debe limitar las llamadas de herramientas, el tiempo de ejecución y el tamaño de respuesta.

### 1.4. Límite de llamadas y resultados parciales

La configuración inicial será de **cinco llamadas a herramientas por investigación**.

| Resultado | Condición |
|---|---|
| `COMPLETED` | Se dispone de evidencia suficiente y el informe pasa las validaciones |
| `PARTIAL` | Existe un informe útil y verificable, pero quedan consultas sin completar |
| `FAILED` | No se puede generar o validar un informe utilizable |

Si se alcanza el límite, no se permiten llamadas adicionales automáticamente. El agente puede devolver un informe parcial con `limitations` poblado, siempre que todas sus afirmaciones verificables tengan soporte.

Si no existe evidencia suficiente para una afirmación, debe omitirse o declararse como no verificada. No se debe fabricar un informe completo para ocultar que faltan datos.

### 1.5. Integración con Nebius

El cliente del proveedor debe residir en `src/server/agent/`.

Variables de configuración:

```
NEBIUS_API_KEY=replace_with_secret
NEBIUS_BASE_URL=replace_with_verified_endpoint
NEBIUS_MODEL=replace_with_supported_model_id
AGENT_MAX_TOOL_CALLS=5
AGENT_TIMEOUT_MS=15000
AGENT_WORKER_CONCURRENCY=2
AGENT_WORKER_POLL_INTERVAL_MS=2000
```

Los valores de URL y modelo son marcadores de configuración. Dev 3 debe verificarlos en la documentación vigente de Nebius para el modelo seleccionado. `AGENT_WORKER_CONCURRENCY` y `AGENT_WORKER_POLL_INTERVAL_MS` configuran el worker descrito en [architecture.md § 7](architecture.md#7-mecanismo-de-investigación-asíncrona) y [database.md § 5](database.md#5-concurrencia-y-recuperación-del-mecanismo-asíncrono-de-investigación); sus valores por defecto son un punto de partida, no valores verificados contra los límites reales de la cuenta de Nebius del hackathon.

La clave se mantiene exclusivamente en el entorno del servidor. No debe exponerse al navegador ni almacenarse en el repositorio.

### 1.6. Prompt del agente

```
Eres el agente investigador de FlowGuard.

Tu función es explicar las anomalías detectadas por el motor
determinista y generar un informe de evidencias.

Reglas obligatorias:
1. No decides si un pago se autoriza o bloquea.
2. La decisión autoritativa proviene del motor de reglas.
3. No inventes importes, políticas, permisos ni aprobaciones.
4. Consulta las herramientas disponibles para recuperar datos.
5. Vincula cada hallazgo con referencias de evidencia existentes.
6. Si falta información, indícalo explícitamente.
7. Propón acciones correctivas, pero no las ejecutes.
8. No modifiques pagos, políticas ni estados.
9. Devuelve el informe conforme al esquema establecido.
10. Nunca sustituyas la decisión persistida por una conclusión
    generada por el modelo.
```

El prompt es una instrucción del modelo, no un control de seguridad suficiente por sí solo. La autoridad real la imponen los permisos del backend, los esquemas y las validaciones.

### 1.7. Validación de informes

Antes de persistir un informe, el backend debe verificar:

- Que el esquema es válido.
- Que cada `execution_id` corresponde a la ejecución investigada.
- Que cada `rule_id` existe en los resultados persistidos.
- Que cada referencia de evidencia existe y pertenece a esa ejecución.
- Que la decisión coincide con la decisión persistida.
- Que las acciones propuestas no se ejecutan automáticamente.

Si una referencia es inválida, el informe se rechaza o se vuelve a solicitar una salida corregida dentro del límite de intentos configurado. Si no se puede obtener una respuesta válida, la investigación termina con `FAILED` (ver tabla de códigos en [error-catalog.md](error-catalog.md)).

### 1.8. Lista de verificación de compatibilidad con Nebius

**Esta lista está pendiente de confirmación por Dev 3 contra la documentación oficial vigente de Nebius Token Factory en el momento de implementar.** Nada de lo siguiente debe asumirse como ya verificado: ni el nombre del modelo, ni el formato exacto de tool calling, ni los límites de la cuenta del hackathon. Hasta que se confirme cada punto, debe tratarse como riesgo abierto (ver también [implementation-checklist.md](implementation-checklist.md)).

Antes de integrar, Dev 3 debe confirmar y documentar en este archivo (reemplazando esta nota) los siguientes puntos, con el enlace o evidencia de la fuente oficial consultada:

| Punto a verificar | Qué confirmar |
|---|---|
| Endpoint compatible | URL base real de la API (reemplaza `NEBIUS_BASE_URL=replace_with_verified_endpoint`) y si expone una interfaz compatible con OpenAI (`/chat/completions`) u otra propia |
| Modelo seleccionado | Identificador exacto de modelo soportado por la cuenta del hackathon (reemplaza `NEBIUS_MODEL=replace_with_supported_model_id`) y si soporta salida estructurada/tool calling |
| Autenticación | Esquema real (`Authorization: Bearer`, cabecera propia, etc.) y vigencia de la clave |
| Formato de mensajes | Estructura exacta de `messages`/`system`/`tools` esperada por la API |
| Soporte de llamadas a herramientas | Si el modelo soporta *function calling* nativo o si hay que simularlo con prompting estructurado |
| Formato de argumentos y resultados | Cómo se serializan los argumentos de herramienta y cómo se devuelve el resultado al modelo |
| Manejo de errores y límites | Códigos de error propios del proveedor, límites de tasa (*rate limits*) y cuotas de la cuenta usada en el hackathon |
| Timeouts | Latencia típica observada, para calibrar `AGENT_TIMEOUT_MS` (15000 ms es un punto de partida, no un valor verificado) |
| Respuesta estructurada y validación JSON | Si el proveedor soporta forzar un JSON Schema de salida o si la validación debe hacerse enteramente en el backend tras recibir texto libre |

Mientras alguno de estos puntos no esté confirmado, el equipo debe seguir desarrollando y probando `server/agent/` contra la respuesta mock descrita en el Bloque 1 del plan de trabajo (ver [implementation-checklist.md](implementation-checklist.md)), para no bloquear a Dev 1/Dev 2.

## Parte 2 — Seguridad y límites de confianza

FlowGuard debe aplicar controles técnicos, no depender únicamente de las instrucciones del agente.

### 2.0. Estrategia de seguridad por entorno

El nivel de control exigido no es el mismo en las tres situaciones en las que corre FlowGuard durante el hackathon. Ningún supuesto de autenticación debe darse por hecho si no está efectivamente implementado.

| Entorno | Descripción | Autenticación de usuario | Alcance de la clave `anon`/publicable | Exposición de `SUPABASE_SERVICE_ROLE_KEY` / `NEBIUS_API_KEY` |
|---|---|---|---|---|
| **Desarrollo local** | Cada developer corre la app en su máquina contra un proyecto Supabase de pruebas | No requerida; se puede usar un usuario fijo de prueba | La app puede usar `NEXT_PUBLIC_SUPABASE_ANON_KEY` solo para inicializar el cliente del SDK en el navegador si se usa (p. ej. Supabase Auth); **ninguna lectura/escritura de datos de negocio pasa por el cliente del navegador directo a la base** | Exclusivamente en variables de entorno del proceso del servidor (`.env.local`, nunca commiteado) |
| **Demo del hackathon** | Instancia desplegada temporalmente para la presentación de 3 minutos, con datos controlados por el equipo | **Decisión cerrada (no "pendiente"): Basic Auth aplicada en middleware del servidor** (`middleware.ts` de Next.js, o el *reverse proxy* del hosting), nunca en el cliente — ver detalle abajo | Igual que desarrollo local | Igual que desarrollo local; variables configuradas en el panel del proveedor de hosting, nunca en el repositorio |
| **Despliegue público** (fuera del alcance confirmado del MVP de 15 días, pero la arquitectura debe soportarlo sin rediseño) | Si el proyecto continúa después del hackathon y se expone a usuarios reales | Requerida en todos los endpoints de escritura (`POST /executions`, `POST /investigate`) y de lectura de datos de negocio (`GET /executions*`); `GET /health` puede permanecer público | La clave `anon` solo se usa para autenticación de sesión (Supabase Auth), nunca para leer/escribir las tablas de negocio directamente | Igual; además se recomienda rotación periódica y un secreto distinto por entorno |

**Qué endpoints requieren autenticación, por entorno:**

- Desarrollo local: ningún endpoint de la API de FlowGuard exige autenticación de usuario final, **pero todos** siguen validando que las peticiones solo puedan originar datos a través del backend (nunca escritura directa a Supabase desde el navegador).
- Demo del hackathon: si la demo se expone en una URL pública (en vez de correr solo en la laptop del equipo durante la presentación), **toda la aplicación** — no solo los endpoints de escritura — queda detrás de autenticación básica HTTP, aplicada antes de que cualquier request llegue a una ruta de Next.js. Esto es una decisión deliberadamente más simple que roles/usuarios reales, apropiada para una demo de 3 minutos frente a un jurado, no para un despliegue público duradero.
- Despliegue público: todos los endpoints de `/api/v1/executions*` requieren una identidad válida (autenticación real de usuario, no Basic Auth compartida); `/api/v1/rules` y `/api/v1/health` pueden permanecer de solo lectura pública.

**Protección concreta de la demo pública (reemplaza cualquier nota de "pendiente de decidir"):**

1. **Mecanismo:** autenticación HTTP básica (usuario/contraseña) verificada en `middleware.ts` de Next.js — se ejecuta en el servidor antes de renderizar cualquier página o atender cualquier ruta de `/api/*`, así que las credenciales **nunca llegan al bundle de JavaScript del navegador** ni se pueden extraer inspeccionando el cliente. Esto es lo que distingue este mecanismo de "un secreto estático incluido en el frontend": el secreto solo existe como variable de entorno del servidor (`DEMO_BASIC_AUTH_USER`/`DEMO_BASIC_AUTH_PASSWORD`, ver `.env.example`) y se compara server-side en cada request.
2. **Alcance:** todas las rutas, incluidas las de solo lectura (`/api/v1/rules`, `/api/v1/health`), porque en este entorno no hay necesidad de exponer nada públicamente sin credenciales — a diferencia del entorno de "despliegue público" de la fila anterior, donde sí se distingue lectura pública de escritura autenticada.
3. **Protección adicional:** límite de solicitudes (`429 RATE_LIMITED`, ya definido en el catálogo de errores) aplicado también durante la demo, para que una URL filtrada accidentalmente no pueda usarse para abuso automatizado incluso si alguien adivinara o interceptara la contraseña compartida.
4. **Responsable y plazo:** Dev 1 implementa el middleware; Dev 4 genera y distribuye la contraseña de la demo al equipo (nunca por un canal público) a más tardar el día 13, como parte de la congelación de funcionalidades (ver [implementation-checklist.md, Bloque 5](implementation-checklist.md#bloque-5--días-1315-congelación-y-entrega)).
5. **Opción por defecto si el equipo decide no exponer una URL pública:** no se necesita nada de lo anterior; la demo corre en `localhost` en la laptop que presenta, y este mecanismo queda documentado pero sin activar. Esta es la opción más simple y la recomendada si el formato del hackathon lo permite.

**Qué puede leer y escribir el frontend:**

- El frontend **nunca** escribe directamente en Supabase. Toda escritura pasa por los Route Handlers de Next.js, que usan `SUPABASE_SERVICE_ROLE_KEY` del lado del servidor.
- El frontend lee exclusivamente a través de la API de FlowGuard (`/api/v1/*`), no mediante el SDK de Supabase apuntando a las tablas de negocio desde el navegador.

**Qué operaciones son exclusivas del backend:**

- Evaluación de reglas, persistencia de `executions`/`rule_evaluations`/`evidence_records`, creación y actualización de `investigation_attempts`/`agent_reports`, y toda llamada a Nebius.

**Políticas RLS necesarias:**

- RLS habilitado en las once tablas, incluida `rate_limit_counters` (ver [`supabase/migrations/001_initial_schema.sql`](https://github.com/FlowGuard-Hackathon/flowguard-supabase/blob/main/migrations/001_initial_schema.sql)), sin políticas `SELECT`/`INSERT`/`UPDATE` para el rol `anon`. Esto deniega por defecto cualquier acceso que no use el rol de servicio. Si en el futuro se necesita lectura directa desde el navegador (no planeado para el MVP), se deben agregar políticas explícitas por tabla y por operación, nunca deshabilitar RLS.

**Cómo se usa la clave `anon`/publicable:**

- Únicamente si se adopta Supabase Auth para inicio de sesión (fuera del alcance confirmado del MVP). No se usa para leer ni escribir las tablas de negocio de FlowGuard.

**Cómo se protegen las claves privilegiadas:**

- `SUPABASE_SERVICE_ROLE_KEY` y `NEBIUS_API_KEY` viven exclusivamente en variables de entorno del proceso de servidor (nunca con prefijo `NEXT_PUBLIC_*`), se configuran en el panel del proveedor de hosting o en `.env.local` no versionado, y nunca se registran en logs ni se incluyen en respuestas de error (ver [error-catalog.md](error-catalog.md)).

**Qué permisos usa el backend para consultar y persistir datos:**

- El backend se conecta a Supabase con `service_role`, que omite RLS por diseño. Esto es intencional: el control de acceso real ocurre en la capa de la API de FlowGuard (validación de contrato, reglas de negocio, permisos de operación), no en RLS. RLS aquí es una red de seguridad contra un acceso directo accidental o indebido a la base, no el mecanismo primario de autorización.

### 2.1. Controles obligatorios

- Validación de entradas en el servidor mediante Zod.
- Autenticación y autorización en endpoints cuando se exponga a usuarios reales.
- Claves de Nebius y credenciales privilegiadas exclusivamente en el servidor.
- Acceso a PostgreSQL limitado por el rol de servicio.
- RLS en tablas expuestas mediante Supabase cuando corresponda.
- Ninguna credencial `service_role` en el frontend.
- Herramientas de IA de solo lectura.
- Registro de cambios de estado y errores.
- Validación de referencias de evidencia.
- Separación entre los datos del cliente y el contexto confiable.
- Límite de tamaño de entrada y de respuesta.
- Control de acceso a los detalles de ejecución.

Los adjuntos y descripciones deben tratarse como datos no confiables. Si contienen texto que intenta dar instrucciones al agente, este debe interpretarlo como contenido de un documento, no como instrucciones para cambiar sus permisos o reglas.

### 2.2. Autorización de operaciones

El MVP simula el proceso y no envía dinero. Aun así, la lógica de autorización debe construirse como si fuera a proteger una operación sensible:

- El frontend no decide si el pago puede avanzar.
- El agente no puede autorizar pagos.
- El backend obtiene los roles y las aprobaciones de fuentes confiables.
- Las reglas obligatorias se ejecutan antes de permitir la continuación.
- Las decisiones quedan asociadas a una versión de política.
- Los errores de persistencia impiden declarar una autorización satisfactoria.

### 2.3. Evidencia y trazabilidad

Cada ejecución debe permitir reconstruir qué ocurrió:

- Datos de entrada validados.
- Contexto confiable consultado.
- Versión del motor y de las políticas.
- Resultado de cada regla.
- Decisión final.
- Evidencias utilizadas.
- Modelo y versión del prompt.
- Estado de investigación.
- Errores y tiempos de respuesta.

Para el MVP, los registros no necesitan un sistema de auditoría empresarial completo, pero no deben depender de mensajes de texto generados por la IA como única fuente de evidencia.

### 2.4. Límite de solicitudes (*rate limiting*)

El catálogo de errores ya reserva `429 RATE_LIMITED`, pero hasta ahora no existía una política implementable. Esta sección la cierra.

**Mecanismo elegido: contador compartido en PostgreSQL, no en memoria ni en un servicio externo.** El proyecto ya depende de PostgreSQL para todo lo demás; añadir Redis u otro almacén solo para esto sería la dependencia innecesaria que la sección 2 de [architecture.md](architecture.md) descarta explícitamente. Un contador en memoria del proceso, en cambio, no sirve como garantía si el hosting llega a correr más de una instancia (incluida la variante serverless de [architecture.md § 7.3](architecture.md#73-supuesto-de-despliegue-y-alternativa-para-hosting-serverless), donde cada invocación puede ser un proceso distinto): un atacante repartiría sus solicitudes entre instancias y cada una vería un contador propio por debajo del límite. Un contador en PostgreSQL es correcto sin importar cuántas instancias atienden solicitudes, porque la fuente de verdad es una sola fila de base de datos, no memoria de proceso.

**Esquema:**

```sql
CREATE TABLE rate_limit_counters (
  rate_key      TEXT NOT NULL,
  window_start  TIMESTAMPTZ NOT NULL,
  request_count INTEGER NOT NULL DEFAULT 0,
  PRIMARY KEY (rate_key, window_start)
);
```

**Función atómica de incremento y verificación** (ver también `supabase/migrations/001_initial_schema.sql`):

```sql
CREATE OR REPLACE FUNCTION fn_check_rate_limit(p_key TEXT, p_window_seconds INT, p_limit INT)
RETURNS TABLE(allowed BOOLEAN, current_count INTEGER, retry_after_seconds INTEGER)
LANGUAGE plpgsql AS $$
DECLARE
  v_window_start TIMESTAMPTZ;
  v_count INTEGER;
BEGIN
  v_window_start := to_timestamp(floor(extract(epoch FROM now()) / p_window_seconds) * p_window_seconds);

  INSERT INTO rate_limit_counters (rate_key, window_start, request_count)
  VALUES (p_key, v_window_start, 1)
  ON CONFLICT (rate_key, window_start)
  DO UPDATE SET request_count = rate_limit_counters.request_count + 1
  RETURNING request_count INTO v_count;

  RETURN QUERY SELECT
    v_count <= p_limit,
    v_count,
    CEIL(EXTRACT(EPOCH FROM (v_window_start + make_interval(secs => p_window_seconds) - now())))::INTEGER;
END;
$$;
```

El `INSERT ... ON CONFLICT ... DO UPDATE` es una sola sentencia atómica: dos solicitudes concurrentes para la misma `rate_key` y ventana nunca pueden leer el mismo valor "antes" del incremento y pisarse una a otra — PostgreSQL serializa la escritura a nivel de fila. Esto es lo que evita que el límite se eluda con solicitudes concurrentes (ver prueba en [testing.md](testing.md)).

**Ventana y algoritmo:** ventana fija (*fixed window*) de `p_window_seconds`, no una ventana deslizante — más simple de razonar e implementar en el plazo del hackathon, a costa de permitir en el peor caso hasta el doble del límite nominal justo en el borde entre dos ventanas. Aceptable para proteger una demo, no para un SLA de producción.

**Endpoints y límites:**

| Endpoint | Límite recomendado | Motivo |
|---|---|---|
| `POST /api/v1/executions` | 10 solicitudes / 60 s por `rate_key` | Costosa: evalúa reglas y escribe varias tablas en una transacción |
| `POST /api/v1/executions/:id/investigate` | 5 solicitudes / 60 s por `rate_key` | Costosa: programa trabajo para el worker y, en cadena, una llamada a Nebius |
| `GET /api/v1/executions`, `GET /api/v1/executions/:id`, `GET /api/v1/executions/:id/evidence`, `GET /api/v1/rules` | 60 solicitudes / 60 s por `rate_key` | Lecturas baratas; el límite solo evita *scraping* agresivo |
| `GET /api/v1/health` | Exento (o un límite muy alto, p. ej. 300/60 s) | Usado por monitoreo/health checks; no debe competir por cuota con tráfico real |

**Identificación del cliente (`rate_key`):** nunca confiar en `X-Forwarded-For` tal cual, porque un cliente puede enviarlo falsificado si el proxy intermedio no lo sobrescribe. La política:

1. Si el hosting elegido tiene un encabezado de proxy confiable y verificado (p. ej. el que añade la plataforma de despliegue, documentado y distinto del que el cliente puede enviar), se usa ese — configurado explícitamente vía `TRUSTED_PROXY_HEADER` (ver `.env.example`), nunca asumido por defecto.
2. Si no hay proxy confiable configurado (p. ej. el servidor corre directo, sin *reverse proxy* delante), se usa la IP real del socket TCP (`req.socket.remoteAddress` en Next.js/Node) — no es falsificable porque es la conexión física, no un encabezado.
3. `rate_key = <ip_resuelta_según_1_o_2> + ':' + <grupo_de_endpoint>` (un grupo por cada fila de la tabla anterior), para que el límite de `POST /executions` no consuma la misma cuota que las lecturas.
4. Este mecanismo es intencionalmente *best-effort* para una demo de hackathon, no un límite de seguridad duro: en el entorno de demo pública ya está detrás de Basic Auth ([§ 2.0](#20-estrategia-de-seguridad-por-entorno)), así que el rate limiting aquí es una segunda capa contra abuso automatizado, no la única defensa.

**Comportamiento ante errores del propio almacén de límites:** si la consulta a `rate_limit_counters` falla (PostgreSQL no disponible, timeout), la política es **fail-open con registro de advertencia**: se permite la solicitud y se registra el error, en vez de bloquear tráfico legítimo de la demo por un fallo de un control secundario de abuso. Esta es una decisión explícita, no un descuido: prioriza que la demo funcione frente a jurados sobre una protección de abuso que, en ese entorno, es secundaria a la Basic Auth ya exigida. Si el proyecto evoluciona a un despliegue público sin Basic Auth por delante, esta política debe revisarse hacia *fail-closed* al menos para `POST /api/v1/executions*`.

**Respuesta `429`:** sigue el formato uniforme de error ([api-contracts.md § 1](api-contracts.md#1-formato-uniforme-de-error)) con `code: "RATE_LIMITED"`, más la cabecera HTTP `Retry-After: <retry_after_seconds>` (el valor que devuelve `fn_check_rate_limit`) y `details: { "limit": <p_limit>, "window_seconds": <p_window_seconds> }`.

**Localhost / desarrollo local:** `RATE_LIMIT_ENABLED=false` desactiva el middleware por completo (ver `.env.example`) — no hay superficie de abuso que proteger en una máquina de desarrollo, y añadir latencia de verificación en cada request local no aporta nada.

**Limpieza:** `rate_limit_counters` crece sin límite si nadie la poda. El mismo bucle del worker ([architecture.md § 7.3](architecture.md#73-supuesto-de-despliegue-y-alternativa-para-hosting-serverless)) ejecuta, en cada ciclo del reconciliador, `DELETE FROM rate_limit_counters WHERE window_start < now() - interval '1 hour'` — housekeeping de bajo riesgo, no afecta la corrección del límite (una ventana vieja ya no participa en ninguna comparación).

## Referencias

- Esquema de persistencia de ejecuciones, reglas y evidencias: [database.md](database.md)
- Contratos del informe de IA y de la respuesta del motor: [api-contracts.md](api-contracts.md)
- Catálogo de errores y su efecto sobre `investigation_status`: [error-catalog.md](error-catalog.md)
- Casos de prueba de fallos del proveedor: [testing.md](testing.md)
- Migración SQL con RLS habilitado: [`supabase/migrations/001_initial_schema.sql`](https://github.com/FlowGuard-Hackathon/flowguard-supabase/blob/main/migrations/001_initial_schema.sql)
- Checklist de verificación de Nebius y riesgos abiertos: [implementation-checklist.md](implementation-checklist.md)
