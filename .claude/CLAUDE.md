# Sistema Principal de Agente Arquitectónico

## Contexto global del proyecto

Este espacio de trabajo está configurado para una aplicación de escritorio empresarial construida sobre el siguiente stack tecnológico:

- **Backend**: .NET Framework 4.7 (C#)
- **Frontend**: XAML / WPF (Windows Presentation Foundation)
- **Patrón arquitectónico**: MVVM (Model-View-ViewModel) con integración de principios de Clean/Hexagonal Architecture en el núcleo C#.
- **Motor de automatización**: Claude Code CLI / Windsurf Workspace

## Propósito de este directorio

La carpeta `.claude/` centraliza la configuración, los perfiles de agentes, los comandos slash y las reglas técnicas que rigen la generación y modificación de código dentro de este repositorio. Cada archivo aquí contenido debe ser considerado de lectura obligatoria antes de realizar cualquier edición estructural.

## Validación de compilación

Antes de considerar completa cualquier tarea, el agente debe verificar que el código generado compila satisfactoriamente mediante herramientas compatibles con .NET Framework 4.7. Los comandos de verificación permitidos incluyen:

- `msbuild` contra la solución o proyecto activo.
- El comando de test del framework en uso si el proyecto cuenta con proyectos de prueba.
- Revisión de errores del compilador de C# y del parser de XAML.

La generación de código que no compile o que produzca advertencias críticas no se considera terminada hasta que se corrijan.

## Recursos compartidos

El agente debe consultar y mantener actualizados los siguientes recursos históricos:

- `docs/lessons-learned/`: Directorio donde se almacenan los aprendizajes por tecnología, errores recurrentes y soluciones aplicadas. El agente debe leer estas lecciones antes de iniciar cualquier implementación de módulo.
- Memoria histórica del agente: Contexto acumulado de decisiones arquitectónicas previas, restricciones de negocio y acuerdos de equipo.

## Directrices de orquestación

1. Todo cambio que afecte múltiples capas (Base de datos -> Lógica C# -> Vistas XAML/ViewModels) debe ser coordinado por el agente `fullstack-engineer`.
2. La separación de responsabilidades MVVM no debe romperse nunca. El código de vista XAML debe depender exclusivamente de Command Binding y propiedades expuestas en ViewModels.
3. Los agentes especializados (`senior-frontend-engineer`, `senior-backend-engineer`, `senior-dba`) operan de forma conjunta siguiendo ciclos de Analizar -> Implementar -> Validar -> Reintentar.
4. La regla de oro del espacio de trabajo: preferir siempre correcciones mínimas en el origen del problema antes de agregar workarounds en capas superiores.

## Referencias de agentes y reglas

- Perfiles de agentes: `.claude/agents/`
- Comandos slash: `.claude/commands/`
- Reglas técnicas: `.claude/rules/`
