# /analyze-module [nombre_modulo]

## Propósito

Realizar un análisis estrictamente de solo lectura de un módulo, mapeando sus dependencias y detectando acoplamientos rígidos, violaciones a MVVM o malas prácticas en XAML/C#.

## Parámetro

- `[nombre_modulo]`: Nombre de la característica o módulo a analizar. Debe coincidir con el nombre de la entidad, ViewModel, vista o carpeta funcional (por ejemplo: `Cliente`, `Facturacion`, `Inventario`).

## Modo de operación

**ESTRICTAMENTE DE SOLO LECTURA.** Este comando no genera, modifica ni elimina archivos. Únicamente produce un reporte de hallazgos.

## Flujo de ejecución

### 1. Localización del módulo

- Buscar archivos que contengan el nombre del módulo en rutas conocidas del proyecto:
  - Entidades y servicios: `**/Entities/`, `**/Services/`, `**/Repositories/`, `**/UseCases/`, `**/Models/`
  - ViewModels: `**/ViewModels/`
  - Vistas: `**/Views/`, `**/Views/**/*.xaml`

### 2. Mapeo de dependencias

- Listar todos los archivos `.cs` y `.xaml` relacionados.
- Identificar referencias entre clases, inyecciones de dependencias, herencias, interfaces implementadas y bindings XAML.
- Detectar dependencias circulares o referencias a capas inferiores desde capas superiores.

### 3. Detección de acoplamientos rígidos

- Buscar referencias directas a clases de infraestructura desde la vista o el ViewModel.
- Identificar código en archivos `.xaml.cs` que contenga lógica de negocio, llamadas a servicios o acceso a datos.
- Detectar bindings a propiedades inexistentes o mal escritas.
- Identificar propiedades no notificables en ViewModels.

### 4. Evaluación de malas prácticas

- Revisar el uso de `async`/`await` y la presencia de bloqueos sincrónicos como `.Result`, `.Wait()` o `.GetAwaiter().GetResult()` innecesarios.
- Verificar que los recursos de datos se gestionen con `using` blocks.
- Revisar capturas de excepciones vacías o genéricas.
- Verificar que los comandos expuestos en el ViewModel sean `ICommand` y no manejadores de eventos directos.

### 5. Reporte de hallazgos

Generar un resumen estructurado con las siguientes secciones:

- **Archivos analizados**: lista de rutas absolutas de archivos revisados.
- **Dependencias encontradas**: diagrama o lista de relaciones entre componentes.
- **Problemas críticos**: violaciones a reglas técnicas que impiden la correcta operación o mantenimiento.
- **Recomendaciones**: acciones concretas de refactorización, ordenadas por prioridad.
- **Riesgos de arquitectura**: descripción de acoplamientos rígidos que podrían causar regresiones.

## Ejemplo de uso

```
/analyze-module Cliente
```

## Restricciones

- No crear, modificar ni eliminar archivos durante el análisis.
- No ejecutar comandos destructivos.
- No generar scripts de base de datos.
- El resultado debe presentarse como texto plano o Markdown, sin aplicar cambios automáticos.
