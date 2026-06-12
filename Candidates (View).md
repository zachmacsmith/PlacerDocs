---
feature: Candidates (View)
group: New
first_commit: ba02630c316c435e071a627f433a21d08f9987e7
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
- frontend/src/views/Candidates.tsx
---
## Capability — what it can do

The Candidates view is a two-tab inspection panel for the matching pipeline's candidate pool.

**"Candidates by Order" sub-tab**
- Presents a dropdown listing every known `OrderSummary` (order ID, lifecycle state, and candidate count), populated from `GET /debug/orders`.
- On order selection, fetches the full `OrderDetail` from `GET /debug/orders/{order_id}` and renders each `CandidateRecord` in a sortable table with columns: candidate ID (truncated), org ID, pipeline stage (colour-coded badge), stage reason, bridge count (`surfaced_by` array length), contacted date, and resolved date.
- A live stage-count summary bar appears alongside the order selector whenever candidates are present, showing a coloured `Badge` and count for every distinct stage present in the loaded set.
- Each candidate row is click-to-expand, revealing two collapsible JSON panels: **Surfaced By** (the list of bridges that surfaced the candidate) and, when present, **Valuation Snapshot** (the raw valuation object attached at the `valued` stage).

**"Org Registry" sub-tab**
- Fetches `GET /debug/orgs` (lazy-loaded only when the tab is first activated) and renders every `OrgRecord` with columns: org ID, EIN, resolution confidence (colour-coded `high`/`medium`/low badge), first-seen event sequence number, and raw membership JSON.
- The subtitle notes that rows are minted on first encounter, communicating the lazy-write nature of the identity store.

**Stage colour semantics** are defined locally in `stageColor()` and map the six canonical pipeline stages — `surfaced`, `screened_out`, `valued`, `planned`, `contacted`, `resolved` — to distinct badge colours (zinc, red, blue, yellow, purple, green respectively).

## Implementation — how it works

`Candidates` is a single React function component in `frontend/src/views/Candidates.tsx`. It manages all data locally with `useState` and side-effects with `useEffect` — there is no global state store or caching layer.

**Data loading strategy**
- `orders` is fetched unconditionally on mount via `api.orders()` → `GET /debug/orders`.
- `orgs` is fetched lazily in a `useEffect` that fires only when `subTab === "orgs"`, keeping the org query off the critical path for the default view.
- `candidates` is fetched whenever `selectedOrder` changes (non-empty) via `api.order(selectedOrder)` → `GET /debug/orders/{order_id}`, which returns `OrderDetail` whose `candidates` array is stored directly. Deselecting the order clears the array.

**Stage count derivation** is a pure in-render reduce over the `candidates` array (`stageCounts` object), recomputed on every render from the current slice — no memoisation.

**Expand/collapse** is controlled by a single `expandedId: string | null` state. Clicking a row toggles between setting `expandedId` to that row's `candidate_id` and clearing it; only one row can be expanded at a time. The expanded detail is rendered as a second `<tr>` with `colSpan={7}` containing `JsonBlock` components from `../components/Table`.

**Component imports**
- `Card` — section container with optional title/subtitle from `../components/Card`.
- `Table`, `Td`, `Badge`, `JsonBlock`, `EmptyState` — all from `../components/Table`.
- `api`, `OrderSummary`, `CandidateRecord`, `OrgRecord` — all from `../api`.

**API module** calls `GET /debug/orders` (list), `GET /debug/orders/{order_id}` (detail with candidates), and `GET /debug/orgs`. All three are unauthenticated debug endpoints (no API-key guard).

## Availability — is it usable right now

The component file `frontend/src/views/Candidates.tsx` is present and complete. The feature table confirms the route `/placer/candidates` is registered with the `candidates` component and carries no guards, so no authentication is required to reach it.

All three backing API endpoints it calls — `GET /debug/orders`, `GET /debug/orders/{order_id}`, and `GET /debug/orgs` — are listed in the feature table as `list_orders`, `get_order`, and `list_orgs` respectively, all with no guards.

No changelog is configured, so there is no documented intent to gate or disable this feature. **The view is available** for use, subject to the backend debug endpoints being reachable and populated with data (an empty order list or empty org registry will render empty-state messages rather than errors).
