# Football Mania — Brain Repo

**The source of truth for everything written down about this application.**

The product is built in two separate git repositories. This third repo is neither of them: it
holds no code, has no build and runs no tests. It exists so that the answer to "how does X work"
lives in one place instead of being reconstructed from two codebases every time.

| Repo | What it is | Where |
|------|-----------|-------|
| **Backend** | Java 21 · Spring Boot 3.4.5 · PostgreSQL · Flyway | [`ricsnsuka/FootMania-Back`](https://github.com/ricsnsuka/FootMania-Back) |
| **Frontend** | Next.js 16 App Router · React 19 · TypeScript · Tailwind 4 | `FootballMania/front/football` |
| **Brain** (this) | All documentation, canonical | [`ricsnsuka/football-mania-brain-repo`](https://github.com/ricsnsuka/football-mania-brain-repo) |

---

## Start here

| If you want | Read |
|-------------|------|
| Where the project actually stands today | [**STATUS.md**](STATUS.md) |
| A map of every document here | [**INDEX.md**](INDEX.md) |
| How the two repos fit together | [architecture/system-overview.md](architecture/system-overview.md) |
| What ships next and why | [product/roadmap.md](product/roadmap.md) |
| What is built, in which repo, and whether it is documented | [product/feature-status.md](product/feature-status.md) |
| The complete database migration history | [architecture/database-migrations.md](architecture/database-migrations.md) |
| The API surface | [API_REFERENCE.md](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/API_REFERENCE.md) — lives in the backend repo |
| Which repo a document belongs in | [**where-documents-live.md**](where-documents-live.md) |
| How to keep this repo true | [CONTRIBUTING.md](CONTRIBUTING.md) |

---

## How this is organised

```
README · STATUS · INDEX · CONTRIBUTING · where-documents-live   ← written for this repo
product/          roadmap and cross-repo feature status
architecture/     system overview and the migration history — things neither repo owns alone
backend/          architecture, features, plans, ADRs, audits, incidents
frontend/         features and the architecture overview
```

`backend/` and `frontend/` keep the tree shape each repo's `docs/` directory had, so the relative
links *between* those documents still resolve. Their links to **source files**
(`src/main/java/...`, `src/features/...`) do not resolve here — those point into the code repos, and
are still correct there.

**Not everything moved, and that is deliberate.** API contracts, the endpoint changelog, deployment
guides, each repo's changelog, the frontend's how-to guides, the agent definitions and the Postman
collection all stayed with their code, because they change in the same commit as the code they
describe — which is the only thing that has reliably kept documentation here accurate. The full
split, with the reasoning and links to everything that stayed, is in
[**where-documents-live.md**](where-documents-live.md).

Nothing is duplicated in both places. The roadmap used to be, in both repos, and both copies
drifted; it is now one file at [product/roadmap.md](product/roadmap.md).

---

## The rule

From 2026-07-31, **this repo is canonical**. Where a document here disagrees with one inside a
code repo, this one is the one to fix and the one to trust. Per-change API contracts still ship in
the same commit as the code that changes them — that discipline is what keeps the contracts honest
— and land here as part of the same work.

Everything imported here was written before this repo existed, so some of it is stale. Rather than
silently rewrite it, the drift found on import is catalogued in
[STATUS.md](STATUS.md#known-documentation-drift) with the correction. Fix a document and strike its
line from that list.
