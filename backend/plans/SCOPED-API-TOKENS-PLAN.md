# Scoped API Tokens — Technical Specification

**Date:** 2026-08-08 (specified), 2026-08-09 (built)
**Status:** ✅ **BUILT, awaiting release.** Backend `FootMania-Back#197`, frontend
`FootMania-Simple-Front#64`. Not yet deployed — see "What was actually built" below for the two
places the implementation departed from this document.
**Priority:** MEDIUM — nothing is blocked on it today, and everything automation-shaped is blocked
behind it
**Estimated Effort:** M (≈1–1½ days backend including the filter and its tests, ≈½ day frontend)
**Migration:** **V39** `api_tokens` — applied in `FootMania-Back#197`
**Depends on:** nothing
**Depended on by:** any wrist, watch, shortcut or third-party automation. See §2.
**Contract:** `docs/api/API-TOKENS-API-CONTRACT.md` — written, in the same commit as the code, per
[CONTRIBUTING rule 2](../../CONTRIBUTING.md)

---

## 1. Requirement Summary

Everything this application can be told to do, it is told through a browser holding a JWT that
expires in a day. That is right for a browser and wrong for everything else.

The motivating case is a wrist: an organiser standing at a five-a-side pitch wants to say *"Hey
Siri, record a goal"* to an Apple Watch and have it reach
`PATCH /api/matches/{id}/stats/live`. That is achievable **today, with no app changes at all** —
Apple's Shortcuts app runs on watchOS and a three-step shortcut (*Dictate Text → Get Contents of
URL → show result*) does exactly this. It is blocked on one thing only: **the shortcut has to carry
a credential, and a one-day JWT is not one.**

So this specification is not about voice. It is about the credential that makes voice — and every
other automation anybody ever asks for — possible:

> A member can mint a **long-lived, narrowly-scoped, revocable token** for one group, see when it
> was last used, and revoke it the moment a phone goes missing.

**What this is not.** It is not an OAuth provider, not a public API programme, and not a
machine-to-machine credential for another service. It is a personal access token, in the sense
GitHub uses the term: a thing one human mints for one purpose and can throw away.

---

## 2. Scope

| In | Out |
|----|-----|
| Mint / list / revoke, from the caller's own settings | Anything an administrator mints on somebody else's behalf |
| One group per token, fixed at mint | A token that follows the caller across groups |
| A small closed set of capability scopes | Scopes that mirror the whole grant model |
| Bearer auth on the existing `Authorization` header | A second header, a query parameter, or cookie auth |
| `last_used_at`, revocation, expiry | Per-request audit logging, or an activity feed |
| Deletion on GDPR erasure | Retaining tokens for forensics after erasure |
| A `fm_` prefix so a leak is greppable | Secret scanning, rotation reminders, leak detection |

Deliberately **not** in scope: the voice grammar, the roster matching, the natural-language
endpoint. Those are a separate piece of work and this one is useful without them — a token is
equally what a Wear OS client, a Tasker task or a clubhouse button would use.

---

## 3. Why not just a longer-lived JWT

It is the obvious cheap answer and it is wrong three times over.

**A JWT cannot be revoked.** They are stateless by design and this deployment verifies them by
signature alone — there is no table to delete a row from. A credential that lives in a shortcut on
a watch for a year is exactly the credential that has to die the day the watch is lost. Anything
revocable needs a stored row, and once there is a stored row, the JWT has stopped being a JWT and
become this.

**A JWT carries the caller's whole self.** The authorities on it are every grant the member holds
in the group, so a shortcut that records a goal would also be able to suspend a member, erase a
player, or finalise a season. The blast radius of a leaked credential should be the size of the job
it was minted for.

**A JWT's lifetime is a global setting.** `app.jwt.expiration-ms` governs every session at once;
lengthening it for a watch would lengthen it for every browser that has ever logged in on a shared
laptop.

---

## 4. Data Model — V39

```sql
CREATE TABLE api_tokens (
    id           BIGSERIAL PRIMARY KEY,
    tenant_id    BIGINT       NOT NULL,
    user_id      BIGINT       NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    -- SHA-256 of the raw token, hex. Never the token itself.
    token_hash   CHAR(64)     NOT NULL,

    -- The first 8 characters of the raw token, stored in clear so a person can tell two of their
    -- own tokens apart in a list. Not a secret: eight characters of a 43-character random string
    -- narrows nothing.
    token_prefix VARCHAR(12)  NOT NULL,

    -- What the person called it. "Watch shortcut", "clubhouse tablet".
    name         VARCHAR(60)  NOT NULL,

    -- Comma-separated scope names, validated against the enum in code. See §5.
    scopes       VARCHAR(255) NOT NULL,

    created_at   TIMESTAMPTZ  NOT NULL DEFAULT now(),
    expires_at   TIMESTAMPTZ  NOT NULL,
    last_used_at TIMESTAMPTZ,
    revoked_at   TIMESTAMPTZ,

    CONSTRAINT uq_api_tokens_hash UNIQUE (token_hash)
);

ALTER TABLE api_tokens
    ADD CONSTRAINT fk_api_tokens_tenant FOREIGN KEY (tenant_id) REFERENCES organizations(id);

-- Authentication looks the token up by hash on every request, so this is the hot path.
-- The UNIQUE above already provides it; named here so the intent is not mistaken for a
-- constraint that happens to be useful.
CREATE INDEX idx_api_tokens_user ON api_tokens (tenant_id, user_id);
```

### Why a hash and not the token

Same reason passwords are hashed: the table is the thing that leaks. A read-only SQL injection, a
backup on somebody's laptop, or a support engineer with production access must not yield working
credentials.

### Why SHA-256 and not bcrypt

Passwords are low-entropy and human-chosen, so they need a *slow* hash to make guessing expensive.
A token here is 32 bytes from a CSPRNG — guessing it is not a threat model, and there is no
dictionary to attack. What matters instead is that **this hash is computed on every authenticated
request**, and bcrypt at a sensible cost factor would add tens of milliseconds to every API call
made by a token. A fast hash over high entropy is the right trade, and it is what GitHub, GitLab
and Stripe all do for the same reason.

No salt, deliberately: a salt exists to stop one precomputed table breaking many rows, and there is
nothing to precompute against 256 bits of randomness. A salt would also make the lookup impossible
— we must find the row *by* the hash, which means the hash has to be deterministic.

### Why `tenant_id` on the row rather than derived

A token is minted for one group and can only ever act in that one. Putting the tenant on the row
makes that a stored fact rather than something re-derived per request from a header the caller
controls — see §7, which is the part of this design most worth reading twice.

### Why `ON DELETE CASCADE` on the user

GDPR erasure must take the tokens with it. A live credential belonging to a deleted account is an
authenticated caller with no account behind it, which is the worst of both worlds: it works, and
nobody owns it. Cascade means erasure cannot forget.

---

## 5. Scopes

A small closed enum, mirrored by a `CHECK`-style validation in code rather than in the database —
unlike `season_awards`, the value is a *list* in one column, so a `CHECK` cannot express it.

| Scope | Grants | For |
|---|---|---|
| `MATCH_STATS_WRITE` | `PATCH /api/matches/{id}/stats/live` | the motivating case — recording a goal from a wrist |
| `MATCH_READ` | `GET /api/matches`, `GET /api/matches/{id}` | finding the in-progress match to write to |
| `PLAYER_READ` | `GET /api/players` | resolving a spoken name to a player id |

**Three, and no more, in the first version.** Every scope added is a permanent widening of what a
leaked token can do, and the honest way to size this set is "what does the watch shortcut actually
need". A fourth can be added when something needs it; a fourth that exists because it seemed
plausible cannot be taken away once somebody has minted a token with it.

### Scopes are intersected with live grants, never substituted for them

**This is the security property that matters.** A token's scopes narrow what the caller can do;
they never widen it. At authentication time the caller's grants are loaded from their **current
membership** exactly as the JWT path does, and the effective permission is the intersection.

So:

- A `MANAGER` who mints a `MATCH_STATS_WRITE` token, and then loses `MANAGER`, has a token that
  authenticates and is refused by `@PreAuthorize`. Nothing extra has to happen for that to be true.
- A member with no grants who somehow obtains a `MATCH_STATS_WRITE` token can still do nothing.
- Suspension or removal from the group revokes every token's usefulness the same instant it revokes
  the browser session's, because both read the same membership.

The alternative — freezing the grants into the token at mint — would make a token a way to keep
authority after losing it. That is the shape of every serious access-control incident.

### Denied regardless of scope

A hard deny-list, checked before scopes:

- anything under `/api/auth/**` — a token must not be able to change the password or mint a session
- anything under `/api/api-tokens/**` — a token must not be able to mint another token or revoke
  its siblings. Credential management is a thing you do as a human, in a browser, having logged in
- anything under `/api/privacy/**` — erasure and export are irreversible and personal
- anything under `/api/admin/platform/**` — the operator surface. A token is a member's credential
  and the operator grant is not a membership at all

The deny-list is not merely "no scope grants these". It is checked separately, so adding a scope
later cannot accidentally open one of them.

---

## 6. Token Format

```
fm_<43 characters of base64url>
```

32 bytes from `SecureRandom`, base64url-encoded without padding, prefixed `fm_`.

**The prefix earns its three characters.** It makes a leaked token identifiable on sight in a log,
a screenshot or a public repository; it is what a secret-scanning rule would match on; and it is
what lets the authentication filter decide which of the two credential families it is holding
without attempting to parse a JWT first (§8).

`token_prefix` stores `fm_` plus the first five characters, which is enough for a person to tell
"Watch shortcut" from "Old watch shortcut" in a list and far too little to help an attacker.

---

## 7. Tenancy — the token carries its own group

This is where a token differs most from a JWT, and getting it wrong would be the quiet kind of
wrong.

A browser sends `X-Group-Id` and `TenantResolver` verifies it against the caller's memberships.
That is correct for a browser, which legitimately switches groups. **A token does not switch
groups.** It was minted for one, and its `tenant_id` column says which.

So, for a token-authenticated request:

1. The tenant is taken from the **token row**, never from the header.
2. If `X-Group-Id` is present and disagrees, the request is **404**, not 403 — the same rule
   `TenantGuard` already follows, and for the same reason: a 403 confirms the other group exists.
3. The membership is still verified live. A token whose group the caller has since left, or been
   suspended from, authenticates nothing.

`TenantResolver.toleratesUnresolvedTenant` and the tenant-agnostic path list are **not** consulted
on this path. Those exist for a caller who has not chosen a group yet, which is a state a token
cannot be in — a token that resolved to no group would be a token that could act in none, and the
honest answer is to refuse it at authentication rather than let it through to fail confusingly
later.

---

## 8. Where it plugs in

`JwtAuthenticationFilter` already reads `Authorization: Bearer …`. Shortcuts, Wear OS and every
HTTP client can set that header, so **the same header carries both credential families** and
nothing else in the stack has to learn a new one.

The branch is on the prefix:

```
Bearer fm_…   → the token path (this document)
Bearer eyJ…   → the existing JWT path, untouched
```

Two options, and the recommendation is the second:

**(a) Extend `JwtAuthenticationFilter`.** Fewer moving parts, but it makes one class responsible
for two authentication schemes with different tenancy rules, and the class is already the most
security-sensitive one in the codebase.

**(b) A separate `ApiTokenAuthenticationFilter`, ordered before it.** ✅ Recommended. It claims a
request only when the bearer value starts with `fm_`, and passes otherwise. The JWT filter is then
literally unmodified — which matters, because the tenancy verification inside it is subtle enough
that the safest change to it is none.

Both produce the same `UserPrincipal`, so `@PreAuthorize` and every controller stay exactly as they
are. The principal gains nothing; the scope check happens in the filter, before the principal is
built, and a scope failure is a **403 with a body naming the missing scope** — a caller debugging a
shortcut at a pitch deserves to be told which scope it needed.

### `last_used_at` must not make every read a write

Updating it on every request turns each authenticated `GET` into a `GET` plus an `UPDATE`, on the
hot path, for a column nobody reads in real time. **Write it at most once per hour per token**: if
`last_used_at` is null or older than an hour, update it; otherwise skip. The column's purpose is
"is this token still in use, roughly", and an hour's resolution answers that completely.

---

## 9. API

| Method | Path | Auth |
|---|---|---|
| `POST` | `/api/api-tokens` | authenticated **by session, never by token** |
| `GET` | `/api/api-tokens` | as above |
| `DELETE` | `/api/api-tokens/{id}` | as above |

```jsonc
POST /api/api-tokens
{ "name": "Watch shortcut", "scopes": ["MATCH_STATS_WRITE", "MATCH_READ"], "expiresInDays": 365 }

201 Created
{
  "id": 4,
  "name": "Watch shortcut",
  "token": "fm_9xQ2…",          // THE ONLY TIME THIS IS EVER RETURNED
  "prefix": "fm_9xQ2",
  "scopes": ["MATCH_STATS_WRITE", "MATCH_READ"],
  "groupId": 1,
  "groupName": "Sunday League",
  "createdAt": "2026-08-08T18:00:00Z",
  "expiresAt": "2027-08-08T18:00:00Z"
}
```

`GET` returns the same shape **without `token`**, for every group the caller belongs to, newest
first, including expired and revoked ones — a token you cannot see is a token you cannot reason
about, and "this one expired in March" is information.

`DELETE` sets `revoked_at` rather than deleting the row, so the list can still show that it existed
and when it was last used. A revoked token authenticates nothing from that moment.

### `expiresInDays`

Constrained to 30 / 90 / 365, defaulting to 365. **No non-expiring option**, and that is a
decision: a credential with no end date is one nobody ever revisits, and the entire value of an
expiry is that it forces a conscious act at least once a year. 30 and 90 exist for somebody trying
something out.

### The token is shown once

Never retrievable afterwards — only its hash is stored, so there is nothing to retrieve. The
frontend must make that unmistakable at mint time, because a person who closes the dialog without
copying it has to mint another, and a person who does not realise that will blame the app.

---

## 10. Frontend

**Settings → My account tab**, below Password, above the privacy section.

The account tab rather than the group tab, even though a token is bound to one group: revoking a
credential is an account-security act, and somebody who has lost a phone needs to see *every* token
they hold in one place, not three lists behind three group switches. Each row names its group.

The list shows name, group, prefix, scopes, created, **last used** and expiry, with a revoke
control. Last used is the column that matters — it is how somebody decides whether the token they
are looking at is the one they still need.

Minting is a modal: name, scope checkboxes with a plain-language line each, expiry select. The
result view shows the token once, with a copy button and a warning that it will not be shown again.
Revoking uses the two-click confirm the app already uses elsewhere; it does not need the
type-the-name gate, because revoking is recoverable by minting a new one — unlike finalising a
season, which is the bar that gate exists for.

Strings in **all three locales**, including the scope descriptions. `MVP_VOTE_OPEN` and
`FEE_CHARGED` both reached production rendering as raw enum names; a scope list is exactly the same
shape of risk.

---

## 11. Risks, and what is accepted

**A bearer token on a watch sits behind no biometric gate.** Anybody holding the unlocked watch can
run the shortcut. Accepted, and mitigated by the scope set: the worst outcome is a fabricated goal
in a five-a-side match, which is visible on the scoresheet and correctable by an administrator.
This is precisely why `MATCH_STATS_WRITE` is narrow and why the deny-list in §5 exists.

**Tokens in shortcuts get shared.** Somebody will send a working shortcut to a teammate rather than
have them mint their own. Mitigated by attribution — the token names its owner, so whatever it does
is recorded as that person — and by the list making a second device's `last_used_at` visible. Not
prevented, and probably not preventable.

**Brute force is not the threat.** 256 bits of entropy is not guessable, so no equivalent of
`LoginThrottle` is specified for this path. **Volume from a leaked token is** — a rate limit per
token is worth adding, and is deliberately left out of the first version rather than designed
badly now.

**The hash makes recovery impossible by design.** A support request of the form "I lost my token,
can you resend it" has exactly one answer: mint another. That is correct and should be said in the
UI.

---

## 12. Test Plan

| Layer | What it must pin |
|---|---|
| Unit | Hashing is deterministic; the prefix is stored in clear and the token is not; expiry, revocation and unknown-token all refuse |
| Unit | **Scopes intersect grants**: a `MATCH_STATS_WRITE` token held by somebody who has lost `MANAGER` is refused |
| Unit | The deny-list refuses `/api/auth`, `/api/api-tokens`, `/api/privacy` and the operator surface **whatever the scopes say** |
| Unit | The tenant comes from the row; a contradicting `X-Group-Id` is 404, not 403 |
| Unit | `last_used_at` is written at most once an hour |
| Slice | `POST` returns the token exactly once; `GET` never returns it |
| Slice | A token cannot mint or revoke a token |
| Integration | V39 applies; erasure cascades tokens away; a token for a group the caller has left authenticates nothing |

The four unit rows in bold territory are the ones worth writing first: each describes a failure that
**authenticates successfully and does the wrong thing**, which is the only class of bug in this
feature that would not announce itself.

---

## 12a. What was actually built

Everything above shipped as specified, with two departures worth recording — both discovered while
writing the code rather than while writing this document.

**The scope→endpoint mapping became an allow-list, and the deny-list stayed anyway.** §5 described
scopes as granting named endpoints and the deny-list as a separate check. Implemented literally,
that leaves a question this document never answered: what happens to an endpoint that no scope
names and no deny rule covers? The answer had to be *refused* — otherwise every endpoint written
after the feature would be reachable by every existing token on the day it merged. So
`ApiTokenScope.requiredFor` returns empty for anything unlisted and the filter treats empty as a
refusal.

That makes the deny-list redundant **today**: none of those paths appear in the allow-list, so all
four are already refused. It is kept, and still checked first, because the allow-list is the thing
that grows. A scope added in two years to cover "read the group's members" must not be one careless
pattern away from also opening password changes.

**The live-membership check lives in the service, not the filter.** §7 said the membership is
verified live and did not say where. It went into `ApiTokenService.ownerIsStillAMember` so the
filter keeps one collaborator and no repository of its own — the same shape `JwtAuthenticationFilter`
has with `TenantResolver`. The practical reason was narrower: the filter is a `Filter` bean, so
every `@WebMvcTest` slice in the codebase scans it and needs its dependencies mocked, and each
dependency is a line added to thirteen existing test files.

**One item from §12 has not run.** The integration tier needs Docker, which the build environment
did not have. `V39` has never been applied against real PostgreSQL — as is also true of `V37` and
`V38`. Three unverified migrations is the one outstanding risk on this feature.

---

## 13. What this unlocks, in order

1. **The watch shortcut.** Dictate → POST → done. No app changes, no App Store, no Capacitor. This
   is the reason the plan exists and it needs nothing beyond §9.
2. **A natural-language endpoint** — `POST /api/matches/{id}/stats/voice` taking a transcript,
   matching it against the confirmed squad for that match (a closed set of ten to fourteen, which
   is what makes it tractable), and returning a *proposed* event to confirm. Separate plan.
3. **Wear OS, Tasker, a clubhouse tablet, a Stream Deck.** All the same credential.
4. **App Intents on iOS**, if Phase 4's Capacitor wrap ever happens — by which point the parsing
   and confirmation layer already exists and is tested, and the native side is a thin front end.

A deliberate note on sequencing: **step 2 is where the product risk is, not step 1.** Whether
anybody records stats live at all is the question worth answering before building a grammar, and
the cheapest way to answer it is the in-app press-to-talk described in
[app-store-strategy](../../architecture/app-store-strategy.md)'s neighbourhood — if nobody uses a
mic button on the screen they are already looking at, nobody will use the watch either.
