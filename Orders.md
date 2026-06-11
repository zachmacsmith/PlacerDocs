---
feature: Orders
group: New
last_synced: '2026-06-11'
last_commit: ba02630c316c435e071a627f433a21d08f9987e7
anchors:
  tables: []
  endpoints:
  - GET /debug/orders
  - GET /debug/orders/{order_id}
  types:
  - CandidateRecord
  - OrderDetail
  - OrderSummary
  api_modules:
  - api.order
  - api.orders
  files:
  - frontend/src/api.ts
  - frontend/src/components/Card.tsx
  - frontend/src/components/Table.tsx
  - frontend/src/views/**
  - frontend/src/views/Orders.tsx
writes: []
reads:
- GET /debug/orders
- GET /debug/orders/{order_id}
---
## Capability — what it can do

The Orders view provides a two-panel browser over the full set of placement orders managed by the Placer controller.

**Order list panel** — renders every order returned by `GET /debug/orders` as a compact selectable row. Each row shows the order's ID (truncated monospace), its lifecycle badge, total volume in pennies (`p`), candidate count, and contacted count. Selection is purely client-side; clicking a row triggers a second fetch for that order's detail.

**Order detail panel** — populated on demand via `GET /debug/orders/{order_id}`. It exposes:

- **Four stat cards**: Total Volume, Remaining volume, Contacted count, Resolved count — all drawn from the `OrderSummary` embedded in `OrderDetail`.
- **Stage Distribution card**: iterates the `stage_counts` map returned by the backend and renders each stage as a colour-coded `Badge` with its integer count.
- **Candidates table**: lists every `CandidateRecord` for the order with columns for Org ID, Stage (badge), stage reason, the number of surfacing bridges, and the `contact_index` scalar extracted from `valuation_snapshot` (displayed as `ι=<value>`).

**Lifecycle colour semantics** are applied by the `lifecycleColor` helper: `active` → green, `fulfilling` → blue, `closed`/unknown → zinc, `rejected_scoped`/`expired` → red, `intake`/`feasibility_triage` → yellow.

**Candidate stage colour semantics** are applied by `stageColor`: `surfaced` → zinc, `screened_out` → red, `valued` → blue, `planned` → yellow, `contacted` → purple, `resolved` → green.

Both panels render `EmptyState` fallback messages when there are no orders or no candidates.


## Implementation — how it works

`Orders` is a React functional component at `frontend/src/views/Orders.tsx`. It maintains two pieces of local state via `useState`:

- `orders: OrderSummary[]` — populated once on mount via a `useEffect` that calls `api.orders()` (`GET /debug/orders`), destructuring the `orders` array from the response envelope.
- `selected: OrderDetail | null` — set by `selectOrder(id)`, which calls `api.order(id)` (`GET /debug/orders/{order_id}`) and stores the full `OrderDetail` response.

The API layer lives in `frontend/src/api.ts`. Both calls are plain `fetch` GETs with no auth headers; they share the module-level `get<T>` helper that throws on non-OK responses. The `OrderSummary` and `OrderDetail` types are exported from that same module and consumed directly by the component.

Layout is a CSS-grid split: one column for the list (`lg:col-span-1`) and two columns for the detail panel (`lg:col-span-2`), collapsing to a single column on smaller viewports. Shared presentational primitives (`Card`, `StatCard`, `Table`, `Td`, `Badge`, `EmptyState`) are imported from `frontend/src/components/Card.tsx` and `frontend/src/components/Table.tsx`.

The `valuation_snapshot` field is typed as `Record<string, unknown> | null`; the component reads the nested `contact_index` key via a type-cast and converts it to string for display, with a `"?"` fallback if the key is absent.

There is no pagination, sorting, filtering, or polling — the full order list is fetched once and detail is fetched per-selection without caching.


## Availability — is it usable right now

The component file `frontend/src/views/Orders.tsx` is present in the codebase and wired to the route `/view/orders` in the feature table. No route guards are registered for this route.

Both backing API endpoints are confirmed in the feature table and in `frontend/src/api.ts`:

- `GET /debug/orders` — feature `list_orders`, no guard.
- `GET /debug/orders/{order_id}` — feature `get_order`, no guard.

The view is available to any user who can reach the frontend; no authentication or capability check is enforced at either the route or the API layer. Code presence alone cannot confirm whether the backend handlers are fully operational in a given deployment — runtime availability depends on whether the backend service is reachable and the database contains order records.

No changelog entries exist for this feature.

