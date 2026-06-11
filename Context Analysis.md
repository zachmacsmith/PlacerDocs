---
feature: Context Analysis
group: Matching
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
  - quantity_registry
  - segments
  endpoints:
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
  - GET /health
  - POST /context-analysis
  - POST /recommendations
  types:
  - CharityRecommendation
  - ContextAnalysisResponse
  - RecommendationRequest
  - RecommendationResponse
  api_modules:
  - placer.api
  - placer.api.debug
  - placer.api.server
  - placer.db
  files:
  - placer/db.py
  - placer/api/server.py::context_analysis
writes:
- placer/api/server.py
reads:
- placer/api/debug.py
- placer/api/server.py
- placer/db.py
---
## Capability — what it can do

`POST /context-analysis` accepts a JSON body containing an EIN (Employer Identification Number) and returns a structured analysis of the corresponding charitable organisation. The response envelope (`ContextAnalysisResponse`) carries three fields: the normalised `ein` string, a free-text `analysis` narrative, and an optional `charity_data` dictionary intended to surface raw organisation attributes alongside the analysis.

The endpoint is part of the Simpli worker contract (spec §2.1) and is structurally parallel to the `POST /recommendations` endpoint served by the same FastAPI application. Both endpoints share the same API-key authentication guard, the same CORS policy, and the same application lifespan (database connection pool).

## Implementation — how it works

The handler lives in `placer/api/server.py` as the `context_analysis` async function, registered on the FastAPI `app` instance via `@app.post("/context-analysis", response_model=ContextAnalysisResponse)`.

**Authentication.** Every request must supply a valid key in the `X-API-Key` HTTP header. The `_verify_api_key` helper checks the header value against the comma-separated list in the `API_KEYS` environment variable, raising HTTP 401 on any mismatch or absence.

**Request shape.** The handler accepts an untyped `dict[str, Any]` body rather than a dedicated Pydantic request model. It extracts and strips the `ein` field manually, returning HTTP 400 if the value is absent or blank.

**Response shape.** Responses are validated against `ContextAnalysisResponse`, a Pydantic `BaseModel` with fields `ein: str`, `analysis: str`, and `charity_data: dict[str, Any] | None`.

**Current stub behaviour.** The handler returns a hardcoded `analysis` string (`"Context analysis not yet implemented in Placer."`) and `charity_data=None`. No database query, no Bayesian inference, and no external service call is made. The implementation is a placeholder that satisfies the Simpli contract surface while the full pipeline is built out.

**Infrastructure wiring.** The FastAPI `app` is initialised with a `lifespan` context manager (defined in the same file) that calls `placer.db.init_pool` on startup and `placer.db.close_pool` on shutdown, giving all handlers access to the shared `psycopg_pool.AsyncConnectionPool` (pool size: min 2, max 10, backed by `DATABASE_URL`). The context-analysis handler itself does not currently use that pool. CORS is configured to allow `POST`, `GET`, and `OPTIONS` requests from the four permitted Simpli origins (`localhost:5173`, `localhost:5174`, `app.simpli.supply`, `app.simplisupply.com`). A debug read-only router (`placer/api/debug.py`) is mounted at `/debug` on the same `app` instance; it does not interact with the context-analysis handler.

## Availability — is it usable right now

The route is **present and reachable** in the codebase (`placer/api/server.py`, confirmed at line 129). It will respond to authenticated `POST /context-analysis` requests at runtime, but the response is a static stub: `analysis` is always the fixed string `"Context analysis not yet implemented in Placer."` and `charity_data` is always `null`. No meaningful organisation data is returned.

**Runtime prerequisites.** The server requires two environment variables to start successfully:
- `DATABASE_URL` — the Postgres connection string; `placer.db.init_pool` raises `RuntimeError` if absent, which aborts the lifespan startup and prevents the application from serving any requests.
- `API_KEYS` — one or more comma-separated keys; without this, every request returns HTTP 401.

The context-analysis handler itself makes no database calls in its current form, but the pool initialisation is a shared lifespan concern that gates the entire application.

**No changelog entries exist** to indicate a target date or planned scope for the full implementation. The discrepancy between the endpoint's presence in the Simpli worker contract and its stub-only behaviour should be considered when evaluating production readiness.
