---
feature: Events
group: Placer
first_commit: 5499bc5a8c1f45be4e6cdc23b3f7414d926340f0
last_synced: '2026-06-11'
last_commit: 87dd52f08e97ba92e8de49eace545f1073d264af
anchors:
  tables:
  - events
  - orders
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
  - OrderConstraints
  - OrgMintedPayload
  - OutcomePayload
  - OverridePayload
  - Provenance
  - RawItem
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
  - placer/contracts.py
  - placer/db.py
  - placer/events/**
  - placer/events/store.py
  - placer/events/types.py
  - placer/identity/types.py
  - placer/api/debug.py::list_events
  - placer/api/debug.py::list_event_kinds
writes:
- events
- orders
reads:
- events
---
## Capability — what it can do

The Events feature is the **append-only spine** of the Placer platform. Every significant action — ingestion, inference, retrieval, decision, outcome, identity minting, belief checkpointing, and lifecycle transitions — is recorded as an immutable event row, making the spine the single authoritative record of what the system did and when.

**Append with payload validation.** `store.append` accepts an `EventKind`, a `Provenance` record (source, trust tier, version fields), an `EntityRefs` bag of cross-entity foreign keys, a free-form JSONB payload, an optional `OrderId`, and the caller-supplied `observed_at` timestamp. Before writing, it validates the payload against the registered Pydantic model for that `EventKind` via `EVENT_PAYLOAD_MODELS` — a typed registry that covers 21 of the 26 defined event kinds. The database assigns an auto-incrementing `seq` (returned as `EventSeq`) and stamps `recorded_at` at insert time.

**Bitemporal queries.** `store.query` supports simultaneous filtering on event kind (`ANY` match over a list), `order_id`, a cursor-style lower bound (`since_seq > N`), and an `as_known_at` ceiling on `recorded_at`. This separates *when something happened* (`observed_at`, supplied by the caller) from *when the system learned about it* (`recorded_at`, set by the database), which is the load-bearing bitemporality noted in the module docstring. Results are returned in ascending `seq` order, up to a configurable limit (default 1 000).

**Sequence watermark.** `store.latest_seq` returns the current maximum `seq` (or 0 on an empty table), used by the `LearnContract` fold operation and other consumers that need a reliable watermark for incremental processing.

**Event kind taxonomy.** `EventKind` (a `StrEnum`) defines 26 kinds across eight families: `ingest.*` (4), `inference.*` (4), `retrieval.*` (1), `decision.*` (6), `outcome.*` (5), `identity.*` (3), `belief.*` (1), `lifecycle.*` (1), and `system.*` (1). Each kind with a registered payload model enforces a typed schema at write time.

**Abstract read/write contracts.** `contracts.py` defines `RecordContract` (append) and `EventLog` (query + latest_seq) as abstract base classes. The concrete `store.py` functions are the reference implementation; the contract boundary makes the spine swappable without touching callers.

**Debug API.** Two read-only HTTP endpoints (`GET /debug/events`, `GET /debug/events/kinds`) expose paginated event rows and per-kind counts directly against the `events` table, bypassing the store module but reading the same underlying data.


## Implementation — how it works

**Storage.** All events land in a single PostgreSQL table named `events`. Columns confirmed by the INSERT and SELECT statements in `store.py`: `seq` (serial PK, returned via `RETURNING`), `event_kind` (string), `order_id` (nullable FK), `entity_refs` (JSONB), `payload` (JSONB), `provenance` (JSONB), `observed_at` (timestamp, caller-supplied), `recorded_at` (timestamp, DB default). The `entity_refs` column stores a serialized `EntityRefs` model, which carries optional references to `OrgId`, `OrderId`, `SegmentId` (list), `StreamId`, `HypothesisId`, `CandidateId`, `ContactId`, `QuantityId`, and `UnspscFamily` — allowing any event to be cross-linked to any entity without nullable FK columns per entity type.

**Payload validation at write time.** `EVENT_PAYLOAD_MODELS` maps each `EventKind` to a Pydantic `BaseModel` subclass. `store.append` calls `model_cls.model_validate(payload)` before executing the INSERT, so invalid payloads raise a Pydantic `ValidationError` that aborts the write. Kinds without a registered model (e.g. `INFERENCE_EMBEDDING`, `INFERENCE_CLASSIFICATION`, `DECISION_STOP`, `SYSTEM_ALLOCATOR_UPDATE`, `INGEST_REGISTRY_SYNC`) pass through unvalidated — five of 26 kinds have no registered payload model.

**Provenance model.** Every event carries a `Provenance` record with `source`, `trust_tier` (`TrustTier` StrEnum: `declared_by_org`, `authoritative_registry`, `scraped`, `llm_inferred`), and optional `adapter_version`, `model_version`, `prompt_version`, and `actor` fields. These are serialized to JSONB with `exclude_none=True` to keep storage compact.

**Factor breakdown (spec §6).** The `FactorBreakdown` model — embedded in `ValuationSnapshotPayload` and `DispatchCandidateRecord` — decomposes acceptance probability into four factors: `p_fit`, `p_want`, `p_cap`, `p_resp`. A `p_accept` property computes the product. `WantProvenance` records whether the `p_want` estimate came from an edge posterior, self-consistency, direct evidence, or a prior. This breakdown is spec-mandatory and surfaced on every valuation event.

**Reason code taxonomy.** `ReasonCode` (a `StrEnum`) defines 17 codes across five families (`FIT`, `WANT`, `CAP`, `RESP`, `OTHER`). A `.family` property derives the `ReasonCodeFamily` from the value prefix. Reason codes appear in `OutcomePayload` for all five outcome event kinds.

**Debug endpoints.** `GET /debug/events` supports `kind`, `order_id`, `limit` (1–1 000, default 100), and `offset` query parameters, returning rows in descending `seq` order alongside a total count. `GET /debug/events/kinds` returns a per-kind count aggregation. Both endpoints query the `events` table directly via raw SQL in `placer/api/debug.py`, independent of the store module.

**Adapter integration.** `placer/adapters/simpli_orders.py` demonstrates the full append path: it fetches from Simpli's platform Postgres, normalizes into `IngestOrderPayload`, and calls `event_store.append` with `EventKind.INGEST_ORDER`, a `Provenance` record (`trust_tier=authoritative_registry`, `adapter_version="simpli_orders/v1"`), and the Placer `OrderId` in both `order_id` and `entity_refs`. The adapter also upserts the `orders` table row (`ON CONFLICT DO NOTHING`) in the same connection context.

**JSON serialization.** `store._to_json` uses `json.dumps` with `default=str` to handle `datetime` and other non-serializable values when casting dicts to JSONB parameters.


## Availability — what is usable right now

**`store.append`, `store.query`, `store.latest_seq`** — all three async functions are fully implemented in `placer/events/store.py` and import cleanly from `placer/events/types` and `placer/identity/types`. They are available to any caller with a `psycopg.AsyncConnection`.

**`GET /debug/events` and `GET /debug/events/kinds`** — both endpoints are registered in `placer/api/debug.py` under the `/debug` router with the `debug` tag. They are read-only and require no authentication guards (none are declared). Availability at runtime depends on whether the debug router is mounted in the application's main router — this was not confirmed from the files read; callers should verify router mounting before relying on these endpoints being reachable.

**`EVENT_PAYLOAD_MODELS` registry** — 21 of 26 `EventKind` values have registered payload models. Five kinds (`INFERENCE_EMBEDDING`, `INFERENCE_CLASSIFICATION`, `DECISION_STOP`, `SYSTEM_ALLOCATOR_UPDATE`, `INGEST_REGISTRY_SYNC`) are present in the `EventKind` enum but absent from `EVENT_PAYLOAD_MODELS`; appending events of these kinds bypasses payload validation.

**Abstract contracts (`RecordContract`, `EventLog`)** — defined in `placer/contracts.py` and importable. No concrete class implementing them via Python inheritance was confirmed in the files read; `store.py` provides equivalent free functions rather than a class implementation.

**`placer/adapters/simpli_orders.py`** — the `ingest_order` entry point is fully implemented and calls `event_store.append`. Runtime availability depends on the `placer.adapters.simpli_db` connection helper (not read) and a live Simpli platform Postgres connection.

**No changelog** — no changelog is configured; there are no pending or announced changes that would affect availability.

