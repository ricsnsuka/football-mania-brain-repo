# Guest Players — Technical Specification

**Date:** 2026-08-01
**Status:** DRAFT (2026-08-01) — nothing built in either repo
**Priority:** HIGH — plans keep falling short of a full pitch, and today only a `MANAGER` can do anything about it
**Estimated Effort:** M (≈1 day backend, ≈1 day frontend)
**Depends on:** `PAYMENT-DELEGATION-PLAN.md` (V20) — a guest's fee is answered for by their inviter via an auto-created delegation. **Build that first.**
**Contract:** `docs/api/MATCH-PLANS-API-CONTRACT.md` and `docs/api/player-API-CONTRACT.md` (updated in the same backend commit)

---

## 1. Requirement Summary

There are times when members have to bring outsiders — a friend, a colleague, a ringer — just to
fill the spots for a match. Today that requires a `MANAGER` twice over: only a `MANAGER` can create
a `Player` (`POST /api/players`), and only a `MANAGER` can set someone else's confirmation
(`PATCH /api/match-plans/{id}/confirmations/{playerId}`). A plain member — the person who actually
knows the friend — can do nothing but send a WhatsApp message and hope a manager is awake.

Three product decisions drive this spec:

1. **Any member can bring a guest** to a plan that still has open spots, without a role grant.
2. **Guests are differentiated** from group members — in the schema, in every group aggregate, and
   in the UI. An outsider on the team sheet is not a member of the group.
3. **The inviter answers for the guest's match fee.** The guest owes on their own ledger row (the
   books stay per-player), but the standing payment delegation created with them makes the inviting
   member the person the organiser chases.

And because guests come back: a guest who keeps turning up can be **promoted** to a full member,
carrying every stat and debt they accumulated, because they were a real `Player` all along.

---

## 2. Scope

| In | Out |
|----|-----|
| Member-initiated guest creation, plan-scoped | Opening `POST /api/players` to non-managers |
| Guest differentiation (schema, aggregates, UI) | `isCore` tiering (roadmap Phase-3 row — overlaps Phase 5, deliberately untouched) |
| Auto-delegation of the guest's fees to the inviter | Any change to charge generation or the split rule |
| Inviter removing their own guest while the poll is open | Invite links, guest self-serve accounts, notifying guests |
| Promotion to member (`MANAGER`) | Auto-promotion on registration |
| Guest phone numbers: **refused** (§10) | Holding any guest contact detail in v1 |

---

## 3. Model decision: a guest is a real `Player`, flagged — not a new kind of thing

**Chosen:** guest = an ordinary, unlinked `Player` row plus two columns: `is_guest` (state) and
`invited_by_player_id` (provenance).

**Rejected:** a separate `guest_participants` table. Everything downstream of a confirmation —
team generation, `PlayerStat`, MVP votability, the fee ledger — requires a `players.id`. A second
identity type means re-plumbing all of it for a person who is, mechanically, just a player without
an account — a population the codebase already treats as first-class (ledger plan §5.1).

**Also rejected:** a `player_type` enum (`MEMBER`/`GUEST`/`CORE`/…). It pre-decides the Phase-3
"tiered invitations" question the roadmap explicitly says not to over-build before Phase 5. A
boolean asserts exactly one fact — *this person is not part of the group* — and composes with
whatever membership model Phase 5 brings (a guest is simply a player with no membership row).

**Why not a role?** The codebase's own argument (`Role.java` javadoc): "is a player" is a schema
fact, not a grant — and so is "is not one of us". Guests have no accounts, so there is nothing to
attach a role to anyway.

State and provenance stay separate on purpose: promotion clears `is_guest` and **keeps**
`invited_by_player_id` — who first brought someone along is history, and history survives
promotion.

---

## 4. Schema — `V21__guest_players.sql`

> Head after the delegation feature is V20. Same standing caveat: Flyway is disabled in tests, and
> per `architecture/database-migrations.md` V17–V19 have themselves never run against a real
> database — this stacks on unvalidated history. `./gradlew integrationTest` is the pre-flight.

```sql
ALTER TABLE players ADD COLUMN is_guest BOOLEAN NOT NULL DEFAULT FALSE;

ALTER TABLE players ADD COLUMN invited_by_player_id BIGINT
    REFERENCES players(id) ON DELETE SET NULL;

CREATE INDEX idx_players_invited_by ON players (invited_by_player_id);

-- A guest has no account by definition. Linking requires promotion first (BR-G8),
-- and the constraint makes the ordering non-negotiable at the data layer.
ALTER TABLE players ADD CONSTRAINT chk_guest_has_no_account
    CHECK (NOT is_guest OR user_id IS NULL);
```

- `DEFAULT FALSE` backfills every existing player as a member — correct by definition, no backfill
  script.
- `ON DELETE SET NULL` on the inviter: deleting (or erasing) an inviter must not cascade into the
  guest they once brought.

---

## 5. Business rules

| # | Rule | Notes |
|---|------|-------|
| BR-G1 | Any authenticated user **with a linked player** may add a guest to a plan | The link is the membership test, same as self-RSVP (`upsertConfirmation` throws "No player linked to your account"). An account with no player is a spectator |
| BR-G2 | Guests are added only while the plan is `PENDING` **and the poll is open** | Same window as self-RSVP, via `assertPollOpen`. Managers get no bypass on this path — past the deadline they already have `adminUpsertConfirmation` for existing players, and can create players outright |
| BR-G3 | Guests **fill empty spots only**: rejected when `confirmedCount >= TOTAL_PLAYERS_NEEDED` for the plan's match type | A member queues onto the waitlist because they benefit from a promotion later; a stranger on a waitlist benefits nobody. Guests never extend the queue |
| BR-G4 | At most **2 active guests per inviter per plan** | Enough for "I'll bring my two mates", not enough to pack the pitch with one member's entourage. Counted live: removing a guest frees the slot |
| BR-G5 | Guest names are deduplicated **per plan**, case-insensitively | Two members adding "João" to the same plan is almost certainly the same João — 409, talk to each other. No global dedupe: the group can know three Joãos across a season |
| BR-G6 | Creation is atomic: guest `Player` + `CONFIRMED` confirmation + delegation (debtor = guest, payer = inviter), one transaction | A guest exists *because* they are coming to this match. No half-created guests |
| BR-G7 | Guests join the **same** `confirmedAt` starter queue as everyone else — no bumping | By BR-G3 a guest occupied a genuinely empty spot at invite time; a member confirming later takes the next rank like anyone. Bumping would silently uninvite a real person who was promised a game, and would fork the single ordering that `withWaitlistStanding` and `selectStarterPlayerIds` deliberately share |
| BR-G8 | A guest row cannot be linked to an account. Promotion (`is_guest → false`) must happen first | `chk_guest_has_no_account` + a guard in `linkMe`. Otherwise anyone who registers could claim a guest row and become a member without anyone deciding that |
| BR-G9 | Promotion is `MANAGER` — joining the group is a roster decision | Flag flips to `FALSE`; `invited_by_player_id` stays; the delegation is **not** auto-ended — money responsibility changes only when the organiser says so, explicitly |
| BR-G10 | Guests are **excluded from every group aggregate**: rankings, leaderboards, top-scorer/assist/streak lists, badge scans, and the scarcity averages | §7. Their *own* stats still accumulate (BR-G11) |
| BR-G11 | A guest's own `skillRating`, streaks and career totals are maintained normally | Repeat guests should be generated onto fair teams, and promotion should carry earned history. `CalculationService`'s per-player updates are not an aggregate and are left alone |
| BR-G12 | The guest owes their own fee; the inviter answers for it | The charge row stays on the guest (`uq_player_charges_plan` and BR-8 of the ledger plan make anything else wrong); the auto-delegation makes the inviter the one the organiser chases. `generateChargesFor` changes not at all |

### Removal

The inviter may remove **their own** guest while the poll is open; a `MANAGER` may remove any guest
until the plan is `GENERATED`. Removal deletes the confirmation, then:

- if the guest `Player` has no `player_stats` and no other confirmations (the overwhelmingly normal
  case — invited, never played, plan changed): **hard-delete** the row and end the delegation. A
  person who never existed in any record can actually be removed; there is no history to protect.
- otherwise (a repeat guest with played matches): deactivate, end the delegation for the removed
  plan's sake only if no other active confirmations remain. Their history stands, like any player's.

This mirrors the existing `PlayerService` posture: hard delete is blocked once `player_stats`
exist; before that it is honest.

---

## 6. API

### `POST /api/match-plans/{planId}/guests` — `isAuthenticated()`

The gate is deliberately **not** a role. Everything interesting is state, not identity, so the
checks live in the service (`@PreAuthorize` cannot express them — the same conclusion
`DraftSessionService` reached, documented at its line 293):

caller has a linked player (BR-G1) → poll open (BR-G2) → spots remain (BR-G3) → inviter under cap
(BR-G4) → name unique on plan (BR-G5) → create all three rows (BR-G6).

```json
{ "name": "João (Rui's friend)", "baseSkillRating": 6, "notes": "coming straight from work" }
```

- `name` required, 2–100, same validation as `PlayerCreateDTO`.
- `baseSkillRating` optional 1–10, **default 5** — the inviter is guessing anyway; 5 matches the
  `skillRating` column default. A manager can correct it later via the normal player update.
- `notes` optional, stored on the `PlayerConfirmation` (500), where RSVP notes already live.
- No `phoneNumber`, by design (§10). No `userId`, by constraint.

Returns `201` with the enriched `PlayerConfirmationDTO` (below).

| Status | Trigger |
|--------|---------|
| `400` | Caller has no linked player; validation failures |
| `404` | No such plan |
| `409` | Poll closed / plan not `PENDING`; no spots left; inviter cap reached; duplicate guest name on this plan |

### `DELETE /api/match-plans/{planId}/guests/{playerId}` — `isAuthenticated()`

In-service: target is a guest with a confirmation on this plan (404 otherwise); caller is the
inviter (poll open) or holds `MANAGER` (plan not `GENERATED`); otherwise `403`. Behaviour per
"Removal" above. `204`.

### `POST /api/players/{id}/promote` — `hasRole('MANAGER')`

`409` if the player is not a guest. Flips the flag, evicts `PLAYERS`/`RANKINGS`/`LEADERBOARDS`
caches (the player just became visible to all three), returns the updated `PlayerDTO`.

### Existing endpoints, touched

- `PATCH /api/match-plans/{id}/confirmations/{playerId}` (`MANAGER`) works on guest rows unchanged
  — a manager can decline a guest without removing them.
- `POST /api/players/{id}/link-me` gains the BR-G8 guard: `409` with "This player is a guest —
  ask a manager to promote them first" before the existing three conflict checks.
- `POST /api/players` (`MANAGER`) is untouched. A manager creating a player creates a member, as
  today. (A manager who wants to add a guest uses the plan-scoped endpoint like everyone else.)

### DTO changes (all additive)

- `PlayerDTO` + `isGuest`, + `invitedByPlayerId` / `invitedByPlayerName` ⚠️ omitted when absent,
  not null (`non_null` inclusion).
- `PlayerConfirmationDTO` + `isGuest`, + `invitedByPlayerId` — the plan modal's member branch never
  loads the roster (`usePlayers` is `enabled: canOverride`), so the confirmation row must carry
  what the chip and the remove button need on its own.
- New `GuestCreateDTO(name, baseSkillRating, notes)`.

---

## 7. Keeping guests out of the group's numbers

The one genuinely dangerous ripple. These queries scan the whole `players` table and every one of
them gets an `AND p.is_guest = FALSE` (or `isGuest = false` JPQL) guard:

| Query | Surface | Why it matters |
|-------|---------|----------------|
| `findActiveForRanking` (and the including-deactivated variant) | Rankings | A ringer with one match would enter the league table |
| `findTopScorers` / `findTopAssisters` / `findLongestStreaks` | Leaderboards | Same |
| `findAverageGoalsPerMatch` / `findAverageAssistsPerMatch` | **Scarcity multipliers** | The quiet one: these averages feed every rating calculation, so one guest's numbers would move **every member's** skill rating |
| `BadgeService` player scans | Badges | A guest cannot cross a threshold into the group's trophy cabinet |

Left alone, deliberately:

- `CalculationService` per-player updates (BR-G11) — not aggregates.
- `MvpVoteService` — a guest is votable-for (they played; denying the crowd its MOTM because the
  best player was a ringer is wrong) and cannot vote (no account — already enforced by
  `participantIds` + authentication, no new code).
- `PushNotificationService` — already skips accountless players as "normal, not an error". The
  ledger plan's warning stands: **do not** add a guard that throws.
- `MatchFeeService.generateChargesFor` — BR-G12. The guest is a confirmed player; the split
  headcount is correct precisely because they are counted.
- Team generation — the guest's `skillRating` participates in balancing. That is the point of
  collecting `baseSkillRating` at invite time.

The regression test that keeps this honest: create a guest, play them through a generated match,
and assert `findAverageGoalsPerMatch`, the rankings list and the leaderboards are **byte-identical**
to the run without the guest.

---

## 8. Promotion — the payoff of §3

`POST /api/players/{id}/promote`, and the row simply stops being a guest. Because the guest was a
real `Player` from day one:

- stats, `skillRating`, streaks and career totals carry over — earned, not migrated;
- every ledger row already sits on their `player_id`; the day they register an account and
  `link-me`, their history and their debts become visible to them (ledger §5.1's "linking later is
  free" holds verbatim);
- they enter rankings/leaderboards from the next cache eviction, with their real record.

The UI makes promotion discoverable rather than buried: a guest's row in `PlayerModal` shows
`totalMatchesPlayed` as "came N times" next to the Promote action — the manager's cue that a
regular should stop being a guest.

Sequencing, fixed by BR-G8/BR-G9: **promote → register → link.** The delegation outlives promotion
until the organiser ends it (BR-D6 makes ending it a no-op on the books).

---

## 9. Frontend (separate repo, `FootMania-Simple-Front`)

| Surface | Change |
|---------|--------|
| `MatchPlanDetailModal.tsx` — member branch | The branch's **first write affordance beyond self-RSVP** (today it is deliberately read-only — this spec is the decision to soften that): a "Bring a guest" button, visible while `pollOpen` and `playersNeeded > 0` (both server-derived — do not re-derive), opening `AddGuestModal`; a remove button on confirmations whose `invitedByPlayerId` is the caller's own player id |
| `MatchPlanDetailModal.tsx` — both branches | "Guest" chip on guest confirmation rows — `.mpc-guest-chip`, sitting beside the status chip exactly as `.mpc-waitlist-chip--*` already does |
| New `AddGuestModal.tsx` | Clone of `CreatePlayerModal` minus phone/active/userId: name, optional skill guess (default 5), optional note. react-hook-form + zod, error messages as i18n key suffixes |
| `PlayersPage.tsx` | Guest chip in table + mobile card; guests **hidden by default** behind a "Show guests" toggle — the page is the group's roster, and guests are not the group |
| `PlayerModal.tsx` | Guest chip in header; "came N times" line; Promote action (`MANAGER`), confirm-dialog like status changes |
| Types/services/hooks | `isGuest`/`invitedBy*` through the zod schemas; `addGuest`/`removeGuest`/`promotePlayer` service calls + mutations. Invalidate `['players','all']` **and** the plan's confirmations key on add/remove — a fresh guest must appear in the canonical players cache or `TeamsRosterView` renders their rating as "—" |
| `globals.css` | `.mpc-guest-chip`, `.players-guest-chip` in `@layer components`, BEM-ish, both themes |
| i18n | New keys in **en/pt/es** (`matchPlans.guests.*`, `players.guest*`). Splice, don't rewrite |
| Visual snapshots | `players`, `player-modal`, `match-plans` are snapshotted in light+dark — `npm run test:visual:update` after the chips land |
| `docs/features/` | Update the frontend `players` and match-plans docs in the same commit as the UI |

The organiser's side needs nothing here: the delegation feature's `BalancesTable` grouping already
shows the guest indented under their inviter, "No account" chip and all.

---

## 10. Privacy — name-only guests

A guest's personal data is entered by **someone else**, about a person who has agreed to a football
match but not to an app. Minimise accordingly:

- **v1 holds a name and a skill guess. `phoneNumber` is refused** (rejected by `GuestCreateDTO`
  having no such field, and documented so nobody adds it casually). The lawful-basis analysis in
  `PRIVACY_AND_DATA_PROTECTION.md` leans on "you joined the group" — a guest joined nothing, so
  hold nothing beyond what a team sheet needs. Inviters can suffix the name ("João (Rui's friend)")
  — a name they chose to write is theirs.
- This also dissolves the `PlayerPiiPolicy` gap discovered in design: the policy has no inviter
  concept, so the inviting member could not see the number they themselves typed. Rather than
  extend the policy, hold no number.
- `PrivacyService.exportForPlayer` / `erasePlayer` already work on any player row, guest or not —
  a guest who invokes GDPR rights is served by the existing paths. Erasure anonymises in place;
  `invited_by_player_id` pointing at an erased inviter renders as `Deleted player #N`.
- Data table gains the two new columns with purpose and retention.

The delegation created for a guest is covered by the delegation plan's §10.

---

## 11. Test plan

| Area | Cases |
|------|-------|
| Migration | `MigrationSchemaValidationIT` green; `chk_guest_has_no_account` rejects a linked guest row |
| addGuest | Happy path creates player + CONFIRMED confirmation + active delegation in one transaction (rollback proves atomicity); 400 no linked player; 409 poll closed / plan CANCELLED / full (BR-G3 boundary: last spot succeeds, next fails) / cap (third guest fails, removal frees) / duplicate name case-insensitively |
| removeGuest | Inviter within poll → 204 + hard delete + delegation ended; inviter after deadline → 403; non-inviter member → 403; `MANAGER` pre-generation → 204; guest with `player_stats` → deactivated, not deleted |
| Queue | Guest ranks by `confirmedAt` like anyone; a later member lands behind an earlier guest; `selectStarterPlayerIds` unchanged |
| Generation & fees | Guest drafted, `PlayerStat` created, charged on their own row; balances show the inviter as delegate payer; the split still sums exactly |
| **Aggregate isolation** | The §7 regression: rankings, leaderboards, both scarcity averages and badge output identical with and without a guest who has played |
| Promotion | `MANAGER` only; 409 on non-guest; caches evicted; player appears in rankings with carried stats; delegation still active after; `linkMe` 409 before promotion, succeeds after |
| Privacy | Guest create carries no phone; export/erasure paths work on a guest row |
| Security | Member cannot reach `POST /api/players`, `adminUpsertConfirmation` or `promote` — the new endpoint widened exactly one door |
| Frontend | "Bring a guest" hidden when poll closed or `playersNeeded == 0`; remove button only on own guests; chips render; i18n keys in all three locales; visual snapshots updated |

> CI invariant on test-package counts applies, as ever.

---

## 12. Order of work

1. **Ship `PAYMENT-DELEGATION-PLAN.md` first** (V20). This spec assumes it.
2. `V21__guest_players.sql` + `Player` fields + `MigrationSchemaValidationIT`.
3. Repository guards (§7) + the aggregate-isolation regression test — before any endpoint exists,
   so a guest row can never have contaminated a number.
4. Service: `addGuest` / `removeGuest` in `MatchPlanService` (it owns plan state and the capacity
   map), `promote` + `linkMe` guard in `PlayerService`. Delegation calls go through
   `MatchFeeService` — separate bean, so `@Transactional` proxying stays honest.
5. Controllers + DTOs + security tests; **both contract files in the same commit**.
6. Frontend in the §9 order; visual snapshots last.
7. Brain repo on ship: move this to SHIPPED with commit refs, feature-status row to ✅, migration
   table row for V21, drift check on `MATCH_PLANS_FEATURE.md` (already stale — do not make it
   staler by documenting guests only here).

---

## 13. Breaking changes

- [x] **None on the wire.** Two nullable columns (default FALSE / NULL), additive DTO fields
      omitted when absent, three new endpoints, one new 409 on `link-me` for rows that could not
      previously exist.
- [ ] **Two deliberate behaviour changes**, called out rather than hidden:
      1. The member view of a match plan gains its first write affordance beyond self-RSVP.
      2. Ranking/leaderboard/scarcity queries gain a `WHERE` clause — a no-op on every existing
         row (`is_guest` defaults FALSE), but the queries are no longer literally "all players".
