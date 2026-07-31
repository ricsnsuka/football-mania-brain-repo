# Progressive Web App (PWA)

**Added in:** v1.0.0 (Roadmap Phase 1)
**Status:** ✅ Installable, offline-capable shell

Phase 1 of the [mobile roadmap](../Football%20App%20-%20Improvement%20and%20Mobile%20Roadmap.md):
make the app installable to a home screen and survive losing the network, with no app-store
dependency and no change to any existing page.

---

## What this gives you

| Capability | How |
|------------|-----|
| Install to home screen / desktop | `src/app/manifest.ts` |
| Runs full screen, no browser chrome | `display: 'standalone'` |
| App icon in tab, launcher and task switcher | `src/app/icon.svg`, `src/app/apple-icon.png`, `public/icons/*` |
| Static assets served with no network | `public/sw.js` |
| A real page instead of the browser error when offline | `src/app/offline/page.tsx` |
| **Previously loaded data readable offline** | `src/lib/queryPersistence.ts` |
| **The user is told how stale that data is** | `src/components/pwa/OfflineBanner.tsx` |
| iOS users can discover installation at all | `src/components/pwa/IosInstallHint.tsx` |

---

## Files

```
src/app/
  icon.svg                      ← the mark; source of truth for every raster size
  apple-icon.png                ← 180×180, generated
  manifest.ts                   ← web app manifest (a route, not a static file)
  offline/page.tsx              ← offline fallback
src/components/pwa/
  ServiceWorkerRegistrar.tsx    ← registers the worker; renders nothing
  IosInstallHint.tsx            ← Safari-only "Add to Home Screen" banner
  OfflineBanner.tsx             ← "Offline — showing data from HH:MM"
src/lib/
  queryPersistence.ts           ← persister, erase-on-auth-change plumbing
src/hooks/
  useOnlineStatus.ts            ← online/offline via React Query's onlineManager
public/
  sw.js                         ← the service worker
  icons/icon-192.png            ← generated
  icons/icon-512.png            ← generated (also serves as the maskable icon)
scripts/
  generate-icons.mjs            ← npm run icons
```

---

## Icons

`src/app/icon.svg` is the only artwork file. Everything else is generated:

```bash
npm run icons
```

To rebrand, replace that one SVG and re-run — nothing else references the artwork. The mark
is a white football on `blue-600` (`#2563eb`), the app's existing primary. It carries its own
background rather than being transparent, so it never disappears against a dark tab strip or
launcher, and the ball occupies the central ~60% of the canvas so Android's maskable crop
(which keeps only the central 80%) cannot clip it.

**Next.js emits the icon `<link>` tags automatically** from the `app/icon.*` and
`app/apple-icon.*` file conventions. Do not add `<link rel="icon">` to `layout.tsx` — it
duplicates what the framework already produces.

---

## Manifest

`src/app/manifest.ts` returns a typed `MetadataRoute.Manifest` and is served at
`/manifest.webmanifest` with `Content-Type: application/manifest+json`. Next emits the
`<link rel="manifest">` tag.

### theme_color vs viewport themeColor — they differ on purpose

Two similar-looking colours that do different jobs:

| Setting | Where | Value | What it tints |
|---------|-------|-------|---------------|
| `theme_color` | `app/manifest.ts` | `#2563eb` | The **installed** app's status bar and task-switcher chrome |
| `themeColor` | `viewport` export in `app/layout.tsx` | `#ffffff` light / `#0a0a0a` dark | The browser address bar while merely **visiting** the site |

The manifest accepts a single colour, so it uses the brand blue that matches the icon. The
meta tag can vary by colour scheme, so it matches the page background and the chrome blends
into the page. `src/tests/app/manifest.test.ts` asserts both.

> ⚠️ **`themeColor` belongs in the `viewport` export, not `metadata`.** It was deprecated in
> `metadata` as of Next.js 14. Putting it back in `metadata` produces a build warning and no
> meta tag.

---

## Service worker

Hand-rolled, ~100 lines, no dependency. The roadmap warned that PWA plugins lag major Next
releases, and that held up: `next-pwa` has been unmaintained since the Next 12 era, and the
Next.js PWA guide notes Serwist "currently requires webpack configuration" — this app builds
with Turbopack. Revisit if the caching needs outgrow the file.

### Caching strategy

| Request | Strategy | Why |
|---------|----------|-----|
| Cross-origin (**the whole API**) | Not handled at all | See below |
| Non-`GET` | Not handled | Never cache a mutation |
| Navigations | Network first → cache → `/offline` | Always current when online; never the browser's error page when not |
| `/_next/static/*`, `/icons/*` | Cache first | Content-hashed, so the filename changes whenever the content does |
| Other same-origin `GET` | Network first → cache | Manifest and stray static files |

### API responses are never in the *service worker* cache

The roadmap specifies "network-first for API calls". API responses are instead cached one
layer up, in React Query (next section), and the service worker never touches them at all.

The reason is that a Cache API entry is keyed by URL and served transparently to any fetch
from that origin, and **every API response here is authenticated and carries personal data** —
names, phone numbers, full match history. Those entries survive logout with nothing in the app
aware they exist. React Query's cache is different in the way that matters: it is only ever
read back into a `QueryClient` that the app owns and erases on every auth transition.

The API is also on a different origin (`NEXT_PUBLIC_API_URL`), so cross-origin requests are
passed straight through by the fetch handler. `src/tests` cannot prove this, so the browser
verification below asserts that no `/api/` entry appears in any cache.

---

## Offline data: persisted React Query cache

`PersistQueryClientProvider` writes the query cache to `localStorage` and restores it on
launch, so an installed app has something to show before its first successful request.

### Storage lifetime, and what pays for it

The auth slice deliberately uses `sessionStorage` so a session ends with the tab. The query
cache uses **`localStorage`** instead, because doing the same would empty it on every launch of
an installed app — precisely when offline data is needed. The cache therefore outlives the
session, and the erasures are what make that safe rather than careless:

| Trigger | Where |
|---------|-------|
| Logout, and any 401 | `clearAuth` in `appStore.ts` |
| Login | `setAuth` — so one person's cache can never be rehydrated into another's session |
| 24 hours elapsed | `maxAge` on the persist client |

The erasure lives **in the store**, not at the four `clearAuth` call sites, so a future call
site cannot forget it. `appStore` cannot import a `QueryClient` — it is a module singleton and
the client is created inside a provider — so `QueryProvider` registers an erase callback with
`lib/queryPersistence.ts` on mount and the store invokes it through that indirection.
`eraseQueryCache` also removes the storage key directly, covering a 401 that arrives before
the provider's effect has run.

### What is not persisted

- **Errored and pending queries.** A restored error would render last session's failure as
  though it had just happened; a restored pending state would show a spinner with nothing in
  flight to resolve it. `shouldPersistQuery` keeps only `success`.
- **Mutations.** `shouldDehydrateMutation: () => false`. Rehydrating one would leave a
  completed write looking pending, and a paused one could fire against a different session.

`gcTime` is raised to match `maxAge`: at the default five minutes, a restored query is already
older than its garbage-collection threshold and is collected before anything renders it — the
cache would persist perfectly and appear to do nothing.

### Saying how stale it is

`OfflineBanner` renders under the navbar while offline, reading the freshest `dataUpdatedAt`
across the cache: *"Offline — showing data from 14:32"*. Restoring the cache is what makes
stale data possible, so the banner is not optional decoration — serving a stale player rating
that looks live is the specific failure the roadmap warns about, and the answer is to say how
old it is rather than to withhold it. Online, React Query refetches on its own and staleness is
measured in seconds, so the banner does not render.

`useOnlineStatus` reads React Query's `onlineManager` rather than `navigator.onLine`, so the UI
and the data layer never disagree about connectivity.

### Updating the worker

Bump `VERSION` in `public/sw.js` on any change to that file or to `PRECACHE_URLS`. The
activate handler deletes every cache whose name does not carry the current version, so that
constant is the only cache-busting lever.

The worker does **not** call `skipWaiting()` on install. Activating immediately would swap the
worker out from under pages that are already open. A new worker takes over once every tab has
closed — or when the page posts `{ type: 'SKIP_WAITING' }`, which is the hook for an "update
available" prompt if one is added later (`NotificationWidget` is the natural place).

### Not registered in development

`ServiceWorkerRegistrar` returns early unless `NODE_ENV === 'production'`. A worker caching
Turbopack's dev chunks makes hot reload behave unpredictably, and every stale-asset bug then
looks like an app bug. **To test the worker, run a production build:**

```bash
npm run build && npm start
```

Service workers require HTTPS or `localhost`; `localhost` is exempt, so this works locally.

---

## iOS: the reason there is no install button

Chrome and Edge fire `beforeinstallprompt` and render their own install affordance. Safari
fires nothing and exposes no API — installing is Share → "Add to Home Screen", undiscoverable
unless the page says so.

The Next.js PWA guide explicitly advises **against** building a custom install button on
`beforeinstallprompt`, because it is not cross-browser and does nothing on Safari iOS. So this
app doesn't: Chrome's built-in prompt is left alone, and only the platform with no prompt at
all gets bespoke UI.

`IosInstallHint` renders a dismissible banner inside the authenticated shell when **all** of:

- the device is iOS — including iPadOS 13+, which reports a *Macintosh* user agent and is
  distinguishable only by `navigator.maxTouchPoints > 1`;
- the app is not already installed (`display-mode: standalone`, or the legacy
  `navigator.standalone`);
- it has not been dismissed.

It is built on `useSyncExternalStore` rather than `useState` + `useEffect`. Those signals are
all external to React, and the server snapshot (`false`) means SSR and the first client render
agree, so there is no hydration mismatch and no `setState`-in-effect — which the project's
React Compiler lint rule rejects outright.

Dismissal persists in `localStorage`, with an in-memory fallback for Safari private browsing,
where storage throws. Without that fallback the failed write would leave the snapshot
reporting "not dismissed" and the button would be inert for the whole session.

> **This gates push.** iOS only delivers web push to PWAs actually added to the home screen
> (Safari 16.4+), so push adoption on iPhone depends on users getting through this flow — see
> [push-notifications.md](push-notifications.md), where the same detection decides whether the
> enable control is offered at all.

---

## Security headers

`next.config.ts` adds a `/sw.js` entry after the catch-all so its narrower headers win:

| Header | Value | Why |
|--------|-------|-----|
| `Content-Type` | `application/javascript; charset=utf-8` | Browsers refuse to register a worker served as anything else |
| `Cache-Control` | `no-cache, no-store, must-revalidate` | A worker served from the HTTP cache keeps serving stale assets after a deploy — the failure mode that makes people distrust PWAs |
| `Content-Security-Policy` | `default-src 'self'; script-src 'self'` | The worker needs no inline script, so it does not inherit the page's `'unsafe-inline'` |

`updateViaCache: 'none'` at the registration site says the same thing as `Cache-Control` from
the client. Both are worth having because either alone can be overridden by a CDN.

The page CSP gained `worker-src 'self'` and `manifest-src 'self'`. Both would have fallen back
to existing directives, but `worker-src` would inherit `script-src`'s `'unsafe-inline'`, and a
manifest that silently fails to load disables installability with no visible symptom.

---

## Verification

### What has been verified

Driven against a production build with real Chrome over the DevTools Protocol, 8/8:

| Check | Result |
|-------|--------|
| Worker registers and reaches `activated` | ✅ scope `/` |
| Page is controlled after one reload | ✅ |
| Precache contains `/offline` | ✅ `fm-precache-v2` |
| Immutable assets cached at runtime | ✅ 24 `/_next/static` entries |
| **No `/api/` entry in any cache** | ✅ |
| A previously visited route loads with the server down | ✅ |
| An uncached route is served the offline page | ✅ |

Lighthouse against the same build: **Performance 92–96, Accessibility 100, Best Practices 93,
SEO 100**. Performance varies by a few points between runs on an identical build; treat a
single number as noise.

`errors-in-console` is the one audit still red, and its only remaining entry is
`ERR_CONNECTION_REFUSED` for `localhost:8080/api/version` — the backend was not running
during the audit. It should clear with the API up.

Three things that verification exposed, recorded because they are easy to get wrong again:

1. **`page.setOfflineMode()` does not make a service worker offline.** CDP network emulation
   applies to the *page* target; the worker's `fetch()` runs in its own target and keeps
   reaching the network. Every "offline" assertion passes while fully online. The only
   unambiguous test from outside a browser is to stop the server.
2. **A user gets no offline support on their first visit.** The worker installs during that
   load, so the requests that built the page were already in flight and never reached its
   fetch handler. From the second load on, everything is cached. Eliminating this would mean
   injecting a build-time asset manifest into the worker, which is the main thing Workbox
   does for you — deliberately not done here.
3. **Next prefetches every navbar link**, so in practice all app routes land in the runtime
   cache without being visited. That is why the app browses offline at all, and why the
   offline-page fallback needs a deliberately evicted entry to observe.

> The verification script is not committed: it lives outside the repo and needs
> `puppeteer-core` plus a Chrome path. Re-create it from this table if it is needed again, or
> use the manual checklist below.

### Manual checklist

Lighthouse **removed its PWA category in v12**, so there is no longer a "PWA audit" score to
run. The checks that matter are now in **Chrome DevTools → Application**:

1. **Manifest** — no errors, icons resolve, "Installable" shown.
2. **Service Workers** — status `activated and is running`.
3. **Cache Storage** — `fm-precache-v1` contains `/offline`.
4. Throttle to **Offline** and reload — the shell loads; a route with no cached copy shows the
   offline page rather than the browser's error page.
5. Install from the address bar, confirm it opens with no browser chrome.
6. On a real iPhone (the simulator does not install PWAs): confirm the hint appears, then
   Share → Add to Home Screen.

Mechanically verifiable without a browser:

```bash
npm run build && npm start
curl -sI localhost:3000/sw.js | grep -i 'content-type\|cache-control'
curl -s  localhost:3000/manifest.webmanifest
curl -s  localhost:3000/login | grep -o '<link rel="manifest"[^>]*>'
```

---

## Tests

| File | Covers |
|------|--------|
| `src/tests/app/manifest.test.ts` | Installability criteria: `standalone`, `start_url`, `short_name` length, 192/512 icons, a maskable icon, and the theme-colour split above |
| `src/tests/app/offline.test.tsx` | Offline page renders, retry reloads, and offers no links (every in-app route needs the network the user has just been shown to lack) |
| `src/tests/components/IosInstallHint.test.tsx` | iPhone, iPad-as-Macintosh, desktop Mac, Android, already-installed, dismissal persistence, and both private-browsing paths |
| `src/tests/lib/queryPersistence.test.ts` | What is and is not persisted, erasure through the registered callback, erasure before any provider has mounted, storage-unavailable path, and the 24-hour bound |
| `src/tests/components/OfflineBanner.test.tsx` | Hidden online, names a timestamp when there is cached data, says so when there is none, uses the freshest entry, announces politely |

`public/sw.js` has no unit tests. It is a browser worker with no module boundary, and mocking
`caches`/`fetch`/`ServiceWorkerGlobalScope` well enough to be meaningful would test the mocks.
It is covered by the browser verification above instead — which is how the one real defect in
it was found: the navigation branch originally cached responses without checking status, so a
404 or 500 would be persisted and then replayed from cache on the next offline visit, turning
a transient server error into a page that looked permanently broken.

## Fixed along the way: the React #418 hydration mismatch

The production build used to log **React error #418** on every first load, and it predated
this work — it reproduced identically on the pre-PWA commit `c9c7876`.

The cause was i18n, not anything PWA-related. Pages are prerendered at build time, where
i18next resolves its dynamic locale import and emits translated HTML; on the client that
import is still in flight during hydration, and with `useSuspense: false` the first client
render produces raw keys instead. React found `footer.theme` where the server had written
`Theme`. Users saw the same thing as a flash of untranslated keys on first paint.

Fixed by bundling `en/common.json` rather than fetching it — see
[the i18n guide](../guides/i18n.md#loading-strategy--en-is-bundled-the-rest-are-lazy), which
also covers the `partialBundledLanguages` flag that keeps `pt`/`es` lazy.

Worth recording how it was found, because the obvious approaches did not work: it does not
reproduce under `next dev` (which renders per request rather than prerendering), and the
production React build only emits a minified error code with no node. Diffing the text nodes
of the server DOM (loaded with JavaScript disabled) against the hydrated DOM located it in
one pass.
