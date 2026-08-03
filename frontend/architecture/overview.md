# Architecture Overview

> Updated as part of the 2026-07-27 cross-cutting remediation (performance, architecture,
> design-system, accessibility, dead-code). This revision corrects several claims a prior
> audit found stale — folder layout, locales, and hooks/tests structure below reflect the
> actual `src/` tree, not the pre-remediation doc.

## Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Framework | Next.js App Router | 16.x |
| Language | TypeScript | 5.x (strict) |
| Styling | Tailwind CSS | v4 — via `@layer components` in `globals.css` only |
| Server state | TanStack Query | v5 |
| Client state | Zustand | v5 |
| i18n | i18next + react-i18next | v26 |
| Testing | Vitest + Testing Library | v4 |
| Schema validation | Zod | at the `apiFetch` boundary — see below |

## Folder Structure

```
src/
├── app/                  # Next.js App Router — pages and layouts
│   ├── layout.tsx        # Root layout (Server Component: providers, pre-paint theme script)
│   ├── page.tsx          # Root redirect ('/' → /dashboard or /login once hydrated)
│   ├── not-found.tsx     # Root 404 page
│   ├── globals.css       # Single source of all component styles (@layer components)
│   ├── (auth)/           # Route group: login layout + page (no navbar)
│   └── (app)/            # Route group: authenticated shell (Navbar + Footer)
│       ├── layout.tsx    # Server Component — renders Navbar/Footer around children
│       ├── error.tsx     # Client Component — translated error boundary + retry
│       ├── loading.tsx   # Route-level loading fallback
│       └── <route>/
│           └── page.tsx  # Client Component — wraps feature page in <AuthGuard><PageContainer>
├── components/           # Shared UI atoms and molecules (used across features)
│   ├── auth/              # AuthGuard, LoginForm(+Lazy), RegisterModal, ChangePasswordModal
│   ├── layout/             # Navbar, Footer, PageContainer
│   └── ui/                 # Pagination, TableSkeleton, DashboardSkeleton, DataStateMessage,
│                            # NotificationWidget, LanguageWidget, ThemeWidget
├── features/             # Feature modules — self-contained
│   └── <feature>/         # e.g. players/, matches/, matchPlans/, dashboard/, teamGeneration/, users/
│       └── *.tsx           # Page + feature-specific components (flat per feature; matches/matchModal/
│                            # is the one exception — split into its own subfolder, see below)
├── hooks/                # TanStack Query hooks, one subfolder per domain: useX.ts
│   ├── auth/ · player/ · match/ · matchPlan/ · user/ · draft/
│   └── useVersion.ts      # not yet moved into its own hooks/version/ subfolder (known, minor drift)
├── lib/                  # Framework-agnostic helpers — apiClient.ts, formatDate.ts, i18n init
├── locales/              # Translation JSON files — en/, pt/, and es/ (all three exist and are
│                          # loaded lazily per-locale; a prior doc omitted `es`)
├── providers/            # React context providers (QueryProvider, I18nProvider, ThemeProvider)
├── services/             # Raw fetch functions (no React), one file per domain
├── store/                # Zustand stores (appStore.ts — auth, locale, theme, notifications)
├── tests/                # All tests, grouped by domain/area rather than mirroring src/ 1:1
│   ├── app/                # error.tsx / loading.tsx / not-found.tsx
│   ├── auth/ · components/ · dashboard/ · draft/ · lib/ · matchPlans/ · matches/ · players/ · store/
│   └── setup.ts
└── types/                # TypeScript type definitions (by domain), including types/api.ts
```

Note on `tests/`: it is organized by feature/domain area (`players/`, `matches/`, `draft/`,
`dashboard/`, `auth/`, …) rather than by the source-tree category folders (`hooks/`,
`services/`, `features/`) a prior version of this doc described — a hook test like
`useDraft.test.ts` lives in `tests/draft/`, not `tests/hooks/`. See
[Testing Guide](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/main/docs/guides/testing.md) for the exact convention.

## Data Flow

```
Backend API
  └── src/services/<domain>.service.ts   (fetch, via apiFetch)
        └── src/hooks/<domain>/use<X>.ts  (TanStack Query)
              └── src/features/*/        (React components)
                    └── Zustand store    (client-only state: auth, locale, theme, notifications)
```

## API Client & Error Handling

`src/lib/apiClient.ts` exports a single `apiFetch<T>(path, init?, schema?)` used by every
service module:

- Attaches `Authorization: Bearer <token>` from `useAppStore` unless `{ skipAuth: true }`.
- On a non-OK response, throws `ApiError` (`src/types/api.ts`) — a real `class ApiError
  extends Error`, so callers use `error instanceof ApiError` rather than an unsafe cast.
  Every query/mutation hook is typed `useQuery<T, ApiError>` / `useMutation<T, ApiError, V>`
  accordingly.
- Accepts an optional `schema?: ZodType<T>` (a zod schema, e.g. built with `pageSchema()`
  from `types/api.ts`). In development a failed parse throws immediately (loud failure to
  surface API drift); in production it fails open — `safeParse` + `console.error`, returning
  the unvalidated JSON — so a schema mismatch degrades rather than takes the app down.
- Every queryFn forwards TanStack's injected `{ signal }` through to `apiFetch`, so an
  in-flight request is aborted when a newer one supersedes it (page changes, filter changes,
  unmount).
- `Page<T>` (Spring-style pagination envelope) also lives in `types/api.ts`, alongside
  `ApiError` — both were previously stranded in `types/user.ts` / `types/auth.ts`.

## Query Layer Pattern: canonical key + `select`

The players data hook (`src/hooks/player/usePlayers.ts`) is the reference implementation for
a pattern other list-heavy features should follow: instead of one query-key variant per
consumer-specific fetch shape, there is **one canonical query** —

```ts
export const playerKeys = { all: ['players', 'all'] as const };

export function usePlayers<TData = Page<PlayerDTO>>(options?: {
  enabled?: boolean;
  select?: (data: Page<PlayerDTO>) => TData;
})
```

— fetching the full dataset once (`{ size: 10000 }`, since the backend does not support
`sort` and the app pages/sorts client-side — see [Players feature doc](../features/players.md)),
with every consumer (the players page, dashboards' player counts, match-creation rosters,
match-plan overrides) deriving its own shape via `select` instead of issuing its own
differently-parameterized query. Two consequences:

- A player mutation invalidates one cache entry (`playerKeys.all`), so it triggers exactly
  one refetch instead of a refetch per previously-fragmented key variant.
- `enabled` lets a consumer skip the fetch entirely when the data can never be used — e.g. a
  BASIC-role user viewing a modal that gates the roster behind `canOverride`.

Adopt this shape (canonical key, `select` for view-specific derivation, `enabled` for
role/permission gating) for any new feature that would otherwise need several near-duplicate
list queries over the same backend resource.

## Auth Pattern: sessionStorage persistence + `AuthGuard` (interim, BFF deferred)

The backend issues a JWT in the login response body with no `Set-Cookie` / httpOnly-cookie
support (see `FRONTEND_ENDPOINT_CHANGES.md` in the backend docs — integration notes below).
Given that constraint, auth state is a Zustand `persist` slice of `useAppStore`
(`src/store/appStore.ts`) written to `sessionStorage` (locale/theme persist separately to
`localStorage`, via one custom storage adapter splitting both under a single `persist()`
call — see the file's own comments for why two nested `persist()` calls don't work with
zustand v5). This is a **deliberate interim decision**: a follow-up is expected to introduce
a BFF route handler that mints a real httpOnly cookie once/if the backend supports it. Until
then:

- `skipHydration: true` on the persist config avoids a server/client hydration mismatch —
  the store never reads `sessionStorage` during the first render.
- A `hasHydrated` flag flips to `true` once `useAppStore.persist.rehydrate()` (triggered once
  from `ThemeProvider` on mount) resolves.
- `src/components/auth/AuthGuard.tsx` — `<AuthGuard requiredRole?>` — centralizes the
  route-guard logic that used to be copy-pasted as a `useEffect` + spinner in every
  `(app)` route's `page.tsx`. It blocks render (and redirect decisions) until `hasHydrated`
  is `true`, then redirects to `/login` if unauthenticated, or `/dashboard` if authenticated
  but missing `requiredRole`. **In practice it is still mounted per-route** (each
  `src/app/(app)/<route>/page.tsx` wraps its feature page in `<AuthGuard>`), not once in the
  shared `(app)/layout.tsx` — the win is one reusable component instead of seven duplicated
  guard implementations, not fewer mount points.

Integration reference: `C:\Users\ricar\IdeaProjects\football\docs\frontend\FRONTEND_ENDPOINT_CHANGES.md`
documents the "store token in secure storage (httpOnly cookie or memory — avoid
localStorage)" guidance this decision is answering, and the `Authorization: Bearer` contract
`apiFetch` implements.

## Code-Splitting: `next/dynamic` for role-gated / click-triggered UI

Heavy, role-gated, or click-triggered feature UI is loaded via `next/dynamic(..., { ssr:
false })` rather than statically imported into a shared page bundle — the pattern
originates in `src/components/auth/LoginFormLazy.tsx` and is now also used for:

- The three role dashboards (`src/app/(app)/dashboard/page.tsx` dynamically imports
  `AdminDashboard` / `MasterDashboard` / `BasicDashboard` — only one is ever rendered per
  user, so the other two roles' code never ships to that user).
- `MatchModal`, `CreateMatchModal`, `CreateManualMatchModal`, `RecalculateMatchesPanel`
  (`src/features/matches/MatchesPage.tsx`) — all click-triggered and/or admin/master-only.
- `DraftSessionsAdminPage` (`src/features/teamGeneration/TeamGenerationPage.tsx`) —
  admin-only.

`MatchModal` itself was also split out of a single 800-line file into
`src/features/matches/matchModal/{MatchModal,TeamScoresheet,TeamsRosterView,RecordMatchForm}.tsx`
plus a colocated `utils.ts`, so the dynamically-imported chunk isn't itself a monolith.

## Styling Contract

- **Zero** Tailwind utility classes inside `.tsx` files
- All styles defined in `src/app/globals.css` under `@layer components`
- Class names follow BEM-inspired convention: `block__element--modifier`
- See [Styling Guide](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/main/docs/guides/styling.md) for the full rule set, including the shared
  `.page-container` width contract and the actual (non-bottom-sheet) modal pattern.

## Version Source

The canonical app version is owned by the backend (`GET /api/version`).  
`package.json` and `CHANGELOG.md` mirror it — see [Version Updater agent](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/main/.github/agents/version-updater.agent.md).
