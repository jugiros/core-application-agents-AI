# Políticas globales de seguridad corporativa

## Alcance

Estas políticas aplican a todo el espacio de trabajo de manera transparente, sin importar el archivo editado, la tecnología involucrada o el agente activo. Su cumplimiento es obligatorio y no negociable.

## 1. No ejecutar comandos destructivos

### Prohibido

- `rm -rf`, `rm -r`, `rmdir /s /q`, `del /f /s /q` o cualquier comando que elimine archivos o directorios de forma masiva.
- `git push --force`, `git push -f` o reescritura de historia de repositorios.
- `DROP`, `TRUNCATE`, `DELETE FROM`, `UPDATE`, `INSERT INTO` o `ALTER TABLE DROP` en bases de datos sin aprobación explícita y respaldo previo.
- Comandos de formateo de disco, limpieza de particiones o manipulación de volúmenes.

### Permitido

- Eliminación individual de archivos confirmados por el usuario mediante herramientas de edición segura.
- `git clean` solo con la bandera `--dry-run` y previa confirmación del usuario.
- Scripts de migración de base de datos que incluyan transacciones explícitas y verificaciones de existencia previa.

## 2. No exponer información sensible

- No incluir contraseñas, tokens, claves API, cadenas de conexión con credenciales, certificados ni datos personales en archivos de configuración, código fuente ni documentación.
- Las cadenas de conexión deben usar autenticación integrada o leer credenciales de almacenes seguros externos.
- No registrar (loggear) datos sensibles de usuarios, incluyendo identificadores personales, contraseñas o información financiera.

## 3. No realizar cambios en producción sin autorización

- Ningún agente puede ejecutar despliegues, scripts de base de datos o cambios de configuración en ambientes productivos sin aprobación explícita del usuario.
- Los scripts T-SQL destruktivos deben ejecutarse primero en un ambiente de desarrollo o pruebas idéntico al productivo.
- Todo cambio en producción debe ser reversible y contar con un plan de rollback.

## 4. Validación antes de finalización

- Antes de dar por terminada cualquier tarea que modifique código, el agente debe verificar que el proyecto compile correctamente con `msbuild` u otra herramienta equivalente.
- Si el proyecto incluye pruebas, deben ejecutarse y aprobarse las relevantes.
- No se debe dejar código que genere advertencias críticas sin justificación documentada.

## 5. Transparencia en las decisiones

- Todo cambio arquitectónico significativo debe ser explicado brevemente en el resumen de la tarea.
- Si el agente no puede resolver un problema dentro de tres intentos, debe escalar al agente `fullstack-engineer` y notificar al usuario.
- Las decisiones que se aparten de las reglas técnicas deben estar justificadas y documentadas en `docs/lessons-learned/`.

## 6. Respeto a la propiedad intelectual y licencias

- No incluir código protegido por licencias incompatibles con el proyecto.
- No copiar fragmentos de código de fuentes sin verificar la licencia de uso.
- Documentar el origen de cualquier recurso de terceros utilizado.

## 7. Registro de lecciones aprendidas

- Todo error recurrente, solución novedosa o restricción técnica descubierta durante el trabajo debe persistirse en `docs/lessons-learned/` mediante el comando `/update-lessons`.
- Esto garantiza que futuros agentes conozcan el contexto histórico y no repitan errores.

## Incumplimiento

El incumplimiento de cualquiera de estas políticas debe ser reportado inmediatamente al usuario. En caso de detectar una acción automática que viole estas políticas, el agente debe detenerse y solicitar confirmación explícita antes de continuar.
