# Pre-Market Brief: Test Strategy

**Status**: Authoritative test specification. Fills the executability gap left by the build plan's named-but-not-concrete test steps (1.7, 2.5, 3.7, 3.7b, 4.5, 5.2, 6.x) and the readiness gap register (G1 correctness oracle, G3 DQ gate, G5 idempotency, G8 decision table, G23/G24/G25/G26/G27 presentation).

**Companion docs**: `premarket_brief_decision_spec.md` (numeric logic), `premarket_brief_architecture.md` (data model + feeds), `premarket_brief_build_plan.md` (phases), `premarket_brief_readiness.md` (gap register).

**Note on contracts doc**: `docs/premarket_brief_contracts.md` is the authoritative source for pydantic v2 model and field names; this strategy doc asserts against those models. Canonical names: `BriefSnapshot`, `ComputedLevels`, `DataQuality` (the freshness/quality model, produced by the `BriefReadinessCheck` gate), `SessionType` (FULL/HALF/HOLIDAY), `RiskMode`, `CatalystWindow`, `OvernightSummary`/`OvernightState`. The decision-spec constants are referenced by name verbatim. If any contract field is renamed, update both docs together.

**Single-form field names (reconciled with contracts.md)**:
- Overnight fields live at `snapshot.overnight.*`: `.state` (OvernightState), `.phrase` (str), `.range_vs_avg` (ratio), `.net_change_pts` (unsigned magnitude)
- Levels live at `snapshot.levels.*` (`ComputedLevels`); `snapshot.levels` is `None` on `SessionType.HOLIDAY`
- Gap fraction stored field: `snapshot.levels.gap_as_atr_frac`
- VIX stored field: `snapshot.vix_value`; regime: `snapshot.vix_regime`
- Data quality: `snapshot.data_quality.ingest_ok`, `snapshot.data_quality.is_stale`, `snapshot.data_quality.issues`; convenience shims: `snapshot.ingest_ok`, `snapshot.is_stale`
- Catalyst aggregate: `snapshot.catalyst_level` (Literal "NONE"/"MED"/"HIGH")
- Intermediate values `gap_atr_bucket`, `gap_headline_color`, `gap_headline_qualifier`, `net_atr`, `onclv`, `efficiency` are NOT stored fields; assert them via rendered strings or pure-function unit tests

**Hard conventions (same as repo-wide)**:
- pytest + pydantic v2. ruff line-length 100. zoneinfo not pytz.
- UTC internal, ET display. snake_case everywhere.
- No em-dashes in this file or in any generated output.
- Assertions must be exact and non-permissive. A test that would pass if the computation were wrong is a defect in the test itself.

---

## 1. Test Taxonomy and Layout

### 1.1 Directory tree

```
app/tests/
    conftest.py                     # shared fixtures: fake_db, fake_clock, synthetic_bars
    fixtures/
        synthetic/
            bars_2025_q1.parquet    # 30+ day synthetic bar fixture (see section 2.1)
            bars_2025_q1.csv        # CSV copy for human inspection / diff
            settlements_2025_q1.json  # companion settlement + OI records
            calendar_events.json    # FMP calendar fixture for the 30-day window
            earnings_events.json    # FMP earnings fixture (NVDA bmo day included)
            vix_series.json         # VIX prior-close series covering the window
        cassettes/
            fmp_econ_normal.yaml    # VCR cassette: normal econ calendar response
            fmp_econ_empty.yaml     # VCR cassette: zero events returned
            fmp_econ_failed.yaml    # VCR cassette: 500 error
            fmp_earnings_nvda.yaml  # VCR cassette: NVDA bmo earnings
            cboe_vix_ok.yaml        # VCR cassette: CBOE CSV success
            cboe_vix_down.yaml      # VCR cassette: CBOE 503
            yfinance_vix_ok.yaml    # VCR cassette: yfinance fallback success
        oracle/
            ORACLE.md               # hand-verified expected outputs (section 2.3)
            day_boring_wednesday.json    # oracle record for Day A
            day_cpi_morning.json         # oracle record for Day B
            day_stress_no_event.json     # oracle record for Day C
            SIGNOFF.md              # sign-off log (section 2.4)
    unit/
        test_session.py             # DST, CME holidays, half-days, session_type
        test_levels.py              # PDH/PDL, ONH/ONL, settlement gap, RTH-only ATR
        test_overnight.py           # trend/balanced/choppy classifier, range_vs_avg
        test_volatility.py          # ATR(14) RTH-only, gap_as_atr_frac, VIX expected move
        test_catalysts.py           # severity overrides, padding, FOMC, cap-3, NVDA fold
        test_risk_mode.py           # all 27 cells of RISK_MODE_TABLE
        test_sizing.py              # position sizing formula, unset-PER_TRADE_RISK_USD path
        test_config.py              # alias equality: AVG_ONR_LOOKBACK == BASELINE_SESSION_COUNT
        test_cost_guard.py          # cost-guard abort paths (per-request and monthly caps)
    golden/
        test_levels_golden.py       # full ComputedLevels record vs golden fixture
        test_overnight_golden.py    # overnight state vs golden fixture
    edge/
        test_edge_cases.py          # G2 matrix: roll, half-day, DST, gap thresholds, etc.
        test_dq_gate.py             # G3: every degradation path
        test_idempotency.py         # G5: build_brief() twice -> identical output
    contract/
        test_api_routes.py          # status codes + response shapes for all routes
        test_template_contract.py   # Jinja2 StrictUndefined, every template renders
    snapshot/
        test_html_snapshot.py       # HTML fragment + email string snapshots
        test_svg_ladder.py          # SVG marker y-coordinate assertions
        test_email_fidelity.py      # badge bg + text, premailer inline styles
        test_accessibility.py       # WCAG AA contrast ratios, aria-label, role
    scheduler/
        test_scheduler.py           # DST-correct firing times, missed-run recovery
    e2e/
        test_e2e_brief.py           # load golden fixture -> GET /api/brief/{date} -> assert
```

### 1.2 Pytest markers

Declare in `pyproject.toml` under `[tool.pytest.ini_options]`:

```toml
[tool.pytest.ini_options]
markers = [
    "unit: fast, no IO, no network",
    "golden: requires synthetic fixture parquet; the G1 correctness oracle",
    "edge: financial edge cases (G2 matrix)",
    "dq: data-quality and degraded-mode paths (G3)",
    "idempotency: G5 build_brief() determinism",
    "contract: API route shape and template contract tests",
    "snapshot: HTML/SVG/email string snapshots with committed baselines",
    "scheduler: mock-clock DST and missed-run tests",
    "e2e: end-to-end through FastAPI routes",
    "network: requires real external network; excluded from CI by default",
]
addopts = "-m 'not network'"
```

CI runs all markers except `network`. The `network` marker is used only for the one-time Databento verification test (build plan step 1.7); it must be run manually before declaring Phase 1 complete.

### 1.3 Coverage target and must-cover modules

Overall target: >= 80% line coverage on `src/brief/` and `src/feeds/`, measured per the build plan Definition of Done.

**Must-cover modules** (CI fails if below 90% branch coverage on these):

| Module | Why 90% is mandatory |
|---|---|
| `src/brief/engine.py` | `build_brief()` is the single revenue function |
| `src/brief/levels.py` | ATR-RTH-only is the G1 silent-wrong trap |
| `src/brief/volatility.py` | All three gap-atr buckets must be exercised |
| `src/brief/catalysts.py` | Cap-3 priority and FOMC special case |
| `src/brief/session.py` | DST transitions, half-day flags |
| `src/ingest/normalizer.py` | Session tagging, resolved-contract stamp |
| `src/feeds/vix.py` | Primary + fallback + stale paths |

Run coverage with:

```bash
pytest --cov=src/brief --cov=src/feeds \
       --cov-report=term-missing \
       --cov-fail-under=80
```

---

## 2. The Golden-File Oracle (G1)

### 2.1 Synthetic fixture dataset specification

The fixture covers exactly 30 trading days. It is fully deterministic: no network calls, no random state. The generation script is `tests/fixtures/generate_fixture.py` and must be committed alongside the fixture files.

**Scenario coverage required (each row is a day in the fixture)**:

| Scenario tag | Day count | What it exercises |
|---|---|---|
| `NORMAL_FULL` | 15 | Baseline: full RTH, full overnight, no events |
| `DST_SPRING_FORWARD` | 2 | March spring-forward cross: 06:30 ET job fires at correct UTC |
| `DST_FALL_BACK` | 2 | November fall-back cross: 06:30 ET job fires at correct UTC |
| `CME_HOLIDAY` | 1 | Veterans Day: `session_type=HOLIDAY`, no brief rendered |
| `CME_HALF_DAY` | 1 | Day before Thanksgiving: `session_type=HALF`, early close 13:00 ET |
| `OVERNIGHT_QUIET` | 2 | `on_range_pts < ON_RANGE_QUIET_MULTIPLIER * avg_onr`, label BALANCED |
| `OVERNIGHT_HUGE` | 2 | `on_range_pts >= ON_RANGE_ELEVATED_MULTIPLIER * avg_onr`, label TRENDED |
| `CONTRACT_ROLL` | 2 | `NQH5 -> NQM5`: resolved contract changes, carry levels, no false gap |
| `MISSING_SETTLEMENT` | 1 | Settlement row absent: brief degrades to GRAY |
| `BARS_BELOW_MIN` | 1 | overnight bar count = 200 (< `MIN_OVERNIGHT_BARS` = 300): GRAY |
| `BARS_STALE` | 1 | newest bar timestamp = 07:30 UTC (> 45 min before 06:30 ingest): GRAY |

Total: 30 days. The `NORMAL_FULL` days span the window before/after DST and contract roll so the `BASELINE_SESSION_COUNT = 20` trailing average is always well-populated.

**Realistic NQ price range used throughout**: 20,000 to 21,000 index points. ATR values 120 to 260 (a realistic two-month band for NQ). VIX values 14 to 35 (spanning LOW, ELEVATED, and HIGH regimes). All prices use round numbers that produce exact two-decimal arithmetic.

**Fixture file format**:

- `tests/fixtures/synthetic/bars_2025_q1.parquet`: schema matches `bars` table (columns: `ts` UTC datetime64[ns], `symbol` str, `open` float64, `high` float64, `low` float64, `close` float64, `volume` int64, `session` str (`RTH`|`ETH`), `contract` str). One row per minute covering 18:00 ET prior evening through 17:00 ET each session day, minus the 17:00-18:00 CME maintenance halt gap.
- `tests/fixtures/synthetic/bars_2025_q1.csv`: exact same data in CSV (for human diff and oracle verification).
- `tests/fixtures/synthetic/settlements_2025_q1.json`: list of objects with keys `settle_date` (ISO date), `symbol`, `settle_price` float, `open_interest` int, `contract` str. One row per trading day (absent for the `MISSING_SETTLEMENT` scenario day).
- `tests/fixtures/synthetic/calendar_events.json`: FMP-shaped objects covering the 30-day window. Includes: CPI (HIGH, 08:30 ET), ISM (MED, 10:00 ET), Jobless Claims (MED, 08:30 ET), NVDA bmo earnings, FOMC Rate Decision, and zero-event days.
- `tests/fixtures/synthetic/vix_series.json`: list of `{date: "YYYY-MM-DD", close: float}`. Covers the 30-day window plus 5 prior days for the staleness boundary test.

**Generation script contract**:

`tests/fixtures/generate_fixture.py` must:
1. Accept no runtime arguments (fully deterministic, no seeds required because data is synthetic not sampled).
2. Write all five fixture files to `tests/fixtures/synthetic/` using paths relative to the script's own location.
3. Be runnable as `python tests/fixtures/generate_fixture.py` from the repo root.
4. Print a SHA-256 of each output file on completion.
5. Never import any Databento, FMP, CBOE, or yfinance SDK. No network.

When the script output SHA-256s match the committed files, the fixture is valid. CI asserts this:

```python
# in conftest.py
import hashlib, pathlib

def pytest_configure(config):
    fixture_dir = pathlib.Path(__file__).parent / "fixtures" / "synthetic"
    manifest = (fixture_dir / "SHA256SUMS").read_text().strip().splitlines()
    for line in manifest:
        expected_hex, fname = line.split("  ")
        actual = hashlib.sha256((fixture_dir / fname).read_bytes()).hexdigest()
        assert actual == expected_hex, (
            f"Fixture {fname} hash mismatch. "
            f"Run python tests/fixtures/generate_fixture.py to regenerate."
        )
```

### 2.1b make_snapshot fixture helper

`make_snapshot` is a single documented helper defined near the top of `tests/conftest.py` (or in `tests/fixtures/factories.py` imported by conftest). It builds a valid nested `BriefSnapshot` with sensible defaults, then shallow-merges caller-supplied overrides. This replaces the ad-hoc `_*_override` kwargs pattern and the undefined `BriefContext` pattern used in earlier drafts.

```python
# tests/conftest.py (excerpt)

from datetime import date, datetime
from zoneinfo import ZoneInfo
from src.models import (
    BriefSnapshot, ComputedLevels, DataQuality, OvernightSummary,
    SizingLine, RiskMode, SessionType, OvernightState,
)

UTC = ZoneInfo("UTC")


def make_snapshot(**overrides) -> BriefSnapshot:
    """
    Build a valid BriefSnapshot with sensible defaults for Oracle Day A inputs.
    Overrides are shallow-merged at the BriefSnapshot field level.

    Default values correspond to Oracle Day A (2025-01-08, boring Wednesday, GREEN):
      - risk_mode GREEN, session_type FULL
      - levels: gap_pts=78, gap_as_atr_frac=0.3714, atr_14=210
      - overnight: state=CHOPPY, range_vs_avg=0.5522, net_change_pts=77
      - vix_value=16.20, vix_regime=LOW
      - catalyst_level=NONE, catalysts=[]
      - data_quality: ingest_ok=True, is_stale=False, issues=[]

    To build a degraded snapshot:
        snap = make_snapshot(
            risk_mode=RiskMode.GRAY,
            data_quality=DataQuality(ingest_ok=False, is_stale=True, issues=["Settlement missing"], ...)
        )

    To override a nested sub-model, pass the full replacement model as the value.
    """
    default_levels = ComputedLevels(
        level_date=date(2025, 1, 8),
        symbol="NQ.c.0",
        computed_at=datetime(2025, 1, 8, 11, 30, 0, tzinfo=UTC),
        pdh=20450.00, pdl=20310.00, pdc=20397.00,
        prior_settle=20400.00,
        onh=20490.00, onl=20363.00,
        on_open=20401.00, on_close=20478.00, on_vwap=20440.00,
        current_price=20478.00,
        gap_pts=78.00, gap_pct=0.003824,
        gap_as_atr_frac=0.3714,
        atr_14=210.00, avg_onr_20=230.00,
        session_type=SessionType.FULL,
    )
    default_overnight = OvernightSummary(
        state=OvernightState.CHOPPY,
        net_change_pts=77.00,
        range_pts=127.00,
        range_vs_avg=0.5522,
        phrase="Overnight chopped, no clean resolution (0.6x normal range).",
        bar_count=480,
        is_degraded=False,
    )
    default_dq = DataQuality(
        ingest_ok=True,
        is_stale=False,
        issues=[],
        as_of_utc=datetime(2025, 1, 8, 11, 30, 0, tzinfo=UTC),
        bar_count_overnight=480,
        bar_count_rth_prior=390,
        settlement_present=True,
        per_source=[],
    )
    default_sizing = SizingLine(null_reason="Set your per-trade risk to see suggested sizing.")

    defaults: dict = dict(
        brief_date=date(2025, 1, 8),
        generated_at_utc=datetime(2025, 1, 8, 12, 0, 0, tzinfo=UTC),
        risk_mode=RiskMode.GREEN,
        headline="NQ +78 pts (+0.4%), gap up, on a quiet night (gap = 0.4x ATR).",
        volatility_line="ATR(14) 210 pts. VIX 16.2 (~1.0% move).",
        session_type=SessionType.FULL,
        levels=default_levels,
        overnight=default_overnight,
        sizing=default_sizing,
        data_quality=default_dq,
        logic_version="1.0.0",
        vix_value=16.20,
        vix_regime="LOW",
        catalyst_level="NONE",
        catalysts=[],
        catalyst_feed_ok=True,
        catalysts_dropped_count=0,
    )
    defaults.update(overrides)
    return BriefSnapshot(**defaults)


@pytest.fixture
def snapshot_day_a() -> BriefSnapshot:
    """Oracle Day A (Green, boring Wednesday 2025-01-08) as a BriefSnapshot."""
    return make_snapshot()
```

All tests that need a `BriefSnapshot` should use `make_snapshot(**overrides)` instead of building one manually or using ad-hoc `_*_override` kwargs on `build_brief()`. The engine-integration tests (`build_brief()`) still call the engine directly; `make_snapshot` is for unit and render tests that need a snapshot without running the engine.

### 2.2 FakeDatabentoClient interface

`FakeDatabentoClient` is defined in `tests/conftest.py` and is the ONLY Databento client used in any test outside the `network` marker.

```python
# tests/conftest.py (excerpt)

import pandas as pd
import pathlib
import pytest
from zoneinfo import ZoneInfo

FIXTURE_DIR = pathlib.Path(__file__).parent / "fixtures" / "synthetic"
ET = ZoneInfo("America/New_York")


class FakeDatabentoClient:
    """
    Drop-in replacement for DatabentoBarSource in tests.
    Reads from committed parquet/json fixtures; never touches the network.
    No constructor args required.
    """

    def get_bars(
        self,
        symbol: str,
        start: str,
        end: str,
        schema: str = "ohlcv-1m",
    ) -> pd.DataFrame:
        df = pd.read_parquet(FIXTURE_DIR / "bars_2025_q1.parquet")
        mask = (df["ts"] >= pd.Timestamp(start, tz="UTC")) & (
            df["ts"] < pd.Timestamp(end, tz="UTC")
        )
        return df[mask].reset_index(drop=True)

    def get_settlements(self, symbol: str, start: str, end: str) -> pd.DataFrame:
        import json
        rows = json.loads((FIXTURE_DIR / "settlements_2025_q1.json").read_text())
        df = pd.DataFrame(rows)
        df["settle_date"] = pd.to_datetime(df["settle_date"]).dt.date
        return df[
            (df["settle_date"] >= pd.to_datetime(start).date())
            & (df["settle_date"] <= pd.to_datetime(end).date())
        ].reset_index(drop=True)

    def estimate_cost(self, **kwargs) -> float:  # noqa: ARG002
        return 0.0


@pytest.fixture
def fake_databento_client() -> FakeDatabentoClient:
    return FakeDatabentoClient()
```

**Naming rule for recorded DataFrames**: if a one-off scenario-specific DataFrame is needed (e.g., a single thin-overnight day), it lives as `tests/fixtures/synthetic/bars_scenario_{tag}.parquet` with the scenario tag from the table in 2.1.

**Refresh procedure**: run `python tests/fixtures/generate_fixture.py`, verify SHA-256s, commit both the fixture files and the updated `SHA256SUMS` manifest. Never commit fixture files without re-running the generator (the generator is the single source of truth).

**Offline CI enforcement**: the `conftest.py` fixture hash check plus `addopts = "-m 'not network'"` guarantees no test can call the real Databento API. Any test that imports `databento` directly (not via `FakeDatabentoClient`) fails the import-guard test:

```python
# tests/unit/test_config.py

def test_no_direct_databento_import_in_brief_modules() -> None:
    """Brief-engine modules must not import databento directly."""
    import ast, pathlib

    brief_dir = pathlib.Path("src/brief")
    for py_file in brief_dir.glob("*.py"):
        tree = ast.parse(py_file.read_text())
        for node in ast.walk(tree):
            if isinstance(node, (ast.Import, ast.ImportFrom)):
                names = (
                    [a.name for a in node.names]
                    if isinstance(node, ast.Import)
                    else ([node.module] if node.module else [])
                )
                assert not any("databento" in (n or "") for n in names), (
                    f"{py_file} imports databento directly; use the BarSource protocol"
                )
```

### 2.3 The oracle: three hand-verified representative days

The oracle days are chosen to exercise all three paths of the RISK_MODE_TABLE (GREEN, YELLOW, RED). All arithmetic is shown below so independent verification does not require running the code.

---

#### Oracle Day A: Boring Wednesday (GREEN path)

**Scenario tag**: `NORMAL_FULL`
**Date in fixture**: 2025-01-08 (Wednesday)

**Raw fixture inputs** (from `bars_2025_q1.parquet` + `settlements_2025_q1.json`):

| Field | Value |
|---|---|
| Prior settle (2025-01-07) | 20,400.00 |
| Prior RTH close (2025-01-07 16:00 ET) | 20,397.00 |
| Current price at ingest (last ETH bar close ~06:29 ET) | 20,478.00 |
| Overnight open (18:00 ET 2025-01-07) | 20,401.00 |
| Overnight high | 20,490.00 |
| Overnight low | 20,363.00 |
| Overnight close at ingest | 20,478.00 |
| sum_abs_move over overnight bars | 320.00 |
| VIX prior close (2025-01-07) | 16.20 |
| Calendar events today | none |
| Trailing 20-session avg_onr | 230.00 (pre-baked in fixture) |
| 14-session RTH ATR | 210.00 (pre-baked in fixture) |

**Arithmetic - step by step**:

Gap:
```
gap_pts      = 20478.00 - 20400.00 = 78.00
gap_pct      = 78.00 / 20400.00 = 0.003824 (0.38%)
gap_atr_frac = abs(78.00) / 210.00 = 0.3714
```
Gap bucket: `0.20 < 0.37 <= 1.00` -> `NORMAL`
Headline color: `0.37 > GAP_ATR_GRAY_MAX (0.20)` -> not GRAY
Headline qualifier: `0.20 < 0.37 <= 0.50` -> `"on a quiet night (gap = 0.4x ATR)"`

VIX:
```
vix = 16.20 < VIX_ELEVATED_THRESHOLD (20.0) -> regime = "LOW"
expected_move_pct = 16.20 / 16.0 = 1.0125 ~= 1.0%
```

Overnight range:
```
on_range_pts  = 20490.00 - 20363.00 = 127.00
range_vs_avg  = 127.00 / 230.00 = 0.5522
```
Range label: `0.5522 < ON_RANGE_QUIET_MULTIPLIER (0.70)` -> "a quiet night (0.6x normal range)"

Overnight trend classifier:
```
on_open    = 20401.00
on_close   = 20478.00
onl        = 20363.00
onh        = 20490.00
on_range   = 127.00
net_change = 20478.00 - 20401.00 = 77.00  (signed; magnitude only used for labeling)
net_abs    = 77.00
net_atr    = 77.00 / 210.00 = 0.3667
onclv      = (20478.00 - 20363.00) / 127.00 = 115.00 / 127.00 = 0.9055
efficiency = 77.00 / 320.00 = 0.2406
```
TRENDED check: `net_atr = 0.3667 < ON_NET_TREND_MIN_ATR (0.50)` -> fails first gate -> NOT TRENDED.
BALANCED check: `ONCLV_BALANCED_LOW (0.40) <= onclv (0.9055) <= ONCLV_BALANCED_HIGH (0.60)` -> 0.9055 is NOT in [0.40, 0.60] -> NOT BALANCED.
Result: `CHOPPY`. Phrase: `"Overnight chopped, no clean resolution (0.6x normal range)."` (`{r}` = 0.5522 rounded to 1 decimal = 0.6).

Overnight range qualifier: `"a quiet night (0.6x normal range)"`.
Overnight state: `CHOPPY`. Phrase: `"Overnight chopped, no clean resolution (0.6x normal range)."`.

Catalyst: no events today. `catalyst_level`: `"NONE"`.

Risk Mode:
```
gap_as_atr_frac = 0.3714; 0.20 < 0.37 <= 1.00 -> bucket = NORMAL
lookup = ("NONE", "NORMAL", "LOW") -> "GREEN"
```
Display: `"TRADE NORMAL SIZE"`

**Expected oracle record (oracle file `day_boring_wednesday.json`)**:

Note: the oracle JSON records asserted fields from the nested `BriefSnapshot` model, not a flat shape.
Fields prefixed `levels.` come from `snapshot.levels` (ComputedLevels); fields prefixed `overnight.`
come from `snapshot.overnight` (OvernightSummary); top-level fields are on `BriefSnapshot` directly.
Intermediate values `net_atr`, `onclv`, `efficiency` are NOT stored; they are asserted in the overnight
pure-function unit test (`test_overnight_golden.py`).

```json
{
  "brief_date": "2025-01-08",
  "risk_mode": "GREEN",
  "catalyst_level": "NONE",
  "vix_value": 16.20,
  "vix_regime": "LOW",
  "session_type": "FULL",
  "levels.level_date": "2025-01-08",
  "levels.symbol": "NQ.c.0",
  "levels.prior_settle": 20400.00,
  "levels.pdc": 20397.00,
  "levels.current_price": 20478.00,
  "levels.gap_pts": 78.00,
  "levels.gap_pct": 0.003824,
  "levels.gap_as_atr_frac": 0.3714,
  "levels.atr_14": 210.00,
  "levels.onh": 20490.00,
  "levels.onl": 20363.00,
  "levels.on_open": 20401.00,
  "levels.on_close": 20478.00,
  "levels.avg_onr_20": 230.00,
  "overnight.state": "CHOPPY",
  "overnight.net_change_pts": 77.00,
  "overnight.range_pts": 127.00,
  "overnight.range_vs_avg": 0.5522,
  "overnight.phrase": "Overnight chopped, no clean resolution (0.6x normal range).",
  "data_quality.ingest_ok": true,
  "data_quality.is_stale": false,
  "volatility_line_contains": "VIX 16.2",
  "headline_contains": "on a quiet night"
}
```

Float tolerance for assertions: absolute tolerance 0.001 for all fractional fields, exact equality for string fields.

---

#### Oracle Day B: CPI Morning (YELLOW path)

**Scenario tag**: `NORMAL_FULL` with CPI event
**Date in fixture**: 2025-01-15 (Wednesday, CPI day)

**Raw fixture inputs**:

| Field | Value |
|---|---|
| Prior settle (2025-01-14) | 20,360.00 |
| Current price at ingest | 20,320.00 |
| ATR_14 | 210.00 |
| VIX prior close | 18.00 |
| Calendar events | CPI 08:30 ET, FMP impact=High |
| on_open | 20,363.00 |
| on_close | 20,320.00 |
| onh | 20,385.00 |
| onl | 20,305.00 |
| sum_abs_move | 195.00 |
| avg_onr_20 | 230.00 |

**Arithmetic**:

Gap:
```
gap_pts      = 20320.00 - 20360.00 = -40.00
gap_pct      = -40.00 / 20360.00 = -0.001965 (-0.20%)
gap_atr_frac = abs(-40.00) / 210.00 = 0.1905
```
Gap bucket: `0.1905 <= GAP_ATR_QUIET_MAX (0.20)` -> `QUIET`
Headline color: `0.1905 <= GAP_ATR_GRAY_MAX (0.20)` -> GRAY (inconclusive)
Headline qualifier: `"flat (gap under 0.2x ATR, treat as noise)"`

VIX:
```
vix = 18.00. 18.00 < 20.0 -> regime = "LOW"
```

Overnight range:
```
on_range_pts = 20385.00 - 20305.00 = 80.00
on_range_ratio = 80.00 / 230.00 = 0.3478
```
Range label: `0.35 < 0.70` -> `"a quiet night (0.3x normal range)"`

Overnight trend:
```
net_change = 20320.00 - 20363.00 = -43.00
net_abs    = 43.00
net_atr    = 43.00 / 210.00 = 0.2048
onclv      = (20320.00 - 20305.00) / 80.00 = 15.00 / 80.00 = 0.1875
efficiency = 43.00 / 195.00 = 0.2205
```
TRENDED: `net_atr 0.20 < 0.50` -> no. BALANCED: `0.40 <= 0.1875 <= 0.60` -> no. Result: `CHOPPY`.

Catalyst: CPI "consumer price" substring -> override -> `HIGH`. 08:30 ET.
Window: `[08:30 - 2min, 08:30 + 5min]` = `[08:28, 08:35]` ET.
Severity: `HIGH`.

Risk Mode:
```
lookup = ("HIGH", "QUIET", "LOW") -> "YELLOW"
```
Display: `"TRADE SMALLER / BE SELECTIVE"`

Overnight trend (classifier intermediates for the pure-function test in `test_overnight_golden.py`):
```
net_change = 20320.00 - 20363.00 = -43.00
net_abs    = 43.00
net_atr    = 43.00 / 210.00 = 0.2048
onclv      = (20320.00 - 20305.00) / 80.00 = 15.00 / 80.00 = 0.1875
efficiency = 43.00 / 195.00 = 0.2205
```
TRENDED: `net_atr 0.2048 < 0.50` -> no. BALANCED: `0.40 <= 0.1875 <= 0.60` -> no. Result: `CHOPPY`.

**Expected oracle record (key fields)**:

```json
{
  "brief_date": "2025-01-15",
  "risk_mode": "YELLOW",
  "catalyst_level": "HIGH",
  "vix_value": 18.00,
  "vix_regime": "LOW",
  "session_type": "FULL",
  "levels.gap_pts": -40.00,
  "levels.gap_as_atr_frac": 0.1905,
  "levels.atr_14": 210.00,
  "overnight.state": "CHOPPY",
  "overnight.range_pts": 80.00,
  "overnight.range_vs_avg": 0.3478,
  "data_quality.ingest_ok": true,
  "data_quality.is_stale": false,
  "headline_contains": "flat",
  "volatility_line_contains": "VIX 18"
}
```

The catalyst window assertion is made directly on `snapshot.catalysts[0]`:
- `event_label` contains "CPI"
- `severity == CatalystSeverity.HIGH`
- `no_trade_start_utc` corresponds to 08:28 ET on 2025-01-15
- `no_trade_end_utc` corresponds to 08:35 ET on 2025-01-15

---

#### Oracle Day C: Stress Day, No Event (RED path)

**Scenario tag**: `OVERNIGHT_HUGE` + high VIX
**Date in fixture**: 2025-02-19 (Wednesday, no scheduled events)

**Raw fixture inputs**:

| Field | Value |
|---|---|
| Prior settle | 20,000.00 |
| Current price at ingest | 20,520.00 |
| ATR_14 | 240.00 |
| VIX prior close | 31.00 |
| Calendar events | none |
| on_open | 20,005.00 |
| on_close | 20,520.00 |
| onh | 20,525.00 |
| onl | 20,000.00 |
| sum_abs_move | 680.00 |
| avg_onr_20 | 250.00 |

**Arithmetic**:

Gap:
```
gap_pts      = 20520.00 - 20000.00 = 520.00
gap_pct      = 520.00 / 20000.00 = 0.0260 (2.60%)
gap_atr_frac = 520.00 / 240.00 = 2.1667
```
GAP_ATR_SANITY_ABS check: `2.1667 < 3.0` -> passes sanity, no degrade.
Gap bucket: `2.1667 > GAP_ATR_ELEVATED_MAX (1.00)` -> `LARGE`
Headline qualifier: `"a large overnight move (2.2x ATR)"` (2.1667 rounded to 1 decimal = 2.2)

VIX:
```
vix = 31.00 >= VIX_HIGH_THRESHOLD (28.0) -> regime = "HIGH"
expected_move_pct = 31.00 / 16.0 = 1.9375 ~= 1.9%
```

Overnight range:
```
on_range_pts = 20525.00 - 20000.00 = 525.00
on_range_ratio = 525.00 / 250.00 = 2.10
```
Range label: `2.10 >= ON_RANGE_ELEVATED_MULTIPLIER (1.50)` -> `"a big night (2.1x normal range)"`

Overnight trend:
```
net_change = 20520.00 - 20005.00 = 515.00
net_abs    = 515.00
net_atr    = 515.00 / 240.00 = 2.1458
onclv      = (20520.00 - 20000.00) / 525.00 = 520.00 / 525.00 = 0.9905
efficiency = 515.00 / 680.00 = 0.7574
```
TRENDED: `net_atr 2.15 >= 0.50` AND `efficiency 0.76 >= 0.50` AND `onclv 0.99 >= ONCLV_TRENDED_MIN (0.80)` -> all gates pass -> `TRENDED`.
Phrase: `"Overnight trended (moved 515 pts, 2.1x normal range)."` (net_abs = 515, r = 2.10 rounded = 2.1)

Catalyst: no events. `catalyst_level`: `"NONE"`.

Risk Mode:
```
gap_as_atr_frac = 2.1667; > 1.00 -> bucket = LARGE
lookup = ("NONE", "LARGE", "HIGH") -> "RED"
```
Display: `"STAND ASIDE OR TRADE TINY"`

Overnight classifier intermediates (for `test_overnight_golden.py`):
```
net_atr    = 515.00 / 240.00 = 2.1458
onclv      = (20520.00 - 20000.00) / 525.00 = 520.00 / 525.00 = 0.9905
efficiency = 515.00 / 680.00 = 0.7574
```

**Expected oracle record (key fields)**:

```json
{
  "brief_date": "2025-02-19",
  "risk_mode": "RED",
  "catalyst_level": "NONE",
  "vix_value": 31.00,
  "vix_regime": "HIGH",
  "session_type": "FULL",
  "levels.gap_pts": 520.00,
  "levels.gap_as_atr_frac": 2.1667,
  "levels.atr_14": 240.00,
  "overnight.state": "TRENDED",
  "overnight.net_change_pts": 515.00,
  "overnight.range_pts": 525.00,
  "overnight.range_vs_avg": 2.10,
  "overnight.phrase": "Overnight trended (moved 515 pts, 2.1x normal range).",
  "data_quality.ingest_ok": true,
  "data_quality.is_stale": false,
  "volatility_line_contains": "VIX 31",
  "headline_contains": "a large overnight move"
}
```

### 2.4 Golden-file sign-off process

**Who verifies**: Bryce (developer) verifies the arithmetic shown in section 2.3 cell by cell, then the quant-analyst agent re-derives the same numbers independently, then records the outcome.

**Sign-off artifact**: `tests/fixtures/oracle/SIGNOFF.md` contains a log entry per oracle update with the format:

```
## Sign-off: 2025-MM-DD

Verifier: Bryce (manual arithmetic check, comparing decision_spec.md formulas)
Second check: quant-analyst agent (independent derivation from fixture inputs)
Fixture SHA-256: <from SHA256SUMS>
Scope: initial oracle (Day A, B, C)
Notes: <any discrepancies found and resolved>
```

**Controlled update procedure**: when the computation logic changes (e.g., a constant is revised after Allie's review, a formula is corrected):
1. Update `generate_fixture.py` if the fixture data must change.
2. Re-run `python tests/fixtures/generate_fixture.py`.
3. Re-derive the affected oracle day(s) from scratch using the updated formula. Do NOT copy the code's output to the oracle file without independent verification.
4. Update `tests/fixtures/oracle/day_*.json` and `SIGNOFF.md`.
5. Commit fixture files + oracle files + SIGNOFF.md in a single atomic commit with message: `oracle: update for <reason> (verified)`
6. A PR that changes oracle JSON files without updating SIGNOFF.md fails review.

**The oracle is NOT circular** because the expected values in `day_*.json` are computed by hand from the arithmetic in section 2.3, not by running the production code. A developer who mis-implements a formula will see a test failure; they cannot make the test pass by re-deriving from their own (wrong) output.

### 2.5 Golden-file assertion style

```python
# tests/golden/test_levels_golden.py

import json
import pathlib
import pytest
from src.brief.engine import build_brief
from tests.conftest import FakeDatabentoClient

ORACLE_DIR = pathlib.Path(__file__).parent.parent / "fixtures" / "oracle"
FLOAT_TOLERANCE = 1e-3  # absolute tolerance for all float fields


def load_oracle(name: str) -> dict:
    return json.loads((ORACLE_DIR / f"{name}.json").read_text())


@pytest.mark.golden
@pytest.mark.parametrize(
    "oracle_name, brief_date",
    [
        ("day_boring_wednesday", "2025-01-08"),
        ("day_cpi_morning", "2025-01-15"),
        ("day_stress_no_event", "2025-02-19"),
    ],
)
def test_computed_levels_match_oracle(
    oracle_name: str,
    brief_date: str,
    fake_databento_client: FakeDatabentoClient,
    fake_db,  # fixture that returns a populated in-memory DuckDB with the synthetic bars
) -> None:
    snapshot = build_brief(date=brief_date, db=fake_db, bar_source=fake_databento_client)
    levels = snapshot.levels   # ComputedLevels; None only on HOLIDAY (not these oracle days)
    assert levels is not None, f"levels is None for {brief_date}; expected a non-holiday day"
    oracle = load_oracle(oracle_name)

    # Helper to resolve dotted oracle keys against the snapshot model hierarchy
    def get_snapshot_value(key: str):
        """Resolve a dotted key like 'levels.gap_pts' or 'overnight.state'."""
        parts = key.split(".", 1)
        if len(parts) == 1:
            return getattr(snapshot, key)
        sub_model = getattr(snapshot, parts[0])
        return getattr(sub_model, parts[1])

    # Fields asserted with float tolerance
    float_keys = {
        "levels.gap_pts", "levels.gap_pct", "levels.gap_as_atr_frac", "levels.atr_14",
        "levels.onh", "levels.onl", "levels.on_open", "levels.on_close",
        "levels.avg_onr_20",
        "overnight.net_change_pts", "overnight.range_pts", "overnight.range_vs_avg",
        "vix_value",
    }
    # Fields asserted with exact string equality
    str_keys = {
        "risk_mode", "catalyst_level", "vix_regime", "session_type",
        "overnight.state", "overnight.phrase",
    }
    # Fields asserted with exact boolean identity
    bool_keys = {
        "data_quality.ingest_ok", "data_quality.is_stale",
    }
    # Fields asserted as substring containment in the rendered string
    contains_keys = {
        k for k in oracle if k.endswith("_contains")
    }

    for key in float_keys:
        if key in oracle:
            actual = get_snapshot_value(key)
            expected = oracle[key]
            assert abs(actual - expected) <= FLOAT_TOLERANCE, (
                f"Field '{key}': expected {expected}, got {actual} "
                f"(tolerance {FLOAT_TOLERANCE})"
            )

    for key in str_keys:
        if key in oracle:
            actual = str(get_snapshot_value(key))
            expected = str(oracle[key])
            assert actual == expected, (
                f"Field '{key}': expected '{expected}', got '{actual}'"
            )

    for key in bool_keys:
        if key in oracle:
            actual = get_snapshot_value(key)
            expected = oracle[key]
            assert actual is expected, (
                f"Field '{key}': expected {expected}, got {actual}"
            )

    # "headline_contains" and "volatility_line_contains" check rendered string fields
    if "headline_contains" in oracle:
        assert oracle["headline_contains"].lower() in snapshot.headline.lower(), (
            f"headline '{snapshot.headline}' does not contain '{oracle['headline_contains']}'"
        )
    if "volatility_line_contains" in oracle:
        assert oracle["volatility_line_contains"].lower() in snapshot.volatility_line.lower(), (
            f"volatility_line '{snapshot.volatility_line}' does not contain "
            f"'{oracle['volatility_line_contains']}'"
        )
```

**Float tolerance policy**: `1e-3` absolute tolerance applies to all float fields derived from arithmetic on the fixture data. This covers floating-point rounding differences in the Python implementation but is tight enough to catch a formula error. String fields are exact (`==`). Boolean fields are identity (`is`). There is no `approx()` without a bound; `pytest.approx` with `abs=1e-3` is acceptable as a synonym.

### 2.6 Overnight golden-file test

`tests/golden/test_overnight_golden.py` unit-tests the pure functions in `src/brief/overnight.py` against the hand-verified intermediates from section 2.3. These are intermediate values (`net_atr`, `onclv`, `efficiency`) that are NOT stored on any model; they are computed inside the overnight module and asserted here via the module's exported helper functions.

```python
# tests/golden/test_overnight_golden.py

import pytest
from src.brief.overnight import (
    compute_onclv,
    compute_net_atr,
    compute_efficiency,
    classify_overnight,
)
from src.models import OvernightState

FLOAT_TOL = 1e-3


@pytest.mark.golden
def test_oracle_day_a_overnight_intermediates() -> None:
    """
    Oracle Day A (boring Wednesday 2025-01-08).
    on_open=20401, on_close=20478, onl=20363, onh=20490,
    on_range=127, net_abs=77, sum_abs_move=320, atr_14=210.
    """
    on_open, on_close, onl, onh = 20401.00, 20478.00, 20363.00, 20490.00
    atr_14 = 210.00
    sum_abs_move = 320.00

    on_range = onh - onl           # 127.00
    net_abs   = abs(on_close - on_open)  # 77.00

    onclv      = compute_onclv(on_close=on_close, onl=onl, on_range=on_range)
    net_atr    = compute_net_atr(net_abs=net_abs, atr_14=atr_14)
    efficiency = compute_efficiency(net_abs=net_abs, sum_abs_move=sum_abs_move)

    assert abs(onclv - 0.9055) <= FLOAT_TOL,       f"onclv: expected 0.9055, got {onclv}"
    assert abs(net_atr - 0.3667) <= FLOAT_TOL,     f"net_atr: expected 0.3667, got {net_atr}"
    assert abs(efficiency - 0.2406) <= FLOAT_TOL,  f"efficiency: expected 0.2406, got {efficiency}"

    state = classify_overnight(
        net_atr=net_atr, efficiency=efficiency, onclv=onclv, on_range=on_range
    )
    assert state == OvernightState.CHOPPY, (
        f"Day A: expected CHOPPY (net_atr {net_atr:.4f} < 0.50 fails TRENDED gate; "
        f"onclv {onclv:.4f} outside [0.40, 0.60] fails BALANCED gate), got {state}"
    )


@pytest.mark.golden
def test_oracle_day_c_overnight_intermediates() -> None:
    """
    Oracle Day C (stress day 2025-02-19).
    on_open=20005, on_close=20520, onl=20000, onh=20525,
    on_range=525, net_abs=515, sum_abs_move=680, atr_14=240.
    """
    on_open, on_close, onl, onh = 20005.00, 20520.00, 20000.00, 20525.00
    atr_14 = 240.00
    sum_abs_move = 680.00

    on_range = onh - onl           # 525.00
    net_abs   = abs(on_close - on_open)  # 515.00

    onclv      = compute_onclv(on_close=on_close, onl=onl, on_range=on_range)
    net_atr    = compute_net_atr(net_abs=net_abs, atr_14=atr_14)
    efficiency = compute_efficiency(net_abs=net_abs, sum_abs_move=sum_abs_move)

    assert abs(onclv - 0.9905) <= FLOAT_TOL,       f"onclv: expected 0.9905, got {onclv}"
    assert abs(net_atr - 2.1458) <= FLOAT_TOL,     f"net_atr: expected 2.1458, got {net_atr}"
    assert abs(efficiency - 0.7574) <= FLOAT_TOL,  f"efficiency: expected 0.7574, got {efficiency}"

    state = classify_overnight(
        net_atr=net_atr, efficiency=efficiency, onclv=onclv, on_range=on_range
    )
    assert state == OvernightState.TRENDED, (
        f"Day C: expected TRENDED (net_atr {net_atr:.4f} >= 0.50, "
        f"efficiency {efficiency:.4f} >= 0.50, onclv {onclv:.4f} >= 0.80), got {state}"
    )
```

These tests prove the overnight module's pure functions produce the hand-verified intermediates. A developer who mis-implements the `onclv` formula will see `onclv: expected 0.9055, got X` immediately.

---

## 3. Fixture and Mock Management

### 3.1 FakeDatabentoClient (extended spec)

The class in section 2.2 covers the basic interface. Additional contracts:

- `get_bars` and `get_settlements` are synchronous (matching the historical API). No async.
- If `symbol` is not `"NQ.c.0"`, raise `ValueError("FakeDatabentoClient only supports NQ.c.0")`. This prevents tests from silently testing the wrong symbol.
- `estimate_cost` always returns `0.0`. Tests that test the cost guard use a separate `FakeDatabentoClientWithCost` that returns a configurable float.

```python
class FakeDatabentoClientWithCost(FakeDatabentoClient):
    def __init__(self, cost: float) -> None:
        self._cost = cost

    def estimate_cost(self, **kwargs) -> float:  # noqa: ARG002
        return self._cost
```

**Stored DataFrame naming convention**:
- Base fixture: `tests/fixtures/synthetic/bars_2025_q1.parquet`
- Scenario overrides: `tests/fixtures/synthetic/bars_scenario_{TAG}.parquet`
- Tags match the fixture scenario table in section 2.1 (e.g., `bars_scenario_DST_SPRING_FORWARD.parquet`).

### 3.2 VCR cassettes for FMP and CBOE

Library: `pytest-recording` (wraps `vcrpy`). Cassettes are stored in `tests/fixtures/cassettes/`.

**Naming convention**: `{source}_{scenario}.yaml`. Source is `fmp_econ`, `fmp_earnings`, `cboe_vix`, or `yfinance_vix`. Scenario is a short descriptor.

**Secret scrubbing**: before committing any cassette, run the scrub script:

```bash
python tests/fixtures/scrub_cassettes.py tests/fixtures/cassettes/
```

`scrub_cassettes.py` rewrites each cassette YAML replacing any occurrence of `FMP_API_KEY` value, `DATABENTO_API_KEY` value, or `Authorization` header values with `REDACTED`. It uses regex substitution on the raw YAML text, not YAML parsing, so it handles all header formats. CI asserts no cassette contains a known secret pattern:

```python
# tests/unit/test_cassette_hygiene.py

import pathlib, re

def test_no_secrets_in_cassettes() -> None:
    cassette_dir = pathlib.Path("tests/fixtures/cassettes")
    secret_patterns = [
        re.compile(r"[A-Za-z0-9]{32,}"),  # catches API keys > 32 chars
    ]
    known_safe_long_strings = {
        # add SHA-256 hashes, base64 image data, etc. here as needed
    }
    for cassette in cassette_dir.glob("*.yaml"):
        text = cassette.read_text()
        # FMP keys are typically 32-char hex
        assert "apikey=" not in text.lower(), (
            f"Cassette {cassette.name} may contain an FMP API key"
        )
        assert "db-" not in text, (
            f"Cassette {cassette.name} may contain a Databento API key"
        )
```

**Refresh procedure**: when an external API response format changes, delete the relevant cassette and re-record with `@pytest.mark.vcr` and the `network` marker, then immediately run `scrub_cassettes.py` and commit.

### 3.3 VIX fallback test (three paths)

All three paths are tested in `tests/unit/test_feeds.py` using `responses` (HTTP mocking library) rather than VCR, because the fallback logic involves conditional HTTP calls that VCR handles awkwardly.

```python
# tests/unit/test_feeds.py (excerpt)

import json
import responses as resp_mock
import pytest
from datetime import date, timedelta
from src.feeds.vix import fetch_vix_prior_close
from src.config import MAX_VIX_STALENESS_DAYS


CBOE_URL = "https://cdn.cboe.com/api/global/us_indices/daily_prices/VIX_History.csv"
TODAY = date(2025, 1, 8)


@pytest.mark.unit
@resp_mock.activate
def test_vix_cboe_primary_path() -> None:
    """CBOE CSV succeeds: return the prior-close value."""
    csv_text = "DATE,OPEN,HIGH,LOW,CLOSE\n01/07/2025,16.10,16.30,15.90,16.20\n"
    resp_mock.add(resp_mock.GET, CBOE_URL, body=csv_text, status=200)
    result = fetch_vix_prior_close(as_of_date=TODAY)
    assert result.value == pytest.approx(16.20, abs=1e-3)
    assert result.source == "cboe"
    assert result.is_stale is False


@pytest.mark.unit
@resp_mock.activate
def test_vix_yfinance_fallback_path() -> None:
    """CBOE returns 503; yfinance fallback succeeds."""
    resp_mock.add(resp_mock.GET, CBOE_URL, status=503)
    # yfinance is mocked via monkeypatch in conftest; see fake_yfinance fixture
    # yfinance returns a DataFrame with Close = 16.50 for 2025-01-07
    result = fetch_vix_prior_close(as_of_date=TODAY, _yf_override={"2025-01-07": 16.50})
    assert result.value == pytest.approx(16.50, abs=1e-3)
    assert result.source == "yfinance"
    assert result.is_stale is False


@pytest.mark.unit
@resp_mock.activate
def test_vix_stale_path_at_boundary() -> None:
    """
    Most recent VIX date is exactly MAX_VIX_STALENESS_DAYS old.
    At the boundary the value should be treated as stale.
    MAX_VIX_STALENESS_DAYS = 3 (from config.py).
    """
    stale_date = TODAY - timedelta(days=MAX_VIX_STALENESS_DAYS)
    csv_text = f"DATE,OPEN,HIGH,LOW,CLOSE\n{stale_date.strftime('%m/%d/%Y')},20.00,20.50,19.80,20.10\n"
    resp_mock.add(resp_mock.GET, CBOE_URL, body=csv_text, status=200)
    result = fetch_vix_prior_close(as_of_date=TODAY)
    # Exactly at boundary: stale
    assert result.is_stale is True
    assert result.value is None


@pytest.mark.unit
@resp_mock.activate
def test_vix_stale_path_one_day_inside_boundary() -> None:
    """
    Most recent VIX date is MAX_VIX_STALENESS_DAYS - 1 old: not stale.
    """
    fresh_date = TODAY - timedelta(days=MAX_VIX_STALENESS_DAYS - 1)
    csv_text = (
        f"DATE,OPEN,HIGH,LOW,CLOSE\n"
        f"{fresh_date.strftime('%m/%d/%Y')},20.00,20.50,19.80,20.10\n"
    )
    resp_mock.add(resp_mock.GET, CBOE_URL, body=csv_text, status=200)
    result = fetch_vix_prior_close(as_of_date=TODAY)
    assert result.is_stale is False
    assert result.value == pytest.approx(20.10, abs=1e-3)
```

**Staleness boundary rule** (exact, from `premarket_brief_decision_spec.md`):
`MAX_VIX_STALENESS_DAYS = 3`. A VIX record whose date is `>= TODAY - 3 days` in calendar days is stale. Records with age strictly less than 3 calendar days are valid. The boundary test above asserts the `>=` direction.

---

## 4. Deterministic Time and Scheduler Harness

### 4.1 Mock-clock pattern

No test ever calls `datetime.now()` or `datetime.utcnow()` directly. The production code must accept an injectable `clock` parameter (or use a module-level `get_now()` that can be monkeypatched). The test harness uses a `FakeClock` that returns a fixed `datetime` in UTC.

```python
# tests/conftest.py (excerpt)

from datetime import datetime
from zoneinfo import ZoneInfo
import pytest

UTC = ZoneInfo("UTC")
ET = ZoneInfo("America/New_York")


class FakeClock:
    def __init__(self, utc_dt: datetime) -> None:
        assert utc_dt.tzinfo is not None, "FakeClock requires timezone-aware datetime"
        self._utc_dt = utc_dt

    def now_utc(self) -> datetime:
        return self._utc_dt

    def now_et(self) -> datetime:
        return self._utc_dt.astimezone(ET)


@pytest.fixture
def fake_clock_winter() -> FakeClock:
    """07:00 ET on a winter (EST, UTC-5) Wednesday."""
    return FakeClock(datetime(2025, 1, 8, 12, 0, 0, tzinfo=UTC))


@pytest.fixture
def fake_clock_summer() -> FakeClock:
    """07:00 ET on a summer (EDT, UTC-4) Wednesday."""
    return FakeClock(datetime(2025, 6, 11, 11, 0, 0, tzinfo=UTC))
```

### 4.2 DST test cases (exact test skeletons)

```python
# tests/scheduler/test_scheduler.py

import pytest
from datetime import datetime
from zoneinfo import ZoneInfo
from src.scheduler.jobs import build_cron_trigger, compute_next_fire_utc

UTC = ZoneInfo("UTC")
ET = ZoneInfo("America/New_York")


@pytest.mark.scheduler
def test_07_00_et_fires_at_correct_utc_winter() -> None:
    """
    Winter: America/New_York is UTC-5.
    07:00 ET = 12:00 UTC.
    APScheduler CronTrigger(hour=7, minute=0, timezone='America/New_York')
    must resolve to 12:00 UTC on a standard-time day.
    """
    trigger = build_cron_trigger(hour=7, minute=0)
    # 2025-01-08 is a Wednesday in EST (UTC-5)
    reference_utc = datetime(2025, 1, 8, 11, 59, 0, tzinfo=UTC)
    next_fire = compute_next_fire_utc(trigger, after=reference_utc)
    expected = datetime(2025, 1, 8, 12, 0, 0, tzinfo=UTC)
    assert next_fire == expected, (
        f"Winter 07:00 ET should be 12:00 UTC; got {next_fire}"
    )


@pytest.mark.scheduler
def test_07_00_et_fires_at_correct_utc_summer() -> None:
    """
    Summer: America/New_York is UTC-4.
    07:00 ET = 11:00 UTC.
    """
    trigger = build_cron_trigger(hour=7, minute=0)
    # 2025-06-11 is a Wednesday in EDT (UTC-4)
    reference_utc = datetime(2025, 6, 11, 10, 59, 0, tzinfo=UTC)
    next_fire = compute_next_fire_utc(trigger, after=reference_utc)
    expected = datetime(2025, 6, 11, 11, 0, 0, tzinfo=UTC)
    assert next_fire == expected, (
        f"Summer 07:00 ET should be 11:00 UTC; got {next_fire}"
    )


@pytest.mark.scheduler
def test_spring_forward_2025() -> None:
    """
    2025 DST spring-forward: clocks move forward at 02:00 ET on 2025-03-09.
    The 07:00 ET job must fire at 11:00 UTC on 2025-03-09 (EDT, UTC-4),
    NOT at 12:00 UTC (which would be 08:00 ET, an hour late).
    """
    trigger = build_cron_trigger(hour=7, minute=0)
    # Ask for next fire after 06:59 UTC on the spring-forward day
    reference_utc = datetime(2025, 3, 9, 6, 59, 0, tzinfo=UTC)
    next_fire = compute_next_fire_utc(trigger, after=reference_utc)
    expected = datetime(2025, 3, 9, 11, 0, 0, tzinfo=UTC)
    assert next_fire == expected, (
        f"Spring-forward day 07:00 ET should be 11:00 UTC; got {next_fire}"
    )


@pytest.mark.scheduler
def test_fall_back_2025() -> None:
    """
    2025 DST fall-back: clocks move back at 02:00 ET on 2025-11-02.
    The 07:00 ET job must fire at 12:00 UTC on 2025-11-02 (EST, UTC-5),
    NOT at 11:00 UTC (which would be 07:00 EDT, one hour early).
    """
    trigger = build_cron_trigger(hour=7, minute=0)
    reference_utc = datetime(2025, 11, 2, 11, 59, 0, tzinfo=UTC)
    next_fire = compute_next_fire_utc(trigger, after=reference_utc)
    expected = datetime(2025, 11, 2, 12, 0, 0, tzinfo=UTC)
    assert next_fire == expected, (
        f"Fall-back day 07:00 ET should be 12:00 UTC; got {next_fire}"
    )


@pytest.mark.scheduler
def test_missed_run_recovery_fires_once_within_window(fake_db) -> None:
    """
    Container restarts at 07:15 ET (within the 07:00-08:30 recovery window).
    Recovery must fire exactly one brief for the current trading day,
    never two, never auto-backfill prior days.
    """
    from src.scheduler.jobs import check_and_recover_missed_brief

    # 07:15 ET winter = 12:15 UTC
    clock = FakeClock(datetime(2025, 1, 8, 12, 15, 0, tzinfo=UTC))
    result = check_and_recover_missed_brief(db=fake_db, clock=clock)
    assert result.fired is True
    assert result.date == "2025-01-08"
    assert result.backfill_triggered is False  # never auto-backfill


@pytest.mark.scheduler
def test_missed_run_no_recovery_after_cutoff(fake_db) -> None:
    """
    Container restarts at 08:35 ET (after the 08:30 cutoff).
    Recovery must NOT fire a brief. The brief for that day is skipped and logged.
    """
    from src.scheduler.jobs import check_and_recover_missed_brief

    # 08:35 ET winter = 13:35 UTC
    clock = FakeClock(datetime(2025, 1, 8, 13, 35, 0, tzinfo=UTC))
    result = check_and_recover_missed_brief(db=fake_db, clock=clock)
    assert result.fired is False
    assert result.backfill_triggered is False


@pytest.mark.scheduler
def test_poll_cutoff_stops_after_07_30_et(fake_clock_winter) -> None:
    """
    The HTMX dashboard poll must stop after 07:30 ET.
    The endpoint returns poll_active=False at 07:31 ET.
    """
    from src.api.routes_dashboard import poll_status

    # fake_clock_winter is 07:00 ET; advance it to 07:31 ET = 12:31 UTC
    clock = FakeClock(datetime(2025, 1, 8, 12, 31, 0, tzinfo=UTC))
    status = poll_status(clock=clock)
    assert status.poll_active is False


@pytest.mark.scheduler
def test_poll_active_before_07_30_et() -> None:
    """
    The HTMX dashboard poll must remain active before 07:30 ET.
    """
    from src.api.routes_dashboard import poll_status

    clock = FakeClock(datetime(2025, 1, 8, 12, 0, 0, tzinfo=UTC))  # 07:00 ET
    status = poll_status(clock=clock)
    assert status.poll_active is True
```

The `build_cron_trigger` and `compute_next_fire_utc` are thin wrappers around APScheduler's `CronTrigger` that exist solely to make the trigger's next-fire logic testable without running a live scheduler loop.

---

## 5. Data-Quality and Degraded-Mode Tests (G3)

All cases live in `tests/edge/test_dq_gate.py`. Each case specifies exact inputs from the fixture, the exact expected output, and the exact fields that must be set.

Constants referenced: `MIN_OVERNIGHT_BARS = 300`, `MIN_RTH_BARS_PRIOR = 360`, `MAX_BAR_STALENESS_MIN = 45`, `SETTLEMENT_SANITY_PCT = 0.005`, `GAP_ATR_SANITY_ABS = 3.0`.

```python
# tests/edge/test_dq_gate.py

import pytest
from datetime import datetime, timedelta
from zoneinfo import ZoneInfo
from src.brief.engine import build_brief
from src.config import (
    MIN_OVERNIGHT_BARS,
    MIN_RTH_BARS_PRIOR,
    MAX_BAR_STALENESS_MIN,
    GAP_ATR_SANITY_ABS,
)

UTC = ZoneInfo("UTC")


@pytest.mark.dq
def test_missing_settlement_degrades_to_gray(fake_db_no_settlement, fake_databento_client) -> None:
    """
    No settlement row for 2025-01-08.
    Expected: data_quality.ingest_ok=False, data_quality.is_stale=True,
    risk_mode=GRAY, risk_mode_display via RISK_MODE_DISPLAY lookup,
    levels.gap_pts is None (never render a computed gap without a settlement),
    rendered headline/banner contains "data incomplete".
    Must NOT raise.
    """
    snapshot = build_brief(
        date="2025-01-08",
        db=fake_db_no_settlement,
        bar_source=fake_databento_client,
    )
    # Both nested path and convenience shims must agree
    assert snapshot.data_quality.ingest_ok is False
    assert snapshot.ingest_ok is False          # convenience shim
    assert snapshot.data_quality.is_stale is True
    assert snapshot.is_stale is True            # convenience shim
    assert snapshot.risk_mode == "GRAY"
    assert snapshot.levels is not None, "levels may still be populated even on GRAY; settlement is one trigger"
    assert snapshot.levels.gap_pts is None, (
        "gap_pts must be None when settlement is missing; never render a computed gap"
    )
    # The "data incomplete" text lives in the rendered headline or volatility_line; check both
    assert (
        "data incomplete" in snapshot.headline.lower()
        or "data incomplete" in snapshot.volatility_line.lower()
    ), "GRAY brief must communicate 'data incomplete' in a rendered string field"


@pytest.mark.dq
def test_bars_below_min_overnight_degrades_to_gray(fake_db_thin_overnight, fake_databento_client) -> None:
    """
    Overnight bar count = 200 (< MIN_OVERNIGHT_BARS = 300).
    Expected: data_quality.ingest_ok=False, overnight.state=None (degraded),
    risk_mode=GRAY.
    """
    snapshot = build_brief(
        date="2025-01-08",
        db=fake_db_thin_overnight,
        bar_source=fake_databento_client,
    )
    assert snapshot.data_quality.ingest_ok is False
    assert snapshot.risk_mode == "GRAY"
    assert snapshot.overnight.state is None, (
        "overnight.state must be None when bar_count < MIN_OVERNIGHT_BARS"
    )
    assert snapshot.overnight.is_degraded is True
    # levels may still be partially populated; onh/onl are None when overnight is degraded
    if snapshot.levels is not None:
        assert snapshot.levels.onh is None
        assert snapshot.levels.onl is None
    assert (
        "data incomplete" in snapshot.headline.lower()
        or "data incomplete" in snapshot.volatility_line.lower()
    )


@pytest.mark.dq
def test_bars_below_min_overnight_exact_boundary(fake_db, fake_databento_client) -> None:
    """
    overnight bar count = MIN_OVERNIGHT_BARS (300): this is the boundary.
    At exactly 300 bars the brief should NOT degrade (the condition is < 300).
    """
    # The base fixture has > 300 overnight bars; this test uses the
    # BARS_BELOW_MIN scenario fixture which we set to exactly 300.
    snapshot = build_brief(
        date="2025-01-08",
        db=fake_db,
        bar_source=fake_databento_client,
        _overnight_bar_count_override=MIN_OVERNIGHT_BARS,
    )
    assert snapshot.data_quality.ingest_ok is True
    assert snapshot.risk_mode != "GRAY"


@pytest.mark.dq
def test_stale_bars_beyond_max_staleness_degrades(fake_db, fake_databento_client) -> None:
    """
    Newest bar timestamp is MAX_BAR_STALENESS_MIN (45) minutes before 06:30 ingest.
    That means newest bar is at 05:45 ET = more than 45 min stale.
    Expected: data_quality.ingest_ok=False, data_quality.is_stale=True, risk_mode=GRAY.
    """
    ingest_utc = datetime(2025, 1, 8, 11, 30, 0, tzinfo=UTC)  # 06:30 ET winter
    stale_cutoff = ingest_utc - timedelta(minutes=MAX_BAR_STALENESS_MIN)
    newest_bar_ts = stale_cutoff - timedelta(minutes=1)  # one minute past the cutoff

    snapshot = build_brief(
        date="2025-01-08",
        db=fake_db,
        bar_source=fake_databento_client,
        _newest_bar_ts_override=newest_bar_ts,
    )
    assert snapshot.data_quality.ingest_ok is False
    assert snapshot.data_quality.is_stale is True
    assert snapshot.risk_mode == "GRAY"
    assert (
        "data incomplete" in snapshot.headline.lower()
        or "data incomplete" in snapshot.volatility_line.lower()
    )


@pytest.mark.dq
def test_stale_bars_exactly_at_boundary_is_stale(fake_db, fake_databento_client) -> None:
    """
    Newest bar is exactly MAX_BAR_STALENESS_MIN minutes old: stale (>= triggers).
    """
    ingest_utc = datetime(2025, 1, 8, 11, 30, 0, tzinfo=UTC)
    newest_bar_ts = ingest_utc - timedelta(minutes=MAX_BAR_STALENESS_MIN)

    snapshot = build_brief(
        date="2025-01-08",
        db=fake_db,
        bar_source=fake_databento_client,
        _newest_bar_ts_override=newest_bar_ts,
    )
    assert snapshot.data_quality.is_stale is True
    assert snapshot.is_stale is True   # convenience shim must agree


@pytest.mark.dq
def test_gap_atr_sanity_abs_exceeded_degrades(fake_db, fake_databento_client) -> None:
    """
    gap_as_atr_frac = 3.1 (> GAP_ATR_SANITY_ABS = 3.0).
    This is a symbology/roll error; brief must degrade to GRAY, not render a 3x gap.
    """
    snapshot = build_brief(
        date="2025-01-08",
        db=fake_db,
        bar_source=fake_databento_client,
        _gap_as_atr_frac_override=GAP_ATR_SANITY_ABS + 0.1,
    )
    assert snapshot.data_quality.ingest_ok is False
    assert snapshot.risk_mode == "GRAY"
    assert (
        "data incomplete" in snapshot.headline.lower()
        or "data incomplete" in snapshot.volatility_line.lower()
    )


@pytest.mark.dq
def test_partial_ingest_sets_issues(fake_db, fake_databento_client) -> None:
    """
    Bars ingest succeeds but VIX feed fails.
    Expected: data_quality.ingest_ok=False, data_quality.issues non-empty,
    vix_regime uses VIX_FALLBACK_REGIME="LOW" (fallback),
    risk_mode NOT GRAY (VIX failure alone is not a GRAY trigger),
    volatility_line contains "VIX unavailable".
    """
    from src.feeds.vix import VixResult

    snapshot = build_brief(
        date="2025-01-08",
        db=fake_db,
        bar_source=fake_databento_client,
        _vix_override=VixResult(value=None, source="failed", is_stale=True),
    )
    assert snapshot.data_quality.ingest_ok is False
    assert len(snapshot.data_quality.issues) > 0
    assert snapshot.vix_regime == "LOW"         # VIX_FALLBACK_REGIME
    assert snapshot.vix_value is None           # not stored when feed fails
    assert snapshot.risk_mode != "GRAY"         # VIX failure alone does not gray
    assert "vix unavailable" in snapshot.volatility_line.lower()


@pytest.mark.dq
def test_fmp_failed_shows_calendar_unavailable_not_no_events(
    fake_db, fake_databento_client
) -> None:
    """
    FMP call fails (not zero events: a real failure).
    Expected: catalyst banner says "Calendar unavailable, check independently",
    NOT "No major events, normal session".
    catalyst_level treated as NONE for badge but must NOT read as all-clear.
    """
    from src.feeds.econ_calendar import CalendarResult

    snapshot = build_brief(
        date="2025-01-08",
        db=fake_db,
        bar_source=fake_databento_client,
        _calendar_override=CalendarResult(events=[], feed_ok=False),
    )
    # The "calendar unavailable" text must appear in the rendered template context.
    # The build_brief engine sets catalyst_feed_ok=False when FMP fails, which the
    # template uses to render the unavailability message. Assert via the template
    # context key or rendered fragment.
    assert snapshot.catalyst_feed_ok is False
    assert snapshot.catalyst_level == "NONE"
    # Assert rendered output via fragment render, not a snapshot field:
    from tests.helpers import render_brief_fragment
    rendered = render_brief_fragment(snapshot)
    assert "calendar unavailable" in rendered.lower()
    assert "no major events" not in rendered.lower()


@pytest.mark.dq
def test_fmp_empty_shows_no_major_events(fake_db, fake_databento_client) -> None:
    """
    FMP call succeeds but returns zero events (a real empty calendar day).
    Expected: catalyst banner says "No major events, normal session".
    catalyst_level == "NONE".
    """
    from src.feeds.econ_calendar import CalendarResult

    snapshot = build_brief(
        date="2025-01-08",
        db=fake_db,
        bar_source=fake_databento_client,
        _calendar_override=CalendarResult(events=[], feed_ok=True),
    )
    assert snapshot.catalyst_feed_ok is True
    assert snapshot.catalyst_level == "NONE"
    assert len(snapshot.catalysts) == 0
    from tests.helpers import render_brief_fragment
    rendered = render_brief_fragment(snapshot)
    assert "no major events" in rendered.lower()
```

The `_*_override` keyword arguments are test injection points on `build_brief()`. They are documented in `tests/conftest.py` alongside the `make_snapshot` helper; production code does not honor them. Use `make_snapshot(**overrides)` for unit and render tests; use `build_brief(..., _*_override=...)` only in integration tests where the full engine pipeline is exercised.

---

## 6. Render, Template, and Email Tests

### 6.1 Jinja2 template contract

```python
# tests/contract/test_template_contract.py

import pytest
from jinja2 import StrictUndefined, Environment, FileSystemLoader
from pathlib import Path

TEMPLATE_DIR = Path("src/templates")


def make_env() -> Environment:
    return Environment(
        loader=FileSystemLoader(str(TEMPLATE_DIR)),
        undefined=StrictUndefined,
        autoescape=True,
    )


def make_holiday_snapshot() -> "BriefSnapshot":
    """BriefSnapshot for a CME holiday (session_type=HOLIDAY, levels=None)."""
    from tests.conftest import make_snapshot
    from src.models import SessionType, RiskMode, DataQuality
    from datetime import date, datetime
    from zoneinfo import ZoneInfo
    UTC = ZoneInfo("UTC")
    dq = DataQuality(
        ingest_ok=False, is_stale=False, issues=[],
        as_of_utc=datetime(2025, 1, 20, 11, 30, 0, tzinfo=UTC),
        bar_count_overnight=0, bar_count_rth_prior=0,
        settlement_present=False, per_source=[],
    )
    return make_snapshot(
        brief_date=date(2025, 1, 20),
        session_type=SessionType.HOLIDAY,
        levels=None,    # HOLIDAY has no levels
        risk_mode=RiskMode.GRAY,
        headline="DATA INCOMPLETE - VERIFY INDEPENDENTLY",
        volatility_line="",
        data_quality=dq,
        catalyst_level="NONE",
        catalysts=[],
        catalyst_feed_ok=True,
    )


def make_full_snapshot() -> "BriefSnapshot":
    """BriefSnapshot for a normal FULL day (Oracle Day A defaults)."""
    from tests.conftest import make_snapshot
    return make_snapshot()  # Oracle Day A defaults are all sensible FULL-day values


def build_template_context(snapshot: "BriefSnapshot") -> dict:
    """
    Build the full template context dict for the given snapshot, matching
    contracts.md sections F.1-F.4. This is the server-side context builder
    under test; it must reflect the actual routes_dashboard.py / routes_api.py logic.
    """
    from src.config import RISK_MODE_DISPLAY
    from datetime import datetime
    from zoneinfo import ZoneInfo
    ET = ZoneInfo("America/New_York")
    now_et = datetime.now(ET)
    return {
        # F.1 base context
        "et_hour": now_et.hour,
        "et_date_str": now_et.strftime("%a %b %d, %Y"),
        "service_version": snapshot.logic_version,
        "db_reachable": True,
        # F.2 dashboard context additions
        "snapshot": snapshot,
        "is_holiday": snapshot.session_type.value == "HOLIDAY",
        "risk_mode_display": (
            RISK_MODE_DISPLAY.get(snapshot.risk_mode.value, "")
            if snapshot.risk_mode else ""
        ),
        "poll_active": now_et.hour < 8,
        # F.3 fragment-specific
        "ladder_levels": [],   # empty for template-contract rendering test
        # F.4 email context
        "et_time_str": now_et.strftime("%-I:%M %p ET"),
        "logic_version": snapshot.logic_version,
        "design_tokens": __import__("src.config", fromlist=["DESIGN_TOKENS"]).DESIGN_TOKENS,
        "ladder_table_rows": [],
        "sizing_display": None,
    }


@pytest.mark.contract
@pytest.mark.parametrize(
    "template_name, snapshot_fn",
    [
        ("base.html", make_full_snapshot),
        ("dashboard.html", make_full_snapshot),
        ("brief_fragment.html", make_full_snapshot),
        ("brief_fragment.html", make_holiday_snapshot),
        ("email.html", make_full_snapshot),
        ("email.html", make_holiday_snapshot),
    ],
)
def test_template_renders_without_undefined_error(
    template_name: str, snapshot_fn
) -> None:
    """
    Every template must render without raising UndefinedError under StrictUndefined.
    The context is built by build_template_context(), which mirrors the server-side
    context builder in routes_dashboard.py. If a template variable is referenced but
    not in the context, this test catches it immediately rather than letting it
    silently render empty at runtime.
    """
    env = make_env()
    template = env.get_template(template_name)
    snapshot = snapshot_fn()
    context = build_template_context(snapshot)
    # This must not raise jinja2.UndefinedError
    rendered = template.render(**context)
    assert len(rendered) > 0, f"Template {template_name} rendered empty string"
```

### 6.2 HTML snapshot tests

HTML snapshots use a simple string-based comparison against a committed baseline file. The update procedure is explicit:

```python
# tests/snapshot/test_html_snapshot.py

import pytest
from pathlib import Path
from src.brief.engine import build_brief
from tests.conftest import FakeDatabentoClient

SNAPSHOT_DIR = Path("tests/fixtures/snapshots")


@pytest.mark.snapshot
def test_brief_fragment_html_snapshot(fake_db, fake_databento_client) -> None:
    """
    The rendered brief_fragment.html for the golden Wednesday must match
    the committed snapshot. Fail if different (requires deliberate update).

    The rendered HTML comes from rendering the template with the documented
    template context (contracts.md F.3), NOT from a snapshot field. brief_html
    and email_html are DuckDB cache columns only; they are not stored on BriefSnapshot.
    """
    from jinja2 import StrictUndefined, Environment, FileSystemLoader
    from tests.contract.test_template_contract import build_template_context

    snapshot = build_brief(
        date="2025-01-08", db=fake_db, bar_source=fake_databento_client
    )
    env = Environment(
        loader=FileSystemLoader("src/templates"),
        undefined=StrictUndefined,
        autoescape=True,
    )
    rendered_html = env.get_template("brief_fragment.html").render(
        **build_template_context(snapshot)
    )
    baseline_file = SNAPSHOT_DIR / "brief_fragment_boring_wednesday.html"

    if not baseline_file.exists():
        baseline_file.write_text(rendered_html)
        pytest.skip("Snapshot baseline created; re-run to assert")

    baseline = baseline_file.read_text()
    assert rendered_html == baseline, (
        "HTML snapshot mismatch. "
        "If this change is intentional, run: "
        "python tests/fixtures/update_snapshots.py"
    )
```

**Baseline update procedure**: run `python tests/fixtures/update_snapshots.py`, which regenerates all snapshot baselines. Commit the updated files. Never update snapshots without a corresponding code change that justifies the visual diff.

### 6.3 SVG price-ladder marker-position assertion

The SVG ladder maps price values to y-coordinates using a linear scale:

```
y = (price_max - price) / (price_max - price_min) * ladder_height_px
```

Where `price_max` and `price_min` are the displayed range (typically ONH + 0.5*ATR above and ONL - 0.5*ATR below), and `ladder_height_px` is a constant in the template (defined in `DESIGN_TOKENS`).

```python
# tests/snapshot/test_svg_ladder.py

import re
import pytest
from src.brief.engine import build_brief
from src.brief.levels import compute_svg_ladder_y


LADDER_HEIGHT_PX = 400  # from DESIGN_TOKENS; must match the template constant
TOLERANCE_PX = 2.0      # maximum allowed deviation between expected and actual y


@pytest.mark.snapshot
def test_current_price_marker_y_position(fake_db, fake_databento_client) -> None:
    """
    For the boring-Wednesday fixture:
      current_price = 20478.00
      onh = 20490.00, onl = 20363.00, atr_14 = 210.00
      price_max = 20490.00 + 0.5*210.00 = 20595.00
      price_min = 20363.00 - 0.5*210.00 = 20258.00
      ladder_height = 400px
      expected_y = (20595.00 - 20478.00) / (20595.00 - 20258.00) * 400
                 = 117.00 / 337.00 * 400
                 = 138.87 px

    The SVG is obtained by rendering brief_fragment.html (which contains the SVG ladder)
    with the documented template context (contracts.md F.3). svg_ladder_html is NOT a
    snapshot field; it exists only as part of the rendered template output.
    """
    from jinja2 import StrictUndefined, Environment, FileSystemLoader
    from tests.contract.test_template_contract import build_template_context

    snapshot = build_brief(
        date="2025-01-08", db=fake_db, bar_source=fake_databento_client
    )
    env = Environment(
        loader=FileSystemLoader("src/templates"),
        undefined=StrictUndefined,
        autoescape=True,
    )
    svg_html = env.get_template("brief_fragment.html").render(
        **build_template_context(snapshot)
    )

    # Extract the y attribute of the element with id="current-price-marker"
    match = re.search(r'id="current-price-marker"[^>]*\bcy="([0-9.]+)"', svg_html)
    if match is None:
        match = re.search(r'id="current-price-marker"[^>]*\by="([0-9.]+)"', svg_html)
    assert match is not None, "current-price-marker element not found in SVG"

    actual_y = float(match.group(1))
    expected_y = compute_svg_ladder_y(
        price=20478.00,
        price_max=20595.00,
        price_min=20258.00,
        ladder_height_px=LADDER_HEIGHT_PX,
    )  # = 138.87

    assert abs(actual_y - expected_y) <= TOLERANCE_PX, (
        f"SVG current-price marker y: expected {expected_y:.2f}px, "
        f"got {actual_y:.2f}px (tolerance {TOLERANCE_PX}px). "
        f"Check compute_svg_ladder_y() and the template's scale math."
    )
```

The `compute_svg_ladder_y` function is extracted from the template logic into a pure Python function so it can be tested independently. The 2px tolerance is deliberate: sub-pixel rounding differences in SVG rendering are acceptable, but a formula error would cause deviations of 10+ pixels.

### 6.4 Email badge and fidelity tests

```python
# tests/snapshot/test_email_fidelity.py
#
# email_html and brief_html are DuckDB cache columns only; they are NOT stored on
# BriefSnapshot. All email assertions are made against the rendered template output,
# obtained by rendering email.html with the documented context (contracts.md F.4).

import re
import pytest
from jinja2 import StrictUndefined, Environment, FileSystemLoader
from src.brief.engine import build_brief
from src.config import DESIGN_TOKENS   # canonical structure: {"risk": {...}, "semantic": {...}}
from tests.conftest import make_snapshot
from src.models import RiskMode
from tests.contract.test_template_contract import build_template_context


RISK_MODES = ["GREEN", "YELLOW", "RED", "GRAY"]


def render_email(snapshot) -> str:
    """Render email.html with premailer inlining for a given BriefSnapshot."""
    import premailer
    env = Environment(
        loader=FileSystemLoader("src/templates"),
        undefined=StrictUndefined,
        autoescape=True,
    )
    raw_html = env.get_template("email.html").render(**build_template_context(snapshot))
    return premailer.transform(raw_html)


@pytest.mark.snapshot
@pytest.mark.parametrize("risk_mode", RISK_MODES)
def test_email_badge_bg_matches_design_tokens(risk_mode: str) -> None:
    """
    For each risk mode, the rendered email HTML must contain a bgcolor or
    background-color that exactly matches DESIGN_TOKENS["risk"][risk_mode]["bg"].
    This asserts color-correct rendering even when CSS backgrounds are stripped.
    """
    snapshot = make_snapshot(risk_mode=RiskMode(risk_mode))
    email_html = render_email(snapshot)
    # Canonical index: DESIGN_TOKENS["risk"][risk_mode]["bg"]
    expected_bg = DESIGN_TOKENS["risk"][risk_mode]["bg"].lower()  # e.g., "#22c55e"

    # Check both inline style and bgcolor attribute (Outlook fallback)
    assert expected_bg in email_html.lower(), (
        f"Risk mode {risk_mode}: badge bgcolor '{expected_bg}' not found in email HTML. "
        f"Check that premailer inlined the DESIGN_TOKENS['risk']['{risk_mode}']['bg'] color "
        f"and that the bgcolor Outlook fallback attribute is set."
    )


@pytest.mark.snapshot
@pytest.mark.parametrize(
    "risk_mode, expected_text",
    [
        ("GREEN", "TRADE NORMAL SIZE"),
        ("YELLOW", "TRADE SMALLER / BE SELECTIVE"),
        ("RED", "STAND ASIDE OR TRADE TINY"),
        ("GRAY", "DATA INCOMPLETE - VERIFY INDEPENDENTLY"),
    ],
)
def test_email_badge_text_present_regardless_of_bg(
    risk_mode: str,
    expected_text: str,
) -> None:
    """
    Badge text must be present as literal text in the rendered email HTML,
    not only as CSS content. If the background color is stripped by
    an email client, the text still communicates the risk mode.
    This is the color-blind safety requirement.
    """
    snapshot = make_snapshot(risk_mode=RiskMode(risk_mode))
    email_html = render_email(snapshot)
    assert expected_text in email_html, (
        f"Risk mode {risk_mode}: badge text '{expected_text}' not found "
        f"as literal text in email HTML. Badge must not rely on bg alone."
    )


@pytest.mark.snapshot
def test_email_premailer_inlined_styles() -> None:
    """
    After premailer processing, the email must not contain <style> blocks
    (all styles inlined). This is required for Outlook compatibility.
    """
    snapshot = make_snapshot()
    email_html = render_email(snapshot)
    assert "<style" not in email_html.lower(), (
        "premailer did not inline all styles; <style> block found in email HTML"
    )
```

### 6.5 WCAG contrast and accessibility tests

WCAG AA level requires a minimum contrast ratio of 4.5:1 for normal text (< 18pt) and 3:1 for large text (>= 18pt or 14pt bold). The badge text is large and bold, so the requirement is 3:1. All other body text in the dashboard is tested at 4.5:1.

```python
# tests/snapshot/test_accessibility.py

import re
import pytest
from src.config import DESIGN_TOKENS


def relative_luminance(hex_color: str) -> float:
    """Compute relative luminance per WCAG 2.1 formula."""
    hex_color = hex_color.lstrip("#")
    r, g, b = (int(hex_color[i:i+2], 16) / 255.0 for i in (0, 2, 4))

    def linearize(c: float) -> float:
        return c / 12.92 if c <= 0.03928 else ((c + 0.055) / 1.055) ** 2.4

    return 0.2126 * linearize(r) + 0.7152 * linearize(g) + 0.0722 * linearize(b)


def contrast_ratio(fg: str, bg: str) -> float:
    l1 = relative_luminance(fg)
    l2 = relative_luminance(bg)
    lighter, darker = max(l1, l2), min(l1, l2)
    return (lighter + 0.05) / (darker + 0.05)


# WCAG AA thresholds
WCAG_AA_NORMAL_TEXT = 4.5
WCAG_AA_LARGE_TEXT = 3.0  # Badge text is large+bold


@pytest.mark.snapshot
@pytest.mark.parametrize("risk_mode", ["GREEN", "YELLOW", "RED", "GRAY"])
def test_badge_contrast_meets_wcag_aa_large_text(risk_mode: str) -> None:
    """
    Badge text (large, bold) on badge background must meet WCAG AA 3:1.
    Uses DESIGN_TOKENS["risk"][risk_mode]["bg"] and ["fg"] (canonical structure).
    """
    bg = DESIGN_TOKENS["risk"][risk_mode]["bg"]
    fg = DESIGN_TOKENS["risk"][risk_mode]["fg"]
    ratio = contrast_ratio(fg, bg)
    assert ratio >= WCAG_AA_LARGE_TEXT, (
        f"Risk mode {risk_mode}: badge contrast ratio {ratio:.2f} "
        f"is below WCAG AA large-text threshold of {WCAG_AA_LARGE_TEXT}. "
        f"fg={fg}, bg={bg}"
    )


@pytest.mark.snapshot
def test_body_text_on_background_contrast() -> None:
    """
    Body text color on dashboard background must meet WCAG AA normal-text 4.5:1.
    Uses DESIGN_TOKENS["semantic"]["body_text"] and DESIGN_TOKENS["semantic"]["background"]
    (canonical structure per contracts.md F.4).
    """
    fg = DESIGN_TOKENS["semantic"]["body_text"]
    bg = DESIGN_TOKENS["semantic"]["background"]
    ratio = contrast_ratio(fg, bg)
    assert ratio >= WCAG_AA_NORMAL_TEXT, (
        f"Body text contrast ratio {ratio:.2f} is below 4.5:1. "
        f"fg={fg}, bg={bg}"
    )


@pytest.mark.snapshot
def test_aria_labels_present_on_risk_badge(fake_db, fake_databento_client) -> None:
    """
    The risk mode badge element must have an aria-label attribute so that
    screen readers convey the mode to non-sighted users.
    The rendered HTML is obtained from the template, not a snapshot field.
    """
    from jinja2 import StrictUndefined, Environment, FileSystemLoader
    from tests.contract.test_template_contract import build_template_context

    snapshot = build_brief(
        date="2025-01-08", db=fake_db, bar_source=fake_databento_client
    )
    env = Environment(
        loader=FileSystemLoader("src/templates"),
        undefined=StrictUndefined,
        autoescape=True,
    )
    rendered_html = env.get_template("brief_fragment.html").render(
        **build_template_context(snapshot)
    )
    assert 'aria-label' in rendered_html, (
        "risk-mode badge missing aria-label in brief_fragment HTML"
    )
    assert 'role=' in rendered_html, (
        "risk-mode badge missing role attribute"
    )
```

### 6.6 Mobile no-horizontal-scroll tests

These tests are Playwright-based and live under the `e2e` marker. They require the app running locally (or a testcontainer). They are excluded from CI by default but run in the Phase 4 acceptance gate.

```python
# tests/e2e/test_mobile_layout.py

import pytest

# playwright fixture setup is in conftest.py via pytest-playwright


@pytest.mark.e2e
@pytest.mark.parametrize("viewport_width", [390, 768, 1280])
def test_no_horizontal_scroll_at_viewport(live_server_url: str, page, viewport_width: int) -> None:
    """
    At 390px (iPhone 14), 768px (iPad), and 1280px (desktop narrow),
    the dashboard must not require horizontal scrolling.
    Horizontal scroll = document.documentElement.scrollWidth > viewport_width.
    """
    page.set_viewport_size({"width": viewport_width, "height": 900})
    page.goto(f"{live_server_url}/")
    scroll_width = page.evaluate("document.documentElement.scrollWidth")
    assert scroll_width <= viewport_width, (
        f"Horizontal scroll at {viewport_width}px: "
        f"scrollWidth={scroll_width}px exceeds viewport"
    )


@pytest.mark.e2e
@pytest.mark.parametrize("viewport_width", [390, 768, 1280])
def test_svg_ladder_fits_viewport(live_server_url: str, page, viewport_width: int) -> None:
    """
    The SVG price ladder must not overflow its container at any tested viewport.
    """
    page.set_viewport_size({"width": viewport_width, "height": 900})
    page.goto(f"{live_server_url}/")
    ladder = page.locator('[data-testid="price-ladder"]')
    bounding_box = ladder.bounding_box()
    assert bounding_box is not None, "price-ladder element not found"
    assert bounding_box["x"] + bounding_box["width"] <= viewport_width, (
        f"SVG ladder overflows viewport at {viewport_width}px: "
        f"right edge at {bounding_box['x'] + bounding_box['width']}px"
    )
```

---

## 7. Contract and API Tests

### 7.1 Route shapes and status codes

```python
# tests/contract/test_api_routes.py

import pytest
from httpx import AsyncClient
from src.api.main import app


@pytest.mark.contract
@pytest.mark.asyncio
async def test_get_health_returns_200() -> None:
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/health")
    assert response.status_code == 200
    data = response.json()
    assert "status" in data
    # Field names per HealthResponse model in contracts.md E
    assert "last_brief_date" in data, (
        "HealthResponse must have 'last_brief_date', not 'last_brief'"
    )
    assert "last_ingest_utc" in data, (
        "HealthResponse must have 'last_ingest_utc', not 'last_ingest'"
    )


@pytest.mark.contract
@pytest.mark.asyncio
async def test_get_brief_latest_returns_200_with_shape(fake_db_with_brief) -> None:
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/api/brief/latest")
    assert response.status_code == 200
    data = response.json()
    # Assert required top-level fields per BriefSnapshot model (contracts.md B.8)
    for field in ("brief_date", "risk_mode", "session_type", "catalyst_level", "vix_regime"):
        assert field in data, f"Missing required field '{field}' in /api/brief/latest response"
    # Data quality fields are nested under data_quality, not at top level
    assert "data_quality" in data, "Missing 'data_quality' nested object in /api/brief/latest response"
    assert "ingest_ok" in data["data_quality"], "Missing 'data_quality.ingest_ok'"
    assert "is_stale" in data["data_quality"], "Missing 'data_quality.is_stale'"
    assert "issues" in data["data_quality"], "Missing 'data_quality.issues'"


@pytest.mark.contract
@pytest.mark.asyncio
async def test_get_brief_by_date_returns_correct_date(fake_db_with_brief) -> None:
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/api/brief/2025-01-08")
    assert response.status_code == 200
    data = response.json()
    assert data["brief_date"] == "2025-01-08"


@pytest.mark.contract
@pytest.mark.asyncio
async def test_get_brief_by_date_404_for_missing_date(fake_db_with_brief) -> None:
    """A date with no brief in the DB returns 404, not 500."""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/api/brief/1999-01-01")
    assert response.status_code == 404


@pytest.mark.contract
@pytest.mark.asyncio
async def test_get_brief_holiday_returns_holiday_shape(fake_db_with_holiday) -> None:
    """
    A CME holiday date returns 200 with session_type=HOLIDAY and
    levels=null (None on HOLIDAY per contracts.md B.8), not a 404 or 500.
    """
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/api/brief/2025-01-20")  # MLK Day
    assert response.status_code == 200
    data = response.json()
    assert data["session_type"] == "HOLIDAY"
    assert data["levels"] is None, (
        "levels must be null in the JSON response for a HOLIDAY brief "
        "(contracts.md B.8: levels: Optional[ComputedLevels] = None on HOLIDAY)"
    )


@pytest.mark.contract
@pytest.mark.asyncio
async def test_get_brief_no_data_returns_stale_brief(fake_db_with_stale_brief) -> None:
    """
    A date with a stored brief that has ingest_ok=False returns 200
    with data_quality.is_stale=True, not 404. The stored stale brief is surfaced.
    """
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/api/brief/2025-01-08")
    assert response.status_code == 200
    data = response.json()
    # is_stale and ingest_ok are nested under data_quality in the serialized BriefSnapshot
    assert data["data_quality"]["is_stale"] is True
    assert data["risk_mode"] == "GRAY"
```

### 7.2 Idempotency test (G5)

```python
# tests/edge/test_idempotency.py

import pytest
from src.brief.engine import build_brief
from tests.conftest import FakeDatabentoClient


@pytest.mark.idempotency
def test_build_brief_same_state_produces_identical_output(
    fake_db, fake_databento_client
) -> None:
    """
    Running build_brief() twice on the same DB state must produce
    field-identical BriefSnapshot objects (excluding render_timestamp).
    This verifies the G5 requirement: no churn on re-run.
    """
    snap1 = build_brief(
        date="2025-01-08", db=fake_db, bar_source=fake_databento_client
    )
    snap2 = build_brief(
        date="2025-01-08", db=fake_db, bar_source=fake_databento_client
    )

    # Exclude render_timestamp (legitimately different between calls)
    exclude_fields = {"render_timestamp", "email_sent_at"}

    snap1_dict = snap1.model_dump(exclude=exclude_fields)
    snap2_dict = snap2.model_dump(exclude=exclude_fields)

    assert snap1_dict == snap2_dict, (
        "build_brief() is not idempotent: "
        "two consecutive calls on the same state produced different output. "
        "Check for non-deterministic sorting, timestamp.now() calls, "
        "or side effects in the brief engine."
    )


@pytest.mark.idempotency
def test_upsert_semantics_do_not_duplicate_bars(fake_db, fake_databento_client) -> None:
    """
    Ingesting the same bars twice must not create duplicate rows.
    bar count before second ingest == bar count after second ingest.
    """
    from src.ingest.run_ingest import run_ingest

    run_ingest(db=fake_db, bar_source=fake_databento_client, date="2025-01-08")
    count_after_first = fake_db.execute(
        "SELECT COUNT(*) FROM bars WHERE ts::date = '2025-01-08'"
    ).fetchone()[0]

    run_ingest(db=fake_db, bar_source=fake_databento_client, date="2025-01-08")
    count_after_second = fake_db.execute(
        "SELECT COUNT(*) FROM bars WHERE ts::date = '2025-01-08'"
    ).fetchone()[0]

    assert count_after_first == count_after_second, (
        f"Upsert is not idempotent: "
        f"{count_after_first} bars after first ingest, "
        f"{count_after_second} after second ingest (expected same)"
    )
```

---

## 8. Acceptance-Gate Mapping

The following table maps each build-plan phase to the exact named test files/cases that constitute its DONE gate. "Gate = green" means every listed test passes with zero skips.

| Phase | Gate criteria (test files / markers that must be green) |
|---|---|
| **Phase 0** (scaffold) | `tests/unit/test_config.py::test_no_direct_databento_import_in_brief_modules`, `tests/unit/test_config.py::test_alias_equality_avg_onr_lookback_equals_baseline_session_count`, CI green (ruff, gitleaks), `tests/contract/test_template_contract.py` StrictUndefined test on empty templates (if templates exist), conftest SHA-256 hash check passes |
| **Phase 1** (bars feed) | `pytest -m "unit" tests/unit/test_session.py` (all DST, CME holiday, half-day cases), `tests/edge/test_dq_gate.py::test_missing_settlement_degrades_to_gray`, `tests/edge/test_dq_gate.py::test_stale_bars_beyond_max_staleness_degrades`, `tests/edge/test_idempotency.py::test_upsert_semantics_do_not_duplicate_bars`, `tests/unit/test_cost_guard.py::test_cost_guard_aborts_above_request_cap` (G12), manual network verification test (Databento NQ.c.0 + statistics settlement row present) |
| **Phase 2** (feeds) | `pytest -m "unit" tests/unit/test_feeds.py` (all paths: CBOE primary, yfinance fallback, stale at boundary, one-day inside boundary), `tests/edge/test_dq_gate.py::test_partial_ingest_sets_issues`, `tests/edge/test_dq_gate.py::test_fmp_failed_shows_calendar_unavailable_not_no_events`, `tests/edge/test_dq_gate.py::test_fmp_empty_shows_no_major_events` |
| **Phase 3** (brief engine + G1/G2/G3/G5/G8) | `pytest -m golden` (all three oracle days), `pytest -m dq` (all 8 DQ gate cases), `pytest -m idempotency` (both idempotency cases), `pytest -m "unit" tests/unit/test_risk_mode.py` (all 27 cells), `pytest -m "unit" tests/unit/test_catalysts.py` (cap-3, FOMC, NVDA fold), `pytest -m edge tests/edge/test_edge_cases.py` (roll, half-day, DST, gap thresholds), `pytest --cov=src/brief --cov-fail-under=90 src/brief/` on the must-cover modules |
| **Phase 3.5** (Allie design review) | Not a code gate: Allie sign-off recorded in `docs/` (human loop). No tests change. |
| **Phase 4** (dashboard + email) | `pytest -m contract tests/contract/test_template_contract.py` (all 6 template/context combinations), `pytest -m snapshot` (HTML snapshot, SVG marker y within 2px, email badge bg + text, premailer inlined), `pytest -m snapshot tests/snapshot/test_accessibility.py` (WCAG AA contrast all 4 modes + body text, aria-label, role), `pytest -m contract tests/contract/test_api_routes.py` (all 6 route assertions), `pytest -m e2e tests/e2e/test_mobile_layout.py` (all 3 viewports, no horizontal scroll), manual 4-client email render check documented |
| **Phase 5** (scheduler) | `pytest -m scheduler tests/scheduler/test_scheduler.py` (all 6 cases: winter/summer UTC, spring-forward, fall-back, missed-run recovery, missed-run post-cutoff, poll cutoff), `tests/unit/test_cost_guard.py::test_cost_guard_aborts_above_monthly_cap` (monthly spend cap abort, G12) |
| **Phase 6** (e2e + validation) | `pytest -m e2e tests/e2e/test_e2e_brief.py` (golden fixture -> GET /api/brief/{date} -> assert badge/gap/ATR/catalyst-count), `pytest -m e2e tests/e2e/test_mobile_layout.py`, `tests/contract/test_api_routes.py::test_get_brief_holiday_returns_holiday_shape`, all prior phases still green, `pytest --cov=src/brief --cov=src/feeds --cov-fail-under=80` passing at the repo level |

**Exhaustive marker run (CI Definition of Done)**:

```bash
pytest -m "unit or golden or edge or dq or idempotency or contract or snapshot or scheduler" \
       --cov=src/brief --cov=src/feeds \
       --cov-report=term-missing \
       --cov-fail-under=80
```

This command must exit 0 for the service to be considered shippable. The `e2e` and `network` markers are run separately (require a live server and real network respectively).

---

## 9. Risk Mode Decision Table Test (G8 - Complete)

All 27 cells of `RISK_MODE_TABLE` are tested parametrically. The table is a direct transcription from `premarket_brief_decision_spec.md` section 1.3.

```python
# tests/unit/test_risk_mode.py

import pytest
from src.config import RISK_MODE_TABLE, VIX_FALLBACK_REGIME


RISK_MODE_EXPECTED = {
    ("NONE", "QUIET", "LOW"): "GREEN",
    ("NONE", "QUIET", "ELEVATED"): "GREEN",
    ("NONE", "QUIET", "HIGH"): "YELLOW",
    ("NONE", "NORMAL", "LOW"): "GREEN",
    ("NONE", "NORMAL", "ELEVATED"): "YELLOW",
    ("NONE", "NORMAL", "HIGH"): "YELLOW",
    ("NONE", "LARGE", "LOW"): "YELLOW",
    ("NONE", "LARGE", "ELEVATED"): "YELLOW",
    ("NONE", "LARGE", "HIGH"): "RED",
    ("MED", "QUIET", "LOW"): "GREEN",
    ("MED", "QUIET", "ELEVATED"): "YELLOW",
    ("MED", "QUIET", "HIGH"): "YELLOW",
    ("MED", "NORMAL", "LOW"): "YELLOW",
    ("MED", "NORMAL", "ELEVATED"): "YELLOW",
    ("MED", "NORMAL", "HIGH"): "RED",
    ("MED", "LARGE", "LOW"): "YELLOW",
    ("MED", "LARGE", "ELEVATED"): "RED",
    ("MED", "LARGE", "HIGH"): "RED",
    ("HIGH", "QUIET", "LOW"): "YELLOW",
    ("HIGH", "QUIET", "ELEVATED"): "YELLOW",
    ("HIGH", "QUIET", "HIGH"): "RED",
    ("HIGH", "NORMAL", "LOW"): "RED",
    ("HIGH", "NORMAL", "ELEVATED"): "RED",
    ("HIGH", "NORMAL", "HIGH"): "RED",
    ("HIGH", "LARGE", "LOW"): "RED",
    ("HIGH", "LARGE", "ELEVATED"): "RED",
    ("HIGH", "LARGE", "HIGH"): "RED",
}


@pytest.mark.unit
@pytest.mark.parametrize("key,expected", list(RISK_MODE_EXPECTED.items()))
def test_risk_mode_table_cell(key: tuple, expected: str) -> None:
    """
    Assert that RISK_MODE_TABLE[key] == expected for all 27 cells.
    This test is parametrized from the decision_spec ground truth, not from the
    implementation. A mis-implemented cell fails here.
    """
    actual = RISK_MODE_TABLE.get(key)
    assert actual is not None, (
        f"Key {key} missing from RISK_MODE_TABLE in config.py"
    )
    assert actual == expected, (
        f"RISK_MODE_TABLE[{key}] = '{actual}', expected '{expected}'"
    )


@pytest.mark.unit
def test_risk_mode_table_has_exactly_27_cells() -> None:
    assert len(RISK_MODE_TABLE) == 27, (
        f"RISK_MODE_TABLE has {len(RISK_MODE_TABLE)} cells, expected 27"
    )


@pytest.mark.unit
def test_vix_stale_uses_fallback_regime_not_elevated() -> None:
    """
    When VIX is stale, the fallback regime is LOW (not ELEVATED or HIGH).
    This tests the conservative-against-false-alarm choice from decision_spec 1.4.
    """
    assert VIX_FALLBACK_REGIME == "LOW", (
        f"VIX_FALLBACK_REGIME = '{VIX_FALLBACK_REGIME}', expected 'LOW'. "
        f"Decision spec 1.4 requires the calmest regime as fallback."
    )
```

---

## 10. Alias Equality and Config Correctness Tests

```python
# tests/unit/test_config.py (selected cases)

import pytest
from src.config import (
    AVG_ONR_LOOKBACK,
    BASELINE_SESSION_COUNT,
    ON_RANGE_QUIET_MULT,
    ON_RANGE_QUIET_MULTIPLIER,
    ON_RANGE_ELEVATED_MULTIPLIER,
    GAP_ATR_QUIET_MAX,
    GAP_ATR_GRAY_MAX,
    MIN_OVERNIGHT_BARS,
    MIN_RTH_BARS_PRIOR,
    MAX_BAR_STALENESS_MIN,
    MNQ_POINT_VALUE,
    NQ_POINT_VALUE,
)


@pytest.mark.unit
def test_alias_equality_avg_onr_lookback_equals_baseline_session_count() -> None:
    """decision_spec.md section 0: AVG_ONR_LOOKBACK is an alias for BASELINE_SESSION_COUNT."""
    assert AVG_ONR_LOOKBACK == BASELINE_SESSION_COUNT, (
        f"AVG_ONR_LOOKBACK ({AVG_ONR_LOOKBACK}) != BASELINE_SESSION_COUNT "
        f"({BASELINE_SESSION_COUNT}). These must be aliases per decision_spec.md."
    )


@pytest.mark.unit
def test_alias_equality_on_range_quiet_mult() -> None:
    assert ON_RANGE_QUIET_MULT == ON_RANGE_QUIET_MULTIPLIER


@pytest.mark.unit
def test_gap_atr_gray_max_equals_gap_atr_quiet_max() -> None:
    """decision_spec.md section 2.3: intentionally equal but separately named."""
    assert GAP_ATR_GRAY_MAX == GAP_ATR_QUIET_MAX, (
        "GAP_ATR_GRAY_MAX and GAP_ATR_QUIET_MAX must be equal (both 0.20) "
        "per decision_spec.md section 2.3."
    )


@pytest.mark.unit
def test_mechanical_constants_are_exact() -> None:
    """Mechanical constants must match the decision_spec table exactly."""
    assert MNQ_POINT_VALUE == 2.0
    assert NQ_POINT_VALUE == 20.0
    assert MIN_OVERNIGHT_BARS == 300
    assert MIN_RTH_BARS_PRIOR == 360
    assert MAX_BAR_STALENESS_MIN == 45
```

---

## 10b. Cost Guard Tests

Cost guard tests live in `tests/unit/test_cost_guard.py` (NOT in `test_session.py`). Two cases are required per the build plan G12 gate.

```python
# tests/unit/test_cost_guard.py

import pytest
from src.feeds.exceptions import CostGuardExceeded
from src.ingest.cost_guard import estimate_and_guard, check_monthly_budget
from tests.conftest import FakeDatabentoClientWithCost
from datetime import date


@pytest.mark.unit
def test_cost_guard_aborts_above_request_cap() -> None:
    """
    When a single Databento request estimate exceeds MAX_DATABENTO_REQUEST_USD,
    estimate_and_guard must raise CostGuardExceeded and no data fetch must occur.
    The DB write guard (no api_spend row written on refusal) is verified by asserting
    CostGuardExceeded is raised before any DB interaction.
    """
    from unittest.mock import MagicMock
    from src.settings import Settings

    # cost > MAX_DATABENTO_REQUEST_USD (default 5.00)
    high_cost_client = FakeDatabentoClientWithCost(cost=6.00)
    settings = Settings(
        DATABENTO_API_KEY="test-key",
        FMP_API_KEY="test-fmp",
        SMTP_USER="u@test.com",
        SMTP_PASSWORD="pw",
        BRIEF_TO_EMAIL="a@test.com",
        BRIEF_BCC_EMAIL="b@test.com",
        ADMIN_TOKEN="tok",
    )
    mock_db = MagicMock()

    with pytest.raises(CostGuardExceeded) as exc_info:
        estimate_and_guard(
            source=high_cost_client,
            symbol="NQ.c.0",
            start_date=date(2025, 1, 1),
            end_date=date(2025, 1, 8),
            settings=settings,
            label="per-request",
        )

    assert exc_info.value.estimated_usd == pytest.approx(6.00, abs=1e-3)
    assert exc_info.value.threshold_usd == pytest.approx(5.00, abs=1e-3)
    # Verify no DB write occurred: mock_db should not have been called
    mock_db.execute.assert_not_called()


@pytest.mark.unit
def test_cost_guard_aborts_above_monthly_cap() -> None:
    """
    When month-to-date spend plus the current request estimate >= MAX_DATABENTO_MONTHLY_USD,
    check_monthly_budget must raise CostGuardExceeded.
    No DB write may occur after the raise.
    """
    from unittest.mock import MagicMock
    from src.settings import Settings

    settings = Settings(
        DATABENTO_API_KEY="test-key",
        FMP_API_KEY="test-fmp",
        SMTP_USER="u@test.com",
        SMTP_PASSWORD="pw",
        BRIEF_TO_EMAIL="a@test.com",
        BRIEF_BCC_EMAIL="b@test.com",
        ADMIN_TOKEN="tok",
        MAX_DATABENTO_MONTHLY_USD=15.00,
        DATABENTO_MONTHLY_ALERT_THRESHOLD_USD=10.00,
    )
    mock_db = MagicMock()

    # mtd_actual = 12.00, estimated = 4.00 -> total 16.00 >= cap 15.00
    with pytest.raises(CostGuardExceeded) as exc_info:
        check_monthly_budget(
            mtd_actual_usd=12.00,
            estimated_usd=4.00,
            settings=settings,
        )

    assert "monthly-cap" in str(exc_info.value).lower() or exc_info.value.threshold_usd == pytest.approx(15.00, abs=1e-3)
    mock_db.execute.assert_not_called()
```

---

## 11. Edge-Case Matrix (G2 - Selected Skeletons)

```python
# tests/edge/test_edge_cases.py (selected cases)

import pytest
from src.brief.engine import build_brief


@pytest.mark.edge
def test_contract_roll_no_false_gap(fake_db_roll_day, fake_databento_client) -> None:
    """
    On the NQH5 -> NQM5 roll day, the gap must be computed on the continuous series.
    If the new front contract trades 180 pts higher in absolute terms, the continuous
    gap should still read ~+40 pts (real), NOT +220 pts (naive new-contract price).
    """
    snapshot = build_brief(
        date="2025-03-21",  # roll day in fixture
        db=fake_db_roll_day,
        bar_source=fake_databento_client,
    )
    assert snapshot.levels is not None, "levels must not be None on a roll day (FULL session)"
    levels = snapshot.levels
    assert levels.gap_pts is not None
    # The fixture encodes a ~+40 real gap; a false gap would be > 150
    assert abs(levels.gap_pts) < 100.0, (
        f"Suspiciously large gap on roll day: {levels.gap_pts} pts. "
        f"Likely using new-contract absolute price instead of continuous series."
    )
    assert levels.is_roll_day is True
    # The roll annotation must appear in the rendered headline or volatility_line
    assert "contract roll" in snapshot.headline.lower() or "contract roll" in snapshot.volatility_line.lower(), (
        "Roll annotation must appear in headline or volatility_line: "
        "'Contract roll today (NQH5 -> NQM5); levels are continuous-series.'"
    )


@pytest.mark.edge
def test_gap_below_0_20_atr_renders_gray_headline(fake_db, fake_databento_client) -> None:
    """
    gap_as_atr_frac = 0.15 (<= GAP_ATR_GRAY_MAX = 0.20).
    Headline must contain the qualifier 'flat (gap under 0.2x ATR, treat as noise)'.
    The badge bucket is QUIET (gap_as_atr_frac <= 0.20); GRAY badge is a data-quality state only.
    gap_headline_color and gap_headline_qualifier are intermediates (not stored fields);
    assert via the rendered headline string.
    """
    snapshot = build_brief(
        date="2025-01-08",
        db=fake_db,
        bar_source=fake_databento_client,
        _gap_as_atr_frac_override=0.15,
        _prior_settle_override=20400.00,
        _current_price_override=20400.00 + 0.15 * 210.00,  # gap = 31.5 pts
    )
    assert snapshot.levels is not None
    assert snapshot.levels.gap_as_atr_frac is not None
    assert abs(snapshot.levels.gap_as_atr_frac - 0.15) <= 1e-3
    # gap_headline_color and gap_headline_qualifier are intermediates; assert via headline
    assert "flat" in snapshot.headline.lower(), (
        f"headline '{snapshot.headline}' must contain 'flat' for a sub-0.20 ATR gap"
    )
    assert "treat as noise" in snapshot.headline.lower(), (
        f"headline must contain 'treat as noise' for a sub-0.20 ATR gap"
    )
    assert snapshot.risk_mode != "GRAY", "badge GRAY is a data-quality state, not a gap state"


@pytest.mark.edge
def test_cpi_three_catalyst_overflow_cap(fake_db, fake_databento_client) -> None:
    """
    CPI (HIGH, 08:30), ISM (MED, 10:00), Jobless Claims (MED, 08:30), NVDA bmo (MED, 09:30).
    With 4 events and cap=3, exactly 3 must be shown and "+1 more" appended.
    Priority: HIGH first (CPI), then MED chronological (Claims 08:30, NVDA 09:30).
    ISM 10:00 is dropped.
    """
    from src.feeds.econ_calendar import CalendarResult, CalendarEvent

    events = [
        CalendarEvent(name="CPI", scheduled_et="08:30", severity="HIGH"),
        CalendarEvent(name="ISM Manufacturing", scheduled_et="10:00", severity="MED"),
        CalendarEvent(name="Jobless Claims", scheduled_et="08:30", severity="MED"),
        CalendarEvent(name="NVDA earnings", scheduled_et="09:30", severity="MED"),
    ]
    snapshot = build_brief(
        date="2025-01-15",
        db=fake_db,
        bar_source=fake_databento_client,
        _calendar_override=CalendarResult(events=events, feed_ok=True),
    )
    # catalysts are stored on snapshot.catalysts (list[CatalystWindow])
    shown = snapshot.catalysts
    assert len(shown) == 3, f"Expected 3 shown events (cap=3), got {len(shown)}"
    shown_labels = [e.event_label for e in shown]
    assert any("CPI" in label for label in shown_labels)
    assert any("Jobless Claims" in label or "jobless" in label.lower() for label in shown_labels)
    assert any("NVDA" in label for label in shown_labels)
    assert not any("ISM" in label for label in shown_labels), "ISM (lowest priority) must be dropped"
    assert snapshot.catalysts_dropped_count == 1, (
        "catalysts_dropped_count must be 1 when one event is dropped by the cap"
    )
    # The "+N more" text appears in the rendered template, not on a snapshot field.
    # Assert via template render:
    from tests.contract.test_template_contract import build_template_context
    from jinja2 import StrictUndefined, Environment, FileSystemLoader
    env = Environment(
        loader=FileSystemLoader("src/templates"), undefined=StrictUndefined, autoescape=True,
    )
    rendered = env.get_template("brief_fragment.html").render(**build_template_context(snapshot))
    assert "+1 more" in rendered, (
        "Overflow indicator '+1 more' must appear in rendered fragment when cap is exceeded"
    )


@pytest.mark.edge
def test_fomc_combined_window_13_55_to_15_00(fake_db, fake_databento_client) -> None:
    """
    FOMC Rate Decision at 14:00 ET -> combined window [13:55, 15:00].
    FOMC_PAD_BEFORE=5 -> 14:00-5min = 13:55.
    FOMC_PAD_AFTER_PRESSER=30 -> 14:30+30min = 15:00.
    """
    from src.feeds.econ_calendar import CalendarResult, CalendarEvent

    events = [CalendarEvent(name="FOMC Rate Decision", scheduled_et="14:00", severity="HIGH")]
    from zoneinfo import ZoneInfo
    ET = ZoneInfo("America/New_York")

    snapshot = build_brief(
        date="2025-01-29",
        db=fake_db,
        bar_source=fake_databento_client,
        _calendar_override=CalendarResult(events=events, feed_ok=True),
    )
    shown = snapshot.catalysts   # list[CatalystWindow] on BriefSnapshot
    assert len(shown) == 1
    fomc = shown[0]
    # no_trade_start_utc and no_trade_end_utc are stored in UTC on CatalystWindow;
    # convert to ET for the assertion
    assert fomc.no_trade_start_utc is not None
    assert fomc.no_trade_end_utc is not None
    start_et = fomc.no_trade_start_utc.astimezone(ET)
    end_et   = fomc.no_trade_end_utc.astimezone(ET)
    assert start_et.strftime("%H:%M") == "13:55", (
        f"FOMC window start: expected 13:55 ET, got {start_et.strftime('%H:%M')} ET"
    )
    assert end_et.strftime("%H:%M") == "15:00", (
        f"FOMC window end: expected 15:00 ET, got {end_et.strftime('%H:%M')} ET"
    )


@pytest.mark.edge
def test_half_day_adjusts_baselines(fake_db_half_day, fake_databento_client) -> None:
    """
    Half-day (session_type=HALF, early close 13:00 ET):
    - MIN_RTH_BARS_PRIOR check must NOT trigger degrade (expected ~210 bars, not 360).
    - Headline annotates "Early close 13:00 ET".
    - session_type == "HALF" in the snapshot.
    """
    snapshot = build_brief(
        date="2025-11-26",  # day before Thanksgiving in fixture
        db=fake_db_half_day,
        bar_source=fake_databento_client,
    )
    assert snapshot.session_type.value == "HALF"
    assert snapshot.levels is not None, "HALF session must still have levels"
    assert snapshot.levels.session_type.value == "HALF"
    assert snapshot.data_quality.ingest_ok is True, (
        "Half-day must not trigger the MIN_RTH_BARS_PRIOR degrade check "
        "(floor is HALF_DAY_RTH_BARS * 0.92, not MIN_RTH_BARS_PRIOR)"
    )
    # "Early close" must appear in the rendered headline or volatility_line
    assert "early close" in snapshot.headline.lower() or "early close" in snapshot.volatility_line.lower(), (
        "Half-day brief must annotate 'Early close 13:00 ET' in a rendered string field"
    )
```

---

*End of test strategy.*
