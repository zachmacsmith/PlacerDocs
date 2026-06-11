---
feature: List Events
group: Events
last_synced: '2026-06-11'
last_commit: ba02630c316c435e071a627f433a21d08f9987e7
anchors:
  tables:
  - belief_checkpoints
  - candidates
  - crosswalk_edges
  - events
  - orders
  - orgs
  - quantity_registry
  - segments
  endpoints:
  - GET /beliefs/checkpoints
  - GET /beliefs/quantities
  - GET /crosswalk
  - GET /debug/beliefs/checkpoints
  - GET /debug/beliefs/quantities
  - GET /debug/crosswalk
  - GET /debug/events
  - GET /debug/events/kinds
  - GET /debug/orders
  - GET /debug/orders/{order_id}
  - GET /debug/orgs
  - GET /debug/segments
  - GET /debug/stats
  - GET /events
  - GET /events/kinds
  - GET /orders
  - GET /orders/{order_id}
  - GET /orgs
  - GET /segments
  - GET /stats
  types:
  - EVENT_PAYLOAD_MODELS
  - EntityRefs
  - Event
  - EventKind
  - EventKindCount
  - EventRecord
  - Provenance
  api_modules:
  - placer.api.debug
  - placer.db
  - placer.events.store
  - placer.events.types
  files:
  - frontend/src/api.ts
  - frontend/src/components/Layout.tsx
  - frontend/src/views/Events.tsx
  - placer/api/**
  - placer/api/debug.py
  - placer/db.py
  - placer/events/store.py
  - placer/events/types.py
writes: []
reads:
- belief_checkpoints
- candidates
- crosswalk_edges
- events
- orders
- orgs
- quantity_registry
- segments
---
## Capability — what it can do

`GET /debug/events` returns a paginated, optionally filtered read-only view of the **event spine** — the append-only ledger that every downstream subsystem in Placer derives from.

**Filtering.** Callers may narrow results by the `kind` query parameter (exact string match against the `event_kind` column) and/or `order_id`. Both filters are optional and composable; omitting both returns the full spine ordered by `seq DESC`.

**Pagination.** `limit` (1–1 000, default 100) and `offset` (≥ 0, default 0) provide offset-based pagination. The response always includes a `total` count that reflects the filtered set, enabling frontend page-count computation independent of the current page.

**Per-event fields returned.** Each record in the `events` array carries: `seq` (monotonically increasing insert sequence), `event_kind`, `order_id`, `entity_refs` (JSONB cross-reference bag), `payload` (JSONB, kind-specific structure), `provenance` (JSONB: source, trust tier, adapter/model versions), `observed_at` (ISO-8601, when the real-world fact occurred), and `recorded_at` (ISO-8601, when the row landed in the DB — the bitemporal second axis).

**Companion endpoint.** `GET /debug/events/kinds` (same file, `list_event_kinds`) returns the distinct set of `event_kind` values with their row counts, allowing UIs to populate filter dropdowns with live cardinalities.

**Known event kinds** (from `placer/events/types.py`, `EventKind` enum): ingestion (`ingest.order`, `ingest.org_snapshot`, `ingest.registry_sync`, `ingest.pallet_group`), inference (`inference.generation`, `inference.embedding`, `inference.rerank`, `inference.classification`), retrieval (`retrieval.query`), decision (`decision.valuation_snapshot`, `decision.dispatch`, `decision.override`, `decision.add_candidate`, `decision.stop`, `decision.reject_scoped`), outcome (`outcome.approval`, `outcome.rejection`, `outcome.disposition`, `outcome.acceptance`, `outcome.no_response`), identity (`identity.org_minted`, `identity.segment_minted`, `identity.segment_merged`), belief (`belief.checkpoint`), lifecycle (`lifecycle.transition`), and system (`system.allocator_update`).

## Implementation — how it works

**Handler.** `list_events` in `placer/api/debug.py` is registered on a FastAPI `APIRouter` with prefix `/debug` and tag `debug`. The full effective path is therefore `GET /debug/events`. Query parameters are declared with FastAPI `Query` validators that enforce the `ge`/`le` bounds server-side. Note: the filter parameter is named `kind` (the URL query string key), which is matched against the `event_kind` column.

**Database access.** The handler borrows a connection from the shared `psycopg` async connection pool via the `get_conn()` async context manager in `placer/db.py`. The pool is initialised lazily from the `DATABASE_URL` environment variable (min 2, max 10 connections). All SQL in this handler is read-only (`SELECT`).

**Two-query pattern.** The handler issues two sequential queries against the `events` table:
1. `SELECT COUNT(*)` over the filtered predicate to populate `total`.
2. A `SELECT seq, event_kind, order_id, entity_refs, payload, provenance, observed_at, recorded_at … ORDER BY seq DESC LIMIT %s OFFSET %s` to fetch the page rows.

The WHERE clause is built by accumulating condition strings and positional parameters, then joining with `AND`; when no filters are provided the clause degrades to the literal `TRUE`, keeping the query planner path consistent.

**Bitemporality.** The `events` table stores two timestamps per row: `observed_at` (business time — when the fact occurred in the real world) and `recorded_at` (system time — the DB `now()` at insert). This separation is load-bearing in `placer/events/store.py`'s `query` function (which supports `as_known_at` point-in-time reads) and is surfaced directly in the debug response.

**Type system.** Payload validation at write time is governed by `EVENT_PAYLOAD_MODELS` in `placer/events/types.py`, which maps each `EventKind` enum value to a Pydantic model. The debug endpoint does not re-validate; it surfaces the raw JSONB from the DB verbatim.

**Frontend integration.** `frontend/src/api.ts` exports `api.events(params)` and `api.eventKinds()`, which call `/debug/events` and `/debug/events/kinds` respectively. The TypeScript interfaces `EventRecord` and `EventKindCount` (both confirmed in `frontend/src/api.ts`) shape those responses on the client side. `frontend/src/views/Events.tsx` uses both: it seeds a kind filter dropdown from `eventKinds()`, drives paginated fetches (page size 50) through `events()`, and renders an expandable row table showing the three JSONB blobs (`entity_refs`, `payload`, `provenance`) inline.

## Availability — is it usable right now

The handler `list_events` is present and fully implemented in `placer/api/debug.py`. The route `GET /debug/events` is registered on the debug router with no authentication guards (confirmed: `guards: []` in the feature table).

**Runtime prerequisite.** The endpoint requires a reachable PostgreSQL database and a valid `DATABASE_URL` environment variable. Without it, `get_conn()` calls `init_pool()`, which raises a `RuntimeError` at first connection attempt; the request will fail with a 500. No in-process fallback or mock is present in the codebase.

**Route table discrepancy.** The feature table registers this feature under `GET /events` (component `list_events`, guards: none), but the handler is mounted at `GET /debug/events` via the `/debug`-prefixed router in `placer/api/debug.py`. No handler for a bare `GET /events` path was found in the codebase. Clients must use `/debug/events`.

**`GET /events/kinds` anchor corrected.** The existing anchor listed `GET /events/kinds`, but the confirmed route — both in `placer/api/debug.py` and in `frontend/src/api.ts` — is `GET /debug/events/kinds`. The bare `/events/kinds` path has no corresponding handler.

**`EventKindCount` and `EventRecord` types.** These are TypeScript interface types defined in `frontend/src/api.ts`, not Python types. They are confirmed present and used by `frontend/src/views/Events.tsx`.

**No changelog entries.** No changelog is configured, so there are no pending intent-vs-code discrepancies beyond those noted above.
