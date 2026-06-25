# CoreApplication

Proyecto .NET 9 con arquitectura hexagonal orientada a Domain-Driven Design (DDD).

## Estructura

- **src/Core/Domain**: Entidades, agregados, value objects, eventos de dominio y excepciones.
- **src/Core/Application**: Casos de uso, DTOs, puertos (interfaces de entrada/salida) y lógica de aplicación.
- **src/Infrastructure/Persistence**: DbContext, repositorios genéricos y migraciones Entity Framework Core.
- **src/Infrastructure/DependencyInjection**: Registro de servicios e inyección de dependencias.
- **src/Presentation/API**: Web API con controladores, middleware y configuración.
- **tests**: Proyectos de pruebas unitarias para Domain, Application e Infrastructure.

## Tecnologías

- .NET 9
- ASP.NET Core
- Entity Framework Core 9 (SQL Server)
- xUnit + Moq

## Cómo ejecutar

```bash
dotnet build CoreApplication.sln
dotnet run --project src/Presentation/API/API.csproj
```
