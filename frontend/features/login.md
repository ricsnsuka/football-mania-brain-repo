# Login Feature

Route: `/login` · Component: `LoginForm` · Auth: Public

---

## Overview

The login page allows users to authenticate using their username or email and password. On success, auth state is stored in memory (Zustand) and the user is redirected to `/dashboard`. If `forcePasswordChange` is true, a change-password modal is shown in-place.

The root route (`/`) redirects to `/dashboard` if the user is already authenticated, or to `/login` otherwise.

A **Register** link on the login page opens a modal for creating a new account (see [Registration](#registration) below).

A **Forgot password?** link sits beside it, and is **always shown** — even on a deployment whose
server cannot send email, because that page is also where somebody is told to ask an admin instead.
The whole flow is its own document: [password-recovery](password-recovery.md). ⚠️ Added in 2.2.0,
which is cut on `next` and not deployed.

---

## Files

| File | Description |
|---|---|
| `src/app/page.tsx` | Root redirect — sends unauthenticated users to `/login`, authenticated to `/dashboard` |
| `src/types/auth.ts` | `LoginRequest`, `LoginResponse`, `AuthUser`, `ApiError` types |
| `src/types/user.ts` | `UserCreateDTO`, `UserDTO` types |
| `src/lib/apiClient.ts` | Base `apiFetch` wrapper — attaches `Authorization: Bearer` header, handles 401 auto-logout |
| `src/services/authService.ts` | `loginUser(data)`, `registerUser(data)` |
| `src/hooks/auth/useLogin.ts` | TanStack Query `useMutation` — calls `loginUser`, updates store |
| `src/hooks/auth/useRegister.ts` | TanStack Query `useMutation` — calls `registerUser` |
| `src/store/appStore.ts` | Zustand store — auth slice: `token`, `user`, `isAuthenticated`, `setAuth`, `clearAuth` |
| `src/components/auth/LoginForm.tsx` | `'use client'` form — react-hook-form + Zod, calls `useLogin`, includes register link |
| `src/components/auth/RegisterModal.tsx` | `'use client'` native `<dialog>` — registration form |
| `src/app/(auth)/layout.tsx` | Unauthenticated shell layout (centred card, no navbar) |
| `src/app/(auth)/login/page.tsx` | Next.js App Router page at `/login` |

---

## API Contract

```
POST /api/auth/login   (public — no JWT required)

Request:  { "identifier": "johndoe", "password": "secret" }
Response: { "token": "...", "userId": 1, "username": "johndoe", "email": "...",
            "role": null, "roles": [], "forcePasswordChange": false }
```

`roles` is the field the client reads; `role` is the deprecated pre-V18 single name, still emitted
for one release because the two repositories deploy separately. `useLogin` takes
`data.roles ?? rolesFromLegacy(data.role)` so either ordering of those deploys works, and
`loginResponseSchema` marks both optional for the same reason. An empty `roles` is a normal
account, not an error.

---

## Registration

```
POST /api/users/register   (public — skipAuth: true)

Request:  { "username", "email", "password", "firstName"?, "lastName"? }
Response: UserDTO (201)
```

> **Not `POST /api/users`.** That is a different, admin-only endpoint taking `UserCreateDTO`,
> which *does* carry `roles`. The public registration endpoint is `/api/users/register` and its
> payload (`UserRegisterDTO`) has no `roles` field at all — the account is always created with no
> grants, whatever the caller sends, because self-service role selection would be a
> privilege-escalation route. An administrator grants roles afterwards via
> `PATCH /api/users/{id}/role`.

The register modal (`RegisterModal`) collects the `UserRegisterDTO` fields with client-side Zod validation:
- `firstName`, `lastName` — optional
- `username` — 3–50 characters
- `email` — valid email format
- `password` — minimum 8 characters; `confirmPassword` matches (client-only, not sent)

There is no role control in the modal, by design.

On success a `success` toast is shown via `addNotification`. On `409 Conflict`, an inline error reports duplicate username/email.

---

## Auth State

Token and user are persisted to **`sessionStorage`** (locale and theme go to `localStorage` — two
genuinely different lifetimes from one `persist()` call). The session therefore **survives a page
refresh** and is scoped to the tab: closing it ends the session, and `clearAuth()` removes all
three auth keys.

Never `localStorage` and never a cookie: `localStorage` would outlive the tab on a shared machine,
and a cookie would be sent on every request, which is what the bearer-token scheme exists to avoid.

> ⚠️ This section previously claimed the token was held in memory only and lost on refresh. That
> stopped being true when persistence was added; it is recorded here because "memory-only" is
> exactly the sort of claim a security review would rely on without re-reading the store.

```ts
// Read auth state anywhere
const { isAuthenticated, user, token } = useAppStore();

// Log out
useAppStore.getState().clearAuth();
```

---

## Environment Variables

Copy `.env.local.example` to `.env.local` and set:

```
NEXT_PUBLIC_API_URL=http://localhost:8080
```

---

## i18n Keys

- Login strings: `login.*` in `src/locales/{en,es,pt}/common.json`
- Register strings: `register.*` in `src/locales/{en,es,pt}/common.json`

---

## Security Notes

- Password never logged or stored beyond the fetch call
- Token never touches `localStorage` (XSS mitigation)
- Any 401 from the API client auto-calls `clearAuth()` and redirects to `/login`
- Form fields use `autoComplete="username"` / `autoComplete="current-password"` / `autoComplete="new-password"` for password manager support
