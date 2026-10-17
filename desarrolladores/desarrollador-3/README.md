# Desarrollador 3 (Puesto 3) — Agente de IA, LLMs, Nebius Token Factory y modelos NVIDIA

Página de inicio del espacio de trabajo de Desarrollador 3 (Puesto 3). Este puesto está reservado para quien coordina el proyecto y permanece dedicado al agente de IA y a los LLMs.

## Archivos de esta carpeta

- [plan-trabajo.md](plan-trabajo.md) — plan completo de los 15 días
- [bitacora.md](bitacora.md) — bitácora diaria
- [notas.md](notas.md) — cuaderno técnico personal
- [bloqueos.md](bloqueos.md) — bloqueos registrados

## Módulo que posee

El agente de IA de FlowGuard: el contrato del informe del agente, el proveedor simulado (mock) y la integración real con Nebius Token Factory, las herramientas de solo lectura, el ciclo de investigación, la validación de respuestas y la gestión de fallos del proveedor. Módulo dentro de [`flowguard-src`](https://github.com/FlowGuard-Hackathon/flowguard-src).

**Rutas de propiedad en `flowguard-src`:** `src/servidor/agente/`, `src/servidor/herramientas/` y `src/contratos/informe-evidencias.schema.ts`. El resto del backend es del puesto 1. Las rutas definen propiedad de mantenimiento, no permiso para trabajar sin coordinación. El puesto 5 puede ayudar con la compatibilidad de los contratos compartidos, pero no modifica `informe-evidencias.schema.ts` por su cuenta ni se apropia de él.

## Responsabilidades

- Definir y mantener el contrato del informe del agente ([contratos-api.md § 10](../../contratos-api.md#10-contrato-del-informe-del-agente)).
- Integrar el modelo NVIDIA de código abierto compatible con los requisitos del hackathon y documentar, con evidencia verificable, las llamadas reales a Nebius Token Factory y el uso de ese modelo.
- Mantener los prompts y las pruebas del agente y del proveedor (respuestas válidas, JSON inválido, referencias inválidas, timeouts y errores del proveedor), y verificar que el sistema siga funcionando con el proveedor simulado cuando no haya credenciales reales.
- Construir el adaptador de Nebius, primero en modo mock y después con la integración real.
- Implementar las herramientas de lectura (`obtener_ejecucion`, `obtener_resultados_reglas`, `obtener_evidencia`, `obtener_contexto_pago`, `obtener_detalle_regla`).
- Completar la checklist de compatibilidad con Nebius ([seguridad.md § 1.8](../../seguridad.md#18-lista-de-verificación-de-compatibilidad-con-nebius)).
- Validar los informes del agente y gestionar timeouts, JSON inválido y referencias inválidas sin alterar la `decision` ya persistida.

## Cómo coordinarse con el resto del equipo

- Dependes de Desarrollador 5 para la coordinación de compatibilidad del contrato del informe del agente, que se congela el día 1, y para la fixture `investigacion-fallida.json`. Tú sigues siendo el propietario del esquema.
- Dependes de Desarrollador 1 para que la migración esté aplicada y `POST /ejecuciones` esté disponible, de forma que las herramientas de lectura tengan datos reales contra los que operar. Tú defines qué información necesita el agente; Desarrollador 1 mantiene el acceso a datos. No creas mecanismos alternativos de acceso a datos para eludir la arquitectura, y Desarrollador 1 no cambia unilateralmente los contratos de tus herramientas.
- Desarrollador 4 automatiza en CI las verificaciones del agente, pero las pruebas del agente las mantienes tú. No expongas credenciales en pipelines ni en logs.
- Los hitos de los días 6, 9, 12 y 15 se verifican de forma cruzada; no apruebas en solitario los componentes que desarrollaste.
- Además de tu rol técnico, coordinas las prioridades generales del proyecto. Eso **no** te hace responsable de todas las tareas de entrega: la demo, el README final, el video y las evidencias se reparten entre los puestos (ver [definicion-de-terminado.md](../../definicion-de-terminado.md#requisitos-del-hackathon-y-evidencias-de-cumplimiento)).
- Desarrollador 2 depende de tu contrato del informe y de `GET /ejecuciones/:id/evidencias` para construir el centro de evidencias.
- La IA investiga y explica, pero **no puede** autorizar pagos, modificar reglas ni ejecutar operaciones de escritura — es una regla de diseño no negociable.
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
