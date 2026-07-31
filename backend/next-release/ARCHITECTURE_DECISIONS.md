# Architecture Decision Records — Concurrency & Performance

> **Status:** Accepted
> **Date:** 2026-05-15
> **Context:** Football Management System — pre-CalculationService implementation

---

## ADR-001: Optimistic Locking on Match (`@Version`)

### Context
Multiple ADMIN/MASTER users can submit `PATCH /api/matches/{id}/complete` for the same
match at the same time. Without a concurrency guard, both requests pass the
`isCompleted = false` check, and the second write silently overwrites the first,
potentially corrupting score and stat data.

### Decision
Add a `@Version Long version` field to the `Match` entity.

JPA increments `version` on every `UPDATE`. If two transactions read the same version
and both try to write, the second commit detects the mismatch and raises
`ObjectOptimisticLockingFailureException`.

`GlobalExceptionHandler` maps this to **HTTP 409 Conflict** with the message:
> *"This resource was modified by another request. Please retry."*

### Implementation
- `model/Match.java` — `@Version private Long version`
- `db/migration/V4__add_match_optimistic_lock.sql` — `ALTER TABLE matches ADD COLUMN version BIGINT NOT NULL DEFAULT 0`
- `exception/GlobalExceptionHandler.java` — `@ExceptionHandler(ObjectOptimisticLockingFailureException.class)` → 409

### Consequences
- ✅ Zero-overhead on non-contended writes (the common case)
- ✅ No distributed lock required — works per-node with no external dependency
- ✅ Safe and idempotent — the losing request gets a clean 409, client can retry
- ⚠️ `MatchDTO` will include `version` in responses if the mapper exposes it — it does **not** (mapper only maps defined DTO fields)

---

## ADR-002: Async `completeMatch` via Spring Transactional Events

### Context
`completeMatch` does significant work before returning a response:
1. Validates scores and team integrity
2. Updates ~14 `PlayerStat` rows
3. Will invoke `CalculationService` for ~14 skill-rating recalculations (slow, not yet implemented)
4. Inserts ~14 `SkillRatingHistory` rows
5. Evicts Caffeine caches

Keeping all of this synchronous would block the HTTP response until CalculationService
completes, which may take hundreds of milliseconds under load.

### Decision
**Split `completeMatch` into a fast synchronous phase and a slow async phase.**

### Architecture

```
POST /api/matches/{id}/complete
           │
           ▼ (synchronous — within one TX)
  MatchService.completeMatch()
    ├── validates business rules
    ├── updates PlayerStat rows (goals/assists/ownGoals/rating/isMvp/matchResult)
    ├── sets match scores + isCompleted = true
    ├── matchRepository.save()
    └── eventPublisher.publishEvent(MatchCompletedEvent)
           │
           │  TX commits → HTTP 200 response returned immediately
           │
           ▼ (asynchronous — AFTER_COMMIT, virtual thread)
  MatchEventListener.onMatchCompleted()
    ├── TODO: calculationService.recalculateMatchRatings(matchId)
    ├── TODO: streakService.updateStreaks(matchId)
    └── evicts "players" + "rankings" caches
```

### Key Design Choices

| Choice | Reason |
|--------|--------|
| `@TransactionalEventListener(AFTER_COMMIT)` | Listener fires only when the TX durably commits. If the TX rolls back (e.g. constraint violation), the listener is never invoked — no partial calculation runs. |
| `@Async("matchEventExecutor")` | Runs on a dedicated virtual-thread executor, fully decoupled from the HTTP response thread. |
| `players` + `rankings` caches evicted async | These caches reflect skill ratings, which only change after CalculationService runs. Evicting them sync before calc runs would be premature. |
| `matches` cache evicted sync | The match `isCompleted` state changes immediately — callers must see the updated match straight away. |
| Exception swallowed in listener | The match is already persisted. Letting a CalculationService failure bubble up would claim the match completion failed when it succeeded. Logged prominently for alerting. |

### Consequences
- ✅ HTTP `completeMatch` response returns in < 50ms (just DB writes + sync cache evict)
- ✅ CalculationService runs in background — no blocking the caller
- ✅ TX rollback → listener never fires (safe)
- ⚠️ Eventual consistency: skill ratings update asynchronously — a `GET /api/players/{id}` immediately after `completeMatch` may return the pre-match rating (until async work + cache eviction complete)
- ⚠️ Async failures are currently only logged — add alerting/retry when CalculationService is implemented

### Files
- `event/MatchCompletedEvent.java` — Spring event record
- `service/MatchEventListener.java` — async listener with TODO hooks for CalculationService
- `config/AsyncConfig.java` — virtual-thread `AsyncTaskExecutor` bean (`matchEventExecutor`)
- `service/MatchService.java` — `completeMatch` changed to `@CacheEvict(MATCHES)` only; publishes event after save

---

## ADR-003: Caffeine Local Cache with Multi-Node Staleness Accepted

### Context
The app may run on multiple pods (K8s HPA is configured). Caffeine is an in-process
cache — evictions on one pod do not propagate to other pods.

### Decision
**Accept the multi-node staleness trade-off. Do not add Redis.**

### Rationale
| Factor | Analysis |
|--------|----------|
| Data criticality | Match and player data is not financially or safety-critical. A stale read within a 10-minute window is acceptable for a football management app. |
| Operational cost | Redis adds infrastructure, connection management, cluster configuration, and additional failure modes. |
| Stack constraint | Redis is explicitly excluded from the project stack. |
| Migration path | If stale cache becomes a problem, `@Cacheable`/`@CacheEvict` annotations require **zero changes**. Only `CacheConfig.java` needs updating to swap Caffeine for a Redis-backed `CacheManager`. |

### Staleness Window
- **Write on Pod A** → Pod A cache evicted immediately, Pod B cache stale for ≤ 10 min (TTL)
- **Worst case**: A match completed on Pod A takes up to 10 minutes to be visible on Pod B without a cache miss

### Mitigation Options (if needed later)
| Option | Trade-off |
|--------|-----------|
| Shorten TTL to 1–2 min | Reduces stale window, increases DB read load |
| Add Redis | Solves staleness, adds infrastructure |
| Sticky sessions | Keeps user on same pod, kills horizontal scaling — **do not use** |


---

## ADR-004: Anthropic Claude as the LLM Vendor for AI Match Reports

> **Status:** Accepted (vendor decision only — no code until Phase 3)
> **Date:** 2026-07-28
> **Context:** Phase 0 of the Improvement & Mobile Roadmap

### Context

Phase 3 of the roadmap proposes AI-written match reports: after a match completes, generate a
short narrative from the scoreline, goals and player stats. The roadmap flags the vendor as a
Phase 0 decision rather than a Phase 3 one, and that is right — it is the only item on the Phase 3
list that adds an external dependency and a recurring cost, and the choice shapes the async job
before it is written. Deciding it now costs nothing; discovering it late means rewriting the job.

Nothing is built here. This ADR fixes the vendor, the model, the call shape and the cost ceiling so
Phase 3 starts from a decision instead of a survey.

### Decision

**Use the Anthropic Claude API, via the official Java SDK, on `claude-opus-5`.**

- Dependency: `com.anthropic:anthropic-java` (Maven Central; 2.34.0 at time of writing)
- Model ID: `claude-opus-5` — 1M context, 128K max output, $5 / $25 per million input / output tokens
- Endpoint: the Messages API, one request per completed match, no streaming
- Credential: `ANTHROPIC_API_KEY` as an environment variable, alongside the existing secrets

### Rationale

| Factor | Analysis |
|--------|----------|
| Fits the existing hook | ADR-002 already established the async, after-commit, failure-tolerant pattern for post-completion work. A match report is exactly that shape: slow, non-critical, must not block or fail `completeMatch`. It reuses `MatchEventListener` and needs no new infrastructure — consistent with the "no Redis, no Kafka" constraint. |
| Java support | An officially maintained first-party Java SDK, which not every vendor offers. No hand-rolled HTTP, no OpenAI-compatible shim. |
| Task fit | The task is short-input, short-output prose generation over structured data — well within any frontier model. Model capability is not the deciding factor here; integration cost and predictability are. |
| Cost | Negligible at this scale — see below. |
| Reversibility | One outbound call behind one service method. Switching vendors later is a contained change, which is why this decision does not warrant more analysis than it has been given. |

### Cost ceiling

A single report is roughly 2,000 input tokens (system prompt plus one match's stats for 10–22
players) and 600 output tokens:

```
input:  2,000 / 1,000,000 × $5   = $0.010
output:   600 / 1,000,000 × $25  = $0.015
                                   ───────
                            ≈ $0.025 per report
```

One match per week is **under $2 per year**. Even a hundred groups each playing weekly is around
$130 per year. The roadmap's warning to "budget for it and rate-limit it before enabling for
everyone" is sound as a principle, but the arithmetic says the exposure is small enough that the
rate limit is a runaway-loop guard, not a cost control.

Guards for Phase 3 to implement regardless:

- **One report per match, ever.** Store the result in a `match_report` column and make generation a
  no-op when it is already populated. A bulk recalculation or a retried event must not re-bill.
- **A per-day ceiling on generations**, so a defect in the event path cannot loop.
- **A feature flag** (`app.ai.match-reports.enabled`, default `false`), matching how
  `app.draft.ttl-hours` and `app.security.public-api-docs` are handled.

### Implementation notes for Phase 3

These are the details most likely to be got wrong from memory, so they are pinned here:

- **Effort, not model tier, is the cost lever.** `output_config.effort` accepts
  `low`/`medium`/`high`/`xhigh`/`max`; the default is `high`. A match report is a short,
  well-specified generation — start at `low` or `medium` and only raise it if the prose is
  visibly worse. Do not silently downgrade the *model* to save money; that is a product decision,
  and at $0.025 a report it buys nothing.
- **Thinking is on by default on `claude-opus-5`.** Omitting the `thinking` parameter runs adaptive
  thinking, and `max_tokens` caps thinking *plus* response text together — so size `max_tokens`
  with headroom, not tightly around the expected prose length, or reports will truncate.
- **Safety classifiers can decline a request.** The response is a normal HTTP 200 with
  `stop_reason: "refusal"` and an empty or partial `content`. Check `stop_reason` before reading
  `content[0]`; a refusal should leave `match_report` null and log, exactly like any other
  post-completion failure under ADR-002.
- **Rate limits are per-tier and `claude-opus-5` has its own bucket**, separate from the Opus 4.x
  pool. Irrelevant at one call per match, but relevant if reports are ever backfilled across a
  season — use the Batch API for that rather than a loop.

### Consequences

- ✅ Phase 3 starts with the vendor, model, call shape and guards already settled
- ✅ No new infrastructure; reuses the ADR-002 async hook
- ⚠️ First external runtime dependency in the system — the deployment now has an outbound
  dependency that can be slow or unavailable. The async, failure-tolerant hook is what keeps that
  from reaching a caller whose match is already saved
- ⚠️ First recurring per-use cost. Small, but it is no longer zero, and it scales with matches
- ⚠️ An API key now has to be provisioned, stored and rotated
