# /update-lessons [tecnología] [descripción]

## Propósito

Automatizar la persistencia de errores, soluciones, patrones útiles y restricciones técnicas en archivos Markdown dentro de `docs/lessons-learned/`, para que futuros agentes las consideren antes de implementar cambios.

## Parámetros

- `[tecnología]`: Categoría tecnológica a la que aplica la lección. Valores permitidos: `backend-net47`, `frontend-xaml`, `dba`, `mvvm`, `clean-architecture`, `general`.
- `[descripción]`: Breve descripción del problema, la solución o el patrón. Debe ser concisa y en español.

## Archivo destino

El comando selecciona automáticamente el archivo de destino según la tecnología:

- `backend-net47` -> `docs/lessons-learned/backend-net47.md`
- `frontend-xaml` -> `docs/lessons-learned/frontend-xaml.md`
- `dba` -> `docs/lessons-learned/dba.md`
- `mvvm` -> `docs/lessons-learned/mvvm.md`
- `clean-architecture` -> `docs/lessons-learned/clean-architecture.md`
- `general` -> `docs/lessons-learned/general.md`

Si el archivo no existe, se crea con una cabecera estándar.

## Estructura de la lección aprendida

Cada entrada agregada al archivo debe seguir el formato siguiente:

```markdown
## [YYYY-MM-DD] [Título breve de la lección]

- **Tecnología:** [tecnología]
- **Contexto:** [breve descripción de la situación]
- **Problema:** [qué ocurrió o qué se intentó resolver]
- **Solución:** [acción concreta que resolvió el problema]
- **Prevención:** [cómo evitar que se repita en el futuro]
- **Referencia:** [ruta o nombre del archivo afectado, si aplica]
```

## Flujo de ejecución

1. **Validar parámetros**: confirmar que la tecnología está dentro del conjunto permitido y que la descripción no está vacía.
2. **Leer archivo existente**: si el archivo destino ya existe, leer su contenido completo para evitar duplicados.
3. **Comprobar duplicados**: buscar una entrada con la misma descripción o similar. Si ya existe, no agregarla y notificar al usuario.
4. **Agregar entrada**: si no existe, añadir la nueva lección al inicio del archivo para mantener el orden cronológico inverso.
5. **Guardar**: escribir el archivo actualizado en `docs/lessons-learned/`.

## Ejemplo de uso

```
/update-lessons frontend-xaml "DataContext no resuelto cuando el UserControl se anida en un TabItem"
```

## Reglas

- No sobrescribir el archivo completo; solo agregar la nueva entrada.
- Mantener un lenguaje claro y técnico en español.
- Incluir siempre la sección de prevención para facilitar la replicación de buenas prácticas.
- No incluir información sensible, credenciales ni datos de producción.
