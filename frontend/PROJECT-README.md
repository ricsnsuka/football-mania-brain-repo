# FootballMania — Frontend

The web client for the Football Management System: a **Next.js 16 / React 19** PWA in TypeScript.

Talks to the Spring Boot API in the [`football`](../../../football) repository. **The two are
separate git repositories, not a monorepo** — an API change and its frontend change are two commits
in two places.

## Getting started

```bash
npm install
npm run dev          # http://localhost:3000
```

The API is expected at the base URL in `.env.local` (`NEXT_PUBLIC_API_URL`). Start the backend with
`./gradlew bootRun --args='--spring.profiles.active=dev'` in the other repo.

> **Push notifications do not work under `npm run dev`.** Next's dev server never registers the
> service worker, so `navigator.serviceWorker.ready` never settles and subscribing hangs with no
> error. Test push against `npm run build && npm start`. This costs more time than it should if you
> do not know it — see [push-notifications](docs/features/push-notifications.md).

## Scripts

| Command | What |
|---------|------|
| `npm run dev` | Dev server, Turbopack |
| `npm run build` / `npm start` | Production build and serve |
| `npm test` | Vitest — 423 unit tests |
| `npm run test:coverage` | Coverage report |
| `npm run test:visual` | Playwright screenshots against the committed baselines |
| `npm run test:visual:update` | Accept the current appearance as the new baseline |
| `npm run lint` | ESLint |
| `npm run type-check` | `tsc --noEmit` |

`test:visual` builds and serves the app itself on **port 3100**, so it never fights a dev server on
3000. First run takes a couple of minutes because of the production build.

## Stack

| Concern | Choice |
|---------|--------|
| Framework | Next.js 16 (App Router) |
| UI | React 19, Tailwind CSS v4 |
| Server state | TanStack Query |
| Client state | zustand, with a splitting storage adapter (auth → `sessionStorage`, preferences → `localStorage`) |
| Forms | react-hook-form + zod |
| i18n | i18next — English, Portuguese, Spanish |
| Tests | Vitest + Testing Library; Playwright for visual regression |

## Two things that will bite you

**The API omits nullable fields — it never sends `null`.** The backend sets
`spring.jackson.default-property-inclusion: non_null`, so an absent value is *missing from the JSON*,
not present-and-null. In TypeScript it arrives as `undefined`, and `x === null` is false for all of
them. Types use the `nullableField` helper (`src/types/api.ts`) to normalise absent → `null` at the
parse boundary. Branch on booleans (`overridden`, `resolved`, `qualified`) where the API provides
them. This has been got wrong more than once.

**Server-supplied limits are not constants.** The ranking qualification threshold, the MOTM voting
window and the leaderboard sizes are admin-configurable. Read `minimumMatchesToQualify`,
`mvpVotingClosesAt` and `limit` from responses rather than hard-coding 3, 24h and 5.

## Layout

```
src/
  app/                  ← App Router. (app) = authenticated shell, (auth) = login
  features/             ← One folder per feature: dashboard, matches, players,
                          rankings, settings, teamGeneration, matchPlans, users
  components/           ← Shared UI, layout, auth guards
  hooks/                ← One folder per domain; queries and mutations
  services/             ← apiFetch wrappers, one per resource
  types/                ← DTOs + zod schemas, validated at the fetch boundary
  store/                ← zustand
  locales/{en,pt,es}/   ← i18n
e2e/                    ← Playwright visual regression + committed baselines
docs/                   ← Feature and guide docs
```

## Docs

Start at [`docs/INDEX.md`](docs/INDEX.md).

- [Styling](docs/guides/styling.md) — semantic tokens, the seven rules, how to verify a change
- [Shared components](docs/guides/shared-components.md)
- [Component conventions](docs/guides/component-conventions.md)
- [i18n](docs/guides/i18n.md)
- Features: [dashboard](docs/features/dashboard.md), [settings](docs/features/settings.md),
  [rankings](docs/features/rankings.md), [MOTM voting](docs/features/motm-voting.md),
  [badges](docs/features/badges.md), [players](docs/features/players.md),
  [PWA](docs/features/pwa.md), [push](docs/features/push-notifications.md)

## Visual regression

Screenshots of every page in both themes, plus the player modal and the admin System section,
compared byte-for-byte against committed baselines.

Baselines are **platform-specific** (`-win32`). Running on Linux or macOS generates its own set
rather than failing, so CI on another OS needs its own generated once.

`maxDiffPixels: 20`, and that number is measured rather than guessed — an unchanged page reproduces
its baseline at exactly 0 differing pixels. If you change it, re-validate the way
[`e2e/README.md`](e2e/README.md) describes: a threshold nobody has probed is a guess. The previous
setting allowed ~11,500 pixels and let a replaced page heading through as green.

## Roles

`BASIC_USER` · `MASTER_USER` · `ADMIN_USER`

Role gates in the UI decide what is *rendered*; the API decides what is *permitted*. Some routes are
open while their nav entry is hidden — `/players` is deliberately reachable by a bookmark even
though `BASIC_USER` is not offered the link. Hiding a nav entry is a usability decision and has
never been a security boundary.
