# Push Notifications

**Added in:** v1.0.0 (Roadmap Phase 2)
**Status:** ✅ Implemented
**Backend contract:** `docs/api/PUSH-API-CONTRACT.md` in the backend repo

Web Push over the raw protocol — no Firebase, no vendor SDK. The backend encrypts per
subscription (RFC 8291) and signs with VAPID (RFC 8292); this side registers the browser,
renders the notification, and owns the preferences UI.

---

## Files

```
src/lib/pushSupport.ts                        Browser plumbing + environment store
src/services/pushService.ts                   API calls
src/hooks/push/usePushNotifications.ts        React Query wrapper + state
src/features/settings/NotificationSettings.tsx  The UI
src/types/push.ts                             DTOs
public/sw.js                                  push + notificationclick handlers
```

Mounted from `app/(app)/dashboard/page.tsx`, below the role dashboard — the settings are
identical for all three roles, so putting them inside each would be three places to keep in
step.

---

## The iOS rule that shapes everything

**Safari only delivers web push to a PWA that has actually been added to the home screen**
(16.4+). In an ordinary iOS tab, `Notification.requestPermission()` either throws or resolves
and then silently never delivers anything.

So `canUsePush()` returns `false` for iOS-not-installed, and the UI shows an explanation
pointing at installation rather than a control. Offering the toggle there would produce a
switch that appears to work and does nothing — the worst of both outcomes.

This is the concrete payoff of Phase 1's install UX: `IosInstallHint` is what gets those users
to a state where push can work at all.

**Every other platform** — Android, desktop Chrome, Edge, Firefox — works in a normal tab, and
the iOS restriction must not leak into them. `pushSupport.test.ts` asserts that explicitly.

---

## Why the environment is an external store, not state

`canUsePush()`, `isIos()`, `isStandalone()` and `Notification.permission` all read
`navigator`/`window`, which do not exist during prerender. Copying them into state inside a
`useEffect` would mean a `setState` in an effect body — which this project's React Compiler
lint rejects — and would risk a hydration mismatch.

`pushSupport.ts` therefore exposes them through `useSyncExternalStore`, with a distinct server
snapshot (`ready: false`) so SSR and the first client render agree by construction. Same
pattern as `IosInstallHint`.

`refreshPushEnvironment()` re-reads it after the permission prompt — the one thing that changes
the environment mid-session.

The snapshot is **cached**: `getSnapshot` must return a referentially stable value or React
compares with `Object.is`, sees a fresh object every call, and re-renders forever.

Only `subscribedEndpoint` is real state, because answering it means asking the browser
asynchronously. It is set from a promise callback rather than synchronously in the effect body,
which is what keeps it clear of the same lint rule.

---

## Subscribing

```
GET /api/push/public-key   → { enabled, publicKey }
        ↓  (user gesture only)
Notification.requestPermission()
        ↓  granted
pushManager.subscribe({ userVisibleOnly: true, applicationServerKey })
        ↓
POST /api/push/subscribe   { endpoint, p256dh, auth, deviceLabel }
```

Three things that bite:

- **`applicationServerKey` must be a `Uint8Array`**, not the base64url string the API returns.
  `urlBase64ToUint8Array` also translates the URL-safe alphabet (`-`/`_`), which a plain
  `atob()` chokes on.
- **It must be backed by a plain `ArrayBuffer`.** Since TypeScript 5.7 the typed arrays are
  generic over their buffer, and `Uint8Array.from()` produces `Uint8Array<ArrayBufferLike>` —
  which could be a `SharedArrayBuffer` and is therefore not a valid `BufferSource`. The helper
  allocates an explicit `ArrayBuffer`; a test asserts it.
- **Permission must come from a user gesture.** Requesting on load is ignored by some browsers
  and shows a hostile prompt in others. There is deliberately no automatic subscribe anywhere.

`userVisibleOnly: true` is required — Chrome rejects `false`. It is the promise that every push
results in something the user can see.

### The gesture has to survive as far as the actual call

`requestNotificationPermission()` is invoked in the **synchronous prefix of the click handler** —
`usePushNotifications` fires it while building the mutation's argument, not inside `mutationFn`:

```ts
const subscribeCallback = useCallback(
  () => subscribeMutate(requestNotificationPermission()),
  [subscribeMutate],
);
```

Safari only honours a permission request while the tap is still active, and is stricter about it
than Chrome. Anything awaited beforehand spends the gesture — a round trip for the VAPID key, or
merely the promise indirection inside TanStack Query before it reaches `mutationFn`. Get this
wrong on iOS and **the prompt never appears, with no error to explain why**, which reads exactly
like "iOS does not support this".

The permission decision is therefore a *parameter* of `createSubscription` rather than something
it asks for. `pushSupport.test.ts` pins the property by asserting the browser call has already
landed without flushing a single microtask.

### Re-subscribing is cheap and expected

The backend upserts on `endpoint`, so sending the same subscription again is safe. Worth doing
on login: endpoints rotate, and on a shared browser the row needs reassigning to whoever is now
signed in.

---

## Unsubscribing

Tell the backend **before** calling `subscription.unsubscribe()` — afterwards the endpoint is
gone and there is nothing left to identify the row with. A browser-side unsubscribe alone
leaves a row the server keeps trying until the push service reports it dead.

---

## Preferences

Rendered from the server's `categories` array, never a hard-coded list, so a category added on
the backend appears here automatically and already switched on. **Absence from `muted` means
on** — there is no third state.

Category labels fall back to the raw name (`t(key, category)`) so a new category that lands
before its translation renders as `MATCH_REMINDER` rather than an empty row.

The toggles shipped with the enable control rather than after it, which the roadmap is explicit
about: retrofitting preferences once people are already over-notified is harder and later.

---

## Service worker

`push` and `notificationclick` in `public/sw.js`. **`VERSION` was bumped to `v3`** — without
that, existing installs keep the old worker and never gain the handlers.

The handler runs with **no application code loaded** — no React, no router, no store — so the
payload carries everything needed: `{ category, title, body, url }`.

- **A push with no data is ignored.** Some services send one purely to wake the worker.
- **Unparseable JSON shows nothing**, rather than a notification built from garbage.
- **`tag: category`** collapses repeats of the same kind instead of stacking them. Three "it's
  your pick" notifications in a row is the fastest route to a muted channel.
- **`notificationclick` focuses an existing window** and navigates it, rather than opening a
  second tab. `includeUncontrolled: true` matters: without it a window not yet controlled by
  this worker is invisible and you get a duplicate tab beside the app already open.

---

## Testing it end to end

The service worker's push path cannot be exercised without a real push service. To check it
locally:

1. Backend: `./gradlew generateVapidKeys`, set the three env vars, restart.
2. Frontend: `npm run build && npm start` — the worker does not register in development.
3. Enable notifications on the dashboard, then trigger something: complete a match, or make a
   draft pick so the next captain is on the clock.
4. DevTools → Application → Service Workers → **Push** sends a synthetic payload without any
   backend at all. Paste `{"category":"MATCH_REMINDER","title":"Test","body":"Hello","url":"/match-plans"}`.

Step 4 is the quickest way to iterate on the handler itself.

---

## Tests

| File | Covers |
|------|--------|
| `src/tests/lib/pushSupport.test.ts` | base64url decoding incl. the `ArrayBuffer` requirement, iPhone/iPad-as-Macintosh/desktop-Mac/Android detection, that the iOS-needs-install rule does not leak to other platforms, the synchronous permission call, and the **installed-iOS subscribe path** end to end — `userVisibleOnly`, the `Uint8Array` key, DTO shape, existing-subscription reuse and rejection of an incomplete subscription |
| `src/tests/components/NotificationSettings.test.tsx` | Renders nothing before the environment is known, each unavailable reason, toggles rendered from the server list, absence-from-muted-means-on, subscribe/unsubscribe wiring, refused permission, and per-category busy state |

`public/sw.js` has no unit tests, for the reason given in [pwa.md](pwa.md) — it is verified in
DevTools and by the browser-driven check described there.
