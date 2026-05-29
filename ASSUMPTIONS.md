# ASSUMPTIONS.md

**Purpose**: Surface the assumptions this project is built on, score them by importance and evidence strength, and identify which to test before committing 9 weeks of build.

**Method**: VUBF framework (Value, Usability, Business viability, Feasibility) applied to the project as currently specified in `CLAUDE.md`, `RESEARCH_PLAN.md`, `DEVELOPMENT_PLAN.md`, and `PROJECT_KICKOFF.md`.

**Companion**: `PROJECT_KICKOFF.md` operationalizes the testing of several assumptions in this doc. This doc maps WHY each item in the kickoff matters; the kickoff captures the answers.

**Status**: `[ ] Draft  [ ] Reviewed with Allie  [ ] Approved`

---

## 0. TL;DR

**Five assumptions are load-bearing and currently have weak evidence**. If any of the five proves false, the MVP either can't be built or won't produce a usable result:

1. Allie can articulate her setups as logical predicates.
2. Allie has 200+ trades of consistent-rules history.
3. A real edge exists in Allie's current trading (Q1 will return positive).
4. Allie will change behavior based on grades the system produces.
5. Bryce can sustain ~10 hrs/week for 9 weeks.

**All five are testable in under 4 hours of combined work BEFORE Phase 0 starts.** That's the highest-ROI block of time on the entire project. Test results go into `PROJECT_KICKOFF.md`.

The remaining ~15 assumptions are either lower-importance, already have strong evidence, or are inherently uncertain (test by building).

---

## 1. The VUBF framework as applied here

### Value Risk — Will Allie actually want/use this?
Standard product-market-fit risk, scoped to a user of 1.

### Usability Risk — Can Allie figure out the output?
Specific to ADHD-friendly report design. If Allie won't read a 4-page HTML report, the rubric inside it is wasted.

### Business Viability Risk — Is this a sustainable investment of Bryce's time?
There is no commercial business here. "Viability" reframed as: does Bryce's effort and Allie's commitment hold long enough to deliver the MVP AND maintain it?

### Feasibility Risk — Can we actually build it?
Most risks live here for a system this technical. Includes data availability, methodology power, and statistical detectability of an edge that may or may not exist.

---

## 2. Full assumption table

Scoring: **Importance** = how much the project fails if this is wrong. **Evidence** = how much we currently know this is true. **Priority** = derived (High importance × Weak evidence = test first).

### Value assumptions

| # | Assumption | Importance | Evidence | Priority |
|---|---|---|---|---|
| V1 | Allie wants a data-driven grading system more than she wants coaching, more journaling discipline, or just more practice | H | M | Test |
| V2 | Allie will read the HTML reports the system produces (not let them pile up unread) | H | L | Test |
| V3 | A grading rubric (vs other tool shapes) is the right form factor for her job-to-be-done | H | L | Test |
| V4 | Allie will change behavior based on the grades — size up A+, skip B, etc. | H | L | Test |
| V5 | The rubric will surface insights Allie doesn't already know intuitively | M | M | Monitor |
| V6 | Allie will trust a data-derived rubric enough to act against her gut when they disagree | H | L | Test |
| V7 | "Deep analysis" framing genuinely matches what Allie wants (vs real-time grading later) | M | L | Test cheaply via kickoff conversation |

### Usability assumptions

| # | Assumption | Importance | Evidence | Priority |
|---|---|---|---|---|
| U1 | ADHD-friendly visual reports are enough; Allie can interpret without statistical training | H | M | Monitor (test at Phase 4 prototype) |
| U2 | Allie's existing Tradezella vocabulary is sufficient for rubric language; no new jargon needed | M | M | Monitor |
| U3 | Allie can apply suggested Tradezella tags without friction | M | M | Monitor |
| U4 | Allie can remember what A+/A/B mean and apply consistently over weeks | M | L | Test eventually |
| U5 | Reports don't require Bryce to walk Allie through them | M | L | Test at Phase 4 with one report cold-read |

### Business viability (project sustainability)

| # | Assumption | Importance | Evidence | Priority |
|---|---|---|---|---|
| B1 | Bryce can sustain ~10 hrs/week for 9 weeks straight | H | M | Test (calendar audit) |
| B2 | Bryce can sustain ~4 hrs/month maintenance post-MVP indefinitely | H | L | Test (set explicit budget) |
| B3 | The project has a clear "done" criterion that prevents perpetual side-project status | M | L | Already addressed in `DEVELOPMENT_PLAN.md` §3 Phase 8; kickoff §7 |
| B4 | Allie's trading continues through the 9-week build + 3-month forward-test | H | M | Monitor; if she pauses trading, project freezes |
| B5 | Vendor stack (~$1.5k/yr) is acceptable to whoever pays | M | M | Confirm at kickoff |
| B6 | Bryce/Allie relationship dynamic sustains through "your trading might not have an edge" conversation | H | L | Test before Phase 0 |
| B7 | The opportunity cost of 9 weeks of Bryce's evenings is acceptable | M | L | Compare to alternatives at kickoff |

### Feasibility assumptions

| # | Assumption | Importance | Evidence | Priority |
|---|---|---|---|---|
| F1 | Allie can articulate her setups as logical predicates ("long if X AND Y AND Z") | H | L | **Test immediately** |
| F2 | Allie has 200+ trades of consistent-rules history in Tradezella | H | M | Test (Tradezella audit, 30 min) |
| F3 | A real edge exists in Allie's current strategy (Q1 returns positive 95% CI on mean R) | H | L | Test at M2; cannot pre-validate without running it |
| F4 | Tradezella export contains all required fields reliably (planned stop, commission, timezone-stamped timestamps) | H | L | Test (export one month, inspect) |
| F5 | A bar data source exists and is accessible (NT / broker API / TradingView / paid feed) | H | L | **Test immediately** (resolve `PROJECT_KICKOFF.md` 4.3) |
| F6 | Tick data exists for order flow features (if not, defer them; not fatal to MVP) | M | L | Test |
| F7 | Allie's rules don't drift during the 9-week build (rule-freeze commitment holds) | H | L | Test by stated commitment + monitor |
| F8 | Counterfactual setup detector can hit >= 85% agreement with Allie | M | L | Test in Phase 3 (also a kill criterion) |
| F9 | At least 3 features will clear FDR-adjusted IC threshold of 0.10 (Q2) | H | L | Test at M4; cannot pre-validate |
| F10 | The rubric will produce monotonic OOS expectancy in walk-forward (Q3) | H | L | Test at M7; cannot pre-validate |
| F11 | The negative-control feature will NOT clear FDR (methodology sanity) | H | UNKNOWN | Test at M4 |
| F12 | Walk-forward will have enough windows (>= 5) to be statistically meaningful given dataset size | M | L | Test at M0 (count trades); QH19 in DEVELOPMENT_PLAN.md §2.8 already addresses |
| F13 | Tradezella's trade-unit aggregation gives a usable unit of analysis (no double-counting partials) | M | L | Test at M1 (QH13) |
| F14 | Allie's mental stops, if used, can be reconstructed well enough to compute R-multiple — OR forward-return outcome is acceptable substitute | M | L | Test at M0; QH18 in DEVELOPMENT_PLAN.md §2.8 |
| F15 | Bryce will not introduce a silent leak (point-in-time, look-ahead, FDR violation) | M | M | Test continuously; F1/F11 fitness functions enforce |

---

## 3. Top 5 assumptions to test BEFORE Phase 0

Selected by High importance × Weak evidence. All five testable in under 4 hours total. **None require code.**

### Test 1: Can Allie articulate her setups as logical predicates? (F1)

**Assumption**: Allie can describe her primary setup precisely enough that a Pine Script or Python predicate can match her mental criteria 85% of the time.

**Riskiest version**: She cannot. Her edge (if any) is too intuitive, contextual, or multi-cue to codify. The counterfactual story breaks, the Pine detector doesn't work, and the analysis is permanently biased by selection effects we can't measure.

**Cheapest test** (60 minutes):
1. Allie verbally describes her #1 setup for 5 minutes.
2. Bryce codifies it as a one-paragraph predicate ("long MNQ if price within 0.3 ATR of EMA20 AND 60m trend is up AND prior bar closed above its midpoint").
3. Bryce screen-shares 10 recent charts (5 that were Allie's setups, 5 that weren't, mixed order). For each, Allie says yes/no in real time. Bryce evaluates the predicate.
4. Compare hit rates.

**Validated**: >= 7/10 agreement on the first try. Iterate on misses with Allie to refine the predicate.

**Invalidated**: < 5/10 OR Allie says "well, it depends on a feeling" more than twice. Project scope must change.

**Invalidation implication**: Drop counterfactual analysis; accept selection bias; narrow MVP to within-taken-trade comparisons only. Be honest in every report that effect sizes are biased downward for features inside Allie's existing mental filter.

---

### Test 2: Does Allie have 200+ trades of consistent-rules history? (F2)

**Assumption**: Allie's Tradezella has at least 200 trades from a window where her entry/exit/sizing rules were stable.

**Riskiest version**: She has fewer trades, OR rule changes splinter the dataset into too-small chunks for any single stable-rules window to clear 200.

**Cheapest test** (30 minutes):
1. Allie pulls a Tradezella export covering all-time.
2. Bryce or Allie tabulates trade count by month.
3. Allie marks every rule change date she remembers (be ruthless; partial recall is worse than no recall).
4. Compute trade count in the longest stable-rules window.

**(PR-REVIEW-P1)** The 200-trade bar here is a *project-can-start* threshold, NOT a *rubric-is-buildable* threshold. They are different and the docs previously conflated them. Per RESEARCH_PLAN.md §1.3, detecting IC = 0.10 after FDR needs N ≈ 800 and a top/bottom-quintile 0.3R lift needs ~1,250–2,000. So 200 trades means "enough to run Q1 expectancy and underpowered univariate exploration," not "enough to validate a weighted rubric." The M0 MDES readout (RESEARCH_PLAN.md §7 M0) states the detectable effect size at the actual N and gates the rubric deliverable accordingly. See `PLAN_REVIEW.md` §0.

**Validated (project can start)**: At least one stable-rules window contains >= 200 trades AND spans >= 6 months. Q1 + exploratory univariate are feasible. A *validated rubric* still requires the MDES check to clear, which at 200–400 trades it almost certainly will not — expect the forward-capture system to be the real MVP.

**Soft pass**: 100-200 trades in the longest stable window. MVP narrows; only QH16-protected (tertile-fallback) analyses possible; walk-forward via combinatorial purged CV instead of rolling.

**Invalidated**: < 100 trades in any stable window. Project becomes "build the capture system, revisit in 6 months."

---

### Test 3: Does a real edge exist in Allie's trading? (V1 + F3, partial)

**Assumption**: Allie's mean per-trade R (net of costs) is positive with 95% bootstrap CI excluding zero, on the trades she's logged.

**Riskiest version**: She's flat or net-losing. The project is no longer "grade A vs B" — it's "find any edge at all," which is a different and harder project.

**Cheapest test** (45 minutes — no code, just spreadsheet math):
1. Allie's Tradezella export → compute net R per trade in a sheet (sum P&L / planned-stop-distance per trade, summing commissions).
2. Compute mean and 95% bootstrap CI on mean (use a free online bootstrap calculator OR a 20-line Python script).
3. Allie's gut answer for her win rate and average R compared to actual.

**Validated**: Mean R > 0 with CI excluding zero. Project proceeds as planned.

**Soft pass**: Mean R > 0 but CI includes zero. Project proceeds but the M2 memo headline says "edge not yet statistically confirmed; rubric construction is exploratory until forward-test."

**Invalidated**: Mean R < 0 OR CI clearly straddles zero with the negative side as wide as positive. Project pauses for strategy review BEFORE grading work. This is RESEARCH_PLAN.md M2 kill criterion landing pre-Phase-0 instead of at week 3 — much cheaper to discover now.

**Bonus output**: Allie's gut win-rate vs reality. If they match, Allie's self-knowledge is high and rubric upside is bigger. If gut is way off, the rubric will fight her intuition the whole time.

---

### Test 4: Will Allie change behavior based on grades? (V4)

**Assumption**: When the system says "A+" or "B," Allie will actually act differently (size differently, take/skip, etc.) — not just nod at the grade and trade the same way.

**Riskiest version**: She won't. Discretionary traders frequently override systematic signals. A correct rubric Allie ignores is worth zero.

**Cheapest test** (45 minutes, two parts):

**Part A — past adherence as predictor (15 min)**: Has Allie ever used a systematic grading or checklist tool? (Paper checklist, simple spreadsheet, broker-built scoring.) If yes, how long did she stick with it? If she abandoned past attempts in 2 weeks, that's the strongest possible predictor.

**Part B — behavioral protocol commitment (30 min)**: Before any code, Allie commits IN WRITING to one of:
- (Strong) A+ = full size, A = half size, B = skip. Most testable.
- (Medium) A+ and A = take normal, B = skip.
- (Weak) All grades = take normal, log only. No behavior change tested.

Make the choice. Sign and date it. This goes in `PROJECT_KICKOFF.md` section 2.3.

**Validated**: Allie has past adherence track record OR commits to Strong or Medium protocol.

**Invalidated**: Past attempts at systematic grading were abandoned within weeks AND Allie will only commit to the Weak protocol. The rubric becomes a journal tool, not a decision tool. MVP scope and success criteria need to be re-cut.

**Invalidation implication**: Re-scope to "automated post-hoc journaling and pattern detection," drop the real-time grading, drop the Pine alerts, drop the compliance metric. Smaller MVP, faster, still useful.

---

### Test 5: Can Bryce sustain 10 hrs/week for 9 weeks? (B1)

**Assumption**: Bryce will dedicate ~10 hours per week, every week, for 9 weeks straight to MVP delivery. Plus ~4 hrs/month maintenance forever.

**Riskiest version**: He can't. Project stretches to 18 weeks, motivation cliff hits at week 10, MVP doesn't ship.

**Cheapest test** (30 minutes):
1. Open Bryce's calendar. Look at the next 9 weeks.
2. Identify the 10 hours per week. Specifically: which evenings, how long.
3. Note conflicts: travel, holidays, other commitments, existing obligations.
4. Be honest. If you find yourself reasoning "well, I could move that," you don't have the 10 hours.

**Validated**: 10 hrs/week is identifiable in the calendar with at most 1-2 weeks of meaningful conflicts.

**Soft pass**: 6-8 hrs/week is realistic. Re-plan the phases at that pace (~13 weeks instead of 9).

**Invalidated**: < 5 hrs/week sustainable. Project scope must shrink dramatically OR the timeline doubles, in which case revisit the opportunity cost test (B7).

---

## 4. Decision rules summary

| If test ... | Then ... |
|---|---|
| F1 invalidated (no falsifiable setup) | Drop counterfactual; narrow MVP; be honest in reports about selection bias |
| F2 invalidated (<100 trades) | Pause project; build forward-capture infra only; revisit in 6 months |
| F3 invalidated (no edge in Q1) | Project pauses for strategy review BEFORE grading work begins |
| V4 invalidated (no behavioral commitment) | Re-cut MVP as post-hoc journaling tool; drop real-time grading and compliance tracking |
| B1 invalidated (calendar doesn't have hours) | Re-plan at realistic pace OR shrink scope OR don't start |
| **Any two of the above invalidate** | **Do not start Phase 0.** Take 4 weeks to address the conditions or kill the project. |

---

## 5. Assumptions that are NOT in the top-5 test set, and why

- **F4 (Tradezella has required fields)**: Cheap to test, but failure is recoverable (we adapt or add manual annotation). Test in Phase 1.
- **F5 (bar data source)**: An architectural blocker, but already explicitly an open question in `PROJECT_KICKOFF.md` 4.3. The pluggable `BarSource` protocol (DEVELOPMENT_PLAN.md §1.5) means the decision is recoverable.
- **F9, F10 (features clear FDR, rubric is monotonic OOS)**: Inherently can't be pre-tested without running the analysis. M4 and M7 are the tests. The plan's kill criteria handle invalidation.
- **F11 (negative control doesn't clear FDR)**: Same — only testable at M4. QH17 in DEVELOPMENT_PLAN.md §2.8 makes invalidation a methodology stop.
- **U1-U5 (usability)**: Can't be pre-tested without an artifact. First real test is the M1 DQ report; Allie cold-reads it. If she doesn't understand it, redesign before M2.
- **B4 (Allie's trading continues)**: Outside Bryce's control. Monitor; if she stops trading mid-project, the system has no purpose and the project freezes.

---

## 6. How to use this doc

**Before Phase 0 starts**:
1. Run the 5 pre-Phase-0 tests above (~4 hours total).
2. Record results in `PROJECT_KICKOFF.md` (most map directly to existing fields).
3. Apply the decision rules. If two or more invalidate, do not start Phase 0.

**During the build**:
- At each milestone memo, the report header lists which assumptions this milestone tests.
- M0 tests F2, F4, F13, F14. M2 tests F3 and surfaces gut-vs-actual. M3 tests F8. M4 tests F9 and F11. M7 tests F10.
- Any invalidation triggers the kill criterion in the corresponding plan section.

**Quarterly post-MVP**:
- Revisit V1, V4, V6, B2 (long-term usage and sustainability assumptions). These don't validate at MVP; they validate over months of forward-test.

**On any major scope or pivot decision**:
- Re-open this doc. New assumptions in the new direction get scored and added. Old assumptions that are no longer relevant get marked superseded (not deleted).

---

## 7. Open questions for the assumption set

- **Are there assumptions I missed entirely?** Allie may surface one in the kickoff that doesn't fit cleanly in VUBF — capture it under whichever category fits least poorly and note the awkwardness.
- **Are any "Strong evidence" ratings actually wishful thinking?** Re-rate any "Strong" that has no specific evidence cited.
- **The relationship dynamic (B6) is the most uncomfortable to test**. Skipping it because it's awkward is the most common way it bites later. If Bryce can't ask Allie "are we OK if this project tells you you don't have an edge?" then the project is on shakier ground than the methodology suggests.

---

*End of assumptions doc. Test the top 5 before Phase 0 begins.*
