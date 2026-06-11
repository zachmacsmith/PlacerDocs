---
feature: Overview
group: New
last_synced: '2026-06-11'
last_commit: ba02630c316c435e071a627f433a21d08f9987e7
anchors:
  tables:
  - belief_checkpoints
  - candidates
  - crosswalk_edges
  - events
  - orders
  - orgs
  - segments
  endpoints:
  - GET /debug/stats
  - GET /health
  types:
  - SystemStats
  api_modules:
  - api.health
  - api.stats
  files:
  - frontend/src/api.ts
  - frontend/src/components/Card.tsx
  - frontend/src/components/Table.tsx
  - frontend/src/views/**
  - frontend/src/views/Overview.tsx
  - placer/api/debug.py
writes: []
reads:
- frontend/src/api.ts
- frontend/src/components/Card.tsx
- frontend/src/components/Table.tsx
- frontend/src/views/Overview.tsx
- placer/api/debug.py
---
## Capability — what it can do

The Overview view is the top-level health and inventory dashboard for the Placer platform. It provides two things simultaneously:

**API health probe.** The component calls `GET /health` on mount and surfaces the returned `status` string (e.g. `"ok"` or `"unreachable"`) in a prominent stat card, giving operators an immediate signal that the FastAPI process is reachable and the service is alive.

**System-wide inventory counts.** The component calls `GET /debug/stats`, which issues `COUNT(*)` queries against seven core tables — `events`, `orgs`, `segments`, `crosswalk_edges`, `candidates`, `orders`, and `belief_checkpoints` — and returns a `SystemStats` payload. The component renders all eight counts (including API status) as a two-row grid of `StatCard` tiles.

**Top event-kind breakdown.** The same `/debug/stats` response carries `top_event_kinds`, the top-10 event kinds by frequency from the `events` spine. The component renders these as a two-column table (kind, count). When the spine is empty the table is replaced by an `EmptyState` message.

**Error surface.** If the `/debug/stats` fetch fails (e.g. the API is down or the database has not been migrated), the component replaces its content with a `Card` showing the error message and a diagnostic hint about port 8000 and pending migrations.

## Implementation — how it works

**Component:** `frontend/src/views/Overview.tsx` — a single functional React component with no routing children.

**Data fetching.** Two parallel `useEffect` fetches fire on mount via `api.stats()` and `api.health()` (both defined in `frontend/src/api.ts`):
- `api.stats()` → `GET /debug/stats` (served by `system_stats` handler in `placer/api/debug.py`). The handler counts rows across all seven tables in a sequential loop and returns the top-10 event kinds from a single aggregation query.
- `api.health()` → `GET /health` (separate health-check route). The resolved `status` string is held in local React state.

State is managed with three `useState` hooks: `stats: SystemStats | null`, `health: string`, and `error: string | null`. The component renders a loading placeholder until `stats` resolves, or an error card if the fetch rejects.

**`SystemStats` type** (defined in `frontend/src/api.ts`):
```ts
interface SystemStats {
  table_counts: Record<string, number>;
  top_event_kinds: { kind: string; count: number }[];
}
```

**Layout.** The render tree is:
- Two `grid` rows of four `StatCard` components each (using `frontend/src/components/Card.tsx`), covering API status + all seven table counts.
- A `Card` titled "Top Event Kinds" containing a `Table` (from `frontend/src/components/Table.tsx`) with `Td` cells rendered in `mono` style, or an `EmptyState` fallback.

**Backend stats endpoint** (`GET /debug/stats` via `placer/api/debug.py`): iterates over a hard-coded list of seven table names, executes `SELECT COUNT(*)` for each, and runs a single `GROUP BY event_kind … LIMIT 10` aggregation. No caching — counts are live on every page load.

## Availability — current state

The Overview view is present and fully wired. The component file `frontend/src/views/Overview.tsx` exists, the route `/view/overview` is registered in the feature table with no guards, and both backend endpoints it depends on (`GET /debug/stats` and `GET /health`) are confirmed implemented in `placer/api/debug.py` and the feature table respectively.

**No auth guard.** The route carries no guards, so no authentication is required to reach this view.

**Runtime dependency:** The component is only meaningful when the Placer API is running on port 8000 and the database has been migrated. If the API is unreachable, the component self-reports an error card with a diagnostic hint. If the database is reachable but tables are empty, the stat cards will show zeros and the event-kind table will show the `EmptyState` message — both are valid rendered states, not errors.

No changelog is configured, so there is no documented intent to flag against the current code state.
