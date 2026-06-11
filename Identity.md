---
feature: Identity
group: Placer
last_synced: '2026-06-11'
last_commit: 5499bc5a8c1f45be4e6cdc23b3f7414d926340f0
anchors:
  tables:
  - SET
  - crosswalk_edges
  - orgs
  - segments
  - streams
  endpoints:
  - GET /crosswalk
  - GET /orgs
  - GET /segments
  types:
  - CandidateId
  - ContactId
  - CrosswalkEdgeKey
  - DonorId
  - Ein
  - EventSeq
  - HypothesisId
  - OrderId
  - OrgId
  - OrgMemberships
  - OrgRecord
  - OrgRole
  - OrgSnapshot
  - PalletGroupId
  - ProductClassificationRoute
  - ProductIdentity
  - QuantityId
  - Resolution
  - ResolutionConfidence
  - ResolutionMethod
  - Segment
  - SegmentId
  - SegmentMembership
  - SegmentStatus
  - SizeBand
  - Stream
  - StreamId
  - UnspscClass
  - UnspscFamily
  api_modules:
  - placer.identity
  - placer.identity.store
  - placer.identity.types
  files:
  - placer/api/debug.py
  - placer/contracts.py
  - placer/identity/**
  - placer/identity/store.py
  - placer/identity/types.py
writes:
- SET
- crosswalk_edges
- orgs
- placer/identity/store.py
- segments
- streams
reads:
- placer/api/debug.py
- placer/contracts.py
- placer/identity/store.py
- placer/identity/types.py
---
## Capability — what it can do

The Identity feature is the **canonical namespace layer** for all entities in the Placer system. Its module (`placer/identity/store.py`) exposes async repository functions that govern four distinct identity spaces:

**Organization resolution and registry**
`resolve_org` is the sole entry point for turning raw external data (EIN, name, address) into a `Resolution` — a `(canonical_id, confidence, method)` triple. Currently only the EIN-join path (`ResolutionMethod.EIN_JOIN`) is implemented; name/address lookup always returns `None`. `mint_org` registers a new `OrgRecord` on first encounter using `ON CONFLICT DO NOTHING` semantics, preserving idempotency across replay. `get_org` fetches a full `OrgRecord` by its `OrgId`.

**Segment lifecycle management**
Segments are named groupings of recipient organizations used throughout the Bayesian matching pipeline. `mint_segment` creates a new segment with free-text definition and optional lineage. `merge_segments` transitions source segments to `status = 'merged'` and mints a replacement target carrying the source IDs as lineage — preserving the full derivation chain. `get_segment` retrieves any segment by `SegmentId`.

**UNSPSC–Segment crosswalk**
`upsert_crosswalk_edge` writes or updates a mapping between a UNSPSC product family (`UnspscFamily`) and a `SegmentId`, attaching an arbitrary `quantity_refs` JSONB payload. `get_crosswalk_edges_for_family` fetches all edges for a given family, enabling the generator to look up candidate segments by product taxonomy.

**Stream registry**
`upsert_stream` maintains a hub-and-spoke donation stream record (`Stream`), linking a hub org to a list of member orgs and recording a `terminal_absorption` float. Writes are idempotent via `ON CONFLICT DO UPDATE`.

**Platform-wide type system**
`placer/identity/types.py` is the authoritative NewType and Pydantic-model source imported by at least nine other feature modules (`adapters`, `beliefs`, `candidates`, `control`, `events`, `generate`, `resolve`, and `contracts`). Key NewTypes include `OrgId`, `Ein`, `SegmentId`, `StreamId`, `OrderId`, `EventSeq`, `CandidateId`, `QuantityId`, `HypothesisId`, `DonorId`, `ContactId`, `UnspscFamily`, and `UnspscClass`. These wrappers prevent cross-space confusion at call sites.

**Read-only debug API**
Three debug endpoints in `placer/api/debug.py` expose identity data for inspection: `GET /orgs` (paginated org listing with memberships), `GET /segments` (filterable by status), and `GET /crosswalk` (filterable by UNSPSC family, joining segment definition text).


## Implementation — how it works

**Persistence model**
All four identity spaces are backed by PostgreSQL tables accessed via `psycopg.AsyncConnection`. Four tables are implicated: `orgs`, `segments`, `crosswalk_edges`, and `streams`. The store never opens its own connection — every function accepts an injected `psycopg.AsyncConnection[Any]`, making transaction boundaries the caller's responsibility.

**Organization record structure**
`OrgRecord` is a Pydantic model containing `org_id`, optional `ein`, a `ResolutionConfidence` enum (`high | medium | provisional`), the integer `EventSeq` of first encounter, and a nested `OrgMemberships` object. `OrgMemberships` carries segment-membership weights and confidences, stream affiliations, an optional `OrgRole` enum (`hub | intermediate | terminal`), an NTEE code, and a `SizeBand` enum. This compound structure is persisted as JSONB in the `memberships` column and validated through `OrgMemberships.model_validate` on read.

**Resolution confidence and method**
`ResolutionConfidence` (`high | medium | provisional`) and `ResolutionMethod` (`ein_join | name_address_block | fuzzy_match | provisional`) are `StrEnum` types stored as plain strings in the `orgs.resolution_confidence` column. Only `EIN_JOIN` resolution is active in `resolve_org`; the `name_address_block` and `fuzzy_match` methods are declared in the type system but have no corresponding lookup branch yet.

**Segment lineage**
Lineage is a Postgres array of `segment_id` strings. `merge_segments` sets `status = 'merged'` on each source segment individually before calling `mint_segment` with the source list as `lineage=`. This means a merge is not atomic across all rows by default; callers must wrap it in a transaction if atomicity is required.

**Crosswalk edges**
Each edge links a `UnspscFamily` string to a `SegmentId` with a composite primary key `(family, segment_id)`. The `quantity_refs` JSONB column carries arbitrary references (e.g., quantity IDs) that the belief-lookup pipeline consults during generation. Upserts overwrite `quantity_refs` on conflict.

**Streams**
`Stream` stores `hub_org_id` (nullable), a Postgres array of `member_org_ids`, and a float `terminal_absorption`. The upsert fully replaces hub, members, and absorption on conflict, so the caller always provides the complete desired state.

**Cross-feature type authority**
`placer/identity/types.py` defines every canonical ID type consumed across the codebase. Types are `NewType` wrappers over primitives (`str` or `int`), ensuring zero runtime overhead while providing static-analysis safety. `ProductIdentity` and `CrosswalkEdgeKey` are Pydantic models defined here for product classification and crosswalk key addressing, respectively.

**Debug API surface**
`placer/api/debug.py` exposes identity data through read-only FastAPI handlers mounted under an `/debug` prefix. `list_orgs` queries the `orgs` table directly; `list_segments` and `list_crosswalk_edges` query `segments` and `crosswalk_edges` (with a JOIN to `segments` for definition text). None of these endpoints call `identity/store.py` functions — they issue raw SQL — so they bypass any business logic layered in the store.


## Availability — is it usable right now

**Store functions** — `resolve_org`, `mint_org`, `get_org`, `mint_segment`, `merge_segments`, `get_segment`, `upsert_crosswalk_edge`, `get_crosswalk_edges_for_family`, and `upsert_stream` — are all present in `placer/identity/store.py` and importable. No build flags or feature guards are in place. Callers must supply a live `psycopg.AsyncConnection`; database availability is a runtime dependency.

**Partial resolution coverage**: `resolve_org` only executes a lookup when `ein` is supplied. Name/address-based resolution (`ResolutionMethod.NAME_ADDRESS_BLOCK`, `ResolutionMethod.FUZZY_MATCH`) is declared in `types.py` but has no implementation in the store — passing `name` or `address` without `ein` always returns `None`.

**Debug API endpoints** — `GET /orgs`, `GET /segments`, and `GET /crosswalk` — exist in `placer/api/debug.py` and appear in the feature table. They are read-only, guard-free, and depend only on the database connection. There is no changelog entry confirming their deployment status; availability in a live environment cannot be verified from source alone.

**Type system** (`placer/identity/types.py`) is unconditionally available to all importing modules; no changelog entries describe pending removals or renames.

No changelog is configured for this project, so no in-flight migrations or planned breaking changes can be assessed.

