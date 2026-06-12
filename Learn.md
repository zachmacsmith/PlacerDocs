---
feature: Learn
group: Placer
first_commit: c66cac868aea2ad3d45474337649f15a7c7db058
last_synced: '2026-06-12'
last_commit: c66cac868aea2ad3d45474337649f15a7c7db058
anchors:
  tables:
  - members
  - system_params
  endpoints: []
  types:
  - CalibrationRecord
  - EnsembleMember
  - EnsembleMemberStatus
  - FoldDirection
  - GateVerdict
  - MemberType
  - ProposerCoverageRecord
  - Route
  - StandingGate
  api_modules:
  - placer.contracts
  - placer.governance.store
  - placer.learn.routing
  - placer.learn.types
  files:
  - placer/contracts.py::LearnContract
  - placer/governance/store.py::list_members
  - placer/governance/store.py::register_member
  - placer/governance/store.py::set_param
  - placer/governance/store.py::toggle_member
  - placer/learn/**
  - placer/learn/routing.py
  - placer/learn/types.py
writes:
- placer/governance/store.py::register_member
- placer/governance/store.py::set_param
- placer/governance/store.py::toggle_member
reads:
- placer/contracts.py::LearnContract
- placer/governance/store.py::list_members
- placer/learn/routing.py
- placer/learn/types.py
---
## Capability — what it can do

The Learn feature is the system's **Bayesian update and validation harness** (spec §8). It provides four interlocking capabilities:

**1. Incremental fold running.** `LearnContract.fold(since_watermark)` performs an incremental conjugate update over events that have arrived since a recorded watermark — O(new events) complexity, not O(all events). The routing table in `placer/learn/routing.py` maps each outcome event kind and disposition reason code to a `(quantity_id, FoldDirection)` pair, so every belief update is a traceable, factor-routed label: approval/rejection routes to `p_donor_approve` (donor-side); acceptance routes to `p_want`, `p_fit`, `p_resp`, and `cap_volume` (charity-side). Capacity-code declines are treated as want-positives (`FoldDirection.SUCCESS` on `p_want`), never want-negatives.

**2. Standing validation gates.** Five `StandingGate` values are defined in `types.py`: `baseline`, `known_item_probe`, `transfer_tripwire`, `interval_coverage`, and `calibration_eprocess`. Gates are evaluated via `LearnContract.evaluate_gates()`, which returns a list of `GateVerdict` records (challenger, incumbent, metric, dataset, value, threshold, passed). The calibration e-process gate (`CALIBRATION_EPROCESS`) runs on the model's own logged predictions only; a firing revokes continuation rights.

**3. Predictor ensemble member registry.** `EnsembleMember` (backed by the `members` DB table) tracks the four governed pool-member kinds: `proposer`, `predictor`, `fold`, and `controller`. Each member carries prior and current weights, version, status (`active` / `suspended` / `retired`), and an optional sunset rule. The governance store (`placer/governance/store.py`) is the **only legal write path** — every registration or status toggle is event-appended first (`SYSTEM_MEMBER_TOGGLE`) and then the derived row is updated referencing that event, making every weight-affecting change auditable.

**4. Calibration monitoring and proposer coverage tracking.** `CalibrationRecord` pairs each logged prediction (quantity, predicted value, realized value) with its prediction and resolution timestamps, accommodating censored (not-yet-resolved) outcomes. `ProposerCoverageRecord` tracks per-proposer placement surface area (surfaced, unique, marginal coverage) by bridge family and UNSPSC family, enabling the vocabulary-health metric (V2 §II) that monitors the rate of `OTHER` reason codes.

**Selector coverage assertion.** `assert_selector_coverage()` in `routing.py` verifies that every `EventKind` is either claimed by at least one route or explicitly listed in `ORPHANED_EVENT_KINDS`. This ships with the fold runner as a nightly assertion (V2 §VII.3).


## Implementation — how it works

**Type layer (`placer/learn/types.py`).** All domain types are Pydantic `BaseModel` or `StrEnum`. The module defines: `GateVerdict`, `StandingGate`, `MemberType`, `EnsembleMemberStatus`, `EnsembleMember`, `CalibrationRecord`, and `ProposerCoverageRecord`. These are shared across the governance store and the contracts layer — no logic lives here, only schema.

**Contract layer (`placer/contracts.py`).** `LearnContract` is an abstract base class with two abstract methods: `fold(since_watermark: EventSeq) -> list[BeliefCheckpoint]` and `evaluate_gates() -> list[GateVerdict]`. The `GateVerdict` type in contracts is a slightly narrower variant (omits `dataset` and `evaluated_at` fields present in the `learn/types.py` version) — concrete implementations bridge the two.

**Routing table (`placer/learn/routing.py`).** A statically declared dict `EVENT_ROUTING` maps `EventKind` → `list[Route]` and `REASON_CODE_ROUTING` maps `ReasonCode` → `list[Route]`. A `Route` is a frozen dataclass of `(quantity_id, FoldDirection, note)`. The `routes_for(event)` function dispatches disposition events through the reason-code table and all others through the event-kind table. `FoldDirection` has four values: `success` (Beta α+1), `failure` (Beta β+1), `observation` (continuous value folds in), and `censoring` (cure-model censoring update — never a step at timeout).

**Governance store (`placer/governance/store.py`).** Provides `list_members`, `register_member`, and `toggle_member` as the only legal write path to the `members` table. Writes are evented: `SYSTEM_MEMBER_TOGGLE` is appended to the event spine first, then the row is mutated with the referencing event sequence number. Parameter changes follow the same pattern via `SYSTEM_PARAM_CHANGE` on the `system_params` table (T4). This ensures every model-weight or system-parameter change is fully auditable and bitemporally comparable (spec V2 §VIII.3).

**Orphaned event kinds.** All non-`outcome.*` event kinds are placed in `ORPHANED_EVENT_KINDS` — they are consumed by audit and learning infrastructure, but their rate (particularly `OTHER` reason codes) feeds the vocabulary-health metric rather than individual belief updates.


## Availability — is it usable right now

**Types and routing table:** `placer/learn/types.py` and `placer/learn/routing.py` are present in the codebase. The type definitions and routing logic are available for use by any concrete fold-runner implementation.

**`LearnContract` abstract interface:** Defined in `placer/contracts.py`. No concrete implementation file of this contract was found in the codebase during investigation — only the abstract base class exists. Fold running and gate evaluation are therefore **specified but not concretely implemented** in the confirmed source.

**`placer/learn/routing.py`:** Referenced in `scripts/synthetic_env.py` (a synthetic environment script), confirming the routing module is importable and used in at least the synthetic test harness.

**Governance store writes:** `placer/governance/store.py` is present and uses `EnsembleMember` types from `learn/types.py`. Member registration, listing, and toggling are implemented functions backed by the `members` (T3) and `system_params` (T4) database tables.

**Route `/placer/learn`:** Listed in the feature table with no guards. No request handler, view template, or API endpoint for this route was found during investigation — the route is registered but its handler implementation was not confirmed in source.

**Changelog:** No changelog is configured; no v1 ship-status claims can be verified against release history. The module docstring mentions "Ships in v1: fold runners, calibration monitor scaffold, gate framework" — this is intent context only, not a confirmed deployment state.

