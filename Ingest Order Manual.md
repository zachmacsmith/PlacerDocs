---
feature: Ingest Order Manual
group: New
first_commit: 07baa96d58d04d94add2aabddffd1dfdd90193e9
last_synced: '2026-06-15'
last_commit: 6dc428c8cfbf577dc8254a42c8b1873db3babcd4
anchors:
  tables:
  - orders
  endpoints:
  - POST /ingest-order
  types:
  - ManualOrderInput
  api_modules:
  - placer.core
  - placer.db
  files:
  - placer/api/debug.py::ingest_order_manual
writes:
- orders
reads:
- placer/api/debug.py
- placer/api/server.py
---
## Capability — what it can do

`POST /debug/ingest-order` provides a manual pathway for creating a donation-placement order directly through the debug API, without going through an upstream donor system. A caller submits donor identity, volume, logistics constraints, and a list of raw items; the endpoint synthesises a fully-formed `ingest.order` event and a corresponding `orders` row in one atomic database transaction, returning the new `order_id` and the event spine sequence number.

The request body is validated against the `ManualOrderInput` Pydantic model, which accepts:
- **`donor_id`** — arbitrary donor identifier (defaults to `"manual-donor"`).
- **`total_pallets`** — numeric volume in pallets.
- **`deadline`** — optional ISO-8601 datetime after which the order expires.
- **`perishable`**, **`hazmat_class`**, **`temperature_controlled`** — logistics constraints folded into `OrderConstraints`.
- **`donor_restrictions`** — list of free-text restriction strings.
- **`items`** — list of `ManualItemInput` objects (`description`, `quantity`, `unit`). When the list is empty the endpoint synthesises a single fallback item using `total_pallets` and the unit `"pallets"`.
- **`notes`** — informational only; stored nowhere persistent, present for caller convenience.

The endpoint sets provenance `source="debug_dashboard"` and `trust_tier=TrustTier.DECLARED_BY_ORG`, fixing the data-quality tier for all manually ingested orders. The generated `order_id` follows the pattern `manual-<12 hex chars>`, making manual orders distinguishable in queries.

## Implementation — how it works

The handler `ingest_order_manual` in `placer/api/debug.py` is mounted on the `debug` `APIRouter` (prefix `/debug`) and registered with the FastAPI application via `app.include_router(debug_router)` in `placer/api/server.py`. No authentication guard is applied to this router or this endpoint.

**Execution sequence:**

1. A new `OrderId` is minted as `manual-<uuid4 hex[:12]>` using Python's `uuid` module and the `OrderId` NewType from `placer/core/ids.py`.
2. `OrderConstraints` and `RawItem` typed objects are constructed from the request body. If `body.items` is empty, a synthetic `RawItem(description="(no items specified)", ...)` is created.
3. An `IngestOrderPayload` is assembled and serialised via `.model_dump(mode="json")`.
4. A `Provenance` object is constructed with hardcoded values: `source="debug_dashboard"`, `trust_tier=TrustTier.DECLARED_BY_ORG`, `adapter_version="manual"`, `actor="dashboard_user"`.
5. A single database connection is opened via `get_conn()`. Within that connection:
   - `append()` (imported as `event_append` from `placer/core/events.py`) validates the payload against the registered `IngestOrderPayload` model, then `INSERT`s into the `events` table and returns the auto-assigned `seq`.
   - An `INSERT INTO orders` statement creates the order row with `lifecycle='intake'` and `mode='placement'`, setting both `total_volume` and `v_remaining` to `body.total_pallets`.
6. The response returns `order_id`, `seq` (cast to `int`), and a human-readable `message`.

Both writes share the same `psycopg.AsyncConnection` but are issued as sequential `execute` calls — there is no explicit `BEGIN`/`COMMIT` transaction wrapper in the handler; commit behaviour is governed by the `get_conn()` context manager's default settings.

## Availability — is it usable right now

The endpoint is **present and functional** in the current codebase. It is registered unconditionally in `placer/api/server.py` — no feature flag, no API-key guard, and no authentication middleware protects it. Any client that can reach the running FastAPI server can call `POST /debug/ingest-order`.

The route is scoped to the `/debug` prefix alongside all other debug inspection endpoints; it is intended for testing and dashboard use rather than production ingestion workflows. The real production ingestion path (fed by upstream donor adapters) is not implemented in this codebase at the time of this writing — `POST /recommendations` returns an empty stub — making this manual endpoint the **only operational way** to introduce a new order into the system.
