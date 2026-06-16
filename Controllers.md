---
feature: Controllers
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
  - placer/controllers/**
writes: []
reads:
- placer/controllers/__init__.py
- placer/controllers/types.py
---
## Capability — what it can do

The Controllers module (`placer/controllers/`) defines the **trigger-and-action harness** that governs how the Placer system advances the state of a donation-matching order in response to external events. It exposes two conceptual controllers referenced throughout the codebase (spec §6, architecture §5):

1. **Per-order controller** — responds to triggers bound to a specific `OrderId` and selects the next operator action to execute against that order's world state.
2. **Global allocator** — manages system-wide resource constraints: the shadow price of one ops hour (`lambda_ops`), a route plan, and per-organisation contact budgets.

### Triggers
Four trigger kinds (`TriggerKind`) drive the per-order controller:
- `NEW_ORDER` — a new ingest order has arrived.
- `OUTCOME_ARRIVED` — an outcome event (acceptance, rejection, no-response, etc.) has been recorded.
- `CLOCK_TICK` — a periodic time pulse, enabling deadline-aware decisions.
- `HUMAN_OVERRIDE` — an operator has intervened.

Each trigger is carried by a `Trigger` message (Pydantic model) consisting of the `TriggerKind`, the target `OrderId`, and an arbitrary `payload` dict.

### Actions
The controller selects from eight `ControllerAction` values — `GENERATE`, `RESOLVE`, `VALUE`, `DISPATCH`, `REGENERATE`, `STOP`, `REJECT_SCOPED`, `WAIT` — which map directly to operator contracts defined in the core contracts module.

### Step result
After executing a step the controller produces a `StepResult` recording the `ControllerAction` taken, the count of events emitted, and an optional snapshot of the `OrderWorldState` after the step.

### Global allocator state
`AllocatorState` holds the three quantities the allocator must always expose: `lambda_ops` (default 50.0 $/hr), an open `route_plan` dict, and `contact_budgets` mapping `org_id` → remaining contacts allowed. The abstract counterpart in `AllocatorContract` (spec §6) mandates that any real implementation supply `lambda_ops()`, `insertion_cost()`, and `contact_budget_remaining()`.

The module has no runtime implementations of its own — it is purely a type/contract layer that other modules implement and consume.

## Implementation — how it works

The module is structured as a two-file package:

- **`placer/controllers/__init__.py`** — package marker; declares the docstring "Controller harness: per-order and global allocator." No executable code.
- **`placer/controllers/types.py`** — all type definitions (spec §6, architecture §5). This is the entire current implementation of the package.

### Type hierarchy

`types.py` imports `OrderWorldState` from the candidates identity module. `OrderWorldState` is the "blackboard" (architecture §5) — a Pydantic model holding the order's `lifecycle` enum (`OrderLifecycleState`), volume accounting (`total_volume`, `v_remaining`, `placed_volume`), the list of `Candidate` records, contact counts, and the order deadline. The controller consumes and optionally returns a snapshot of this object in `StepResult.order_state_after`.

### Relationship to contracts
The abstract interface for the global allocator, `AllocatorContract`, lives in the core contracts module. `AllocatorState` in this package is its corresponding concrete data class — a simple Pydantic model rather than an ABC. The `lambda_ops` field in `AllocatorState` defaults to 50.0 $/hr, consistent with the synthetic adapter seed and the demo seed script.

### Stateless step model
The docstring in `types.py` specifies the intended execution model: the controller is a **stateless function** invoked on triggers. It reads world state, selects the next operator, executes, appends events to the event spine, and returns. No controller state persists between triggers — all durability is delegated to the append-only event log.

### No concrete controller implementations found
A search across the entire codebase found that `TriggerKind`, `ControllerAction`, `StepResult`, and `AllocatorState` are defined only in `placer/controllers/types.py` and are not imported anywhere else. No module currently instantiates or calls a controller step function. The harness is defined but its runtime body has not yet been authored.

## Availability — is it usable right now

**Partially available as a type contract; not yet executable.**

The type definitions in `placer/controllers/types.py` are importable and valid Pydantic/StrEnum models. Code that needs to reference `TriggerKind`, `ControllerAction`, `Trigger`, `StepResult`, or `AllocatorState` can do so today.

However:
- **No concrete controller implementations exist.** No function or class in the codebase implements the trigger-dispatch loop described in the `types.py` docstring. The step that receives a `Trigger`, reads `OrderWorldState`, selects a `ControllerAction`, executes an operator, and appends events is not present.
- **`AllocatorState` is not wired to `AllocatorContract`.** The core contracts module defines three abstract methods on `AllocatorContract`. No class inheriting from it and backed by `AllocatorState` was found.
- **No imports from this package anywhere in the codebase.** A global search returned zero hits for `from placer.controllers` — the package is not yet consumed by any other module, API handler, or background job.
- **`types.py` was recently renamed** from `placer/control/types.py` to `placer/controllers/types.py`; its internal imports were updated to reflect current module locations (`OrderId` from the core IDs module, `OrderWorldState` from the identity candidates module). Callers that referenced the old path would need to update their imports, though no such callers were found.
- **No changelog entry.** There is no changelog configured, so no intent signal exists about planned completion date.

The route `/placer/controllers` is listed in the feature table but resolves to the package `__init__.py`, which contains only a docstring. Any UI or API surface at that path would depend on controller implementations that do not yet exist.
