# Where documents live

Decided 2026-07-31, when this repo became the source of truth.

## The rule

> **A document stays in a code repo if and only if it changes in the same commit as the code it
> describes. Everything else lives here.**

That is not a taste judgement, it is what the evidence said. An audit of both repos on 2026-07-31
found nine documents that were *wrong*, not merely old — a changelog twelve commits behind, a
migration table that stopped at V13 of 19, an index omitting seven of its own files. One category
came through untouched: the **API contracts**, the only documents governed by a rule that they ship
in the same commit as the change. Proximity to the change is what keeps a document honest. Distance
from it is what lets it rot.

So: documents that a code change invalidates stay next to that code, where the person making the
change cannot miss them. Documents that describe what the system *is*, why it is that way, or what
happened, live here, where both repos can see one copy.

**Nothing is duplicated.** Two copies of a document have no owner, and that is precisely the
mechanism that produced the drift above.

---

## Backend — [`ricsnsuka/FootMania-Back`](https://github.com/ricsnsuka/FootMania-Back)

**Stays there:**

| Path | Why |
|---|---|
| [`docs/api/`](https://github.com/ricsnsuka/FootMania-Back/tree/master/docs/api) | The API contract — 14 per-surface contracts plus `API_REFERENCE.md`. Changing an endpoint, DTO or status code updates its contract in the same commit. **This is the rule that works; do not weaken it** |
| [`docs/frontend/`](https://github.com/ricsnsuka/FootMania-Back/tree/master/docs/frontend) | The append-only endpoint changelog and the draft SSE guide. A backend change is what creates the entry |
| [`docs/deployment/`](https://github.com/ricsnsuka/FootMania-Back/tree/master/docs/deployment) | Heroku guide — tied to `Procfile`, `system.properties`, `Dockerfile` |
| [`CHANGELOG.md`](https://github.com/ricsnsuka/FootMania-Back/blob/master/CHANGELOG.md) | Release history for that artefact |
| [`.github/copilot-instructions.md`](https://github.com/ricsnsuka/FootMania-Back/blob/master/.github/copilot-instructions.md) · [`.github/agents/`](https://github.com/ricsnsuka/FootMania-Back/tree/master/.github/agents) | Conventions and the 14 agent definitions — executable configuration, useless away from the code it drives |
| [`README.md`](https://github.com/ricsnsuka/FootMania-Back/blob/master/README.md) | The repo's landing page |
| `postman/` | A machine-readable artefact, versioned with the API it exercises |

**Moved here:** architecture narrative (`backend/architecture/`), feature deep-dives
(`backend/features/`), plans and handoffs (`backend/plans/`), ADRs and release notes
(`backend/next-release/`), security audits (`backend/security/`), incidents (`backend/fixes/`).

## Frontend — [`ricsnsuka/FootMania-Simple-Front`](https://github.com/ricsnsuka/FootMania-Simple-Front)

**Stays there:**

| Path | Why |
|---|---|
| [`docs/guides/`](https://github.com/ricsnsuka/FootMania-Simple-Front/tree/v1.0.0/docs/guides) | Getting started, component conventions, shared primitives, styling, i18n, testing, Netlify deployment — how to write and ship code *in that repo* |
| [`AGENTS.md`](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/v1.0.0/AGENTS.md) · `CLAUDE.md` | "This is NOT the Next.js you know" — conventions, read before writing code |
| [`README.md`](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/v1.0.0/README.md) | The repo's landing page |

**Moved here:** all feature documentation (`frontend/features/`) and the architecture overview
(`frontend/architecture/`).

---

## Consequences worth knowing

**The backend agent pipeline was rewired.** Seven agent definitions wrote into `docs/features/`,
`docs/plans/`, `docs/next-release/`, `docs/security/` and `docs/fixes/`. Left alone they would have
recreated every directory that moved, and the drift would have restarted within a week. They now
name this repo's paths directly — `../football-mania-brain-repo/backend/…` — which assumes it is
cloned alongside the code repos. `documentation-organizer` carries a scope note telling it to
organise here, not there.

**Writing documentation is now sometimes two commits in two repos.** That is already true of every
API change in this project, and it is the price of one copy of the truth. The alternative — a copy
in each repo — is what was just cleaned up.

**The roadmap was duplicated in both repos and both copies drifted.** It is now one file:
[product/roadmap.md](product/roadmap.md).
