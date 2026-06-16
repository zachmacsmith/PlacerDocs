---
feature: Members
group: New
first_commit: 6dc428c8cfbf577dc8254a42c8b1873db3babcd4
last_synced: '2026-06-16'
last_commit: e75bcb47650c6c56370cc10193be5f08ae3490e8
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
- placer/members/compilers/__init__.py
- placer/members/compilers/incumbent_rules.py
- placer/members/folds/__init__.py
- placer/members/predictors/__init__.py
- placer/members/proposers/__init__.py
- placer/members/proposers/bridge_battery.py
- placer/members/proposers/incumbent.py
- placer/members/proposers/retrieval.py
---
## Capability — what it can do

The Members feature defines and manages the **T3 extensible member pool** — the governed registry of all participants in Placer's Bayesian ensemble (spec V2 §VIII.3). The pool holds four member kinds (`MemberType`): **proposers**, **predictors**, **folds**, and **controllers**, each represented as an `EnsembleMember` record.

Three operations on the pool are provided by `placer/core/member_registry.py`:

- **`list_members`** — queries the `members` table, optionally filtered by `MemberType`, returning a typed list of `EnsembleMember` objects ordered by registration time.
- **`register_member`** — admits a new member into the pool via an evented insert: a `system.member_toggle` event is appended to the event spine first, then the `members` row is written referencing it.
- **`toggle_member`** — transitions an existing member between statuses (`active`, `suspended`, `retired`) via the same evented pattern: event first, row update second.

Within the `placer/members/` package, `placer/members/proposers/bridge_battery.py` defines the **bridge battery** — the founding set of proposer members used by the generation pipeline. It enumerates five bridge families (`BridgeFamily`), twenty bridge mediators (`BridgeMediator`), and their groupings, along with a four-mediator always-on `CORE_BATTERY`, five conditional trigger rules (`ConditionalTrigger`), and six elicitation methods (`ElicitationMethod`). Output types for the generation stage (`Hypothesis`, `HypothesisProvenance`, `SegmentAssignment`, `MintProposal`, `CanonicalizeResult`) are also declared there.

A second proposer, `placer/members/proposers/incumbent.py`, implements the **incumbent proposer** (M1 §3): it reads charities that Simpli's mission-match system previously approved on similar orders, resolves them into Placer org space, and surfaces them as `Candidate` records. This provides Placer with real, history-backed candidates immediately without requiring its own generation pipeline.

A new **compilers subpackage** (`placer/members/compilers/`) houses tools that translate the incumbent system's implicit knowledge into Placer's constraint model. `placer/members/compilers/incumbent_rules.py` mines eight hardcoded restriction rules from Simpli's mission-match worker (M1 §6) — covering hazmat certification, refrigeration, geographic distance, regulatory blocks, lot-size minimums, and item-policy restrictions (alcohol, tobacco, expired goods) — and emits each as a `constraint.gate_compiled` event at `human_confirmed` provenance tier, seeding the `gate_instances` table.

## Implementation — how it works

**Governance invariant.** The spec mandates that T3 member-pool changes may only occur through evented writes (spec V2 §VIII.3). `member_registry.py` enforces this by calling `placer.core.events.append` (the sacred single INSERT site) before any `members`-table mutation. Every registration and toggle is therefore fully represented in the append-only `events` table, enabling deterministic replay and historical auditing.

**Event kinds.** Both `register_member` and `toggle_member` emit `EventKind.SYSTEM_MEMBER_TOGGLE` (`"system.member_toggle"`), validated at append time against the `MemberTogglePayload` schema (`member_id`, `from_status`, `to_status`, `actor`, `reason`). Payload validation is enforced inside `events.append` via the `EVENT_PAYLOAD_MODELS` registry.

**Database schema.** The `members` table stores: `member_id`, `member_type`, `owner`, `estimand_conformity` (the quantity a predictor or fold targets), `prior_weight`, `current_weight`, `version`, `status`, `sunset_rule`, `registered_at`, and `last_updated`. Status transitions are done with a bare `UPDATE`; the event record is the authoritative history.

**Proposer structure.** `placer/members/proposers/bridge_battery.py` enumerates the 20 mediators across 5 families and declares `MEDIATOR_TO_FAMILY` for O(1) lookup. Conditional triggers allow additional mediators to activate based on item properties (e.g. `cas_present`, `high_volume`). The `Hypothesis` type carries the generation output including provenance, bridge identity, claimed generality level (`ClaimedGenerality`), segment assignments, and optional inherited priors. `MintProposal` and `CanonicalizeResult` represent the downstream canonicalization step.

**Incumbent proposer.** `placer/members/proposers/incumbent.py` queries Simpli's `mission_match_resolved` table for charities approved on past orders sharing the same donor (`company_id`), resolves each EIN against Placer's org identity store (minting a provisional org record if none exists), and returns a ranked list of `Candidate` objects tagged with `surfaced_by=incumbent_match`. It is idempotent: the same EIN surfaced multiple times yields a single `Candidate` via `DISTINCT ON`.

**Incumbent rule compiler.** `placer/members/compilers/incumbent_rules.py` is a one-shot script (`python -m placer.members.compilers.incumbent_rules`) that iterates over eight `INCUMBENT_RULES` entries, skips any whose `gate_id` already exists in `gate_instances`, and for each new rule emits a `constraint.gate_compiled` event (via `placer.core.events.append`) then inserts a row into `gate_instances` with `compile_form=HARD_GATE`, `epsilon=0.0`, and `status=active`. The provenance tier is `human_confirmed`, enabling downstream contradiction detection and ε-leak relaxation as evidence accumulates.

**Separation of concerns.** The `placer/members/**` package owns proposer type definitions and the compilers tooling; operational CRUD against the pool lives in a core registry module; the `EnsembleMember` and `MemberType` types are defined in a shared learn-types module shared across the learn and governance subsystems. A sibling module (`placer/members/proposers/retrieval.py`) declares retrieval-stream types (`RetrievalStream`, `RetrievalResult`, `FusionResult`, `ResolutionDiagnostics`) used by the resolution pipeline. The `placer/members/folds/` and `placer/members/predictors/` subpackages are scaffolded (empty `__init__.py`) but contain no implementation yet.

## Availability — is it usable right now

The route `/placer/members` is registered in the feature table and the package root `placer/members/__init__.py` exists, but its body contains only a docstring (`"""T3 — extensible member pool (spec V2 §VIII.3)."""`). No route handler, view component, or controller logic was found in the component file or any file directly routing to `/placer/members`.

The **backend registry logic** (`list_members`, `register_member`, `toggle_member`) is fully implemented and internally consistent. The **bridge battery type definitions** in `placer/members/proposers/bridge_battery.py` are complete and consumed by the generation pipeline. The **retrieval-stream type definitions** in `placer/members/proposers/retrieval.py` are similarly complete.

The **incumbent proposer** (`placer/members/proposers/incumbent.py`) is implemented and callable as a library function; it has no HTTP endpoint and is not wired to any route handler. The **incumbent rule compiler** (`placer/members/compilers/incumbent_rules.py`) is a runnable script that seeds the `gate_instances` table; it is designed for one-shot administrative execution, not for ongoing user-facing requests.

The `placer/members/folds/` and `placer/members/predictors/` subpackages exist as empty scaffolding with no implementation.

No API endpoint, HTTP route handler, or frontend component was found that exposes member-pool management to callers at the `/placer/members` path. Code presence does not confirm user-facing availability. The feature should be treated as **infrastructure-complete but interface-absent** — the pool and its associated tooling can be used programmatically, but no exposed endpoint or UI surface for doing so was confirmed.
