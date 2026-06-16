---
feature: Governance
group: New
first_commit: c66cac868aea2ad3d45474337649f15a7c7db058
last_synced: '2026-06-15'
last_commit: 7fdd634f38a1f90503ccc57e81634980b0f2102f
anchors:
  tables:
  - members
  - system_params
  endpoints: []
  types: []
  api_modules:
  - placer.core
  - placer.learn
  files:
  - placer/core/**
writes:
- members
- system_params
reads:
- placer/core/member_registry.py
- placer/learn/types.py
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
| `toggle_member` | `members` | Evented status transition (`registered` → `shadow` → `live` → `retired`); raises `ValueError` for unknown members. Returns event `seq`. |

All writes tag provenance as `source="admin_endpoint"` with trust tier `DECLARED_BY_ORG`. Bypassing these helpers and writing directly to either table violates spec V2 §VIII.3: an unevented parameter change corrupts every downstream comparison that spans it.

Ensemble members span five governed kinds defined in `placer.learn.types.MemberType`: `proposer`, `predictor`, `fold`, `controller`, and `compiler` (added in V2.1 to handle declarative-statement compilation).

## Implementation — how it works

`placer/core/member_registry.py` is a pure async repository layer backed by `psycopg.AsyncConnection`. It has no HTTP handlers of its own; it is a callable library consumed by whatever admin surface invokes it.

**Event-first write protocol** (spec V2 §VIII.3): both `set_param` and the two member-write helpers call `placer.core.events.append` first. The `append` function validates the payload dict against the registered Pydantic model (`ParamChangePayload` for `system.param_change`, `MemberTogglePayload` for `system.member_toggle`) before the INSERT into the `events` table. Only after `append` returns a sequence number does the governance store execute the DML against `system_params` or `members`. The `seq` value is embedded in the `updated_by_event` column of `system_params`, creating a verifiable causal link between the row state and its authorising event.

**`set_param` detail**: reads the current value with `get_param` to populate `old_value` in the event payload, then issues a JSONB-typed upsert (`::jsonb` cast on the value column). The param value is serialised via `json.dumps` before the cast.

**`register_member` vs `toggle_member`**: both emit `SYSTEM_MEMBER_TOGGLE` events. `register_member` hard-codes `from_status="unregistered"` and uses `ON CONFLICT (member_id) DO NOTHING` to remain idempotent. `toggle_member` fetches the current status first, surfacing it in the event payload for audit continuity.

**Type dependencies**: `EnsembleMember`, `MemberStatus`, and `MemberType` are defined in the learn feature's type module rather than a dedicated governance types file. (`EnsembleMemberStatus` remains as a backward-compatibility alias for `MemberStatus` but is deprecated.) `EventKind`, `Provenance`, and `TrustTier` come from `placer/core/events.py`, the consolidated event spine module that also houses the `append` function itself.

## Availability — is it usable right now

`placer/core/member_registry.py` is present and fully implemented. However, **no HTTP route, API endpoint, or calling module referencing the governance functions** was found anywhere in the codebase — the `/placer/governance` route listed in the feature table has no confirmed handler and `placer.core.member_registry` is not imported by any other module. The store functions are reachable only if an admin surface (not yet present in the code) or a test wires up a database connection and calls them directly.

Consequently:
- The **read path** (`get_param`, `list_members`) is functional as library code but not exposed over any network interface.
- The **write paths** (`set_param`, `register_member`, `toggle_member`) are implemented and enforce the event-first invariant correctly in isolation, but are similarly unreachable without a calling layer.
- Migration `003_v21_constraints_attributes.sql` is present and updates `members.status` to the V2.1 lifecycle (`registered | shadow | live | retired`), but whether this migration has been applied to any running database is not determinable from code alone.
- `EnsembleMemberStatus` (the old type name) survives as a deprecated backward-compatibility alias for `MemberStatus`; callers should migrate to `MemberStatus`.
- No changelog entries exist to clarify intended release timing.

The feature should be considered **library-complete but not yet deployed** — the store exists and is correct, but the admin surface needed to invoke it is absent.
