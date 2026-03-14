---
name: inai-astro-sdk
description: "Integrate InAI Auth SDK into Astro applications. Use this skill whenever the user wants to add authentication, login, signup, middleware protection, role-based access control (RBAC), MFA, or session management to an Astro app using @inai-dev/astro. Also trigger when the user mentions InAI auth with Astro, asks about protecting routes in Astro, needs server-side auth helpers (auth(), currentUser()), wants to set up auth API routes in Astro, or is building an SSR Astro site that needs authentication. Covers both the integration plugin and direct @inai-dev/backend usage."
compatibility: "Requires Node.js 18+ and an Astro 6+ project with output: server (SSR)"
metadata:
  author: inai-team
  version: "1.0.0"
  framework: astro
---

# InAI Auth SDK for Astro

This skill guides you through integrating InAI Auth into Astro 6+ applications using the `@inai-dev/astro` package.

## Key Facts

- **API URL**: Always `https://apiauth.inai.dev` — hardcoded in the SDK, never configurable
- **Required env var**: `INAI_PUBLISHABLE_KEY=pk_live_...`
- **Package**: `@inai-dev/astro` (depends on `@inai-dev/backend` and `@inai-dev/shared`)
- **Requirement**: Astro must be in `output: "server"` (SSR) mode
- **No client-side provider** — Astro integration is server-only; for React islands, use `@inai-dev/react` separately

## Installation

```bash
npm install @inai-dev/astro
```

## Integration Consists of 3 Pieces

1. **Integration plugin** — `inaiAuth()` in `astro.config.mjs`
2. **Middleware** — `inaiAstroMiddleware()` in `src/middleware.ts`
3. **API Routes** — `createAuthRoutes()` in `src/pages/api/auth/[path].ts`

Server helpers (`auth()`, `currentUser()`) are available in any `.astro` page or API endpoint.

## 1. Integration Plugin

```ts
// astro.config.mjs
import { defineConfig } from "astro/config";
import { inaiAuth } from "@inai-dev/astro";

export default defineConfig({
  output: "server", // Required for auth
  integrations: [inaiAuth()],
});
```

The plugin is currently minimal and exists mainly for future extensibility.

## 2. Middleware

```ts
// src/middleware.ts
import { inaiAstroMiddleware } from "@inai-dev/astro/middleware";

export const onRequest = inaiAstroMiddleware({
  publicRoutes: ["/", "/about", "/pricing"],
  signInUrl: "/login",
});
```

### Middleware behavior

1. Checks `auth_token` cookie
2. Decodes JWT claims (no signature verification — the API is the security boundary)
3. Populates `Astro.locals.auth` with an `AuthObject`
4. Redirects unauthenticated users on protected routes to `signInUrl`
5. Auto-refreshes expired tokens when `refresh_token` exists

### Composing with other middleware

```ts
// src/middleware.ts
import { sequence } from "astro:middleware";
import { inaiAstroMiddleware } from "@inai-dev/astro/middleware";

const authMiddleware = inaiAstroMiddleware({
  publicRoutes: ["/", "/about"],
  signInUrl: "/login",
});

export const onRequest = sequence(authMiddleware, myOtherMiddleware);
```

Use Astro's built-in `sequence()` to chain middleware — the auth middleware populates `Astro.locals.auth` for downstream middleware and pages.

## 3. API Routes

```ts
// src/pages/api/auth/[path].ts
import { createAuthRoutes } from "@inai-dev/astro/api-routes";

const routes = createAuthRoutes({
  publishableKey: process.env.INAI_PUBLISHABLE_KEY,
});

export const ALL = routes.ALL;
```

Creates these endpoints automatically:
- `POST /api/auth/login` — Login, sets httpOnly cookies. Returns `{ user }` or `{ mfa_required, mfa_token }`
- `POST /api/auth/register` — Registration
- `POST /api/auth/mfa-challenge` — TOTP MFA verification
- `POST /api/auth/refresh` — Token rotation (also called by middleware automatically)
- `POST /api/auth/logout` — Invalidate session, clear cookies

## Server Helpers

### auth()

Synchronous. Returns the `AuthObject` from `Astro.locals.auth` (populated by middleware).

```astro
---
// src/pages/dashboard.astro
import { auth } from "@inai-dev/astro/server";

const authObj = auth(Astro);

if (!authObj?.userId) {
  return Astro.redirect("/login");
}

const canManage = authObj.has({ role: "admin" });
---

<h1>Dashboard</h1>
{canManage && <a href="/admin">Admin Panel</a>}
```

### currentUser()

Async. Fetches the full `UserResource` from the API using the access token.

```astro
---
import { currentUser } from "@inai-dev/astro/server";

const user = await currentUser(Astro);
if (!user) return Astro.redirect("/login");
---

<p>{user.email}</p>
<p>{user.firstName} {user.lastName}</p>
```

You can optionally pass a `publishableKey`:

```ts
const user = await currentUser(Astro, {
  publishableKey: process.env.INAI_PUBLISHABLE_KEY,
});
```

### In API Endpoints

```ts
// src/pages/api/profile.ts
import type { APIRoute } from "astro";
import { auth } from "@inai-dev/astro/server";

export const GET: APIRoute = async (context) => {
  const authObj = auth(context);
  if (!authObj?.userId) {
    return new Response(JSON.stringify({ error: "Unauthorized" }), { status: 401 });
  }

  return new Response(JSON.stringify({ userId: authObj.userId }), {
    headers: { "Content-Type": "application/json" },
  });
};
```

### Cookie Helpers

For advanced use cases (custom auth flows, manual token management):

```ts
import { setAuthCookies, clearAuthCookies } from "@inai-dev/astro/server";

// Set auth cookies after manual authentication
setAuthCookies(Astro.cookies, tokens, user);

// Clear all auth cookies (manual logout)
clearAuthCookies(Astro.cookies);
```

## Auth Object Shape

```typescript
{
  userId: string | null
  tenantId: string | null
  appId: string | null
  envId: string | null
  orgId: string | null
  orgRole: string | null
  sessionId: string | null
  getToken(): Promise<string | null>
  has(params: { role?: string; permission?: string }): boolean
}
```

## Client-Side Auth (from .astro pages or islands)

Since Astro is server-first, client-side auth is done via fetch to the API routes:

```ts
// Login
const res = await fetch("/api/auth/login", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ email, password }),
});
const data = await res.json();

if (data.mfa_required) {
  // Show MFA form, then:
  await fetch("/api/auth/mfa-challenge", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ mfa_token: data.mfa_token, code: totpCode }),
  });
}

// Logout
await fetch("/api/auth/logout", { method: "POST" });
```

## React Islands with Auth

If you use React islands in Astro and need client-side auth state, install `@inai-dev/react` separately:

```bash
npm install @inai-dev/react
```

Then wrap your island with the provider:

```tsx
// src/components/AuthIsland.tsx
import { InAIAuthProvider, useAuth, SignedIn, SignedOut } from "@inai-dev/react";

export function AuthIsland() {
  return (
    <InAIAuthProvider>
      <SignedIn>
        <p>Authenticated!</p>
      </SignedIn>
      <SignedOut>
        <a href="/login">Sign in</a>
      </SignedOut>
    </InAIAuthProvider>
  );
}
```

```astro
---
// src/pages/index.astro
import { AuthIsland } from "../components/AuthIsland";
---

<AuthIsland client:load />
```

## Alternative: Direct Backend Client

You can use `@inai-dev/backend` directly without the Astro integration package for full control:

```ts
// src/lib/auth-client.ts
import { InAIAuthClient } from "@inai-dev/backend";

export const authClient = new InAIAuthClient({
  publishableKey: process.env.INAI_PUBLISHABLE_KEY,
});
```

```astro
---
import { authClient } from "../lib/auth-client";

const token = Astro.cookies.get("auth_token")?.value;
if (!token) return Astro.redirect("/login");

try {
  const { data: user } = await authClient.getMe(token);
} catch {
  return Astro.redirect("/login");
}
---

<h1>Dashboard</h1>
```

This approach gives you access to all `InAIAuthClient` methods (login, register, refresh, MFA, organization management, etc.) but you handle cookies and middleware yourself.

## Cookie Architecture

| Cookie | Purpose | httpOnly | Path | MaxAge |
|--------|---------|----------|------|--------|
| `auth_token` | Access JWT | Yes | `/` | Token expiry |
| `refresh_token` | Refresh JWT | Yes | `/api/auth` | 7 days |
| `auth_session` | User data (readable by JS) | No | `/` | Token expiry |

The `auth_session` cookie enables React islands to hydrate auth state without a network request.

## Common Patterns

### Protected page with role check

```astro
---
import { auth } from "@inai-dev/astro/server";

const authObj = auth(Astro);
if (!authObj?.userId) return Astro.redirect("/login");
if (!authObj.has({ role: "admin" })) return Astro.redirect("/unauthorized");
---

<h1>Admin Panel</h1>
```

### Login page

```astro
---
import { auth } from "@inai-dev/astro/server";

// Redirect if already signed in
const authObj = auth(Astro);
if (authObj?.userId) return Astro.redirect("/dashboard");
---

<form id="login-form">
  <input type="email" name="email" required />
  <input type="password" name="password" required />
  <button type="submit">Sign In</button>
</form>

<script>
  document.getElementById("login-form")?.addEventListener("submit", async (e) => {
    e.preventDefault();
    const form = e.target as HTMLFormElement;
    const data = Object.fromEntries(new FormData(form));

    const res = await fetch("/api/auth/login", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(data),
    });

    if (res.ok) {
      window.location.href = "/dashboard";
    }
  });
</script>
```

## Source Code Reference

When you need to check implementation details, the source files are at:

- `packages/astro/src/middleware.ts` — Middleware implementation
- `packages/astro/src/server.ts` — auth(), currentUser(), cookie helpers
- `packages/astro/src/api-routes.ts` — createAuthRoutes()
- `packages/astro/src/integration.ts` — Astro integration plugin
- `packages/backend/src/client.ts` — InAIAuthClient (core API client)
- `packages/shared/src/constants.ts` — Cookie names, URLs, headers
