# Reglas de Caché — Redis en Microservicio Financiero (.NET Core 9)

## Alcance

Estas reglas se aplican a todos los archivos que interactúan con Redis: `**/*CacheService*.cs`, `**/*Repository*.cs` (cuando usen caché), `**/Infrastructure/Redis/**`, y handlers de Queries en `**/Application/UseCases/Queries/**`.

---

## Regla 1 — TTL obligatorio en toda operación de escritura

- **Ninguna clave en Redis puede persistirse sin un TTL explícito**. Claves sin expiración son una vulnerabilidad de disponibilidad y un riesgo de acumulación de datos obsoletos.
- TTL mínimos por tipo de dato financiero:

| Tipo de dato | TTL recomendado | Justificación |
|---|---|---|
| Saldo de cuenta | 30 segundos | Datos volátiles — transferencias frecuentes |
| Historial de transferencias (paginado) | 60 segundos | Consultas repetidas con datos estables a corto plazo |
| Idempotency key de transferencia | 24 horas | Ventana de protección contra duplicados |
| Sesión de usuario autenticado | 15 minutos | Alineado con expiración del access token OAuth 2.0 |
| Configuración de tasas/límites | 5 minutos | Puede cambiar por administración |

```csharp
// CORRECTO — SetAsync con TTL explícito
await _cache.SetAsync("balance:account-123", dto, TimeSpan.FromSeconds(30), cancellationToken).ConfigureAwait(false);

// INCORRECTO — sin TTL
await _cache.SetAsync("balance:account-123", dto, cancellationToken); // ← PROHIBIDO
```

---

## Regla 2 — Patrón Cache-Aside exclusivo para Queries

- Las Queries implementan Cache-Aside: **verificar Redis → si miss, consultar BD → actualizar Redis → retornar**.
- Los Commands **nunca leen de Redis** para obtener datos de negocio; solo escriben para idempotencia o invalidan claves.
- La actualización del caché en una Query ocurre **después** de obtener datos de la BD, no antes.

```csharp
// CORRECTO — Cache-Aside en Query Handler
public async Task<Result<BalanceDto>> Handle(GetBalanceQuery request, CancellationToken cancellationToken)
{
    var cacheKey = $"balance:{request.AccountId}";

    // 1. Verificar caché
    var cached = await _cache.GetAsync<BalanceDto>(cacheKey, cancellationToken).ConfigureAwait(false);
    if (cached is not null)
        return Result<BalanceDto>.Success(cached);

    // 2. Cache miss — consultar BD
    var account = await _repository.GetByIdAsync(request.AccountId, cancellationToken).ConfigureAwait(false);
    if (account is null)
        return Result<BalanceDto>.Failure("Cuenta no encontrada.");

    // 3. Actualizar caché
    var dto = _mapper.Map<BalanceDto>(account);
    await _cache.SetAsync(cacheKey, dto, TimeSpan.FromSeconds(30), cancellationToken).ConfigureAwait(false);

    return Result<BalanceDto>.Success(dto);
}
```

---

## Regla 3 — Invalidación de caché en Commands transaccionales

- Todo Command que modifique el saldo de una cuenta debe invalidar las claves de caché correspondientes **después** de confirmar la escritura en BD (post-commit).
- La invalidación es **best-effort**: si falla la invalidación de Redis, no se revierte la transacción. El TTL garantiza la consistencia eventual.
- Invalidar tanto la cuenta origen como la cuenta destino en transferencias.

```csharp
// CORRECTO — invalidación post-commit en Command Handler
await _unitOfWork.SaveChangesAsync(cancellationToken).ConfigureAwait(false); // commit primero

// Luego invalidar caché (best-effort)
await _cache.DeleteAsync($"balance:{request.SourceAccountId}", cancellationToken).ConfigureAwait(false);
await _cache.DeleteAsync($"balance:{request.TargetAccountId}", cancellationToken).ConfigureAwait(false);

// INCORRECTO — invalidar antes del commit
await _cache.DeleteAsync($"balance:{request.SourceAccountId}", cancellationToken); // ← invalidar antes es race condition
await _unitOfWork.SaveChangesAsync(cancellationToken);
```

---

## Regla 4 — Idempotencia de transferencias con Redis

- Antes de procesar cualquier transferencia, verificar la existencia del `IdempotencyKey` en Redis.
- Si la clave existe, retornar `Result.Failure("Transferencia ya procesada.")` sin re-ejecutar la operación.
- La clave de idempotencia se persiste **después** del commit exitoso, con TTL de 24 horas.
- El formato de clave de idempotencia debe ser: `transfer:idem:{idempotencyKey}`.

```csharp
// CORRECTO — verificación de idempotencia al inicio del handler
var idempotencyKey = $"transfer:idem:{request.IdempotencyKey}";

if (await _cache.ExistsAsync(idempotencyKey, cancellationToken).ConfigureAwait(false))
    return Result<Guid>.Failure("Transferencia ya procesada (idempotente).");

// ... procesar transferencia ...

// Registrar clave de idempotencia tras commit exitoso
await _cache.SetAsync(idempotencyKey, transferId.ToString(), TimeSpan.FromHours(24), cancellationToken).ConfigureAwait(false);
```

---

## Regla 5 — Convención de nombres de claves Redis

- Las claves deben seguir el formato: `{entidad}:{identificador}` o `{entidad}:{operación}:{identificador}`.
- Usar minúsculas y guiones medios para separar palabras compuestas.
- Nunca incluir datos sensibles (montos, números de cuenta completos, PII) en el nombre de la clave.

```
// CORRECTO
balance:{accountId}                    → saldo de cuenta
transfer:idem:{idempotencyKey}         → idempotencia de transferencia
session:{userId}                       → sesión de usuario
transfer:history:{accountId}:{page}    → historial paginado

// INCORRECTO
saldo-cuenta-12345678-monto-5000       → expone número de cuenta y monto
Balance_Account_{accountId}            → PascalCase no estándar en Redis
```

---

## Regla 6 — Abstracción de Redis en puerto de Application

- La capa de Application no referencia `StackExchange.Redis` directamente. Usa el puerto `ICacheService` definido en `Application/Ports/Driven/`.
- La implementación concreta de `ICacheService` con `StackExchange.Redis` pertenece exclusivamente a la capa de Infrastructure.

```csharp
// CORRECTO — puerto en Application
public interface ICacheService
{
    Task<T?> GetAsync<T>(string key, CancellationToken cancellationToken = default);
    Task SetAsync<T>(string key, T value, TimeSpan ttl, CancellationToken cancellationToken = default);
    Task DeleteAsync(string key, CancellationToken cancellationToken = default);
    Task<bool> ExistsAsync(string key, CancellationToken cancellationToken = default);
}

// CORRECTO — implementación en Infrastructure
public sealed class RedisCacheService : ICacheService
{
    private readonly IDatabase _database;

    public RedisCacheService(IConnectionMultiplexer connection)
    {
        ArgumentNullException.ThrowIfNull(connection);
        _database = connection.GetDatabase();
    }
    // ...
}

// INCORRECTO — importar StackExchange.Redis en Application
using StackExchange.Redis; // ← PROHIBIDO en Application o Domain
```

---

## Regla 7 — Manejo de errores de conexión Redis

- Si Redis no está disponible, el sistema debe degradar gracefulmente al origen de datos (BD) sin lanzar excepciones no controladas al cliente.
- Implementar circuit breaker con política de retry en la capa de Infrastructure (Polly v8 o Microsoft.Extensions.Resilience).
- Registrar errores de Redis como `Warning` (no `Error`) cuando el sistema puede operar sin caché.

```csharp
// CORRECTO — degradación graceful ante fallo de Redis
try
{
    var cached = await _database.StringGetAsync(key).ConfigureAwait(false);
    if (cached.HasValue)
        return JsonSerializer.Deserialize<T>(cached!);
}
catch (RedisConnectionException ex)
{
    _logger.LogWarning(ex, "Redis no disponible para clave {CacheKey}. Continuando sin caché.", key);
}

return default; // Cache miss graceful
```

---

## Sanciones por incumplimiento

Si un archivo incumple estas reglas, el agente debe:

1. Reportar la infracción con número de regla y archivo afectado.
2. Proponer la corrección mínima para alcanzar el cumplimiento.
3. Coordinar con `senior-cqrs-specialist` para validar la estrategia de caché resultante.
4. No considerar la tarea completa hasta que el archivo esté conforme.
