---
feature: Core
group: New
first_commit: 6dc428c8cfbf577dc8254a42c8b1873db3babcd4
last_synced: '2026-06-15'
last_commit: 7fdd634f38a1f90503ccc57e81634980b0f2102f
anchors:
  tables: []
  endpoints: []
  types: []
  api_modules: []
  files:
  - placer/core/**
writes: []
reads:
- placer/core/__init__.py
- placer/core/contracts.py
- placer/core/events.py
- placer/core/ids.py
- placer/core/member_registry.py
- placer/core/predicates.py
- placer/core/quantities.py
---
## Capability — what it can do

The core package is the constitutional layer (spec V2 §VIII.3) of the Placer platform. It provides four categories of foundational capability that every other module consumes but may not extend or replace without a spec-level migration:

**ID spaces** (`placer/core/ids.py`): Defines all canonical identifier types — `OrgId`, `OrderId`, `CandidateId`, `SegmentId`, `HypothesisId`, `QuantityId`, `AttributeId`, `GateId`, `EventSeq`, and others — as Python `NewType` wrappers that prevent cross-space confusion at call sites. `resolve(raw) → (canonical_id, confidence)` is declared here as the sole entry point into any ID space, accompanied by `Resolution`, `ResolutionConfidence`, and `ResolutionMethod` models.

**Event spine** (`placer/core/events.py`): Maintains the append-only event table that everything downstream derives from. The module owns the full `EventKind` registry (60+ kinds spanning ingestion, inference, retrieval, decision, outcome, identity, constraint gate lifecycle, review workflow, belief, lifecycle, and system governance events), all typed payload models (`IngestOrderPayload`, `DispatchPayload`, `OutcomePayload`, V2.1 constraint/attribute/elicitation/review payloads, etc.), the `ReasonCode` / `ReasonCodeFamily` taxonomy, the `FactorBreakdown` model (the mandatory gate-and-factor decomposition: `p_accept = gates_pass_indicator × p_fit_residual × p_want × p_cap × p_resp`), and the `Provenance` / `TrustTier` structs. `EntityRefs` now carries optional `attribute_id` and `gate_id` fields to link events to V2.1 constraint and attribute entities. The `append()` function is the **single INSERT site** into the `events` table; no other module in the codebase may issue a direct `INSERT INTO events` (enforced by a conformance test). A `query()` function and `latest_seq()` helper are also provided for read access.

**Quantity registry** (`placer/core/quantities.py`): Declares the closed set of belief quantities — `p_want`, `p_donor_approve`, `p_fit`, `p_fit_archetype`, `p_resp`, `p_resp_pooled`, `resp_time`, `resp_time_pooled`, `cap_volume`, `cap_volume_pooled`, `lambda_ops`, `donor_reliability`, `p_attr`, `member_weight`, `proposer_coverage_credit`, `member_calibration` — each with a typed estimand sentence, `QuantityType`, and `IndexSchema`. Two new index schemas support V2.1 quantities: `attribute_org` (size-band × NTEE attribute posteriors) and `member_context` (ensemble member stacking weights and calibration scores). Minting rules are legislative: a new quantity requires a real-evidence fold and at least one consumer.

**Resolver semantics** (`placer/core/resolver.py`): Implements the read-time, precision-weighted belief resolution protocol (spec V2 §IV.3–IV.5). Three composition modes are present: `resolve_belief_from_checkpoint()` for direct lookup, `resolve_inherited_belief()` for membership-weighted mixture, and `partially_pool()` for conjugate shrinkage toward a prior. `resolve_tree_backoff()` implements coherence-priced hierarchical backoff, where certainty reduces with incoherence rather than tree height (§IV.5). Conjugate fold helpers `beta_fold_success()` and `beta_fold_failure()` are also defined here. This module carries a documented import exception: it imports `beliefs.representations` (a T2a module) because the resolver semantics require representation types to be expressed at all — the only departure from core's otherwise-inward-only import discipline, enforced by a conformance test.

**Constraint layer** (`placer/core/predicates.py`): Declares the closed constitutional set of constraint predicates as `PredicateId` (a `StrEnum`), organised into five groups per spec V2.1 §IV.8.2: item attributes (`HAZMAT_CLASS`, `FAMILY_SUBSUMPTION`, `REFRIGERATION`, `EXPIRY`, `LOT_SIZE`, `CONDITION_GRADE`, `TEMPERATURE_RANGE`), org attributes (`ORG_CAPABILITY`, `ORG_RESTRICTION`, `ORG_CERTIFICATION`), geography (`GEOGRAPHIC_DISTANCE`, `GEOGRAPHIC_EXCLUSION`, `SERVICE_AREA`), regulatory/compliance (`REGULATORY_BLOCK`), and donor-sourced (`DONOR_RESTRICTION`, `DONOR_EXCLUSION`). `RESTRICTION_CONFLICT` is retained as a legacy alias mapping to the V2.0 skeleton. The vocabulary is closed — adding a predicate is a T1 change requiring migration and rationale. Beyond the predicate vocabulary, this module also owns the three closed **compile forms** (`CompileForm`: `HARD_GATE`, `TRANSFORMATION_PRECONDITION`, `PRIOR_INJECTION`) per §IV.8.1, the **`GateInstance`** runtime model (T2b, derived from `constraint.*` events; tracks `gate_id`, `compile_form`, `predicate_id`, `predicate_args`, `scope`, `provenance_tier`, TTL, epsilon, and lifecycle counters), the **`GateStatus`** enum (`ACTIVE`, `DEMOTED`, `RETIRED`, `SUPERSEDED`), and the **`SegmentStatus`** enum (`PROVISIONAL`, `ESTABLISHED`, `MERGED`, `ARCHIVED`). The syntax ceiling is conjunctions over registered predicates plus subsumption and distance; disjunction-of-conjunctions, negation-of-segments, and recursion are prohibited.

**Contracts** (`placer/core/contracts.py`): Defines the frozen abstract interfaces — `IngestContract`, `GenerateContract`, `CanonicalizeContract`, `CompileContract`, `ResolveContract`, `ValueContract`, `DispatchContract`, `RecordContract`, `LearnContract`, `OutcomeSourceContract`, `BeliefStore`, `EventLog`, `AllocatorContract`, and `ProjectionContract` — that partition the system into independently testable and replaceable components. `CompileContract` is the new compile-phase interface: its single `compile(statement, source_event, provenance) → CompileResult | None` method turns a declarative statement into a typed constraint (a `CompileResult` bearing `form: CompileForm`, `predicate_expression`, `scope`, and optional `strength`), returning `None` if the statement is uncompilable and must remain flagged as free-text. The `DispatchKind` enum includes `ELICIT` alongside `REGENERATE`, `STOP`, and `REJECT_SCOPED`, authorising the dispatcher to request attribute elicitation from an operator. Changing a contract is a re-architecture event, not a refactor.

**Member registry** (`placer/core/member_registry.py`): Provides the only legal write path for T4 system parameters and T3 ensemble members. `set_param()` appends a `system.param_change` event before updating `system_params`; `register_member()` and `toggle_member()` append `system.member_toggle` events before modifying the `members` table. This event-first discipline ensures every comparison spanning a parameter change remains auditable during replay.

## Implementation — how it works

Core is intentionally dependency-free with respect to the rest of the Placer codebase. The module docstring declares this explicitly: "core imports nothing internal" — the one documented exception being `placer/core/resolver.py → beliefs.representations`, which is enforced by a conformance test rather than asserted by convention alone.

**Event storage model**: Events are persisted as a single append-only PostgreSQL table. Each row carries `seq` (DB-assigned integer sequence), `event_kind`, `order_id`, `entity_refs` (JSONB), `payload` (JSONB), `provenance` (JSONB), `observed_at`, and `recorded_at`. Bitemporality is load-bearing: `observed_at` records when something happened in the world; `recorded_at` (defaulted to `now()` by the DB) records when Placer learned of it. The `as_known_at` parameter on `query()` filters by `recorded_at`, enabling historical replay at any past knowledge state. Payload Pydantic models are validated at `append()` time; storage is raw JSONB.

**Belief resolver**: All resolver functions operate over `BeliefCheckpoint` objects (from `beliefs.representations`) and return `Belief` instances. Mixture outputs are never moment-matched away — spec §IV.3 requires that the mixture distribution be preserved as a `MixtureStats` attached to the `Belief._stats` slot. Precision weighting is membership-weight × `n_eff`. Conjugate `BetaStats` / `NormalStats` sufficient statistics flow through the tree backoff without lossy conversion until the final output, where `_collapse_to_parametric()` is used only when mixing requires it. Credible interval computation delegates to `scipy.stats.beta` and `scipy.stats.norm`; mixture quantiles are solved by bisection.

**Gate-aware factor decomposition**: `FactorBreakdown` now carries a `gates_pass: bool` flag (default `True`) and a `blocking_gate_ids: list[str]` field alongside the four probability factors. The `p_accept` property applies an indicator: `gates_pass_indicator × p_fit_residual × p_want × p_cap × p_resp`, where the residual fit factor (`p_fit_residual`) represents fit probability after hard gates have been evaluated. When `gates_pass` is `False`, `p_accept` is zero regardless of the factor values, and `blocking_gate_ids` records which gate instances caused the block.

**ID type safety**: All ID types are `NewType(str)` wrappers. There is no runtime enforcement — the safety guarantee is purely at the type-checker level, preventing callers from accidentally passing an `OrgId` where an `OrderId` is expected.

**Member registry write discipline**: `set_param()` and `register_member()` / `toggle_member()` call `core.events.append()` first and perform the SQL mutation second, within the same database connection. An unevented parameter change would corrupt every comparison spanning it during replay; the event-first ordering is the spec-mandated guard.

**Import direction enforcement**: A dedicated conformance test verifies that no core module (other than `placer/core/resolver.py`) imports from any non-core internal package. This makes core safe to load in isolation for schema inspection, migration tooling, and test fixtures.

## Availability — what is usable right now

All seven source files (`placer/core/ids.py`, `placer/core/events.py`, `placer/core/quantities.py`, `placer/core/resolver.py`, `placer/core/contracts.py`, `placer/core/member_registry.py`, `placer/core/predicates.py`) are present in the repository and constitute the active foundation of the platform. Every other Placer module imports from at least one of them; the codebase would not function without them.

- **`placer/core/ids.py`** — All ID types (including V2.1 additions `AttributeId` and `GateId`) and `Resolution` / `ResolutionConfidence` / `ResolutionMethod` models are confirmed present and consumed widely across identity, beliefs, adapters, members, and controllers.
- **`placer/core/events.py`** — The `EventKind` registry (60+ kinds, including V2.1 constraint gate lifecycle, review workflow, attribute identity, and elicitation decision/outcome events), all payload models, `FactorBreakdown` (with `gates_pass`, `p_fit_residual`, `blocking_gate_ids`, and the gate-indicator `p_accept` property), `ExplorationSlotPurpose` (now including `GATE_LEAK_TRIAL`), `Provenance`, `TrustTier`, and the `append()` / `query()` / `latest_seq()` functions are confirmed present. `EntityRefs` carries `attribute_id` and `gate_id` fields. The `MemberTogglePayload.to_status` field documents the lifecycle states `registered | shadow | live | retired`. The sacred-append invariant (single INSERT site) is enforced by a conformance test.
- **`placer/core/quantities.py`** — The `REGISTERED_QUANTITIES` dictionary (16 quantities, up from 12) and `IndexSchema` / `QuantityType` enums are confirmed present. The `attribute_org` and `member_context` index schemas, and the `p_attr`, `member_weight`, `proposer_coverage_credit`, and `member_calibration` quantities are new in V2.1 and imported by downstream belief modules.
- **`placer/core/predicates.py`** — The `PredicateId` enum (17 named predicates across item-attribute, org-attribute, geography, regulatory, and donor-sourced groups) is confirmed present, per spec V2.1 §IV.8.2. The V2.0 `RESTRICTION_CONFLICT` predicate is retained as a legacy alias. The module now also exports `CompileForm` (three closed compile forms per §IV.8.1), `GateInstance` (the runtime gate Pydantic model), `GateStatus` (four lifecycle states), and `SegmentStatus` (four segment lifecycle states). `GateInstance` depends on `GateId` from `placer/core/ids.py`; Pydantic is an additional runtime dependency.
- **`placer/core/resolver.py`** — All resolver functions are confirmed present. The `scipy.stats` dependency is used for credible interval and quantile computations; availability at runtime depends on `scipy` being installed in the deployment environment.
- **`placer/core/contracts.py`** — All abstract base classes are confirmed present, including the new `CompileContract` / `CompileResult` pair (compile-phase interface) and the `DispatchKind` enum which includes `ELICIT`. These are interfaces only; no concrete implementation lives in core itself.
- **`placer/core/member_registry.py`** — Evented write helpers for `system_params` and `members` tables are confirmed present. The `toggle_member()` function now accepts `MemberStatus` (renamed from `EnsembleMemberStatus`) from the learn types module; the on-disk status values are unchanged.

No access guards are declared on the `/placer/core` route. There is no changelog configured, so no pending-but-unmerged changes can be confirmed or denied from that source.
