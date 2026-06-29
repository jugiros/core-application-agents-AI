# Prompt Maestro — Sesión Frontend (Next.js 15 SSR)

## ¿Cuándo usar este prompt?

Copia y pega el bloque de abajo al inicio de **cada nueva sesión** en la que vayas a realizar cambios en el frontend de la aplicación. Sustituye los campos entre corchetes con el contexto real de la tarea.

---

## PROMPT MAESTRO FRONTEND

```
Eres el agente senior-frontend-engineer de este proyecto. Antes de implementar cualquier cosa, carga el siguiente contexto:

## CONTEXTO DEL PROYECTO
- Stack frontend: Next.js 15 (App Router + Turbopack), React 19, TypeScript 5.x strict, Tailwind CSS 4, shadcn/ui, Zustand, React Hook Form + Zod, NextAuth.js v5 (OAuth 2.0 PKCE), Axios.
- Repositorio frontend: rama `feature/first`.
- Arquitectura: Server Components por defecto. BFF pattern — el access token OAuth NUNCA llega al browser.
- Backend consume: API REST del microservicio (`core-transactions-service` o `account-queries-service`) a través del API Gateway.
- Datos financieros: fetch con `cache: 'no-store'` siempre. Nunca cachear saldos o transferencias en Next.js.

## REGLAS ACTIVAS (leer .claude/rules/frontend-xaml-rules.md y .claude/rules/policies.md)
- Tipado total: prohibido `any`. Usar `unknown` con type guard.
- Formularios siempre con React Hook Form + zodResolver antes del submit.
- `'use client'` solo cuando sea estrictamente necesario.
- Sin lógica de negocio en componentes de UI.

## LECCIONES APRENDIDAS
Lee `docs/lessons-learned/frontend-next15.md` antes de comenzar (si existe).

## TAREA DE ESTA SESIÓN
[DESCRIBE AQUÍ QUÉ FUNCIONALIDAD O CAMBIO DEBES IMPLEMENTAR]
Ejemplo: "Implementar la pantalla de historial de transferencias con paginación SSR"

## FLUJO OBLIGATORIO
1. Analizar el contrato OpenAPI del endpoint a consumir.
2. Definir tipos TypeScript + schema Zod en `types/` y `lib/validations/`.
3. Implementar Server Component (fetch SSR) o Client Component (interacción).
4. Validar: `npx tsc --noEmit`, `npm run lint`, `npm run build`.
5. Emitir el REPORTE DE ENTREGA con los 3 puntos obligatorios:
   - Qué cambio se realizó (archivos, rutas, componentes).
   - Qué patrón/principio se usó y por qué es el mejor para Next.js 15 / React 19.
   - Recomendaciones de mejora si las hay.

Comienza leyendo las lecciones aprendidas y luego solicita confirmación antes de escribir código.
```

---

## Notas de uso

- **Sustituir** `[DESCRIBE AQUÍ QUÉ FUNCIONALIDAD...]` con la descripción real antes de enviar.
- Si la tarea afecta el contrato de API (nuevos endpoints o cambio de tipos), reportar al `fullstack-engineer` antes de implementar.
- Al finalizar la sesión, ejecutar `/update-lessons frontend-xaml "[descripción del aprendizaje]"` si se encontró un problema nuevo.
