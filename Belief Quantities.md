---
feature: Belief Quantities
group: Beliefs
last_synced: '2026-06-11'
last_commit: 5499bc5a8c1f45be4e6cdc23b3f7414d926340f0
anchors:
  tables:
  - belief_checkpoints
  - candidates
  - events
  - quantity_registry
  - segments
  endpoints:
  - GET /beliefs/checkpoints
  - GET /beliefs/quantities
  - GET /crosswalk
  - GET /debug/beliefs/quantities
  - GET /events
  - GET /events/kinds
  - GET /orders
  - GET /orders/{order_id}
  - GET /orgs
  - GET /segments
  - GET /stats
  types:
  - QuantityRecord
  api_modules:
  - placer.api.debug
  - placer.beliefs.store
  - placer.db
  files:
  - frontend/src/api.ts
  - placer/api/**
  - placer/api/debug.py
  - placer/api/server.py
  - placer/beliefs/store.py
  - placer/db.py
writes: []
reads:
- belief_checkpoints
- candidates
- events
- quantity_registry
- segments
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

The response shape is `{ "quantities": [ QuantityRecord, … ] }` where `QuantityRecord` is defined in `frontend/src/api.ts`. The endpoint takes no query parameters; it always returns the full registry ordered by `quantity_id`.

## Implementation — how it works

The handler `list_quantities` in `placer/api/debug.py` is registered on the `APIRouter` with `prefix="/debug"` and included into the main FastAPI application in `placer/api/server.py` via `app.include_router(debug_router)`. The effective mounted path is therefore `/debug/beliefs/quantities`.

**Database query.** The handler opens a connection from the shared async pool (`placer/db.py`, backed by `psycopg_pool.AsyncConnectionPool`, configured from the `DATABASE_URL` environment variable) and executes a single SQL query:

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

**Related tables.** `quantity_registry` is the authoritative catalogue of estimands. `belief_checkpoints` holds the conjugate sufficient-statistics snapshots that `placer/beliefs/store.py` writes via `save_checkpoint`. The `n_eff` column on a checkpoint is the effective sample size computed at fold time and stored alongside the sufficient statistics (e.g. `BetaStats`, `NormalStats`, `ScalarStats`).

**No authentication guard.** The debug router carries no API-key middleware; the endpoint is unauthenticated and is intended for internal dashboard and operator use only. The frontend TypeScript client (`frontend/src/api.ts`, `api.quantities()`) calls this path directly.

## Availability — is it usable right now

The route `GET /debug/beliefs/quantities` is **present and active** in the current codebase. The handler `list_quantities` exists in `placer/api/debug.py` and the router is unconditionally mounted in `placer/api/server.py` with no feature flag or guard.

**Runtime prerequisites for meaningful results:**
- `DATABASE_URL` environment variable must be set; if absent, `placer/db.py` raises `RuntimeError` before any query is executed.
- The `quantity_registry` and `belief_checkpoints` tables must exist in the database. No migration helper is visible in the examined source; their presence depends on schema provisioning outside this file.
- The endpoint will return `{ "quantities": [] }` (an empty list) if the `quantity_registry` table is empty, which is the state of a freshly seeded environment before any belief-system quantities have been registered.

**No changelog entries** are available to indicate planned changes or deprecation. The endpoint carries no API-key requirement, so there is no credential barrier to access, but this also means it should not be exposed on a public network interface without an external authentication layer.
