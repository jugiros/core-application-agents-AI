# Lecciones aprendidas

Este directorio almacena el conocimiento histórico del proyecto, organizado por tecnología y área de responsabilidad.

## Propósito

Cada agente debe leer las lecciones correspondientes antes de iniciar una tarea. Las entradas deben seguir el formato definido en `.claude/commands/update-lessons.md`.

## Archivos esperados

- `backend-net47.md` — Lecciones sobre C#, .NET Framework 4.7, LINQ, servicios, repositorios y excepciones.
- `frontend-xaml.md` — Lecciones sobre WPF, XAML, MVVM, ViewModels, bindings y conversores.
- `dba.md` — Lecciones sobre esquemas relacionales, migraciones, scripts T-SQL y optimización.
- `mvvm.md` — Lecciones generales sobre el patrón MVVM y la separación de responsabilidades.
- `clean-architecture.md` — Lecciones sobre inyección de dependencias, capas de dominio y aislamiento.
- `general.md` — Lecciones transversales que no encajan en una categoría específica.

## Cómo agregar una nueva lección

Utilizar el comando slash:

```
/update-lessons [tecnología] [descripción]
```

Por ejemplo:

```
/update-lessons frontend-xaml "DataContext no resuelto cuando el UserControl se anida en un TabItem"
```
