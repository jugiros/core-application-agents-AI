# Reglas de Calidad de Código — SonarLint (Backend C# + Frontend TypeScript)

## Alcance

Aplica a todos los archivos de **todos los microservicios backend** (`.cs`) y al **repositorio frontend** (`fintech-frontend`) (`.ts`, `.tsx`). SonarLint valida automáticamente en el IDE y en cada PR.

---

## BACKEND — SonarLint para C# (.NET Core 9)

### Configuración en el IDE

**VS Code:**
1. Instalar extensión: `SonarLint` (publisher: SonarSource) — ID: `SonarSource.sonarlint-vscode`
2. No requiere configuración adicional — analiza automáticamente archivos `.cs` abiertos.

**Visual Studio 2022:**
1. Extensions → Manage Extensions → buscar "SonarLint for Visual Studio"
2. Instalar y reiniciar.

**Rider:**
1. Settings → Plugins → buscar "SonarLint" → instalar.

### Paquete NuGet para análisis en CI (agregar a test projects)

```xml
<!-- En Domain.Tests.csproj, Application.Tests.csproj, Infrastructure.Tests.csproj -->
<PackageReference Include="SonarAnalyzer.CSharp" Version="10.3.0.106239">
  <PrivateAssets>all</PrivateAssets>
  <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
</PackageReference>
```

### `.editorconfig` recomendado (backend)

Crear `.editorconfig` en la raíz de cada microservicio:

```ini
# .editorconfig — .NET Core 9 Fintech Microservices
root = true

[*.cs]
indent_style = space
indent_size = 4
end_of_line = crlf
charset = utf-8-bom
trim_trailing_whitespace = true
insert_final_newline = true

# Naming
dotnet_naming_rule.private_fields.symbols = private_field_symbols
dotnet_naming_rule.private_fields.style = underscore_camel_case_style
dotnet_naming_rule.private_fields.severity = warning
dotnet_naming_symbols.private_field_symbols.applicable_kinds = field
dotnet_naming_symbols.private_field_symbols.applicable_accessibilities = private
dotnet_naming_style.underscore_camel_case_style.capitalization = camel_case
dotnet_naming_style.underscore_camel_case_style.required_prefix = _

# Sonar-alineados
dotnet_diagnostic.S1135.severity = warning    # Track uses of "TODO" tags
dotnet_diagnostic.S2139.severity = error      # Exceptions should be either logged or rethrown
dotnet_diagnostic.S3925.severity = error      # "ISerializable" should be implemented correctly
dotnet_diagnostic.S4457.severity = error      # Split method into two (sync/async)
dotnet_diagnostic.CA2016.severity = warning   # Forward CancellationToken
dotnet_diagnostic.CA1822.severity = warning   # Mark members as static
dotnet_diagnostic.CA1848.severity = warning   # Use LoggerMessage for high-performance logging
```

### Reglas SonarLint críticas para C# (estas NO deben violarse)

| Código Sonar | Descripción | Acción |
|---|---|---|
| `S1066` | Mergeable if statements | Simplificar ifs anidados |
| `S1199` | Nested code blocks | Extraer método |
| `S2184` | Result of integer division should be checked | Siempre verificar divisiones |
| `S2259` | Null pointer dereference | Usar null checks o `ArgumentNullException.ThrowIfNull` |
| `S3925` | Implement ISerializable correctly | Solo serializar lo necesario |
| `S4457` | Split into sync/async method pairs | No mezclar sync/await |
| `S6966` | Await asynchronous method | Nunca `.Result` ni `.Wait()` en async context |
| `CA2016` | Forward CancellationToken | Siempre propagar el token |

### Comandos de análisis (CI/CD)

```bash
# Instalar dotnet-sonarscanner globalmente (solo una vez)
dotnet tool install --global dotnet-sonarscanner

# Análisis completo (requiere SonarQube/SonarCloud)
dotnet sonarscanner begin \
  /k:"fintech-core-transactions" \
  /d:sonar.host.url="http://localhost:9000" \
  /d:sonar.login="your-token"

dotnet build PruebaNetCoreProject.sln

dotnet sonarscanner end /d:sonar.login="your-token"

# Análisis local rápido (solo IDE SonarLint — sin SonarQube)
# El análisis ocurre automáticamente al abrir archivos
```

---

## FRONTEND — SonarLint para TypeScript/Next.js 15

### Configuración en el IDE

**VS Code (recomendado para Next.js):**
1. Instalar extensión: `SonarLint` (publisher: SonarSource)
2. SonarLint detecta automáticamente `.ts` y `.tsx` y aplica reglas SonarJS.

### ESLint con plugin SonarJS

Agregar al `package.json` de `fintech-frontend`:

```bash
npm install -D eslint-plugin-sonarjs@^3.0.1
```

`.eslintrc.json` (crear en raíz de `fintech-frontend/`):

```json
{
  "extends": [
    "next/core-web-vitals",
    "next/typescript",
    "plugin:sonarjs/recommended"
  ],
  "plugins": ["sonarjs"],
  "rules": {
    "sonarjs/no-duplicate-string": ["error", { "threshold": 3 }],
    "sonarjs/cognitive-complexity": ["error", 15],
    "sonarjs/no-identical-functions": "error",
    "sonarjs/no-empty-collection": "error",
    "sonarjs/no-unused-collection": "error",
    "sonarjs/prefer-immediate-return": "warn",
    "sonarjs/no-nested-template-literals": "warn",
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/explicit-function-return-type": "off",
    "@typescript-eslint/no-unused-vars": ["error", { "argsIgnorePattern": "^_" }],
    "no-console": ["warn", { "allow": ["error"] }]
  }
}
```

### Reglas SonarJS críticas para TypeScript (estas NO deben violarse)

| Regla | Descripción | Acción |
|---|---|---|
| `sonarjs/no-duplicate-string` | Strings literales duplicados | Extraer a constante |
| `sonarjs/cognitive-complexity` | Complejidad cognitiva > 15 | Extraer función |
| `sonarjs/no-identical-functions` | Funciones idénticas | Extraer utilidad compartida |
| `@typescript-eslint/no-explicit-any` | Prohibido `any` | Usar tipo explícito o `unknown` |
| `no-console` | `console.log` en producción | Usar logger o eliminar |

### Comandos de análisis (frontend)

```bash
# Verificar reglas ESLint + SonarJS
npm run lint

# Auto-fix donde sea posible
npm run lint -- --fix

# Type check
npx tsc --noEmit

# Build completo (incluye ESLint como parte del proceso)
npm run build
```

---

## Regla general — Flujo de validación de calidad en cada entrega

```
Backend:
  1. dotnet build --no-incremental   ← Sin warnings (TreatWarningsAsErrors=true)
  2. dotnet test                     ← Todos los tests verdes
  3. SonarLint IDE → sin issues críticos ni bloqueantes

Frontend:
  1. npm run lint                    ← Sin warnings ni errores ESLint/SonarJS
  2. npx tsc --noEmit               ← Sin errores TypeScript
  3. npm run build                   ← Build de producción exitoso
  4. SonarLint IDE → sin issues críticos
```

---

## Regla de prohibición SonarLint

Si SonarLint reporta un issue de severidad **Critical** o **Blocker**, el código **no puede mergearse** hasta resolverlo. No se usan supresiones (`#pragma warning disable` ni `// NOSONAR`) sin justificación documentada.

Si se suprime un issue Sonar, el PR debe incluir un comentario que explique:
1. Por qué la supresión es necesaria
2. Qué riesgo existe
3. Quién aprobó la supresión (senior-security-architect para temas de seguridad)
