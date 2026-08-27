# Push Notifications

**Added in:** v1.0.0 (Roadmap Phase 2) · **account-scoped since 2.2.0**
**Status:** ✅ Implemented — ⚠️ 2.2.0's changes are cut on `next` and **not deployed**
**Backend contract:** `docs/api/PUSH-API-CONTRACT.md` in the backend repo

Web Push over the raw protocol — no Firebase, no vendor SDK. The backend encrypts per
subscription (RFC 8291) and signs with VAPID (RFC 8292); this side registers the browser,
renders the notification, and owns the preferences UI.

---

## Files

```
src/lib/pushSupport.ts                        Browser plumbing + environment store
src/lib/pushSignOut.ts                        Releases the channel at sign-out
src/services/pushService.ts                   API calls
src/hooks/push/usePushNotifications.ts        React Query wrapper + state
src/hooks/push/usePushDeviceClaim.ts          Whose channel is this? — asks the server
src/components/pwa/PushDeviceClaim.tsx        Mounted app-wide from the root layout
src/features/settings/NotificationSettings.tsx  The UI
src/types/push.ts                             DTOs
public/sw.js                                  push + notificationclick handlers
```

The settings UI is mounted from `app/(app)/dashboard/page.tsx`, below the role dashboard — the
settings are identical for all three roles, so putting them inside each would be three places to
keep in step.

**`PushDeviceClaim` is mounted from the root layout instead, and that placement is load-bearing.**
The claim is what takes the channel off whoever held it previously, so it has to run at sign-in.
Deferring it until somebody opened settings would leave a previous user's notifications arriving on
the current user's screen for as long as nobody visited that page — which is indefinitely.

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

The backend upserts on `(user_id, endpoint)`, so sending the same subscription again is safe.

---

## The channel is the browser's; the registration is the account's

**This is the distinction the feature got wrong until 2.2.0, and it is worth stating before
anything else on this page.** A browser mints **one push endpoint per origin** and hands back the
same one whoever is signed in. So the endpoint identifies the *channel* — a browser install — and
not the *registration*, which is a per-account fact. `V12` keyed the row on `endpoint` alone and
conflated the two.

On any device that saw two accounts, that had two consequences:

- The row kept pointing at whoever registered first, so **their notifications were delivered to and
  displayed on a device somebody else was now signed in on.** The service worker renders whatever
  arrives: it has no session and cannot check.
- The second account **had no registration and no way to get one.** This side asked the browser
  *"is there a subscription?"* rather than the server *"is it mine?"*, so the toggle already read
  **on**, `/subscribe` was never called, and nothing was ever received.

`V42` keys a registration on `(user_id, endpoint)` with a partial unique index allowing at most one
**active** row per endpoint, and only active rows are sent to. Ownership still moves — there is
physically one channel per browser — but the displaced account keeps a **dormant** row rather than
being erased, so switching back to it on that device restores the choice instead of asking for
consent a second time. A dormant row is not a quieter subscription; it is not a subscription.

### Ownership is a server fact, so it is read from the server

`usePushDeviceClaim` POSTs `/api/push/claim`, which binds the channel to the caller and reports
whether they are subscribed. **It never creates a registration** — claiming and subscribing are
different acts, and merging them would opt somebody in by signing in.

The answer is cached **per account**. A shared cache key would reproduce the original bug one layer
up: two accounts on one device would agree on an answer that is true for only one of them.

The claim also closes the leak when nobody signs out cleanly — tokens expire, tabs get closed —
because *arriving* at a device takes the channel off whoever last held it, without needing the
previous session to have done anything.

`DELETE /api/push/claim` releases it at sign-out, dormant rather than deleted. That call is capped
at 2.5s and swallows failures: somebody handing their phone over has to be able to log out on a
flaky connection, and the claim at the next sign-in reconciles ownership anyway. **The browser
subscription is deliberately not torn down** — notification permission is a device-level grant, and
re-prompting risks a denial, which is permanent.

### The toggle has three states, not two

Until the server has answered, it says **"Checking this device"** rather than asserting **Off**.
Asserting Off would flicker for somebody who has it on, and — worse — invite them to press a
control that would then do nothing they expected.

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

`push` and `notificationclick` in `public/sw.js`. **`VERSION` is `v4`** (bumped from `v3` in
2.2.0) — without a bump, existing installs keep the old worker and never gain what changed.

The handler runs with **no application code loaded** — no React, no router, no store — so the
payload carries everything needed: `{ category, title, body, url, when }`.

**`when` is new in 2.2.0, and it exists because the server has no timezone it could correctly
pick** — the schema stores none, anywhere. `MATCH_REMINDER` used to concatenate
`Instant.toString()` into the body, so people were reading `2026-07-31T19:00:00Z` on a lock screen.
The instant now travels in the payload and the worker renders it as `DD-MON-YYYY HH:mm` in the
device's own zone.

Two decisions in that rendering are worth keeping:

- **Built from date parts rather than `toLocaleString`**, so the format is fixed instead of varying
  with the browser's locale.
- **Appended to the body rather than substituted into a placeholder.** Workers update on their own
  schedule, so an older worker that does not know about `when` will be receiving these payloads for
  a while. Appending means it drops the time; substituting would have it render a token literally.

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
| `src/tests/hooks/usePushDeviceClaim.test.tsx` | The claim is per-account, the cache key is not shared between accounts, and the toggle's third state while the server has not answered |
| `src/tests/lib/pushSignOut.test.ts` | Release before the token is cleared, the 2.5s cap, failures swallowed, and that the browser subscription is left in place |
| `src/tests/lib/serviceWorkerPush.test.ts` | `sw.js`'s first tests, **run against the real shipped file** rather than a copy — `when` rendering, the fixed date format, and that a payload without `when` still produces a notification |

`public/sw.js` had no unit tests until 2.2.0, for the reason given in [pwa.md](pwa.md). It now has
`serviceWorkerPush.test.ts`, which loads the shipped file rather than a fixture — the DevTools and
browser-driven checks described there are still what covers the parts a test cannot reach.
