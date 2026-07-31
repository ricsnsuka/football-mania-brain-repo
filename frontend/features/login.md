# Login Feature

Route: `/login` · Component: `LoginForm` · Auth: Public

---

## Overview

The login page allows users to authenticate using their username or email and password. On success, auth state is stored in memory (Zustand) and the user is redirected to `/dashboard`. If `forcePasswordChange` is true, a change-password modal is shown in-place.

The root route (`/`) redirects to `/dashboard` if the user is already authenticated, or to `/login` otherwise.

A **Register** link on the login page opens a modal for creating a new account (see [Registration](#registration) below).

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
Response: { "token": "...", "userId": 1, "username": "johndoe",
            "email": "...", "role": "a member with no roles", "forcePasswordChange": false }
```

---

## Registration

```
POST /api/users   (skipAuth: true)

Request:  { "username", "email", "password", "firstName", "lastName", "role" }
Response: UserDTO (201)
```

The register modal (`RegisterModal`) collects all `UserCreateDTO` fields with client-side Zod validation:
- `firstName`, `lastName` — required
- `username` — 3–50 characters
- `email` — valid email format
- `password` — minimum 8 characters; `confirmPassword` matches (client-only, not sent)
- `role` — select: `a member with no roles` (default) | `MANAGER` | `ADMIN`

On success a `success` toast is shown via `addNotification`. On `409 Conflict`, an inline error reports duplicate username/email.

---

## Auth State

Token is stored **in Zustand memory only** — never written to `localStorage`, `sessionStorage`, or cookies. It is lost on page refresh and the user must re-login.

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
