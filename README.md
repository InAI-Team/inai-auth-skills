# InAI Auth SDK — Agent Skills

Skills para integrar [InAI Auth](https://apiauth.inai.dev) en aplicaciones Next.js y Astro. Compatible con multiples agentes de codigo.

## Agentes compatibles

Las skills se instalan automaticamente en todos los agentes soportados:

| Agente | Tipo |
|--------|------|
| Claude Code | symlink |
| Cursor | universal |
| Codex | universal |
| Gemini CLI | universal |
| GitHub Copilot | universal |
| Kiro CLI | symlink |
| OpenCode | universal |
| Antigravity | symlink |

## Skills disponibles

| Skill | Framework | Paquete | Descripcion |
|-------|-----------|---------|-------------|
| `inai-nextjs-sdk` | Next.js 14+ | `@inai-dev/nextjs` | Auth completo: Provider, middleware, API routes, hooks, componentes, RBAC, MFA |
| `inai-astro-sdk` | Astro 6+ | `@inai-dev/astro` | Auth SSR: Integration plugin, middleware, API routes, server helpers, RBAC, MFA |

## Instalacion

### Listar skills disponibles

```bash
npx skills add InAI-Team/inai-auth-skills --list
```

### Instalar una skill

```bash
# Next.js
npx skills add InAI-Team/inai-auth-skills --skill inai-nextjs-sdk

# Astro
npx skills add InAI-Team/inai-auth-skills --skill inai-astro-sdk
```

### Instalar todas

```bash
npx skills add InAI-Team/inai-auth-skills --all
```

El instalador detecta automaticamente los agentes disponibles en tu sistema y configura las skills para cada uno (symlink o copia universal segun el agente).

## Requisitos

- Un agente de codigo compatible (ver tabla arriba)
- Variable de entorno `INAI_PUBLISHABLE_KEY` configurada en tu proyecto (`pk_live_...`)

## Que cubre cada skill

### inai-nextjs-sdk

- `<InAIAuthProvider>` en root layout
- API routes (`/api/auth/*`) con `createAuthRoutes()`
- Middleware con proteccion de rutas y auto-refresh de tokens
- Server helpers: `auth()`, `currentUser()`, `configureAuth()`
- Client hooks: `useAuth()`, `useUser()`, `useSession()`, `useOrganization()`
- Componentes: `<SignIn>`, `<UserButton>`, `<SignedIn>`, `<SignedOut>`, `<Protect>`
- Auth modes: `"app"` (usuarios finales) y `"platform"` (admin panels)

### inai-astro-sdk

- Integration plugin `inaiAuth()` en `astro.config.mjs`
- Middleware `inaiAstroMiddleware()` con proteccion de rutas
- API routes (`/api/auth/*`) con `createAuthRoutes()`
- Server helpers: `auth(Astro)`, `currentUser(Astro)`
- Cookie helpers: `setAuthCookies()`, `clearAuthCookies()`
- Soporte para React islands con `@inai-dev/react`
- Uso directo de `InAIAuthClient` desde `@inai-dev/backend`

## SDK Packages

- [`@inai-dev/nextjs`](https://www.npmjs.com/package/@inai-dev/nextjs) — Next.js SDK
- [`@inai-dev/astro`](https://www.npmjs.com/package/@inai-dev/astro) — Astro SDK
- [`@inai-dev/react`](https://www.npmjs.com/package/@inai-dev/react) — React hooks y componentes
- [`@inai-dev/backend`](https://www.npmjs.com/package/@inai-dev/backend) — Backend client
