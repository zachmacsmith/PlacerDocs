---
feature: Order Detail
group: Orders
last_synced: '2026-06-11'
last_commit: 87dd52f08e97ba92e8de49eace545f1073d264af
anchors:
  tables:
  - candidates
  - orders
  endpoints:
  - GET /debug/orders/{order_id}
  - GET /orders/{order_id}
  types: []
  api_modules:
  - placer.db
  - placer.events
  files:
  - placer/db.py
  - placer/api/debug.py::get_order
writes: []
reads:
- candidates
- orders
---
## Capability — what it can do

`GET /orders/{order_id}` provides a complete, single-order view composed of three parts returned in one JSON response:

- **Order header** — core scalar fields from the `orders` table: `order_id`, `lifecycle`, `mode`, `total_volume`, `v_remaining`, `deadline`, `created_at`, and `updated_at`.
- **Stage summary** — a `stage_counts` map that aggregates all candidates for the order by pipeline stage (e.g. `contacted`, `resolved`), giving an at-a-glance funnel snapshot without having to count the full candidate list.
- **Candidate roster** — the full list of `candidates` rows linked to the order, each carrying `candidate_id`, `org_id`, `surfaced_by`, `stage`, `stage_reason`, `valuation_snapshot`, `contacted_at`, and `resolved_at`. The list is ordered by `stage` then `candidate_id`.

If the requested `order_id` does not exist in the `orders` table the endpoint returns `{"error": "not found"}` with HTTP 200 (no explicit 404 is raised in the handler).

## Implementation — how it works

The handler `get_order` lives in `placer/api/debug.py` and is registered on the `APIRouter` with prefix `/debug` and tag `debug`. The full resolved path is therefore `GET /debug/orders/{order_id}`.

**Database access** is handled through `placer/db.get_conn()`, an async context manager that vends a connection from a `psycopg_pool.AsyncConnectionPool` (min 2, max 10 connections) initialised from the `DATABASE_URL` environment variable. The pool is created lazily on the first call to `init_pool()` with `open=False` then immediately awaited open; subsequent calls reuse the same global instance. All queries are raw parameterised SQL; there is no ORM layer.

**Query plan** (two sequential round-trips per request):
1. A point-lookup on `orders` by primary key returns the single order row.
2. A second query fetches all `candidates` rows for that `order_id`, ordered by `(stage, candidate_id)`.

**Stage counts** are computed in Python by iterating the candidate result set and accumulating a `dict[str, int]` keyed on the `stage` column — no extra database aggregation query is made.

All timestamp columns (`deadline`, `created_at`, `updated_at`, `contacted_at`, `resolved_at`) are serialised to ISO-8601 strings via `.isoformat()`, with `None` passed through as JSON `null`.

The endpoint carries no authentication guards; it is part of the read-only debug surface intended for dashboard/operator use.

## Availability — is it usable right now

The handler `get_order` is confirmed present in `placer/api/debug.py` and is wired to `GET /debug/orders/{order_id}` via the `APIRouter` (prefix `/debug`, tag `debug`). No authentication guards or feature flags are declared on this route or anywhere in the file.

**Runtime prerequisites** that must be satisfied for the endpoint to respond successfully:
- `DATABASE_URL` must be set in the environment and point to a reachable Postgres instance. If absent, `get_database_url()` in `placer/db.py` raises `RuntimeError("DATABASE_URL not set")` on the first request; this is unhandled and will produce a 500-class response.
- The pool is opened lazily on the first call to `init_pool()`; there is no application-startup health check visible in this file, so a misconfigured `DATABASE_URL` will not surface until the first request arrives.
- The `orders` and `candidates` tables must exist in the target database. A missing table will produce a database-level error rather than a handled application response.

No changelog is configured, so no in-flight intent context is available. No staged rollout, guard, or flag was found in the source. The endpoint is reachable in any environment where the application is deployed and the above prerequisites are met.
