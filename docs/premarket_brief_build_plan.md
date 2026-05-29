# Pre-Market Brief: Step-by-Step Build Plan and Agent Assignments

**Purpose**: A complete, ordered build guide for the `premarket` service, naming WHICH agent to use for each step. Companion to `premarket_brief_design.md` (product + content) and `premarket_brief_architecture.md` (the blueprint this executes).

**Audience**: Bryce, or a future Claude Code session driving the build.

**Working model**: This is a `~/docker/premarket/` service on homehub, built test-first, in vertical slices, with a review gate at the end of each phase. The bar feed (Feed A) is the only hard blocker and comes first.

---

## 0. How to run this build

### Orchestration model
- **One phase at a time.** Each phase has a clear deliverable and a review gate. Do not start phase N+1 until phase N's gate passes.
- **Test-first.** Invoke `superpowers:test-driven-development` before writing implementation code for any phase. Every feature/compute function gets a synthetic-fixture test plus, where relevant, a point-in-time / session-boundary test.
- **Plan before code.** For the first build session, invoke `superpowers:writing-plans` to turn this doc into a living checklist, then `superpowers:executing-plans` (or `superpowers:subagent-driven-development`) to drive it with review checkpoints.
- **Review gate every phase.** End each phase with `feature-dev:code-reviewer` (or `voltagent-qa-sec:code-reviewer`) and, for the secrets/email/exposure surface, `voltagent-qa-sec:security-auditor`. Use the `superpowers:requesting-code-review` skill to frame the request.
- **Isolation.** Build in a git worktree (`superpowers:using-git-worktrees`) or a dedicated branch; this service is a new repo under `~/docker/premarket/`, separate from FuturesTradeAnalyzer.

### Who decides vs who builds
The three open decisions in `premarket_brief_architecture.md` section 8 (Feed A vendor, SMTP path, FMP tier) are Bryce's calls and gate Phase 1. Agents build once those are set.

### Dispatch note
Agent names below are the subagent types available in this environment. Dispatch them via the Agent tool. Independent steps within a phase can run in parallel (one message, multiple Agent calls). If you want fully deterministic multi-agent orchestration with fan-out and review loops, drive it with a Workflow (opt-in) using these same agent assignments.

---

## Phase 0 — Decisions and scaffold (Days 1-2)

**Goal**: Repo skeleton exists; the three blocking decisions are made; CI and tooling run green on an empty app.

| Step | What | Agent |
|---|---|---|
| 0.1 | Confirm Feed A vendor (Databento vs NT8), SMTP path, FMP tier | Bryce (decision) |
| 0.2 | Scaffold `~/docker/premarket/app/` tree, `pyproject.toml` (uv), `ruff` + `pytest` config, `.python-version`, `.env.example`, `.gitignore` | `voltagent-dev-exp:build-engineer` or `voltagent-lang:python-pro` |
| 0.3 | `src/config.py` with all named constants (session boundaries, ATR/ONR lookbacks, `DST_ZONE`, `NDX_COMPONENTS`) | `voltagent-lang:python-pro` |
| 0.4 | `src/contracts/bar_source.py` protocol (adapt from FuturesTradeAnalyzer) | `voltagent-core-dev:backend-developer` |
| 0.5 | Empty FastAPI app + `/health` route so the container boots | `voltagent-lang:fastapi-developer` |

**Gate**: `uv run pytest` and `ruff check` pass on the skeleton; `docker compose up` boots and `/health` responds. Review with `feature-dev:code-reviewer`.

---

## Phase 1 — Bar feed vertical slice (Days 3-6) — THE BLOCKER

**Goal**: Session-tagged overnight bars + settlements land in DuckDB and are verifiable by eye. Nothing else is built until this works.

| Step | What | Agent |
|---|---|---|
| 1.1 | `src/feeds/bars_databento.py` (or `bars_nt8.py`) implementing `BarSource`; fetch one week of NQ bars | `voltagent-data-ai:data-engineer` (lead) + `voltagent-lang:python-pro` |
| 1.2 | `src/store/db.py` + `schema.sql`: DuckDB init, `bars` + `settlements` tables | `voltagent-data-ai:database-optimizer` |
| 1.3 | `src/ingest/normalizer.py`: UTC normalization, session tagging (RTH/ETH), OHLC integrity DQ | `voltagent-data-ai:data-engineer` |
| 1.4 | `src/brief/session.py`: `is_rth()`/`is_eth()`, CME halt + holiday calendar, `ZoneInfo` boundaries | `voltagent-lang:python-pro` |
| 1.5 | `tests/test_session.py` (DST March/Nov boundaries, CME holidays) + the **verification test** (continuous 18:00 ET -> 17:00 ET series with only the halt gap) | `voltagent-qa-sec:test-automator` |

**Gate (HARD KILL)**: bars query back from DuckDB with a correct `session` split and the verification test passes. If no available source yields session-tagged Globex bars + settlements after a genuine attempt, the brief cannot be built as specified. Stop and revisit Feed A.

---

## Phase 2 — Remaining feeds (Days 7-9)

**Goal**: VIX, econ calendar, earnings all fetch and parse independently, behind tests.

| Step | What | Agent |
|---|---|---|
| 2.1 | `src/feeds/vix.py` (CBOE daily CSV or `yfinance ^VIX`, prior close) | `voltagent-data-ai:data-engineer` |
| 2.2 | `src/feeds/econ_calendar.py` (FMP; normalize importance; ET timestamps) | `voltagent-data-ai:data-engineer` |
| 2.3 | `src/feeds/earnings.py` (FMP; filter to `NDX_COMPONENTS`; `bmo/amc` timing) | `voltagent-data-ai:data-engineer` |
| 2.4 | `src/ingest/run_ingest.py` orchestrator returning per-feed success/error | `voltagent-core-dev:backend-developer` |
| 2.5 | `tests/test_feeds.py` with mocked HTTP responses; `calendar_events`/`earnings_events` tables | `voltagent-qa-sec:test-automator` |

**Gate**: `run_ingest` runs clean against real APIs; each feed has passing mocked-response tests. Review with `feature-dev:code-reviewer`.

---

## Phase 3 — Brief engine (Days 10-13)

**Goal**: Compute a full `BriefSnapshot` from the database. No UI yet. This is where the quant/risk content spec becomes code.

| Step | What | Agent |
|---|---|---|
| 3.1 | `src/brief/levels.py` (PDH/PDL, ONH/ONL, ON VWAP, conditional weekly/monthly H/L, settlement-based gap) | `voltagent-lang:python-pro` |
| 3.2 | `src/brief/overnight.py` (net change, ON range vs trailing-20 avg, trend-vs-chop) | `voltagent-lang:python-pro` |
| 3.3 | `src/brief/volatility.py` (ATR(14), gap-as-fraction-of-ATR, VIX expected move) | `voltagent-lang:python-pro` |
| 3.4 | `src/brief/catalysts.py` (merge + time-sort econ/earnings, filter high/medium, cap 3) | `voltagent-lang:python-pro` |
| 3.5 | **Risk Mode badge logic** + sizing line (GREEN/YELLOW/RED rules; dollar-risk-constant sizing; echoes Allie's pre-set limits) | `voltagent-domains:risk-manager` (spec) + `voltagent-lang:python-pro` (impl) |
| 3.6 | `src/brief/engine.py` `build_brief()`; `computed_levels` + `brief_snapshots` tables | `voltagent-core-dev:backend-developer` |
| 3.7 | Synthetic-fixture tests for every compute module with known expected outputs | `voltagent-qa-sec:test-automator` |
| 3.8 | **Content correctness review**: levels, no-direction rule, cut list honored, false-precision rounding | `voltagent-domains:quant-analyst` + `voltagent-domains:risk-manager` |

**Gate**: `build_brief(date)` returns a fully populated, correct `BriefSnapshot` against known inputs; quant + risk sign off on content. Review code with `feature-dev:code-reviewer`.

---

## Phase 4 — Dashboard + email (Days 14-17)

**Goal**: The brief is visible at `premarket.home` and delivered by 07:00 email, in the ADHD-friendly layout.

| Step | What | Agent |
|---|---|---|
| 4.1 | ADHD-friendly layout: Risk Mode badge, headline, catalyst banner, **price-ladder visual**, vol/sizing line; green/red/gray convention; one-screen-then-drill-down | `voltagent-core-dev:ui-designer` |
| 4.2 | `templates/base.html`, `dashboard.html`, `brief_fragment.html`; HTMX poll-to-refresh during 06:00-10:00 ET only | `voltagent-core-dev:frontend-developer` |
| 4.3 | `api/routes_dashboard.py` + `routes_api.py` (`/`, `/api/brief/latest`, `/api/brief/{date}`, `/health`) | `voltagent-lang:fastapi-developer` |
| 4.4 | `email_sender/send.py` (SMTP+TLS) + `templates/email.html` with inlined CSS (premailer) | `voltagent-core-dev:backend-developer` |
| 4.5 | Accessibility / scannability pass (color-contrast, mobile, 30-second readability) | `voltagent-qa-sec:accessibility-tester` |

**Gate**: `premarket.home` renders the brief; a test email arrives and renders correctly in Gmail and Apple Mail; mobile layout verified (use the `verify` skill or `voltagent-qa-sec:ui-ux-tester`).

---

## Phase 5 — Scheduler, recovery, deployment, alerting (Days 18-20)

**Goal**: The 07:00 ET job fires unattended, recovers from a missed run, and fails LOUDLY.

| Step | What | Agent |
|---|---|---|
| 5.1 | `scheduler/jobs.py`: APScheduler ET CronTriggers (06:30 ingest, 07:00 brief+email, 17:30 settlement); missed-run recovery on startup | `voltagent-lang:python-pro` |
| 5.2 | `tests/test_scheduler.py`: jobs fire at correct ET times across DST transitions (mock clock) | `voltagent-qa-sec:test-automator` |
| 5.3 | Final `docker-compose.yml`, `Dockerfile`, healthcheck, bind-mount for backup coverage, `.env` wiring | `voltagent-infra:docker-expert` |
| 5.4 | NPM proxy host + AdGuard CNAME for `premarket.home`; join `npm_network` | `voltagent-infra:devops-engineer` |
| 5.5 | Triple failure alerting: MQTT publish to `premarket/alerts` -> HA phone notify; SMTP failsafe; Uptime Kuma monitor on `/health` | `voltagent-infra:sre-engineer` |
| 5.6 | Secrets + exposure review (`.env` handling, loopback bind, no secrets in image, SMTP creds) | `voltagent-qa-sec:security-auditor` |

**Gate**: container runs 48 hours unattended with the 07:00 job firing and logging; a deliberately broken feed triggers a phone alert via at least one channel.

---

## Phase 6 — Validation and ship (Days 21-22)

**Goal**: Allie can use it on a real trading morning.

| Step | What | Agent |
|---|---|---|
| 6.1 | Replay `build_brief()` on 3 real historical dates; verify PDH/PDL/gap against a trusted NT chart | `voltagent-data-ai:data-analyst` |
| 6.2 | Edge cases: DST days, CME holiday ("no trading today"), half-day | `voltagent-qa-sec:test-automator` |
| 6.3 | End-to-end + content acceptance (does a real morning read correctly, is it scannable in 30s) | `voltagent-domains:quant-analyst` + `voltagent-domains:risk-manager` |
| 6.4 | Final review + README for the service | `feature-dev:code-reviewer` + `voltagent-dev-exp:readme-generator` |

**Gate**: a real-morning brief renders at `premarket.home`, the 07:00 email is received and correct, and quant + risk accept the content. Ship.

---

## Component -> agent quick reference

| Component | Primary agent | Support / review |
|---|---|---|
| Repo scaffold, tooling, CI | `voltagent-dev-exp:build-engineer` | `voltagent-lang:python-pro` |
| Bar feed + ingestion (Feed A) | `voltagent-data-ai:data-engineer` | `voltagent-lang:python-pro` |
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

- `superpowers:brainstorming` — before locking any phase's approach if it is unclear.
- `superpowers:writing-plans` then `superpowers:executing-plans` — turn this doc into a tracked checklist and execute with review checkpoints.
- `superpowers:test-driven-development` — required discipline per phase.
- `superpowers:using-git-worktrees` — isolate the build.
- `superpowers:requesting-code-review` / `superpowers:receiving-code-review` — the per-phase review gate.
- `superpowers:finishing-a-development-branch` — when the service is done and ready to integrate.

## Critical-path summary

Feed A (Phase 1) is the only hard blocker; everything downstream depends on session-tagged bars. The vertical-slice MVP is Phases 1-4 (bars -> compute -> dashboard + email). Phases 5-6 make it unattended-reliable and validated. Estimated ~22 days at half-time; the bar-feed slice is the highest-risk and front-loaded deliberately.
