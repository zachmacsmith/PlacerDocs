---
feature: Members
group: New
first_commit: 6dc428c8cfbf577dc8254a42c8b1873db3babcd4
last_synced: '2026-06-15'
last_commit: 6dc428c8cfbf577dc8254a42c8b1873db3babcd4
anchors:
  tables:
  - members
  endpoints: []
  types:
  - BridgeFamily
  - BridgeMediator
  - CONDITIONAL_TRIGGERS
  - CORE_BATTERY
  - CanonicalizeResult
  - ConditionalTrigger
  - ElicitationMethod
  - EnsembleMember
  - EnsembleMemberStatus
  - Hypothesis
  - HypothesisProvenance
  - MEDIATOR_TO_FAMILY
  - MemberTogglePayload
  - MemberType
  api_modules:
  - placer/core/events.py
  - placer/core/member_registry.py
  - placer/learn/types.py
  files:
  - placer/core/member_registry.py::_admin_provenance
  - placer/core/member_registry.py::list_members
  - placer/core/member_registry.py::register_member
  - placer/core/member_registry.py::toggle_member
  - placer/members/**
  - placer/members/__init__.py
  - placer/members/proposers/__init__.py
  - placer/members/proposers/bridge_battery.py
writes:
- placer/core/member_registry.py
- placer/members/__init__.py
- placer/members/proposers/__init__.py
- placer/members/proposers/bridge_battery.py
reads:
- placer/core/events.py
- placer/core/member_registry.py
- placer/learn/types.py
- placer/members/__init__.py
- placer/members/proposers/__init__.py
- placer/members/proposers/bridge_battery.py
---
## Capability — what it can do

The Members feature defines and manages **T3 — the extensible member pool**: the governed registry of all ensemble participants that the Placer Bayesian placement controller permits to operate (spec V2 §VIII.3).

The pool tracks four distinct member kinds (via `MemberType`):
- **Proposer** — constructs candidate sets of recipient organisations.
- **Predictor** — scores registered belief quantities.
- **Fold** — participates in learning/validation fold runs.
- **Controller** — governs placement decisions.

Each registered `EnsembleMember` carries identity (`member_id`, `owner`, `version`), behavioural metadata (`estimand_conformity`, `prior_weight`, `current_weight`, `sunset_rule`), and a lifecycle status (`active`, `suspended`, `retired`).

The `placer/members/proposers/` sub-package groups the concrete proposer implementations. The bridge battery (`bridge_battery.py`) houses the founding proposers: 20 named `BridgeMediator` values organised into five `BridgeFamily` families (Consumption, People, Context, Institutional, Analogical), a four-mediator **core battery** that is always active, and a set of `ConditionalTrigger` rules that activate additional mediators based on item attributes (e.g. `cas_present`, `high_volume`).

## Implementation — how it works

**Evented write path.** All mutations to the `members` database table are routed exclusively through `placer/core/member_registry.py`. The module enforces the spec V2 §VIII.3 invariant: an event is appended to the sacred event spine *first*, then the derived `members` row is written. A mutation that skips the event would corrupt any downstream comparison that spans it.

- `register_member` — inserts a new pool member by emitting `system.member_toggle` (from `"unregistered"` → target status) and then `INSERT INTO members … ON CONFLICT DO NOTHING`.
- `toggle_member` — transitions an existing member between `active`, `suspended`, and `retired` by emitting `system.member_toggle` and then `UPDATE members SET status = …`.
- `list_members` — read-only query over the `members` table, optionally filtered by `MemberType`.

Both write functions use the `_admin_provenance` helper, which stamps events with `source="admin_endpoint"` and `TrustTier.DECLARED_BY_ORG`.

**Event payload.** Toggle events are validated against `MemberTogglePayload` (defined in `placer/core/events.py`), which captures `member_id`, `from_status`, `to_status`, `actor`, and `reason`.

**Bridge battery proposers.** `placer/members/proposers/bridge_battery.py` is self-contained: it declares the `BridgeFamily` / `BridgeMediator` enumerations, the `MEDIATOR_TO_FAMILY` lookup, `CORE_BATTERY` (four always-on mediators), `CONDITIONAL_TRIGGERS`, `ElicitationMethod`, and the `Hypothesis` / `CanonicalizeResult` output types. These types are consumed by the Generate pipeline when constructing hypotheses for candidate retrieval.

**Type ownership.** `EnsembleMember`, `EnsembleMemberStatus`, and `MemberType` are defined in `placer/learn/types.py` and imported by the registry — they are shared with the Learn feature.

## Availability — is it usable right now

The **data layer is fully implemented**: `placer/core/member_registry.py` provides working `list_members`, `register_member`, and `toggle_member` functions backed by the `members` PostgreSQL table.

The **bridge battery proposer types** (`BridgeFamily`, `BridgeMediator`, `CORE_BATTERY`, `CONDITIONAL_TRIGGERS`) are fully defined in `placer/members/proposers/bridge_battery.py` and are available for use by the Generate pipeline.

However, **no HTTP route or API endpoint for the Members pool is registered** in the feature table. The `/placer/members` route appears in the routing configuration but the component file (`placer/members/__init__.py`) contains only the spec-reference docstring — no handler, view, or UI component is present. The member registry write functions (`register_member`, `toggle_member`) are also not called from any discoverable caller in the codebase beyond their definition; there is no evidence they are wired to an admin API at this time.

**Summary:** The T3 member pool storage and type layer exist; the proposer battery types exist. Direct user- or operator-facing access to read or mutate the member pool via the `/placer/members` route is not yet implemented.
