# RESEARCH_PLAN.md

**Status:** Draft for trader sign-off. No code is written until this is approved.
**Owner:** Quant research lead.
**Companion doc:** `CLAUDE.md` (engineering spec). This plan supersedes the *Build Order* section of `CLAUDE.md` until research milestones 0 through 7 are complete.

> **Note on source material.** The CLAUDE.md spec was summarized in the engagement brief but is not yet checked into the repo. This plan cites the brief's section names (*Feature Categories*, *Data Contracts*, *Build Order*, *Anti-Goals*, etc.). Any disagreement between this plan and the eventual CLAUDE.md must be resolved in favor of this plan or escalated to the trader.

---

## 0. TL;DR

We are not yet in a position to grade setups. Before any grading rubric is built, we have to (a) prove a counterfactual dataset can be assembled, (b) confirm the historical sample size is large enough to detect the edge the trader thinks he has, and (c) lock the trader's rules so the study isn't measuring a moving target.

**The most likely failure mode of this project is not bad modeling. It is building a confident-looking rubric on top of 120 trades with a moving rule set and no counterfactual.** This plan is structured to kill that failure mode before it happens.

The research is gated by three explicit go / no-go decisions:

1. **Counterfactual capture works** (milestone 2). Without it, the rubric is unfalsifiable.
2. **Univariate analysis finds at least three features that survive FDR correction** (milestone 4). If nothing survives, expand data or accept that the edge is in execution, not selection.
3. **Walk-forward OOS expectancy is at least 50% of in-sample** (milestone 7). If it's not, the rubric is overfit and ships as v0 only with a confidence band that says so.

Expected research duration before first production code line: **~10 weeks**.

---

## 1. Critique of the Spec

The CLAUDE.md spec is structurally fine for an engineering brief and structurally weak for a research design. Specific issues, by section:

### 1.1 *Feature Categories* is incomplete for futures microstructure

The feature list (per the brief: price action, volume, indicators, structure) is what you'd write if you came from equities swing trading. For intraday futures, the following are not nice-to-haves, they are first-order:

- **Order flow imbalance** at signal bar and at entry bar (bid-initiated vs ask-initiated volume). This is the dominant published intraday signal in futures. See Cont, Kukanov, Stoikov (2014), *The Price Impact of Order Book Events*.
- **Cumulative delta divergence** between signal and prior swing.
- **Bid/ask spread regime** (median spread in last N minutes vs session median). Spread blowouts are the canonical liquidity-vacuum tell.
- **Time-of-day liquidity bucket**, not just "time of day." Open auction (first 15 min), morning trend (15 to 90 min), midday chop (90 to 240 min), close auction, overnight. Each has a different base rate. This is non-negotiable.
- **Realized vol vs implied vol** for the underlying (where applicable, e.g., ES vs VIX term structure). Captures regime in one number.
- **Volume profile context**: distance from session VWAP, distance from prior day's VPOC, location within developing value area. These are commonly priced and commonly tradable.
- **Session-relative ATR rank**, not raw ATR. ATR of 12 means nothing without context.
- **Auction state**: balancing vs trending vs initiative. Even a crude proxy (range vs ATR, drive vs rotation) is better than nothing.

If these are not in CLAUDE.md's feature list, the rubric will be modeling chart patterns in isolation from the tape, and the trader's actual edge (which is probably partially tape-reading) will be invisible to the model.

### 1.2 *Expectancy by grade bucket* is necessary but insufficient

The spec proposes expectancy per A+/A/B as the success metric. That is the right outcome metric but it's a weak diagnostic for rubric quality:

- **Information Coefficient** (rank correlation between rubric score and realized R) is what tells us whether the rubric *orders* trades correctly. A rubric can have flat expectancy across buckets and still rank-order well, or vice versa.
- **Hit-rate vs payoff decomposition** is necessary because a 50% hit / 2R rubric and a 70% hit / 1R rubric have the same expectancy and entirely different psychological and execution profiles.
- **Sharpe per grade** (annualized, accounting for trade frequency per grade) matters because A+ being rare is different from A+ being frequent.
- **Brier score** if we ever frame grades as probability of success. The brief doesn't go there, but it should be considered.

**Decision:** primary metric remains expectancy by bucket, but the rubric must additionally clear thresholds on IC and Sharpe per grade before promotion. See section 5.

### 1.3 *Sample size thresholds are wishful*

The brief specifies "30 trades per bucket" and "70/30 walk-forward." Both are too generous to the project.

**Power calculation for the bucket comparison.** Suppose A+ true expectancy is +0.6R and B true expectancy is +0.1R, with a within-bucket R-multiple SD of ~1.5R (typical for intraday futures discretionary). For a two-sample one-sided test at alpha=0.05 with power 0.80, the required N per bucket is:

```
n ≈ 2 × (z_{1-α} + z_{1-β})² × σ² / Δ²
n ≈ 2 × (1.645 + 0.842)² × 1.5² / 0.5²
n ≈ 2 × 6.19 × 2.25 / 0.25
n ≈ 111 per bucket
```

So roughly **100 to 150 trades per bucket** to reliably detect a 0.5R lift. Thirty per bucket detects only a ~1.0R lift, which the trader will already see by eye. The spec's threshold doesn't pass the smell test for the effect sizes that matter.

**Walk-forward.** A single 70/30 split is fragile, especially when the 30% may include a regime the 70% didn't. Use **expanding-window walk-forward with embargo** (Lopez de Prado, 2018, *Advances in Financial Machine Learning*, ch. 7). Embargo of ~5 trading days prevents leakage from same-day fills and overlapping MFE/MAE windows.

**Decision:** bucket threshold rises to 100 trades per bucket as the analytical floor; walk-forward becomes expanding window with monthly refit and 5-day embargo. If we don't have the sample, we don't build the rubric, we collect more data.

### 1.4 Survivorship bias is unaddressed

The trader's logbook is a survivor cohort. He already filtered the setups he was willing to take, based on discretionary judgment that we are now trying to systematize. **Building a grading rubric on taken trades alone teaches the model the trader's existing filter, not the underlying edge.** The model will look like it grades well because it's predicting what the trader will take, not what will work.

This is the single most common quant mistake on discretionary trader data and it is not mentioned in the brief.

**Decision:** counterfactual capture (untaken-but-fired setups) is a research prerequisite, not an extension. See milestone 2 and section 3.2.

### 1.5 Selection bias compounds the survivorship problem

Adjacent to 1.4: even among taken trades, the entry timing within a fired setup is a choice. Two traders looking at the same Pine alert can enter at different prices. The MFE/MAE distribution is therefore conditional on the trader's entry-timing skill, not just the setup. We can't separate "good setup" from "good entry within setup" without timestamp-accurate signal-fire data.

**Decision:** Pine alerts must log signal-bar OHLC and timestamp at *fire time*, not at trader-acknowledgment time. Entry-vs-signal slippage becomes a feature.

### 1.6 Multiple comparisons risk is unacknowledged

With 25+ candidate features and multiple thresholds tested per feature, the family-wise error rate without correction is approaching certainty of at least one false positive. See Harvey, Liu, Zhu (2016), *and the Cross-Section of Expected Returns*, Review of Financial Studies, for the canonical treatment in finance. Also Bailey & Lopez de Prado (2014), *The Deflated Sharpe Ratio*.

**Decision:** all univariate tests carry **Benjamini-Hochberg FDR correction** at q=0.10. Final Sharpe metrics are reported as **deflated Sharpe** accounting for the number of trials. The brief's silence on this would have produced a rubric stuffed with noise features.

### 1.7 *Anti-Goals* should explicitly anti-goal "modeling the trader"

If CLAUDE.md's anti-goals don't already include "*do not build a model that predicts the trader's filter rather than the market's behavior*," it should. This is the failure mode that the counterfactual dataset is designed to prevent, and it should be named.

---

## 2. Research Questions

Eight primary research questions, ranked by expected information value (highest first). Each is falsifiable. Each has a sample-size requirement.

### RQ1 (highest IV). Is there a counterfactual edge at all?

**Hypothesis:** Among setups that fired, *taken* setups have a meaningfully better realized R-multiple distribution than *untaken-but-fired* setups would have produced if simulated to default stop/target.
**Null:** No distributional difference between taken and untaken-simulated.
**Test:** Two-sample Kolmogorov-Smirnov on simulated R-multiple distribution. Mann-Whitney for median lift.
**Sample required:** ~150 taken, ~150 untaken-fired, post-embargo.
**Falsified if:** KS p > 0.10 after FDR, and median lift < 0.1R. **This kills the project**: it means the discretionary filter is noise and there's nothing to systematize.

### RQ2. Does any composite score rank-order outcomes?

**Hypothesis:** A linear combination of N≤8 pre-entry features produces IC ≥ 0.10 (Spearman) against realized R-multiple, statistically significant under permutation test.
**Null:** IC indistinguishable from zero.
**Test:** Permutation test on IC (10,000 shuffles), with FDR correction across feature-combination trials.
**Sample required:** ~250 trades for IC stability (Coqueret & Guida, 2020, *Machine Learning for Factor Investing*).
**Falsified if:** No combination clears IC=0.05 OOS. Rubric concept itself is not supported.

### RQ3. Are A+ and B *distributionally* different, not just on the mean?

**Hypothesis:** Top-tercile-scored trades have right-shifted *and* tighter-tailed R-multiple distributions than bottom tercile (i.e., better expectancy and lower variance).
**Null:** Same distribution shape.
**Test:** Mann-Whitney for location + Levene's for scale + KS for shape.
**Sample required:** ~100 per tercile.
**Falsified if:** Shape tests show no difference. Rubric grades expectancy but not consistency, and sizing logic downstream must reflect that.

### RQ4. Is the edge stable across regimes?

**Hypothesis:** IC of the rubric score in high-vol regime is within 30% of IC in low-vol regime (using realized vol percentile as regime split).
**Null:** Regime-specific IC differs materially; one regime is masking the other.
**Test:** Bootstrapped CI on IC per regime; non-overlap of CIs flags regime dependence.
**Sample required:** ~75 trades per regime bucket, post-embargo.
**Falsified if:** Edge is single-regime. Rubric must be regime-conditional or scope must be narrowed.

### RQ5. Which features are robustly important across folds?

**Hypothesis:** A subset of features appears in the top-5 importance ranking in ≥80% of walk-forward folds.
**Null:** Importance ranking is unstable; no feature is consistently informative.
**Test:** SHAP-based permutation importance per fold; rank-stability via Kendall's W across folds.
**Sample required:** At least 6 folds, so ~250 trades.
**Falsified if:** No feature is consistently top-ranked. The model is fitting fold-specific noise.

### RQ6. Does conditioning on time-of-day bucket improve all of the above?

**Hypothesis:** Same RQ2 / RQ3 analyses, stratified by time-of-day bucket, yield uniformly higher within-bucket IC than the pooled estimate.
**Null:** Pooled IC ≥ within-bucket IC for at least half of buckets.
**Test:** Compare bootstrapped IC distributions.
**Sample required:** ~50 trades per bucket × 4 buckets = 200 minimum.
**Falsified if:** Time-of-day is irrelevant (unlikely but worth verifying).

### RQ7. Is MFE/MAE structure separable from setup quality?

**Hypothesis:** MFE distribution can be predicted from pre-entry features alone (i.e., the setup tells you the favorable excursion). If yes, stop placement is a function of grade. If no, stops are independent of grade.
**Null:** Pre-entry features don't predict MFE distribution.
**Test:** Quantile regression of MFE on pre-entry feature vector.
**Sample required:** ~250 trades.
**Falsified if:** No predictive power. Implication: grading is for entry filtering, not stop calibration.

### RQ8. Does OOS expectancy decay stay bounded?

**Hypothesis:** Realized OOS expectancy is at least 50% of in-sample expectancy for the same grade bucket.
**Null:** OOS expectancy < 50% of IS, i.e., severe overfit.
**Test:** Bootstrap CI on the IS/OOS ratio per bucket.
**Sample required:** Walk-forward design provides this implicitly; need ≥3 OOS windows.
**Falsified if:** Decay > 50%. Rubric ships with caveat or is rebuilt with stricter regularization.

---

## 3. Dataset Requirements

### 3.1 Minimum viable sample

| Dimension | Floor | Target | Rationale |
|---|---|---|---|
| Taken trades | 250 | 400 | RQ2 stability, RQ8 OOS folds |
| Untaken-but-fired setups | 150 | 300 | RQ1 counterfactual power |
| Calendar window | 6 months | 12 months | Cross-regime coverage |
| Instruments | 1 (ES or NQ) | 2 (add micros) | Avoid cross-instrument confounds; pool only if normalized |
| Distinct time-of-day buckets covered | 4 | 4 | RQ6 stratification |
| Trading days with at least one fire | 60 | 120 | Day-effect averaging |

**If the floor isn't met, the project pauses at milestone 1 for more data collection.** This is not negotiable. A small-sample rubric will look great in-sample and fail in production. We do not ship a rubric we know to be underpowered.

### 3.2 The counterfactual dataset

This is the hard one. The trader has not been logging setups he didn't take. Without that log, every analysis below is contaminated.

**Two-track capture plan:**

1. **Forward capture (immediate).** Pine Script alert on the existing setup logic fires to a structured log (CSV via webhook, or TradingView alert export). Every fire is logged, including ones the trader didn't act on. This starts day 1 of milestone 2.
2. **Backfill (best effort).** NinjaTrader 8 Market Replay reproduces the setup signal on historical bars for the in-sample period. The Pine condition is approximated in NinjaScript or post-processed against historical OHLCV. Acceptance threshold: ≥70% of historical fires recoverable. If below, the IS / OOS structure changes (we use forward capture only).

**What we log per fire:**

- Timestamp (signal bar close)
- Instrument
- All pre-entry features (price, volume, microstructure, regime context)
- Trader action: taken / passed / missed (the last category matters for behavioral analysis)
- If taken: entry timestamp + slippage; if passed: a free-text reason (becomes a categorical feature)

### 3.3 Contamination risks in the trader's existing record

Material risks specific to a one-year-into-it discretionary trader:

- **Rule changes mid-period.** Trader almost certainly tweaked setup definitions, stop logic, target logic. Need a chronological *Rule Ledger* from the trader (see section 8, question 3).
- **Instrument switches.** ES vs MES vs NQ vs MNQ. R-multiples normalize this but tick-by-tick microstructure does not. We may need per-instrument analysis at minimum.
- **Sizing changes.** If sizing changed mid-period, raw P&L is non-comparable. R-multiple is the unit of analysis, not dollars.
- **Session changes.** RTH vs ETH inclusion. Treat as separate samples until proven otherwise.
- **Broker / platform changes.** Slippage and fill quality differ. Verify single-broker assumption.
- **Tradezella tagging drift.** Tags may have been added retroactively. Treat tag history with skepticism; prefer tags applied within 24h of close.

### 3.4 Data quality checks (gate to enter milestone 3)

- Timestamp reconciliation between NT8 fills, Tradezella entries, and TV alert log. Threshold: 95% match within 5 seconds.
- Slippage distribution sanity (no entries more than 3 ticks improved vs signal close).
- Missing-feature audit per fire (no more than 5% missing on any feature).
- Time zone normalization to exchange time (CT for CME products).
- Holiday and half-day handling: holiday sessions excluded from primary analysis, kept for stress test.
- Duplicate detection (same setup fires twice within N bars: keep first or collapse, decide explicitly).

---

## 4. Feature Taxonomy

### 4.1 Refined feature list

Beyond what CLAUDE.md proposes, the following must be included or explicitly justified out:

**Price action context (pre-entry only)**
- Distance from session VWAP (z-scored within session)
- Distance from prior day VPOC
- Position relative to developing value area (above / inside / below)
- Distance from N-period EMA (20, 50)
- Recent swing structure (HH/HL/LH/LL coded)

**Volume / order flow**
- Cumulative delta at signal bar
- Delta divergence vs price over last K bars
- Relative volume (signal bar volume / N-bar rolling median)
- Order flow imbalance over last M ticks (where tick data is available)
- Bid/ask spread regime (current vs session median)

**Volatility / regime**
- Realized vol percentile (rolling 20-day, instrument-conditional)
- ATR rank within session
- Range expansion vs prior day range
- VIX level and VIX term-structure slope (for ES/NQ)

**Time / calendar**
- Time-of-day bucket (open, morning, midday, close, ETH)
- Day of week
- Days since FOMC / CPI / NFP (event proximity)
- Half-day flag

**Setup-specific**
- Bars since prior signal
- Setup chain count (Nth consecutive signal in same direction)
- Distance to nearest higher-timeframe level

**Behavioral / execution (separate cohort, not used in pre-entry grading but used in RQ7)**
- Signal-to-entry slippage (ticks)
- Discretionary modifiers logged at entry (manual override fields)

### 4.2 Collinearity handling

Likely high-collinearity pairs to address before multivariate work:

| Pair | Why collinear | Resolution |
|---|---|---|
| Distance from VWAP, distance from EMA20 | Both track trend center | Keep one per regime: VWAP in range, EMA in trend |
| ATR, realized vol percentile | Both vol proxies | Keep realized vol percentile (regime-aware) |
| Volume, bid/ask spread | Both liquidity proxies | Keep both with collinearity-robust modeling (L1, tree-based) |
| Cumulative delta, OFI | Both order flow | Keep both; OFI is more granular if tick data available |

**Approach:** Pearson and Spearman correlation matrices computed in milestone 3. Pairs with |r| > 0.85 flagged for one-at-a-time inclusion in multivariate stage.

### 4.3 Transformations

- **Continuous features:** percentile-rank within instrument and within time-of-day bucket. Z-scoring is regime-fragile and not the default.
- **Categorical features:** target-encoding only inside walk-forward folds to prevent leakage. Otherwise one-hot.
- **Missing values:** explicit missing-indicator column rather than imputation, for pre-entry features where missingness may itself be informative (e.g., VIX missing on overnight session).
- **Outliers:** Winsorize at 1st / 99th percentile per instrument. Do not drop, do not impute to median.

### 4.4 Pre-entry only constraint

**Hard rule:** every feature used in grading must be computable strictly from data available at signal-bar close. No MFE-, MAE-, or exit-derived feature is allowed in the rubric input. Those features are *outcomes* and including them is the most common form of leakage on this kind of dataset.

A whitelist of allowed features will be locked at end of milestone 3 and reviewed by the trader.

---

## 5. Analytical Methodology

### 5.1 Order of analysis

1. **Data quality gate** (milestone 1)
2. **Descriptive** (milestone 3): per-feature distributions, calendar coverage, regime mix
3. **Univariate** (milestone 4): each feature tested for marginal lift, FDR-corrected
4. **Bivariate / interaction** (milestone 5): pairwise interactions on shortlist
5. **Multivariate** (milestone 6): gradient-boosted decision tree with SHAP for ranking, then **linearized** for rubric construction
6. **Walk-forward validation** (milestone 7)

The discipline of running univariate before multivariate is non-negotiable. It produces the trader-readable scoreboard that the rubric will be defended against if it ever underperforms in production.

### 5.2 Specific statistical tests

| Question | Test | Why |
|---|---|---|
| Median R-multiple difference between buckets | Mann-Whitney U | R-multiple distributions are heavy-tailed; t-test assumes normality we don't have |
| Distributional shape | Kolmogorov-Smirnov | Detects scale + location shifts |
| Variance equality | Levene's test | More robust than F-test under non-normality |
| Confidence interval on expectancy | Stationary bootstrap (Politis & Romano, 1994) | Preserves time-series dependence |
| Feature importance significance | Permutation test (10,000 reps) | Distribution-free; controls for sample-specific quirks |
| Family-wise error control | Benjamini-Hochberg FDR at q=0.10 | Less conservative than Bonferroni, appropriate for correlated features |
| Sharpe quality | Deflated Sharpe (Bailey & Lopez de Prado, 2014) | Accounts for trial count |
| Rank correlation | Spearman with permutation p-value | Robust to outliers in R-multiple |

### 5.3 Interaction discovery

Three-pass approach:

1. **Theory-led:** trader nominates 5 to 10 interaction pairs he believes matter (e.g., trend strength × time-of-day). Tested explicitly.
2. **SHAP interaction values:** gradient-boosted tree fit to R-multiple as continuous target. Top SHAP interactions surfaced.
3. **Manual validation:** any interaction surviving SHAP screen is plotted as a 2D conditional expectancy heatmap and inspected for sample sufficiency per cell (min 15 trades per cell, else collapse).

Interactions discovered in step 2 alone are treated as suspect until they survive walk-forward.

### 5.4 From analytical output to rubric weights

This is the step most easily fudged. Concrete translation:

1. Define grading target: top-tercile vs bottom-tercile of realized R-multiple (in-sample only).
2. Fit **L1-regularized logistic regression** (sparse coefficients) on the standardized pre-entry feature shortlist, target = is-top-tercile.
3. Take coefficients, sort by magnitude, retain features with non-zero coefficients.
4. **Convert to integer weights** via greedy rounding: round normalized coefficients to nearest integer in {-3, -2, -1, 0, 1, 2, 3}.
5. Verify that integer-weight rubric preserves rank correlation with raw-coefficient rubric (Spearman ≥ 0.90 in-sample). If not, expand weight range to ±5 and retry.
6. Map composite score to A+/A/B buckets via in-sample quantile thresholds (e.g., top 25% = A+, next 35% = A, rest = B).

Example: if "distance-to-VWAP-zscore" has standardized L1 coefficient of 0.42 and "ATR-rank" has 0.13, the integer weights become +3 and +1 respectively.

The trader can read the integer-weight rubric. He cannot read SHAP values. **Interpretability is a non-functional requirement.**

### 5.5 Walk-forward protocol

- **Window type:** expanding (use all available history up to t, refit at month boundaries).
- **Embargo:** 5 trading days between train cutoff and test start. Prevents same-week MFE/MAE windows from leaking.
- **Refit cadence:** monthly.
- **Fold count:** at least 6 OOS months. If sample doesn't support this, reduce refit cadence (e.g., bi-monthly).
- **Reporting:** per-fold IC, per-fold expectancy by bucket, per-fold Sharpe; aggregated with bootstrapped CI.

### 5.6 Promotion criteria (v_n to v_n+1)

| Criterion | Threshold |
|---|---|
| OOS IC | ≥ 0.08 (deflated for trial count) |
| OOS expectancy by top bucket | ≥ 50% of in-sample value, lower-bound bootstrap |
| OOS Sharpe by top bucket | Within 30% of in-sample |
| Feature stability (RQ5) | ≥ 80% of top-5 features stable across folds |
| Interpretability | Integer-weight rubric, ≤ 8 features |

A version that fails any of these does not get promoted. Period.

---

## 6. Risk Register

| Risk | Probability | Impact | Detection | Mitigation |
|---|---|---|---|---|
| **Overfitting** | High | High | OOS expectancy < 50% IS; OOS IC < 0.05 | Embargoed expanding-window CV; L1 regularization; integer-weight constraint; deflated Sharpe |
| **Look-ahead bias** | Medium | High | Implausibly high IS metrics; features available only post-entry | Strict point-in-time feature computation; whitelist of allowed pre-entry features; lag-check unit test in milestone 3 |
| **Regime change** | High | Medium | KS test on feature distribution between IS and OOS; OOS Sharpe degradation isolated to one period | Regime-conditional analysis; report by sub-period; flag rubric as regime-aware where needed |
| **Trader behavior drift mid-study** | Medium | High | Pre- vs post-study entry-timing distribution differs; setup-fire counts change | Lock rules during measurement window; require trader to log any rule change; rerun analysis with regime indicator |
| **Feature leakage (post-entry into pre-entry)** | High | High | Feature has implausibly high IC (>0.30); feature definition involves data after signal bar close | Hard whitelist; manual audit of every feature definition; automated test that signal-bar timestamp is max input timestamp |
| **Multiple comparisons inflating apparent edge** | High | Medium | Many marginal p-values clustered just under 0.05; deflated Sharpe << raw Sharpe | BH-FDR at q=0.10; deflated Sharpe; pre-registration of which features are primary vs exploratory |
| **Small sample** | High | High | Bootstrap CIs wider than effect size; feature rank instability | Pause at milestone 1 floor; expand to micros if needed; reduce candidate features before multivariate |
| **Counterfactual capture failure** | Medium | Critical | <70% backfill recovery; forward log has gaps | Forward capture is gating; if backfill fails, OOS window is the only valid test set |
| **Modeling the trader instead of the market** | High | Critical | Rubric scores correlate with "trader took it" more than with realized R | Train against realized R, not against "taken" label; counterfactual dataset provides the negative class |
| **Tradezella / NT8 / TV time mismatch** | Medium | Medium | Failed reconciliation at milestone 1 | Time-zone normalization; tolerance window; fall back to NT8 as ground truth |
| **Survivorship by trader filter** | High | High | Taken-only analysis shows artificially high baseline expectancy | Counterfactual dataset (milestone 2) is gating |
| **Trader changes rules in response to early findings** | Medium | High | Mid-study fire-rate shift | Lock rule-set agreement before milestone 3; any change restarts the IS window |

---

## 7. Sequenced Research Milestones

These supersede the *Build Order* section of CLAUDE.md until milestone 8 is reached.

### Milestone overview

| # | Milestone | Objective | Inputs | Methods | Deliverable | Decision Gate | Time |
|---|---|---|---|---|---|---|---|
| 0 | Sign-off | Approve this plan | This doc, trader review | Discussion | Signed `RESEARCH_PLAN.md`; trader-answered open questions | Go: trader signs and answers section 8 | 3 days |
| 1 | Data audit | Validate inputs and reconcile sources | NT8 export, Tradezella export, TV alert log (if any) | Schema check, timestamp recon, slippage sanity | Data quality memo with green/yellow/red flags | Go: ≥95% reconciled; floor sample met. No-go: pause for data collection | 1 week |
| 2 | Counterfactual capture | Build untaken-but-fired log | Pine script alert, NT8 Market Replay backfill | Forward capture + backfill | Counterfactual dataset (CSV); coverage report | Go: ≥70% backfill OR ≥150 forward fires within capture window. No-go: project paused | 2 weeks (forward capture starts immediately and runs in parallel) |
| 3 | Descriptive | Characterize data | All trade + counterfactual data | Univariate distributions, calendar coverage, feature collinearity matrix, regime mix | Chart pack + summary memo; finalized feature whitelist | None (informational), but produces feature whitelist sign-off | 3 days |
| 4 | Univariate | Identify candidate features | Trade data + feature whitelist | Mann-Whitney + FDR; per-feature IC; bucketed expectancy plots | Feature scoreboard | Go: ≥3 features survive FDR at q=0.10. No-go: expand data or accept null result | 1 week |
| 5 | Bivariate / interactions | Find feature interactions | Top features from milestone 4 | SHAP interactions, theory-led pairs, conditional heatmaps | Interaction memo | None (informational) | 1 week |
| 6 | Rubric v0 | Construct candidate rubric | Surviving features + interactions | L1 logit, integer weight conversion, in-sample bucket thresholds | Rubric spec v0 (markdown table of weights, score-to-grade map) | None (informational) | 1 week |
| 7 | Walk-forward validation | Test rubric OOS | Rubric v0 + full dataset | Expanding-window CV with embargo, per-fold metrics, deflated Sharpe | Validation report with promotion-criteria table | Go: meets all criteria in section 5.6. No-go: revise and re-validate, or ship as observation-only | 2 weeks |
| 8 | Hand-off to engineering | Promote rubric to production | All artifacts | Eng review of rubric spec, feature computation requirements | Production rubric spec, list of features to wire into the CLAUDE.md pipeline | Go: trader + eng sign-off | 1 week |

**Total time before first production code line: ~10 calendar weeks**, of which ~7 are analytical effort (the rest is data audit + counterfactual capture which is partially calendar-bound).

### What "go / no-go" actually means at each gate

- **Milestone 1 no-go:** project pauses; trader keeps trading; data collection continues; we revisit in 60 days.
- **Milestone 2 no-go:** entire project pauses. We do not build a rubric without a counterfactual. Trader can choose to commit to forward logging for 90 days and we restart milestone 2 then.
- **Milestone 4 no-go:** two options. (a) Expand data and retry. (b) Accept that the edge isn't in selection and pivot the project to *execution analytics* (sizing, exit, slippage) rather than setup grading. This is a real outcome and should not be treated as failure.
- **Milestone 7 no-go:** rubric ships as *observation only* (display grade in journal, do not size on it) until the next data collection cycle. The trader does not adjust behavior based on a rubric that didn't validate.

---

## 8. Open Questions to Escalate to Bryce

These must be answered before milestone 1 begins. They are research-design questions, distinct from the engineering-scope questions in CLAUDE.md.

1. **Stop placement methodology.** Fixed ticks? Fixed R? Structural (last swing)? Volatility-based (1× ATR)? Has this changed in the period under study?

2. **Sizing methodology.** Fixed contracts per setup? R-adjusted per setup? Conviction-weighted (and if so, by what)? Has this changed?

3. **Rule ledger.** Provide a chronological log of every rule change in the past 12 months. If undocumented, reconstruct from memory with best-effort dates. Without this, regime analysis cannot separate market regime from rule regime.

4. **Self-described edge.** In two paragraphs: what do you believe your edge is? What kind of setup do you think you grade well by eye? We will test that belief explicitly. If the model disagrees with you, we want to know.

5. **Counterfactual logging history.** Have you ever logged setups you didn't take? If yes, where and from when? If no, will you commit to a 90-day forward log window?

6. **Instrument scope.** ES only? ES + NQ? Add micros? Day-only or include ETH? This determines pooling and normalization.

7. **Setup definition objectivity.** Is the setup fully definable in Pine Script, or is there a discretionary "feel" component? If the latter, what does the feel detect and can we proxy it?

8. **MFE/MAE measurement convention.** Measured from signal bar OHLC or from entry fill? Stop hit measured at stop touch or stop fill?

9. **Exit logic.** All-out at target? Scale-out? Trail? Discretionary exit on context change? Does the exit rule depend on the entry grade currently, or is it independent?

10. **Tradezella tag history.** Were grade tags applied at trade close or added later in review? Date stamps available?

11. **Sample period boundary.** When does "the period under study" start and end? Is there an obvious cutover (instrument change, broker change, strategy change) that should split it?

12. **Risk tolerance for OOS underperformance.** If OOS expectancy is 50% of IS, do you still want to use the rubric? At 30%? This determines our promotion threshold tightness.

13. **Definition of "missed" vs "passed."** A setup you saw and rejected is different from one you didn't see fire. Both should be logged but coded differently. Does the current journal distinguish?

14. **Cost assumptions.** Per-trade commission + fee + assumed slippage in ticks. Used in expectancy calculations.

---

## Appendix A. References

- Bailey, D. H., & Lopez de Prado, M. (2014). *The Deflated Sharpe Ratio: Correcting for Selection Bias, Backtest Overfitting, and Non-Normality.* Journal of Portfolio Management.
- Cont, R., Kukanov, A., & Stoikov, S. (2014). *The Price Impact of Order Book Events.* Journal of Financial Econometrics.
- Coqueret, G., & Guida, T. (2020). *Machine Learning for Factor Investing.* CRC Press.
- Harvey, C. R., Liu, Y., & Zhu, H. (2016). *... and the Cross-Section of Expected Returns.* Review of Financial Studies.
- Lopez de Prado, M. (2018). *Advances in Financial Machine Learning.* Wiley. (See ch. 7 on cross-validation in finance.)
- Politis, D. N., & Romano, J. P. (1994). *The Stationary Bootstrap.* Journal of the American Statistical Association.

## Appendix B. Definition of Done for This Plan

A quant lead reading this document cold should be able to:

1. Predict what the first three weeks of analysis will produce: a data quality memo, a counterfactual dataset (or a no-go), and a finalized feature whitelist.
2. Know what decision will be made at each gate: see section 7 *Decision Gate* column.
3. Recognize when the study is being run incorrectly: any deviation from the test specifications in section 5.2, any post-entry feature in the whitelist, any analysis that proceeds past a no-go gate without explicit re-scoping.
4. Estimate whether the proposed sample size can answer the proposed questions: power calculation in section 1.3, sample requirements per RQ in section 2.

If any of those four cannot be done from this document, return it for revision.
