---
feature: Controllers
group: New
first_commit: 6dc428c8cfbf577dc8254a42c8b1873db3babcd4
last_synced: '2026-06-15'
last_commit: 6dc428c8cfbf577dc8254a42c8b1873db3babcd4
anchors:
  tables: []
  endpoints: []
  types:
  - AllocatorState
  - ControllerAction
  - StepResult
  - Trigger
  - TriggerKind
  api_modules:
  - placer.controllers.types
  files:
  - placer/controllers/**
  - placer/controllers/__init__.py
  - placer/controllers/types.py
writes: []
reads:
- placer/core/contracts.py::AllocatorContract
- placer/identity/candidates.py::OrderWorldState
---
## Capability — what it can do

The Controllers package (`placer/controllers/`) defines the **event-driven harness** that governs how per-order processing and global resource allocation are initiated and recorded. It establishes two complementary control surfaces:

**Per-order controller** — A stateless unit invoked on four trigger kinds (`new_order`, `outcome_arrived`, `clock_tick`, `human_override`). Given a `Trigger` carrying an `OrderId` and an optional payload, it reads the order's `OrderWorldState`, selects the next operator according to the active policy, executes that operator, appends resulting events, and returns a `StepResult` that records which `ControllerAction` was taken, how many events were emitted, and the updated `OrderWorldState`.

The eight `ControllerAction` values (`generate`, `resolve`, `value`, `dispatch`, `regenerate`, `stop`, `reject_scoped`, `wait`) map directly onto the operator contracts defined in `placer/core/contracts.py`, making the controller the single routing point between lifecycle triggers and pipeline operators.

**Global allocator** — A companion object whose `AllocatorState` carries three scalars that parameterise every order simultaneously: `lambda_ops` (the shadow price of one ops-hour, defaulting to $50/hr), a `route_plan` dict, and per-org `contact_budgets` (frequency caps). These are the inputs consumed by the `AllocatorContract` abstract interface in `placer/core/contracts.py`, which exposes `lambda_ops()`, `insertion_cost(org_id, route_plan)`, and `contact_budget_remaining(org_id)`.

## Implementation — how it works

`placer/controllers/__init__.py` is a namespace stub; all substantive definitions live in `placer/controllers/types.py`.

**Type model.** Four Pydantic/StrEnum constructs carry the full controller contract:

| Type | Role |
|---|---|
| `TriggerKind` (StrEnum) | Four lifecycle events that cause the controller to fire |
| `Trigger` (BaseModel) | Wire envelope: `kind`, `order_id`, `payload` |
| `ControllerAction` (StrEnum) | Exhaustive set of actions the controller can dispatch |
| `StepResult` (BaseModel) | Return value: action taken, events emitted, optional post-step `OrderWorldState` |
| `AllocatorState` (BaseModel) | Global allocator snapshot: `lambda_ops`, `route_plan`, `contact_budgets` |

**Execution model.** The module docstring (spec §6, architecture §5) describes the controller as a *stateless function* — it receives a trigger, reads world state, makes a single-step policy decision, and returns. No controller loop or scheduler is implemented inside this package; the package only furnishes the types that any concrete implementation must use.

**Integration points.** `StepResult.order_state_after` is typed as `OrderWorldState | None`, imported from `placer.identity.candidates`, which is the canonical blackboard for order lifecycle, candidate list, volume remaining, and contact counts. `AllocatorState.lambda_ops` mirrors the `lambda_ops` field used in `placer/core/events.py` (dispatch event schema) and `placer/value/types.py` (insertion-cost formula), ensuring the allocator's shadow price flows consistently through valuation.

**Abstract contracts.** The `AllocatorContract` ABC in `placer/core/contracts.py` is the interface that a concrete allocator must implement against `AllocatorState`; the controllers package owns the state shape while `contracts.py` owns the callable interface. No concrete implementation of either the per-order controller or the global allocator was found in the codebase — only the type harness and abstract contracts exist.

## Availability — what is live

The controllers package exposes **types and interfaces only**. No concrete per-order controller class and no concrete global allocator implementation were found anywhere in the codebase — `TriggerKind`, `Trigger`, `ControllerAction`, `StepResult`, and `AllocatorState` are defined but nothing imports or instantiates them outside `placer/controllers/types.py` itself.

The `AllocatorContract` ABC in `placer/core/contracts.py` declares the allocator interface, and `AllocatorState` in this package provides its default state shape (λ_ops = $50/hr, empty route plan, no contact budgets), but no runtime binding wires them together.

**Consequently, no trigger-driven order stepping or global allocation is currently active at runtime.** The `/placer/controllers` route in the feature table registers the package, but there is no HTTP handler, background worker, or scheduler attached to it. This package should be understood as a **design contract** — it fixes the type signatures that future controller implementations must conform to, consistent with spec §6 and architecture §5, but those implementations have not yet been built.
