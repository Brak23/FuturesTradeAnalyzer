# RESEARCH_PLAN.md

**Status**: Draft v0.1, pre-engineering. Requires Allie sign-off before any code is written.
**Author**: Quant research lead.
**Companion doc**: `CLAUDE.md` (engineering spec).
**Scope**: Research design only. No code, no repo layout, no library choices beyond what `CLAUDE.md` already names.

---

## 0. TL;DR

This plan exists because `CLAUDE.md` is a good engineering spec and a mediocre research spec. The engineering plan is ready to build a pipeline. The research plan it implies would produce a confident-looking rubric on top of statistical noise. Before any code, we need:

1. A spec critique that names the load-bearing assumptions that will, if wrong, sink the project.
2. Falsifiable research questions, with power calcs, instead of "find what separates A+ from B."
3. A counterfactual dataset (setups not taken), not just the taken-trade record.
4. A multiple-comparisons regime, because 25+ features will produce spurious univariate edges by chance.
5. A go/no-go gate after walk-forward, before any live grading system is deployed.

If the historical dataset is small (which it almost certainly is at ~1 year of trading), the most likely correct outcome of this research phase is **"we do not yet have enough data to build a rubric, here is the capture plan to get there in 6 to 9 months."** That is a successful outcome, not a failure. The failure mode is shipping a rubric trained on 150 trades and 20 features and discovering the edge was illusory after another six months of using it.

---

## 1. Critique of `CLAUDE.md`

The spec is well-organized and the anti-goals section is unusually mature. The following items still need correction before research can begin.

### 1.1 The feature list is missing the canonical intraday futures edge factors

`CLAUDE.md` section **Enriched trade record** lists price location, volatility regime, volume, temporal, trigger quality, HTF context, market context, trader state. For intraday futures specifically, the largest body of empirical work (Cont/Kukanov/Stoikov 2014 on order flow imbalance; Easley/Lopez de Prado/O'Hara 2012 on VPIN; Brogaard et al. on price impact) points at **order flow microstructure**, which the spec omits entirely.

Add to feature taxonomy:

| Feature | Why it matters for futures |
|---|---|
| **Cumulative delta at entry** (signed volume since session open) | Direct measure of aggressor pressure; well-documented intraday futures factor |
| **Delta divergence at entry bar** (delta sign vs. price direction on last N bars) | Common practitioner signal, testable |
| **Bid/ask spread regime** | Matters for CL and GC; less for ES/NQ in RTH. If we trade thin contracts, this is a cost-of-entry feature |
| **Trade size distribution at entry minute** (large lot ratio) | Proxy for institutional participation |
| **Open auction proximity** (minutes from RTH open, separately from session bucket) | First 15 minutes have a documented different return distribution |
| **Settlement print proximity** (3:00 PM ET) | ES/NQ have a known closing auction effect |
| **Scheduled event distance** (minutes to FOMC, CPI, NFP, EIA, ECB, BOJ for CL/GC overnight) | Event clustering changes vol regime |
| **Futures-specific calendar** (roll week effects, contract month changes) | Front-month switches contaminate price series if not handled |
| **Globex vs. RTH state** (which session, time since session transition) | Different liquidity regimes |
| **Correlated instrument state** (ES leading NQ, or DX state for CL) | For non-ES products, cross-instrument signal is often the dominant feature |

The spec also lacks **point-in-time discipline**: every feature must be computed using only data available at `entry_time - 1 tick`. The spec implies this but does not name it.

### 1.2 Expectancy by grade bucket is a weak primary metric

`CLAUDE.md` section **Build order step 4** proposes "for each feature, bucket and compute win rate / avg R / expectancy." Expectancy is necessary but it conflates hit rate and payoff, both of which can be confounded by stop placement.

Use instead, as primary metrics:

1. **Forward returns at fixed horizons** (t+5, t+15, t+30, t+60 minutes) as the unbiased outcome label. This sidesteps the stop-placement confound.
2. **Information Coefficient (IC)** (Spearman rank correlation between feature value and forward return) for continuous features.
3. **Expectancy lift** (E[R | feature in top quintile] minus E[R | baseline]) with bootstrap CI as the practitioner-facing translation, **after** IC has established a relationship exists.
4. **Hit-rate vs. payoff decomposition** to know which lever the feature pulls. A feature that improves hit rate without changing payoff is operationally different from one that lets winners run.
5. **Deflated Sharpe Ratio** (Bailey and Lopez de Prado 2014) when comparing many candidate rubric versions to a baseline. Without deflation, the best of N strategies will beat baseline by luck.

R-multiples stay as a reporting metric. They are not the primary statistical target until we have audited how stops were placed (see open question 1).

### 1.3 The 30-trades-per-factor threshold is below statistical power

`CLAUDE.md` **Anti-goals** specifies "30+ trades of evidence per factor and walk-forward validation." This is folk-statistics. A power calculation for the realistic effect size in discretionary day trading looks like:

- Per-trade R has typical sigma roughly 1.0R to 1.5R for systematic edges.
- We care about effect sizes of 0.2R to 0.4R lift on the top bucket (anything smaller is operationally not worth grading for).
- For a two-sample test of mean R difference of 0.3R, sigma 1.2R, alpha 0.05, beta 0.20, the required N per arm is approximately **250 trades**. With Bonferroni correction across 25 features, alpha 0.002 pushes this to roughly 400 per arm.
- For a top vs. bottom quintile comparison, you need ~5x the per-arm count of taken trades, so ~1,250 to 2,000 total taken trades to detect a 0.3R lift with confidence.

If we have 200 to 400 trades (the realistic year-one bucket), we are statistically underpowered for univariate effect sizes below 0.5R. This must be stated upfront. The mitigation is **multi-method triangulation** (IC, expectancy lift, permutation importance, walk-forward stability) rather than relying on any single significance test, plus **strict FDR control** (Benjamini-Hochberg, Harvey and Liu 2014 on multiple-testing inflation in financial research) when running univariate splits across many features.

### 1.4 70/30 walk-forward is naive

`CLAUDE.md` **Build order step 6** proposes "split data 70/30 by time, validate rubric out-of-sample." Single-split validation produces a single OOS estimate with no error bars and is dependent on the arbitrary split point. Replace with:

- **Rolling-origin walk-forward**: train on 6 months, test on next 1 month, step 1 month, repeat across all available history. Yields N estimates of OOS performance with a distribution you can compare to in-sample.
- **Combinatorial purged cross-validation** (Lopez de Prado 2018) if dataset is too small for time-series CV. Purges and embargoes around test folds to prevent leakage.
- **Acceptance criterion stated in advance**: A+ OOS expectancy must exceed A OOS expectancy by some pre-specified delta (proposed: 0.3R with bootstrap CI excluding zero) in at least 60% of walk-forward windows.

### 1.5 Survivorship and selection bias are not addressed

`CLAUDE.md` **Anti-goals** says "everything is computed from price/volume/time." This is correct as a feature constraint. It does not solve the problem that **we are only studying trades Allie decided to take**.

The taken-trade dataset has a hidden filter (Allie's discretion). If a feature like "VWAP distance" was already in her mental rubric, the trades she took will have a restricted range on that feature, and the univariate split will under-estimate its true effect (range restriction, Sackett 1979). Conversely, a feature she ignored may look unusually strong in the taken sample because the strong cases happened to coincide with whatever she *was* looking at (confounding).

The fix is **the counterfactual dataset**: every time the setup logic fires, capture it, regardless of whether Allie took the trade. The CLAUDE.md spec gestures at this in step 7 (Pine alerts) and step 8 (Sheets) but does not call out that the analytics is fundamentally biased without it. This needs to be the **highest priority milestone in parallel with the historical work**, because each day without forward capture is a day of biased data accumulating.

### 1.6 Multiple-comparisons risk across 25+ features

If we run 25 univariate splits at alpha 0.05, we expect 1.25 false positives by chance alone. The spec's anti-goal "30+ trades of evidence per factor" does not address this. We must apply Benjamini-Hochberg FDR control across the full feature set, and we must report the number of tests run on every analytics output. Bailey and Lopez de Prado (2014) and Harvey, Liu, and Zhu (2016) on "false discoveries in financial economics" are the canonical references.

### 1.7 Pine Script historical depth is a hard constraint

`CLAUDE.md` **Build order step 7** proposes Pine Script for real-time capture. Pine has bar-count limits (varies by plan, typically 5,000 to 20,000 bars on screen, lower with `request.security` calls) and `request.security` has well-documented repaint risks on lower-than-chart timeframes. The feature payload sent in the alert must be computed only with confirmed bars (`barstate.isconfirmed`) and only using `lookahead=barmerge.lookahead_off`. This is an implementation detail but it changes what feature set is actually computable in Pine vs. what has to be computed in Python from the chosen bar data source. Decide before scoping which features live in which engine.

### 1.8 Tradezella's stop and target capture is unreliable for mental stops

`CLAUDE.md` **Trade record schema** lists `stop_price` and `target_price` with the parenthetical "planned, pulled from Tradezella if available, otherwise reconstruct." If Allie uses mental stops, moves stops on the fly, or simply doesn't enter them in Tradezella, the stored record will be missing or post-hoc. R-multiple computation depends on the **intended** stop at entry, not the realized exit. This is an open question to confirm with Allie (see section 8) and may force us to use forward-return horizons as the primary outcome rather than R.

### 1.9 The rubric promotion criteria are undefined

`CLAUDE.md` **Repository structure** has `rubric/versioning.py` and version YAMLs but no specification of when v_n → v_n+1 is justified. This plan defines it (section 5.6).

### 1.10 "No predicting market direction" is hand-wavy

`CLAUDE.md` **Anti-goals** says the system does not forecast trade outcomes. Several proposed features (HTF trend stack, VIX regime, SPY trend state) are de facto directional forecasts. The line we will draw is: **features condition on observable state at entry, not on predicted future state**. A trend stack is observable. A "predicted breakout success probability from an upstream ML model" is not. Document this in `CLAUDE.md` if accepted.

---

## 2. Research questions

Five primary questions plus three secondary. Ranked by expected information value. Each is stated as a falsifiable hypothesis with operational definitions, test, and power estimate.

### Q1 (primary, highest IV): Does the trader currently have positive net expectancy after costs?

This must be answered before any rubric work. If the answer is "no" or "uncertain at the available sample size," the project is no longer "grade A vs. B" and becomes "find any edge at all."

- **H1**: Mean per-trade R, net of commissions and slippage, is greater than zero with 95% bootstrap CI excluding zero.
- **H0**: Mean per-trade R is indistinguishable from zero.
- **Test**: Stationary bootstrap (Politis and Romano 1994) of mean R, with block length tuned to the autocorrelation function of the R series. 10,000 resamples.
- **Sample required**: N=200 trades gives ~0.1R CI half-width at typical sigma. N=100 gives ~0.15R. Below 100, the question is not meaningfully answerable.
- **Falsified by**: 95% CI on mean R that includes zero, or that is negative.
- **Decision**: If H0 cannot be rejected, the project pauses and becomes a "find any edge" study. We do not grade B vs. A on a strategy that does not have an aggregate edge.

### Q2 (primary): Does any subset of features produce an out-of-sample IC distinguishable from zero after FDR correction?

- **H1**: At least three features have walk-forward median IC magnitude greater than 0.10, with BH-FDR-adjusted p-value below 0.05.
- **H0**: No feature has IC distinguishable from zero after FDR.
- **Test**: Spearman rank correlation between feature value and forward 30-minute return per trade. IC computed per walk-forward window. p-value via permutation test (shuffle labels within window, 10,000 permutations). BH-FDR adjustment across all features tested.
- **Sample required**: To detect IC=0.15 at alpha=0.002 (after Bonferroni for 25 features) and beta=0.20, N ≈ 350 trades. For IC=0.10, N ≈ 800. Sets the floor on what effect sizes are detectable at our sample size.
- **Falsified by**: No feature passing the threshold across walk-forward windows.
- **Decision**: If falsified, the rubric cannot be built from the current data. Either collect more or refine the feature set. Do not lower the bar.

### Q3 (primary): Does the rubric, built on in-sample data, produce monotonic expectancy across grade buckets out-of-sample?

This is the load-bearing question. A rubric that does not produce A+ > A > B in OOS is not a rubric, it is a memorization of in-sample noise.

- **H1**: In walk-forward, A+ bucket mean R exceeds A bucket by at least 0.3R, and A exceeds B by at least 0.2R, with bootstrap CI on the deltas excluding zero, in at least 60% of walk-forward windows.
- **H0**: Grade buckets are not monotonically ordered in OOS, or order reverses in more than 40% of windows.
- **Test**: Rolling-origin walk-forward (6mo train, 1mo test, 1mo step). Bootstrap CI on bucket-mean R differences within each test window.
- **Sample required**: With 12 months of history and 25 trades per month average, ~300 total trades, supports ~6 walk-forward windows. Tight. May need to extend train window or accept fewer windows.
- **Falsified by**: Buckets not monotonically ordered, or order ordered only by overlap with in-sample-fitted noise.
- **Decision**: This is the go/no-go gate (M8). Fail = do not deploy rubric.

### Q4 (primary): Does the trader's own pre-trade conviction correlate with ex-post outcome better than chance, and does it agree with the data-derived grade?

If Allie already grades trades mentally (A+ / A / B), we should test that signal against outcomes before assuming the data-derived rubric is better. It is plausible that her discretion contains information not captured in any rubric we can build.

- **H1**: Pre-trade self-graded conviction has IC > 0.10 with forward 30-min return.
- **H0**: Self-graded conviction is uncorrelated with outcome.
- **Test**: Spearman rank correlation, requires Allie to log a 1-5 conviction score on entry going forward. Cannot be done retroactively.
- **Sample required**: 100+ live trades with conviction logged.
- **Falsified by**: Conviction IC indistinguishable from zero.
- **Decision**: If H1 holds and IC exceeds data-derived rubric IC, the rubric is downgraded to "decision support" not "decision maker," and we investigate what Allie sees that the features miss.

### Q5 (primary): Is the edge regime-conditional?

If the edge exists only in certain regimes (high vol, trend days, post-FOMC, etc.), we should know that before deploying a regime-agnostic rubric.

- **H1**: At least one regime split (e.g., VIX above vs. below median, ATR percentile above vs. below 50, trend day vs. range day) produces a statistically significant difference in feature ICs.
- **H0**: ICs are stable across regimes.
- **Test**: Interaction term test (Chow-style for continuous splits, or compare regime-conditional ICs with bootstrap CI overlap).
- **Sample required**: Effectively doubles the sample requirement of Q2 because we split.
- **Falsified by**: No regime-conditional ICs.
- **Decision**: If edge is regime-conditional, the rubric becomes regime-conditional. The first feature in the rubric is the regime detector.

### Q6 (secondary): Has the edge attenuated in the most recent 90 days?

Tests for regime change at the meta level.

- **H1**: Mean R in trailing 90 days differs from mean R in prior 180 days by more than 0.2R.
- **H0**: Mean R is stable across the two windows.
- **Test**: Two-sample comparison with bootstrap.
- **Decision**: If edge has decayed, the rubric needs to be built on recent data only, and the walk-forward needs to account for non-stationarity.

### Q7 (secondary): Does any pairwise feature interaction add expectancy lift beyond the sum of marginals?

- **H1**: At least one (feature_i, feature_j) bucket has E[R | top tertile of both] greater than the sum of E[R | top tertile of i alone] and E[R | top tertile of j alone] minus baseline, with bootstrap CI on the interaction term excluding zero.
- **Test**: Two-way ANOVA on R with feature tertile factors, interaction term tested with permutation.
- **Decision**: Interactions inform whether the rubric needs multiplicative weights or threshold combinations rather than additive scoring.

### Q8 (secondary): Does stop placement methodology bias R-multiple comparisons?

- **H1**: R distributions differ systematically when stop is structure-based vs. fixed-tick (or whatever the actual options are).
- **Test**: KS test on R distributions partitioned by stop type.
- **Decision**: If yes, we use forward-horizon returns as the primary outcome, not R.

---

## 3. Dataset requirements

### 3.1 Historical taken trades

| Requirement | Target | Rationale |
|---|---|---|
| Minimum trade count | 300 net of QC drops | Floor for any meaningful univariate analysis with FDR |
| Preferred trade count | 600+ | Allows true OOS walk-forward |
| Calendar duration | 12 months minimum | Spans seasonal regimes (summer doldrums, Q4 vol, holiday weeks, earnings cycles, multiple FOMCs) |
| Per-instrument minimum | 100 trades | If we want per-instrument rubrics. Otherwise pool with instrument as a feature |
| Per-setup minimum | 100 trades | Same logic. Required if rubric differs by setup. |

**Action item before research begins**: count Allie's actual trades by month, instrument, and setup. If the table is sparser than the targets above, the research plan needs to be re-scoped (likely toward collecting more data forward rather than mining the past).

### 3.2 Counterfactual dataset (the part most discretionary trader projects skip)

**Required to address selection bias.** Two sources:

1. **Forward capture from this point on**: every time the Pine setup detector fires, the alert hits the Google Sheet whether Allie took the trade or not. The sheet schema needs a `taken` boolean and a `reason_not_taken` free-text field (codified to categories after a few weeks).
2. **Backward reconstruction**: replay the Pine setup logic over the prior 12 months of bars and emit a setup record for every trigger. Cross-reference against actual trade fills to mark taken vs. not.

The backward reconstruction is the only way to study selection bias on historical data. It depends on the Pine setup detector being a reasonable proxy for Allie's actual mental criteria, which is itself a question (see open question 4).

### 3.3 Contamination risks in the historical record

| Risk | Detection | Action |
|---|---|---|
| **Rule changes mid-period** | Ask Allie to list date-stamped rule changes. Visualize R timeline by day. | Subset analysis to stable rule periods, or use rule version as a feature |
| **Instrument switches** | Count trades per instrument by month | Pool only if statistical tests support pooling |
| **Sizing changes** (1 lot vs. 3 lot) | Look at qty distribution over time | Normalize by R (already done) but flag any period of qty-driven psych changes |
| **Account changes** (sim to live, account size jumps) | Ask Allie. Trade journal entries. | Subset to live-only, or include as covariate |
| **Commission and slippage changes** | Broker statements | Apply current commission to historical to normalize, or use gross R |
| **Roll contamination** | Front-month switches in NT data | Use continuous-contract construction (back-adjusted or ratio-adjusted) for bars; flag trades within 2 days of roll |
| **DST and holiday handling** | Inspect timestamps around DST boundaries | Always use UTC internally, convert at display only (CLAUDE.md already says this) |
| **Missing bars or ticks** | Bar count audit vs. expected per session | Flag and exclude affected trades |

### 3.4 Data quality checks (must run before analysis)

These run once on ingestion and again on any new data:

1. Timestamp continuity (no gaps, no duplicates, no future timestamps).
2. Bar OHLC integrity (high >= max(open, close), low <= min(open, close), high >= low).
3. Trade fills match bars (entry price falls within the entry-bar OHLC).
4. P&L recomputation matches reported P&L within rounding (cross-check NT reported P&L against `(exit - entry) * qty * tick_value - commission`).
5. R-multiple sanity (no trades with R outside [-5, +20]; if any, investigate).
6. Session label correctness (trade entry times within RTH or labeled ETH).
7. Instrument symbol normalization (ES, MES, MNQ, NQ are different products with different tick values).
8. Roll boundary flags (mark trades within N days of front-month switch).
9. Holiday and half-day calendar (CME holiday schedule applied).
10. Commission completeness (no zero-commission trades unless intentionally sim).

Any data quality failure rate above 5% pauses the project until resolved.

---

## 4. Feature taxonomy with statistical justification

### 4.1 Refined feature list with priors

| Category | Feature | Empirical prior support | Notes |
|---|---|---|---|
| Price location | VWAP distance in ATR units | Strong (microstructure, intraday mean-reversion lit) | Primary location feature |
| Price location | EMA20 distance in ATR units | Moderate (trend-following) | Likely collinear with VWAP in trends |
| Price location | EMA50/EMA200 distance | Weak intraday | Keep for HTF context |
| Price location | Distance from prior-day H/L, ON H/L, premarket H/L | Strong (well-documented level effects) | High signal in opening hour |
| Volatility | ATR(14) value | Used as normalizer, not signal directly | |
| Volatility | ATR percentile vs. trailing 60d | Moderate (vol regime) | Continuous and rank versions |
| Volatility | Range expansion ratio (current bar range / 20-bar avg) | Moderate (volatility clustering, ARCH) | |
| Volume | rvol at entry minute | Strong (practitioner consensus, ON futures lit) | Primary volume feature |
| Volume | Cumulative volume vs. typical | Moderate | Redundant with rvol, test for collinearity |
| **Order flow (new)** | **Cumulative delta at entry** | **Strong (Cont/Kukanov/Stoikov 2014)** | **High priority addition** |
| **Order flow (new)** | **Delta divergence flag** | **Moderate (practitioner)** | **Test if Pine/NT can compute reliably** |
| **Order flow (new)** | **Large lot ratio at entry minute** | **Moderate** | **Requires tick data** |
| Temporal | Minute of session (RTH normalized) | Strong (opening hour effect) | Continuous, not bucketed |
| Temporal | Session bucket | Weaker than continuous minute | Keep as categorical |
| Temporal | Day of week | Weak | Test but expect noise |
| Temporal | Trade sequence # of day | Moderate (tilt) | Trader-state feature |
| Temporal | Minutes since last trade | Moderate (tilt) | Trader-state feature |
| Trigger quality | Body-to-range ratio | Practitioner support | |
| Trigger quality | Upper/lower wick ratios | Practitioner support | |
| Trigger quality | Close position in range | Practitioner support | |
| HTF context | HTF trend direction (60m, 240m, daily) | Moderate (trend-following lit) | Categorical, three values |
| HTF context | HTF MA stack alignment | Moderate | Boolean per timeframe |
| Market context | SPY trend state | Moderate for ES/NQ, weaker for CL/GC | Cross-instrument |
| Market context | VIX level | Moderate (vol regime) | |
| Market context | VIX percentile bucket | Moderate | |
| Market context | Scheduled event proximity | Strong (event studies) | Mins to next FOMC/CPI/NFP/EIA |
| **Market context (new)** | **Roll week flag** | **Required for clean prices** | **Filter or feature** |
| Trader state | P&L state at entry (green/red/flat, $ and R) | Moderate (psych, prior performance effects) | |
| Trader state | Consecutive wins/losses going in | Moderate | |

Target initial feature count: 25 to 30 after de-collinearization.

### 4.2 Collinearity handling

Many of these are pairwise correlated. Procedure:

1. Compute Pearson and Spearman correlation matrices on all features.
2. Drop or combine features with absolute correlation > 0.80 (rule of thumb: redundancy starts hurting attribution above this). Keep the one with stronger prior support.
3. Run Variance Inflation Factor (VIF) check on remaining features. VIF > 5 flags multicollinearity that will distort multivariate models.
4. Optional: feature clustering (hierarchical clustering on correlation distance) to identify clusters and pick representatives.

Expected collinear pairs to inspect first:
- VWAP distance vs. EMA20 distance.
- ATR percentile vs. range expansion ratio.
- rvol vs. cumulative volume vs. typical.
- HTF trend direction features across timeframes.

### 4.3 Categorical vs. continuous, missing values, outliers

| Type | Handling |
|---|---|
| Continuous | Z-score per instrument for cross-instrument comparison; raw values for within-instrument analysis. ATR-normalized distances stay in ATR units. |
| Categorical | One-hot for ML models; keep as-is for univariate splits and rubric application. |
| Boolean | 0/1 integer. |
| Missing | Distinguish "not applicable" (e.g., scheduled event proximity when no event in next 24h, treat as max value or separate category) from "data quality failure" (drop trade). |
| Outliers | Winsorize at 1st and 99th percentile per instrument for univariate analysis. Do not drop. Record both winsorized and raw versions. R outcomes are inherently fat-tailed; do not winsorize the label. |

### 4.4 Transformations before analysis

Apply in this order:
1. **Per-instrument percentile rank** for features whose absolute scale differs across instruments (rvol, ATR, distance features).
2. **ATR-normalization** for distance features (already in spec).
3. **Regime-conditional binning** for features expected to be regime-dependent (e.g., distance from VWAP behaves differently on trend days vs. range days). Do this only if Q5 confirms regime effect; otherwise unnecessary complexity.
4. **No PCA or factor compression at the rubric stage**. The rubric needs to be human-readable. PCA is acceptable in the multivariate ML pass (M9) for importance estimation only.

---

## 5. Analytical methodology

### 5.1 Order of analysis

1. **Descriptive**: outcome distribution (R histogram, MFE/MAE, win rate, payoff ratio, by instrument, by month, by session). Detect contamination here.
2. **Q1 answer**: bootstrap CI on mean R. Gate.
3. **Univariate**: IC and expectancy lift per feature, BH-FDR adjusted.
4. **Bivariate**: top-feature pairwise interactions, ANOVA-style.
5. **Multivariate** (later): tree-based importance (random forest, gradient boosting), SHAP for interaction discovery. Informs rubric refinement, not rubric replacement.
6. **Validation**: walk-forward of rubric versions.
7. **Counterfactual validation**: rubric applied to setups not taken, simulated forward returns.

### 5.2 Test selection

| Question | Test | Justification |
|---|---|---|
| Is mean R > 0? | Stationary bootstrap on mean | Handles serial dependence in R series. t-test fails normality |
| Is feature IC > 0? | Spearman correlation + permutation p-value | Rank-based, no Gaussian assumption |
| Do two R distributions differ? | Mann-Whitney U | Non-parametric, R is fat-tailed |
| Is mean R different between buckets? | Bootstrap CI on mean difference | Same fat-tail handling |
| Is feature importance > zero? | Permutation importance (shuffle feature, observe drop in metric) | Robust to correlated features (Strobl et al. 2007) |
| Multi-strategy comparison | Deflated Sharpe Ratio | Corrects for selection bias when picking the best of N (Bailey and Lopez de Prado 2014) |
| Multiple comparisons | Benjamini-Hochberg FDR | Less conservative than Bonferroni, appropriate for exploratory analysis (Harvey, Liu, Zhu 2016) |

### 5.3 Confidence intervals

Use **bootstrap CIs throughout**, not parametric. Specifically:
- **Stationary bootstrap** (Politis and Romano 1994) for time series, with block length selected via Politis-White automatic method.
- **10,000 resamples** for all reported CIs.
- Report 90% and 95% CIs.

### 5.4 Interaction discovery

Three stages:
1. **Manual pairwise** for top 6 features from univariate, using ANOVA interaction terms. Sample size dictates we can only test a handful of pairs without inflating false positives.
2. **Tree-based importance** via random forest, with permutation importance not Gini.
3. **SHAP** for interaction values on a tuned gradient boosting model. Per `CLAUDE.md` **Anti-goals**, this informs the rubric, it does not become the rubric.

### 5.5 Rubric weight derivation

Proposed translation from analytics to rubric weights:

| IC (median, walk-forward OOS) | Expectancy lift (top quintile vs. baseline, R) | Rubric weight |
|---|---|---|
| > 0.20 | > 0.4 | 3 |
| 0.10 to 0.20 | 0.2 to 0.4 | 2 |
| 0.05 to 0.10 | 0.1 to 0.2 | 1 |
| < 0.05 | < 0.1 | excluded |

**Rules**:
- A feature must pass BOTH IC and expectancy thresholds to get the higher weight.
- Total rubric uses **no more than 8 features**. This is a regularization choice driven by the sample size argument in 1.3 (rule of thumb: at least 30 trades per feature in the rubric, capping at 8 means we need 240 trades minimum, which aligns with section 3.1).
- Direction (sign) of weight follows the feature's IC sign in-sample.
- Grade thresholds: A+ = top 20% of rubric scores in-sample, A = next 30%, B = bottom 50%. Re-validated each walk-forward window.
- Thresholds and weights are **stored in versioned YAML** per `CLAUDE.md` and never changed without a documented decision and re-validation.

### 5.6 Walk-forward protocol

| Parameter | Value | Justification |
|---|---|---|
| Window type | Rolling, fixed-size train | Stationarity assumption is weakest in trading, prefer rolling to expanding |
| Train length | 6 months | Compromise between sample size and recency |
| Test length | 1 month | Granular enough to detect regime breaks |
| Step | 1 month | Standard |
| Refit cadence | Every step | We are testing rubric stability, not just any single fit |
| Embargo | 1 trading day between train and test | Prevents leakage from features computed near the boundary |
| Purge | None on point-in-time features. Required if any feature uses forward-looking smoothing (which it should not) | |

### 5.7 Rubric promotion (v_n to v_{n+1})

A new rubric version is promoted only when **all** of the following hold:

1. Walk-forward OOS A+ minus A spread is >= 0.3R in median across windows, with bootstrap CI excluding zero.
2. A+ minus A spread is >= 0.2R in at least 60% of individual windows.
3. A minus B spread is >= 0.15R median, CI excluding zero.
4. OOS A+ expectancy is statistically indistinguishable (CI overlap) from in-sample A+ expectancy. Catches overfitting.
5. New rubric beats prior rubric (v_n) on the most recent walk-forward window's OOS data by at least 0.1R on A+ expectancy.
6. Counterfactual test: applied to setups not taken (once we have that dataset), simulated forward returns show same monotonicity.
7. Sample size for the change is documented and meets section 3.1 floor.

Any change failing any of these stays in a branch and does not become canonical.

---

## 6. Risk register

| # | Risk | Probability | Impact | Detection | Mitigation |
|---|---|---|---|---|---|
| R1 | **Overfitting** to in-sample noise | High | High | OOS-IS gap > 50% on key metrics; rubric weights unstable across walk-forward windows | Cap at 8 features. FDR correction. Walk-forward as primary validation. Hold out a final "untouched" 3 months. |
| R2 | **Look-ahead bias** in features | Medium | High | Point-in-time replay test: recompute features as of entry minus 1 tick; compare to "as of now." | Feature code reviewed for any use of close-only or future data. `barstate.isconfirmed` in Pine. Strict timestamp guard in Python feature functions. |
| R3 | **Selection bias** (taken trades only) | Certain | High | No detection possible without counterfactual data | Counterfactual capture (forward + backward) as a parallel high-priority track |
| R4 | **Regime change** between IS and OOS | High | Medium | Rolling OOS performance trend; Q6 result | Rolling walk-forward sees this. Regime-conditional rubrics if Q5 supports. Recency-weighted training. |
| R5 | **Hawthorne effect** (measurement changes behavior) | High | Medium | Compare trade quality metrics pre/post the start of conscious grading | Blind grading (compute grade post-trade, do not show pre-entry) during the study. Reveal only after baseline established. |
| R6 | **Feature leakage** (feature correlated with outcome via shared cause) | Medium | High | Audit each feature for dependence on exit or post-entry data; correlate features with stop and target placement | Code review on every feature function. Test with synthetic data where ground truth is known. |
| R7 | **Multiple-comparisons false positives** | Certain | High | Naive p-values across 25+ features will produce false discoveries | BH-FDR throughout. Deflated Sharpe for multi-rubric comparison. Report number of tests on every result. |
| R8 | **Sample size insufficient** for desired effect | High | High | Power calculations in section 2 | State the floor effect size detectable at current N on every result. Refuse to claim significance below it. |
| R9 | **Stop placement contamination** of R-multiple | Medium | Medium | Q8 test | Use forward-horizon returns as primary outcome if Q8 confirms |
| R10 | **Roll contamination** in bar data | Medium | Medium | Visual audit of price series around roll dates | Continuous-contract construction. Flag and possibly exclude trades within 2 days of roll. |
| R11 | **Pine Script repaint or lookahead** in feature payload | Medium | High | Replay Pine alerts against historical bars and compare to chart values | `barstate.isconfirmed`. `lookahead=barmerge.lookahead_off`. Cross-check Pine values against Python recomputation. |
| R12 | **Tradezella sync drift** between broker fills and Tradezella record | Low | Medium | Spot-check broker statements vs Tradezella export monthly | Tradezella is canonical for the analysis. If broker integration drops trades, the analysis is wrong upstream. |
| R13 | **Trader rule changes** mid-study | High | High | Quarterly check-in with Allie, journal review | Subset analysis. Treat rule version as a feature or a partition. |
| R14 | **Misattribution** of edge to feature vs. confounder | High | Medium | Bivariate analysis; tree-based importance; SHAP | Multivariate validation phase before rubric finalization |
| R15 | **Counterfactual replay inaccurate** to Allie's actual mental criteria | Medium | High | Manual review of 30 random replayed setups with Allie. She labels which she would have taken; compare to setup detector output. | Iterate Pine setup logic until agreement > 85%. Calibration before relying on counterfactual stats. |

---

## 7. Sequenced research milestones

Replaces `CLAUDE.md` **Build order**. The CLAUDE.md build order is correct as an engineering sequence, but research milestones must precede most of it. The mapping:

- M0 to M3 are pure research, no production code. Notebooks only.
- M4 to M6 are the rubric construction loop.
- M7 is the go/no-go gate.
- M8 onward is where the CLAUDE.md build order resumes.

Time estimates assume ~half-time effort.

### M0: Stakeholder alignment and dataset audit (Week 1)
- **Objective**: Answer the open questions in section 8. Count actual trades by month, instrument, setup.
- **Inputs**: Allie's answers. Tradezella trade export.
- **Methods**: Read trade log. Tabulate. Interview Allie.
- **Deliverable**: `notebooks/00_dataset_audit.ipynb`. Memo of decisions and counts. Updated open questions list.
- **Decision gate**: Do we have enough taken trades to start (target 200+)? If not, this entire research phase pauses and we go straight to forward-capture infrastructure (M3-only) to accumulate data.
- **Kill criteria**: <100 taken trades, or fewer than 6 months of consistent rules. In that case, project becomes "build the capture system and revisit in 6 months."

### M1: Data ingestion and quality (Week 2)
- **Objective**: Get clean trade and bar dataframes that pass all DQ checks in section 3.4.
- **Inputs**: Tradezella trade export, bar data from the chosen source (`CLAUDE.md` "Stack context"; resolved in `PROJECT_KICKOFF.md` section 4.3).
- **Methods**: Ingest, normalize, run DQ suite.
- **Deliverable**: `data/enriched/trades.parquet` (un-enriched yet, just clean trades + bars). DQ report.
- **Decision gate**: DQ pass rate >= 95%. Below that, fix sources before proceeding.

### M2: Descriptive baseline and Q1 (Week 3)
- **Objective**: Answer Q1 (does Allie have positive expectancy?). Produce baseline metrics.
- **Inputs**: M1 output.
- **Methods**: Descriptive stats, bootstrap CI on mean R, hit rate, payoff, distribution plots.
- **Deliverable**: Baseline chart pack and memo.
- **Decision gate**: **Q1 answer**. If 95% CI on mean R excludes zero positively, proceed. If it includes zero, the project changes shape.
- **Kill criteria**: Mean R 95% CI negative (Allie is net-losing at current sample). Project pauses for review of whether the strategy itself needs work before grading.

### M3: Counterfactual capture infrastructure (Weeks 2 to 4, parallel)
- **Objective**: Stand up Pine setup detector(s) and webhook to Sheet. Backward-replay setup detector over last 12 months of bars.
- **Inputs**: Allie's setup definitions (open question 5). 12 months of bar data.
- **Methods**: Pine Script implementation. Apps Script webhook receiver. Python replay of setup logic.
- **Deliverable**: Forward-capture pipeline running. Historical replay dataset of setups (taken + not).
- **Decision gate**: Setup detector agreement with Allie on 30 manual-labeled samples >= 85%. If lower, iterate.
- **Note**: This runs in parallel because every day without it accrues biased data.

### M4: Univariate feature analysis (Weeks 4 to 5)
- **Objective**: Answer Q2 (do any features pass FDR-adjusted significance?).
- **Inputs**: M1 trades + M3 counterfactuals (when available; historical for now).
- **Methods**: Compute all features, run IC + expectancy lift + bootstrap CI + permutation p-values + BH-FDR for each.
- **Deliverable**: Ranked feature table. Per-feature distribution plots. Memo on which features look promising.
- **Decision gate**: At least 3 features pass FDR-adjusted IC threshold of 0.10. If not, stop and revisit feature set or sample size.
- **Kill criteria**: Zero features pass. Either features are wrong, sample is too small, or no edge structure exists in this data. Revisit M0.

### M5: Bivariate and interaction analysis (Week 6)
- **Objective**: Answer Q7. Identify any interaction effects that matter for rubric construction.
- **Inputs**: M4 winners.
- **Methods**: Pairwise ANOVA on top 6 features. Heatmaps of expectancy by tertile-tertile cells.
- **Deliverable**: Interaction memo. Updated feature shortlist.
- **Decision gate**: None hard. Informs M6.

### M6: Rubric v1 (Week 7)
- **Objective**: Hand-built rubric YAML from M4 and M5 outputs.
- **Inputs**: M4, M5.
- **Methods**: Apply weight derivation table (5.5). Choose features. Set thresholds.
- **Deliverable**: `rubric/versions/v1.yaml`. In-sample validation that A+ > A > B by section 5.7 thresholds.
- **Decision gate**: In-sample monotonicity holds. (Mandatory; in-sample failure ends the rubric attempt.)

### M7: Walk-forward OOS validation (Weeks 8 to 9). **GO / NO-GO GATE.**
- **Objective**: Answer Q3.
- **Inputs**: M6 rubric, full historical dataset.
- **Methods**: Section 5.6 protocol.
- **Deliverable**: Walk-forward results memo. Bucket-expectancy chart per window. Stability metrics.
- **Decision gate**: Section 5.7 promotion criteria.
- **GO**: Proceed to M8 engineering work.
- **NO-GO**: Project does not graduate from research. We either collect more data (return to M3 and wait), or we accept that the edge is not large enough to systematize and the rubric does not get deployed. **This is the most important gate in the plan.** Failing it is a successful outcome of research: it has correctly identified that we should not waste engineering effort on noise.

### M8: Engineering build (CLAUDE.md build order resumes)
Conditional on M7 = GO. The CLAUDE.md build order takes over from here, with the modification that several research milestones have already produced parquet outputs and notebook artifacts that the engineering work consumes.

### M9: Forward-test the rubric live (3 months minimum)
- **Objective**: Validate that the rubric works in real-time, not just on retrospective data. Answer Q4 (does Allie's intuition agree).
- **Inputs**: Live setups, live grading, live outcomes.
- **Methods**: Compare ex-ante grade vs. ex-post outcome. Track expectancy by grade.
- **Deliverable**: Monthly forward-test report.
- **Decision gate**: After 3 months, OOS expectancy by grade should match walk-forward expectations within bootstrap CI. If not, rubric is retrained.

### M10: Refinement loop
Quarterly re-fit on accumulated data, with promotion criteria from 5.7.

---

## 8. Open questions to escalate to Allie (research design only)

These are different from the engineering open questions at the end of `CLAUDE.md`. These must be answered before research design is final.

1. **Stop placement methodology**: Fixed-tick? ATR-based? Structure-based? Mental? Combination? If mental, is the intended stop logged anywhere at entry, or only recoverable post-hoc from where you would have exited? Affects whether R-multiple or forward-return is the primary outcome.

2. **Position sizing logic**: Do you size all trades 1 lot, or does conviction translate to contract count? If sizing varies, that itself is a labeled grade we should test against the rubric (does your conviction-sized trade outperform your default-sized trade?).

3. **Rule changes in the historical window**: List with dates every change to your entry criteria, exit criteria, stop methodology, sizing, instrument, time-of-day filter in the period under study. Be ruthless; partial recall is worse than no recall.

4. **What you believe your edge is**: In one paragraph, what do you think makes your A+ setups A+? Concrete factors, your honest belief. We will explicitly test this belief against the data. If the data agrees, that's a confidence multiplier. If it disagrees, that's the most important finding the study produces.

5. **Setup definitions, written in falsifiable terms**: For each setup in scope, the entry trigger as a logical predicate (e.g., "long ES if price within 0.3 ATR of EMA20 AND 60m trend is up AND prior bar made a higher low"). Without this, the Pine detector cannot be built and the counterfactual is broken.

6. **Records of setups not taken**: Do you have any historical record (screenshots, journal entries, screen recordings) of setups you saw and passed on? Even partial would help bound the counterfactual replay validation.

7. **Self-graded conviction logging**: Will you commit to logging a 1-5 pre-trade conviction score on every trade going forward? Required for Q4. Five seconds per trade. If no, Q4 is dropped.

8. **Account context**: Was any of the historical period sim, copy-traded, or otherwise behaviorally different from your current live trading? Account size changes? Funded-account challenge periods? These all affect psych and sizing.

9. **Time horizon for system use**: Once a rubric is built, do you intend to use the live grade to decide whether to take a setup, only to filter sizing, or only to journal after-the-fact? The first is the strongest test but the highest contamination risk (your behavior changes the data you collect for refinement). Decide upfront.

10. **A setup or factor you'd bet has zero edge**: Name one. We will include it in the analysis as a negative control. If our methodology finds an edge there, the methodology is broken before it finds anything real.

11. **What counts as a "trade"**: A scratch, a quick scalp, a partial scale, a re-entry after stop. Define the unit of analysis. (Tradezella aggregates by "trade" in its own way; we may need to disaggregate or aggregate differently.)

12. **Hard constraints on the rubric**: Are there factors you refuse to grade on for any reason (e.g., "I don't care about day-of-week, don't put it in the rubric even if data says it matters")? Codify upfront so we don't build something you won't use.

---

## 9. Acceptance for this plan

This document is approved when:

1. Allie has answered, in writing, all items in section 8.
2. The dataset audit (M0) has confirmed the sample size required by section 2 is achievable, or has determined that research is paused for forward capture.
3. The risks in section 6 are accepted, and the mitigations are agreed to as binding.
4. The go/no-go criterion in M7 is accepted as a real gate that can return a "no go" without it being a failure of the research lead.

If any of those four is not done, the plan is not approved.

---

## 10. References

- Bailey, D. H., and Lopez de Prado, M. (2014). The Deflated Sharpe Ratio: Correcting for Selection Bias, Backtest Overfitting, and Non-Normality. *Journal of Portfolio Management*.
- Cont, R., Kukanov, A., and Stoikov, S. (2014). The Price Impact of Order Book Events. *Journal of Financial Econometrics*.
- Easley, D., Lopez de Prado, M., and O'Hara, M. (2012). Flow Toxicity and Liquidity in a High-Frequency World. *Review of Financial Studies*.
- Harvey, C. R., Liu, Y., and Zhu, H. (2016). ... and the Cross-Section of Expected Returns. *Review of Financial Studies*. (Multiple-testing inflation in finance.)
- Lopez de Prado, M. (2018). *Advances in Financial Machine Learning*. (Combinatorial purged CV, walk-forward best practices.)
- Politis, D. N., and Romano, J. P. (1994). The Stationary Bootstrap. *Journal of the American Statistical Association*.
- Sackett, D. L. (1979). Bias in Analytic Research. *Journal of Chronic Diseases*. (Range restriction and selection bias.)
- Strobl, C., Boulesteix, A.-L., Zeileis, A., and Hothorn, T. (2007). Bias in Random Forest Variable Importance Measures. *BMC Bioinformatics*. (Permutation importance.)

---

*End of plan. Awaiting Allie sign-off and answers to section 8.*
