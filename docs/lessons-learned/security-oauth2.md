# Lecciones Aprendidas — Seguridad OAuth 2.0, JWT e ISO 27001

## Propósito

Registro histórico de decisiones de seguridad, vulnerabilidades detectadas y soluciones aplicadas para el microservicio financiero. Todo agente debe leer este archivo antes de implementar o modificar cualquier endpoint o capa de seguridad.

---

## LS-001 — Los claims NO se acceden desde la capa de Application

**Fecha**: 2026-06  
**Agente**: senior-security-architect  
**Categoría**: Hexagonal — Desacoplamiento de seguridad

**Situación**: Un Query Handler inyectó `IHttpContextAccessor` para obtener el `userId` del token JWT directamente en la capa de Application, violando el aislamiento hexagonal.

**Decisión**: Los claims se extraen en el Controller (Adaptador Inbound) y se pasan como parámetros tipados en el Command/Query. La Application nunca accede a `HttpContext`.

```csharp
// CORRECTO — extraer en Controller, pasar como parámetro
var userId = User.FindFirstValue(ClaimTypes.NameIdentifier)
    ?? throw new UnauthorizedAccessException("Token sin identificador de usuario.");

var query = new GetBalanceQuery(accountId, UserId: userId);
```

**Verificación**: Ningún archivo en `Application/` importa `Microsoft.AspNetCore.Http`.

---

## LS-002 — ClockSkew = TimeSpan.Zero es obligatorio para tokens financieros

**Fecha**: 2026-06  
**Agente**: senior-security-architect  
**Categoría**: JWT — Validación estricta

**Situación**: Con `ClockSkew` por defecto (5 minutos), un token expirado podía seguir siendo válido hasta 5 minutos después de su `exp`. En un sistema financiero, esto representa una ventana de vulnerabilidad inaceptable.

**Decisión**: `ClockSkew = TimeSpan.Zero` es obligatorio en la configuración de JWT. Un token expirado es rechazado inmediatamente.

```csharp
options.TokenValidationParameters = new TokenValidationParameters
{
    ClockSkew = TimeSpan.Zero // ← obligatorio en Fintech
};
```

---

## LS-003 — [AllowAnonymous] está prohibido en endpoints financieros

**Fecha**: 2026-06  
**Agente**: senior-security-architect  
**Categoría**: OAuth 2.0 — Mínimo privilegio

**Situación**: Durante el desarrollo, se agregó `[AllowAnonymous]` a un endpoint de consulta de saldo para facilitar pruebas. El endpoint llegó a QA sin el atributo `[Authorize]` correcto.

**Decisión**: `[AllowAnonymous]` está prohibido en cualquier endpoint que maneje datos financieros. Para pruebas, usar tokens de prueba válidos del Identity Provider de desarrollo.

**Proceso de revisión**: El `senior-security-architect` valida todos los controllers antes de cualquier merge a rama principal.

---

## LS-004 — Los errores de seguridad no exponen detalles técnicos al cliente

**Fecha**: 2026-06  
**Agente**: senior-security-architect  
**Categoría**: ISO 27001 — Manejo ético de errores

**Situación**: El handler de excepciones inicial retornaba el mensaje completo de `SecurityTokenExpiredException` al cliente, exponiendo el nombre de la librería y detalles del Identity Provider.

**Decisión**: El `GlobalExceptionHandler` mapea todas las excepciones de seguridad a respuestas genéricas. Solo el código HTTP y un mensaje simple llegan al cliente. El detalle técnico se registra en logs internos (sin datos sensibles).

```csharp
SecurityTokenExpiredException => (401, "Token expirado."),
SecurityTokenException        => (401, "Token inválido."),
UnauthorizedAccessException   => (403, "Acceso denegado."),
```

---

## LS-005 — Los scopes BIAN deben ser granulares, nunca genéricos

**Fecha**: 2026-06  
**Agente**: senior-security-architect  
**Categoría**: OAuth 2.0 — Principio de mínimo privilegio

**Situación**: Se diseñó un scope genérico `accounts:manage` que cubría tanto lectura de saldos como ejecución de transferencias. Un usuario con rol `viewer` podía ejecutar transferencias si obtenía ese scope.

**Decisión**: Los scopes son granulares por operación BIAN:
- `current-account-balance:read` — solo lectura de saldo
- `payment-execution:write` — solo escritura de transferencias
- `payment-execution:read` — solo lectura de historial

Un scope de lectura nunca otorga permisos de escritura.

---

## LS-006 — Los logs de auditoría nunca contienen datos financieros en texto plano

**Fecha**: 2026-06  
**Agente**: senior-security-architect  
**Categoría**: ISO 27001 — Confidencialidad de datos

**Situación**: Se registró un log de debug que incluía el monto de la transferencia y el número de cuenta completo para facilitar el diagnóstico.

**Decisión**: Los logs de auditoría registran:
- `userId` (no nombre ni documento)
- `accountId` o hash del número de cuenta (nunca el número completo)
- `action` (nombre de la operación)
- `timestamp` (UTC)
- `result` (success / failure)

**Prohibido en logs**: montos, números de cuenta completos, nombres de titulares, tokens JWT, datos de sesión.

---

## LS-007 — Los secretos de OAuth no van en appsettings.json del repositorio

**Fecha**: 2026-06  
**Agente**: senior-security-architect  
**Categoría**: ISO 27001 — Gestión de secretos

**Situación**: Las credenciales del Identity Provider de desarrollo fueron commiteadas en `appsettings.Development.json` y llegaron al repositorio.

**Decisión**: `appsettings.json` y `appsettings.Development.json` solo contienen estructuras vacías (claves sin valor). Los valores reales se proveen mediante:
- Variables de entorno en CI/CD y producción.
- `dotnet user-secrets` en desarrollo local.
- Azure Key Vault / AWS Secrets Manager en ambientes cloud.

**Proceso de revisión**: Agregar regla en `.gitignore` para `secrets.json`. Revisar en code review que ningún `appsettings*.json` contenga valores de credenciales.

---

## LS-008 — RequireHttpsMetadata = true siempre en producción

**Fecha**: 2026-06  
**Agente**: senior-security-architect  
**Categoría**: OAuth 2.0 — Seguridad en tránsito

**Situación**: `RequireHttpsMetadata = false` fue dejado para facilitar el desarrollo local y pasó a producción sin revertirse, permitiendo que tokens fueran transmitidos por HTTP.

**Decisión**: `RequireHttpsMetadata = true` siempre. Para desarrollo local sin HTTPS, usar variables de entorno para sobrescribir solo en ambiente `Development` con aprobación explícita del equipo.

---

## Cómo agregar una nueva lección

```
## LS-{numero} — {título corto}

**Fecha**: YYYY-MM  
**Agente**: {nombre-del-agente}  
**Categoría**: {categoría — OAuth / JWT / ISO 27001 / BIAN}

**Situación**: Descripción del problema de seguridad encontrado.

**Decisión**: Solución aplicada y razón de seguridad.

**Verificación** (opcional): Cómo verificar que no se repita.
```
