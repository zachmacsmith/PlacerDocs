---
feature: API
group: Placer
first_commit: 5499bc5a8c1f45be4e6cdc23b3f7414d926340f0
last_synced: '2026-06-16'
last_commit: cf05efbdebcc895b1cbd9fbfa7376868d3584da1
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
  - GET /events
  - GET /events/kinds
  - GET /orders
  - GET /orders/{order_id}
  - GET /orgs
  - GET /segments
  - GET /stats
  - POST /ingest-order
  types:
  - ManualItemInput
  - ManualOrderInput
  api_modules:
  - placer.core
  - placer.db
  files:
  - placer/api/**
writes:
- orders
reads:
- belief_checkpoints
- candidates
- crosswalk_edges
- events
- orgs
- placer/api/debug.py
- quantity_registry
- segments
---
## Capability — what it can do

The API feature exposes two distinct groups of HTTP endpoints served by a single FastAPI application (`placer/api/server.py`).

**Debug read-only introspection** (`placer/api/debug.py`, router prefix `/debug`):
All endpoints are unauthenticated read-only views over the core database tables, intended for dashboard and operational visibility.

| Endpoint | Purpose |
|---|---|
| `GET /debug/events` | Paginated event log with optional filters on `event_kind` and `order_id` |
| `GET /debug/events/kinds` | Aggregated count per distinct `event_kind` |
| `GET /debug/orders` | Paginated order list with per-order candidate stage counts (contacted / resolved) |
| `GET /debug/orders/{order_id}` | Full detail for a single order including all candidates and stage breakdown |
| `GET /debug/beliefs/quantities` | All registered quantities with checkpoint count and average effective sample size (`n_eff`) |
| `GET /debug/beliefs/checkpoints` | Paginated belief checkpoints with optional filter on `quantity_id` |
| `GET /debug/crosswalk` | Crosswalk edges joined to segments, optionally filtered by `family` |
| `GET /debug/segments` | Segment records optionally filtered by `status` |
| `GET /debug/orgs` | Org records ordered by first-seen event sequence |
| `GET /debug/stats` | Row counts for all seven core tables plus a top-10 event kind breakdown |

**Worker contract endpoints** (`placer/api/server.py`, no prefix), authenticated via `X-API-Key` header validated against the `API_KEYS` environment variable:

| Endpoint | Purpose |
|---|---|
| `POST /recommendations` | Returns a ranked list of charity `CharityRecommendation` objects for a given `order_id` |
| `POST /context-analysis` | Returns a `ContextAnalysisResponse` for a given EIN |
| `GET /health` | Liveness probe returning `{"status": "ok", "service": "placer"}` |

The server also configures CORS for `localhost:5173`, `localhost:5174`, `app.simpli.supply`, and `app.simplisupply.com`, permitting `POST`, `OPTIONS`, and `GET` with `Content-Type` and `X-API-Key` headers.

## Implementation — how it works

**Application bootstrap** (`placer/api/server.py`): A FastAPI app is instantiated with a `lifespan` context manager that calls `init_pool()` on startup, spawns a background `asyncio.Task` that runs the belief-learner's `run_periodic()` loop every 30 seconds, and on shutdown cancels that task before calling `close_pool()`. The debug router is mounted via `app.include_router(debug_router)`, inheriting the `/debug` prefix declared inside the router itself.

**Database connectivity** (`placer/db.py`): A module-level `psycopg_pool.AsyncConnectionPool` (min 2, max 10) is lazily initialised from the `DATABASE_URL` environment variable. `init_pool()` is idempotent — it returns the existing pool if already open. The `get_conn()` async context manager calls `init_pool()` on each invocation (no-op if already initialised), then acquires a connection via `pool.connection()` and yields a `psycopg.AsyncConnection[tuple]`; the pool returns the connection automatically on context exit. All debug endpoints share this single pool.

**Debug endpoint pattern**: Every handler in `debug.py` opens a `get_conn()` context, issues one or two raw SQL queries against the named table(s), fetches all result rows as plain tuples, and serialises them into a dict. Datetime columns are converted with `.isoformat()` before inclusion. Pagination is enforced via FastAPI `Query` bounds on `limit`/`offset` parameters at the handler level; no ORM layer is involved.

**Worker contract endpoints** (`server.py`): `POST /recommendations` and `POST /context-analysis` verify the `X-API-Key` header against a comma-separated `API_KEYS` environment variable before processing. `POST /recommendations` is fully wired: it calls `recommend()` from the pipeline module, commits the resulting database writes, and assembles a `RecommendationResponse` whose `CharityRecommendation` entries carry Placer-specific `factor_breakdown` (five probability scores plus a gate-pass flag) and `provenance_bridges` derived from each candidate's surfaced-by set. `POST /context-analysis` still returns a static `"Context analysis not yet implemented in Placer."` message.

**Pydantic models**: `RecommendationRequest`, `RecommendationResponse`, `CharityRecommendation`, and `ContextAnalysisResponse` are declared in `server.py` and match Simpli's incumbent mission-match worker contract, with additive Placer-specific fields (`factor_breakdown`, `provenance_bridges`) that are contract-compatible.

## Availability — is it usable right now

**Debug endpoints** (`GET /debug/*`): All ten debug endpoints are fully implemented in `placer/api/debug.py` and mounted in `placer/api/server.py` with no guards. They are available to any client that can reach the server — no API key is required. They depend on a reachable PostgreSQL instance and a valid `DATABASE_URL` environment variable; requests will fail at connection time if these are absent.

**`GET /health`**: Implemented and unauthenticated; always returns `{"status": "ok", "service": "placer"}` without touching the database.

**`POST /recommendations`**: Fully implemented and guarded by `X-API-Key` / `API_KEYS` env-var. The endpoint calls the real `recommend()` pipeline, commits database writes, and returns a ranked `RecommendationResponse` with per-candidate factor breakdowns and provenance bridges. Version metadata is `"placer_version": "0.1.0"`. Requires a reachable PostgreSQL instance and valid `API_KEYS`; returns a 401 if the key is absent or invalid.

**`POST /context-analysis`**: Implemented and guarded by `X-API-Key` / `API_KEYS` env-var. Returns a static `"Context analysis not yet implemented in Placer."` message. No analysis logic is connected.

**Background belief learner**: On server startup a periodic task calls the belief-learner's fold loop every 30 seconds. This runs automatically as part of the server process; no separate worker is needed, but it depends on the same database pool as the rest of the application.

**Anchor correction**: Previous anchor entries for unprefixed routes (`GET /events`, `GET /orders`, `GET /orgs`, etc.) have been removed — neither `debug.py` nor `server.py` defines routes at those paths. Only the `/debug/*` prefixed routes and the three worker-contract endpoints exist in current code.

No changelog is configured; there are no changelog-vs-code discrepancies to flag beyond the above.
