---
feature: Events (View)
group: New
last_synced: '2026-06-11'
last_commit: ba02630c316c435e071a627f433a21d08f9987e7
anchors:
  tables:
  - events
  endpoints:
  - GET /debug/events
  - GET /debug/events/kinds
  types:
  - EventKindCount
  - EventRecord
  api_modules:
  - api.eventKinds
  - api.events
  files:
  - frontend/src/api.ts
  - frontend/src/components/Card.tsx
  - frontend/src/components/Table.tsx
  - frontend/src/views/**
  - frontend/src/views/Events.tsx
  - placer/api/debug.py
writes: []
reads:
- frontend/src/api.ts
- frontend/src/components/Card.tsx
- frontend/src/components/Table.tsx
- frontend/src/views/Events.tsx
- placer/api/debug.py
---
## Capability — what it can do

The Events view provides a paginated, filterable read-only log of every record stored in the `events` table — the central spine of the Placer placement controller. Each row in the log is an `EventRecord` carrying a monotonically increasing sequence number (`seq`), an `event_kind` string, an optional `order_id` foreign key, two timestamps (`observed_at` / `recorded_at`), and three JSON blobs: `entity_refs`, `payload`, and `provenance`.

Two filter controls narrow the visible set:

- **Event-kind selector** — a `<select>` populated from `GET /debug/events/kinds`, which returns every distinct `event_kind` together with its row count. Choosing a kind restricts the main query to that kind alone.
- **Order-ID text field** — a free-text input that filters by the `order_id` column, enabling inspection of all events associated with a single order.

Results are fetched from `GET /debug/events` in pages of 50 rows, ordered by `seq DESC` (newest first). A total-count figure is displayed, and Previous / Next pagination controls appear whenever the total exceeds one page. Resetting either filter resets the page index to zero to avoid stale offsets.

Each table row is clickable and expands inline to reveal the three JSON blobs (`entity_refs`, `payload`, `provenance`) rendered as pretty-printed `JsonBlock` components. Clicking the same row again collapses the detail panel (toggle behaviour; only one row can be expanded at a time).

Color-coded `Badge` labels communicate event-kind semantics at a glance:

| Prefix | Badge color |
|---|---|
| `ingest*` | blue |
| `inference*` | purple |
| `decision*` | yellow |
| `outcome*` | green |
| `identity*` / `belief*` | zinc (neutral) |
| (other) | zinc (neutral) |

## Implementation — how it works

**Component:** `frontend/src/views/Events.tsx` — a single functional React component, `Events()`, with no props.

**State:** Six `useState` hooks manage the view — `events` (the current page of `EventRecord[]`), `kinds` (`EventKindCount[]` for the filter dropdown), `total` (total matching row count), `filter` (selected event kind), `orderFilter` (order-ID substring), and `expanded` (the `seq` of the currently expanded row, or `null`). `page` (zero-based) and the constant `pageSize = 50` drive pagination arithmetic.

**Data fetching:** Two `useEffect` hooks separate concerns.
- A mount-once effect calls `api.eventKinds()` → `GET /debug/events/kinds` to populate the kind selector.
- A second effect, gated on `[filter, orderFilter, page]`, calls `api.events({ kind, order_id, limit, offset })` → `GET /debug/events` and writes both `events` and `total` into state.

Both API functions live in `frontend/src/api.ts` and build query strings before delegating to the shared `get<T>()` fetch helper. The backend handler `list_events` in `placer/api/debug.py` (mounted under the `/debug` router prefix) executes two SQL statements against the `events` table: a `COUNT(*)` for the total and a `SELECT … ORDER BY seq DESC LIMIT %s OFFSET %s` for the page of rows. Optional `WHERE event_kind = %s` and/or `WHERE order_id = %s` clauses are appended when filters are active.

**UI composition:** The view uses shared components exclusively — `Card` as the table container, `Table` / `Td` for the data grid, `Badge` for event-kind labels, `JsonBlock` for the expandable JSON panels, and `EmptyState` for the zero-results case. All components are imported from `frontend/src/components/Table.tsx` and `frontend/src/components/Card.tsx`.

## Availability — is it usable right now

The route `/view/events` is listed in the feature table as the **Events** view (component: `events`) with no guards, meaning no authentication or feature-flag gate is applied at the routing layer.

The two backing API endpoints — `GET /debug/events` (`list_events`) and `GET /debug/events/kinds` (`list_event_kinds`) — are both confirmed present in `placer/api/debug.py` and are referenced directly by `frontend/src/api.ts`. Both are guarded only by the `/debug` router prefix; no API-key middleware is declared on those handlers.

**Functional pre-condition:** The `events` table must exist and be reachable via `placer.db.get_conn()`. If the database is empty, the view renders correctly but shows the `EmptyState` ("No events match the current filters.") rather than any rows. Pagination controls are hidden when `total ≤ 50`.

No changelog entries are configured, so there are no pending intent signals to reconcile against the current code state. The feature appears fully wired end-to-end in the current codebase.
