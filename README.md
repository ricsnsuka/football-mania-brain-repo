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
| The API surface | [backend/api/API_REFERENCE.md](backend/api/API_REFERENCE.md) |
| How to keep this repo true | [CONTRIBUTING.md](CONTRIBUTING.md) |

---

## How this is organised

```
README.md · STATUS.md · INDEX.md · CONTRIBUTING.md   ← the curated layer, written for this repo
product/          roadmap and cross-repo feature status
architecture/     system overview and the migration history — things neither repo owns alone
backend/          mirror of the backend's docs/ tree, plus its changelog and conventions
frontend/         mirror of the frontend's docs/ tree, plus its conventions
```

`backend/` and `frontend/` are **faithful mirrors** of each repo's `docs/` directory, imported on
2026-07-31. The tree shape was kept exactly as it was so that the relative links *between* those
documents still resolve. The trade-off is that their internal links to **source files**
(`src/main/java/...`, `src/features/...`) do not resolve here — those point into the code repos,
and are still correct there.

Two things were deliberately **not** imported:

- **The 14 agent definitions** in the backend's `.github/agents/`. They are executable
  configuration for the pipeline that drives that repo, so they have to live beside the code they
  act on. A summary of what the pipeline is lives in
  [backend/AGENT-PIPELINE.md](backend/AGENT-PIPELINE.md).
- **The Postman collection** (`postman/`). It is a machine-readable artefact, versioned with the
  API it exercises. Its current state is recorded in [STATUS.md](STATUS.md) — it is well behind.

The roadmap existed identically in both repos; the duplicates were removed and the single copy
lives at [product/roadmap.md](product/roadmap.md).

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
