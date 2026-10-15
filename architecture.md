# Arquitectura de FlowGuard

## 1. Decisiones arquitectónicas definitivas

La recomendación para este equipo es una arquitectura híbrida: **frontend monolítico** y **backend modular**, con servicios desplegables de forma independiente cuando sea necesario.

- **Frontend monolítico** — Next.js · React · TypeScript. Panel de control · Simulador · Centro de evidencias. HTTPS · JSON · API versionada.
- **API Gateway / Backend for Frontend** — Validación · Autenticación · Orquestación.
- **Motor de reglas** — Decide `ALLOW` o `BLOCK` con reglas deterministas.
- **Servicio de IA** — Investiga evidencias mediante herramientas de lectura.
- **Supabase / PostgreSQL** — Solicitudes · Ejecuciones · Reglas · Evidencias · Informes.
- **Nebius Token Factory** — Proveedor externo consumido exclusivamente desde el servicio de IA.

### 1.1. Qué significa esta arquitectura

- El frontend se desarrolla y despliega como una aplicación.
- El backend mantiene módulos con responsabilidades separadas.
- El motor de reglas no depende de Nebius.
- El agente puede comunicarse con el motor y la persistencia mediante interfaces controladas.
- Los servicios pueden desplegarse juntos durante el MVP y separarse posteriormente sin modificar los contratos públicos.
- PostgreSQL es la fuente persistente de verdad; el modelo de IA nunca se convierte en la autoridad sobre las decisiones.

> **Importante:** no se recomienda implementar un API Gateway independiente, un broker de eventos y cuatro microservicios desplegados desde el primer día. Para 15 días, la API de Next.js puede cumplir la función de entrada y orquestación. La separación física de servicios se hará cuando aporte aislamiento o escalabilidad real.

### 1.2. Stack tecnológico

| Capa | Tecnología | Función |
|---|---|---|
| Frontend | Next.js, React, TypeScript | Interfaz de usuario |
| API | Next.js Route Handlers | Endpoints y orquestación |
| Motor de reglas | TypeScript puro | Evaluación determinista |
| Agente | Nebius Token Factory | Investigación y explicación |
| Base de datos | Supabase, PostgreSQL | Persistencia |
| Validación | Zod | Esquemas y contratos |
| Pruebas unitarias | Vitest | Motor, validaciones y servicios |
| Pruebas E2E | Playwright | Flujos completos |
| Control de versiones | GitHub | Ramas, revisiones y CI |

No se recomienda introducir FastAPI en paralelo salvo que exista una implementación previa que lo justifique. Dos frameworks de backend desde cero añaden contratos, despliegues y mantenimiento adicionales sin resolver una necesidad del MVP.

## 2. Objetivos y límites del MVP

### 2.1. Funcionalidades incluidas

- Panel de control de ejecuciones.
- Simulador de solicitudes de pago.
- Cinco reglas de negocio.
- Evaluación determinista.
- Bloqueo de ejecuciones con infracciones.
- Investigación con herramientas de lectura.
- Integración con Nebius.
- Informe estructurado de evidencias.
- Historial persistido.
- Manejo de fallos del proveedor.
- Pruebas de contrato, reglas e integración.

### 2.2. Funcionalidades excluidas

- Transferencias bancarias reales.
- Integraciones con bancos o ERP.
- Constructor visual genérico de workflows.
- Editor de políticas arbitrarias por usuarios.
- Agentes autónomos con permisos de escritura.
- Sistema completo de roles empresariales y administración multiempresa.
- Orquestación distribuida compleja.
- Despliegue obligatorio de todos los módulos como microservicios independientes.

El simulador puede modificar parámetros de prueba, pero no puede cambiar por sí mismo las políticas confiables ni falsificar una aprobación.

### 2.3. Criterios generales de aceptación

El MVP se considera funcional cuando un pago de $8,500 MXN puede superar las reglas configuradas, mientras que uno de $15,000 MXN sin aprobación válida queda bloqueado y produce un informe de investigación verificable.

La aplicación debe seguir mostrando la decisión correcta aunque Nebius no responda.

## 3. Estructura del repositorio

**Decisión actualizada:** en vez de un único repositorio desde el día 1, el equipo trabaja con **un repositorio por carpeta/módulo**, cada uno con su propio responsable y su propio historial, bajo la organización `FlowGuard-Hackathon`: [`flowguard-docs`](https://github.com/FlowGuard-Hackathon/flowguard-docs) (este repositorio), [`flowguard-supabase`](https://github.com/FlowGuard-Hackathon/flowguard-supabase), [`flowguard-src`](https://github.com/FlowGuard-Hackathon/flowguard-src), [`flowguard-app`](https://github.com/FlowGuard-Hackathon/flowguard-app), [`flowguard-mocks`](https://github.com/FlowGuard-Hackathon/flowguard-mocks) y [`flowguard-tests`](https://github.com/FlowGuard-Hackathon/flowguard-tests). Esto permite que cada developer empuje a su propio repo sin esperar revisión de los demás mientras los contratos compartidos (este repositorio) se mantienen estables.

Al final, los seis repositorios se integran en **[`flowguard`](https://github.com/FlowGuard-Hackathon/flowguard)** mediante `git subtree`, que conserva cada uno como una carpeta normal dentro de un único repositorio combinado — sin perder la posibilidad de seguir empujando a cada repo individual si hiciera falta. El árbol resultante en `flowguard` es el siguiente:

```
flowguard/
├── app/
│   ├── dashboard/
│   ├── simulator/
│   ├── evidence/
│   └── api/
│       └── v1/
│           ├── executions/
│           ├── rules/
│           └── health/
├── src/
│   ├── contracts/
│   │   ├── payment.schema.ts
│   │   ├── execution.schema.ts
│   │   ├── engine.schema.ts
│   │   └── evidence-report.schema.ts
│   ├── server/
│   │   ├── rules/
│   │   ├── executions/
│   │   ├── persistence/
│   │   ├── agent/
│   │   └── tools/
│   └── shared/
├── supabase/
│   └── migrations/
├── mocks/
│   ├── payment-approved.json
│   ├── payment-blocked.json
│   └── investigation-failed.json
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── contract/
│   ├── e2e/
│   └── fixtures/
├── docs/
│   ├── README.md
│   ├── architecture.md
│   ├── api-contracts.md
│   ├── rules.md
│   ├── database.md
│   ├── security.md
│   ├── testing.md
│   ├── demo-script.md
│   ├── error-catalog.md
│   └── implementation-checklist.md
├── .env.example
├── package.json
└── README.md
```

> **Nota sobre el estado actual:** mientras `flowguard-src`, `flowguard-app`, `flowguard-mocks` y `flowguard-tests` solo tengan un README inicial, el árbol anterior describe el **destino acordado** una vez el equipo haya empezado a trabajar y se haya hecho el `subtree` final — no el contenido real de cada repositorio hoy. Ver el README de cada repositorio individual para su alcance exacto en este momento.

### 3.1. Propiedad de carpetas

| Developer | Propiedad principal | No debe modificar sin coordinación |
|---|---|---|
| Dev 1 | `server/rules/`, `executions/`, `persistence/`, migraciones y API de entrada | Cliente Nebius y lógica interna del agente |
| Dev 2 | `app/dashboard/`, `app/simulator/`, `app/evidence/` y componentes | Reglas de negocio del backend |
| Dev 3 | `server/agent/`, `server/tools/` y adaptador Nebius | Decisión determinista y esquema de persistencia |
| Dev 4 | `mocks/`, `tests/`, `docs/` y coordinación de contratos | No modifica módulos ajenos sin acuerdo con su responsable |

`src/contracts/` es una zona compartida. Dev 4 coordina sus versiones, pero Dev 1 y Dev 3 deben aprobar los cambios que afecten sus interfaces.

### Reglas de colaboración

- Cada developer crea ramas cortas.
- Todo cambio relevante pasa por pull request.
- La rama principal debe permanecer funcional.
- Los cambios de contrato deben incluir fixtures y pruebas actualizadas.
- La integración se ejecuta diariamente.
- Ningún módulo debe depender de implementaciones privadas de otro módulo si puede depender de una interfaz estable.

## 4. Arquitectura lógica y recorrido de una ejecución

### 4.1. Flujo de datos

1. El usuario configura una solicitud en el simulador.
2. El frontend envía la solicitud a `POST /api/v1/executions`.
3. El backend valida el contrato y obtiene los datos confiables del solicitante, la autorización, el presupuesto y los adjuntos.
4. El motor evalúa las cinco reglas.
5. La decisión y los resultados se guardan en PostgreSQL.
6. Si existe una infracción o un dato obligatorio no verificable, la decisión queda en `BLOCK`.
7. El servicio de IA recibe el identificador de ejecución y consulta las herramientas de lectura.
8. El agente prepara el informe de evidencias.
9. El backend valida las referencias, guarda el informe y conserva la decisión original.
10. El frontend muestra el resultado y el historial.

### 4.2. Secuencia técnica

Participantes: Usuario → Frontend Next.js → API/Orquestador → Motor de reglas → PostgreSQL → Servicio IA → Nebius Token Factory.

1. Usuario configura solicitud.
2. Frontend envía `POST /api/v1/executions`.
3. API valida contrato e identidad.
4. API obtiene contexto confiable (presupuesto, permisos, adjuntos).
5. Motor evalúa solicitud y políticas → `ALLOW` o `BLOCK` + resultados.
6. API persiste decisión y evidencias en PostgreSQL → confirmación de persistencia.
7. API solicita al servicio de IA investigar `execution_id`.
8. Servicio de IA solicita herramientas de lectura a Nebius.
9. Nebius consulta reglas y evidencias → recibe registros verificados.
10. Nebius analiza evidencia → produce informe estructurado.
11. Servicio de IA valida el informe y lo persiste.
12. API responde con decisión + estado de investigación.
13. Frontend muestra resultado y evidencia al usuario.

El diagrama muestra el recorrido conceptual. En la implementación, el servicio de IA podrá solicitar herramientas varias veces, dentro de los límites configurados.

> **Invariante de seguridad:** la llamada a Nebius solo puede comenzar después de persistir la decisión del motor. Si la persistencia falla, el sistema no debe presentar la operación como permitida.

### 4.3. Separación entre decisión e investigación

Hay **tres** resultados independientes, nunca colapsados en un solo campo (ver el modelo completo en [database.md § 2](database.md#2-modelo-de-estados-unificado)):

- `decision`: el resultado de negocio de las reglas (`ALLOW`/`BLOCK`).
- `execution_status`: el progreso técnico del pipeline de evaluación/persistencia.
- `investigation_status`: el progreso técnico del trabajo de la IA.

Ejemplo:

```json
{
  "decision": "BLOCK",
  "execution_status": "PERSISTED",
  "investigation_status": "FAILED"
}
```

Esto significa que el pago sigue bloqueado y que la ejecución se guardó correctamente, aunque no se haya podido generar el informe de investigación.

Nunca se debe interpretar `investigation_status = COMPLETED` como una autorización ni permitir que un informe modifique `decision`. Tampoco se usa `COMPLETED` para `execution_status`, ni `ALLOW`/`BLOCK` como estado técnico.

## 5. Resiliencia, escalabilidad y aislamiento de fallos

Este apartado es una de las decisiones más importantes de FlowGuard. La arquitectura debe permitir que un componente falle sin invalidar las decisiones ya tomadas.

### 5.1. Comportamiento ante fallos

| Fallo | Comportamiento esperado |
|---|---|
| Nebius no responde | Conservar decisión y marcar investigación fallida |
| Nebius genera JSON inválido | Rechazar informe y registrar error |
| Referencia de evidencia inexistente | Rechazar el hallazgo no verificable |
| Base de datos no disponible | No declarar una operación autorizada sin persistencia |
| Frontend pierde conexión | Mantener registros persistidos; permitir consulta posterior |
| Solicitud duplicada | Resolver mediante idempotencia |
| Presupuesto no verificable | Bloquear por condición indeterminada |
| Servicio de IA reiniciado | Reintentar la investigación sin recalcular automáticamente la decisión |
| Servicio de reglas no disponible | No autorizar por defecto |

### 5.2. Estrategia de reintentos

Los reintentos deben aplicarse solamente a errores transitorios y a operaciones idempotentes.

Para Nebius:

- Establecer un timeout inicial de 15 segundos, configurable.
- Limitar los intentos de recuperación.
- Aplicar una pausa progresiva entre reintentos.
- No repetir llamadas de herramientas indefinidamente.
- Registrar el código de error y la duración de la solicitud.
- No convertir un error del proveedor en un resultado `ALLOW`.

Los límites de cinco llamadas a herramientas y de reintentos al proveedor son controles distintos. Una llamada fallida no debe producir un ciclo ilimitado de nuevas herramientas.

### 5.3. Comunicación entre servicios

Si el servicio de IA se despliega independientemente, se comunicará con el backend mediante HTTP interno y contratos JSON versionados.

El motor de reglas puede permanecer dentro del servicio de ejecución durante el MVP. Si posteriormente se separa, debe exponer una interfaz versionada y no permitir que el agente lo use para cambiar la decisión persistida.

Para esta etapa no es necesario introducir Kafka, RabbitMQ o un sistema de eventos. La investigación puede iniciarse mediante una llamada controlada del backend y, si se necesita desacoplarla, mediante una tarea asíncrona persistida.

### 5.4. Cómo evolucionar a microservicios

1. Mantener módulos independientes dentro del repositorio.
2. Definir contratos y pruebas de integración.
3. Extraer primero el servicio de IA, porque depende de un proveedor externo y necesita límites de tiempo propios.
4. Evaluar posteriormente la extracción del motor de reglas si requiere despliegue, escalabilidad o aislamiento independiente.
5. Mantener la base de datos como fuente de verdad, con accesos restringidos y contratos de persistencia estables.

El objetivo es poder separar servicios sin reescribir el frontend ni cambiar el significado de las decisiones.

## 6. Frontend y experiencia de usuario

El frontend tiene tres módulos principales. Todos consumen la API y comparten los mismos tipos de respuesta; ninguno implementa sus propias reglas de autorización.

### 6.1. Panel de control

Debe mostrar:

- Total de ejecuciones.
- Ejecuciones permitidas.
- Ejecuciones bloqueadas.
- Investigaciones pendientes o fallidas.
- Historial de solicitudes.
- Acceso al detalle de cada ejecución.

Cada fila del historial debe identificar el importe, el solicitante, la decisión, el estado de investigación y la fecha de ejecución.

### 6.2. Simulador de pagos

El simulador permite modificar los datos del escenario, incluyendo:

- Importe.
- Categoría.
- Solicitante de prueba.
- Estado de autorización dentro del fixture.
- Referencia de factura.
- Presupuesto de demostración.

Los campos de autorización y verificación solo pueden seleccionar registros de prueba controlados por el backend. No pueden convertir una solicitud en aprobada mediante un valor enviado libremente por el cliente.

### 6.3. Centro de evidencias

El centro muestra:

- Decisión del motor.
- Reglas aprobadas, fallidas e indeterminadas.
- Valores observados y esperados.
- Referencias de evidencia.
- Informe del agente.
- Limitaciones de la investigación.
- Acción correctiva propuesta.

Debe distinguir claramente entre una infracción confirmada y una condición que no se pudo verificar.

### 6.4. Estados de interfaz

El frontend debe contemplar los siguientes estados:

- Cargando.
- Solicitud recibida.
- Evaluando reglas.
- Bloqueado o permitido.
- Investigación en curso.
- Informe completo.
- Informe parcial.
- Investigación fallida.
- Error de API.

No es necesario introducir WebSockets para el MVP. Se puede utilizar una respuesta de API que incluya el resultado inicial y, cuando la investigación se ejecute de forma asíncrona, consultar periódicamente el detalle de la ejecución durante un intervalo limitado.

La interfaz debe mostrar datos reales del backend. No debe simular que una regla ha sido verificada o que Nebius ha terminado una investigación cuando esto no haya ocurrido.

## 7. Mecanismo de investigación asíncrona

La documentación previa establecía que la investigación "se programa después de confirmar la persistencia", pero no fijaba **cómo** avanza de `PENDING` a un estado terminal una vez programada. Esta sección cierra esa decisión.

### 7.1. Alternativas consideradas

| Alternativa | Ventaja | Por qué no es la elegida para el MVP |
|---|---|---|
| **A. Iniciar la investigación en el mismo proceso de servidor, justo después de persistir, sin tabla de trabajo** | Más simple de programar al inicio | No sobrevive un reinicio del proceso ni una caída a mitad de la llamada a Nebius: si el servidor se reinicia mientras la llamada está en curso, el trabajo se pierde silenciosamente y nada vuelve a intentarlo. Incompatible con el requisito de que "los trabajos pendientes no deben perderse silenciosamente" |
| **B. Tarea pendiente persistida en PostgreSQL (`investigation_attempts`), procesada por un worker** | Sobrevive reinicios porque el estado vive en la base, no en memoria de proceso; recuperación de trabajos abandonados mediante un *reaper*; sin dependencias nuevas (usa PostgreSQL, que ya es la base oficial) | Requiere un proceso de servidor de larga duración para el bucle de sondeo (ver § 7.3) |
| **C. Cola o servicio externo (p. ej. un broker de mensajes, o un servicio de colas gestionado)** | Entrega garantizada y reintentos gestionados por la plataforma | Agrega una dependencia operativa nueva (credenciales, despliegue, monitoreo propio) que la sección 2 de este documento ya descarta explícitamente para el MVP de 15 días, sin que la fiabilidad adicional sea necesaria: PostgreSQL ya ofrece las garantías que se necesitan (ver § 7.2) |

**Decisión: alternativa B.** Una tabla de trabajo en PostgreSQL (`investigation_attempts`, que de todos modos existe para el historial de intentos) actúa también como cola persistente, reclamada por un worker mediante bloqueo a nivel de fila. El detalle de las sentencias SQL exactas (reclamo, *reaper*, *fencing* contra resultados tardíos) está en [database.md § 5](database.md#5-concurrencia-y-recuperación-del-mecanismo-asíncrono-de-investigación); esta sección solo justifica la elección y fija el supuesto de despliegue.

### 7.2. Por qué la alternativa B cumple los requisitos obligatorios

- **La decisión se calcula y persiste antes de que exista ninguna llamada a Nebius:** la transacción de [database.md § 3](database.md#3-persistencia-segura) confirma `executions`/`decision` y el primer `investigation_attempts` (`PENDING`) juntos, de forma atómica, pero ningún paso de esa transacción llama al proveedor.
- **La respuesta inicial no depende de que Nebius termine:** `POST /executions` responde en cuanto confirma su transacción. La llamada real al proveedor ocurre después, de forma asíncrona, cuando el worker reclama el intento (database.md § 5.2). El cliente recibe la decisión sin esperar a que esa llamada empiece siquiera.
- **Una caída de Nebius no cambia `decision`:** ningún paso del worker escribe en `executions.decision`; solo escribe `investigation_status`, y solo a través de `fn_finalize_investigation_attempt` (database.md § 5.4), que es atómica.
- **Mecanismo concreto que avanza `PENDING → RUNNING → terminal`:** el bucle del worker (database.md § 5.2) y `fn_finalize_investigation_attempt` (§ 5.4), ambos operando sobre trabajos creados exclusivamente por `fn_schedule_investigation_attempt` (§ 5.1).
- **Los trabajos pendientes no se pierden silenciosamente:** viven en `investigation_attempts`, no en memoria; un reinicio del worker simplemente deja de sondear hasta que vuelve a levantarse, y entonces retoma los `PENDING` existentes. El caso límite de que la propia creación del trabajo fallara a medias ya no existe por construcción (se fusionó con la transacción de la decisión), y el reconciliador de database.md § 5.6 cubre el residuo de riesgo que esa garantía no alcanza (datos heredados, bugs futuros).
- **Los reintentos no duplican investigaciones activas:** el bloqueo `SELECT ... FOR UPDATE` dentro de `fn_schedule_investigation_attempt` (database.md § 5.1) impide crear un segundo intento mientras uno sigue `PENDING`/`RUNNING`, sin importar si quien la invoca es `POST /investigate`, la transacción de creación o el reconciliador.
- **Recuperación tras un reinicio:** al arrancar, el worker no necesita ningún estado especial de "recuperación" distinto de su ciclo normal: simplemente vuelve a sondear `investigation_attempts` por `status = 'PENDING'` (trabajos que nunca llegaron a reclamarse), el *reaper* resuelve los que quedaron `RUNNING` sin resolverse (database.md § 5.3), y el reconciliador resuelve los que quedaron `NOT_STARTED` sin ningún intento asociado (database.md § 5.6).
- **Ninguna finalización queda parcialmente confirmada:** marcar un intento como terminal, persistir su informe y propagar el resultado a `executions` ocurre dentro de una sola función PL/pgSQL (`fn_finalize_investigation_attempt`, database.md § 5.4) invocada como una sola sentencia — no como varias llamadas HTTP envueltas en una transacción de aplicación, que no daría atomicidad real si el proceso muere entre una llamada y la siguiente.
- **Timeout, concurrencia y fallos:** `AGENT_TIMEOUT_MS` acota cada llamada individual a Nebius; `AGENT_WORKER_CONCURRENCY` acota cuántos intentos procesa el worker a la vez (para no saturar el rate limit del proveedor, pendiente de confirmar en [security.md § 1.8](security.md#18-lista-de-verificación-de-compatibilidad-con-nebius)); el umbral de abandono del *reaper* (`2 × AGENT_TIMEOUT_MS + 10000` ms recomendado) determina cuánto tiempo puede estar un intento `RUNNING` antes de considerarse perdido; `RECONCILER_INTERVAL_MS` acota cada cuánto se ejecuta el reconciliador de § 5.6.

### 7.3. Supuesto de despliegue y alternativa para hosting serverless

**Supuesto por defecto:** el backend de FlowGuard se despliega como **un proceso Node.js de larga duración** (por ejemplo, un servicio en Render, Railway, Fly.io, un contenedor en un VM, o cualquier plataforma que mantenga el proceso vivo entre solicitudes). Bajo este supuesto, el propio proceso del servidor Next.js ejecuta el bucle de sondeo del worker (p. ej. iniciado una sola vez al arrancar el servidor, con un intervalo de `AGENT_WORKER_POLL_INTERVAL_MS`). Esta es una propiedad del hosting elegido, **no** una limitación universal de Next.js: Next.js no impide procesos de fondo; lo que los impide es específicamente un entorno de ejecución *serverless* de vida ultracorta como las Funciones Serverless de Vercel en su modo estándar, donde el proceso se congela o termina entre solicitudes y no hay garantía de que un `setInterval` siga corriendo.

**Alternativa compatible si el equipo despliega en una plataforma estrictamente serverless** (sin proceso persistente disponible): sustituir el bucle de sondeo en memoria por una función programada (p. ej. un *cron job* de la plataforma, invocada cada 1–2 minutos) que llama a un endpoint interno protegido (no público) y ejecuta un solo ciclo de § 7.2 (reclamar hasta `AGENT_WORKER_CONCURRENCY` trabajos, procesarlos, y dejar que la propia invocación termine). La tabla `investigation_attempts` y las sentencias con `FOR UPDATE SKIP LOCKED` funcionan igual en ambos casos porque la garantía vive en PostgreSQL, no en el proceso que la invoca; lo único que cambia es quién dispara cada ciclo. La latencia percibida por el usuario entre `PENDING` y `RUNNING` sube de "casi inmediata" (worker persistente sondeando cada pocos segundos) a "hasta el intervalo del cron" (típicamente 1–2 minutos), lo cual es aceptable para una demo de hackathon pero debe comunicarse como una diferencia real, no ocultarse.

**Decisión pendiente de equipo:** qué plataforma de hosting se usa en la práctica determina cuál de las dos variantes aplica. Responsable: Dev 1, con el resto del equipo informado, a más tardar el día 3 (bloquea el diseño del endpoint interno del worker). Opción por defecto si no se decide antes: supuesto de proceso persistente (variante principal), por ser la que da menor latencia y no requiere un endpoint interno adicional.

## Plan de trabajo de 15 días (resumen)

La planificación se divide en cinco bloques de tres días. Dev 4 coordina contratos, fixtures y pruebas desde el primer día para que los otros tres developers trabajen en paralelo.

| Bloque | Días | Foco | Hito obligatorio |
|---|---|---|---|
| 1 | 1–3 | Contratos congelados, esqueleto del motor de reglas, frontend con mocks, adaptador Nebius con mock | Base común de integración lista |
| 2 | 4–6 | Persistencia real, simulador conectado a la API, herramientas de lectura | Recorrido completo (sin IA real) funcionando end-to-end |
| 3 | 7–9 | Integración real de Nebius, centro de evidencias, pruebas E2E | Ejecución bloqueada produce informe real o `investigation_status = FAILED` explícito |
| 4 | 10–12 | Resiliencia, idempotencia, estados de error, regresión | MVP end-to-end estable incluso si Nebius falla |
| 5 | 13–15 | Congelación, ensayo de demo, documentación final | Versión estable, demo ensayada |

El desglose día a día por developer — entregable concreto, dependencias, criterio de aceptación, prueba que lo demuestra, riesgo de bloqueo y alternativa con mocks — vive en **[implementation-checklist.md](implementation-checklist.md)**, para no duplicar el mismo contenido en dos documentos. La prioridad no es terminar cada módulo aisladamente: es obtener pronto una integración vertical funcional (objetivo del día 6) y ampliar la cobertura sin romperla.

## Decisiones que deben quedar congeladas el día 1

| Decisión | Acuerdo |
|---|---|
| Frontend | Aplicación monolítica Next.js/React |
| Backend | Modular, con servicios independientes cuando aporten valor |
| Motor de reglas | Código determinista independiente de la IA |
| Agente | Nebius Token Factory, con herramientas de lectura |
| Base de datos | Supabase/PostgreSQL |
| Reglas | Cinco reglas obligatorias |
| Umbral de autorización | $10,000 MXN en la política de demo |
| Escenario válido | $8,500 MXN |
| Escenario bloqueado | $15,000 MXN sin aprobación |
| Límite de herramientas | Cinco llamadas por investigación |
| Informe parcial | Permitido si las afirmaciones y referencias son verificables |
| Persistencia | La decisión se guarda antes de investigar |
| Regla temporal | Sustituida por verificación de factura |
| Transferencias reales | Fuera del alcance |
| Objetivo del día 6 | Primer recorrido completo sin dependencia de la IA real |

## Conclusión arquitectónica

Para FlowGuard, la prioridad es conseguir aislamiento de fallos sin convertir un hackathon de 15 días en un proyecto de infraestructura.

El frontend monolítico facilita el trabajo de Dev 2. Los módulos independientes permiten que Dev 1 y Dev 3 trabajen sin pisarse. Los contratos compartidos y las pruebas de Dev 4 reducen los problemas de integración. La separación de la decisión y la investigación garantiza que una caída de Nebius no autorice una operación incorrecta.

La regla principal de diseño es innegociable: **el motor de reglas decide, el backend persiste, la IA investiga y el frontend presenta.**

Esa separación constituye la base de una solución que puede demostrarse en tres minutos y evolucionar posteriormente hacia servicios independientes sin tener que rediseñar todo el sistema.

## Referencias

- Modelo de estados unificado y esquema de persistencia: [database.md](database.md)
- Contratos de API completos: [api-contracts.md](api-contracts.md)
- Estrategia de seguridad por entorno: [security.md](security.md)
- Checklist día a día con entregables y riesgos: [implementation-checklist.md](implementation-checklist.md)
- Catálogo de errores: [error-catalog.md](error-catalog.md)
