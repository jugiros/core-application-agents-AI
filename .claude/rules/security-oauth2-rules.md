# Reglas de Seguridad — OAuth 2.0, JWT e ISO 27001 para APIs Financieras

## Alcance

Estas reglas se aplican a todos los archivos de la capa de Presentación (`**/*Controller.cs`, `**/Middleware/**`, `**/Filters/**`) y a los archivos de configuración de seguridad (`Program.cs`, `ServiceRegistration.cs`). Basadas en `Reglas_Validacion_Seguridad.md` y `Perfil_Arquitecto_Seguridad.md`.

---

## Regla 1 — Desacoplamiento de seguridad (Arquitectura Hexagonal)

- La validación de tokens OAuth 2.0 ocurre **exclusivamente** en Adaptadores Inbound (Controllers, Middleware).
- El **Dominio no debe tener dependencias** de librerías de seguridad: `Microsoft.AspNetCore.Authorization`, `IdentityModel`, `System.Security.Claims`.
- La Aplicación no accede a `IHttpContextAccessor` ni a `ClaimsPrincipal` directamente. Los claims se encapsulan en un objeto tipado (`ICurrentUserContext`) que se inyecta como puerto de Application.

```csharp
// CORRECTO — puerto en Application para claims del usuario
public interface ICurrentUserContext
{
    string UserId { get; }
    IReadOnlyCollection<string> Scopes { get; }
    bool HasScope(string scope);
}

// CORRECTO — implementación en Infrastructure/Adapters
public sealed class HttpCurrentUserContext : ICurrentUserContext
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public string UserId =>
        _httpContextAccessor.HttpContext?.User.FindFirstValue(ClaimTypes.NameIdentifier)
        ?? throw new UnauthorizedAccessException("Usuario no autenticado.");

    public IReadOnlyCollection<string> Scopes =>
        _httpContextAccessor.HttpContext?.User.FindAll("scope")
            .Select(c => c.Value).ToList().AsReadOnly()
        ?? [];

    public bool HasScope(string scope) => Scopes.Contains(scope, StringComparer.OrdinalIgnoreCase);
}

// INCORRECTO — acceso a claims en Application layer
public class GetBalanceQueryHandler
{
    private readonly IHttpContextAccessor _httpContextAccessor; // ← PROHIBIDO en Application
}
```

---

## Regla 2 — Validación estricta del token JWT

- Configurar validación de todos los claims estándar: `ValidateIssuer`, `ValidateAudience`, `ValidateLifetime`, `ValidateIssuerSigningKey`.
- El `TokenValidationParameters` debe establecer `ClockSkew = TimeSpan.Zero` para evitar ventanas de tolerancia inseguras.
- Usar HTTPS exclusivamente. Rechazar tokens transmitidos por HTTP.

```csharp
// CORRECTO — configuración estricta JWT en ServiceRegistration
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = configuration["OAuth:Authority"];
        options.Audience  = configuration["OAuth:Audience"];
        options.RequireHttpsMetadata = true;
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer           = true,
            ValidateAudience         = true,
            ValidateLifetime         = true,
            ValidateIssuerSigningKey = true,
            ClockSkew                = TimeSpan.Zero
        };
    });
```

---

## Regla 3 — Validación estricta de contratos OpenAPI

- Toda API debe exponer un contrato OpenAPI (`/swagger` o `/openapi`) completo antes de estar disponible en cualquier ambiente.
- Ningún dato llega al Dominio sin haber sido validado contra el esquema OpenAPI y el validador FluentValidation del Command/Query.
- Los modelos de request (`FromBody`, `FromRoute`, `FromQuery`) deben usar tipos primitivos o DTOs simples, nunca entidades de dominio.

```csharp
// CORRECTO — endpoint con contrato OpenAPI documentado
[HttpPost("transfer")]
[Authorize(Policy = "RequireTransferWrite")]
[ProducesResponseType(typeof(Guid), StatusCodes.Status202Accepted)]
[ProducesResponseType(StatusCodes.Status400BadRequest)]
[ProducesResponseType(StatusCodes.Status401Unauthorized)]
[ProducesResponseType(StatusCodes.Status403Forbidden)]
[ProducesResponseType(StatusCodes.Status409Conflict)]
public async Task<IActionResult> ExecuteTransfer(
    [FromBody] ExecuteTransferRequest request,
    CancellationToken cancellationToken)
{ ... }
```

---

## Regla 4 — Manejo ético de errores (ISO 27001)

- El `GlobalExceptionHandler` es el único punto que transforma excepciones en respuestas HTTP.
- **Nunca** exponer al cliente: stack traces, mensajes de excepción internos, nombres de tablas, detalles del Identity Provider, rutas de archivos del servidor.
- Usar el mapeo estándar:

| Tipo de excepción | Código HTTP | Mensaje al cliente |
|---|---|---|
| `SecurityTokenExpiredException` | 401 | "Token expirado." |
| `SecurityTokenException` | 401 | "Token inválido." |
| `UnauthorizedAccessException` | 403 | "Acceso denegado." |
| `DomainException` | 422 | Mensaje de negocio (sin detalles técnicos) |
| `ValidationException` | 400 | Lista de errores de validación |
| `NotFoundException` | 404 | "Recurso no encontrado." |
| Cualquier otra `Exception` | 500 | "Error interno del servidor." |

---

## Regla 5 — Principio de mínimo privilegio (ISO 27001 + BIAN)

- Cada endpoint lleva `[Authorize(Policy = "...")]` con la política específica del scope BIAN requerido.
- Las políticas se definen en `ServiceRegistration.cs` usando `RequireClaim("scope", "nombre-del-scope")`.
- El scope `current-account-balance:read` **no** otorga acceso a `payment-execution:write`.
- Se prohibe el uso de `[AllowAnonymous]` en endpoints que manejen datos financieros.

```csharp
// CORRECTO — registro de políticas por scope BIAN
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("RequireBalanceRead", policy =>
        policy.RequireClaim("scope", "current-account-balance:read"));

    options.AddPolicy("RequireTransferWrite", policy =>
        policy.RequireClaim("scope", "payment-execution:write"));

    options.AddPolicy("RequireTransferRead", policy =>
        policy.RequireClaim("scope", "payment-execution:read"));

    options.AddPolicy("RequireAudit", policy =>
        policy.RequireClaim("scope", "transaction-audit:read"));
});

// INCORRECTO — [AllowAnonymous] en endpoint financiero
[AllowAnonymous]
[HttpGet("{id}/balance")]
public async Task<IActionResult> GetBalance(...) { } // ← PROHIBIDO
```

---

## Regla 6 — Auditoría sin exposición de datos sensibles (ISO 27001)

- Los logs de auditoría deben registrar: `userId` (no PII), `action`, `resourceId` (no número de cuenta completo), `timestamp` (UTC), `result` (success/failure), `ipAddress` (hasheada o anonimizada según política).
- **Prohibido** en logs: montos en texto plano, números de cuenta completos, datos de identificación personal (nombre, documento), tokens JWT completos.
- Usar `ILogger<T>` con message templates estructurados. Nunca concatenar strings con datos sensibles.

```csharp
// CORRECTO — log de auditoría estructurado sin datos sensibles
_logger.LogInformation(
    "Transferencia {Result} — Usuario: {UserId} — Origen: {SourceAccountIdHash} — Destino: {TargetAccountIdHash}",
    result.IsSuccess ? "exitosa" : "fallida",
    userId,
    HashAccountId(request.SourceAccountId),
    HashAccountId(request.TargetAccountId));

// INCORRECTO — log con datos sensibles
_logger.LogInformation($"Transferencia de ${amount} de cuenta {sourceAccount} a {targetAccount}"); // ← PROHIBIDO
```

---

## Regla 7 — Configuración segura de secretos

- Las credenciales del Identity Provider, cadenas de conexión y claves de Redis se leen desde variables de entorno o almacén seguro (Azure Key Vault, AWS Secrets Manager, HashiCorp Vault).
- **Nunca** hardcodear secretos en `appsettings.json`, `appsettings.Development.json` ni en código fuente.
- Los archivos `appsettings.json` publicados en repositorio solo contienen estructuras sin valores sensibles (placeholders o vacíos).

```json
// CORRECTO — appsettings.json sin secretos
{
  "OAuth": {
    "Authority": "",
    "Audience": ""
  },
  "ConnectionStrings": {
    "DefaultConnection": ""
  },
  "Redis": {
    "ConnectionString": ""
  }
}
```

---

## Sanciones por incumplimiento

Si un archivo incumple estas reglas, el agente debe:

1. Reportar la infracción con número de regla y línea afectada.
2. Escalar al `senior-security-architect` para validación.
3. No considerar el endpoint como apto para producción hasta que esté conforme.
4. Documentar la lección en `docs/lessons-learned/security-oauth2.md`.
