# Reglas de Event-Driven Architecture (EDA) con Kafka

## Contexto: Arquitectura de 3 Microservicios

```
[auth-service]              → MySQL (credenciales, tokens)
       │
[core-transactions-service] → MySQL (write side: ACID)  ──► KAFKA ──► [account-queries-service]
                                                                              │
                                                                    MongoDB (read models)
                                                                    Redis (caché L1)
```

**Principio EDA**: `core-transactions-service` es productor. `account-queries-service` es consumidor. El evento viaja por Kafka garantizando consistencia eventual sin acoplamiento directo.

---

## Regla 1 — Naming Convention de Topics Kafka

**Formato**: `{empresa}.{dominio}.{evento-en-kebab-case}`

| Topic | Producido por | Consumido por | Descripción |
|---|---|---|---|
| `fintech.transactions.executed` | core-transactions | account-queries | Transferencia exitosa en MySQL |
| `fintech.transactions.failed` | core-transactions | account-queries | Transferencia fallida |
| `fintech.accounts.balance-updated` | core-transactions | account-queries | Saldo actualizado tras una operación |
| `fintech.accounts.status-changed` | core-transactions | account-queries | Estado de cuenta cambiado |
| `fintech.auth.user-registered` | auth-service | core-transactions | Usuario nuevo registrado |
| `fintech.auth.session-revoked` | auth-service | core-transactions | Sesión o token revocado |

**Configuración de topics** (crear al iniciar Kafka):
```bash
# Particiones: 3, Replication factor: 1 (desarrollo), 3 (producción)
docker exec kafka-fintech kafka-topics.sh \
  --create --topic fintech.transactions.executed \
  --bootstrap-server localhost:9092 \
  --partitions 3 --replication-factor 1

docker exec kafka-fintech kafka-topics.sh \
  --create --topic fintech.accounts.balance-updated \
  --bootstrap-server localhost:9092 \
  --partitions 3 --replication-factor 1
```

---

## Regla 2 — Consumer Groups

**Formato**: `{microservicio}-consumer-group`

| Consumer Group | Microservicio | Topics que consume |
|---|---|---|
| `account-queries-consumer-group` | account-queries-service | `fintech.transactions.*`, `fintech.accounts.*` |
| `core-transactions-consumer-group` | core-transactions-service | `fintech.auth.*` |

---

## Regla 3 — Esquema de Eventos de Dominio (formato estándar)

Todo evento debe seguir esta estructura:

```csharp
public abstract record DomainEventMessage
{
    public Guid EventId       { get; init; } = Guid.NewGuid();
    public string EventType   { get; init; } = string.Empty;
    public string Topic       { get; init; } = string.Empty;
    public DateTime OccurredOn { get; init; } = DateTime.UtcNow;
    public int SchemaVersion  { get; init; } = 1;
}

public record TransactionExecutedMessage : DomainEventMessage
{
    public Guid   TransactionId    { get; init; }
    public Guid   SourceAccountId  { get; init; }
    public Guid   TargetAccountId  { get; init; }
    public decimal Amount          { get; init; }
    public string Currency         { get; init; } = "COP";
    public decimal NewSourceBalance { get; init; }
    public decimal NewTargetBalance { get; init; }
    public string IdempotencyKey   { get; init; } = string.Empty;
}

public record BalanceUpdatedMessage : DomainEventMessage
{
    public Guid    AccountId   { get; init; }
    public decimal NewBalance  { get; init; }
    public string  Currency    { get; init; } = "COP";
    public DateTime UpdatedAt  { get; init; }
}
```

---

## Regla 4 — Patrón Outbox (garantía de entrega at-least-once)

**Problema**: Si la publicación a Kafka falla después de confirmar la transacción MySQL, el evento se pierde.

**Solución**: Outbox Pattern — guardar el evento en MySQL en la misma transacción, publicar a Kafka de forma asíncrona.

```
Command Handler:
  1. Ejecutar transacción en MySQL (Account + Transfer)
  2. INSERT en tabla `outbox_messages` (mismo transaction scope)
  3. COMMIT → garantía ACID

Background Service (Outbox Publisher):
  4. SELECT mensajes no publicados de `outbox_messages`
  5. Publicar a Kafka
  6. UPDATE `outbox_messages` SET published_at = NOW()
```

**Tabla MySQL `outbox_messages`**:
```sql
CREATE TABLE IF NOT EXISTS outbox_messages (
    id           CHAR(36)     NOT NULL PRIMARY KEY,
    event_type   VARCHAR(100) NOT NULL,
    topic        VARCHAR(200) NOT NULL,
    payload      JSON         NOT NULL,
    created_at   DATETIME(3)  NOT NULL,
    published_at DATETIME(3)  NULL,
    INDEX IX_outbox_unpublished (published_at, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## Regla 5 — Librería recomendada: MassTransit + Kafka

**NuGet packages** (para cuando se implemente — no para la sesión actual):
```xml
<!-- core-transactions-service (Producer) -->
<PackageReference Include="MassTransit" Version="8.3.6" />
<PackageReference Include="MassTransit.Kafka" Version="8.3.6" />

<!-- account-queries-service (Consumer) -->
<PackageReference Include="MassTransit" Version="8.3.6" />
<PackageReference Include="MassTransit.Kafka" Version="8.3.6" />
```

**Por qué MassTransit sobre Confluent.Kafka directo**:
- Abstrae el broker (permite cambiar a RabbitMQ sin tocar business logic)
- Retry policies integradas con Polly
- Dead Letter Queue automático
- Serialización JSON con System.Text.Json por defecto
- Integración natural con DI y el patrón hexagonal

**Puerto de dominio para publicación** (Application layer):
```csharp
// Application/Ports/Driven/IEventPublisher.cs
public interface IEventPublisher
{
    Task PublishAsync<TEvent>(TEvent @event, CancellationToken cancellationToken = default)
        where TEvent : DomainEventMessage;
}
```

**Adaptador de infraestructura** (Infrastructure layer):
```csharp
// Infrastructure/Kafka/KafkaEventPublisher.cs
public sealed class KafkaEventPublisher : IEventPublisher
{
    private readonly IPublishEndpoint _publishEndpoint;

    public KafkaEventPublisher(IPublishEndpoint publishEndpoint)
    {
        _publishEndpoint = publishEndpoint;
    }

    public async Task PublishAsync<TEvent>(TEvent @event, CancellationToken cancellationToken = default)
        where TEvent : DomainEventMessage
    {
        await _publishEndpoint.Publish(@event, cancellationToken).ConfigureAwait(false);
    }
}
```

---

## Regla 6 — Flujo completo de un Command con evento EDA

```
1. Frontend → API Gateway → core-transactions-service
2. ExecuteTransferCommand → MediatR → ExecuteTransferCommandHandler
3. CommandHandler:
   a. Validar idempotencia en Redis
   b. Ejecutar transferencia en MySQL
   c. INSERT en outbox_messages (misma transacción)
   d. COMMIT MySQL
4. OutboxPublisher (background):
   a. Leer outbox_messages no publicados
   b. IEventPublisher.PublishAsync(TransactionExecutedMessage)
   c. Marcar como publicado
5. Kafka topic: fintech.transactions.executed
6. account-queries-service (Consumer):
   a. Recibir TransactionExecutedMessage
   b. UPDATE MongoDB: balance_read_models
   c. DELETE Redis: balance:{accountId}
   d. INSERT MongoDB: transfer_history
```

---

## Regla 7 — Consumer rules (account-queries-service)

```csharp
// El consumer NUNCA escribe en MySQL
// El consumer SOLO actualiza MongoDB y Redis
// El consumer es idempotente: verificar si el evento ya fue procesado

public sealed class TransactionExecutedConsumer : IConsumer<TransactionExecutedMessage>
{
    private readonly IBalanceReadModel _readModel;
    private readonly ICacheService _cache;

    public async Task Consume(ConsumeContext<TransactionExecutedMessage> context)
    {
        var msg = context.Message;

        // Actualizar MongoDB (read model)
        await _readModel.UpsertAsync(new BalanceDocument
        {
            AccountId     = msg.SourceAccountId,
            Balance       = msg.NewSourceBalance,
            LastUpdatedAt = msg.OccurredOn
        }, context.CancellationToken).ConfigureAwait(false);

        // Invalidar Redis
        await _cache.DeleteAsync($"balance:{msg.SourceAccountId}", context.CancellationToken).ConfigureAwait(false);
    }
}
```

---

## Regla 8 — Validación de Kafka

### Comandos de validación

```bash
# 1. Verificar Kafka está levantado
docker exec kafka-fintech kafka-broker-api-versions.sh --bootstrap-server localhost:9092

# 2. Listar topics
docker exec kafka-fintech kafka-topics.sh --list --bootstrap-server localhost:9092

# 3. Describir un topic específico
docker exec kafka-fintech kafka-topics.sh \
  --describe --topic fintech.transactions.executed \
  --bootstrap-server localhost:9092

# 4. Leer mensajes de un topic (debugging)
docker exec kafka-fintech kafka-console-consumer.sh \
  --topic fintech.transactions.executed \
  --bootstrap-server localhost:9092 \
  --from-beginning \
  --max-messages 10

# 5. Publicar mensaje de prueba
echo '{"eventType":"TEST","occuredOn":"2026-06-25"}' | \
  docker exec -i kafka-fintech kafka-console-producer.sh \
  --topic fintech.transactions.executed \
  --bootstrap-server localhost:9092

# 6. Ver consumer groups
docker exec kafka-fintech kafka-consumer-groups.sh \
  --list --bootstrap-server localhost:9092

# 7. Kafka UI (navegador)
# http://localhost:8090
```

---

## Regla 9 — Prohibiciones EDA

- **Nunca llamar directamente a la API de otro microservicio** para sincronizar datos entre el write side y el read side. Toda sincronización es por eventos Kafka.
- **Nunca publicar eventos desde el Domain layer** — la publicación es responsabilidad de Infrastructure a través del puerto `IEventPublisher`.
- **Nunca procesar un evento dos veces** — el consumer verifica idempotencia consultando MongoDB antes de aplicar el cambio.
- **Nunca usar Kafka para comandos sincrónicos** — Kafka es para eventos asíncronos. Las operaciones que necesitan respuesta inmediata usan HTTP a través del API Gateway.
