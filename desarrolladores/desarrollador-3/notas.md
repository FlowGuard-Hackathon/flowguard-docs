# Cuaderno técnico — Desarrollador 3

Notas personales de trabajo. Cada nota debe incluir fecha, contexto y estado, para que otro integrante del equipo pueda entenderla sin preguntar. No se registran secretos, tokens, contraseñas ni credenciales aquí ni en ningún otro archivo de este repositorio.

## Decisiones técnicas

<!-- Formato sugerido por nota:
### [fecha] Título de la decisión
- Contexto:
- Decisión tomada:
- Alternativas consideradas:
- Estado: Propuesta | Aprobada | Revertida
-->

### [2026-10-16] Esqueleto inicial del módulo del agente en flowguard-src

- Contexto: inicio del Bloque 1 (días 1-3) del plan. Había que arrancar el adaptador de Nebius sin depender todavía de credenciales reales ni de la persistencia de Dev 1.
- Decisión tomada: inicializar `flowguard-src` con TypeScript + Zod + Vitest, y construir primero el proveedor **mock** (`ProveedorAgenteSimulado`) en vez de empezar directo con `ProveedorAgenteNebius`, siguiendo la secuencia del plan (día 2 = mock, día 7-9 = integración real).
- Alternativas consideradas: ninguna — el plan ya fija el mock como paso previo obligatorio antes de la integración real (ver [checklist-implementacion.md, Bloque 1](../../checklist-implementacion.md#bloque-1--días-13-congelación-de-contratos-y-estructura-del-proyecto)).
- Estado: Aprobada (PR fusionado).
- Referencia: [flowguard-src PR #1](https://github.com/FlowGuard-Hackathon/flowguard-src/pull/1).

### [2026-10-16] Las 5 herramientas de lectura quedan como stub, no como mock de datos

- Contexto: las herramientas (`obtener_ejecucion`, `obtener_resultados_reglas`, `obtener_evidencia`, `obtener_contexto_pago`, `obtener_detalle_regla`) necesitan datos reales de `src/servidor/persistencia/`, que todavía no existe (es de Dev 1).
- Decisión tomada: cada herramienta lanza `NotImplementedError` explícito en vez de devolver datos inventados, para no simular un comportamiento que no es real.
- Alternativas consideradas: devolver datos mock por herramienta — descartada porque el plan (día 4-5) indica implementarlas "contra datos mock" solo como paso intermedio documentado, no como sustituto silencioso de la persistencia real; se prefirió dejar explícito el bloqueo.
- Estado: Revertida el 2026-10-17 por la decisión siguiente.

### [2026-10-17] Herramientas de lectura sobre una interfaz de acceso a datos

- Contexto: el día 4 pide las herramientas contra datos mock, pero Dev 1 todavía no existe como persona ni hay persistencia. Hacía falta avanzar sin inventar la persistencia y sin que Dev 1 tenga que reescribir el módulo después.
- Decisión tomada: las herramientas reciben un `FuenteDatosHerramientas` (interfaz de solo lectura con 5 métodos). Hoy se usa `FuenteDatosEnMemoria` con datos simulados; Dev 1 implementa la versión real en `src/servidor/persistencia/` y basta con pasarla a `crearHerramientasLectura`. Contrato: si no existe → `null`; si hay una falla real → `throw`, que la herramienta convierte en `DEPENDENCIA_NO_DISPONIBLE` sin filtrar detalles. Parámetros con Zod `.strict()` para rechazar parámetros de más que mande el modelo. Solo códigos del catálogo cerrado.
- Alternativas consideradas: (a) mantener los stubs: no cumplía el día 4; (b) que las herramientas consulten Supabase directo: rompe la propiedad por rutas (la persistencia es de Dev 1) y obliga a credenciales para probar.
- Estado: Fusionada en [flowguard-src PR #5](https://github.com/FlowGuard-Hackathon/flowguard-src/pull/5).

### [2026-10-17] Identificadores del sistema en español

- Contexto: equipo hispanohablante y todavía sin código de backend ni base de datos, así que el costo del cambio es mínimo.
- Decisión tomada: todo pasa al español (API, BD, errores, variables de entorno, código y documentos) según [glosario.md](../../glosario.md). Los textos visibles van en catálogos de mensajes (`src/mensajes/es.ts`) para poder agregar idiomas sin tocar la lógica. Las bitácoras no se traducen porque son registro histórico.
- Alternativas consideradas: mantener los contratos en inglés y solo traducir mensajes; descartada por decisión de coordinación.
- Estado: Aprobada; migración en curso (docs hechos, faltan supabase, src, app y el resto).

### [2026-10-17] Sin tecnologías fuera del stack

- Contexto: para repetir la verificación de la migración SQL renombrada se propuso usar Docker.
- Decisión tomada: no se agregan herramientas ni tecnologías fuera del stack documentado mientras el equipo gana experiencia. La migración se verificará al aplicarla en el proyecto Supabase de desarrollo (Dev 1) y hasta entonces queda marcada como pendiente de verificar.
- Alternativas consideradas: Docker con PostgreSQL desechable (descartada por la curva de aprendizaje del equipo).
- Estado: Aprobada.

### [2026-10-17] Convenciones de commits, ramas y comentarios

- Decisión tomada: commits con Conventional Commits en español ([convencion-commits.md](../../convencion-commits.md)), ramas `tipo/devN-descripcion` ([convencion-ramas.md](../../convencion-ramas.md)), sin líneas `Co-Authored-By` ni menciones a herramientas de IA. En el código, pocos comentarios y solo donde aportan (contratos no obvios, supuestos, trampas), en tono coloquial con términos técnicos.
- Estado: Aprobada.

## Hallazgos

<!-- Formato sugerido:
### [fecha] Título del hallazgo
- Contexto:
- Qué se encontró:
- Impacto:
- Estado: Confirmado | En investigación
-->

### [2026-10-17] La relación regla ↔ evidencia no está modelada en la base de datos

- Contexto: al definir lo que devuelve `obtener_resultados_reglas`.
- Qué se encontró: `contratos-api.md § 3` devuelve `refs_evidencia` en cada regla, pero `evaluaciones_reglas` no tiene columna para eso y `base-datos.md` no explica cómo se relacionan.
- Impacto: Dev 1 no puede implementar `obtenerResultadosReglas` sin decidirlo. Registrado como pregunta abierta.
- Estado: Confirmado.

### [2026-10-17] El repo final `flowguard` anidaría las carpetas con `git subtree`

- Qué se encontró: `flowguard-src` y `flowguard-app` traen su `package.json` en la raíz y el código en subcarpetas (`src/`, `app/`). Con `git subtree --prefix=src` / `--prefix=app` quedarían `src/src/...` y `app/app/...`.
- Impacto: hay que decidirlo antes de la primera integración (no antes del día 6). Dueños: Dev 1 y Dev 2.
- Estado: Confirmado; anotado en el README de `flowguard-app`.

### [2026-10-17] Vitest 5 pide Node 22.12+, pero las pruebas pasan con Node 20

- Qué se encontró: `npm install` en `flowguard-src` avisa `EBADENGINE` con Node 20.20; `npm test` y `typecheck` pasan igual.
- Impacto: la CI (Dev 4) debe fijar la versión de Node; conviene usar 22 para no depender de la advertencia.
- Estado: Confirmado.

### [2026-10-17] Requisitos del hackathon con conflicto o sin cubrir

- Qué se encontró: el acceso "sin restricciones" para la evaluación choca con la Basic Auth de la demo pública (`seguridad.md § 2.0`); ningún repositorio tiene licencia todavía; el guion de demo dura 3 minutos y el video debe durar menos.
- Impacto: decisiones de coordinación pendientes, registradas en [definicion-de-terminado.md](../../definicion-de-terminado.md#requisitos-del-hackathon-y-evidencias-de-cumplimiento).
- Estado: Confirmado.

## Ideas pendientes de validar

<!-- Formato sugerido:
### [fecha] Idea
- Contexto:
- Por qué podría valer la pena:
- Qué falta para validarla:
- Estado: Pendiente de validar
-->

## Comandos útiles

<!-- Formato sugerido:
### [fecha] Descripción del comando
```
comando aquí
```
- Contexto / cuándo usarlo:
-->

### [2026-10-17] Verificar `flowguard-src` antes de un PR

```
npm test
npm run typecheck
npx vitest run src/servidor/herramientas/herramientas.test.ts
```
- Contexto: las dos primeras deben pasar siempre antes de abrir un PR. La tercera corre solo la suite de las herramientas.

### [2026-10-17] Comprobar la identidad antes del primer push en un clon nuevo

```
git log -1 --format="%an <%ae>"
```
- Contexto: la configuración global de esta máquina usa otra cuenta; en los clones de FlowGuard se fijó la identidad del equipo a nivel de repositorio.

## Referencias a archivos o Pull Requests

<!-- Formato sugerido:
### [fecha] Referencia
- Archivo o PR:
- Por qué es relevante:
-->

### [2026-10-16] flowguard-src PR #1 — esqueleto del agente

- Archivo o PR: [flowguard-src#1](https://github.com/FlowGuard-Hackathon/flowguard-src/pull/1) (rama `dev3/agent-skeleton`, fusionada a `main`)
- Por qué es relevante: es el punto de partida de todo mi trabajo. Contiene:
  - `src/contratos/informe-evidencias.schema.ts` — contrato Zod del informe, según [contratos-api.md §10](../../contratos-api.md#10-contrato-del-informe-del-agente).
  - `src/servidor/agente/proveedorSimulado.ts` — proveedor mock funcional (probado con `npm test`).
  - `src/servidor/agente/proveedorNebius.ts` — stub, bloqueado hasta completar la checklist del día 7.
  - `src/servidor/agente/validarInforme.ts`, `prompt.ts`, `index.ts` (selección por `AGENTE_PROVEEDOR`).
  - `src/servidor/herramientas/` — firmas de las 5 herramientas de lectura, pendientes de `src/servidor/persistencia/` de Dev 1.
  - `proveedorSimulado.test.ts` — prueba unitaria exigida en el día 3 del plan, pasando.
- Nota: los nombres de esta lista ya siguen el [glosario](../../glosario.md); en el PR #1 los archivos se llamaban con los nombres anteriores en inglés (`src/contracts/evidence-report.schema.ts`, `src/server/agent/mockProvider.ts`, etc.).

### [2026-10-17] PRs del día

- `flowguard-docs` #4–#7: reorganización en cinco puestos. #8: glosario y convenciones de commits y ramas.
- `flowguard-src` #2: responsables por rutas. #3: esqueleto del backend. #4: guía operativa en el README.
- `flowguard-app` #2: esqueleto Next.js. #3: guía operativa en el README.
- `flowguard-tests`, `flowguard-mocks`, `flowguard` #2: responsables actualizados.
- Por qué es relevante: es la base sobre la que arrancan Dev 1 y Dev 2, y la razón de los nombres en español.

## Preguntas para el equipo

<!-- Formato sugerido:
### [fecha] Pregunta
- Contexto:
- Pregunta:
- Estado: Abierta | Respondida (anotar la respuesta y quién la dio)
-->

### [2026-10-17] Para Dev 1: lo que falta para conectar las herramientas a datos reales

- Contexto: `FuenteDatosHerramientas` ya define qué necesita el agente; faltan decisiones de persistencia.
- Preguntas:
  1. ¿De dónde salen las `refs_evidencia` de cada regla, si `evaluaciones_reglas` no tiene columna para ellas?
  2. ¿La `autorizacion_verificada` de `obtener_contexto_pago` sale de un `registros_evidencia` tipo `REGISTRO_APROBACION` o del contexto confiable? Sin registro se devuelve `null`, nunca "aprobado".
  3. ¿Cuál es el formato real de `reglas_politica.configuracion_json`? Las claves del mock son supuestos.
  4. ¿Cómo se resuelve la política activa (`definiciones_flujo.activa` o la `version_politica` de la ejecución)?
- Estado: Abierta (el puesto 1 todavía no tiene persona asignada).
