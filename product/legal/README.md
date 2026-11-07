# Legal documents

**Status: drafts, started 2026-11-07. Not in use and not legal advice.** A lawyer should review both
before anybody is asked to accept them. Nothing in the apps shows or records them yet.

| Document | Shown when | Who accepts |
|---|---|---|
| [Terms of Service](TERMS-OF-SERVICE.md) | Creating an account | Everyone with an account |
| [Group Organiser Terms](GROUP-ORGANISER-TERMS.md) | Creating a group | The person founding the group |

The Terms of Service apply to everyone. The Group Organiser Terms add to them for the person who
founds a group, because a group holds data about other people and an account does not.

Like the privacy policy, these files are the source of truth, and the apps show the same text in all
three languages. If either changes, the screens change with it.

## To settle before these go live

Each is marked `[OWNER: …]` in the text.

- **Who the operator is:** a person or a company, its legal name and its address.
- **Minimum age.** The sign-up form has no age check today. The age a child can consent alone varies
  by country in the EU, so this one needs advice.
- **Governing law and courts.**
- **The Apple end user terms.** The App Store has a standard licence, and an app with its own terms has
  to fit alongside it.
- **Whether group data needs a data processing agreement.** See the note in the
  [privacy document](../../backend/features/PRIVACY_AND_DATA_PROTECTION.md#data-processing-agreement).
  The Group Organiser Terms carry a short version for review.

## What has to be built to present them

None of this exists yet.

- **Backend:** a record of who accepted which version and when, for both documents. `POST
  /api/users/register` and `POST /api/groups` refuse without an acceptance of the current version.
- **Web and iOS:** an unticked box with a link to the document on the sign-up and create-group
  screens, and public pages for both, reachable without signing in, as `/privacy` is.
- **Existing users:** a one-time prompt at the next sign-in, and the same when a version changes in a
  way that needs a new acceptance.
- **Translations:** Portuguese and Spanish, reviewed rather than only machine translated.

## Depends on

- ~~The privacy policy brought up to date~~
  ([FootMania-Simple-Front#325](https://github.com/ricsnsuka/FootMania-Simple-Front/issues/325)).
  Merged into the web's `next` on 2026-11-08; live with the next web release.
- ~~The Terms of Service promise a way to block someone~~
  ([FootMania-Back#399](https://github.com/ricsnsuka/FootMania-Back/issues/399)). Live in 3.20.0 on the
  backend and the web; the iPhone half ships with the next TestFlight build.
- ~~The Group Organiser Terms describe deleting a group~~
  ([FootMania-Back#398](https://github.com/ricsnsuka/FootMania-Back/issues/398)). Live in 3.20.0 on the
  backend and the web; the iPhone half ships with the next TestFlight build.

The plumbing is tracked in [football-mania-brain-repo#124](https://github.com/ricsnsuka/football-mania-brain-repo/issues/124),
for 3.21.0.
