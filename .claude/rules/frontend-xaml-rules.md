# Reglas Frontend — XAML y ViewModels

## Alcance

Estas reglas condicionales se aplican automáticamente a todos los archivos que coincidan con los patrones:

- `**/*.xaml`
- `**/*ViewModel.cs`
- `**/*.xaml.cs` (revisión de code-behind)

## Reglas críticas obligatorias

### 1. Uso estricto de Data Context explícito

- Toda vista (`Window`, `UserControl`, `Page`) debe declarar su `DataContext` de forma explícita, ya sea:
  - Asignado en el contenedor padre (`<views:ClienteView DataContext="{Binding ClienteViewModel}"/>`).
  - Asignado en el constructor del contenedor principal de la aplicación.
  - Configurado mediante un `DataTemplate` de recursos.
- No se permite que el `DataContext` se resuelva de manera implícita o accidental por herencia visual sin que esté documentado.

### 2. Prohibición de Code-Behind complejo en archivos `.xaml.cs`

- Los archivos `.xaml.cs` deben limitarse a:
  - La llamada a `InitializeComponent()` en el constructor.
  - Manejadores de eventos que únicamente delegan a un comando del ViewModel.
  - Inicialización de recursos de presentación que no involucren lógica de negocio.
- Está prohibido:
  - Acceder a repositorios, servicios o base de datos desde el code-behind.
  - Implementar lógica de validación, cálculo o transformación de negocio en la vista.
  - Modificar propiedades del modelo directamente desde el code-behind.

### 3. Uso adecuado de conversores de datos (IValueConverter)

- Cada vez que un valor del ViewModel requiera transformación visual (formatos, visibilidad, colores, habilitación condicional), debe usarse un `IValueConverter`.
- Los conversores deben ser pequeños, testeables y registrados en los recursos de la vista o en `App.xaml`.
- No se permite incrustar lógica de conversión directamente en el XAML mediante expresiones complejas o bindings multi-capa sin conversores.

### 4. Propiedades reactivas notificables

- Todo ViewModel debe implementar `INotifyPropertyChanged`.
- Cada propiedad expuesta debe lanzar el evento `PropertyChanged` cuando su valor cambie.
- El nombre de la propiedad pasado en `PropertyChangedEventArgs` debe coincidir exactamente con el nombre de la propiedad.

### 5. Comandos en lugar de eventos directos

- Las acciones de usuario deben vincularse a comandos `ICommand` expuestos desde el ViewModel.
- Queda prohibido usar manejadores de eventos directos en controles (`Click`, `SelectionChanged`, etc.) para ejecutar lógica de negocio.
- Los manejadores de eventos solo están permitidos si su única función es invocar un comando del ViewModel.

### 6. DRY y MVVM

- No repetir lógica visual en múltiples vistas. Extraer estilos, plantillas y controles a `App.xaml` o diccionarios de recursos compartidos.
- Cada vista debe tener un único propósito de presentación y depender exclusivamente de su ViewModel.

## Ejemplo de vista conforme

```xml
<UserControl x:Class="MiApp.Views.ClienteView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="clr-namespace:MiApp.ViewModels"
             xmlns:conv="clr-namespace:MiApp.Converters">
    <UserControl.Resources>
        <conv:BoolToVisibilityConverter x:Key="BoolToVisibilityConverter" />
    </UserControl.Resources>
    <Grid DataContext="{Binding ClienteViewModel}">
        <TextBox Text="{Binding Nombre, UpdateSourceTrigger=PropertyChanged}" />
        <Button Content="Guardar"
                Command="{Binding GuardarCommand}"
                Visibility="{Binding PuedeGuardar, Converter={StaticResource BoolToVisibilityConverter}}" />
    </Grid>
</UserControl>
```

## Sanciones por incumplimiento

Si un archivo incumple estas reglas, el agente debe:

1. Reportar la infracción en el análisis.
2. Proponer la refactorización mínima para alcanzar el cumplimiento.
3. No considerar la tarea completa hasta que el archivo esté conforme.
