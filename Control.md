---
feature: Control
group: Placer
last_synced: '2026-06-12'
last_commit: c66cac868aea2ad3d45474337649f15a7c7db058
anchors:
  tables: []
  endpoints: []
  types:
  - AllocatorState
  - ControllerAction
  - OrderId
  - OrderWorldState
  - StepResult
  - Trigger
  - TriggerKind
  api_modules:
  - placer.candidates
  - placer.candidates.types
  - placer.contracts
  - placer.control.types
  - placer.identity
  - placer.identity.types
  - placer.value.types
  files:
  - placer/control/**
  - placer/control/types.py
writes:
- placer/control/types.py
reads:
- placer/candidates/types.py
- placer/contracts.py
- placer/control/types.py
- placer/identity/types.py
- placer/value/types.py
---
## Capability — what it can do

The Control feature defines the **stateless controller harness** that drives every placement order through its lifecycle. It provides two distinct controllers — a per-order controller and a global allocator — along with the full vocabulary of triggers, actions, and result types that connect them to the operator pipeline.

**Trigger intake.** The `Trigger` / `TriggerKind` pair (`placer/control/types.py`) declares the four event classes that can wake a controller: `new_order`, `outcome_arrived`, `clock_tick`, and `human_override`. Each trigger carries an `OrderId` and an open `payload` dict, allowing callers to attach context without breaking the contract.

**Action vocabulary.** `ControllerAction` enumerates eight discrete instructions the controller can issue to operators: `generate`, `resolve`, `value`, `dispatch`, `regenerate`, `stop`, `reject_scoped`, and `wait`. This exhaustive set mirrors the operator contracts in `placer/contracts.py` and closes the loop between the controller's decision and the pipeline's execution.

**Step result.** `StepResult` is the per-invocation return envelope: which action was taken, how many events were emitted, and the post-step `OrderWorldState` snapshot (optional; omitted when the action did not mutate state). This makes each controller execution observable and auditable.

**Global allocator state.** `AllocatorState` holds the three inputs the allocator needs to price any candidate: `lambda_ops` (shadow price of one ops-hour, defaulting to $50/hr), a `route_plan` dict (cheapest-insertion structure), and `contact_budgets` (per-`OrgId` frequency caps). This state is consumed by `AllocatorContract` implementations in `placer/contracts.py`, which expose `lambda_ops()`, `insertion_cost()`, and `contact_budget_remaining()` as the live query surface. The `lambda_ops` value also feeds the `ContactCost` formula (`c_raw + λ_ops × h_i`) defined in `placer/value/types.py`.

## Implementation — how it works

All types are defined in a single module, `placer/control/types.py`, with no runtime logic — the file is a pure schema layer.

**Type graph.** `Trigger` imports `OrderId` from `placer/identity/types.py` (a `NewType` wrapper over `str`) and `TriggerKind` from the same file. `StepResult` imports `OrderWorldState` from `placer/candidates/types.py`, which is the full blackboard model: lifecycle state, deadline, remaining volume, hypotheses list, candidates list, contact and accept counts, and placed volume. This means every controller step that returns a state snapshot carries the complete order picture.

**Two-controller design.** The module docstring (aligned with spec §6 / architecture §5) names two distinct controllers: a per-order controller that sequences operators for a single order, and a global allocator that manages cross-order resource constraints. `AllocatorState` is the state bag for the second controller; the per-order controller's state lives entirely in `OrderWorldState`. No concrete controller implementations are present in `placer/control/` — the contracts live in `placer/contracts.py` (`AllocatorContract`, `DispatchContract`, `ValueContract`, etc.), and implementations are expected to live behind those abstract interfaces.

**Statelesness contract.** The docstring explicitly declares the controller as a stateless function: it reads `OrderWorldState`, selects the next operator per the active policy, executes, appends events, and returns. `StepResult.events_emitted` is the observable record of that append step.

**Trivial allocator stub.** `AllocatorState` ships with safe defaults (`lambda_ops=50.0`, empty `route_plan`, empty `contact_budgets`) so the system can run before a real allocator is wired in. The docstring notes: "Trivial implementation: constant λ_ops, static route plan. Interface exists so it is never retrofitted."

**`lambda_ops` propagation.** The shadow-price value originates in `AllocatorState`, flows through `AllocatorContract.lambda_ops()` (in `contracts.py`), and is consumed in `ContactCost.total` (`placer/value/types.py`) as well as recorded directly on dispatch events (see `events/types.py` field `lambda_ops: float`). This creates an auditable chain: the allocator's current price appears in every dispatch event.

## Availability — what is currently usable

**Type definitions — present.** All types in `placer/control/types.py` (`TriggerKind`, `Trigger`, `ControllerAction`, `StepResult`, `AllocatorState`) are fully defined and importable.

**Concrete controller logic — absent.** No file under `placer/control/` other than `types.py` was found in the codebase. There is no concrete per-order controller or global allocator implementation present; only the abstract `AllocatorContract` in `placer/contracts.py` and the type schema here. Code-presence of the type layer does not imply a runnable controller.

**`from placer.control` imports — zero.** Search across the full codebase found no module importing from `placer.control`. The types are defined but not yet wired into any active pipeline path, API handler, or script.

**`AllocatorState` defaults.** The $50/hr `lambda_ops` default and the seed-demo scripts (`scripts/seed_demo.py`) both use the same value, indicating the default is operative in demo/synthetic environments. Whether it is read from a live `AllocatorState` instance or hard-coded at call sites in those scripts cannot be confirmed from the source alone.

**No changelog.** No changelog is configured; no changelog-vs-code discrepancies to flag.
