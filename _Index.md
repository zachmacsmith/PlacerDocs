# Placer Documentation Index

> Bayesian placement controller for charity-donation matching

**35 documented features** across 9 groups. Source commit: `e75bcb47650c`

## Source directory map

```
placer/
  adapters/  → [[Adapters]]
  api/  → [[API]], [[Belief Checkpoints]], [[Belief Quantities]], [[Context Analysis]]
  beliefs/
  candidates/
  control/  → [[Control]], [[Controllers]]
  controllers/  → [[Control]], [[Controllers]]
  core/  → [[Beliefs]], [[Core]], [[Governance]]
  events/
  generate/
  governance/
  identity/  → [[Candidates]], [[Identity]]
  learn/  → [[Learn]]
  learning/
  members/  → [[Generate]], [[Members]], [[Resolve]]
  monitors/
  projections/  → [[Projections]]
  resolve/
  value/  → [[Value]]

frontend/src/
  views/  → [[Beliefs (View)]], [[Candidates (View)]], [[Events (View)]], [[Ingest]]
  components/
```

## Features by group


### Beliefs

- **[[Belief Checkpoints]]** — GET /beliefs/checkpoints exposes a paginated, read-only view of the belief_checkpoints table — the persistent snapshots that record the B...
  - tables: `belief_checkpoints`  —  endpoints: `GET /beliefs/checkpoints`
- **[[Belief Quantities]]** — GET /beliefs/quantities (served at the prefixed path /debug/beliefs/quantities) provides a read-only, aggregated catalogue of every quant...
  - tables: `belief_checkpoints`, `quantity_registry`  —  endpoints: `GET /beliefs/quantities`


### Events

- **[[Event Kinds]]** — GET /debug/events/kinds returns a frequency census of every distinct event_kind value that exists in the events table.
  - tables: `events`  —  endpoints: `GET /events/kinds`
- **[[List Events]]** — GET /debug/events returns a paginated, optionally filtered read-only view of the event spine — the append-only ledger that every downstre...
  - tables: `events`  —  endpoints: `GET /events`


### Matching

- **[[Context Analysis]]** — POST /context-analysis accepts a JSON body containing an EIN (Employer Identification Number) and returns a structured analysis of the co...
  - endpoints: `POST /context-analysis`
- **[[Recommendations]]** — POST /recommendations is the primary external-facing endpoint of the Placer platform.
  - endpoints: `POST /recommendations`


### Orders

- **[[List Orders]]** — GET /orders returns a paginated snapshot of placement orders held in the orders table, enriched with per-order candidate pipeline counts ...
  - tables: `candidates`, `orders`  —  endpoints: `GET /orders`
- **[[Order Detail]]** — GET /orders/{order_id} provides a complete, single-order view composed of three parts returned in one JSON response:.
  - tables: `candidates`, `orders`  —  endpoints: `GET /orders/{order_id}`


### Organizations

- **[[List Orgs]]** — GET /orgs exposes a paginated read of the org registry — the canonical set of charity organisations that Placer has encountered and resol...
  - tables: `orgs`  —  endpoints: `GET /orgs`


### Placer

- **[[API]]** — The API feature exposes two distinct groups of HTTP endpoints served by a single FastAPI application (placer/api/server.py).
  - tables: `belief_checkpoints`, `candidates`, `crosswalk_edges`, `events` +4  —  endpoints: `GET /beliefs/checkpoints`, `GET /beliefs/quantities` +9
- **[[Adapters]]** — The Adapters feature provides the sole schema-coupling boundary between Placer and Simpli's platform Postgres.
- **[[Beliefs]]** — The Beliefs feature is the platform's Bayesian state-of-knowledge layer.
- **[[Candidates]]** — The Candidates feature is the persistence and lifecycle backbone for the placement funnel.
  - tables: `candidates`, `orders`
- **[[Control]]** — The Control feature defines the stateless controller harness that drives every placement order through its lifecycle.
- **[[Generate]]** — The Generate feature defines the full type vocabulary for hypothesis generation — the pipeline stage that converts a donated-item descrip...
- **[[Identity]]** — The Identity feature is the canonical namespace layer for all entities in the Placer system.
  - tables: `crosswalk_edges`, `orgs`, `segments`, `streams`
- **[[Learn]]** — The Learn feature is the system's Bayesian update and validation harness (spec §8).
- **[[Projections]]** — The Projections module defines five distinct read-only views — called projections — that can be computed over an OrderWorldState without ...
- **[[Resolve]]** — The Resolve feature is workflow stage 3–4 of the placement pipeline (spec §3).
- **[[Value]]** — The Value feature defines the pure valuation function for the placement controller: given a candidate organisation, the current order wor...


### Segments

- **[[Crosswalk Edges]]** — The Crosswalk Edges feature exposes a read-only view of the crosswalk_edges table, which records the mapping between UNSPSC product famil...
  - tables: `crosswalk_edges`, `segments`  —  endpoints: `GET /crosswalk`
- **[[List Segments]]** — GET /segments returns an optionally filtered, top-N snapshot of all segment records stored in the segments table.
  - tables: `segments`  —  endpoints: `GET /segments`


### System

- **[[Health Check]]** — GET /health is a lightweight liveness probe for the Placer service.
  - endpoints: `GET /health`
- **[[System Stats]]** — GET /stats returns a single, consolidated snapshot of the entire Placer database in one call.
  - tables: `events`  —  endpoints: `GET /stats`


### New

- **[[Beliefs (View)]]** — The Beliefs view is a four-sub-tab read-only dashboard that exposes the Bayesian belief state of the placement controller across four rel...
- **[[Candidates (View)]]** — The Candidates view is a two-tab inspection panel for the matching pipeline's candidate pool.
- **[[Controllers]]** — The Controllers module defines the trigger-and-action harness that governs how the Placer system advances the state of a donation-matchin...
- **[[Core]]** — The core package is the constitutional layer (spec V2 §VIII.3) of the Placer platform.
- **[[Events (View)]]** — The Events view provides a paginated, filterable read-only log of every record stored in the events table — the central spine of the Plac...
- **[[Governance]]** — Governance provides the only legal write paths for two protected tables: system_params (Tier 4 — system-level configuration) and members ...
  - tables: `members`, `system_params`
- **[[Ingest]]** — The Ingest view is a manual order-creation tool.
  - tables: `events`, `orders`  —  endpoints: `POST /debug/ingest-order`
- **[[Ingest Order Manual]]** — POST /debug/ingest-order provides a manual pathway for creating a donation-placement order directly through the debug API, without going ...
  - tables: `orders`  —  endpoints: `POST /ingest-order`
- **[[Members]]** — The Members feature defines and manages the T3 extensible member pool — the governed registry of all participants in Placer's Bayesian en...
- **[[Orders]]** — The Orders view provides a two-panel browser over the full set of placement orders managed by the Placer controller.
- **[[Overview]]** — The Overview view is the top-level health and inventory dashboard for the Placer platform.

## Data flow — table access

| Table | Writers | Readers |
|-------|---------|---------|
| `belief_checkpoints` |  | [[API]], [[Belief Checkpoints]], [[Belief Quantities]] |
| `candidates` | [[Candidates]] | [[API]], [[List Orders]], [[Order Detail]] |
| `crosswalk_edges` | [[Identity]] | [[API]], [[Crosswalk Edges]] |
| `events` |  | [[API]], [[Event Kinds]], [[Ingest]], [[List Events]], [[System Stats]] |
| `members` | [[Governance]] |  |
| `orders` | [[API]], [[Ingest Order Manual]] | [[Candidates]], [[Ingest]], [[List Orders]], [[Order Detail]] |
| `orgs` | [[Identity]] | [[API]], [[List Orgs]] |
| `quantity_registry` |  | [[API]], [[Belief Quantities]] |
| `segments` | [[Identity]] | [[API]], [[Crosswalk Edges]], [[List Segments]] |
| `streams` | [[Identity]] |  |
| `system_params` | [[Governance]] |  |
