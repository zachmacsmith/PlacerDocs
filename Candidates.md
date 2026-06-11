---
feature: Candidates
group: Placer
last_synced: '2026-06-11'
last_commit: 87dd52f08e97ba92e8de49eace545f1073d264af
anchors:
  tables:
  - candidates
  - orders
  endpoints: []
  types:
  - Candidate
  - CandidateId
  - CandidateStage
  - ExplorationSlotPurpose
  - FactorBreakdown
  - OrderId
  - OrderLifecycleState
  - OrderWorldState
  - OrgId
  - SurfacedBy
  - ValuationSnapshot
  api_modules:
  - placer.candidates
  - placer.candidates.store
  - placer.candidates.types
  - placer.contracts
  - placer.events.types
  - placer.identity
  - placer.identity.types
  files:
  - placer/candidates/**
  - placer/candidates/store.py
  - placer/candidates/types.py
  - placer/contracts.py
  - placer/control/types.py
  - placer/events/types.py
  - placer/identity/types.py
writes:
- SET
- candidates
- placer/candidates/store.py
reads:
- placer/candidates/store.py
- placer/candidates/types.py
- placer/contracts.py
- placer/control/types.py
- placer/events/types.py
- placer/identity/types.py
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

The feature also owns the **canonical type definitions** consumed across the platform (imported by `placer.contracts` and `placer.control.types`):

- `Candidate` — the core record with provenance (`surfaced_by: list[SurfacedBy]`), lifecycle position (`stage`, `stage_reason`), and a frozen `ValuationSnapshot` captured at decision time.
- `CandidateStage` — a six-value state machine: `SURFACED → SCREENED_OUT | VALUED → PLANNED → CONTACTED → RESOLVED`.
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

**Type system.** Types are defined in `placer/candidates/types.py` and imported by `placer/contracts.py` and `placer/control/types.py`. `ValuationSnapshot` depends on `FactorBreakdown` and `ExplorationSlotPurpose` from `placer/events/types.py`. Identity NewTypes (`CandidateId`, `OrderId`, `OrgId`) come from `placer/identity/types.py`.

## Availability — is it usable right now

All four store functions (`upsert_candidate`, `transition_stage`, `get_candidates_for_order`, `get_order_state`) and the full type hierarchy are present in source at `placer/candidates/store.py` and `placer/candidates/types.py`. The types are actively imported by `placer/contracts.py` and `placer/control/types.py`, indicating they are load-bearing at the platform level.

No HTTP endpoints, CLI entrypoints, or route handlers are registered for this module — it is a repository layer invoked programmatically by higher-level controllers. No feature guards are present.

There is no `placer/candidates/__init__.py` found in the codebase; the module boundary is the two files `store.py` and `types.py` only. No changelog is configured, so no pending or in-flight changes can be confirmed from that source. Code presence is confirmed; runtime availability depends on a live `psycopg.AsyncConnection` to a PostgreSQL instance with the `candidates` and `orders` tables provisioned.
