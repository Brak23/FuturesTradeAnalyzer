# Pre-Market Brief: Numeric Decision Spec

**Status**: Proposed numeric spec. Fills the build-plan gap where Phase 3 (steps 3.0 - 3.6) NAMES decision logic but does not DEFINE the numbers.
**Audience**: Bryce (developer) and any future Claude Code session building `src/brief/` and `src/config.py`.
**Authority**: Subordinate to `premarket_brief_design.md` (esp. section 3A) and `premarket_brief_architecture.md`. Where this doc adds numbers, it does not contradict those; where the readiness doc (G7/G8/G10d) already named a constant, this doc reuses that exact name.
**Scope**: MNQ and NQ only. One reference series (`NQ.c.0`), labeled for both contracts. All times America/New_York (ET), DST-aware via `zoneinfo`; internal storage UTC.

> Tagging convention used throughout:
> - `[MECHANICAL]`: market-structural or arithmetic; not a judgment call. Safe to hard-code.
> - `[PROPOSED - confirm with Allie]`: a personal-risk or trading-domain default. Coded as written, but flagged for Allie to ratify (consolidated in the OPEN FOR ALLIE list at the end).
> - `[PROPOSED - confirm with Bryce]`: an engineering/operational default Bryce owns.
>
> No value is left blank. There is no "TBD" anywhere. Every default is concrete and codeable today.

---

## 0. Constants table (the config.py contract)

Every named constant this spec defines, with proposed value, unit, tag, and the decision it governs. This table is the authoritative source; the prose sections below explain and justify each. Constants marked "(readiness)" already appear in `premarket_brief_readiness.md` and are reused verbatim.

| Constant | Value | Unit | Tag | Governs |
|---|---|---|---|---|
| `VIX_ELEVATED_THRESHOLD` | `20.0` | VIX index points | `[PROPOSED - confirm with Allie]` | VIX regime LOW vs ELEVATED cutoff (item 1) |
| `VIX_HIGH_THRESHOLD` | `28.0` | VIX index points | `[PROPOSED - confirm with Allie]` | VIX regime ELEVATED vs HIGH cutoff (item 1) |
| `VIX_FALLBACK_REGIME` | `"LOW"` | enum | `[MECHANICAL]` | Regime used when VIX stale/missing (item 1) |
| `MAX_VIX_STALENESS_DAYS` | `3` | calendar days | `[MECHANICAL]` (readiness G10d) | VIX prior-close age before "VIX unavailable" |
| `GAP_ATR_QUIET_MAX` | `0.20` | ATR fraction | `[MECHANICAL]` | Upper edge of QUIET gap bucket (item 1, 2) |
| `GAP_ATR_ELEVATED_MAX` | `1.00` | ATR fraction | `[MECHANICAL]` | Upper edge of NORMAL gap bucket; above = LARGE (item 1, 2) |
| `GAP_ATR_GRAY_MAX` | `0.20` | ATR fraction | `[MECHANICAL]` | At/under this, headline renders GRAY (inconclusive) (item 2) |
| `ATR_LOOKBACK` | `14` | RTH daily bars | `[MECHANICAL]` | ATR period (item 2) |
| `AVG_ONR_LOOKBACK` | `20` | sessions | `[PROPOSED - confirm with Allie]` | Trailing avg overnight-range window (item 2) |
| `BASELINE_SESSION_COUNT` | `20` | complete sessions | `[PROPOSED - confirm with Allie]` (readiness G7) | Alias/driver of `AVG_ONR_LOOKBACK`; excludes half/holiday (item 2) |
| `ON_RANGE_QUIET_MULTIPLIER` | `0.70` | x avg ONR | `[PROPOSED - confirm with Allie]` (readiness G7) | Below this = "quiet night" (item 2) |
| `ON_RANGE_ELEVATED_MULTIPLIER` | `1.50` | x avg ONR | `[PROPOSED - confirm with Allie]` (readiness G7) | At/above this = "big night" (item 2) |
| `NO_TRADE_PAD_BEFORE_HIGH` | `2` | minutes | `[PROPOSED - confirm with Allie]` | Minutes before a HIGH event the window opens (item 3) |
| `NO_TRADE_PAD_AFTER_HIGH` | `5` | minutes | `[PROPOSED - confirm with Allie]` | Minutes after a HIGH event the window stays open (item 3) |
| `NO_TRADE_PAD_BEFORE_MED` | `1` | minutes | `[PROPOSED - confirm with Allie]` | Minutes before a MED event (item 3) |
| `NO_TRADE_PAD_AFTER_MED` | `3` | minutes | `[PROPOSED - confirm with Allie]` | Minutes after a MED event (item 3) |
| `FOMC_PAD_BEFORE` | `5` | minutes | `[PROPOSED - confirm with Allie]` | Minutes before FOMC statement (14:00 ET) (item 3) |
| `FOMC_PAD_AFTER_PRESSER` | `30` | minutes | `[PROPOSED - confirm with Allie]` | Minutes after the presser start (14:30 ET) (item 3) |
| `CATALYST_MAX_SHOWN` | `3` | events | `[MECHANICAL]` (design 3A) | Cap on catalyst banner lines (item 3) |
| `SESSION_END_ET` | `"16:00"` | ET clock | `[PROPOSED - confirm with Allie]` | End of Allie's relevant window for catalyst inclusion (item 3) |
| `ONCLV_TRENDED_MIN` | `0.80` | fraction [0,1] | `[PROPOSED - confirm with Allie]` | Close-location at/above (or at/below 1-this) = directional close (item 4) |
| `ONCLV_BALANCED_LOW` | `0.40` | fraction [0,1] | `[PROPOSED - confirm with Allie]` | Lower edge of BALANCED close-location band (item 4) |
| `ONCLV_BALANCED_HIGH` | `0.60` | fraction [0,1] | `[PROPOSED - confirm with Allie]` | Upper edge of BALANCED close-location band (item 4) |
| `ON_NET_TREND_MIN_ATR` | `0.50` | ATR fraction | `[PROPOSED - confirm with Allie]` | Net-change magnitude needed to call the night TRENDED (item 4) |
| `ON_EFFICIENCY_TREND_MIN` | `0.50` | ratio [0,1] | `[PROPOSED - confirm with Allie]` | Net-move / range path-efficiency for TRENDED (item 4) |
| `PER_TRADE_RISK_USD` | `unset` (None) | USD | `[PROPOSED - confirm with Allie]` | Allie's dollar risk per trade; sizing hidden if unset (item 5) |
| `DAILY_STOP_USD` | `unset` (None) | USD | `[PROPOSED - confirm with Allie]` | Allie's daily stop; "done after N stop-outs" hidden if unset (item 5) |
| `STOP_ATR_MULT` | `0.50` | x daily ATR | `[PROPOSED - confirm with Allie]` | Default stop distance when Allie has no fixed stop (item 5) |
| `MNQ_POINT_VALUE` | `2.0` | USD/point | `[MECHANICAL]` | MNQ dollar value per index point (item 5) |
| `NQ_POINT_VALUE` | `20.0` | USD/point | `[MECHANICAL]` | NQ dollar value per index point (item 5) |
| `MAX_CONTRACTS` | `20` | contracts | `[PROPOSED - confirm with Allie]` | Advisory sizing safety cap (item 5) |
| `LEVEL_PROXIMITY_ATR` | `1.0` | ATR fraction | `[PROPOSED - confirm with Allie]` | Show weekly/monthly H/L only within this many ATR (item 6) |
| `ROLL_CARRY_LEVELS` | `True` | bool | `[MECHANICAL]` | On roll, carry prior levels (no false gap) (item 6) |
| `MIN_OVERNIGHT_BARS` | `300` | 1-min bars | `[PROPOSED - confirm with Bryce]` | Below this overnight bar count = degraded GRAY (item 6) |
| `MIN_RTH_BARS_PRIOR` | `360` | 1-min bars | `[PROPOSED - confirm with Bryce]` | Below this prior-RTH bar count = degraded GRAY (item 6) |
| `MAX_BAR_STALENESS_MIN` | `45` | minutes | `[PROPOSED - confirm with Bryce]` | Newest bar older than this at 06:30 = stale -> degraded (item 6) |
| `SETTLEMENT_SANITY_PCT` | `0.005` | fraction | `[MECHANICAL]` | Settle must be within 0.5% of price or degrade (item 6) |
| `GAP_ATR_SANITY_ABS` | `3.0` | ATR fraction | `[MECHANICAL]` | abs(gap-as-ATR) above this = degrade (item 6) |
| `HALF_DAY_RTH_BARS` | `210` | 1-min bars | `[MECHANICAL]` | Expected RTH bar count for a half-day (13:00 ET close, ~210 of 390 full-session bars); half-day floor for MIN_RTH_BARS_PRIOR check = `HALF_DAY_RTH_BARS * 0.92` (item 6) |

Count: 38 named constants in this table. 12 `[MECHANICAL]`, 23 `[PROPOSED - confirm with Allie]`, 3 `[PROPOSED - confirm with Bryce]`. (`AVG_ONR_LOOKBACK` and `BASELINE_SESSION_COUNT` are the same underlying value via the alias rule below, as are the `ON_RANGE_*_MULT` short forms; counting the aliases as distinct names because each appears in code.)

> Note on the two G7 names: `premarket_brief_readiness.md` named these `ON_RANGE_QUIET_MULTIPLIER` / `ON_RANGE_ELEVATED_MULTIPLIER` and `BASELINE_SESSION_COUNT`. The original Phase-3.0 task text used the shorter `ON_RANGE_QUIET_MULT` / `AVG_ONR_LOOKBACK`. To avoid two sources of truth, `config.py` declares the readiness names as canonical and the short names as module-level aliases (`AVG_ONR_LOOKBACK = BASELINE_SESSION_COUNT`, `ON_RANGE_QUIET_MULT = ON_RANGE_QUIET_MULTIPLIER`). Tests assert the aliases are identical.

---

## 1. Risk Mode badge decision table (Phase 3.0 / 3.5, G8)

### 1.1 Inputs and their buckets

The badge is a pure function of three bucketed inputs:

```
risk_mode = RISK_MODE_TABLE[(catalyst_severity, gap_atr_bucket, vix_regime)]
```

**catalyst_severity** in `{NONE, MED, HIGH}`: the single highest severity among today's qualifying catalysts that fall in or before Allie's session window (see item 3 for the mapping). NONE = no MED or HIGH event in window.

**gap_atr_bucket** in `{QUIET, NORMAL, LARGE}`, from `abs(gap_pts) / atr_14`:
- `QUIET`: `gap_atr <= GAP_ATR_QUIET_MAX` (<= 0.20)
- `NORMAL`: `GAP_ATR_QUIET_MAX < gap_atr <= GAP_ATR_ELEVATED_MAX` (0.20 to 1.00)
- `LARGE`: `gap_atr > GAP_ATR_ELEVATED_MAX` (> 1.00)

**vix_regime** in `{LOW, ELEVATED, HIGH}`, from the prior VIX close:
- `LOW`: `vix < VIX_ELEVATED_THRESHOLD` (< 20.0)
- `ELEVATED`: `VIX_ELEVATED_THRESHOLD <= vix < VIX_HIGH_THRESHOLD` (20.0 to 28.0)
- `HIGH`: `vix >= VIX_HIGH_THRESHOLD` (>= 28.0)

**VIX cutoff justification** `[PROPOSED - confirm with Allie]`: the VIX spends the large majority of its time in the low-to-high teens; a sustained reading above ~20 is the conventional line between a calm tape and an anxious one, and readings at/above the high 20s mark genuinely stressed regimes (corrections, shocks) where intraday ranges roughly double. 20 and 28 are round, defensible cuts that match how desks talk about "VIX over 20" and "VIX near 30." These are the two values most likely to be tuned after Allie sees a few weeks of briefs.

### 1.2 Catalyst is the dominant axis (tone rule)

Per design 3A, the badge does the synthesis so a one-year trader does not have to. The hierarchy, in plain terms:

- A **HIGH** timed catalyst in/before her session is the single most dangerous thing in the brief: stops do not protect through the print. It forces at least YELLOW and, in any non-quiet tape, RED. This is the only axis that can produce RED on its own.
- **Volatility (gap + VIX)** modulates: a big gap or a stressed VIX pushes toward YELLOW even with no event, because the day is structurally wider than her plan assumes.
- The **tone is calming, never exciting** (design 3A behavioral guards). The table only ever escalates risk for a concrete reason and biases toward the calmer color on ties. There is no cell that says RED for "big move setting up." RED means "a known hazard is on the clock," nothing more.

### 1.3 The full 27-cell table

Rows are grouped by catalyst severity. Read as `(catalyst, gap_atr_bucket, vix_regime) -> mode`.

| catalyst | gap_atr | VIX LOW | VIX ELEVATED | VIX HIGH |
|---|---|---|---|---|
| **NONE** | QUIET   | GREEN  | GREEN  | YELLOW |
| **NONE** | NORMAL  | GREEN  | YELLOW | YELLOW |
| **NONE** | LARGE   | YELLOW | YELLOW | RED    |
| **MED**  | QUIET   | GREEN  | YELLOW | YELLOW |
| **MED**  | NORMAL  | YELLOW | YELLOW | RED    |
| **MED**  | LARGE   | YELLOW | RED    | RED    |
| **HIGH** | QUIET   | YELLOW | YELLOW | RED    |
| **HIGH** | NORMAL  | RED    | RED    | RED    |
| **HIGH** | LARGE   | RED    | RED    | RED    |

Reading the logic:
- **HIGH catalyst** is never GREEN. It is RED whenever the tape is not quiet, and even on a quiet-gap morning it is at least YELLOW (because the event has not happened yet; the quiet gap is pre-print noise, not safety).
- **NONE catalyst** is the only group that is GREEN across most cells. It only reaches RED in the single corner where the overnight gap is LARGE (> 1 ATR) AND VIX is HIGH (>= 28): a structurally violent day with no scheduled anchor, which still warrants standing down.
- **MED catalyst** sits between: never RED on a quiet tape, RED once both volatility axes lean elevated/high.

This maps directly to a dict literal in `config.py`:

```
RISK_MODE_TABLE = {
    ("NONE","QUIET","LOW"):"GREEN",  ("NONE","QUIET","ELEVATED"):"GREEN",  ("NONE","QUIET","HIGH"):"YELLOW",
    ("NONE","NORMAL","LOW"):"GREEN", ("NONE","NORMAL","ELEVATED"):"YELLOW",("NONE","NORMAL","HIGH"):"YELLOW",
    ("NONE","LARGE","LOW"):"YELLOW", ("NONE","LARGE","ELEVATED"):"YELLOW", ("NONE","LARGE","HIGH"):"RED",
    ("MED","QUIET","LOW"):"GREEN",   ("MED","QUIET","ELEVATED"):"YELLOW",  ("MED","QUIET","HIGH"):"YELLOW",
    ("MED","NORMAL","LOW"):"YELLOW", ("MED","NORMAL","ELEVATED"):"YELLOW", ("MED","NORMAL","HIGH"):"RED",
    ("MED","LARGE","LOW"):"YELLOW",  ("MED","LARGE","ELEVATED"):"RED",     ("MED","LARGE","HIGH"):"RED",
    ("HIGH","QUIET","LOW"):"YELLOW", ("HIGH","QUIET","ELEVATED"):"YELLOW", ("HIGH","QUIET","HIGH"):"RED",
    ("HIGH","NORMAL","LOW"):"RED",   ("HIGH","NORMAL","ELEVATED"):"RED",   ("HIGH","NORMAL","HIGH"):"RED",
    ("HIGH","LARGE","LOW"):"RED",    ("HIGH","LARGE","ELEVATED"):"RED",    ("HIGH","LARGE","HIGH"):"RED",
}
```

`test_risk_mode.py` is parametrized over all 27 keys against this table.

### 1.4 Fallback when an input is missing

- **VIX stale or missing** (older than `MAX_VIX_STALENESS_DAYS` = 3, or feed failed): treat `vix_regime = VIX_FALLBACK_REGIME = "LOW"` for table lookup, and the volatility line prints "VIX unavailable." Rationale: VIX sits behind ATR, not beside it (design 3A); the gap-as-ATR axis already captures the realized overnight move, so defaulting VIX to the calmest regime avoids a false escalation from a data outage. This is the deliberately conservative-against-false-alarm choice; it never relaxes the catalyst axis, which is where real danger lives.
- **Gap or ATR missing** (settlement missing, or RTH bar count below floor): the brief is in the degraded/GRAY state (item 6); the badge renders GRAY with banner "Data incomplete, verify independently" and the table is not consulted. GRAY is a fourth, non-table state that supersedes the 27 cells.
- **Catalyst feed (FMP) empty or failed**: treat `catalyst_severity = NONE` for the table, but the catalyst banner says "Calendar unavailable, check independently" rather than "No major events." A feed failure must not read as an all-clear.

### 1.5 Worked examples

1. **Boring Wednesday.** No CPI/FOMC/mega-cap (`NONE`). Gap +78 pts, ATR 210 -> `78/210 = 0.37` -> `NORMAL`. VIX 16.2 -> `LOW`. Lookup `("NONE","NORMAL","LOW")` -> **GREEN**. Badge: "TRADE NORMAL SIZE."
2. **CPI morning.** 08:30 CPI is `HIGH`. Gap -40 pts, ATR 210 -> `0.19` -> `QUIET` (pre-print calm). VIX 18 -> `LOW`. Lookup `("HIGH","QUIET","LOW")` -> **YELLOW**, with the catalyst banner driving the no-trade window. Quiet gap does not buy GREEN because the number has not hit.
3. **Stress day, no event.** `NONE`. Gap +520 pts, ATR 240 -> `2.17` -> `LARGE`. VIX 31 -> `HIGH`. Lookup `("NONE","LARGE","HIGH")` -> **RED**. Badge: "STAND ASIDE OR TRADE TINY." The only NONE-catalyst RED corner.

### 1.6 Display strings (per mode)

- GREEN: `"TRADE NORMAL SIZE"`
- YELLOW: `"TRADE SMALLER / BE SELECTIVE"`
- RED: `"STAND ASIDE OR TRADE TINY"`
- GRAY: `"DATA INCOMPLETE - VERIFY INDEPENDENTLY"`

(Exact casing/glyph per the UX spec; this defines the text content the engine must emit.)

---

## 2. Volatility qualifier and ATR (Phase 3.3, G1)

### 2.1 ATR definition

`ATR_LOOKBACK = 14` `[MECHANICAL]`. ATR is computed on **RTH-only daily bars** (09:30-16:00 ET sessions), NOT on the 24h Globex series. This is the G1 silent-wrong trap: `ohlcv-1m` spans the full overnight, so a naive daily resample would inflate the true range. `volatility.py` must aggregate 1-min RTH bars into a daily RTH OHLC first, then run Wilder's ATR(14) over those daily bars.

True range per RTH day uses the prior RTH close:
```
tr_t = max(high_t - low_t, abs(high_t - prior_rth_close), abs(low_t - prior_rth_close))
atr_14 = Wilder_smoothed_mean(tr, 14)
```

### 2.2 Gap as a fraction of ATR

The gap is measured against prior **settlement** (not last trade), per design and architecture:
```
gap_pts      = current_price - prior_settle          # signed
gap_pct      = gap_pts / prior_settle
gap_atr_frac = abs(gap_pts) / atr_14                 # unsigned magnitude for bucketing
```
`current_price` is the last 1-min bar close at the 06:30 ingest. Sign is preserved separately for the green/red coloring of the headline; the bucket and the badge use the magnitude.

### 2.3 GRAY (inconclusive) threshold

`GAP_ATR_GRAY_MAX = 0.20` `[MECHANICAL]` (design 3A: "render GRAY when the gap is under ~0.2 ATR"). If `gap_atr_frac <= GAP_ATR_GRAY_MAX`, the **headline line color** is GRAY (a non-event should not read like an event). Note this is the same edge value as `GAP_ATR_QUIET_MAX`; they are intentionally equal but separately named because one governs headline COLOR and the other governs the badge BUCKET, and they could diverge later.

### 2.4 Headline qualifier bands (exact words)

The qualifier baked into the headline maps `gap_atr_frac` to one phrase:

| Band | Condition | Qualifier words |
|---|---|---|
| inconclusive | `gap_atr_frac <= 0.20` | `"flat (gap under 0.2x ATR, treat as noise)"` |
| quiet | `0.20 < gap_atr_frac <= 0.50` | `"on a quiet night (gap = {x}x ATR)"` |
| normal | `0.50 < gap_atr_frac <= 1.00` | `"a normal-size gap ({x}x ATR)"` |
| large | `gap_atr_frac > 1.00` | `"a large overnight move ({x}x ATR)"` |

`{x}` is `gap_atr_frac` rounded to 1 decimal. The 0.50 internal split (quiet vs normal) is `[PROPOSED - confirm with Allie]`; the 0.20 and 1.00 edges are mechanical (they equal the badge bucket edges). Example full headline: `"NQ +78 pts (+0.4%), gap up, just under yesterday's high, on a quiet night (gap = 0.4x ATR)."`

### 2.5 Overnight range vs average ("big night / quiet night")

Computed in `overnight.py`, distinct from the gap qualifier (the gap is the open-vs-settle jump; this is how wide the night ranged).

```
on_range_pts   = onh - onl
avg_onr        = mean of on_range_pts over the last BASELINE_SESSION_COUNT complete sessions
on_range_ratio = on_range_pts / avg_onr
```

`BASELINE_SESSION_COUNT = 20` `[PROPOSED - confirm with Allie]`, counting only complete non-half, non-holiday sessions (the half/holiday exclusion is the G6 dependency). `AVG_ONR_LOOKBACK` is an alias of this constant.

Bands:

| Condition | Label words |
|---|---|
| `on_range_ratio < ON_RANGE_QUIET_MULTIPLIER` (< 0.70) | `"a quiet night ({r}x normal range)"` |
| `0.70 <= on_range_ratio < ON_RANGE_ELEVATED_MULTIPLIER` (0.70 to 1.50) | `"a normal-range night"` |
| `on_range_ratio >= ON_RANGE_ELEVATED_MULTIPLIER` (>= 1.50) | `"a big night ({r}x normal range)"` |

`{r}` rounded to 1 decimal. Multipliers `[PROPOSED - confirm with Allie]` (readiness G7): 0.70 and 1.50 are symmetric-ish round bands around 1.0x that avoid flapping on near-average nights.

### 2.6 VIX expected daily move

`expected_move_pct = vix / 16.0` `[MECHANICAL]` (design: VIX/~16 approximates the one-day percent move; 16 = sqrt(252) rounded). Displayed as e.g. `"VIX 16.2 ~1.0% move"`. If VIX stale, this line reads `"VIX unavailable"`.

### 2.7 Worked examples

1. ATR 210, gap +78 -> `0.37x` -> quiet band -> "on a quiet night (gap = 0.4x ATR)." ONH 20,460 ONL 20,255 -> range 205; avg_onr 230 -> ratio 0.89 -> "a normal-range night."
2. ATR 240, gap +520 -> `2.17x` -> large band -> "a large overnight move (2.2x ATR)." Range 600, avg 250 -> ratio 2.4 -> "a big night (2.4x normal range)."
3. ATR 200, gap +30 -> `0.15x` -> inconclusive -> headline GRAY, "flat (gap under 0.2x ATR, treat as noise)."

---

## 3. Catalyst no-trade windows (Phase 3.4, G2)

### 3.1 Severity classification (lookup + override)

FMP provides an `impact` field (`Low`/`Medium`/`High`). The engine uses a two-step rule:

1. **Override table first** (index-futures-specific; keyword match on the normalized event name, case-insensitive substring). This wins over FMP's own field because FMP's generic impact rating undersells events that specifically move NQ.
2. **Fall back to FMP `impact`** for anything not in the override table: FMP `High` -> `HIGH`, `Medium` -> `MED`, `Low` or missing -> ignored.

Override table (`CATALYST_SEVERITY_OVERRIDES` in `config.py`):

| Keyword (substring, lower-cased) | Severity | Note |
|---|---|---|
| `cpi`, `consumer price` | `HIGH` | 08:30 ET, top index mover |
| `pce` | `HIGH` | Fed's preferred inflation gauge |
| `nonfarm`, `nfp`, `employment situation` | `HIGH` | 08:30 ET jobs report |
| `fomc`, `federal funds`, `rate decision`, `interest rate decision` | `HIGH` | 14:00 ET statement (see 3.3) |
| `ppi`, `producer price` | `HIGH` | 08:30 ET |
| `retail sales` | `HIGH` | 08:30 ET |
| `jobless claims`, `initial claims`, `continuing claims` | `MED` | 08:30 ET, weekly, lower surprise impact |
| `ism`, `pmi` | `MED` | 10:00 ET data |
| `consumer confidence`, `consumer sentiment`, `umich` | `MED` | 10:00 ET data |
| `jolts`, `job openings` | `MED` | 10:00 ET data |
| `gdp` | `MED` | revisions often pre-leaked |
| `powell`, `fed chair`, `fed speak`, `fed president`, `fomc member` | `MED` | Fed speakers (see 3.3 for Chair-presser exception) |
| any name containing `holiday`, `auction`, `bill`, `bond` (Treasury auctions) | ignored | not an index-futures intraday driver |

Anything else: use the FMP `impact` fallback.

### 3.2 Padding per severity (the no-trade window math)

A qualifying event at clock time `T` produces a window `[T - pad_before, T + pad_after]` in ET:

| Severity | Before constant | After constant | Window for an 08:30 print |
|---|---|---|---|
| HIGH | `NO_TRADE_PAD_BEFORE_HIGH = 2` | `NO_TRADE_PAD_AFTER_HIGH = 5` | `08:28 - 08:35` |
| MED | `NO_TRADE_PAD_BEFORE_MED = 1` | `NO_TRADE_PAD_AFTER_MED = 3` | `08:29 - 08:33` |

This reconciles exactly to the design's worked example "Avoid 08:28-08:35 (CPI)" (2 before, 5 after) `[PROPOSED - confirm with Allie]`. The asymmetry (more padding after than before) is deliberate: the violent move and the whipsaw are post-print.

### 3.3 FOMC special case (statement + presser)

FOMC is two timed hazards, not one. They are rendered as a single combined HIGH window spanning from just before the statement to well after the presser begins:
```
window = [14:00 - FOMC_PAD_BEFORE, 14:30 + FOMC_PAD_AFTER_PRESSER]  ->  13:55 - 15:00 ET
```
`FOMC_PAD_BEFORE = 5`, `FOMC_PAD_AFTER_PRESSER = 30` `[PROPOSED - confirm with Allie]`. Shown only if Allie's session runs that late (see 3.5). The **Fed Chair press conference** specifically is HIGH (it overrides the generic `MED` Fed-speaker rule via the `fomc`/`rate decision` match collapsing statement+presser into one HIGH window); ordinary Fed-speaker appearances stay MED.

### 3.4 Cap-at-3 tie-break (CATALYST_MAX_SHOWN)

`CATALYST_MAX_SHOWN = 3` `[MECHANICAL]` (design 3A). When more than 3 qualifying events exist, select the 3 shown by this exact priority:

1. **Severity descending**: all HIGH before any MED.
2. **Within the same severity, chronological ascending** (earliest ET first), because the earliest hazard is the one she hits first and must time-block for.
3. Ties broken by alphabetical event name (deterministic, for idempotency, G5).

The banner then renders the chosen 3 in **chronological order** (not severity order) so the timeline reads top-to-bottom as her morning unfolds. If events were dropped by the cap, append a muted line `"+N more lower-priority events (see drill-down)"`.

### 3.5 Session-window filter

Only events with `T <= SESSION_END_ET` are eligible (`SESSION_END_ET = "16:00"` `[PROPOSED - confirm with Allie]`; if Allie quits at noon, lower this and the FOMC/afternoon events drop out). Events before 06:30 that have already printed are excluded from the no-trade banner but their result may still color the overnight story. The catalyst severity feeding the Risk Mode badge (item 1) uses this same in-window set.

### 3.6 Mega-cap earnings folding

Pre-market mega-cap earnings (an `NDX_COMPONENTS` name reporting `bmo`, or `amc` from the prior close) are folded into the SAME catalyst list and COUNT AGAINST the cap of 3:
- A mega-cap (AAPL, MSFT, NVDA, AMZN, GOOGL, META and peers) reporting `bmo` today or `amc` yesterday -> severity `MED`, pseudo-time = `09:30` (the reaction lands at the open) unless FMP gives a specific time.
- They are NOT a timed no-trade window the way an 08:30 print is; instead the line reads `"NVDA earnings reaction at the open - expect a wider, faster open"` rather than a clock window.
- Because they share the cap, a morning with CPI + FOMC + 2 mega-caps shows CPI (HIGH), FOMC (HIGH), then the earlier-severity-then-chronological winner of the earnings, and appends "+1 more."

### 3.7 Empty / failed feed

- FMP returns zero events for today AND the call succeeded: banner says `"No major events, normal session"` (design 3A: saying so is decision-useful). Severity NONE.
- FMP call FAILED: banner says `"Calendar unavailable, check independently"`. Severity treated NONE for the badge but this is NOT an all-clear (item 1.4).

### 3.8 Worked examples

1. **CPI day.** FMP: "CPI" 08:30 High. Override -> HIGH. Window `[08:30-2, 08:30+5]` = `08:28 - 08:35`. Banner: `"Avoid 08:28-08:35 (CPI, high impact). Do not hold through the print."`
2. **Quad-event morning.** CPI 08:30 (HIGH), Jobless Claims 08:30 (MED), ISM 10:00 (MED), NVDA bmo (MED, 09:30). Cap 3, priority: HIGH first -> CPI; then MED chronological -> Jobless Claims 08:30, then NVDA 09:30. ISM 10:00 dropped. Rendered chronologically: CPI 08:28-08:35, Jobless 08:29-08:33, NVDA "reaction at the open"; append "+1 more lower-priority event."
3. **FOMC day.** "FOMC Rate Decision" 14:00. Combined HIGH window `13:55 - 15:00`. Shown because 15:00 <= 16:00 session end.

---

## 4. Overnight trend vs chop (Phase 3.2, currently undefined)

### 4.1 The no-direction guardrail

The repo-wide hard rule (design 3A, parent CLAUDE.md anti-goals): the brief reports STATE, never a lean. This classifier therefore outputs a **non-directional shape label only**. It is allowed to say the night TRENDED; it is forbidden to say it trended UP. It may show net-change magnitude and range (numbers Allie can read the sign off herself if she wants), but the engine never emits "bullish," "uptrend," "bias," or any directional adjective.

### 4.2 The computation (all from overnight 1-min bars, 18:00 prior ET to 06:30 ET)

```
on_open   = first overnight bar open
on_close  = last overnight bar close (at ingest)
onh, onl  = overnight high, low
on_range  = onh - onl
net_change = on_close - on_open                       # signed, but only its MAGNITUDE is used for labeling
net_abs    = abs(net_change)

# Close-location value: where the session closed within its range, 0=at low, 1=at high
onclv = (on_close - onl) / on_range                   # guard on_range > 0

# Net magnitude as a fraction of daily ATR
net_atr = net_abs / atr_14

# Path efficiency: how much of the distance traveled was net directional travel.
# sum_abs_move = sum of abs(close_t - close_{t-1}) over overnight bars (the path length)
efficiency = net_abs / sum_abs_move                   # 0 = pure round-trip chop, 1 = straight line
```

`onclv` and `efficiency` are the two non-directional shape measures; `net_atr` gates whether the move is big enough to be worth a "trended" label at all.

### 4.3 Classification (3 non-directional labels)

```
if net_atr >= ON_NET_TREND_MIN_ATR
   and efficiency >= ON_EFFICIENCY_TREND_MIN
   and (onclv >= ONCLV_TRENDED_MIN or onclv <= 1 - ONCLV_TRENDED_MIN):
       label = "TRENDED"          # directional travel, closed near an extreme, took a fairly straight path
elif ONCLV_BALANCED_LOW <= onclv <= ONCLV_BALANCED_HIGH and net_atr < ON_NET_TREND_MIN_ATR:
       label = "BALANCED"         # closed mid-range with little net change: two-sided, rotational
else:
       label = "CHOPPY"           # moved around but did not resolve: low efficiency or mid close with motion
```

Constants `[PROPOSED - confirm with Allie]`: `ON_NET_TREND_MIN_ATR = 0.50`, `ON_EFFICIENCY_TREND_MIN = 0.50`, `ONCLV_TRENDED_MIN = 0.80`, `ONCLV_BALANCED_LOW = 0.40`, `ONCLV_BALANCED_HIGH = 0.60`.

### 4.4 What is SHOWN vs WITHHELD

**Shown** (non-directional): the label word, the overnight range in points, and net-change MAGNITUDE in points (unsigned in the trend phrase, though the price ladder and headline already carry the signed gap so Allie is not blind to direction; the point is the trend CLASSIFIER never editorializes a direction).

**Withheld** (never emitted by `overnight.py`): "up"/"down"/"higher"/"lower" attached to the trend label, any "continuation"/"reversal" framing, any "bias," any probability-of-follow-through. The phrase template enforces this:

| Label | Phrase (exact) |
|---|---|
| TRENDED | `"Overnight trended (moved {net_abs} pts, {r}x normal range)."` |
| BALANCED | `"Overnight balanced and two-sided ({r}x normal range)."` |
| CHOPPY | `"Overnight chopped, no clean resolution ({r}x normal range)."` |

`{net_abs}` is the unsigned point magnitude; `{r}` is `on_range_ratio` from item 2.5. The signed direction lives only in the headline gap and the ladder, which are observable-state, not a lean.

### 4.5 Fallback

If `on_range == 0` or overnight bar count `< MIN_OVERNIGHT_BARS` (item 6): label is omitted and the overnight story reads `"Overnight data incomplete."` The brief is in degraded state.

### 4.6 Worked examples

1. on_open 20,340, on_close 20,455, onl 20,335, onh 20,460 -> range 125, net +115, net_abs 115. ATR 210 -> net_atr 0.55 (>=0.50). onclv = (20455-20335)/125 = 0.96 (>=0.80). Suppose sum_abs_move 180 -> efficiency 0.64 (>=0.50). -> **TRENDED**. Phrase: "Overnight trended (moved 115 pts, 0.5x normal range)." No direction stated.
2. on_open 20,400, on_close 20,405, onl 20,330, onh 20,470 -> range 140, net_abs 5, net_atr 0.02. onclv = (20405-20330)/140 = 0.54 (in 0.40-0.60). -> **BALANCED**. Phrase: "Overnight balanced and two-sided (0.6x normal range)."
3. on_open 20,400, on_close 20,300, range 260, net_abs 100, net_atr 0.48 (<0.50), sum_abs_move 520 -> efficiency 0.19. onclv 0.20 (not mid, but fails trend gates). -> **CHOPPY**. Phrase: "Overnight chopped, no clean resolution (1.1x normal range)."

---

## 5. Position sizing (dollar-risk-constant) (Phase 3.5)

### 5.1 The formula

```
stop_distance_points = Allie's fixed stop if she has one, else STOP_ATR_MULT * atr_14
risk_per_contract    = stop_distance_points * point_value          # point_value per instrument
contracts            = floor(PER_TRADE_RISK_USD / risk_per_contract)
contracts            = min(contracts, MAX_CONTRACTS)
```

`point_value`: `MNQ_POINT_VALUE = 2.0` USD/pt, `NQ_POINT_VALUE = 20.0` USD/pt `[MECHANICAL]`. The line shows BOTH instruments' contract counts since the levels are shared.

`STOP_ATR_MULT = 0.50` `[PROPOSED - confirm with Allie]`: a default stop of half the daily ATR is a reasonable intraday swing stop for MNQ/NQ. If Allie has a fixed stop methodology (e.g. a fixed 30-point stop, or a structure stop), that value replaces the ATR-derived one and `STOP_ATR_MULT` is unused. The line states which basis was used.

### 5.2 Where Allie's inputs live and the unset rule

`PER_TRADE_RISK_USD` and `DAILY_STOP_USD` live in `config.py` (analytical/personal constants), defaulting to `None` (unset). **Hard rule** (design 3A config dependency): if `PER_TRADE_RISK_USD` is unset, the sizing line shows NOTHING computed; it shows `"Set your per-trade risk to see suggested sizing."` The brief never invents an aggressive dollar number. Same for the daily-stop reminder if `DAILY_STOP_USD` is unset.

The "done after N stop-outs" reminder:
```
n_stop_outs = floor(DAILY_STOP_USD / PER_TRADE_RISK_USD)     # only if both set
```
Rendered (RED/YELLOW days, design 3A): `"First loss is information, second loss is your signal to slow down. You are done after about {n_stop_outs} full stop-outs."`

### 5.3 Safety cap and advisory framing

`MAX_CONTRACTS = 20` `[PROPOSED - confirm with Allie]` caps the suggested count regardless of math (guards against a tiny-stop day suggesting an enormous size). The line is **advisory only** and labeled as such: `"Suggested (advisory): ..."`. It is not an instruction and never auto-sizes anything.

### 5.4 Display string

```
"High-vol day: wider stop, fewer contracts to hold your dollar risk fixed.
 At ${PER_TRADE_RISK_USD}/trade and a {stop_distance_points}-pt stop:
 ~{nq_contracts} NQ or ~{mnq_contracts} MNQ (advisory)."
```

### 5.5 Worked examples (PER_TRADE_RISK_USD = 300 for illustration)

1. ATR 210, no fixed stop -> stop = `0.50 * 210 = 105` pts. NQ risk/contract = `105 * 20 = 2100`; `floor(300/2100) = 0` -> shows `0 NQ` but `floor(300 / (105*2)) = floor(1.43) = 1 MNQ`. Line: "~0 NQ or ~1 MNQ (advisory)" plus a note that NQ risk exceeds her per-trade budget at this stop.
2. ATR 120, fixed 30-pt stop -> NQ risk/contract `30*20=600`; `floor(300/600)=0` NQ; MNQ `floor(300/60)=5` MNQ. Line: "~0 NQ or ~5 MNQ."
3. `PER_TRADE_RISK_USD` unset -> "Set your per-trade risk to see suggested sizing." No numbers.

---

## 6. Price-level edge cases (Phase 3.1 / 3.6, G2/G3)

### 6.1 Continuous-contract roll behavior

`ROLL_CARRY_LEVELS = True` `[MECHANICAL]`. On a roll (front month switches, detected by the resolved `contract` stamp changing or by open-interest crossover from the `statistics` schema), the **prior-day, weekly, and monthly H/L levels CARRY OVER as the values that were true on the continuous series; they do NOT reset to the new contract's own history.** The continuous symbol `NQ.c.0` already stitches the series; the gap is always measured `current_price - prior_settle` where both are on the same continuous series. Rationale: a roll introduces a price-level discontinuity (the new contract trades at a different absolute price due to cost-of-carry). If levels reset to the raw new-contract history, the brief would print a false multi-hundred-point "gap" on roll day. The fix: levels and the gap are always computed within the un-adjusted continuous series Databento returns, and on roll day the brief annotates `"Contract roll today ({old} -> {new}); levels are continuous-series."` so the carry is visible, not silent. This matches architecture section 0 cascade item 2 and FuturesTradeAnalyzer ADR-0011 (un-adjusted, session-anchored).

### 6.2 Weekly/monthly proximity suppression

`LEVEL_PROXIMITY_ATR = 1.0` `[PROPOSED - confirm with Allie]` (design cut list: "show ONLY when current price is within ~1 ATR"). A weekly or monthly H/L level is displayed only if:
```
abs(current_price - level) <= LEVEL_PROXIMITY_ATR * atr_14
```
Otherwise it is suppressed from the ladder entirely (most days, correctly, they will not appear). PDH/PDL/ONH/ONL/ON-VWAP are ALWAYS shown (they are intraday-relevant every day); only weekly/monthly are conditional. Worked example: ATR 210, current 20,418, monthly high 20,500 -> distance 82 <= 210 -> shown. Monthly low 19,900 -> distance 518 > 210 -> suppressed.

### 6.3 Degraded / GRAY state thresholds (feed into ingest_ok=False)

These map directly to the G3 `BriefReadinessCheck`. Any one failing forces the degraded GRAY brief with the visible "Data incomplete, verify independently" banner; none crash.

| Constant | Value | Trigger -> effect |
|---|---|---|
| `MIN_OVERNIGHT_BARS` | `300` `[PROPOSED - confirm with Bryce]` | overnight 1-min bar count < 300 (of ~750 in a full night) -> overnight story + ONH/ONL degraded, `ingest_ok=False` |
| `MIN_RTH_BARS_PRIOR` | `360` `[PROPOSED - confirm with Bryce]` | prior RTH 1-min bar count < 360 (of 390 full) -> PDH/PDL and ATR suspect (also signals a half-day, see G6) -> degrade unless `session_type=HALF` |
| `MAX_BAR_STALENESS_MIN` | `45` `[PROPOSED - confirm with Bryce]` | newest bar timestamp older than 45 min before the 06:30 ingest -> stale -> `ingest_ok=False`, `is_stale=True` |
| settlement missing | n/a | no `statistics` settlement row for prior date -> gap cannot be trusted -> degrade (this is the explicit architecture failure mode) |
| `SETTLEMENT_SANITY_PCT` | `0.005` `[MECHANICAL]` | abs(prior_settle - current_price)/current_price > 0.5% sanity is a soft check; a >0.5% deviation is allowed (real gaps happen) but logged; this constant is the assert bound for the golden-file sanity test, not a degrade trigger by itself |
| `GAP_ATR_SANITY_ABS` | `3.0` `[MECHANICAL]` | abs(gap_atr_frac) > 3.0 -> implausible, almost always a roll/symbology error -> degrade and alert (matches readiness "gap-as-ATR in [-3,+3]") |

`MIN_RTH_BARS_PRIOR` interacts with G6: on a CME half-day the prior RTH legitimately has ~210 bars (close at 13:00 ET), so the check is conditioned on `session_type`. FULL requires 360+ bars; HALF requires the half-day floor `HALF_DAY_RTH_BARS * 0.92` (= `210 * 0.92` = ~193 bars). `HALF_DAY_RTH_BARS = 210` is a named constant in `config.py` [MECHANICAL] (13:00 ET close yields ~210 of 390 full-session 1-min bars).

### 6.4 Worked examples

1. **Roll day.** `contract` changes `NQH5 -> NQM5`; OI crossover confirms. New contract trades 180 pts higher in absolute terms. Because levels and gap stay on the continuous series, gap reads e.g. +40 pts (real), NOT +220. Annotation: "Contract roll today (NQH5 -> NQM5); levels are continuous-series."
2. **Thin overnight (holiday eve).** Only 240 overnight bars present (< 300). Overnight story degrades to "Overnight data incomplete"; badge GRAY; banner shown; `ingest_ok=False`.
3. **Symbology error.** gap_atr_frac computed at 6.8 (> 3.0). Degrade + MQTT alert to Bryce; brief does not render a misleading 7-ATR gap.

---

## OPEN FOR ALLIE (consolidated)

Every `[PROPOSED - confirm with Allie]` value, for Bryce to review with Allie in one pass. All are coded as the stated default; ratifying them is a tuning exercise, not a blocker.

1. `VIX_ELEVATED_THRESHOLD = 20.0`, `VIX_HIGH_THRESHOLD = 28.0` (item 1): the VIX cuts that drive the badge's volatility axis.
2. `AVG_ONR_LOOKBACK` / `BASELINE_SESSION_COUNT = 20` sessions (item 2): trailing window for "normal" overnight range. (Matches design open question 5.)
3. `ON_RANGE_QUIET_MULTIPLIER = 0.70`, `ON_RANGE_ELEVATED_MULTIPLIER = 1.50` (item 2): the "quiet night / big night" bands.
4. Headline quiet-vs-normal internal split at `0.50` ATR (item 2.4).
5. `NO_TRADE_PAD_BEFORE_HIGH = 2`, `NO_TRADE_PAD_AFTER_HIGH = 5`, `NO_TRADE_PAD_BEFORE_MED = 1`, `NO_TRADE_PAD_AFTER_MED = 3` (item 3): how wide the timer windows are.
6. `FOMC_PAD_BEFORE = 5`, `FOMC_PAD_AFTER_PRESSER = 30` (item 3): the FOMC combined window.
7. `SESSION_END_ET = "16:00"` (item 3): how late in the day to include catalysts. Lower this if Allie stops trading earlier (e.g. by 11:00 ET).
8. Catalyst severity overrides (item 3.1): confirm CPI/PPI/NFP/PCE/retail/FOMC = HIGH, claims/ISM/PMI/sentiment/JOLTS/GDP/Fed-speakers = MED matches how she actually trades them. Mega-cap earnings = MED at the open (item 3.6).
9. `ON_NET_TREND_MIN_ATR = 0.50`, `ON_EFFICIENCY_TREND_MIN = 0.50`, `ONCLV_TRENDED_MIN = 0.80`, `ONCLV_BALANCED_LOW = 0.40`, `ONCLV_BALANCED_HIGH = 0.60` (item 4): the trended/balanced/choppy boundaries.
10. `PER_TRADE_RISK_USD` and `DAILY_STOP_USD` (item 5): her actual numbers (currently unset; sizing hidden until provided). Highest-value items: nothing personal shows until these are set.
11. `STOP_ATR_MULT = 0.50` (item 5): default stop basis, OR confirm she has a fixed stop methodology that replaces it.
12. `MAX_CONTRACTS = 20` (item 5): advisory sizing cap.
13. `LEVEL_PROXIMITY_ATR = 1.0` (item 6): how near a weekly/monthly level must be to show.

OPEN FOR BRYCE (operational, item 6.3): `MIN_OVERNIGHT_BARS = 300`, `MIN_RTH_BARS_PRIOR = 360`, `MAX_BAR_STALENESS_MIN = 45`. Tune against the first weeks of real Databento ingest volumes.
