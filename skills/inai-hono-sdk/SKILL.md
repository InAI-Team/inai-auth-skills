---
name: inai-hono-sdk
description: "Integrate InAI Auth SDK into Hono applications. Use this skill whenever the user wants to add authentication, login, signup, middleware protection, role-based access control (RBAC), MFA, or session management to a Hono app using @inai-dev/hono. Also trigger when the user mentions InAI auth with Hono, asks about protecting routes in Hono, needs auth middleware for Hono or Cloudflare Workers, wants to set up auth API routes in Hono, or is building an edge-first API with Hono that needs authentication. Covers middleware setup, route protection, cookie management, and both app and platform auth modes."
---

# InAI Auth SDK for Hono

This skill guides you through integrating InAI Auth into Hono 4+ applications using the `@inai-dev/hono` package. Works with Cloudflare Workers, Deno, Bun, and Node.js.

## Key Facts

- **API URL**: Always `https://apiauth.inai.dev` — hardcoded in the SDK, never configurable
- **Required env var**: `INAI_PUBLISHABLE_KEY=pk_live_...`
- **Package**: `@inai-dev/hono` (depends on `@inai-dev/backend` and `@inai-dev/shared`)
- **Auth modes**: `"app"` (end users) and `"platform"` (admin/developer panels)
- **Peer dep**: `hono >= 4.0.0`

## Installation

```bash
npm install @inai-dev/hono
```

## Integration Consists of 3 Pieces

1. **Middleware** — `inaiAuthMiddleware()` as global middleware
2. **API Routes** — `createAuthRoutes()` mounted as a sub-app
3. **Route protection** — `requireAuth()` for per-route authorization

## 1. Middleware Setup

```ts
import { Hono } from "hono";
import { inaiAuthMiddleware } from "@inai-dev/hono/middleware";

const app = new Hono();

app.use(
  "*",
  inaiAuthMiddleware({
    publicRoutes: ["/", "/health", "/login", "/register"],
  })
);
```

### Middleware behavior

1. Checks if request path matches `publicRoutes` — if so, sets `inaiAuth` context to `null` and proceeds
2. Extracts token from `Authorization: Bearer <token>` header or `auth_token` cookie
3. If token is expired and `refresh_token` cookie exists → auto-refresh
4. Verifies JWT signature using ES256 via JWKS, builds `AuthObject` and stores in Hono context as `inaiAuth`
5. If no valid auth → calls `onUnauthorized` handler (default: 401 JSON response)

### Middleware configuration

```ts
inaiAuthMiddleware({
  authMode: "app",          // "app" (default) or "platform"
  publicRoutes: ["/", "/health"],  // string[] or (path: string) => boolean
  onUnauthorized: (c) => {
    return c.json({ error: "Unauthorized" }, 401);
  },
  // jwksUrl: "https://apiauth.inai.dev/.well-known/jwks.json", // optional override
})
```

### publicRoutes patterns

```ts
// String array
publicRoutes: ["/", "/health", "/api/public/*"]

// Function
publicRoutes: (path) => path.startsWith("/public/")
```

## 2. API Routes

```ts
import { createAuthRoutes } from "@inai-dev/hono/api-routes";

const authRoutes = createAuthRoutes();
app.route("/api/auth", authRoutes);
```

Creates these endpoints automatically:
- `POST /api/auth/login` — Login, sets httpOnly cookies. Returns `{ user }` or `{ mfa_required, mfa_token }`
- `POST /api/auth/register` — Registration. Returns `{ user }` or `{ needs_email_verification, user }`
- `POST /api/auth/mfa-challenge` — TOTP MFA verification
- `POST /api/auth/refresh` — Token rotation (also called by middleware automatically)
- `POST /api/auth/logout` — Invalidate session, clear cookies

### Platform Auth (admin panels)

For admin/developer panels, use platform auth mode:

```ts
app.use(
  "*",
  inaiAuthMiddleware({
    authMode: "platform",
    publicRoutes: ["/login"],
  })
);
```

Platform mode uses `/api/platform/auth/*` endpoints internally. No publishable key needed.

## 3. Route Protection

### requireAuth middleware

```ts
import { requireAuth } from "@inai-dev/hono/middleware";

// Require any authenticated user
app.get("/api/profile", requireAuth(), (c) => {
  const auth = getAuth(c);
  return c.json({ userId: auth?.userId });
});

// Require specific role
app.get("/api/admin", requireAuth({ role: "admin" }), (c) => {
  return c.json({ message: "Admin access granted" });
});

// Require specific permission
app.put("/api/posts/:id", requireAuth({ permission: "posts:write" }), (c) => {
  // Update post...
});
```

### Manual auth check with getAuth

```ts
import { getAuth } from "@inai-dev/hono";

app.get("/api/data", (c) => {
  const auth = getAuth(c);
  if (!auth?.userId) {
    return c.json({ error: "Not authenticated" }, 401);
  }

  if (auth.has({ role: "admin" })) {
    // Return admin data
  }

  return c.json({ userId: auth.userId });
});
```

## Context Extension

The middleware extends Hono's context:

```ts
// c.get("inaiAuth") is available after middleware runs
const auth = c.get("inaiAuth");
auth?.userId      // string | null
auth?.tenantId    // string | null
auth?.orgId       // string | null
auth?.orgRole     // string | null
auth?.sessionId   // string | null
auth?.roles       // string[]
auth?.permissions // string[]
auth?.has({ role: "admin" })           // boolean
auth?.has({ permission: "posts:write" }) // boolean
auth?.getToken()  // Promise<string | null>
```

Hono's `ContextVariableMap` is augmented:

```ts
declare module "hono" {
  interface ContextVariableMap {
    inaiAuth: AuthObject | null;
  }
}
```

## Cookie Architecture

| Cookie | Purpose | httpOnly | Path | MaxAge |
|--------|---------|----------|------|--------|
| `auth_token` | Access JWT | Yes | `/` | Token expiry |
| `refresh_token` | Refresh JWT | Yes | `/` | 7 days |
| `auth_session` | User data (readable by JS) | No | `/` | Token expiry |

- Production (`NODE_ENV=production`): `secure: true` on all cookies
- Development: `secure: false` for http://localhost

## Cookie Helpers

For custom auth flows or manual token management:

```ts
import { setAuthCookies, clearAuthCookies } from "@inai-dev/hono";

// After manual authentication
setAuthCookies(c, tokens, user);

// Manual logout
clearAuthCookies(c);
```

## Token Extraction

```ts
import { getTokenFromContext, getRefreshTokenFromContext } from "@inai-dev/hono";

// Gets token from Authorization header or auth_token cookie
const token = getTokenFromContext(c);

// Gets refresh token from cookie only
const refreshToken = getRefreshTokenFromContext(c);
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
  roles: string[]
  permissions: string[]
  getToken(): Promise<string | null>
  has(params: { role?: string; permission?: string }): boolean
}
```

## Common Patterns

### Full Hono app with auth

```ts
import { Hono } from "hono";
import {
  inaiAuthMiddleware,
  requireAuth,
  createAuthRoutes,
  getAuth,
} from "@inai-dev/hono";

const app = new Hono();

// Global auth middleware
app.use(
  "*",
  inaiAuthMiddleware({
    publicRoutes: ["/", "/health", "/login", "/register"],
  })
);

// Auth routes
app.route("/api/auth", createAuthRoutes());

// Public route
app.get("/health", (c) => c.json({ status: "ok" }));

// Protected route
app.get("/api/me", requireAuth(), (c) => {
  const auth = getAuth(c);
  return c.json({ userId: auth?.userId });
});

// Admin-only route
app.delete("/api/users/:id", requireAuth({ role: "admin" }), (c) => {
  // Delete user...
  return c.json({ deleted: true });
});

export default app;
```

### Cloudflare Workers deployment

```ts
// src/index.ts
import { Hono } from "hono";
import { inaiAuthMiddleware, createAuthRoutes } from "@inai-dev/hono";

type Bindings = {
  INAI_PUBLISHABLE_KEY: string;
};

const app = new Hono<{ Bindings: Bindings }>();

app.use("*", (c, next) => {
  return inaiAuthMiddleware({
    publicRoutes: ["/"],
    publishableKey: c.env.INAI_PUBLISHABLE_KEY,
  })(c, next);
});

app.route("/api/auth", createAuthRoutes());

export default app;
```

### Login form handler (client-side)

```ts
const res = await fetch("/api/auth/login", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ email, password }),
});
const data = await res.json();

if (data.mfa_required) {
  await fetch("/api/auth/mfa-challenge", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ mfa_token: data.mfa_token, code: totpCode }),
  });
}
```

## Source Code Reference

When you need to check implementation details, the source files are at:

- `packages/hono/src/middleware.ts` — inaiAuthMiddleware(), requireAuth()
- `packages/hono/src/api-routes.ts` — createAuthRoutes()
- `packages/hono/src/helpers.ts` — Cookie & token context utilities
- `packages/hono/src/types.ts` — TypeScript interfaces & Hono type augmentation
- `packages/backend/src/client.ts` — InAIAuthClient (core API client)
- `packages/shared/src/constants.ts` — Cookie names, URLs, headers
- `packages/shared/src/jwks.ts` — JWKSClient (JWKS key fetching, caching, error throttling)
- `packages/shared/src/jwt.ts` — ES256 verification, JWT decoding
