# Lecciones Aprendidas

Este directorio almacena el conocimiento histórico del proyecto, organizado por tecnología y área de responsabilidad.

## Propósito

**Todo agente debe leer las lecciones correspondientes antes de iniciar cualquier implementación.** Las entradas documentan decisiones arquitectónicas, errores recurrentes y soluciones aplicadas para evitar repetirlos.

## Archivos activos (stack .NET Core 9 + Hexagonal + CQRS)

- `backend-net9.md` — Lecciones sobre .NET Core 9, CQRS, MediatR, Hexagonal Architecture, Result Pattern y asincronía.
- `security-oauth2.md` — Lecciones sobre OAuth 2.0, JWT, ISO 27001, scopes BIAN y auditoría financiera.
- `redis-cache.md` — Lecciones sobre Redis, Cache-Aside, idempotencia de transferencias y TTL. *(pendiente de creación)*
- `dba.md` — Lecciones sobre EF Core 9, esquemas de transferencias, índices y migraciones. *(pendiente de creación)*
- `general.md` — Lecciones transversales que no encajan en una categoría específica. *(pendiente de creación)*

## Archivos legacy (stack previo — solo referencia histórica)

- `backend-net47.md` — Lecciones sobre .NET Framework 4.7. **No aplicar al stack actual.**
- `frontend-xaml.md` — Lecciones sobre WPF/XAML/MVVM. **No aplicar hasta definir el frontend.**

## Formato de entrada

```
## L{X}-{numero} — {título corto}

**Fecha**: YYYY-MM  
**Agente**: {nombre-del-agente}  
**Categoría**: {categoría}

**Situación**: Descripción del problema encontrado.

**Decisión**: Solución aplicada y razón.

**Verificación** (opcional): Cómo confirmar que no se repita.
```

Donde `{X}` es el prefijo del archivo:
- `LL` → `backend-net9.md`
- `LS` → `security-oauth2.md`
- `LR` → `redis-cache.md`
- `LD` → `dba.md`
