# Pre-Market Brief: Interface Contracts

**Status**: Authoritative contract document. All modules under `src/` MUST implement against
these contracts without inventing alternative data shapes, field names, or constant values.

**How to use this doc**: Read the section for the layer you are building. Each section gives
you a concrete, buildable spec: a pydantic class you paste into your module, a SQL DDL you run
against DuckDB, a Python Protocol you implement, or an HTTP contract you mount in FastAPI. Where
a value is defined by the decision spec, this doc references it by constant name rather than
repeating the number - look up the value in section A's `Settings` class or in
`docs/premarket_brief_decision_spec.md`. If there is a conflict between this doc and the
decision spec on a NUMERIC value, the decision spec wins. If there is a conflict on a FIELD NAME
or TYPE, this doc wins. Everything else defers to `docs/premarket_brief_architecture.md`.

---

## Table of Contents

- [A. Config Contract (Settings)](#a-config-contract)
- [B. Domain Models](#b-domain-models)
  - [B.1 Enums](#b1-enums)
  - [B.2 Bar](#b2-bar)
  - [B.3 ComputedLevels](#b3-computedlevels)
  - [B.4 CatalystWindow](#b4-catalystwindow)
  - [B.5 SizingLine](#b5-sizingline)
  - [B.6 OvernightSummary](#b6-overnightsummary)
  - [B.7 DataQuality](#b7-dataquality)
  - [B.8 BriefSnapshot](#b8-briefsnapshot)
- [C. Feed Adapter Contracts](#c-feed-adapter-contracts)
  - [C.1 Exceptions](#c1-exceptions)
  - [C.2 BarSource Protocol](#c2-barsource-protocol)
  - [C.3 SettlementSource Protocol](#c3-settlementsource-protocol)
  - [C.4 CalendarSource Protocol](#c4-calendarsource-protocol)
  - [C.5 EarningsSource Protocol](#c5-earningssource-protocol)
  - [C.6 VixSource Protocol](#c6-vixsource-protocol)
  - [C.7 Cost Guard Contract](#c7-cost-guard-contract)
  - [C.8 Retry Policy](#c8-retry-policy)
- [D. Persistence Contract (DuckDB DDL)](#d-persistence-contract)
  - [D.1 bars](#d1-bars)
  - [D.2 settlements](#d2-settlements)
  - [D.3 computed_levels](#d3-computed_levels)
  - [D.4 calendar_events](#d4-calendar_events)
  - [D.5 earnings_events](#d5-earnings_events)
  - [D.6 brief_snapshots](#d6-brief_snapshots)
  - [D.7 api_spend](#d7-api_spend)
  - [D.8 schema_version](#d8-schema_version)
  - [D.9 Concurrency and migration policy](#d9-concurrency-and-migration-policy)
- [E. HTTP / API Contract](#e-http--api-contract)
- [F. Template Context Contract](#f-template-context-contract)

---

## A. Config Contract

`src/config.py` is the single source of truth for every named constant. It contains two
categories: (1) pydantic-settings `Settings` for deployment/ops values that come from the
environment, and (2) module-level constants for analytical/trading values that are code
defaults and are NOT loaded from the environment. Both are shown below.

### A.1 Pydantic-settings class (deployment config, from environment / .env)

```python
# src/settings.py
from __future__ import annotations

from pydantic import Field, field_validator
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    """
    Deployment and operational configuration loaded from environment variables.
    All secrets live here; the analytical constants live in config.py as module-level
    literals so they are importable without an environment being present.

    Validation runs at startup; a missing required field raises immediately.
    """

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="ignore",
    )

    # --- Databento feed ---
    DATABENTO_API_KEY: str = Field(
        ...,
        description="Databento API key. Required.",
    )
    DATABENTO_DATASET: str = Field(
        default="GLBX.MDP3",
        description="Databento dataset identifier.",
    )
    DATABENTO_SYMBOL: str = Field(
        default="NQ.c.0",
        description="Continuous front-month symbol used for all bar and settlement pulls.",
    )
    MAX_DATABENTO_REQUEST_USD: float = Field(
        default=5.00,
        gt=0,
        description=(
            "Cost guard: ingest/cost_guard.py refuses any single Databento request whose "
            "metadata.get_cost() estimate exceeds this value. Prevents symbology mistakes "
            "from running up a bill."
        ),
    )
    MAX_DATABENTO_MONTHLY_USD: float = Field(
        default=15.00,
        gt=0,
        description=(
            "Monthly spend cap. When month-to-date spend in api_spend table reaches this "
            "threshold, new requests are refused and an alert fires. "
            "See DATABENTO_MONTHLY_ALERT_THRESHOLD_USD for the warning-only level."
        ),
    )
    DATABENTO_MONTHLY_ALERT_THRESHOLD_USD: float = Field(
        default=10.00,
        gt=0,
        description=(
            "When month-to-date spend reaches this fraction of MAX_DATABENTO_MONTHLY_USD, "
            "fire an alert but do NOT refuse the request. Must be < MAX_DATABENTO_MONTHLY_USD."
        ),
    )

    @field_validator("DATABENTO_MONTHLY_ALERT_THRESHOLD_USD", mode="after")
    @classmethod
    def alert_below_cap(cls, v: float, info: object) -> float:
        cap = getattr(info.data, "MAX_DATABENTO_MONTHLY_USD", None)
        if cap is not None and v >= cap:
            raise ValueError(
                "DATABENTO_MONTHLY_ALERT_THRESHOLD_USD must be less than "
                "MAX_DATABENTO_MONTHLY_USD"
            )
        return v

    # --- FMP (Financial Modeling Prep) ---
    FMP_API_KEY: str = Field(
        ...,
        description="FMP API key. Required for econ calendar and earnings feeds.",
    )

    # --- SMTP / email delivery ---
    SMTP_HOST: str = Field(
        default="smtp.gmail.com",
        description="SMTP server hostname.",
    )
    SMTP_PORT: int = Field(
        default=587,
        ge=1,
        le=65535,
        description="SMTP server port. 587 = STARTTLS.",
    )
    SMTP_USER: str = Field(
        ...,
        description="SMTP authentication username.",
    )
    SMTP_PASSWORD: str = Field(
        ...,
        description="SMTP authentication password or app-specific password.",
    )
    BRIEF_TO_EMAIL: str = Field(
        ...,
        description="Primary recipient email address (Allie).",
    )
    BRIEF_BCC_EMAIL: str = Field(
        ...,
        description="BCC recipient for every brief email and failure alerts (Bryce).",
    )

    # --- MQTT alerting ---
    MQTT_HOST: str = Field(
        default="192.168.50.200",
        description="Mosquitto broker host on the IoT subnet.",
    )
    MQTT_PORT: int = Field(
        default=1883,
        ge=1,
        le=65535,
        description="Mosquitto broker port.",
    )
    MQTT_ALERT_TOPIC: str = Field(
        default="premarket/alerts",
        description="MQTT topic for failure alerts consumed by Home Assistant.",
    )

    # --- Paths ---
    DB_PATH: str = Field(
        default="/data/premarket.duckdb",
        description="Absolute path to the DuckDB file inside the container.",
    )

    # --- Dashboard / history ---
    BRIEF_HISTORY_DAYS: int = Field(
        default=90,
        gt=0,
        description="How many days of brief_snapshots to retain before pruning.",
    )

    # --- Admin auth token ---
    ADMIN_TOKEN: str = Field(
        ...,
        description=(
            "Bearer token for admin-only endpoints (POST /api/admin/run-now, "
            "GET /api/admin/runs). Must be set in the environment."
        ),
    )
```

### A.2 Analytical / trading constants (module-level literals in config.py)

These are NOT loaded from the environment. They are hard-coded defaults that Allie or Bryce
can ratify and then change in code. Values match exactly the decision spec constants table.
Aliases are declared explicitly so tests can assert they remain equal.

```python
# src/config.py
from __future__ import annotations

from zoneinfo import ZoneInfo

# ---------------------------------------------------------------------------
# Timezone
# ---------------------------------------------------------------------------
DST_ZONE: ZoneInfo = ZoneInfo("America/New_York")

# ---------------------------------------------------------------------------
# VIX regime thresholds  [PROPOSED - confirm with Allie]
# Reference: decision_spec section 1.1
# ---------------------------------------------------------------------------
VIX_ELEVATED_THRESHOLD: float = 20.0   # VIX index points; LOW vs ELEVATED cutoff
VIX_HIGH_THRESHOLD: float = 28.0       # VIX index points; ELEVATED vs HIGH cutoff
VIX_FALLBACK_REGIME: str = "LOW"       # regime used when VIX is stale/missing [MECHANICAL]
MAX_VIX_STALENESS_DAYS: int = 3        # calendar days before "VIX unavailable" [MECHANICAL]

# ---------------------------------------------------------------------------
# Gap / ATR buckets  [MECHANICAL]
# Reference: decision_spec section 1.1 and 2.3
# ---------------------------------------------------------------------------
GAP_ATR_QUIET_MAX: float = 0.20        # <= this -> QUIET bucket for badge
GAP_ATR_ELEVATED_MAX: float = 1.00    # <= this -> NORMAL bucket; above -> LARGE
GAP_ATR_GRAY_MAX: float = 0.20        # <= this -> headline color is GRAY (inconclusive)
                                       # NOTE: intentionally equal to GAP_ATR_QUIET_MAX but
                                       # separately named; they govern different outputs and
                                       # may diverge. Do not deduplicate.

# ---------------------------------------------------------------------------
# ATR computation  [MECHANICAL]
# Reference: decision_spec section 2.1
# ATR is computed on RTH-only daily bars (09:30-16:00 ET), NOT the 24h Globex series.
# ---------------------------------------------------------------------------
ATR_LOOKBACK: int = 14                 # RTH daily bars

# ---------------------------------------------------------------------------
# Overnight range baseline  [PROPOSED - confirm with Allie]
# Reference: decision_spec section 2.2 and 2.5
# ---------------------------------------------------------------------------
BASELINE_SESSION_COUNT: int = 20       # complete non-half, non-holiday sessions
AVG_ONR_LOOKBACK: int = BASELINE_SESSION_COUNT   # alias; tests assert equality

ON_RANGE_QUIET_MULTIPLIER: float = 0.70    # on_range/avg < this -> "quiet night"
ON_RANGE_ELEVATED_MULTIPLIER: float = 1.50 # on_range/avg >= this -> "big night"
# Short-form aliases for backward compat with Phase-3 task text
ON_RANGE_QUIET_MULT: float = ON_RANGE_QUIET_MULTIPLIER
ON_RANGE_ELEVATED_MULT: float = ON_RANGE_ELEVATED_MULTIPLIER

# ---------------------------------------------------------------------------
# Catalyst no-trade window padding  [PROPOSED - confirm with Allie]
# Reference: decision_spec section 3.2
# ---------------------------------------------------------------------------
NO_TRADE_PAD_BEFORE_HIGH: int = 2      # minutes before a HIGH-severity event
NO_TRADE_PAD_AFTER_HIGH: int = 5       # minutes after a HIGH-severity event
NO_TRADE_PAD_BEFORE_MED: int = 1       # minutes before a MED-severity event
NO_TRADE_PAD_AFTER_MED: int = 3        # minutes after a MED-severity event

# FOMC special case  [PROPOSED - confirm with Allie]
# Reference: decision_spec section 3.3
# Combined window: [14:00 - FOMC_PAD_BEFORE, 14:30 + FOMC_PAD_AFTER_PRESSER]
FOMC_PAD_BEFORE: int = 5               # minutes before 14:00 ET statement
FOMC_PAD_AFTER_PRESSER: int = 30       # minutes after 14:30 ET presser start
                                        # -> window is 13:55 - 15:00 ET

# Cap on catalyst banner lines  [MECHANICAL]
# Reference: decision_spec section 3.4 and design 3A
CATALYST_MAX_SHOWN: int = 3

# Session end for catalyst inclusion  [PROPOSED - confirm with Allie]
# Reference: decision_spec section 3.5
# Format: "HH:MM" in ET. Lower this if Allie stops trading earlier.
SESSION_END_ET: str = "16:00"

# ---------------------------------------------------------------------------
# Overnight state classification  [PROPOSED - confirm with Allie]
# Reference: decision_spec section 4.2 and 4.3
# ---------------------------------------------------------------------------
ONCLV_TRENDED_MIN: float = 0.80       # close-location at/above (or at/below 1-this) -> directional
ONCLV_BALANCED_LOW: float = 0.40      # lower edge of balanced close-location band
ONCLV_BALANCED_HIGH: float = 0.60     # upper edge of balanced close-location band
ON_NET_TREND_MIN_ATR: float = 0.50    # net-change magnitude as ATR fraction; gate for TRENDED
ON_EFFICIENCY_TREND_MIN: float = 0.50 # path efficiency ratio; gate for TRENDED

# ---------------------------------------------------------------------------
# Position sizing  [PROPOSED - confirm with Allie unless marked MECHANICAL]
# Reference: decision_spec section 5
# ---------------------------------------------------------------------------
PER_TRADE_RISK_USD: float | None = None   # Allie's dollar risk per trade; None = sizing hidden
DAILY_STOP_USD: float | None = None       # Allie's daily stop; None = reminder hidden
STOP_ATR_MULT: float = 0.50               # default stop = this * atr_14 when no fixed stop
MNQ_POINT_VALUE: float = 2.0             # USD per index point [MECHANICAL]
NQ_POINT_VALUE: float = 20.0             # USD per index point [MECHANICAL]
MAX_CONTRACTS: int = 20                   # advisory safety cap [PROPOSED - confirm with Allie]

# ---------------------------------------------------------------------------
# Level proximity / display  [PROPOSED - confirm with Allie]
# Reference: decision_spec section 6.2
# ---------------------------------------------------------------------------
LEVEL_PROXIMITY_ATR: float = 1.0    # show weekly/monthly H/L only within this many ATR
ROLL_CARRY_LEVELS: bool = True       # on roll, carry prior levels (no false gap) [MECHANICAL]

# ---------------------------------------------------------------------------
# Data quality / degraded-brief thresholds  [PROPOSED - confirm with Bryce]
# Reference: decision_spec section 6.3
# ---------------------------------------------------------------------------
MIN_OVERNIGHT_BARS: int = 300        # 1-min bars; below -> overnight story degraded, ingest_ok=False
MIN_RTH_BARS_PRIOR: int = 360        # 1-min bars; below -> PDH/PDL/ATR suspect (FULL session)
                                      # HALF-session floor = expected_half_bars * 0.92 (derived)
MAX_BAR_STALENESS_MIN: int = 45      # minutes; newest bar older than this at 06:30 -> is_stale=True

SETTLEMENT_SANITY_PCT: float = 0.005  # 0.5%; sanity-test bound; NOT a hard degrade trigger [MECHANICAL]
GAP_ATR_SANITY_ABS: float = 3.0       # abs(gap_atr_frac) > this -> implausible, degrade + alert [MECHANICAL]

# ---------------------------------------------------------------------------
# Risk Mode decision lookup table  [MECHANICAL]
# Reference: decision_spec section 1.3
# Keys: (catalyst_severity, gap_atr_bucket, vix_regime) -> RiskMode string
# catalyst_severity in {"NONE", "MED", "HIGH"}
# gap_atr_bucket in {"QUIET", "NORMAL", "LARGE"}
# vix_regime in {"LOW", "ELEVATED", "HIGH"}
# ---------------------------------------------------------------------------
RISK_MODE_TABLE: dict[tuple[str, str, str], str] = {
    ("NONE", "QUIET",  "LOW"):      "GREEN",
    ("NONE", "QUIET",  "ELEVATED"): "GREEN",
    ("NONE", "QUIET",  "HIGH"):     "YELLOW",
    ("NONE", "NORMAL", "LOW"):      "GREEN",
    ("NONE", "NORMAL", "ELEVATED"): "YELLOW",
    ("NONE", "NORMAL", "HIGH"):     "YELLOW",
    ("NONE", "LARGE",  "LOW"):      "YELLOW",
    ("NONE", "LARGE",  "ELEVATED"): "YELLOW",
    ("NONE", "LARGE",  "HIGH"):     "RED",
    ("MED",  "QUIET",  "LOW"):      "GREEN",
    ("MED",  "QUIET",  "ELEVATED"): "YELLOW",
    ("MED",  "QUIET",  "HIGH"):     "YELLOW",
    ("MED",  "NORMAL", "LOW"):      "YELLOW",
    ("MED",  "NORMAL", "ELEVATED"): "YELLOW",
    ("MED",  "NORMAL", "HIGH"):     "RED",
    ("MED",  "LARGE",  "LOW"):      "YELLOW",
    ("MED",  "LARGE",  "ELEVATED"): "RED",
    ("MED",  "LARGE",  "HIGH"):     "RED",
    ("HIGH", "QUIET",  "LOW"):      "YELLOW",
    ("HIGH", "QUIET",  "ELEVATED"): "YELLOW",
    ("HIGH", "QUIET",  "HIGH"):     "RED",
    ("HIGH", "NORMAL", "LOW"):      "RED",
    ("HIGH", "NORMAL", "ELEVATED"): "RED",
    ("HIGH", "NORMAL", "HIGH"):     "RED",
    ("HIGH", "LARGE",  "LOW"):      "RED",
    ("HIGH", "LARGE",  "ELEVATED"): "RED",
    ("HIGH", "LARGE",  "HIGH"):     "RED",
}

# ---------------------------------------------------------------------------
# Risk Mode display strings  [MECHANICAL]
# Reference: decision_spec section 1.6
# ---------------------------------------------------------------------------
RISK_MODE_DISPLAY: dict[str, str] = {
    "GREEN":  "TRADE NORMAL SIZE",
    "YELLOW": "TRADE SMALLER / BE SELECTIVE",
    "RED":    "STAND ASIDE OR TRADE TINY",
    "GRAY":   "DATA INCOMPLETE - VERIFY INDEPENDENTLY",
}

# ---------------------------------------------------------------------------
# Catalyst severity overrides  [PROPOSED - confirm with Allie]
# Reference: decision_spec section 3.1
# Keys are lowercase substrings matched case-insensitively against normalized event name.
# Values are "HIGH" or "MED". Anything matching "ignored" keywords is discarded entirely.
# Override wins over FMP's own impact field.
# ---------------------------------------------------------------------------
CATALYST_SEVERITY_OVERRIDES: dict[str, str] = {
    "cpi":                    "HIGH",
    "consumer price":         "HIGH",
    "pce":                    "HIGH",
    "nonfarm":                "HIGH",
    "nfp":                    "HIGH",
    "employment situation":   "HIGH",
    "fomc":                   "HIGH",
    "federal funds":          "HIGH",
    "rate decision":          "HIGH",
    "interest rate decision": "HIGH",
    "ppi":                    "HIGH",
    "producer price":         "HIGH",
    "retail sales":           "HIGH",
    "jobless claims":         "MED",
    "initial claims":         "MED",
    "continuing claims":      "MED",
    "ism":                    "MED",
    "pmi":                    "MED",
    "consumer confidence":    "MED",
    "consumer sentiment":     "MED",
    "umich":                  "MED",
    "jolts":                  "MED",
    "job openings":           "MED",
    "gdp":                    "MED",
    "powell":                 "MED",
    "fed chair":              "MED",
    "fed speak":              "MED",
    "fed president":          "MED",
    "fomc member":            "MED",
}

# Keywords that cause an event to be IGNORED entirely (not HIGH/MED, not LOW - discarded)
CATALYST_IGNORE_KEYWORDS: list[str] = [
    "holiday",
    "auction",
    " bill",   # leading space avoids matching "jobless claims" etc.
    "bond",
]

# Nasdaq-100 mega-cap components tracked for earnings folding.
# Reference: architecture section 2 and decision_spec section 3.6.
# Refresh quarterly.
NDX_COMPONENTS: list[str] = [
    "AAPL", "MSFT", "NVDA", "AMZN", "GOOGL", "GOOG", "META",
    "TSLA", "AVGO", "COST", "NFLX", "AMD", "ADBE", "QCOM",
    "INTC", "TXN", "AMAT", "MU", "LRCX", "KLAC",
]

# Logic version string - bump on any change to brief computation logic
LOGIC_VERSION: str = "1.0.0"
```

---

## B. Domain Models

All models live under `src/` (recommended: `src/models.py` or `src/brief/models.py`).
Import `Settings` from `src/settings.py` and constants from `src/config.py`. Never inline
constants inside model definitions.

### B.1 Enums

```python
# src/models.py  (partial)
from __future__ import annotations

from enum import Enum


class RiskMode(str, Enum):
    """
    The synthesized risk badge for the day. GRAY is not a table output - it is the
    degraded state that supersedes the 27-cell lookup when data is insufficient.
    Reference: decision_spec sections 1.3 and 1.4.
    """
    GREEN  = "GREEN"
    YELLOW = "YELLOW"
    RED    = "RED"
    GRAY   = "GRAY"   # data incomplete; badge table is NOT consulted in this state


class SessionType(str, Enum):
    """
    CME session classification for the date being briefed.
    Reference: architecture section 5, build_plan Phase 1 step 1.6.
    """
    FULL    = "FULL"      # normal RTH session, 09:30-16:00 ET
    HALF    = "HALF"      # CME early-close day (e.g. day before holiday), closes ~13:00 ET
    HOLIDAY = "HOLIDAY"   # CME market holiday; no RTH session; brief renders "no session" state


class CatalystSeverity(str, Enum):
    """
    Severity of a single catalyst event after the override table and FMP fallback are applied.
    Reference: decision_spec section 3.1.
    NONE is not stored per-event; it is the aggregate value when no MED/HIGH events exist.
    """
    HIGH = "HIGH"
    MED  = "MED"


class OvernightState(str, Enum):
    """
    Non-directional shape label for the overnight session.
    The engine NEVER attaches a direction to this label.
    Reference: decision_spec section 4.3.
    """
    TRENDED  = "TRENDED"   # directional travel, closed near range extreme, efficient path
    BALANCED = "BALANCED"  # closed mid-range with little net change; two-sided/rotational
    CHOPPY   = "CHOPPY"    # moved around but did not resolve; low efficiency or ambiguous close
```

### B.2 Bar

```python
from datetime import datetime
from pydantic import BaseModel, Field


class Bar(BaseModel):
    """
    One normalized 1-minute OHLCV bar. Written by ingest/normalizer.py and stored in
    the `bars` table. The `session` tag is derived from the bar timestamp in session.py,
    not from a Databento field.
    """
    ts: datetime = Field(
        description=(
            "Bar open timestamp in UTC. All internal storage is UTC; convert to ET only "
            "at display time using DST_ZONE."
        )
    )
    symbol: str = Field(
        description="Continuous symbol, e.g. 'NQ.c.0'. Matches DATABENTO_SYMBOL."
    )
    contract: str = Field(
        description=(
            "Resolved explicit contract, e.g. 'NQH5'. Stamped by normalizer from "
            "Databento's instrument metadata. Used for roll-week awareness."
        )
    )
    open: float = Field(description="Bar open price in index points.")
    high: float = Field(description="Bar high price in index points.")
    low: float = Field(description="Bar low price in index points.")
    close: float = Field(description="Bar close price in index points.")
    volume: int = Field(description="Bar volume in contracts.")
    session: str = Field(
        description=(
            "Session tag: 'RTH' (09:30-16:00 ET) or 'ETH' (all other Globex hours). "
            "Derived from ts by session.py; never comes from the vendor."
        )
    )
    source: str = Field(
        default="databento",
        description="Feed source identifier. Always 'databento' for current architecture.",
    )
```

### B.3 ComputedLevels

```python
from datetime import date, datetime
from typing import Optional
from pydantic import BaseModel, Field


class ComputedLevels(BaseModel):
    """
    The full pre-computed daily level set written by brief/engine.py at the 06:30 ingest
    and stored in the `computed_levels` table. The brief reads from this; it does NOT
    recompute levels at request time.

    Field nullability: nullable fields (Optional[float]) become None when data is insufficient
    (e.g. missing settlement, insufficient bar count). A None level triggers degraded/GRAY state
    via DataQuality. Non-nullable fields always have a value or the brief is blocked entirely.

    All price fields are in NQ index points. Gap fields are dimensionless fractions
    unless "_pts" or "_pct" suffix is explicit.
    """
    level_date: date = Field(description="The trading date this level set applies to (ET date).")
    symbol: str = Field(description="Continuous symbol, e.g. 'NQ.c.0'.")
    computed_at: datetime = Field(description="UTC timestamp when this row was written.")

    # --- Prior RTH session levels ---
    pdh: float = Field(description="Prior-day RTH high in index points.")
    pdl: float = Field(description="Prior-day RTH low in index points.")
    pdc: float = Field(description="Prior-day RTH close in index points.")
    prior_settle: float = Field(
        description=(
            "Official CME prior-day settlement price from the statistics schema. "
            "This is the gap reference, NOT pdc. "
            "Source: settlements table, never derived from bars."
        )
    )

    # --- Overnight session levels ---
    onh: float = Field(description="Overnight (ETH) high in index points.")
    onl: float = Field(description="Overnight (ETH) low in index points.")
    on_open: float = Field(description="First overnight bar open price in index points.")
    on_close: float = Field(
        description=(
            "Last overnight bar close at the 06:30 ingest time in index points. "
            "This is the current_price proxy used for gap computation."
        )
    )
    on_vwap: float = Field(description="Overnight session VWAP in index points.")

    # --- Weekly and monthly levels (conditionally displayed) ---
    # Null when no bars exist yet for the current week/month (e.g. Monday open).
    # Display is suppressed when abs(current_price - level) > LEVEL_PROXIMITY_ATR * atr_14.
    # Reference: decision_spec section 6.2.
    weekly_high: Optional[float] = Field(
        default=None,
        description=(
            "Current-week high in index points. Null if current week has no prior bars. "
            "Display is proximity-gated by LEVEL_PROXIMITY_ATR."
        ),
    )
    weekly_low: Optional[float] = Field(
        default=None,
        description="Current-week low in index points. Same nullability as weekly_high.",
    )
    monthly_high: Optional[float] = Field(
        default=None,
        description=(
            "Current-month high in index points. Null if current month has no prior bars. "
            "Display is proximity-gated by LEVEL_PROXIMITY_ATR."
        ),
    )
    monthly_low: Optional[float] = Field(
        default=None,
        description="Current-month low in index points. Same nullability as monthly_high.",
    )

    # --- Volatility / ATR ---
    atr_14: Optional[float] = Field(
        default=None,
        description=(
            "Wilder ATR(14) computed on RTH-only daily bars. Index points. "
            "Null if insufficient RTH history (< ATR_LOOKBACK + 1 days). "
            "Null triggers degraded/GRAY state."
        ),
    )
    avg_onr_20: Optional[float] = Field(
        default=None,
        description=(
            "Average overnight range over BASELINE_SESSION_COUNT (20) complete "
            "non-half, non-holiday sessions. Index points. "
            "Null if insufficient history."
        ),
    )

    # --- Gap computation (all reference prior_settle, not pdc) ---
    current_price: float = Field(
        description="Last 1-min bar close at the 06:30 ingest. Same as on_close."
    )
    gap_pts: float = Field(
        description=(
            "current_price - prior_settle. Signed: positive = gap up, negative = gap down. "
            "Index points."
        )
    )
    gap_pct: float = Field(
        description="gap_pts / prior_settle. Signed fraction (e.g. 0.004 = +0.4%)."
    )
    gap_as_atr_frac: Optional[float] = Field(
        default=None,
        description=(
            "abs(gap_pts) / atr_14. Unsigned magnitude used for badge bucketing and "
            "headline color. Null when atr_14 is None."
        ),
    )

    # --- Roll / proximity flags ---
    is_roll_day: bool = Field(
        default=False,
        description=(
            "True when the front-month contract changed since the prior session "
            "(detected by resolved contract stamp change or OI crossover). "
            "When True, brief annotates 'Contract roll today; levels are continuous-series.' "
            "Reference: decision_spec section 6.1 and ROLL_CARRY_LEVELS."
        ),
    )
    roll_from_contract: Optional[str] = Field(
        default=None,
        description=(
            "Prior front-month contract identifier, e.g. 'NQH5'. "
            "Non-null only when is_roll_day=True."
        ),
    )
    roll_to_contract: Optional[str] = Field(
        default=None,
        description="New front-month contract identifier. Non-null only when is_roll_day=True.",
    )
    weekly_high_in_proximity: bool = Field(
        default=False,
        description=(
            "True when weekly_high is non-null and within LEVEL_PROXIMITY_ATR * atr_14 "
            "of current_price. Pre-computed so templates need no math."
        ),
    )
    weekly_low_in_proximity: bool = Field(
        default=False,
        description="True when weekly_low is within proximity. See weekly_high_in_proximity.",
    )
    monthly_high_in_proximity: bool = Field(
        default=False,
        description="True when monthly_high is within proximity.",
    )
    monthly_low_in_proximity: bool = Field(
        default=False,
        description="True when monthly_low is within proximity.",
    )

    # --- Session classification ---
    session_type: SessionType = Field(
        description="FULL, HALF, or HOLIDAY for this trading date."
    )
```

### B.4 CatalystWindow

```python
from datetime import datetime
from typing import Optional
from pydantic import BaseModel, Field
from .models import CatalystSeverity  # same file in practice


class CatalystWindow(BaseModel):
    """
    A single qualifying catalyst event with its derived no-trade window.
    Written by brief/catalysts.py. Stored inside BriefSnapshot.catalysts.

    All times are stored in UTC. Templates display them in ET. The no-trade window
    is computed and stored here so templates never need to perform time math.
    """
    event_label: str = Field(
        description=(
            "Human-readable event name for display. Examples: 'CPI', 'FOMC Rate Decision', "
            "'NVDA earnings reaction at the open'. This is the rendered string, not the raw "
            "FMP event name."
        )
    )
    severity: CatalystSeverity = Field(
        description="HIGH or MED after override table and FMP fallback are applied."
    )
    event_time_utc: Optional[datetime] = Field(
        default=None,
        description=(
            "Scheduled event time in UTC. None for mega-cap earnings where the reaction "
            "time is the RTH open rather than a specific clock time."
        ),
    )
    no_trade_start_utc: Optional[datetime] = Field(
        default=None,
        description=(
            "Start of the no-trade window in UTC. "
            "= event_time - pad_before (from NO_TRADE_PAD_BEFORE_HIGH/MED). "
            "None for mega-cap earnings (no timed window; textual note instead)."
        ),
    )
    no_trade_end_utc: Optional[datetime] = Field(
        default=None,
        description=(
            "End of the no-trade window in UTC. "
            "= event_time + pad_after (from NO_TRADE_PAD_AFTER_HIGH/MED). "
            "FOMC: end = 14:30 ET + FOMC_PAD_AFTER_PRESSER. "
            "None for mega-cap earnings."
        ),
    )
    source: str = Field(
        description=(
            "Feed source: 'fmp_calendar', 'fmp_earnings', or 'override_table'. "
            "'override_table' when the severity was determined by CATALYST_SEVERITY_OVERRIDES "
            "rather than FMP's impact field."
        )
    )
    is_earnings: bool = Field(
        default=False,
        description=(
            "True for mega-cap earnings events (folded in from FMP earnings feed). "
            "These have no timed window; no_trade_start/end are None."
        ),
    )
```

### B.5 SizingLine

```python
from typing import Optional
from pydantic import BaseModel, Field


class SizingLine(BaseModel):
    """
    Advisory position sizing for the day. Null-state is represented by the
    `null_reason` field rather than by making the whole model optional, so templates
    always receive a SizingLine and can branch on null_reason.

    Reference: decision_spec section 5.
    All USD fields are in US dollars. stop_points is in NQ index points.
    """
    per_trade_risk_usd: Optional[float] = Field(
        default=None,
        description=(
            "Allie's configured PER_TRADE_RISK_USD. None when unset; in that case "
            "null_reason is set and no computed fields are populated."
        ),
    )
    stop_points: Optional[float] = Field(
        default=None,
        description=(
            "Stop distance in index points. Either Allie's fixed stop, or "
            "STOP_ATR_MULT * atr_14 when no fixed stop is configured. "
            "None when per_trade_risk_usd is None."
        ),
    )
    stop_basis: Optional[str] = Field(
        default=None,
        description=(
            "How stop_points was determined: 'fixed' (Allie's configured stop) or "
            "'atr_mult' (STOP_ATR_MULT * atr_14). None when per_trade_risk_usd is None."
        ),
    )
    nq_contracts: Optional[int] = Field(
        default=None,
        description=(
            "Advisory NQ contract count = floor(per_trade_risk_usd / (stop_points * NQ_POINT_VALUE)), "
            "capped at MAX_CONTRACTS. May be 0 when risk-per-contract exceeds per_trade_risk_usd. "
            "None when per_trade_risk_usd is None."
        ),
    )
    mnq_contracts: Optional[int] = Field(
        default=None,
        description=(
            "Advisory MNQ contract count = floor(per_trade_risk_usd / (stop_points * MNQ_POINT_VALUE)), "
            "capped at MAX_CONTRACTS. None when per_trade_risk_usd is None."
        ),
    )
    is_capped: bool = Field(
        default=False,
        description=(
            "True when either nq_contracts or mnq_contracts was reduced to MAX_CONTRACTS "
            "by the safety cap."
        ),
    )
    daily_stop_usd: Optional[float] = Field(
        default=None,
        description=(
            "Allie's configured DAILY_STOP_USD. None when unset. "
            "When set, the template renders the stop-out reminder."
        ),
    )
    n_stop_outs: Optional[int] = Field(
        default=None,
        description=(
            "floor(daily_stop_usd / per_trade_risk_usd). The 'done after N full stop-outs' "
            "reminder value. None when either daily_stop_usd or per_trade_risk_usd is None."
        ),
    )
    null_reason: Optional[str] = Field(
        default=None,
        description=(
            "When PER_TRADE_RISK_USD is unset, this contains the display string: "
            "'Set your per-trade risk to see suggested sizing.' "
            "Non-null means ALL computed numeric fields are None and the template shows "
            "only this message."
        ),
    )
```

### B.6 OvernightSummary

```python
from typing import Optional
from pydantic import BaseModel, Field
from .models import OvernightState


class OvernightSummary(BaseModel):
    """
    Non-directional overnight session summary. Computed by brief/overnight.py.
    The engine MUST NEVER populate a direction-carrying string in any field.
    Forbidden words in any field value: 'up', 'down', 'higher', 'lower',
    'bullish', 'bearish', 'continuation', 'reversal', 'bias'.
    Reference: decision_spec section 4.1 and 4.4.
    """
    state: Optional[OvernightState] = Field(
        default=None,
        description=(
            "TRENDED, BALANCED, or CHOPPY. None when bar count < MIN_OVERNIGHT_BARS or "
            "overnight range is zero (degraded); template renders 'Overnight data incomplete.'"
        ),
    )
    net_change_pts: Optional[float] = Field(
        default=None,
        description=(
            "UNSIGNED magnitude of on_close - on_open in index points. "
            "The sign is intentionally withheld from this field to enforce the no-direction "
            "rule. The signed gap is carried separately in ComputedLevels.gap_pts. "
            "None when state is None."
        ),
    )
    range_pts: Optional[float] = Field(
        default=None,
        description=(
            "onh - onl in index points. None when state is None."
        ),
    )
    range_vs_avg: Optional[float] = Field(
        default=None,
        description=(
            "on_range_pts / avg_onr_20. Dimensionless ratio. "
            "< ON_RANGE_QUIET_MULTIPLIER -> 'quiet night'; "
            ">= ON_RANGE_ELEVATED_MULTIPLIER -> 'big night'. "
            "None when state is None or avg_onr_20 is None."
        ),
    )
    phrase: Optional[str] = Field(
        default=None,
        description=(
            "Pre-rendered display phrase from the exact templates in decision_spec section 4.4. "
            "Examples: 'Overnight trended (moved 115 pts, 0.5x normal range).' "
            "None when state is None. "
            "ENFORCEMENT: phrase must not contain directional words. Tests assert this."
        ),
    )
    bar_count: int = Field(
        default=0,
        description="Number of 1-min overnight bars used in the computation.",
    )
    is_degraded: bool = Field(
        default=False,
        description=(
            "True when bar_count < MIN_OVERNIGHT_BARS or on_range == 0. "
            "When True, state/net_change_pts/range_pts/range_vs_avg/phrase are all None."
        ),
    )
```

### B.7 DataQuality

```python
from datetime import datetime
from typing import Optional
from pydantic import BaseModel, Field


class SourceFreshness(BaseModel):
    """Per-feed freshness record nested inside DataQuality."""
    source: str = Field(description="Feed identifier: 'databento', 'fmp_calendar', 'fmp_earnings', 'vix'.")
    ok: bool = Field(description="True if the feed returned usable data.")
    last_updated_utc: Optional[datetime] = Field(
        default=None,
        description="UTC timestamp of the most recent successful data point from this source.",
    )
    error_message: Optional[str] = Field(
        default=None,
        description="Error description when ok=False. None when ok=True.",
    )


class DataQuality(BaseModel):
    """
    Data quality and freshness state for the brief. Produced by the BriefReadinessCheck
    in brief/engine.py. Written to brief_snapshots.ingest_ok and issues[].
    Reference: architecture section 5 and decision_spec section 6.3.
    """
    ingest_ok: bool = Field(
        description=(
            "False if ANY of the following hold: "
            "overnight bar count < MIN_OVERNIGHT_BARS, "
            "newest bar older than MAX_BAR_STALENESS_MIN minutes at 06:30 ingest, "
            "prior settlement is missing, "
            "gap_atr_frac > GAP_ATR_SANITY_ABS. "
            "When False, the brief renders GRAY with the 'Data incomplete' banner."
        )
    )
    is_stale: bool = Field(
        description=(
            "True specifically when the newest bar is older than MAX_BAR_STALENESS_MIN "
            "minutes at the 06:30 ingest time. Subset of ingest_ok=False triggers."
        )
    )
    issues: list[str] = Field(
        default_factory=list,
        description=(
            "Human-readable list of data quality issues found. Empty when ingest_ok=True. "
            "Examples: 'Overnight bar count 240 < MIN_OVERNIGHT_BARS (300)', "
            "'Settlement missing for prior date', "
            "'VIX data older than MAX_VIX_STALENESS_DAYS (3 days)'."
        ),
    )
    as_of_utc: datetime = Field(
        description="UTC timestamp when the data quality check ran."
    )
    bar_count_overnight: int = Field(
        description="Number of 1-min overnight bars present at ingest time."
    )
    bar_count_rth_prior: int = Field(
        description="Number of 1-min RTH bars present for the prior session."
    )
    newest_bar_utc: Optional[datetime] = Field(
        default=None,
        description="UTC timestamp of the most recent bar in the bars table. None if no bars.",
    )
    settlement_present: bool = Field(
        description="True when a valid prior-day settlement row exists in the settlements table.",
    )
    per_source: list[SourceFreshness] = Field(
        default_factory=list,
        description="Per-feed freshness records for databento, fmp_calendar, fmp_earnings, vix.",
    )
```

### B.8 BriefSnapshot

```python
from datetime import date, datetime
from typing import Literal, Optional
from pydantic import BaseModel, Field
from .models import RiskMode, SessionType
from .computed_levels import ComputedLevels      # same package
from .catalyst_window import CatalystWindow
from .sizing_line import SizingLine
from .overnight_summary import OvernightSummary
from .data_quality import DataQuality


class BriefSnapshot(BaseModel):
    """
    The top-level object for one day's pre-market brief. Persisted to brief_snapshots
    as brief_json (model_dump_json()) and also used as the FastAPI response model.

    BriefSnapshot is the contract between the brief engine and ALL consumers:
    - brief/engine.py produces it
    - api/routes_api.py serializes it as JSON
    - templates receive it (or a subset) as template context
    - email_sender/send.py renders it to HTML

    Serialization: model_dump_json(mode='json') -> stored in brief_snapshots.brief_json.
    Deserialization: model_validate_json(row.brief_json) -> restored for rendering.
    The brief_json column is the source of truth; brief_html is a cache.
    """
    brief_date: date = Field(
        description=(
            "The ET trading date this brief covers. This is the date in the user's "
            "timezone (America/New_York), not the UTC date."
        )
    )
    symbols: list[str] = Field(
        default=["MNQ", "NQ"],
        description=(
            "Instruments covered by this brief. Always [MNQ, NQ]. Both share the same "
            "NQ.c.0 reference series; this field records that both are covered."
        ),
    )
    generated_at_utc: datetime = Field(
        description="UTC timestamp when build_brief() completed and wrote this snapshot."
    )
    risk_mode: RiskMode = Field(
        description=(
            "The synthesized risk badge. GRAY when ingest_ok=False. "
            "GREEN/YELLOW/RED from RISK_MODE_TABLE when ingest_ok=True. "
            "Reference: decision_spec section 1."
        )
    )
    headline: str = Field(
        description=(
            "One-line summary rendered by the engine. Example: "
            "'NQ +78 pts (+0.4%), gap up, just under yesterday's high, "
            "on a quiet night (gap = 0.4x ATR).' "
            "The engine assembles this from ComputedLevels fields; "
            "it is stored here so templates never reconstruct it."
        )
    )
    volatility_line: str = Field(
        description=(
            "One-line volatility context. Example: 'ATR(14) 210 pts. VIX 16.2 (~1.0% move).' "
            "If VIX is unavailable, this reads 'ATR(14) 210 pts. VIX unavailable.' "
            "Reference: decision_spec sections 2.3 and 2.6."
        )
    )
    session_type: SessionType = Field(
        description="FULL, HALF, or HOLIDAY for brief_date."
    )
    levels: Optional[ComputedLevels] = Field(
        default=None,
        description=(
            "Full pre-computed level set. See ComputedLevels model. "
            "None when session_type == SessionType.HOLIDAY (no RTH session; no levels computed). "
            "Templates must check 'if snapshot.levels' before accessing any sub-field."
        ),
    )
    catalysts: list[CatalystWindow] = Field(
        default_factory=list,
        description=(
            "Qualifying catalyst events, already sorted chronologically and capped at "
            "CATALYST_MAX_SHOWN (3). Empty list means 'No major events' (not feed failure). "
            "Feed failure is indicated by catalyst_feed_ok=False."
        ),
    )
    catalyst_feed_ok: bool = Field(
        default=True,
        description=(
            "False when the FMP calendar call failed. When False, the catalyst banner "
            "renders 'Calendar unavailable, check independently' regardless of catalysts list. "
            "Reference: decision_spec section 3.7."
        ),
    )
    catalysts_dropped_count: int = Field(
        default=0,
        description=(
            "Number of qualifying events dropped by the CATALYST_MAX_SHOWN cap. "
            "When > 0, template appends '+N more lower-priority events (see drill-down)'."
        ),
    )
    overnight: OvernightSummary = Field(
        description="Non-directional overnight session summary. See OvernightSummary model."
    )
    sizing: SizingLine = Field(
        description=(
            "Advisory sizing line. Always present; check sizing.null_reason to know whether "
            "PER_TRADE_RISK_USD was configured."
        )
    )
    data_quality: DataQuality = Field(
        description="Data quality and freshness state. risk_mode=GRAY when ingest_ok=False."
    )
    logic_version: str = Field(
        description=(
            "Version string from LOGIC_VERSION constant. Stored so historical snapshots "
            "can be re-rendered or diffed against the producing engine version."
        )
    )
    vix_value: Optional[float] = Field(
        default=None,
        description=(
            "Prior VIX close used for this brief. None when VIX was unavailable or stale. "
            "In that case vix_regime='LOW' (VIX_FALLBACK_REGIME) and volatility_line says "
            "'VIX unavailable'."
        ),
    )
    vix_regime: str = Field(
        default="LOW",
        description=(
            "VIX regime bucket used for RISK_MODE_TABLE lookup: 'LOW', 'ELEVATED', or 'HIGH'. "
            "Defaults to VIX_FALLBACK_REGIME when vix_value is None. "
            "Reference: decision_spec section 1.1."
        ),
    )
    catalyst_level: Literal["NONE", "MED", "HIGH"] = Field(
        default="NONE",
        description=(
            "Aggregate catalyst severity for the day: the maximum CatalystSeverity across all "
            "qualifying events, or 'NONE' when no MED or HIGH events qualify. "
            "This is the value fed into RISK_MODE_TABLE as the catalyst_severity dimension. "
            "Do not add NONE to CatalystSeverity (that enum is per-event only). "
            "Reference: decision_spec section 1.1."
        ),
    )

    # --- Convenience delegation properties ---
    # These are read-only shims so common assertions stay terse. They delegate to
    # data_quality and do NOT store separate values. Tests may use either form;
    # prefer the nested form (snapshot.data_quality.ingest_ok) in new assertions.

    @property
    def ingest_ok(self) -> bool:
        """Delegation to self.data_quality.ingest_ok."""
        return self.data_quality.ingest_ok

    @property
    def is_stale(self) -> bool:
        """Delegation to self.data_quality.is_stale."""
        return self.data_quality.is_stale
```

---

## C. Feed Adapter Contracts

### C.1 Exceptions

```python
# src/feeds/exceptions.py
from __future__ import annotations


class FeedError(Exception):
    """Base class for all feed adapter errors."""
    def __init__(self, source: str, message: str) -> None:
        self.source = source
        super().__init__(f"[{source}] {message}")


class FeedTimeoutError(FeedError):
    """
    A single attempt to the upstream API timed out. Tenacity catches this and retries.
    Raised by: any feed adapter when an HTTP/SDK call exceeds its per-attempt timeout.
    """


class FeedRetryExhaustedError(FeedError):
    """
    All tenacity retry attempts have been exhausted without a successful response.
    Raised by: the tenacity `reraise=False` callback after max attempts.
    Callers (ingest/run_ingest.py) catch this and set the feed's SourceFreshness.ok=False.
    """
    def __init__(self, source: str, attempts: int, last_error: Exception) -> None:
        self.attempts = attempts
        self.last_error = last_error
        super().__init__(source, f"Exhausted {attempts} attempts. Last error: {last_error}")


class CostGuardExceeded(FeedError):
    """
    A Databento request was refused because its estimated cost exceeds the configured
    threshold (MAX_DATABENTO_REQUEST_USD or MAX_DATABENTO_MONTHLY_USD).
    Raised by: ingest/cost_guard.py before the SDK call is made.
    Callers log this and DO NOT retry; a cost guard refusal is intentional, not transient.
    """
    def __init__(self, estimated_usd: float, threshold_usd: float, label: str) -> None:
        self.estimated_usd = estimated_usd
        self.threshold_usd = threshold_usd
        super().__init__(
            "databento",
            f"Estimated cost ${estimated_usd:.4f} exceeds {label} threshold "
            f"${threshold_usd:.2f}. Request refused.",
        )
```

### C.2 BarSource Protocol

```python
# src/contracts/bar_source.py
from __future__ import annotations

from datetime import date
from typing import Protocol, runtime_checkable

import pandas as pd

from ..feeds.exceptions import CostGuardExceeded, FeedRetryExhaustedError, FeedTimeoutError


@runtime_checkable
class BarSource(Protocol):
    """
    Protocol for the bar data feed adapter. The concrete implementation is
    DatabentoBarSource in src/feeds/bars_databento.py.

    All date parameters are ET dates (America/New_York). Returned DataFrames have
    UTC-normalized timestamps in a 'ts' column. The caller (normalizer.py) performs
    session tagging and contract stamping after receiving the raw DataFrame.

    Raises:
        FeedTimeoutError: on a per-attempt timeout (tenacity will retry).
        FeedRetryExhaustedError: when all retry attempts are exhausted.
        CostGuardExceeded: when cost_guard.py refuses the request before making it.
    """

    def fetch_bars(
        self,
        symbol: str,
        start_date: date,
        end_date: date,
    ) -> pd.DataFrame:
        """
        Fetch 1-minute OHLCV bars for [start_date, end_date] inclusive (ET dates).
        Returns a DataFrame with columns:
            ts (datetime, UTC), open (float), high (float), low (float),
            close (float), volume (int), contract (str).
        Rows are sorted ascending by ts.
        Empty DataFrame is valid (holiday / no data); raises only on fetch failure.
        """
        ...

    def estimate_cost_usd(
        self,
        symbol: str,
        start_date: date,
        end_date: date,
    ) -> float:
        """
        Return the Databento metadata.get_cost() estimate in USD for the bars request.
        Called by cost_guard.py BEFORE fetch_bars. Must not make the actual data request.
        """
        ...
```

### C.3 SettlementSource Protocol

```python
# src/contracts/bar_source.py  (continued, same file)
from datetime import date
from typing import Optional, Protocol

import pandas as pd


class SettlementSource(Protocol):
    """
    Protocol for the official settlement + open-interest feed.
    Concrete implementation: DatabentoBarSource (same class; two protocols, one impl).
    Settlements come from the Databento `statistics` schema, never from bars.

    Raises: FeedTimeoutError, FeedRetryExhaustedError, CostGuardExceeded.
    """

    def fetch_settlements(
        self,
        symbol: str,
        start_date: date,
        end_date: date,
    ) -> pd.DataFrame:
        """
        Fetch official settlements for [start_date, end_date] inclusive.
        Returns a DataFrame with columns:
            settle_date (date, ET), symbol (str), settlement_price (float),
            open_interest (int), contract (str).
        One row per (settle_date, symbol). Empty DataFrame is valid.
        """
        ...

    def fetch_settlement_for_date(
        self,
        symbol: str,
        settle_date: date,
    ) -> Optional[float]:
        """
        Convenience method: return the settlement price for a single date, or None
        if not available. Used by engine.py for the gap computation guard.
        """
        ...
```

### C.4 CalendarSource Protocol

```python
from datetime import date
from typing import Protocol

import pandas as pd


class CalendarSource(Protocol):
    """
    Protocol for the economic calendar feed. Concrete implementation: FMPCalendarSource
    in src/feeds/econ_calendar.py.

    Raises: FeedTimeoutError, FeedRetryExhaustedError (no CostGuard; not Databento).
    """

    def fetch_events(
        self,
        event_date: date,
    ) -> pd.DataFrame:
        """
        Fetch economic events for event_date (ET date).
        Returns a DataFrame with columns:
            event_date (date), event_name (str), scheduled_et (str, "HH:MM"),
            importance (str, FMP's raw: "Low"/"Medium"/"High"),
            actual (str, nullable), forecast (str, nullable), previous (str, nullable).
        Empty DataFrame is valid (no events scheduled). Never raises on empty; raises
        only on fetch failure (network error, auth failure, etc.).
        """
        ...
```

### C.5 EarningsSource Protocol

```python
from datetime import date
from typing import Protocol

import pandas as pd


class EarningsSource(Protocol):
    """
    Protocol for the earnings calendar feed. Concrete implementation: FMPEarningsSource
    in src/feeds/earnings.py.

    Raises: FeedTimeoutError, FeedRetryExhaustedError.
    """

    def fetch_earnings(
        self,
        report_date: date,
    ) -> pd.DataFrame:
        """
        Fetch earnings reports for report_date (ET date) for NDX_COMPONENTS tickers.
        Returns a DataFrame with columns:
            symbol (str), report_date (date), timing (str, 'bmo' or 'amc'),
            eps_estimate (float, nullable), eps_actual (float, nullable).
        Empty DataFrame is valid. Filter to NDX_COMPONENTS is applied by the adapter,
        not by the caller.
        """
        ...
```

### C.6 VixSource Protocol

```python
from datetime import date
from typing import Optional, Protocol


class VixSource(Protocol):
    """
    Protocol for the VIX prior-close feed. Concrete implementation: CBOEVixSource
    in src/feeds/vix.py (primary: CBOE daily CSV; fallback: yfinance ^VIX).

    Raises: FeedRetryExhaustedError when both CBOE and yfinance fail.
    Does NOT raise FeedTimeoutError directly; the adapter handles timeouts internally
    and retries before exhaustion.
    """

    def fetch_prior_close(
        self,
        as_of_date: date,
    ) -> Optional[float]:
        """
        Return the most recent available VIX prior close as of as_of_date (ET date).
        Returns None when the most recent available VIX value is older than
        MAX_VIX_STALENESS_DAYS calendar days relative to as_of_date. The caller then
        uses VIX_FALLBACK_REGIME and logs the staleness.
        """
        ...
```

### C.7 Cost Guard Contract

```python
# src/ingest/cost_guard.py
from __future__ import annotations

from datetime import date

from ..contracts.bar_source import BarSource
from ..feeds.exceptions import CostGuardExceeded
from ..settings import Settings


def estimate_and_guard(
    source: BarSource,
    symbol: str,
    start_date: date,
    end_date: date,
    settings: Settings,
    label: str = "per-request",
) -> float:
    """
    Call source.estimate_cost_usd(), log the estimate, and raise CostGuardExceeded
    if the estimate exceeds settings.MAX_DATABENTO_REQUEST_USD.

    Returns the estimated cost in USD when the guard passes.

    This function MUST be called before every fetch_bars or fetch_settlements call.
    The caller receives the cost estimate for logging; it is also persisted to api_spend
    as the pre-request estimate (actual cost updated post-request).

    Raises:
        CostGuardExceeded: when estimate > MAX_DATABENTO_REQUEST_USD.
        Any exception from source.estimate_cost_usd() propagates as-is (treat as
        FeedRetryExhaustedError at the call site if the SDK raises).
    """
    ...


def check_monthly_budget(
    mtd_actual_usd: float,
    estimated_usd: float,
    settings: Settings,
) -> None:
    """
    Check month-to-date spend plus the current request estimate against thresholds.

    - If mtd_actual_usd + estimated_usd >= MAX_DATABENTO_MONTHLY_USD: raise CostGuardExceeded
      with label='monthly-cap'.
    - If mtd_actual_usd + estimated_usd >= DATABENTO_MONTHLY_ALERT_THRESHOLD_USD: log a
      WARNING and publish an MQTT alert but do NOT raise.

    mtd_actual_usd is the sum of actual_cost_usd from api_spend for the current calendar month.
    """
    ...
```

### C.8 Retry Policy

All feed adapters that make network calls use tenacity with these concrete settings.
The values below are the contract; deviating from them requires updating this doc.

```python
# Paste this block at the top of each feed adapter module that uses tenacity.
# src/feeds/_retry.py  (shared helper, imported by each adapter)
from __future__ import annotations

import tenacity

# Applies to: Databento SDK calls, FMP HTTP calls, CBOE HTTP calls, yfinance calls.
# Does NOT apply to cost_guard.estimate_cost_usd() (that is a metadata call, not data).
FEED_RETRY_POLICY = dict(
    stop=tenacity.stop_after_attempt(3),
    wait=tenacity.wait_exponential(multiplier=1, min=2, max=10),
    retry=tenacity.retry_if_exception_type((
        TimeoutError,
        ConnectionError,
        # Databento SDK: databento.BentoError subclasses that are transient
    )),
    reraise=False,          # on exhaustion, raise FeedRetryExhaustedError (see below)
    before_sleep=tenacity.before_sleep_log(logger, logging.WARNING),
)
# Callers wrap with:
#   @tenacity.retry(**FEED_RETRY_POLICY, retry_error_callback=_make_exhausted_error)
# where _make_exhausted_error wraps the last exception in FeedRetryExhaustedError.
```

Concrete values: 3 attempts maximum; wait 2 seconds after attempt 1, up to 10 seconds.
Per-attempt HTTP timeout: 30 seconds (set at the HTTP client / SDK session level, not in
tenacity). These values are intentional given the 06:30 ingest SLA of < 10 minutes total.

---

## D. Persistence Contract

All DDL is managed in `src/store/schema.sql` with `CREATE TABLE IF NOT EXISTS`. Schema
changes require a migration file under `src/store/migrations/V{n}__{description}.sql` and a
version bump in `schema_version`. The migration runner in `db.py` applies all pending
migrations at startup in version order.

### D.9 Concurrency and migration policy (read before the DDL)

DuckDB is a single-writer, multiple-reader embedded database. The architecture uses one
container and one process, so there is no concurrent-write concern at the process level.
However, APScheduler jobs and the FastAPI request handler share the same process.

Rules:
- All DuckDB write operations (INSERT, UPDATE, UPSERT) acquire an `asyncio.Lock` managed
  in `db.py` before executing. The lock is a module-level singleton.
- Read-only queries (SELECT) do NOT acquire the write lock. DuckDB supports concurrent reads.
- The alternative pattern (write to a staging file, then `COPY`) may be used for bulk
  ingest batches. When used, the lock is held for the duration of the RENAME/REPLACE.
- Schema migrations run once at startup before APScheduler starts, holding the lock.

Upsert semantics per table: see "Upsert key" column in each DDL block.

### D.1 bars

```sql
-- Upsert key: (ts, symbol) -- INSERT OR REPLACE
-- Retention: indefinite (~50 MB / 5 years for one instrument)
-- Index: on (ts, symbol) for time-range queries; on (symbol, session) for level aggregations.
CREATE TABLE IF NOT EXISTS bars (
    ts              TIMESTAMPTZ NOT NULL,   -- UTC; primary sort key
    symbol          VARCHAR     NOT NULL,   -- 'NQ.c.0'
    contract        VARCHAR     NOT NULL,   -- resolved explicit month, e.g. 'NQH5'
    open            DOUBLE      NOT NULL,
    high            DOUBLE      NOT NULL,
    low             DOUBLE      NOT NULL,
    close           DOUBLE      NOT NULL,
    volume          BIGINT      NOT NULL,
    session         VARCHAR     NOT NULL,   -- 'RTH' or 'ETH'; never NULL
    source          VARCHAR     NOT NULL DEFAULT 'databento',
    PRIMARY KEY (ts, symbol)
);

CREATE INDEX IF NOT EXISTS idx_bars_symbol_session ON bars (symbol, session, ts);
```

### D.2 settlements

```sql
-- Upsert key: (settle_date, symbol) -- INSERT OR REPLACE
-- Retention: indefinite
-- Source: Databento `statistics` schema exclusively. Never derived from bars.
CREATE TABLE IF NOT EXISTS settlements (
    settle_date       DATE        NOT NULL,   -- ET date; prior-day settlement
    symbol            VARCHAR     NOT NULL,   -- 'NQ.c.0'
    contract          VARCHAR     NOT NULL,   -- resolved explicit month, e.g. 'NQH5'
    settlement_price  DOUBLE      NOT NULL,
    open_interest     BIGINT,                 -- NULL if not returned by statistics schema
    source            VARCHAR     NOT NULL DEFAULT 'databento',
    PRIMARY KEY (settle_date, symbol)
);
```

### D.3 computed_levels

```sql
-- Upsert key: (level_date, symbol) -- INSERT OR REPLACE
-- Retention: BRIEF_HISTORY_DAYS (90 days); pruned by daily job.
-- Written once per day by engine.py at 06:30 ingest; overwritten if engine re-runs.
CREATE TABLE IF NOT EXISTS computed_levels (
    level_date              DATE        NOT NULL,
    symbol                  VARCHAR     NOT NULL,
    computed_at             TIMESTAMPTZ NOT NULL,   -- UTC

    -- Prior RTH levels
    pdh                     DOUBLE      NOT NULL,
    pdl                     DOUBLE      NOT NULL,
    pdc                     DOUBLE      NOT NULL,
    prior_settle            DOUBLE      NOT NULL,

    -- Overnight levels
    onh                     DOUBLE      NOT NULL,
    onl                     DOUBLE      NOT NULL,
    on_open                 DOUBLE      NOT NULL,
    on_close                DOUBLE      NOT NULL,
    on_vwap                 DOUBLE      NOT NULL,

    -- Conditional weekly / monthly (NULL when not available or not in proximity)
    weekly_high             DOUBLE,
    weekly_low              DOUBLE,
    monthly_high            DOUBLE,
    monthly_low             DOUBLE,

    -- Volatility
    atr_14                  DOUBLE,     -- NULL if insufficient history
    avg_onr_20              DOUBLE,     -- NULL if insufficient history

    -- Gap fields
    current_price           DOUBLE      NOT NULL,
    gap_pts                 DOUBLE      NOT NULL,
    gap_pct                 DOUBLE      NOT NULL,
    gap_as_atr_frac         DOUBLE,     -- NULL when atr_14 is NULL

    -- Roll flags
    is_roll_day             BOOLEAN     NOT NULL DEFAULT FALSE,
    roll_from_contract      VARCHAR,    -- NULL when is_roll_day=FALSE
    roll_to_contract        VARCHAR,    -- NULL when is_roll_day=FALSE

    -- Proximity flags (pre-computed)
    weekly_high_in_proximity    BOOLEAN NOT NULL DEFAULT FALSE,
    weekly_low_in_proximity     BOOLEAN NOT NULL DEFAULT FALSE,
    monthly_high_in_proximity   BOOLEAN NOT NULL DEFAULT FALSE,
    monthly_low_in_proximity    BOOLEAN NOT NULL DEFAULT FALSE,

    -- Session classification
    session_type            VARCHAR     NOT NULL,   -- 'FULL', 'HALF', or 'HOLIDAY'

    PRIMARY KEY (level_date, symbol)
);
```

### D.4 calendar_events

```sql
-- Upsert key: event_id (hash of date + normalized event_name + scheduled_et) -- INSERT OR REPLACE
-- Retention: 30 days; pruned by daily job.
-- event_id computed as: md5(event_date || '|' || lower(trim(event_name)) || '|' || scheduled_et)
-- This provides idempotency across repeated FMP calls for the same day (G5).
CREATE TABLE IF NOT EXISTS calendar_events (
    event_id        VARCHAR     NOT NULL PRIMARY KEY,
    event_date      DATE        NOT NULL,           -- ET date
    event_name      VARCHAR     NOT NULL,
    scheduled_et    VARCHAR     NOT NULL,           -- "HH:MM" format in ET
    importance      VARCHAR     NOT NULL,           -- FMP raw: 'Low', 'Medium', 'High'
    actual          VARCHAR,
    forecast        VARCHAR,
    previous        VARCHAR,
    fetched_at      TIMESTAMPTZ NOT NULL            -- UTC; when this row was written
);

CREATE INDEX IF NOT EXISTS idx_calendar_events_date ON calendar_events (event_date);
```

### D.5 earnings_events

```sql
-- Upsert key: event_id (hash of report_date + symbol + timing) -- INSERT OR REPLACE
-- Retention: 30 days; pruned by daily job.
CREATE TABLE IF NOT EXISTS earnings_events (
    event_id        VARCHAR     NOT NULL PRIMARY KEY,
    symbol          VARCHAR     NOT NULL,
    report_date     DATE        NOT NULL,   -- ET date of the earnings report
    timing          VARCHAR     NOT NULL,   -- 'bmo' (before market open) or 'amc' (after market close)
    eps_estimate    DOUBLE,
    eps_actual      DOUBLE,
    fetched_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_earnings_events_date ON earnings_events (report_date);
```

### D.6 brief_snapshots

```sql
-- Upsert key: brief_date -- INSERT OR REPLACE
-- Retention: BRIEF_HISTORY_DAYS (90 days); pruned by daily job.
-- brief_json is the serialized BriefSnapshot (model_dump_json()). It is the source of truth.
-- brief_html and email_html are render caches; they may be regenerated from brief_json.
-- NOTE: brief_json contains levels=null when session_type='HOLIDAY' (no RTH session).
CREATE TABLE IF NOT EXISTS brief_snapshots (
    brief_date      DATE        NOT NULL PRIMARY KEY,
    brief_json      TEXT        NOT NULL,       -- BriefSnapshot.model_dump_json()
    brief_html      TEXT,                       -- rendered fragment HTML; NULL until rendered
    email_html      TEXT,                       -- rendered email HTML; NULL until sent
    email_sent_at   TIMESTAMPTZ,               -- NULL if not sent
    email_error     TEXT,                       -- error message if send failed; NULL on success
    ingest_ok       BOOLEAN     NOT NULL,       -- mirrors DataQuality.ingest_ok for quick queries
    issues          TEXT[],                     -- mirrors DataQuality.issues
    catalyst_level  VARCHAR     NOT NULL DEFAULT 'NONE',  -- mirrors BriefSnapshot.catalyst_level
    generated_at    TIMESTAMPTZ NOT NULL,
    logic_version   VARCHAR     NOT NULL
);
```

### D.7 api_spend

```sql
-- Upsert key: request_id (UUID generated by cost_guard.py) -- INSERT only; never updated except
-- to fill actual_cost_usd after the request completes.
-- Retention: indefinite (small table; monthly rollup adequate).
CREATE TABLE IF NOT EXISTS api_spend (
    request_id          VARCHAR     NOT NULL PRIMARY KEY,
    requested_at        TIMESTAMPTZ NOT NULL,
    source              VARCHAR     NOT NULL DEFAULT 'databento',
    symbol              VARCHAR     NOT NULL,
    start_date          DATE        NOT NULL,
    end_date            DATE        NOT NULL,
    schema_name         VARCHAR     NOT NULL,   -- 'ohlcv-1m' or 'statistics'
    estimated_cost_usd  DOUBLE      NOT NULL,
    actual_cost_usd     DOUBLE,                 -- NULL until post-request; filled by cost_guard
    status              VARCHAR     NOT NULL    -- 'pending', 'completed', 'refused', 'failed'
);

CREATE INDEX IF NOT EXISTS idx_api_spend_month ON api_spend (
    DATE_TRUNC('month', requested_at), source
);
```

### D.8 schema_version

```sql
-- Single-row table. Migration runner reads version, applies V{n} files in order, updates.
CREATE TABLE IF NOT EXISTS schema_version (
    version     INTEGER     NOT NULL,
    applied_at  TIMESTAMPTZ NOT NULL,
    description VARCHAR     NOT NULL
);
-- Seed row (only inserted if table is empty):
-- INSERT INTO schema_version VALUES (1, now(), 'initial schema');
```

---

## E. HTTP / API Contract

All routes are mounted in `src/api/main.py`. The FastAPI `app` has `docs_url=None` and
`redoc_url=None` in production. Response models reference the pydantic models defined in
section B. Auth for admin endpoints uses a `Bearer {ADMIN_TOKEN}` header check in a
FastAPI dependency (`src/api/deps.py`).

```python
# Route summary table (prose below expands each entry)
# METHOD  PATH                        AUTH     RESPONSE MODEL         STATUS CODES
# GET     /                           none     HTML (Jinja2)          200, 503
# GET     /api/brief/latest           none     BriefSnapshot (JSON)   200, 404, 503
# GET     /api/brief/{date}           none     BriefSnapshot (JSON)   200, 404, 410
# GET     /health                     none     HealthResponse (JSON)  200, 503
# GET     /api/fragment/brief         none     HTML fragment          200, 204, 503
# POST    /api/admin/run-now          Bearer   AdminRunResponse       202, 401, 409
# GET     /api/admin/runs             Bearer   list[RunRecord]        200, 401
```

### Route details

```python
# src/api/routes_dashboard.py

# GET /
# Renders base.html + dashboard.html using the latest brief_snapshot.
# If no brief exists for today and it is a holiday (SessionType.HOLIDAY),
# renders the "no session today" state (HTTP 200, not 404).
# On a HOLIDAY, snapshot.levels is None; templates must guard with "if snapshot.levels".
# If no brief exists and the service has never generated one, returns HTTP 503
# with a body indicating the service is initializing.
# HTMX poll: dashboard.html includes hx-get="/api/fragment/brief" hx-trigger="every 30s"
# gated off by Alpine.js after 07:30 ET (server-injected ET hour variable stops the poll).
# No client-side Date math; ET hour is injected as template variable `et_hour` (int 0-23).

# GET /api/fragment/brief
# HTMX endpoint. Returns the brief_fragment.html partial (not a full page).
# Used by the dashboard's auto-refresh during the 07:00-07:30 ET pre-market window.
# Returns HTTP 204 (No Content) when brief_date == today and is_stale=False
# (signals HTMX to keep current content unchanged).
# Returns HTTP 200 with the fragment when new content is available.
# Returns HTTP 503 when ingest is actively running (brief not yet ready).
```

```python
# src/api/routes_api.py

from datetime import date
from fastapi import APIRouter, Depends, HTTPException, Header
from pydantic import BaseModel

from ..models import BriefSnapshot

router = APIRouter(prefix="/api")


# GET /api/brief/latest
# Returns the most recently generated BriefSnapshot as JSON.
# 200: BriefSnapshot (even if risk_mode=GRAY; caller checks data_quality.ingest_ok).
# 404: No brief has ever been generated (service not yet initialized).
# 503: Brief for today is not yet available and the ingest job is running.


# GET /api/brief/{date}
# Path param: date in "YYYY-MM-DD" format (ET date).
# 200: BriefSnapshot for the requested date.
# 404: No brief exists for the requested date (not a trading day, or before service start).
# 410 Gone: The date is older than BRIEF_HISTORY_DAYS and has been pruned.
# Validation: if date is in the future, raise HTTP 422.


# GET /health
# No auth required. Used by Uptime Kuma and the Docker healthcheck.
class HealthResponse(BaseModel):
    status: str                         # "ok" or "degraded"
    last_brief_date: date | None        # ET date of the most recent brief_snapshot
    last_ingest_utc: str | None         # ISO UTC timestamp of the last successful ingest
    db_reachable: bool                  # True if DuckDB can be queried
    mtd_spend_usd: float                # month-to-date Databento spend from api_spend table

# 200: HealthResponse with status="ok" when db_reachable=True and last_brief is recent.
# 503: HealthResponse with status="degraded" when db_reachable=False or last_brief is
#      missing for a trading day that should have one.


# POST /api/admin/run-now
# Requires: Authorization: Bearer {ADMIN_TOKEN}
# Triggers an immediate ingest + brief generation job outside the scheduled window.
# 202 Accepted: job enqueued; returns AdminRunResponse with job_id.
# 401 Unauthorized: missing or invalid token.
# 409 Conflict: a job is already running; returns AdminRunResponse with is_running=True.
class AdminRunResponse(BaseModel):
    job_id: str
    is_running: bool
    message: str


# GET /api/admin/runs?days=7
# Requires: Authorization: Bearer {ADMIN_TOKEN}
# Returns the last N days of job run records from brief_snapshots.
# Default: days=7. Max: days=90 (capped at BRIEF_HISTORY_DAYS).
# 200: list[RunRecord]
# 401: missing or invalid token.
class RunRecord(BaseModel):
    brief_date: date
    generated_at_utc: str
    ingest_ok: bool
    issues: list[str]
    risk_mode: str
    email_sent_at: str | None
    logic_version: str
```

---

## F. Template Context Contract

All templates use Jinja2 with `StrictUndefined`. If a variable referenced in a template is
not present in the context dict, Jinja2 raises `UndefinedError` at render time. Tests in
`tests/test_templates.py` render every template against a fixture `BriefSnapshot` (including
a holiday snapshot with `risk_mode=GRAY` and `session_type=HOLIDAY`) and assert no
`UndefinedError`. This is the template contract test (build_plan G11t).

ET time injection rule: the server injects `et_hour` (Python `int`, 0-23, from
`datetime.now(DST_ZONE).hour`) into every template context. Templates MUST NOT use JavaScript
`new Date()` or any client-side timezone math. Alpine.js may use `et_hour` as a data
attribute after the server has injected it into the HTML.

### F.1 base.html context

```python
# Every page that extends base.html receives this context dict.
# base_context: dict[str, object] = {
#     "et_hour": int,           # 0-23; server-side ET hour at render time
#     "et_date_str": str,       # "Mon May 26, 2025" formatted for display
#     "service_version": str,   # from LOGIC_VERSION; shown in footer
#     "db_reachable": bool,     # shown in footer health indicator
# }
```

### F.2 dashboard.html context

```python
# dashboard.html extends base.html. Context includes all of base_context plus:
# dashboard_context: dict[str, object] = {
#     **base_context,
#     "snapshot": BriefSnapshot | None,
#         # None when no brief has been generated yet (service initializing state).
#         # Templates must check `if snapshot` before accessing any field.
#     "is_holiday": bool,
#         # True when snapshot is None and today is a CME holiday;
#         # or when snapshot.session_type == SessionType.HOLIDAY.
#     "risk_mode_display": str,
#         # RISK_MODE_DISPLAY[snapshot.risk_mode.value] pre-looked-up server-side.
#         # Empty string when snapshot is None.
#     "poll_active": bool,
#         # True when et_hour < 8 (before 08:00 ET); Alpine.js uses this to gate the HTMX poll.
#         # The poll stops after 07:30 ET; this flag gives Alpine its initial value.
# }
# Note: BriefSnapshot is passed as `snapshot`, not destructured. Templates access
# snapshot.levels.onh, snapshot.catalysts, etc. directly. This avoids context sprawl
# and makes it obvious when a field is added to the model.
```

### F.3 brief_fragment.html context

```python
# brief_fragment.html is the HTMX partial. It is NOT a full page and does NOT extend base.html.
# It renders only the brief content block (risk badge, headline, catalyst banner,
# price ladder, volatility + sizing line).
# fragment_context: dict[str, object] = {
#     "snapshot": BriefSnapshot,   # always present; 204 returned instead of rendering when stale
#     "risk_mode_display": str,    # pre-looked-up display string
#     "et_hour": int,              # for the "generated at HH:MM ET" line
#     "ladder_levels": list[dict], # pre-sorted for price ladder rendering; see below
# }
#
# ladder_levels schema (server assembles, template only renders):
# Each dict:
# {
#     "label": str,        # e.g. "ONH", "PDH", "ON VWAP", "ONL"
#     "price": float,      # index points
#     "distance_pts": float,  # current_price - price; signed; positive = above current
#     "is_nearest_above": bool,  # True for the single nearest level above current price
#     "is_nearest_below": bool,  # True for the single nearest level below current price
#     "is_current": bool,  # True for the "NOW" marker row
# }
# Levels are sorted descending by price. The "NOW" row is inserted at the correct position.
# Weekly/monthly levels are included only when their *_in_proximity flag is True.
```

### F.4 email.html context

```python
# email.html is a standalone table-based email template (no base.html inheritance).
# Inline CSS is applied via premailer before sending. The template must render
# correctly with images blocked (Outlook, Apple Mail fallback behavior).
# email_context: dict[str, object] = {
#     "snapshot": BriefSnapshot,
#     "risk_mode_display": str,
#     "et_date_str": str,          # "Monday, May 26, 2025"
#     "et_time_str": str,          # "7:00 AM ET"
#     "logic_version": str,
#     "design_tokens": DESIGN_TOKENS,   # see below
#     "ladder_table_rows": list[dict],  # same schema as ladder_levels in F.3
#         # For email, rendered as an HTML table, not an SVG.
#         # Columns: label, price, distance_pts.
#         # Bolded rows: is_nearest_above=True or is_nearest_below=True.
#     "sizing_display": str | None,
#         # Pre-rendered sizing line for email. None when PER_TRADE_RISK_USD is unset.
#         # Example: "~0 NQ or ~1 MNQ (advisory) at a 105-pt stop."
# }

# DESIGN_TOKENS canonical structure (passed to email.html for inline style generation,
# and imported by tests in tests/snapshot/test_email_fidelity.py and
# tests/snapshot/test_accessibility.py).
#
# There is exactly ONE DESIGN_TOKENS dict, keyed by two top-level categories:
#   "risk"     -- per-RiskMode badge colors, keyed by mode string
#   "semantic" -- global semantic color aliases for body, background, surface, etc.
#
# DESIGN_TOKENS: dict[str, dict] = {
#     "risk": {
#         "GREEN":  {"bg": "#22c55e", "fg": "#ffffff"},
#         "YELLOW": {"bg": "#eab308", "fg": "#000000"},
#         "RED":    {"bg": "#ef4444", "fg": "#ffffff"},
#         "GRAY":   {"bg": "#6b7280", "fg": "#ffffff"},
#     },
#     "semantic": {
#         "body_text":   "#f1f5f9",    # primary text color (used by accessibility test)
#         "background":  "#0f172a",    # email/dashboard background (dark)
#         "surface":     "#1e293b",    # card/section surface
#         "muted":       "#94a3b8",    # secondary / muted text
#         "border":      "#334155",    # table borders
#     },
#     "font": {
#         "family":      "Inter, Arial, sans-serif",
#         "mono":        "'JetBrains Mono', 'Courier New', monospace",
#         "size_base":   "15px",
#         "size_mono":   "14px",
#     },
#     "spacing": {
#         "cell_pad":    "8px 12px",
#         "section_gap": "16px",
#     },
# }
#
# Usage in tests:
#   Badge bg color:     DESIGN_TOKENS["risk"][risk_mode]["bg"]
#   Badge fg color:     DESIGN_TOKENS["risk"][risk_mode]["fg"]
#   Body text:          DESIGN_TOKENS["semantic"]["body_text"]
#   Background:         DESIGN_TOKENS["semantic"]["background"]
#
# Badge text must be present in the email even when background color is blocked.
# The badge cell must include the text "GREEN" / "YELLOW" / "RED" / "GRAY" as literal
# text content (not just background color), so Outlook plain-text mode is legible.
# Tests assert this (build_plan G23).
```

---

*End of contracts document.*
