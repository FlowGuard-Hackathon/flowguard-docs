# Plan de trabajo — Desarrollador 5 (Contratos, mocks y apoyo general)

Responsabilidades: contratos compartidos, mocks y datos simulados, coordinación de la suite compartida `flowguard-tests` y apoyo temporal al resto del equipo.

Este plan es **flexible y basado en necesidades reales**, no un calendario de quince días. Las fechas y los hitos siguen siendo los de [plan-desarrollo.md](../../plan-desarrollo.md) e [checklist-implementacion.md](../../checklist-implementacion.md), que son la fuente de verdad. Nada se marca como completado hasta que exista evidencia verificable, según [definicion-de-terminado.md](../../definicion-de-terminado.md).

## 1. Trabajo base: contratos y mocks

Estas dos responsabilidades **no se desplazan sistemáticamente** por el trabajo de apoyo. Si el apoyo amenaza con retrasar contratos o fixtures de los que otros dependen, se prioriza según la sección 3 y se avisa al equipo.

### 1.1 Contratos compartidos

| Trabajo | Detalle | Con quién | Estado |
|---|---|---|---|
| Proponer `pago.schema.ts`, `ejecucion.schema.ts` y `motor.schema.ts` | Se alinean con [contratos-api.md](../../contratos-api.md); no se cambian campos, estados ni semántica. Dev 1 los valida | Dev 1 | Pendiente |
| Coordinar la compatibilidad con `informe-evidencias.schema.ts` | Su propiedad es de Dev 3; Dev 5 no lo modifica sin coordinación y aprobación de Dev 3 y de los demás responsables afectados | Dev 3 | Pendiente |
| Mantener ejemplos de solicitudes y respuestas actualizados | Solo tras la aprobación de cada cambio de contrato | Responsables afectados | Pendiente |
| Registrar propuestas de modificación de contratos | Cada propuesta queda anotada y se coordina su aprobación con los responsables afectados | Responsables afectados | Pendiente |
| Congelación de contratos (día 1 del calendario) | Los cinco puestos confirman por escrito que no necesitan campos adicionales | Todos | Pendiente |

### 1.2 Mocks y datos simulados (`flowguard-mocks`)

| Trabajo | Detalle | Con quién | Estado |
|---|---|---|---|
| `mocks/pago-aprobado.json` y `mocks/pago-bloqueado.json` | Los necesita Dev 2 desde el Bloque 1 | Dev 2 | Pendiente |
| `mocks/investigacion-fallida.json` | Lo espera Dev 3 para cerrar el día 3 de su plan (ver su [plan-trabajo.md](../desarrollador-3/plan-trabajo.md)) | Dev 3 | Pendiente |
| Prueba de contrato sobre los tres fixtures | Cada fixture valida contra su esquema Zod; Dev 4 la ejecuta en CI cuando el script exista | Dev 4 | Pendiente |
| Escenarios válidos, inválidos e incompletos | Solicitudes, facturas, presupuestos y autorizaciones, según los casos de [reglas.md](../../reglas.md) y [pruebas.md](../../pruebas.md) | Dev 1, Dev 2, Dev 3 | Pendiente |

Los mocks son datos simulados, **no evidencias reales**: no se usan para afirmar que una integración real con Nebius o Supabase fue validada.

### 1.3 Suite compartida `flowguard-tests`

- Coordina la suite compartida y mantiene las pruebas de contrato y de compatibilidad transversal.
- **No** implementa todas las pruebas de todos los módulos: los propietarios mantienen las pruebas funcionales de sus módulos (ver [pruebas.md § Responsables de las pruebas](../../pruebas.md#responsables-de-las-pruebas)).
- Registra los defectos transversales con pasos de reproducción; el propietario del módulo afectado los corrige.

## 2. Apoyo general: proceso por solicitud

Proceso diario sugerido:

1. Revisar las solicitudes y los bloqueos abiertos (los propios de los demás en sus `bloqueos.md` y lo que lleguen por el equipo).
2. Ordenarlos con el método de la sección 3.
3. Acordar con el responsable del módulo el alcance de la intervención antes de empezar.
4. Reproducir el problema o pedir la información que falte.
5. Investigar; proponer o implementar la solución junto con el responsable.
6. Ejecutar las verificaciones pertinentes.
7. Registrar el resultado en [bitacora.md](bitacora.md) y las notas reutilizables en [notas.md](notas.md).
8. Quedar disponible para la siguiente necesidad.

**Criterios para empezar una intervención:** existe una solicitud con módulo y archivo afectados, comportamiento esperado y observado, error, pasos para reproducirlo, acciones ya intentadas, ayuda concreta, urgencia y tareas bloqueadas (si no hay información suficiente, se pide antes de empezar), y el responsable acordó el alcance.

**Criterios para terminar una intervención:** el responsable del módulo validó la solución; se ejecutaron las pruebas pertinentes; el resultado, las limitaciones y los pendientes quedaron registrados; y, si hay riesgos abiertos, se anotaron en [bloqueos.md](bloqueos.md).

### Reglas para trabajar sobre código ajeno

- No se modifica código de otro módulo sin coordinar el alcance con su propietario.
- No se cambian contratos compartidos por iniciativa propia.
- No se toman decisiones de arquitectura.
- No se asume la propiedad permanente de ningún módulo ni se aceptan tantas tareas de apoyo que se deje de estar disponible para los bloqueos más importantes.
- No se convierte este puesto en el único capaz de resolver problemas de integración.

## 3. Cómo priorizar

1. Bloqueos que impiden ejecutar o probar el producto.
2. Bloqueos que detienen a varios integrantes.
3. Defectos que impiden demostrar el flujo principal.
4. Tareas que amenazan una dependencia crítica o una fecha límite.
5. Mejoras menores o limpieza.

Si dos bloqueos tienen la misma importancia, se consulta a los responsables afectados y al equipo.

## 4. Verificar y registrar una solución

- El responsable original valida la solución y mantiene la propiedad de su módulo.
- Si la solución afecta a varios módulos o contratos, intervienen los responsables afectados antes de integrar el cambio.
- Se anotan las pruebas ejecutadas y sus resultados reales; no se declara aprobada una prueba que no se ejecutó.

## 5. Lo que este puesto no hace

No asume CI/CD, despliegues, DevOps, backend, frontend ni agente de IA, ni la propiedad exclusiva de todo el QA funcional. Los hitos de los días 6, 9, 12 y 15 se verifican de forma cruzada entre puestos: Dev 5 coordina los aspectos transversales y la compatibilidad, pero no aprueba en solitario.

## Referencias

- [checklist-implementacion.md](../../checklist-implementacion.md) — detalle técnico completo por fila
- [integracion-y-dependencias.md](../../integracion-y-dependencias.md) — contratos, dependencias y procedimiento de apoyo
- [pruebas.md](../../pruebas.md) — responsables por tipo de prueba
