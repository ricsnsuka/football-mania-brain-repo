# Push Notifications — API Contract

**Date:** 2026-07-29
**Version:** v1.0.0 (Roadmap Phase 2)
**Status:** APPROVED — backend complete (subscriptions, send path, all five triggers).
**Delivery observed end to end on 2026-07-29**: a `MATCH_COMPLETED` notification signed with
VAPID, encrypted per RFC 8291, accepted by FCM and rendered by the browser, 925 ms after the
match was completed. Until that date the feature had never once worked — see
[Delivery has been observed](#delivery-has-been-observed-once-and-what-it-took) below.

---

## Scope

Web Push (RFC 8291 payload encryption, RFC 8292 VAPID auth) — no Firebase, no vendor SDK. Four
endpoints for registering a browser and managing per-category preferences. Purely additive.

Base path: `/api/push`

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `GET` | `/api/push/public-key` | VAPID public key + whether push is enabled | **Public** |
| `POST` | `/api/push/subscribe` | Register this browser | Any authenticated |
| `DELETE` | `/api/push/subscribe?endpoint=…` | Remove this browser | Any authenticated |
| `GET` | `/api/push/preferences` | Categories + which are muted | Any authenticated |
| `PUT` | `/api/push/preferences/{category}?enabled=…` | Toggle one category | Any authenticated |

Every authenticated endpoint acts on **the calling user**, taken from the JWT principal. There
is no path or body parameter naming a subject, so nothing a caller can change reaches another
person's devices or preferences.

---

## Design decisions

### Why `/public-key` is public

It is the half of the keypair meant to be handed out — every browser embeds it in its
subscription request. It is listed explicitly in `SecurityConfig.PUBLIC_PATHS` rather than
opening `/api/push/**`, which would also expose subscribe and preferences.

It doubles as a capability probe: `enabled` is `false` when the server has no VAPID keys
configured, so the frontend can hide the feature instead of offering a button that cannot work.

### Why subscribe is an upsert on `endpoint`

The endpoint **is** the identity of a browser install. Browsers re-subscribe whenever the push
service rotates keys, and treating each as new would leave one person receiving one copy of
every notification per stale row. `POST /subscribe` therefore updates an existing row with the
same endpoint rather than inserting.

It also **reassigns ownership**: on a shared browser, someone logs out and a colleague logs in,
and the browser hands over the same endpoint. Reassigning is what stops the first person's
notifications arriving on the second person's screen.

### Why mutes, not preferences

Every category is **on by default**; a row exists only for an explicit opt-out. A new category
ships enabled with no backfill, and the table stays empty for users who never change anything.

`GET /preferences` returns the full category list alongside the muted subset, so the frontend
renders toggles without hard-coding categories — adding one on the backend makes it appear in
the UI, already on.

---

## Endpoints

### GET /api/push/public-key

**Auth:** none.

```json
{ "enabled": true, "publicKey": "BIKK8JsbtHXTGqITzMqvFzUKiqbE1knX-sPtIfxDRyXkj4X9cNFzqRLSCcMJVVCLqO2sxezxFq5h-uBU0GYkR_o" }
```

When `enabled` is `false`, `publicKey` is an empty string and no subscription is possible.

---

### POST /api/push/subscribe

**Auth:** `isAuthenticated()`. **Success:** `204 No Content`.

**Request body** — field names match the browser's `PushSubscription` shape so the frontend can
forward it with no reshaping:

```json
{
  "endpoint": "https://fcm.googleapis.com/fcm/send/dCh4…",
  "p256dh": "BNcRdreALRFXTkOOUHK1EtK2wtaz5Ry4YfYCA_0QTpQtUbVlUls0VJXg7A8u-Ts1XbjhazAkj7I99e8QcYP7DkM",
  "auth": "tBHItJI5svbpez7KI4CCXg",
  "deviceLabel": "Pixel 8 — Chrome"
}
```

| Field | Required | Notes |
|-------|----------|-------|
| `endpoint` | ✅ | Push service URL. No format validation — the set of valid hosts is not ours to police. |
| `p256dh` | ✅ | base64url; **must decode to a 65-byte uncompressed P-256 point** |
| `auth` | ✅ | base64url; **must decode to exactly 16 bytes** |
| `deviceLabel` | — | Free text, ≤255 chars, display only |

#### Error responses

| Status | Trigger | `message` |
|--------|---------|-----------|
| `400` | `p256dh` / `auth` not valid base64url | `"p256dh is not valid base64url"` |
| `400` | `p256dh` wrong length | `"p256dh must decode to a 65-byte uncompressed P-256 point, got 64"` |
| `400` | `auth` wrong length | `"auth must decode to a 16-byte secret, got 12"` |
| `400` | Point not on the P-256 curve | `"Not a valid P-256 public key"` |
| `403` | Unauthenticated | *(Spring Security — no body)* |
| `409` | More than 20 devices registered | `"Too many registered devices; remove one before adding another"` |

> Keys are validated **at registration, not at send time**. A subscription that can never be
> encrypted for is worse than useless — it is retried on every notification and only ever fails.

---

### DELETE /api/push/subscribe

**Auth:** `isAuthenticated()`. **Query:** `endpoint` (required). **Success:** `204 No Content`.

**Idempotent and deliberately silent.** Succeeds whether or not the endpoint was registered, and
whether or not it belonged to the caller. A `404` here would confirm to an attacker that a given
endpoint is registered to somebody; the delete itself is scoped to the caller's own rows, so an
endpoint belonging to someone else is untouched.

---

### GET /api/push/preferences

**Auth:** `isAuthenticated()`. **Success:** `200`.

```json
{
  "categories": [
    "DRAFT_YOUR_TURN",
    "DRAFT_COMPLETED",
    "MATCH_COMPLETED",
    "CONFIRMATION_DEADLINE",
    "MATCH_REMINDER",
    "MVP_VOTE_OPEN",
    "FEE_CHARGED"
  ],
  "muted": ["MATCH_COMPLETED"]
}
```

`categories` is the full server-side list in display order. `muted` is the subset switched off.
**A category absent from `muted` is on** — there is no third state.

Stale mute rows naming a category that no longer exists are filtered out, so `muted` is always a
subset of `categories`.

---

### PUT /api/push/preferences/{category}

**Auth:** `isAuthenticated()`. **Query:** `enabled` (boolean, required).
**Success:** `200` with the full `NotificationPreferencesDTO` — a toggle UI has to reconcile
against the server anyway, and returning the state saves a follow-up request.

**Idempotent:** setting a category to the state it is already in is a no-op, which matters for a
toggle a flaky connection may deliver twice.

| Status | Trigger | `message` |
|--------|---------|-----------|
| `400` | Unknown category | `"Unknown notification category: FOO"` |
| `403` | Unauthenticated | *(Spring Security)* |

---

## Notification categories

Deliberately short. The roadmap's finding was that competitors are praised for notifying
sparingly and complained about when noisy, so each addition must earn its place against the risk
of the whole channel being muted.

| Category | Sent when | Target | Trigger |
|----------|-----------|--------|---------|
| `DRAFT_YOUR_TURN` | The draft opens, or a pick puts the next captain on the clock | That captain only | `DraftPushNotifier` on `DraftStateChangedEvent("OPENED"` / `"PICK")` |
| `DRAFT_COMPLETED` | A draft finished and teams are known | Everyone in the draft, deduped | `DraftPushNotifier` on `("COMPLETED")` |
| `MATCH_COMPLETED` | A match completed **and ratings were recalculated** | Everyone who played | `MatchEventListener` |
| `CONFIRMATION_DEADLINE` | Deadline within 24h and this player has not responded | Only `PENDING` responders | `ReminderScheduler`, hourly |
| `MATCH_REMINDER` | Match is today or tomorrow | Only `CONFIRMED` players | `ReminderScheduler`, hourly |
| `MVP_VOTE_OPEN` | The crowd MOTM poll opened, immediately after `MATCH_COMPLETED` | Only players who appeared — exactly who may vote | `MatchEventListener` |
| `FEE_CHARGED` | A match fee was charged, once at creation — never as a running reminder | The charged player only | `MatchFeeService.generateCharges` |

Notice what is **not** sent: a pick by another captain (visible on the SSE stream to anyone
watching), a cancelled or converted draft, a deadline reminder to someone who already answered,
or a match reminder to someone who declined. Each of those was a candidate and each was dropped —
the roadmap's finding was that noisy apps get their notifications switched off wholesale, so a
notification has to be worth interrupting somebody for.

**`MATCH_COMPLETED` is sent after the rating recalculation succeeds, not when the match is
saved.** Notifying earlier would tell someone their rating changed and then send them to a screen
still showing the old one — and on the optimistic-lock retry path, a notification sent before a
failed attempt could not be taken back.

---

## What the browser receives

The service worker gets this JSON, decrypted. It renders with **no application code running**,
so everything it needs is in the payload:

```json
{
  "category": "DRAFT_YOUR_TURN",
  "title": "It's your pick",
  "body": "Reds vs Blues",
  "url": "/draft-sessions"
}
```

`url` is an in-app path for the `notificationclick` handler to open.

---

## Frontend Migration Notes

### 1. Feature detection, in this order

```ts
// 1. Does the browser support it at all?
const supported = 'serviceWorker' in navigator && 'PushManager' in window;

// 2. Is the server configured for it?
const { enabled, publicKey } = await fetch('/api/push/public-key').then(r => r.json());

// 3. iOS ONLY delivers push to an installed PWA (Safari 16.4+).
const isIos = /iPad|iPhone|iPod/.test(navigator.userAgent)
  || (/Macintosh/.test(navigator.userAgent) && navigator.maxTouchPoints > 1);
const isInstalled = window.matchMedia('(display-mode: standalone)').matches
  || (navigator as any).standalone === true;

const canOffer = supported && enabled && (!isIos || isInstalled);
```

On iOS in a browser tab, `Notification.requestPermission()` either throws or silently never
delivers. **Do not show the enable control there** — point at the install hint instead
(`IosInstallHint`, already shipped in Phase 1). This is the concrete reason Phase 1's install UX
mattered.

### 2. Subscribing

`applicationServerKey` must be a `Uint8Array`, not the base64url string:

```ts
function urlBase64ToUint8Array(base64: string) {
  const padded = (base64 + '='.repeat((4 - base64.length % 4) % 4))
    .replace(/-/g, '+').replace(/_/g, '/');
  return Uint8Array.from(atob(padded), c => c.charCodeAt(0));
}

const registration = await navigator.serviceWorker.ready;
const sub = await registration.pushManager.subscribe({
  userVisibleOnly: true,                     // required; Chrome rejects false
  applicationServerKey: urlBase64ToUint8Array(publicKey),
});

const json = sub.toJSON();                   // { endpoint, keys: { p256dh, auth } }
await apiFetch('/api/push/subscribe', {
  method: 'POST',
  body: JSON.stringify({
    endpoint: json.endpoint,
    p256dh: json.keys!.p256dh,
    auth: json.keys!.auth,
    deviceLabel: navigator.userAgent.slice(0, 255),
  }),
});
```

**Permission must be requested from a user gesture.** Calling it on page load is ignored by some
browsers and shows a hostile prompt on others. Put it behind an explicit "Enable notifications"
control.

**Re-send the subscription on every login.** The endpoint can change (push services rotate
them), and on a shared browser the row must be reassigned to whoever is now signed in. The
endpoint upsert makes this cheap and safe to repeat.

### 3. Unsubscribing

Do both, in this order — a browser-side unsubscribe alone leaves a row the server keeps trying
to use until the push service reports it gone:

```ts
const sub = await registration.pushManager.getSubscription();
if (sub) {
  await apiFetch(`/api/push/subscribe?endpoint=${encodeURIComponent(sub.endpoint)}`,
    { method: 'DELETE' });
  await sub.unsubscribe();
}
```

### 4. Service worker handlers

Add to `public/sw.js` — and **bump `VERSION`**, or existing installs keep the old worker:

```js
self.addEventListener('push', (event) => {
  if (!event.data) return;
  const { title, body, url, category } = event.data.json();
  event.waitUntil(self.registration.showNotification(title, {
    body,
    icon: '/icons/icon-192.png',
    badge: '/icons/icon-192.png',
    data: { url },
    // Collapses repeats of the same kind rather than stacking them.
    tag: category,
  }));
});

self.addEventListener('notificationclick', (event) => {
  event.notification.close();
  const url = event.notification.data?.url ?? '/';
  event.waitUntil((async () => {
    // Focus an existing tab if there is one; opening a second is the common annoyance here.
    const all = await clients.matchAll({ type: 'window', includeUncontrolled: true });
    for (const client of all) {
      if (client.url.includes(url) && 'focus' in client) return client.focus();
    }
    return clients.openWindow(url);
  })());
});
```

### 5. Preferences UI

Render from `categories`, not a hard-coded list, so a new backend category appears without a
frontend change. Treat absence from `muted` as on. Ship the toggles **with** the enable control —
the roadmap is explicit that retrofitting preferences after people are already over-notified is
harder and later than doing it now.

---

## Privacy

Push subscriptions are personal data: the endpoint identifies one specific browser install.

- **Export** (`GET /api/privacy/me/export`) reports them under `notifications`: device label,
  push service **host**, registration time, last notified time. The `p256dh` and `auth` keys are
  deliberately **excluded** — they are a capability to push to that browser, not facts about the
  person, and exporting them would put a working send capability into a file the subject may
  forward on. The full endpoint is excluded for the same reason.
- **Erasure** (`DELETE /api/privacy/me`) removes them: both tables cascade from `users`, and
  erasure deletes that row.

See [PRIVACY_AND_DATA_PROTECTION.md](../features/PRIVACY_AND_DATA_PROTECTION.md).

---

## Operations

Generate a keypair once:

```bash
./gradlew generateVapidKeys
```

Set `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT` (a `mailto:` or `https:` URL —
RFC 8292 requires it and some push services reject tokens without it).

> ⚠️ **Rotating the public key invalidates every existing subscription.** Browsers bind to it at
> subscribe time, so everyone must re-subscribe. Treat it as permanent.

Leaving the keys blank disables push cleanly: sends are skipped, `/public-key` reports
`enabled: false`, and the application boots normally. That is the default for local development
and CI.

### ⚠️ Push cannot be tested against the frontend dev server

**`next dev` never registers the service worker.** `ServiceWorkerRegistrar` returns early unless
`NODE_ENV === 'production'` — deliberately, because a worker caching Turbopack's dev chunks makes
hot reload unpredictable and every stale-asset bug then looks like an app bug.

Without a registered worker there is no push subscription, and the failure is **silent in the
worst way**: `navigator.serviceWorker.ready` does not reject when nothing is registered, it simply
**never settles**. So `createSubscription` shows the permission prompt, the user accepts, and the
function then hangs forever at the next line. No request reaches `POST /api/push/subscribe`, no row
appears in `push_subscriptions`, the toggle sits in its disabled "subscribing" state, and **no
error appears anywhere** — not in the console, not in the network tab, not in the server log.

To exercise the real delivery path you need a production build:

```bash
npm run build && npm run start   # serves on :3000, same as dev
```

`http://localhost` is a secure context, so no HTTPS or tunnel is needed.

> **Unregister the worker before returning to `next dev`** — DevTools → Application → Service
> Workers → Unregister. Otherwise it keeps serving cached production assets over the dev server,
> which is the exact failure the `NODE_ENV` guard exists to prevent.

The DevTools → Application → Service Workers → **Push** button is **not** a substitute. It fires a
synthetic payload straight at the service-worker handler and bypasses VAPID, the encryption, the
push service and the entire backend send path — i.e. everything that could actually be broken.

### Delivery has been observed once, and what it took

The first successful delivery was 2026-07-29. Everything below had to be true simultaneously, and
three of the four were wrong at some point on the way there — recorded because each failure was
silent, and the next person will otherwise rediscover them one at a time.

| Requirement | Failure mode when wrong |
|---|---|
| VAPID keys configured | `/public-key` reports `enabled:false`; the UI hides the control |
| **Production frontend build** | `next dev` registers no service worker; subscribing hangs forever with no error (see above) |
| **Browser willing to reach its push service** | Brave disables Google push messaging by default → `AbortError: Registration failed - push service error` at `pushManager.subscribe`, before any request reaches this API |
| **`aud` serialised as a JSON string** | FCM answers `403 permission denied: invalid aud claim` and every send fails |

> The `aud` bug is the one worth remembering. The *value* was correct; only its JSON shape was
> wrong — `["https://fcm.googleapis.com"]` instead of `"https://fcm.googleapis.com"`. JJWT models
> `aud` as a collection and serialises a single value as a one-element array, which RFC 7519
> §4.1.3 permits and FCM rejects. It survived a test that parsed the token back, because JJWT's
> parser normalises both shapes into the same `Set` — a round-trip assertion **cannot** see the
> difference. `VapidSignerTest` now asserts on the raw serialised payload instead.

**The whole send path is silent by design**, which is why this took so long to find.
`WebPushSender` never throws, `deliver()` catches everything, `notifyParticipants()` catches
again. That is correct — a push service being down must not roll back a completed match — but it
means a permanent misconfiguration and a transient outage look identical from outside, and neither
reaches a user, an HTTP response or a database row.

**`push_subscriptions.last_used_at` is the only externally visible evidence of success.** It is
written *only* on `Result.DELIVERED` (a 2xx from the push service), so a non-null value proves a
send was accepted. A row that exists with `last_used_at` still null after a trigger fired means
every send to it failed. That is the first thing to check, and it is also why the GDPR export's
"last notified time" can legitimately be null for a user who has been notified many times
unsuccessfully.

### Running more than one instance

The two reminder jobs are time-driven, so on two dynos they would each find the same plans in
the same window and send everything twice. The guard is a **conditional update** —
`SET …_reminder_sent_at = now WHERE id = ? AND …_reminder_sent_at IS NULL` (V13) — so the
database arbitrates: the loser updates zero rows and sends nothing. Reading the flag and then
writing it would leave a window in which both instances see `NULL`.

`@EnableScheduling` is therefore safe to run everywhere, and the same guard stops a restart
inside the window re-notifying on every tick.

**Reminders are at-most-once, not exactly-once.** If a claim succeeds and the process dies
before the sends finish, that reminder is lost. That is the deliberate trade: a missed reminder
is a small annoyance, a duplicate at 3am is why people disable notifications.

---

## Breaking Changes

- [x] **None.** Five new endpoints, two new tables, one new field on the privacy export
  (`notifications`). No existing endpoint, DTO, field or status code changes.

One additive change to be aware of: `PersonalDataExportDTO` gained a `notifications` object. It
is always present — never `null` — with empty arrays when there is nothing to report.
