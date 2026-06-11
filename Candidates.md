---
feature: Candidates
group: Placer
last_synced: '2026-06-11'
last_commit: 5499bc5a8c1f45be4e6cdc23b3f7414d926340f0
anchors:
  tables:
  - SET
  - candidates
  - orders
  endpoints: []
  types:
  - Candidate
  - CandidateId
  - CandidateStage
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
  - placer.identity
  files:
  - placer/candidates/**
  - placer/candidates/store.py
  - placer/candidates/types.py
  - placer/contracts.py
  - placer/identity/types.py
writes:
- SET
- candidates
reads:
- candidates
- orders
---
## Capability — what it can do

The Candidates feature is the **persistence and lifecycle backbone for the placement funnel**. It manages every org that the system has ever surfaced as a potential match for a donation order, from initial surfacing through final resolution.

Four concrete operations are provided:

| Function | Purpose |
|---|---|
| `upsert_candidate` | Insert a new candidate or overwrite `surfaced_by`, `stage`, `stage_reason`, and `valuation_snapshot` on conflict by `candidate_id` |
| `transition_stage` | Advance or revert a single candidate's `stage` and `stage_reason` without touching other fields |
| `get_candidates_for_order` | Return all candidates tied to an order, optionally filtered to a specific `CandidateStage` |
| `get_order_state` | Assemble a full `OrderWorldState` blackboard: order metadata from the `orders` table joined with all its candidates, plus derived counts (`contact_count`, `accept_count`) |

The full set is retained — including screened-out candidates — because rejects form the denominator for funnel metrics, the exploration population for retrieval-level decisions, and the fallback queue within a stream (per the module docstring and `types.py` header).

The feature also owns the **canonical type definitions** consumed across the platform:

- `Candidate` — the core record with provenance (`surfaced_by: list[SurfacedBy]`), lifecycle position (`stage`, `stage_reason`), and a frozen `ValuationSnapshot` captured at decision time.
- `CandidateStage` — a six-value state machine: `SURFACED → SCREENED_OUT | VALUED → PLANNED → CONTACTED → RESOLVED`.
- `SurfacedBy` — provenance entry linking a candidate to a bridge, hypothesis, segments, rank, and stream slot (`a`/`b`/`c`).
- `ValuationSnapshot` — immutable record of `FactorBreakdown`, EMC, `contact_index`, standard deviation, and optional `ExplorationSlotPurpose` frozen at valuation time.
- `OrderWorldState` — the system blackboard: order lifecycle, mode, volume figures, deadline, full candidate list, and contact/accept counts.
- `OrderLifecycleState` — a seven-value order lifecycle enum: `INTAKE → FEASIBILITY_TRIAGE → ACTIVE → FULFILLING → CLOSED | REJECTED_SCOPED | EXPIRED`.

## Implementation — how it works

**Storage layer.** All reads and writes target two PostgreSQL tables accessed via `psycopg.AsyncConnection`:
- `candidates` — columns: `candidate_id` (PK), `order_id`, `org_id`, `surfaced_by` (JSONB), `stage`, `stage_reason`, `valuation_snapshot` (JSONB), `contacted_at`, `resolved_at`.
- `orders` — columns consumed: `order_id`, `lifecycle`, `mode`, `total_volume`, `v_remaining`, `deadline`.

JSONB fields (`surfaced_by`, `valuation_snapshot`) are serialized via Pydantic `model_dump()` with `json.dumps(..., default=str)` on write, and deserialized with `model_validate()` on read through the private `_row_to_candidate` helper.

**Upsert semantics.** `upsert_candidate` uses `INSERT … ON CONFLICT (candidate_id) DO UPDATE`. Fields updated on conflict: `surfaced_by`, `stage`, `stage_reason`, `valuation_snapshot`, `contacted_at`, `resolved_at`. Fields that are *not* updated on conflict (e.g., `order_id`, `org_id`) remain as originally inserted, preserving provenance stability.

**Stage transitions.** `transition_stage` is intentionally narrow: it issues a targeted `UPDATE candidates SET stage = %s, stage_reason = %s WHERE candidate_id = %s`. No state-machine enforcement or guard conditions are applied in this layer; callers (e.g., `control`, `resolve`) are responsible for valid transition logic.

**World-state assembly.** `get_order_state` joins `orders` and `candidates` in application code rather than SQL. It calls `get_candidates_for_order` (which issues a separate `SELECT`) then derives `contact_count` (stage is `CONTACTED` or `RESOLVED`) and `accept_count` (stage is `RESOLVED` **and** `valuation_snapshot` is non-null) from the result set in memory.

**Type system.** Types are defined in `placer/candidates/types.py` and imported by `placer/contracts.py` (for `ResolveContract`, `ValueContract`, `DispatchContract`) and `placer/control/types.py`, making `Candidate` and `OrderWorldState` the shared lingua franca for the placement pipeline's operator contracts.

## Availability — is it usable right now

`placer/candidates/store.py` and `placer/candidates/types.py` are present in the codebase and fully implemented. All four async store functions are callable wherever a `psycopg.AsyncConnection` is provided.

- **No HTTP endpoint** is registered directly at `/placer/candidates` — the route appears in the feature table as a component route, not an API route with a registered handler. There is no evidence of a REST interface exposing candidate data directly to external callers.
- **No authentication guard** is declared for this feature.
- **Consumers confirmed in source:** `placer/contracts.py` (imports `Candidate`, `OrderWorldState`), `placer/control/types.py` (imports `OrderWorldState`). The store functions themselves are not yet imported in any file found by search outside `store.py` itself and `scripts/seed_demo.py` (a seed script, not production routing).
- **Database schema not found in source:** No `CREATE TABLE candidates` migration or schema file was located. The table is assumed to exist in the deployment target; its absence at runtime would cause all store operations to fail.
- No changelog is configured, so there is no declared roadmap or release gate to report against.
