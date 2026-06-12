---
feature: Value
group: Placer
first_commit: c66cac868aea2ad3d45474337649f15a7c7db058
last_synced: '2026-06-12'
last_commit: c66cac868aea2ad3d45474337649f15a7c7db058
anchors:
  tables: []
  endpoints: []
  types:
  - ContactCost
  - FactorBreakdown
  - Valuation
  - ValuationSnapshot
  - ValuationSnapshotPayload
  - ValueContract
  api_modules:
  - placer.events
  - placer/contracts.py
  - placer/value/types.py
  files:
  - placer/contracts.py::Valuation
  - placer/contracts.py::ValueContract
  - placer/value/**
  - placer/value/types.py
writes: []
reads:
- placer/candidates/types.py
- placer/contracts.py
- placer/events/types.py
- placer/projections/types.py
- placer/value/types.py
---
## Capability — what it can do

The Value feature defines the **pure valuation function** for the placement controller: given a candidate organisation, the current order world-state, and the belief store, it produces a fully-decomposed `Valuation` that tells the controller exactly how much it is worth contacting that candidate.

Two types are provided in `placer/value/types.py`:

**`Valuation`** — the mandatory output of the Value operator. It carries:
- `factor_breakdown` (`FactorBreakdown`) — the four-factor decomposition `P(accept) = p_fit × p_want × p_cap × p_resp`, with `want_provenance` (tracing whether the want signal came from an edge posterior, self-consistency, evidence, or prior) and a `cap_interval`.
- `emc` (Expected Matched Contribution) — `P(accept) × E[min(C, V_r)] × u`, the value of a single contact attempt in matched-volume units.
- `contact_index` (`ι`) — `(EMC − insertion_cost) / contact_cost`; the dimensionless score the dispatch layer ranks candidates by.
- `insertion_cost` — optional cheapest-insertion cost sourced from the route allocator.
- `contact_cost` — the total cost of one contact attempt.
- `intervals` — a dictionary of named confidence intervals (used by Thompson-sampling and UCB exploration).
- `sd` — optional standard deviation field for Thompson/UCB sampling.

**`ContactCost`** — models the cost formula `c_i = c_raw + λ_ops × h_i` (spec §6.1), where `c_raw` is the raw monetary contact cost, `λ_ops` is the shadow price of one ops-hour, and `h_i` is the expected ops-hours consumed. The `.total` property evaluates the sum.

The formal contract for the Value operator is declared in `placer/contracts.py` as `ValueContract`, which specifies that `value(candidate, order_state, beliefs) → Valuation` must be a **pure function** — no side effects, no mutations. `factor_breakdown` is contractually mandatory in every output.

## Implementation — how it works

Value is deliberately **type-only** at the `placer/value/` level. `placer/value/types.py` contains the output schema (`Valuation`, `ContactCost`) and imports `FactorBreakdown` from `placer/events/types.py`. No computation lives in this module; implementations must satisfy `ValueContract` (defined in `placer/contracts.py`).

**Data flow through the wider system:**

1. `FactorBreakdown` (defined in `placer/events/types.py`) is the shared type that carries `p_fit`, `p_want`, `p_cap`, `p_resp`, `want_provenance`, and `cap_interval`. It is imported by Value types, the contracts module, the candidates store, and the projections layer — making it the authoritative decomposition object across the pipeline.

2. Once a `Valuation` is computed, its fields are mirrored into two downstream frozen records:
   - `ValuationSnapshot` (`placer/candidates/types.py`) — frozen onto each `Candidate` record at decision time; includes `sd` and `exploration_slot`.
   - `ValuationSnapshotPayload` (`placer/events/types.py`) — written to the event spine as a `decision.valuation_snapshot` event, providing a bitemporal audit trail.

3. The `contact_index` (`ι`) field produced by `Valuation` flows into `RankedRecommendation` (`placer/projections/types.py`), where it is used alongside `factor_breakdown` and `emc` for the rank projection surfaced to the operator.

4. `ContactCost` encapsulates the allocator's shadow-pricing formula. The `lambda_ops` scalar (shadow price of one ops-hour) is also carried in `DispatchPayload`, ensuring the dispatch layer and the valuation layer agree on the same marginal cost signal.

The `ValueContract` abstract base class in `placer/contracts.py` declares the pure-function boundary explicitly: any implementation must accept `(Candidate, OrderWorldState, BeliefStore)` and return a `Valuation` synchronously, with no async I/O permitted by the signature.

## Availability — is it usable right now

`placer/value/types.py` is present in the codebase and is imported transitively by the candidates store, projections layer, and contracts module. The **type definitions** are fully available and actively used by surrounding features.

The **`ValueContract`** abstract interface is defined in `placer/contracts.py`, but no concrete implementation class satisfying `ValueContract` was found in the codebase via search. The valuation computation itself — the logic that maps `(candidate, order_state, beliefs)` to a populated `Valuation` — does not appear to have a discoverable implementation file at this time. Code presence of the types and contract does not confirm that a live, callable valuation function is wired into the controller loop.

`ContactCost` is defined but not referenced outside `placer/value/types.py`; whether it is instantiated by an allocator implementation is not confirmed by available code.

The route `/placer/value` appears in the feature table but no route handler, API endpoint, or view component serving that path was found during investigation. The route's backend behaviour and any UI exposure are currently unconfirmed.
