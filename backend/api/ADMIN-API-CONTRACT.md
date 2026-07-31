# Admin Settings & System — API Contract

**Date:** 2026-07-30
**Version:** v1.0.0
**Status:** APPROVED — backend complete (settings CRUD, system health, badge backfill, cache eviction)

---

## Scope

Administrative operations that belong to the system rather than to any one match or player. One new
table (V16 `app_settings`), one new controller, four endpoints.

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `GET` | `/api/admin/settings` | Every configurable setting with its bounds and provenance | `ADMIN` |
| `PATCH` | `/api/admin/settings` | Change one or more settings | `ADMIN` |
| `GET` | `/api/admin/system-health` | Push configuration and data counts | `ADMIN` |
| `POST` | `/api/admin/badges/backfill` | Award badges across the whole match history | `ADMIN` |
| `POST` | `/api/admin/caches/evict` | Clear cached reads on this node | `ADMIN` |

**`MANAGER` is refused, not just unlisted.** Master runs the squad — players, matches, planning.
These are system concerns and are explicitly not theirs; there are tests asserting the `403`.

---

## Design decisions

### Settings have no rows until somebody changes something

`app_settings` is created empty. Defaults live in the `AppSetting` enum, which also owns each
setting's accepted range. A row exists only where an admin has overridden a default.

This is what makes the whole thing cheap to evolve: adjusting a default ships as an ordinary code
change and reaches every install that never touched it, adding a setting is an enum constant rather
than a migration, and deleting a row is the reset path — one that cannot drift from the code default
the way a seeded copy of it would. Seeding today's values would freeze them into every install and
make the enum defaults dead code the moment the table was populated.

### Bounds travel with the value

`GET` returns `min`, `max` and `defaultValue` alongside `value`. **Do not hard-code them in the
client.** A form carrying its own copy drifts from the server's the first time either changes, and
the failure mode is a field that looks valid and is rejected on save.

### Reads are forgiving, writes are strict

A stored value that no longer parses as an integer, or that falls outside a range a later version
narrowed, is logged and the **default is served**. Both are reachable without anybody doing anything
wrong — tightening a range below a value that was legal when it was saved is an ordinary release.
Refusing to build the league table over a stale row would turn a cosmetic problem into an outage.

Writes get no such latitude: out-of-range values are `400`.

### Cross-field validation runs against the resulting state

`LEADERBOARD_DEFAULT_LIMIT` must not exceed `LEADERBOARD_MAX_LIMIT`, or the configured default would
be clamped away on every request and never be what anybody received.

That is checked against the state the request would **produce**, not the request itself. Lowering the
maximum below a default the request does not carry is still rejected. Raising both together in one
request is accepted.

### An unknown setting name is `400`, not a silent no-op

Dropping it would return `200` with a body showing the setting unchanged — which reads as "the server
refused my value", a far more confusing failure than being told the name is wrong.

### Changing settings evicts dependent caches

The rankings cache is keyed on `'table-' + includeInactive` and the leaderboards cache on the limit.
Neither key includes these settings, so a change would otherwise not appear until the 10-minute TTL
expired. `PATCH` evicts both.

---

## Endpoints

### GET /api/admin/settings

**Success:** `200`.

```json
[
  {
    "name": "MVP_VOTING_WINDOW_HOURS",
    "key": "mvp.voting.window.hours",
    "value": 48,
    "defaultValue": 24,
    "min": 1,
    "max": 168,
    "overridden": true,
    "updatedAt": "2026-07-30T09:14:22Z",
    "updatedBy": "ricardo"
  },
  {
    "name": "RANKING_MINIMUM_MATCHES",
    "key": "ranking.minimum.matches",
    "value": 3,
    "defaultValue": 3,
    "min": 1,
    "max": 50,
    "overridden": false
  }
]
```

| Field | Notes |
|-------|-------|
| `name` | The enum constant. **This is what `PATCH` expects as its key** |
| `key` | Storage key. Shown for support; not used in requests |
| `value` | The effective value — override if there is one, otherwise `defaultValue` |
| `overridden` | `false` means still on the default, i.e. no row exists |
| `updatedAt` | **Absent** when not overridden |
| `updatedBy` | **Absent** when not overridden, or when the author's account has since been deleted |

> ⚠️ **Nullable fields are omitted, not sent as `null`** (`spring.jackson.default-property-inclusion:
> non_null`). `updatedAt` and `updatedBy` are **absent** for a setting still on its default — they
> arrive as `undefined`, and `x === null` is false. Branch on `overridden`, which is always present.

### The settings

| Name | Default | Range | Effect |
|------|---------|-------|--------|
| `MVP_VOTING_WINDOW_HOURS` | 24 | 1–168 | How long crowd MOTM voting stays open after completion |
| `RANKING_MINIMUM_MATCHES` | 3 | 1–50 | Completed matches needed before the league table gives a rank |
| `LEADERBOARD_DEFAULT_LIMIT` | 5 | 1–100 | Entries per category when the caller does not ask |
| `LEADERBOARD_MAX_LIMIT` | 25 | 1–100 | Ceiling on entries per category |

> **`MVP_VOTING_WINDOW_HOURS` never applies retroactively.** The closing time is stamped onto the
> match at completion, so polls already open keep the window they were opened with. Shortening it
> cannot close a vote somebody is part-way through casting.

---

### PATCH /api/admin/settings

**Success:** `200` with the full settings list, in the same shape as `GET` — a settings form has to
reconcile against the server anyway, and returning the new state saves a follow-up request.

```json
{ "values": { "MVP_VOTING_WINDOW_HOURS": 48, "RANKING_MINIMUM_MATCHES": 5 } }
```

Omitted settings are left alone.

| Status | Trigger |
|--------|---------|
| `400` | Unknown setting name |
| `400` | Value outside that setting's range |
| `400` | Resulting `LEADERBOARD_DEFAULT_LIMIT` above `LEADERBOARD_MAX_LIMIT` |
| `400` | Empty `values` object |
| `403` | Caller is not `ADMIN` |

---

### GET /api/admin/system-health

**Success:** `200`.

```json
{
  "vapidConfigured": true,
  "vapidSubject": "mailto:ops@example.com",
  "vapidPublicKey": "BJ7…",
  "pushSubscriptions": 3,
  "badgesAwarded": 42,
  "completedMatches": 17,
  "appVersion": "1.0.0"
}
```

**No key material is ever returned.** The VAPID *public* key is included because it is handed to
every subscribing browser anyway. The private key is not read by this path at all — a secret that is
never loaded cannot be leaked by a logging change somebody makes later, and there is a test asserting
no private-key field appears in the response.

This exists because the push stack has one failure mode that is invisible from the UI: everything
looks configured, subscriptions exist, and every send fails. That happened — a VAPID JWT whose `aud`
claim serialised as a one-element array, which FCM rejects with `403 permission denied: invalid aud
claim`. Surfacing configuration state turns "notifications are broken" into a first question with an
answer.

---

### POST /api/admin/badges/backfill

**Success:** `200`.

```json
{ "matchesProcessed": 17, "badgesAwarded": 42, "matchesFailed": 0 }
```

V15 shipped **without** a backfill on purpose: awarding everything at once would stamp the whole
roster with today's date and make `awarded_at` claim the group earned everything simultaneously. This
is the deliberate version, and it differs where it matters — matches are replayed oldest-first, so
each award still cites the match that actually earned it, and only `awarded_at` reflects the
backfill.

**Idempotent.** Awarding is an insert that `uq_player_badges` may reject, so a second run over the
same history awards nothing new. That is the same property that makes bulk rating recalculation safe,
and it is why this needs no "already run" flag.

**Each match is awarded in its own transaction**, so a failure part-way through keeps everything
already awarded; `matchesFailed` counts what was skipped. Slow on a long history, which is why it is
an explicit action rather than something that runs at startup.

---

### POST /api/admin/caches/evict

**Success:** `200`.

```json
{
  "clearedCount": 10,
  "caches": ["players", "rankings", "leaderboards", "…"],
  "note": "Caches are per-node; only the node that served this request was cleared."
}
```

For when data is corrected outside the application and a cached view goes stale.

**The per-node caveat is in the response body, not only here.** Caffeine is per-process, so on a
multi-node deployment this clears the node that happened to serve the request and no other. An admin
who clears a cache and still sees stale data from another pod should not have to go looking for why.

---

## Frontend notes

- **Read `min`/`max`/`defaultValue` from the response.** They are not constants.
- `overridden: false` is worth showing differently from a set value — it means "still on the default"
  and is the state a reset returns to.
- `PATCH` returns the full list, so the form can replace its state from the response rather than
  re-fetching.
- Backfill and recalculation are slow. They return only when finished; give them a pending state and
  do not race a second click.
