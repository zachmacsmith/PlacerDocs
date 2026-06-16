---
feature: Members
group: New
first_commit: 6dc428c8cfbf577dc8254a42c8b1873db3babcd4
last_synced: '2026-06-15'
last_commit: 6dc428c8cfbf577dc8254a42c8b1873db3babcd4
anchors:
  tables: []
  endpoints: []
  types: []
  api_modules: []
  files:
  - placer/members/**
writes: []
reads:
- placer/members/__init__.py
- placer/members/proposers/__init__.py
- placer/members/proposers/bridge_battery.py
- placer/members/proposers/retrieval.py
---
## Capability — what it can do

The Members feature defines and manages the **T3 extensible member pool** — the governed registry of all participants in Placer's Bayesian ensemble (spec V2 §VIII.3). The pool holds four member kinds (`MemberType`): **proposers**, **predictors**, **folds**, and **controllers**, each represented as an `EnsembleMember` record.

Three operations on the pool are provided by `placer/core/member_registry.py`:

- **`list_members`** — queries the `members` table, optionally filtered by `MemberType`, returning a typed list of `EnsembleMember` objects ordered by registration time.
- **`register_member`** — admits a new member into the pool via an evented insert: a `system.member_toggle` event is appended to the event spine first, then the `members` row is written referencing it.
- **`toggle_member`** — transitions an existing member between statuses (`active`, `suspended`, `retired`) via the same evented pattern: event first, row update second.

Within the `placer/members/` package, `placer/members/proposers/bridge_battery.py` defines the **bridge battery** — the founding set of proposer members used by the generation pipeline. It enumerates five bridge families (`BridgeFamily`), twenty bridge mediators (`BridgeMediator`), and their groupings, along with a four-mediator always-on `CORE_BATTERY`, five conditional trigger rules (`ConditionalTrigger`), and six elicitation methods (`ElicitationMethod`). Output types for the generation stage (`Hypothesis`, `HypothesisProvenance`, `SegmentAssignment`, `MintProposal`, `CanonicalizeResult`) are also declared there.

## Implementation — how it works

**Governance invariant.** The spec mandates that T3 member-pool changes may only occur through evented writes (spec V2 §VIII.3). `member_registry.py` enforces this by calling `placer.core.events.append` (the sacred single INSERT site) before any `members`-table mutation. Every registration and toggle is therefore fully represented in the append-only `events` table, enabling deterministic replay and historical auditing.

**Event kinds.** Both `register_member` and `toggle_member` emit `EventKind.SYSTEM_MEMBER_TOGGLE` (`"system.member_toggle"`), validated at append time against the `MemberTogglePayload` schema (`member_id`, `from_status`, `to_status`, `actor`, `reason`). Payload validation is enforced inside `events.append` via the `EVENT_PAYLOAD_MODELS` registry.

**Database schema.** The `members` table stores: `member_id`, `member_type`, `owner`, `estimand_conformity` (the quantity a predictor or fold targets), `prior_weight`, `current_weight`, `version`, `status`, `sunset_rule`, `registered_at`, and `last_updated`. Status transitions are done with a bare `UPDATE`; the event record is the authoritative history.

**Proposer structure.** `placer/members/proposers/bridge_battery.py` enumerates the 20 mediators across 5 families and declares `MEDIATOR_TO_FAMILY` for O(1) lookup. Conditional triggers allow additional mediators to activate based on item properties (e.g. `cas_present`, `high_volume`). The `Hypothesis` type carries the generation output including provenance, bridge identity, claimed generality level (`ClaimedGenerality`), segment assignments, and optional inherited priors. `MintProposal` and `CanonicalizeResult` represent the downstream canonicalization step.

**Separation of concerns.** The `placer/members/**` package owns proposer type definitions; operational CRUD against the pool lives in a core registry module; the `EnsembleMember` and `MemberType` types are defined in a shared learn-types module shared across the learn and governance subsystems. A sibling module (`placer/members/proposers/retrieval.py`) declares retrieval-stream types (`RetrievalStream`, `RetrievalResult`, `FusionResult`, `ResolutionDiagnostics`) used by the resolution pipeline.

## Availability — is it usable right now

The route `/placer/members` is registered in the feature table and the package root `placer/members/__init__.py` exists, but its body contains only a docstring (`"""T3 — extensible member pool (spec V2 §VIII.3)."""`). No route handler, view component, or controller logic was found in the component file or any file directly routing to `/placer/members`.

The **backend registry logic** (`list_members`, `register_member`, `toggle_member`) is fully implemented and internally consistent. The **bridge battery type definitions** in `placer/members/proposers/bridge_battery.py` are complete and consumed by the generation pipeline. The **retrieval-stream type definitions** in `placer/members/proposers/retrieval.py` are similarly complete (renamed from their prior location in the resolve subsystem).

However, no API endpoint, HTTP route handler, or frontend component was found that exposes member-pool management to callers at the `/placer/members` path. Code presence does not confirm user-facing availability. The feature should be treated as **infrastructure-complete but interface-absent** — the pool can be queried and mutated programmatically, but no exposed endpoint or UI surface for doing so was confirmed.
