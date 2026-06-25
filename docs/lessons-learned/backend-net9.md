# Lecciones Aprendidas — Backend .NET Core 9, Hexagonal y CQRS

## Propósito

Registro histórico de decisiones arquitectónicas, errores recurrentes y soluciones aplicadas para el stack .NET Core 9 con arquitectura Hexagonal y CQRS. Todo agente debe leer este archivo antes de iniciar una implementación de backend.

---

## LL-001 — El Dominio no referencia librerías de infraestructura

**Fecha**: 2026-06  
**Agente**: senior-backend-engineer  
**Categoría**: Arquitectura Hexagonal — SRP + DIP

**Situación**: Al agregar EF Core al proyecto Domain por conveniencia, el Dominio adquirió dependencia directa de `Microsoft.EntityFrameworkCore`, rompiendo el aislamiento hexagonal.

**Decisión**: El proyecto `Domain.csproj` no tiene ninguna referencia a paquetes de infraestructura. Las interfaces de repositorio (puertos) se definen en `Application`. Las implementaciones concretas van en `Infrastructure/Persistence`.

**Verificación**: `Domain.csproj` no debe contener `<PackageReference Include="Microsoft.EntityFrameworkCore..." />`.

---

## LL-002 — Commands retornan solo Id, nunca DTOs completos

**Fecha**: 2026-06  
**Agente**: senior-cqrs-specialist  
**Categoría**: CQRS — Separación Commands/Queries

**Situación**: Se implementó un `CreateTransferCommand` que retornaba `TransferDto` completo para evitar un segundo viaje al servidor. Esto viola CQRS: los Commands modifican estado y los Queries leen estado.

**Decisión**: Los Commands retornan `Result<Guid>` (el Id del recurso creado) o `Result<Unit>`. El cliente usa ese Id para hacer un Query posterior si necesita datos completos.

**Justificación**: KISS y separación de responsabilidades. La UI puede hacer el Query de enriquecimiento cuando lo necesite.

---

## LL-003 — TTL obligatorio en Redis: nunca omitirlo

**Fecha**: 2026-06  
**Agente**: senior-cqrs-specialist  
**Categoría**: Redis — Consistencia y mantenibilidad

**Situación**: Una clave de caché de saldo fue creada sin TTL durante una prueba de integración. En producción, el saldo quedó obsoleto indefinidamente hasta que se reinició el servicio Redis.

**Decisión**: La interfaz `ICacheService.SetAsync` requiere `TimeSpan ttl` como parámetro obligatorio (sin sobrecarga sin TTL). Ningún `SetAsync` puede compilar sin TTL.

**Verificación**: Revisar que `ICacheService` no tenga sobrecargas sin TTL.

---

## LL-004 — ConfigureAwait(false) en Application e Infrastructure, no en Controllers

**Fecha**: 2026-06  
**Agente**: senior-backend-engineer  
**Categoría**: Asincronía .NET Core 9

**Situación**: Se aplicó `ConfigureAwait(false)` de forma uniforme en todos los archivos, incluyendo Controllers. En ASP.NET Core 9 esto es innecesario en la capa de Presentación pues no hay SynchronizationContext problemático.

**Decisión**: Usar `ConfigureAwait(false)` en `Application` e `Infrastructure`. En `Controllers` (Presentation) no es necesario pero tampoco incorrecto. Priorizar consistencia por capa.

---

## LL-005 — La invalidación de caché ocurre post-commit, no pre-commit

**Fecha**: 2026-06  
**Agente**: senior-cqrs-specialist  
**Categoría**: Redis — Consistencia transaccional

**Situación**: Se intentó invalidar la clave de caché del saldo antes de llamar a `SaveChangesAsync()`. Si la transacción fallaba, el caché quedaba invalidado pero la BD no se había actualizado, creando una inconsistencia temporal mayor al TTL.

**Decisión**: La secuencia siempre es: (1) `SaveChangesAsync` → (2) Invalidar Redis. La invalidación es best-effort: si falla Redis, el TTL garantiza consistencia eventual.

---

## LL-006 — Usar records inmutables para Commands y Queries

**Fecha**: 2026-06  
**Agente**: senior-cqrs-specialist  
**Categoría**: CQRS — Inmutabilidad de mensajes

**Situación**: Se implementaron Commands como clases con setters públicos. Un bug permitió que un Pipeline Behavior modificara el Command en tránsito, causando valores incorrectos en el handler.

**Decisión**: Todo Command y Query es un `record` de C#. Los records son inmutables por diseño y previenen mutación accidental en el pipeline de MediatR.

```csharp
// CORRECTO
public record ExecuteTransferCommand(Guid SourceAccountId, Guid TargetAccountId, decimal Amount, string IdempotencyKey) : IRequest<Result<Guid>>;
```

---

## LL-007 — ValidationBehavior debe registrarse antes del handler en el pipeline

**Fecha**: 2026-06  
**Agente**: senior-cqrs-specialist  
**Categoría**: MediatR — Pipeline Behaviors

**Situación**: Al registrar los behaviors en orden incorrecto, la validación ocurría después del handler, permitiendo que datos inválidos llegaran al Dominio.

**Decisión**: El orden de registro en `ServiceRegistration.cs` determina el orden de ejecución:
1. `LoggingBehavior<,>`
2. `ValidationBehavior<,>`
3. `IdempotencyBehavior<,>`
4. Handler (implícito, siempre último)

```csharp
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(IdempotencyBehavior<,>));
```

---

## LL-008 — IUnitOfWork no debe exponerse en Queries

**Fecha**: 2026-06  
**Agente**: senior-backend-engineer  
**Categoría**: CQRS — Separación de responsabilidades

**Situación**: Un Query Handler inyectó `IUnitOfWork` por conveniencia, aunque solo lo usaba para acceder al `DbContext`. Esto crea riesgo de llamadas accidentales a `SaveChangesAsync` en una Query.

**Decisión**: Los Query Handlers inyectan directamente el repositorio de lectura (`IAccountRepository`) o un `IReadOnlyRepository<T>` especializado. `IUnitOfWork` solo se inyecta en Command Handlers.

---

## Cómo agregar una nueva lección

Usar el formato:

```
## LL-{numero} — {título corto}

**Fecha**: YYYY-MM  
**Agente**: {nombre-del-agente}  
**Categoría**: {categoría}

**Situación**: Descripción del problema encontrado.

**Decisión**: Solución aplicada y razón.

**Verificación** (opcional): Cómo verificar que no se repita.
```
