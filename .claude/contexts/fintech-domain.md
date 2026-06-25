# Contexto de Dominio Fintech — Lenguaje Ubicuo

## Propósito

Este archivo define el **lenguaje ubicuo** del dominio financiero. Todo agente debe leer este contexto antes de nombrar clases, métodos, propiedades, Commands, Queries o tablas de base de datos. El código debe reflejar fielmente el negocio bancario, no términos técnicos genéricos.

---

## Bounded Contexts del microservicio

### 1. Gestión de Cuentas (`Account Management`)

Dominio responsable de la información de cuentas bancarias y sus saldos.

| Término del dominio | Clase C# | Descripción |
|---|---|---|
| Cuenta (Account) | `Account` (AggregateRoot) | Entidad central que representa una cuenta bancaria de un titular |
| Titular de Cuenta | `AccountHolder` (ValueObject) | Propietario de una cuenta — identificado por `AccountHolderId` |
| Saldo | `Balance` (ValueObject) | Valor monetario actual de la cuenta — siempre positivo o cero |
| Número de Cuenta | `AccountNumber` (ValueObject) | Identificador único de negocio (no el Id técnico) — inmutable |
| Moneda | `Currency` (ValueObject) | ISO 4217 — ej. `COP`, `USD`, `EUR` |
| Estado de Cuenta | `AccountStatus` (Enum) | `Active`, `Suspended`, `Closed` |
| Límite de Transferencia | `TransferLimit` (ValueObject) | Monto máximo permitido por operación |

### 2. Ejecución de Transferencias (`Payment Execution`)

Dominio BIAN responsable de las instrucciones de pago entre cuentas.

| Término del dominio | Clase C# | Descripción |
|---|---|---|
| Transferencia | `Transfer` (AggregateRoot) | Operación de movimiento de fondos entre dos cuentas |
| Instrucción de Pago | `PaymentInstruction` (ValueObject) | Detalle técnico de la transferencia (cuenta origen, destino, monto) |
| Monto de Transferencia | `TransferAmount` (ValueObject) | Valor monetario de la operación — siempre mayor que cero |
| Referencia de Transferencia | `TransferReference` (ValueObject) | Identificador único de negocio de la transferencia |
| Estado de Transferencia | `TransferStatus` (Enum) | `Pending`, `Processing`, `Completed`, `Failed`, `Reversed` |
| Clave de Idempotencia | `IdempotencyKey` (ValueObject) | Token único generado por el cliente para garantizar unicidad |
| Fecha de Valor | `ValueDate` | Fecha en que la transferencia toma efecto contable |

### 3. Auditoría Transaccional (`Transaction Audit`)

Dominio BIAN responsable del registro histórico de operaciones.

| Término del dominio | Clase C# | Descripción |
|---|---|---|
| Registro de Auditoría | `AuditEntry` (Entity) | Inmutable. Registra cada operación con actor, acción y resultado |
| Actor | `AuditActor` (ValueObject) | Usuario que ejecutó la acción — identificado por `UserId` |
| Evento de Dominio | `IDomainEvent` | Base de todos los eventos de dominio — `TransferCompleted`, `AccountSuspended` |

---

## Eventos de dominio

| Evento | Descripción | Bounded Context |
|---|---|---|
| `TransferInitiated` | Una transferencia fue solicitada y validada | Payment Execution |
| `TransferCompleted` | Una transferencia fue procesada exitosamente | Payment Execution |
| `TransferFailed` | Una transferencia falló por regla de negocio o error técnico | Payment Execution |
| `BalanceUpdated` | El saldo de una cuenta fue modificado | Account Management |
| `AccountSuspended` | Una cuenta fue suspendida por política | Account Management |

---

## Commands del sistema (escritura — modifican estado)

| Command | Handler | Descripción |
|---|---|---|
| `ExecuteTransferCommand` | `ExecuteTransferCommandHandler` | Inicia y procesa una transferencia entre cuentas |
| `ReverseTransferCommand` | `ReverseTransferCommandHandler` | Reversa una transferencia completada |
| `SuspendAccountCommand` | `SuspendAccountCommandHandler` | Suspende una cuenta activa |

## Queries del sistema (lectura — no modifican estado)

| Query | Handler | Descripción |
|---|---|---|
| `GetBalanceQuery` | `GetBalanceQueryHandler` | Obtiene el saldo actual de una cuenta (con Redis Cache-Aside) |
| `GetTransferByIdQuery` | `GetTransferByIdQueryHandler` | Obtiene el detalle de una transferencia específica |
| `GetTransferHistoryQuery` | `GetTransferHistoryQueryHandler` | Obtiene el historial paginado de transferencias de una cuenta |

---

## Reglas de negocio críticas

1. **Fondos suficientes**: Una transferencia solo puede ejecutarse si el saldo disponible de la cuenta origen es mayor o igual al monto solicitado más las comisiones aplicables.
2. **Límite por operación**: El monto de una transferencia no puede superar el `TransferLimit` configurado para la cuenta origen.
3. **Cuentas activas**: Solo las cuentas con `AccountStatus.Active` pueden participar en transferencias (ni origen ni destino pueden estar `Suspended` o `Closed`).
4. **Misma moneda**: La transferencia requiere que la cuenta origen y la cuenta destino operen en la misma `Currency`.
5. **Idempotencia**: Dos solicitudes con el mismo `IdempotencyKey` deben producir exactamente un resultado, independientemente de cuántas veces se invoque.
6. **Inmutabilidad de auditoría**: Los `AuditEntry` nunca se modifican ni eliminan. Solo se crean.

---

## Terminología prohibida (anti-patterns de naming)

| Término genérico | Usar en cambio |
|---|---|
| `Data`, `Info`, `Object` | Nombre del concepto de negocio concreto |
| `DoTransfer`, `ProcessPayment` | `ExecuteTransfer`, `InitiatePayment` |
| `GetData`, `FetchInfo` | `GetBalance`, `GetTransferHistory` |
| `User` (en contexto financiero) | `AccountHolder`, `AuditActor` |
| `Amount` (sin contexto) | `TransferAmount`, `Balance` |
| `Status` (sin prefijo) | `AccountStatus`, `TransferStatus` |

---

## Scopes BIAN v12 y mapping

| Dominio BIAN | Scope | Operación |
|---|---|---|
| Current Account (`CA`) | `current-account-balance:read` | Consultar saldo |
| Payment Execution (`PE`) | `payment-execution:write` | Ejecutar transferencia |
| Payment Execution (`PE`) | `payment-execution:read` | Consultar historial |
| Transaction Audit (`TA`) | `transaction-audit:read` | Auditar transacciones |

---

## Notas de arquitectura

- El Dominio usa **lenguaje ubicuo**: los nombres de clases y métodos reflejan términos del glosario de esta página.
- Los DTOs de la capa de Application pueden usar nombres más simples para la API, pero los mappers deben hacer el puente explícito.
- Los nombres de tablas de BD deben alinearse con este lenguaje ubicuo (coordinado con `senior-dba`).
