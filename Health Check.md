---
feature: Health Check
group: System
last_synced: '2026-06-11'
last_commit: 5499bc5a8c1f45be4e6cdc23b3f7414d926340f0
anchors:
  tables: []
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
  - placer/api/**
  - placer/api/debug.py
  - placer/api/server.py
  - placer/db.py
writes: []
reads: []
---
## Capability — what it can do

`GET /health` is a lightweight liveness probe for the Placer service. It accepts no parameters and requires no authentication. It responds with a fixed JSON object — `{"status": "ok", "service": "placer"}` — confirming that the FastAPI process is running and able to handle HTTP requests. No database query is performed and no application logic is exercised; the endpoint is intentionally side-effect-free so that load balancers, orchestrators, and uptime monitors can call it at any frequency without operational cost.

## Implementation — how it works

The endpoint is defined in `placer/api/server.py` as the async function `health()`, decorated with `@app.get("/health")`. It is registered on the top-level `FastAPI` application instance (`app`) alongside the two authenticated contract endpoints (`POST /recommendations`, `POST /context-analysis`) and the debug sub-router (`placer/api/debug.py`, mounted at `/debug`).

**Application lifecycle.** The `app` object uses a `lifespan` context manager (also in `server.py`) that calls `init_pool()` from `placer/db.py` on startup and `close_pool()` on shutdown. `init_pool` opens an `AsyncConnectionPool` (via `psycopg_pool`) sized 2–10 connections, driven by the `DATABASE_URL` environment variable. The health endpoint itself does **not** touch this pool; it returns before any DB interaction.

**Auth bypass.** All other non-debug routes call the private `_verify_api_key` helper, which checks the `X-API-Key` header against a comma-separated `API_KEYS` environment variable and raises HTTP 401 on failure. `health()` intentionally omits this check.

**CORS.** The server registers `CORSMiddleware` allowing `GET` from a fixed allowlist of origins (`localhost:5173`, `localhost:5174`, `app.simpli.supply`, `app.simplisupply.com`), so the health endpoint is reachable via browser-based monitoring dashboards hosted on those origins.

**Return type.** The handler's return type annotation is `dict[str, str]`; FastAPI serialises this directly to JSON with a `200 OK` response. No Pydantic response model is declared for this route, so no schema validation overhead occurs on the response path.

## Availability — current state

The endpoint is present and fully operational in source. `GET /health` is unconditionally registered on the `FastAPI` app in `placer/api/server.py`; there are no feature flags, guards, or middleware that could block it.

**Runtime prerequisites.** The only hard requirement is that the FastAPI process starts successfully. The `lifespan` hook will attempt to open a Postgres connection pool via `DATABASE_URL` at startup; if `DATABASE_URL` is unset or unreachable, the application will fail to start and the endpoint will never become available. Once the application is up, the health endpoint itself has no further dependency on the database.

**No changelog context.** No changelog is configured for this project, so there is no documented intent to add, change, or remove this endpoint beyond what the current code reflects.
