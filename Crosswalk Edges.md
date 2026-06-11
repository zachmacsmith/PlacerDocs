---
feature: Crosswalk Edges
group: Segments
last_synced: '2026-06-11'
last_commit: 87dd52f08e97ba92e8de49eace545f1073d264af
anchors:
  tables:
  - belief_checkpoints
  - candidates
  - crosswalk_edges
  - events
  - orders
  - orgs
  - quantity_registry
  - segments
  endpoints:
  - GET /beliefs/checkpoints
  - GET /beliefs/quantities
  - GET /crosswalk
  - GET /events
  - GET /events/kinds
  - GET /orders
  - GET /orders/{order_id}
  - GET /orgs
  - GET /segments
  - GET /stats
  types:
  - CrosswalkEdge
  - CrosswalkEdgeKey
  - SegmentId
  - SegmentStatus
  - UnspscFamily
  api_modules:
  - placer.api.debug
  - placer.db
  - placer.identity.store
  files:
  - frontend/src/api.ts
  - frontend/src/views/Beliefs.tsx
  - frontend/src/views/Overview.tsx
  - placer/db.py
  - placer/identity/store.py
  - placer/identity/types.py
  - placer/api/debug.py::list_crosswalk_edges
writes:
- crosswalk_edges
- orgs
- segments
reads:
- belief_checkpoints
- candidates
- crosswalk_edges
- events
- orders
- orgs
- quantity_registry
- segments
---
## Capability — what it can do

The Crosswalk Edges feature exposes a read-only view of the `crosswalk_edges` table, which records the mapping between UNSPSC product families and demand segments. This join table is the structural backbone of Placer's Bayesian placement logic: each edge declares that a given product family is associated with a given segment, and carries a `quantity_refs` JSONB blob that links the edge to one or more belief quantities tracked in the `quantity_registry`.

The `GET /crosswalk` endpoint (`list_crosswalk_edges` in `placer/api/debug.py`) returns a flat list of edges wrapped in an `{ "edges": [...] }` envelope. Each result row includes:
- `family` — the UNSPSC family string (`UnspscFamily` NewType) identifying the product class
- `segment_id` — the canonical demand segment (`SegmentId` NewType)
- `quantity_refs` — a free-form JSON object mapping named references to belief quantity identifiers
- `segment_definition` — the human-readable `definition_text` of the joined segment
- `segment_status` — the current lifecycle status of the segment (`active` or `merged`)

An optional `family` query parameter narrows results to a single UNSPSC family. The `limit` parameter accepts values from 1 to 1000, defaulting to 200. Results are sorted deterministically by `(family, segment_id)`.

The `crosswalk_edges` table is also included in the row-count summary surfaced by `GET /stats`, and its count is rendered in the `Overview` dashboard as a `StatCard` labelled "Crosswalk Edges". Edges are written by `upsert_crosswalk_edge` in `placer/identity/store.py` and are consumed at belief-query time by `get_crosswalk_edges_for_family` in the same module.

## Implementation — how it works

**Handler** (`placer/api/debug.py` → `list_crosswalk_edges`): The handler is mounted on `router` (an `APIRouter` with prefix `/debug`). It acquires a pooled async Postgres connection via `get_conn()` (`placer/db.py`), which draws from an `AsyncConnectionPool` (min 2, max 10 connections) configured from the `DATABASE_URL` environment variable. The query is a two-table inner join:

```sql
SELECT ce.family, ce.segment_id, ce.quantity_refs,
       s.definition_text, s.status
FROM crosswalk_edges ce
JOIN segments s ON s.segment_id = ce.segment_id
WHERE {family filter or TRUE}
ORDER BY ce.family, ce.segment_id
LIMIT %s
```

The join enriches each raw edge with its segment's `definition_text` and `status`, avoiding a round-trip when callers need human-readable context. The `quantity_refs` column is returned as-is from its JSONB storage in Postgres.

**Write path** (`placer/identity/store.py` → `upsert_crosswalk_edge`): Edges are written with an `ON CONFLICT (family, segment_id) DO UPDATE SET quantity_refs = EXCLUDED.quantity_refs` upsert, meaning the composite key `(family, segment_id)` is the primary address of a crosswalk edge — modelled explicitly as the `CrosswalkEdgeKey` frozen Pydantic model in `placer/identity/types.py`. The `quantity_refs` payload is JSON-serialised before insert.

**Read path within the belief system** (`placer/identity/store.py` → `get_crosswalk_edges_for_family`): The belief subsystem queries edges by family only (no join to `segments`), returning `(SegmentId, quantity_refs)` tuples. This is the hot path used during placement scoring; the debug endpoint is a separate, richer read intended for inspection only.

**Frontend** (`frontend/src/views/Beliefs.tsx`): The `Beliefs` view contains a "Crosswalk" sub-tab that calls `api.crosswalk()` (`frontend/src/api.ts`) and renders a table of edges with columns for family, segment ID, definition, a colour-coded status badge (`green` for `active`, `yellow` for `merged`), and quantity refs JSON. The sub-tab is lazy-loaded: the API call fires only when the "Crosswalk" tab is selected, triggered by a `useEffect` dependency on `subTab`. The `CrosswalkEdge` TypeScript interface in `frontend/src/api.ts` mirrors the API response shape exactly. The `api.crosswalk()` call accepts an optional `{ family?: string }` params object, which is forwarded as a `family` query-string parameter. When `quantity_refs` is empty the cell renders `—` rather than `{}`.

## Availability — current code state

**Backend:** The `GET /crosswalk` route is present and fully implemented in `placer/api/debug.py` (`list_crosswalk_edges`). It depends on the `crosswalk_edges` and `segments` tables being present in Postgres and the `DATABASE_URL` environment variable being set. No authentication guards are registered on the route. The connection pool is initialised lazily on first use via `init_pool()` in `placer/db.py`.

**Write path:** `upsert_crosswalk_edge` and `get_crosswalk_edges_for_family` are both present in `placer/identity/store.py`. The upsert uses `ON CONFLICT (family, segment_id)`, so a unique constraint on that composite key must exist in the schema for writes to succeed without error.

**Stats endpoint:** `GET /stats` iterates a hardcoded list of tables that includes `crosswalk_edges`, confirmed in `placer/api/debug.py` (`system_stats`). The count feeds the `Overview` dashboard `StatCard` labelled "Crosswalk Edges", confirmed in `frontend/src/views/Overview.tsx`.

**Frontend:** The "Crosswalk" sub-tab is present in `frontend/src/views/Beliefs.tsx` and the `CrosswalkEdge` TypeScript interface is defined in `frontend/src/api.ts`. The `api.crosswalk()` function supports an optional `family` filter parameter. There is no `limit` parameter exposed in the TypeScript client — only the server-side default of 200 applies from the frontend.

**No changelog entries** were found; there are no pending or in-flight changes to assess against the code state.
