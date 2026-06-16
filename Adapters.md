---
feature: Adapters
group: Placer
first_commit: 5499bc5a8c1f45be4e6cdc23b3f7414d926340f0
last_synced: '2026-06-16'
last_commit: cf05efbdebcc895b1cbd9fbfa7376868d3584da1
anchors:
  tables: []
  endpoints: []
  types: []
  api_modules: []
  files:
  - placer/adapters/**
writes: []
reads:
- placer/adapters/seed_historical.py
- placer/adapters/simpli_outcomes.py
---
## Capability — what it can do

The Adapters feature provides the **sole schema-coupling boundary** between Placer and Simpli's platform Postgres. It is composed of five confirmed modules:

- **`simpli_db`** — manages an async connection pool (`psycopg_pool.AsyncConnectionPool`) to Simpli's Postgres instance. Exposes `simpli_conn()`, a lazy-initialising async context manager that any adapter can acquire a connection from. Pool size is bounded at min=1 / max=4.

- **`simpli_orders`** (`simpli_orders/v1`) — a read-only adapter that fetches one order at a time from Simpli's `orders`, `pallet_groups`, and `items` tables, normalises the rows into a typed `IngestOrderPayload`, and appends an `INGEST_ORDER` event with `TrustTier.AUTHORITATIVE_REGISTRY` provenance.

- **`simpli_writeback`** (`simpli_writeback/v1`) — the single write path to Simpli. Logs a `DECISION_VALUATION_SNAPSHOT` propensity event to Placer's own store **before** any row is pushed to Simpli's `m2_workflow_instances` table, enforcing the governing "propensity-before-actuation" rule (spec §2.3, governing rule 4).

- **`simpli_outcomes`** (`simpli_outcomes/v1`) — a polling adapter that queries `mission_match_resolved` (approval/rejection outcomes) and `m2_workflow_charity_status_events` (outreach disposition changes) since a watermark derived from the latest already-ingested outcome event. Maps Simpli's free-text rejection reasons to a structured `ReasonCode` taxonomy using conservative regex heuristics, always preserving the original `reason_text`. Each outcome event is enriched with org-identity data: the charity EIN is resolved to a Placer `OrgId` (minting a provisional org if none exists), and the org's segment memberships and crosswalk family are attached to `EntityRefs`.

- **`seed_historical`** — a one-shot CLI script (M1 §4) that bootstraps the belief system from Simpli's full history. Pulls all resolved outcomes from epoch (2020-01-01) via `SimpliOutcomePoller`, then runs the fold loop in batches of 5,000 events until all events are consumed and initial belief checkpoints are written. Also provides `count_simpli_outcomes()` for a quick pre-seed inventory of available outcomes in Simpli. Run with `python -m placer.adapters.seed_historical`.

Abstract contracts for all three Simpli adapters (and for M2 retrieval and LLM inference) are declared in **`types.py`**: `SimpliOrderAdapter`, `SimpliWritebackAdapter`, `OutcomeSource`, `M2RetrievalAdapter`, and `InferenceAdapter`. These ABCs define the swap point that allows, for example, the polling outcome adapter to be replaced by a webhook receiver without touching downstream code.

- **`synthetic`** — a full-loop harness (spec V2 §VIII.2, §X M0) that exercises the entire ingest → identity → candidates → propensity → dispatch → outcomes → routing-table folds → checkpoint refresh → resolver read pipeline with zero external credentials. It implements `OutcomeSourceContract` via `SyntheticOutcomeSource`, generates plausible scripted outcomes for contacted candidates, and performs inline belief-checkpoint folds using the routing table. Intended for debugging mechanics, not threshold tuning. Run with `python -m placer.adapters.synthetic`.

## Implementation — how it works

**Connection pool (`simpli_db`)**  
A module-level `_pool` singleton of type `psycopg_pool.AsyncConnectionPool` is initialised lazily on first call to `init_simpli_pool()`. The DSN is read from the `SIMPLI_PLATFORM_URL` environment variable via `get_simpli_url()`; the function raises `RuntimeError` immediately if the variable is absent. `close_simpli_pool()` drains and nullifies the singleton. The `simpli_conn()` async context manager calls `init_simpli_pool()` on every acquisition, making it safe to call before explicit pool initialisation. The pool is created with `open=False` and then explicitly opened, giving the caller control over the open-wait.

> **Lifecycle note:** `init_simpli_pool` and `close_simpli_pool` are **not** wired into the FastAPI `lifespan` handler in `placer/api/server.py` (which manages only Placer's own pool via `placer.db`). The Simpli pool is therefore initialised on-demand at first use and is never explicitly torn down during a clean server shutdown.

**Order ingestion (`simpli_orders`)**  
`fetch_order_raw()` executes three sequential `SELECT` queries within a single `simpli_conn()` context: order header, pallet groups, and per-group items. `normalize()` derives `IngestOrderPayload` including `OrderConstraints` (perishable, hazmat class, donor restrictions from `client_notes`). `ingest_order()` then opens a Placer `get_conn()` context to append the event and upsert a row into Placer's own `orders` table.

**Write-back (`simpli_writeback`)**  
`write_recommendations()` uses a strict two-phase commit: (1) opens `placer.db.get_conn()`, appends the propensity event, and lets the context manager commit; (2) only then opens `simpli_conn()` and executes an `INSERT … ON CONFLICT DO UPDATE` against `m2_workflow_instances`. There is no code path that reaches the Simpli write without first committing the propensity record. The function also accepts `workflow_name`, `created_by`, and `search_filters` parameters that are forwarded directly into the Simpli row.

**Outcome polling (`simpli_outcomes`)**  
`SimpliOutcomePoller.pull(since)` calls two private methods in sequence. `_pull_mission_match_resolved` reads rows where `created_at > since OR matched_at > since`; skips rows where `client_approved IS NULL` (not yet resolved); maps approved rows to `OUTCOME_APPROVAL` and rejected rows to `OUTCOME_REJECTION` with a `map_reason_text()` code. `_pull_outreach_status_events` joins `m2_workflow_charity_status_events → m2_workflow_charities → m2_workflow_instances` and maps only `interested` and `not_interested` statuses (other statuses are silently skipped). The poll watermark is derived by querying `MAX(observed_at)` from Placer's own `events` table, filtered to `event_kind LIKE 'outcome.%'` from a `simpli_platform` source.

Before appending each event, both private methods call `_resolve_or_mint_org(conn, ein)`: if the EIN matches an existing identity-store entry, the canonical `OrgId` is returned; otherwise a provisional org is minted with `ResolutionConfidence.PROVISIONAL`. `_lookup_org_edges(conn, org_id)` then retrieves the org's segment memberships from the identity store and queries `crosswalk_edges` for an associated family name. Both are attached to `EntityRefs` alongside the `org_id` before the event is appended.

**Historical seeding (`seed_historical`)**  
`seed_outcomes()` instantiates a `SimpliOutcomePoller` with a hard-coded epoch of 2020-01-01 UTC, ingests all resolved outcomes, then calls `_fold_all()`. `_fold_all()` runs `fold_events()` in a loop with `batch_size=5000`, committing after each batch and advancing the watermark, until a batch returns zero events processed. The entry point `main()` first calls `count_simpli_outcomes()` to log a pre-seed inventory (counting rows in `mission_match_resolved` where `client_approved IS NOT NULL` and all rows in `m2_workflow_charity_status_events`), then executes the full seed.

**Reason-code mapping**  
`map_reason_text()` applies an ordered list of regex rules against lowercased free text. First match wins; no match returns `ReasonCode.OTHER`. The raw text is always stored in `OutcomePayload.reason_text` regardless of mapping outcome. The taxonomy covers capacity (`CAP_FULL_NOW`, `CAP_VOLUME_EXCEEDED`), want (`WANT_NO_USE`, `WANT_ALREADY_SUPPLIED`, `WANT_LOW_PRIORITY`), fit (`FIT_RESTRICTION_CONFLICT`, `FIT_GEOGRAPHIC`, `FIT_REGULATORY`), and response (`RESP_NO_RESPONSE`, `RESP_DECLINED`) families.

## Availability — is it usable right now

**Connection infrastructure (`simpli_db`):** All functions — `get_simpli_url()`, `init_simpli_pool()`, `close_simpli_pool()`, and `simpli_conn()` — are present and importable. Runtime availability depends entirely on the `SIMPLI_PLATFORM_URL` environment variable being set and pointing to a reachable Postgres instance. If the variable is absent, `get_simpli_url()` raises `RuntimeError` on the first connection attempt. The pool is **not** pre-warmed at server startup: the server's `lifespan` handler calls only `placer.db.init_pool()` / `close_pool()` — the Simpli pool is never explicitly opened or closed by the server lifecycle.

**`simpli_orders` (read adapter):** Code is present and logically complete (`fetch_order_raw`, `normalize`, `ingest_order`). `ingest_order()` is callable but is not wired to any HTTP endpoint or background task visible in the current codebase. It is available as a library function only.

**`simpli_writeback` (write adapter):** Code is present and enforces the propensity-before-actuation invariant. `write_recommendations()` is callable but not reachable through any currently registered route — `POST /recommendations` returns a stub empty response and does not invoke `write_recommendations()`.

**`simpli_outcomes` (outcome poller):** `SimpliOutcomePoller` is present and fully implements the `OutcomeSource` interface, including org-identity resolution and `EntityRefs` enrichment via `_resolve_or_mint_org()` and `_lookup_org_edges()`. No scheduler, background task, or route that calls `SimpliOutcomePoller.pull()` was found in the codebase. The poller is available as a library class but is not actively scheduled.

**`seed_historical` (historical bootstrap):** `seed_historical.py` is present and runnable directly (`python -m placer.adapters.seed_historical`). It requires both `SIMPLI_PLATFORM_URL` (for Simpli access) and `DATABASE_URL` (for Placer's own Postgres). It is a one-shot operator tool targeting M1 §4 bootstrap — not a production component invoked by any route or scheduled task.

**`synthetic` (full-loop harness):** The synthetic harness is present and runnable directly (via its `__main__` guard). It requires only a `DATABASE_URL` environment variable pointing to a local Placer Postgres instance — no Simpli credentials needed. It is a developer/debug tool, not a production component.

**Abstract contracts (`types.py`):** All five ABCs (`SimpliOrderAdapter`, `SimpliWritebackAdapter`, `OutcomeSource`, `M2RetrievalAdapter`, `InferenceAdapter`) are present and define stable swap points, but no concrete non-Simpli implementations were found.

**Summary:** All adapter modules are logically complete and compile cleanly, but none are connected to the live API surface. The platform is in an M0 stub state (`POST /recommendations` returns an empty list). The adapter layer is ready for integration but is not yet invoked by the request pipeline. The `seed_historical` script provides the M1 §4 bootstrap path for pre-populating belief checkpoints from Simpli's full history.
