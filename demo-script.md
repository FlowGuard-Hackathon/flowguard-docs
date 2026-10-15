# Guion de demostración del hackathon — 3 minutos

El objetivo es demostrar que FlowGuard no es simplemente un chatbot que explica reglas: comprueba una operación, bloquea una infracción y genera evidencia verificable.

## 1. Preparación

Antes de comenzar:

- Tener la aplicación iniciada.
- Verificar la conexión con Supabase y Nebius.
- Preparar el presupuesto de $25,000 MXN.
- Configurar el umbral de autorización en $10,000 MXN.
- Tener disponible una factura de prueba verificada.
- Cargar los escenarios de $8,500 y $15,000.
- Comprobar que el historial y el centro de evidencias funcionan.

## 2. Guion cronometrado

| Tiempo | Duración | Paso | Qué mostrar |
|---|---|---|---|
| 0:00 | 30 s | Presentación del problema | Mostrar el panel y explicar que los errores de un proceso automatizado pueden avanzar si nadie verifica las condiciones de negocio. |
| 0:30 | 35 s | Escenario válido | Ejecutar el pago de $8,500 MXN. Mostrar las cinco reglas cumplidas y el resultado `ALLOW`. |
| 1:05 | 50 s | Provocar una infracción | Cambiar el monto a $15,000 MXN y dejar la autorización como pendiente. Ejecutar el escenario y mostrar `PAY-001: FAIL`, `decision = BLOCK`, `execution_status = PERSISTED` y `may_advance = false`. |
| 1:55 | 40 s | Investigación de la IA | Abrir el centro de evidencias. Mostrar las referencias de la solicitud y la autorización, la explicación de Nebius y la recomendación de solicitar aprobación. |
| 2:35 | 25 s | Cierre | Mostrar las dos ejecuciones en el historial y destacar la separación entre decisión, investigación y evidencia. |

### Frases clave por paso

- **Presentación:** "FlowGuard comprueba las reglas mediante código; la IA investiga y explica, pero no tiene autoridad para aprobar".
- **Escenario válido:** "El sistema verifica el monto, los permisos, la categoría, la factura y el presupuesto antes de permitir continuar".
- **Infracción:** "El motor bloquea la operación antes de llamar a la IA. No depende de que el modelo detecte el problema".
- **Investigación:** "El agente consulta los registros del backend, relaciona los datos y explica la causa. Cada hallazgo debe respaldarse con evidencia existente".
- **Cierre:** "FlowGuard no solo detecta una anomalía: evita que avance y deja trazabilidad de lo ocurrido".

## 3. Plan de contingencia de la demo

Si Nebius falla durante la presentación:

1. Mostrar la decisión `BLOCK` ya persistida (`execution_status = PERSISTED`).
2. Mostrar las evidencias generadas por el motor.
3. Mostrar `investigation_status = FAILED` y el error de proveedor (ver [error-catalog.md](error-catalog.md)).
4. Explicar que el sistema mantiene el bloqueo aunque no pueda completar la investigación, y que se puede reintentar la investigación con `POST /investigate` sin afectar la decisión.

Se puede disponer de un informe previamente generado para ensayos y validaciones, pero debe identificarse como fixture y nunca presentarse como una respuesta generada en vivo.

## Referencias

- Contratos de la decisión y del informe usados en la demo: [api-contracts.md](api-contracts.md)
- Comportamiento ante fallo de Nebius: [architecture.md](architecture.md#51-comportamiento-ante-fallos)
- Códigos de error mostrados en el plan de contingencia: [error-catalog.md](error-catalog.md)
