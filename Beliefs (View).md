---
feature: Beliefs (View)
group: New
last_synced: '2026-06-11'
last_commit: 07baa96d58d04d94add2aabddffd1dfdd90193e9
anchors:
  tables: []
  endpoints: []
  types: []
  api_modules: []
  files:
  - frontend/src/views/**
writes: []
reads:
- frontend/src/api.ts
- frontend/src/views/Ingest.tsx
---
## Capability — what it can do

The Beliefs view is a four-sub-tab read-only dashboard that exposes the Bayesian belief state of the placement controller across four related data domains:

- **Quantity Registry** — lists every belief quantity registered in the system. Each row shows the quantity ID, its type, the natural-language estimand sentence that defines what is being estimated, the index schema, the total number of checkpoints stored, and the rolling average effective sample size (`avg_n_eff`).

- **Checkpoints** — shows the persisted sufficient-statistics snapshots that back each belief quantity. A per-quantity dropdown filter (populated from the Quantity Registry data already in state) narrows the table. Rows are expandable in-place to reveal the full `index_key` and `sufficient_stats` JSON objects rendered via a scrollable `JsonBlock`. Each row also surfaces `n_eff`, `log_watermark`, and the timestamp the checkpoint was computed.

- **Crosswalk** — lists all product-family × demand-segment compounding edges. Each edge carries its family label, segment ID, a human-readable segment definition, an active/inactive status badge, and a map of quantity references for that edge.

- **Segments** — lists demand segments with their version, definition text, status (active or otherwise), lineage chain (parent segment IDs), and creation timestamp.

All four sub-tabs are read-only; no create, update, or delete operations are present in this view.

## Implementation — how it works

`Beliefs.tsx` is a single React function component (`Beliefs`) mounted at the frontend route `/view/beliefs`. It manages four independent slices of server state with `useState` and coordinates data fetching with two `useEffect` hooks.

**Data loading strategy.** The Quantity Registry is fetched unconditionally on mount via `api.quantities()` (→ `GET /debug/beliefs/quantities`) because the quantities list also populates the filter dropdown on the Checkpoints sub-tab. The remaining three sub-tabs are fetched lazily: a second `useEffect` keyed on `[subTab, quantityFilter]` fires only when the active sub-tab is `"checkpoints"`, `"crosswalk"`, or `"segments"`, calling `api.checkpoints()` (→ `GET /debug/beliefs/checkpoints?quantity_id=…`), `api.crosswalk()` (→ `GET /debug/crosswalk`), or `api.segments()` (→ `GET /debug/segments`) respectively. There is no cache invalidation or refetch-on-focus; data is fetched once per tab activation and re-fetched whenever `quantityFilter` changes on the Checkpoints sub-tab.

**Local state.** `subTab: SubTab` drives tab selection (`"quantities" | "checkpoints" | "crosswalk" | "segments"`). `quantityFilter: string` holds the quantity ID selected in the Checkpoints dropdown (empty string means "all"). `expandedCp: number | null` tracks the ID of the one checkpoint row currently expanded to show its JSON detail panel; clicking the same row again collapses it.

**Rendering.** The tab bar is rendered as a row of `<button>` elements with active-state styling. Each sub-tab panel is conditionally rendered (`{subTab === "…" && …}`). All four panels use the shared `Card` wrapper from `../components/Card`, `Table` / `Td` / `Badge` / `JsonBlock` / `EmptyState` from `../components/Table`, and the typed records `QuantityRecord`, `CheckpointRecord`, `CrosswalkEdge`, and `SegmentRecord` imported from `../api`. Badge colors encode semantic meaning: blue for quantity IDs, green for "active" statuses, and yellow for any other status value.

**API module.** All network calls go through the `api` singleton in `frontend/src/api.ts`, which issues plain `fetch` GET requests with no authentication headers — consistent with the debug-namespace endpoints being guard-free.

## Availability — is it usable right now

The component file `frontend/src/views/Beliefs.tsx` is present and complete. The feature table confirms the frontend route `/placer/beliefs` with feature name `beliefs` and no guards, indicating no authentication barrier is enforced at the routing layer.

All four API endpoints called by this view appear in the feature table as guard-free routes:
- `GET /beliefs/quantities` → `list_quantities`
- `GET /beliefs/checkpoints` → `list_checkpoints`
- `GET /crosswalk` → `list_crosswalk_edges`
- `GET /segments` → `list_segments`

The view is therefore reachable and its data calls are structurally wired. However, runtime availability of meaningful data depends on whether the backend has registered belief quantities, computed checkpoints, and minted crosswalk edges and segments — all of which are driven by upstream placement activity. Empty-state messages (`"No quantities registered."`, `"No checkpoints yet."`, `"No crosswalk edges yet."`, `"No segments minted yet."`) are displayed when the server returns empty collections.

No changelog is configured, so there is no documented intent context to reconcile against the current code state.
