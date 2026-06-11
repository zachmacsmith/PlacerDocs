---
feature: Beliefs
group: Placer
last_synced: '2026-06-11'
last_commit: 5499bc5a8c1f45be4e6cdc23b3f7414d926340f0
anchors:
  tables:
  - SET
  - belief_checkpoints
  - quantity_registry
  endpoints:
  - GET /beliefs/checkpoints
  - GET /beliefs/quantities
  types:
  - Belief
  - BeliefCheckpoint
  - BeliefIndexKey
  - BeliefQuery
  - BeliefStore
  - BetaStats
  - CapacityCoordinates
  - CureModelStats
  - EdgeIndexKey
  - FactorName
  - FitCoordinates
  - HistogramStats
  - IndexSchema
  - NormalStats
  - OrgChannelIndexKey
  - OrgIndexKey
  - OrgSizeChannelIndexKey
  - ProvenanceComponent
  - QuantityRegistryEntry
  - QuantityType
  - RespondCoordinates
  - ScalarStats
  - SegmentIndexKey
  - StreamIndexKey
  - SufficientStats
  - SystemIndexKey
  - WantCoordinates
  api_modules:
  - placer.api.debug
  - placer.beliefs
  - placer.beliefs.store
  - placer.beliefs.types
  - placer.contracts
  - placer.identity
  files:
  - placer/api/debug.py
  - placer/beliefs/**
  - placer/beliefs/store.py
  - placer/beliefs/types.py
  - placer/contracts.py
writes:
- SET
- belief_checkpoints
reads:
- belief_checkpoints
- quantity_registry
---
## Capability — what it can do

The Beliefs feature is the platform's Bayesian state-of-knowledge layer. It provides a **uniform belief interface** — a `(point, interval, n_eff, staleness_seconds, provenance_mix)` structure — that every downstream consumer (Value, Generate, Resolve, Projections) reads without knowledge of how the internal distribution is represented.

**Quantity registry.** Eight named quantities are pre-registered (`p_want`, `p_cap`, `cap_volume`, `p_resp`, `resp_time`, `p_fit`, `lambda_ops`, `stream_absorption`), each described by an estimand sentence, a `QuantityType`, and an index schema that governs which index keys are valid for it (org, edge, org_channel, stream, system, segment).

**Checkpoint persistence.** `save_checkpoint` upserts a `BeliefCheckpoint` row into the `belief_checkpoints` table keyed on `(quantity_id, index_key)`. `get_checkpoint` retrieves it. Each checkpoint carries `sufficient_stats` (a JSON blob), `n_eff`, version tags (`selector_version`, `fold_version`), and a `log_watermark` that records which event-log position the checkpoint was computed from.

**Sufficient-statistics representations.** Four conjugate families are supported: `BetaStats` (probability quantities), `NormalStats` (volume / duration / money), `CureModelStats` (time-to-response with never-responders), and `HistogramStats` (empirical distributions). A `ScalarStats` type handles deterministic overrides. The internal family is hidden behind the `Belief` interface; `sample()` dispatches to the correct distribution.

**Resolver skeleton (spec §5.4).** Three pure functions implement read-time belief resolution:
- `resolve_belief_from_checkpoint` — converts a single checkpoint to a `Belief`, returning a flat Beta(1,1) prior when no checkpoint exists.
- `resolve_inherited_belief` — computes a precision-weighted mixture of edge posteriors, applying variance-mixture approximation and a `scipy.stats` credible interval.
- `partially_pool` — blends an org's own-data belief with an inherited mixture, weighting by `n_own / (n_own + c)` (James–Stein-style partial pooling, default shrinkage constant `c = 5`).

**Conjugate fold helpers.** `beta_fold_success` and `beta_fold_failure` perform conjugate Beta updates in O(1).

**Debug API exposure.** Two read-only debug endpoints surface belief state: `GET /beliefs/quantities` (joined with `quantity_registry`) and `GET /beliefs/checkpoints` (filterable by `quantity_id`).

**Abstract contract.** `BeliefStore` in `placer/contracts.py` is the interface through which Generate and Value operators access beliefs; `store.py` provides the concrete persistence primitives that any `BeliefStore` implementation builds on.

## Implementation — how it works

**Storage layer (`placer/beliefs/store.py`).** The two async functions `save_checkpoint` / `get_checkpoint` use a raw `psycopg.AsyncConnection`. `save_checkpoint` issues an `INSERT … ON CONFLICT (quantity_id, index_key) DO UPDATE`, making every write idempotent. `index_key` and `sufficient_stats` are stored as `jsonb`; `log_watermark` is stored as a plain integer (cast from `EventSeq`).

**Type hierarchy (`placer/beliefs/types.py`).** All types are Pydantic `BaseModel` subclasses. `SufficientStats` is a discriminated union on the `kind` literal field. `BeliefIndexKey` is a parallel discriminated union on `schema`. The `Belief` model exposes `_stats` as a `PrivateAttr`; it is populated by the resolver functions and consumed only by `sample()`, preserving encapsulation across the public interface.

**Resolver pipeline.** `_parse_stats` deserialises the raw `dict` from the DB into the correct `SufficientStats` subtype. `_point_estimate` and `_variance` are pure `match`-dispatch helpers. `_interval` calls `scipy.stats.beta.ppf` for Beta posteriors and `scipy.stats.norm.ppf` for Normal ones. `_flat_prior` returns `BetaStats(α=1, β=1)` with `n_eff=0` and `staleness=∞`, serving as the universal fallback.

**Partial pooling formula.** `partially_pool(org, inherited, c=5.0)` computes `w_own = n_eff_own / (n_eff_own + c)`. Point estimate and interval bounds are linearly interpolated. Provenance components are re-weighted: own component gets `w_own`, each inherited component is scaled by `(1 − w_own)`.

**Contracts wiring (`placer/contracts.py`).** `BeliefStore` is an abstract class with two methods: `belief(query: BeliefQuery) → Belief` and `checkpoint(quantity_id, index_key) → BeliefCheckpoint | None`. `GenerateContract.generate` and `ValueContract.value` both receive a `BeliefStore` parameter, making the belief layer the shared read dependency for the core algorithmic operators.

**Debug API (`placer/api/debug.py`).** `list_quantities` joins `quantity_registry` with `belief_checkpoints` to return checkpoint counts and average `n_eff` per quantity. `list_checkpoints` queries `belief_checkpoints` directly with optional `quantity_id` filter, returning up to 500 rows ordered by `computed_at DESC`.

## Availability — current code state

**Persistence primitives** (`save_checkpoint`, `get_checkpoint`) are present in `placer/beliefs/store.py` and target the `belief_checkpoints` postgres table. These functions are importable and syntactically complete, but no call sites outside the file itself were found in the codebase — they are not yet invoked by any wired-up `BeliefStore` implementation that was confirmed in source.

**Resolver functions** (`resolve_belief_from_checkpoint`, `resolve_inherited_belief`, `partially_pool`) are defined and complete. No confirmed call sites were found outside `store.py` itself, indicating the resolver skeleton is implemented but not yet wired into a live `BeliefStore` concrete class.

**Type definitions** in `placer/beliefs/types.py` are fully present and imported by `placer/contracts.py`, confirming that the type contract is active across the platform.

**Debug API endpoints** `GET /beliefs/quantities` and `GET /beliefs/checkpoints` are registered in `placer/api/debug.py` under the `/debug` prefix and are available for read-only inspection of belief state, subject to the database being populated.

**`BeliefStore` abstract contract** is defined in `placer/contracts.py` and referenced as a parameter type in `GenerateContract` and `ValueContract`, but no concrete implementation class was found in the searched source. Downstream operators that depend on `BeliefStore` cannot function until a concrete implementation is provided.

**`scipy` dependency** is required at runtime by `_interval` (Beta and Normal credible intervals) and `resolve_inherited_belief` (Normal mixture intervals); absence of `scipy` in the environment would cause `ImportError` at belief-resolution time, not at import time.

No changelog was configured; no discrepancies between stated intent and code were identified.
