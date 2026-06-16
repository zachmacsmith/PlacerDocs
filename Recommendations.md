---
feature: Recommendations
group: Matching
first_commit: 5499bc5a8c1f45be4e6cdc23b3f7414d926340f0
last_synced: '2026-06-16'
last_commit: e75bcb47650c6c56370cc10193be5f08ae3490e8
anchors:
  tables: []
  endpoints:
  - POST /recommendations
  types:
  - CharityRecommendation
  - RecommendationRequest
  - RecommendationResponse
  api_modules:
  - placer.adapters
  - placer.api
  - placer.core
  - placer.db
  - placer.pipeline
  files:
  - placer/api/server.py::recommendations
writes: []
reads:
- placer/api/server.py
---
## Capability — what it can do

`POST /recommendations` is the primary external-facing endpoint of the Placer platform. It accepts a donation-order identifier and returns a ranked list of charity (`CharityRecommendation`) objects that Placer judges to be the best match for that order.

Each `CharityRecommendation` carries a rich profile: EIN, name, NTEE code, geolocation fields (latitude, longitude, street, city, state, zip, timezone), contact details, mission and about statements, affiliation flags, boost preferences, a cosine-style `about_similarity` score, and optional `rerank_reasoning` text. Two Placer-native extension fields — `factor_breakdown` (per-signal score decomposition) and `provenance_bridges` (lineage trace of evidence sources) — are included in the schema and are additive and contract-compatible with the incumbent Simpli mission-match worker interface this endpoint mirrors (spec §2.1).

The response envelope (`RecommendationResponse`) also carries a `metadata` dict for diagnostic values such as total candidate counts and version tags.

The endpoint accepts an optional `top_n` query parameter (1–100, default 20) to cap the result list, and an optional `filters` dict and `admin_context` string in the request body for caller-side narrowing and context injection.

## Implementation — how it works

The handler lives in `placer/api/server.py` and is mounted on the FastAPI `app` instance created in the same file. Authentication is enforced before any business logic: the handler calls `_verify_api_key`, which reads the comma-separated `API_KEYS` environment variable and raises HTTP 401 for a missing or unrecognised `X-API-Key` header. An empty `order_id` is rejected with HTTP 400.

The application lifecycle (`lifespan`) initialises and tears down a connection pool (via `placer.db`) shared across all routes, and also spawns a background asyncio task that calls `run_periodic` from `placer.beliefs.learner` every 30 seconds. This task folds new belief evidence continuously while the server is running; it is cancelled cleanly on shutdown.

The server is CORS-enabled for `localhost:5173`, `localhost:5174`, `app.simpli.supply`, and `app.simplisupply.com`.

**Live pipeline (M1):** After validating the request, the handler acquires a database connection and delegates to `placer.pipeline.recommend`, which executes the full sequence: *propose → gate → value → rank*. Concretely, it fetches or generates candidates for the order via `placer.pipeline`, evaluates hard-stop gates, scores each passing candidate using Bayesian posteriors for five signals (`p_want`, `p_fit_residual`, `p_cap`, `p_resp`, `p_accept`), sorts by descending `p_accept`, and returns the top-`n` results. The `factor_breakdown` field on each `CharityRecommendation` exposes these five probability values plus a `gates_pass` boolean. `metadata` in the response includes `placer_version`, `total_proposed`, `gate_blocked`, `scored`, and `returned` counts.

## Availability — is it usable right now

The route `POST /recommendations` is **present in code, fully wired, and capable of returning real ranked recommendations** once the server is running with the required environment and data prerequisites satisfied.

**Runtime prerequisites that must all be satisfied before the server starts:**
- `DATABASE_URL` must be set and point to a reachable Postgres instance; the connection pool (managed via `placer.db`) is initialised at startup and startup fails without it.
- `API_KEYS` must contain at least one non-empty value; an absent or empty variable causes every request to fail with HTTP 401.

**Data prerequisites for non-empty results:**
- An order record matching the supplied `order_id` must exist in the database. If no order is found, the pipeline returns immediately with `metadata.error = "order not found"` and an empty recommendations list.
- Candidate charities must be reachable (either already stored for the order or proposable via `placer.pipeline`). If no candidates survive gate evaluation, the ranked list will be empty even for a valid order.

**Background belief task:** The server continuously updates Bayesian belief checkpoints via a 30-second periodic fold task (`placer.beliefs.learner`). Scores improve as more evidence is accumulated; a freshly seeded database will produce reasonable but prior-dominated probability estimates until sufficient interaction data accumulates.

**Response version tag:** `metadata.placer_version` is now `"0.1.0"` (the `-stub` suffix was removed when the pipeline was wired in).

**Known limitations:**
- No changelog entries exist to describe the specific data seeding requirements or minimum viable dataset for meaningful recommendations.
- The `CharityRecommendation` schema includes fields (name, NTEE code, geolocation, contact details, etc.) that the pipeline does not currently populate — only `ein`, `factor_breakdown`, and `provenance_bridges` are set by the handler. Other fields remain `null` unless populated by a future enrichment stage.
