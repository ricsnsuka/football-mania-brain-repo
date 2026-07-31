# Achievement Badges

Milestones a player has earned, shown as a row of chips in the player modal under the stats grid.
Read-only: `GET /api/players/{id}/badges`.

## Names are localised, with the server's name as the fallback

The API sends `displayName` precisely so a new badge needs no frontend deploy — but that name is
English, and this app ships in three languages. So each badge is looked up by its **stable
identifier** with the server's name as the i18next `defaultValue`:

```tsx
t(`badges.names.${earned.badge}`, { defaultValue: earned.displayName })
```

That keeps both properties, and neither approach alone would:

- a badge the locale files know reads in the user's language;
- a badge added to the backend tomorrow renders — in English — the moment it is awarded, instead of
  showing a raw `FIFTY_GOALS` until three locale files are updated.

Switching on `displayName` would throw away the stable key; ignoring it would throw away the
graceful fallback. `src/tests/badges/PlayerBadges.test.tsx` covers the fallback specifically, with
a badge the locale deliberately does not know.

## The section hides itself

`PlayerBadges` renders **nothing at all** when there are no badges, while loading, or if the fetch
fails. An empty heading over an empty list is worse than silence on a profile — and the modal is
opened to see a *player*, so a badge request failing must not take the rest of it down.

## The catalogue

Nine badges, all derived from aggregates that already exist. Adding one is a backend enum entry
plus a threshold; the frontend needs a locale key only if it should be translated.

| Badge | Earned when |
|-------|-------------|
| `FIRST_MATCH` / `TEN_MATCHES` / `FIFTY_MATCHES` | 1 / 10 / 50 completed matches |
| `FIRST_GOAL` / `TEN_GOALS` / `FIFTY_GOALS` | 1 / 10 / 50 career goals |
| `FIRST_ASSIST` | 1 career assist |
| `WIN_STREAK_5` | `longestStreak` reaches 5 |
| `FIRST_MVP` | Named MVP by an **admin** at least once (not the crowd MOTM result) |

Declaration order in the backend enum is the display order.

## Two things to expect

**A burst on the first match after deployment.** Existing players are deliberately not backfilled —
every badge would otherwise land with the same timestamp, claiming the whole roster earned
everything at once. So the first completed match awards each participant their entire history in one
go. A "new!" highlight keyed on `awardedAt` being recent would light up every profile that day.

**Awards are permanent.** There is no revocation path anywhere. A later downward stat amendment does
not remove a badge — `awardedAt` records that a threshold was crossed at that moment, which stays
true regardless of later edits.

## `matchId` may be absent

Omitted (not `null`) when the citing match has been deleted — the award outlives the match record.
Guard before linking to it.

## File map

| Path | Role |
|------|------|
| `src/features/players/PlayerBadges.tsx` | The chips section |
| `src/hooks/badge/useBadges.ts` | `usePlayerBadges` |
| `src/services/badgeService.ts` | `fetchPlayerBadges` |
| `src/types/badge.ts` | Types + zod schema |

`PlayerModal.test.tsx` **stubs** this section rather than mocking its hook, for the same reason
`MatchModal.test.tsx` stubs the MOTM panel.

## i18n keys (`badges` namespace)

`title`, `awarded`, and `names.<BADGE_ID>` for each catalogue entry.

Backend contract: `docs/api/BADGES-API-CONTRACT.md`.
