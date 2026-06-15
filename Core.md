---
feature: Core
group: New
first_commit: 6dc428c8cfbf577dc8254a42c8b1873db3babcd4
last_synced: '2026-06-15'
last_commit: 6dc428c8cfbf577dc8254a42c8b1873db3babcd4
anchors:
  tables:
  - events
  - members
  - system_params
  endpoints: []
  types:
  - AddCandidatePayload
  - AllocatorContract
  - BeliefCheckpointPayload
  - BeliefStore
  - CandidateId
  - CanonicalizeContract
  - ContactId
  - DispatchAction
  - DispatchCandidateRecord
  - DispatchContract
  - DispatchKind
  - DispatchPayload
  - DonorId
  - Ein
  - EntityRefs
  - Event
  - EventKind
  - EventLog
  - EventSeq
  - ExplorationSlotPurpose
  - FactorBreakdown
  - GateVerdict
  - GenerateContract
  - Hypothesis
  - HypothesisId
  - HypothesisProvenance
  - IndexSchema
  - InferenceGenerationPayload
  - InferenceRerankPayload
  - IngestContract
  - IngestOrderPayload
  - IngestOrgSnapshotPayload
  - IngestPalletGroupPayload
  - LearnContract
  - LifecycleTransitionPayload
  - MemberTogglePayload
  - MembershipUpdatePayload
  - MintProposal
  - ModelReleasePayload
  - OrderId
  - OrgId
  - OrgMintedPayload
  - OrgRemapPayload
  - OutcomePayload
  - OutcomeSourceContract
  - OverridePayload
  - PalletGroupId
  - ParamChangePayload
  - ProjectionContract
  - Provenance
  - QuantityId
  - QuantityRegistryEntry
  - QuantityType
  - ReasonCode
  - ReasonCodeFamily
  - RecordContract
  - RejectScopedPayload
  - Resolution
  - ResolutionConfidence
  - ResolutionMethod
  - ResolveContract
  - RetrievalQueryPayload
  - SegmentAssignment
  - SegmentId
  - SegmentMergedPayload
  - SegmentMintedPayload
  - StreamId
  - TrustTier
  - UnspscClass
  - UnspscFamily
  - Valuation
  - ValuationSnapshotPayload
  - ValueContract
  - WantProvenance
  api_modules:
  - placer/core/events.py::append
  - placer/core/events.py::latest_seq
  - placer/core/events.py::query
  - placer/core/member_registry.py::get_param
  - placer/core/member_registry.py::list_members
  - placer/core/member_registry.py::register_member
  - placer/core/member_registry.py::set_param
  - placer/core/member_registry.py::toggle_member
  - placer/core/resolver.py::beta_fold_failure
  - placer/core/resolver.py::beta_fold_success
  - placer/core/resolver.py::coherence_priced_c
  - placer/core/resolver.py::partially_pool
  - placer/core/resolver.py::resolve_belief_from_checkpoint
  - placer/core/resolver.py::resolve_inherited_belief
  - placer/core/resolver.py::resolve_tree_backoff
  files:
  - placer/core/**
  - placer/core/__init__.py
  - placer/core/contracts.py
  - placer/core/events.py
  - placer/core/ids.py
  - placer/core/member_registry.py
  - placer/core/quantities.py
  - placer/core/resolver.py
writes:
- placer/core/events.py::append
- placer/core/member_registry.py::register_member
- placer/core/member_registry.py::set_param
- placer/core/member_registry.py::toggle_member
reads:
- placer/core/events.py::latest_seq
- placer/core/events.py::query
- placer/core/member_registry.py::get_param
- placer/core/member_registry.py::list_members
---
## Capability — what it can do

`placer/core` is the constitutional layer (spec V2 §VIII.3, T1 tier) that defines the platform's immutable shared vocabulary. It provides five distinct capabilities that all other modules depend on:

1. **ID spaces** (`ids.py`): Twelve `NewType`-wrapped canonical identifier types (`OrgId`, `OrderId`, `CandidateId`, `SegmentId`, `HypothesisId`, `QuantityId`, `EventSeq`, `OrgId`, `DonorId`, `ContactId`, `StreamId`, `PalletGroupId`, `Ein`, `UnspscFamily`, `UnspscClass`) plus a `Resolution` model carrying `ResolutionConfidence` and `ResolutionMethod`. Type wrappers prevent cross-space confusion at call sites; `resolve(raw) → (canonical_id, confidence)` is the declared only door into each space.

2. **Event spine** (`events.py`): The complete `EventKind` registry (40 named kinds across ingestion, inference, retrieval, decision, outcome, identity, belief, lifecycle, and system namespaces), all typed payload Pydantic models, `TrustTier`, `Provenance`, `EntityRefs`, `ReasonCode`/`ReasonCodeFamily`, and `FactorBreakdown`. The `append()` coroutine is the **single insert site** into the `events` table — no other code may `INSERT INTO events` (enforced by `tests/conformance/test_sacred_append.py`). A `query()` coroutine and `latest_seq()` complete the read interface.

3. **Quantity registry** (`quantities.py`): Fourteen named belief quantities (e.g. `p_want`, `p_fit`, `p_resp`, `cap_volume`, `lambda_ops`, `donor_reliability`) with typed estimand text, `QuantityType` (Prob, Volume, Rate, Duration, Money, Weights), and `IndexSchema` assignments. Minting rules (estimand + ≥1 real-evidence fold + ≥1 consumer) are enforced as convention, not code.

4. **Operator contracts** (`contracts.py`): Abstract base classes forming the complete interface surface for every stage in the pipeline — `IngestContract`, `GenerateContract`, `CanonicalizeContract`, `ResolveContract`, `ValueContract` (pure function, `FactorBreakdown` mandatory), `DispatchContract`, `RecordContract`, `LearnContract`, `OutcomeSourceContract`, `BeliefStore`, `EventLog`, `AllocatorContract`, and `ProjectionContract`. Changing any contract is a re-architecture, not a refactor.

5. **Resolver semantics** (`resolver.py`): Conjugate-and-mixture belief composition functions — `resolve_belief_from_checkpoint`, `resolve_inherited_belief`, `partially_pool`, `resolve_tree_backoff`, and `coherence_priced_c` — that implement spec V2 §IV.3–IV.5 (precision-weighted, partial-pooling, coherence-priced tree backoff). Also provides `beta_fold_success`/`beta_fold_failure` conjugate update helpers.

6. **Member registry** (`member_registry.py`): Evented write helpers (`set_param`, `register_member`, `toggle_member`) for the T4 system-params and T3 ensemble-members tables, where the event is appended before the derived row is updated — the only legal write path for those tables.

## Implementation — how it works

**Import discipline**: `core` imports nothing else internal, with exactly one documented exception — `core.resolver` imports `beliefs.representations` for its representation types, because resolver semantics are constitutional but cannot be expressed without them. This constraint is enforced by `tests/conformance/test_import_direction.py`. All other modules import from core, never the reverse.

**Event spine and sacred append**: `events.append()` validates the payload against the registered `EVENT_PAYLOAD_MODELS` dict (event kind → Pydantic model) before issuing a single `INSERT INTO events … RETURNING seq`. The returned `EventSeq` integer is the append's only side-effect. Bitemporality is load-bearing: `observed_at` (when the fact occurred) is caller-supplied; `recorded_at` defaults to `now()` in the database, enabling point-in-time replay. The `query()` function supports filtering by kind list, order, sequence watermark, and `as_known_at` for bitemporal reads.

**Resolver**: Operates entirely on `BeliefCheckpoint` sufficient-stats dicts (parsed into `BetaStats`, `NormalStats`, `ScalarStats`, or `MixtureStats`). `partially_pool` combines an org-level belief with an inherited mixture via a precision-weighted shrinkage coefficient. `resolve_tree_backoff` walks a list of `(level_name, checkpoint, dispersion)` tuples, computing `coherence_priced_c` at each level so shrinkage toward the parent is reduced when the tree is internally incoherent, not simply when the tree is tall. Interval computation delegates to `scipy.stats.beta` and `scipy.stats.norm`; mixture intervals use bisection over the mixture CDF.

**Contracts**: All contracts are abstract base classes with no implementation logic. The `ValueContract.value()` is declared as a pure function (no `async`). `FactorBreakdown.p_accept` is a derived `@property` (product of four factor probabilities) and the only computed value in core.

**Member registry writes**: `set_param` and `register_member`/`toggle_member` call `core.events.append` first, capturing the returned `EventSeq`, then perform the `INSERT`/`UPDATE` on `system_params` or `members`, referencing the event seq. An unevented parameter change would corrupt every comparison spanning it.

**Tables written**: `events` (via `append`), `system_params`, `members`. **Tables read**: `events`, `system_params`, `members`.

## Availability — is it usable right now

`placer/core` is a library module, not a service endpoint. It is available as an import to all other modules in the codebase. No HTTP route, auth guard, or feature flag governs its use; it is unconditionally loaded wherever it is imported.

The `events.append` function requires a live `psycopg.AsyncConnection` and a populated `events` table with the expected schema. The `member_registry` helpers additionally require `system_params` and `members` tables. Availability of these capabilities at runtime depends on database connectivity, not on any application-level guard.

`core.resolver` has a runtime dependency on `scipy` (for `scipy.stats.beta` and `scipy.stats.norm`); if `scipy` is absent the resolver will raise an `ImportError` on first use of interval computation.

No changelog is configured. No discrepancies between changelog claims and code state are present.
