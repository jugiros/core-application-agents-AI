# Sistema Principal de Agente Arquitectónico — Fintech Platform

## Mapa de repositorios (SEPARADOS — no mezclar en una misma sesión)

| Repositorio | Rol | Stack | Prompt maestro | Estado |
|---|---|---|---|---|
| `auth-service` | Microservicio 1 — Identidad OAuth 2.0 | .NET Core 9 + MySQL | `prompt-maestro.md` | ⏳ Por crear |
| `core-transactions-service` (`PruebaNetCoreProject`) | Microservicio 2 — Write side CQRS | .NET Core 9 + MySQL + Kafka | `prompt-maestro.md` | ✅ Scaffolding |
| `account-queries-service` | Microservicio 3 — Read side CQRS | .NET Core 9 + MongoDB + Redis + Kafka | `prompt-maestro.md` | ⏳ Por crear |
| `api-gateway` | Gateway YARP | .NET Core 9 | `prompt-maestro.md` | ⏳ Por crear |
| `fintech-frontend` | Portal web SSR | Next.js 15 + TypeScript | `prompt-maestro-frontend.md` | ⏳ Por crear |

> **Regla absoluta**: El prompt de backend NO coordina tareas de frontend y viceversa. Cada sesión usa el prompt maestro correspondiente.

---

## Arquitectura completa del sistema

```
┌─────────────────────────────────────────────────────────┐
│  fintech-frontend  (Next.js 15 SSR — rama: feature/first)│
│  BFF: tokens OAuth solo en servidor                       │
└──────────────────────────┬──────────────────────────────┘
                           │ HTTPS
┌──────────────────────────▼──────────────────────────────┐
│              api-gateway  (YARP — .NET Core 9)           │
│   /auth/*  →  auth-service                               │
│   /transactions/*  →  core-transactions-service          │
│   /accounts/*  →  account-queries-service                │
└───────┬──────────────────┬──────────────────────────────┘
        │                  │
┌───────▼──────┐  ┌────────▼────────────────────────────┐
│ auth-service  │  │  core-transactions-service           │
│ MySQL         │  │  MySQL (write — ACID)                │
│ OAuth 2.0     │  │  MediatR Commands                    │
│ JWT emisión   │  │  Outbox → KAFKA PRODUCER ──────────►│
└───────────────┘  └─────────────────────────────────────┘
                                                           │
                              ┌────────────────────────────▼──────┐
                              │  account-queries-service            │
                              │  KAFKA CONSUMER                     │
                              │  MongoDB (read models — eventual)   │
                              │  Redis (caché L1 — 30 s TTL)       │
                              └───────────────────────────────────┘
```

---

## Infraestructura Docker (compartida por todos los servicios)

```
docker/docker-compose.infra.yml  ← Levantar con: docker compose -f docker/docker-compose.infra.yml up -d
```

| Contenedor | Imagen | Puerto | Validación |
|---|---|---|---|
| `mysql-fintech` | `mysql:8.0` | 3306 | `docker exec mysql-fintech mysqladmin ping -u root -pfintechroot2024` |
| `mongodb-fintech` | `mongo:7.0` | 27017 | `docker exec mongodb-fintech mongosh --eval "db.adminCommand('ping')"` |
| `redis-fintech` | `redis:7-alpine` | 6379 | `docker exec redis-fintech redis-cli -a fintech2024 ping` |
| `kafka-fintech` | `bitnami/kafka:3.7` | 9092 | `docker exec kafka-fintech kafka-topics.sh --list --bootstrap-server localhost:9092` |
| `kafka-ui-fintech` | `provectuslabs/kafka-ui` | 8090 | http://localhost:8090 |

> Reglas Docker completas: `.claude/rules/docker-infrastructure.md`

---

## Kafka — Event-Driven Architecture (EDA)

| Topic Kafka | Productor | Consumidor |
|---|---|---|
| `fintech.transactions.executed` | core-transactions-service | account-queries-service |
| `fintech.accounts.balance-updated` | core-transactions-service | account-queries-service |
| `fintech.auth.user-registered` | auth-service | core-transactions-service |

**Patrón**: Outbox Pattern (MySQL mismo transaction scope) → Kafka → MongoDB/Redis update.
**Librería**: MassTransit 8.x con transporte Kafka.
**Puerto Application**: `IEventPublisher` (adaptador en Infrastructure — nunca en Domain).

> Reglas EDA completas: `.claude/rules/kafka-eda-rules.md`

---

## Calidad de código — SonarLint obligatorio

| Repositorio | Herramienta | Comando de validación |
|---|---|---|
| Todos los servicios .NET | SonarLint IDE + `SonarAnalyzer.CSharp` | `dotnet build` sin warnings |
| `fintech-frontend` | SonarLint IDE + `eslint-plugin-sonarjs` | `npm run lint` sin errores |

> Reglas SonarLint completas: `.claude/rules/sonar-code-quality.md`

---

## MCP Servers disponibles (validación directa de datos)

| MCP Server | Base de datos | Uso por agente |
|---|---|---|
| `mysql-fintech` | MySQL Docker | `senior-dba`, `senior-cqrs-specialist` |
| `mongodb-fintech` | MongoDB Docker | `senior-dba`, `senior-cqrs-specialist` |
| `redis-fintech` | Redis Docker | `senior-cqrs-specialist` |

> Configurar también en **Windsurf Settings → MCP**.

---

## Principios rectores del proyecto

Todo agente activo en este espacio debe aplicar en **cada cambio**:

- **SOLID**: SRP en cada clase, OCP mediante puertos abstractos, DIP a través de interfaces en Application.
- **DRY**: No duplicar lógica entre handlers. Pipeline Behaviors para preocupaciones transversales.
- **KISS**: Preferir soluciones simples y directas.
- **YAGNI**: No implementar funcionalidad no requerida explícitamente.
- **Boy Scout Rule**: Dejar cada archivo mejor de como se encontró.

**Al entregar cada tarea**, el agente debe incluir el bloque de validación de principios (ver `design-principles-validation.md`): qué hizo, qué usó, qué principios aplicó.

---

## Capas hexagonales (aplica a cada microservicio backend)

```
[Presentation / Adaptadores Inbound]
  └── Controllers — validan OAuth 2.0, delegan a MediatR
      Middleware  — GlobalExceptionHandler, CorrelationId, JWT validation

[Application / Puertos y Casos de Uso]
  ├── Commands  → escriben en MySQL (core-transactions) o MySQL auth (auth-service)
  ├── Queries   → leen desde MongoDB + Redis (account-queries)
  ├── Behaviors → LoggingBehavior → ValidationBehavior → IdempotencyBehavior
  └── Ports/Driven → IAccountWriteRepository, IBalanceReadModel,
                     ICacheService, IEventPublisher, IUnitOfWork

[Domain]
  ├── Entities / AggregateRoots (Account, Transfer, User)
  ├── ValueObjects (Balance, AccountNumber, Currency, TransferAmount)
  ├── Domain Events (TransferCompleted, BalanceUpdated) — publicados vía IEventPublisher
  └── Domain Exceptions — CERO dependencias externas

[Infrastructure / Adaptadores Outbound]
  ├── MySQL (Pomelo EF Core 9) — write side + outbox_messages
  ├── MongoDB (MongoDB.Driver 3) — read models (account-queries únicamente)
  ├── Redis (StackExchange.Redis) — caché L1 + idempotencia
  ├── Kafka (MassTransit 8.x) — publicación y consumo de eventos EDA
  └── DependencyInjection — composición de todos los adaptadores
```

---

## Estrategia de datos por microservicio

| Microservicio | Base de datos | Justificación |
|---|---|---|
| auth-service | MySQL | ACID para credenciales y políticas de seguridad |
| core-transactions-service | MySQL (write) | ACID para transferencias — nunca consistencia eventual |
| account-queries-service | MongoDB + Redis | Lectura optimizada post-evento Kafka |
| Outbox (en core-transactions) | MySQL (tabla `outbox_messages`) | Garantía at-least-once delivery a Kafka |

---

## Validación de compilación y calidad (checklist mínimo por sesión)

```
Backend:
  [ ] docker compose -f docker/docker-compose.infra.yml ps  ← todos healthy
  [ ] dotnet build {solucion}.sln --verbosity minimal       ← 0 errores, 0 warnings
  [ ] dotnet test                                            ← 0 pruebas regresionadas
  [ ] SonarLint IDE                                          ← sin issues críticos

Frontend:
  [ ] npm run lint                                           ← 0 errores ESLint/SonarJS
  [ ] npx tsc --noEmit                                       ← 0 errores TypeScript
  [ ] npm run build                                          ← build producción exitoso
```

---

## Regla de seguridad inmutable

> **El Domain layer nunca importa librerías de seguridad ni infraestructura. La validación OAuth 2.0 ocurre exclusivamente en los Adaptadores Inbound (Controllers/Middleware). Los tokens nunca llegan al Domain ni al Application layer.**

---

## Referencias rápidas

| Tipo | Ubicación |
|---|---|
| Perfiles de agentes | `.claude/agents/` |
| Reglas técnicas | `.claude/rules/` |
| Contextos de dominio | `.claude/contexts/` |
| Lecciones aprendidas | `docs/lessons-learned/` |
