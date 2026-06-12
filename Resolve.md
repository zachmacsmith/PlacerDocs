---
feature: Resolve
group: Placer
first_commit: c66cac868aea2ad3d45474337649f15a7c7db058
last_synced: '2026-06-12'
last_commit: c66cac868aea2ad3d45474337649f15a7c7db058
anchors:
  tables: []
  endpoints: []
  types:
  - FusionResult
  - ResolutionDiagnostics
  - RetrievalResult
  - RetrievalStream
  api_modules:
  - placer.identity
  files:
  - placer/contracts.py::ResolveContract
  - placer/resolve/**
  - placer/resolve/types.py
writes: []
reads:
- placer/contracts.py::ResolveContract
- placer/identity/types.py::OrgId
- placer/resolve/types.py
---
## Capability — what it can do

The Resolve feature is workflow stage 3–4 of the placement pipeline (spec §3). Its responsibility is to translate a list of scored hypotheses into a deduplicated, ranked list of candidate organisations. Three distinct retrieval streams run in parallel:

- **Stream A — Onboarded** (`RetrievalStream.ONBOARDED`): exhaustive scoring over the already-onboarded org corpus; no external retrieval needed.
- **Stream B — Full Corpus** (`RetrievalStream.FULL_CORPUS`): a recall-then-precision stack that searches the broader org database.
- **Stream C — Additional Search** (`RetrievalStream.ADDITIONAL_SEARCH`): triggered conditionally by product structure or high-volume quantity signals, supplementing Streams A and B.

Each per-stream hit is captured in a `RetrievalResult`, which records the organisation (`OrgId`), optional EIN, the originating stream, the query formulation used, a raw retrieval score, and optional reranking fields (`rerank_score`, `rerank_verdict`). Results are tied back to the generating hypothesis via `hypothesis_id` and the bridge mediator family via `bridge_family`.

After retrieval, results from all three streams are merged through **Reciprocal Rank Fusion (RRF)** augmented with cross-family corroboration. `FusionResult` holds the RRF score, a corroboration bonus, a count of distinct bridge families that corroborated the org, lists of contributing bridge mediators and hypotheses, and the `final_score` that downstream ranking consumes.

Per-run observability is provided by `ResolutionDiagnostics`, which records per-stream hit counts, the count surviving reranking, and the count surviving fusion — enabling funnel analysis at the archetype (hypothesis) level as described in spec §3.4.

The operator contract for this stage is `ResolveContract` (defined in `placer/contracts.py`): an abstract async `resolve(hypotheses, order_id) → list[Candidate]` that guarantees full candidate provenance on output.


## Implementation — how it works

`placer/resolve/types.py` is the canonical type layer for the Resolve stage and is the sole file in the `placer/resolve/` module currently present in the codebase.

**Type hierarchy:**

| Type | Role |
|---|---|
| `RetrievalStream` (StrEnum) | Identifies stream A/B/C by single-letter value |
| `RetrievalResult` (Pydantic) | One org hit from one stream, pre-fusion |
| `FusionResult` (Pydantic) | Post-RRF merged record with corroboration metadata |
| `ResolutionDiagnostics` (Pydantic) | Per-hypothesis funnel counters across all stages |

**Dependencies:**
- `OrgId` is imported from `placer.identity.types`, anchoring Resolve firmly in the platform's canonical identity space.
- `hypothesis_id` and `bridge_family` on `RetrievalResult` match the same fields on candidate surfacing provenance (`placer/candidates/types.py::SurfacedBy`) and on learn-layer records (`placer/learn/types.py`), establishing the traceability chain from retrieval hit back to the generating hypothesis.

**Fusion mechanics (modelled, not implemented here):**
`FusionResult.corroboration_bonus` and `distinct_family_count` encode the cross-family corroboration step: an org that surfaces under multiple bridge families (e.g. both consumption and people) receives a bonus, encoding the intuition that multi-angle corroboration is a stronger placement signal than a single-family hit. `contributing_bridges` and `contributing_hypotheses` carry the full audit trail for that bonus.

**Contract boundary:**
`ResolveContract` in `placer/contracts.py` is the abstract interface that any concrete implementation must satisfy. It is the only place in the codebase that references `resolve` as an operator function, confirming that no concrete implementation file is yet present — the types file alone is committed.


## Availability — is it usable right now

**Partially defined; no concrete implementation confirmed.**

`placer/resolve/types.py` is present and provides the complete type contract for the retrieval, fusion, and diagnostics data structures. `ResolveContract` is declared in `placer/contracts.py`.

However, no concrete class implementing `ResolveContract` was found in the codebase, and no file other than `types.py` exists under `placer/resolve/`. The `RetrievalResult`, `FusionResult`, and `ResolutionDiagnostics` types are not imported anywhere outside their own module (no `from placer.resolve` import sites were found). This means the Resolve stage is **typed but not yet wired** into the pipeline at runtime.

The feature carries no access guards. There are no changelog entries to cross-reference. Until a concrete `ResolveContract` implementation is registered and called from the control loop or API layer, this feature does not execute at runtime despite having a well-specified type contract.

