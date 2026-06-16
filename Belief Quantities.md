---
feature: Belief Quantities
group: Beliefs
first_commit: 5499bc5a8c1f45be4e6cdc23b3f7414d926340f0
last_synced: '2026-06-15'
last_commit: 6dc428c8cfbf577dc8254a42c8b1873db3babcd4
anchors:
  tables:
  - belief_checkpoints
  - quantity_registry
  endpoints:
  - GET /beliefs/quantities
  types: []
  api_modules:
  - placer.db
  files:
  - placer/api/debug.py::list_quantities
writes: []
reads:
- belief_checkpoints
- placer/api/debug.py
- quantity_registry
---
## Capability — what it can do

`GET /beliefs/quantities` (served at the prefixed path `/debug/beliefs/quantities`) provides a read-only, aggregated catalogue of every **quantity** registered in Placer's belief system. A quantity is a named, versioned estimand — a probabilistic quantity the system tracks beliefs about, such as a donation-response rate for a particular org segment.

For each quantity the endpoint returns:

| Field | Meaning |
|---|---|
| `quantity_id` | Stable string identifier for the estimand |
| `type` | Quantity family (e.g. `beta`, `normal`, `scalar`) |
| `estimand_text` | Human-readable description of what is being estimated |
| `index_schema` | JSON schema describing the key dimensions used to slice the belief space |
| `selector_version` | Version tag of the event-selector logic used to accumulate evidence |
| `fold_version` | Version tag of the conjugate-fold function applied to each event |
| `checkpoint_count` | Number of `belief_checkpoints` rows currently stored for this quantity |
| `avg_n_eff` | Mean effective sample size across all checkpoints, indicating posterior concentration |

The response shape is `{ "quantities": [ QuantityRecord, … ] }`. The endpoint takes no query parameters; it always returns the full registry ordered by `quantity_id`.

## Implementation — how it works

The handler `list_quantities` in `placer/api/debug.py` is registered on the `APIRouter` with `prefix="/debug"` and included into the main FastAPI application in `placer/api/server.py` via `app.include_router(debug_router)`. The effective mounted path is therefore `/debug/beliefs/quantities`.

**Database query.** The handler opens a connection from the shared async pool (module `placer.db`, backed by `psycopg_pool.AsyncConnectionPool`, configured from the `DATABASE_URL` environment variable) and executes a single SQL query:

```sql
SELECT qr.quantity_id, qr.type, qr.estimand_text, qr.index_schema,
       qr.selector_version, qr.fold_version,
       COUNT(bc.id)   AS checkpoint_count,
       AVG(bc.n_eff)  AS avg_n_eff
FROM quantity_registry qr
LEFT JOIN belief_checkpoints bc ON bc.quantity_id = qr.quantity_id
GROUP BY qr.quantity_id
ORDER BY qr.quantity_id
```

The `LEFT JOIN` means quantities with no checkpoints yet are still included, with `checkpoint_count = 0` and `avg_n_eff = 0.0` (the handler coalesces `None` to `0.0` in Python).

**Related tables.** `quantity_registry` is the authoritative catalogue of estimands. `belief_checkpoints` holds the conjugate sufficient-statistics snapshots written by the belief subsystem. The `n_eff` column on a checkpoint is the effective sample size computed at fold time and stored alongside the sufficient statistics (e.g. `BetaStats`, `NormalStats`, `ScalarStats`).

**No authentication guard.** The debug router carries no API-key middleware; the endpoint is unauthenticated and is intended for internal dashboard and operator use only.

## Availability — is it usable right now

The route `GET /debug/beliefs/quantities` (registered as `GET /beliefs/quantities` on the `debug` router with `prefix="/debug"`) is **present and active** in the current codebase. The handler `list_quantities` exists in `placer/api/debug.py` and the `debug_router` is unconditionally mounted in `placer/api/server.py` via `app.include_router(debug_router)` — no feature flag, guard, or conditional import is present.

**Runtime prerequisites for meaningful results:**
- `DATABASE_URL` environment variable must be set; if absent, the `placer.db` module raises `RuntimeError("DATABASE_URL not set")` before any query is executed. The pool is initialised lazily on the first `get_conn()` call (pool min_size=2, max_size=10).
- The `quantity_registry` and `belief_checkpoints` tables must exist in the database. No migration helper is visible in the examined source files; their presence depends on schema provisioning outside this codebase.
- The endpoint returns `{ "quantities": [] }` if `quantity_registry` is empty — the expected state of a freshly provisioned environment before any belief quantities have been registered.

**No authentication barrier.** The debug router applies no `X-API-Key` check (unlike the `/recommendations` and `/context-analysis` routes on the main app). The endpoint must not be exposed on a public network interface without an external authentication or network layer.

**No changelog entries** are available to indicate planned changes or deprecation. All code paths confirmed above match the current source exactly.
