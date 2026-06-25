# Reglas Backend — .NET Core 9, Arquitectura Hexagonal y CQRS

## Alcance

Estas reglas se aplican a todos los archivos C# del microservicio financiero (`**/*.cs`), excluyendo explícitamente los archivos de ViewModel o presentación cuando aplique. Reemplazan a `backend-net47-rules.md` para el stack .NET Core 9.

---

## Regla 1 — Separación de capas Hexagonal (SRP + DIP)

- El **Dominio** (`Domain`) no importa ninguna librería de infraestructura, seguridad, persistencia ni framework.
  - Namespaces prohibidos en Domain: `Microsoft.EntityFrameworkCore`, `Microsoft.AspNetCore.*`, `StackExchange.Redis`, `IdentityModel`, `System.Data`.
- La **Aplicación** (`Application`) solo depende del Dominio y de interfaces (puertos) propias. Nunca referencia implementaciones de infraestructura directamente.
- La **Infraestructura** (`Infrastructure`) implementa los puertos definidos en Application. Es la única capa que puede importar librerías externas.

```csharp
// CORRECTO — puerto en Application
public interface IAccountRepository
{
    Task<Account?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
}

// CORRECTO — adaptador en Infrastructure
public sealed class AccountRepository : IAccountRepository
{
    private readonly ApplicationDbContext _context;
    // ...
}

// INCORRECTO — Domain importando EF Core
using Microsoft.EntityFrameworkCore; // ← PROHIBIDO en Domain
```

---

## Regla 2 — CQRS: Commands y Queries estrictamente separados

- **Commands** (`IRequest<Result<T>>`): modifican estado. Máximo retornan `Result<Guid>` (Id) o `Result<Unit>`.
- **Queries** (`IRequest<Result<T>>`): solo leen estado. Nunca llaman a `SaveChangesAsync()` ni modifican Redis.
- Un handler no puede ser Command y Query simultáneamente. Si se detecta esta situación, dividir en dos handlers.
- Cada Command/Query es un `record` inmutable.

```csharp
// CORRECTO — Command inmutable
public record ExecuteTransferCommand(
    Guid SourceAccountId,
    Guid TargetAccountId,
    decimal Amount,
    string IdempotencyKey) : IRequest<Result<Guid>>;

// CORRECTO — Query inmutable
public record GetBalanceQuery(Guid AccountId, string UserId) : IRequest<Result<BalanceDto>>;

// INCORRECTO — Command que retorna datos completos de negocio
public record CreateAccountCommand(...) : IRequest<AccountDto>; // ← retornar DTO completo viola CQRS
```

---

## Regla 3 — Gestión correcta del ciclo de vida de recursos

- Todo recurso `IDisposable` se libera mediante bloques `using` o mediante la gestión del contenedor de DI (Scoped/Transient).
- El `DbContext` se inyecta como `Scoped`. Nunca como `Singleton`.
- Las conexiones a Redis se gestionan mediante `IConnectionMultiplexer` registrado como `Singleton`.

```csharp
// CORRECTO — DbContext como Scoped (gestionado por DI)
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(connectionString), ServiceLifetime.Scoped);

// CORRECTO — Redis como Singleton
builder.Services.AddSingleton<IConnectionMultiplexer>(
    ConnectionMultiplexer.Connect(redisConnectionString));
```

---

## Regla 4 — Validación de parámetros (SOLID + Fail-Fast)

- Todo constructor de servicio, handler o repositorio valida sus dependencias con `ArgumentNullException.ThrowIfNull()`.
- Todo Command y Query debe tener un validador `FluentValidation` asociado en `Application/UseCases/Validators/`.
- La validación se ejecuta en el `ValidationBehavior` del pipeline de MediatR, nunca inline en el handler.

```csharp
// CORRECTO — validación de dependencias en constructor
public sealed class GetBalanceQueryHandler : IRequestHandler<GetBalanceQuery, Result<BalanceDto>>
{
    public GetBalanceQueryHandler(IAccountRepository repository, ICacheService cache)
    {
        ArgumentNullException.ThrowIfNull(repository);
        ArgumentNullException.ThrowIfNull(cache);
        _repository = repository;
        _cache = cache;
    }
}

// CORRECTO — validador FluentValidation separado
public sealed class ExecuteTransferCommandValidator : AbstractValidator<ExecuteTransferCommand>
{
    public ExecuteTransferCommandValidator()
    {
        RuleFor(x => x.SourceAccountId).NotEmpty();
        RuleFor(x => x.TargetAccountId).NotEmpty()
            .NotEqual(x => x.SourceAccountId).WithMessage("La cuenta origen y destino no pueden ser iguales.");
        RuleFor(x => x.Amount).GreaterThan(0).WithMessage("El monto debe ser mayor que cero.");
        RuleFor(x => x.IdempotencyKey).NotEmpty().MaximumLength(100);
    }
}
```

---

## Regla 5 — Asincronía en .NET Core 9

- Usar `async`/`await` para toda operación de I/O (base de datos, Redis, HTTP).
- No usar `.Result`, `.Wait()` ni `.GetAwaiter().GetResult()` salvo casos documentados con justificación.
- En librerías de Application e Infrastructure usar `ConfigureAwait(false)`.
- En controllers (Presentation) no es necesario `ConfigureAwait(false)` ya que ASP.NET Core 9 no tiene SynchronizationContext problemático.

```csharp
// CORRECTO — Application/Infrastructure
var account = await _repository.GetByIdAsync(id, cancellationToken).ConfigureAwait(false);

// CORRECTO — Controller (sin ConfigureAwait)
var result = await _mediator.Send(query, cancellationToken);
```

---

## Regla 6 — Control estructurado de excepciones

- El Dominio lanza `DomainException` para violaciones de reglas de negocio.
- La Infraestructura transforma excepciones técnicas en excepciones de dominio cuando la capa superior no debe conocer detalles técnicos.
- El `GlobalExceptionHandler` (Presentation) es el único punto que mapea excepciones a respuestas HTTP.
- Nunca usar capturas genéricas vacías (`catch (Exception) { }`).

```csharp
// CORRECTO — transformación en Infrastructure
try
{
    return await _dbSet.FindAsync([id], cancellationToken).ConfigureAwait(false);
}
catch (DbUpdateException ex)
{
    _logger.LogError(ex, "Error al consultar cuenta {AccountId}", id);
    throw new RepositoryException("Error al recuperar la cuenta.", ex);
}

// INCORRECTO — captura genérica vacía
catch (Exception) { } // ← PROHIBIDO
```

---

## Regla 7 — Result Pattern (DRY + KISS)

- Los handlers retornan `Result<T>` en lugar de lanzar excepciones para flujos de negocio esperados.
- Las excepciones se reservan para situaciones verdaderamente excepcionales (errores de infraestructura, violaciones de contrato).
- No crear múltiples tipos de Result; usar el `Result<T>` definido en `Application/Common/Result.cs`.

```csharp
// CORRECTO — flujo de negocio con Result
if (account is null)
    return Result<BalanceDto>.Failure("Cuenta no encontrada.");

return Result<BalanceDto>.Success(dto);

// INCORRECTO — usar excepciones para flujo esperado
if (account is null)
    throw new AccountNotFoundException(id); // ← usar Result, no excepción
```

---

## Regla 8 — Uso correcto de LINQ

- Preferir ejecución diferida (`Where`, `Select`, `Take`, `Skip`) sobre materialización completa.
- Nunca usar `ToList()` o `ToArray()` a mitad de una consulta compuesta; materializar solo al final.
- Asegurar que las consultas LINQ sobre EF Core sean traducibles a SQL (no usar métodos de cliente que fuercen evaluación en memoria).

```csharp
// CORRECTO — consulta diferida, materialización al final
var balances = await _context.Accounts
    .Where(a => a.OwnerId == userId && a.IsActive)
    .Select(a => new BalanceSummaryDto(a.Id, a.Balance))
    .ToListAsync(cancellationToken)
    .ConfigureAwait(false);

// INCORRECTO — materialización prematura
var all = await _context.Accounts.ToListAsync(); // carga toda la tabla
var filtered = all.Where(a => a.OwnerId == userId); // filtra en memoria
```

---

## Sanciones por incumplimiento

Si un archivo viola estas reglas, el agente debe:

1. Reportar la infracción con el número de regla y la línea afectada.
2. Proponer la refactorización mínima para alcanzar el cumplimiento.
3. Aplicar Boy Scout Rule: si la violación está en un archivo que se está editando, corregirla en el mismo cambio.
4. No considerar la tarea completa hasta que el archivo esté conforme.
