---
feature: Controllers
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
  - placer/controllers/**
writes: []
reads:
- placer/controllers/__init__.py
- placer/controllers/dispatch.py
- placer/controllers/gate_evaluator.py
- placer/controllers/types.py
---
## Capability — what it can do

The Controllers module defines the **trigger-and-action harness** that governs how the Placer system advances the state of a donation-matching order in response to external events. It exposes two conceptual controllers referenced throughout the codebase (spec §6, architecture §5):

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

The types defined here are now consumed by concrete controller modules within the same package.

## Implementation — how it works

The module is structured as a multi-file package within `placer/controllers/**`:

- **`__init__.py`** — package marker; declares the docstring "Controller harness: per-order and global allocator." No executable code.
- **`types.py`** — all shared type definitions (spec §6, architecture §5): `TriggerKind`, `Trigger`, `ControllerAction`, `StepResult`, `AllocatorState`.
- **`dispatch.py`** — the concrete dispatch controller (spec §6, governing rule 4). Implements the trigger-to-action loop.
- **`gate_evaluator.py`** — gate evaluation against candidates (M1 §5). Pure predicate evaluation backed by DB-loaded `GateInstance` records.

### Type hierarchy

`types.py` imports `OrderWorldState` from the candidates identity module. `OrderWorldState` is the "blackboard" (architecture §5) — a Pydantic model holding the order's `lifecycle` enum (`OrderLifecycleState`), volume accounting (`total_volume`, `v_remaining`, `placed_volume`), the list of `Candidate` records, contact counts, and the order deadline. The controller consumes and optionally returns a snapshot of this object in `StepResult.order_state_after`.

### Relationship to contracts
The abstract interface for the global allocator, `AllocatorContract`, lives in the core contracts module. `AllocatorState` in this package is its corresponding concrete data class — a simple Pydantic model rather than an ABC. The `lambda_ops` field in `AllocatorState` defaults to 50.0 $/hr, consistent with the synthetic adapter seed and the demo seed script.

### Stateless step model
The docstring in `types.py` specifies the intended execution model: the controller is a **stateless function** invoked on triggers. It reads world state, selects the next operator, executes, appends events to the event spine, and returns. No controller state persists between triggers — all durability is delegated to the append-only event log.

### Dispatch controller (`dispatch.py`)

`handle_trigger` is the main entry point. It loads `OrderWorldState` via the candidate store, then branches on `TriggerKind`:

- **`NEW_ORDER`** — calls the pipeline's `recommend()`, then `_create_and_dispatch_batch`.
- **`OUTCOME_ARRIVED` / `CLOCK_TICK`** — re-runs `recommend()` only if `v_remaining > 0`, then dispatches another batch if candidates remain.
- Any other kind — returns `ControllerAction.WAIT`.

Batch dispatch follows the **propensity-before-actuation** invariant (spec §6, rule 4): `PLANNED` stage transitions and the DB commit happen before `write_recommendations` is called on the Simpli writeback adapter, and `CONTACTED` transitions follow after. The default batch size is 10; candidates with `p_accept < 0.05` are excluded by `_select_batch`.

### Gate evaluator (`gate_evaluator.py`)

`evaluate_gates_for_candidates` loads all `status = 'active'` `GateInstance` rows from the `gate_instances` table, then for each candidate that is not already `SCREENED_OUT` or `GATE_BLOCKED`, evaluates every applicable gate predicate. Scope matching (`global`, `org:`, `order:`) determines applicability; only `CompileForm.HARD_GATE` predicates can block. Supported predicates include `HAZMAT_CLASS`, `REFRIGERATION`, `GEOGRAPHIC_DISTANCE`, `GEOGRAPHIC_EXCLUSION`, `ORG_RESTRICTION`, `ORG_CAPABILITY`, `REGULATORY_BLOCK`, and `LOT_SIZE`. Blocked candidates receive `CandidateStage.GATE_BLOCKED` and a `stage_reason` listing the blocking gate IDs.

## Availability — is it usable right now

**The type contract layer is fully importable. The dispatch and gate-evaluation controllers are implemented but their end-to-end availability depends on surrounding infrastructure.**

The types in `types.py` (`TriggerKind`, `ControllerAction`, `Trigger`, `StepResult`, `AllocatorState`) are valid Pydantic/StrEnum models and are now actively imported by `dispatch.py`.

The dispatch controller (`dispatch.py`) and gate evaluator (`gate_evaluator.py`) are present and executable in the package, with the following caveats:

- **Database dependency.** Both files require a live `psycopg.AsyncConnection`. `gate_evaluator.py` queries `gate_instances`, `orgs`, `attribute_values`, and `attributes` tables; `dispatch.py` relies on `get_order_state` and `transition_stage` from the candidate store. These must be seeded and migrated before the controllers can run successfully.
- **Pipeline dependency.** `dispatch.py` calls `recommend()` from the pipeline module. Pipeline availability is a prerequisite for `NEW_ORDER` and `OUTCOME_ARRIVED`/`CLOCK_TICK` trigger handling.
- **Simpli writeback adapter.** `dispatch.py` calls `write_recommendations` from the Simpli writeback adapter. If the adapter is not configured or Simpli is unreachable, `_create_and_dispatch_batch` will fail after candidates have already been staged as `PLANNED`.
- **`AllocatorState` is not wired to `AllocatorContract`.** The core contracts module defines three abstract methods on `AllocatorContract`. No class inheriting from it and backed by `AllocatorState` was found — the global allocator remains a type-only contract.
- **`HUMAN_OVERRIDE` trigger not handled.** `handle_trigger` returns `ControllerAction.WAIT` for any `TriggerKind` not explicitly matched, which includes `HUMAN_OVERRIDE`.
- **No changelog entry.** There is no changelog configured, so no intent signal exists about planned completion date or deployment status.

The route `/placer/controllers` resolves to the package `__init__.py`, which contains only a docstring — there is no HTTP handler at that path. The controllers are invoked programmatically, not via an HTTP route.
