---
feature: Governance
group: New
last_synced: '2026-06-12'
last_commit: c66cac868aea2ad3d45474337649f15a7c7db058
anchors:
  tables:
  - events
  - members
  - system_params
  endpoints: []
  types:
  - EnsembleMember
  - EnsembleMemberStatus
  - MemberTogglePayload
  - MemberType
  - ModelReleasePayload
  - OrgRemapPayload
  - ParamChangePayload
  - Provenance
  - TrustTier
  api_modules:
  - placer.events
  - placer.learn
  files:
  - placer/events/store.py::append
  - placer/events/types.py::EventKind
  - placer/events/types.py::MemberTogglePayload
  - placer/events/types.py::ParamChangePayload
  - placer/governance/**
  - placer/governance/store.py
  - placer/learn/types.py::EnsembleMember
  - placer/learn/types.py::EnsembleMemberStatus
  - placer/learn/types.py::MemberType
writes:
- members
- system_params
reads:
- placer/events/types.py
- placer/governance/store.py
---
## Capability — what it can do

Governance provides the **only legal write paths** for two protected tables: `system_params` (Tier 4 — system-level configuration) and `members` (Tier 3 — the predictor/proposer ensemble registry). Every mutation is event-sourced: a `system.param_change` or `system.member_toggle` event is appended to the event spine *before* the derived row is updated, and the row carries a back-reference (`updated_by_event`) to that event sequence number.

The store exposes five async functions:

| Function | Table | Description |
|---|---|---|
| `get_param` | `system_params` | Point-read a parameter by key; returns raw JSONB value. |
| `set_param` | `system_params` | Evented upsert: captures old value, fires `system.param_change`, then upserts the row. Returns the event `seq`. |
| `list_members` | `members` | Returns all registered `EnsembleMember` records, optionally filtered by `MemberType`. |
| `register_member` | `members` | Evented insert of a new ensemble member; fires `system.member_toggle` with `from_status="unregistered"`. Ignores conflicts (`ON CONFLICT DO NOTHING`). Returns event `seq`. |
| `toggle_member` | `members` | Evented status transition (`active` → `suspended` → `retired`); raises `ValueError` for unknown members. Returns event `seq`. |

All writes tag provenance as `source="admin_endpoint"` with trust tier `DECLARED_BY_ORG`. Bypassing these helpers and writing directly to either table violates spec V2 §VIII.3: an unevented parameter change corrupts every downstream comparison that spans it.

Ensemble members span four governed kinds defined in `placer.learn.types.MemberType`: `proposer`, `predictor`, `fold`, and `controller`.

## Implementation — how it works

`placer/governance/store.py` is a pure async repository layer backed by `psycopg.AsyncConnection`. It has no HTTP handlers of its own; it is a callable library consumed by whatever admin surface invokes it.

**Event-first write protocol** (spec V2 §VIII.3): both `set_param` and the two member-write helpers call `placer.events.store.append` first. The `append` function validates the payload dict against the registered Pydantic model (`ParamChangePayload` for `system.param_change`, `MemberTogglePayload` for `system.member_toggle`) before the INSERT into the `events` table. Only after `append` returns a sequence number does the governance store execute the DML against `system_params` or `members`. The `seq` value is embedded in the `updated_by_event` column of `system_params`, creating a verifiable causal link between the row state and its authorising event.

**`set_param` detail**: reads the current value with `get_param` to populate `old_value` in the event payload, then issues a JSONB-typed upsert (`::jsonb` cast on the value column). The param value is serialised via `json.dumps` before the cast.

**`register_member` vs `toggle_member`**: both emit `SYSTEM_MEMBER_TOGGLE` events. `register_member` hard-codes `from_status="unregistered"` and uses `ON CONFLICT (member_id) DO NOTHING` to remain idempotent. `toggle_member` fetches the current status first, surfacing it in the event payload for audit continuity.

**Type dependencies**: `EnsembleMember`, `EnsembleMemberStatus`, and `MemberType` are defined in `placer/learn/types.py` (the learn feature's type module) rather than a dedicated governance types file. `EventKind`, `Provenance`, and `TrustTier` come from `placer/events/types.py`.

## Availability — is it usable right now

`placer/governance/store.py` is present and fully implemented. However, **no HTTP route, API endpoint, or calling module referencing `placer.governance`** was found anywhere in the codebase — the `/placer/governance` route listed in the feature table has no confirmed handler and `governance.store` is not imported by any other module. The store functions are reachable only if an admin surface (not yet present in the code) or a test wires up a database connection and calls them directly.

Consequently:
- The **read path** (`get_param`, `list_members`) is functional as library code but not exposed over any network interface.
- The **write paths** (`set_param`, `register_member`, `toggle_member`) are implemented and enforce the event-first invariant correctly in isolation, but are similarly unreachable without a calling layer.
- No changelog entries exist to clarify intended release timing.

The feature should be considered **library-complete but not yet deployed** — the store exists and is correct, but the admin surface needed to invoke it is absent.
