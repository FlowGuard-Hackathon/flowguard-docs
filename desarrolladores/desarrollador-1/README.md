# Desarrollador 1 (Puesto 1) — Backend, motor determinista y base de datos

Página de inicio del espacio de trabajo de Desarrollador 1 (Puesto 1). La persona que ocupará este puesto aún no está asignada.

## Archivos de esta carpeta

- [plan-trabajo.md](plan-trabajo.md) — plan completo de los 15 días
- [bitacora.md](bitacora.md) — bitácora diaria
- [notas.md](notas.md) — cuaderno técnico personal
- [bloqueos.md](bloqueos.md) — bloqueos registrados

## Módulo que posee

Backend de FlowGuard: API con Next.js Route Handlers, validación con Zod, el motor determinista de las cinco reglas de negocio, la persistencia en Supabase/PostgreSQL, la idempotencia, el procesamiento asíncrono del worker de investigación, y la seguridad y manejo de errores del backend. Repositorios principales: [`flowguard-src`](https://github.com/FlowGuard-Hackathon/flowguard-src) (todo el backend **excepto** `src/servidor/agente/`, `src/servidor/herramientas/` y `src/contratos/informe-evidencias.schema.ts`, que son del puesto 3) y [`flowguard-supabase`](https://github.com/FlowGuard-Hackathon/flowguard-supabase).

## Responsabilidades

- Implementar y mantener el motor de reglas (`evaluarPago`) y las cinco reglas `PAG-001` a `PAG-005`.
- Diseñar e implementar el esquema de base de datos y aplicar las migraciones.
- Implementar los endpoints de la API (`POST /ejecuciones`, `GET /ejecuciones`, etc.) según [contratos-api.md](../../contratos-api.md).
- Implementar el worker asíncrono, el reaper y el reconciliador de ejecuciones.
- Aplicar rate limiting y los controles de seguridad del backend descritos en [seguridad.md](../../seguridad.md).
- Validar los contratos compartidos `pago`, `ejecucion` y `motor` que propone el puesto 5.
- Mantener las pruebas del motor de reglas, la persistencia y la integración del backend.
- Implementar el middleware de Basic Auth de la demo y generar y distribuir las credenciales por un canal apropiado, sin guardarlas en Git ni imprimirlas en logs ([seguridad.md § 2.0](../../seguridad.md#20-estrategia-de-seguridad-por-entorno)).
- Coordinar la integración técnica del backend en el repositorio final `flowguard`, siguiendo la estrategia `git subtree` ya documentada.
- Colaborar con el puesto 4 para que los pipelines de CI ejecuten las verificaciones pertinentes del backend, y ser responsable técnico de la ejecución y configuración propia del backend.

## Cómo coordinarse con el resto del equipo

- Desarrollador 2 depende de que los endpoints de la API estén disponibles y respeten el contrato congelado — cualquier cambio de contrato se documenta primero en este repositorio.
- Desarrollador 3 depende de la migración aplicada para que las herramientas de lectura del agente tengan datos reales contra los que operar. Tú mantienes el acceso a datos; Desarrollador 3 define qué información necesita el agente. Acuerden las interfaces antes de implementar (ver [integracion-y-dependencias.md](../../integracion-y-dependencias.md)).
- Desarrollador 5 propone los contratos `pago`, `ejecucion` y `motor` para que los valides, y mantiene las fixtures que necesita el backend.
- Desarrollador 4 automatiza en CI las verificaciones de tu módulo; las pruebas del backend las mantienes tú. Los hitos de los días 6, 9, 12 y 15 se verifican de forma cruzada, no por una sola persona.
- Ver el detalle completo de dependencias en [integracion-y-dependencias.md](../../integracion-y-dependencias.md).

## Límites

No delegues al LLM la decisión final de autorización, no cambies las reglas de negocio ni los contratos compartidos por tu cuenta, y no asumas el trabajo del frontend o del agente de IA.

## Reglas de actualización y colaboración

1. Actualiza tu bitácora ([bitacora.md](bitacora.md)) al terminar tu jornada.
2. Tus notas ([notas.md](notas.md)) deben incluir contexto suficiente para que otro integrante las entienda sin preguntar.
3. Registra y comunica los bloqueos importantes en [bloqueos.md](bloqueos.md).
4. Las decisiones que afecten a la arquitectura o a varios módulos también deben reflejarse en la documentación general (`plan-desarrollo.md`, `integracion-y-dependencias.md`, o el documento técnico correspondiente).
5. Una tarea no se considera completada solo porque esté anotada como terminada: debe existir evidencia verificable, según [definicion-de-terminado.md](../../definicion-de-terminado.md).
6. Los archivos de esta carpeta pueden modificarse mediante Pull Requests pequeños.
7. Los documentos generales de la raíz del repositorio son compartidos y no deben contradecir los contratos técnicos existentes.

Estos archivos Markdown son compartidos: los cinco puestos pueden proponer cambios si tienen permisos de escritura en GitHub. La carpeta organiza el trabajo, no establece permisos privados.
