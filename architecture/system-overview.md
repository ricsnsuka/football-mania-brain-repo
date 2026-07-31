# System Overview

How the two repositories fit together. For depth inside either half, read
[backend/architecture/ARCHITECTURE.md](../backend/architecture/ARCHITECTURE.md) and
[frontend/architecture/overview.md](../frontend/architecture/overview.md).

---

## Shape

```
┌─────────────────────────────┐         ┌──────────────────────────────┐
│  Next.js 16 · React 19      │  HTTPS  │  Spring Boot 3.4.5 · Java 21 │
│  App Router · TypeScript    │ ──────► │  One deployable. No Redis,   │
│  Tailwind 4 · TanStack      │  JWT    │  no Kafka, no message bus.   │
│  Query · Zustand · i18next  │ ◄────── │                              │
│  PWA + service worker       │  SSE    │                              │
└─────────────────────────────┘         └──────────────┬───────────────┘
        │                                              │
        │ Web Push (VAPID, RFC 8291/8292)              │ JDBC
        ▼                                              ▼
┌─────────────────────┐                     ┌────────────────────────┐
│  Browser push       │ ◄────────────────── │  PostgreSQL + Flyway   │
│  service (FCM etc.) │   from the backend  │  V1 … V19              │
└─────────────────────┘                     └────────────────────────┘
```

**It is a monolith, on purpose.** The case for and against splitting it is written up in
[backend/features/MICROSERVICES_ARCHITECTURE.md](../backend/features/MICROSERVICES_ARCHITECTURE.md);
the conclusion is that at this scale the coordination cost buys nothing. That philosophy — no new
infrastructure unless the feature genuinely cannot work without it — is why push was implemented
against the raw Web Push protocol rather than pulling in Firebase, and why the cache is in-process
Caffeine rather than Redis.

---

## The three integration points

**1. REST + JWT.** The only routes that are not authenticated are login, health, version and the
push public key. Roles are **composable** since V18 — a user holds a *set* of roles, so the person
who runs the matches, the person who handles the money and the person who administers the system
need not be the same. Do not treat `ADMIN` as a superset of the others; the frontend had that bug
and fixed it in `418256e`. Contract:
[backend/api/ROLES-API-CONTRACT.md](../backend/api/ROLES-API-CONTRACT.md).

**2. Server-Sent Events**, for the live draft only. The draft is the one screen where several
people act on shared state simultaneously. Everything else polls through TanStack Query, which is
sufficient and much cheaper to reason about. Guide:
[backend/frontend/DRAFT_SESSION_SSE_GUIDE.md](../backend/frontend/DRAFT_SESSION_SSE_GUIDE.md).

**3. Web Push**, backend → browser, bypassing the frontend entirely. Seven categories, every one on
by default, with opt-outs stored rather than preferences — so a new category ships enabled with no
backfill. Contract: [backend/api/PUSH-API-CONTRACT.md](../backend/api/PUSH-API-CONTRACT.md).

> **iOS delivers push only to an installed PWA.** That is why Phase 1 (installability) had to land
> before Phase 2 was worth anything, and why the install-prompt UX is not cosmetic.

---

## Two conventions worth knowing before you change anything

**A null field is omitted from JSON, not sent as `null`.** The backend sets
`default-property-inclusion: non_null`. Any nullable field in a contract is *absent* when unset, and
absent is a different claim from `null` — `totalCostCents` absent means "no cost recorded", `0`
means "the match was free". In tests, `jsonPath().doesNotExist()` passes for present-but-null too;
`doesNotHaveJsonPath()` is the assertion that actually distinguishes them.

**Work that must not fail the caller runs after the commit.** Rating recalculation, push sends and
badge awards all hang off `MatchEventListener` on the ADR-002 pattern: a transactional event, async,
in its own transaction. A push service being unreachable must never undo a saved match. The
consequence is that these paths cannot report failure to a caller who is already gone, so they
retry, log, and mark (`matches.needs_recalc`) instead. See
[backend/next-release/ARCHITECTURE_DECISIONS.md](../backend/next-release/ARCHITECTURE_DECISIONS.md)
— ADR-001 optimistic locking, ADR-002 async completion, ADR-003 Caffeine with accepted staleness,
ADR-004 the LLM vendor choice for the not-yet-built AI match reports.

---

## Where the data model is load-bearing

- **`Match` has no foreign key to `MatchPlan`.** A plan is a proposal; a match is what happened.
  This is why the fee hangs off the *plan* (the money is owed when the pitch is booked) and why
  V17 could not backfill its terminal state precisely.
- **Ratings are recomputed, never incremented.** `skill_rating_history` stores what each
  application consumed (V7) so a recalculation reverses exactly, even after a stat is amended.
- **Erasure anonymises in place**, because deleting a player would rewrite other players' match
  records through the cascade. See
  [backend/features/PRIVACY_AND_DATA_PROTECTION.md](../backend/features/PRIVACY_AND_DATA_PROTECTION.md).
- **The ledger is append-only** and the balance is derived. No allocation of payments to specific
  charges — the question people ask is "are we square?", which is one number.

---

## Deployment

| | Where | Notes |
|---|---|---|
| Backend | Heroku (`footmania`) | [backend/deployment/HEROKU_DEPLOYMENT_GUIDE.md](../backend/deployment/HEROKU_DEPLOYMENT_GUIDE.md) · Flyway runs on boot, so a deploy *is* a migration |
| Frontend | Netlify | [frontend/guides/netlify-deployment.md](../frontend/guides/netlify-deployment.md) |

Because Flyway runs on startup, **deploying the backend applies every pending migration**. See
[database-migrations.md](database-migrations.md) for what is currently pending and why V18 deserves
care.
