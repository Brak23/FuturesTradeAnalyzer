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

- `justfile` (or `Makefile`) at root with common tasks: `just ingest`, `just dq`, `just features`, `just analyze`, `just report`.
- `.github/workflows/ci.yml` for the CI pipeline.
- `scripts/` for one-off CLI entry points if `click` commands inside `src/` are not ergonomic.

---

## 2. Development approach

### 2.1 Operating principles

1. **Notebook-first, then promote.** Research lives in `notebooks/`. Once a pattern stabilizes (function shape, fixture format, output schema), it moves to `src/`. Avoid premature engineering of code that the analysis will discard.
2. **Test-first for feature code.** Every feature function has a synthetic-bar unit test AND a point-in-time guard test. This is non-negotiable because point-in-time leakage is silent and irreversible.
3. **Contracts via pydantic.** Trade, setup observation, feature output, rubric YAML all have pydantic models. Schema drift fails loudly at the boundary.
4. **Deterministic outputs.** Every analytics run records inputs hash + git SHA + seed + rubric version in the report header (per CLAUDE.md coding standards). Re-running the same command on the same inputs reproduces identical output.
5. **Kill criteria are real.** Each milestone in `RESEARCH_PLAN.md` M0-M7 has an explicit kill criterion. If one fires, the project stops and re-scopes. We do not soften the criterion to keep building.
6. **No premature abstraction.** First feature in each category is concrete and inline. Abstract base classes / strategy patterns appear only when the second and third features prove the pattern. Three concrete is better than premature.
7. **Reports are the deliverable.** A passing test suite is necessary but not sufficient. Every milestone produces a markdown or HTML report with the actual numbers. Bryce reviews for technical correctness; Allie reads the report-for-trader version. If a report is so dense Allie won't read it, it doesn't count as shipped (see `CLAUDE.md` "How to design outputs for Allie").

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
| End-to-end | Sample 10-trade dataset → full pipeline → expected report | One `pytest` slow-marked test |

Property-based tests via `hypothesis` for any feature that takes numeric inputs: assert no exception on the full domain, assert monotonicity where defined.

### 2.5 Documentation

- `README.md` — Bryce's developer onboarding. How to install, ingest, run analytics, render reports. Not consumed by Allie.
- `READING_THE_REPORT.md` (post-Phase-4) — short, plain-language guide for Allie on how to interpret the HTML output.
- `CLAUDE.md` — engineering spec (existing).
- `RESEARCH_PLAN.md` — methodology (existing).
- `DEVELOPMENT_PLAN.md` — this doc.
- Per-milestone memo in `data/reports/MXX_memo.md` written when the milestone completes.
- Docstrings on public functions only; no doc bloat.

---

## 3. Phases

Phases map 1:1 to `RESEARCH_PLAN.md` milestones with the engineering tasks called out. Time estimates assume half-time effort.

### Phase 0 — Bootstrap (Days 1-3)

**Goal**: Repo is ready to receive code. No analytics yet.

**Tasks**:
- Initialize `pyproject.toml` with the dependencies in section 1.2.
- Set up `uv` (or `poetry`) venv.
- Configure `ruff`, `mypy`, `pytest`, `pre-commit`.
- Add `.github/workflows/ci.yml`.
- Create `src/`, `tests/`, `notebooks/`, `data/raw/`, `data/enriched/`, `data/reports/`, `rubric/versions/` directories.
- Add `.gitignore` (parquet outputs, jupyter checkpoints, `.env`, `data/raw/`).
- Add `justfile` with task stubs.
- Write `src/config.py` with placeholder constants (ATR lookback = 14, RTH open = 09:30 ET, FDR alpha = 0.05, bootstrap N = 10000, etc.).
- README skeleton.

**Deliverable**: empty pipeline that runs `just test` and CI passes.

### Phase 1 — M0 Dataset audit (Week 1)

**Goal**: Know what data we have before writing more code.

**Tasks**:
- Allie exports trades CSV from Tradezella (last 12 months, all instruments, all fields). Place under `data/raw/tradezella/`.
- If bar data source per `PROJECT_KICKOFF.md` section 4.3 is NT, Allie also exports bars. Otherwise stub the bar fetch interface and defer until that source is wired in Phase 2.
- `notebooks/00_dataset_audit.ipynb`: count trades by month, instrument (MNQ vs NQ), setup, account context. Plot timeline. Identify rule-change dates.
- Memo: `data/reports/M0_dataset_audit.md`.

**Kill criterion**: <100 taken trades or <6 months of consistent rules. If hit, project pauses; only counterfactual / forward-capture (Phase 3) proceeds.

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
- `src/analytics/expectancy.py`: stationary bootstrap on mean R; bootstrap CI on mean forward-30min return. Use forward returns as primary if `stop_type = "mental"` dominates the dataset (decided at M0).
- `notebooks/02_expectancy_baseline.ipynb`: produce M2 memo.
- **MVP feature set** (~8 features chosen for sample-size discipline):
  - `feat_price_vwap_dist_atr` (distance from VWAP in ATR units)
  - `feat_price_prior_day_high_dist_atr`
  - `feat_price_prior_day_low_dist_atr`
  - `feat_temporal_minute_of_session`
  - `feat_temporal_open_auction_proximity`
  - `feat_volatility_atr_pct_60d`
  - `feat_volume_rvol_20d`
  - `feat_orderflow_cum_delta_session` (if tick data available; if not, deferred)
- `src/features/<file>.py` for each domain.
- `src/join.py`: for each trade, slice bar window, call each feature function with `as_of = entry_time`.
- `src/features/outcome.py`: compute MFE/MAE in R + forward returns at t+5/15/30/60.
- Unit tests + point-in-time guard tests for every feature.
- `just features` produces `data/enriched/trades_features.parquet`.

**Deliverable**: enriched parquet + M2 memo + feature implementation with passing tests.

**Kill criterion** (M2): 95% bootstrap CI on mean R negative. Project pauses; strategy review before grading work.

### Phase 5 — M4 Univariate analytics + M5 Bivariate (Weeks 4-6)

**Goal**: Answer Q2 (does any feature have a real edge?). Identify top features for the rubric.

**Tasks**:
- `src/analytics/univariate.py`: per feature, compute Spearman IC vs forward-30min return; bootstrap CI on IC; permutation p-value (10k shuffles, fixed seed); BH-FDR across all features.
- `src/analytics/distributions.py`: MFE/MAE histograms by feature quintile.
- `notebooks/03_univariate_analysis.ipynb`: ranked feature table + plots + memo.
- `src/analytics/interactions.py`: pairwise ANOVA on top 6 features.
- `notebooks/04_bivariate_interactions.ipynb`: interaction memo.

**Deliverable**: M4 memo with FDR-adjusted feature ranking; M5 interaction memo.

**Kill criterion** (M4): zero features pass FDR-adjusted IC threshold of 0.10. Revisit feature set or sample size; do not lower the bar.

### Phase 6 — M6 Rubric v1 + M7 Walk-forward (Weeks 7-9). **MVP COMPLETE AT END OF M7.**

**Goal**: Build rubric v1 from analytics output. Validate via rolling-origin walk-forward. Pass or fail the deployment gate.

**Tasks**:
- `src/rubric/builder.py`: applies the weight derivation table (CLAUDE.md). Selects <= 8 features. Sets grade thresholds (A+ top 20%, A next 30%, B bottom 50%).
- `rubric/versions/v1.yaml`: hand-reviewed output of the builder.
- `src/rubric/apply.py`: pure function `(trade_features, rubric) -> grade`.
- `src/rubric/versioning.py`: enforces promotion criteria; refuses to overwrite a promoted version.
- `notebooks/05_rubric_v1.ipynb`: in-sample monotonicity check.
- `src/analytics/walkforward.py`: rolling-origin 6mo train / 1mo test / 1mo step, 1-day embargo. Refits rubric per window.
- `src/analytics/deflated_sharpe.py`: Bailey & Lopez de Prado 2014 for the v_n vs v_{n+1} comparison.
- `notebooks/06_walkforward_validation.ipynb`: bucket-expectancy chart per window, stability metrics, memo.
- `src/reports/html.py`: HTML report template; emits M7 report with all reproducibility headers.

**Deliverable**: rubric v1 YAML + walk-forward report + M7 go/no-go memo.

**Decision gate**: section 5.7 of RESEARCH_PLAN.md (all 7 promotion criteria).

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
- `src/ingest/ninjatrader.py`
- `src/ingest/tradingview.py` (backward replay only; forward webhook is Phase 3 parallel work)
- `src/dq/` (all 10 checks)
- `src/features/price.py`, `temporal.py`, `volatility.py`, `volume.py`, `orderflow.py` (cumulative delta only, if tick data permits), `outcome.py`
- `src/join.py`
- `src/analytics/expectancy.py`, `univariate.py`, `distributions.py`, `interactions.py`, `walkforward.py`, `deflated_sharpe.py`
- `src/rubric/builder.py`, `apply.py`, `versioning.py`
- `src/reports/html.py` (single template covering Q1, univariate, walk-forward outputs)
- `src/config.py`

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
4. M7 walk-forward report exists and contains an explicit "GO" or "NO GO" verdict with the 7 promotion criteria checked.

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
