# Rank Ladder — Tiers and Form Points in place of the raw rating

**Date:** 2026-09-04
**Status:** 🟡 **Step 1 merged into `next` — [FootMania-Back#275](https://github.com/ricsnsuka/FootMania-Back/pull/275)
(2026-09-04), not released, not deployed.** The engine, V50, the replay hook, the
simulation test and the calibration SQL; nothing reads the ladder yet and no contract changed.
Steps 2–5 unbuilt. Owner decisions taken 2026-09-04 (§9), including the five edge-case decisions
in §10. Every constant except the 75 FP division is a placeholder until the step 1 replay (§8)
reports the real rating distribution.

**What building step 1 added to the spec** (not known when §6 was written):
- The replay's exactness rule is per row, not per run: the reverse restores a row's ladder
  snapshot only when nothing later stands on the player's chain
  (`SkillRatingHistoryRepository.existsLaterMovement`), which inside a whole-set replay is always.
  A single-match recalculation of an old match nets the row's movement off the re-apply instead,
  the rating's own approximation on that path. Deletion needs nothing: it already unwinds to the
  end of every chain it touches.
- The seed is taken once. A row's "before" records the seeded rung, so a later replay restores to
  it rather than seeding again from a different rating — which means **recalibrating the bands
  does nothing until the snapshots are wiped**. The calibration SQL carries the reseed block.
- The single-match path has no way to know a transition row's reset should be recomputed unless
  its snapshot was restored, so the row carries a transient `ladderRestored` flag from the lift
  to the re-anchor. The first replay after V50 recomputes on "no snapshot" instead.
**⚠️ Changes the seasons feature:** starting a season now finalises the one running (§4, §10.2).
The seasons contract, the start modal and [FE seasons](../../frontend/features/seasons.md) all
change with it.
**Rendered proposal:** [claude.ai artifact 994f0454](https://claude.ai/code/artifact/994f0454-dfb5-498a-918d-deb8a89df3cb)
(Rev 3). This file is the canonical copy; the artifact is the pretty one.
**Target release:** `3.5.0`, off `next` — additive API, new feature, no breaking change.
**Estimated effort:** L. Backend ≈3 days (engine, migration, replay, contract), frontend ≈3 days
(chip, rankings, ladder chart, admin form). The halves are independent behind the tenant switch.
**Depends on:** the whole-set replay unwind from `3.4.2`
([FootMania-Back#272](https://github.com/ricsnsuka/FootMania-Back/pull/272)) — see §6.
Supersedes the FEAT-5 rating history chart's *view*, not its data
([backlog-2026-09 §FEAT-5](../../product/backlog-2026-09.md#feat-5-built-2026-09-03--what-the-spec-did-not-know)).
**Contract:** `docs/api/RANKING-TIERS-API-CONTRACT.md` — written in the same commit as the
endpoints, per rule 2 in [CONTRIBUTING.md](../../CONTRIBUTING.md).

---

## 1. What changes, in one paragraph

Today the league table sorts players by `skillRating`, a 1–10 number that moves 20–35% of the way
toward each match rating ([CALCULATION_SERVICE](../features/CALCULATION_SERVICE.md)). The number is
honest but unreadable: 6.83 versus 6.91 means nothing to a player, and it can fall after a good
night if the match rating still sat below the average. This plan keeps that number, stops leading
with it, and puts a ladder in front of it. Every completed match pays or charges **Form Points
(FP)**. FP fills a division, divisions fill a tier, and the tier is what the player sees, shares
and chases. The raw rating is one hover away, never gone.

This is the split most competitive games use: a visible ladder driven by a hidden rating. The
hidden rating keeps the ladder honest, the ladder makes progress legible.

**Why "Form Points".** XP was rejected by the owner. Form is football's own word for how a player
is playing right now, and the points move with exactly that: results and performance, never
attendance. FP abbreviates cleanly in a chip and reads the same in Portuguese and Spanish.

## 2. The ladder

Seven tiers and one title. Six tiers have three divisions of **75 FP** each, read downward like the
games do: III is the entry, I is the door to the next tier. Master has no divisions, and the best
Masters hold **Best of the Crop**.

| Tier | Divisions | Anchor band on `skillRating` | Tier centre |
|---|---|---|---|
| Iron | III · II · I | < 4.5 | 4.0 |
| Bronze | III · II · I | 4.5 – 5.5 | 5.0 |
| Silver | III · II · I | 5.5 – 6.25 | 5.9 |
| Gold | III · II · I | 6.25 – 7.0 | 6.6 |
| Platinum | III · II · I | 7.0 – 7.75 | 7.4 |
| Diamond | III · II · I | 7.75 – 8.5 | 8.1 |
| Master | none, open FP | ≥ 8.5 | 8.9 |
| Best of the Crop | title, derived | — | — |

The anchor band is what §3's anchor term reads. **It is not a threshold that moves a player by
itself**: tiers move only when a match completes, a season resets, or a recalculation replays.

### Rules of the rungs

- **75 FP per division.** Reaching 75 promotes to the next division. The overflow carries over,
  capped at 20, so a huge night doesn't skip a rung.
- **Falling below 0 demotes** one division and lands the player on 40 FP, so one bad night doesn't
  bounce them straight back down.
- **Promotion shield.** For the 3 matches after any promotion FP floors at 0 instead of demoting.
- **Master has no ceiling.** FP keeps counting, and that count orders Masters among themselves.
- **Best of the Crop is a title, not a rung.** After every completed match the top Masters by FP
  hold it: **3 seats in a group under 30 active players, 10% of the group from 30 up** (30 → 3,
  50 → 5). With fewer Masters than seats, only those hold it; with no Masters, nobody does. It
  cannot be earned and kept, only held. "Active" is the set the rankings return without
  `includeInactive`.
- **Iron III floors at 0.** Nobody falls off the ladder.

Iron–Diamond give 18 rungs of 75. A season is roughly 20–30 matches per regular, so **the
calibration target is: a player genuinely better than their rung moves three or four divisions in
a season, and a player at their level hovers.** Every constant in §3 serves that.

## 3. What a match pays

Result sets the base, the player's own performance sets the size, and three multipliers decide how
fast the ladder moves for this player at this point in their career.

```
result  = WIN +15 · DRAW +5 · LOSS −10
form    = 8 × (matchRating − 6.0)        // the existing per-match rating, 1–10, base 6.0
raw     = result + form

raw > 0:  ΔFP = raw × tierGain[tier] × pace × (1 + anchor)
raw ≤ 0:  ΔFP = raw × 1.0            × pace × (1 − anchor)

tierGain = Iron 1.50 · Bronze 1.35 · Silver 1.20 · Gold 1.00 · Platinum 0.85 · Diamond 0.70 · Master 0.60
pace     = max( placement ? 2.0 : 1.0 ,  1 + 0.5 × max(0, 1 − careerMatches / 20) )
anchor   = clamp( (skillRating − tierCentre) / 2 ,  −0.5 , +0.5 )
```

### Why these pieces

- **The match rating already encodes the night.** Goals on the escalating ladder, assists, timing,
  decisiveness, the team-defence term, own goals. Reusing it means the ladder rewards the same
  things the rating engine has been tuned to reward, and nothing has to be scored twice.
- **Result still matters on its own.** Without the result term, a brace in a 2–5 loss would outpay
  a clean-sheet win. With it, a hero on a losing side comes out roughly even and a passenger on a
  winning side gains, but never as much as a contributor.
- **Gains shrink with tier, losses do not.** That is the "gradually gaining less" the idea asked
  for, and it is also what makes Diamond hard to *hold*, not just hard to reach. An Iron player
  gains 1.5× what a Gold player gains for the same night; a loss costs both the same.
- **Pace is the new-player and placement boost.** A debutant moves at 1.5×, fading to 1× by their
  20th career match. Placement matches move at 2× regardless. The larger of the two applies, they
  never stack.
- **Anchor is the hidden rating pulling the ladder toward the truth.** A Gold II player whose skill
  rating says Platinum gains up to 1.5× and loses as little as 0.5×, and climbs. A Gold II player
  whose rating says Silver gains less and loses more, and drifts down. Without this term a ladder
  is pure grind; with it, FP converges to where the engine thinks the player belongs, while form
  and results still decide the pace.

### Worked nights

Veteran player, neutral anchor. Match ratings are the typical outcomes documented in the rating
engine. A division is 75, so the top row is most of a division in one night at Iron and under half
of one at Diamond.

| Night | Rating | Raw | Iron | Gold | Diamond |
|---|---:|---:|---:|---:|---:|
| Hat-trick, clean-sheet win | 9.6 | +43.8 | +66 | +44 | +31 |
| Brace on a 3–1 win | 8.9 | +38.2 | +57 | +38 | +27 |
| Passenger on a 2–0 win | 7.0 | +23.0 | +35 | +23 | +16 |
| Passenger on a 7–6 win | 6.2 | +16.6 | +25 | +17 | +12 |
| Passenger on a 0–0 draw | 6.6 | +9.8 | +15 | +10 | +7 |
| Sole scorer's brace on a 2–3 loss | 8.0 | +6.0 | +9 | +6 | +4 |
| Passenger on a 0–2 loss | 4.9 | −18.8 | −19 | −19 | −19 |
| Own goal, 1–4 loss | 4.0 | −26.0 | −26 | −26 | −26 |

Two Gold II players, both passengers, one win and one loss each. The anchor is the only difference:

| Player | Skill rating | Anchor | Win night | Loss night | Net |
|---|---:|---:|---:|---:|---:|
| Rated above Gold (Platinum grade) | 7.4 | +0.40 | +32 | −11 | +21 |
| Rated at Gold's centre | 6.6 | 0.00 | +23 | −19 | +4 |
| Rated below Gold (Silver grade) | 5.9 | −0.35 | +15 | −25 | −10 |

> ⚠️ **Every constant in this section is a placeholder.** The tier bands in particular assume
> ratings cluster between 5.5 and 8, which the engine's defaults suggest but nobody has measured.
> Step 1 of §8 replays the real history with the ladder switched off and prints the resulting
> tier distribution. Tune bands and multipliers against that before anyone sees a tier.

## 4. Placements and seasons

The first **three completed matches of every season are placement matches**. That is already the
number of matches the table requires before it gives a player a position
(`RankingService.MINIMUM_MATCHES_TO_QUALIFY = 3`), so the concept replaces "unqualified" rather
than adding to it.

### Finalise: the soft reset

The reset runs inside **finalise**, on the same per-player transition row that carries the rating
movement. Every player drops, active or not, played or not — a season away is a season of no form,
and two seasons away is two drops (§10.3). How far depends on the tier, and where they land inside
the new tier depends on the FP they held.

**Starting a season finalises the one running** (owner decision, 2026-09-04 — see §10.2). Today
start only clears the current flag and the displaced season's ratings are never taken; that
"starting is not finalising" rule goes. The one exception: a displaced season with **no completed
match** is displaced only, nothing to take. The standalone finalise endpoint stays, for a group
that wants a break between seasons.

| Was | Lands in | Which division |
|---|---|---|
| Master (Best of the Crop included) | Platinum | By FP held: 150+ → I, 75–149 → II, under 75 → III |
| Diamond I / II / III | Gold | Same division as held: Diamond II → Gold II |
| Platinum I / II / III | Silver | Same division as held |
| Gold, Silver, Bronze, Iron | Up to 3 divisions lower | By FP in the division: 0–24 → 3 divisions, 25–49 → 2, 50–74 → 1. Floors at Iron III |

FP inside the division is carried over in every case (Masters carry FP modulo 75), so a player who
was close to promoting stays close after the drop. Two Masters, one on 40 FP and one on 190, start
the season in Platinum III and Platinum I. Two Gold I players, one on 10 FP and one on 60, start in
Silver I and Gold II.

**Why FP decides the drop.** A Gold I on 74 FP and a Gold I on 1 FP are not the same player; the
first was one good night from Platinum. Reading the owner's rule as "more FP, smaller drop" keeps
that difference through the reset instead of flattening it. The rejected reading — a fixed
three-division drop with FP deciding only the landing offset — is simpler but lands both in the
same division. Confirmed by the owner 2026-09-04.

### Placements

1. **Hidden.** The player's tier is hidden and the table shows *Placement 1 of 3*. FP moves at 2×
   pace and promotions and demotions apply silently underneath.
2. **Reveal.** After the third completed match the tier is shown, with a "Placed" moment and a
   notification. Three decent nights at 2× recover most of what the reset took; three poor ones
   confirm it.

A brand-new player has nothing to reset. Their seed comes from the skill rating the admin assigned,
mapped onto the tier band with FP spread linearly over that tier's 225, and their pace is 2×
through placements then the 1.5× debutant fade.

### Season end

- The final tier, division and FP are frozen into the season's standings. Past-season rankings
  already read the last history row per player; this adds the ladder fields to that row.
- "Highest tier reached" and "Biggest climb" are natural additions to the stored season awards
  (V37). Follow-up, not scope.
- The season-end rating formula is untouched. The reset above replaces the earlier idea of
  re-seeding from that rating; the rating only feeds the anchor.

## 5. The progression chart

The rating chart on the profile (FEAT-5) becomes a ladder chart: the player's tier and FP over
time, with the raw rating available on hover.

- **The line is ladder position**, the player's absolute standing in FP counted from Iron III at 0
  (`tier × 225 + divisionsClimbed × 75 + FP`). FP on its own resets at every promotion and would
  saw-tooth; the absolute position is what climbs.
- **The y-axis is labelled in tiers and divisions, not numbers.** Each tier is a coloured band,
  each division a faint gridline. Nobody needs to know that Gold II starts at 750.
- **Promotions and demotions are markers** on the line, in the same green and red the table uses.
  Hovering a point shows the match, the FP change, the tier, and the raw rating before and after.
- **Season boundaries are dashed verticals.** The soft reset shows as the drop it is; the placement
  matches that follow are drawn dashed until the reveal.
- The FEAT-5 view and its open backlog fixes are superseded by this one. The raw-rating series
  stays reachable as a toggle for admins, and nothing about the history rows is lost — the ladder
  columns are added to the same rows FEAT-5 reads.

## 6. Fitting it into the codebase

The pieces already exist in the right shape: a per-match rating, an ordered history table, an
in-order replay for recalculation, seasons with transition rows, notifications, badges. The ladder
is a second quantity carried through the same pipeline.

### Data model

| Table | Change | Notes |
|---|---|---|
| `players` | + `rank_tier` smallint, `rank_division` smallint (null for Master), `form_points` int, `rank_shield_left` smallint, `season_placements_played` smallint | Current state lives beside `skill_rating`, exactly as the rating does today. Best of the Crop is derived at read time from active-player count and the Masters' FP, never stored |
| `skill_rating_history` | + `fp_before`, `fp_after`, `tier_before`, `division_before`, `tier_after`, `division_after`, `rank_event` (PROMOTED, DEMOTED, PLACED, SEASON_RESET, null) | One row per player per match already exists. Extending it keeps rating and ladder movement in one record and gives the chart its series. The per-player **season-transition row** (match null, written by finalise) carries the reset: `*_before` is the pre-reset state, `*_after` the post-reset state, `rank_event = SEASON_RESET` |
| Migration | `V10__rank_ladder.sql`, nullable columns with a backfill in step 1 | Additive. Rolls out under a tenant setting, so the columns can exist before the feature does. Record it in [database-migrations](../../architecture/database-migrations.md) |

### Engine

- A new `RankLadderService` applies ΔFP after `CalculationService` has written a player's match
  rating for a completed match. It is the only writer of the ladder columns.
- **Recalculation must replay the ladder in order.** ΔFP depends on state at the time (tier
  multiplier, shield, placement count, career matches), so the reverse pass cannot subtract deltas
  the way it does for the rating. It restores each player's `*_before` snapshot from the earliest
  unwound row, then the forward pass re-applies. The whole-set unwind from 3.4.2 (#272) is the
  prerequisite that makes this safe; without it a partial unwind would restore a snapshot that
  later rows still assume.
- **Transition rows are half carried, half recomputed.** The replay already lifts a transition row
  inside a player's span and re-anchors its rating movement on the way forward, because that
  movement averaged the whole group at a moment that has passed. The ladder half of the same row
  is deterministic from the FP the rebuilt chain has reached, so the re-anchor **recomputes** the
  reset (§4) rather than carrying it.
- **Deleting or amending a completed match needs no ladder rule.** `MatchService` already unwinds
  that match plus everything after it, newest first, deletes or amends, and re-applies the rest.
  The ladder rides that span like any replay.
- **Guests are not special-cased in the engine** (§10.3). Their ladder is maintained like their
  rating; only the reads hide it.
- Admin edits to the base rating and manual rating adjustments do not move tiers. They change the
  anchor, which changes what the next matches pay.
- The ladder is switched per tenant: `ranking.mode = RATING | TIERS` in platform settings. In
  RATING mode the columns are still maintained, so switching on is instant and switching off loses
  nothing.
- Best of the Crop seats: `activePlayers >= 30 ? floor(activePlayers × 0.10) : 3`.

### API, additive

```
GET /api/rankings                       // entries gain:
  "rank": {
    "tier": "GOLD", "division": 2, "formPoints": 48, "pointsToNext": 27,
    "placement": { "played": 3, "required": 3 },
    "shieldMatchesLeft": 0,
    "lastChange": { "formPoints": 27, "event": null },
    "bestOfCrop": false
  }
  // sort: tier desc, division asc (I above III), formPoints desc, then skillRating, then played
  // skillRating stays in the payload: the chip hover, team generation and the admin form read it

GET /api/players/{id}/rating-history    // rows gain fpBefore, fpAfter, tierBefore, divisionBefore,
                                        // tierAfter, divisionAfter, rankEvent; the chart reads these
GET /api/rankings/ladder                // optional: tiers, divisions, bands and the division size,
                                        // so the UI does not hardcode them
NotificationCategory RANK               // promoted, demoted, placed, Best of the Crop gained or lost
```

Contract file `docs/api/RANKING-TIERS-API-CONTRACT.md` and an entry in
`docs/frontend/FRONTEND_ENDPOINT_CHANGES.md` ship in the same commit as the change.

### Frontend

- **Rankings page** leads with the tier chip and division. Sorting, qualification and the streak
  columns stay.
- **The tier chip is one component**, used on the rankings page, profile card, player modal,
  players list and match rosters. Hovering it shows the raw rating; on touch, a tap-and-hold does
  the same. That is the only place the number appears for regular players.
- **Team generation, balance gauge, captain pick** keep using `skillRating` and keep showing it
  plainly. Balancing on tiers would throw away resolution for no gain.
- **Admin player form** keeps the rating field visible and editable as today, with the current tier
  shown read-only beside it.
- **Profile chart** becomes the ladder chart of §5.
- Tier names and "Best of the Crop" stay in English across `en`, `es` and `pt` as proper nouns.
  "Placement", "promoted", "demoted" and the FP label are translated — in all three locales, per
  the ship checklist.

## 7. Rollout

1. **Engine, migration, replay backfill, switched off.** Ship the service and columns under
   `ranking.mode = RATING`. The backfill is **the existing bulk recalculation with an empty body,
   called once per group** (§10.1) — no new tooling. Run it on a staging copy first and read the
   calibration SQL (§8); calibrate bands and multipliers there, on real matches, before anyone
   sees a tier. Verify with a full `./gradlew build`, never `test` alone.
2. **API and contract.** Additive fields on rankings and rating history, the notification category,
   the contract file and the frontend changelog entry. **Plus the seasons change** (§10.2): start
   finalises, the seasons contract's "Starting is not finalising" section is rewritten, and the
   start modal's warning says what will now happen — ratings taken, awards computed, poll opened,
   ladder reset — in all three locales.
3. **Rankings, chip, chart and profile UI.** The rankings page, the shared tier chip with its hover,
   the ladder chart, the admin form. There is no preview environment, so the owner checks the UI
   locally before the release is cut.
4. **Switch on, release 3.5.0.** Flip the tenant setting; everyone is already placed by the
   backfill, so day one shows a full ladder rather than 30 players in placements. Backend deploys
   before the frontend, and merged is not deployed.
5. **Later.** Tier badges on the existing badge system, season awards for highest tier and biggest
   climb.

## 8. Calibration, before anyone sees a tier

The step 1 replay answers the questions this document cannot:

- Where do ratings actually cluster? If the bulk sits in 6.0–7.5, the Silver/Gold/Platinum bands
  are too narrow and Iron and Master are empty.
- How many divisions does a typical regular move in a season at these constants? The target is
  three or four for a player above their rung, roughly zero for a player at it.
- Does the anchor dominate? If two players with the same nights end up three divisions apart
  purely on `skillRating`, the ±0.5 clamp is too wide.

**The report is SQL on a staging copy** (owner decision, §10.5): a script in `scripts/database/`
run after the backfill on a production copy made with the existing `copy-prod-to-staging.ps1`. No
API surface, easy to iterate, and the ladder columns on `players` and `skill_rating_history` hold
everything it needs. It prints, per group:

- tier and division counts for qualified players, and the raw-rating histogram against the bands;
- per player per season: net FP, divisions moved, and the share of movement that came from the
  anchor term versus result and form (recomputable from the row's rating and match rating);
- the archetype check: players whose rating sits a tier above their rung, players at level, and
  passengers on a 50% record — how many divisions each group moved.

**A deterministic simulation test** ships with the engine: synthetic seasons, archetype players,
and assertions on the targets above (three or four divisions for the over-rated, roughly zero at
level). It is what stops the constants drifting silently in a later change; the SQL is what sets
them in the first place.

Record the distribution and the constants finally chosen in this file under a "Calibrated" heading,
with the date.

## 9. Decisions

| Decision | Status | Outcome |
|---|---|---|
| The name | ✅ decided 2026-09-04 | Form Points, FP. Not XP |
| The summit | ✅ decided | Master is the top tier. Best of the Crop is the title held by its best members |
| Where the raw rating shows | ✅ decided | On hover of any tier chip, plainly on the team generation page, and in the admin player form |
| Best of the Crop seats | ✅ decided | Three in groups under 30 active players, 10% of the group from 30 up |
| Division size | ✅ decided | 75 FP. Overflow cap and demotion landing scaled to 20 and 40 |
| Soft reset | ✅ decided | Master → Platinum, Diamond → Gold, Platinum → Silver, the rest up to three divisions. More FP means a smaller drop, or a higher landing division for Masters. FP inside the division carries over |
| Progression chart | ✅ decided | Tier and FP over time, on absolute ladder position with tier bands, raw rating on hover |
| Backfill vehicle | ✅ decided | The existing bulk recalculation, empty body, once per group (§10.1) |
| Reset trigger | ✅ decided | Inside finalise; **starting a season finalises the running one** unless it has no completed match; standalone finalise stays (§10.2) |
| Guests | ✅ decided | Ladder maintained like their rating, hidden from rankings, never a seat, never counted for seat size (§10.3) |
| Skipped seasons | ✅ decided | Every player resets at every finalise, played or not (§10.3) |
| Calibration report | ✅ decided | SQL on a staging copy, plus a simulation test in the engine (§10.5, §8) |
| Other constants | 🟡 calibrate | Result base, form weight, tier gains, pace, anchor and the rating bands wait for the step 1 replay (§8) |

## 10. Edge cases and the replay — decided 2026-09-04

Five things the proposal left unstated, walked through one by one with the owner. Each was checked
against the code before the question was asked; where the code already had the answer, that is
recorded rather than a new rule.

### 10.1 Where the replay gets the match rating

Nowhere stored. The whole-set replay (#272) unwinds every selected match newest-first and re-rates
them oldest-first, computing each player's match rating inside the forward pass. `RankLadderService`
hooks into that pass right after the rating is written, so the input is always fresh and nothing is
read back from `player_stats.rating` or reconstructed from history rows.

**Consequence:** the backfill is not a tool. It is one bulk recalculation of every completed match
with the ladder switched on for the tenant — `POST /api/matches/recalculate` with an empty body,
called by a group admin once per group. That is also the calibration run and the recovery path if
the constants change later. A completed match with no stat rows is already skipped by the rating
replay, so it pays nothing on the ladder either. The replay deletes and re-inserts every touched
history row; the write timestamp changes, the match timestamp is preserved, and the chart reads
the preserved one.

*Rejected:* a platform-operator one-shot across tenants (surface to build and secure, pays off only
with many groups); a startup migration job (a long replay inside boot on a Basic dyno).

### 10.2 Season resets inside a recalculation — and the seasons change

The rating engine's transition row is *carried* through a replay, never recomputed, because it
averages the whole group at a moment that has passed. The ladder reset is deterministic from the
player's FP at that point, so the replay **recomputes** it when it re-anchors the row.

That needed a concrete trigger, and the owner chose one that changes the seasons feature:
**starting a season finalises the one running**, in one transaction — rating transition, awards,
Ballon d'Or poll, end date, and now the ladder reset and the placement counter zeroed. The
"starting is not finalising" rule that the seasons surface was built around goes, with these
refinements:

- a displaced season with **no completed match** is displaced only — nothing to take, no awards
  for nobody, no poll with nobody in it;
- the standalone `POST /api/seasons/{id}/finalise` **stays**, so a group can close its books and
  take a break with no current season, as today;
- a match recorded into an already finalised season still pays FP against the live ladder, exactly
  as it already moves the live rating; a replay puts it back in its place before the reset.

What changes with it, same commit: `SEASONS-API-CONTRACT.md` (the start endpoint and its
"Starting is not finalising" section), the `StartSeasonModal` warning in `en`/`pt`/`es`,
`SeasonService.start`'s javadoc, and [FE seasons](../../frontend/features/seasons.md).

*Rejected:* lazy reset at a player's first match of a newer season (robust to a never-finalised
season, but the owner wants the season closed properly instead); reset on start as a bare event
(leaves a never-started season unreset with no trace).

### 10.3 The players the rules did not mention

| Who | Rule | Why |
|---|---|---|
| **Guests** | Ladder maintained silently, hidden from rankings, never a Best of the Crop seat, never counted for seat size. A promoted guest appears with what they earned | The rating engine does not special-case guests either: they earn a rating, streaks and totals, are excluded from the rankings and the group mean, and keep everything on promotion. Same shape, no engine branch |
| **Mid-season joiners** | Nothing special | Placement counter starts at 0, so their first three completed matches are placements, seeded from the admin-assigned rating |
| **Inactive players** | Reset like everyone; out of the rankings and ineligible for a seat while inactive; back in on reactivation | The rating transition already applies to all players, active and inactive |
| **A whole season away** | Reset at every finalise, played or not — two seasons away is two drops. Placements at 2× are the way back | Simplest rule, and a ladder about form should not remember a Master who has not played in a year. *Rejected:* reset only if they played (a Master could hold Master through years of absence); a softer one-division dormancy drop (a third rule to explain, test and replay) |
| **Deleted or amended matches** | Nothing special | `MatchService` already unwinds the match plus everything after it, newest first, and re-applies the rest. The ladder restores each player's pre-span snapshot from the earliest unwound row |
| **Anonymised players** | Nothing special | History rows survive anonymisation today; the ladder columns on them do too |

### 10.4 Nights with no performance signal

There are none. A match has no walkover or abandoned state, every player placed in a team gets a
stat row when the match is created, and the engine rates every stat row on completion. Anyone on
the roster of a completed match has a match rating, so the FP formula always has its input.

**Inherited limitation, not a ladder rule:** a no-show left on the roster is rated as a passenger
today, because the app has no "did not play" flag, and will earn passenger FP tomorrow. If that
ever matters, the fix is a flag on `player_stats` that the rating engine honours, and the ladder
follows for free.

### 10.5 How calibration is judged

SQL on a staging copy, plus the simulation test — both specified in §8. *Rejected:* an admin
endpoint (API surface with a contract to maintain for a one-time exercise); a log dump at the end
of the replay (Heroku logs are awkward at length and gone once they scroll).
