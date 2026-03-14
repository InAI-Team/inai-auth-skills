---
name: inai-react-sdk
description: "Integrate InAI Auth SDK into React applications. Use this skill whenever the user wants to add authentication UI, auth hooks, login/signup components, role-based rendering, or session management to a React app using @inai-dev/react. Also trigger when the user mentions InAI auth with React, asks about auth hooks (useAuth, useUser, useSignIn, useSignUp), needs pre-built auth components (SignIn, UserButton, SignedIn, SignedOut, Protect), wants context-based auth state, or is building React islands in Astro that need authentication. This is the client-side React package — for Next.js use @inai-dev/nextjs, for Astro use @inai-dev/astro."
---

# InAI Auth SDK for React

This skill guides you through integrating InAI Auth into React applications using the `@inai-dev/react` package. This is the client-side React package — it provides the context provider, hooks, and pre-built components that all other framework SDKs build upon.

## Key Facts

- **API URL**: Always `https://apiauth.inai.dev` — hardcoded in the SDK, never configurable
- **Package**: `@inai-dev/react`
- **Peer dep**: `react >= 18.0.0`
- **Auth state source**: Reads `auth_session` cookie on mount (set by server-side auth routes)
- **No server-side logic** — this package is client-only. Use `@inai-dev/nextjs`, `@inai-dev/astro`, `@inai-dev/express`, or `@inai-dev/hono` for server-side auth

## Installation

```bash
npm install @inai-dev/react
```

## Integration Consists of 3 Pieces

1. **Provider** — `<InAIAuthProvider>` wrapping your app
2. **Hooks** — `useAuth()`, `useUser()`, `useSignIn()`, `useSignUp()`, `useSession()`, `useOrganization()`
3. **Components** — `<SignIn>`, `<UserButton>`, `<SignedIn>`, `<SignedOut>`, `<Protect>`, `<PermissionGate>`, `<OrganizationSwitcher>`

## 1. Provider Setup

```tsx
import { InAIAuthProvider } from "@inai-dev/react";

function App() {
  return (
    <InAIAuthProvider>
      <YourApp />
    </InAIAuthProvider>
  );
}
```

The provider is zero-config. On mount, it reads the `auth_session` cookie (set by server-side auth routes) to hydrate client-side auth state without a network request.

### How initialization works

1. Provider mounts → reads `document.cookie` for `auth_session`
2. Parses JSON cookie value containing user data, permissions, org context
3. Sets `isLoaded = true` and broadcasts state to all child hooks/components
4. If no cookie → `isLoaded = true`, `isSignedIn = false`

## 2. Hooks

All hooks require `<InAIAuthProvider>` as an ancestor.

### useAuth()

Core hook for authentication state and actions.

```tsx
import { useAuth } from "@inai-dev/react";

function MyComponent() {
  const { isLoaded, isSignedIn, userId, has, signOut } = useAuth();

  if (!isLoaded) return <LoadingSpinner />;
  if (!isSignedIn) return <a href="/login">Sign in</a>;

  const isAdmin = has({ role: "admin" });
  const canEdit = has({ permission: "posts:write" });

  return (
    <div>
      <p>User: {userId}</p>
      {isAdmin && <a href="/admin">Admin Panel</a>}
      <button onClick={signOut}>Sign Out</button>
    </div>
  );
}
```

**Returns**: `{ isLoaded, isSignedIn, userId, has, signOut }`

- `has({ role?: string; permission?: string }): boolean` — check role or permission
- `signOut(): Promise<void>` — calls `/api/auth/logout`, clears state, redirects to `/login`

### useUser()

Returns user profile data.

```tsx
import { useUser } from "@inai-dev/react";

function Profile() {
  const { isLoaded, isSignedIn, user } = useUser();
  if (!isLoaded || !isSignedIn) return null;

  return (
    <div>
      <img src={user.avatarUrl} alt={user.firstName} />
      <h2>{user.firstName} {user.lastName}</h2>
      <p>{user.email}</p>
    </div>
  );
}
```

**Returns**: `{ isLoaded, isSignedIn, user: UserResource | null }`

### useSession()

Returns session and organization context.

```tsx
import { useSession } from "@inai-dev/react";

function SessionInfo() {
  const { isLoaded, isSignedIn, userId, tenantId, orgId, orgRole } = useSession();
  // ...
}
```

**Returns**: `{ isLoaded, isSignedIn, userId, tenantId, orgId, orgRole }`

### useOrganization()

Returns only organization-scoped data.

```tsx
import { useOrganization } from "@inai-dev/react";

function OrgInfo() {
  const { isLoaded, orgId, orgRole } = useOrganization();
  // ...
}
```

**Returns**: `{ isLoaded, orgId, orgRole }`

### useSignIn()

Handles the complete sign-in flow, including MFA.

```tsx
import { useSignIn } from "@inai-dev/react";

function LoginForm() {
  const { signIn, isLoading, error, status, reset } = useSignIn();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    const result = await signIn.create({
      identifier: email,
      password,
    });

    if (result.status === "needs_mfa") {
      // Show MFA form — mfa_token is stored internally
    } else if (result.status === "complete") {
      window.location.href = "/dashboard";
    }
  };

  const handleMFA = async (code: string) => {
    const result = await signIn.attemptMFA({ code });
    if (result.status === "complete") {
      window.location.href = "/dashboard";
    }
  };

  // ...
}
```

**Returns**: `{ signIn, isLoading, error, status, reset }`

- `status`: `"idle"` | `"loading"` | `"needs_mfa"` | `"complete"` | `"error"`
- `signIn.create({ identifier, password })` — initial login attempt
- `signIn.attemptMFA({ code })` — verify TOTP MFA code
- `reset()` — reset state to idle

### useSignUp()

Handles registration flow with email verification support.

```tsx
import { useSignUp } from "@inai-dev/react";

function RegisterForm() {
  const { signUp, isLoading, error, status, reset } = useSignUp();

  const handleSubmit = async () => {
    const result = await signUp.create({
      email,
      password,
      firstName,
      lastName,
    });

    if (result.status === "needs_email_verification") {
      // Show "check your email" message
    } else if (result.status === "complete") {
      window.location.href = "/dashboard";
    }
  };

  // ...
}
```

**Returns**: `{ signUp, isLoading, error, status, reset }`

- `status`: `"idle"` | `"loading"` | `"needs_email_verification"` | `"complete"` | `"error"`

## 3. Pre-Built Components

### SignedIn / SignedOut

Conditional rendering based on auth state.

```tsx
import { SignedIn, SignedOut } from "@inai-dev/react";

function Navbar() {
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

- `<SignedIn>` — renders children only when `isLoaded && isSignedIn`
- `<SignedOut>` — renders children only when `isLoaded && !isSignedIn`

### Protect

Role/permission gate.

```tsx
import { Protect } from "@inai-dev/react";

<Protect role="admin" fallback={<p>Access denied</p>}>
  <AdminPanel />
</Protect>

<Protect permission="posts:write">
  <Editor />
</Protect>
```

**Props**: `{ role?, permission?, fallback?, children }`

### PermissionGate

Alternative permission-based gate (same behavior as Protect).

```tsx
import { PermissionGate } from "@inai-dev/react";

<PermissionGate permission="billing:manage" fallback={<p>No access</p>}>
  <BillingSettings />
</PermissionGate>
```

**Props**: `{ permission?, role?, fallback?, children }`

### UserButton

Avatar dropdown with user info and sign-out.

```tsx
import { UserButton } from "@inai-dev/react";

<UserButton
  afterSignOutUrl="/"
  showName
  menuItems={[
    { label: "Settings", onClick: () => router.push("/settings") },
    { label: "Profile", onClick: () => router.push("/profile") },
  ]}
  appearance={{
    buttonSize: 36,
    buttonBg: "#1a1a2e",
    menuBg: "#1a1a1a",
    menuBorder: "#333",
  }}
/>
```

**Props**:
- `afterSignOutUrl?: string` — redirect after sign out
- `showName?: boolean` — display user name next to avatar
- `menuItems?: { label: string; onClick: () => void }[]` — custom menu items
- `appearance?: { buttonSize, buttonBg, menuBg, menuBorder }` — styling

**Features**: Avatar with initials fallback, keyboard navigation (Arrow Up/Down, Escape), click-outside dismissal, accessible ARIA labels.

### SignIn

Complete login form with MFA support.

```tsx
import { SignIn } from "@inai-dev/react";

<SignIn
  redirectUrl="/dashboard"
  onSuccess={() => console.log("Signed in!")}
  onMFARequired={(mfaToken) => router.push("/mfa")}
/>
```

**Props**:
- `redirectUrl?: string` — redirect on success (reloads page if omitted)
- `onSuccess?: () => void` — callback on successful sign-in
- `onMFARequired?: (mfaToken: string) => void` — callback if MFA needed

Two-step form: email/password → MFA code (if required). Unstyled HTML for easy customization.

### OrganizationSwitcher

Dropdown to switch between user's organizations.

```tsx
import { OrganizationSwitcher } from "@inai-dev/react";

<OrganizationSwitcher />
```

Fetches orgs from `/api/organizations`, switches via `POST /api/auth/set-active-organization`, auto-reloads page after switch.

## UserResource Shape

```typescript
{
  id: string
  tenantId: string
  email: string
  firstName: string | null
  lastName: string | null
  avatarUrl: string | null
  isActive: boolean
  emailVerified: boolean
  mfaEnabled: boolean
  externalId: string | null
  roles: string[]
  createdAt: string
  updatedAt: string
}
```

## API Routes Expected

The React SDK expects these server-side routes to exist (created by `@inai-dev/nextjs`, `@inai-dev/astro`, `@inai-dev/express`, or `@inai-dev/hono`):

| Endpoint | Method | Used by |
|----------|--------|---------|
| `/api/auth/login` | POST | `useSignIn()`, `<SignIn>` |
| `/api/auth/register` | POST | `useSignUp()` |
| `/api/auth/logout` | POST | `useAuth().signOut()`, `<UserButton>` |
| `/api/auth/refresh` | POST | `useAuth().refreshSession()` |
| `/api/auth/mfa-challenge` | POST | `useSignIn().attemptMFA()`, `<SignIn>` |
| `/api/auth/set-active-organization` | POST | `<OrganizationSwitcher>` |
| `/api/organizations` | GET | `<OrganizationSwitcher>` |

## Using with Astro Islands

When using React components inside Astro pages:

```tsx
// src/components/AuthIsland.tsx
import { InAIAuthProvider, SignedIn, SignedOut, UserButton } from "@inai-dev/react";

export function AuthIsland() {
  return (
    <InAIAuthProvider>
      <SignedIn>
        <UserButton showName afterSignOutUrl="/login" />
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
import { AuthIsland } from "../components/AuthIsland";
---
<AuthIsland client:load />
```

## Cookie Architecture

The provider reads one cookie on mount:

| Cookie | Purpose | httpOnly | Set by |
|--------|---------|----------|--------|
| `auth_session` | User data + permissions (JSON) | No | Server-side auth routes |

The `auth_session` cookie contains:
```typescript
{
  user: UserResource
  expiresAt: string
  permissions: string[]
  orgId?: string
  orgRole?: string
  appId?: string
  envId?: string
}
```

## Source Code Reference

When you need to check implementation details, the source files are at:

- `packages/react/src/context.tsx` — InAIAuthProvider, context creation
- `packages/react/src/hooks/use-auth.ts` — useAuth()
- `packages/react/src/hooks/use-user.ts` — useUser()
- `packages/react/src/hooks/use-session.ts` — useSession()
- `packages/react/src/hooks/use-organization.ts` — useOrganization()
- `packages/react/src/hooks/use-sign-in.ts` — useSignIn()
- `packages/react/src/hooks/use-sign-up.ts` — useSignUp()
- `packages/react/src/components/protect.tsx` — Protect
- `packages/react/src/components/signed-in.tsx` — SignedIn
- `packages/react/src/components/signed-out.tsx` — SignedOut
- `packages/react/src/components/permission-gate.tsx` — PermissionGate
- `packages/react/src/components/user-button.tsx` — UserButton
- `packages/react/src/components/sign-in.tsx` — SignIn
- `packages/react/src/components/org-switcher.tsx` — OrganizationSwitcher
