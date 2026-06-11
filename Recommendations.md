---
feature: Recommendations
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
  - placer/api/server.py::recommendations
writes:
- placer/api/server.py
reads:
- placer/api/debug.py
- placer/api/server.py
- placer/db.py
---
## Capability — what it can do

`POST /recommendations` is the primary external-facing endpoint of the Placer platform. It accepts a donation-order identifier and returns a ranked list of charity (`CharityRecommendation`) objects that Placer judges to be the best match for that order.

Each `CharityRecommendation` carries a rich profile: EIN, name, NTEE code, geolocation fields (latitude, longitude, street, city, state, zip, timezone), contact details, mission and about statements, affiliation flags, boost preferences, a cosine-style `about_similarity` score, and optional `rerank_reasoning` text. Two Placer-native extension fields — `factor_breakdown` (per-signal score decomposition) and `provenance_bridges` (lineage trace of evidence sources) — are included in the schema and are additive and contract-compatible with the incumbent Simpli mission-match worker interface this endpoint mirrors (spec §2.1).

The response envelope (`RecommendationResponse`) also carries a `metadata` dict for diagnostic values such as total candidate counts and version tags.

The endpoint accepts an optional `top_n` query parameter (1–100, default 20) to cap the result list, and an optional `filters` dict and `admin_context` string in the request body for caller-side narrowing and context injection.

## Implementation — how it works

The handler lives in `placer/api/server.py` and is mounted on the FastAPI `app` instance created in the same file. Authentication is enforced before any business logic: the handler calls `_verify_api_key`, which reads the comma-separated `API_KEYS` environment variable and raises HTTP 401 for a missing or unrecognised `X-API-Key` header. An empty `order_id` is rejected with HTTP 400.

The application lifecycle (`lifespan`) initialises and tears down a `psycopg_pool.AsyncConnectionPool` (pool size 2–10) via `placer/db.py`, which is shared across all routes including the debug router (`placer/api/debug.py`).

The server is CORS-enabled for `localhost:5173`, `localhost:5174`, `app.simpli.supply`, and `app.simplisupply.com`.

**Current pipeline state (M0 stub):** The handler currently returns an empty `recommendations` list with `metadata.placer_version = "0.1.0-stub"` and `total_candidates = 0`. The source code documents the intended full pipeline as: *ingest → generate → resolve → value → rank* — but none of those pipeline stages are wired into this handler yet. The `CharityRecommendation` and `RecommendationResponse` Pydantic models are fully defined and match the Simpli worker contract, ready for the pipeline to populate them.

## Availability — is it usable right now

The route `POST /recommendations` is **present in code and reachable** once the server is running with a valid `DATABASE_URL` and at least one entry in the `API_KEYS` environment variable. The endpoint accepts well-formed requests and returns a structurally valid `RecommendationResponse`.

However, **it unconditionally returns an empty recommendations list** — confirmed directly in `placer/api/server.py` at the M0 stub comment. No charity candidates are surfaced, ranked, or scored regardless of the `order_id` supplied. The response satisfies the Simpli §2.1 wire contract but carries zero actionable data.

**Runtime prerequisites that must all be satisfied before the server starts:**
- `DATABASE_URL` must be set and point to a reachable Postgres instance (enforced in `placer/db.py` at pool open time).
- `API_KEYS` must contain at least one non-empty value; an absent or empty variable causes every request to fail with HTTP 401.

**Known limitations:**
- No changelog entries exist to indicate a target date or completion status for the pipeline stages. The full `CharityRecommendation` schema implies rich per-signal scoring, but the discrepancy between that schema and the current stub output is a confirmed, in-code limitation until the ingest → generate → resolve → value → rank stages are wired in.
