# Contexto del Proyecto — Estado Actual
<!-- 
  INSTRUCCIÓN PARA EL AGENTE:
  Este archivo es la fuente de verdad del estado del proyecto en cada sesión.
  ACTUALIZAR este archivo al FINAL de cada sesión de trabajo con los cambios realizados.
  El prompt-maestro.md lo carga al inicio de cada nueva sesión.
-->

## Metadatos

| Campo | Valor |
|---|---|
| **Proyecto** | Fintech Platform — Transferencias, Saldos e Identidad (3 Microservicios) |
| **Repositorio de agentes** | `core-application-agents-AI` |
| **Microservicio 1** | `auth-service` — OAuth 2.0, MySQL (⏳ Por crear) |
| **Microservicio 2** | `core-transactions-service` (`PruebaNetCoreProject`) — Write CQRS, MySQL + Kafka (✅ Scaffolding) |
| **Microservicio 3** | `account-queries-service` — Read CQRS, MongoDB + Redis + Kafka (⏳ Por crear) |
| **API Gateway** | `api-gateway` — YARP .NET Core 9 (⏳ Por crear) |
| **Repositorio frontend** | `fintech-frontend` (rama: `feature/first`) — Next.js 15 SSR (⏳ Por crear) |
| **Infraestructura** | Docker Compose: MySQL 8, MongoDB 7, Redis 7, apache/kafka:latest (KRaft) + Kafka UI — ✅ corriendo |
| **Mensajería EDA** | Apache Kafka + MassTransit 8.x — Outbox Pattern |
| **Calidad de código** | SonarLint IDE + `SonarAnalyzer.CSharp` (backend) + `eslint-plugin-sonarjs` (frontend) |
| **Stack backend** | .NET Core 9, Hexagonal + DDD, CQRS (MediatR), MySQL 8 + MongoDB 7 + Redis 7, OAuth 2.0 + JWT |
| **Stack frontend** | Next.js 15 (SSR), React 19, TypeScript, Tailwind CSS 4, shadcn/ui, NextAuth.js v5 |
| **Fase actual** | `FASE 1 — Infraestructura Docker levantada y validada` |
| **Última sesión** | 2026-06-25 (sesión 2) |
| **Próxima tarea prioritaria** | Entidades DDD: Account + Transfer (AggregateRoot) + Value Objects (Balance, AccountNumber, Currency, TransferAmount) en `core-transactions-service` |
| **Prompt backend** | `.claude/prompt-maestro.md` + `.claude/contexts/project-context.md` |
| **Prompt frontend** | `.claude/prompt-maestro-frontend.md` + `.claude/contexts/project-context.md` |

---

## Estado de implementación

### Backend — `PruebaNetCoreProject`

#### ✅ Completado

| Componente | Ubicación | Notas |
|---|---|---|
| Estructura de directorios hexagonal | `PruebaNetCoreProject/src/` | Domain, Application, Infrastructure, Presentation |
| Proyectos .csproj configurados | `src/**/*.csproj`, `tests/**/*.csproj` | Net9.0, TreatWarningsAsErrors |
| Entidades base (DDD) | `Domain/Common/Entity.cs`, `AggregateRoot.cs`, `ValueObject.cs` | Sin lógica de negocio aún |
| Interfaces base | `Domain/Common/IRepository.cs`, `IUnitOfWork.cs`, `IDomainEvent.cs` | Vacías — base establecida |
| CQRS interfaces | `Application/Common/ICommandHandler.cs`, `IQueryHandler.cs` | Base establecida |
| Result pattern | `Application/Common/Result.cs` | Genérico — listo para uso |
| ApplicationDbContext | `Infrastructure/Persistence/Contexts/ApplicationDbContext.cs` | EF Core 9 — sin DbSets de negocio |
| GenericRepository | `Infrastructure/Persistence/Repositories/GenericRepository.cs` | Base — sin implementaciones específicas |
| ServiceRegistration | `Infrastructure/DependencyInjection/ServiceRegistration.cs` | ✅ Migrado: Pomelo MySQL 9 + MongoDB 7 + Redis 7 + MediatR + FluentValidation |
| ICacheService (puerto) | `Application/Ports/Driven/ICacheService.cs` | ✅ Creado |
| IAccountWriteRepository (puerto) | `Application/Ports/Driven/IAccountWriteRepository.cs` | ✅ Creado (placeholder) |
| IBalanceReadModel (puerto) | `Application/Ports/Driven/IBalanceReadModel.cs` | ✅ Creado |
| MongoDbContext | `Infrastructure/Persistence/MongoDB/Contexts/MongoDbContext.cs` | ✅ Creado |
| MongoDbSettings | `Infrastructure/Persistence/MongoDB/Settings/MongoDbSettings.cs` | ✅ Creado |
| RedisCacheService | `Infrastructure/Persistence/Redis/Services/RedisCacheService.cs` | ✅ Implementado |
| Program.cs | `Presentation/API/Program.cs` | ✅ JWT Bearer + Swagger + UseAuthentication |
| appsettings.json | `Presentation/API/appsettings.json` | ✅ MySQL + MongoDB + Redis + JWT keys |
| appsettings.Development.json | `Presentation/API/appsettings.Development.json` | ✅ Strings Docker locales |
| Program.cs | `Presentation/API/Program.cs` | Minimal API + controllers configurados |
| ApiControllerBase | `Presentation/API/Controllers/ApiControllerBase.cs` | Base para controllers de negocio |
| Solution file | `PruebaNetCoreProject.sln` | Todos los proyectos incluidos |
| Tests placeholder | `tests/Domain.Tests/`, `Application.Tests/`, `Infrastructure.Tests/` | UnitTest1.cs — sin tests reales |

#### 🔄 En progreso

| Componente | Estado | Responsable |
|---|---|---|
| Entidades de negocio (Account, Transfer) | **Pendiente** | `senior-backend-engineer` + `senior-cqrs-specialist` |
| Value Objects (Balance, AccountNumber, Currency, TransferAmount) | **Pendiente** | `senior-backend-engineer` |
| Commands (ExecuteTransferCommand) | **Pendiente** | `senior-cqrs-specialist` |
| Queries (GetBalanceQuery, GetTransferHistoryQuery) | **Pendiente** | `senior-cqrs-specialist` |
| MediatR + Pipeline Behaviors | **Pendiente** | `senior-cqrs-specialist` |
| ICacheService (puerto) + RedisCacheService (adaptador) | **Pendiente** | `senior-cqrs-specialist` |
| GlobalExceptionHandler | **Pendiente** | `senior-security-architect` |
| OAuth 2.0 JWT middleware | **Pendiente** | `senior-security-architect` |
| Controllers (AccountController, TransferController) | **Pendiente** | `senior-backend-engineer` + `senior-security-architect` |
| FluentValidation validators | **Pendiente** | `senior-cqrs-specialist` |
| EF Core migrations | **Pendiente** | `senior-dba` |

#### ❌ Pendiente de inicio

**Infraestructura Docker (✅ COMPLETADO — sesión 2026-06-25):**

| Componente | Estado | Notas |
|---|---|---|
| `docker/docker-compose.infra.yml` | ✅ Creado | apache/kafka:latest (KRaft), mysql:8.0, mongo:7.0, redis:7-alpine, kafka-ui |
| `Dockerfile` multi-stage .NET 9 | ✅ Creado | SDK 9.0 build → aspnet:9.0 runtime, non-root user |
| `.dockerignore` | ✅ Creado | bin/, obj/, .git/, .vs/ excluidos |
| MySQL `mysql-fintech` | ✅ Running (healthy) | puerto 3306, `mysqld is alive` |
| MongoDB `mongodb-fintech` | ✅ Running (healthy) | puerto 27017, `{ ok: 1 }` |
| Redis `redis-fintech` | ✅ Running (healthy) | puerto 6379, `PONG` |
| Kafka `kafka-fintech` | ✅ Running (healthy) | puerto 9092, KRaft sin Zookeeper |
| Kafka UI `kafka-ui-fintech` | ✅ Running | http://localhost:8090 |

**Nota imagen Kafka:** `bitnami/kafka` ya no es gratuito en Docker Hub (ahora es comercial). Se usa `apache/kafka:latest` (imagen oficial Apache) con variables KRaft nativas (`KAFKA_NODE_ID`, `KAFKA_PROCESS_ROLES`, etc.). Actualizar regla en `docker-infrastructure.md`.

**SonarLint (✅ COMPLETADO — sesión 2026-06-25):**

| Componente | Estado | Acción |
|---|---|---|
| Instalar extensión SonarLint en IDE | ⚠️ Pendiente usuario | VS Code / Rider / Visual Studio |
| `.editorconfig` en backend | ✅ Creado | `PruebaNetCoreProject/.editorconfig` — reglas S* + CA* |
| `eslint-plugin-sonarjs` al frontend | ⏳ Cuando se cree frontend | `npm install -D eslint-plugin-sonarjs` |

**Microservicios por crear:**

| Componente | Prioridad | Descripción |
|---|---|---|
| `auth-service` (nuevo repo) | 🔴 Alta | OAuth 2.0 server, MySQL, Hexagonal |
| `account-queries-service` (nuevo repo) | 🔴 Alta | Read CQRS, MongoDB + Redis + Kafka consumer |
| `api-gateway` (nuevo repo) | 🔴 Alta | YARP, routing a 3 servicios |

**`core-transactions-service` (`PruebaNetCoreProject`) — Siguiente capa de negocio:**

| Componente | Prioridad |
|---|---|
| Entidades DDD: Account, Transfer (AggregateRoot) | 🔴 Alta |
| Value Objects: Balance, AccountNumber, Currency, TransferAmount | 🔴 Alta |
| EF Core migrations (MySQL) con Pomelo | 🔴 Alta |
| Tabla `outbox_messages` en MySQL (Outbox Pattern) | 🔴 Alta |
| Commands: ExecuteTransferCommand + handler | 🔴 Alta |
| Queries: GetBalanceQuery, GetTransferHistoryQuery | 🔴 Alta |
| MediatR Pipeline Behaviors (Logging, Validation, Idempotency) | 🔴 Alta |
| IEventPublisher puerto + KafkaEventPublisher adaptador (MassTransit) | Media |
| OutboxPublisher background service | Media |
| GlobalExceptionHandler | 🔴 Alta |
| OAuth 2.0 JWT middleware | 🔴 Alta |
| Controllers (AccountController, TransferController) | 🔴 Alta |
| Health Checks endpoint | Media |
| Circuit Breaker (Polly v8) | Media |
| Correlation ID middleware | Media |
| Tests unitarios reales (Domain, Application) | 🔴 Alta |

**Frontend (`fintech-frontend`):**

| Componente | Prioridad |
|---|---|
| Crear repo con `npx create-next-app@latest` | 🔴 Alta |
| Instalar shadcn/ui, Lucide, NextAuth.js v5, Zustand, Zod | 🔴 Alta |
| Configurar `eslint-plugin-sonarjs` + `.eslintrc.json` | 🔴 Alta |
| Crear `Dockerfile` multi-stage Next.js 15 | 🔴 Alta |
| Middleware de autenticación OAuth PKCE | 🔴 Alta |
| Página de saldo (Server Component SSR) | Media |
| Formulario de transferencia (Client Component) | Media |
| Indicador de consistencia eventual (Kafka lag) | Baja |

---

### Configuración de Agentes — `core-application-agents-AI`

#### ✅ Completado (acumulado de sesiones anteriores)

| Archivo | Acción |
|---|---|
| `.claude/CLAUDE.md` | Actualizado — 3 microservicios, Docker, Kafka/EDA, API Gateway, SonarLint |
| `.claude/settings.json` | Actualizado — stack completo + MCP servers |
| `.claude/prompt-maestro.md` | Actualizado — BACKEND ONLY, validación Docker, 3 servicios, SonarLint |
| `.claude/prompt-maestro-frontend.md` | Creado — FRONTEND ONLY, validación Docker, SonarLint, API Gateway context |
| `.claude/agents/senior-security-architect.md` | Creado — OAuth 2.0, JWT, ISO 27001, BIAN |
| `.claude/agents/senior-cqrs-specialist.md` | Actualizado — CQRS, MediatR, Redis + Kafka EDA events |
| `.claude/agents/senior-dba.md` | Reescrito — MySQL + MongoDB + Redis + Docker + MCP |
| `.claude/agents/senior-frontend-engineer.md` | Reescrito — Next.js 15 SSR + TypeScript + BFF + SonarLint |
| `.claude/agents/fullstack-engineer.md` | Actualizado — 3 microservicios, Kafka, Docker, SonarLint |
| `.claude/rules/backend-net9-hexagonal-cqrs.md` | Creado — 8 reglas |
| `.claude/rules/security-oauth2-rules.md` | Creado — 7 reglas |
| `.claude/rules/redis-cache-rules.md` | Creado — 7 reglas |
| `.claude/rules/databases-multi-provider.md` | Creado — 8 reglas (MySQL/MongoDB/Redis) |
| `.claude/rules/docker-infrastructure.md` | Creado — 8 reglas + docker-compose + Dockerfiles + validación |
| `.claude/rules/kafka-eda-rules.md` | Creado — 9 reglas EDA + topics + Outbox + MassTransit |
| `.claude/rules/sonar-code-quality.md` | Creado — SonarLint C# y TypeScript + comandos de análisis |
| `.claude/rules/design-principles-validation.md` | Creado — SOLID + DRY + KISS + YAGNI obligatorio |
| `.claude/contexts/fintech-domain.md` | Creado — lenguaje ubicuo DDD |
| `docs/lessons-learned/backend-net9.md` | Creado — 8 lecciones .NET 9 + CQRS |
| `docs/lessons-learned/security-oauth2.md` | Creado — 8 lecciones OAuth 2.0 |
| `PruebaNetCoreProject/Persistence.csproj` | Actualizado — Pomelo 9 + MongoDB.Driver 3 + StackExchange.Redis + EF Design |
| `PruebaNetCoreProject/Application.csproj` | Actualizado — MediatR 12 + FluentValidation 11 |
| `PruebaNetCoreProject/API.csproj` | Actualizado — JWT Bearer 9 + Swashbuckle 7 |
| `PruebaNetCoreProject/ServiceRegistration.cs` | Reescrito — MySQL + MongoDB + Redis + MediatR (4 métodos privados) |
| `PruebaNetCoreProject/Program.cs` | Actualizado — JWT + Swagger + UseAuthentication |
| `PruebaNetCoreProject/appsettings*.json` | Actualizado — MySQL + MongoDB + Redis + JWT (sin SQL Server) |
| `PruebaNetCoreProject/ICacheService.cs` | Creado — puerto Application |
| `PruebaNetCoreProject/IBalanceReadModel.cs` | Creado — puerto Application |
| `PruebaNetCoreProject/MongoDbContext.cs` | Creado — adaptador Infrastructure |
| `PruebaNetCoreProject/RedisCacheService.cs` | Creado — adaptador Infrastructure |
| `PruebaNetCoreProject/docker/docker-compose.infra.yml` | ✅ Creado (sesión 2) — apache/kafka:latest, mysql:8.0, mongo:7.0, redis:7-alpine, kafka-ui |
| `PruebaNetCoreProject/Dockerfile` | ✅ Creado (sesión 2) — multi-stage SDK 9.0 → aspnet:9.0, non-root user |
| `PruebaNetCoreProject/.dockerignore` | ✅ Creado (sesión 2) |
| `PruebaNetCoreProject/.editorconfig` | ✅ Creado (sesión 2) — SonarLint C# rules (S*, CA*) |

---

## Decisiones arquitectónicas activas

| Decisión | Razón | Fecha |
|---|---|---|
| **3 Microservicios** (Auth + CoreTransactions + AccountQueries) | Separación de Bounded Contexts — DDD; escalado independiente | 2026-06 |
| **API Gateway YARP** | Frontend solo conoce un punto de entrada; routing transparente | 2026-06 |
| **Kafka como message broker EDA** | Desacoplamiento temporal entre write side y read side; at-least-once delivery | 2026-06 |
| **MassTransit 8.x sobre Confluent.Kafka directo** | Abstrae el broker, retry integrado, Dead Letter Queue, DI nativo | 2026-06 |
| **Outbox Pattern** (MySQL + background service) | Garantiza at-least-once delivery sin 2PC — ACID + eventual | 2026-06 |
| **Docker Compose para toda la infraestructura** | Reproducibilidad del entorno; elimina dependencia de XAMPP | 2026-06 |
| **apache/kafka:latest con KRaft** (sin Zookeeper) | `bitnami/kafka` ya no es gratuito en Docker Hub (comercial). `apache/kafka` es la imagen oficial Apache — variables KRaft nativas | 2026-06-25 |
| **SonarLint obligatorio** (IDE + CI) | Calidad de código; bloquear issues Critical/Blocker antes de merge | 2026-06 |
| CQRS con MediatR (no EventSourcing) | Simplicidad y madurez del ecosistema .NET 9 | 2026-06 |
| Commands → MySQL / Queries → MongoDB + Redis | ACID para escritura, velocidad para lectura | 2026-06 |
| MySQL 8 (Docker, reemplaza XAMPP) | Reproducibilidad; mismo contenedor en dev/staging/prod | 2026-06 |
| MongoDB para read models y audit log | Schema flexible, append-only para auditoría, paginación eficiente | 2026-06 |
| Cache-Aside (no Read-Through) | Control explícito de TTL y degradación graceful | 2026-06 |
| Redis TTL mínimo 30s para saldos | Datos volátiles — transferencias frecuentes | 2026-06 |
| Result<T> en lugar de excepciones para flujos esperados | DRY y control de flujo predecible | 2026-06 |
| ClockSkew = TimeSpan.Zero | Ventana de tolerancia inaceptable en Fintech | 2026-06 |
| Dominio no importa librerías de seguridad | Aislamiento hexagonal — regla inmutable | 2026-06 |
| Frontend SSR (Next.js 15) — NO SPA | Saldos y transferencias renderizados en servidor; token OAuth nunca en browser | 2026-06 |
| BFF pattern (Route Handlers Next.js) | El access token vive en el servidor Next.js, no en el cliente | 2026-06 |
| Repos separados backend/frontend | Ciclos de deploy independientes, equipos desacoplados | 2026-06 |

---

## Restricciones activas

- **Docker** es el entorno de referencia para toda la infraestructura. XAMPP ya no es el entorno primario — MySQL corre en contenedor `mysql-fintech`.
- **Docker Desktop** debe estar instalado y corriendo antes de iniciar cualquier sesión.
- El **Identity Provider** (Keycloak/Duende) no está seleccionado aún. Usar configuración por variables de entorno.
- **No hay lógica de negocio** implementada en `PruebaNetCoreProject`. Solo scaffolding + infraestructura base.
- Los repositorios `auth-service`, `account-queries-service`, `api-gateway` y `fintech-frontend` **no existen aún**.
- **SonarLint** debe estar instalado en el IDE antes de comenzar a generar código en cualquier repo.
- Los **MCP servers** apuntan a los contenedores Docker (no a XAMPP). Deben instalarse: `npm install -g @benborla29/mcp-server-mysql @mongodb-js/mongodb-mcp-server redis-mcp` y configurarse en **Windsurf Settings → MCP**.

---

## Cómo actualizar este archivo

Al finalizar cada sesión, el agente orquestador (`fullstack-engineer`) debe actualizar:
1. La fecha en `Última sesión`
2. Los ítems en `✅ Completado` con los componentes terminados en esa sesión
3. Mover ítems de `🔄 En progreso` a `✅ Completado` cuando corresponda
4. Actualizar `Próxima tarea prioritaria`
5. Agregar nuevas `Decisiones arquitectónicas` si se tomaron durante la sesión
