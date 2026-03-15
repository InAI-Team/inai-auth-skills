---
name: inai-nextjs-sdk
description: "Integrate InAI Auth SDK into Next.js applications. Use this skill whenever the user wants to add authentication, login, signup, middleware protection, role-based access control (RBAC), MFA, or session management to a Next.js app using @inai-dev/nextjs. Also trigger when the user mentions InAI auth with Next.js, asks about protecting routes in Next.js, needs auth hooks (useAuth, useUser), wants to set up auth API routes, or is building an admin panel that needs platform authentication. Covers both app user auth and platform/admin auth flows."
compatibility: "Requires Node.js 18+ and a Next.js 14+ project"
metadata:
  author: inai-team
  version: "1.0.0"
  framework: nextjs
---

# InAI Auth SDK for Next.js

This skill guides you through integrating InAI Auth into Next.js 14+ applications using the `@inai-dev/nextjs` package.

## Key Facts

- **API URL**: Always `https://apiauth.inai.dev` — hardcoded in the SDK, never configurable
- **Required env var**: `INAI_PUBLISHABLE_KEY=pk_live_...`
- **Package**: `@inai-dev/nextjs` (depends on `@inai-dev/react` and `@inai-dev/backend`)
- **Auth modes**: `"app"` (end users) and `"platform"` (admin/developer panels)

## Installation

```bash
npm install @inai-dev/nextjs
```

## Integration Consists of 4 Pieces

1. **Provider** — `<InAIAuthProvider>` in root layout
2. **API Route** — `app/api/auth/[...inai]/route.ts`
3. **Middleware** — `middleware.ts` at project root
4. **Server helpers** — `auth()`, `currentUser()` in server components/actions

## 1. Provider Setup

```tsx
// app/layout.tsx
import { InAIAuthProvider } from "@inai-dev/nextjs";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <InAIAuthProvider>
          {children}
        </InAIAuthProvider>
      </body>
    </html>
  );
}
```

The provider reads the `auth_session` cookie on mount to hydrate client-side state.

## 2. API Routes

### App User Auth (standard)

```ts
// app/api/auth/[...inai]/route.ts
import { createAuthRoutes } from "@inai-dev/nextjs/server";

const { GET, POST } = createAuthRoutes();
export { GET, POST };
```

Creates these endpoints automatically:
- `POST /api/auth/login` — Login, sets httpOnly cookies
- `POST /api/auth/register` — Registration
- `POST /api/auth/mfa-challenge` — TOTP MFA verification
- `POST /api/auth/refresh` — Token rotation
- `POST /api/auth/logout` — Invalidate session, clear cookies

### Platform Auth (admin panels)

```ts
// app/api/auth/[...inai]/route.ts
import { createPlatformAuthRoutes } from "@inai-dev/nextjs/server";

const { GET, POST } = createPlatformAuthRoutes();
export { GET, POST };
```

Uses `/api/platform/auth/*` endpoints. No publishable key needed for platform auth.

## 3. Middleware

```ts
// middleware.ts
import { inaiAuthMiddleware } from "@inai-dev/nextjs/middleware";

export default inaiAuthMiddleware({
  publicRoutes: ["/", "/about", "/pricing", "/login", "/register"],
  signInUrl: "/login",
  // jwksUrl: "https://apiauth.inai.dev/.well-known/jwks.json", // optional override
});

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico).*)"],
};
```

### Middleware behavior
1. Built-in public routes (`/_next/*`, `/favicon.ico`, `/api/*`, `signInUrl`) pass through
2. Your `publicRoutes` pass through
3. Other requests verify `auth_token` JWT signature using ES256 via JWKS (cached 5 min, auto-retry on key rotation)
4. Expired token + valid `refresh_token` → auto-refresh via `/api/auth/refresh`
5. No valid auth → redirect to `signInUrl` with `returnTo` param

### publicRoutes as function

```ts
publicRoutes: (req) => req.nextUrl.pathname.startsWith("/public/")
```

### Hooks: beforeAuth / afterAuth

```ts
export default inaiAuthMiddleware({
  publicRoutes: ["/"],
  signInUrl: "/login",
  beforeAuth: (req) => {
    // Return NextResponse to short-circuit
  },
  afterAuth: (auth, req) => {
    // Role-based routing
    if (req.nextUrl.pathname.startsWith("/admin") && !auth.has({ role: "admin" })) {
      return NextResponse.redirect(new URL("/unauthorized", req.url));
    }
  },
});
```

### createRouteMatcher

```ts
import { inaiAuthMiddleware, createRouteMatcher } from "@inai-dev/nextjs/middleware";

const isAdminRoute = createRouteMatcher(["/admin(.*)"]);

export default inaiAuthMiddleware({
  publicRoutes: ["/", "/login"],
  afterAuth: (auth, req) => {
    if (isAdminRoute(req) && !auth.has({ role: "admin" })) {
      return NextResponse.redirect(new URL("/", req.url));
    }
  },
});
```

### Composing with other middleware (withInAIAuth)

```ts
import { withInAIAuth } from "@inai-dev/nextjs/middleware";
import createIntlMiddleware from "next-intl/middleware";

const intlMiddleware = createIntlMiddleware({ locales: ["en", "es"], defaultLocale: "en" });

export default withInAIAuth(intlMiddleware, {
  publicRoutes: ["/", "/login"],
  signInUrl: "/login",
});
```

## 4. Server Helpers

### auth()

```tsx
// Server Component
import { auth } from "@inai-dev/nextjs/server";

export default async function Page() {
  const { userId, has, protect, redirectToSignIn } = await auth();

  // Option 1: Manual check
  if (!userId) redirectToSignIn({ returnTo: "/dashboard" });

  // Option 2: protect() — redirects if not authenticated/authorized
  const { userId: uid } = protect({ role: "admin" });

  // Option 3: Conditional rendering
  const canEdit = has({ permission: "content:write" });
  return <div>{canEdit && <EditButton />}</div>;
}
```

### currentUser()

```tsx
import { currentUser } from "@inai-dev/nextjs/server";

// From session cookie (fast, no network)
const user = await currentUser();

// Fresh from API
const freshUser = await currentUser({ fresh: true });
```

### configureAuth() (optional)

```ts
import { configureAuth } from "@inai-dev/nextjs/server";

configureAuth({
  signInUrl: "/sign-in",
  signUpUrl: "/sign-up",
  afterSignInUrl: "/dashboard",
  afterSignOutUrl: "/sign-in",
  publishableKey: process.env.INAI_PUBLISHABLE_KEY,
});
```

## Client Hooks (from @inai-dev/nextjs)

| Hook | Returns |
|------|---------|
| `useAuth()` | `isLoaded`, `isSignedIn`, `userId`, `roles`, `permissions`, `has()`, `signOut()` |
| `useUser()` | `isLoaded`, `isSignedIn`, `user` (full UserResource) |
| `useSession()` | `isLoaded`, `isSignedIn`, `userId`, `tenantId`, `orgId`, `orgRole`, `roles`, `permissions` |
| `useOrganization()` | `isLoaded`, `orgId`, `orgRole` |

## Pre-Built Components

| Component | Purpose |
|-----------|---------|
| `<SignIn>` | Complete login form with MFA support |
| `<UserButton>` | Avatar dropdown with user info and sign-out |
| `<SignedIn>` | Render children only when authenticated |
| `<SignedOut>` | Render children only when unauthenticated |
| `<Protect>` | Render children if user has required role/permission |
| `<PermissionGate>` | Role/permission gating |
| `<OrganizationSwitcher>` | Org selector dropdown |

```tsx
"use client";
import { SignedIn, SignedOut, UserButton } from "@inai-dev/nextjs";

export function Navbar() {
  return (
    <nav>
      <SignedIn>
        <UserButton showName afterSignOutUrl="/login" />
      </SignedIn>
      <SignedOut>
        <a href="/login">Sign in</a>
      </SignedOut>
    </nav>
  );
}
```

## Cookie Architecture

| Cookie | Purpose | httpOnly | Path | MaxAge |
|--------|---------|----------|------|--------|
| `auth_token` | Access JWT | Yes | `/` | Token expiry |
| `refresh_token` | Refresh JWT | Yes | `/api/auth` | 7 days |
| `auth_session` | User data (readable by JS) | No | `/` | Token expiry |

The `auth_session` cookie is what the `InAIAuthProvider` reads on mount to hydrate client state without a network request.

## Server Actions

```ts
"use server";
import { auth } from "@inai-dev/nextjs/server";

export async function updateProfile(formData: FormData) {
  const { protect } = await auth();
  protect(); // userId guaranteed non-null after this
  // ... update logic
}
```

## Auth Object Shape

Both server and client auth objects share this structure:

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

Server-only additions: `protect()`, `redirectToSignIn()`.

## Common Patterns

### Login form (client-side)

```tsx
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
// Redirect to dashboard
```

### Role-based page protection

```tsx
import { auth } from "@inai-dev/nextjs/server";

export default async function AdminPage() {
  const { protect } = await auth();
  protect({ role: "admin", redirectTo: "/unauthorized" });
  return <AdminDashboard />;
}
```

## Source Code Reference

When you need to check implementation details, the source files are at:

- `packages/nextjs/src/middleware.ts` — Middleware implementation
- `packages/nextjs/src/server.ts` — auth(), currentUser(), configureAuth()
- `packages/nextjs/src/api-routes.ts` — createAuthRoutes()
- `packages/nextjs/src/platform-api-routes.ts` — createPlatformAuthRoutes()
- `packages/nextjs/src/config.ts` — Configuration management
- `packages/nextjs/src/cookies.ts` — Cookie utilities
- `packages/react/src/context.tsx` — InAIAuthProvider
- `packages/react/src/hooks/` — All React hooks
- `packages/react/src/components/` — Pre-built components
- `packages/shared/src/jwks.ts` — JWKSClient (JWKS key fetching, caching, error throttling)
- `packages/shared/src/jwt.ts` — ES256 verification, JWT decoding
