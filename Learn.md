---
feature: Learn
group: Placer
first_commit: c66cac868aea2ad3d45474337649f15a7c7db058
last_synced: '2026-06-15'
last_commit: 7fdd634f38a1f90503ccc57e81634980b0f2102f
anchors:
  tables: []
  endpoints: []
  types:
  - CalibrationRecord
  - EnsembleMember
  - EnsembleMemberStatus
  - GateVerdict
  - MemberStatus
  - MemberType
  - ProposerCoverageRecord
  - StandingGate
  api_modules: []
  files:
  - placer/learn/**
writes: []
reads:
- placer/learn/types.py
---
## Capability — what it can do

The Learn feature is the system's **Bayesian update and validation harness** (spec §8). It provides four interlocking capabilities:

**1. Incremental fold running.** `LearnContract.fold(since_watermark)` performs an incremental conjugate update over events that have arrived since a recorded watermark — O(new events) complexity, not O(all events). The routing table in `placer/beliefs/fold_runner.py` maps each outcome event kind and disposition reason code to a `(quantity_id, FoldDirection)` pair, so every belief update is a traceable, factor-routed label: approval/rejection routes to `p_donor_approve` (donor-side); acceptance routes to `p_want`, `p_fit`, `p_resp`, and `cap_volume` (charity-side). Capacity-code declines are treated as want-positives (`FoldDirection.SUCCESS` on `p_want`), never want-negatives.

**2. Standing validation gates.** Five `StandingGate` values are defined in `types.py`: `baseline`, `known_item_probe`, `transfer_tripwire`, `interval_coverage`, and `calibration_eprocess`. Gates are evaluated via `LearnContract.evaluate_gates()`, which returns a list of `GateVerdict` records (challenger, incumbent, metric, dataset, value, threshold, passed). The calibration e-process gate (`CALIBRATION_EPROCESS`) runs on the model's own logged predictions only; a firing revokes continuation rights.

**3. Predictor ensemble member registry.** `EnsembleMember` (backed by the `members` DB table) tracks the five governed pool-member kinds: `proposer`, `predictor`, `fold`, `controller`, and `compiler` (added in spec V2.1; compilers convert declarative statements into typed constraints). Each member carries prior and current weights, version, status (`registered` → `shadow` → `live` → `retired`), and an optional sunset rule. Shadow members predict on everything but serve nothing — the mandatory evaluation period before promotion. Proposer promotion from shadow to live requires human disposition. The member registry (`placer/core/member_registry.py`) is the **only legal write path** — every registration or status toggle is event-appended first (`SYSTEM_MEMBER_TOGGLE`) and then the derived row is updated referencing that event, making every weight-affecting change auditable.

**4. Calibration monitoring and proposer coverage tracking.** `CalibrationRecord` pairs each logged prediction (quantity, predicted value, realized value) with its prediction and resolution timestamps, accommodating censored (not-yet-resolved) outcomes. `ProposerCoverageRecord` tracks per-proposer placement surface area (surfaced, unique, marginal coverage) by bridge family and UNSPSC family, enabling the vocabulary-health metric (V2 §II) that monitors the rate of `OTHER` reason codes.

**Selector coverage assertion.** `assert_selector_coverage()` in `placer/beliefs/fold_runner.py` verifies that every `EventKind` is either claimed by at least one route or explicitly listed in `ORPHANED_EVENT_KINDS`. This ships with the fold runner as a nightly assertion (V2 §VII.3).


## Implementation — how it works

**Type layer (`placer/learn/types.py`).** All domain types are Pydantic `BaseModel` or `StrEnum`. The module defines: `GateVerdict`, `StandingGate`, `MemberType`, `MemberStatus` (with `EnsembleMemberStatus` retained as a backward-compat alias pending migration), `EnsembleMember`, `CalibrationRecord`, and `ProposerCoverageRecord`. These are shared across the member registry and the contracts layer — no logic lives here, only schema.

**Contract layer (`placer/core/contracts.py`).** `LearnContract` is an abstract base class with two abstract methods: `fold(since_watermark: EventSeq) -> list[BeliefCheckpoint]` and `evaluate_gates() -> list[GateVerdict]`. The `GateVerdict` type in contracts is a slightly narrower variant (omits `dataset` and `evaluated_at` fields present in the full learn-types definition) — concrete implementations bridge the two.

**Routing table (`placer/beliefs/fold_runner.py`).** A statically declared dict `EVENT_ROUTING` maps `EventKind` → `list[Route]` and `REASON_CODE_ROUTING` maps `ReasonCode` → `list[Route]`. A `Route` is a frozen dataclass of `(quantity_id, FoldDirection, note)`. The `routes_for(event)` function dispatches disposition events through the reason-code table and all others through the event-kind table. `FoldDirection` has four values: `success` (Beta α+1), `failure` (Beta β+1), `observation` (continuous value folds in), and `censoring` (cure-model censoring update — never a step at timeout).

**Member registry (`placer/core/member_registry.py`).** Provides `list_members`, `register_member`, and `toggle_member` as the only legal write path to the `members` table. Writes are evented: `SYSTEM_MEMBER_TOGGLE` is appended to the event spine first, then the row is mutated with the referencing event sequence number. Parameter changes follow the same pattern via `SYSTEM_PARAM_CHANGE` on the `system_params` table (T4) through `set_param`. This ensures every model-weight or system-parameter change is fully auditable and bitemporally comparable (spec V2 §VIII.3).

**Orphaned event kinds.** All non-`outcome.*` event kinds are placed in `ORPHANED_EVENT_KINDS` — they are consumed by audit and learning infrastructure, but their rate (particularly `OTHER` reason codes) feeds the vocabulary-health metric rather than individual belief updates.


## Availability — is it usable right now

**Types (`placer/learn/**`):** Present in the codebase. Defines `GateVerdict`, `StandingGate`, `MemberType`, `MemberStatus` (with `EnsembleMemberStatus` as a backward-compat alias), `EnsembleMember`, `CalibrationRecord`, and `ProposerCoverageRecord` — all confirmed by direct file read. `MemberType` now enumerates five kinds: `proposer`, `predictor`, `fold`, `controller`, and `compiler` (V2.1). `MemberStatus` defines a four-stage lifecycle (`registered`, `shadow`, `live`, `retired`), replacing the prior two-state `EnsembleMemberStatus`; the alias is present but flagged for removal once all call-sites migrate. Actively imported by the member registry module, confirming these types are in live use.

**Routing table and fold utilities (`placer/beliefs/fold_runner.py`):** Present. Contains `EVENT_ROUTING`, `REASON_CODE_ROUTING`, `FoldDirection`, `Route`, `routes_for`, and `assert_selector_coverage`. Imported by the synthetic adapter and the golden-replay test suite — the routing module is importable and exercised by at least the synthetic harness and test infrastructure. This file is an external dependency of the Learn feature; it lives outside the `placer/learn/` subtree but was confirmed by source search.

**`LearnContract` abstract interface (`placer/core/contracts.py`):** Defined as an abstract base class. No concrete implementation of this contract was found in the codebase during investigation — fold running and gate evaluation are therefore **specified but not concretely implemented** in the confirmed source. This file is an external dependency of the Learn feature; it lives outside the `placer/learn/` subtree but was confirmed by source search.

**Member registry (`placer/core/member_registry.py`):** Present. `list_members`, `register_member`, `toggle_member`, and `set_param` are implemented functions backed by the `members` (T3) and `system_params` (T4) database tables. This file is an external dependency of the Learn feature; it lives outside the `placer/learn/` subtree but was confirmed by source search.

**Route `/placer/learn`:** Listed in the feature table with no guards. No request handler, view template, or API endpoint for this route was found during investigation — the route is registered but its handler implementation was not confirmed in source.

**Changelog:** No changelog is configured; no v1 ship-status claims can be verified against release history. The module docstring mentions "Ships in v1: fold runners, calibration monitor scaffold, gate framework" — this is intent context only, not a confirmed deployment state.
