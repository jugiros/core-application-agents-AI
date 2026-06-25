# Regla de Validación de Principios de Diseño

## Alcance

Esta regla aplica a **toda** creación o modificación de clase, función, handler, repositorio, controller, behavior, service o integración dentro del microservicio financiero. Es transversal a todas las capas (Domain, Application, Infrastructure, Presentation).

---

## Obligación del agente

Cada vez que un agente genere o modifique código, **debe adjuntar en su respuesta** un bloque de validación de principios con el siguiente formato. Este bloque va después del código, no dentro de él.

---

## Formato de bloque de validación (obligatorio en toda respuesta con código)

```
### Validación de principios — {NombreDeClaseOFunción}

**Patrón de diseño utilizado:**
- Nombre: {Repository / Command Handler / Query Handler / Factory / Strategy / Observer / Decorator / Pipeline / etc.}
- Justificación: {Por qué se eligió este patrón en este contexto específico}

**SOLID aplicado:**
- [S] SRP — {¿Sí/No? — Explicación: esta clase tiene una única responsabilidad porque...}
- [O] OCP — {¿Aplica? — Explicación: está abierta a extensión por... / no aplica porque...}
- [L] LSP — {¿Aplica? — Explicación: la subclase puede reemplazar... / no aplica porque...}
- [I] ISP — {¿Aplica? — Explicación: la interfaz es específica para... / no aplica porque...}
- [D] DIP — {¿Sí/No? — Explicación: depende de la abstracción {INombreInterfaz}, no de la implementación concreta}

**DRY (Don't Repeat Yourself):**
- {¿Sí/No?} — {Explicación: la lógica de {X} está centralizada en {Y} / se detectó duplicación en {Z} que se corrigió porque...}

**KISS (Keep It Simple, Stupid):**
- {¿Sí/No?} — {Explicación: la implementación usa {N} líneas para resolver {problema} sin capas de abstracción innecesarias / se simplificó eliminando {X} porque...}

**YAGNI (You Aren't Gonna Need It):**
- {¿Sí/No?} — {Explicación: solo se implementó lo necesario para {caso de uso} / se eliminó {X} que no era requerido aún porque...}

**Boy Scout Rule:**
- {¿Se mejoró algo preexistente? — Sí: {qué se mejoró y en qué archivo} / No: no se encontraron violaciones en los archivos tocados}

**Recomendaciones de mejora:**
1. {Recomendación concreta — prioridad: Alta/Media/Baja}
2. {Recomendación concreta — prioridad: Alta/Media/Baja}
(Si no hay recomendaciones, indicar: "Sin recomendaciones pendientes para esta implementación.")
```

---

## Ejemplo de bloque de validación completo

Para el handler `GetBalanceQueryHandler`:

```
### Validación de principios — GetBalanceQueryHandler

**Patrón de diseño utilizado:**
- Nombre: Query Handler (CQRS) + Cache-Aside
- Justificación: La separación Command/Query garantiza que las consultas de saldo no generen efectos secundarios. Cache-Aside se eligió sobre Read-Through porque permite control explícito del TTL y degradación graceful ante fallos de Redis.

**SOLID aplicado:**
- [S] SRP — Sí. Esta clase tiene una única responsabilidad: obtener el saldo de una cuenta, primero desde Redis y luego desde BD si hay cache miss.
- [O] OCP — No aplica directamente. El handler es final (sealed). Se puede extender el comportamiento mediante Pipeline Behaviors sin modificar el handler.
- [L] LSP — No aplica. No hay jerarquía de handlers.
- [I] ISP — Sí. Depende de IAccountRepository (solo GetByIdAsync) y ICacheService (solo GetAsync/SetAsync), no de interfaces monolíticas.
- [D] DIP — Sí. Depende de IAccountRepository e ICacheService (abstracciones), no de ApplicationDbContext ni IDatabase de Redis directamente.

**DRY:**
- Sí. La lógica de construcción de la clave Redis está centralizada en la constante CacheKeyPrefix. No hay duplicación de lógica de caché entre handlers.

**KISS:**
- Sí. El flujo es lineal: verificar caché → si miss, consultar BD → actualizar caché → retornar. Sin ramas innecesarias ni capas de abstracción adicionales.

**YAGNI:**
- Sí. Solo se implementó la consulta de saldo. No se agregaron métodos de historial ni proyecciones porque no son requeridos por este caso de uso.

**Boy Scout Rule:**
- Sí. Se encontró que ICacheService en la versión anterior no tenía ExistsAsync. Se añadió el método al puerto durante esta implementación.

**Recomendaciones de mejora:**
1. Agregar métricas de cache hit/miss ratio mediante ILogger o un contador de Prometheus — prioridad: Media
2. Considerar IMemoryCache como segunda capa (L1 local) antes de Redis (L2 distribuido) si la latencia de Redis supera 5ms — prioridad: Baja
```

---

## Integración con agentes

Cada agente especializado aplica esta validación desde su perspectiva:

- **`senior-backend-engineer`**: Valida SRP, DIP, DRY en capas Domain y Application.
- **`senior-cqrs-specialist`**: Valida separación Commands/Queries, patrones CQRS y Cache-Aside.
- **`senior-security-architect`**: Valida que los patrones de seguridad no contaminen el Dominio. Valida ISP en puertos de seguridad.
- **`senior-dba`**: Valida que el patrón Repository sea consistente con los índices de BD definidos.
- **`fullstack-engineer`** (orquestador): Revisa que los bloques de validación de todos los agentes sean consistentes entre sí antes de dar la tarea por completada.

---

## Sanciones por omisión

Si un agente entrega código sin el bloque de validación de principios:

1. La tarea **no se considera completa**.
2. El `fullstack-engineer` debe solicitar el bloque faltante antes de continuar.
3. Se agrega una entrada en `docs/lessons-learned/general.md` documentando la omisión.
