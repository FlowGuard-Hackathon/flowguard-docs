# Definition of Done

Este documento amplía, a nivel de todo el equipo y de cada tarea individual, los "Criterios de salida" y la "Definition of Done" ya definidos en [pruebas.md § 4](pruebas.md#4-criterios-de-salida). No los sustituye: si hay cualquier discrepancia, [pruebas.md](pruebas.md) es la fuente de verdad técnica sobre pruebas, y este documento es la checklist operativa que cada desarrollador aplica antes de marcar una tarea como terminada.

## Principio general

Una tarea no se considera terminada porque esté anotada como terminada en un `plan-trabajo.md` o en una bitácora. Debe existir evidencia verificable: una prueba que pasa, una respuesta real de la API, una captura de la UI, o una revisión de otro desarrollador.

## Criterios obligatorios por tarea

Antes de marcar una tarea como `Completado` en cualquier `plan-trabajo.md` o `bitacora.md`, debe cumplirse todo lo aplicable:

- [ ] **Código revisado**: el cambio pasó por Pull Request, no se integró directamente a la rama principal del repositorio correspondiente.
- [ ] **Contrato respetado**: si la tarea toca un contrato compartido (esquema Zod, modelo de estados, catálogo de errores), el cambio no contradice [contratos-api.md](contratos-api.md), [base-datos.md](base-datos.md) ni [catalogo-errores.md](catalogo-errores.md). Si el contrato necesita cambiar, el cambio se documenta primero en `flowguard-docs`.
- [ ] **Pruebas ejecutadas**: existe al menos la prueba mínima indicada en la fila correspondiente de [checklist-implementacion.md](checklist-implementacion.md) (unitaria, de integración o E2E, según [pruebas.md](pruebas.md)), y se ejecutó realmente — no se asume que "debería pasar".
- [ ] **Comportamiento verificable**: el resultado puede reproducirse por otro desarrollador (por ejemplo, un endpoint real responde lo esperado, o la UI muestra el estado documentado), no solo describirse.
- [ ] **Errores tratados**: los casos de error relevantes (ver [catalogo-errores.md](catalogo-errores.md)) no rompen el flujo ni cambian la `decision` ya persistida.
- [ ] **Sin secretos en el repositorio**: no se incluyen tokens, contraseñas ni credenciales de Nebius o Supabase en ningún archivo versionado, incluidas notas (`notas.md`) y bitácoras (`bitacora.md`).
- [ ] **Documentación actualizada**: si la tarea afecta a la arquitectura, a un contrato o a más de un módulo, se refleja en la documentación general (`plan-desarrollo.md`, `integracion-y-dependencias.md`, o el documento técnico correspondiente en `flowguard-docs`), no solo en la nota personal del desarrollador.
- [ ] **Integración compatible**: el cambio no rompe el trabajo de otro desarrollador que depende del mismo contrato (ver [integracion-y-dependencias.md](integracion-y-dependencias.md)).

## Criterios por hito de bloque

Cada uno de los cinco hitos obligatorios del plan de 15 días (ver [plan-desarrollo.md](plan-desarrollo.md)) tiene, además, su propio criterio de aceptación específico, ya definido en [checklist-implementacion.md](checklist-implementacion.md):

| Bloque | Hito obligatorio |
|---|---|
| 1 (días 1–3) | Contratos congelados y confirmados por escrito por los cinco puestos |
| 2 (días 4–6) | Crear una ejecución, evaluar las cinco reglas, persistir la decisión y mostrarla en el panel, con la IA en modo mock |
| 3 (días 7–9) | Una ejecución bloqueada produce un informe real o un `estado_investigacion = FALLIDA` explícito y correctamente comunicado |
| 4 (días 10–12) | El MVP funciona de extremo a extremo y conserva la decisión cuando falla Nebius |
| 5 (días 13–15) | Versión estable, pruebas críticas aprobadas, README reproducible y demo ensayada |

Un hito de bloque no se considera cumplido solo porque cada tarea individual esté marcada como completada: se verifica mediante un ensayo interno con evidencia (ver la columna "Prueba que lo demuestra" de cada bloque en [checklist-implementacion.md](checklist-implementacion.md)). Con la distribución de cinco puestos, ese ensayo se verifica de forma **cruzada** entre puestos (antes lo verificaba un solo puesto): **ninguna persona aprueba en solitario los componentes que ella misma desarrolló**. El requisito de ensayo con evidencia no se elimina ni se sustituye por el simple éxito de un pipeline.

### Verificación cruzada de los hitos de los días 6, 9, 12 y 15

| Hito | Qué se verifica | Quién aporta y verifica |
|---|---|---|
| Día 6 | Revisión cruzada de los módulos involucrados en el recorrido end-to-end con IA en modo mock | Los responsables aportan sus pruebas y evidencia; otro puesto revisa cada módulo |
| Día 9 | Verificación de la integración y de los escenarios de fallo del proveedor | Dev 3 aporta las pruebas del agente y del proveedor; Dev 1 y Dev 2 verifican persistencia y presentación; Dev 5 coordina lo transversal |
| Día 12 | Regresión conjunta, registro de defectos y evaluación de los pendientes | Propietarios de cada módulo; Dev 5 registra los defectos transversales |
| Día 15 | Ensayo final de la demo, verificación de datos reiniciables y revisión del checklist de aceptación | Verificación cruzada entre puestos; Dev 2 (experiencia de demo), Dev 1 (datos reiniciables), Dev 5 (consistencia documental) |

En todos los hitos: **Dev 4** aporta resultados reproducibles de CI/CD como evidencia complementaria; **Dev 5** coordina los aspectos transversales, la compatibilidad y el registro de defectos; y quien coordina el proyecto organiza las revisiones y registra las decisiones. Los criterios de aceptación específicos de cada hito se conservan tal como están en [checklist-implementacion.md](checklist-implementacion.md).

## Requisitos del hackathon y evidencias de cumplimiento

Esta sección lista los requisitos de participación en el Nebius x NVIDIA Global AI Hackathon y quién aporta la evidencia de cada uno. **Que un requisito figure aquí no significa que esté cumplido**: ninguno se marca como `Verificado` sin evidencia.

- **Fuente:** la lista de requisitos proporcionada por el equipo al reorganizar la documentación. No se pudo contrastar contra el texto oficial de las reglas durante esa tarea; cualquier detalle que no pueda verificarse queda como `Pendiente de verificar`. Antes de la entrega hay que cotejar cada fila contra las reglas oficiales vigentes.
- **Estados posibles:** `Pendiente de verificar`, `Verificado` o `No aplicable con justificación`.
- La asignación de responsables es una propuesta operativa.

| Requisito | Evidencia necesaria | Responsable propuesto | Estado | Observaciones o enlace a la evidencia |
|---|---|---|---|---|
| Aplicación funcional que se ejecute en Nebius Token Factory o Nebius AI Cloud | Aplicación en ejecución y prueba del servicio de Nebius utilizado | Dev 3 (Nebius), Dev 1 (backend y persistencia), Dev 2 (flujo visible para el evaluador) | Pendiente de verificar | La plataforma de hosting sigue sin decidirse ([arquitectura.md § 7.3](arquitectura.md#73-supuesto-de-despliegue-y-alternativa-para-hosting-serverless)) y no existe aún el backend completo |
| Uso de al menos un modelo open source de NVIDIA | Nombre del modelo, configuración y registro de una llamada real | Dev 3 | Pendiente de verificar | `proveedorNebius.ts` está pendiente de implementarse; URL y modelo son marcadores de configuración ([seguridad.md § 1.8](seguridad.md#18-lista-de-verificación-de-compatibilidad-con-nebius)) |
| Repositorio público | URL pública del repositorio de la entrega | Dev 5 (consistencia), decisión final de quien coordina el proyecto | Pendiente de verificar | Los siete repositorios figuran como públicos en GitHub; el repositorio final `flowguard` aún no tiene carpetas integradas |
| Licencia permitida por las reglas (por ejemplo MIT, Apache 2.0 o MPL 2.0) | Archivo de licencia en el repositorio de la entrega | Decisión de quien coordina el proyecto; Dev 5 verifica la consistencia | Pendiente de verificar | Ningún repositorio contiene hoy un archivo de licencia |
| README con instrucciones de configuración y ejecución | README reproducible, probado por alguien que no lo escribió | Cada propietario aporta las instrucciones de su módulo; Dev 5 revisa la consistencia documental; Dev 4 aporta la reproducibilidad del pipeline | Pendiente de verificar | |
| Demo funcional o build de prueba | Demo ensayada (hito del día 15) | Dev 2 (experiencia de demo), Dev 1 (backend) | Pendiente de verificar | Guion en [guion-demo.md](guion-demo.md) |
| Video público de YouTube de menos de tres minutos | Enlace público al video | Dev 2 (experiencia y guion), Dev 3 (explicación de Nebius y NVIDIA) | Pendiente de verificar | El guion actual se titula "3 minutos": confirmar que dura **menos** de tres minutos |
| Explicación del modelo NVIDIA utilizado y de cómo Nebius aceleró el flujo de trabajo | Texto en la entrega y mención en el video | Dev 3 | Pendiente de verificar | Depende de la integración real con Nebius |
| Selección de la categoría o track correspondiente | Track elegido en el envío | Quien coordina el proyecto | Pendiente de verificar | El track no está documentado en los repositorios |
| Feedback solicitado por el hackathon | Feedback enviado | Quien coordina el proyecto | Pendiente de verificar | El contenido exacto solicitado no está documentado en los repositorios |
| Acceso gratuito y sin restricciones durante la evaluación | Aplicación accesible a los evaluadores sin pago | Dev 1 y Dev 2; decisión de quien coordina el proyecto | Pendiente de verificar | **Conflicto documentado, sin resolver:** [seguridad.md § 2.0](seguridad.md#20-estrategia-de-seguridad-por-entorno) protege la demo pública con Basic Auth. No se cambia el modelo de seguridad; requiere una decisión del equipo (por ejemplo, entregar las credenciales a los evaluadores) |
| Materiales de entrega en inglés o con las traducciones requeridas | Materiales de entrega revisados | Dev 5 (consistencia), cada puesto aporta sus materiales | Pendiente de verificar | Aparecía en solicitudes anteriores pero no en la lista actual; confirmar si aplica |

### Responsabilidades de entrega por puesto

| Puesto | Aporta |
|---|---|
| Dev 1 | Evidencia del backend y la persistencia; validación de la ejecución del backend |
| Dev 2 | Experiencia de demo y flujo visible para el evaluador |
| Dev 3 | Evidencia del uso real de Nebius Token Factory y del modelo NVIDIA |
| Dev 4 | Reproducibilidad del pipeline y comprobaciones automatizadas |
| Dev 5 | Consistencia documental, contratos, fixtures y coordinación de evidencias transversales |
| Quien coordina el proyecto | Supervisión, seguimiento de pendientes y decisión final sobre la entrega |

La preparación de la demo, el README final, el video y las evidencias se reparte entre estos puestos; no se asigna automáticamente a quien coordina el proyecto. El puesto 5 puede ayudar con tareas concretas cuando exista un bloqueo, sin convertirse en responsable permanente de la entrega.

## Lo que NO cuenta como "terminado"

- Una tarea marcada `Completado` sin una prueba o revisión que lo respalde.
- Un endpoint que "debería funcionar" pero no se probó contra una base de datos real.
- Una nota en `notas.md` describiendo una decisión técnica que no se reflejó en la documentación general cuando afecta a otros desarrolladores.
- Un bloqueo resuelto sin registrar su resolución en `bloqueos.md`.

## Referencias

- Criterios de salida técnicos y Definition of Done original: [pruebas.md § 4](pruebas.md#4-criterios-de-salida)
- Plan de pruebas completo: [pruebas.md](pruebas.md)
- Catálogo de errores: [catalogo-errores.md](catalogo-errores.md)
- Cómo se integran los cambios entre desarrolladores: [integracion-y-dependencias.md](integracion-y-dependencias.md)
