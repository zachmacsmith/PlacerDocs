---
feature: List Segments
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
  - GET /health
  - GET /orders
  - GET /orders/{order_id}
  - GET /orgs
  - GET /segments
  - GET /stats
  - POST /context-analysis
  - POST /recommendations
  types: []
  api_modules:
  - placer.api.debug
  - placer.api.server
  - placer.db
  files:
  - placer/db.py
  - placer/api/debug.py::list_segments
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

`GET /segments` returns an optionally filtered, top-N snapshot of all segment records stored in the `segments` table. Each result item surfaces six fields:

| Field | Description |
|---|---|
| `segment_id` | Stable identifier for the segment |
| `version` | Schema or definition version of the segment |
| `definition_text` | Human-readable or machine-readable segment definition |
| `status` | Lifecycle status of the segment (e.g. active, deprecated) |
| `lineage` | Provenance chain describing how the segment was derived |
| `created_at` | ISO-8601 timestamp of segment creation |

The endpoint accepts two query parameters:
- **`status`** *(optional string)* — filters results to segments whose `status` column matches the given value exactly.
- **`limit`** *(optional integer, 1–500, default 100)* — caps the number of rows returned.

Results are ordered newest-first by `created_at`. No authentication guard is declared on this route.

## Implementation — how it works

The handler `list_segments` lives in `placer/api/debug.py`, registered on a FastAPI `APIRouter` with the prefix `/debug` and tag `debug`. The full mounted path in the application is `/debug/segments` — confirmed via `app.include_router(debug_router)` in `placer/api/server.py` with no additional prefix.

**Database access** is performed via `placer/db.py`'s `get_conn()` async context manager, which draws a connection from a `psycopg_pool.AsyncConnectionPool` backed by the `DATABASE_URL` environment variable. The pool is initialised lazily on first use (min 2, max 10 connections) and is explicitly opened during the FastAPI `lifespan` startup hook in `server.py`.

**Query construction** uses a lightweight dynamic `WHERE` clause: if `status` is provided it appends `status = %s` to the condition list; otherwise the predicate defaults to `TRUE`. A single parameterised SQL statement then reads from `segments` ordered by `created_at DESC LIMIT %s`. There is no `OFFSET` parameter — the endpoint is effectively a "top-N by recency" view rather than a fully cursor-paginated one.

The handler is part of the broader `debug` router that also exposes read-only views over `events`, `orders`, `beliefs`, `crosswalk_edges`, and `orgs`. All routes in this module are described as **read-only views over the spine** and carry no write side-effects.

## Availability — is it usable right now

The handler `list_segments` is present in `placer/api/debug.py` and is confirmed mounted in the live application: `placer/api/server.py` imports the `debug` router and calls `app.include_router(debug_router)` unconditionally. The full path `GET /debug/segments` is therefore reachable whenever the server is running.

No authentication or authorisation guards are applied to this route (unlike `POST /recommendations` and `POST /context-analysis`, which require an `X-API-Key` header).

**Runtime prerequisites that must be satisfied before the endpoint is functional:**
- `DATABASE_URL` must be set in the environment; `get_database_url()` in `placer/db.py` raises `RuntimeError` at pool-init time if absent.
- The pool is opened eagerly during the FastAPI `lifespan` startup hook, so a missing or unreachable database will prevent the server from starting rather than failing at request time.
- The `segments` table must exist and be reachable in the target Postgres instance.

No changelog entries exist and no feature flags or conditional mounts are present in the source. Assuming `DATABASE_URL` is set and the database schema is applied, the endpoint is functional.
