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

The access token and user are persisted to **`sessionStorage`** (locale and theme go to
`localStorage` — two genuinely different lifetimes from one `persist()` call). The access token
therefore **survives a page refresh** and is scoped to the tab.

### Staying signed in: the refresh token (decided 2026-11-07, web #313)

Since backend V69, login also returns a **refresh token**. The web keeps it in **`localStorage`**
(key `refreshToken`), with a copy of the user and last group (`session.user`,
`session.activeGroupId`) kept only while a refresh token exists. A new tab, or an installed app
relaunched, restores from that copy with no access token, and the first request renews one through
`POST /api/auth/refresh` (`src/lib/sessionRefresh.ts`). `clearAuth()` forgets all three.

**Why `localStorage` and not an httpOnly cookie.** A cookie page scripts cannot read is the better
place in principle, and it was the first choice. It does not work on this deployment: the web is on
`amadora-foot.netlify.app` and the API on Heroku, and `netlify.app` and `herokuapp.com` are both on
the public suffix list, so the two are different *sites*. A cookie the API sets is third-party, and
Safari blocks third-party cookies by default, including in the app installed to an iPhone home
screen, which is most of the audience. The alternatives were weighed and set aside for now:

| Option | Why not now |
|---|---|
| Netlify proxy for `/api/*`, cookie first-party | Every call takes an extra hop, the chat and draft live streams (SSE) would run through the proxy, and the backend needs a cookie and CSRF change |
| A shared custom domain (`app.` and `api.` of one domain) | Needs a domain and DNS, Heroku and Netlify setup first. **This is the upgrade path:** with it, move the refresh token to an httpOnly `SameSite` cookie |

**What the choice costs, and why it is acceptable.** A script injected into the page could read the
refresh token. Such a script can already read the access token in `sessionStorage`, so there is no
new reader. A refresh token outlives the tab on a shared machine, which is what the old rule here
guarded against. Signing out ends it, and so does a password change anywhere. Every use rotates it,
and a spent token presented again ends that device's session, which limits what a stolen copy is
worth. It lasts 60 days since last use. The access token stays at 24h until both clients refresh.

**Renewal rules** (`sessionRefresh.ts`):
- The API answers an ended session with **403**, which is also its answer to a missing role, so the
  client reads the access token's `exp` claim. It renews before a request when the token is missing
  or within 60 seconds of expiry. After a 403, it renews and retries once only if the token had
  actually run out.
- Renewals are serialised, because a spent token sent again more than 30 seconds later signs the
  device out. Inside a tab, concurrent requests share one promise. Across tabs, a Web Lock
  (`football-mania-session-refresh`) does the same where the browser supports it. The refresh token is
  always read from storage at the moment of use, never from memory.
- A refused renewal clears the session and goes to `/login?next=…`. A renewal that cannot be
  attempted (offline, 5xx, 429) keeps the session.

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
- The access token never touches `localStorage`. The refresh token does, by decision; see
  *Staying signed in* above for why, and for the upgrade path to an httpOnly cookie
- Any 401 from the API client auto-calls `clearAuth()` and redirects to `/login`
- Form fields use `autoComplete="username"` / `autoComplete="current-password"` / `autoComplete="new-password"` for password manager support
