# Settings Feature

`/settings` — everything about you, and for an admin, about the system.

> **Before this, there was no settings page and no profile page.** Notification and privacy settings
> rendered at the bottom of the dashboard, theme and language were widgets in the footer, rating
> recalculation sat on the Matches page for admins, and both role dashboards carried a "My Profile"
> card pointing at `/profile` with `disabled` on it. Four places, none of them an account page.

## Sections, in order

| # | Section | Who | Component |
|---|---------|-----|-----------|
| 1 | Profile | All | `ProfileSettings` |
| 2 | Your player | All | `LinkedPlayerSettings` |
| 3 | Appearance | All | `AppearanceSettings` |
| 4 | Notifications | All | `NotificationSettings` |
| 5 | System | `ADMIN` | `SystemSettings` |
| 6 | Your data | All | `PrivacySettings` |

Ordered by how often something is wanted and how much damage it does. Profile first because it is
why most people arrive; data export and account deletion last because they are irreversible and
should be somewhere a person lands deliberately.

**Stacked, not tabbed.** Tabs would hide the destructive section behind a click, which sounds safer
and is not: it makes it something you have to already know about, and the GDPR rights it carries are
ones people are entitled to find.

## Profile

Name, email and password. No new backend was needed — `PATCH /api/users/{id}` is authorised as
`hasRole('ADMIN') or #id == authentication.principal.id`, so it works for every role without a
role check in the UI.

**The user id comes from the session, never a form field.** One that could be edited would be an
authorisation bug waiting to be found.

**An emptied field sends `undefined`, not `""`.** The API treats an absent field as "leave alone"
and an empty string as "set it to blank", so clearing a name must send nothing rather than nothing-
in-quotes.

Username and role are shown but not editable: username is the login identity, and a role you could
grant yourself would not be a role. Both change only through admin.

Password reuses the existing `ChangePasswordModal`.

## Your player

Which player on the roster is you, via `LinkedPlayerBanner` and `SelfLinkPlayerModal` — moved here
from `features/dashboard`.

The dashboard keeps a *prompt* for the unlinked case, because being unlinked is worth interrupting
somebody about; **managing** the link is a settings action. See [dashboard](dashboard.md).

## Appearance

Theme and language, moved out of the footer, which no longer has any controls at all.

Presented as explicit choices rather than the footer's cycle button. Cycling is a reasonable
affordance in a corner of the chrome where space is the constraint and the current value is visible;
it is a poor one on a settings page, where the question is "what are my options" and a control that
reveals them only by being pressed three times answers it badly.

Both write to the same `appStore` slices the widgets used, so persistence and the system-preference
listener are unchanged — this is a different control over the same state.

`role="radiogroup"` rather than a row of buttons: these are states of one setting, and a screen
reader should announce "2 of 3" rather than three unrelated controls.

## System (admin only)

Guarded in the UI *and* by the API. The endpoints are `ADMIN` and would refuse anybody else, so
the UI guard is about not rendering controls that cannot work — not about keeping a secret.
`MANAGER` is refused too: master runs the squad, this is the system.

| Block | What |
|-------|------|
| Push health | Whether VAPID is configured, the subject, subscription count, badge and match counts, build version |
| Competition rules | The four settings from `GET /api/admin/settings` |
| Maintenance | Badge backfill, cache eviction, rating recalculation |

**Bounds come from the server with each value** — `min`, `max` and `defaultValue` are rendered from
the response, never hard-coded. A form carrying its own copy drifts the first time either changes,
and the failure mode is a field that looks valid and is rejected on save.

Edits are held in local `draft` state and only sent on Save, so a half-typed number is never
submitted. An out-of-range value marks the input `aria-invalid` and disables Save rather than
letting the request go and surfacing a 400.

> The draft is cleared on a **successful save and only then**. Clearing it whenever the settings
> query changes identity also fires on a background refetch, which would delete whatever the admin
> was part-way through typing for no reason they could see. `eslint`'s `react-hooks/set-state-in-
> effect` caught that during development.

Rating recalculation (`RecalculateMatchesPanel`) moved here from the Matches page. A maintenance
tool on a content page is surprising in both directions — invisible to the admin looking for it,
clutter for the one who is not.

The endpoint contract — including the cross-field rule on the leaderboard limits and why an unknown
setting name is a `400` — is `docs/api/ADMIN-API-CONTRACT.md` in the **backend** repo. Not linked
relatively: the two repos are separate checkouts, so a relative path here would resolve to nothing.

## Reaching it

The **account menu** on the right of the navbar (`AccountMenu`), behind your initials: Settings and
Logout. Settings is deliberately *not* in the main nav row — it is not a destination you browse
between, it is where your own things are, and that row is already long enough for an admin.

That menu replaced a bare "Logout" button, which was the only account-shaped control in the chrome
and had no route to an account behind it.

## File map

| Layer | File |
|-------|------|
| Route | `src/app/(app)/settings/page.tsx` |
| Page | `src/features/settings/SettingsPage.tsx` |
| Sections | `ProfileSettings.tsx`, `LinkedPlayerSettings.tsx`, `AppearanceSettings.tsx`, `NotificationSettings.tsx`, `SystemSettings.tsx`, `PrivacySettings.tsx` |
| Admin data | `src/hooks/admin/useAdmin.ts`, `src/services/adminService.ts`, `src/types/admin.ts` |
| Account menu | `src/components/layout/AccountMenu.tsx` |

## i18n

`settings.*` in each `locales/<lang>/common.json`. Setting labels are keyed by enum constant
(`settings.system.settings.MVP_VOTING_WINDOW_HOURS`) with the storage key as the i18next
`defaultValue`, so a setting added to the backend renders — as its raw key — before translations
catch up, rather than showing nothing.

## CSS

`.settings-*` and `.account-menu__*` in `globals.css`. `.settings-section__row` deliberately shares
its shape with `.privacy-settings__row`: one visual language for "a labelled thing with a control on
the right", so the sections read as siblings rather than as separate features sharing a page.
