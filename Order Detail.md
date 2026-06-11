---
feature: Order Detail
group: Orders
last_synced: '2026-06-11'
last_commit: 5499bc5a8c1f45be4e6cdc23b3f7414d926340f0
anchors:
  tables:
  - belief_checkpoints
  - candidates
  - events
  - orders
  - segments
  endpoints:
  - GET /beliefs/checkpoints
  - GET /beliefs/quantities
  - GET /crosswalk
  - GET /debug/orders/{order_id}
  - GET /events
  - GET /events/kinds
  - GET /orders
  - GET /orders/{order_id}
  - GET /orgs
  - GET /segments
  - GET /stats
  types: []
  api_modules:
  - placer.api.debug
  - placer.db
  files:
  - placer/api/**
  - placer/api/debug.py
  - placer/db.py
writes: []
reads:
- belief_checkpoints
- candidates
- events
- orders
- segments
---
## Capability — what it can do

`GET /orders/{order_id}` provides a complete, single-order view composed of three parts returned in one JSON response:

- **Order header** — core scalar fields from the `orders` table: `order_id`, `lifecycle`, `mode`, `total_volume`, `v_remaining`, `deadline`, `created_at`, and `updated_at`.
- **Stage summary** — a `stage_counts` map that aggregates all candidates for the order by pipeline stage (e.g. `contacted`, `resolved`), giving an at-a-glance funnel snapshot without having to count the full candidate list.
- **Candidate roster** — the full list of `candidates` rows linked to the order, each carrying `candidate_id`, `org_id`, `surfaced_by`, `stage`, `stage_reason`, `valuation_snapshot`, `contacted_at`, and `resolved_at`. The list is ordered by `stage` then `candidate_id`.

If the requested `order_id` does not exist in the `orders` table the endpoint returns `{"error": "not found"}` with HTTP 200 (no explicit 404 is raised in the handler).

## Implementation — how it works

The handler `get_order` lives in `placer/api/debug.py` and is registered on the `APIRouter` with prefix `/debug` and tag `debug`. The full resolved path is therefore `GET /debug/orders/{order_id}`.

**Database access** is handled through `placer/db.get_conn()`, an async context manager that vends a connection from a `psycopg_pool.AsyncConnectionPool` (min 2, max 10 connections) initialised from the `DATABASE_URL` environment variable. All queries are raw parameterised SQL; there is no ORM layer.

**Query plan** (two sequential round-trips per request):
1. A point-lookup on `orders` by primary key returns the single order row.
2. A second query fetches all `candidates` rows for that `order_id`, ordered by `(stage, candidate_id)`.

**Stage counts** are computed in Python by iterating the candidate result set and accumulating a `dict[str, int]` keyed on the `stage` column — no extra database aggregation query is made.

All timestamp columns (`deadline`, `created_at`, `updated_at`, `contacted_at`, `resolved_at`) are serialised to ISO-8601 strings via `.isoformat()`, with `None` passed through as JSON `null`.

The endpoint carries no authentication guards; it is part of the read-only debug surface intended for dashboard/operator use.

## Availability — is it usable right now

The handler `get_order` is confirmed present in `placer/api/debug.py` and is wired to `GET /debug/orders/{order_id}` via the `APIRouter`. No authentication guards are declared on this route.

**Runtime prerequisites that must be satisfied for the endpoint to respond successfully:**
- The `DATABASE_URL` environment variable must be set and point to a reachable Postgres instance; if absent, `get_conn()` raises `RuntimeError("DATABASE_URL not set")` on the first request.
- The `orders` and `candidates` tables must exist in the target database; missing tables will produce a database-level error rather than a handled response.

No feature flags, staged rollouts, or guards were found in the source. There is no changelog configured, so no in-flight intent context is available. The endpoint should be reachable in any environment where the application is deployed and the database is accessible.
