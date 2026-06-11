---
feature: Event Kinds
group: Events
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
  - GET /debug/beliefs/checkpoints
  - GET /debug/beliefs/quantities
  - GET /debug/crosswalk
  - GET /debug/events
  - GET /debug/events/kinds
  - GET /debug/orders
  - GET /debug/orders/{order_id}
  - GET /debug/orgs
  - GET /debug/segments
  - GET /debug/stats
  - GET /events
  - GET /events/kinds
  - GET /orders
  - GET /orders/{order_id}
  - GET /orgs
  - GET /segments
  - GET /stats
  types:
  - Event
  - EventKind
  - EventSeq
  api_modules:
  - placer.api.debug
  - placer.api.server
  - placer.db
  - placer.events.types
  files:
  - placer/db.py
  - placer/events/types.py
  - placer/api/debug.py::list_event_kinds
writes: []
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

`GET /debug/events/kinds` returns a frequency census of every distinct `event_kind` value that exists in the `events` table. The response is a JSON object with a single `kinds` array; each element carries two fields: `kind` (the string value of the kind, e.g. `"ingest.order"` or `"outcome.approval"`) and `count` (the integer number of rows bearing that kind). Results are sorted alphabetically by kind string.

The endpoint provides a quick, zero-argument way to discover which categories of system activity have actually been recorded in a given deployment. The full taxonomy of valid kinds is defined by the `EventKind` `StrEnum` in `placer/events/types.py`, which currently enumerates **26 values** across eight namespaces: `ingest` (4), `inference` (4), `retrieval` (1), `decision` (6), `outcome` (5), `identity` (3), `belief` (1), `lifecycle` (1), and `system` (1). The endpoint surfaces only kinds that have at least one row; kinds with no rows are absent from the response.

## Implementation — how it works

The handler `list_event_kinds` lives in `placer/api/debug.py` and is registered on a `fastapi.APIRouter` with `prefix="/debug"` and `tag="debug"`. It is mounted on the main `FastAPI` application in `placer/api/server.py` via `app.include_router(debug_router)`, producing the effective path `GET /debug/events/kinds`.

On each request the handler acquires an async Postgres connection from the shared pool managed by `placer/db.py` (backed by `psycopg` + `psycopg_pool`, sized 2–10 connections, configured via the `DATABASE_URL` environment variable) and executes a single aggregation query:

```sql
SELECT event_kind, COUNT(*)
FROM events
GROUP BY event_kind
ORDER BY event_kind
```

All rows are fetched and serialised inline into the `{"kinds": [...]}` response dict. There is no pagination, caching, or streaming; the full result is materialised in memory before the response is returned.

No authentication guard is applied to this router or this endpoint — `app.include_router` is called without a dependency, and no `Header`-based API-key check (as used on `POST /recommendations` and `POST /context-analysis`) is present. CORS is governed by the application-level `CORSMiddleware` configured in `placer/api/server.py`.

The `event_kind` column in the `events` table stores the `.value` of an `EventKind` enum member (e.g. the string `"decision.valuation_snapshot"`), written by `placer/events/store.py` at append time.

## Availability — is it usable right now

The route `GET /debug/events/kinds` is present in the codebase and wired into the running application with no feature flag or guard. It is accessible to any client that can reach the server — **no API key is required**. This makes it a fully open introspection endpoint; operators should be aware that it exposes event-spine activity metadata to unauthenticated callers.

The response is only meaningful once the `events` table has been populated; on a fresh or empty database the handler returns `{"kinds": []}` without error.

There is no changelog configured for this project, so no announced changes or deprecations are pending.
