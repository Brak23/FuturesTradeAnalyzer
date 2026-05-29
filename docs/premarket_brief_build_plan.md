# Pre-Market Brief: Step-by-Step Build Plan and Agent Assignments

**Purpose**: A complete, ordered build guide for the `premarket` service, naming WHICH agent to use for each step. Companion to `premarket_brief_design.md` (product + content) and `premarket_brief_architecture.md` (the blueprint this executes).

**Audience**: Bryce, or a future Claude Code session driving the build.

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
| 0.3 | `src/config.py` (analytical constants: session boundaries, ATR/ONR lookbacks incl. `BASELINE_SESSION_COUNT`, `ON_RANGE_QUIET/ELEVATED_MULTIPLIER` (G7), `DST_ZONE`, `NDX_COMPONENTS`) | `voltagent-lang:python-pro` |
| 0.3b | **`src/settings.py` via `pydantic-settings` (G11s)**: load + validate deployment config on startup (API keys present, SMTP port, positive `MAX_DATABENTO_REQUEST_USD`/`MAX_DATABENTO_MONTHLY_USD`). **`structlog` config with a redaction processor (G13)** masking `*_API_KEY`/`*_PASSWORD`/`authorization`; tested as a fitness function | `voltagent-lang:python-pro` |
| 0.4 | `src/contracts/bar_source.py` protocol (adapt from FuturesTradeAnalyzer) | `voltagent-core-dev:backend-developer` |
| 0.5 | Empty FastAPI app + `/health`; **`docs_url=None`/`redoc_url=None` + `nosniff`/CSP headers (G15s)**; **Jinja2 `StrictUndefined` (G11t)** | `voltagent-lang:fastapi-developer` |
| 0.6 | `justfile` (G11j): `dev/test/lint/backfill/ingest/brief/check-cost` | `voltagent-dev-exp:build-engineer` |

**Gate**: `just test` and `ruff check` pass; CI green; `gitleaks` clean; redaction-processor test passes; `docker compose up` boots and `/health` responds. Review with `feature-dev:code-reviewer`.

---

## Phase 1 - Databento bar feed vertical slice (Days 4-7) - THE BLOCKER

**Goal**: Full-session bars AND official settlements land in DuckDB from Databento and are verifiable by eye. Nothing else is built until this works.

| Step | What | Agent |
|---|---|---|
| 1.1 | `src/feeds/bars_databento.py`: `DatabentoBarSource(BarSource)` using `GLBX.MDP3`, `NQ.c.0`, Historical API. `ohlcv-1m` + matching `statistics` settlement/OI. **Wrap calls in `tenacity` retries/backoff with typed `FeedTimeoutError` vs `FeedRetryExhaustedError` (G10)** | `voltagent-data-ai:data-engineer` (lead) + `voltagent-lang:python-pro` |
| 1.2 | `src/ingest/cost_guard.py`: `metadata.get_cost` per request, abort above `MAX_DATABENTO_REQUEST_USD`, log actual cost. **Persist month-to-date spend (`api_spend` table) and refuse/alert above `MAX_DATABENTO_MONTHLY_USD` (G12)** | `voltagent-lang:python-pro` |
| 1.3 | `src/store/db.py` + `schema.sql`: DuckDB init, `bars` (+`contract`,`source`) + `settlements` (+`open_interest`) + `api_spend` + `schema_version` tables. **Single shared connection via FastAPI lifespan + `asyncio.Lock` on writes, or staging+RENAME (G9)**; **ordered `migrations/V{n}__*.sql` run at startup (G11m)**; **explicit `INSERT OR REPLACE` upsert semantics (G5)** | `voltagent-data-ai:database-optimizer` |
| 1.4 | `src/ingest/normalizer.py`: UTC normalization, session tag (RTH/ETH from ts), resolved-contract stamp, OHLC integrity DQ | `voltagent-data-ai:data-engineer` |
| 1.5 | `src/ingest/backfill.py`: one-time historical seed via batch/`get_range` (depth per decision). **Guard: only run if DB absent/corrupt; never auto-trigger (G12/G19)** | `voltagent-data-ai:data-engineer` |
| 1.6 | `src/brief/session.py`: `is_rth()`/`is_eth()`, CME halt + holiday calendar, **`session_type` FULL/HALF/HOLIDAY (G6)**, `ZoneInfo` boundaries | `voltagent-lang:python-pro` |
| 1.7 | `tests/test_session.py` (DST March/Nov, CME holidays, **half-days G6**) + the **verification test** (`NQ.c.0` continuous 18:00->17:00 ET with only the halt gap, AND a `statistics` settlement row). **Partial-ingest test: `ingest_ok=False` path when bars stale or settlement missing (G3)**; **cost-guard abort-path test (G12)** | `voltagent-qa-sec:test-automator` |

**Gate (HARD KILL)**: bars + settlements query back with a correct `session` split, a resolved `contract`, and the verification test passes; the partial-ingest path sets `ingest_ok=False`. If Databento does not return the overnight bars or the settlement record, stop and fix symbology/schema (`NQ.c.0`, `statistics`) before building further.

---

## Phase 2 - Remaining feeds (Days 8-10)

**Goal**: VIX, econ calendar, earnings all fetch and parse independently, behind tests.

| Step | What | Agent |
|---|---|---|
| 2.1 | `src/feeds/vix.py` (CBOE daily CSV, **fallback to `yfinance ^VIX`; `MAX_VIX_STALENESS_DAYS=3` -> "VIX unavailable", badge treats VIX neutral, G10d**) | `voltagent-data-ai:data-engineer` |
| 2.2 | `src/feeds/econ_calendar.py` (FMP; normalize importance; ET timestamps; `tenacity` retries G10) | `voltagent-data-ai:data-engineer` |
| 2.3 | `src/feeds/earnings.py` (FMP; filter to `NDX_COMPONENTS`; `bmo/amc` timing; retries G10) | `voltagent-data-ai:data-engineer` |
| 2.4 | `src/ingest/run_ingest.py` orchestrator returning per-feed success/error (feeds `ingest_ok`/`ingest_errors[]`) | `voltagent-core-dev:backend-developer` |
| 2.5 | `tests/test_feeds.py`: **fixture strategy = `FakeDatabentoClient` + recorded DataFrames for the SDK, `responses` for FMP/CBOE HTTP (architect C5)**; empty-FMP and missing-VIX graceful paths; `calendar_events`/`earnings_events` tables | `voltagent-qa-sec:test-automator` |

**Gate**: `run_ingest` runs clean against real APIs; each feed has passing recorded/mocked tests incl. the empty/missing-data degradation paths. Review with `feature-dev:code-reviewer`.

---

## Phase 3 - Brief engine (Days 11-14)

**Goal**: Compute a full `BriefSnapshot` from the database. No UI yet. This is where the quant/risk content spec becomes code.

| Step | What | Agent |
|---|---|---|
| 3.0 | **Risk Mode decision table (G8)**: risk-manager produces a numeric table over (catalyst: none/med/high) x (gap-ATR: <0.2 / 0.2-1.0 / >1.0) x (VIX: low/elevated/high) -> GREEN/YELLOW/RED, with thresholds written into `config.py`. PRECEDES the Risk Mode impl step 3.5 within this phase (not the Phase 3.5 design-review gate) | `voltagent-domains:risk-manager` |
| 3.1 | `src/brief/levels.py` (PDH/PDL, ONH/ONL, ON VWAP, conditional weekly/monthly H/L, settlement-based gap; **ATR uses RTH-only bars, G1**) | `voltagent-lang:python-pro` |
| 3.2 | `src/brief/overnight.py` (net change, ON range vs trailing-20 avg using `BASELINE_SESSION_COUNT` holiday/half-day-excluded, **thresholds from config G7**, trend-vs-chop) | `voltagent-lang:python-pro` |
| 3.3 | `src/brief/volatility.py` (ATR(14), gap-as-fraction-of-ATR, VIX expected move) | `voltagent-lang:python-pro` |
| 3.4 | `src/brief/catalysts.py` (merge + time-sort econ/earnings, filter high/medium, cap 3) | `voltagent-lang:python-pro` |
| 3.5 | **Risk Mode badge + sizing line** impl to the 3.0 table; dollar-risk-constant sizing echoing Allie's pre-set limits | `voltagent-lang:python-pro` |
| 3.6 | `src/brief/engine.py` `build_brief()`; `computed_levels` + `brief_snapshots`. **`BriefReadinessCheck`/DQ-freshness gate (G3)**: missing settlement / stale bars / low bar-count -> degraded GRAY brief + visible "data incomplete" banner, never silent; sanity asserts (ONH>ONL, settle within ~0.5% of price, gap-ATR in [-3,+3]) | `voltagent-core-dev:backend-developer` |
| 3.7 | **Golden-file oracle (G1)**: 30+ day synthetic fixture (DST cross, holiday, half-day, quiet+huge overnight); hand-verified; `test_levels_golden.py` asserts the full computed record. Plus per-module synthetic tests | `voltagent-qa-sec:test-automator` |
| 3.7b | **Edge-case matrix (G2)** `test_edge_cases.py`: roll/contract change (no false gap), half-day, DST, gap<0.2 ATR, flat-then-recover, CPI 3-catalyst overflow, partial feed, empty FMP, missing VIX. **Idempotency/determinism test (G5)**. **Risk Mode decision-table test (G8)** | `voltagent-qa-sec:test-automator` |
| 3.8 | **Content correctness review**: levels, no-direction rule, cut list honored, false-precision rounding; sign off against the golden file | `voltagent-domains:quant-analyst` + `voltagent-domains:risk-manager` |

**Gate**: `build_brief(date)` returns a correct `BriefSnapshot` against the golden file; the DQ gate degrades (not crashes) on stale/partial data; edge-case + idempotency + decision-table tests pass; quant + risk sign off. Review code with `feature-dev:code-reviewer`.

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
| 4.2 | `templates/base.html`, `dashboard.html`, `brief_fragment.html` per the UX spec; the **SVG price-ladder**; **ET-hour injected server-side (no client Date math)**; HTMX poll gated off after 07:30 ET via Alpine; expand/collapse drill-down; **07:00 point-in-time snapshot, frozen timestamp, no churn (G26)**; **stale-indicator + no-data/holiday states wired to `is_stale`/`session_type` (G3/G6)** | `voltagent-core-dev:frontend-developer` |
| 4.3 | `api/routes_dashboard.py` + `routes_api.py` (`/`, `/api/brief/latest`, `/api/brief/{date}`, `/health`). **Admin (token-protected): `POST /api/admin/run-now` (G10b), `GET /api/admin/runs?days=7` (G10c)** | `voltagent-lang:fastapi-developer` |
| 4.4 | `templates/email.html` (table-based, inlined CSS via `DESIGN_TOKENS`, ladder-as-table, Outlook `bgcolor` fallbacks, badge TEXT legible even if bg dropped) + `email_sender/send.py` SMTP+TLS, premailer inlining | `voltagent-core-dev:frontend-developer` + `voltagent-core-dev:backend-developer` |
| 4.5 | **Test suite (G11t/G23/G24/G25/G26):** Jinja2 template-contract test (renders every template incl. holiday `BriefSnapshot`, no `UndefinedError`); HTML snapshot of fragment + email; SVG marker `y` within 2px of expected; email badge bg matches `DESIGN_TOKENS` per mode + badge text present; mock-clock stale/poll-boundary tests; programmatic WCAG-contrast + non-color-encoding (glyph/`aria-label`/`role`) assertions | `voltagent-qa-sec:test-automator` |
| 4.6 | Accessibility + **mobile validation (G27)** at 390/768/1280px (no horizontal scroll, ladder fits), 30-second readability; Playwright visual reg if in scope | `voltagent-qa-sec:accessibility-tester` + `voltagent-qa-sec:ui-ux-tester` |

**Gate**: `premarket.home` renders the brief and all states (normal / RED / stale / holiday); template-contract, snapshot, contrast, and email-fidelity tests pass; a test email arrives in INBOX (not spam, G14d) and renders in Gmail + Apple Mail + Outlook; mobile verified at the three widths.

---

## Phase 5 - Scheduler, recovery, deployment, alerting (Days 20-22)

**Goal**: The 07:00 ET job fires unattended, recovers from a missed run, and fails LOUDLY.

| Step | What | Agent |
|---|---|---|
| 5.1 | `scheduler/jobs.py`: APScheduler ET CronTriggers (06:30 ingest, 07:00 brief+email, 17:30 settlement); **missed-run recovery capped at ONE brief, never auto-backfill (G12)**; SLA: ingest <10min, brief <5min | `voltagent-lang:python-pro` |
| 5.2 | `tests/test_scheduler.py`: jobs fire at correct ET times across DST transitions (mock clock) | `voltagent-qa-sec:test-automator` |
| 5.3 | Final `docker-compose.yml`, `Dockerfile`, healthcheck, bind-mount backup, `.env`. **Container hardening (G14): non-root `USER`, `read_only`+`tmpfs:/tmp`, `cap_drop:[ALL]`, `no-new-privileges`, mem/pids/cpus limits** | `voltagent-infra:docker-expert` |
| 5.4 | NPM proxy host + AdGuard CNAME for `premarket.home`; join `npm_network`. **NPM Access List (Basic Auth) + HTTPS at NPM so the credential is not plaintext on the LAN (G11)** | `voltagent-infra:devops-engineer` |
| 5.5 | Triple failure alerting: MQTT `premarket/alerts` -> HA phone notify; SMTP failsafe; Uptime Kuma on `/health`. **Terse payloads (date/failed/stage, no raw error/config), failure-only idempotent HA automation (G14c)**; alert when MTD spend nears the monthly cap (G12) | `voltagent-infra:sre-engineer` |
| 5.6 | Security + exposure review (G11/G13/G14 verification: auth in place, no secrets in image/logs, hardening applied); **vendor-outage runbook `docs/outages.md` (G20)**; **ownership/cost note: Databento owner Bryce, ~$3-8/mo, annual key rotation, cost on `/health` (G21)** | `voltagent-qa-sec:security-auditor` + Bryce |

**Gate**: container runs 48 hours unattended with the 07:00 job firing and logging; `premarket.home` requires auth; a deliberately broken feed triggers a phone alert via at least one channel; the monthly-spend-cap path is tested.

---

## Phase 6 - Validation, rollout, and ship (Days 23-25 build, then a ~1-week unattended soak)

**Goal**: Allie relies on it on a real trading morning, and it is operable for the long run.

| Step | What | Agent |
|---|---|---|
| 6.1 | **Automated end-to-end test (G4)**: load golden fixture, `GET /api/brief/{date}`, assert badge/gap/ATR/catalyst-count + snapshot the render. Plus replay 3 real historical dates vs a trusted NT chart | `voltagent-data-ai:data-analyst` + `voltagent-qa-sec:test-automator` |
| 6.2 | Edge cases at the deployment layer: DST days, CME holiday ("no session"), half-day early close | `voltagent-qa-sec:test-automator` |
| 6.3 | **Structured UAT (G15)**: Allie validates a checklist on a real brief (e.g. identifies Risk Mode + correct no-trade window in <30s unaided, sizing line shows her dollar risk). PRECEDES the quant/risk content sign-off | Allie (sign-off) + `voltagent-domains:quant-analyst` + `voltagent-domains:risk-manager` |
| 6.4 | **Rollout / cutover runbook (G17)**: confirm email lands in inbox not spam; Allie bookmarks `premarket.home` on her phone; one-page "first morning checklist"; Day-1 handoff ping | Bryce + Allie |
| 6.5 | **Disaster recovery (G19)** `docs/disaster_recovery.md`: restore DuckDB BEFORE any re-backfill; rebuild sequence; one tested restore; add a quarterly restore check to `~/maintenance/` | `voltagent-infra:sre-engineer` |
| 6.6 | **Feedback cadence (G18)**: weekly month 1, biweekly month 2, monthly after; `CHANGELOG.md`; zero-opens-3-days -> same-day check-in | Bryce + Allie |
| 6.7 | Final review + README for the service | `feature-dev:code-reviewer` + `voltagent-dev-exp:readme-generator` |

**Gate (service Definition of Done, below)**: all DoD criteria met.

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

Schedule: phases run sequentially (each gate must pass before the next starts), Day 1 through Day 25 of active build. Definition-of-Done criterion #4 then requires **5 real trading mornings** of unattended operation, which is roughly one additional calendar week of soak with no active dev, putting the all-in estimate at **~25 active build-days / ~28 calendar days** at half-time (the folded-in correctness, hardening, and human-loop work added ~4-6 days over the original 22).

The five build-blockers (G1/G3/G11/G12/G16) are front-loaded deliberately: **G3** (DQ gate) first surfaces in Phase 1 and completes in Phase 3; **G1** (correctness oracle) in Phase 3; **G16** (Allie design review) is the Phase 3.5 gate; **G11** (dashboard auth) in Phase 5; **G12** (spend cap) begins in Phase 1 (request guard + MTD table) and completes in Phase 5 (recovery cap + alerting). None can be deferred to "after ship."
