# PROJECT_KICKOFF.md

**Purpose**: Capture everything we need from Allie (the trader this system is being built for) before Phase 0 starts. This is the operating contract for the project plus the data inventory plus the unresolved decisions, in one place.

**Project context**:
- **Allie** is the trader. ~1 year of experience, primary instruments MNQ and NQ, logs trades in Tradezella. Has ADHD. End consumer of the reports and grades.
- **Bryce** is sole developer and project owner. Builds and maintains the pipeline. Not the trader.

**How to use**: Allie fills in every placeholder marked `[FILL IN]`. Bryce facilitates and answers any technical questions. Sections marked **(required for kickoff)** block Phase 0 if blank. Sections marked **(required for relevant phase)** block that phase only. Doc is signed at the bottom when complete.

**Estimated time to complete**: 90 minutes if Bryce facilitates a single sitting. 2-3 hours if Allie fills in solo and Bryce reviews after.

**Report/UX note**: Allie has ADHD. Every artifact she consumes (HTML reports, dashboards, alerts) needs to be visually scannable: short sections, color-coded, conclusion-first, no dense tables-of-numbers as the primary view. Dense detail is fine BELOW the summary, not as the lead.

**Status**: `[ ] Draft  [ ] Reviewed  [ ] Signed`

---

## 1. Project framing **(required for kickoff)**

### 1.1 The job to be done

Describe, in one paragraph, the moment in your trading day when this system "wins." What time of day, what does it tell you, what do you do next?

> `[FILL IN]`

### 1.2 Pick the dominant job

Check ONE. We can revisit, but we need a primary to scope the MVP.

- [ ] **Real-time decision support**: grade tells me to take, skip, or size differently before I click buy.
- [ ] **Post-hoc journaling**: grade helps me categorize trades for review; no behavior change in the moment.
- [ ] **Confidence calibration**: I have a mental rubric; I want to test whether it matches the data.
- [ ] **Proof of edge**: I want to know whether I actually have an edge at all, before doing more discretionary work.
- [ ] **Self-control / tilt prevention**: I want a system that pushes back when I'm about to take a B trade in a frustrated state.

### 1.3 Success vision

A year from now, in one paragraph, what does success look like? Be concrete about behavior, not metrics.

> `[FILL IN]`

### 1.4 Smallest acceptable outcome

If M7 returns "no go, wait 6 more months for data," what is the smallest deliverable from this project you would still consider worth the 9 weeks?

> `[FILL IN]`

---

## 2. Operating commitments **(required for kickoff)**

These are the behavioral contracts that make the methodology valid. Each is a real commitment, not a vibe.

### 2.1 Rule-freeze commitment

I commit to making NO changes to entry criteria, exit criteria, stop methodology, sizing, or instrument set during the 9-week build (Phase 0 through M7).

If a brilliant idea hits me mid-build, I will log it for "post-MVP" and not implement.

- [ ] Yes, I commit.
- [ ] No, this is unrealistic. I propose instead: `[FILL IN alternative, e.g., "freeze through M4 only"]`
- [ ] I have no current rules to freeze (this is itself a flag; talk to me before proceeding).

### 2.2 M7 verdict acceptance

I accept that M7 = NO GO is a successful project outcome. I will not renegotiate the promotion criteria in section 5.7 of RESEARCH_PLAN.md mid-project to force a GO verdict.

If M7 = NO GO, my pre-committed next step is: `[FILL IN: wait 6mo, abandon, scope down, etc.]`

- [ ] Yes, I commit.
- [ ] No, here's what I want to change about the M7 bar: `[FILL IN]`

### 2.3 Behavioral protocol (what an A+ vs A vs B will mean)

This decides BEFORE we build what grades operationally mean. Pick one.

- [ ] **Strong test**: A+ = full size, A = half size, B = skip.
- [ ] **Medium test**: A+ and A = take normal size, B = skip.
- [ ] **Soft test**: All grades = take normal, log only. (No real test of rubric value; lowest behavioral risk.)
- [ ] Other: `[FILL IN]`

Specifically: **if an A+ alert fires tomorrow morning and the rubric disagrees with my gut, I will:**

> `[FILL IN]`

### 2.4 Compliance threshold

We will measure % of A+ alerts taken and % of B alerts skipped during forward-test. If compliance drops below `[FILL IN: suggest 70%]` for `[FILL IN: suggest 30 days]`, we pause and reassess whether the rubric is the product or willpower is.

### 2.5 Hypothesis pre-registration

Before any analysis runs, write down your current hypotheses. We compare them to data at M4.

- **Feature I believe is my biggest edge factor**: `[FILL IN]`
- **Feature I believe is overrated by other traders**: `[FILL IN]`
- **Feature I believe has zero edge (negative control)**: `[FILL IN]`
- **One thing I expect to be wrong about**: `[FILL IN]`

### 2.6 External reviewer

One person (other than me/Claude) who will look at rubric v1 before promotion for a 30-min sanity check. Even a trading-literate friend, ex-colleague, or paid consultant for an hour.

> Name / role: `[FILL IN]`
> If "nobody," acknowledge: `[ ] I acknowledge this is a single-point-of-failure risk and accept it.`

---

## 3. Trading reality **(required for kickoff)**

### 3.1 Self-assessment, by gut

Used as a baseline calibration check at M2.

- Current overall win rate (your guess): `[FILL IN]` %
- Current overall average R per trade (your guess): `[FILL IN]` R
- Current monthly P&L trend (your guess): `[FILL IN: green / red / flat / mixed]`
- Confidence that you're currently net-positive after costs: `[FILL IN: 1-10]`

### 3.2 History

- Date you started actively trading futures: `[FILL IN]`
- Approximate total trades in NT history: `[FILL IN]`
- Approximate trades in the last 90 days: `[FILL IN]`
- Most recent rule change date and what changed: `[FILL IN]`
- List ALL date-stamped rule changes in the last 12 months. Be ruthless; partial recall is worse than no recall.

| Date | What changed |
|---|---|
| `[FILL IN]` | `[FILL IN]` |
| | |
| | |

### 3.3 Account context

For each period in the last 12 months, mark account type. Affects whether trades pool or split.

| Period | Account type | Notes |
|---|---|---|
| `[FILL IN]` | `[ ] Live  [ ] Sim  [ ] Funded challenge  [ ] Other`  | |
| | | |

### 3.4 Instruments

Default scope for MVP is MNQ and NQ (pre-confirmed in kickoff context). Confirm and add detail.

- [ ] Confirmed: MNQ + NQ only for MVP.
- [ ] Modify: also include `[FILL IN]`
- [ ] Modify: only `[FILL IN]` (drop NQ or MNQ)

Approximate split MNQ vs NQ in last 12 months:
- MNQ: `[FILL IN]` % of trades
- NQ: `[FILL IN]` % of trades

Do you treat MNQ and NQ as the same setup with different sizing, or do you trade them differently?
- [ ] Same setup, different size (pool in analysis)
- [ ] Different (separate rubrics or separate analysis)
- [ ] Other: `[FILL IN]`

### 3.5 Stop placement methodology (highest priority unknown)

This drives whether R-multiple or forward-horizon return is our primary outcome.

- [ ] Fixed-tick (e.g., always 8 ticks)
- [ ] ATR-based (e.g., 1 ATR below entry)
- [ ] Structure-based (recent swing low/high)
- [ ] Mental (I exit when it looks wrong)
- [ ] Combination: `[FILL IN]`

If mental: is the intended stop logged anywhere at entry, or is it only recoverable post-hoc from where you actually exited? `[FILL IN]`

### 3.6 Position sizing logic

- [ ] Always 1 contract
- [ ] Always N contracts (specify): `[FILL IN]`
- [ ] Varies by conviction
- [ ] Varies by setup type
- [ ] Other: `[FILL IN]`

If varies: describe the sizing rule: `[FILL IN]`

### 3.7 Setups in scope for MVP

For each setup you want graded (1-3 max for MVP):

#### Setup 1
- Name: `[FILL IN]`
- One-paragraph description in plain English: `[FILL IN]`
- Entry criterion as a logical predicate (e.g., "long ES if price within 0.3 ATR of EMA20 AND 60m trend is up AND prior bar made a higher low"): `[FILL IN]`
- Typical exit rule: `[FILL IN]`
- Approximate frequency (per day or per week): `[FILL IN]`
- Approximate trades in last 12 months: `[FILL IN]`

#### Setup 2 (if applicable)
- Name: `[FILL IN]`
- ... (same fields)

#### Setup 3 (if applicable)
- Name: `[FILL IN]`
- ... (same fields)

### 3.8 What counts as a "trade"

Define the unit of analysis. Tradezella aggregates differently; we may need to disaggregate.

- A scratch (entered, exited at break-even within seconds): `[ ] count  [ ] drop`
- A quick scalp (<2 min): `[ ] count  [ ] drop`
- A partial scale (entered 3 lots, exited 1 then 2): `[ ] one trade  [ ] multiple trades`
- A re-entry after a stop: `[ ] one trade  [ ] separate trade`
- Other edge cases: `[FILL IN]`

### 3.9 What you believe your edge is

In one paragraph, your honest belief about what makes your A+ setups A+. Concrete factors. This is tested directly against the data at M4.

> `[FILL IN]`

### 3.10 Hard constraints on the rubric

Factors you refuse to grade on for any reason, even if data says they matter.

- `[FILL IN, e.g., "I don't care about day-of-week, don't put it in"]`
- `[FILL IN]`
- `[FILL IN]`

---

## 4. Data inventory **(required for Phase 1)**

### 4.1 Tradezella (canonical source for trade fills)

- Tradezella plan tier: `[ ] Free  [ ] Premium  [ ] Other: [FILL IN]`
- Earliest trade date logged in Tradezella: `[FILL IN]`
- Approximate total trades logged: `[FILL IN]`
- Have you ever cleared/reset Tradezella history? When: `[FILL IN]`
- Can you provide a sample CSV export (last 30 days): `[ ] Attached  [ ] To do by [date]`
- Field completeness check, do these export reliably for every trade:
  - Entry/exit timestamp with timezone: `[ ] Yes  [ ] Sometimes  [ ] No`
  - Planned stop price: `[ ] Yes  [ ] Sometimes  [ ] No`
  - Planned target price: `[ ] Yes  [ ] Sometimes  [ ] No`
  - Commission: `[ ] Yes  [ ] Sometimes  [ ] No`
- Existing tag taxonomy in use:
  - [ ] Yes, paste current tags: `[FILL IN]`
  - [ ] Minimal: `[FILL IN]`
  - [ ] None
- Existing custom fields (notes, mood, conviction, etc.): `[FILL IN]`

### 4.2 Broker / execution platform

What do you actually click buy/sell in?

- [ ] NinjaTrader 8 (then Tradezella syncs from NT)
- [ ] Tradovate (direct)
- [ ] TopstepX
- [ ] AMP Futures
- [ ] Other: `[FILL IN]`

How does data flow into Tradezella:
- [ ] Auto-sync (which integration): `[FILL IN]`
- [ ] Manual CSV upload
- [ ] Other: `[FILL IN]`

### 4.3 Bar data source (open architectural question)

Tradezella stores trade fills but NOT OHLCV bars or tick data. We need a source for the bar context around each entry to compute features. Options:

- [ ] **NinjaTrader 8** (free if Allie uses it for execution; stores historical bars locally). NT8 build: `[FILL IN if applicable]`
- [ ] **TradingView export** (chart-by-chart manual export; limited history and granularity)
- [ ] **Broker API** (Tradovate, etc., if available): `[FILL IN]`
- [ ] **Paid data feed** (Databento ~$100/mo, Polygon, CQG): `[FILL IN]`
- [ ] **None of the above** (this is a blocker; resolve before Phase 1)

Preferred source for MVP: `[FILL IN]`

### 4.4 Tick data

Order flow features (cumulative delta, large lot ratio, bid/ask) require tick-level data. They are NOT required for MVP but they unlock the "added" feature categories in CLAUDE.md.

- Do you have tick-level history from any source: `[ ] Yes since [date]  [ ] No  [ ] Forward-only from when capture is enabled`
- If yes, source: `[FILL IN]`
- Acceptable to defer order flow features to post-MVP: `[ ] Yes  [ ] No, must include`

### 4.5 TradingView

- Plan tier: `[ ] Free  [ ] Pro  [ ] Pro+  [ ] Premium`
- Note: Premium ($60/mo) required for webhook alerts to URLs. If not on Premium AND we want real-time Pine alerts, plan needs to change. For deep-analysis-only MVP, TradingView Free may be sufficient.

### 4.6 Conviction logging

Will Allie commit to logging a 1-5 pre-trade conviction score on EVERY trade going forward (5 seconds per trade)? Required for Q4 in RESEARCH_PLAN.md.

Practically: add a custom field in Tradezella for conviction, OR write it into the trade note before clicking buy.

- [ ] Yes, via Tradezella custom field
- [ ] Yes, via trade note
- [ ] No (Q4 dropped from research scope)

### 4.7 Historical setup records

Does Allie have any historical record of setups SEEN but PASSED on? Screenshots, journal entries, screen recordings. Even partial helps bound counterfactual replay validation.

- [ ] Yes, format: `[FILL IN]`
- [ ] No

---

## 5. Vendor / tooling **(required for Phase 1)**

### 5.1 Annual tooling budget (hard cap)

> `$[FILL IN]` per year

### 5.2 Current monthly recurring vendor costs

| Vendor | $/mo | Notes |
|---|---|---|
| TradingView | `[FILL IN]` | |
| NinjaTrader (license/lease) | `[FILL IN]` | |
| Tradezella | `[FILL IN]` | |
| Data feed (Kinetick, CQG, etc.) | `[FILL IN]` | |
| Funded account fees | `[FILL IN]` | |
| Other | `[FILL IN]` | |

### 5.3 Google account for Sheet + Apps Script (only needed if real-time Pine alerts in scope)

- Account to use: `[FILL IN]`
- Owner of the sheet: `[FILL IN: Bryce / Allie / shared]`
- Permissions model: `[FILL IN]`

### 5.4 GitHub repo

- Repo visibility: `[ ] Private  [ ] Public`
- Repo owner: Bryce (sole dev).
- Does Allie need any read access (to look at outputs in the repo): `[ ] Yes  [ ] No, she only consumes reports outside the repo`

---

## 6. Project constraints **(required for kickoff)**

### 6.1 Time availability

Honest sustained hours per week you can put into this:

- During build (Phases 0-6, ~9 weeks): `[FILL IN]` hours/week
- During forward-test (Phase 7, months 4-6): `[FILL IN]` hours/week
- Post-MVP maintenance (Phase 8 ongoing): `[FILL IN]` hours/month

**Whatever you answer for maintenance, halve it.** Can the system live within that?

> `[FILL IN]`

### 6.2 Opportunity cost

If you couldn't build this system, what's the next-best use of the same 9 weeks toward trading improvement?

> `[FILL IN]`

Should that alternative happen INSTEAD or IN PARALLEL?

- [ ] Instead (pause project, do the alternative)
- [ ] In parallel (both)
- [ ] System only

### 6.3 Past systematic-grading attempts

Have you tried any systematic grading before (paper checklist, simple spreadsheet)?

> `[FILL IN]`

If yes, how long did you stick with it before stopping or modifying?

> `[FILL IN]`

(Past adherence is the best predictor of future adherence. Honest answer changes the product risk.)

### 6.4 Build vs. love-building motivation

Honest self-assessment:

- [ ] I want the tool, smallest version that works
- [ ] I love building and the tool is partly an excuse
- [ ] Both, roughly equal

Different answers imply different scopes. Answer changes the MVP.

---

## 7. Done criteria **(required for kickoff)**

### 7.1 Project exit criteria

When does this project enter "maintenance only, no new features" mode?

Proposed default: **rubric weights have not moved by 2+ levels across two consecutive quarterly refits AND live forward-test expectancy matches walk-forward within CI for 6 months.**

- [ ] Accept default.
- [ ] Modify: `[FILL IN]`

### 7.2 Project abandonment criteria

What conditions would make you stop the project entirely?

> `[FILL IN]`

### 7.3 Stability + autoregression policy

If a quarterly refit moves any feature weight by 2+ levels, default policy is: **pause, investigate, and run the new rubric in shadow for one quarter before going live.**

- [ ] Accept default.
- [ ] Modify: `[FILL IN]`

---

## 8. Open questions to resolve before Phase 0 kicks off

These are unresolved blockers. Owner and target date for each.

| # | Question | Owner | Target date | Status |
|---|---|---|---|---|
| 1 | Setup definitions in falsifiable predicate form (section 3.7) | Allie | | |
| 2 | Stop methodology decision (section 3.5) drives primary outcome metric | Allie | | |
| 3 | Rule-freeze commitment (section 2.1) | Allie | | |
| 4 | M7 NO-GO acceptance and pre-committed next step (section 2.2) | Allie + Bryce | | |
| 5 | Behavioral protocol (section 2.3) | Allie | | |
| 6 | External reviewer identified (section 2.6) | Bryce | | |
| 7 | Sample Tradezella export CSV (section 4.1) | Allie | | |
| 8 | Broker / execution platform confirmed (section 4.2) | Allie | | |
| 9 | Bar data source decided (section 4.3) — architectural blocker | Bryce + Allie | | |
| 10 | Tick data availability confirmed (section 4.4) | Allie | | |
| 11 | TradingView plan tier check (section 4.5) | Allie | | |
| 12 | Conviction logging commitment (section 4.6) | Allie | | |

---

## 9. Sign-off

This document represents the operating contract for the project. By signing:

- Allie confirms the commitments in sections 2 and 6 are real, not aspirational.
- Allie confirms the data inventory in section 4 is accurate to best knowledge.
- The development team accepts the constraints and scope.
- The MVP scope is approved as defined in `DEVELOPMENT_PLAN.md` section 4, modified by any constraints captured here.

Signed:

- **Allie (trader)**: `[NAME]` Date: `[DATE]`
- **Development lead**: `[NAME]` Date: `[DATE]`

---

## 10. Triggers to revisit this doc

This document is re-opened (not silently amended) if any of the following occur:

- A rule change is requested mid-build (violates section 2.1).
- M7 returns NO GO and pre-committed next step needs activation (section 2.2).
- Compliance drops below threshold (section 2.4).
- A new setup is proposed for scope (section 3.7).
- Vendor pricing changes meaningfully (section 5.1).
- Allie's time availability changes by more than 50% (section 6.1).
- Hypotheses from section 2.5 disagree sharply with M4 results.

---

*End of kickoff doc. Phase 0 does not begin until this is signed.*
