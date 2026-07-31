# Match Fee Ledger — API Contract

**Date:** 2026-07-31
**Version:** v1.0.0
**Status:** APPROVED — backend complete (V19, entities, service, endpoints, privacy, tests)
**Plan:** `docs/plans/MATCH-FEE-LEDGER-PLAN.md`

---

## Scope

Track who owes the group money for match fees and who has paid. **No money moves through this
application** — payment happens over MB WAY between phones, and a payment row is somebody's word
that it arrived. Collecting in-app would make someone a merchant of record, with a PSP contract,
KYC and one person legally receiving everyone else's money.

One migration (V19), two new tables, one new column on `match_plans`, nine endpoints.

| Method | Path | Auth |
|--------|------|------|
| `GET` | `/api/payments/me` | authenticated |
| `GET` | `/api/payments/balances` | `ORGANIZER` |
| `GET` | `/api/players/{id}/payments` | `ORGANIZER` |
| `POST` | `/api/payments` | `ORGANIZER` |
| `POST` | `/api/payments/{id}/void` | `ORGANIZER` |
| `POST` | `/api/players/{id}/charges` | `ORGANIZER` |
| `POST` | `/api/charges/{id}/void` | `ORGANIZER` |
| `PUT` | `/api/match-plans/{id}/cost` | `MANAGER` or `ORGANIZER` |
| `POST` | `/api/match-plans/{id}/charges` | `MANAGER` or `ORGANIZER` |

The split follows who knows the fact. **What the pitch cost** is known by whoever booked it, which
is the manager. **Whether money arrived** is only ever known to the person it was sent to, so every
payment operation is `ORGANIZER` alone.

`ADMIN` appears nowhere. Under the flat role model it grants system administration, not custody of
the group's money — an administrator who also collects fees holds `ORGANIZER` too, and says so.

---

## Design decisions

### Amounts are integer cents, and the sign convention is stated once

`balanceCents` is **negative when the player owes**, positive when in credit. It is never inverted
downstream — a DTO that flipped it "for display" is how a debt eventually gets shown as savings.

No decimal type crosses the wire. A euro amount in a floating-point field will eventually be off by
a cent, and it will be off in a record about money. Currency is EUR and is not carried.

### A running balance, not per-charge settlement

`balance = SUM(payments) − SUM(charges)`. There is no `payment_allocations` table tying a payment
to particular charges.

Real behaviour is *"here's €20, that covers me for a while"*, which a settlement model has to
express as a partial allocation across four charges, one of them in the future. **The cost is that
"has Ricardo paid for the 8th?" is unanswerable** — only "Ricardo is €10 down". Adding allocation
later leaves every existing row meaning what it already meant.

### The rounding rule

The plan stores a **total**; the per-head amount is computed once, at generation, when the headcount
is final. €70 across 13 does not divide.

```
base      = total / n
remainder = total % n        → the first `remainder` shares get one cent more
```

Players are ordered by **ascending id**, so the same people pay the extra cent on every
regeneration and "why did I pay a cent more than Bruno" has a stable answer. The shares sum to the
total exactly; without this the group quietly loses up to `n−1` cents per match and the books never
balance to zero.

### Charges are frozen at creation

Each row carries its own settled amount. `PUT /match-plans/{id}/cost` therefore does **not** alter
charges already generated — correcting a typo in the pitch price must not silently rewrite what
people already owe.

### Generation is idempotent

`uq_player_charges_plan UNIQUE (match_plan_id, player_id)` is the guarantee; the pre-check in the
service is only an optimisation. Two concurrent calls can both pass the check, and the loser's
`DataIntegrityViolationException` is swallowed because the player ends up charged exactly once
either way — the same reasoning as `uq_player_badges`.

Ad-hoc charges carry `match_plan_id = NULL`, and Postgres does not treat NULLs as equal in a
UNIQUE, so a player may hold any number of them. **That is load-bearing, not incidental.**

### Nothing is ever deleted

Waiving a charge and correcting a mistaken one are the same operation with different reasons. Both
set `voided_at` / `voided_by` / `void_reason`, and voided rows are excluded from every balance. A
deleted financial record is an argument waiting to happen.

### Players with no account are charged like anybody else

Both tables key on `player_id`, **never `user_id`**. Plenty of people turn up and will never install
the app; an organiser's list of who owes money is worthless if it silently omits half the pitch.

Consequences:

- `GET /payments/balances` starts from the **roster**, not the ledger. A player who has never been
  charged appears at zero rather than being absent.
- Each row carries `contactable` — whether a reminder reaches them in-app or has to be a WhatsApp
  message. Without it, "notified and ignoring me" is indistinguishable from "never heard about it".
- Linking an account later needs no backfill. The history was already theirs.

### Balances are never cached

A few dozen rows of arithmetic, and a stale number about money is far worse than a stale league
table.

---

## `GET /api/payments/me`

**Success:** `200`.

```json
{
  "playerId": 4,
  "playerName": "Ricardo",
  "balanceCents": -1000,
  "charges": [
    { "id": 91, "amountCents": 500, "description": "Friday 5-a-side",
      "matchPlanId": 33, "createdAt": "2026-08-07T21:30:00Z", "voided": false }
  ],
  "payments": [
    { "id": 12, "amountCents": 2000, "paidAt": "2026-08-01T18:00:00Z",
      "note": "MB WAY", "createdAt": "2026-08-02T09:00:00Z", "voided": false }
  ]
}
```

| Status | Trigger |
|--------|---------|
| `409` | The caller has no linked player. An empty ledger would read as "you're square", which is a different claim |
| `403` | Unauthenticated |

> ⚠️ `matchPlanId`, `description`, `note`, `voidedAt` and `voidReason` are **omitted when absent**,
> not null — `spring.jackson.default-property-inclusion: non_null`. They arrive as `undefined` in
> TypeScript. Use the `nullableField` helper.

`matchPlanId` is absent for an ad-hoc charge **and** once the plan has been deleted — the FK is
`ON DELETE SET NULL`, because the pitch was still paid for. `description` is what still says what
it was for.

## `GET /api/payments/balances`

**Success:** `200`. Most in debt first, then by name.

```json
[
  { "playerId": 7, "playerName": "Bruno", "balanceCents": -1500, "contactable": false, "active": true },
  { "playerId": 4, "playerName": "Ricardo", "balanceCents": -500, "contactable": true, "active": true },
  { "playerId": 2, "playerName": "Ana", "balanceCents": 0, "contactable": true, "active": true }
]
```

Includes every **active** player, plus any inactive player who still owes — leaving the group is not
a way to settle up.

## `POST /api/payments`

```json
{ "playerId": 4, "amountCents": 2000, "paidAt": "2026-08-01T18:00:00Z", "note": "MB WAY" }
```

| Status | Trigger |
|--------|---------|
| `400` | `amountCents` ≤ 0, or `paidAt` in the future — money that has not moved yet is not a payment |
| `403` | Caller does not hold `ORGANIZER` |
| `404` | No such player |

## `POST /api/payments/{id}/void` · `POST /api/charges/{id}/void`

Body is optional: `{ "reason": "Recorded twice" }`.

| Status | Trigger |
|--------|---------|
| `409` | Already voided |
| `404` | No such row |

## `PUT /api/match-plans/{id}/cost`

```json
{ "totalCostCents": 7000 }
```

`null` clears it. **Zero means the match was free**, which is different from "not known yet" — a
null cost generates no charges and is not an error. Returns `204`.

## `POST /api/match-plans/{id}/charges`

Generates charges for everybody confirmed. Returns `200` with the charges **this call created** —
an empty array when there was nothing to do (no cost recorded, free match, nobody confirmed, or all
charges already exist).

Normally unnecessary: generation runs automatically at the CONFIRMED → GENERATED transition, when
teams are drawn. This exists for when the cost was not known at that moment.

---

## Notifications

One new category, `FEE_CHARGED`, fired **once** per player when their charge is created — not as a
running reminder. A recurring "you still owe €10" to a friend is a different social object from a
match reminder and the group should turn that on deliberately rather than inherit it.

Regeneration notifies nobody: only charges created by that call are announced. Players with no
linked account receive nothing, which the push layer already treats as normal.

The category list is server-sent from `GET /api/push/preferences`, so it appears in the UI without
a frontend deploy. Every category is on by default; a mute row is an explicit opt-out.

---

## Settings

`MATCH_DEFAULT_COST_CENTS` (`match.default.cost.cents`), default `7000`, range `0–100000`.

**A pre-fill for the plan creation form, nothing more.** Each plan stores its own cost and each
charge freezes its own share, so changing this reaches only plans created afterwards. Reading it
live would mean an admin editing this figure retroactively altered what people already owed.

---

## Privacy

`player_charges` and `player_payments` are financial records about identified people.

- **Export** reports them under `charges` and `payments`, always present and empty rather than
  absent. Both `exportForUser` and `exportForPlayer` go through the same builder — which matters,
  because a player with no account is reachable only by the second and is exactly the population
  most likely to have a balance they cannot see for themselves.
- Voided rows **are** included: "charged, then withdrawn" is a different fact from "never charged".
- **Erasure leaves the amounts in place**, following `player_stats` — they are the group's books,
  and a charge attached to `Deleted player #5` attributes nothing to anybody.
- **Erasure does scrub the audit columns.** `created_by`, `recorded_by` and `voided_by` hold a
  username, which identifies a person as surely as a name; leaving "who recorded this payment"
  answerable across the ledger would leave the erasure incomplete. Note the common case is an
  erased **organiser**, not an erased player.

`PersonalDataExportDTO` gained `charges` and `payments`. Additive.

---

## Breaking changes

- [x] **None.** Two new tables, one nullable column, nine new endpoints, one new setting, one new
      notification category, two additive fields on `PersonalDataExportDTO`.

> **V19 has not run anywhere.** Flyway is disabled in tests (`ddl-auto: create-drop`), so the CHECK
> constraints and the UNIQUE are unexercised by the suite and execute for the first time against a
> real database. It creates tables rather than altering data, so it is far less risky than V18 —
> but it does add a column to `match_plans`.
