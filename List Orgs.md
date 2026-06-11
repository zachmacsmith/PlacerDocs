---
feature: List Orgs
group: Organizations
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
  - segments
  endpoints:
  - GET /beliefs/checkpoints
  - GET /beliefs/quantities
  - GET /crosswalk
  - GET /debug/orgs
  - GET /events
  - GET /events/kinds
  - GET /orders
  - GET /orders/{order_id}
  - GET /orgs
  - GET /segments
  - GET /stats
  types:
  - Ein
  - EventSeq
  - OrgId
  - OrgMemberships
  - OrgRecord
  - OrgRole
  - ResolutionConfidence
  - ResolutionMethod
  - SegmentMembership
  - SizeBand
  api_modules:
  - frontend/src/api.ts
  - placer.db
  - placer/api/debug.py
  files:
  - frontend/src/api.ts
  - frontend/src/views/Candidates.tsx
  - placer/db.py
  - placer/identity/store.py
  - placer/identity/types.py
  - placer/api/debug.py::list_orgs
writes: []
reads:
- belief_checkpoints
- candidates
- crosswalk_edges
- events
- frontend/src/api.ts
- frontend/src/views/Candidates.tsx
- orders
- orgs
- placer/api/debug.py
- placer/db.py
- placer/identity/store.py
- placer/identity/types.py
- segments
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

**Handler** — `list_orgs` in `placer/api/debug.py`, mounted on the `APIRouter` at prefix `/debug` with tag `debug`. The effective path is therefore `GET /debug/orgs`.

**Database access** — A single `SELECT org_id, ein, resolution_confidence, first_seen_seq, memberships FROM orgs ORDER BY first_seen_seq DESC LIMIT %s` is issued against the `orgs` table. The connection is acquired from the shared async `psycopg_pool.AsyncConnectionPool` managed by `placer/db.py`, which is initialised lazily from the `DATABASE_URL` environment variable with `min_size=2` and `max_size=10` connections. The pool is opened on first use via `init_pool()` and can be torn down via `close_pool()`; the `get_conn()` async context manager wraps both steps for callers.

**Serialisation** — Each database row is projected into a plain Python dict. `memberships` is returned as-is (the driver deserialises the Postgres JSONB column automatically); no Pydantic model is used at the API layer. The response envelope is `{"orgs": [...]}`.

**Identity model** — Org rows are originally written by `placer/identity/store.py::mint_org`, which inserts with `ON CONFLICT (org_id) DO NOTHING`. `resolve_org` performs the EIN lookup that precedes minting; a separate `get_org` function enables point-lookup by `org_id` at the store layer. The `OrgRecord`, `OrgMemberships`, `ResolutionConfidence`, and related types are defined in `placer/identity/types.py`. `ResolutionMethod` enumerates four strategies: `ein_join`, `name_address_block`, `fuzzy_match`, and `provisional`; only `ein_join` is currently wired in `resolve_org`.

**Frontend consumption** — The TypeScript client in `frontend/src/api.ts` calls `api.orgs()` → `GET /debug/orgs` and types the response with the `OrgRecord` interface. The result is rendered in the **Org Registry** sub-tab of `frontend/src/views/Candidates.tsx`, which lazy-loads on tab activation (via a `useEffect` gated on `subTab === "orgs"`) and displays org ID, EIN, a colour-coded confidence badge (`high` → green, `medium` → yellow, `provisional` → zinc), first-seen sequence, and a truncated JSON rendering of memberships.

## Availability — is it usable right now

The handler `list_orgs` is present and complete in `placer/api/debug.py`. It carries no authentication guards and is part of the `debug` router, meaning it is available to any caller that can reach the API server — no API key or session token is required.

Operability depends on two runtime conditions confirmed in source:
- **`DATABASE_URL` environment variable** must be set and point to a reachable Postgres instance. `placer/db.py::get_database_url()` raises `RuntimeError("DATABASE_URL not set")` before any connection attempt if the variable is absent; this bubbles as an unhandled 500 on the first request.
- **`orgs` table** must exist in the target database with the columns `org_id`, `ein`, `resolution_confidence`, `first_seen_seq`, and `memberships`. No migration is run automatically at startup.

The `GET /debug/orgs` route is confirmed mounted at that path; the feature table also records a bare `GET /orgs` route identity for this feature — both refer to the same handler. No changelog is configured, so there is no record of staged or in-progress changes to this endpoint. The frontend `Candidates.tsx` view actively consumes this endpoint via `api.orgs()` in the Org Registry sub-tab, confirming end-to-end wiring in the current codebase.
