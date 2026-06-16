---
feature: List Events
group: Events
first_commit: 5499bc5a8c1f45be4e6cdc23b3f7414d926340f0
last_synced: '2026-06-15'
last_commit: 6dc428c8cfbf577dc8254a42c8b1873db3babcd4
anchors:
  tables:
  - events
  endpoints:
  - GET /events
  types: []
  api_modules:
  - placer.db
  files:
  - placer/api/debug.py::list_events
writes: []
reads:
- events
---
## Capability — what it can do

`GET /debug/events` returns a paginated, optionally filtered read-only view of the **event spine** — the append-only ledger that every downstream subsystem in Placer derives from.

**Filtering.** Callers may narrow results by the `kind` query parameter (exact string match against the `event_kind` column) and/or `order_id`. Both filters are optional and composable; omitting both returns the full spine ordered by `seq DESC`.

**Pagination.** `limit` (1–1 000, default 100) and `offset` (≥ 0, default 0) provide offset-based pagination. The response always includes a `total` count that reflects the filtered set, enabling frontend page-count computation independent of the current page.

**Per-event fields returned.** Each record in the `events` array carries: `seq` (monotonically increasing insert sequence), `event_kind`, `order_id`, `entity_refs` (JSONB cross-reference bag), `payload` (JSONB, kind-specific structure), `provenance` (JSONB: source, trust tier, adapter/model versions), `observed_at` (ISO-8601, when the real-world fact occurred), and `recorded_at` (ISO-8601, when the row landed in the DB — the bitemporal second axis).

**Companion endpoint.** `GET /debug/events/kinds` (same file, `list_event_kinds`) returns the distinct set of `event_kind` values with their row counts, allowing UIs to populate filter dropdowns with live cardinalities.

**Known event kinds** (from `placer/core/events.py`, `EventKind` enum): ingestion (`ingest.order`, `ingest.org_snapshot`, `ingest.registry_sync`, `ingest.pallet_group`), inference (`inference.generation`, `inference.embedding`, `inference.rerank`, `inference.classification`), retrieval (`retrieval.query`), decision (`decision.valuation_snapshot`, `decision.dispatch`, `decision.override`, `decision.add_candidate`, `decision.stop`, `decision.reject_scoped`), outcome (`outcome.approval`, `outcome.rejection`, `outcome.disposition`, `outcome.acceptance`, `outcome.no_response`), identity (`identity.org_minted`, `identity.segment_minted`, `identity.segment_merged`, `identity.org_remap`, `identity.membership_update`), belief (`belief.checkpoint`), lifecycle (`lifecycle.transition`), and system (`system.allocator_update`, `system.param_change`, `system.member_toggle`, `system.model_release`).

## Implementation — how it works

**Handler.** `list_events` in `placer/api/debug.py` is registered on a FastAPI `APIRouter` with prefix `/debug` and tag `debug`. The full effective path is therefore `GET /debug/events`. Query parameters are declared with FastAPI `Query` validators that enforce the `ge`/`le` bounds server-side. Note: the filter parameter is named `kind` (the URL query string key), which is matched against the `event_kind` column.

**Router mounting.** `placer/api/server.py` imports the debug router as `debug_router` and registers it via `app.include_router(debug_router)` with no additional prefix. The `/debug` prefix is entirely owned by the router declaration in `placer/api/debug.py`.

**Database access.** The handler borrows a connection from the shared `psycopg` async connection pool via the `get_conn()` async context manager in `placer/db.py`. The pool is initialised at application startup by the FastAPI `lifespan` handler in `server.py` (which calls `init_pool()` before yielding), configured from the `DATABASE_URL` environment variable (min 2, max 10 connections). All SQL in this handler is read-only (`SELECT`).

**Two-query pattern.** The handler issues two sequential queries against the `events` table:
1. `SELECT COUNT(*)` over the filtered predicate to populate `total`.
2. A `SELECT seq, event_kind, order_id, entity_refs, payload, provenance, observed_at, recorded_at … ORDER BY seq DESC LIMIT %s OFFSET %s` to fetch the page rows.

The WHERE clause is built by accumulating condition strings and positional parameters, then joining with `AND`; when no filters are provided the clause degrades to the literal `TRUE`, keeping the query planner path consistent.

**Bitemporality.** The `events` table stores two timestamps per row: `observed_at` (business time — when the fact occurred in the real world) and `recorded_at` (system time — the DB `now()` at insert). This separation is load-bearing in `placer/core/events.py`'s `query` function (which supports `as_known_at` point-in-time reads) and is surfaced directly in the debug response.

**Type system.** Payload validation at write time is governed by `EVENT_PAYLOAD_MODELS` in `placer/core/events.py`, which maps each `EventKind` enum value to a Pydantic model. Not every `EventKind` has a registered payload model: `inference.embedding`, `inference.classification`, `decision.stop`, `system.allocator_update`, and `ingest.registry_sync` are defined in the enum but absent from `EVENT_PAYLOAD_MODELS`, meaning their payloads pass through without structured validation at write time. The debug endpoint does not re-validate; it surfaces the raw JSONB from the DB verbatim.

**Frontend integration.** `frontend/src/api.ts` exports `api.events(params)` and `api.eventKinds()`, which call `/debug/events` and `/debug/events/kinds` respectively. The TypeScript interfaces `EventRecord` and `EventKindCount` (both confirmed in `frontend/src/api.ts`) shape those responses on the client side. `frontend/src/views/Events.tsx` uses both: it seeds a kind filter dropdown from `eventKinds()`, drives paginated fetches (page size 50) through `events()`, and renders an expandable row table showing the three JSONB blobs (`entity_refs`, `payload`, `provenance`) inline.

## Availability — is it usable right now

The handler `list_events` is present and fully implemented in `placer/api/debug.py`. The route `GET /debug/events` is registered on the debug router and mounted on the application in `placer/api/server.py` with no authentication guards.

**Runtime prerequisite.** A reachable PostgreSQL database and a valid `DATABASE_URL` environment variable are required. The connection pool is opened eagerly during the FastAPI `lifespan` startup hook (`init_pool()` in `server.py`); a missing or unreachable `DATABASE_URL` will prevent the application from starting rather than failing per-request. No in-process fallback or mock is present in the codebase.

**Route table discrepancy.** The feature table registers this feature under `GET /events` (guards: none), but the confirmed path — in both `placer/api/debug.py` and `frontend/src/api.ts` — is `GET /debug/events`. No handler for a bare `GET /events` path exists in the codebase. Clients must use `/debug/events`.

**`GET /events/kinds` anchor corrected.** The current anchor list includes `GET /events/kinds`, but the only confirmed handler is `GET /debug/events/kinds` (in `placer/api/debug.py`; also hardcoded in `frontend/src/api.ts`). The bare `/events/kinds` path has no corresponding handler and has been removed from the anchor block.

**`EventKindCount` and `EventRecord` types.** These are TypeScript interface types defined in `frontend/src/api.ts`, not Python types. Both are confirmed present and actively used by `frontend/src/views/Events.tsx`.

**Unregistered event kinds.** Four `EventKind` values — `inference.embedding`, `inference.classification`, `decision.stop`, and `system.allocator_update` — plus `ingest.registry_sync` have no entry in `EVENT_PAYLOAD_MODELS` (confirmed in `placer/core/events.py`). Events with these kinds can be stored and are fully readable via this endpoint, but their payloads are not validated against a typed model at write time. Note: `retrieval.query` was previously unregistered but now has a registered payload model (`RetrievalQueryPayload`).

**No changelog entries.** No changelog is configured; there are no pending intent-vs-code discrepancies beyond those noted above.
