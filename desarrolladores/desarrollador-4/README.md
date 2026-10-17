# Desarrollador 4 (Puesto 4) — CI/CD

Página de inicio del espacio de trabajo de Desarrollador 4 (Puesto 4). La persona que ocupará este puesto aún no está asignada.

## Archivos de esta carpeta

- [plan-trabajo.md](plan-trabajo.md) — plan de CI/CD **propuesto, pendiente de aprobación**, y reasignación de las tareas del plan anterior
- [bitacora.md](bitacora.md) — bitácora diaria
- [notas.md](notas.md) — cuaderno técnico personal
- [bloqueos.md](bloqueos.md) — bloqueos registrados

## Qué posee este puesto

La integración continua y el despliegue continuo de FlowGuard: los workflows de GitHub Actions, las comprobaciones automatizadas de los repositorios y la documentación operativa de los pipelines. No hay workflows creados todavía en ninguno de los siete repositorios (verificado en `main`). El repositorio de trabajo se determina según la estructura real del proyecto; no se crea un repositorio nuevo ni se introduce una plataforma distinta de GitHub, que [arquitectura.md](../../arquitectura.md) ya fija para ramas, revisiones y CI.

## Responsabilidades

- Configurar y mantener GitHub Actions.
- Automatizar lint, comprobación de tipos, pruebas y build **cuando los scripts correspondientes existan**.
- Automatizar la ejecución de las pruebas que mantienen los propietarios de cada módulo.
- Detectar fallos de integración mediante los resultados de los pipelines y comunicarlos.
- Mantener los workflows y documentar cómo se ejecutan y diagnostican.
- Coordinar la automatización de despliegue únicamente conforme a la infraestructura ya acordada.
- Ayudar a que la integración del repositorio final `flowguard` (`git subtree`) sea reproducible.
- Aportar resultados reproducibles de CI como evidencia **complementaria** de los hitos de los días 6, 9, 12 y 15.

## Qué NO es este puesto

No es responsable funcional de QA, de los contratos, de los fixtures, de los módulos de aplicación ni de `flowguard-tests`. El propietario técnico corrige los defectos funcionales que el pipeline detecte. Las tareas del plan anterior de este puesto (contratos, fixtures, pruebas, QA, ensayos y revisiones) se reasignaron: ver la tabla en [plan-trabajo.md](plan-trabajo.md#tareas-del-plan-anterior-de-desarrollador-4-y-su-nuevo-responsable).

## Cómo coordinarse con el resto del equipo

- Con Desarrollador 1: comprobaciones y despliegue del backend, y la integración técnica del backend en `flowguard`.
- Con Desarrollador 2: comprobaciones del frontend.
- Con Desarrollador 3: verificaciones del agente y su proveedor, sin exponer credenciales.
- Con Desarrollador 5: contratos y mocks necesarios para validar las integraciones.
- Ver el detalle completo de dependencias en [integracion-y-dependencias.md](../../integracion-y-dependencias.md).

## Reglas de actualización y colaboración

1. Actualiza tu bitácora ([bitacora.md](bitacora.md)) al terminar tu jornada.
2. Tus notas ([notas.md](notas.md)) deben incluir contexto suficiente para que otro integrante las entienda sin preguntar.
3. Registra y comunica los bloqueos importantes en [bloqueos.md](bloqueos.md).
4. Las decisiones que afecten a la arquitectura o a varios módulos también deben reflejarse en la documentación general (`plan-desarrollo.md`, `integracion-y-dependencias.md`, o el documento técnico correspondiente).
5. Una tarea no se considera completada solo porque esté anotada como terminada: debe existir evidencia verificable, según [definicion-de-terminado.md](../../definicion-de-terminado.md).
6. Los archivos de esta carpeta pueden modificarse mediante Pull Requests pequeños.
7. Los documentos generales de la raíz del repositorio son compartidos y no deben contradecir los contratos técnicos existentes.

Estos archivos Markdown son compartidos: los cinco puestos pueden proponer cambios si tienen permisos de escritura en GitHub. La carpeta organiza el trabajo, no establece permisos privados.
