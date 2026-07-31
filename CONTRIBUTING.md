# Keeping this repo true

A source of truth is only worth the trust people put in it. One wrong table read as authoritative
does more damage than the same table in a repo nobody believed.

## The rules

**1. This repo is canonical.** When a document here and one in a code repo disagree, this is the
one to fix and to trust.

**2. Per-change API contracts still ship with the code.** A change that touches the API updates its
contract in `docs/api/` in the *same commit* — that is what keeps contracts honest, because the
person changing the endpoint is the person who knows what changed. Bring it here as part of the
same work.

**3. Write why, not what.** The code already says what. These documents earn their keep by
recording the decision, the alternative that was rejected, and the trap that was hit. The migration
history could be reconstructed accurately in an afternoon precisely because every migration opens
with a comment explaining why it exists.

**4. State the status, and date it.** Any document describing work in progress gets a `Status:` line
that says whether it is a draft, shipped, or abandoned. A plan that shipped and still says "DRAFT —
not implemented" sends the next reader to build it again.

**5. Never silently narrow.** If a document is wrong and you cannot fix it now, add it to the drift
table in [STATUS.md](STATUS.md) rather than leaving it to be discovered.

**6. Link, don't copy.** The roadmap was duplicated in two repos and both copies drifted. One file,
many links.

## When something ships

- [ ] Contract updated in the code repo, same commit as the code
- [ ] Feature doc here updated — or written, if the feature is new
- [ ] [product/feature-status.md](product/feature-status.md) row updated
- [ ] Migration added to [architecture/database-migrations.md](architecture/database-migrations.md)
      if the schema moved
- [ ] User-visible strings exist in **all three locales** — `en`, `pt`, `es`. A missing key falls
      back to the raw enum name and reaches production looking like `FEE_CHARGED`
- [ ] [STATUS.md](STATUS.md) still accurate

## Structure

Curated documents live at the top level and in `product/` and `architecture/`. `backend/` and
`frontend/` mirror each code repo's `docs/` tree — keep that shape, because the relative links
inside those documents depend on it. A new document about one side goes in that side's tree; a
document about how the two fit together goes in `architecture/`.
