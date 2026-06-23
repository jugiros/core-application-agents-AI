# Reglas Backend — .NET Framework 4.7 (C#)

## Alcance

Estas reglas condicionales se aplican automáticamente a todos los archivos C# del backend (`**/*.cs`), excluyendo explícitamente los archivos de ViewModel (`**/*ViewModel.cs`), que se rigen por `.claude/rules/frontend-xaml-rules.md`.

## Reglas críticas obligatorias

### 1. Gestión correcta del ciclo de vida de recursos de datos

- Todo recurso que implemente `IDisposable` debe liberarse correctamente.
- Preferir la declaración `using` para objetos de acceso a datos, conexiones, lectores, streams y contextos de ORM.
- En operaciones que involucren múltiples recursos disposables, usar bloques `using` anidados o la sintaxis `using` con múltiples declaraciones si el compilador de .NET 4.7 lo permite.

```csharp
using (var conexion = new SqlConnection(_cadenaConexion))
using (var comando = new SqlCommand("SELECT * FROM Cliente", conexion))
{
    conexion.Open();
    using (var reader = comando.ExecuteReader())
    {
        while (reader.Read())
        {
            // Procesar fila
        }
    }
}
```

### 2. Validación de tipos estructurados

- Los métodos públicos de servicios y casos de uso deben validar los parámetros de entrada.
- Usar `ArgumentNullException`, `ArgumentException` o excepciones de dominio personalizadas para señalar valores inválidos.
- No propagar valores nulos o inconsistentes hacia capas inferiores (repositorios, base de datos).

### 3. Asincronía compatible con .NET 4.7

- Utilizar `Task`, `async` y `await` para operaciones de I/O, acceso a red o base de datos.
- Evitar el bloqueo sincrónico de operaciones asíncronas. No usar `.Result`, `.Wait()` ni `.GetAwaiter().GetResult()` salvo en casos justificados y documentados.
- Usar `ConfigureAwait(false)` en bibliotecas de lógica de negocio que no requieren contexto de sincronización UI.
- Recordar que `async`/`await` está disponible en .NET Framework 4.5 y superiores, incluyendo 4.7.

### 4. Inyección de dependencias y aislamiento

- Los servicios de aplicación y dominio deben depender de abstracciones (interfaces), no de implementaciones concretas.
- Los repositorios concretos deben implementar interfaces definidas en capas superiores.
- Los controladores o puntos de entrada de la capa de presentación deben recibir sus dependencias por inyección, no mediante `new` directo de servicios de infraestructura.

### 5. Control estructurado de excepciones

- Capturar excepciones de tipo específico, nunca capturas vacías genéricas.
- Registrar el error con el nivel adecuado antes de relanzar o transformar la excepción.
- Transformar excepciones de infraestructura en excepciones de dominio cuando la capa superior no deba conocer detalles técnicos.

```csharp
try
{
    return await _repositorio.ObtenerPorIdAsync(id).ConfigureAwait(false);
}
catch (DataException ex)
{
    _logger.Error(ex, "Error al consultar el cliente {ClienteId}", id);
    throw new ServicioException("No se pudo recuperar la información del cliente.", ex);
}
```

### 6. Uso correcto de LINQ

- Preferir LINQ deferred execution para consultas que se materialicen cerca del origen de datos.
- No materializar colecciones completas en memoria si solo se requiere un subconjunto; usar `Where`, `Select`, `Take`, `Skip`, etc.
- Asegurar que las consultas LINQ contra ORM sean traducibles a SQL.

### 7. Separación de responsabilidades

- No incluir lógica de presentación en servicios de backend.
- No incluir acceso directo a UI desde clases de dominio o aplicación.
- Mantener el dominio independiente de frameworks de infraestructura.

## Sanciones por incumplimiento

Si un archivo incumple estas reglas, el agente debe:

1. Reportar la infracción en el análisis.
2. Proponer la refactorización mínima para alcanzar el cumplimiento.
3. No considerar la tarea completa hasta que el archivo esté conforme.
