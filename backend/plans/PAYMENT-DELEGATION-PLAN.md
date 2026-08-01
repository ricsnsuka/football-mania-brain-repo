# Payment Delegation — Technical Specification

**Date:** 2026-08-01
**Status:** DRAFT (2026-08-01) — nothing built in either repo
**Priority:** MEDIUM
**Estimated Effort:** S (≈½ day backend, ≈½–1 day frontend)
**Depends on:** `MATCH-FEE-LEDGER-PLAN.md` *(satisfied — shipped in `828db3b` / `a3efac0`)*
**Depended on by:** `GUEST-PLAYERS-PLAN.md` — guest fees are an automatic instance of this feature. Build this first.
**Contract:** `docs/api/PAYMENTS-API-CONTRACT.md` (updated in the same backend commit, per repo convention)

---

## 1. Requirement Summary

Sometimes one player hands the organiser money for several people — a lift-sharing group, a couple,
a parent and kid, or (the case that motivated this) a member covering the guest they brought. Today
the ledger cannot say so: `player_payments` has exactly one `player_id`, the organiser's balances
table lists everyone as their own debtor, and "Bruno pays for those two" lives in the organiser's
memory or in a free-text `note` nothing parses.

**Payment delegation** makes that arrangement a record: a player (the *debtor*) can have another
player (the *payer*) designated as responsible for settling their debt. The organiser's view then
answers two questions at once — *who do I chase* (the payer, with the players they cover) and *what
does each individual owe* (the per-player breakdown, unchanged).

As with the ledger itself: **no money moves through this application.** A delegation changes who
the organiser talks to, not where a cent is recorded.

---

## 2. Scope

| In | Out |
|----|-----|
| Standing debtor → payer designation | Per-charge or per-match delegation |
| Who-covers-whom in the organiser's balances view | Any change to how charges are generated or split |
| Recording who physically handed over a payment | Transferring debt between ledgers |
| Lump-sum entry allocated across covered debtors | Delegation chains (A pays for B who pays for C) |
| Ending / replacing a delegation | Debtor consent flows — the organiser records reality |

---

## 3. Model decision: a responsibility layer, not a ledger change

**Chosen:** a new `payment_delegations` table holding a *standing* debtor → payer mapping, plus one
nullable provenance column on `player_payments` (`paid_by_player_id`) recording who physically
handed the money over. Charges, payments and balances stay keyed per player, exactly as V19 left
them.

**Rejected:** moving or duplicating the debtor's charge onto the payer's ledger.

Rationale:

- `uq_player_charges_plan UNIQUE (match_plan_id, player_id)` means a payer confirmed on the same
  plan **cannot** hold a second charge row for it. The constraint is load-bearing (BR-4 of the
  ledger plan); delegation must live above it, not fight it.
- BR-8 freezes a charge's amount at creation, and the split rule (BR-11) proves the per-head
  amounts sum to the plan's total. Folding someone else's share into a charge breaks both.
- The requirement itself asks for **the breakdown to survive**: "the organiser knows which players
  the payer is paying for *and still has a breakdown of each player's debts*." Keeping every row on
  the debtor makes the breakdown correct by construction rather than by bookkeeping.
- Ending a delegation is then trivial: the mapping ends, and every ledger row already sits on the
  right person. Nothing to unwind, nothing to reallocate.

The same reasoning applies on the payment side: a lump sum from a payer covering three debtors is
recorded as **three `player_payments` rows** — each on the debtor whose balance it settles, each
stamped `paid_by_player_id` = the payer. `balanceFor()` and the sign convention are untouched;
"who handed me the cash" becomes a recorded fact instead of a memory.

---

## 4. Schema — `V20__payment_delegation.sql`

> V19 is today's head. Same caveat as V17–V19: **Flyway is disabled in tests**
> (`ddl-auto: create-drop`), so the constraints below first run for real against a live database —
> and per `architecture/database-migrations.md`, V17–V19 themselves have not run there yet either.

```sql
CREATE TABLE payment_delegations (
    id                BIGSERIAL   PRIMARY KEY,
    debtor_player_id  BIGINT      NOT NULL REFERENCES players(id) ON DELETE CASCADE,
    payer_player_id   BIGINT      NOT NULL REFERENCES players(id) ON DELETE CASCADE,
    created_by        VARCHAR(50),
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at          TIMESTAMPTZ,
    ended_by          VARCHAR(50),
    CONSTRAINT chk_no_self_delegation CHECK (debtor_player_id <> payer_player_id)
);

-- One *active* delegate per debtor. History rows (ended_at set) can accumulate freely.
CREATE UNIQUE INDEX uq_active_delegation_per_debtor
    ON payment_delegations (debtor_player_id) WHERE ended_at IS NULL;

CREATE INDEX idx_payment_delegations_payer ON payment_delegations (payer_player_id);

ALTER TABLE player_payments
    ADD COLUMN paid_by_player_id BIGINT REFERENCES players(id) ON DELETE SET NULL;
```

Notes:

- Delegations follow the ledger's audit posture: **never deleted, ended** (`ended_at`/`ended_by`).
  Who was responsible for whom last season is history, and history is not editable.
- `ON DELETE CASCADE` on both player FKs is safe for the same reason it is on the ledger tables:
  GDPR erasure anonymises in place, so the cascade never fires for a real person.
- `paid_by_player_id` is `ON DELETE SET NULL` — a payment record must survive its payer.
- The partial unique index is the guarantee that a debtor has at most one active payer; the service
  pre-check is an optimisation, same pattern as `uq_player_charges_plan`.

---

## 5. Business rules

| # | Rule | Notes |
|---|------|-------|
| BR-D1 | A debtor has **at most one active payer** | `uq_active_delegation_per_debtor`. Replacing = end the old row, insert a new one |
| BR-D2 | No self-delegation | `chk_no_self_delegation` |
| BR-D3 | **No chains, resolved or stored.** A player who is an active *debtor* cannot be assigned as a *payer*, and a player who is an active *payer* cannot be given a delegate | Enforced in-service (a CHECK cannot see other rows). One level keeps "who do I chase" a lookup, not a graph walk. Relaxing this later is additive |
| BR-D4 | Delegation never moves a ledger row | Charges stay on the debtor; payments stay on the debtor. Only `paid_by_player_id` records the hand that held the cash |
| BR-D5 | A delegation is ended, never deleted | Same as ledger voids: a record about money responsibility is an argument waiting to happen |
| BR-D6 | Ending a delegation changes nothing retroactively | Outstanding balances already sit on each debtor; the organiser simply goes back to chasing them directly |
| BR-D7 | An inactive or unlinked player can be a debtor **or** a payer | The ledger is player-keyed, never user-keyed (ledger §5.1); delegation inherits that. A payer with no account is unusual but legal — the organiser chases them off-app, as ever |

### Why the debtor is not asked

The organiser records arrangements the players already made between themselves — the same trust
model as `recordPayment`, where the organiser asserts money arrived. A consent flow would require
the debtor to have an account, which BR-D7 (and the guest feature this enables) explicitly does not
assume.

---

## 6. Entities

`PaymentDelegation` in `pt.rics.demo.football.model`, following the existing Lombok
`@Getter/@Setter/@Builder` shape, `@ManyToOne(fetch = LAZY)` on both `debtor` and `payer`,
plus the usual fully-qualified `@GeneratedValue(strategy = jakarta.persistence.GenerationType.IDENTITY)`
(the package has its own `GenerationType` enum — the trap is documented on `PlayerCharge`).

```java
@Transient
public boolean isActive() {
    return endedAt == null;
}
```

`PlayerPayment` gains `@ManyToOne(fetch = LAZY) @JoinColumn(name = "paid_by_player_id") Player paidBy;`
— nullable, meaning "the debtor themselves, or unrecorded".

`PaymentDelegationRepository`: `findByDebtorIdAndEndedAtIsNull`, `findAllByPayerIdAndEndedAtIsNull`,
`findAllActive` (fetch-joined for the balances query), `findAllTouchedBy(String)` for GDPR
audit-column scrubbing — the same quartet shape as the ledger repositories.

---

## 7. Service

Delegation logic lives in **`MatchFeeService`** — it is a fact about money responsibility, read in
the same breath as balances, and the bean already holds every repository the checks need. No new
bean, no proxy trap: nothing here is called from within another `@Transactional` method of the
same class.

```java
@Transactional
public DelegationDTO setDelegate(Long debtorId, Long payerId, String createdBy)
@Transactional
public void endDelegation(Long debtorId, String endedBy)
```

`setDelegate` validates: both players exist (404); not the same player (400); BR-D3 both directions
(409, message says which end of a chain was attempted); then ends any existing active delegation
for the debtor and inserts the new row — "replace" is one call, per BR-D1.

`recordPayment(...)` gains an optional `paidByPlayerId` pass-through (404 if unknown). It does
**not** validate the payer against active delegations — the organiser may record that Bruno paid
for Rui even where no standing arrangement exists. The delegation table says who is *responsible*;
`paid_by` says what *happened*.

`allBalances()` loads active delegations once and decorates each `PlayerBalanceDTO` with its
debtor-side delegate, if any. No second aggregate: "payer covers N" is derived by the caller from
the same list (§9).

---

## 8. API

All paths under `/api`, in `PaymentController`. Money domain ⇒ **`ORGANIZER`**, consistent with
every payment operation; `ADMIN` continues to appear nowhere in this controller.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `PUT` | `/players/{debtorId}/payment-delegation` | `ORGANIZER` | Body `{ "payerPlayerId": 7 }`. Create or replace the debtor's active delegation |
| `DELETE` | `/players/{debtorId}/payment-delegation` | `ORGANIZER` | End the active delegation (404 if none) |

`PUT` rather than `POST` because the resource is singular per debtor (BR-D1) and the call is
idempotent — re-asserting the same payer changes nothing.

| Status | Trigger |
|--------|---------|
| `400` | `payerPlayerId` missing, or equal to `debtorId` |
| `404` | Unknown debtor or payer; `DELETE` with no active delegation |
| `409` | BR-D3 — either party is already on the other side of an active delegation |
| `403` | Caller lacks `ORGANIZER` |

### DTO changes (all additive)

- `PlayerBalanceDTO` + `delegatePayerId` / `delegatePayerName` — present only when the row's player
  has an active payer. ⚠️ **Omitted when absent, not null** (`default-property-inclusion:
  non_null`) — frontend uses the `nullableField` helper.
- `PaymentDTO` + `paidByPlayerId` / `paidByPlayerName` — same omission rule.
- `RecordPaymentDTO` + optional `paidByPlayerId`.
- New `DelegationDTO(Long debtorPlayerId, Long payerPlayerId, String payerPlayerName, Instant createdAt)`
  as the `PUT` response, nested in `PaymentDTOs` like the rest.

**There is no new list endpoint.** "How many/which players does this payer cover" is a grouping of
the balances list the organiser already fetches — every covered debtor carries `delegatePayerId`,
so the client groups by it. Same posture as waitlist standing: derived, not stored.

### `GET /payments/balances` (unchanged path, richer rows)

```json
[
  { "playerId": 9,  "playerName": "Rui",    "balanceCents": -500,
    "contactable": false, "active": true,
    "delegatePayerId": 4, "delegatePayerName": "Bruno" },
  { "playerId": 4,  "playerName": "Bruno",  "balanceCents": -500,
    "contactable": true,  "active": true }
]
```

The organiser's arithmetic — "Bruno owes €5 of his own and answers for Rui's €5, so I ask Bruno for
€10" — is presentation, computed at the render edge like currency formatting. The API never sums
two people's money into one number.

---

## 9. Frontend (separate repo, `FootMania-Simple-Front`)

| Surface | Change |
|---------|--------|
| `BalancesTable.tsx` | Group covered debtors under their payer: payer row gains a "Paying for N" chip and an expandable indent of covered debtors; a covered debtor elsewhere in the list renders a "via {payer}" chip. Chip precedents already in this table: `contactable` → "No account", `active` → "Inactive" |
| `BalancesTable.tsx` row actions | `ORGANIZER`: "Set payer…" / "Remove payer" driving the two endpoints |
| `RecordPaymentModal.tsx` | When opened from a payer with covered debtors: list payer + covered debtors with owed amounts pre-filled, allocate the received sum across them, submit **N `POST /payments` calls, one per debtor**, each with `paidByPlayerId` set. When opened from an uncovered player: unchanged single-row flow, plus an optional "paid by" picker |
| `LedgerHistory.tsx` | Payment entries show "paid by {name}" when `paidByPlayerId` differs from the ledger's player |
| `paymentService.ts` / `usePayments.ts` | `setDelegate`, `endDelegation` mutations; existing `['payments']` prefix invalidation already covers every read |
| `docs/features/payments.md` | ⚠️ Missing since the ledger shipped (drift table). Write it as part of this change, covering the ledger UI *and* delegation |

> i18n: new keys under `payments.*` in **en/pt/es**. Do not `JSON.stringify` the locale files —
> splice the block in (ledger plan §12 has the anchor pattern).
> Visual snapshots: the payments views are not currently snapshotted, but confirm before assuming —
> `e2e/visual.spec.ts` is the list.

Nothing on `/payments/me` changes for a plain member: their own balance is their own balance,
whoever answers for it. Whether a debtor should *see* "Bruno covers you" is a group decision
(§12) — the data supports it later without schema change.

---

## 10. Privacy

`PRIVACY_AND_DATA_PROTECTION.md` requires three places, all three:

1. **Data table** — add `payment_delegations` (purpose: recording who answers for whose match
   fees; retention: kept ended, scrubbed on erasure) and `player_payments.paid_by_player_id`.
2. **`PrivacyService`** — export: a player's delegations (both directions, active and ended) join
   the export under `paymentDelegations`, and payments gain their `paidBy` reference — **in both
   `exportForUser` and `exportForPlayer`**, since a debtor with no account is only reachable via
   the second (the ledger plan's §11 lesson). Erasure: rows survive; `created_by`/`ended_by` are
   scrubbed via `findAllTouchedBy`, and the anonymised player name renders as `Deleted player #N`
   wherever a delegation or `paid_by` points at them — attribution decays, the record stands.
3. **`PrivacyServiceTest`** — export includes them; erasure does not remove them.

---

## 11. Test plan

| Area | Cases |
|------|-------|
| Lifecycle | set → replace (old row ended, new active) → end; `PUT` idempotent on same payer; `DELETE` with none → 404 |
| Constraints | second active delegation for a debtor rejected by the index (constraint-as-guarantee, service pre-check bypassed); self-delegation rejected |
| BR-D3 | payer-who-is-a-debtor rejected; debtor-who-is-a-payer rejected; both 409 |
| Payments | `paid_by` persisted and exported; lump-sum flow is N rows each on its own debtor, sums preserved; `recordPayment` with unknown `paidByPlayerId` → 404 |
| Balances | delegate fields present exactly on covered debtors; absent (not null) otherwise; sums unchanged by any delegation operation |
| Unlinked/inactive | BR-D7 — unlinked debtor and unlinked payer both work end to end |
| Security | non-`ORGANIZER` (including `ADMIN` and `MANAGER` holders) cannot touch either endpoint — proving the roles stay flat |
| Privacy | export includes delegations in both export paths; erasure scrubs actors, keeps rows |

> **CI invariant:** `build.gradle` asserts
> `testControllers + testServices + testSecurity + testApplication == test` — new classes land in
> one of those packages or the build fails on the count.

---

## 12. Decisions for the group

1. **Does a debtor see who covers them?** Default no (mirrors the balances-visibility default).
   One additive field on `/payments/me` if yes.
2. **Should a payer see the balances of the players they cover?** Default no — `/payments/balances`
   stays `ORGANIZER`. Worth revisiting if payers ask "how much do I owe for everyone".
3. **Notify the payer when a covered debtor is charged?** Default no for v1; the `FEE_CHARGED`
   push category still goes to the debtor (who may have no account, in which case nobody is
   notified — as today).

---

## 13. Order of work

1. `V20__payment_delegation.sql` + entity + repository; `MigrationSchemaValidationIT` green.
2. `MatchFeeService.setDelegate` / `endDelegation` / `recordPayment(paidBy)` / decorated
   `allBalances()` + service tests.
3. `PaymentController` endpoints + DTOs + security tests; **update
   `docs/api/PAYMENTS-API-CONTRACT.md` in the same commit.**
4. `PrivacyService` + data table + tests.
5. Frontend: service/hooks → `BalancesTable` grouping + actions → `RecordPaymentModal` payer flow →
   `LedgerHistory` → i18n ×3 → write `docs/features/payments.md`.

---

## 14. Breaking changes

- [x] **None.** One new table, one nullable column, additive DTO fields (omitted when absent),
      two new endpoints. Existing rows and every existing response keep their exact meaning.
