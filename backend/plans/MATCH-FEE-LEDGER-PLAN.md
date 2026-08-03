# Match Fee Ledger — Technical Specification

**Date:** 2026-07-31
**Status:** **SHIPPED** — backend `828db3b` (migration `V19`, `MatchFeeService`, `PaymentController`,
`PAYMENTS-API-CONTRACT.md`) and frontend `a3efac0` / `4f47bf5`. The document below is kept as the
design record: it is why the feature has the shape it has, not a description of pending work.
**Priority:** MEDIUM
**Estimated Effort:** M (≈1 day backend, ≈1 day frontend) — held, roughly
**Depends on:** `ROLE-MODEL-MIGRATION-PLAN.md` — every endpoint here is gated on `ORGANIZER`,
which does not exist until that ships. Build it first, or this feature will be written against
`hasAnyRole('ADMIN_USER','MASTER_USER')` and migrated a week later. *(Satisfied: roles shipped
first, in `d6b908f`.)*

---

## 1. Requirement Summary

The group pays a pitch fee for every match. Track who owes what and who has settled up, so the
organiser stops chasing people from memory.

**No money moves through this application.** Payment happens over MB WAY between phones, exactly
as it does today. This is a bookkeeping feature: the app records amounts and states that someone
has paid, on the word of the organiser.

That boundary is the whole point. The moment the app *collects* money, somebody becomes a merchant
of record with a PSP contract, KYC, PSD2 obligations and an income-tax question. See §14 for how a
real payment integration bolts onto this later without any of it being wasted.

---

## 2. Scope

| In | Out |
|----|-----|
| Per-plan cost, split across the squad | Card / MB WAY / any payment initiation |
| Recording payments received | Refunds, chargebacks, currency other than EUR |
| Per-player running balance | Per-charge settlement (see §4) |
| Ad-hoc charges and waivers | Invoices, receipts, VAT, accounting export |
| Own-balance visibility for players | Player-to-player debts (only player ↔ the pot) |

---

## 3. The two tables, and why the fee hangs off the plan

`Match` has **no foreign key to `MatchPlan`** — this was confirmed in V17, where GENERATED could
not be backfilled precisely because nothing links a match back to the plan that produced it.

So the fee attaches to the **`MatchPlan`**, not the `Match`:

- The plan is what exists when the pitch is booked, which is when the money is actually owed.
- The plan owns `PlayerConfirmation`, so the set of people who committed is already there.
- A plan that is cancelled after a non-refundable booking still has a cost. A `Match` that never
  happened does not exist to hang it on.
- Matches created through `POST /api/matches/manual` have no plan and therefore no fee. That is
  correct — those are ad-hoc records, frequently historical, and charging for them retroactively is
  not a thing anyone wants.

---

## 4. Model decision: running balance, not per-charge settlement

**Chosen:** two append-only tables — `player_charges` (what you owe) and `player_payments` (what
you handed over) — with a player's balance derived as `SUM(payments) − SUM(charges)`.

**Rejected:** allocating each payment to specific charges via a `payment_allocations` join table.

Rationale:

- Real behaviour is `"here's €20, that covers me for a while"`. A settlement model has to express
  that as a partial allocation across four charges, one of which is in the future. A running
  balance expresses it as one row.
- Overpayment and pre-payment need no special case. The balance simply goes positive.
- The question people actually ask is *"are we square?"*, which is one number.

**The trade-off, stated plainly:** you cannot answer *"has Ricardo paid for the 8th of August?"* —
only *"Ricardo is €10 down"*. If the group later decides it needs per-match settlement, that is
**additive**: add `payment_allocations`, and every existing row still means what it meant. Nothing
here has to be rewritten. This is the same posture as `AppSetting`'s "integers only, for now".

---

## 5. Business rules

| # | Rule | Notes |
|---|------|-------|
| BR-1 | All amounts are **integer cents**, never a floating-point type | A `double` euro amount will eventually be off by a cent, and it will be off in a record about money |
| BR-2 | Currency is EUR, hardcoded, with no column | Single group, single country. Extension point: add `currency` to both tables and to the DTOs |
| BR-3 | Charges are generated when a plan reaches **GENERATED**, for every player whose confirmation is `CONFIRMED` | That is the moment the squad is final and the pitch is committed |
| BR-4 | `UNIQUE (match_plan_id, player_id)` on `player_charges` | Prevents a double charge. The constraint is the guarantee; any pre-check is an optimisation — the same reasoning as `uq_player_badges` |
| BR-5 | Ad-hoc charges carry `match_plan_id = NULL` | Postgres does not treat NULLs as equal in a UNIQUE, so a player can hold several ad-hoc charges without fighting BR-4. This is load-bearing, not incidental |
| BR-6 | Nothing is ever deleted. Corrections and waivers set `voided_at` | A deleted financial record is an argument waiting to happen |
| BR-7 | Voided rows are excluded from every balance | |
| BR-8 | A charge's amount is **frozen at creation** | Editing the plan's cost afterwards must not silently rewrite what people already owe |
| BR-9 | Balances are never cached | They are ~20 rows of arithmetic, and a stale number about money is far worse than a stale league table |
| BR-10 | A player with no linked `AppUser` is charged, counted and chased like anybody else — they simply cannot see it themselves | See §5.1. The organiser manages them entirely |

### The rounding rule (BR-11)

The plan stores a **total** cost; the per-head amount is computed once, at charge generation, when
the headcount is final.

`€70.00 / 14 = €5.00` is the happy path. `€70.00 / 13` is not.

```
base      = total_cents / n          (integer division)
remainder = total_cents % n
```

The first `remainder` players, **ordered by ascending player id**, are charged `base + 1`; the rest
`base`. The charges then sum to exactly `total_cents`.

Ordering by id rather than at random makes it reproducible, which matters when someone asks why
they paid a cent more than Bruno. Without this rule the group quietly loses up to `n−1` cents per
match, and the books never balance to zero.

---

## 5.1 Players without an account

Plenty of people turn up to play and will never install the app. **Their debt counts exactly the
same.** An organiser's list of who owes money is worthless if it silently omits half the pitch.

This works because **both tables key on `player_id`, never `user_id`** — the account link is
nowhere in the ledger. That is the whole mechanism, and it is worth stating because the natural
mistake is to reach for `users` when building the balances query.

The surrounding code already supports it end to end:

| Step | Why an unlinked player is fine |
|------|-------------------------------|
| Getting confirmed | `MatchPlanService.adminUpsertConfirmation(planId, **playerId**, dto)` takes a player id and never consults the link. A manager confirms them on their behalf |
| Being charged | Charges are generated from `PlayerConfirmation`, which is player-keyed |
| Being notified | `PushNotificationService.notifyPlayer` takes a player id, and `deliver` already skips a player with no linked account as *"normal, not an error"* (`PushNotificationService:90`). No new guard needed — but **do not** add one that throws |
| Being erased | `PrivacyService.erasePlayer(playerId)` and `exportForPlayer(playerId)` already exist for precisely this population |

### Consequences to build for

1. **`GET /payments/balances` lists every active player, not every user.** The query starts from
   `players`, left-joins the ledger, and never touches `users`. A player who has never been
   charged appears with a balance of zero rather than being absent — "who owes" is only trustworthy
   if the list is the whole roster.
2. **The balances response carries a `contactable` flag** (`linkedUserId` present). An organiser
   chasing €5 needs to know whether a reminder will reach them in-app or whether this one has to be
   a WhatsApp message. Without it the organiser cannot tell the difference between "notified and
   ignoring me" and "never heard about it".
3. **`GET /payments/me` still 409s** for a caller with no linked player. That is the *user* side of
   the same coin and is unchanged.
4. **Linking later is free.** When someone finally registers and runs `linkMe`, every historical
   charge and payment is already theirs, because it was keyed on the player all along. No backfill,
   no reconciliation, no migration — the debt simply becomes visible to them.

---

## 6. Schema — `V19__match_fee_ledger.sql`

> **V19, not V18** — `V18__composable_roles.sql` belongs to the role migration this depends on.
>
> V17 is today's head. **Flyway is disabled in tests** (`ddl-auto: create-drop`), so every CHECK
> constraint below is unexercised by the suite and will run for the first time against a real
> database. Same caveat as V17.

```sql
ALTER TABLE match_plans
    ADD COLUMN total_cost_cents INTEGER
        CHECK (total_cost_cents IS NULL OR total_cost_cents >= 0);

CREATE TABLE player_charges (
    id              BIGSERIAL PRIMARY KEY,
    player_id       BIGINT      NOT NULL REFERENCES players(id) ON DELETE CASCADE,
    match_plan_id   BIGINT               REFERENCES match_plans(id) ON DELETE SET NULL,
    amount_cents    INTEGER     NOT NULL CHECK (amount_cents > 0),
    description     VARCHAR(255),
    voided_at       TIMESTAMPTZ,
    voided_by       VARCHAR(50),
    void_reason     VARCHAR(255),
    created_by      VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT uq_player_charges_plan UNIQUE (match_plan_id, player_id)
);

CREATE TABLE player_payments (
    id              BIGSERIAL PRIMARY KEY,
    player_id       BIGINT      NOT NULL REFERENCES players(id) ON DELETE CASCADE,
    amount_cents    INTEGER     NOT NULL CHECK (amount_cents > 0),
    paid_at         TIMESTAMPTZ NOT NULL,
    note            VARCHAR(255),
    voided_at       TIMESTAMPTZ,
    voided_by       VARCHAR(50),
    void_reason     VARCHAR(255),
    recorded_by     VARCHAR(50) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_player_charges_player  ON player_charges(player_id);
CREATE INDEX idx_player_payments_player ON player_payments(player_id);
```

Notes:

- `ON DELETE SET NULL` on the plan reference, not CASCADE. Deleting a plan must not erase the debt
  it created — the pitch was still paid for. The charge survives, orphaned but intact, and its
  `description` still says what it was for.
- `ON DELETE CASCADE` on the player is safe because GDPR erasure **anonymises in place** rather
  than deleting (see §11), so the cascade never actually fires for a real person.
- `paid_at` is separate from `created_at`: the organiser records on Tuesday that you paid on Friday.
- Both FK columns get an index, per V8's precedent.

---

## 7. Entities

`PlayerCharge` and `PlayerPayment`, in `pt.rics.demo.football.model`, following the existing
Lombok `@Getter/@Setter/@Builder` shape with `@ManyToOne(fetch = LAZY)` on `player` and
`matchPlan`.

Both get a derived helper:

```java
@Transient
public boolean isVoided() {
    return voidedAt != null;
}
```

`MatchPlan` gains `private Integer totalCostCents;` — nullable, meaning "no cost recorded". Not
zero, which means "this one was free".

---

## 8. Charge generation

One service method, two callers:

```java
// MatchFeeService
@Transactional
public List<PlayerCharge> generateChargesFor(MatchPlan plan) { ... }
```

- Called automatically from `MatchPlanService` at the CONFIRMED → GENERATED transition.
- Called explicitly by `POST /api/match-plans/{id}/charges` when the cost was not known at
  generation time.

If `totalCostCents` is null it is a no-op — no cost, no charges, no error. Re-running it is safe:
BR-4 makes the insert fail per-player, and `DataIntegrityViolationException` is swallowed because
the player is charged either way.

> **Trap.** This must not be a private method called from within `MatchPlanService`. A same-bean
> call bypasses the proxy and `@Transactional` silently does nothing — the trap that already
> produced `ReminderScheduler`/`ReminderDispatcher`, `MvpResolutionScheduler`/`Dispatcher` and
> `BadgeBackfillService`. `MatchFeeService` is a separate bean for exactly this reason.

### No-shows

A player who confirmed, was drafted, and did not turn up **is charged**, because the charge is
written at generation. For most groups that is the desired rule — the pitch was booked for them.
Where it is not, the organiser voids the charge with a reason. Flagged in §13 as a decision for the
group, but the default costs nothing to reverse.

---

## 9. API

All paths under `/api`. Contract doc to ship in the same commit as the implementation, per repo
convention: `docs/api/PAYMENTS-API-CONTRACT.md`.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/payments/me` | authenticated | Caller's balance + full history |
| `GET` | `/payments/balances` | `ORGANIZER` | **Every active player's** balance, most in debt first — see §5.1 |
| `GET` | `/players/{id}/payments` | `ORGANIZER` | One player's charges and payments |
| `POST` | `/payments` | `ORGANIZER` | Record a payment received |
| `POST` | `/payments/{id}/void` | `ORGANIZER` | Void a payment recorded in error |
| `POST` | `/players/{id}/charges` | `ORGANIZER` | Ad-hoc charge (`match_plan_id` NULL) |
| `POST` | `/charges/{id}/void` | `ORGANIZER` | Waive or correct a charge |
| `PUT` | `/match-plans/{id}/cost` | `ORGANIZER` or `MANAGER` | Set the plan's total cost |
| `POST` | `/match-plans/{id}/charges` | `ORGANIZER` or `MANAGER` | Generate charges for a plan (idempotent) |

The split follows who knows the fact. **What the pitch cost** is known by whoever booked it, which
is the manager — so setting the cost and triggering generation is open to both. **Whether money
arrived** is only ever known to the person it was sent to, so every payment operation is
`ORGANIZER` alone.

Note that `GROUP_ADMIN` appears nowhere. Under the flat role model it grants system administration, not
custody of the group's money — an administrator who also collects the fees holds `ORGANIZER` too,
and says so.

### `GET /payments/me`

```json
{
  "playerId": 4,
  "playerName": "Ricardo",
  "balanceCents": -1000,
  "charges": [
    { "id": 91, "amountCents": 500, "description": "Friday 5-a-side",
      "matchPlanId": 33, "createdAt": "2026-08-07T21:30:00Z" }
  ],
  "payments": [
    { "id": 12, "amountCents": 2000, "paidAt": "2026-08-01T18:00:00Z",
      "note": "MB WAY" }
  ]
}
```

`balanceCents` is **negative when the player owes**, positive when in credit. Sign convention
stated once here and never inverted in a DTO.

Returns `409` when the caller has no linked player — the same shape `PLAYER-LINK-ME` already uses,
rather than an empty ledger that would read as "you're square".

> ⚠️ `matchPlanId`, `description`, `note` and every `void*` field are **omitted when absent**, not
> null — `spring.jackson.default-property-inclusion: non_null`. Use the `nullableField` helper on
> the frontend.

### `POST /payments`

```json
{ "playerId": 4, "amountCents": 2000, "paidAt": "2026-08-01T18:00:00Z", "note": "MB WAY" }
```

| Status | Trigger |
|--------|---------|
| `400` | `amountCents` ≤ 0, or `paidAt` in the future |
| `404` | No such player |
| `403` | Not GROUP_ADMIN/MASTER |

### Visibility

A player without `ORGANIZER` sees **their own balance only**. Other people's debts are a meaningful
step up in sensitivity from match statistics, and "who owes" is a conversation for the organiser to
start. Trivially relaxed later by widening `/payments/balances` — §13.

`ORGANIZER` also earns contact-detail visibility in `PlayerPiiPolicy`, which the role migration
handles. That is deliberate rather than incidental: contact details include the phone number, and a
phone number is how an MB WAY payment is addressed.

---

## 10. Settings

One new `AppSetting` constant:

```java
MATCH_DEFAULT_COST_CENTS("match.default.cost.cents", 7000, 0, 100_000),
```

€70 default, €0–€1000 range. Zero is allowed and means a free pitch, which happens.

The integer-cents choice lands cleanly on `AppSetting` being **integer-only** — no type
discriminator needed, and the "give the enum a `type` field" extension point stays unclaimed.

**This is a pre-fill for the plan creation form only.** The plan stores its own cost, and charges
freeze theirs (BR-8). Reading the setting live would mean changing the default retroactively
altered what people already owed — the same class of bug that `LEADERBOARD_MAX_LIMIT` clamping the
default away turned out to be.

`AppSettingsService.update` needs no new eviction: no cache holds a cost.

---

## 11. Privacy

Financial records about identified people. `PRIVACY_AND_DATA_PROTECTION.md` requires **three**
places to change — all three, or the audit is wrong:

1. **The data table** — add `player_charges` and `player_payments` with purpose and retention.
2. **`PrivacyService`** — export under `charges` and `payments` (always present, empty rather than
   absent, matching `badges`). Erasure **leaves both untouched**, following `player_stats`: the
   group's books are a record of the group, and deleting a debt rewrites shared history. The rows
   survive attached to `Deleted player #5`, attributing nothing to anybody.

   **Both export paths, not just one.** `exportForUser(userId)` and `exportForPlayer(playerId)` are
   separate methods, and a player with no account is only ever reachable through the second. Adding
   the fields to `exportForUser` alone would leave exactly the population from §5.1 with an
   incomplete subject-access response — the people least able to notice.
3. **`PrivacyServiceTest`** — a case proving export includes them and erasure does not remove them.

`PersonalDataExportDTO` gains two additive fields. No breaking change.

---

## 12. Notifications and frontend

**Push.** A new `PAYMENTS` category in the preference enum, fired once after charge generation:
*"€5 for Friday 5-a-side — you're now €10 down."* The category list is server-sent from
`GET /api/push/preferences`, so it appears without a frontend deploy.

Chasing reminders are deliberately **not** in scope for v1. A recurring automated debt notification
to a friend is a different social object from a match reminder, and the group should decide it
deliberately rather than inherit it.

**Frontend** (separate repo, `FootMania-Simple-Front`):

| Surface | Change |
|---------|--------|
| `/payments` | New route. Own balance + history for everyone; `ORGANIZER` additionally sees the full balances table with record-payment and void actions |
| Dashboard | A balance line in `AwaitingYou` when the caller owes something — this is exactly the "upcoming notices" the overview redesign was for |
| `MatchPlanDetailModal` | Cost field and the resulting per-player charges, `ORGANIZER` or `MANAGER` |
| Settings → System | Default match cost, alongside the existing admin settings (`GROUP_ADMIN` — it is an `AppSetting`) |
| `docs/features/payments.md` | New |

Money formatting goes through `Intl.NumberFormat(locale, { style: 'currency', currency: 'EUR' })`
on `cents / 100`, computed at the render edge. Cents stay integers everywhere else.

> i18n: `payments.*` in en/pt/es. **Do not `JSON.stringify` the locale files** — pt and es use
> compact one-line objects and a rewrite produces an 800-line diff for 40 keys. Splice the new block
> in, anchored to `${eol}  "payments": {`.

---

## 13. Decisions for the group

1. **No-shows pay?** Default yes (§8). Voidable per charge.
2. **Can everyone see everyone's balance?** Default no (§9). One-line change if the group prefers
   full transparency, which many do.
3. **How many organisers?** The role model allows any number. One is the obvious start; a second
   is worth having before the first goes on holiday, since nobody else can record a payment.
4. **What happens to a balance at season rollover?** Default: nothing, it rolls forward. Zeroing
   would need a settlement event, which is a bigger feature.

> *"Who may record a payment"* was previously listed here as an open question. The role model
> answers it: `ORGANIZER`, held by whoever actually receives the money.

---

## 14. How MB WAY bolts on later

Nothing in this spec is wasted if the group later wants automated collection through a Portuguese
PSP (ifthenpay, Easypay, Eupago):

- A PSP integration is a **payment initiation** layer. It produces exactly one thing this ledger
  does not already have: a `player_payments` row written by a webhook instead of by a person.
- `player_payments` gains `provider`, `provider_reference` and a status, with
  `UNIQUE (provider, provider_reference)` for webhook idempotency — the same constraint-as-guarantee
  pattern used throughout.
- `Player.phoneNumber` already exists, which is what MB WAY pushes are addressed to.
- What does *not* come free: a merchant account, a secret-verified unauthenticated webhook endpoint,
  a PENDING → PAID/EXPIRED state machine with a reconciliation job, per-transaction fees on ~€5
  payments, and a processor disclosure in the privacy policy.

Build the ledger first and the group may well find that the chasing, not the collecting, was the
whole problem.

---

## 15. Test plan

| Area | Cases |
|------|-------|
| Splitting | Exact division; remainder distributed to lowest ids; shares sum to the total; `n = 0`; cost null |
| Idempotency | Generating twice for the same plan adds nothing |
| Balance | Sign convention; voided rows excluded; empty ledger is 0, not an error |
| Ad-hoc | Two NULL-plan charges for one player both persist (BR-5) |
| **Unlinked players** | A player with `user == null` is charged by generation; appears in `/payments/balances` with the right amount and `contactable: false`; charge generation does **not** throw when notifying them; linking afterwards exposes the pre-existing balance unchanged |
| Security | A player without `ORGANIZER` cannot reach `/payments/balances` or another player's ledger — including a user holding `GROUP_ADMIN` or `MANAGER`, which is what proves the roles are genuinely flat |
| Privacy | Export includes both tables; erasure leaves them |

> **CI invariant:** `build.gradle` asserts
> `testControllers + testServices + testSecurity + testApplication == test`. New test classes must
> land in one of those packages or the build fails on the count, not on a failing test.

---

## 16. Breaking changes

- [x] **None.** Two new tables, one nullable column on `match_plans`, one new `AppSetting`
      constant, two additive fields on `PersonalDataExportDTO`.
