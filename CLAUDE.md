# Futures Setup Grading & Edge Discovery System

## Project purpose

Build a repeatable, data-driven analytics system that takes a working-but-mushy futures trading strategy and empirically defines what separates A+ setups from A setups from B setups. The system must:

1. Ingest historical trade data and market context automatically
2. Enrich every trade with 20+ objectively-measured features at entry time
3. Run edge-discovery analytics (univariate splits, MFE/MAE distributions, feature interactions, walk-forward validation) to identify which factors actually drive outcomes
4. Produce a weighted grading rubric derived from data, not intuition
5. Capture real-time setup observations going forward (including setups not taken) for ongoing rubric refinement

The user is ~1 year into futures trading, treats this as a quant problem, and refuses to do manual chart review when a script can do it.

## Stack context

- **NinjaTrader 8** — primary execution platform. Source of trade fills, historical tick/minute futures data. Stored locally in `Documents/NinjaTrader 8/db/`. Exports go to CSV (UTF-8, UTC timestamps, 3-month max blocks via UI).
- **TradingView** — primary analysis platform. Pine Script v6 for feature computation and setup detection. Webhook alerts for real-time capture.
- **Tradezella** — outcome dashboard. Auto-syncs from NinjaTrader. Holds Strategy field, tag categories (Strategy, Grade, Confluence, Mistake, Context), and reporting. Not the source of analytical truth — it's the monitor on top of the lab.
- **Python 3.11+** — analysis layer. pandas, polars, scikit-learn, shap, matplotlib/plotly, jupyter.
- **Google Sheets + Apps Script** — real-time webhook receiver for TradingView alerts. Free, no Zapier.

## Architecture

```
┌─────────────────────┐         ┌─────────────────────┐
│   NinjaTrader 8     │         │    TradingView      │
│  - Trade fills CSV  │         │  - Pine Script      │
│  - OHLCV bars CSV   │         │    setup detector   │
│  - Indicator values │         │  - Feature payload  │
│    via Aeromir or   │         │    in alert JSON    │
│    custom NinjaScript│         │                     │
└──────────┬──────────┘         └──────────┬──────────┘
           │                               │
           │ CSV exports                   │ webhook (real-time)
           ▼                               ▼
┌─────────────────────┐         ┌─────────────────────┐
│  Python join &      │         │  Google Apps Script │
│  enrichment layer   │         │  → Google Sheet     │
│  (historical)       │         │  (forward log)      │
└──────────┬──────────┘         └──────────┬──────────┘
           │                               │
           └───────────────┬───────────────┘
                           ▼
                ┌─────────────────────┐
                │  Enriched dataset   │
                │  (parquet/csv)      │
                │  One row per trade  │
                │  or setup obs       │
                └──────────┬──────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │  Analytics layer    │
                │  - Univariate       │
                │  - MFE/MAE          │
                │  - Interactions     │
                │  - Feature import.  │
                │  - Walk-forward     │
                └──────────┬──────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │  Rubric versioning  │
                │  + HTML report      │
                └─────────────────────┘
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
├── CLAUDE.md                      # this file
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
│   ├── ingest/
│   │   ├── ninjatrader.py         # parse NT trade & bar CSVs
│   │   ├── tradingview.py         # pull from Google Sheet via API
│   │   └── tradezella.py          # parse TZ export for tag reconciliation
│   ├── features/
│   │   ├── price.py               # MA distances, ATR, structure
│   │   ├── volume.py              # rvol, volume profile basics
│   │   ├── volatility.py          # ATR regime, range expansion
│   │   ├── temporal.py            # session bucket, day of week, time since last trade
│   │   ├── context.py             # SPY/VIX state, news day flags
│   │   └── outcome.py             # MFE, MAE, time-to-target, R-multiple
│   ├── join.py                    # merge trades with bars at entry timestamp
│   ├── analytics/
│   │   ├── univariate.py          # split + expectancy per factor
│   │   ├── distributions.py       # MFE/MAE histograms by factor
│   │   ├── interactions.py        # pairwise factor combinations
│   │   ├── importance.py          # sklearn + SHAP feature importance
│   │   └── walkforward.py         # in-sample/out-of-sample validation
│   ├── rubric/
│   │   ├── builder.py             # derive weights from analytics output
│   │   ├── versioning.py          # track rubric changes over time
│   │   └── apply.py               # score a trade given features + rubric
│   └── reports/
│       ├── html.py                # jinja2 templates for analytics output
│       └── tradezella_export.py   # generate tag suggestions for TZ
├── pine/
│   ├── setup_trend_pullback.pine  # per-setup detector + grader
│   ├── setup_breakout.pine
│   └── README.md                  # how to install in TradingView
├── webhook/
│   ├── apps_script.gs             # Google Apps Script for Sheet ingestion
│   ├── schema.md                  # JSON payload contract
│   └── README.md                  # setup guide
├── notebooks/
│   ├── 01_initial_exploration.ipynb
│   ├── 02_feature_engineering.ipynb
│   ├── 03_univariate_analysis.ipynb
│   ├── 04_rubric_v1.ipynb
│   └── 05_walkforward_validation.ipynb
└── tests/
    ├── test_ingest.py
    ├── test_features.py
    └── test_rubric.py
```

## Core data contracts

### Trade record (post-ingest, pre-enrichment)
Required fields from NinjaTrader export, normalized:
- `trade_id` (string, unique)
- `symbol` (e.g., "ES 03-26")
- `entry_time` (UTC datetime, ISO 8601)
- `exit_time` (UTC datetime, ISO 8601)
- `direction` ("long" | "short")
- `qty` (int, contracts)
- `entry_price` (float)
- `exit_price` (float)
- `stop_price` (float, planned — pulled from NT if available, otherwise reconstruct)
- `target_price` (float, planned — optional)
- `pnl_dollars` (float, net of commission)
- `pnl_r` (float, computed from stop distance)
- `commission` (float)

### Enriched trade record
Above + ~25 feature columns. Naming convention: `feat_<domain>_<name>` so analytics can auto-discover feature columns by prefix.

Categories of features the system MUST compute:
- **Price location**: distance from VWAP / EMA20 / EMA50 / EMA200 in ATR units; distance from prior day H/L; distance from overnight H/L; distance from premarket H/L
- **Volatility regime**: ATR(14) value; ATR percentile vs trailing 60 days; range expansion ratio
- **Volume**: relative volume at entry minute vs 20-day same-minute average; cumulative volume vs typical at that time
- **Temporal**: minute of session (ET, normalized to RTH open); session bucket (open/morning/midday/afternoon/close); day of week; trade sequence # of the day; minutes since last trade
- **Trigger quality**: entry candle body-to-range ratio; upper wick ratio; lower wick ratio; close position in range
- **Higher timeframe context**: HTF (60m, 240m, daily) trend direction; HTF MA stack alignment
- **Market context**: SPY trend state at entry; VIX level and percentile bucket; economic calendar flag (FOMC, CPI, NFP, etc.)
- **Trader state**: P&L state when entered (green/red/flat in $ and R); consecutive wins/losses going in

### Outcome record (computed from bars between entry and exit, plus a forward window)
- `mfe_r` (max favorable excursion in R)
- `mae_r` (max adverse excursion in R)
- `time_to_mfe_min`
- `time_to_mae_min`
- `hit_1r_before_stop` (bool)
- `hit_2r_before_stop` (bool)
- `hit_3r_before_stop` (bool)
- `bars_in_trade`
- `mfe_post_exit_30min` (did it keep going after you exited — exit quality signal)

## Coding standards

- **Python**: type hints required on all public functions. Use `pydantic` models for the trade/feature contracts above. Pandas for analysis, polars acceptable if performance matters.
- **Formatting**: ruff for lint + format. Line length 100.
- **Testing**: pytest. Every ingest function gets a test with a fixture CSV. Every feature function gets a unit test with a synthetic bar series and asserted output value.
- **No magic numbers** in feature code. ATR lookback, MA periods, session start times all go in `src/config.py` as named constants. The rubric weights live in versioned YAML files under `rubric/versions/`.
- **Reproducibility**: every analytics run writes its inputs hash, code git SHA, and rubric version to the report header. Set random seeds (numpy, sklearn) for any stochastic step.
- **Timezone discipline**: store everything UTC internally. Convert to America/New_York only at the display layer or when bucketing by RTH session. Use `zoneinfo`, never `pytz`.
- **No em-dashes in generated reports or docs**. Use commas, parens, or sentence breaks instead.

## How to talk to the user

- **Direct answers first, context after.** Bryce processes information faster when the conclusion leads.
- **ADHD-friendly formatting**: short sections, scannable bullets, bold the key terms. Avoid walls of prose. Avoid em-dashes.
- **Show, don't ask.** If a decision can be defaulted with a reasonable choice, default it and note the assumption. Only ask when there's a real fork that changes the build.
- **Skepticism is welcome.** If a request would lead to overfitting, p-hacking, or premature optimization, push back with the reasoning. Bryce is a Senior IT PM and Platform Architect; treat him as a technical peer.
- **Cite sources for any market/platform claim**. NinjaTrader behavior, TradingView API limits, Pine Script syntax all change. Don't guess from training data.

## Build order (suggested)

If starting from zero, build in this sequence. Each step produces something usable before moving on.

1. **`src/ingest/ninjatrader.py`** — parse a trade CSV and a bars CSV into pandas DataFrames with the schema above. Write tests. *Output: clean trade + bar dataframes.*
2. **`src/features/` minimum viable set** — price location (VWAP/EMA distances), temporal (minute of session, day of week), outcome (MFE/MAE/R). Maybe 8 features. *Output: enriched dataframe with the basics.*
3. **`src/join.py`** — for each trade, slice the relevant bar window and call feature functions. *Output: one enriched parquet file.*
4. **`notebooks/03_univariate_analysis.ipynb`** — for each feature, bucket and compute win rate / avg R / expectancy. Visualize. *Output: ranked list of candidate edge factors.*
5. **`notebooks/04_rubric_v1.ipynb`** — hand-build first rubric YAML from the top factors. Apply to historical trades. Check that A+ > A > B in expectancy on the in-sample data. *Output: `rubric/versions/v1.yaml`.*
6. **`src/analytics/walkforward.py`** — split data 70/30 by time, validate rubric out-of-sample. *Output: pass/fail decision on v1.*
7. **Pine Script for one setup** — replicate the rubric in Pine, fire alerts with feature payload. *Output: real-time observation capture.*
8. **Google Apps Script webhook receiver** — write alerts to Sheet. *Output: forward-looking dataset accumulating.*
9. **Remaining feature categories** (volume, volatility regime, HTF context, market context, trader state). Re-run analytics. Refine rubric to v2. *Output: `rubric/versions/v2.yaml` + improved out-of-sample stats.*
10. **`src/analytics/importance.py`** — sklearn + SHAP for non-linear interactions. *Output: factor combinations that hand-built rubric missed.*

## Anti-goals

Things this project explicitly does NOT do:

- **No automated execution.** This is an analytics and grading system, not a trading bot. Pine alerts inform; the human pulls the trigger in NinjaTrader.
- **No black-box ML for grading.** Feature importance from ML informs the rubric, but the rubric itself stays as transparent weighted-checklist YAML that Bryce can read and reason about. No model deployed as the grader.
- **No predicting market direction.** This system grades setup *quality* using factors known at entry. It does not forecast whether a given trade will win.
- **No overfitting to small samples.** Any rubric change requires 30+ trades of evidence per factor and walk-forward validation. Document the sample size on every claim.
- **No subjective fields in the enriched dataset.** Everything is computed from price/volume/time. "Trigger looked clean" gets translated to body-ratio thresholds. If you can't define it numerically, it doesn't go in the rubric.

## Open questions to resolve in conversation

These need Bryce's input before key build decisions:

1. **Which 1–3 setups are in scope first?** Need the actual setup definitions before writing Pine Script and feature lists.
2. **Which futures contracts?** ES, NQ, CL, GC, MES/MNQ? Affects tick value, session times, and which market context features matter.
3. **How much trade history exists?** Drives whether walk-forward is feasible immediately or whether we wait for more data.
4. **Existing Tradezella tag taxonomy?** If tags already exist, normalize to those names instead of inventing new ones.
5. **Comfortable with pandas/notebooks, or prefer everything as CLI scripts with HTML reports?** Affects how the analytics layer is exposed.

## References

- NinjaTrader 8 historical data export: https://ninjatrader.com/support/helpguides/nt8/exporting.htm
- Aeromir Data Exporter (chart indicators in CSV): https://futures.aeromir.com/data-exporter
- TradingView Pine Script v6 docs: https://www.tradingview.com/pine-script-docs/
- TradingView alert webhooks: https://www.tradingview.com/support/solutions/43000529348/
- Tradezella tag system: https://help.tradezella.com/en/articles/5858184-tag-management-in-tradezella
- QuantNomad TradingView → Google Sheets webhook guide: https://quantnomad.com/how-to-build-an-automatic-trading-journal-from-tradingview-alerts-free-no-zapier/
