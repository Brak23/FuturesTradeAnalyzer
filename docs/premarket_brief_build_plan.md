# Pre-Market Brief: Step-by-Step Build Plan and Agent Assignments

**Purpose**: A complete, ordered build guide for the `premarket` service, naming WHICH agent to use for each step. Companion to `premarket_brief_design.md` (product + content) and `premarket_brief_architecture.md` (the blueprint this executes).

**Audience**: Bryce, or a future Claude Code session driving the build.

> **Detail layer (build against these, do not re-decide).** This plan says WHAT and WHO; three companion specs say exactly HOW, so no step has to invent a decision mid-build:
> - **`premarket_brief_decision_spec.md`** - every numeric threshold, formula, and rule (the full 27-cell Risk Mode table, trend/chop classifier, catalyst tiering + padding, sizing formula, level edge cases) as 38 named constants. Items tagged `[PROPOSED - confirm with Allie]` are defaulted but pending her sign-off (consolidated in that doc's "OPEN FOR ALLIE" list).
> - **`premarket_brief_contracts.md`** - the single `Settings` (pydantic-settings) class holding all constants, every pydantic v2 model (`BriefSnapshot`, `ComputedLevels`, `DataQuality`, etc.), the feed `Protocol` signatures, the DuckDB DDL, the FastAPI route shapes, and the Jinja context dicts. This is the interface source of truth; modules implement these, they do not redefine them.
> - **`premarket_brief_test_strategy.md`** - the concrete test taxonomy, the golden-file oracle (3 hand-verified days with shown arithmetic), fixture/mock management, the DST scheduler harness, and (section 8) a **phase -> named-test acceptance-gate table** that makes "phase done" a boolean (those tests green), not a judgment call.
>
> Where a step below references a constant or model by name, its definition lives in one of these three docs. Each phase gate now ends with **Acceptance**, pointing at the test-strategy gate for that phase.

**Working model**: This is a `~/docker/premarket/` service on homehub, built test-first, in vertical slices, with a review gate at the end of each phase. The bar feed (Feed A) is the only hard blocker and comes first.

> The readiness gaps (`premarket_brief_readiness.md`) are now **folded inline into the phases below**, tagged with their G-IDs for traceability. The five build-blockers are **G1** (correctness oracle), **G3** (DQ/freshness gate), **G11** (dashboard auth), **G12** (monthly spend cap), **G16** (Allie design review, Phase 3.5). The service-level **Definition of Done** is at the end.

---

## 0. How to run this build

### Orchestration model
- **One phase at a time.** Each phase has a clear deliverable and a review gate. Do not start phase N+1 until phase N's gate passes.
- **Test-first.** Invoke `superpowers:test-driven-development` before writing implementation code for any phase. Every feature/compute function gets a synthetic-fixture test plus, where relevant, a point-in-time / session-boundary test.
- **Plan before code.** For the first build session, invoke `superpowers:writing-plans` to turn this doc into a living checklist, then `superpowers:executing-plans` (or `superpowers:subagent-driven-development`) to drive it with review checkpoints.
- **Review gate every phase.** End each phase with `feature-dev:code-reviewer` (or `voltagent-qa-sec:code-reviewer`) and, for the secrets/email/exposure surface, `voltagent-qa-sec:security-auditor`. Use the `superpowers:requesting-code-review` skill to frame the request.
- **Isolation.** Build in a git worktree (`superpowers:using-git-worktrees`) or a dedicated branch; this service is a new repo under `~/docker/premarket/`, separate from FuturesTradeAnalyzer.

### Who decides vs who builds
**Feed A is decided: Databento `GLBX.MDP3`** (`ohlcv-1m` bars + `statistics` settlements, continuous symbol `NQ.c.0`, Historical API). See `premarket_brief_vendor_research.md`. Remaining open calls (`premarket_brief_architecture.md` section 8): SMTP path, FMP tier, and backfill depth. None block Phase 0; backfill depth is needed before Phase 1's seed step.

### Dispatch note
Agent names below are the subagent types available in this environment. Dispatch them via the Agent tool. Independent steps within a phase can run in parallel (one message, multiple Agent calls). If you want fully deterministic multi-agent orchestration with fan-out and review loops, drive it with a Workflow (opt-in) using these same agent assignments.

---

## Phase 0 - Decisions, scaffold, and hardening baseline (Days 1-3)

**Goal**: Repo skeleton exists; decisions are made; CI, secrets hygiene, and config validation run green on an empty app. (Expanded per readiness G11s/G11j/G11t/G13/G14b/G22; secrets posture moved here from Phase 5.)

| Step | What | Agent |
|---|---|---|
| 0.1 | Feed A = Databento (DONE). Register Databento + FMP accounts, get API keys; pick SMTP path (relay or dedicated Gmail app-password, NOT self-hosted, G14d) + backfill depth. **Confirm Databento licensing permits intra-household email + private LAN display (G22)**; note in `docs/licensing.md` | Bryce (decision) |
| 0.2 | Scaffold `~/docker/premarket/app/` tree, `pyproject.toml` (uv), `ruff` + `pytest`, `.python-version`, `.env.example`, `.gitignore` (`.env`, `/data`, `/logs`) | `voltagent-dev-exp:build-engineer` or `voltagent-lang:python-pro` |
| 0.2b | **Secrets hygiene (G13)**: `gitleaks` pre-commit + `.github/workflows/ci.yml` (ruff, pytest+cov, docker build) running OFFLINE on synthetic fixtures; mirror parent `DEVELOPMENT_PLAN.md` 1.7. Add `pip-audit`/`osv-scanner` + Trivy image scan + Dependabot (G14b) | `voltagent-infra:security-engineer` + `voltagent-dev-exp:build-engineer` |
| 0.3 | `src/config.py` / `src/settings.py`: **transcribe the single `Settings` (pydantic-settings) class from `contracts.md` section A verbatim** - it already carries all 38 decision-spec constants (`BASELINE_SESSION_COUNT`, `ON_RANGE_QUIET/ELEVATED_MULTIPLIER` (G7), VIX/gap-ATR/padding/sizing/staleness thresholds) plus infra constants, with defaults = the decision-spec proposed values. Add a test asserting the documented aliases (`AVG_ONR_LOOKBACK`==`BASELINE_SESSION_COUNT`, `ON_RANGE_QUIET_MULT`==`ON_RANGE_QUIET_MULTIPLIER`) so names cannot drift | `voltagent-lang:python-pro` |
| 0.3b | **`Settings` startup validation (G11s)** per `contracts.md` A: fail fast if API keys absent, SMTP port invalid, or `MAX_DATABENTO_REQUEST_USD`/`MAX_DATABENTO_MONTHLY_USD` not positive. **`structlog` config with a redaction processor (G13)** masking `*_API_KEY`/`*_PASSWORD`/`authorization`; tested as a fitness function | `voltagent-lang:python-pro` |
| 0.4 | `src/contracts/` models + protocols: **implement the pydantic v2 models and the `BarSource`/`SettlementSource`/`CalendarSource`/`EarningsSource`/`VixSource` `Protocol` classes exactly as defined in `contracts.md` sections B and C** (incl. the typed exceptions `FeedTimeoutError`/`FeedRetryExhaustedError`/`CostGuardExceeded`) | `voltagent-core-dev:backend-developer` |
| 0.5 | Empty FastAPI app + `/health`; **`docs_url=None`/`redoc_url=None` + `nosniff`/CSP headers (G15s)**; **Jinja2 `StrictUndefined` (G11t)** | `voltagent-lang:fastapi-developer` |
| 0.6 | `justfile` (G11j): `dev/test/lint/backfill/ingest/brief/check-cost` | `voltagent-dev-exp:build-engineer` |

**Gate**: `just test` and `ruff check` pass; CI green; `gitleaks` clean; redaction-processor test passes; `docker compose up` boots and `/health` responds. Review with `feature-dev:code-reviewer`.
**Acceptance**: the Phase 0 row of the `test_strategy.md` section 8 gate table is green (settings-validation, alias-equality, and redaction fitness tests).

---

## Phase 1 - Databento bar feed vertical slice (Days 4-7) - THE BLOCKER

**Goal**: Full-session bars AND official settlements land in DuckDB from Databento and are verifiable by eye. Nothing else is built until this works.

| Step | What | Agent |
|---|---|---|
| 1.1 | `src/feeds/bars_databento.py`: `DatabentoBarSource(BarSource)` using `GLBX.MDP3`, `NQ.c.0`, Historical API. `ohlcv-1m` + matching `statistics` settlement/OI. **Wrap calls in `tenacity` retries/backoff with typed `FeedTimeoutError` vs `FeedRetryExhaustedError` (G10)** | `voltagent-data-ai:data-engineer` (lead) + `voltagent-lang:python-pro` |
| 1.2 | `src/ingest/cost_guard.py` to the **cost-guard contract in `contracts.md` C** (`estimate_request_usd()` signature, refuse-above-threshold via `CostGuardExceeded`): `metadata.get_cost` per request, abort above `MAX_DATABENTO_REQUEST_USD`, log actual cost. **Persist month-to-date spend (`api_spend` table) and refuse/alert above `MAX_DATABENTO_MONTHLY_USD` (G12)** | `voltagent-lang:python-pro` |
| 1.3 | `src/store/db.py` + `schema.sql` to the **DuckDB DDL in `contracts.md` D** (`bars`, `settlements`, `api_spend`, `schema_version`; PKs, types, nullability, per-table upsert keys as specified). **Single shared connection via FastAPI lifespan + `asyncio.Lock` on writes, or staging+RENAME (G9)**; **ordered `migrations/V{n}__*.sql` run at startup (G11m)**; **explicit `INSERT OR REPLACE` upsert semantics (G5)** | `voltagent-data-ai:database-optimizer` |
| 1.4 | `src/ingest/normalizer.py` -> returns the `Bar` model from `contracts.md` B: UTC normalization, session tag (RTH/ETH from ts), resolved-contract stamp, OHLC integrity DQ | `voltagent-data-ai:data-engineer` |
| 1.5 | `src/ingest/backfill.py`: one-time historical seed via batch/`get_range` (depth per decision). **Guard: only run if DB absent/corrupt; never auto-trigger (G12/G19)** | `voltagent-data-ai:data-engineer` |
| 1.6 | `src/brief/session.py`: `is_rth()`/`is_eth()`, CME halt + holiday calendar, **`SessionType` FULL/HALF/HOLIDAY enum from `contracts.md` B (G6)** (half-day RTH closes 13:00 ET per the CME calendar; the prior-RTH bar floor `MIN_RTH_BARS_PRIOR` is `session_type`-conditioned per `decision_spec.md`), `ZoneInfo` boundaries | `voltagent-lang:python-pro` |
| 1.7 | `tests/unit/test_session.py` (DST March/Nov, CME holidays, **half-days G6**) + the **verification test** (`NQ.c.0` continuous 18:00->17:00 ET with only the halt gap, AND a `statistics` settlement row). **Partial-ingest test: `data_quality.ingest_ok=False` path when bars below `MIN_OVERNIGHT_BARS` or staleness exceeds `MAX_BAR_STALENESS_MIN` or settlement missing (decision-spec thresholds, G3)**; **`tests/unit/test_cost_guard.py::test_cost_guard_aborts_above_request_cap` (G12)**. Implement to `test_strategy.md` sections 4-5 | `voltagent-qa-sec:test-automator` |

**Gate (HARD KILL)**: bars + settlements query back with a correct `session` split, a resolved `contract`, and the verification test passes; the partial-ingest path sets `data_quality.ingest_ok=False`. If Databento does not return the overnight bars or the settlement record, stop and fix symbology/schema (`NQ.c.0`, `statistics`) before building further.
**Acceptance**: the Phase 1 row of the `test_strategy.md` section 8 gate table is green (verification, DST/half-day session, partial-ingest boundary, and `test_cost_guard.py::test_cost_guard_aborts_above_request_cap` tests).

---

## Phase 2 - Remaining feeds (Days 8-10)

**Goal**: VIX, econ calendar, earnings all fetch and parse independently, behind tests.

| Step | What | Agent |
|---|---|---|
| 2.1 | `src/feeds/vix.py` (CBOE daily CSV, **fallback to `yfinance ^VIX`; `MAX_VIX_STALENESS_DAYS=3` -> "VIX unavailable", badge treats VIX neutral, G10d**) | `voltagent-data-ai:data-engineer` |
| 2.2 | `src/feeds/econ_calendar.py` (FMP; normalize importance; ET timestamps; `tenacity` retries G10) | `voltagent-data-ai:data-engineer` |
| 2.3 | `src/feeds/earnings.py` (FMP; filter to `NDX_COMPONENTS`; `bmo/amc` timing; retries G10) | `voltagent-data-ai:data-engineer` |
| 2.4 | `src/ingest/run_ingest.py` orchestrator returning per-feed success/error (feeds `ingest_ok`/`ingest_errors[]`) | `voltagent-core-dev:backend-developer` |
| 2.5 | `tests/test_feeds.py` to the **fixture+mock management in `test_strategy.md` section 3**: `FakeDatabentoClient` (raises on any symbol != `NQ.c.0`) + recorded DataFrames, `responses` cassettes for FMP/CBOE (secrets scrubbed), SHA-256 fixture manifest so no test hits the network; the VIX CBOE->yfinance->stale path incl. the `MAX_VIX_STALENESS_DAYS` boundary; empty-FMP vs FMP-failure as distinct assertions; `calendar_events`/`earnings_events` tables | `voltagent-qa-sec:test-automator` |

**Gate**: `run_ingest` runs clean against real APIs; each feed has passing recorded/mocked tests incl. the empty/missing-data degradation paths. Review with `feature-dev:code-reviewer`.
**Acceptance**: the Phase 2 row of the `test_strategy.md` section 8 gate table is green (per-feed recorded tests, VIX fallback/staleness boundary, empty-vs-failure distinction).

---

## Phase 3 - Brief engine (Days 11-14)

**Goal**: Compute a full `BriefSnapshot` (the `contracts.md` B model) from the database, implementing the formulas in `decision_spec.md`. No UI yet. This is where the quant/risk content spec becomes code; the numbers are already specified, so this phase transcribes rather than invents.

> **Phase entry gate 3.0 must clear before any of 3.1-3.6 are coded.** It is distinct from the Phase 3.5 *design-review* gate; the "3.5" in step number 3.5 below is the Risk Mode *implementation* step within this phase.

| Step | What | Agent |
|---|---|---|
| 3.0 | **Entry gate - resolve + lock the decision spec (G8)**: the full 27-cell Risk Mode table and all thresholds already exist in `decision_spec.md`; this step (a) walks the "OPEN FOR ALLIE" list with Allie and either confirms the proposed defaults or substitutes her numbers, (b) confirms the `[PROPOSED - confirm with Bryce]` operational floors, (c) verifies all values are present in the `Settings` class from Phase 0. No compute code (3.1-3.6) starts until this is locked | `voltagent-domains:risk-manager` + Allie/Bryce sign-off |
| 3.1 | `src/brief/levels.py` -> `ComputedLevels` model, per `decision_spec.md` section 6 (PDH/PDL, ONH/ONL, ON VWAP, conditional weekly/monthly H/L via `LEVEL_PROXIMITY_ATR`, roll carry-over rule, settlement-based gap; **ATR uses RTH-only bars, G1**) | `voltagent-lang:python-pro` |
| 3.2 | `src/brief/overnight.py` -> `OvernightSummary` model, per `decision_spec.md` section 4: the non-directional TRENDED/BALANCED/CHOPPY classifier (close-location-value + net-ATR + path efficiency thresholds), ON range vs trailing-`BASELINE_SESSION_COUNT` avg (**thresholds from config G7**), and the explicit shown-vs-withheld fields enforcing the no-direction rule | `voltagent-lang:python-pro` |
| 3.3 | `src/brief/volatility.py` per `decision_spec.md` section 2: `ATR_LOOKBACK`, gap-as-fraction-of-ATR formula, `GAP_ATR_GRAY_MAX` inconclusive band, headline qualifier word-bands, VIX expected move | `voltagent-lang:python-pro` |
| 3.4 | `src/brief/catalysts.py` -> `list[CatalystWindow]`, per `decision_spec.md` section 3: the HIGH/MED severity lookup table (+ FMP-impact fallback), per-severity `NO_TRADE_PAD_*` windows, combined FOMC window, and the cap-3 tie-break (severity -> chronological -> alphabetical) incl. pre-market mega-cap earnings | `voltagent-lang:python-pro` |
| 3.5 | **Risk Mode badge + sizing line** -> `RiskMode` + `SizingLine` models, impl the 27-cell table and the `floor(per_trade_risk / (stop_pts * point_value))` formula from `decision_spec.md` sections 1 + 5 (MNQ=2/NQ=20 point value, `MAX_CONTRACTS` cap, unset-> show-nothing rule, VIX-stale-> treat-as-LOW fallback) | `voltagent-lang:python-pro` |
| 3.6 | `src/brief/engine.py` `build_brief()` -> persists `computed_levels` + `brief_snapshots`. **`BriefReadinessCheck` -> `DataQuality` (G3)**: missing settlement / bars < `MIN_OVERNIGHT_BARS` / staleness > `MAX_BAR_STALENESS_MIN` -> degraded GRAY brief + visible "data incomplete" banner, never silent; sanity asserts per `decision_spec.md` (`ONH>ONL`, settle within `SETTLEMENT_SANITY_PCT` of price, gap-ATR within `GAP_ATR_SANITY_ABS`) | `voltagent-core-dev:backend-developer` |
| 3.7 | **Golden-file oracle (G1)** per `test_strategy.md` section 2: the 30+ day synthetic fixture (DST cross, holiday, half-day, quiet+huge overnight, roll, missing-settlement) with the documented deterministic generator; `test_levels_golden.py` asserts the full computed record against the 3 hand-verified oracle days (arithmetic shown in the strategy doc); float tolerance + sign-off procedure as specified | `voltagent-qa-sec:test-automator` |
| 3.7b | **Edge-case matrix (G2)** `test_edge_cases.py` per `test_strategy.md` section 5: roll/contract change (no false gap), half-day, DST, gap<`GAP_ATR_GRAY_MAX`, flat-then-recover, CPI 3-catalyst overflow, partial feed, empty FMP, missing VIX. **Idempotency/determinism test (G5)**. **27-cell Risk Mode decision-table test (G8)** | `voltagent-qa-sec:test-automator` |
| 3.8 | **Content correctness review**: levels, no-direction rule, cut list honored, false-precision rounding; sign off against the golden file | `voltagent-domains:quant-analyst` + `voltagent-domains:risk-manager` |

**Gate**: `build_brief(date)` returns a correct `BriefSnapshot` against the golden file; the DQ gate degrades (not crashes) on stale/partial data; edge-case + idempotency + decision-table tests pass; quant + risk sign off. Review code with `feature-dev:code-reviewer`.
**Acceptance**: the Phase 3 row of the `test_strategy.md` section 8 gate table is green (golden-oracle, edge-case matrix, idempotency, 27-cell decision-table, and DQ-degradation tests).

---

## Phase 3.5 - Allie design review (Day 15) - GATE (G16)

**Goal**: Validate the UX on a prototype with the actual end user BEFORE any Phase 4 code. ADHD-friendliness is not code-reviewable.

| Step | What | Agent |
|---|---|---|
| 3.5.1 | ui-designer renders a clickable/static prototype of the brief (normal day + a RED CPI day) from a sample `BriefSnapshot`, per the UX spec | `voltagent-core-dev:ui-designer` |
| 3.5.2 | Allie reviews: confirms she can read the whole brief in ~30s, the price ladder is legible at a glance, and the Risk Mode badge reads as calm not alarmist | Allie (sign-off) + Bryce |

**Gate**: Allie sign-off recorded (as a GitHub issue/note). Feedback folded into the UX spec before Phase 4. No Phase 4 code starts until this passes.

---

## Phase 4 - Dashboard + email (Days 16-19)

**Goal**: The brief is visible at `premarket.home` and delivered by 07:00 email, in the ADHD-friendly layout. **Implement to the full visual spec in `premarket_brief_ux_design.md`** (design tokens, component anatomy, the SVG price ladder, states, email variant).

| Step | What | Agent |
|---|---|---|
| 4.1 | Translate UX-spec tokens into `base.html` CSS custom properties (dark theme, Inter + JetBrains Mono tabular figures, spacing/radius/motion); confirm component specs | `voltagent-core-dev:ui-designer` |
| 4.2 | `templates/base.html`, `dashboard.html`, `brief_fragment.html` to the **template-context dicts in `contracts.md` F** and the UX spec; the **SVG price-ladder**; **ET-hour injected server-side (no client Date math)**; HTMX poll gated off after 07:30 ET via Alpine, fragment endpoint returns 204 when unchanged; expand/collapse drill-down; **07:00 point-in-time snapshot, frozen timestamp, no churn (G26)**; **stale-indicator + no-data/holiday states wired to `DataQuality.is_stale`/`SessionType` (G3/G6)** | `voltagent-core-dev:frontend-developer` |
| 4.3 | `api/routes_dashboard.py` + `routes_api.py` to the **route contracts in `contracts.md` E** (`/`, `/api/brief/latest`, `/api/brief/{date}`, `/health`, `/api/fragment/brief`; exact response models, 404/holiday/no-data status codes). **Admin (token-protected): `POST /api/admin/run-now` (G10b), `GET /api/admin/runs?days=7` (G10c)** | `voltagent-lang:fastapi-developer` |
| 4.4 | `templates/email.html` to the **`DESIGN_TOKENS` dict in `contracts.md` F.4** (table-based, inlined CSS, ladder-as-table, Outlook `bgcolor` fallbacks, badge TEXT legible even if bg dropped) + `email_sender/send.py` SMTP+TLS, premailer inlining | `voltagent-core-dev:frontend-developer` + `voltagent-core-dev:backend-developer` |
| 4.5 | **Test suite per `test_strategy.md` section 6 (G11t/G23/G24/G25/G26):** Jinja2 template-contract test (renders every template incl. holiday `BriefSnapshot`, no `UndefinedError` under `StrictUndefined`); HTML snapshot of fragment + email with the documented baseline-update process; SVG marker `y` within the specified tolerance of the derived-expected value; email badge bg matches `DESIGN_TOKENS` per `RiskMode` + badge text present; mock-clock stale/poll-boundary tests; WCAG AA contrast (exact ratio threshold + the named fg/bg pairs) + non-color-encoding (glyph/`aria-label`/`role`) assertions | `voltagent-qa-sec:test-automator` |
| 4.6 | Accessibility + **mobile validation (G27)** at 390/768/1280px (no horizontal scroll, ladder fits), 30-second readability; Playwright visual reg if in scope | `voltagent-qa-sec:accessibility-tester` + `voltagent-qa-sec:ui-ux-tester` |

**Gate**: `premarket.home` renders the brief and all states (normal / RED / stale / holiday); template-contract, snapshot, contrast, and email-fidelity tests pass; a test email arrives in INBOX (not spam, G14d) and renders in Gmail + Apple Mail + Outlook; mobile verified at the three widths.
**Acceptance**: the Phase 4 row of the `test_strategy.md` section 8 gate table is green (template-contract, snapshot, SVG-marker, email-fidelity, contrast/non-color-encoding, and route-contract tests).

---

## Phase 5 - Scheduler, recovery, deployment, alerting (Days 20-22)

**Goal**: The 07:00 ET job fires unattended, recovers from a missed run, and fails LOUDLY.

| Step | What | Agent |
|---|---|---|
| 5.1 | `scheduler/jobs.py`: APScheduler ET CronTriggers (06:30 ingest, 07:00 brief+email, 17:30 settlement); **missed-run recovery capped at ONE brief, never auto-backfill (G12)**; SLA: ingest <10min, brief <5min | `voltagent-lang:python-pro` |
| 5.2 | `tests/test_scheduler.py` per `test_strategy.md` section 4: the mock-clock harness (no real `Date.now`), the 6 DST cases (winter/summer offset, 2025 spring-forward + fall-back with exact expected UTC instants), missed-run recovery inside vs outside the cutoff window, and the 07:30 poll-active/inactive boundary | `voltagent-qa-sec:test-automator` |
| 5.3 | Final `docker-compose.yml`, `Dockerfile`, healthcheck, bind-mount backup, `.env`. **Container hardening (G14): non-root `USER`, `read_only`+`tmpfs:/tmp`, `cap_drop:[ALL]`, `no-new-privileges`, mem/pids/cpus limits** | `voltagent-infra:docker-expert` |
| 5.4 | NPM proxy host + AdGuard CNAME for `premarket.home`; join `npm_network`. **NPM Access List (Basic Auth) + HTTPS at NPM so the credential is not plaintext on the LAN (G11)** | `voltagent-infra:devops-engineer` |
| 5.5 | Triple failure alerting: MQTT `premarket/alerts` -> HA phone notify; SMTP failsafe; Uptime Kuma on `/health`. **Terse payloads (date/failed/stage, no raw error/config), failure-only idempotent HA automation (G14c)**; alert when MTD spend nears the monthly cap (G12) | `voltagent-infra:sre-engineer` |
| 5.6 | Security + exposure review (G11/G13/G14 verification: auth in place, no secrets in image/logs, hardening applied); **vendor-outage runbook `docs/outages.md` (G20)**; **ownership/cost note: Databento owner Bryce, ~$3-8/mo, annual key rotation, cost on `/health` (G21)** | `voltagent-qa-sec:security-auditor` + Bryce |

**Gate**: container runs 48 hours unattended with the 07:00 job firing and logging; `premarket.home` requires auth; a deliberately broken feed triggers a phone alert via at least one channel; the monthly-spend-cap path is tested.
**Acceptance**: the Phase 5 row of the `test_strategy.md` section 8 gate table is green (DST scheduler cases, missed-run recovery, alerting-path, and `test_cost_guard.py::test_cost_guard_aborts_above_monthly_cap` tests).

---

## Phase 6 - Validation, rollout, and ship (Days 23-25 build, then a ~1-week unattended soak)

**Goal**: Allie relies on it on a real trading morning, and it is operable for the long run.

| Step | What | Agent |
|---|---|---|
| 6.1 | **Automated end-to-end test (G4)** per `test_strategy.md` sections 1 + 7: load golden fixture, `GET /api/brief/{date}`, assert badge/gap/ATR/catalyst-count against the oracle + snapshot the render; the 07:00 snapshot idempotency (same date -> identical snapshot, no churn). Plus replay 3 real historical dates vs a trusted NT chart | `voltagent-data-ai:data-analyst` + `voltagent-qa-sec:test-automator` |
| 6.2 | Edge cases at the deployment layer: DST days, CME holiday ("no session"), half-day early close | `voltagent-qa-sec:test-automator` |
| 6.3 | **Structured UAT (G15)**: Allie validates a checklist on a real brief (e.g. identifies Risk Mode + correct no-trade window in <30s unaided, sizing line shows her dollar risk). PRECEDES the quant/risk content sign-off | Allie (sign-off) + `voltagent-domains:quant-analyst` + `voltagent-domains:risk-manager` |
| 6.4 | **Rollout / cutover runbook (G17)**: confirm email lands in inbox not spam; Allie bookmarks `premarket.home` on her phone; one-page "first morning checklist"; Day-1 handoff ping | Bryce + Allie |
| 6.5 | **Disaster recovery (G19)** `docs/disaster_recovery.md`: restore DuckDB BEFORE any re-backfill; rebuild sequence; one tested restore; add a quarterly restore check to `~/maintenance/` | `voltagent-infra:sre-engineer` |
| 6.6 | **Feedback cadence (G18)**: weekly month 1, biweekly month 2, monthly after; `CHANGELOG.md`; zero-opens-3-days -> same-day check-in | Bryce + Allie |
| 6.7 | Final review + README for the service | `feature-dev:code-reviewer` + `voltagent-dev-exp:readme-generator` |

**Gate (service Definition of Done, below)**: all DoD criteria met.
**Acceptance**: the Phase 6 row of the `test_strategy.md` section 8 gate table is green (automated e2e against the oracle, snapshot idempotency, deployment-layer edge cases).

---

## Definition of Done (service-level)

The service ships when ALL hold (from `premarket_brief_readiness.md`):

1. All phase gates passed; `pytest` coverage >= 80% on `brief/` and `feeds/`; CI green; `gitleaks` clean.
2. Golden-file oracle (G1), edge-case matrix (G2), idempotency (G5), and end-to-end render (G4) tests pass.
3. The DQ/freshness gate (G3) degrades visibly on stale/partial/missing data, never silently.
4. Runs unattended for **5 real trading mornings** with correct briefs archived in `brief_snapshots`.
5. **Allie** signs off on the UAT checklist (G15); the rollout runbook (G17) is complete.
6. Dashboard is authenticated (G11); monthly Databento spend cap is live (G12); DR runbook + one tested restore exist (G19).

A CME-holiday day rendering the correct "no session" state counts as a pass.

---

## Component -> agent quick reference

| Component | Primary agent | Support / review |
|---|---|---|
| Repo scaffold, tooling, CI | `voltagent-dev-exp:build-engineer` | `voltagent-lang:python-pro` |
| Databento bar + settlement feed (Feed A) | `voltagent-data-ai:data-engineer` | `voltagent-lang:python-pro` |
| DuckDB schema + queries | `voltagent-data-ai:database-optimizer` | |
| Session / DST / CME-calendar logic | `voltagent-lang:python-pro` | `voltagent-qa-sec:test-automator` |
| Calendar / earnings / VIX feeds | `voltagent-data-ai:data-engineer` | |
| Brief compute (levels/overnight/vol) | `voltagent-lang:python-pro` | `voltagent-domains:quant-analyst` |
| Risk Mode + sizing logic | `voltagent-domains:risk-manager` | `voltagent-lang:python-pro` |
| FastAPI routes / app | `voltagent-lang:fastapi-developer` | `voltagent-core-dev:backend-developer` |
| Dashboard UI / templates | `voltagent-core-dev:ui-designer` | `voltagent-core-dev:frontend-developer` |
| Email delivery | `voltagent-core-dev:backend-developer` | |
| Docker / compose / Dockerfile | `voltagent-infra:docker-expert` | |
| Reverse proxy / DNS / deploy | `voltagent-infra:devops-engineer` | |
| Scheduler + alerting + healthcheck | `voltagent-infra:sre-engineer` | `voltagent-lang:python-pro` |
| Tests (all layers) | `voltagent-qa-sec:test-automator` | |
| Content correctness | `voltagent-domains:quant-analyst` + `voltagent-domains:risk-manager` | |
| Code review (per phase) | `feature-dev:code-reviewer` | `voltagent-qa-sec:code-reviewer` |
| Security / secrets review | `voltagent-qa-sec:security-auditor` | |
| Accessibility / UX | `voltagent-qa-sec:accessibility-tester` | `voltagent-qa-sec:ui-ux-tester` |

## Process skills to drive it

- `superpowers:brainstorming` - before locking any phase's approach if it is unclear.
- `superpowers:writing-plans` then `superpowers:executing-plans` - turn this doc into a tracked checklist and execute with review checkpoints.
- `superpowers:test-driven-development` - required discipline per phase.
- `superpowers:using-git-worktrees` - isolate the build.
- `superpowers:requesting-code-review` / `superpowers:receiving-code-review` - the per-phase review gate.
- `superpowers:finishing-a-development-branch` - when the service is done and ready to integrate.

## Critical-path summary

Feed A (Phase 1) is the only hard technical blocker; everything downstream depends on session-tagged bars + settlements. The vertical-slice MVP is Phases 1-4 (bars -> compute -> dashboard + email). Phase 3.5 (Allie design review) gates Phase 4. Phases 5-6 make it secure, unattended-reliable, and validated.

Schedule: phases run sequentially (each gate must pass before the next starts), Day 1 through Day 25 of active build as the **nominal** path. The nominal ranges assume no first-try surprises; three phases carry the most schedule risk and should be budgeted with a contingency day each:
- **Phase 1** (Databento): if `NQ.c.0` symbology or the `statistics` settlement schema does not return as expected on the first pull, +1-3 days. This is the hard blocker; do not start Phase 2 until its HARD KILL gate is green.
- **Phase 2** (3 feeds): VIX dual-path (CBOE + yfinance fallback) plus FMP/CBOE parsing surprises realistically push to ~4 days.
- **Phase 5** (deploy): NPM auth + HTTPS, container hardening, and triple-alerting wiring realistically push to ~4 days.

Realistic active build is therefore **~28-30 days**, not 25. Definition-of-Done criterion #4 then requires **5 real trading mornings** of unattended operation (one additional calendar week of soak with no active dev), putting the all-in estimate at **~28-30 active build-days / ~33-37 calendar days** at half-time. Track the actual vs nominal at each gate so slippage is visible early rather than at the end.

The five build-blockers (G1/G3/G11/G12/G16) are front-loaded deliberately: **G3** (DQ gate) first surfaces in Phase 1 and completes in Phase 3; **G1** (correctness oracle) in Phase 3; **G16** (Allie design review) is the Phase 3.5 gate; **G11** (dashboard auth) in Phase 5; **G12** (spend cap) begins in Phase 1 (request guard + MTD table) and completes in Phase 5 (recovery cap + alerting). None can be deferred to "after ship."
