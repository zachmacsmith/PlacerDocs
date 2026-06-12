---
feature: Generate
group: Placer
last_synced: '2026-06-12'
last_commit: c66cac868aea2ad3d45474337649f15a7c7db058
anchors:
  tables: []
  endpoints: []
  types:
  - BridgeFamily
  - BridgeMediator
  - CanonicalizeResult
  - ClaimedGenerality
  - ConditionalTrigger
  - ElicitationMethod
  - Hypothesis
  - HypothesisProvenance
  - MintProposal
  - SegmentAssignment
  api_modules:
  - placer.identity
  - placer/contracts.py::CanonicalizeContract
  - placer/contracts.py::GenerateContract
  - placer/contracts.py::Hypothesis
  - placer/contracts.py::HypothesisProvenance
  - placer/contracts.py::MintProposal
  - placer/events/types.py::GeneratedHypothesis
  - placer/events/types.py::InferenceGenerationPayload
  files:
  - placer/generate/**
  - placer/generate/types.py
writes: []
reads:
- placer/contracts.py
- placer/events/types.py
- placer/generate/types.py
- placer/identity/types.py
---
## Capability — what it can do

The Generate feature defines the full type vocabulary for **hypothesis generation** — the pipeline stage that converts a donated-item description into a ranked list of recipient-segment hypotheses, each annotated with placement intent and provenance.

It provides three distinct layers of vocabulary:

**Bridge taxonomy.** Twenty `BridgeMediator` values (e.g. `use_function`, `beneficiary_population`, `venue_setting`) are organised into five `BridgeFamily` groups — Consumption, People, Context, Institutional, and Analogical — via the `MEDIATOR_TO_FAMILY` lookup table. This taxonomy is the semantic vocabulary the generator uses to structure its reasoning about why a recipient organisation might want a given item.

**Battery selection.** `CORE_BATTERY` names four mediators that fire unconditionally on every item: `use_function`, `activity_program`, `beneficiary_population`, and `venue_setting`. A separate `CONDITIONAL_TRIGGERS` list (five rules keyed on conditions such as `cas_present`, `brand_value_consumer`, `raw_intermediate`, `regulated_perishable`, and `high_volume`) adds further mediators when product-identity signals are present.

**Elicitation methods.** Six `ElicitationMethod` values (`direct_generation`, `persona_simulation`, `taxonomy_sweep`, `inversion`, `abundance_counterfactual`, `negative_generation`) parameterise how a mediator prompt is posed to the underlying model.

**Hypothesis structure.** The `Hypothesis` model carries the generated text, soft attributes, hard constraints, segment weight assignments (`SegmentAssignment`), an optional `inherited_prior` (used in multi-level generation), and a `HypothesisProvenance` record. Provenance includes the bridge family, mediator, elicitation method, rank, self-consistency frequency, a `tree_level` (0 = item description, 1+ = ancestor descriptions per V2 §V), and a `ClaimedGenerality` (`item`, `family`, or `class`). Claimed generality seeds priors at the claimed level; evidence always lands at the most specific resolved level.

**Canonicalisation result.** `CanonicalizeResult` represents the output of routing a hypothesis to existing segments: a list of `SegmentAssignment` weights plus an optional `MintProposal` (definition text + source hypothesis IDs) when no suitable existing segment matches.

## Implementation — how it works

`placer/generate/types.py` is a pure-types module with no I/O: it imports only `pydantic`, `enum.StrEnum`, and `HypothesisId`/`SegmentId` from `placer.identity.types`, then declares all structs, enumerations, and module-level constants used by downstream operator implementations.

**Operator contract.** `placer/contracts.py` defines `GenerateContract`, the abstract interface that a concrete generator must implement. Its single abstract method `generate(item, constraints, beliefs) → list[Hypothesis]` is documented as "consults beliefs first (lookup-before-generate as edges accrete)", meaning the generator is expected to check the `BeliefStore` for an existing crosswalk edge before invoking a model. The contract also includes `CanonicalizeContract` (maps a hypothesis to segment assignments or a mint proposal) as a sister operator in the same pipeline stage.

**Event spine integration.** `placer/events/types.py` defines `InferenceGenerationPayload` and its embedded `GeneratedHypothesis` model. Each generation run that reaches the event spine is recorded under `EventKind.INFERENCE_GENERATION`, carrying model version, prompt version, bridge ID, per-hypothesis data, and token count. This is the audit record that `Learn` folds into belief checkpoints.

**Parallel type definitions.** `placer/contracts.py` contains its own `Hypothesis`, `SegmentAssignment`, `HypothesisProvenance`, and `MintProposal` models that mirror those in `placer/generate/types.py` but are typed more loosely (bridge family/mediator stored as plain `str` rather than `StrEnum`). The contracts file is the frozen operator interface; the generate types file is the richer internal representation. Callers within the generate subsystem use the typed enumerations; the contract layer uses string fields for cross-boundary stability.

## Availability — is it usable right now

`placer/generate/types.py` is present in the codebase and importable. The route `/placer/generate` appears in the feature table with no guards, indicating no authentication barrier is declared at the routing layer.

However, `GenerateContract` in `placer/contracts.py` is an abstract base class with no concrete implementation found in the searched codebase. No file was found importing `placer.generate.types` outside of the contracts and events modules, and no concrete `generate()` handler was located. **The type vocabulary and abstract contract are defined; a runtime implementation that executes hypothesis generation has not been confirmed in the code.** Whether the `/placer/generate` route returns generated hypotheses in a deployed environment cannot be verified from the current source state alone.

No changelog is configured, so there is no release-intent context to reconcile against.
