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

Candidate sources (from `PROJECT_KICKOFF.md` 4.3), ranked for THIS use:
- **NinjaTrader 8** if Allie's data feed includes ETH: local, free, session-aware. Best fit if she already runs NT for execution.
- **Paid feed (Databento / similar)**: clean session tags and settlements out of the box.
- **TradingView export**: workable but manual and history-limited.
- **RTH-only broker exports**: disqualified for this feature.

Because MNQ and NQ share the underlying, Feed A only needs ONE instrument's bars (use NQ, the more liquid full-size contract, as the reference series; label output for both).

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

1. **Bar source**: does Allie's NinjaTrader feed include ETH (overnight) bars and settlements? If yes, NT is the cheapest path. If not, which paid feed.
2. **VIX exactness**: is prior-close VIX good enough at 07:00, or do we want VX-futures-overnight as a live proxy?
3. **Calendar / earnings feed**: pick a source (API vs maintained list) and confirm cost fits the tooling budget (`PROJECT_KICKOFF.md` 5.1).
4. **Delivery details**: which email account sends; same Google account as the forward-capture Sheet; one combined morning email or separate sections.
5. **Average overnight range baseline**: trailing how many sessions (proposed: 20 prior sessions, RTH-calendar aware).
