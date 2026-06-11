---
feature: Adapters
group: Placer
last_synced: '2026-06-11'
last_commit: 5499bc5a8c1f45be4e6cdc23b3f7414d926340f0
anchors:
  tables:
  - events
  - items
  - m2_workflow_charities
  - m2_workflow_charity_status_events
  - m2_workflow_instances
  - mission_match_resolved
  - orders
  - pallet_groups
  endpoints: []
  types:
  - InferenceAdapter
  - IngestOrderPayload
  - M2RetrievalAdapter
  - NormalizedRecord
  - OrderConstraints
  - OutcomePayload
  - OutcomeSource
  - RawItem
  - ReasonCode
  - SimpliOrderAdapter
  - SimpliOutcomePoller
  - SimpliWritebackAdapter
  api_modules:
  - psycopg
  - psycopg_pool
  files:
  - placer/adapters/**
  - placer/adapters/simpli_db.py
  - placer/adapters/simpli_orders.py
  - placer/adapters/simpli_outcomes.py
  - placer/adapters/simpli_writeback.py
  - placer/adapters/types.py
writes:
- placer/adapters/simpli_writeback.py
reads:
- placer/adapters/simpli_db.py
- placer/adapters/simpli_orders.py
- placer/adapters/simpli_outcomes.py
- placer/adapters/types.py
- placer/api/server.py
---
## Capability — what it can do

The Adapters feature provides the **sole schema-coupling boundary** between Placer and Simpli's platform Postgres. It is composed of four confirmed modules:

- **`simpli_db`** — manages an async connection pool (`psycopg_pool.AsyncConnectionPool`) to Simpli's Postgres instance. Exposes `simpli_conn()`, a lazy-initialising async context manager that any adapter can acquire a connection from. Pool size is bounded at min=1 / max=4.

- **`simpli_orders`** (`simpli_orders/v1`) — a read-only adapter that fetches one order at a time from Simpli's `orders`, `pallet_groups`, and `items` tables, normalises the rows into a typed `IngestOrderPayload`, and appends an `INGEST_ORDER` event with `TrustTier.AUTHORITATIVE_REGISTRY` provenance.

- **`simpli_writeback`** (`simpli_writeback/v1`) — the single write path to Simpli. Logs a `DECISION_VALUATION_SNAPSHOT` propensity event to Placer's own store **before** any row is pushed to Simpli's `m2_workflow_instances` table, enforcing the governing "propensity-before-actuation" rule (spec §2.3, governing rule 4).

- **`simpli_outcomes`** (`simpli_outcomes/v1`) — a polling adapter that queries `mission_match_resolved` (approval/rejection outcomes) and `m2_workflow_charity_status_events` (outreach disposition changes) since a watermark derived from the latest already-ingested outcome event. Maps Simpli's free-text rejection reasons to a structured `ReasonCode` taxonomy using conservative regex heuristics, always preserving the original `reason_text`.

Abstract contracts for all three Simpli adapters (and for M2 retrieval and LLM inference) are declared in **`types.py`**: `SimpliOrderAdapter`, `SimpliWritebackAdapter`, `OutcomeSource`, `M2RetrievalAdapter`, and `InferenceAdapter`. These ABCs define the swap point that allows, for example, the polling outcome adapter to be replaced by a webhook receiver without touching downstream code.

## Implementation — how it works

**Connection pool (`simpli_db`)**  
A module-level `_pool` singleton of type `psycopg_pool.AsyncConnectionPool` is initialised lazily on first call to `init_simpli_pool()`. The DSN is read from the `SIMPLI_PLATFORM_URL` environment variable; the function raises `RuntimeError` immediately if the variable is absent. `close_simpli_pool()` drains and nullifies the singleton. The `simpli_conn()` async context manager calls `init_simpli_pool()` on every acquisition, making it safe to call before explicit pool initialisation.

> **Lifecycle note:** `init_simpli_pool` and `close_simpli_pool` are **not** wired into the FastAPI `lifespan` handler in `placer/api/server.py` (which manages only Placer's own pool via `placer.db`). The Simpli pool is therefore initialised on-demand at first use and is never explicitly torn down during a clean server shutdown.

**Order ingestion (`simpli_orders`)**  
`fetch_order_raw()` executes three sequential `SELECT` queries within a single `simpli_conn()` context: order header, pallet groups, and per-group items. `normalize()` derives `IngestOrderPayload` including `OrderConstraints` (perishable, hazmat class, donor restrictions from `client_notes`). `ingest_order()` then opens a Placer `get_conn()` context to append the event and upsert a row into Placer's own `orders` table.

**Write-back (`simpli_writeback`)**  
`write_recommendations()` uses a strict two-phase commit: (1) opens `placer.db.get_conn()`, appends the propensity event, and lets the context manager commit; (2) only then opens `simpli_conn()` and executes an `INSERT … ON CONFLICT DO UPDATE` against `m2_workflow_instances`. There is no code path that reaches the Simpli write without first committing the propensity record.

**Outcome polling (`simpli_outcomes`)**  
`SimpliOutcomePoller.pull(since)` calls two private methods in sequence. `_pull_mission_match_resolved` reads rows where `created_at > since OR matched_at > since`; skips rows where `client_approved IS NULL` (not yet resolved); maps approved rows to `OUTCOME_APPROVAL` and rejected rows to `OUTCOME_REJECTION` with a `map_reason_text()` code. `_pull_outreach_status_events` joins `m2_workflow_charity_status_events → m2_workflow_charities → m2_workflow_instances` and maps only `interested` and `not_interested` statuses (other statuses are silently skipped). The poll watermark is derived by querying `MAX(observed_at)` from Placer's own `events` table, filtered to `event_kind LIKE 'outcome.%'` from a `simpli_platform` source.

**Reason-code mapping**  
`map_reason_text()` applies an ordered list of compiled-regex rules against lowercased free text. First match wins; no match returns `ReasonCode.OTHER`. The raw text is always stored in `OutcomePayload.reason_text` regardless of mapping outcome.

## Availability — is it usable right now

**Connection infrastructure:** `simpli_db.py` is present and importable. Runtime availability depends on the `SIMPLI_PLATFORM_URL` environment variable being set and pointing to a reachable Postgres instance. If the variable is absent, `get_simpli_url()` raises `RuntimeError` on first connection attempt. The pool is **not** pre-warmed at server startup — it initialises lazily on first use.

**`simpli_orders` (read adapter):** Code is present. `ingest_order()` is callable but is not wired to any HTTP endpoint or background task visible in the current codebase. It is available as a library function only.

**`simpli_writeback` (write adapter):** Code is present and enforces the propensity-before-actuation invariant. Like the orders adapter, it is not reachable through any currently registered route — the `POST /recommendations` endpoint returns a stub empty response and does not invoke `write_recommendations()`.

**`simpli_outcomes` (outcome poller):** `SimpliOutcomePoller` is present and implements the `OutcomeSource` interface. No scheduler, background task, or route that calls `SimpliOutcomePoller.pull()` was found in the codebase. The poller is available as a library class but is not actively scheduled.

**Abstract contracts (`types.py`):** All five ABCs (`SimpliOrderAdapter`, `SimpliWritebackAdapter`, `OutcomeSource`, `M2RetrievalAdapter`, `InferenceAdapter`) are present and define stable swap points, but no concrete non-Simpli implementations were found.

**Summary:** All adapter modules compile and are logically complete, but none are reachable via the live API surface. The platform is in an M0 stub state; the adapter layer is ready for integration but not yet connected to the request pipeline.
