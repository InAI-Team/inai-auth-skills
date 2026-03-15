# InAI Auth SDK — Agent Skills

Skills para integrar [InAI Auth](https://apiauth.inai.dev) en aplicaciones Next.js, Astro, Express, Hono y React. Compatible con multiples agentes de codigo.

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
| `inai-nextjs-sdk` | Next.js 14+ | `@inai-dev/nextjs` | Auth completo: Provider, middleware con verificación ES256 via JWKS, API routes, hooks, componentes, RBAC, MFA, y auto-refresh de tokens |
| `inai-astro-sdk` | Astro 6+ | `@inai-dev/astro` | Auth SSR: Integration plugin, middleware con verificación ES256 via JWKS, API routes, server helpers, RBAC, MFA |
| `inai-express-sdk` | Express 4+ | `@inai-dev/express` | Auth middleware con verificación ES256 via JWKS, API routes, route protection, cookie management |
| `inai-hono-sdk` | Hono 4+ | `@inai-dev/hono` | Auth middleware con verificación ES256 via JWKS, API routes, route protection, Cloudflare Workers |
| `inai-react-sdk` | React 18+ | `@inai-dev/react` | Provider, hooks, pre-built components, client-side auth state |

## Instalacion

### Listar skills disponibles

```bash
npx skills add InAI-Team/inai-auth-skills --list
```

### Instalar una skill

```bash
# Next.js (full-stack)
npx skills add InAI-Team/inai-auth-skills --skill inai-nextjs-sdk

# Astro (SSR)
npx skills add InAI-Team/inai-auth-skills --skill inai-astro-sdk

# Express (Node.js API)
npx skills add InAI-Team/inai-auth-skills --skill inai-express-sdk

# Hono (edge/Workers)
npx skills add InAI-Team/inai-auth-skills --skill inai-hono-sdk

# React (client-side)
npx skills add InAI-Team/inai-auth-skills --skill inai-react-sdk
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

### inai-express-sdk

- Middleware global `inaiAuthMiddleware()` con auto-refresh de tokens
- API routes (`/api/auth/*`) con `createAuthRoutes()` (Express Router)
- Route protection con `requireAuth({ role?, permission? })`
- `getAuth(req)` para acceder al auth object desde cualquier handler
- Cookie helpers: `setAuthCookies()`, `clearAuthCookies()`
- Token extraction: `getTokenFromRequest()`, `getRefreshTokenFromRequest()`
- Extension de `req.auth` en Express Request

### inai-hono-sdk

- Middleware global `inaiAuthMiddleware()` con auto-refresh de tokens
- API routes (`/api/auth/*`) con `createAuthRoutes()` (Hono sub-app)
- Route protection con `requireAuth({ role?, permission? })`
- `getAuth(c)` para acceder al auth object desde context
- Cookie helpers: `setAuthCookies()`, `clearAuthCookies()`
- Token extraction: `getTokenFromContext()`, `getRefreshTokenFromContext()`
- Soporte nativo para Cloudflare Workers con bindings

### inai-react-sdk

- `<InAIAuthProvider>` zero-config (lee `auth_session` cookie)
- Hooks: `useAuth()`, `useUser()`, `useSession()`, `useOrganization()`, `useSignIn()`, `useSignUp()`
- Componentes: `<SignIn>`, `<UserButton>`, `<SignedIn>`, `<SignedOut>`, `<Protect>`, `<PermissionGate>`, `<OrganizationSwitcher>`
- Soporte para React islands en Astro

## SDK Packages

- [`@inai-dev/nextjs`](https://www.npmjs.com/package/@inai-dev/nextjs) — Next.js SDK
- [`@inai-dev/astro`](https://www.npmjs.com/package/@inai-dev/astro) — Astro SDK
- [`@inai-dev/express`](https://www.npmjs.com/package/@inai-dev/express) — Express SDK
- [`@inai-dev/hono`](https://www.npmjs.com/package/@inai-dev/hono) — Hono SDK
- [`@inai-dev/react`](https://www.npmjs.com/package/@inai-dev/react) — React hooks y componentes
- [`@inai-dev/backend`](https://www.npmjs.com/package/@inai-dev/backend) — Backend client

## Soporte

Visita [https://inai.dev](https://inai.dev) para documentación, guías y soporte.
