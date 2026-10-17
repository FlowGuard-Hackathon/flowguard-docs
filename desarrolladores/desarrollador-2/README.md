# Desarrollador 2 (Puesto 2) — Frontend y experiencia de usuario

Página de inicio del espacio de trabajo de Desarrollador 2 (Puesto 2). La persona que ocupará este puesto aún no está asignada.

## Archivos de esta carpeta

- [plan-trabajo.md](plan-trabajo.md) — plan completo de los 15 días
- [bitacora.md](bitacora.md) — bitácora diaria
- [notas.md](notas.md) — cuaderno técnico personal
- [bloqueos.md](bloqueos.md) — bloqueos registrados

## Módulo que posee

La aplicación Next.js de FlowGuard: el formulario de pagos simulados, el dashboard de evaluaciones, el historial y detalle de pagos, la visualización de reglas, evidencias, bloqueos y errores, y la integración con la API desde el frontend. Repositorio principal: [`flowguard-app`](https://github.com/FlowGuard-Hackathon/flowguard-app).

## Responsabilidades

- Construir el simulador de pagos y conectarlo a la API real.
- Construir el dashboard de ejecuciones, el historial y la vista de detalle.
- Construir el centro de evidencias que muestra reglas, evidencias e informe del agente sin datos inventados.
- Representar correctamente todos los estados de interfaz (carga, error, bloqueo) descritos en [arquitectura.md § 6.4](../../arquitectura.md#64-estados-de-interfaz).
- Mantener pruebas de interfaz (Playwright) contra mocks y contra la API real, incluidos los flujos E2E de usuario de tu módulo.
- Verificar el acceso a la demo desde la interfaz (Basic Auth de la demo pública, [seguridad.md § 2.0](../../seguridad.md#20-estrategia-de-seguridad-por-entorno)), preparar la experiencia de demo visible para el evaluador y dejar la aplicación lista para compilarse y comprobarse en CI.
- Distinguir en la documentación entre una interfaz conectada a mocks y una conectada al backend real.

## Cómo coordinarse con el resto del equipo

- Desde el día 1 trabajas contra las fixtures de Desarrollador 5 (`mocks/pago-aprobado.json`, `mocks/pago-bloqueado.json`), sin esperar a que la API real exista.
- Dependes de Desarrollador 1 para que `POST /api/v1/ejecuciones` y los demás endpoints estén disponibles antes de conectar el simulador real.
- Dependes de Desarrollador 3 para el contrato del informe del agente y para que `GET /ejecuciones/:id/evidencias` devuelva datos reales.
- Desarrollador 4 configura las verificaciones de CI de `flowguard-app`, pero no se convierte en propietario de tu código: sigues siendo responsable funcional del frontend.
- Para el acceso a la demo dependes de Desarrollador 1, que genera y distribuye las credenciales.
- Ver el detalle completo de dependencias en [integracion-y-dependencias.md](../../integracion-y-dependencias.md).

## Límites

No implementes una segunda fuente de verdad de las reglas de negocio, no autorices pagos desde el frontend, no inventes endpoints ni esquemas, y no incluyas credenciales privadas en código accesible al navegador.

## Reglas de actualización y colaboración

1. Actualiza tu bitácora ([bitacora.md](bitacora.md)) al terminar tu jornada.
2. Tus notas ([notas.md](notas.md)) deben incluir contexto suficiente para que otro integrante las entienda sin preguntar.
3. Registra y comunica los bloqueos importantes en [bloqueos.md](bloqueos.md).
4. Las decisiones que afecten a la arquitectura o a varios módulos también deben reflejarse en la documentación general (`plan-desarrollo.md`, `integracion-y-dependencias.md`, o el documento técnico correspondiente).
5. Una tarea no se considera completada solo porque esté anotada como terminada: debe existir evidencia verificable, según [definicion-de-terminado.md](../../definicion-de-terminado.md).
6. Los archivos de esta carpeta pueden modificarse mediante Pull Requests pequeños.
7. Los documentos generales de la raíz del repositorio son compartidos y no deben contradecir los contratos técnicos existentes.

Estos archivos Markdown son compartidos: los cinco puestos pueden proponer cambios si tienen permisos de escritura en GitHub. La carpeta organiza el trabajo, no establece permisos privados.
