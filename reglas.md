# Motor de reglas deterministas

## 1. Principios de implementación

El motor debe ser una biblioteca independiente que reciba datos validados y un contexto confiable.

```ts
evaluarPago(
  solicitud,
  contextoConfiable,
  instantaneaPolitica
): DecisionMotor
```

No debe importar React, el cliente de Nebius ni componentes del frontend.

Las reglas se ejecutan en un orden estable, y cada resultado contiene:

- Identificador y versión de regla.
- Estado.
- Datos observados.
- Datos esperados.
- Mensaje explicativo.
- Referencias a evidencia.
- Severidad.

## 2. Las cinco reglas

| ID | Regla | Condición para pasar |
|---|---|---|
| PAG-001 | Límite de autorización | Si el importe supera $10,000 MXN, existe una aprobación válida |
| PAG-002 | Categoría permitida | Categoría en `OPERACIONES`, `SOFTWARE` o `VIAJES` |
| PAG-003 | Separación de funciones | El aprobador es distinto del solicitante y tiene permiso |
| PAG-004 | Factura verificada | Existe factura confirmada por el backend |
| PAG-005 | Presupuesto disponible | El importe no supera el saldo disponible |

Las reglas de ejemplo son políticas configurables para el MVP, no políticas universales.

### PAG-001: límite de autorización

```ts
if (montoCentavos > umbralAprobacionCentavos) {
  return aprobacionSupervisorValida
    ? CUMPLE
    : NO_CUMPLE;
}
return CUMPLE;
```

`aprobacionSupervisorValida` debe derivarse de un registro confiable, verificando la identidad, el permiso y la vigencia de la aprobación. No basta con que el cliente envíe `APROBADO`.

### PAG-002: categoría

```ts
const categoriasPermitidas = [
  "OPERACIONES",
  "SOFTWARE",
  "VIAJES"
];

return categoriasPermitidas.includes(categoria)
  ? CUMPLE
  : NO_CUMPLE;
```

La lista definitiva debe estar en la política del backend, no duplicada como autoridad en el frontend.

### PAG-003: separación de funciones

```ts
const aprobadorValido =
  solicitanteId !== aprobadorId &&
  aprobadorActivo &&
  aprobadorTienePermiso;

return aprobadorValido ? CUMPLE : NO_CUMPLE;
```

Cuando una aprobación especial no sea necesaria, la regla debe evaluar los permisos que correspondan al proceso. No debe exigir que todos los pagos tengan un supervisor solo porque el ejemplo contiene ese campo.

### PAG-004: factura verificada

El backend consulta el registro del adjunto **en el contexto confiable**, nunca en el payload del cliente. Una factura no se considera verificada solo porque el request incluya `"verificada": true` o un campo equivalente: ese valor, si el cliente llega a enviarlo, se ignora. La única fuente válida es la resolución que el backend hace de `adjunto_id`/`referencia` contra sus propios registros (ver [contratos-api.md § 3](contratos-api.md#3-post-apiv1executions)). Si la referencia no existe o la factura no está verificada ahí, la regla falla. Si la fuente de verificación no responde, el resultado será `INDETERMINADO`.

### PAG-005: presupuesto

```ts
return montoCentavos <= presupuestoDisponibleCentavos
  ? CUMPLE
  : NO_CUMPLE;
```

La comprobación debe utilizar un saldo confiable. Si dos operaciones concurrentes consumen presupuesto, la reserva o actualización del saldo debe implementarse con transacciones y control de concurrencia. Para el MVP basta con simular el presupuesto y comprobarlo sin mover dinero real.

## 3. Resultado global

```ts
const debeBloquear = resultados.some(
  resultado =>
    resultado.estado === "NO_CUMPLE" ||
    resultado.estado === "INDETERMINADO"
);

const decision = debeBloquear ? "BLOQUEAR" : "PERMITIR";
```

La evaluación debe producir los resultados de todas las reglas, incluso si la primera falla, para permitir una explicación completa en el panel. Nótese que `INDETERMINADO` tiene el mismo efecto que `NO_CUMPLE` sobre `debeBloquear`: una condición obligatoria que no se puede verificar bloquea la operación igual que una infracción confirmada (política *fail-closed*, ver [contratos-api.md § 0](contratos-api.md#0-modelo-de-estados-resumen)), aunque el informe de evidencias debe seguir distinguiendo ambos casos en su texto, sin presentarlos como el mismo hecho.

Cada ejecución de este resultado global queda asociada a `version_politica` (la versión de `reglas_politica` vigente al momento de evaluar). Dos ejecuciones con el mismo `solicitud_pago` pero evaluadas bajo versiones de política distintas pueden producir decisiones distintas de forma legítima; esto se audita mediante `ejecuciones.version_politica` (ver [base-datos.md § 1.6](base-datos.md#16-executions)).

## 4. Casos de prueba obligatorios

1. $8,500 MXN sin aprobación especial: pasa PAG-001.
2. $10,000 MXN: no activa la condición `monto > 10000`.
3. $10,000.01 MXN: requiere aprobación.
4. $15,000 MXN sin aprobación válida: falla PAG-001.
5. Categoría desconocida: falla PAG-002.
6. Solicitante intentando autorizar su propio pago: falla PAG-003.
7. Factura ausente: falla PAG-004.
8. Fuente de verificación de factura no disponible: PAG-004 indeterminada.
9. Presupuesto insuficiente: falla PAG-005.
10. Presupuesto no verificable: PAG-005 indeterminada.

Ver también el detalle de ejecución de estos casos como pruebas unitarias en [pruebas.md](pruebas.md#1-pruebas-unitarias).
