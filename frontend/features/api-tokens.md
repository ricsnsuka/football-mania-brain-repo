# API Tokens Feature

**Added:** 2026-08-09 · **Status:** 🟡 built, unreleased —
`FootMania-Simple-Front#64`, backend `FootMania-Back#197`
**Where:** Settings → **My account**, between Notifications and Your data
**Backend contract:** [API-TOKENS-API-CONTRACT.md](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/API-TOKENS-API-CONTRACT.md)
· **Plan:** [SCOPED-API-TOKENS-PLAN](../../backend/plans/SCOPED-API-TOKENS-PLAN.md)

---

## What it is

A credential something other than a browser can carry. Every request this app makes is
authenticated by a JWT that expires in a day, which is right for a browser and useless to a shortcut
on a watch, a clubhouse tablet, or a script. A token is the other kind: long-lived, narrowly scoped,
bound to one group, revocable.

The motivating case was an organiser at the side of a pitch saying *"Hey Siri, record a goal"*. That
needs no app changes at all — Shortcuts runs on watchOS and can already reach
`PATCH /api/matches/{id}/stats/live`. It was blocked on one thing only: the shortcut has to carry a
credential, and a one-day JWT is not one.

## Files

| File | What |
|---|---|
| `types/apiToken.ts` | DTOs, zod schemas, the scope and expiry constants, `apiTokenStatus()` |
| `services/apiTokenService.ts` | `listApiTokens`, `createApiToken`, `revokeApiToken` |
| `hooks/apiToken/useApiTokens.ts` | `useApiTokens`, `useCreateApiToken`, `useRevokeApiToken` |
| `features/apiTokens/ApiTokenSettings.tsx` | The section — header, states, list |
| `features/apiTokens/ApiTokenRow.tsx` | One token, with its two-click revoke |
| `features/apiTokens/CreateApiTokenModal.tsx` | The mint form **and** the one-time reveal |

## The account tab, not the group tab

Every other per-group surface lives on the tab titled with the group's name. This one does not, and
that is the first thing to understand about it.

A token belongs to exactly one group, is frozen to it at mint, and cannot act in another. By the
tab logic that is group territory. But the moment this section matters is the moment a phone goes
missing, and somebody in that state needs to see **every** credential they hold at once — not to
discover the third one only after switching groups twice. Revoking is an account-security act, so it
sits with the account.

The endpoint agrees: `GET /api/api-tokens` deliberately spans groups. It is the one list in this app
not scoped by `X-Group-Id`, which is why the query key carries no group either — keying it per group
would cache the same answer several times and refetch it on every switch for nothing.

Each row names its group. That label is what keeps the exception legible.

## The one-time reveal

The mint modal has two views and has to be one dialog, because the token exists only in the moment
between them. Only a hash is stored server-side, so closing the reveal destroys the only copy
outside the caller's clipboard. There is no screen anywhere that can show it again and no support
path that can recover it.

Three consequences, all deliberate:

- **The warning is stated twice** — once in the subtitle, once in the amber block around the token.
- **The close button reads "I have copied it"**, not "Close". The label is the last chance to say
  that closing is destructive.
- **The token renders as selectable monospace text beside the copy button**, not only behind it. A
  denied clipboard permission — or an insecure origin — must not be the difference between having a
  credential and having to mint another. The failed-copy path also says so explicitly rather than
  leaving a button that appears to do nothing.

The minted secret lives in component state and **nowhere else**. In particular it is never put in
the query cache, which this app persists across browser restarts: a credential surviving in storage
because a screen once displayed it is exactly the leak the server's hashing exists to prevent.

## Scopes are described, not named

`MATCH_STATS_WRITE` means nothing to the person choosing it. Each checkbox carries a plain-language
line saying what the token could do with it, in all three locales, because the whole value of a
narrow credential is somebody picking the narrow option — and they only will if the narrow option is
legible.

| Scope | Shown as | Line under it |
|---|---|---|
| `MATCH_READ` | Read matches | See the list of matches and find the one in progress |
| `PLAYER_READ` | Read players | See the squad, so a spoken or typed name can be matched |
| `MATCH_STATS_WRITE` | Record match stats | Add goals and assists to a match being played |

Declaration order is display order, and it reads as the sequence a shortcut actually performs: find
the match, resolve the name, record the goal.

**Localising these is not optional.** `MVP_VOTE_OPEN` and `FEE_CHARGED` both reached production
rendering as raw enum names; a scope list is the same shape of risk, and this one is in front of
somebody making a security decision.

### The note about grants

The modal states that a token can never do more than its owner can. The server intersects a token's
scopes with the caller's **live** grants in that group on every request, so ticking
`MATCH_STATS_WRITE` grants nothing on its own.

Without that line, a member without `MANAGER` mints a stats-writing token, watches it 403, and
reasonably concludes the feature is broken.

## Reading the list

`apiTokenStatus()` derives three states from `revokedAt` and `expiresAt` rather than reading a
fourth field — the same reasoning as `seasonStatus()`: a status string alongside the two facts is a
second copy of them, and second copies are what disagree later.

**Revoked is checked before expired.** A token can be both, and revoked is the more useful to show:
one is a deliberate act, the other is the calendar catching up, and showing "expired" would make a
decision look like time passing.

**Last used is the field that matters.** Name and scopes say what a token was *for*; last used is
the only thing that says whether it still is, and it is what somebody actually decides on. It gets
its own labelled field, and a note that it is recorded **at most once an hour** — without that, a
shortcut fired two minutes ago looks broken.

**Revoked and expired tokens stay in the list**, dimmed but readable. A token you cannot see is a
token you cannot reason about, and "this one expired in March" is the answer to why a shortcut
stopped working. Revoke is only offered on a live row, since revoking a dead one is a `204` that
changes nothing.

## Revoking

Two clicks, not the type-the-name gate account deletion uses. The difference is recoverability: a
wrongly revoked token costs a minute and a new one. That gate exists for the things nothing can
undo, and spending it here would make it mean less where it counts.

## States

| State | What renders | Why |
|---|---|---|
| Loading | A line of text | — |
| Empty | "Most people never need one…" | A bare empty state would read as something missing |
| `404` | "Not available on this deployment yet", neutral | The path has no id in it, so a 404 can only mean the endpoint is not answering — the backend half of a release is not out. An error treatment would send somebody to check a fine connection |
| Other error | `DataStateMessage` with retry | A real failure, and this is a settings section with a visible retry — hence `retry: false` on the query |
| No active group | Mint disabled, with a line saying why | `POST` takes its group from the request, so there would be nothing to freeze onto the row. **The list still renders** — a token for a group you have left is exactly the one worth revoking |

## Visual baselines

`settings-light` and `settings-dark` both change: the section is on the account tab, which is what
those two capture. `settings-admin-*` and `settings-platform-*` do not — neither tab renders account
sections.

⚠️ They were **not** regenerated in `FootMania-Simple-Front#64`. The baselines are `win32` and the
branch was built on Linux, where regenerating adds a second platform's snapshots rather than
updating the existing ones.
