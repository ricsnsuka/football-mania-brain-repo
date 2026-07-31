# Shared UI Primitives

Four small components in `src/components/ui/` are the canonical primitives for pagination,
loading, and empty/error states. They came out of the 2026-07-27 remediation, which found the
same three problems (pagination, loading, empty/error) reimplemented slightly differently on
almost every list page. **Reach for these instead of hand-rolling a new pagination control,
skeleton block, or empty-state message** — a new list/table feature should not need bespoke
CSS or markup for any of the three.

## `Pagination`

**Location**: `src/components/ui/Pagination.tsx`

Generic pagination molecule with no assumptions about what is being paginated (client-side
sliced array or server-paginated `Page<T>` — both work).

| Prop | Type | Description |
|------|------|-------------|
| `page` | `number` | Current page, **0-indexed** (matches `Page<T>.number` and typical local `page` state) |
| `totalPages` | `number` | Total number of pages |
| `totalElements` | `number` | Total item count, shown via `pagination.totalItems` |
| `pageSize` | `number` | Current page size |
| `pageSizeOptions` | `number[]` | Options rendered in the size `<select>` |
| `onPageChange` | `(page: number) => void` | Called with the next 0-indexed page |
| `onPageSizeChange` | `(size: number) => void` | Called when the size selector changes |

Renders as `<nav className="pagination">` with position text ("Page {page+1} of
{totalPages}"), a total-items count, a page-size `<select>`, and prev/next buttons — using
the `pagination.*` i18n keys (`position`, `totalItems`, `pageSize`, `previous`, `next`).
Callers own resetting `page` to `0` on filter/size changes; `Pagination` itself is stateless.

## `TableSkeleton`

**Location**: `src/components/ui/TableSkeleton.tsx`

Geometry-matched loading placeholder for a list rendered as either a table (desktop) or a
stack of cards (the mobile card-collapse pattern used by players/users/etc.) — renders
`rows` placeholder rows/cards so real content mounts without layout shift once data resolves.

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `variant` | `'table' \| 'card'` | — (required) | Which geometry to render |
| `rows` | `number` | `5` | Number of placeholder rows/cards |
| `columns` | `number` | `5` | Table variant only — number of placeholder columns |

Renders `role="status" aria-live="polite"` with a visually-hidden `common.loading` label, so
assistive tech announces the loading state once, rather than reading through placeholder
markup (which is `aria-hidden`).

## `DashboardSkeleton`

**Location**: `src/components/ui/DashboardSkeleton.tsx`

Structured loading placeholder for dashboard routes — a header bar plus a grid of
card-shaped blocks matching `DashboardNavCard` layout, instead of one undifferentiated
pulsing rectangle.

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `cards` | `number` | `6` | Number of placeholder nav-card blocks in the grid |

Same `role="status" aria-live="polite"` + visually-hidden label pattern as `TableSkeleton`.
Note its CSS is namespaced `dash-skeleton*`, not `dashboard-skeleton*` — that class name was
already taken by an older single-block skeleton rule (`.dashboard-skeleton`, still used
ad hoc in the three role dashboards). Wiring the dashboards over to this component fully is
a follow-up integration step, not yet done everywhere.

## `DataStateMessage`

**Location**: `src/components/ui/DataStateMessage.tsx`

Unified empty vs. error state for lists — previously every list page rendered its own plain
"no results" / "failed to load" text with no visual distinction between the two and no retry
affordance on error.

| Prop | Type | Description |
|------|------|-------------|
| `state` | `'empty' \| 'error'` | Which state to render |
| `onRetry` | `() => void` (optional) | If provided and `state === 'error'`, renders a retry button |

Uses `data-state-message--empty` / `data-state-message--error` modifier classes so styling
can (and does) give errors distinct treatment, and sets `role="alert"` for the error case
vs. `role="status"` for empty, so screen readers announce failures more assertively than a
merely-empty list.

## Typical usage

```tsx
{isLoading ? (
  <TableSkeleton variant="table" rows={pageSize} columns={8} />
) : isError ? (
  <DataStateMessage state="error" onRetry={refetch} />
) : items.length === 0 ? (
  <DataStateMessage state="empty" />
) : (
  <MyTable items={items} />
)}

{totalElements > 0 && (
  <Pagination
    page={page}
    totalPages={totalPages}
    totalElements={totalElements}
    pageSize={pageSize}
    pageSizeOptions={[5, 10, 20]}
    onPageChange={setPage}
    onPageSizeChange={handlePageSizeChange}
  />
)}
```

See `src/features/players/PlayersPage.tsx` for a full reference implementation combining all
four (mirrored for both the desktop table and mobile card variants).
