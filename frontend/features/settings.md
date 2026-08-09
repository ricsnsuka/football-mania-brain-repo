# Settings Feature

`/settings` — three tabs, one per scope: you, the group you are acting in, and the platform.

> **Before this, there was no settings page and no profile page.** Notification and privacy settings
> rendered at the bottom of the dashboard, theme and language were widgets in the footer, rating
> recalculation sat on the Matches page for admins, and both role dashboards carried a "My Profile"
> card pointing at `/profile` with `disabled` on it. Four places, none of them an account page.

## Tabs and sections (owner-requested split, 2026-08-02)

| Tab | Section | Who | Component |
|-----|---------|-----|-----------|
| **My account** (default) | Profile | All | `ProfileSettings` |
| | Appearance | All | `AppearanceSettings` |
| | Notifications | All | `NotificationSettings` |
| | Your data | All | `PrivacySettings` |
| **{group name}** | Group name | `GROUP_ADMIN` | `GroupSettings` |
| | Your player | All | `LinkedPlayerSettings` |
| | Seasons | `GROUP_ADMIN` | `SeasonSettings` — from `features/seasons/` |
| | System | `GROUP_ADMIN` | `SystemSettings` |
| **Platform** | Creation codes | operator only — the tab does not render otherwise | `PlatformSettings` |

The tabs are the scopes tenancy created. Account settings follow the person across groups
(notifications are platform-level per the tenancy contract — a device belongs to a person, not a
group). The group tab is **titled with the active group's name**, like the Navbar brand and the
members page, so what "these settings" means changes legibly when you switch groups. Platform is
gated on the operator grant alone, which group `GROUP_ADMIN` deliberately does not imply.

**Seasons joined the group tab on 2026-08-07**, and the reasoning is the tab's own: a season belongs
to the active group and changes when you switch, which is what this tab is for. It shipped first as
a `/seasons` route with a navbar entry and was moved — [seasons](seasons.md#why-settings-and-not-a-page)
records why. It sits immediately above System, so the calendar and the rules read as one subject.

**The page was stacked before it was tabbed, and the stacking's reason survived the change.**
Stacking existed so the destructive GDPR section could not hide behind a click nobody knows to
make. The tabs keep that property by construction: privacy lives on the **default** tab and
renders last on it, so the landing view still ends at the person's data rights. The unit test that
guarded "privacy is last" now guards exactly that.

Tab markup and CSS mirror the match modal's tabs — the one established tab pattern in the app.
Sections were already lazy-loaded; tabs compound it, since a section on an unvisited tab is never
mounted at all.

## Profile

Name, email and password. No new backend was needed — `PATCH /api/users/{id}` is authorised as
`hasRole('GROUP_ADMIN') or #id == authentication.principal.id`, so it works for every role without a
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

Guarded in the UI *and* by the API. The endpoints are `GROUP_ADMIN` and would refuse anybody else, so
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
| Sections | `ProfileSettings.tsx`, `GroupSettings.tsx`, `LinkedPlayerSettings.tsx`, `AppearanceSettings.tsx`, `NotificationSettings.tsx`, `SystemSettings.tsx`, `PrivacySettings.tsx` |
| Composed from elsewhere | `features/seasons/SeasonSettings.tsx` — see [seasons](seasons.md). A feature with its own service, hooks and three modals lives with the feature; this page decides who sees it |
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
