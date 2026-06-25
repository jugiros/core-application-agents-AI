# Sistema Principal de Agente Arquitectónico — Fintech Microservicio

## Contexto global del proyecto

Este espacio de trabajo está configurado para un **microservicio financiero** de backend construido sobre el siguiente stack tecnológico:

- **Backend**: .NET Core 9 (C#) — microservicio REST API
- **Arquitectura**: Hexagonal (Puertos y Adaptadores) + Domain-Driven Design (DDD)
- **Patrón CQRS**: MediatR para separación de Commands y Queries
- **Caché distribuida**: Redis (Cache-Aside para saldos, idempotencia de transferencias)
- **Seguridad**: OAuth 2.0 + JWT — validación en capa de Infraestructura (Adaptadores Inbound)
- **Motor de automatización**: Claude Code CLI / Windsurf Workspace
- **Dominio de negocio**: Transferencias bancarias y visualización de saldos (BIAN v12)

## Principios rectores del proyecto

Todo agente activo en este espacio debe aplicar en cada cambio:

- **SOLID**: SRP en cada clase, OCP mediante puertos abstractos, DIP a través de interfaces en Application.
- **DRY**: No duplicar lógica entre Command Handlers y Query Handlers. Usar Pipeline Behaviors para preocupaciones transversales.
- **KISS**: Preferir soluciones simples y directas. Un handler = una responsabilidad.
- **YAGNI**: No implementar funcionalidad no requerida explícitamente. No crear capas de abstracción anticipadas.
- **Boy Scout Rule**: Dejar cada archivo mejor de como se encontró. Si se detecta una violación a estas reglas en archivos tocados, corregirla en el mismo commit.

## Capas de la arquitectura hexagonal

```
[Presentation / Adaptadores Inbound]
  └── Controllers (ASP.NET Core) — validan OAuth 2.0, extraen claims, delegan a MediatR
[Application / Puertos]
  ├── Commands  (escritura, modifican estado)
  ├── Queries   (lectura, consultan estado, usan caché Redis)
  ├── Behaviors (Logging, Validation, Idempotency)
  └── Ports/Driven (interfaces de repositorios, caché, mensajería)
[Domain]
  ├── Entities / AggregateRoots
  ├── ValueObjects
  ├── Domain Events
  └── Domain Exceptions — sin dependencias de infraestructura
[Infrastructure / Adaptadores Outbound]
  ├── Persistence (EF Core 9 + SQL Server)
  ├── Redis (StackExchange.Redis)
  └── DependencyInjection (registro de servicios)
```

## Validación de compilación

Antes de considerar completa cualquier tarea, el agente debe verificar:

- `dotnet build` contra la solución sin errores ni advertencias (`TreatWarningsAsErrors = true`).
- `dotnet test` para pruebas unitarias relevantes.
- Revisión de que el Dominio no importa librerías de infraestructura o seguridad.

## Recursos compartidos

- `docs/lessons-learned/`: Lecciones por tecnología. **Leer antes de iniciar cualquier implementación**.
- `.claude/contexts/fintech-domain.md`: Lenguaje ubicuo del dominio financiero — leer antes de nombrar clases o métodos.
- Memoria histórica del agente: Contexto acumulado de decisiones arquitectónicas y restricciones de negocio.

## Directrices de orquestación

1. Todo cambio que afecte múltiples capas debe ser coordinado por el agente `fullstack-engineer`.
2. Todo endpoint nuevo debe ser revisado por el `senior-security-architect` antes de considerarse completo.
3. Todo Command o Query nuevo debe ser diseñado por el `senior-cqrs-specialist`.
4. Los agentes especializados operan en ciclos: **Analizar → Implementar → Validar → Reintentar (máx. 3 veces)**.
5. Regla de oro: preferir correcciones mínimas en el origen del problema antes de agregar workarounds en capas superiores.

## Regla de seguridad inmutable

> **El Dominio nunca debe importar librerías de seguridad (`Microsoft.AspNetCore.Authorization`, `IdentityModel`, `System.Security.Claims`, etc.). La validación de tokens OAuth 2.0 ocurre exclusivamente en los Adaptadores Inbound (Controllers/Middleware).**

## Referencias de agentes y reglas

- Perfiles de agentes: `.claude/agents/`
- Reglas técnicas: `.claude/rules/`
- Contextos de dominio: `.claude/contexts/`
- Comandos slash: `.claude/commands/`
