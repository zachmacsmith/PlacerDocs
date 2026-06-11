---
feature: System Stats
group: System
last_synced: '2026-06-11'
last_commit: 5499bc5a8c1f45be4e6cdc23b3f7414d926340f0
anchors:
  tables:
  - belief_checkpoints
  - candidates
  - crosswalk_edges
  - events
  - orders
  - orgs
  - segments
  endpoints:
  - GET /beliefs/checkpoints
  - GET /beliefs/quantities
  - GET /crosswalk
  - GET /debug/stats
  - GET /events
  - GET /events/kinds
  - GET /orders
  - GET /orders/{order_id}
  - GET /orgs
  - GET /segments
  - GET /stats
  types:
  - dict[str, Any]
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
- crosswalk_edges
- events
- orders
- orgs
- segments
---
## Capability — what it can do

`GET /stats` returns a single, consolidated snapshot of the entire Placer database in one call. It reports two things:

1. **Row counts for every core table** — `events`, `orgs`, `segments`, `crosswalk_edges`, `candidates`, `orders`, and `belief_checkpoints` — surfaced as a `table_counts` map.
2. **Top-10 event-kind distribution** — a ranked list of `{kind, count}` pairs drawn from the `events` table, giving an at-a-glance read on system activity mix.

The endpoint is intentionally read-only and requires no authentication (no guards). It is part of the `debug` router (`prefix=/debug`, tag `debug`), meaning the fully-qualified path is `GET /debug/stats`.

## Implementation — how it works

The handler `system_stats()` lives in `placer/api/debug.py` and is registered on a FastAPI `APIRouter` with `prefix="/debug"`.

**Database access** is mediated by `placer/db.py`'s `get_conn()` async context manager, which leases a connection from a module-level `psycopg_pool.AsyncConnectionPool` (2–10 connections, configured via the `DATABASE_URL` environment variable). The pool is initialised lazily on first use.

**Count loop** — the handler iterates a hard-coded list of seven table names and issues a bare `SELECT COUNT(*) FROM <table>` for each, accumulating results into a `counts` dict. The `# noqa: S608` pragma on the f-string query acknowledges the dynamic SQL but the table names are literal constants, not user input.

**Event-kind aggregation** — a single follow-up query groups the `events` table by `event_kind`, orders by count descending, and returns at most 10 rows.

Both query results are assembled into a JSON-serialisable `dict[str, Any]` and returned directly by FastAPI without any intermediate schema/model validation layer.

## Availability — is it usable right now

The handler `system_stats` is present in `placer/api/debug.py` and is wired to `GET /debug/stats` on the debug router. The route carries no guards (no API-key or auth requirement), matching the feature table entry.

**Prerequisites for a successful response:**
- The `DATABASE_URL` environment variable must be set to a reachable Postgres instance; the pool will raise `RuntimeError` at connection time if it is absent.
- All seven counted tables (`events`, `orgs`, `segments`, `crosswalk_edges`, `candidates`, `orders`, `belief_checkpoints`) must exist in the target database; the handler does not perform existence checks and will raise a database error if any table is missing.

No changelog entries exist, so there are no declared in-progress or planned changes to assess. Based solely on code state, the endpoint is available in the current build to any caller that can reach the service network and satisfies the database prerequisites above.
