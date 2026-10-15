# Motor de reglas deterministas

## 1. Principios de implementación

El motor debe ser una biblioteca independiente que reciba datos validados y un contexto confiable.

```ts
evaluatePayment(
  request,
  trustedContext,
  policySnapshot
): EngineDecision
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
| PAY-001 | Límite de autorización | Si el importe supera $10,000 MXN, existe una aprobación válida |
| PAY-002 | Categoría permitida | Categoría en `OPERATIONS`, `SOFTWARE` o `TRAVEL` |
| PAY-003 | Separación de funciones | El aprobador es distinto del solicitante y tiene permiso |
| PAY-004 | Factura verificada | Existe factura confirmada por el backend |
| PAY-005 | Presupuesto disponible | El importe no supera el saldo disponible |

Las reglas de ejemplo son políticas configurables para el MVP, no políticas universales.

### PAY-001: límite de autorización

```ts
if (amountMinor > approvalThresholdMinor) {
  return validSupervisorApproval
    ? PASS
    : FAIL;
}
return PASS;
```

`validSupervisorApproval` debe derivarse de un registro confiable, verificando la identidad, el permiso y la vigencia de la aprobación. No basta con que el cliente envíe `APPROVED`.

### PAY-002: categoría

```ts
const allowedCategories = [
  "OPERATIONS",
  "SOFTWARE",
  "TRAVEL"
];

return allowedCategories.includes(category)
  ? PASS
  : FAIL;
```

La lista definitiva debe estar en la política del backend, no duplicada como autoridad en el frontend.

### PAY-003: separación de funciones

```ts
const validApprover =
  requesterId !== approverId &&
  approverIsActive &&
  approverHasPermission;

return validApprover ? PASS : FAIL;
```

Cuando una aprobación especial no sea necesaria, la regla debe evaluar los permisos que correspondan al proceso. No debe exigir que todos los pagos tengan un supervisor solo porque el ejemplo contiene ese campo.

### PAY-004: factura verificada

El backend consulta el registro del adjunto **en el contexto confiable**, nunca en el payload del cliente. Una factura no se considera verificada solo porque el request incluya `"verified": true` o un campo equivalente: ese valor, si el cliente llega a enviarlo, se ignora. La única fuente válida es la resolución que el backend hace de `attachment_id`/`reference` contra sus propios registros (ver [api-contracts.md § 3](api-contracts.md#3-post-apiv1executions)). Si la referencia no existe o la factura no está verificada ahí, la regla falla. Si la fuente de verificación no responde, el resultado será `INDETERMINATE`.

### PAY-005: presupuesto

```ts
return amountMinor <= availableBudgetMinor
  ? PASS
  : FAIL;
```

La comprobación debe utilizar un saldo confiable. Si dos operaciones concurrentes consumen presupuesto, la reserva o actualización del saldo debe implementarse con transacciones y control de concurrencia. Para el MVP basta con simular el presupuesto y comprobarlo sin mover dinero real.

## 3. Resultado global

```ts
const mustBlock = results.some(
  result =>
    result.status === "FAIL" ||
    result.status === "INDETERMINATE"
);

const decision = mustBlock ? "BLOCK" : "ALLOW";
```

La evaluación debe producir los resultados de todas las reglas, incluso si la primera falla, para permitir una explicación completa en el panel. Nótese que `INDETERMINATE` tiene el mismo efecto que `FAIL` sobre `mustBlock`: una condición obligatoria que no se puede verificar bloquea la operación igual que una infracción confirmada (política *fail-closed*, ver [api-contracts.md § 0](api-contracts.md#0-modelo-de-estados-resumen)), aunque el informe de evidencias debe seguir distinguiendo ambos casos en su texto, sin presentarlos como el mismo hecho.

Cada ejecución de este resultado global queda asociada a `policy_version` (la versión de `policy_rules` vigente al momento de evaluar). Dos ejecuciones con el mismo `payment_request` pero evaluadas bajo versiones de política distintas pueden producir decisiones distintas de forma legítima; esto se audita mediante `executions.policy_version` (ver [database.md § 1.6](database.md#16-executions)).

## 4. Casos de prueba obligatorios

1. $8,500 MXN sin aprobación especial: pasa PAY-001.
2. $10,000 MXN: no activa la condición `amount > 10000`.
3. $10,000.01 MXN: requiere aprobación.
4. $15,000 MXN sin aprobación válida: falla PAY-001.
5. Categoría desconocida: falla PAY-002.
6. Solicitante intentando autorizar su propio pago: falla PAY-003.
7. Factura ausente: falla PAY-004.
8. Fuente de verificación de factura no disponible: PAY-004 indeterminada.
9. Presupuesto insuficiente: falla PAY-005.
10. Presupuesto no verificable: PAY-005 indeterminada.

Ver también el detalle de ejecución de estos casos como pruebas unitarias en [testing.md](testing.md#1-pruebas-unitarias).
