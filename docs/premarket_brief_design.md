# Pre-Market Brief: Design and Data Requirements

**Status**: Proposed design. Not yet scheduled for a phase.
**Audience**: Bryce (developer) + Allie (trader, end consumer).
**Relationship to MVP**: Independent of the grading rubric. This is a read-only consumer of bar data and a calendar feed. It has no dependency on Allie's trade history, so it can ship on its own and remains useful even if the M0 MDES check shelves the rubric.
**Instruments in scope**: MNQ and NQ only.
**Delivery**: Email (push at 07:00 ET) plus a Google Sheet / dashboard view for drill-down.

> Note on MNQ vs NQ: both track the same underlying (the Nasdaq-100 future). Their prices move together to the tick; they differ only in contract size, tick value, and liquidity. So the brief computes one set of price levels and labels it for both contracts. It does not need two parallel analyses.

---

## 1. TL;DR

A 07:00 ET brief that tells Allie, in one glance: where price is now relative to the levels that matter (prior day, overnight, week, month), what happened overnight, how volatile the setup is, and what scheduled catalysts hit today. The whole thing is conclusion-first and color-coded per the ADHD-friendly report rules in `CLAUDE.md`.

The build is gated by one hard requirement: **a bar data source that provides session-tagged overnight (Globex) data plus official daily settlements.** Without that, half the brief cannot be computed. This is a stricter version of the bar-source decision already open in `PROJECT_KICKOFF.md` section 4.3.

---

## 2. Data requirements (the core question)

Five feeds. Feed A is load-bearing. The minimum viable brief is Feed A plus Feed D.

### Feed A: Session-tagged intraday bars for the Nasdaq-100 future (REQUIRED)

This is the whole game. To separate "yesterday's range" from "the overnight move," the bar data must distinguish the regular session from the overnight session. It must provide:

| Requirement | Why | Failure mode if missing |
|---|---|---|
| **Overnight / Globex bars** (the ~23h session: Sun-Fri 18:00 ET to 17:00 ET next day, with the 17:00-18:00 ET halt) | Overnight high/low and overnight trend live here | An RTH-only feed (09:30-16:00) cannot produce overnight levels at all |
| **Per-bar session tag** (RTH vs ETH) | Lets you segment prior-day RTH range vs overnight range | "Yesterday's high" silently becomes a 24h high, which is the wrong level |
| **Official daily settlement price** | Futures gap math is measured against the prior settlement, not the last tick | Gap size is off by the difference between last-trade and settlement |
| **History depth: full current calendar month** of intraday bars (or correctly-defined daily bars) | Monthly and weekly high/low lookbacks | Monthly H/L is wrong or unavailable |
| **Resolution: 1-min or 5-min** | Plenty for levels and overnight VWAP | None; finer is unnecessary |

**Source: DECIDED -> Databento `GLBX.MDP3`** (see `premarket_brief_vendor_research.md` for the comparison and `premarket_brief_architecture.md` section 0 for the cascade). Bars via the `ohlcv-1m` schema (full Globex session); official settlements + open interest via the `statistics` schema; continuous front-month symbology `NQ.c.0`; Historical API only. Pay-as-you-go, single-instrument cost is small. The cheapest fallback (NinjaTrader + local ETH feed) is documented but not chosen.

Because MNQ and NQ share the underlying, Feed A only needs ONE instrument's bars (use NQ via `NQ.c.0` as the reference series; label output for both).

### Feed C: Volatility reference (recommended)

- **VIX** prior close plus current value. Caveat: cash VIX does not trade at 07:00 ET, so the brief uses the prior VIX close, optionally with **VX futures** overnight as a live proxy. Decide how exact this needs to be.
- Used to print an **expected daily move** (rough rule: VIX divided by ~16 approximates the expected one-day percent move).

### Feed D: Economic calendar (REQUIRED, new dependency)

Structured events for **today**, with ET timestamps and an importance rating: CPI, PPI, NFP, retail sales, jobless claims (the 08:30 ET prints move index futures most), 10:00 ET data, 14:00 ET FOMC, and Fed speakers. The project today only models event *distance* as a feature (QH12). The brief needs the actual **calendar feed** for the day, from an API or a maintained source.

### Feed E: Earnings calendar (recommended, Nasdaq-specific)

The Nasdaq-100 is tech-heavy, so **mega-cap earnings** (AAPL, MSFT, NVDA, AMZN, GOOGL, META and similar) before the open or after the prior close move NQ hard. Needs an earnings calendar with before-market / after-market timing.

### Out of scope (per current decision)

Cross-asset context (ES, DXY, yields, crude, gold, global indices). ES would be the single highest-value addition if revisited later, since it leads NQ, but it is not in scope now.

---

## 3. What the brief contains

Conclusion-first, scannable, color-coded (green up / red down / gray inconclusive), per `CLAUDE.md` "How to design outputs for Allie."

1. **Headline (one line)**: gap up or down vs prior settlement (points and percent), and where price sits in the overnight range. Example: "NQ +0.4% (gap up 78 pts), trading near the top of a quiet overnight range."

2. **Key levels** (each with current price marked relative to it):
   - Prior-day RTH **high / low / close / settlement** (PDH / PDL)
   - **Overnight high / low** (ONH / ONL)
   - **Prior-week** high / low
   - **Current-month** high / low
   - **Overnight VWAP**, plus weekly and monthly open

3. **Overnight story**: net change since the 18:00 ET reopen, overnight range size **relative to the average overnight range** (was it a big night or a quiet one), and whether price trended or chopped.

4. **Volatility context**: daily ATR, gap size expressed **as a fraction of ATR** (a 0.3-ATR gap behaves differently from a 1.5-ATR gap), VIX and the implied expected daily move.

5. **Today's catalysts**: timed economic events and mega-cap earnings, color-coded by importance. The line that changes how she trades the open is "08:30 CPI, high impact."

Each contract (MNQ, NQ) gets its tick-value annotation so the points-based levels translate to dollars for whichever she is trading, but the levels themselves are shared.

---

## 3A. Curated content spec (quant-analyst + risk-manager review)

A quant-analyst and risk-manager reviewed the content above. Both converged on one principle: **the cut list matters more than the keep list. The brief should feel boring on a boring day, and that restraint is the product.** A wall of numbers at 07:00 for a one-year ADHD trader anchors her, induces overtrading, or freezes her. This section TIGHTENS section 3; where they conflict, this section wins.

### The brief, top to bottom (5 blocks, one phone screen)

1. **Risk Mode badge (new, goes first).** One color-coded line that does the synthesis for her, because a one-year trader will not turn six data points into a sizing decision at 07:00:
   - GREEN: normal day, trade your plan, normal size.
   - YELLOW: trade smaller / be selective (elevated implied move, oversized overnight range, OPEX, mega-cap earnings reaction).
   - RED: stand aside or trade tiny until the catalyst clears (high-impact timed event in or before her session).
   Everything below the badge is just the evidence behind it.

2. **Headline line** with the volatility qualifier baked in. Example: "NQ +78 pts (+0.4%), gap up, just under yesterday's high, on a quiet night (gap = 0.3x ATR)." Color the line green/red by direction; render GRAY (inconclusive) when the gap is under ~0.2 ATR so a non-event does not read like an event.

3. **Catalyst banner.** High and medium impact only, capped at 3, expressed as clock-time no-trade windows she can set a timer for: "Avoid 08:28-08:35 (CPI)." When nothing is scheduled, SAY SO ("No major events, normal session"); that is itself decision-useful. Hard implication stated plainly: do not hold through an 08:30 print, stops do not protect you through the number. Flag FOMC 14:00 / presser 14:30 as a hard no-trade window if her session runs that late.

4. **Price ladder (visual, not a table).** Current price marked in its true position between levels; nearest level above and below bolded; distance shown in points AND ATR-fractions so proximity is encoded by position, not mental math.

   ```
        --- ONH      20,460  (+42)
   NOW  20,418  -----------------------
        --- PDH      20,405  (-13)   [nearest below]
        --- ON VWAP  20,360
        --- PDL      20,290
        --- ONL      20,255
   ```

5. **Volatility + sizing line.** Daily ATR, gap-as-fraction-of-ATR, and the core risk principle in plain terms: **dollar risk per trade is the constant, contract count is the variable.** "High-vol day: wider stop, fewer contracts to hold your dollar risk fixed." Plus a daily-stop reminder framed as "you are done after about N full stop-outs."

### The cut list (both reviewers agreed)

- **Prior-day close** as its own displayed level: redundant with settlement, false precision. Keep settlement (it is the gap reference); do not display close.
- **Weekly / monthly high-low unconditionally:** show ONLY when current price is within ~1 ATR; suppress otherwise. Most days they will not appear, which is correct.
- **Weekly open and monthly open:** cut entirely; not day-trade-relevant.
- **VX-futures live proxy** (open question 2 below): do not build it. Prior-close VIX is sufficient, and VIX itself sits behind ATR, not beside it (borderline cut from the glance entirely).
- **Cross-asset / ES:** stays out. ES leads NQ on a seconds-to-minutes basis (a live-tape signal), not a 07:00 item; at 07:00 it is redundant with the NQ overnight story.
- **Any directional language or bias:** hard rule. Aligns with the repo-wide "no predicting market direction" anti-goal. The brief reports STATE, never a lean. This is the number-one source of pre-market overconfidence.
- **Win-rate / P&L / streaks:** never in the morning brief; that is an anchor that feeds revenge or overconfidence. Performance analysis belongs in the post-hoc reports, kept on a separate surface.
- **Precision beyond 1 point**, raw unfiltered calendar dumps, and multiple pivot/Fib systems: all out.

### Behavioral guards (ADHD, ~1 year experience)

- Lead with the verdict, cap the surface area (one screen, then drill-down).
- The brief only ever REDUCES risk, never excites. Neutral, flat tone; no "big move setting up." It is a pre-flight checklist, not a hype reel.
- Convert events into clock-time no-trade windows (timer-able), to compensate for time-blindness.
- On RED/YELLOW days, a standing one-line reminder: "First loss is information, second loss is your signal to slow down. Daily stop = N." Pre-commit the limit before the emotional state arrives.
- Normalize standing aside: "On a CPI morning, doing nothing until 08:35 is a winning decision."
- Identical layout every day so the one thing that changed (the red event line, the YELLOW badge) pops.

### Config dependency

The Risk Mode badge and the sizing line require Allie's PRE-CONFIGURED per-trade risk and daily-stop values. The brief echoes her numbers, it never invents aggressive ones; if unset, it says so and shows nothing rather than guessing. This maps onto the stop-methodology and sizing open questions already parked in `RESEARCH_PLAN.md` section 8 (items 1-2).

---

## 4. Dependencies and decisions that gate the build

1. **Bar source must include session-tagged Globex data plus settlements.** Resolve `PROJECT_KICKOFF.md` 4.3 with this stricter requirement in mind. This is the single blocking decision.
2. **Timezone and DST discipline.** Everything internal in UTC; sessions defined in America/New_York; CME holiday and half-day calendar applied (same rules already in the project). The 07:00 fire time and the 18:00 / 09:30 / 16:00 ET session boundaries all shift with DST.
3. **"Yesterday" means the prior RTH session,** not the rolling 24h. Depends on the session tag from Feed A.
4. **Calendar and earnings feeds** are net-new data sources not in the current stack.
5. **Scheduling and delivery.** The brief fires unattended at 07:00 ET, so it needs a scheduler plus two delivery paths (email push and a Sheet/dashboard view). This pulls forward the scheduled-run concern that `DEVELOPMENT_PLAN.md` parks at enhancement E18; a small cloud scheduler (or a cron on an always-on host) plus a mail send and a Sheets write covers it.

---

## 5. How it fits the existing project

Mostly reuses existing pieces:
- The `BarSource` protocol (`src/contracts/bar_source.py`) and `src/config.py` session-time constants.
- The UTC / DST / CME-calendar discipline already specified.
- The ADHD-friendly HTML report layer (`src/reports/html.py`) and its color conventions.

New surface area:
- A calendar feed adapter and an earnings feed adapter.
- A 07:00 scheduler and an email delivery path (the Sheet write can reuse the Apps Script / `gspread` path already planned for forward capture).
- A `premarket` module that reads bars, computes levels, and renders the brief. It is a pure read-only consumer; no trade history, no rubric dependency.

Because it is decoupled from the rubric, it is a clean standalone deliverable. It also doubles as a forcing function to resolve the bar-source decision early, which the rubric work needs anyway.

---

## 6. Open questions for Allie / Bryce

1. **Bar source**: RESOLVED -> Databento `GLBX.MDP3` (`ohlcv-1m` + `statistics`, `NQ.c.0`). See `premarket_brief_vendor_research.md`.
2. **VIX exactness**: prior-close VIX is sufficient at 07:00; the VX-futures live proxy is cut (per section 3A). Closed.
3. **Calendar / earnings feed**: FMP free tier recommended; confirm registration and that it fits the tooling budget (`PROJECT_KICKOFF.md` 5.1).
4. **Delivery details**: which email account sends; same Google account as the forward-capture Sheet; one combined morning email or separate sections.
5. **Average overnight range baseline**: trailing how many sessions (proposed: 20 prior sessions, RTH-calendar aware).
6. **Backfill depth** (new, from the Databento decision): how many years of history to seed at first run (recommend 3-5 years for stable ATR/monthly baselines; one-time, small cost).
