# Requisitos de las vistas del frontend

Lista de requisitos de UI para las vistas de `flowguard-app`, aportada por el equipo el 2026-10-17. Propietario: Dev 2. Complementa — no sustituye — [contratos-api.md](contratos-api.md) y [arquitectura.md § 6](arquitectura.md#6-frontend-y-experiencia-de-usuario): este documento dice **qué debe mostrar cada vista**, no define campos ni estados nuevos. Todo nombre de campo o valor usado aquí sigue el [glosario](glosario.md).

## 0. Cómo usar este documento

- Es un documento de requisitos de interfaz, no un contrato técnico. Una vista solo puede mostrar los campos que `contratos-api.md` ya expone; no se inventan campos nuevos sin pasar primero por ese documento y por Dev 5 (contratos compartidos).
- **Las vistas 1 (inicio de sesión) y 8 (gestión de usuarios) quedan fuera del alcance confirmado del MVP de 15 días.** [arquitectura.md § 2.2](arquitectura.md#22-funcionalidades-excluidas) excluye explícitamente "un sistema completo de roles empresariales"; [seguridad.md § 2.0](seguridad.md#20-estrategia-de-seguridad-por-entorno) solo define Basic Auth para proteger la demo pública, no sesiones de usuario con roles. La arquitectura debe soportar esto sin rediseño, pero no se implementa en el MVP salvo que el equipo decida explícitamente ampliar el alcance y lo registre en [arquitectura.md](arquitectura.md). Mientras tanto quedan documentadas como **propuesta para una fase posterior**.
- Las demás vistas (2 a 7, y los elementos compartidos) sí están dentro del MVP y se implementan contra los endpoints de `contratos-api.md` a medida que existan.
- Nada de esto cambia las cinco reglas de negocio, el modelo de datos ni los contratos existentes.

## 1. Inicio de sesión *(fuera del alcance confirmado del MVP — propuesta)*

- Logotipo y nombre FlowGuard.
- Campo de correo electrónico.
- Campo de contraseña con opción para mostrarla.
- Botón «Iniciar sesión».
- Recuperación de contraseña, si está implementada.
- Mensajes de error y estado de carga.
- Validación de credenciales y creación de sesión segura.

Requiere adoptar autenticación de usuario real (por ejemplo Supabase Auth, ya prevista como opción en [seguridad.md § 2.0](seguridad.md#20-estrategia-de-seguridad-por-entorno)), que hoy no está implementada. No confundir con la Basic Auth de la demo pública, que protege toda la app con una sola credencial compartida y no identifica usuarios.

## 2. Panel de supervisión

- Resumen de solicitudes evaluadas.
- Cantidad de decisiones `PERMITIR`.
- Cantidad de decisiones `BLOQUEAR`.
- Reglas en estado `INDETERMINADO` o ejecuciones con investigación pendiente de revisión.
- Actividad reciente.
- Alertas relevantes.
- Reglas con incumplimientos frecuentes.
- Botones «Simular aprobación» y «Ver historial».

**Aclarar antes de implementar:** `decision` (la decisión global de una ejecución) solo tiene dos valores, `PERMITIR` o `BLOQUEAR` — no existe un tercer valor `INDETERMINADO` a ese nivel ([contratos-api.md § 0](contratos-api.md#0-modelo-de-estados-resumen)). `INDETERMINADO` sí existe como resultado de una regla individual (`CUMPLE`/`NO_CUMPLE`/`INDETERMINADO`). "Pendiente de revisión" probablemente se refiere a `estado_investigacion` en `PENDIENTE` o `EN_CURSO`. Confirmar con Dev 1 y Dev 5 a qué corresponde exactamente antes de construir este contador.

## 3. Solicitudes de pago

- Tabla de solicitudes.
- `ejecucion_id`.
- Solicitante y proveedor.
- Concepto e importe.
- Fecha de creación.
- Estado de evaluación (`estado_ejecucion`, `decision`).
- Regla determinante (`regla_id`), cuando exista.
- Buscador y filtros por estado, categoría y fecha.
- Acción «Ver detalle».
- Botón «Nueva solicitud», según los permisos del usuario.

Corresponde a `GET /api/v1/ejecuciones` ([contratos-api.md § 4](contratos-api.md#4-get-apiv1ejecuciones)), que ya admite filtros por `decision`, `estado_investigacion` y rango de fechas (`desde`/`hasta`), y paginación (`pagina`, `tamano_pagina`). El campo "proveedor" de este requisito no es `agente.proveedor` (Nebius o simulado): hay que confirmar si se refiere al proveedor/beneficiario del pago, que no está en el contrato actual, y registrarlo como pregunta para Dev 1 y Dev 5 si falta.

## 4. Detalle de solicitud

- `ejecucion_id`, solicitante, proveedor, concepto e importe.
- Decisión final `PERMITIR` o `BLOQUEAR`.
- Estados `CUMPLE`, `NO_CUMPLE` o `INDETERMINADO` por regla, cuando corresponda.
- Evaluación de las reglas `PAG-001` a `PAG-005`.
- Resultado de cada regla: `CUMPLE`, `NO_CUMPLE` o `INDETERMINADO`.
- Motivo de cada resultado (`mensaje`, `observado`, `esperado`).
- Evidencias asociadas (`refs_evidencia`).
- Fecha de evaluación e `ejecucion_id`.
- Botón «Ver evidencias».
- Botón para regresar al listado.
- Explicación de la regla que provocó el bloqueo.

Corresponde a `GET /api/v1/ejecuciones/:id` ([contratos-api.md § 5](contratos-api.md#5-get-apiv1ejecucionesid)). El arreglo `reglas` siempre trae las cinco reglas en orden `PAG-001`…`PAG-005`, no solo la que falló.

## 5. Simulador de aprobación

- Solicitante y aprobador.
- Proveedor y concepto.
- Importe en MXN.
- Categoría: `OPERACIONES`, `SOFTWARE` o `VIAJES`.
- Identificador de factura.
- Presupuesto disponible.
- Estado de verificación de factura.
- Existencia de autorización de supervisor.
- Umbral configurado de autorización.
- Botones «Evaluar solicitud» y «Limpiar formulario».
- Resultado final y evaluación de las cinco reglas.
- Evidencias utilizadas, motivos e `ejecucion_id`.
- Aviso de que no se ejecutan pagos reales.

Corresponde a `POST /api/v1/ejecuciones` ([contratos-api.md § 3](contratos-api.md#3-post-apiv1ejecuciones)). **El simulador no puede enviar** `estado_aprobacion`, roles, permisos de aprobador ni el resultado de verificación de factura: esos datos los resuelve el backend contra el contexto confiable, nunca el payload del cliente (misma sección). Los campos "autorización de supervisor" y "estado de verificación de factura" del formulario son para simular el escenario de prueba (vía fixtures/contexto confiable), no para que el usuario los declare como verdaderos directamente.

## 6. Centro de evidencias

- `ref_evidencia`.
- Solicitud (`ejecucion_id`) e `intento_id` relacionados.
- Regla asociada (`regla_id`).
- `tipo_evidencia`.
- Fuente de origen (`origen`).
- Estado de verificación.
- Fecha de obtención.
- Descripción o fragmento relevante (`contenido`).
- Acceso al documento, si está disponible.
- Filtros por solicitud, regla y estado.
- Diferenciación entre evidencia verificada, insuficiente y no disponible.
- Identificación de hallazgos generados por la IA (`investigacion.hallazgos`).

Corresponde a `GET /api/v1/ejecuciones/:id/evidencias` ([contratos-api.md § 6](contratos-api.md#6-get-apiv1ejecucionesidevidencias)). Si no existe ningún intento de investigación todavía, `informe_agente` es `null` y esta vista debe mostrar ese estado explícitamente, no simular un informe.

## 7. Historial de ejecuciones

- `ejecucion_id`.
- `ejecucion_id` relacionado (si esta vista es distinta del listado de la vista 3, confirmar la diferencia con Dev 2; de lo contrario, ambas podrían ser la misma vista con distintos filtros).
- Fecha y hora.
- Decisión final.
- Regla determinante.
- Estado de investigación del agente (`estado_investigacion`).
- Estado de ejecución (`estado_ejecucion`).
- Usuario que inició el proceso, si se registra.
- Búsqueda y filtros.
- Acción para consultar el detalle.
- Acceso a los resultados y evidencias relacionados.
- Consulta de registros sin permitir modificar decisiones pasadas: ninguna vista del frontend puede escribir sobre `decision` una vez persistida ([integracion-y-dependencias.md § Separación de responsabilidades](integracion-y-dependencias.md#separación-de-responsabilidades-no-negociable)).

## 8. Gestión de usuarios *(fuera del alcance confirmado del MVP — propuesta)*

- Nombre y correo electrónico.
- Rol asignado.
- Estado de la cuenta.
- Último acceso, si se registra.
- Botón «Invitar usuario» o «Crear usuario».
- Acciones para cambiar roles o desactivar cuentas.
- Confirmación de cambios sensibles.
- Acceso restringido a administradores autorizados.

### Roles propuestos (propuesta, no implementados)

- **Administrador:** gestiona usuarios y configuración.
- **Solicitante:** registra solicitudes.
- **Supervisor:** registra autorizaciones válidas conforme a los permisos.
- **Auditor:** consulta evaluaciones y evidencias.

Esta vista depende de la vista 1 (inicio de sesión) y de un modelo de roles que hoy no existe en [base-datos.md](base-datos.md) ni en [seguridad.md](seguridad.md). No se implementa en el MVP; queda documentada para cuando el equipo decida ampliar el alcance.

## 9. Configuración de reglas y parámetros

- `regla_id` y nombre de cada regla.
- Descripción.
- Parámetros relevantes (`configuracion`).
- Estado de configuración (`habilitada`).
- Umbral de autorización de supervisor.
- Categorías permitidas.
- `version_politica`.
- Acceso a la documentación ([reglas.md](reglas.md)).

Reglas del MVP ([reglas.md § 2](reglas.md#2-las-cinco-reglas)):

- `PAG-001`: autorización de supervisor.
- `PAG-002`: categorías permitidas.
- `PAG-003`: separación de funciones.
- `PAG-004`: factura verificada mediante evidencia confiable.
- `PAG-005`: presupuesto suficiente.

**Inicialmente esta vista es de solo consulta.** Cualquier modificación futura de las reglas desde la interfaz debe tener permisos, validación del backend y trazabilidad de cambios — hoy el frontend no puede escribir reglas, solo mostrarlas ([rules.md § 1, principios de implementación](reglas.md#1-principios-de-implementación)).

## Elementos compartidos entre las vistas

- Navegación: **pendiente de confirmar.** Esta lista pide navegación lateral, pero la primera implementación de navegación en `flowguard-app` (rama `feat/dev2-layout-navegacion`) ya usa un encabezado horizontal con los enlaces `/panel`, `/simulador` y `/evidencias`. Decidir cuál es la navegación definitiva antes de construir más vistas sobre una u otra, para no rehacer trabajo.
- Logotipo y nombre FlowGuard.
- Perfil del usuario y cierre de sesión *(depende de la vista 1, fuera del alcance confirmado del MVP)*.
- Diseño adaptable a distintos tamaños de pantalla.
- Estados y mensajes consistentes, según los definidos en [arquitectura.md § 6.4](arquitectura.md#64-estados-de-interfaz) (dentro de § 6, Frontend y experiencia de usuario): cargando, solicitud recibida, evaluando reglas, bloqueado o permitido, investigación en curso, informe completo, informe parcial, investigación fallida, error de API.
- Indicadores de carga, error y ausencia de datos.
- Permisos según el rol *(depende de la vista 8, fuera del alcance confirmado del MVP)*.
- Identificadores para mantener la trazabilidad (`ejecucion_id`, `intento_id`, `ref_evidencia`).

La interfaz debe mostrar datos reales del backend: no debe simular que una regla fue verificada o que el agente terminó una investigación cuando eso no ocurrió ([arquitectura.md § 6.4](arquitectura.md#64-estados-de-interfaz)).

## Conflictos y preguntas abiertas

Sin resolver todavía; no se decide nada aquí por iniciativa propia:

1. **Alcance de inicio de sesión y gestión de usuarios.** Están fuera de lo documentado como MVP. Necesitan una decisión explícita del equipo para entrar al alcance, con su propio modelo de datos y de permisos.
2. **"Operaciones indeterminadas o pendientes" a nivel de panel (vista 2).** Aclarar si se refiere a reglas individuales en `INDETERMINADO` o a `estado_investigacion` sin resolver.
3. **Campo "proveedor" (vistas 3 y 5).** No corresponde a `agente.proveedor` del informe del agente. Confirmar si se refiere al beneficiario del pago y si hace falta agregarlo al contrato.
4. **Navegación lateral vs. superior.** Ver "Elementos compartidos" arriba.
5. **Vistas 3 y 7.** Podrían superponerse (listado de solicitudes vs. historial de ejecuciones). Confirmar si son la misma vista con distintos filtros o dos vistas distintas antes de construir ambas.

## Referencias

- Contratos de API que cada vista consume: [contratos-api.md](contratos-api.md)
- Estados de interfaz y principios de UX: [arquitectura.md § 6](arquitectura.md#6-frontend-y-experiencia-de-usuario)
- Las cinco reglas de negocio: [reglas.md](reglas.md)
- Catálogo de errores: [catalogo-errores.md](catalogo-errores.md)
- Nombres oficiales de campos y valores: [glosario.md](glosario.md)
- Plan de trabajo del puesto 2: [desarrolladores/desarrollador-2/plan-trabajo.md](desarrolladores/desarrollador-2/plan-trabajo.md)
