# Privacy & Your Data

**Added in:** v1.0.0
**Status:** ✅ Implemented — operator contact details still a placeholder
**Backend contract:** `docs/api/PRIVACY-API-CONTRACT.md` in the backend repo

The frontend for the GDPR rights the backend has served since Phase 0: access, portability and
erasure. Until this landed, all four `/api/privacy/*` endpoints were working, tested and
completely unreachable from the UI.

---

## Files

```
src/services/privacyService.ts                 API calls + the blob download
src/features/settings/PrivacySettings.tsx      Self-service: export, delete account
src/features/players/PlayerPrivacyZone.tsx     Admin, acting on a player's behalf
src/app/privacy/page.tsx                       Public policy page
src/types/privacy.ts
```

`PrivacySettings` sits at the bottom of the dashboard — deleting an account is irreversible, so
it should be somewhere a person arrives deliberately rather than the first thing under the fold.
`PlayerPrivacyZone` is in the player modal, below the existing delete zone.

---

## The export cannot be a link

Two constraints that rule out the obvious implementation:

- **`<a href download>` will not work.** The endpoint requires an `Authorization` header and a
  navigation request cannot carry one.
- **`apiFetch` will not work either.** It parses JSON; the point here is to get the bytes
  untouched and hand them to the browser.

So `downloadExport` uses a bare `fetch` with the token, takes a `blob()`, and drives a synthetic
anchor. `URL.revokeObjectURL` afterwards — without it the blob is held for the lifetime of the
document, which is sloppy on a page someone leaves open.

---

## Deleting an account: three things the UI must get right

**1. Say what survives, before offering the button.** Erasure anonymises rather than deletes:
the account, name and phone number go; match results and statistics stay in the group history
with no name attached. That is not a detail — someone expecting every trace to vanish should
find out on this screen, not afterwards. The explanatory text is asserted by a test so it cannot
be quietly dropped.

**2. Require the username typed, not a second click.** This cannot be undone by support. A
confirm dialog is a reflex; typing your own username is a deliberate act.

**3. Clear the session and redirect on success.** This is the one that would look like a bug:
the JWT stays *syntactically* valid after the account behind it is gone, so leaving the user
signed in means every subsequent request fails for no visible reason. `clearAuth()` then
`router.replace('/login')`.

Equally, a **failed** erasure must leave them signed in — signing someone out of an account that
still exists is its own bug. Both directions are tested.

---

## The admin flow exists for people who cannot ask

`PlayerPrivacyZone` is not a convenience. A player an admin created who never registered has no
account, therefore no login, therefore no way to reach the self-service controls. Without this,
the backend capability is unreachable for exactly the people most likely to need it.

It is kept visually separate from the existing delete zone because the two do different things:
**delete** removes a player who has no history; **erase** keeps the history and removes the
person. Same confirmation discipline — the player's name typed exactly.

After erasing, the player query cache is invalidated: the roster now shows a tombstone name and
an inactive player, so a cached list is wrong.

---

## The public policy page

`app/privacy/page.tsx`, deliberately **outside the `(app)` route group**. Both app stores
require a policy reachable without signing in (Phase 4), and a page behind the auth guard does
not satisfy that. It is a Server Component with no client hooks, so it renders with no
JavaScript, and it prerenders statically.

Its content mirrors `PRIVACY_AND_DATA_PROTECTION.md` in the backend repo, which is the source of
truth for what is actually stored. **If that changes, change this.**

> ⚠️ **Still a placeholder:** the operator's identity and contact address read
> `[operator email]`. That must be filled in before any public launch or store submission. The
> page is accurate about the system's behaviour, which is what an engineer can verify; it is not
> legal advice.

---

## Tests

| File | Covers |
|------|--------|
| `src/tests/components/PrivacySettings.test.tsx` | Export triggers, first click only reveals the confirmation, exact-username gate, session cleared and redirected on success, session **kept** on failure, the survives-deletion warning, policy link |
| `src/tests/components/PlayerPrivacyZone.test.tsx` | Export by player id, first click only reveals, exact-name gate, erase called with the right id, warning shown before the confirm control |
