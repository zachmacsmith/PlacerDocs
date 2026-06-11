---
feature: List Orders
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
  - GET /debug/orders
  - GET /events
  - GET /events/kinds
  - GET /orders
  - GET /orders/{order_id}
  - GET /orgs
  - GET /segments
  - GET /stats
  types: []
  api_modules:
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

`GET /orders` returns a paginated snapshot of placement orders held in the `orders` table, enriched with per-order candidate pipeline counts drawn from the `candidates` table.

Each record in the response carries:

- **Identity & lifecycle** — `order_id`, `lifecycle` state (e.g. open, closed), and `mode` (the placement strategy in use).
- **Volume accounting** — `total_volume` (the original donation target) and `v_remaining` (unallocated volume still to be placed).
- **Timing** — `deadline`, `created_at`, and `updated_at`, all serialised as ISO-8601 strings.
- **Candidate pipeline summary** — `candidate_count` (all candidates linked to the order), `contacted_count` (candidates whose stage is `contacted`), and `resolved_count` (candidates whose stage is `resolved`). These are computed in-query via filtered aggregation against the `candidates` table, so no secondary request is required.

The `limit` query parameter (1–200, default 50) controls how many orders are returned. Results are ordered by `created_at DESC`, so the most recently created orders appear first.

## Implementation — how it works

The handler `list_orders` lives in `placer/api/debug.py` and is registered on a FastAPI `APIRouter` with the prefix `/debug` and tag `debug`. The effective mounted path exposed to clients is `GET /debug/orders`.

**Database access** is performed through `placer.db.get_conn`, an async context manager that lazily initialises a `psycopg_pool.AsyncConnectionPool` (min 2, max 10 connections) backed by the `DATABASE_URL` environment variable. The pool is module-level and shared across all requests.

**Query** — a single SQL statement performs a `LEFT JOIN` between `orders` and `candidates` on `order_id`, groups by `orders.order_id`, and uses two `FILTER (WHERE ...)` clauses on `COUNT` to compute `contacted_count` and `resolved_count` in one round-trip. Row columns are addressed positionally (`r[0]`–`r[10]`) and mapped to a plain `dict` before returning.

**Response shape** — the handler returns `dict[str, Any]` with a single top-level key `"orders"` whose value is the list of order dicts. FastAPI serialises this directly to JSON; no Pydantic response model is declared, so no schema validation is applied to the output.

**No guards** are declared on this route; it carries no API-key or authentication requirement as defined in the feature table and the router registration.

## Availability — is it usable right now

The handler is **present in source** (`placer/api/debug.py`, `list_orders` function) and the route is registered on the `/debug` router. Assuming the router is mounted into the main FastAPI application and the `DATABASE_URL` environment variable is set and reachable, the endpoint is operational.

**Preconditions for a successful response:**
- `DATABASE_URL` must be configured; its absence raises `RuntimeError` at connection time.
- The `orders` and `candidates` tables must exist in the target database.

**Access control:** No guards are applied. The endpoint is unauthenticated by design, consistent with its placement in the `debug` tag. It should be treated as an internal/operator-facing surface and not exposed publicly without additional network-layer protection.

No changelog entries exist, so there are no declared in-progress or removed behaviours to flag.
