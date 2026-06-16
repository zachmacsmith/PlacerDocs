---
feature: Candidates
group: Placer
first_commit: 5499bc5a8c1f45be4e6cdc23b3f7414d926340f0
last_synced: '2026-06-15'
last_commit: 952dd6cc874c7cc4402ceb32f94baf759278beb3
anchors:
  tables:
  - candidates
  - orders
  endpoints: []
  types: []
  api_modules:
  - placer.core
  - placer.identity
  files:
  - placer/identity/**
writes:
- candidates
reads:
- orders
- placer/identity/candidates.py
---
## Capability — what it can do

The Candidates feature is the **persistence and lifecycle backbone for the placement funnel**. It manages every org that the system has ever surfaced as a potential match for a donation order, from initial surfacing through final resolution.

Four concrete operations are provided:

| Function | Purpose |
|---|---|
| `upsert_candidate` | Insert a new candidate or overwrite `surfaced_by`, `stage`, `stage_reason`, `valuation_snapshot`, `contacted_at`, and `resolved_at` on conflict by `candidate_id` |
| `transition_stage` | Advance or revert a single candidate's `stage` and `stage_reason` without touching other fields |
| `get_candidates_for_order` | Return all candidates tied to an order, optionally filtered to a specific `CandidateStage` |
| `get_order_state` | Assemble a full `OrderWorldState` blackboard: order metadata from the `orders` table joined with all its candidates, plus derived counts (`contact_count`, `accept_count`) |

The full set is retained — including screened-out candidates — because rejects form the denominator for funnel metrics, the exploration population for retrieval-level decisions, and the fallback queue within a stream (per the module docstring and `types.py` header).

The feature also owns the **canonical type definitions** consumed across the platform (imported by the core contracts module and the controller type layer):

- `Candidate` — the core record with provenance (`surfaced_by: list[SurfacedBy]`), lifecycle position (`stage`, `stage_reason`), and a frozen `ValuationSnapshot` captured at decision time.
- `CandidateStage` — a seven-value state machine: `SURFACED → SCREENED_OUT | GATE_BLOCKED | VALUED → PLANNED → CONTACTED → RESOLVED`. `GATE_BLOCKED` is a rejection terminal distinct from `SCREENED_OUT`, earned when a candidate fails a compiled gate predicate rather than a retrieval-level filter.
- `SurfacedBy` — provenance entry linking a candidate to a bridge, hypothesis, segments, rank, and stream slot (`a`/`b`/`c`).
- `ValuationSnapshot` — immutable record of `FactorBreakdown`, EMC, `contact_index`, standard deviation, and optional `ExplorationSlotPurpose` frozen at valuation time.
- `OrderWorldState` — the system blackboard: order lifecycle, mode, volume figures (`total_volume`, `v_remaining`, `placed_volume`), deadline, full candidate list, optional `hypotheses` list, and contact/accept counts.
- `OrderLifecycleState` — a seven-value order lifecycle enum: `INTAKE → FEASIBILITY_TRIAGE → ACTIVE → FULFILLING → CLOSED | REJECTED_SCOPED | EXPIRED`.

## Implementation — how it works

**Storage layer.** All reads and writes target two PostgreSQL tables accessed via `psycopg.AsyncConnection`:
- `candidates` — columns: `candidate_id` (PK), `order_id`, `org_id`, `surfaced_by` (JSONB), `stage`, `stage_reason`, `valuation_snapshot` (JSONB), `contacted_at`, `resolved_at`.
- `orders` — columns consumed: `order_id`, `lifecycle`, `mode`, `total_volume`, `v_remaining`, `deadline`.

JSONB fields (`surfaced_by`, `valuation_snapshot`) are serialized via Pydantic `model_dump()` with `json.dumps(..., default=str)` on write, and deserialized with `model_validate()` on read through the private `_row_to_candidate` helper.

**Upsert semantics.** `upsert_candidate` uses `INSERT … ON CONFLICT (candidate_id) DO UPDATE`. The initial `INSERT` column list is `(candidate_id, order_id, org_id, surfaced_by, stage, stage_reason, valuation_snapshot)`; `contacted_at` and `resolved_at` are not in the insert column list and therefore default to `NULL` on first insert. On conflict, the `DO UPDATE SET` clause overwrites `surfaced_by`, `stage`, `stage_reason`, `valuation_snapshot`, `contacted_at`, and `resolved_at`. Fields `order_id` and `org_id` are never updated on conflict, preserving provenance stability.

**Stage transitions.** `transition_stage` is intentionally narrow: it issues a targeted `UPDATE candidates SET stage = %s, stage_reason = %s WHERE candidate_id = %s`. No state-machine enforcement or guard conditions are applied in this layer; callers (e.g., `control`, `resolve`) are responsible for valid transition logic.

**World-state assembly.** `get_order_state` joins `orders` and `candidates` in application code rather than SQL. It calls `get_candidates_for_order` (which issues a separate `SELECT`) then derives `contact_count` (stage is `CONTACTED` or `RESOLVED`) and `accept_count` (stage is `RESOLVED` **and** `valuation_snapshot` is non-null) from the result set in memory. The returned `OrderWorldState` also carries `hypotheses` (defaults to `[]`) and `placed_volume` (defaults to `0.0`); these fields are not populated by `get_order_state` itself and must be set by callers if needed.

**Type system.** Types are defined in `placer/identity/candidates.py` and imported by `placer/core/contracts.py` and `placer/controllers/types.py`. `ValuationSnapshot` depends on `FactorBreakdown` and `ExplorationSlotPurpose` from `placer/core/events.py`. Identity NewTypes (`CandidateId`, `OrderId`, `OrgId`) come from `placer/core/ids.py`.

## Availability — is it usable right now

All four store functions (`upsert_candidate`, `transition_stage`, `get_candidates_for_order`, `get_order_state`) are present in source at `placer/identity/candidate_store.py`. The full type hierarchy (`Candidate`, `CandidateStage`, `SurfacedBy`, `ValuationSnapshot`, `OrderWorldState`, `OrderLifecycleState`) is defined in `placer/identity/candidates.py`. These types are actively imported by `placer/core/contracts.py` and `placer/controllers/types.py`, confirming they are load-bearing at the platform level.

`CandidateStage` now carries seven values including the newly added `GATE_BLOCKED` terminal. Code presence is confirmed; whether higher-level controllers have been updated to emit or handle `GATE_BLOCKED` transitions is outside the scope of this module.

The former `placer/candidates/` package (previously containing `store.py` and `types.py`) no longer exists in the codebase; all functionality has migrated into `placer/identity/candidate_store.py` and `placer/identity/candidates.py`. Identity NewTypes (`CandidateId`, `OrderId`, `OrgId`) now originate from `placer/core/ids.py`; `FactorBreakdown` and `ExplorationSlotPurpose` from `placer/core/events.py`.

No HTTP endpoints, CLI entrypoints, or route handlers are registered for this module — it is a repository layer invoked programmatically by higher-level controllers. No feature guards are present.

No changelog is configured, so no pending or in-flight changes can be confirmed from that source. Code presence is confirmed; runtime availability depends on a live `psycopg.AsyncConnection` to a PostgreSQL instance with the `candidates` and `orders` tables provisioned.
