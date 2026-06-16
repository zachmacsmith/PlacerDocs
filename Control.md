---
feature: Control
group: Placer
first_commit: c66cac868aea2ad3d45474337649f15a7c7db058
last_synced: '2026-06-16'
last_commit: e75bcb47650c6c56370cc10193be5f08ae3490e8
anchors:
  tables: []
  endpoints: []
  types:
  - AllocatorState
  - StepResult
  - Trigger
  api_modules:
  - placer.core
  - placer.identity
  files:
  - placer/controllers/**
writes: []
reads:
- placer/controllers/dispatch.py
- placer/controllers/gate_evaluator.py
- placer/controllers/types.py
- placer/core/contracts.py
- placer/core/events.py
- placer/pipeline.py
---
## Capability — what it can do

The Control feature defines the **stateless controller harness** that drives every placement order through its lifecycle. It provides two distinct controllers — a per-order controller and a global allocator — along with the full vocabulary of triggers, actions, and result types that connect them to the operator pipeline.

**Trigger intake.** The `Trigger` / `TriggerKind` pair (`placer/controllers/types.py`) declares the four event classes that can wake a controller: `new_order`, `outcome_arrived`, `clock_tick`, and `human_override`. Each trigger carries an `OrderId` and an open `payload` dict, allowing callers to attach context without breaking the contract.

**Action vocabulary.** `ControllerAction` enumerates eight discrete instructions the controller can issue to operators: `generate`, `resolve`, `value`, `dispatch`, `regenerate`, `stop`, `reject_scoped`, and `wait`. This exhaustive set mirrors the operator contracts defined in the core contracts module and closes the loop between the controller's decision and the pipeline's execution.

**Step result.** `StepResult` is the per-invocation return envelope: which action was taken, how many events were emitted, and the post-step `OrderWorldState` snapshot (optional; omitted when the action did not mutate state). This makes each controller execution observable and auditable.

**Global allocator state.** `AllocatorState` holds the three inputs the allocator needs to price any candidate: `lambda_ops` (shadow price of one ops-hour, defaulting to $50/hr), a `route_plan` dict (cheapest-insertion structure), and `contact_budgets` (per-`OrgId` frequency caps). This state is consumed by `AllocatorContract` implementations in the core contracts module, which expose `lambda_ops()`, `insertion_cost()`, and `contact_budget_remaining()` as the live query surface. The `lambda_ops` value also feeds the `ContactCost` formula (`c_raw + λ_ops × h_i`) defined in `placer/value/types.py`.

## Implementation — how it works

All types are defined in a single module, `placer/controllers/types.py`, with no runtime logic — the file is a pure schema layer.

**Type graph.** `Trigger` imports `OrderId` from `placer.core.ids` (a `NewType` wrapper over `str`) and `TriggerKind` from the same file. `StepResult` imports `OrderWorldState` from `placer.identity.candidates`, which is the full blackboard model: lifecycle state, deadline, remaining volume, hypotheses list, candidates list, contact and accept counts, and placed volume. This means every controller step that returns a state snapshot carries the complete order picture.

**Two-controller design.** The module docstring (aligned with spec §6 / architecture §5) names two distinct controllers: a per-order controller that sequences operators for a single order, and a global allocator that manages cross-order resource constraints. `AllocatorState` is the state bag for the second controller; the per-order controller's state lives entirely in `OrderWorldState`. Concrete implementations are now present under `placer/controllers/`: `dispatch.py` provides the per-order dispatch controller, and `gate_evaluator.py` provides hard-gate evaluation. The abstract contracts (`AllocatorContract`, `DispatchContract`, `ValueContract`, etc.) still live in the core contracts module.

**Statelesness contract.** The docstring explicitly declares the controller as a stateless function: it reads `OrderWorldState`, selects the next operator per the active policy, executes, appends events, and returns. `StepResult.events_emitted` is the observable record of that append step.

**Trivial allocator stub.** `AllocatorState` ships with safe defaults (`lambda_ops=50.0`, empty `route_plan`, empty `contact_budgets`) so the system can run before a real allocator is wired in. The docstring notes: "Trivial implementation: constant λ_ops, static route plan. Interface exists so it is never retrofitted."

**`lambda_ops` propagation.** The shadow-price value originates in `AllocatorState`, flows through `AllocatorContract.lambda_ops()` (in the core contracts module), and is consumed in `ContactCost.total` (in the value types module) as well as recorded directly on dispatch events (a `lambda_ops: float` field in the core events module). This creates an auditable chain: the allocator's current price appears in every dispatch event.

## Availability — what is currently usable

**Type definitions — present.** All types in `placer/controllers/types.py` (`TriggerKind`, `Trigger`, `ControllerAction`, `StepResult`, `AllocatorState`) are fully defined and importable.

**Dispatch controller — present.** `placer/controllers/dispatch.py` is a concrete implementation of the per-order dispatch controller (spec §6, governing rule 4). Its `handle_trigger` entry point accepts a `Trigger` and routes to one of three internal paths based on `TriggerKind`: `NEW_ORDER` pipelines fresh candidates, `OUTCOME_ARRIVED` and `CLOCK_TICK` both re-score and re-dispatch after new data. Batches are capped at 10 candidates with a minimum `p_accept` threshold of 0.05. The propensity-before-actuation guarantee is enforced at this layer: a `DECISION_VALUATION_SNAPSHOT` event is committed before any write to the downstream platform.

**Gate evaluator — present and wired.** `placer/controllers/gate_evaluator.py` implements hard-gate evaluation (M1 §5). Its `evaluate_gates_for_candidates` function is imported and called by the pipeline module, making it an active path in candidate processing. It loads `GateInstance` records from the `gate_instances` table, checks scope (global / org-scoped / order-scoped), and evaluates `HARD_GATE` predicates including `HAZMAT_CLASS`, `REFRIGERATION`, `GEOGRAPHIC_DISTANCE`, `GEOGRAPHIC_EXCLUSION`, `ORG_RESTRICTION`, `ORG_CAPABILITY`, `REGULATORY_BLOCK`, and `LOT_SIZE`. Candidates that fail any applicable gate are stamped `GATE_BLOCKED` with a reason string.

**Global allocator — absent.** No concrete global allocator implementation was found. `AllocatorState` and the abstract `AllocatorContract` in the core contracts module define the interface, but no live cross-order allocator drives it. Code-presence of the type layer does not imply a runnable allocator.

**`AllocatorState` defaults.** The $50/hr `lambda_ops` default is operative in demo/synthetic environments. Whether it is read from a live `AllocatorState` instance or hard-coded at call sites in those environments cannot be confirmed from the source alone.

**No changelog.** No changelog is configured; no changelog-vs-code discrepancies to flag.
