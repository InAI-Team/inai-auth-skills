---
name: inai-express-sdk
description: "Integrate InAI Auth SDK into Express applications. Use this skill whenever the user wants to add authentication, login, signup, middleware protection, role-based access control (RBAC), MFA, or session management to an Express app using @inai-dev/express. Also trigger when the user mentions InAI auth with Express, asks about protecting routes in Express, needs auth middleware for Express, wants to set up auth API routes in Express, or is building a Node.js API server that needs authentication. Covers middleware setup, route protection, cookie management, and both app and platform auth modes."
---

# InAI Auth SDK for Express

This skill guides you through integrating InAI Auth into Express 4+ applications using the `@inai-dev/express` package.

## Key Facts

- **API URL**: Always `https://apiauth.inai.dev` — hardcoded in the SDK, never configurable
- **Required env var**: `INAI_PUBLISHABLE_KEY=pk_live_...`
- **Package**: `@inai-dev/express` (depends on `@inai-dev/backend` and `@inai-dev/shared`)
- **Auth modes**: `"app"` (end users) and `"platform"` (admin/developer panels)
- **Peer dep**: `express >= 4.17.0`

## Installation

```bash
npm install @inai-dev/express
```

## Integration Consists of 3 Pieces

1. **Middleware** — `inaiAuthMiddleware()` as global middleware
2. **API Routes** — `createAuthRoutes()` mounted at `/api/auth`
3. **Route protection** — `requireAuth()` for per-route authorization

## 1. Middleware Setup

```ts
import express from "express";
import { inaiAuthMiddleware } from "@inai-dev/express/middleware";

const app = express();
app.use(express.json());

app.use(
  inaiAuthMiddleware({
    publicRoutes: ["/", "/health", "/api/public/*"],
    signInUrl: "/login",
  })
);
```

### Middleware behavior

1. Checks if request path matches `publicRoutes` — if so, sets `req.auth = null` and proceeds
2. Extracts token from `Authorization: Bearer <token>` header or `auth_token` cookie
3. If token is expired and `refresh_token` cookie exists → auto-refresh
4. Verifies JWT signature using ES256 via JWKS, builds `AuthObject` and attaches to `req.auth`
5. If no valid auth → calls `onUnauthorized` handler (default: 401 JSON response)

### Middleware configuration

```ts
inaiAuthMiddleware({
  authMode: "app",          // "app" (default) or "platform"
  publicRoutes: ["/", "/health"],  // string[] or (req) => boolean
  onUnauthorized: (req, res, next) => {
    res.status(401).json({ error: "Unauthorized" });
  },
  beforeAuth: (req, res) => {
    // Return true to skip auth entirely
  },
  afterAuth: (auth, req, res) => {
    // Called after successful auth — useful for logging
  },
  // jwksUrl: "https://apiauth.inai.dev/.well-known/jwks.json", // optional override
})
```

### publicRoutes patterns

```ts
// Exact match
publicRoutes: ["/health", "/api/public/status"]

// Wildcard
publicRoutes: ["/api/public/*"]

// Function
publicRoutes: (req) => req.path.startsWith("/public/")
```

## 2. API Routes

```ts
import { createAuthRoutes } from "@inai-dev/express/api-routes";

app.use("/api/auth", createAuthRoutes());
```

Creates these endpoints automatically:
- `POST /api/auth/login` — Login, sets httpOnly cookies. Returns `{ user }` or `{ mfa_required, mfa_token }`
- `POST /api/auth/register` — Registration. Returns `{ user }` or `{ needs_email_verification, user }`
- `POST /api/auth/mfa-challenge` — TOTP MFA verification
- `POST /api/auth/refresh` — Token rotation (also called by middleware automatically)
- `POST /api/auth/logout` — Invalidate session, clear cookies

### Platform Auth (admin panels)

For admin/developer panels, use platform auth mode. No publishable key needed:

```ts
app.use(
  inaiAuthMiddleware({
    authMode: "platform",
    publicRoutes: ["/login"],
  })
);
```

Platform mode uses `/api/platform/auth/*` endpoints internally.

## 3. Route Protection

### requireAuth middleware

```ts
import { requireAuth } from "@inai-dev/express/middleware";

// Require any authenticated user
app.get("/api/profile", requireAuth(), (req, res) => {
  res.json({ userId: req.auth!.userId });
});

// Require specific role
app.get("/api/admin", requireAuth({ role: "admin" }), (req, res) => {
  res.json({ message: "Admin access granted" });
});

// Require specific permission
app.put("/api/posts/:id", requireAuth({ permission: "posts:write" }), (req, res) => {
  // Update post...
});
```

### Manual auth check with getAuth

```ts
import { getAuth } from "@inai-dev/express";

app.get("/api/data", (req, res) => {
  const auth = getAuth(req);
  if (!auth?.userId) {
    return res.status(401).json({ error: "Not authenticated" });
  }

  if (auth.has({ role: "admin" })) {
    // Return admin data
  }

  res.json({ userId: auth.userId });
});
```

## Request Extension

The middleware extends Express's Request type:

```ts
// req.auth is available after middleware runs
req.auth?.userId      // string | null
req.auth?.tenantId    // string | null
req.auth?.orgId       // string | null
req.auth?.orgRole     // string | null
req.auth?.sessionId   // string | null
req.auth?.has({ role: "admin" })           // boolean
req.auth?.has({ permission: "posts:write" }) // boolean
req.auth?.getToken()  // Promise<string | null>
```

## Cookie Architecture

| Cookie | Purpose | httpOnly | Path | MaxAge |
|--------|---------|----------|------|--------|
| `auth_token` | Access JWT | Yes | `/` | Token expiry |
| `refresh_token` | Refresh JWT | Yes | `/api/auth` | 7 days |
| `auth_session` | User data (readable by JS) | No | `/` | Token expiry |

- Production (`NODE_ENV=production`): `secure: true` on all cookies
- Development: `secure: false` for http://localhost

## Cookie Helpers

For custom auth flows or manual token management:

```ts
import { setAuthCookies, clearAuthCookies } from "@inai-dev/express";

// After manual authentication
setAuthCookies(res, tokens, user);

// Manual logout
clearAuthCookies(res);
```

## Token Extraction

```ts
import { getTokenFromRequest, getRefreshTokenFromRequest } from "@inai-dev/express";

// Gets token from Authorization header or auth_token cookie
const token = getTokenFromRequest(req);

// Gets refresh token from cookie only
const refreshToken = getRefreshTokenFromRequest(req);
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

### Full Express app with auth

```ts
import express from "express";
import { inaiAuthMiddleware, requireAuth, createAuthRoutes, getAuth } from "@inai-dev/express";

const app = express();
app.use(express.json());

// Global auth middleware
app.use(
  inaiAuthMiddleware({
    publicRoutes: ["/", "/health", "/login", "/register"],
  })
);

// Auth routes
app.use("/api/auth", createAuthRoutes());

// Public route
app.get("/health", (req, res) => res.json({ status: "ok" }));

// Protected route
app.get("/api/me", requireAuth(), (req, res) => {
  res.json({ userId: req.auth!.userId });
});

// Admin-only route
app.delete("/api/users/:id", requireAuth({ role: "admin" }), (req, res) => {
  // Delete user...
});

app.listen(3000);
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
  // Show MFA input, then submit:
  await fetch("/api/auth/mfa-challenge", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ mfa_token: data.mfa_token, code: totpCode }),
  });
}
```

## Source Code Reference

When you need to check implementation details, the source files are at:

- `packages/express/src/middleware.ts` — inaiAuthMiddleware(), requireAuth()
- `packages/express/src/api-routes.ts` — createAuthRoutes()
- `packages/express/src/helpers.ts` — Cookie & token management utilities
- `packages/express/src/types.ts` — TypeScript interfaces
- `packages/backend/src/client.ts` — InAIAuthClient (core API client)
- `packages/shared/src/constants.ts` — Cookie names, URLs, headers
- `packages/shared/src/jwks.ts` — JWKSClient (JWKS key fetching, caching, error throttling)
- `packages/shared/src/jwt.ts` — ES256 verification, JWT decoding
