---
feature: Belief Checkpoints
group: Beliefs
last_synced: '2026-06-11'
last_commit: 87dd52f08e97ba92e8de49eace545f1073d264af
anchors:
  tables:
  - belief_checkpoints
  endpoints:
  - GET /beliefs/checkpoints
  types: []
  api_modules:
  - placer.db
  - placer.events
  files:
  - placer/db.py
  - placer/api/debug.py::list_checkpoints
writes: []
reads:
- belief_checkpoints
---
## Capability — what it can do

`GET /beliefs/checkpoints` exposes a paginated, read-only view of the `belief_checkpoints` table — the persistent snapshots that record the Bayesian posterior state computed for every tracked quantity at a point in time.

Each checkpoint record surfaces:

- **Identity & lineage** — `id`, `quantity_id`, `index_key`, `selector_version`, `fold_version`
- **Posterior content** — `sufficient_stats` (the serialised sufficient-statistic vector the controller feeds back into the Bayesian update loop)
- **Convergence signal** — `n_eff`, the effective sample size, which indicates how well-determined the posterior is
- **Replay anchor** — `log_watermark`, the event-log sequence number up to which this checkpoint was computed, enabling incremental recomputation
- **Provenance** — `computed_at` timestamp (ISO-8601)

An optional `quantity_id` query parameter narrows results to checkpoints belonging to a single registered quantity. The `limit` parameter (1–500, default 100) controls page size. Results are ordered newest-first by `computed_at`.

## Implementation — how it works

The handler `list_checkpoints` is defined in `placer/api/debug.py` and registered on a FastAPI `APIRouter` with prefix `/debug` and tag `debug`. The route therefore resolves to `GET /debug/beliefs/checkpoints` under the router mount, exposed here as `GET /beliefs/checkpoints` per the feature table.

On each request the handler:

1. Opens a pooled async Postgres connection via `placer.db.get_conn`, which draws from a `psycopg_pool.AsyncConnectionPool` initialised from the `DATABASE_URL` environment variable. The pool is configured with `min_size=2` and `max_size=10` and is lazily opened on first use by `init_pool`.
2. Builds a dynamic `WHERE` clause: if `quantity_id` is supplied it appends `quantity_id = %s`; otherwise the clause degenerates to `TRUE`.
3. Executes a single parameterised `SELECT` against `belief_checkpoints`, ordering by `computed_at DESC` and applying the `LIMIT` bound.
4. Serialises each row into a plain dict, converting the `computed_at` timestamp to ISO-8601 if present, and returns a top-level `{ "checkpoints": [...] }` JSON envelope.

No joins are performed — the handler reads `belief_checkpoints` in isolation. The sibling handler `list_quantities` (`GET /beliefs/quantities`) performs a `LEFT JOIN belief_checkpoints` to aggregate `checkpoint_count` and `avg_n_eff` per quantity, providing a complementary summary view.

The endpoint carries no authentication guard (confirmed: guards `[]` in the feature table). All handlers in `debug.py` are read-only; the module docstring explicitly describes the file as "Read-only views over the spine, beliefs, candidates, orders, and quantity registry."

## Availability — is it usable right now

The handler `list_checkpoints` is present and complete in `placer/api/debug.py`. The route is registered with no authentication guard, so any caller with network access to the API process can invoke it.

**Runtime prerequisites:**
- The `DATABASE_URL` environment variable must be set and reachable; its absence causes `placer.db.get_database_url` to raise a `RuntimeError` before any connection is attempted.
- The connection pool (`psycopg_pool.AsyncConnectionPool`, min 2 / max 10) is initialised lazily on the first request; a misconfigured or unreachable database will surface as a pool-open error at that point.
- The `belief_checkpoints` and `quantity_registry` tables must exist in the target Postgres schema. The endpoint will fail with a database error if migrations have not been applied.

**Data availability:** The endpoint returns an empty `checkpoints` list if no belief checkpoints have been computed yet — the Bayesian learning pipeline must have run at least one update cycle for meaningful data to appear.

No changelog entries are configured, so there is no external intent signal to reconcile against. The feature is available based solely on confirmed code presence.
