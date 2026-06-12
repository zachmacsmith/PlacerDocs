---
feature: Identity
group: Placer
first_commit: 5499bc5a8c1f45be4e6cdc23b3f7414d926340f0
last_synced: '2026-06-11'
last_commit: 87dd52f08e97ba92e8de49eace545f1073d264af
anchors:
  tables:
  - crosswalk_edges
  - orgs
  - segments
  - streams
  endpoints:
  - GET /debug/crosswalk
  - GET /debug/orgs
  - GET /debug/segments
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
  - placer/contracts.py
  - placer/identity/**
  - placer/identity/store.py
  - placer/identity/types.py
  - placer/api/debug.py::list_crosswalk_edges
  - placer/api/debug.py::list_orgs
  - placer/api/debug.py::list_segments
writes:
- SET
- crosswalk_edges
- orgs
- placer/identity/store.py
- segments
- streams
reads:
- placer/api/debug.py
- placer/api/server.py
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
`placer/identity/types.py` is the authoritative NewType and Pydantic-model source imported by at least ten other modules across the codebase: `adapters` (three files), `beliefs`, `candidates`, `control`, `contracts`, `events`, `generate`, and `resolve`. Key NewTypes include `OrgId`, `Ein`, `SegmentId`, `StreamId`, `OrderId`, `EventSeq`, `CandidateId`, `QuantityId`, `HypothesisId`, `DonorId`, `ContactId`, `UnspscFamily`, and `UnspscClass`. These wrappers prevent cross-space confusion at call sites. `OrgSnapshot` is declared in the type system but is not currently consumed by any store function.

**Read-only debug API**
Three debug endpoints in `placer/api/debug.py` expose identity data for inspection: `GET /debug/orgs` (ordered listing of orgs by first-seen sequence, returning raw JSONB memberships), `GET /debug/segments` (filterable by status), and `GET /debug/crosswalk` (filterable by UNSPSC family, joining segment definition text and status).

## Implementation — how it works

**Persistence model**
All four identity spaces are backed by PostgreSQL tables accessed via `psycopg.AsyncConnection`. Four tables are implicated: `orgs`, `segments`, `crosswalk_edges`, and `streams`. The store never opens its own connection — every function accepts an injected `psycopg.AsyncConnection[Any]`, making transaction boundaries the caller's responsibility.

**Organization record structure**
`OrgRecord` is a Pydantic model containing `org_id`, optional `ein`, a `ResolutionConfidence` enum (`high | medium | provisional`), the integer `EventSeq` of first encounter, and a nested `OrgMemberships` object. `OrgMemberships` carries a list of `SegmentMembership` records (each with `weight` and `confidence` floats), stream affiliations, an optional `OrgRole` enum (`hub | intermediate | terminal`), an NTEE code, and a `SizeBand` enum. This compound structure is persisted as JSONB in the `memberships` column and validated through `OrgMemberships.model_validate` on read. The debug API returns the raw JSONB dict rather than a validated model.

**Resolution confidence and method**
`ResolutionConfidence` (`high | medium | provisional`) and `ResolutionMethod` (`ein_join | name_address_block | fuzzy_match | provisional`) are `StrEnum` types stored as plain strings in the `orgs.resolution_confidence` column. Only `EIN_JOIN` resolution is active in `resolve_org`; the `name_address_block` and `fuzzy_match` methods are declared in the type system but have no corresponding implementation branches.

**Segment versioning and lineage**
Segments carry an integer `version` field and a `lineage` list of `SegmentId`s stored as a PostgreSQL array. `mint_segment` always writes `version=1`; the version field is read back from the database on `get_segment` but is not incremented on merge — the lineage array is the primary derivation record. `SegmentStatus` is a `StrEnum` with two values: `active` and `merged`.

**Crosswalk persistence**
`crosswalk_edges` uses a composite primary key of `(family, segment_id)`. The `quantity_refs` column is JSONB, serialized with `json.dumps`. The debug endpoint joins `crosswalk_edges` to `segments` to co-locate segment definition text alongside each edge.

**Stream persistence**
`streams` uses `stream_id` as the primary key. `member_org_ids` is stored as a PostgreSQL text array (cast via list comprehension to `str`). `terminal_absorption` is a nullable float. All three mutable fields are overwritten on conflict, making every `upsert_stream` call a full replacement.

**API layer**
The debug router (`prefix="/debug"`) is registered directly on the FastAPI `app` instance in `placer/api/server.py` with no additional path prefix. It shares the same lifespan-managed connection pool as the primary recommendation endpoints.

## Availability — what is usable right now

**Store functions** (`placer/identity/store.py`): All six read/write functions — `resolve_org`, `mint_org`, `get_org`, `mint_segment`, `merge_segments`, `get_segment`, `upsert_crosswalk_edge`, `get_crosswalk_edges_for_family`, `upsert_stream` — are present and importable. They require a live `psycopg.AsyncConnection` and depend on the `orgs`, `segments`, `crosswalk_edges`, and `streams` tables existing in the database. No migration tooling was confirmed in scope; table presence is a deployment prerequisite.

**Resolution completeness**: `resolve_org` only resolves via EIN lookup. Callers passing only `name` or `address` will always receive `None`. The `name_address_block` and `fuzzy_match` resolution methods are defined as `StrEnum` values but have no active code paths.

**Type system** (`placer/identity/types.py`): All NewTypes and Pydantic models are importable and in active use across ten modules. `OrgSnapshot` is defined but not used by any store function — it is available as a type but has no persistence surface.

**Debug API endpoints**: `GET /debug/orgs`, `GET /debug/segments`, and `GET /debug/crosswalk` are registered on the live FastAPI app and reachable without authentication (the debug router carries no auth guard). Availability depends on the server being running and the connection pool being initialized. These endpoints are read-only and return raw database rows; they are not part of the documented external contract (`/recommendations`, `/context-analysis`).
