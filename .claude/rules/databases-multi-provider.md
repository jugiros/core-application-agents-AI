# Reglas de Polipersistencia — MySQL, MongoDB y Redis

## Alcance

Estas reglas aplican a todos los archivos de infraestructura de datos: `**/Infrastructure/MySQL/**`, `**/Infrastructure/MongoDB/**`, `**/Infrastructure/Redis/**`, `**/Infrastructure/DependencyInjection/**`, y a todos los handlers CQRS que interactúan con cualquier base de datos.

---

## Regla 1 — Qué base de datos usar para cada operación (arquitectura de decisión)

| Caso de uso | Base de datos | Razón |
|---|---|---|
| Crear / modificar Account o Transfer | **MySQL** | ACID — consistencia fuerte no negociable en operaciones financieras |
| Consultar saldo de cuenta | **MongoDB** (read model) + **Redis** (caché L1) | Lectura eventual de alta velocidad desde proyección denormalizada |
| Consultar historial de transferencias | **MongoDB** | Documentos pre-calculados con paginación eficiente via `skip/limit` |
| Registro de auditoría | **MongoDB** | Append-only, schema flexible, no requiere transacciones |
| Claves de idempotencia | **Redis** | TTL automático, operación atómica `SET NX` |
| Sesiones y caché de tokens | **Redis** | TTL configurable, volatilidad aceptada |
| Configuración de límites/tarifas | **Redis** | Actualización frecuente, lectura de baja latencia |

**Regla de oro**: Si la operación requiere ACID → MySQL. Si es solo lectura y puede tolerar consistencia eventual → MongoDB. Si necesita expiración automática → Redis.

---

## Regla 2 — Abstracciones de puertos por base de datos (DIP)

Cada almacén de datos tiene su propio puerto en `Application/Ports/Driven/`. Nunca mezclar interfaces de distintas bases de datos:

```csharp
// CORRECTO — un puerto por almacén
public interface IAccountWriteRepository   // → implementado con MySQL
{
    Task AddAsync(Account account, CancellationToken ct = default);
    Task<Account?> GetByIdAsync(Guid id, CancellationToken ct = default);
}

public interface IBalanceReadModel          // → implementado con MongoDB
{
    Task<BalanceDocument?> GetAsync(Guid accountId, CancellationToken ct = default);
    Task UpsertAsync(BalanceDocument document, CancellationToken ct = default);
}

public interface ICacheService              // → implementado con Redis
{
    Task<T?> GetAsync<T>(string key, CancellationToken ct = default);
    Task SetAsync<T>(string key, T value, TimeSpan ttl, CancellationToken ct = default);
    Task DeleteAsync(string key, CancellationToken ct = default);
    Task<bool> ExistsAsync(string key, CancellationToken ct = default);
}

// INCORRECTO — mezclar MySQL y MongoDB en una interfaz
public interface IAccountRepository
{
    Task<Account?> GetFromMySqlAsync(...);     // ← write model
    Task<BalanceDocument?> GetFromMongoAsync(...); // ← read model — NO mezclar
}
```

---

## Regla 3 — Commands usan MySQL; Queries usan MongoDB + Redis

- **Command Handlers** inyectan `IAccountWriteRepository` (MySQL) e `IUnitOfWork` (EF Core).
- **Query Handlers** inyectan `IBalanceReadModel` o `ITransferReadRepository` (MongoDB) e `ICacheService` (Redis).
- **Ningún Command Handler** accede directamente a MongoDB o Redis para datos de negocio.
- **Ningún Query Handler** llama a `IUnitOfWork.SaveChangesAsync()`.

```csharp
// CORRECTO — Command Handler usa MySQL
public sealed class ExecuteTransferCommandHandler : IRequestHandler<ExecuteTransferCommand, Result<Guid>>
{
    public ExecuteTransferCommandHandler(
        IAccountWriteRepository repository,  // ← MySQL
        IUnitOfWork unitOfWork,              // ← EF Core / MySQL
        ICacheService cache)                 // ← Redis solo para idempotencia
    { }
}

// CORRECTO — Query Handler usa MongoDB + Redis
public sealed class GetBalanceQueryHandler : IRequestHandler<GetBalanceQuery, Result<BalanceDto>>
{
    public GetBalanceQueryHandler(
        IBalanceReadModel readModel,  // ← MongoDB
        ICacheService cache)         // ← Redis
    { }
}
```

---

## Regla 4 — Proyecciones: sincronización MySQL → MongoDB

Cuando un Command modifica el estado en MySQL, debe publicar un Domain Event que actualice el read model en MongoDB. El mecanismo es:

1. Command Handler escribe en MySQL y emite evento de dominio (ej. `TransferCompletedEvent`).
2. Domain Event Handler (en Infrastructure) actualiza la colección MongoDB correspondiente.
3. Domain Event Handler invalida la clave Redis del saldo afectado.

```csharp
// CORRECTO — event handler que actualiza read model MongoDB
public sealed class TransferCompletedEventHandler : INotificationHandler<TransferCompletedEvent>
{
    private readonly IBalanceReadModel _readModel;
    private readonly ICacheService _cache;

    public async Task Handle(TransferCompletedEvent notification, CancellationToken ct)
    {
        // Actualizar read model en MongoDB
        await _readModel.UpsertAsync(new BalanceDocument
        {
            AccountId     = notification.SourceAccountId,
            Balance       = notification.NewSourceBalance,
            LastUpdatedAt = notification.OccurredOn
        }, ct).ConfigureAwait(false);

        // Invalidar caché Redis
        await _cache.DeleteAsync($"balance:{notification.SourceAccountId}", ct).ConfigureAwait(false);
    }
}
```

---

## Regla 5 — Configuración de conexiones: solo desde variables de entorno

Ninguna cadena de conexión (MySQL, MongoDB, Redis) se hardcodea en `appsettings.json` ni en código fuente.

```json
// CORRECTO — appsettings.json con estructura vacía
{
  "ConnectionStrings": {
    "MySQL": "",
    "MongoDB": "",
    "Redis": ""
  }
}
```

```csharp
// CORRECTO — leer desde configuración (que viene de env vars en producción)
var mysqlConn   = configuration.GetConnectionString("MySQL")!;
var mongoConn   = configuration.GetConnectionString("MongoDB")!;
var redisConn   = configuration.GetConnectionString("Redis")!;
```

---

## Regla 6 — Registro de servicios separado por almacén (SRP en DI)

Crear un método de extensión por base de datos en `Infrastructure/DependencyInjection/`:

```csharp
// CORRECTO — métodos separados por almacén
public static class ServiceRegistration
{
    public static IServiceCollection AddMySqlPersistence(this IServiceCollection services, string connectionString)
    {
        services.AddDbContext<FintechDbContext>(options =>
            options.UseMySql(connectionString, ServerVersion.AutoDetect(connectionString),
                sql => sql.EnableRetryOnFailure(3)));
        services.AddScoped<IUnitOfWork>(sp => sp.GetRequiredService<FintechDbContext>());
        services.AddScoped<IAccountWriteRepository, AccountMySqlRepository>();
        return services;
    }

    public static IServiceCollection AddMongoDbReadStore(this IServiceCollection services, string connectionString)
    {
        services.AddSingleton<IMongoClient>(new MongoClient(connectionString));
        services.AddScoped<IBalanceReadModel, BalanceMongoReadModel>();
        services.AddScoped<ITransferReadRepository, TransferMongoReadRepository>();
        return services;
    }

    public static IServiceCollection AddRedisCache(this IServiceCollection services, string connectionString)
    {
        services.AddSingleton<IConnectionMultiplexer>(ConnectionMultiplexer.Connect(connectionString));
        services.AddSingleton<ICacheService, RedisCacheService>();
        return services;
    }
}
```

---

## Regla 7 — Esquema de colecciones MongoDB (naming convention)

| Colección | Documento principal | Índices requeridos |
|---|---|---|
| `balance_read_models` | `BalanceDocument` | `accountId` (unique), `lastUpdatedAt` |
| `transfer_history` | `TransferHistoryDocument` | `accountId + createdAt` (compound), `transferId` (unique) |
| `audit_log` | `AuditDocument` | `userId + timestamp`, `action` |
| `domain_events` | `DomainEventDocument` | `aggregateId + occurredOn`, `eventType` |

---

## Regla 8 — Validación con MCP (uso de herramientas MCP en agentes)

Los agentes `senior-dba` y `senior-cqrs-specialist` deben usar los MCP servers para validar estado antes de completar una tarea:

| Validación | MCP Server | Comando de ejemplo |
|---|---|---|
| Confirmar que tabla MySQL existe | `mysql-fintech` | `SHOW TABLES LIKE 'transfers'` |
| Confirmar índice en MySQL | `mysql-fintech` | `SHOW INDEX FROM accounts` |
| Confirmar colección MongoDB | `mongodb-fintech` | `db.getCollectionNames()` |
| Confirmar índice en MongoDB | `mongodb-fintech` | `db.balance_read_models.getIndexes()` |
| Confirmar clave en Redis | `redis-fintech` | `EXISTS balance:{accountId}` |
| Confirmar TTL en Redis | `redis-fintech` | `TTL balance:{accountId}` |

---

## Sanciones por incumplimiento

Si se detecta que un Command Handler lee de MongoDB o un Query Handler escribe en MySQL:

1. Reportar la violación con el archivo y línea afectada.
2. Rediseñar siguiendo el patrón correcto (Regla 3).
3. Agregar la lección en `docs/lessons-learned/backend-net9.md`.
4. No aprobar el PR hasta que esté corregido.
