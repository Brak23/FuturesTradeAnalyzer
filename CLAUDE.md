# Futures Setup Grading & Edge Discovery System

## Project purpose

Build a repeatable, data-driven analytics system that takes Allie's working-but-mushy futures trading and empirically defines what separates A+ setups from A setups from B setups. Primary framing is **deep post-hoc analysis of Allie's Tradezella-logged trades**; a real-time grading rubric is a possible later output but not the MVP centerpiece. The system must:

1. Ingest Allie's trade history from Tradezella (canonical source) plus bar/market context from a TBD bar-data source.
2. Enrich every trade with 25-30 objectively-measured features computed strictly at entry time (point-in-time discipline, see section "Point-in-time discipline").
3. Run edge-discovery analytics (univariate IC + expectancy lift with FDR correction, MFE/MAE distributions, bivariate interactions, rolling-origin walk-forward) to identify which factors actually drive forward returns.
4. Produce a weighted grading rubric derived from data, not intuition, capped at 8 features and stored as versioned YAML.
5. Capture setup observations going forward (including setups not taken) so the dataset is not biased by Allie's discretion. Real-time capture in scope only after Phase 6 MVP; backward replay is required earlier.

### Project roles

- **Allie**: the trader. ~1 year of futures experience, trades MNQ and NQ primarily, logs every trade in Tradezella. Has ADHD. End consumer of the reports, grades, and any dashboards. Owner of all trading-domain decisions (setup definitions, behavioral protocols, conviction logging commitment).
- **Bryce**: sole developer and project owner. Builds and maintains the pipeline. Not the trader. Owns architectural and scope decisions; defers to Allie on trading-domain questions.

**Companion documents**:
- `RESEARCH_PLAN.md`: statistical methodology and acceptance gates.
- `DEVELOPMENT_PLAN.md`: phased build plan, MVP scope, enhancement backlog.
- `PROJECT_KICKOFF.md`: pre-Phase-0 operating contract; must be signed by Allie + Bryce before code is written.

Any conflict between CLAUDE.md and RESEARCH_PLAN.md resolves in favor of RESEARCH_PLAN.md until that doc is amended.

## Stack context

- **Tradezella** — **canonical source for trade fills** (entries, exits, P&L, Allie's tags and notes). All historical trade data flows in from here via CSV export. Holds Strategy field, tag categories (Strategy, Grade, Confluence, Mistake, Context), custom fields, and reporting UI Allie already uses. Stop and target prices may or may not be reliably populated; verified in `PROJECT_KICKOFF.md` section 4.1.
- **Bar data source** — **TBD architectural decision** (`PROJECT_KICKOFF.md` section 4.3). Options: NinjaTrader 8 (if Allie uses it for execution), broker API (Tradovate / TopstepX / AMP), TradingView export, or a paid feed (Databento, Polygon). Required for feature computation; Tradezella does not store OHLCV. **This is a Phase-1 blocker.**
- **NinjaTrader 8** — possible bar data source AND possible execution platform; not assumed. If Allie uses NT, bar data is local (`Documents/NinjaTrader 8/db/`) and free. If she does not, NT is out of scope.
- **TradingView** — analysis platform. Pine Script v6 for setup detection and (post-MVP) real-time alert capture. Subject to bar-count limits and `request.security` repaint risks (see section "Pine Script constraints"). Premium tier ($60/mo) required for webhook alerts; not required for analysis-only MVP.
- **Python 3.11+** — analysis layer. pandas, polars, scikit-learn, shap, matplotlib/plotly, jupyter. Bootstrap and permutation tests use numpy with explicit seeds.
- **Google Sheets + Apps Script** — webhook receiver for TradingView alerts. Only needed if real-time capture is in scope (post-MVP under current "deep analysis" framing).

## Architecture

```
┌─────────────────────┐         ┌─────────────────────┐
│   NinjaTrader 8     │         │    TradingView      │
│  - Trade fills CSV  │         │  - Pine Script      │
│  - OHLCV bars CSV   │         │    setup detector   │
│  - Indicator values │         │  - Feature payload  │
│    via Aeromir or   │         │    in alert JSON    │
│    custom NinjaScript│        │                     │
└──────────┬──────────┘         └──────────┬──────────┘
           │                               │
           │ CSV exports                   │ webhook (real-time)
           ▼                               ▼
┌─────────────────────┐         ┌─────────────────────┐
│  Python join &      │         │  Google Apps Script │
│  enrichment layer   │         │  → Google Sheet     │
│  (historical)       │         │  (forward log,      │
│                     │         │   includes setups   │
│                     │         │   not taken)        │
└──────────┬──────────┘         └──────────┬──────────┘
           │                               │
           └───────────────┬───────────────┘
                           ▼
                ┌─────────────────────┐
                │  Data quality gate  │
                │  (10 checks; >5%    │
                │  failure pauses)    │
                └──────────┬──────────┘
                           ▼
                ┌─────────────────────┐
                │  Enriched dataset   │
                │  (parquet)          │
                │  One row per trade  │
                │  or setup obs       │
                └──────────┬──────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │  Analytics layer    │
                │  - Q1 expectancy    │
                │    bootstrap        │
                │  - Univariate IC +  │
                │    BH-FDR           │
                │  - MFE/MAE          │
                │  - Bivariate ANOVA  │
                │  - Permutation imp. │
                │  - Rolling walk-fwd │
                └──────────┬──────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │  Rubric versioning  │
                │  (YAML, <=8 feats)  │
                │  + HTML report      │
                └──────────┬──────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │  Tradezella tags    │
                │  applied for live   │
                │  outcome tracking   │
                └─────────────────────┘
```

## Repository structure

```
.
├── CLAUDE.md                      # this file (engineering spec)
├── RESEARCH_PLAN.md               # statistical methodology, gates, open questions
├── README.md                      # human-facing setup guide
├── pyproject.toml                 # uv or poetry, Python 3.11+
├── .env.example                   # for any API keys, webhook secrets
├── data/
│   ├── raw/
│   │   ├── ninjatrader/           # raw NT exports (trades, bars)
│   │   └── tradingview/           # exported alert logs from Sheets
│   ├── enriched/                  # joined + feature-engineered parquet
│   └── reports/                   # generated HTML/PDF analytics
├── src/
│   ├── __init__.py
│   ├── config.py                  # named constants: ATR lookback, MA periods, session times, FDR alpha, bootstrap params, walk-forward window sizes
│   ├── ingest/
│   │   ├── ninjatrader.py         # parse NT trade & bar CSVs, normalize to UTC
│   │   ├── tradingview.py         # pull from Google Sheet via API
│   │   └── tradezella.py          # parse TZ export for tag reconciliation
│   ├── dq/
│   │   ├── checks.py              # 10 DQ checks (see section "Data quality")
│   │   └── report.py              # DQ pass/fail report; >5% failure pauses pipeline
│   ├── features/
│   │   ├── price.py               # MA distances, ATR, structure, level distances
│   │   ├── volume.py              # rvol, cumulative volume vs typical
│   │   ├── orderflow.py           # cumulative delta, delta divergence, large lot ratio, bid/ask spread regime
│   │   ├── volatility.py          # ATR regime, range expansion
│   │   ├── temporal.py            # session bucket, minute of session, day of week, trade sequence, open auction proximity, settlement print proximity
│   │   ├── context.py             # SPY/VIX state, scheduled event distance, Globex vs RTH state, correlated instrument state
│   │   ├── futures_calendar.py    # roll week flags, contract-month switches
│   │   ├── trader_state.py        # P&L state, consecutive W/L, trade sequence
│   │   └── outcome.py             # MFE, MAE, time-to-target, R-multiple, forward returns at t+5/15/30/60
│   ├── join.py                    # merge trades with bars at entry timestamp; enforce point-in-time guard
│   ├── analytics/
│   │   ├── expectancy.py          # Q1 bootstrap CI on mean R
│   │   ├── univariate.py          # IC (Spearman) + expectancy lift per feature; permutation p-values; BH-FDR adjustment
│   │   ├── distributions.py       # MFE/MAE histograms by factor
│   │   ├── interactions.py        # pairwise ANOVA on top features
│   │   ├── importance.py          # permutation importance (Strobl et al. 2007), SHAP for interaction discovery
│   │   ├── deflated_sharpe.py     # Bailey & Lopez de Prado 2014 for multi-rubric comparison
│   │   └── walkforward.py         # rolling-origin 6mo train / 1mo test / 1mo step; combinatorial purged CV alternative
│   ├── rubric/
│   │   ├── builder.py             # derive weights from analytics output per weight table
│   │   ├── versioning.py          # track rubric changes; enforce promotion criteria
│   │   └── apply.py               # score a trade given features + rubric
│   └── reports/
│       ├── html.py                # jinja2 templates; every report includes inputs hash, git SHA, rubric version, sample size, # of tests run, FDR alpha
│       └── tradezella_export.py   # generate tag suggestions for TZ
├── pine/
│   ├── setup_trend_pullback.pine  # per-setup detector + grader; barstate.isconfirmed; lookahead=barmerge.lookahead_off
│   ├── setup_breakout.pine
│   └── README.md                  # how to install in TradingView
├── webhook/
│   ├── apps_script.gs             # Google Apps Script for Sheet ingestion; includes taken/reason_not_taken fields
│   ├── schema.md                  # JSON payload contract
│   └── README.md                  # setup guide
├── rubric/
│   └── versions/                  # v1.yaml, v2.yaml, ... never edited after promotion
├── notebooks/
│   ├── 00_dataset_audit.ipynb     # M0: trade counts by month/instrument/setup
│   ├── 01_initial_exploration.ipynb
│   ├── 02_expectancy_baseline.ipynb
│   ├── 03_univariate_analysis.ipynb
│   ├── 04_bivariate_interactions.ipynb
│   ├── 05_rubric_v1.ipynb
│   └── 06_walkforward_validation.ipynb
└── tests/
    ├── test_ingest.py
    ├── test_dq.py
    ├── test_features.py           # synthetic bar series, asserted feature values, point-in-time guard tests
    ├── test_orderflow.py
    └── test_rubric.py
```

## Core data contracts

### Trade record (post-ingest, pre-enrichment)
Required fields from NinjaTrader export, normalized:
- `trade_id` (string, unique)
- `symbol` (e.g., "ES 03-26")
- `instrument_root` (e.g., "ES"; needed for cross-month roll handling)
- `entry_time` (UTC datetime, ISO 8601)
- `exit_time` (UTC datetime, ISO 8601)
- `direction` ("long" | "short")
- `qty` (int, contracts)
- `entry_price` (float)
- `exit_price` (float)
- `stop_price` (float, planned — pulled from NT if available, otherwise reconstruct from journal). See "Stop placement caveat" below.
- `stop_type` (enum: "fixed_tick", "atr", "structure", "mental", "unknown"). Tracked for Q8.
- `target_price` (float, planned — optional)
- `pnl_dollars` (float, net of commission)
- `pnl_r` (float, computed from stop distance; null if stop is "mental" and not reconstructable)
- `commission` (float)
- `roll_proximity_days` (int; bars within 2 days of front-month switch flagged)
- `account_context` (enum: "live", "sim", "funded_challenge", etc.; from journal)

**Stop placement caveat**: if Allie uses mental stops or doesn't log them in Tradezella, the trade record will not have a reliable `stop_price`. R-multiple computation then depends on a post-hoc reconstruction, which biases the metric. When `stop_type = "mental"` and stop is not journaled, the primary outcome shifts to forward-horizon returns (see "Outcome record") rather than R. This is engineering open question 6 and is highest priority to resolve with Allie via `PROJECT_KICKOFF.md` section 3.5.

### Enriched trade record
Above + ~25-30 feature columns. Naming convention: `feat_<domain>_<name>` so analytics can auto-discover feature columns by prefix.

Categories of features the system MUST compute. Categories marked **(added)** were absent from the v0.1 spec per RESEARCH_PLAN.md section 1.1.

- **Price location**:
  - Distance from VWAP / EMA20 / EMA50 / EMA200 in ATR units.
  - Distance from prior-day H/L, overnight H/L, premarket H/L (high signal in opening hour).
- **Volatility regime**:
  - ATR(14) value (used as normalizer, not direct signal).
  - ATR percentile vs trailing 60 days.
  - Range expansion ratio (current bar range / 20-bar avg).
- **Volume**:
  - Relative volume at entry minute vs 20-day same-minute average.
  - Cumulative volume vs typical at that time (test for collinearity with rvol).
- **Order flow microstructure (added)**: empirical support: Cont/Kukanov/Stoikov 2014, Easley/Lopez de Prado/O'Hara 2012.
  - **Cumulative delta at entry** (signed volume since session open). High priority.
  - **Delta divergence flag** at entry bar (delta sign vs price direction on last N bars).
  - **Bid/ask spread regime** (matters for CL/GC; less for ES/NQ in RTH).
  - **Large lot ratio at entry minute** (institutional participation proxy; requires tick data).
- **Temporal**:
  - Minute of session (ET, normalized to RTH open). Use continuous, not bucketed.
  - Session bucket (open/morning/midday/afternoon/close) as a secondary categorical.
  - Day of week (test but expect noise).
  - Trade sequence # of the day (trader state).
  - Minutes since last trade (trader state).
  - **Open auction proximity (added)**: minutes from RTH open, separate from session bucket. First 15 minutes have a documented different return distribution.
  - **Settlement print proximity (added)**: minutes to 3:00 PM ET. ES/NQ closing auction effect.
- **Trigger quality** (practitioner support; treat as moderate prior):
  - Entry candle body-to-range ratio.
  - Upper wick ratio, lower wick ratio.
  - Close position in range.
- **Higher timeframe context**:
  - HTF (60m, 240m, daily) trend direction (categorical, three values).
  - HTF MA stack alignment (boolean per timeframe).
- **Market context**:
  - SPY trend state at entry (moderate for ES/NQ, weaker for CL/GC).
  - VIX level and percentile bucket.
  - **Scheduled event distance (added)**: minutes to next FOMC, CPI, NFP, EIA, ECB, BOJ (CL/GC overnight). Event clustering changes vol regime.
  - **Globex vs RTH state (added)**: which session, time since session transition.
  - **Correlated instrument state (added)**: ES leading NQ, DX state for CL. For non-ES products, cross-instrument signal is often the dominant feature.
- **Futures calendar (added)**:
  - **Roll week flag**: front-month switches contaminate price series. Use as filter or feature.
- **Trader state**:
  - P&L state when entered (green/red/flat in $ and R).
  - Consecutive wins/losses going in.

#### Point-in-time discipline (mandatory)
Every feature must be computed using only data available at `entry_time - 1 tick`. No close-only references, no future smoothing, no `request.security` with `lookahead=barmerge.lookahead_on`. Python feature functions must accept an `as_of` timestamp and raise if any input row's timestamp is `>= as_of`. Tests assert this on every feature. This is the most common source of leakage and silently inflates IC.

#### Collinearity handling
Many of these features are pairwise correlated (VWAP vs EMA20 distance, ATR percentile vs range expansion, rvol vs cumulative volume, HTF trend across timeframes). Procedure before analytics:
1. Pearson and Spearman correlation matrices on all features.
2. Drop or combine pairs with |corr| > 0.80; keep the one with stronger prior support.
3. Variance Inflation Factor check; VIF > 5 flags multicollinearity.
4. Optional hierarchical clustering on correlation distance for cluster representatives.

#### Transformations
Applied in this order:
1. Per-instrument percentile rank for features whose absolute scale differs across instruments.
2. ATR-normalization for distance features.
3. Regime-conditional binning only if RESEARCH_PLAN.md Q5 confirms regime effect.
4. **No PCA at the rubric stage.** The rubric must be human-readable. PCA is acceptable inside `analytics/importance.py` for importance estimation only.

#### Outliers and missing values
- **Continuous**: Z-score per instrument for cross-instrument comparison; raw within-instrument.
- **Boolean**: 0/1 integer.
- **Missing**: distinguish "not applicable" (e.g., no scheduled event in next 24h, encode as max value or separate category) from "data quality failure" (drop trade).
- **Outliers**: winsorize features at 1st / 99th percentile per instrument. **Do not winsorize the outcome (R is fat-tailed by nature).**

### Outcome record (computed from bars between entry and exit, plus forward windows)

Primary outcome metrics:
- **Forward returns at fixed horizons**: `fwd_ret_5min`, `fwd_ret_15min`, `fwd_ret_30min`, `fwd_ret_60min` (price change from entry to entry + horizon, normalized by entry-time ATR). These are the **primary statistical target** because they sidestep stop-placement confounds.
- `mfe_r` (max favorable excursion in R; only meaningful when stop is reliably captured).
- `mae_r` (max adverse excursion in R; same caveat).
- `time_to_mfe_min`, `time_to_mae_min`.
- `hit_1r_before_stop`, `hit_2r_before_stop`, `hit_3r_before_stop` (bool; only when stop captured).
- `bars_in_trade`.
- `mfe_post_exit_30min` (exit-quality signal).

R-multiple stays as a **reporting** metric, not the primary statistical target, until RESEARCH_PLAN.md Q8 (stop-placement contamination) is resolved.

### Setup observation record (counterfactual, captured forward and via backward replay)

For each setup-detector firing, taken or not:
- `setup_id`, `setup_name`, `fired_at` (UTC).
- All feature columns above (same naming).
- `taken` (bool).
- `reason_not_taken` (free text initially; codified to categories after several weeks).
- If taken, foreign key to `trade_id`.

This dataset is the only way to study selection bias. Without it the analytics under-estimates effects on features inside Allie's existing mental rubric and over-estimates effects on features that happen to coincide with her existing filters. Every day without forward capture accrues biased data, so M3 (counterfactual infrastructure) runs in parallel with the historical analytics where feasible. Under the current "deep analysis" framing, real-time forward capture may slip to post-MVP; backward replay from historical bars is still required at M3.

## Statistical methodology (summary; full detail in RESEARCH_PLAN.md)

### Primary outcome metrics, in order of preference
1. **Information Coefficient (IC)**: Spearman rank correlation between feature value and forward 30-minute return per trade. Computed per walk-forward window. p-value via permutation (10,000 shuffles).
2. **Expectancy lift**: E[R | feature in top quintile] minus E[R | baseline], with bootstrap CI. Reported only after IC establishes a relationship exists.
3. **Hit-rate vs payoff decomposition** for each feature that clears the IC threshold (tells you which lever the feature pulls).
4. **Deflated Sharpe Ratio** (Bailey & Lopez de Prado 2014) when comparing many candidate rubric versions to a baseline.

### Multiple-comparisons control
Twenty-five univariate splits at alpha 0.05 produce ~1.25 false positives by chance. **All univariate analytics apply Benjamini-Hochberg FDR correction across the full feature set.** Every analytics output reports the number of tests run, the FDR alpha, and which features cleared. Reference: Harvey, Liu, and Zhu 2016.

### Sample-size floors
The "30 trades per factor" heuristic in v0.1 was below statistical power. Per RESEARCH_PLAN.md section 1.3:
- To detect mean R difference of 0.3R, sigma 1.2R, alpha 0.05, beta 0.20: ~250 trades per arm.
- With Bonferroni across 25 features (alpha 0.002): ~400 per arm.
- Top-vs-bottom-quintile comparison: ~1,250 to 2,000 total trades to detect 0.3R lift with confidence.
- At 200-400 trades (realistic year-one bucket), univariate effect sizes below ~0.5R are statistically undetectable.

**Mitigation**: multi-method triangulation (IC + expectancy lift + permutation importance + walk-forward stability) plus strict FDR. Do not lower the bar by reporting single-test p-values.

**Every analytics report must state the minimum effect size detectable at the current N** and refuse to claim significance below it.

### Walk-forward protocol (replaces the 70/30 single-split in v0.1)
| Parameter | Value | Justification |
|---|---|---|
| Window type | Rolling, fixed-size train | Stationarity is weakest in trading |
| Train length | 6 months | Compromise between sample size and recency |
| Test length | 1 month | Granular enough to detect regime breaks |
| Step | 1 month | Standard |
| Refit cadence | Every step | Tests rubric stability, not just any single fit |
| Embargo | 1 trading day between train and test | Prevents boundary leakage |

For smaller datasets, fall back to **combinatorial purged cross-validation** (Lopez de Prado 2018) with purges and embargoes around test folds.

### Confidence intervals
- **Stationary bootstrap** (Politis & Romano 1994) for time series, block length via Politis-White automatic method.
- **10,000 resamples** for all reported CIs.
- Report both 90% and 95%.

## Rubric construction

### Weight derivation table

| IC (median walk-forward OOS) | Expectancy lift (top quintile vs baseline) | Weight |
|---|---|---|
| > 0.20 | > 0.4R | 3 |
| 0.10 to 0.20 | 0.2 to 0.4R | 2 |
| 0.05 to 0.10 | 0.1 to 0.2R | 1 |
| < 0.05 | < 0.1R | excluded |

### Rules
- A feature must pass **both** IC and expectancy thresholds to earn the higher weight.
- The rubric uses **no more than 8 features**. Driven by sample-size floor: 8 features × ~30 trades/feature lower bound = 240 minimum, aligned with the dataset target.
- Weight sign follows the feature's IC sign in-sample.
- Grade thresholds: A+ = top 20% of rubric scores in-sample, A = next 30%, B = bottom 50%. Re-validated each walk-forward window.
- Weights and thresholds live in versioned YAML under `rubric/versions/`. Once promoted, a version is never edited; new analyses produce a new version.

### Rubric promotion (v_n → v_{n+1})

A new version is promoted only when **all** hold:
1. Walk-forward OOS A+ minus A spread >= 0.3R median across windows, bootstrap CI excluding zero.
2. A+ minus A spread >= 0.2R in at least 60% of individual windows.
3. A minus B spread >= 0.15R median, CI excluding zero.
4. OOS A+ expectancy is statistically indistinguishable from in-sample A+ expectancy (CI overlap; catches overfitting).
5. New rubric beats the prior rubric on the most recent walk-forward window by at least 0.1R on A+ expectancy.
6. Counterfactual test: applied to setups not taken, simulated forward returns show the same monotonicity.
7. Sample size for the change is documented and meets the floors above.

Anything failing stays in a branch and does not become canonical.

## Data quality

Run on ingestion and again on every new data load. Pipeline pauses on any failure rate above 5%.

1. Timestamp continuity (no gaps, duplicates, future timestamps).
2. Bar OHLC integrity (high >= max(open, close), low <= min(open, close), high >= low).
3. Trade fills match bars (entry price within entry-bar OHLC).
4. P&L recomputation matches reported P&L within rounding.
5. R-multiple sanity (no R outside [-5, +20]; investigate anything outside).
6. Session label correctness (entries within RTH or labeled ETH).
7. Instrument symbol normalization (ES, MES, MNQ, NQ are different products with different tick values).
8. Roll boundary flags (trades within 2 days of front-month switch).
9. CME holiday and half-day calendar applied.
10. Commission completeness (no zero-commission trades unless intentionally sim).

## Pine Script constraints

Pine has bar-count limits (typically 5,000 to 20,000 bars on screen, lower with `request.security` calls) and well-documented repaint risks. Feature payloads MUST:
- Compute only on confirmed bars (`barstate.isconfirmed`).
- Use `lookahead=barmerge.lookahead_off` in any `request.security` call.
- Be cross-checked against Python recomputation on the same bar window before being trusted (R11 in RESEARCH_PLAN.md risk register).

Decide before scoping which features live in Pine (real-time, computable from chart bars) and which live in Python (historical, from NT exports with full tick fidelity). Order flow microstructure features generally require tick data and belong in Python.

## "No predicting market direction" — clarified

Several features (HTF trend stack, VIX regime, SPY trend state) are de facto directional signals. The line we draw:

**Features condition on observable state at entry. They do not condition on predicted future state.**

A trend stack is observable. A "predicted breakout success probability from an upstream ML model" is not. ML inside `analytics/importance.py` informs the rubric; it does not become the rubric. The grader stays a transparent weighted-checklist YAML.

## Coding standards

- **Python**: type hints on all public functions. `pydantic` models for trade / feature / setup-observation contracts. pandas for analysis; polars acceptable when performance matters.
- **Formatting**: ruff for lint + format. Line length 100.
- **Testing**: pytest. Every ingest function gets a test with a fixture CSV. Every feature function gets a unit test with a synthetic bar series AND a point-in-time guard test (asserting it fails if given future data).
- **No magic numbers** in feature code. ATR lookback, MA periods, session start times, FDR alpha, bootstrap block length, walk-forward window sizes all live in `src/config.py` as named constants. Rubric weights live in versioned YAML.
- **Reproducibility**: every analytics run writes inputs hash, code git SHA, rubric version, sample size, number of tests run, and FDR alpha to the report header. Set random seeds (numpy, sklearn) on every stochastic step.
- **Timezone discipline**: everything UTC internally. Convert to America/New_York only at display or when bucketing by RTH. Use `zoneinfo`, never `pytz`.
- **No em-dashes** in generated reports or docs. Use commas, parens, or sentence breaks instead.

## How to talk to Bryce (the developer / project owner)

This section governs how the AI assistant communicates with Bryce in chat / PR review / planning.

- **Direct answers first, context after.** Lead with the conclusion.
- **Scannable formatting**: short sections, bullets, bold the key terms. Avoid walls of prose. Avoid em-dashes.
- **Show, don't ask.** If a decision can be defaulted with a reasonable choice, default it and note the assumption. Only ask when there's a real fork that changes the build.
- **Skepticism is welcome.** If a request would lead to overfitting, p-hacking, or premature optimization, push back with the reasoning. Treat Bryce as a technical peer (Senior IT PM and Platform Architect background).
- **Cite sources for any market/platform claim.** NinjaTrader behavior, Tradezella export schema, TradingView API limits, Pine Script syntax all change. Don't guess from training data.
- **Report sample size and number of tests** on every statistical claim. "Feature X has IC 0.12" is incomplete. "Feature X has IC 0.12, N=240, BH-FDR adjusted across 25 tests, walk-forward median across 6 windows" is the right shape.

## How to design outputs for Allie (the trader / end consumer)

This section governs HTML reports, dashboards, alert payloads, exported tag suggestions, README text Allie reads. Allie has ADHD and is the actual customer; reports she can't quickly parse are reports she won't use.

- **Conclusion-first layout.** Every report starts with a one-line takeaway and a 3-bullet summary. Dense detail lives below the summary, not as the lead.
- **Visual over numeric.** A color-coded bar / heatmap / histogram with one annotated insight beats a 20-row table of p-values. The table can exist below the chart, not above it.
- **Color coding consistent project-wide**: green = winning / positive edge / passing gate; red = losing / negative edge / failing gate; gray = inconclusive / insufficient sample.
- **Plain language for stats.** Never write "BH-FDR-adjusted Spearman IC of 0.13 (95% bootstrap CI 0.04-0.22)." Write "Feature X is meaningfully correlated with winners (medium-strong signal, sample size large enough to trust)." Put the technical detail in a collapsible "stats detail" footer or appendix.
- **Per-trade language tied to Allie's vocabulary.** Use the tag names she already uses in Tradezella, not invented analytics jargon. If she calls it "fakeout entry," don't relabel it "Type-3 trigger" in her report.
- **One page, then drill-down.** Default dashboard view is one screen, no scrolling. Click to expand for detail.
- **Short, named insights.** "Your A+ trades on MNQ in the 9:30-10:00 ET window are your highest-edge bucket" beats a paragraph.
- **Honest uncertainty.** When sample size is small, say so plainly in the headline, not in a footnote. "Only 18 trades in this bucket, treat as a hint not a finding."

## Build order

The CLAUDE.md build order is a useful engineering sequence but it must follow the research milestones in RESEARCH_PLAN.md sections M0-M7. **Do not write production-grade analytics code until M2 (Q1 expectancy gate) and M4 (univariate FDR gate) have passed.** Notebooks only before that.

Mapped:

1. **M0 dataset audit** → `notebooks/00_dataset_audit.ipynb`. Count trades by month/instrument/setup. Kill criterion: <100 taken trades or <6 months of consistent rules.
2. **M1 + `src/ingest/ninjatrader.py` + `src/dq/`** → parse NT CSVs, run DQ suite. Kill: DQ pass rate < 95%.
3. **M2** → `notebooks/02_expectancy_baseline.ipynb` answering Q1. Bootstrap CI on mean R. Kill: 95% CI on mean R negative.
4. **M3 (parallel where feasible; real-time may slip to post-MVP)** → Pine setup detector + backward replay. Stand up the counterfactual capture. Validate detector agreement with Allie >= 85% on 30 manual-labeled samples.
5. **`src/features/` minimum viable set** → price location, temporal, outcome (forward returns and MFE/MAE). 8-10 features. Point-in-time tests for each.
6. **`src/join.py`** → for each trade, slice bar window, call feature functions with `as_of=entry_time`. Output enriched parquet.
7. **M4** → `notebooks/03_univariate_analysis.ipynb`. IC + expectancy lift + bootstrap CI + permutation p + BH-FDR across all features. Kill: zero features clear FDR-adjusted IC threshold of 0.10.
8. **M5 + `notebooks/04_bivariate_interactions.ipynb`** → pairwise ANOVA on top 6 features.
9. **M6 + `notebooks/05_rubric_v1.ipynb`** → hand-built `rubric/versions/v1.yaml`. In-sample monotonicity check (mandatory).
10. **M7 + `src/analytics/walkforward.py`** → rolling-origin walk-forward. **GO/NO-GO GATE.** Pass = engineering build resumes. Fail = collect more data or accept that the rubric does not deploy.
11. **Conditional on M7=GO**: remaining feature categories (order flow, HTF context, market context, trader state), Pine alerts in production, refinement to v2.
12. **M9 forward-test (3 months minimum)** → live ex-ante grade vs ex-post outcome tracking. Answers Q4 (does Allie's intuition agree).
13. **`src/analytics/importance.py`** → sklearn + SHAP for non-linear interaction discovery. Informs the rubric; does not replace it.

## Anti-goals

Things this project explicitly does NOT do:

- **No automated execution.** This is an analytics and grading system, not a trading bot. Pine alerts inform; the human pulls the trigger in NinjaTrader.
- **No black-box ML as the grader.** Feature importance from ML informs the rubric. The rubric itself stays as transparent weighted-checklist YAML Allie can read and reason about.
- **No predicting market direction beyond observable state.** Features condition on what is true at entry, never on a predicted future. See section "No predicting market direction".
- **No overfitting to small samples.** Any rubric change requires the sample-size floors in section "Sample-size floors" and walk-forward validation per section "Walk-forward protocol".
- **No subjective fields in the enriched dataset.** Everything is computed from price/volume/time/order-flow. "Trigger looked clean" gets translated to body-ratio thresholds. If you can't define it numerically, it doesn't go in the rubric.
- **No naive p-values across many features.** BH-FDR or Bonferroni mandatory. Report number of tests on every claim.
- **No single-split 70/30 validation.** Rolling-origin walk-forward is the minimum acceptable validation.
- **No taken-trade-only analysis once counterfactual capture is live.** Selection bias is a known structural problem; the counterfactual dataset addresses it.
- **No conviction-based features in the rubric until Q4 has logged 100+ trades.** Pre-trade conviction may carry information but cannot be reconstructed retroactively; shipping it untested would conflate trader skill with feature edge.
- **No rubric deployment without passing M7.** The walk-forward go/no-go gate is real. A "no go" outcome is a successful research finding, not a failure.

## Open questions

Two pools. Engineering open questions affect build scoping. Research open questions in RESEARCH_PLAN.md section 8 must be answered before research design is final; those take priority.

### Engineering open questions

1. **Which 1-3 setups are in scope first?** Need the actual setup definitions from Allie before writing Pine Script and feature lists. See RESEARCH_PLAN.md open question 5 for the falsifiable-predicate format required.
2. **MNQ vs NQ pooling**: confirmed scope is MNQ + NQ. Treat as same-setup-different-sizing (pooled) or as separate populations? `PROJECT_KICKOFF.md` section 3.4.
3. **Bar data source (architectural blocker).** Tradezella stores fills, not OHLCV. NT, broker API, TradingView export, or paid feed. `PROJECT_KICKOFF.md` section 4.3 must be resolved before Phase 1.
4. **How much Tradezella trade history exists?** Drives whether walk-forward is feasible immediately or whether the project pauses for forward capture (M0 kill criterion).
5. **Existing Tradezella tag taxonomy?** Allie already uses Tradezella; whatever tags / custom fields she has become the vocabulary for the rubric outputs. Do not invent new names.
6. **Output format for Allie**: HTML report file emailed weekly, hosted dashboard, Tradezella custom fields populated automatically, or all three? Bryce decision informed by Allie preference.
7. **Stop placement methodology (highest priority).** Determines whether R-multiple or forward-horizon return is the primary outcome. See RESEARCH_PLAN.md open question 1.
8. **Will Allie log 1-5 pre-trade conviction going forward** (via Tradezella custom field or trade note)? Required for Q4. If no, Q4 is dropped.

## References

- Bailey, D. H., and Lopez de Prado, M. (2014). The Deflated Sharpe Ratio: Correcting for Selection Bias, Backtest Overfitting, and Non-Normality. *Journal of Portfolio Management*.
- Cont, R., Kukanov, A., and Stoikov, S. (2014). The Price Impact of Order Book Events. *Journal of Financial Econometrics*.
- Easley, D., Lopez de Prado, M., and O'Hara, M. (2012). Flow Toxicity and Liquidity in a High-Frequency World. *Review of Financial Studies*.
- Harvey, C. R., Liu, Y., and Zhu, H. (2016). ... and the Cross-Section of Expected Returns. *Review of Financial Studies*.
- Lopez de Prado, M. (2018). *Advances in Financial Machine Learning*.
- Politis, D. N., and Romano, J. P. (1994). The Stationary Bootstrap. *Journal of the American Statistical Association*.
- Sackett, D. L. (1979). Bias in Analytic Research. *Journal of Chronic Diseases*.
- Strobl, C., Boulesteix, A.-L., Zeileis, A., and Hothorn, T. (2007). Bias in Random Forest Variable Importance Measures. *BMC Bioinformatics*.
- NinjaTrader 8 historical data export: https://ninjatrader.com/support/helpguides/nt8/exporting.htm
- Aeromir Data Exporter (chart indicators in CSV): https://futures.aeromir.com/data-exporter
- TradingView Pine Script v6 docs: https://www.tradingview.com/pine-script-docs/
- TradingView alert webhooks: https://www.tradingview.com/support/solutions/43000529348/
- Tradezella tag system: https://help.tradezella.com/en/articles/5858184-tag-management-in-tradezella
- QuantNomad TradingView → Google Sheets webhook guide: https://quantnomad.com/how-to-build-an-automatic-trading-journal-from-tradingview-alerts-free-no-zapier/
