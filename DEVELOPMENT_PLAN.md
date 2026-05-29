# DEVELOPMENT_PLAN.md

**Role**: Full stack developer planning doc.
**Inputs**: `CLAUDE.md` (engineering spec), `RESEARCH_PLAN.md` (statistical methodology and gates), `PROJECT_KICKOFF.md` (pre-Phase-0 operating contract).
**Audience**: Bryce (sole developer) + any future implementer.
**End consumer of the system**: Allie (the trader; ~1 yr experience, MNQ/NQ, logs in Tradezella, has ADHD).
**Status**: Draft v0.2. Not approved. Pending signed `PROJECT_KICKOFF.md`.

---

## 0. TL;DR

This is a **data + analytics product** Bryce builds for Allie. There is no traditional UI. The actors are:

1. **Allie (end consumer)** — reads HTML reports designed for ADHD-friendly scanning; her tags/custom fields in Tradezella become the system's vocabulary.
2. **Bryce (developer + maintainer)** — runs the pipeline, reviews technical correctness, owns CI / repo / dependency management.
3. **Tradezella** — canonical source for trade fills; also possible sink for tag suggestions.
4. **Bar data source (TBD)** — supplies OHLCV for feature computation. NT, broker API, TradingView export, or paid feed; resolved in `PROJECT_KICKOFF.md` section 4.3 before Phase 1.
5. **TradingView (post-MVP)** — Pine Script setup detection and real-time webhook capture, only after Phase 6.

The product is the **pipeline + the rubric YAML + the reports Allie actually reads**. Everything else is plumbing.

**MVP definition** (Phase 6 below): A reproducible pipeline that ingests Allie's Tradezella trade history, computes a minimal feature set against bars from the chosen data source with point-in-time discipline, answers Q1 (positive expectancy?), runs FDR-corrected univariate analytics, produces rubric v1 YAML, validates v1 via rolling-origin walk-forward, and renders an ADHD-friendly HTML report Allie can read. The MVP is "successful" whether the walk-forward gate passes or fails. A "no go" outcome that prevents shipping a noise-rubric is a positive result.

**Estimated MVP effort**: 7-9 weeks at half-time pace, matching RESEARCH_PLAN.md milestones M0-M7.

**Critical path**: ingest → DQ → expectancy baseline (gate) → MVP features → univariate FDR (gate) → rubric v1 → walk-forward (gate).

**Counterfactual capture (Phase 3) runs in parallel** because every day without it accrues biased data.

**Architectural posture**: pluggable source protocols (no concrete vendor coupling outside `src/ingest/`), immutable parquet layers with schema-versioned manifests, architectural fitness functions enforced in CI, lightweight ADRs for every fork. Scale-appropriate: no microservices, no event bus, no service mesh. The pipeline is a monolith because one trader and one developer is a monolith problem.

---

## 1. Tech stack

### 1.1 Languages and runtimes

| Layer | Choice | Why |
|---|---|---|
| Analytics core | **Python 3.11+** | Specified in CLAUDE.md. Pandas/polars ecosystem, scikit-learn, SHAP, pydantic. |
| Real-time signal detection | **Pine Script v6** | Native to TradingView, only way to fire on confirmed bars in real-time. |
| Webhook ingestion | **Google Apps Script** (JavaScript V8 runtime) | Free, integrated with Sheets, no Zapier dependency. |
| Notebooks | **Jupyter / IPython** | Research phases (M0-M5) live in notebooks before code is promoted to `src/`. |
| Orchestration | **Bash + `just` (or Make)** | Simple task runner. No Airflow or Prefect at this scale. Reproducibility comes from deterministic Python scripts, not orchestration DAGs. |

### 1.2 Python dependencies

Grouped by purpose. Pinned versions live in `pyproject.toml`.

**Data + IO**
- `pandas >= 2.2` (primary dataframe lib)
- `polars >= 0.20` (used selectively where pandas is too slow, e.g., bar joins on millions of rows)
- `pyarrow` (parquet IO)
- `pydantic >= 2.6` (trade/feature/setup contracts)
- `python-dateutil`, `zoneinfo` (timezone discipline; `pytz` explicitly banned)
- `pyyaml` (rubric YAML)
- `gspread` + `google-auth` (read forward-capture sheet)

**Statistics + ML**
- `numpy`, `scipy` (bootstrap, permutation tests, Spearman)
- `arch` (stationary bootstrap; alternative `recombinator` package)
- `statsmodels` (ANOVA, regression diagnostics, VIF)
- `scikit-learn` (random forest, gradient boosting, permutation importance)
- `shap` (interaction discovery, M9 only)
- `mlxtend` (combinatorial purged CV reference impl, or custom)

**Visualization**
- `matplotlib` + `seaborn` (static plots in reports)
- `plotly` (interactive plots in notebooks)
- `jinja2` (HTML report templates)

**Dev tooling**
- `pytest`, `pytest-cov`, `hypothesis` (property-based tests for features)
- `ruff` (lint + format)
- `mypy` or `pyright` (type checking; choose one)
- `pre-commit` (hook ruff + mypy)
- `uv` (preferred) or `poetry` (dependency + venv management)
- `click` or `typer` (CLI commands: `analyze run`, `rubric apply`, `dq check`)

**Storage**
- **Parquet on local disk** for enriched datasets. No DB initially.
- **DuckDB** added in Phase 7+ if ad-hoc SQL queries against parquet become friction.
- **SQLite** only if a small registry of rubric versions / report metadata is needed; default to YAML/JSON files.

### 1.3 Infra

| Concern | Choice | Rationale |
|---|---|---|
| Source control | GitHub | Already in use. Bryce/Claude Code on the web integration. |
| CI | **GitHub Actions** | Run `ruff`, `mypy`, `pytest`, coverage on every PR. Block merge on red. |
| Secrets | `.env` (local) + GitHub Actions secrets | No production secrets management needed yet. |
| Compute | **Local laptop** for M0-M7. | Datasets are MB-scale, not GB-scale. No cloud needed. |
| Cloud (later) | Cloud Run + Cloud Storage **if** we ever need scheduled re-runs. Probably never required. |
| Data versioning | Lightweight: parquet + a sidecar `manifest.json` (inputs hash, git SHA, rubric version, row count, generation timestamp). DVC is overkill. |
| Observability | Structured logs to stdout (`structlog` or stdlib `logging`); HTML report headers carry the same metadata. |

### 1.4 Repo structure

Already specified in `CLAUDE.md`. No deviations. Confirming the additions this plan introduces:

- `justfile` (or `Makefile`) at root with common tasks: `just ingest`, `just dq`, `just features`, `just analyze`, `just report`, `just fitness` (architectural assertions).
- `.github/workflows/ci.yml` for the CI pipeline.
- `scripts/` for one-off CLI entry points if `click` commands inside `src/` are not ergonomic.
- `src/contracts/` for pluggable source protocols (see 1.5).
- `src/migrations/` for parquet schema migrations (see 1.6).
- `docs/adr/` for Architecture Decision Records (see 2.7).

### 1.5 Component contracts and source protocols

Two seams in the pipeline benefit from explicit protocols rather than concrete class couplings, because the vendor decision behind them is either undecided (bar data) or could change (broker integration). Protocol-first design isolates the rest of the pipeline from these vendor decisions.

**Trade source protocol** (`src/contracts/trade_source.py`):
- `TradeSource.fetch(start_dt, end_dt) -> Iterable[TradeRecord]`
- `TradeSource.metadata() -> SourceMetadata` (name, version, last-updated, row count)
- Concrete impls under `src/ingest/`: `TradezellaCSVSource` (canonical for MVP), `TradezellaAPISource` (post-MVP if API access is added).

**Bar source protocol** (`src/contracts/bar_source.py`):
- `BarSource.fetch(symbol, start_dt, end_dt, resolution) -> pd.DataFrame`
- `BarSource.has_tick_data() -> bool` (gates whether order flow features can run)
- `BarSource.metadata() -> SourceMetadata`
- Concrete impls (pick ONE for MVP per `PROJECT_KICKOFF.md` section 4.3): `NTBarSource`, `TradovateBarSource`, `TradingViewExportSource`, `DatabentoBarSource`.

**Rule** (fitness-function-enforced; see 2.6): no module outside `src/contracts/` or `src/ingest/` may import a concrete source class. All downstream code depends on the protocol. This is what lets us defer the bar-source decision until after Phase 0 without painting the architecture into a corner.

### 1.6 Data architecture

**Layered, immutable parquet storage** (nothing edits in place):

| Layer | Contents | Lifecycle |
|---|---|---|
| `data/raw/<source>/` | Exact source output (Tradezella CSV, bar CSV, etc.) | Append-only; never modified after landing |
| `data/enriched/trades_clean.parquet` | Post-ingest + DQ-passing trades, pydantic-validated | Regenerated when source or ingest code changes |
| `data/enriched/bars_clean.parquet` | Post-ingest + DQ-passing bars | Regenerated when source or ingest code changes |
| `data/enriched/trades_features.parquet` | `TradeRecord` + auto-discovered `feat_*` columns | Regenerated when features or join change |
| `data/enriched/setup_observations.parquet` | Counterfactual (taken + not-taken setups) | Append-only forward; backward-replay block regenerates |
| `data/reports/` | HTML / markdown / charts | Fully disposable; reproducible from enriched + git SHA |

**Schema versioning**: every parquet file ships with a sidecar `<filename>.manifest.json`:

```json
{
  "schema_version": 3,
  "schema_hash": "sha256:abc...",
  "source_inputs_hash": "sha256:def...",
  "code_git_sha": "1364587",
  "rubric_version": "v1",
  "row_count": 247,
  "generated_at": "2026-05-28T14:32:00Z",
  "generator": "src.join:enrich_trades"
}
```

The loader refuses to consume a parquet whose `schema_hash` doesn't match the pydantic model in code. Loud failure beats silent drift.

**Migrations**: schema-breaking changes increment `schema_version` and add a script under `src/migrations/NNNN_description.py` that reads the old schema and writes the new. Backward read-compatibility for at least one prior version.

**Idempotency guarantee**: re-running any stage on the same input parquet produces a bit-identical output (modulo the `generated_at` timestamp in the manifest). This is enforceable in CI as a fitness function.

### 1.7 Secrets and access control

MVP scope (local laptop, single developer):

- `.env` (gitignored) holds: Tradezella API token if API ingestion is chosen, Google service account JSON path if Sheet integration is enabled, broker API credentials if the bar source is a broker.
- `.env.example` checked in with placeholder values. No real secret ever in git.
- Pre-commit hook runs `detect-secrets` or `gitleaks` to catch accidents.
- Per-vendor token rotation policy in `docs/secrets.md`, even if it's just "rotate annually."

Data sensitivity: Allie's trade history contains money figures and timestamps tying her to specific trades. Treat `data/enriched/` and `data/reports/` as sensitive:

- Both gitignored.
- CI runs only on synthetic fixtures, never real data.
- Reports are not shared externally without explicit Allie approval.
- If outputs ever upload anywhere (Gist, Slack, email), they pass through a redaction step that strips identifying detail or masks P&L.

Cloud migration path (E18 enhancement): Google Secret Manager or 1Password CLI. Out of scope for MVP.

### 1.8 Observability

Minimum bar for an analytics pipeline where the most dangerous failures are silent (point-in-time leakage, FDR violation, schema drift):

- **Structured logging** via `structlog` to stdout. Every stage emits `stage_start`, `stage_end` with duration, and any `warning`/`error` events with structured context (sample size, file paths, hash values).
- **Reproducibility headers** on every report (already specified in coding standards; restated here as an architectural commitment).
- **Failure-mode catalog** in `docs/failure_modes.md`: what each kill criterion looks like in logs, what a DQ failure prints, what a point-in-time guard violation prints, what a fitness function failure prints. So Bryce can diagnose without re-reading the code.
- **Run summaries**: every `just <stage>` command ends with a one-line summary (rows in, rows out, duration, key metric). Allie never sees these; they're for Bryce.

Out of scope for MVP:
- Metric aggregation (Prometheus, StatsD).
- Distributed tracing (OpenTelemetry).
- Alerting (PagerDuty, email).

These come back into scope if the pipeline ever runs unattended on a schedule (E18 enhancement).

---

## 2. Development approach

### 2.1 Operating principles

1. **Notebook-first, then promote.** Research lives in `notebooks/`. Once a pattern stabilizes (function shape, fixture format, output schema), it moves to `src/`. Avoid premature engineering of code that the analysis will discard.
2. **Test-first for feature code.** Every feature function has a synthetic-bar unit test AND a point-in-time guard test. This is non-negotiable because point-in-time leakage is silent and irreversible.
3. **Contracts via pydantic.** Trade, setup observation, feature output, rubric YAML all have pydantic models. Schema drift fails loudly at the boundary.
4. **Deterministic outputs.** Every analytics run records inputs hash + git SHA + seed + rubric version in the report header (per CLAUDE.md coding standards). Re-running the same command on the same inputs reproduces identical output.
5. **Kill criteria are real.** Each milestone in `RESEARCH_PLAN.md` M0-M7 has an explicit kill criterion. If one fires, the project stops and re-scopes. We do not soften the criterion to keep building.
6. **No premature abstraction, two exceptions.** First feature in each category is concrete and inline; abstract base classes appear only when the second and third features prove the pattern. The two exceptions where protocols come FIRST: trade source and bar source (1.5), because vendor uncertainty is the reason to abstract early.
7. **Reports are the deliverable.** A passing test suite is necessary but not sufficient. Every milestone produces a markdown or HTML report with the actual numbers. Bryce reviews for technical correctness; Allie reads the report-for-trader version. If a report is so dense Allie won't read it, it doesn't count as shipped (see `CLAUDE.md` "How to design outputs for Allie").
8. **Reversibility classification.** Every architectural decision is classified as **one-way** (hard to reverse: data model schemas, the trade-source contract shape, the FDR alpha) or **two-way** (cheap to reverse: pandas vs polars inside a function, jinja2 template layout, plot library). One-way decisions get an ADR (2.7) before merge; two-way decisions don't need one.
9. **Fitness functions over conventions.** "We always do X" is a convention and erodes. "CI rejects merge if X is violated" is a fitness function and holds (2.6). Convert important conventions to fitness functions before they erode.
10. **Strangler over rewrite.** When a component needs replacement (e.g., bar source vendor swap, parquet → DuckDB), introduce the new behind the existing protocol, run both in parallel briefly to confirm parity, then cut over. Never a flag-day rewrite.

### 2.2 Branching and commits

- `main` is the canonical branch. CI must pass.
- Feature work on short-lived branches: `feat/ingest-nt`, `feat/orderflow-features`, `analytics/walkforward`.
- One PR per milestone where possible. Smaller is better.
- Commit messages reference the milestone (`M2: add bootstrap CI on mean R`).

### 2.3 Quality gates per PR

CI runs:
1. `ruff check` + `ruff format --check`.
2. `mypy src/`.
3. `pytest -q --cov=src --cov-fail-under=80`.
4. (Optional) `pytest -m point_in_time` as a separate marked job that runs ALL feature point-in-time guard tests.

Merge blocked on red.

### 2.4 Testing strategy

| Layer | Test style | Tools |
|---|---|---|
| Ingest | Fixture CSV → expected DataFrame | `pytest` + small CSV fixtures |
| DQ checks | Synthetic dirty data → expected failure rows | `pytest` |
| Features | Synthetic bar series → asserted feature value; point-in-time guard test | `pytest` + `hypothesis` for ranges |
| Analytics | Synthetic data with known IC → expected output within tolerance | `pytest` with fixed seed |
| Rubric apply | Hand-built rubric + hand-built trade → expected grade | `pytest` |
| **Source contracts** | Each concrete `TradeSource` / `BarSource` impl satisfies the protocol's full surface (returns correct types, handles empty range, raises on invalid input) | `pytest` parametrized over all impls |
| **Fitness** | Architectural assertions (see 2.6) | `pytest -m fitness`, separate CI job |
| **Characterization** | Lock current rubric output: applying `v1.yaml` to a fixed trade set produces a known-grade vector. Catches unintended weight changes during refactors. | `pytest` with golden file |
| End-to-end | Sample 10-trade dataset → full pipeline → expected report | One `pytest` slow-marked test |

Property-based tests via `hypothesis` for any feature that takes numeric inputs: assert no exception on the full domain, assert monotonicity where defined.

### 2.5 Documentation

- `README.md` — Bryce's developer onboarding. How to install, ingest, run analytics, render reports. Not consumed by Allie.
- `READING_THE_REPORT.md` (post-Phase-4) — short, plain-language guide for Allie on how to interpret the HTML output.
- `CLAUDE.md` — engineering spec (existing).
- `RESEARCH_PLAN.md` — methodology (existing).
- `DEVELOPMENT_PLAN.md` — this doc.
- `docs/adr/NNNN-slug.md` — Architecture Decision Records (see 2.7).
- `docs/secrets.md` — vendor token inventory and rotation policy.
- `docs/failure_modes.md` — observability and diagnostic catalog (see 1.8).
- Per-milestone memo in `data/reports/MXX_memo.md` written when the milestone completes.
- Docstrings on public functions only; no doc bloat.

### 2.6 Architectural fitness functions

Fitness functions are testable assertions about the system's quality attributes, enforced in CI. They turn the "we will be careful about X" conventions into "merge is blocked if X is violated" guarantees. Every fitness function is a regular pytest test marked `@pytest.mark.fitness`, runs in a dedicated CI job, and blocks merge on failure.

The minimum MVP set (write as you go, all in place by end of Phase 6):

| # | Fitness function | Why it matters | Failure looks like |
|---|---|---|---|
| F1 | **Point-in-time integrity**: every feature function in `src/features/` rejects any input row whose timestamp >= `as_of`. Enforced by introspecting each feature module. | Leakage is the #1 silent killer of backtest validity. | Test imports a feature, calls it with a synthetic future-timestamp row, expects an exception, fails if none raised. |
| F2 | **Reproducibility headers present**: every report generator writes the full reproducibility tuple (inputs hash, git SHA, rubric version, sample size, n_tests, FDR alpha, generated_at). | Without this, no result can be re-traced. | Test parses generated HTML, asserts all keys exist. |
| F3 | **No magic numbers outside config**: numeric literals in `src/features/`, `src/analytics/`, `src/rubric/` outside an allow list (0, 1, -1, common math constants) fail lint. | Hidden thresholds are technical debt that causes drift. | Custom ruff rule or grep step in CI. |
| F4 | **No concrete source imports outside ingest**: any module outside `src/contracts/` and `src/ingest/` importing a concrete source class fails. | Protects the protocol abstraction (1.5). | Grep test in CI. |
| F5 | **Schema integrity**: every parquet output has a corresponding manifest.json with matching schema_hash. Pipeline refuses to read mismatched parquets. | Schema drift silently corrupts downstream stages. | Loader raises on mismatch; integration test asserts. |
| F6 | **FDR control applied**: any analytics function that runs multiple tests includes `n_tests` and `fdr_alpha` in its output. | Multiple-comparisons inflation is the #1 silent killer of "discovered" edges. | Test introspects analytics module return shapes. |
| F7 | **Rubric size cap**: any rubric YAML loaded has <= 8 features. | Enforces the sample-size discipline in CLAUDE.md. | Loader validation raises; test asserts on a fixture with 9 features. |
| F8 | **Idempotency**: re-running any stage on the same input parquet produces a bit-identical output parquet (modulo manifest timestamps). | Detects accidental non-determinism (unset seeds, dict iteration order). | Slow-marked test runs each stage twice, diffs output. |
| F9 | **Coverage threshold**: `pytest-cov` enforces >= 80% line coverage on `src/`. | Floor on test discipline. | CI fail. |
| F10 | **Point-in-time guard tests exist per feature**: introspects `src/features/` and asserts every public feature function has a corresponding `test_<name>_rejects_future_data` test. | Catches feature additions that forget the guard test. | Test enumerates features, looks for test names, fails on missing. |
| F11 | **Truncation invariance for aggregating features (QH5/QH6)**: every aggregating feature (VWAP, cumulative delta, ATR percentile, anything with internal accumulation) satisfies `f(df_full, as_of=t) == f(df_truncated_at_t, as_of=t)`. | Catches subtle leaks where the AGGREGATION uses future data even when the called row's timestamp is in the past. F1 alone does NOT catch this. | Parametrized test feeds each aggregating feature both versions of the dataframe, asserts equality. |
| F12 | **Bucket-size floor (QH16)**: any expectancy-by-quintile report rejects buckets with N < 25 and falls back to tertiles with a "small sample" flag. | Prevents 10-observation buckets being reported as estimates. | Test feeds a 50-row fixture, asserts the report degrades to tertiles + flag. |

**Failures of any fitness function are architectural regressions, not normal bugs.** They take precedence over feature work. Adding a new fitness function is itself a small ADR (2.7).

### 2.7 Architecture Decision Records (ADRs)

Every fork that affects more than one module, locks in a vendor, or is expensive to reverse gets a one-page ADR in `docs/adr/NNNN-slug.md`. Cheap-to-reverse decisions (lib swap inside a single function, plot style) do not.

**Format** (kept deliberately short; an ADR is a paragraph each, not a doc):

```markdown
# ADR-NNNN: Slug

**Status**: Proposed | Accepted | Superseded by ADR-MMMM
**Date**: YYYY-MM-DD
**Decider**: Bryce

## Context
What's the situation forcing a decision.

## Options considered
1. Option A (with one-line pro/con)
2. Option B (with one-line pro/con)
3. Option C (with one-line pro/con)

## Decision
The chosen option, in one sentence.

## Consequences
Good: what this unlocks.
Bad: what this costs or constrains.
Neutral: anything else worth noting.
```

**Initial ADRs to write before Phase 1** (each one paragraph per section, takes ~15 min):

| # | Title | Why it's an ADR not a code comment |
|---|---|---|
| ADR-0001 | Tradezella as canonical trade source | Reverses previous "NT canonical" stance; affects ingest, DQ, downstream features |
| ADR-0002 | Bar data source selection | One-way decision; concrete choice locked per `PROJECT_KICKOFF.md` 4.3 |
| ADR-0003 | Parquet + sidecar manifest over a database | Storage architecture; affects every pipeline stage |
| ADR-0004 | Rolling-origin walk-forward over k-fold CV | Validation strategy; affects analytics module and acceptance gates |
| ADR-0005 | Pluggable source protocols (TradeSource, BarSource) | Cross-cutting; enforced by F4 fitness function |
| ADR-0006 | Forward-horizon returns as primary outcome metric | Conditional on stop-methodology answer; flag superseding ADR if R-multiple becomes primary |
| ADR-0007 | Notebook-first then promote (not src-first) | Sets the working model; explains why Phase 1-4 deliverables look like notebooks |
| ADR-0008 | Local-laptop monolith over cloud / microservices | Scope decision; rules out an entire class of complexity for the foreseeable horizon |
| ADR-0009 | Order flow classification method (Lee-Ready default, tick-rule fallback) | Quant-methodology decision (QH8); affects every downstream conclusion that uses cumulative delta |
| ADR-0010 | Net-of-costs as primary metric, gross as reporting only | Quant-methodology decision (QH1); affects every threshold and gate in the project |
| ADR-0011 | **Continuous-contract construction / bar-adjustment method (PR-REVIEW-P5)** | One-way decision currently undefined ("back-adjusted or ratio-adjusted", RESEARCH_PLAN.md §3.3). Back-adjustment distorts historical absolute price and silently corrupts ATR-normalized distance features and percentage forward-returns — the dominant feature family for MNQ/NQ. Decide **per-feature**: session-anchored features (VWAP, cum-delta, rvol, intraday levels) never cross a roll and want un-adjusted front-month bars; only multi-day lookbacks (ATR percentile vs 60 days, prior-day levels) need a continuous series. Write before any feature code in Phase 4. |

**ADRs are versioned in git, never deleted.** Superseded ones are marked superseded with a link to the replacement. This is the project's architectural history.

**ADR cadence**: write one when a decision is being made, not after. Re-opening a decision (e.g., switching bar source vendor) supersedes the prior ADR rather than editing it.

### 2.8 Quantitative methodology hazards (logical gaps and resolutions)

Sections 1.5-1.8 and 2.6-2.7 protect against engineering errors. The list below catalogs **quantitative-methodology errors that would invalidate results even with perfect engineering**. Each has a specific resolution baked into the relevant phase. The hazards below are not paranoia: every one of them is a documented failure mode in discretionary-trading analysis.

#### Outcome and label hygiene

| # | Hazard | The logical problem | Resolution and phase |
|---|---|---|---|
| QH1 | **Transaction cost realism** | Plan treats R-multiple and forward returns as gross. Real economics include commission (~$0.50/MNQ contract, ~$2.50/NQ at typical brokers), exchange and regulatory fees, and slippage. A "0.3R lift" gross may evaporate or invert net of costs. The asymmetry MNQ vs NQ also matters: MNQ has 1/10 the tick value but commission scales much less than 1/10. | Compute BOTH gross and net at outcome stage (`src/features/outcome.py`). All rubric thresholds and acceptance gates use NET. Per-instrument cost lookup table in `src/config.py`. Slippage modeled as: taken trades use realized fill price (no model needed); counterfactual setups use entry-bar close +/- half-spread estimate. Phase 4. |
| QH2 | **Forward-return reference price ambiguity** | "Forward 30min return" can use entry FILL price, entry BAR close, entry BAR open, or midpoint. Each gives different numbers and different IC. Currently unspecified. | Canonical: entry FILL price for taken trades; entry-bar close for counterfactual setups (no fill exists). Documented in `src/features/outcome.py` docstring. Test asserts both code paths. Phase 4. |
| QH3 | **Forward-return window overlap** | Trades taken 5 minutes apart have forward-30min windows that overlap by 25 minutes. The "independent samples" assumption underlying permutation tests and naive bootstrap is violated; significance inflates. | Tag trades whose forward windows overlap. Report ICs both ways (overlapping included / excluded). Stationary bootstrap block length must cover the longest overlap. Phase 5. |
| QH4 | **MFE/MAE granularity** | Bar-by-bar MFE/MAE understates true excursion: a 1-minute bar shows the high and low but not the path within. Two trades with identical 1-min bar MFE may have very different tick-level MFE. | Compute MFE/MAE from tick data when `BarSource.has_tick_data()` returns true; otherwise use 1-second or 5-second bars for the MFE/MAE window even if entry features use 1-minute. Document the granularity in the outcome module. Phase 4. |

#### Feature computation leaks (subtler than the §2.6 F1 point-in-time guard)

The F1 fitness function catches the obvious case (passing a future-timestamp row to a feature). It does NOT catch the subtle cases below, where the feature is structurally leaky even when called correctly.

| # | Hazard | The logical problem | Resolution and phase |
|---|---|---|---|
| QH5 | **Session-VWAP using future intraday data** | "VWAP of today's session" at 10am can naively use `df.vwap.iloc[-1]` and return the 4pm value. Even without F1 violation (timestamp is "today"), the AGGREGATION is over future data. | VWAP feature accepts `as_of` and computes cumulative VWAP over rows strictly before `as_of`. Test asserts `vwap(df_full, as_of=t)` equals `vwap(df_truncated_at_t, as_of=t)`. New fitness function F11 generalizes this: any aggregating feature must be invariant under truncation of post-`as_of` rows. Phase 4. |
| QH6 | **Trailing-percentile using today's incomplete data** | "ATR percentile vs trailing 60 days" — if "trailing" includes today's in-progress ATR (depends on today's full range, including post-entry bars), it's a leak. | Percentile computed against the 60 prior CLOSED sessions only. Today is never in the percentile window. Test asserts. Phase 4. |
| QH7 | **Indicator warmup** | EMA(200) needs 200 bars to stabilize. The first 200 bars of any feature series produce values that depend on the initialization choice (zero-fill vs forward-fill vs NA). Currently unspecified. | Drop the first `MAX_INDICATOR_LOOKBACK` bars (default 200, in `src/config.py`) per session OR mark them with a `warmup_incomplete` flag. Features in warmup period are excluded from IC computation. Phase 4. |
| QH8 | **Order flow methodology under-specified** | "Cumulative delta at entry" — computed how? Tick rule (uptick = buy)? Bid/ask classification (Lee-Ready 1991)? Trade-at-bid vs trade-at-ask? Each gives different results, sometimes substantially. CLAUDE.md hand-waves this. | Write **ADR-0009: Order flow classification method**. Default: Lee-Ready bid/ask classification if quote data available; tick-rule fallback when only trade data exists. Test against a synthetic series with known ground truth. Phase 0 for the ADR; Phase 4 if order flow features ship in MVP. |

#### Sample composition and confounds

| # | Hazard | The logical problem | Resolution and phase |
|---|---|---|---|
| QH9 | **MNQ + NQ pooling without test** | Plan asks Allie whether she treats them the same. The right answer is to TEST it (KS test on R distributions, stability of feature ICs across the two instruments). Pooling without testing masks instrument-specific regimes. | Phase 4 adds a pooling test: KS on R distributions and IC-rank-correlation across instruments. If KS rejects same-distribution OR Spearman rank-correlation of feature ICs across instruments < 0.7, run separate rubrics. Result documented in M4 memo. |
| QH10 | **Time-of-day as confound, not just feature** | Allie may concentrate trades in certain hours. Any feature whose value correlates with time-of-day will get spurious IC from the time-of-day return distribution. Including "minute of session" as a feature does NOT control for this; it just gives time-of-day its own slot. | Phase 5 univariate analysis runs each feature in TWO modes: raw IC and time-of-day-stratified IC (compute IC within each session bucket, then aggregate). If raw and stratified IC diverge sharply, the feature is at least partly a time-of-day proxy. Flagged in report. |
| QH11 | **Trade duration as confound** | Allie's setups span scalp (30 sec) to swing (multi-hour). A forward 30min return is overwhelming for a scalp (most of the move is the entry) and noise-dominated for a swing (the holding period is longer than the label). Pooling forces a one-size horizon that fits neither. | Phase 4 deliverable: compute forward returns at multiple horizons (5/15/30/60 minutes). Phase 5 also runs IC per trade-duration tertile. Phase 6 rubric promotion criteria checked per duration segment. If feature edges differ wildly by duration, separate rubrics or duration as a rubric input. |
| QH12 | **News / scheduled event contamination** | Forward 30min return through an FOMC release is dominated by the release, not the setup. The `feat_market_event_distance` feature exists in CLAUDE.md but the analysis plan doesn't say what to DO about contaminated forward returns. | Phase 4 outcome computation also produces a `forward_return_excl_event` variant that masks (sets to NA) when the forward window contains a major scheduled event. Phase 5 runs IC against both. If they diverge, events are dominating the signal. |
| QH13 | **Trade unit of analysis fuzziness** | Tradezella aggregates partials in configurable ways. The same 5-lot trade scaled out in 3 chunks can register as 1 trade or 3 depending on settings. This affects EVERY downstream metric (trade count, win rate, R, IC). | Phase 1 ingest documents the Tradezella aggregation policy in use (via Allie's account settings). Adds `trade_unit_policy` to every parquet manifest. Re-aggregation script in `scripts/regroup_partials.py` if policy needs to change later. |
| QH14 | **Survivorship / logging bias in Tradezella history** | The dataset is "trades Allie logged in Tradezella." Trades she forgot to log, days she skipped, trades cancelled before fill — systematically missing. | Phase 1 cross-checks Tradezella trade count vs broker statement trade count for one sample month. Documents the difference. If logging gap >5%, flag systemic bias risk in M0 memo and consider whether the analysis stratifies by data-quality period. |

#### Statistical inference

| # | Hazard | The logical problem | Resolution and phase |
|---|---|---|---|
| QH15 | **Bootstrap block length not validated against actual autocorrelation** | Stationary bootstrap with Politis-White auto-selection is specified, but the plan doesn't verify the chosen block is appropriate for Allie's R series. Trade outcomes are often autocorrelated within-day (tilt, regime persistence). | Phase 4 adds ACF/PACF plot of the R series to the M2 memo, plus the auto-selected vs manually-validated block length comparison. If autocorrelation persists past auto-selected block, override. |
| QH16 | **Minimum bucket size for quintile expectancy** | "E[R | top quintile]" with N=50 total trades gives 10 obs/bucket. That's a sample, not an estimate; bootstrap CI will be too wide to claim anything. Currently no floor. | Phase 5 enforces: refuse to report quintile expectancy lift when any bucket size < 25. Fall back to tertiles or terciles below that, with explicit "small sample" flag in the report. New fitness function F12 enforces the floor. |
| QH17 | **Negative control feature not operationalized** | RESEARCH_PLAN.md open question 10 says Allie names a feature she'd bet has zero edge as a methodology sanity check. DEVELOPMENT_PLAN.md doesn't include it anywhere. | Phase 4 adds the negative control feature (named by Allie during kickoff hypothesis pre-registration) to the feature set under the name `feat_negative_control_<slug>`. Phase 5 reports whether it spuriously clears FDR. **If it does, the methodology has a leak and analysis pauses for diagnosis.** |
| QH18 | **Stop-type segmentation not in build plan** | RESEARCH_PLAN.md Q8 is methodologically critical (R-multiple validity depends on stop type) but DEVELOPMENT_PLAN.md doesn't include the analysis. | Phase 5 adds KS test on R distributions partitioned by `stop_type`. If distributions differ significantly (p < 0.05), the primary outcome metric switches to forward returns for the rubric build, even if R was the default. |
| QH19 | **Walk-forward window count below statistical relevance** | 12 months minus 6mo train minus 1mo embargo = ~5 OOS windows. The promotion criterion "monotonic in 60% of windows" means 3-of-5; that has binomial p ~ 0.5 under chance, i.e., barely distinguishable from a coin flip. | Phase 6 reports the binomial p-value of the "60% of windows monotonic" requirement given the actual window count. If p > 0.10, the criterion is underpowered; the report says so in the headline and additionally requires combinatorial purged CV (Lopez de Prado 2018) as a supplement. |
| QH20 | **Probability of Backtest Overfitting (PBO) not computed** | Deflated Sharpe is specified. PBO (Bailey, Borwein, Lopez de Prado 2017) is more directly relevant when comparing rubric versions: PBO answers "what's the probability my best in-sample rubric is actually the best out-of-sample?" | Phase 6 adds PBO computation in `src/analytics/pbo.py` alongside deflated Sharpe for v_n vs v_n+1 comparisons. PBO > 0.5 means the chosen rubric is more likely overfit than skilled; promotion fails. |

**How to use this catalog**: every analytics report and every milestone memo cites which hazards it addresses, and which it explicitly does not. A claim that ignores a relevant hazard is not "preliminary," it is wrong.

---

## 3. Phases

Phases map 1:1 to `RESEARCH_PLAN.md` milestones with the engineering tasks called out. Time estimates assume half-time effort.

### Phase 0 — Bootstrap (Days 1-3)

**Goal**: Repo is ready to receive code; architectural scaffolding is in place. No analytics yet.

**Tasks**:
- Initialize `pyproject.toml` with the dependencies in section 1.2.
- Set up `uv` (or `poetry`) venv.
- Configure `ruff`, `mypy`, `pytest`, `pre-commit`, `detect-secrets`.
- Add `.github/workflows/ci.yml` with TWO jobs: regular `pytest` and `pytest -m fitness` (the fitness-function job runs even if regular tests fail, to surface architectural regressions early).
- Create `src/`, `src/contracts/`, `src/migrations/`, `tests/`, `notebooks/`, `data/raw/`, `data/enriched/`, `data/reports/`, `rubric/versions/`, `docs/adr/` directories.
- Add `.gitignore` (parquet outputs, jupyter checkpoints, `.env`, `data/raw/`, `data/enriched/`, `data/reports/`).
- Add `justfile` with task stubs including `just fitness`.
- Write `src/config.py` with placeholder constants (ATR lookback = 14, RTH open = 09:30 ET, FDR alpha = 0.05, bootstrap N = 10000, etc.).
- Write protocol stubs in `src/contracts/trade_source.py` and `src/contracts/bar_source.py` (1.5). No concrete impls yet.
- Write the first three fitness functions (F3 magic-numbers grep, F4 no-concrete-source-imports grep, F9 coverage threshold) so the harness exists before any feature code is written.
- Write the first eight ADRs (2.7) — each ~15 min, total ~2 hours.
- `docs/secrets.md` and `docs/failure_modes.md` skeletons.
- README skeleton.

**Deliverable**: empty pipeline that runs `just test`, `just fitness`, and CI passes. ADRs 0001-0008 merged. Protocols defined.

### Phase 1 — M0 Dataset audit (Week 1)

**Goal**: Know what data we have before writing more code.

**Tasks**:
- Allie exports trades CSV from Tradezella (last 12 months, all instruments, all fields). Place under `data/raw/tradezella/`.
- If bar data source per `PROJECT_KICKOFF.md` section 4.3 is NT, Allie also exports bars. Otherwise stub the bar fetch interface and defer until that source is wired in Phase 2.
- `notebooks/00_dataset_audit.ipynb`: count trades by month, instrument (MNQ vs NQ), setup, account context. Plot timeline. Identify rule-change dates.
- **QH13 trade-unit policy**: document Tradezella's partial-fill aggregation policy currently in use; record it in the memo.
- **QH14 logging-gap audit**: for one sample month, cross-check Tradezella trade count vs broker statement count. Report % gap.
- **MDES readout (PR-REVIEW-P0)**: compute the minimum detectable effect size (R and IC) at the actual stable-rules trade count; print it as the M0 memo headline. If MDES > ~0.4–0.5R, the memo declares the walk-forward-validated rubric a stretch deliverable and re-scopes the primary MVP to the forward-capture + counterfactual system (Phase 3) plus Q1/exploratory univariate. See RESEARCH_PLAN.md §7 M0 and `PLAN_REVIEW.md` §0.
- **Freeze pre-registration (PR-REVIEW-P2)**: commit hashed `analysis_preregistration.yaml` (feature list, canonical label horizon, test, thresholds) before any feature IC is computed. New fitness function asserts executed analysis matches it.
- **Initialize test ledger (PR-REVIEW-P4)**: create `test_ledger.json`; every statistical test across all milestones and future refits increments it; FDR/PBO read from it.
- Memo: `data/reports/M0_dataset_audit.md`, **led by the MDES readout**.

**Kill criterion**: <100 taken trades or <6 months of consistent rules. If hit, project pauses; only counterfactual / forward-capture (Phase 3) proceeds. Additional soft warning: logging gap >5% triggers a stratification plan. **Rubric-feasibility gate (separate from the start gate)**: if the MDES readout shows the §5.7 promotion thresholds are undetectable at the available N (near-certain at 200–400 trades), the rubric becomes a stretch goal and the forward-capture system is the MVP — this is a successful M0 outcome, not a failure.

### Phase 2 — M1 Ingest + DQ (Week 2)

**Goal**: Clean trade and bar dataframes that pass all 10 DQ checks.

**Tasks**:
- `src/ingest/tradezella.py`: parse Tradezella CSV export into pydantic-validated DataFrames. UTC normalization. Symbol-root extraction. Map Allie's existing tags/custom fields into the trade record.
- `src/ingest/bars.py`: bar loader for the chosen source (NT CSV, broker API, TradingView export, or paid feed adapter). Interface designed so the source can be swapped without touching downstream code.
- `src/dq/checks.py`: implement all 10 DQ checks per CLAUDE.md.
- `src/dq/report.py`: produce DQ report (HTML + markdown) with the ADHD-friendly layout from `CLAUDE.md` "How to design outputs for Allie."
- Tests: fixture CSVs covering happy path + each DQ failure mode for Tradezella + the chosen bar source.
- `just ingest` and `just dq` work end to end.

**Deliverable**: `data/enriched/trades_clean.parquet`, `data/enriched/bars_clean.parquet`, `data/reports/M1_dq_report.html`.

**Kill criterion**: DQ pass rate <95%. Fix data source before proceeding.

### Phase 3 — M3 Counterfactual capture (Weeks 2-4, parallel with Phase 2-4)

**Goal**: Forward-capture infrastructure live so the counterfactual dataset starts accumulating immediately. Backward replay over 12 months of bars.

**Tasks**:
- **Pine Script setup detector** for first 1-2 setups (depends on engineering open question 1 + RESEARCH_PLAN.md open question 5: setup definitions as falsifiable predicates).
  - `barstate.isconfirmed` enforced.
  - `lookahead=barmerge.lookahead_off` on all `request.security`.
  - Alert payload: full feature snapshot + setup id + timestamp.
- **Apps Script webhook receiver** (`webhook/apps_script.gs`):
  - Parses alert JSON.
  - Appends row to Google Sheet with columns matching the setup observation record schema.
  - Adds `taken` boolean (default false; Allie updates after the trading day) and `reason_not_taken` (free text initially).
- **Backward replay** (`src/ingest/tradingview.py` + a `scripts/replay_setups.py`):
  - Reapply setup logic over 12 months of bars in Python (mirror of Pine logic).
  - Cross-reference against actual Tradezella fills to mark `taken`.
- **Detector calibration**: 30 manually-labeled setups from Allie, compare to detector output, iterate until agreement >= 85%.

**Deliverable**: Sheet receiving live alerts; `data/enriched/setup_observations.parquet` from backward replay.

**Kill criterion**: Detector agreement <85% after 3 iterations. Re-scope setup definition with Allie.

### Phase 4 — M2 Expectancy baseline + MVP features (Weeks 3-4)

**Goal**: Answer Q1. Build the minimum feature set required for Phase 5.

**Tasks**:
- `src/analytics/expectancy.py`: stationary bootstrap on mean R AND mean forward-30min return, both **gross and net of costs (QH1)**. Use forward returns as primary if `stop_type = "mental"` dominates the dataset (decided at M0).
- **QH1 cost lookup**: per-instrument commission, exchange, and regulatory fees in `src/config.py` (MNQ vs NQ); slippage model for counterfactual setups documented in `src/features/outcome.py`.
- **QH15 autocorrelation diagnosis**: ACF/PACF plot of R series; report Politis-White auto-block alongside a manually-validated block. Both in M2 memo.
- `notebooks/02_expectancy_baseline.ipynb`: produce M2 memo. Headline numbers are NET, gross shown below.
- **MVP feature set** (~8 features chosen for sample-size discipline, plus negative control):
  - `feat_price_vwap_dist_atr` (distance from VWAP in ATR units; QH5 truncation-invariant)
  - `feat_price_prior_day_high_dist_atr`
  - `feat_price_prior_day_low_dist_atr`
  - `feat_temporal_minute_of_session`
  - `feat_temporal_open_auction_proximity`
  - `feat_volatility_atr_pct_60d` (QH6 prior-closed-sessions only)
  - `feat_volume_rvol_20d`
  - `feat_orderflow_cum_delta_session` (if tick data available, classified per ADR-0009 / QH8; if not, deferred)
  - `feat_negative_control_<slug>` (**QH17**, named by Allie at kickoff; must NOT clear FDR at Phase 5 or methodology is broken)
- `src/features/<file>.py` for each domain. **QH7 warmup**: drop first `MAX_INDICATOR_LOOKBACK` bars per session or flag `warmup_incomplete=True`; warmup rows excluded from analytics.
- `src/join.py`: for each trade, slice bar window, call each feature function with `as_of = entry_time`.
- `src/features/outcome.py`: compute
  - MFE/MAE in R (**QH4**: tick-data when `BarSource.has_tick_data()`, else 5-second bars for the excursion window even if entry features are 1-minute).
  - Forward returns at t+5/15/30/60 minutes (**QH11** multiple horizons).
  - Both raw and `forward_return_excl_event` masked variants (**QH12**: NA when forward window contains FOMC/CPI/NFP/EIA).
  - Reference price: entry FILL for taken trades, entry-bar close for counterfactual (**QH2**; documented in module docstring + tested both paths).
- **QH9 pooling test**: KS test on R distributions MNQ vs NQ; Spearman rank-correlation of feature ICs across the two. Verdict (pool / separate) written into M4 memo.
- Unit tests + point-in-time guard tests for every feature, plus **truncation-invariance tests for every aggregating feature (new fitness function F11)**.
- `just features` produces `data/enriched/trades_features.parquet`.

**Deliverable**: enriched parquet + M2 memo (with gross/net headline + ACF diagnosis) + feature implementation with passing tests + QH9 pooling verdict.

**Kill criterion** (M2): 95% bootstrap CI on **net** mean R negative. Project pauses; strategy review before grading work.

**Additional methodology kill**: if `feat_negative_control_<slug>` clears the FDR-adjusted IC threshold at Phase 5 (QH17), the pipeline has a leak; stop and diagnose before trusting any other result.

### Phase 5 — M4 Univariate analytics + M5 Bivariate (Weeks 4-6)

**Goal**: Answer Q2 (does any feature have a real edge?). Identify top features for the rubric.

**Tasks**:
- `src/analytics/univariate.py`: per feature, compute Spearman IC vs forward-30min return; bootstrap CI on IC; permutation p-value (10k shuffles, fixed seed); BH-FDR across all features.
- **QH3 overlap handling**: tag trades whose forward windows overlap with adjacent trades; report IC two ways (with and without overlapping trades). Stationary bootstrap block length covers the longest overlap.
- **QH10 time-of-day stratification**: every feature reported in TWO modes — raw IC and time-of-day-stratified IC. Sharp divergence flagged as "feature is partly a time-of-day proxy."
- **QH11 duration-segmented IC**: every feature reported per trade-duration tertile (short / medium / long). Big differences trigger Phase 6 discussion of duration-conditional rubrics.
- **QH16 bucket-size floor**: refuse to report quintile expectancy lift when any bucket size < 25; fall back to tertiles/terciles with explicit "small sample" flag. Enforced by new fitness function F12.
- **QH18 stop-type segmentation**: KS test on R distributions partitioned by `stop_type`. If p < 0.05, switch primary outcome to forward returns even if R was the default. Phase 6 inherits the switched outcome.
- **QH17 negative-control reporting**: the M4 memo prominently states whether `feat_negative_control_<slug>` cleared FDR. If yes, **methodology stop**: full diagnosis before any other conclusion.
- `src/analytics/distributions.py`: MFE/MAE histograms by feature quintile.
- `notebooks/03_univariate_analysis.ipynb`: ranked feature table + plots + memo.
- `src/analytics/interactions.py`: pairwise ANOVA on top 6 features.
- `notebooks/04_bivariate_interactions.ipynb`: interaction memo.

**Deliverable**: M4 memo with FDR-adjusted feature ranking (raw + time-stratified + duration-segmented), negative-control verdict, stop-type segmentation verdict; M5 interaction memo.

**Kill criterion** (M4): zero features pass FDR-adjusted IC threshold of 0.10 in BOTH raw and time-stratified modes. Revisit feature set or sample size; do not lower the bar.

**Methodology kill (overrides M4 kill criterion)**: negative-control feature clears FDR. Pipeline has a leak; stop and diagnose before trusting any other result.

### Phase 6 — M6 Rubric v1 + M7 Walk-forward (Weeks 7-9). **MVP COMPLETE AT END OF M7.**

**Goal**: Build rubric v1 from analytics output. Validate via rolling-origin walk-forward. Pass or fail the deployment gate.

**Tasks**:
- `src/rubric/builder.py`: applies the weight derivation table (CLAUDE.md). Selects <= 8 features. Sets grade thresholds (A+ top 20%, A next 30%, B bottom 50%). Inputs are the **net** outcome metric chosen at Phase 4 (forward returns if QH18 stop-type switch fired or stop_type=mental dominates; otherwise R-multiple).
- `rubric/versions/v1.yaml`: hand-reviewed output of the builder.
- `src/rubric/apply.py`: pure function `(trade_features, rubric) -> grade`.
- `src/rubric/versioning.py`: enforces promotion criteria; refuses to overwrite a promoted version. Fitness function F7 (rubric size cap <= 8) blocks load on violation.
- `notebooks/05_rubric_v1.ipynb`: in-sample monotonicity check. **QH11**: monotonicity also checked per trade-duration segment, not just pooled.
- `src/analytics/walkforward.py`: rolling-origin 6mo train / 1mo test / 1mo step, 1-day embargo. Refits rubric per window.
- **QH19 walk-forward power check**: report the binomial p-value of the "60% of windows monotonic" promotion criterion given the actual window count. If p > 0.10, the criterion is underpowered; M7 report headline says so AND combinatorial purged CV runs as a supplemental validator (`src/analytics/cpcv.py`).
- `src/analytics/deflated_sharpe.py`: Bailey & Lopez de Prado 2014 for v_n vs v_{n+1} comparison.
- `src/analytics/pbo.py`: **QH20** Probability of Backtest Overfitting (Bailey, Borwein, Lopez de Prado 2017). PBO > 0.5 means the chosen rubric is more likely overfit than skilled; promotion fails.
- `notebooks/06_walkforward_validation.ipynb`: bucket-expectancy chart per window, stability metrics, memo. Headline declares whether the dataset is statistically powered for the promotion criteria, BEFORE reporting the verdict.
- `src/reports/html.py`: HTML report template; emits M7 report with all reproducibility headers AND the QH19 power statement up top.

**Deliverable**: rubric v1 YAML + walk-forward report + M7 go/no-go memo.

**Decision gate**: section 5.7 of RESEARCH_PLAN.md (all 7 promotion criteria) **PLUS** the QH20 PBO check (PBO < 0.5) and the QH19 power check (criterion is statistically powered or supplemental CPCV concurs).

**GO**: Phase 7 begins; engineering continues.
**NO-GO**: rubric v1 is shelved. Project enters waiting state for more data, or scope is reduced to forward capture only.

**MVP DEFINITION FOR THIS PROJECT = END OF PHASE 6.** Anything beyond is enhancement.

### Phase 7 — Live forward-test (Months 4-6, conditional on M7=GO)

**Goal**: Validate the rubric in real-time, not just on retrospective data. Answer Q4.

**Tasks**:
- Add live grading to the Pine setup detector (compute rubric score in Pine, include in alert).
- Sheet captures pre-trade grade + Allie's actual decision + 1-5 conviction (if Q4 committed to in open question 7).
- Weekly join: alert grade + actual Tradezella trade outcome.
- Monthly forward-test report (`scripts/forward_test_report.py`): expectancy by grade vs walk-forward expectations; CI overlap check. Rendered in ADHD-friendly layout for Allie.
- Tradezella tag export (`src/reports/tradezella_export.py`): emit suggested tags per trade for Allie to apply (or auto-apply via Tradezella API if available).

**Deliverable**: monthly report; tag suggestions; conviction log.

**Decision gate**: after 3 months, OOS expectancy by grade matches walk-forward expectations within CI. If not, retrain.

### Phase 8 — Refinement loop (ongoing, quarterly)

**Goal**: Add features, expand setups, re-fit rubric quarterly per promotion criteria.

**Tasks**:
- Add remaining feature categories (full order flow set, HTF context, market context, trader state) — drives `rubric/versions/v2.yaml`.
- Pine Script for additional setups.
- `src/analytics/importance.py`: random forest + permutation importance + SHAP for interaction discovery. Informs rubric; does not become rubric.
- Re-run walk-forward on accumulated data per promotion criteria.

---

## 4. MVP definition (explicit scope)

### 4.1 In scope for MVP

**Code modules**:
- `src/contracts/trade_source.py`, `bar_source.py` (protocols)
- `src/ingest/tradezella_csv.py` (concrete `TradezellaCSVSource`)
- `src/ingest/<chosen_bar_source>.py` (concrete `BarSource` for the chosen vendor per ADR-0002)
- `src/ingest/tradingview.py` (backward setup replay; forward webhook is Phase 3 parallel work)
- `src/dq/` (all 10 checks)
- `src/features/price.py`, `temporal.py`, `volatility.py`, `volume.py`, `orderflow.py` (cumulative delta only, if tick data permits), `outcome.py`
- `src/join.py`
- `src/analytics/expectancy.py`, `univariate.py`, `distributions.py`, `interactions.py`, `walkforward.py`, `deflated_sharpe.py`, `pbo.py`, `cpcv.py`
- `src/rubric/builder.py`, `apply.py`, `versioning.py`
- `src/reports/html.py` (jinja2 templates covering Q1, univariate, walk-forward outputs; ADHD-friendly layout per CLAUDE.md)
- `src/migrations/` (empty at MVP; structure in place for first schema bump)
- `src/config.py`

**Architectural artifacts** (in addition to code):
- ADRs 0001-0010 written.
- Fitness functions F1-F12 implemented and CI-enforced.
- `docs/adr/`, `docs/secrets.md`, `docs/failure_modes.md` populated.
- Schema manifests on every enriched parquet.

**Quant-methodology artifacts** (per §2.8):
- Net-of-costs computation (QH1) + per-instrument cost table.
- Forward-return reference price documented and tested (QH2).
- Forward-window overlap handling (QH3) and ACF-validated bootstrap block (QH15).
- Tick-data MFE/MAE when available (QH4).
- Truncation-invariant aggregating features (QH5/QH6/F11).
- Order flow classification per ADR-0009 (QH8).
- MNQ/NQ pooling test verdict (QH9).
- Time-of-day stratified IC reporting (QH10).
- Multi-horizon forward returns + duration-segmented IC (QH11).
- Event-masked forward-return variant (QH12).
- Trade-unit policy documented in manifest (QH13).
- Logging-gap audit at M0 (QH14).
- Bucket-size floor with tertile fallback (QH16/F12).
- Negative-control feature in feature set (QH17).
- Stop-type segmentation analysis (QH18).
- Walk-forward power check + CPCV supplement (QH19).
- PBO computed for rubric promotion (QH20).

**Pine Script**:
- One setup detector firing alerts with feature payload (depends on which setup is in scope).

**Webhook**:
- Apps Script writing to Sheet with `taken` and `reason_not_taken` fields.

**Reports**:
- M0 audit memo
- M1 DQ report
- M2 expectancy memo
- M3 detector calibration memo
- M4 univariate FDR report
- M5 bivariate memo
- M6 rubric v1 YAML
- M7 walk-forward go/no-go report

**Tests**:
- Ingest, DQ, every feature (with point-in-time guard), rubric apply, end-to-end smoke test.

### 4.2 Out of scope for MVP (explicitly deferred)

- Multi-setup support beyond the first 1-2 setups.
- Multi-instrument per-instrument rubrics (single pooled rubric only; instrument as feature).
- Order flow microstructure features beyond cumulative delta (delta divergence, large lot ratio, bid/ask spread).
- HTF context features (60m/240m/daily trend stack).
- Market context features (SPY trend, VIX bucket, scheduled event distance).
- Trader state features.
- SHAP / ML importance analysis (M9 territory).
- Tradezella tag automation.
- Conviction logging integration (depends on Allie's commitment in open question 7).
- Real-time live grading in production trading.
- Any UI beyond HTML reports.
- Notifications / alerts to phone.
- DuckDB layer.
- Cloud deployment.

### 4.3 MVP success criteria

The MVP is successful when **all** hold:
1. `just ingest && just dq && just features && just analyze && just report` runs end to end on Allie's actual Tradezella trade history.
2. All milestone memos (M0-M7) produced with timestamps, sample sizes, # of tests run, FDR alpha, git SHA in every header.
3. Test suite passes; coverage >= 80%; CI green.
4. **`just fitness` passes**: all 10 architectural fitness functions (2.6) green.
5. **Source protocols are honored**: F4 fitness function confirms no concrete-source imports leak outside `src/ingest/`. Bar source can be swapped by writing one new file under `src/ingest/` and updating one config line.
6. **ADRs 0001-0008 merged**; any decisions made during the build that diverged from them are captured as superseding ADRs.
7. M7 walk-forward report exists and contains an explicit "GO" or "NO GO" verdict with the 7 promotion criteria checked.

A "NO GO" verdict counts as success. The MVP delivered the truth.

---

## 5. Enhancements (post-MVP backlog)

Ranked by **value × ease**, not just one or the other.

### 5.1 High value, moderate effort

| # | Enhancement | Drives |
|---|---|---|
| E1 | **Forward-capture sheet fully wired** with codified `reason_not_taken` categories | Counterfactual dataset enrichment; selection-bias correction strength |
| E2 | **Full order flow feature set** (delta divergence, large lot ratio, bid/ask spread regime) | RESEARCH_PLAN.md section 1.1 high-priority features; may unlock rubric v2 |
| E3 | **Live grading in Pine Script** matching the YAML rubric, included in alert payload | Real-time A+/A/B determination at signal time; required for Q4 forward-test |
| E4 | **Quarterly auto-refit** harness with promotion-criteria check | Removes Bryce's manual refit loop; makes refinement cheap |
| E5 | **HTML dashboard** (one-screen visual summary of latest rubric, recent OOS, sample size, detectable effect size; ADHD-friendly per CLAUDE.md) | Allie's daily glance; primary consumption surface |

### 5.2 Medium value

| # | Enhancement | Drives |
|---|---|---|
| E6 | **DuckDB query layer** over enriched parquet | Ad-hoc SQL exploration without notebook overhead |
| E7 | **Tradezella API integration** (if API exists) to auto-apply tags | Removes manual tagging; tighter outcome feedback loop |
| E8 | **HTF context features** (60m/240m/daily trend stack) | Per RESEARCH_PLAN.md feature table |
| E9 | **Market context features** (VIX bucket, scheduled event distance, SPY trend) | Per RESEARCH_PLAN.md |
| E10 | **Trader state features** (P&L state, consecutive W/L, tilt detection) | Behavioral edge factors |
| E11 | **Conviction logging UI** (simple Sheet form or Apps Script web app) | Q4 unlocked |
| E12 | **Regime-conditional rubrics** (if Q5 confirms regime effects) | Per RESEARCH_PLAN.md Q5 |
| E13 | **Per-setup rubrics** (different weights per setup if section 3.1 per-setup minimum is met) | Higher fidelity grading |
| E14 | **SHAP interaction analysis** (M9) | Find feature combinations the additive rubric misses |

### 5.3 Lower value or speculative

| # | Enhancement | Drives |
|---|---|---|
| E15 | Mobile journal entry app | Capture context the system can't observe (mood, sleep). Risk: subjective fields polluting dataset. |
| E16 | Slack / Discord notification on A+ setups firing | Convenience. Risk: behavioral coupling reduces ability to forward-test. |
| E17 | Web UI (Streamlit or FastAPI) replacing notebooks | Convenience only. Notebooks already work for the audience size of 1. |
| E18 | Cloud-scheduled re-runs (Cloud Run + Cloud Scheduler) | Only useful if reports need to land in Allie's inbox without Bryce running the pipeline manually. Defer until weekly cadence is real. |
| E19 | Multi-account / multi-trader support | YAGNI for a single-trader system. |
| E20 | Live broker integration / automated execution | Anti-goal in CLAUDE.md. Do not build. |

---

## 6. Risk register (engineering)

This complements RESEARCH_PLAN.md section 6 (statistical risks). Engineering-specific risks below.

| # | Risk | Probability | Impact | Mitigation |
|---|---|---|---|---|
| ER1 | NinjaTrader CSV schema drift between versions | Medium | Medium | Pydantic ingest layer fails loudly; pin NT version in README; add new test fixture per schema change |
| ER2 | Pine Script repaint despite `barstate.isconfirmed` (e.g., bug in custom indicator) | Medium | High | R11 in RESEARCH_PLAN.md; cross-validate Pine output against Python recomputation on overlap window |
| ER3 | Google Sheet quota / Apps Script trigger limits hit | Low | Medium | Apps Script has 90-min/day quota on free tier; monitor; fall back to direct webhook → small Flask app on Cloud Run if hit |
| ER4 | Tick data unavailable for backfilling order flow features | High | Medium | Depends on bar/tick data source per `PROJECT_KICKOFF.md` section 4.4. If Allie has no historical tick data, accept that order flow features are forward-only |
| ER5 | Notebook outputs check in to git | Medium | Low | `.gitignore` patterns; pre-commit hook to strip outputs (`nbstripout`) |
| ER6 | Magic numbers leak back into feature code | Medium | Medium | Lint rule (custom ruff or grep in CI) flagging numeric literals outside `src/config.py` |
| ER7 | Test data fixtures diverge from real NT format | Medium | Medium | Re-generate fixtures from real exports quarterly; document the generation script in `tests/fixtures/README.md` |
| ER8 | Walk-forward implementation has off-by-one window leakage | Medium | High | Synthetic test with planted signal at known window boundary; assert OOS detects it correctly |
| ER9 | Reproducibility metadata (git SHA, hash) missing from a report | Low | Medium | Centralize report-header construction in `src/reports/html.py`; tests assert header presence |
| ER10 | Dependency on a since-deprecated library mid-project (e.g., `pytz`-only API) | Low | Low | Pin versions; `pip-audit` in CI; quarterly dependency review |
| ER11 | Concrete source class leaks past the protocol boundary (e.g., feature code starts importing `TradezellaCSVSource` directly), making future vendor swap require deep rewrite | Medium | High | F4 fitness function blocks merge; ADR-0005 documents the rule |
| ER12 | Parquet schema drift between code and existing data (loader silently coerces or feature columns disappear) | Medium | High | F5 fitness function (manifest schema_hash match) blocks load; migration script required for any schema change |
| ER13 | Fitness functions added at end of project rather than as you go (so they enforce nothing during the build) | Medium | Medium | Phase 0 deliverable requires F3/F4/F9 in place before any feature code is written |
| ER14 | ADRs become checkbox theater rather than real decisions (filled in retroactively) | Medium | Low | Write ADR-0001-0010 in Phase 0 BEFORE the decisions are baked into code; any new ADR is written at decision time, not after |
| ER15 | **Gross vs net confusion** (QH1): rubric thresholds set in gross, applied in net (or vice versa); apparent edge evaporates in production | Medium | High | All gates use NET; M2 memo headline says "NET" explicitly; CI check on report templates for the word "gross" without "net" alongside |
| ER16 | **Hidden look-ahead in aggregating features** (QH5/QH6): F1 passes but VWAP/percentile uses post-`as_of` data via internal accumulation | High at first impl, Low after F11 | High | F11 fitness function (truncation invariance) catches at merge time |
| ER17 | **Negative-control feature clears FDR** (QH17): methodology has a leak the codified checks didn't catch | Medium | High | Negative-control feature is mandatory in MVP; clearing it triggers methodology stop, not a "well, maybe it's real" path |
| ER18 | **Walk-forward criteria pass at noise level** (QH19): 3-of-5 windows monotonic looks like a pass but is barely distinguishable from a coin flip | High at current data size | High | QH19 binomial power check in M7 report; CPCV supplement runs whenever power is borderline |
| ER19 | **PBO not computed → rubric promoted despite high overfit probability** (QH20) | Medium | High | PBO is a hard gate in §3 Phase 6 decision criteria, not a "report only" metric |
| ER20 | **Forward-window overlap inflates significance** (QH3): closely-spaced trades treated as independent samples; permutation tests over-reject | Medium | High | Overlap tagging in Phase 5; report dual ICs; bootstrap block length covers max overlap |

---

## 7. Open questions (engineering, beyond what CLAUDE.md asks)

Already escalated in CLAUDE.md and RESEARCH_PLAN.md. Restating the ones that block development:

1. **Tradezella export schema**: need a sample CSV from Allie's account before writing the ingest parser. Field coverage (esp. planned stop, commission) varies by broker integration.
2. **Bar data source decision** (`PROJECT_KICKOFF.md` section 4.3): NT / broker API / TradingView export / paid feed. Blocks Phase 1.
3. **Tick data availability**: does Allie have tick-level history (from NT or otherwise) for order flow features, or only minute bars?
3. **Google account for Sheet + Apps Script**: which account, who owns the sheet, what's the share permission model.
4. **TradingView plan**: Free / Pro / Pro+ / Premium affects bar count limit, alerts/month limit, webhook availability. Premium required for webhook alerts.
5. **NinjaTrader version** (NT8 confirmed in spec, but need build number; affects export schema).
6. **Acceptance of half-time pace**: 7-9 weeks to MVP assumes that. If full-time, halve; if quarter-time, double.
7. **CI minutes budget**: GitHub Actions free tier is 2000 min/month for private repos. Should be fine but worth confirming repo visibility.

---

## 8. Acceptance for this plan

Plan is approved when:
1. Allie + Bryce confirm the MVP scope in section 4 matches expectations (Allie on what the system needs to tell her; Bryce on what he can sustain to build).
2. Engineering open questions in section 7 are answered (or accepted as risk).
3. Phase ordering is agreed, especially that Phase 3 runs parallel to Phase 2 and 4.
4. Half-time pace (7-9 weeks) is accepted, or the timeline is renegotiated.
5. The "MVP success includes a NO GO walk-forward verdict" framing is accepted.

---

*End of plan. Awaiting signed `PROJECT_KICKOFF.md` from Allie + Bryce.*
