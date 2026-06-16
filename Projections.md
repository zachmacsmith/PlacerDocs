---
feature: Projections
group: Placer
first_commit: c66cac868aea2ad3d45474337649f15a7c7db058
last_synced: '2026-06-15'
last_commit: 6dc428c8cfbf577dc8254a42c8b1873db3babcd4
anchors:
  tables: []
  endpoints: []
  types:
  - AuditProjection
  - FeasibilityProjection
  - PresentProjection
  - RankProjection
  - RankedRecommendation
  - SizeProjection
  api_modules:
  - placer.core
  files:
  - placer/projections/**
writes: []
reads:
- placer/core/contracts.py
- placer/core/events.py
- placer/projections/types.py
---
## Capability — what it can do

The Projections module defines five distinct read-only views — called **projections** — that can be computed over an `OrderWorldState` without mutating any table. Each projection answers a different operational question:

| Projection | Contract method | Core question |
|---|---|---|
| `RankProjection` | `rank(state)` | Which charities should be recommended, and in what order? |
| `SizeProjection` | `size(state)` | What is the market-size posterior as a point estimate and credible interval? |
| `FeasibilityProjection` | `feasible(state)` | Is the order feasibly serviceable, and at what scope? |
| `PresentProjection` | `present(state)` | What is the operator-facing plan for this order right now? |
| `AuditProjection` | `audit(state, t, T)` | Why were specific decisions made, expressed as a bitemporal why-trace? |

`RankProjection` is the richest: it returns a list of `RankedRecommendation` objects, each carrying the full `FactorBreakdown` (p_fit × p_want × p_cap × p_resp) sourced from `placer.core.events`, an expected monetary contribution (`emc`), a `contact_index`, provenance bridges, cross-family signal count, geographic fields, and optional LLM rerank reasoning — everything the worker contract and downstream operator UI need.

`PresentProjection` composes the others: it embeds a `SizeProjection` and a `list[RankedRecommendation]` inside a single order-level snapshot that also exposes lifecycle state, remaining volume, a candidate-stage count summary, and the next recommended action.

`AuditProjection` is bitemporal: it carries both `as_of` (event-time) and `as_known_at` (record-time) and exposes decision sequences, belief snapshots, and candidate trajectories for full reproducibility.

The abstract `ProjectionContract` in `placer/core/contracts.py` (spec §7) is the frozen interface behind which all implementations live; changing it is a re-architecture, not a refactor.

## Implementation — how it works

`placer/projections/types.py` owns only the Pydantic **type definitions** — it contains no computation logic. All five projection shapes are pure `BaseModel` subclasses. The module's docstring is explicit: *"Projections are pure views over world state. The deliverable is the state; the list is a rendering. No endpoint mutates tables directly."*

The computation contract is declared in `placer/core/contracts.py` via `ProjectionContract(ABC)`, an abstract base class with five async methods (`rank`, `size`, `feasible`, `present`, `audit`). No concrete implementation of `ProjectionContract` was found in the codebase at the time of this writing — only the abstract declaration exists. The `POST /recommendations` endpoint in `placer/api/server.py` is the intended consumer of a `rank` projection but currently returns a stub empty response with a comment noting it will be "replaced by the full pipeline: ingest → generate → resolve → value → rank."

**Key type dependencies:**
- `RankedRecommendation` and `PresentProjection.top_candidates` both import `FactorBreakdown` from `placer/core/events.py`, grounding recommendation scores in the same factor model used throughout the event spine.
- `PresentProjection` embeds `SizeProjection` as an optional field, making market-size a composable sub-projection rather than a separate call.
- All types are plain Pydantic `BaseModel`s with no validators beyond field typing, keeping serialization side-effect-free.

Because the module is types-only and carries no `__init__` registrations or side effects, importing it has no runtime cost.

## Availability — is it usable right now

**Type definitions:** Available. `placer/projections/types.py` is present and importable with all five projection shapes and `RankedRecommendation` fully defined. `FactorBreakdown` is sourced from `placer/core/events.py` (moved from the former `placer/events/types.py`).

**Abstract contract:** Available. `ProjectionContract` is declared in `placer/core/contracts.py` and specifies the full five-method interface.

**Concrete projection logic:** Not yet implemented. No class implementing `ProjectionContract` was found anywhere in the codebase. The five projection computations (rank, size, feasible, present, audit) have no runtime backing.

**API surface:** Partially available. `POST /recommendations` (guarded by `api_key`) is the intended delivery point for `RankProjection` results, but the handler currently returns a hard-coded empty stub. `POST /context-analysis` is similarly stubbed. No HTTP endpoints exist for `SizeProjection`, `FeasibilityProjection`, `PresentProjection`, or `AuditProjection`.

**Summary:** The schema contract and abstract interface are production-ready; the computation implementations and most API wiring are pending. Code presence of the type definitions does not imply user-accessible projection results.
