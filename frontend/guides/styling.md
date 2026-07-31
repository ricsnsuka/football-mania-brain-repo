# Styling Guide

## Core Rules

| # | Rule | Enforcement |
|---|------|-------------|
| 1 | **Zero Tailwind in JSX** — no utility classes in `.tsx` files | Hard rule, no exceptions |
| 2 | **Single style file** — all CSS lives in `src/app/globals.css` under `@layer components` | Hard rule |
| 3 | **BEM class names** — `block__element--modifier`, kebab-case | Hard rule for **new** blocks; see the note below on existing ones |
| 4 | **`w-full px-4 py-8` for page roots** — page components fill the `PageContainer`, no `max-w-*` or `mx-auto` | Hard rule |
| 5 | **Semantic colour tokens** — `bg-surface`, `text-ink-muted`, `border-line` — not raw `gray-*` pairs | Hard rule for colours that have a token |
| 6 | **Dark mode via `dark:` variant** — never use `@media (prefers-color-scheme)` | Hard rule |
| 7 | **Responsive variants inside CSS** — never in JSX | Hard rule |

> **Rule 3, honestly.** The file currently holds ~870 hyphenated class names (`players-card-stat-label`,
> `match-modal-header`) against ~370 true BEM ones. The hyphenated majority predates this rule. Use
> BEM for new blocks; **do not** rename an existing feature's classes purely to comply, because a
> half-renamed file is less navigable than a consistently old-fashioned one.

> **Rule 4 is not decorative.** `.rankings-page` shipped without it and sat flush against the
> container while every sibling page had padding. It is the kind of thing that reads as "this page
> looks wrong" without anyone being able to say why.

---

## Zero Tailwind in JSX

No Tailwind utility classes may appear inside any `.tsx` file. All styles are defined in `src/app/globals.css`.

```tsx
// ✅ correct
<div className="player-card">
  <span className="player-card__name player-card__name--highlighted">…</span>
</div>

// ❌ wrong
<div className="flex items-center gap-4 rounded-lg bg-white dark:bg-gray-900">
```

The only exception is a single semantic class per element. Multiple classes are allowed only when they are BEM modifier combinations (block + modifier):

```tsx
// ✅ allowed — block class + modifier class
<button className={`filter-btn ${active ? 'filter-btn--active' : ''}`}>
```

---

## Where Styles Live

**Single file**: `src/app/globals.css`

```css
@import "tailwindcss";

/* custom dark-mode variant (uses .dark class, not media query) */
@custom-variant dark (&:where(.dark, .dark *));

@layer components {
  /* ── Player Card ──────────────────────────────────────────── */
  .player-card {
    @apply flex flex-col rounded-xl border border-line bg-surface;
  }

  .player-card__header {
    @apply flex items-center gap-4 p-4;
  }

  .player-card__name--highlighted {
    @apply font-bold text-accent;
  }
}
```

Note what is **absent** above: no `dark:` variants. The tokens carry the theme themselves.

---

## Colour Tokens

Colours are named for **what they are for**, not which grey they happen to be. Defined once in
`:root`, overridden in `.dark`, exposed through `@theme inline`.

| Token | Utility | Light | Dark | Use for |
|-------|---------|-------|------|---------|
| `--surface` | `bg-surface` | white | gray-900 | A raised card or panel |
| `--surface-sunken` | `bg-surface-sunken` | gray-50 | gray-800 | An inset well inside a card |
| `--surface-ghost` | `bg-surface-ghost` | white | transparent | Outline / secondary buttons |
| `--surface-control` | `bg-surface-control` | white | gray-800 | Selects and raised controls in a modal |
| `--surface-selected` | `bg-surface-selected` | white | gray-700 | The chosen segment of a filter group |
| `--line` | `border-line` | gray-200 | gray-700 | Hairline between components |
| `--line-soft` | `border-line-soft` | gray-100 | gray-800 | Hairline between rows inside one |
| `--ink` | `text-ink` | gray-900 | gray-100 | Headings, primary numbers |
| `--ink-body` | `text-ink-body` | gray-700 | gray-300 | Body text |
| `--ink-muted` | `text-ink-muted` | gray-500 | gray-400 | Secondary text, labels |
| `--ink-faint` | `text-ink-faint` | gray-400 | gray-500 | Timestamps, hints |
| `--ink-ghost` | `text-ink-ghost` | gray-400 | gray-600 | Separators, disabled affordances |
| `--ink-static` | `text-ink-static` | gray-500 | gray-500 | **Same in both themes** — an inactive chip should read as switched off |
| `--accent` | `text-accent` | blue-600 | blue-400 | Links, active state, primary numbers |

**Why.** Dark mode used to be restated by hand at every call site — 41% of all `@apply` lines
carried a `dark:` variant and `dark:text-gray-400` alone appeared 87 times. Each was somewhere the
light/dark pairing could drift, and a repaint meant editing hundreds of declarations. A token flips
once.

```css
/* ✅ correct */
.thing { @apply border border-line bg-surface text-ink-muted; }

/* ❌ wrong — hand-paired, and one of the two will eventually be forgotten */
.thing { @apply border border-gray-200 bg-white text-gray-500
                dark:border-gray-700 dark:bg-gray-900 dark:text-gray-400; }
```

**Adding a token** means three edits: `:root`, `.dark`, and `@theme inline`. Alias the Tailwind
scale variable (`var(--color-gray-200)`) rather than a hex literal, so the value stays identical to
the utility it replaces.

**Still write `dark:` by hand** for anything with no token — brand colours, `blue-950/40` hover
states, translucent overlays. Roughly 480 such variants remain and they are correct.

---

## Shared Primitives

Seven recurring ideas, declared with `@utility` so they can be `@apply`-ed:

| Primitive | Is |
|-----------|-----|
| `ui-title` | A card or modal section heading |
| `ui-caption` | Secondary prose under a heading |
| `ui-label` | The small caption beneath a statistic |
| `ui-hint` | Tertiary detail — timestamps, counts |
| `ui-chip-success` / `ui-chip-danger` / `ui-chip-info` | Status chip **palette** (not the shape) |

```css
.my-feature__label { @apply ui-label; }
```

> **`@utility`, not a plain class.** Tailwind v4 refuses to `@apply` a class defined in
> `@layer components` — it errors with `Cannot apply unknown utility class`. A registered utility
> can be applied. This is why the primitives are declared above the `@layer components` block.

Do **not** add primitives for pure layout. `flex flex-col gap-3` appears 10 times and stays that
way: a `stack-3` utility would be less readable than the Tailwind it replaced. A primitive should
carry meaning.

---

## Class Naming Convention

BEM-inspired: `block__element--modifier`

| Part | Pattern | Example |
|------|---------|---------|
| Block | `feature-name` (kebab-case) | `player-card`, `match-row`, `dashboard` |
| Element | `block__element` | `player-card__avatar`, `match-row__score` |
| Modifier | `block--modifier` or `block__element--modifier` | `player-card--inactive`, `filter-btn--active` |

```css
/* ✅ correct */
.player-card { … }
.player-card__avatar { … }
.player-card__name { … }
.player-card__name--highlighted { … }
.player-card--inactive { … }

/* ❌ wrong — utility soup */
.flex-row-card-thing { … }
```

---

## Organization Inside globals.css

Group related classes under a single-line comment header. Maintain the existing alphabetical order within each feature area. Keep infrastructure sections (Navbar, Footer, Buttons, Forms, Modals) at the top, and page/feature sections below.

```css
@layer components {

  /* ── Navbar ──────────────────────────────────────────────── */
  .navbar { … }
  .navbar-brand { … }

  /* ── Footer ──────────────────────────────────────────────── */
  .footer { … }

  /* ── Page container ──────────────────────────────────────── */
  .page-container { … }

  /* ── Buttons ─────────────────────────────────────────────── */
  .btn-primary { … }
  .btn-secondary { … }

  /* ── Modal ───────────────────────────────────────────────── */
  .modal-dialog { … }

  /* ── Dashboard ───────────────────────────────────────────── */
  .dashboard { … }

  /* ── Players page ────────────────────────────────────────── */
  .players-page { … }
}
```

---

## Page Layout Pattern

All authenticated page components are wrapped by `PageContainer`, whose `.page-container`
class (`src/app/globals.css`) is `w-full max-w-5xl mx-auto` — a real, centered
max-width constraint, not the "80% width" this doc previously (and incorrectly) claimed
before `.page-container` had any rules at all. Page root classes must **not** re-constrain
the width — this includes the dashboard route, whose own `.dashboard` class used to carry a
redundant `max-w-5xl mx-auto` that caused a width jump versus other pages; it now only owns
vertical rhythm (`w-full px-4 py-6 …`) and relies on the shared `.page-container` for width.

```css
/* ✅ correct — fills the container */
.players-page {
  @apply w-full px-4 py-8 flex flex-col gap-6;
}

/* ❌ wrong — fights the container */
.players-page {
  @apply max-w-6xl mx-auto px-4 py-8 flex flex-col gap-6;
}
```

The `PageContainer` is applied once in `src/app/layout.tsx`. Feature components must never add their own centering or max-width constraint.

---

## Dark Mode

The app uses a `.dark` class toggled on `<html>` — **not** the `prefers-color-scheme` media query. Tailwind's `dark:` variant works because `globals.css` declares:

```css
@custom-variant dark (&:where(.dark, .dark *));
```

**Prefer a token over a hand-written pair.** Where one exists, it already carries both themes:

```css
/* ✅ correct */
.stat-card { @apply bg-surface text-ink; }

/* ⚠️ works, but hand-maintained — only for colours with no token */
.stat-card { @apply bg-violet-100 text-violet-700
                    dark:bg-violet-900/60 dark:text-violet-300; }

/* ❌ wrong — a light colour with no dark counterpart at all */
.stat-card { @apply bg-white text-gray-900; }
```

The last case is the one to watch: it does not fail, it just stays white in dark mode. A handful
exist in the file today and are deliberate (modal close buttons at `text-gray-400` read fine in
both), but a new one is almost always an oversight.

---

## Responsive Variants

Apply breakpoints inside CSS classes, never in JSX:

```css
/* ✅ correct */
.player-card {
  @apply p-3 sm:p-5 lg:p-6;
}

/* ❌ wrong — breakpoints in JSX */
<div className="p-3 sm:p-5 lg:p-6">
```

---

## Shared Patterns

> ⚠️ **Most of the class names in this section are templates to copy, not classes that exist.**
> Only `.pagination`, `.pagination-footer` and `.modal-dialog` are real shared classes in
> `globals.css`. `.status-chip`, `.filter-bar`, `.filter-btn` and `.feature-table` are **not
> defined anywhere** — each feature writes its own (`players-filter-btn`, `rankings-filter-btn`,
> `matches-filter-btn`, …). Reading this section as an inventory of available classes will send you
> looking for something that was never there. Copy the shape; name it for your block.

### Chips / Badges

Status chips use a base class + colour modifier. Take the palette from a `ui-chip-*` primitive so
the colours stay in one place:

```css
.my-feature__chip {
  @apply rounded-full px-2.5 py-0.5 text-xs font-semibold;
}
.my-feature__chip--active   { @apply ui-chip-success; }
.my-feature__chip--inactive { @apply bg-surface-sunken text-ink-static; }
```

The shape (padding, radius, weight) deliberately stays with the feature — chips differ in size by
context — while the palette comes from the primitive.

### Filter Toolbars

```css
.filter-bar {
  @apply flex items-center gap-2 flex-wrap;
}
.filter-btn {
  @apply rounded-lg px-3 py-1.5 text-sm font-medium border transition-colors
         focus:outline-none focus:ring-2 focus:ring-blue-500
         border-gray-200 bg-white text-gray-600
         hover:bg-gray-50 hover:text-gray-900
         dark:border-gray-700 dark:bg-gray-900 dark:text-gray-400
         dark:hover:bg-gray-800 dark:hover:text-gray-200;
}
.filter-btn--active {
  @apply border-blue-500 bg-blue-50 text-blue-700
         dark:border-blue-500 dark:bg-blue-950 dark:text-blue-300;
}
```

### Data Tables

Tables use `border-collapse`, semantic `th`/`td` selectors scoped to the block, and colored result spans:

```css
.feature-table {
  @apply w-full border-collapse text-sm;
}
.feature-table thead tr {
  @apply border-b border-gray-100 dark:border-gray-800;
}
.feature-table th {
  @apply px-3 py-2 text-left text-xs font-semibold uppercase tracking-wider
         text-gray-400 dark:text-gray-500;
}
.feature-table tbody tr {
  @apply border-b border-gray-50 dark:border-gray-800/60;
}
.feature-table td {
  @apply px-3 py-2 text-gray-700 dark:text-gray-300;
}
```

### Modals

Use the shared `modal-dialog` class (native `<dialog>` element). Feature-specific modals extend it:

```css
/* shared — already in globals.css */
.modal-dialog { … }
.modal-header { … }
.modal-title { … }
.modal-body { … }
.modal-actions { … }

/* feature override only when needed */
.player-modal {
  @apply fixed bottom-0 left-0 right-0 w-full rounded-t-2xl rounded-b-none
         sm:top-1/2 sm:left-1/2 sm:-translate-x-1/2 sm:-translate-y-1/2
         sm:bottom-auto sm:w-[calc(100%-2rem)] sm:max-w-lg sm:rounded-xl
         bg-white shadow-2xl max-h-[85dvh] overflow-y-auto
         dark:bg-gray-900 dark:text-gray-100;
  border: none;
}
```

All modals (`.modal-dialog`, `.player-modal`, `.match-modal`, `.create-player-modal`,
`.create-match-modal`, `.match-plan-modal`, …) use the **same centered
`top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2`** placement at every breakpoint — there
is no mobile bottom-sheet variant. (A prior version of this doc described a
`bottom-0 rounded-t-2xl` → centered-at-`sm:` bottom-sheet pattern; it was never implemented
and has been removed from this guide. If a bottom-sheet pattern is wanted in the future,
add it as new CSS and document it here — don't assume it already exists.)
```

### Pagination

```css
.pagination {
  @apply flex items-center justify-between gap-4 flex-wrap;
}
.pagination-info {
  @apply text-sm text-gray-500 dark:text-gray-400;
}
.pagination-controls {
  @apply flex items-center gap-2;
}
.pagination-btn {
  @apply rounded-lg px-3 py-1.5 text-sm font-medium border transition-colors
         focus:outline-none focus:ring-2 focus:ring-blue-500
         border-gray-200 bg-white text-gray-600
         hover:bg-gray-50 hover:text-gray-900
         disabled:opacity-40 disabled:cursor-not-allowed
         dark:border-gray-700 dark:bg-gray-900 dark:text-gray-400
         dark:hover:bg-gray-800 dark:hover:text-gray-200;
}
```

### Loading / Skeleton States

```css
.feature-skeleton {
  @apply w-full px-4 py-8 h-96 rounded-xl
         bg-gray-100 animate-pulse dark:bg-gray-800;
}
```

Full-screen redirect guards use `.dashboard-loading` + `.dashboard-loading-spinner` (shared, already in globals.css).

---

## Minimum Touch Target

All interactive elements (buttons, links, inputs) must meet the 44 px minimum touch target:

```css
.my-btn {
  @apply min-h-[44px];
}
```

---

## What Never to Do

| Forbidden | Correct alternative |
|-----------|-------------------|
| `<div className="flex items-center gap-2">` | `<div className="player-card__meta">` with CSS in globals.css |
| `max-w-* mx-auto` on a page root class | `w-full` — the `PageContainer` owns centering |
| `@media (prefers-color-scheme: dark)` | `dark:` variant (`.dark` class) |
| Inline `style={{}}` for layout | CSS class in globals.css |
| Creating a new `*.module.css` file | Add to globals.css under `@layer components` |
| Defining types inline in component files | Import from `src/types/` |

---

## Verifying a Styling Change

`npm run lint`, `npm run type-check` and the unit tests say **nothing** about appearance. A colour
swapped for a near neighbour compiles, lints, and passes 415 tests.

```bash
npm run test:visual          # screenshots every page + the player modal, both themes
npm run test:visual:update   # accept a deliberate change as the new baseline
```

Run it before and after any change to `globals.css`. It has already caught two real regressions:
a resting border that turned blue in dark mode, and a page root missing its `w-full px-4 py-8`.

> **`--update-snapshots` is how a real regression quietly becomes the new normal.** Look at the
> diff in `test-results/` first. See `e2e/README.md`.

Note the suite screenshots **pages plus the player modal**. Content that only appears in another
modal or behind an interaction is not covered — if you style something there, add a case rather
than assuming green means protected.
