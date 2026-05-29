# PLAN_REVIEW.md

**Purpose**: Quant-analyst review of the planning package (`CLAUDE.md`, `RESEARCH_PLAN.md`, `DEVELOPMENT_PLAN.md`, `ASSUMPTIONS.md`, `PROJECT_KICKOFF.md`) and the prioritized improvements adopted from it.

**Status**: Proposed. Inline patches in the companion docs reference this file by section (PR-REVIEW-Pn).

**Reviewer framing**: This is an unusually self-aware package. The 20-hazard catalog (DEVELOPMENT_PLAN §2.8), the FDR / deflated-Sharpe / PBO stack, the counterfactual-dataset insistence, and the "NO-GO is a success" framing are correct and rare. The review does not talk the plan out of obvious mistakes; the obvious mistakes are already caught. The central risk is different: **the rigor is disproportionate to the sample size, and the plan has not internalized that its own power math makes the headline deliverable (a walk-forward-validated rubric) almost certainly unbuildable on year-one data.** The fixes below mostly make the plan act on its own conclusion sooner and more consistently.

---

## 0. Highest-leverage finding

The package is **internally contradictory on sample size**, and the contradiction is load-bearing:

- RESEARCH_PLAN §1.3 / CLAUDE.md "Sample-size floors": detecting IC = 0.10 after FDR needs **N ≈ 800**; a top-vs-bottom-quintile 0.3R lift needs **~1,250–2,000 total trades**.
- ASSUMPTIONS F2: green-lights the build at **200 trades** (soft pass 100–200).
- Weight-derivation table (RESEARCH_PLAN §5.5 / CLAUDE.md): assigns rubric weight to ICs as low as **0.05–0.10**, which §1.3 itself says require N ≈ 800 to detect.

So the go-bar is set 4–10× below what the plan's own power analysis says is detectable. QH19 catches the walk-forward-window version of this at Phase 6 (week 9). The deeper version — that even the M4 univariate stage is underpowered for the IC thresholds in the weight table — is **not** flagged as a gate. The single highest-leverage change is to move the power verdict to **M0 (week 1)** and re-scope the MVP around what underpowered data can actually deliver.

Note this also reconciles a framing split already latent in the docs: CLAUDE.md frames the project as *"deep post-hoc analysis... a rubric is a possible later output, not the MVP centerpiece,"* while RESEARCH_PLAN / DEVELOPMENT_PLAN still march to a rubric at M6/M7. Re-scoping around the analysis + forward-capture system is not a reversal; it makes the milestone structure match CLAUDE.md's own stated framing.

---

## 1. Methodology hazards still missing or under-specified (beyond the 20)

1. **Sample-size contradiction not gated** (see §0). Fix: print minimum detectable effect size (MDES) at the actual post-M0 trade count in the M0 memo headline. If MDES > ~0.5R (near-certain at 200–400 trades), declare the rubric a stretch deliverable in week 1, not week 9.

2. **"8 features × 30 trades = 240" is folk-statistics the plan elsewhere bans.** §1.3 explicitly debunks the 30-trades-per-factor heuristic; §5.5 and CLAUDE.md rubric rules then use it to size the feature cap. The 240 floor has no power basis. Fix: justify the cap from events-per-variable / shrinkage logic, or state plainly that 8 is a pragmatic ceiling unrelated to detectability.

3. **Continuous-contract construction is undefined and one-way.** §3.3 / CLAUDE.md say "back-adjusted or ratio-adjusted" without choosing. Back-adjustment distorts historical absolute price, which silently breaks every ATR-normalized distance feature and every percentage forward-return — the dominant feature family for MNQ/NQ. Fix: **ADR-0011**; decide per-feature (session-anchored features such as VWAP / cum-delta / rvol never cross a roll and want un-adjusted front-month bars; only multi-day lookbacks need a continuous series).

4. **NQ vs MNQ pooling treated too lightly for order-flow features.** QH9 adds an overall pooling test, but cum-delta and large-lot-ratio distributions differ by participant composition, not just scale. Fix: make the QH9 test a hard gate for order-flow features specifically; pool price/temporal/vol if it passes, keep order-flow per-instrument regardless.

5. **Counterfactual slippage is optimistic and untested.** QH1 models skipped-setup fills at bar close ± half-spread. The setups Allie skipped may be exactly the thin/fast ones with worse fills — biasing the counterfactual edge upward. Fix: report counterfactual edge at half-spread, one-spread, and two-spread cost. If promotion criterion 6 only holds at half-spread, it is not robust.

6. **No backtest-vs-live reconciliation.** Phase 7's gate ("OOS expectancy by grade matches walk-forward within CI") passes trivially when both CIs are huge (~5 windows, 3 months forward). Fix: pre-register expected live expectancy + CI before Phase 7; define the shortfall magnitude (not "CI overlap") that triggers retrain.

7. **Behavioral feedback loop under-mitigated.** R5 blind grading conflicts with the real-time decision-support job (§1.2). Once Allie acts on live grades, Phase 8 refit data is endogenous to the rubric. Fix: keep a permanently-blind control stream so refits always have an exogenous comparison.

8. **Genuinely missing:**
   - **No project-lifetime multiple-testing ledger.** FDR is applied within one 25-feature sweep; nothing tracks total tests across milestones and across quarterly refits. Over two years of refits the project-level false-discovery rate is uncontrolled. This is the biggest currently-unmitigated statistical hazard.
   - **Label-horizon multiplicity.** 4 horizons × 25 features = 100 tests; FDR is applied across 25, not 100. Either pre-commit the horizon (CLAUDE.md already names 30-min as primary — make that binding) or correct across feature×horizon.
   - **No effective-sample-size accounting for slow features.** VIX level, ATR percentile, HTF trend are near-constant across same-day trades; their effective N is distinct regime-days (~50–100), not trades (~300). Power calcs assume per-trade independence these features lack. Fix: report effective N alongside trade-N for high-within-day-autocorrelation features.

---

## 2. Research-design rigor

The validation framework is sound in form (rolling-origin walk-forward + embargo, purged CPCV fallback, deflated Sharpe, PBO, stationary bootstrap, BH-FDR, correctly cited). Where it falls short:

1. **The 1-month walk-forward window is infeasible.** ~25 trades/month / 3 buckets ≈ 5 A+ trades/window → bucket-mean CI ≈ ±1R. Promotion criteria (A+ minus A ≥ 0.3R, CI excluding zero) cannot be met by construction — not for lack of edge, but for lack of measurable resolution. Fix: use **CPCV from the start** (not as fallback) and report trades-per-fold prominently. Folds < 30 trades are descriptive, not inferential.

2. **No truly untouched final hold-out.** R1 mentions "hold out a final untouched 3 months"; the build (Phases 5–6) never operationalizes it and §5.7 never references it. Everything flows through the walk-forward the analyst iterates on. Fix: carve a locked most-recent-N-trades hold-out, touched exactly once after the rubric is frozen.

3. **"Edge" is defined four ways without one decision rule** (Q1 net R, Q2 IC, Q3 monotonic OOS, Q4 conviction IC, Q5 regime-conditional IC). A rubric can pass Q2 and fail Q3 — the most likely outcome at this N. Fix: canonical go/no-go = **Q3 (monotonic OOS) + PBO < 0.5**. Q1/Q2 are upstream gates; Q4/Q5 are diagnostics.

4. **Deflated Sharpe is vestigial.** DSR needs a maintained trial count to deflate against; the plan keeps none, and the primary metrics are IC / bucket expectancy, not Sharpe. PBO (QH20) is the right tool and is chosen. Fix: either drop DSR or define and maintain the trial count it consumes (ties to the test ledger).

5. **One negative control is too few.** QH17 injects a single control — one Bernoulli trial against a leak. Fix: inject 3–5 synthetic random features; if more than the FDR-expected number clear, the pipeline leaks.

---

## 3. Prioritized recommendations (adopted)

Ranked by leverage. Each maps to inline patches tagged `PR-REVIEW-Pn` in the companion docs.

- **P0 — Move the power/MDES verdict to M0 and re-scope the MVP around it.** If MDES > ~0.4–0.5R, make the forward-capture + counterfactual system the primary MVP and the rubric a stretch deliverable. Saves 8 weeks and removes the week-9 temptation to lower the bar.
- **P1 — Resolve the 200-trade vs ~800-trade contradiction** before sunk cost makes it resolve itself downward. Pick one coherent standard.
- **P2 — Locked final hold-out + frozen, hashed analysis pre-registration** (`analysis_preregistration.yaml`, checked by a fitness function). The validation set is the deliverable's credibility at this sample size.
- **P3 — CPCV from the start; report trades-per-fold.**
- **P4 — Project-lifetime test ledger** feeding FDR and PBO.
- **P5 — ADR-0011 continuous-contract / bar-adjustment** before any feature code; decide per-feature.
- **P6 — Multiple negative controls; FDR across feature×horizon (or bind the 30-min horizon); counterfactual slippage bands.**
- **P7 — Name and instrument the behavioral feedback loop** (permanent blind control stream into Phase 8).

Two cheap additions surfaced alongside:
- **"Minimum tradeable edge" in dollars**, decided with Allie at kickoff (everything is in R/IC; a 0.2R edge on a small stop may be below commission+slippage variance).
- **Unit-of-analysis decision rule for scale-outs** (one row per entry with a canonical weighted-average exit), not just "ask Allie."

---

## 4. What the plan already gets right (keep)

- 20-hazard catalog with per-phase resolutions (DEVELOPMENT_PLAN §2.8).
- FDR throughout; "report N and number of tests on every claim."
- Counterfactual dataset as a first-class, parallel track.
- Net-of-costs as the gate metric (QH1), forward returns as primary outcome when stops are mental (QH2/QH18).
- "NO-GO is a successful outcome" framing and pre-committed M7 acceptance (KICKOFF §2.2).
- Point-in-time discipline + truncation-invariance fitness function (F1/F11).

The recommendations above do not replace any of this; they close the gaps it leaves.
