---
feature: Ingest
group: New
last_synced: '2026-06-11'
last_commit: 07baa96d58d04d94add2aabddffd1dfdd90193e9
anchors:
  tables:
  - events
  - orders
  endpoints:
  - POST /debug/ingest-order
  types:
  - EntityRefs
  - EventKind.INGEST_ORDER
  - IngestOrderPayload
  - IngestOrderResult
  - ManualItemInput
  - ManualOrderInput
  - OrderConstraints
  - Provenance
  - RawItem
  api_modules:
  - frontend/src/api.ts::api.ingestOrder
  files:
  - frontend/src/views/**
  - frontend/src/views/Ingest.tsx
  - placer/api/debug.py::ingest_order_manual
  - placer/events/store.py::append
writes:
- frontend/src/views/Ingest.tsx
- placer/api/debug.py::ingest_order_manual
reads: []
---
## Capability — what it can do

The Ingest view is a manual order-creation tool. It lets a dashboard user compose and submit a fully-structured donation-matching order directly from the browser without going through an external adapter or message queue.

The form captures the complete set of fields that define an `ingest.order` event:

- **Identity & volume** — donor ID (free text, defaulting to `"manual-donor"`) and total pallet count.
- **Deadline** — optional datetime picker; omitting it leaves the deadline null.
- **Constraints** — three independent flags (perishable, temperature-controlled, hazmat class) plus a comma-separated donor-restrictions list that is split and trimmed client-side before submission.
- **Line items** — a dynamic list of `ManualItemInput` rows, each carrying a description, numeric quantity, and a unit chosen from `pallets / cases / lbs / units`. At least one row is always present; the list can be extended or (if more than one row exists) shrunk.
- **Notes** — optional free-text annotation.

On submission the component calls `api.ingestOrder(body)`, which POSTs a `ManualOrderInput` JSON body to `POST /debug/ingest-order`. The backend handler (`ingest_order_manual`) mints a random `order_id` in the form `manual-<12 hex chars>`, validates the payload against `IngestOrderPayload`, appends an `ingest.order` event to the event spine via `event_store.append`, and inserts a row into the `orders` table with `lifecycle = 'intake'` and `mode = 'placement'`. The response carries `order_id`, `seq` (the event spine sequence number), and a human-readable message.

If no line items have a non-empty description, the backend falls back to a synthetic item `"(no items specified)"` at the given pallet count, ensuring the payload is always valid.

On success the component renders a green confirmation badge showing the `order_id` and event `seq`. On failure it displays the raw HTTP error text.

## Implementation — how it works

**Frontend (`frontend/src/views/Ingest.tsx`)** — Pure React functional component using only `useState`. All form state is local; there is no global store interaction. The `handleSubmit` function constructs a `ManualOrderInput` object from local state, applies client-side transformations (comma-split for restrictions, empty-description filter for items), then delegates to `api.ingestOrder`. Response state is one of three variants: idle, `IngestOrderResult` (success), or string error.

**API client (`frontend/src/api.ts` → `api.ingestOrder`)** — Thin `POST` wrapper using the shared `post<T>` helper. Base URL is `""` (same-origin), so the request lands at the server's `/debug/ingest-order` route.

**Backend handler (`placer/api/debug.py::ingest_order_manual`)** — Registered under the `/debug` FastAPI router as `POST /debug/ingest-order`. The handler:
1. Generates a `manual-<hex>` `OrderId` via `uuid.uuid4()`.
2. Builds typed domain objects: `OrderConstraints`, `RawItem` list, and `IngestOrderPayload`.
3. Synthesises a default item if the submitted list is empty.
4. Constructs a `Provenance` record with `source="debug_dashboard"`, `trust_tier=TrustTier.DECLARED_BY_ORG`, and `actor="dashboard_user"` — hardcoded, not derived from any authentication context.
5. Calls `event_store.append` (in `placer/events/store.py::append`), which validates the payload against `IngestOrderPayload` via `model_validate`, then INSERTs into the `events` table and RETURNs the assigned `seq`.
6. INSERTs into the `orders` table within the same connection context (not the same transaction — the two writes are sequential but not atomic).
7. Returns `{ order_id, seq, message }`.

**Event-type chain** — `EventKind.INGEST_ORDER = "ingest.order"` → `IngestOrderPayload` (containing nested `OrderConstraints` and `list[RawItem]`), registered in `EVENT_PAYLOAD_MODELS` in `placer/events/types.py`. The store validates against this model before every insert.

Shared UI primitives used: `Card` (from `frontend/src/components/Card.tsx`) for the outer container and `Badge` (from `frontend/src/components/Table.tsx`) for the green "Created" label in the success state.

## Availability — is it usable right now

The route `/view/ingest` is **not listed** in the current feature table, which means it is absent from the application's registered navigation/routing configuration. The component file exists and is complete, and the backend endpoint `POST /debug/ingest-order` is present in `placer/api/debug.py`, but if the route is not wired into the router the view is unreachable from the dashboard.

There are no route guards declared for this path. There is no changelog configured, so no additional intent context is available.

There is no authentication on the backend endpoint — provenance is hardcoded to `trust_tier=DECLARED_BY_ORG` and `actor="dashboard_user"` regardless of who makes the request. Any caller with network access to the `/debug` router can ingest orders.

The two database writes (event spine append + `orders` INSERT) are not wrapped in a single transaction; a failure between them could leave an event with no corresponding order row.
