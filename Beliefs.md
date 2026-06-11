---
feature: Beliefs
group: Placer
last_synced: '2026-06-11'
last_commit: 87dd52f08e97ba92e8de49eace545f1073d264af
anchors:
  tables:
  - SET
  - belief_checkpoints
  - quantity_registry
  endpoints:
  - GET /debug/beliefs/checkpoints
  - GET /debug/beliefs/quantities
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
  - FactorCoordinates
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

**Quantity registry.** Eight named quantities are pre-registered (`p_want`, `p_cap`, `cap_volume`, `p_resp`, `resp_time`, `p_fit`, `lambda_ops`, `stream_absorption`), each described by an estimand sentence, a `QuantityType`, and an `IndexSchema` that governs which index keys are valid for it (org, edge, org_channel, stream, system, segment).

**Checkpoint persistence.** `save_checkpoint` upserts a `BeliefCheckpoint` row into the `belief_checkpoints` table keyed on `(quantity_id, index_key)`. `get_checkpoint` retrieves it. Each checkpoint carries `sufficient_stats` (a JSON blob), `n_eff`, version tags (`selector_version`, `fold_version`), and a `log_watermark` that records which event-log position the checkpoint was computed from.

**Sufficient-statistics representations.** Five conjugate families are defined: `BetaStats` (probability quantities), `NormalStats` (volume / duration / money), `CureModelStats` (time-to-response with never-responders), `HistogramStats` (empirical distributions), and `ScalarStats` (deterministic overrides). The internal family is hidden behind the `Belief` interface; `sample()` dispatches to the correct distribution.

**Resolver skeleton (spec §5.4).** Three pure functions implement read-time belief resolution:
- `resolve_belief_from_checkpoint` — converts a single checkpoint to a `Belief`, returning a flat Beta(1,1) prior when no checkpoint exists.
- `resolve_inherited_belief` — computes a precision-weighted mixture of edge posteriors, applying a variance-mixture approximation and a `scipy.stats` credible interval.
- `partially_pool` — blends an org's own-data belief with an inherited mixture, weighting by `n_own / (n_own + c)` (James–Stein-style partial pooling, default shrinkage constant `c = 5`).

**Conjugate fold helpers.** `beta_fold_success` and `beta_fold_failure` perform conjugate Beta updates in O(1).

**Factor coordinate systems.** `WantCoordinates`, `CapacityCoordinates`, `RespondCoordinates`, and `FitCoordinates` define per-factor inheritance geometry (spec §5.4), unified under the `FactorCoordinates` discriminated union.

**Debug API exposure.** Two read-only debug endpoints surface belief state: `GET /beliefs/quantities` (joined with `quantity_registry`, returning checkpoint counts and average `n_eff` per quantity) and `GET /beliefs/checkpoints` (filterable by `quantity_id`, paginated up to 500 rows).

**Abstract contract.** `BeliefStore` in `placer/contracts.py` is the interface through which Generate and Value operators access beliefs; its two abstract methods are `belief(query: BeliefQuery) -> Belief` and `checkpoint(quantity_id, index_key) -> BeliefCheckpoint | None`. `store.py` provides the concrete persistence primitives that any `BeliefStore` implementation builds on.

## Implementation — how it works

**Storage layer (`placer/beliefs/store.py`).** The two async functions `save_checkpoint` / `get_checkpoint` use a raw `psycopg.AsyncConnection`. `save_checkpoint` issues an `INSERT … ON CONFLICT (quantity_id, index_key) DO UPDATE`, making every write idempotent. `index_key` and `sufficient_stats` are stored as `jsonb`; `log_watermark` is stored as a plain integer (cast from `EventSeq`).

**Type hierarchy (`placer/beliefs/types.py`).** All types are Pydantic `BaseModel` subclasses. `SufficientStats` is a discriminated union on the `kind` literal field (`"beta" | "normal" | "cure_model" | "histogram" | "scalar"`). `BeliefIndexKey` is a parallel discriminated union on the `schema` alias field. The `Belief` model exposes `_stats` as a `PrivateAttr`; it is populated by the resolver functions and consumed only by `sample()`, preserving encapsulation across the public interface.

**Resolver pipeline.** `_parse_stats` deserialises the raw `dict` from the DB into the correct `SufficientStats` subtype. It handles `"beta"`, `"normal"`, and `"scalar"` kinds; unknown kinds (including `"cure_model"` and `"histogram"`) fall back silently to `BetaStats(α=1, β=1)` — these two stats types are defined in the type hierarchy but not yet wired into the deserialisation or sampling paths (`_sample_from_stats` raises `NotImplementedError` for them). `_point_estimate` and `_variance` are pure `match`-dispatch helpers. `_interval` calls `scipy.stats.beta.ppf` for Beta posteriors and `scipy.stats.norm.ppf` for Normal ones. `_flat_prior` returns a fixed `(0.05, 0.95)` interval with `BetaStats(α=1, β=1)`, `n_eff=0`, and `staleness=∞`, serving as the universal fallback.

**Partial pooling formula.** `partially_pool(org_belief, inherited_belief, c=5.0)` computes `w_own = n_eff / (n_eff + c)` and mixes point estimates and interval bounds linearly. Note: the function parameter `c` (float) is shadowed by the loop variable `c` in the `provenance_mix` comprehension over `inherited_belief.provenance_mix` — a latent naming collision in the current source.

**Mixture variance.** `resolve_inherited_belief` approximates mixture variance as `Σ wᵢ · (σᵢ² + (μᵢ − μ̄)²)` (law of total variance), then computes a Normal credible interval over the mixture using `scipy.stats.norm.ppf`.

**Debug endpoints (`placer/api/debug.py`).** Both belief endpoints are mounted under the `/debug` FastAPI router (prefix `/debug`), meaning the actual HTTP paths are `GET /debug/beliefs/quantities` and `GET /debug/beliefs/checkpoints`. The quantities endpoint JOINs `quantity_registry` with `belief_checkpoints` to surface aggregate statistics per quantity.

## Availability — what is usable right now

**Persistence primitives — available.** `save_checkpoint` and `get_checkpoint` in `placer/beliefs/store.py` are fully implemented and exercisable against any `psycopg.AsyncConnection` backed by a Postgres instance with the `belief_checkpoints` table present.

**Resolver functions — available (with caveats).** `resolve_belief_from_checkpoint`, `resolve_inherited_belief`, and `partially_pool` are all implemented as pure Python functions with no outstanding stubs. However, `CureModelStats` and `HistogramStats` — while present in the `SufficientStats` union — have no deserialisation path in `_parse_stats` (unknown kinds fall back to `BetaStats(1,1)`) and no sampling implementation in `_sample_from_stats` (raises `NotImplementedError`). Any checkpoint stored with `kind = "cure_model"` or `kind = "histogram"` will silently resolve to a flat prior rather than the intended posterior.

**Conjugate fold helpers — available.** `beta_fold_success` and `beta_fold_failure` are complete and have no dependencies beyond `BetaStats`.

**Debug API endpoints — available under `/debug` prefix.** `GET /debug/beliefs/quantities` and `GET /debug/beliefs/checkpoints` are registered on the FastAPI debug router in `placer/api/debug.py`. Both are read-only. The checkpoints endpoint accepts an optional `quantity_id` filter and a `limit` (1–500, default 100).

**`BeliefStore` abstract contract — defined, not concretely implemented in this module.** The ABC is present in `placer/contracts.py` and consumed by `GenerateContract` and `ValueContract`, but no concrete `BeliefStore` subclass is provided by the beliefs module itself. Callers must supply their own implementation wrapping the persistence primitives.

**`CureModelStats` / `HistogramStats` — type-only.** Both types are importable and Pydantic-valid, but the round-trip from DB → `SufficientStats` → `Belief.sample()` is not complete for either family. No changelog entry describes planned completion; status is indeterminate.
