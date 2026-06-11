---
feature: Events
group: Placer
last_synced: '2026-06-11'
last_commit: 5499bc5a8c1f45be4e6cdc23b3f7414d926340f0
anchors:
  tables:
  - events
  endpoints:
  - GET /debug/events
  - GET /debug/events/kinds
  types:
  - AddCandidatePayload
  - BeliefCheckpointPayload
  - DispatchCandidateRecord
  - DispatchPayload
  - EVENT_PAYLOAD_MODELS
  - EntityRefs
  - Event
  - EventKind
  - EventSeq
  - ExplorationSlotPurpose
  - FactorBreakdown
  - GeneratedHypothesis
  - InferenceGenerationPayload
  - InferenceRerankPayload
  - IngestOrderPayload
  - IngestOrgSnapshotPayload
  - IngestPalletGroupPayload
  - LifecycleTransitionPayload
  - OrgMintedPayload
  - OutcomePayload
  - OverridePayload
  - Provenance
  - ReasonCode
  - ReasonCodeFamily
  - RejectScopedPayload
  - RetrievalQueryPayload
  - SegmentMergedPayload
  - SegmentMintedPayload
  - TrustTier
  - ValuationSnapshotPayload
  - WantProvenance
  api_modules:
  - placer.contracts
  - placer.db
  - placer.events
  - placer.events.store
  - placer.events.types
  - placer.identity
  - placer.identity.types
  files:
  - placer/adapters/simpli_orders.py
  - placer/adapters/simpli_outcomes.py
  - placer/adapters/simpli_writeback.py
  - placer/api/debug.py
  - placer/contracts.py
  - placer/db.py
  - placer/events/**
  - placer/events/store.py
  - placer/events/types.py
  - placer/identity/types.py
writes:
- events
reads:
- GET /debug/events
- GET /debug/events/kinds
---
## Capability — what it can do

The Events feature is the **append-only spine** of the Placer platform. Every significant action — ingestion, inference, retrieval, decision, outcome, identity minting, belief checkpointing, and lifecycle transitions — is recorded as an immutable event row, making the spine the single authoritative record of what the system did and when.

**Append with payload validation.** `store.append` accepts an `EventKind`, a `Provenance` record (source, trust tier, version fields), an `EntityRefs` bag of cross-entity foreign keys, a free-form JSONB payload, an optional `OrderId`, and the caller-supplied `observed_at` timestamp. Before writing, it validates the payload against the registered Pydantic model for that `EventKind` via `EVENT_PAYLOAD_MODELS` — a typed registry that covers 21 of the 27 defined event kinds. The database assigns an auto-incrementing `seq` (returned as `EventSeq`) and stamps `recorded_at` at insert time.

**Bitemporal queries.** `store.query` supports simultaneous filtering on event kind (`ANY` match over a list), `order_id`, a cursor-style lower bound (`since_seq > N`), and an `as_known_at` ceiling on `recorded_at`. This separates *when something happened* (`observed_at`, supplied by the caller) from *when the system learned about it* (`recorded_at`, set by the database), which is the load-bearing bitemporality noted in the module docstring. Results are returned in ascending `seq` order, up to a configurable limit (default 1 000).

**Sequence watermark.** `store.latest_seq` returns the current maximum `seq` (or 0 on an empty table), used by the `LearnContract` fold operation and other consumers that need a reliable watermark for incremental processing.

**Event kind taxonomy.** `EventKind` (a `StrEnum`) defines 27 kinds across seven families: `ingest.*`, `inference.*`, `retrieval.*`, `decision.*`, `outcome.*`, `identity.*`, `belief.*`, `lifecycle.*`, and `system.*`. Each kind with a registered payload model enforces a typed schema at write time.

**Abstract read/write contracts.** `contracts.py` defines `RecordContract` (append) and `EventLog` (query + latest_seq) as abstract base classes. The concrete `store.py` functions are the reference implementation; the contract boundary makes the spine swappable without touching callers.

**Debug API.** Two read-only HTTP endpoints (`GET /debug/events`, `GET /debug/events/kinds`) expose paginated event rows and per-kind counts directly against the `events` table, bypassing the store module but reading the same underlying data.


## Implementation — how it works

**Storage.** All events land in a single PostgreSQL table named `events`. Columns confirmed by the INSERT and SELECT statements in `store.py`: `seq` (serial PK, returned via `RETURNING`), `event_kind` (string), `order_id` (nullable FK), `entity_refs` (JSONB), `payload` (JSONB), `provenance` (JSONB), `observed_at` (timestamp, caller-supplied), `recorded_at` (timestamp, DB default). The `entity_refs` column stores a serialized `EntityRefs` model, which carries optional references to `OrgId`, `OrderId`, `SegmentId`, `StreamId`, `HypothesisId`, `CandidateId`, `ContactId`, `QuantityId`, and `UnspscFamily` — allowing any event to be cross-linked to any entity without nullable FK columns for each.

**Payload validation at write time.** `EVENT_PAYLOAD_MODELS` maps each `EventKind` to a Pydantic `BaseModel` subclass. `store.append` calls `model_cls.model_validate(payload)` before executing the INSERT, so invalid payloads raise a Pydantic `ValidationError` that aborts the write. Kinds without a registered model (e.g. `INFERENCE_EMBEDDING`, `INFERENCE_CLASSIFICATION`, `RETRIEVAL_QUERY` has one but some system/decision kinds do not) pass through unvalidated.

**Provenance model.** Every event carries a `Provenance` record with `source`, `trust_tier` (`TrustTier` StrEnum: `declared_by_org`, `authoritative_registry`, `scraped`, `llm_inferred`), and optional `adapter_version`, `model_version`, `prompt_version`, and `actor`. Provenance is serialised to JSONB via `_to_json`.

**Async I/O.** All three public functions are `async` and accept a `psycopg.AsyncConnection`. Connection acquisition is handled by callers via `placer.db.get_conn`, a context-manager backed by a `psycopg_pool.AsyncConnectionPool` sized 2–10, configured from the `DATABASE_URL` environment variable.

**Adapter write path.** The three confirmed callers of `store.append` are `placer/adapters/simpli_orders.py`, `placer/adapters/simpli_outcomes.py`, and `placer/adapters/simpli_writeback.py`. All import the store as `event_store` and call `event_store.append(...)` inside database transactions, making the adapters the primary write producers.

**Debug read path.** `placer/api/debug.py` contains `list_events` (`GET /debug/events`) and `list_event_kinds` (`GET /debug/events/kinds`). These handlers query the `events` table directly with raw SQL (not through `store.query`), supporting `kind` and `order_id` filters plus `limit`/`offset` pagination and a `COUNT(*)` total. They are mounted under the `/debug` router prefix within the FastAPI application.

**`FactorBreakdown` cross-cutting type.** `placer/events/types.py` also defines `FactorBreakdown` and `ValuationSnapshotPayload`, which are imported and reused by `placer/value/types.py`, `placer/candidates/types.py`, and `placer/projections/types.py`, making `placer.events.types` a shared type library beyond its event-spine role.


## Availability — is it usable right now

**Write path (append):** Code is present and in active use by the three adapter modules (`simpli_orders`, `simpli_outcomes`, `simpli_writeback`). Writes are gated on `DATABASE_URL` being set and the `events` table existing; no schema migration file was confirmed in this investigation, so table availability depends on external provisioning.

**Read path — programmatic (`store.query`, `store.latest_seq`):** Both functions are implemented. No direct callers of `store.query` were found in this investigation beyond the abstract `EventLog` contract in `contracts.py`; concrete wiring to a live handler could not be confirmed. `store.latest_seq` is referenced in `contracts.py`'s `EventLog.latest_seq` abstraction but no runtime binding was located.

**Read path — HTTP (`GET /debug/events`, `GET /debug/events/kinds`):** Both endpoints are defined in `placer/api/debug.py` under the `/debug` prefix router. Whether this router is registered in the main FastAPI application was not confirmed in this investigation; availability to HTTP clients is therefore **uncertain**. No authentication guard is declared on these routes.

**No changelog:** No changelog is configured, so there are no pending or recently shipped changes to characterise. All capability claims above derive solely from confirmed source code.

