---
feature: List Orgs
group: Organizations
first_commit: 5499bc5a8c1f45be4e6cdc23b3f7414d926340f0
last_synced: '2026-06-15'
last_commit: 6dc428c8cfbf577dc8254a42c8b1873db3babcd4
anchors:
  tables:
  - orgs
  endpoints:
  - GET /orgs
  types: []
  api_modules:
  - placer.db
  files:
  - placer/api/debug.py::list_orgs
writes: []
reads:
- orgs
- placer/api/debug.py
---
## Capability — what it can do

`GET /orgs` exposes a paginated read of the **org registry** — the canonical set of charity organisations that Placer has encountered and resolved. Each row returned represents one `OrgRecord` and carries the following fields:

| Field | Meaning |
|---|---|
| `org_id` | Canonical organisation identifier (`OrgId` NewType wrapping a string) |
| `ein` | Employer Identification Number, nullable; populated only when the org was resolved via EIN join |
| `resolution_confidence` | One of `high`, `medium`, or `provisional` — reflects certainty of the identity resolution that minted the row |
| `first_seen_seq` | Event-log sequence number at which the org was first encountered |
| `memberships` | JSONB blob encoding segment memberships, stream IDs, NTEE code, size band, and org role |

A single query parameter controls page size: `limit` (integer, 1–200, default 50). Results are ordered newest-first by `first_seen_seq`. The endpoint is unauthenticated and carries no guards.

The registry is a **lazy write-once structure**: rows are minted by `identity.store.mint_org` on first encounter and are never deleted through this interface. The debug endpoint provides visibility into that accumulated state without modifying it.

## Implementation — how it works

**Handler** — `list_orgs` in `placer/api/debug.py`, mounted on the `APIRouter` at prefix `/debug` with tag `debug`. The effective path is `GET /debug/orgs` (registered in the feature table as `GET /orgs`).

**Database access** — A single `SELECT org_id, ein, resolution_confidence, first_seen_seq, memberships FROM orgs ORDER BY first_seen_seq DESC LIMIT %s` is issued against the `orgs` table. The connection is acquired from the shared async `psycopg_pool.AsyncConnectionPool` managed by `placer/db.py`, which is initialised lazily from the `DATABASE_URL` environment variable with `min_size=2` and `max_size=10` connections. The pool is opened on first use via `init_pool()` and can be torn down via `close_pool()`; the `get_conn()` async context manager wraps both steps for callers.

**Serialisation** — Each database row is projected into a plain Python dict. `memberships` is returned as-is (the driver deserialises the Postgres JSONB column automatically); no Pydantic model is used at the API layer. The response envelope is `{"orgs": [...]}`.

**Identity model** — Org rows are originally written by `identity.store.mint_org`, which inserts with `ON CONFLICT (org_id) DO NOTHING`. `resolve_org` performs the EIN lookup that precedes minting; a separate `get_org` function enables point-lookup by `org_id` at the store layer. `ResolutionMethod` enumerates four strategies: `ein_join`, `name_address_block`, `fuzzy_match`, and `provisional`; only `ein_join` is currently wired in `resolve_org`.

**Frontend consumption** — No frontend source files are present in the current codebase anchors for this feature; frontend wiring cannot be confirmed from available code.

## Availability — is it usable right now

The handler `list_orgs` is present and complete in `placer/api/debug.py`. It carries no authentication guards and is part of the `debug` router, meaning it is available to any caller that can reach the API server — no API key or session token is required.

Operability depends on two runtime conditions confirmed in source:
- **`DATABASE_URL` environment variable** must be set and point to a reachable Postgres instance. `placer/db.py::get_database_url()` raises `RuntimeError("DATABASE_URL not set")` before any connection attempt if the variable is absent; this bubbles as an unhandled 500 on the first request.
- **`orgs` table** must exist in the target database with the columns `org_id`, `ein`, `resolution_confidence`, `first_seen_seq`, and `memberships`. No migration is run automatically at startup.

The route is mounted at `GET /debug/orgs` (the feature table records it as `GET /orgs`). No changelog is configured, so there is no record of staged or in-progress changes to this endpoint. Frontend consumption of this endpoint cannot be confirmed — no frontend source files are present in the current codebase anchors for this feature.
