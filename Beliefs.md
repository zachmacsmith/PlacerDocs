---
feature: Beliefs
group: Placer
last_synced: '2026-06-12'
last_commit: c66cac868aea2ad3d45474337649f15a7c7db058
anchors:
  tables:
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
  - MixtureComponent
  - MixtureStats
  - NormalStats
  - OrgChannelIndexKey
  - OrgIndexKey
  - OrgSizeChannelIndexKey
  - ParametricStats
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
  - placer/api/debug.py::list_checkpoints
  - placer/api/debug.py::list_quantities
  - placer/beliefs/**
  - placer/beliefs/store.py
  - placer/beliefs/store.py::_collapse_to_parametric
  - placer/beliefs/store.py::_mixture_quantile
  - placer/beliefs/store.py::_staleness_seconds
  - placer/beliefs/store.py::coherence_priced_c
  - placer/beliefs/store.py::resolve_tree_backoff
  - placer/beliefs/types.py
  - placer/contracts.py
writes:
- belief_checkpoints
reads:
- placer/beliefs/store.py
---
## Capability — what it can do

The Beliefs feature is the platform's Bayesian state-of-knowledge layer. It provides a **uniform belief interface** — a `(point, interval, n_eff, staleness_seconds, provenance_mix)` structure — that every downstream consumer (Value, Generate, Resolve, Projections) reads without knowledge of how the internal distribution is represented.

**Quantity registry.** Twelve named quantities are pre-registered in `REGISTERED_QUANTITIES`: `p_want`, `p_donor_approve`, `p_fit`, `p_fit_archetype`, `p_resp`, `p_resp_pooled`, `resp_time`, `resp_time_pooled`, `cap_volume`, `cap_volume_pooled`, `lambda_ops`, and `donor_reliability`. Each is described by an estimand sentence, a `QuantityType`, and an `IndexSchema` that governs which index keys are valid for it (org, edge, org_channel, archetype, size_channel, ntee_size, donor, system). Pooled-parent entries (e.g. `p_fit_archetype`, `cap_volume_pooled`) are registry citizens with their own checkpoints, enabling read-time inheritance without round-trip writes.

**Checkpoint persistence.** `save_checkpoint` upserts a `BeliefCheckpoint` row into the `belief_checkpoints` table keyed on `(quantity_id, index_key)`. `get_checkpoint` retrieves it. Each checkpoint carries `sufficient_stats` (a JSON blob), `n_eff`, version tags (`selector_version`, `fold_version`), and a `log_watermark` that records which event-log position the checkpoint was computed from.

**Sufficient-statistics representations.** Five parametric families (`BetaStats`, `NormalStats`, `CureModelStats`, `HistogramStats`, `ScalarStats`) are collected under the `ParametricStats` union. A sixth type, `MixtureStats`, composes parametric checkpoints at read time: it holds a list of `MixtureComponent` (weight + `ParametricStats`) and is never written to `belief_checkpoints` (cheap-point rule). `SufficientStats` is the top-level discriminated union covering both parametric and mixture forms. The internal family is hidden behind the `Belief` interface; `sample()` dispatches to the correct distribution, and for mixtures it draws a component by weight before sampling it.

**Resolver skeleton (spec V2 §IV.3–IV.5).** Five pure functions implement read-time belief resolution:
- `resolve_belief_from_checkpoint` — converts a single checkpoint to a `Belief`, returning a flat Beta(1,1) prior when no checkpoint exists.
- `resolve_inherited_belief` — builds a `MixtureStats` from precision-weighted edge posteriors; the output is the mixture itself (never moment-matched). Disagreeing edges yield a multimodal prior that Thompson sampling can explore; credible intervals are exact mixture quantiles via bisection.
- `partially_pool` — blends an org's own-data belief with an inherited mixture, weighting by `n_own / (n_own + c)` (James–Stein-style partial pooling, default shrinkage constant `c = 5`). The pooled belief is itself a `MixtureStats`, flattening both sources, so `sample()` remains honest about both.
- `coherence_priced_c` — computes the pseudo-count worth of a parent prior as `obs_variance / (parent_variance + sibling_dispersion)`. High sibling dispersion drives `c` toward zero; a perfectly coherent parent yields a very large `c`. This prices incoherence, not tree height.
- `resolve_tree_backoff` — recursive shrinkage blending from root to leaf simultaneously (spec V2 §IV.5). At each level: `θ̂_l = (n_l·ȳ_l + c_l·θ̂_parent) / (n_l + c_l)`. Evidence is written once at the most specific level; ancestors are blended at read time. The output `Belief._stats` is re-expressed in the leaf family's parametric form (cheap-point rule) so `sample()` works.

**Conjugate fold helpers.** `beta_fold_success` and `beta_fold_failure` perform conjugate Beta updates in O(1).

**Factor coordinate systems.** `WantCoordinates`, `CapacityCoordinates`, `RespondCoordinates`, and `FitCoordinates` define per-factor inheritance geometry (spec §5.4), unified under the `FactorCoordinates` discriminated union.

**Debug API exposure.** Two read-only debug endpoints surface belief state: `GET /beliefs/quantities` (joined with `quantity_registry`, returning checkpoint counts and average `n_eff` per quantity) and `GET /beliefs/checkpoints` (filterable by `quantity_id`, paginated up to 500 rows).

**Abstract contract.** `BeliefStore` in `placer/contracts.py` is the interface through which Generate and Value operators access beliefs; its two abstract methods are `belief(query: BeliefQuery) -> Belief` and `checkpoint(quantity_id, index_key) -> BeliefCheckpoint | None`. `store.py` provides the concrete persistence primitives that any `BeliefStore` implementation builds on.

## Implementation — how it works

**Storage layer (`placer/beliefs/store.py`).** The two async functions `save_checkpoint` / `get_checkpoint` use a raw `psycopg.AsyncConnection`. `save_checkpoint` issues an `INSERT … ON CONFLICT (quantity_id, index_key) DO UPDATE`, making every write idempotent. `index_key` and `sufficient_stats` are stored as `jsonb`; `log_watermark` is stored as a plain integer (cast from `EventSeq`).

**Type hierarchy (`placer/beliefs/types.py`).** All types are Pydantic `BaseModel` subclasses. `SufficientStats` is a discriminated union on the `kind` literal field (`"beta" | "normal" | "cure_model" | "histogram" | "scalar"`). `BeliefIndexKey` is a parallel discriminated union on the `schema` alias field. The `Belief` model exposes `_stats` as a `PrivateAttr`; it is populated by the resolver functions and consumed only by `sample()`, preserving encapsulation across the public interface.

**Resolver pipeline.** `_parse_stats` deserialises the raw `dict` from the DB into the correct `SufficientStats` subtype. It handles `"beta"`, `"normal"`, and `"scalar"` kinds; unknown kinds (including `"cure_model"` and `"histogram"`) fall back silently to `BetaStats(α=1, β=1)` — these two stats types are defined in the type hierarchy but not yet wired into the deserialisation or sampling paths (`_sample_from_stats` raises `NotImplementedError` for them). `_point_estimate` and `_variance` are pure `match`-dispatch helpers. `_interval` calls `scipy.stats.beta.ppf` for Beta posteriors and `scipy.stats.norm.ppf` for Normal ones. `_flat_prior` returns a fixed `(0.05, 0.95)` interval with `BetaStats(α=1, β=1)`, `n_eff=0`, and `staleness=∞`, serving as the universal fallback.

**Partial pooling formula.** `partially_pool(org_belief, inherited_belief, c=5.0)` computes `w_own = n_eff / (n_eff + c)` and constructs a `MixtureStats` by scaling each belief's component weights by `w_own` and `w_inh = 1 − w_own` respectively, then flattening into a single mixture. `point` and `interval` are derived from the mixture via `_point_estimate` and `_interval` (exact quantile bisection for mixtures). If neither belief carries `_stats`, the function falls back to linear interpolation of point and interval bounds. The provenance_mix comprehension variable is `pc` (not `c`), avoiding the shadowing of the pooling constant that was present in the previous implementation.

**Mixture interval.** `resolve_inherited_belief` no longer moment-matches the mixture. Instead it stores a `MixtureStats` on `Belief._stats` and delegates `_interval` to `_mixture_quantile`, which inverts the weighted CDF by bisection (80 iterations) over the component support. This preserves multimodality — a bimodal mixture yields a wide interval that is not a Normal approximation. The law-of-total-variance helper `_variance` is still defined for `MixtureStats` and is used internally by `resolve_tree_backoff`.

**Internal resolver helpers.** `_staleness_seconds(computed_at)` computes elapsed seconds from a checkpoint timestamp, tolerating naive datetimes from older rows by treating them as UTC. `_collapse_to_parametric(mixture)` is an internal fallback that moment-matches a `MixtureStats` to a `NormalStats`; it is only applied to nested mixtures encountered inside checkpoints (which should not exist per the cheap-point rule), never to top-level resolver outputs. `_cdf` and `_mixture_quantile` support the bisection-based interval computation for `MixtureStats`.

**Debug endpoints (`placer/api/debug.py`).** Both belief endpoints are mounted under the `/debug` FastAPI router (prefix `/debug`), meaning the actual HTTP paths are `GET /debug/beliefs/quantities` and `GET /debug/beliefs/checkpoints`. The quantities endpoint JOINs `quantity_registry` with `belief_checkpoints` to surface aggregate statistics per quantity.

## Availability — what is usable right now

**Persistence primitives — available.** `save_checkpoint` and `get_checkpoint` in `placer/beliefs/store.py` are fully implemented and exercisable against any `psycopg.AsyncConnection` backed by a Postgres instance with the `belief_checkpoints` table present.

**Resolver functions — available.** `resolve_belief_from_checkpoint`, `resolve_inherited_belief`, `partially_pool`, `coherence_priced_c`, and `resolve_tree_backoff` are all implemented as pure Python functions with no outstanding stubs. `resolve_inherited_belief` and `partially_pool` produce true `MixtureStats` objects (never moment-matched); `resolve_tree_backoff` re-expresses the blended result in the leaf family's parametric form and works across any number of tree levels.

**Mixture sampling — available.** `MixtureStats` is handled in `_sample_from_stats`: the implementation draws a component by weight then samples it, preserving multimodality for Thompson sampling. `CureModelStats` and `HistogramStats` still raise `NotImplementedError` in `_sample_from_stats`; any checkpoint stored with `kind = "cure_model"` or `kind = "histogram"` will silently resolve to a flat prior via `_parse_stats` (unknown kinds fall back to `BetaStats(1,1)`) and will not sample correctly if somehow bypassed to `_sample_from_stats`.

**Conjugate fold helpers — available.** `beta_fold_success` and `beta_fold_failure` are complete and have no dependencies beyond `BetaStats`.

**Debug API endpoints — available under `/debug` prefix.** `GET /debug/beliefs/quantities` and `GET /debug/beliefs/checkpoints` are registered on the FastAPI debug router in `placer/api/debug.py`. Both are read-only. The checkpoints endpoint accepts an optional `quantity_id` filter and a `limit` (1–500, default 100).

**`BeliefStore` abstract contract — defined, not concretely implemented in this module.** The ABC is present in `placer/contracts.py` and consumed by `GenerateContract` and `ValueContract`, but no concrete `BeliefStore` subclass is provided by the beliefs module itself. Callers must supply their own implementation wrapping the persistence primitives.

**`CureModelStats` / `HistogramStats` — type-only.** Both types are importable and Pydantic-valid, but the round-trip from DB → `SufficientStats` → `Belief.sample()` is not complete for either family. No changelog entry describes planned completion; status is indeterminate.
