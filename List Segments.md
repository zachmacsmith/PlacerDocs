---
feature: List Segments
group: Segments
last_synced: '2026-06-11'
last_commit: 5499bc5a8c1f45be4e6cdc23b3f7414d926340f0
anchors:
  tables:
  - belief_checkpoints
  - candidates
  - events
  - segments
  endpoints:
  - GET /beliefs/checkpoints
  - GET /beliefs/quantities
  - GET /crosswalk
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
- segments
---
## Capability — what it can do

`GET /segments` returns a paginated, optionally filtered snapshot of all segment records stored in the `segments` table. Each result item surfaces six fields:

| Field | Description |
|---|---|
| `segment_id` | Stable identifier for the segment |
| `version` | Schema or definition version of the segment |
| `definition_text` | Human-readable or machine-readable segment definition |
| `status` | Lifecycle status of the segment (e.g. active, deprecated) |
| `lineage` | Provenance chain describing how the segment was derived |
| `created_at` | ISO-8601 timestamp of segment creation |

The endpoint accepts two query parameters:
- **`status`** *(optional string)* — filters results to segments whose `status` column matches the given value exactly.
- **`limit`** *(optional integer, 1–500, default 100)* — caps the number of rows returned.

Results are ordered newest-first by `created_at`. No authentication guard is declared on this route.

## Implementation — how it works

The handler `list_segments` lives in `placer/api/debug.py`, registered on a FastAPI `APIRouter` with the prefix `/debug` and tag `debug`. The full mounted path in the application is therefore `/debug/segments`.

**Database access** is performed via `placer/db.py`'s `get_conn()` async context manager, which draws a connection from a `psycopg_pool.AsyncConnectionPool` backed by the `DATABASE_URL` environment variable. The pool is initialised lazily on first use (min 2, max 10 connections).

**Query construction** uses a lightweight dynamic `WHERE` clause: if `status` is provided it appends `status = %s` to the condition list; otherwise the predicate defaults to `TRUE`. A single parameterised SQL statement then reads from `segments` ordered by `created_at DESC LIMIT %s`. There is no `OFFSET` parameter — the endpoint is effectively a "top-N by recency" view rather than a fully cursor-paginated one.

The handler is part of the broader `debug` router that also exposes read-only views over `events`, `orders`, `beliefs`, `crosswalk_edges`, and `orgs`. All routes in this module are described as **read-only views over the spine** and carry no write side-effects.

## Availability — is it usable right now

The handler `list_segments` is present in `placer/api/debug.py` and registered on the `APIRouter`. No authentication or authorisation guards are applied to this route.

**Runtime prerequisites that must be satisfied before the endpoint is reachable:**
- `DATABASE_URL` must be set in the environment; the pool will raise `RuntimeError` at connection time if it is absent.
- The `segments` table must exist and be reachable in the target Postgres instance.
- The `debug` router must be included in the top-level FastAPI application (not confirmed from this file alone — verify in the application entry point).

No changelog entries exist, so there is no documented intent to gate or remove this endpoint. Assuming the router is mounted correctly and the database is available, the endpoint is expected to be functional.
