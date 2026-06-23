# /implement-module [nombre_modulo]

## Propósito

Automatiza la orquestación de la creación de una característica de inicio a fin, siguiendo la secuencia Entidad -> ViewModel -> Vista XAML, respetando la arquitectura MVVM y Clean Architecture del proyecto.

## Parámetro

- `[nombre_modulo]`: Nombre de la característica o módulo a implementar. Debe estar en PascalCase y representar una entidad de dominio o funcionalidad clara (por ejemplo: `Cliente`, `Facturacion`, `Inventario`).

## Flujo de ejecución

### 1. Preparación

- Leer `.claude/CLAUDE.md` para establecer el contexto global.
- Leer las lecciones aprendidas en `docs/lessons-learned/` que correspondan a `backend-net47`, `frontend-xaml` y `policies`.
- Activar el agente `fullstack-engineer` como orquestador principal.

### 2. Diseño de la entidad de dominio

- Activar el agente `senior-backend-engineer`.
- Definir la entidad principal en la capa de dominio (por ejemplo, `Cliente.cs`).
- Incluir propiedades, validaciones básicas y relaciones si aplica.
- Asegurar que la entidad no dependa de frameworks de infraestructura.

### 3. Servicio y repositorio

- Definir el contrato `I[nombre_modulo]Repository` en la capa de aplicación o dominio.
- Implementar el repositorio concreto en la capa de infraestructura usando `using` para recursos de datos.
- Crear el servicio de aplicación `I[nombre_modulo]Service` con las operaciones necesarias, manejo de excepciones estructurado y `async`/`await` cuando corresponda.

### 4. ViewModel

- Activar el agente `senior-frontend-engineer`.
- Crear el ViewModel `[nombre_modulo]ViewModel.cs` implementando `INotifyPropertyChanged`.
- Exponer propiedades reactivas que reflejen los datos de la entidad.
- Exponer comandos `ICommand` para las acciones del usuario (guardar, cancelar, buscar, etc.).
- Inyectar el servicio de aplicación mediante el constructor.

### 5. Vista XAML

- Crear la vista `[nombre_modulo]View.xaml` como `UserControl` o `Window` según el contexto de la aplicación.
- Asignar el `DataContext` de forma explícita.
- Diseñar el layout con bindings a propiedades y comandos del ViewModel.
- Utilizar `IValueConverter` cuando sea necesario para transformaciones visuales.
- Mantener el archivo `.xaml.cs` con código mínimo: solo llamada a `InitializeComponent()` y delegación a comandos si es estrictamente necesario.

### 6. Integración y registro

- Registrar el servicio, repositorio y ViewModel en el contenedor de dependencias o inicializador de la aplicación.
- Asegurar que la vista pueda ser instanciada y resuelta correctamente.

### 7. Validación

- Ejecutar `msbuild` para compilar la solución.
- Revisar que no existan errores de binding en XAML.
- Verificar que el ViewModel notifique cambios y que los comandos estén vinculados.

### 8. Documentación

- Si durante la implementación se identifica un error recurrente, un patrón útil o una restricción técnica, ejecutar `/update-lessons [tecnología] [descripción]` para persistirlo.

## Ejemplo de uso

```
/implement-module Cliente
```

## Restricciones

- No se permite agregar lógica de negocio en la vista XAML ni en su code-behind.
- Todo cambio de base de datos debe ser propuesto por el agente `senior-dba` antes de ser aplicado.
- La tarea no se considera completa hasta que el proyecto compile exitosamente.
