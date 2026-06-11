---
feature: List Orders
group: Orders
last_synced: '2026-06-11'
last_commit: 87dd52f08e97ba92e8de49eace545f1073d264af
anchors:
  tables:
  - candidates
  - orders
  endpoints:
  - GET /debug/orders
  - GET /orders
  types: []
  api_modules:
  - placer.db
  - placer.events
  files:
  - placer/db.py
  - placer/api/debug.py::list_orders
writes: []
reads:
- candidates
- orders
---
## Capability — what it can do

`GET /orders` returns a paginated snapshot of placement orders held in the `orders` table, enriched with per-order candidate pipeline counts drawn from the `candidates` table.

Each record in the response carries:

- **Identity & lifecycle** — `order_id`, `lifecycle` state (e.g. open, closed), and `mode` (the placement strategy in use).
- **Volume accounting** — `total_volume` (the original donation target) and `v_remaining` (unallocated volume still to be placed).
- **Timing** — `deadline`, `created_at`, and `updated_at`, all serialised as ISO-8601 strings.
- **Candidate pipeline summary** — `candidate_count` (all candidates linked to the order), `contacted_count` (candidates whose stage is `contacted`), and `resolved_count` (candidates whose stage is `resolved`). These are computed in-query via filtered aggregation against the `candidates` table, so no secondary request is required.

The `limit` query parameter (1–200, default 50) controls how many orders are returned. Results are ordered by `created_at DESC`, so the most recently created orders appear first.

## Implementation — how it works

The handler `list_orders` lives in `placer/api/debug.py` and is registered on a FastAPI `APIRouter` with the prefix `/debug` and tag `debug`. The effective mounted path exposed to clients is `GET /debug/orders`.

**Database access** is performed through `placer.db.get_conn`, an async context manager that lazily initialises a `psycopg_pool.AsyncConnectionPool` (min 2, max 10 connections) backed by the `DATABASE_URL` environment variable via `placer.db.init_pool`. The pool is module-level and shared across all requests; `placer.db.close_pool` tears it down on shutdown.

**Query** — a single SQL statement performs a `LEFT JOIN` between `orders` and `candidates` on `order_id`, groups by `orders.order_id`, and uses two `FILTER (WHERE ...)` clauses on `COUNT` to compute `contacted_count` and `resolved_count` in one round-trip. Row columns are addressed positionally (`r[0]`–`r[10]`) and mapped to a plain `dict` before returning.

**Response shape** — the handler returns `dict[str, Any]` with a single top-level key `"orders"` whose value is the list of order dicts. FastAPI serialises this directly to JSON; no Pydantic response model is declared, so no schema validation is applied to the output.

**No guards** are declared on this route; it carries no API-key or authentication requirement as defined in the feature table and the router registration.

## Availability — is it usable right now

The handler `list_orders` is **present in source** (`placer/api/debug.py`) and registered on the `/debug` router at `GET /debug/orders`. The feature table confirms `GET /orders` maps to this component with no guards. Assuming the router is mounted into the main FastAPI application and runtime preconditions are met, the endpoint is operational.

**Preconditions for a successful response:**
- `DATABASE_URL` must be set in the environment; its absence causes `placer.db.get_database_url` to raise `RuntimeError` at connection time.
- The `orders` and `candidates` tables must exist in the target database.
- The async connection pool must have been opened (via `placer.db.init_pool`); `get_conn` calls this lazily on first use.

**Access control:** No guards are applied (confirmed in both the feature table and router registration). The endpoint is unauthenticated by design, consistent with its `debug` tag. It should be treated as an internal/operator-facing surface and not exposed publicly without additional network-layer protection.

**Changelog:** No changelog is configured; there are no declared in-progress, unreleased, or removed behaviours to flag.
