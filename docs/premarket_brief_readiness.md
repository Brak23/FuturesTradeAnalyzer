# Pre-Market Brief: Implementation Readiness and Gap Register

**Purpose**: What is missing to take the pre-market brief from "well-planned" to "complete, buildable, shippable end to end." Synthesized from four completeness-critic reviews (technical architecture, delivery/PM, QA/test, security/ops). Companion to `premarket_brief_{design,ux_design,architecture,build_plan,vendor_research}.md`.

**How to read**: gaps are grouped by theme, tiered CRITICAL / IMPORTANT / NICE, each with a one-line closure and the build-plan phase it slots into. The plan deltas at the end summarize the new gates and phases this adds.

---

## 0. Definition of Done (was missing entirely)

The build plan had per-phase review gates but no service-level "done." The service is DONE when ALL hold:

1. All phase gates passed; `pytest` coverage >= 80% on `brief/` and `feeds/`; CI green.
2. The golden-file correctness oracle (G1 below) and the edge-case matrix (G2) pass.
3. The brief refuses to render on stale/partial/failed-DQ data and surfaces it visibly (G3).
4. Runs **unattended for 5 real trading mornings** with correct briefs archived in `brief_snapshots`.
5. **Allie** reads a live brief, signs off on the UAT checklist (G15), and the rollout runbook (G17) is done.
6. Dashboard is authenticated (G11); monthly spend cap is live (G12); DR runbook + one tested restore exist (G19).

A NO-data day (CME holiday) rendering the correct "no session" state counts as a pass, not a failure.

---

## 1. Correctness and data integrity (the "wrong brief at 07:00" class)

These are the gaps that could silently ship a wrong brief on a high-stakes morning. Highest priority.

| ID | Gap | Tier | Closure | Phase |
|---|---|---|---|---|
| G1 | **No correctness oracle.** No golden-file pinning bar inputs to exact expected PDH/PDL/settlement/ONH/ONL/ON-VWAP/gap/ATR/avg-ONR. ATR must use RTH-only bars while `ohlcv-1m` spans the full Globex session, a silent-wrong trap. | CRITICAL | Build a 30+ day synthetic `bars`+`settlements` fixture (1 DST cross, 1 holiday, 1 half-day, quiet + huge overnight); hand-verify every field; `test_levels_golden.py` asserts the full record. quant-analyst signs the hand-verification. | 3 (3.7/3.8) |
| G2 | **No financial-edge-case matrix.** roll week / contract change (false huge gap), CME half-day, DST spring/fall, gap < 0.2 ATR threshold, flat-then-recover overnight, CPI/FOMC 3-catalyst overflow, partial/late feed, empty FMP, missing VIX. | CRITICAL | `test_edge_cases.py`, one test per scenario off the golden fixture. | 3 + 6.2 |
| G3 | **No data-freshness / DQ gate inside `build_brief()`.** If settlements fail but bars succeed, gap renders wrong silently. No min-bar-count or staleness guard; no sanity asserts (ONH>ONL, settle within ~0.5% of price, gap-as-ATR in [-3,+3]). | CRITICAL | `BriefReadinessCheck` before render: missing settlement / stale bars / low bar-count -> degraded brief (GRAY badge, visible "Data incomplete, verify independently" banner), never silent. Connect `ingest_ok=False` -> `is_stale=True` -> stale UI. | 1.7, 3, 4.2 |
| G4 | **No end-to-end render test.** Phase 6 replay is a manual spot-check, no automated regression through the FastAPI route + templates + SVG. | CRITICAL | `test_e2e_brief.py`: load golden fixture, `GET /api/brief/{date}`, assert badge/gap/ATR/catalyst-count, snapshot the rendered fragment. | 4.3 + 6.1 |
| G5 | **No idempotency/determinism.** Re-run after a 06:30-07:00 restart could produce a different brief than the archived one; no `INSERT OR REPLACE` policy for `bars`/`settlements`/`brief_snapshots`. | CRITICAL | Explicit upsert semantics; `build_brief()` twice on same state asserts field-identical (excl. render timestamp). | 1.3, 3.7 |
| G6 | **CME half-day / early-close not handled at the 07:00 layer.** Truncated RTH changes PDH/PDL; no `session_type` (FULL/HALF/HOLIDAY). | CRITICAL | Add `session_type` to the CME calendar in `session.py`; half-day annotates headline ("Early close 13:00 ET") and adjusts baselines; tested. | 1.6, 1.7, 6.2 |
| G7 | **Categorical thresholds undefined.** "quiet/elevated night," "choppy," the 20-session baseline (weekend/holiday/half-day exclusion) are prose, not numbers. | IMPORTANT | Put `ON_RANGE_QUIET/ELEVATED_MULTIPLIER`, `BASELINE_SESSION_COUNT` (complete non-half non-holiday sessions) in `config.py`; render the actual threshold in the drill-down; tested. | 3 |
| G8 | **Risk Mode badge has no decision table.** GREEN/YELLOW/RED depends on VIX x gap-ATR x OPEX x earnings x catalyst, specified only in prose. | IMPORTANT | risk-manager produces a numeric decision table (thresholds in `config.py`) BEFORE impl; `test_risk_mode.py` parametrized from it. | 3.5 (precedes impl) |

---

## 2. Reliability and external-dependency resilience

| ID | Gap | Tier | Closure | Phase |
|---|---|---|---|---|
| G9 | **DuckDB write/read concurrency unspecified.** Scheduler writes while the API polls every 90s; single-file DuckDB can lock-fault or tear reads. | CRITICAL | Single shared connection via FastAPI lifespan + `asyncio.Lock` around writes, or write-to-staging + `RENAME`. Document in `db.py`. | 1.3 |
| G10 | **No API resilience.** No timeouts/retries/backoff for Databento or FMP; a transient 06:30 blip means no brief and a spurious alert. | IMPORTANT | Wrap calls with `tenacity` (3 tries, expo backoff); typed `FeedTimeoutError` vs `FeedRetryExhaustedError` so transient retries and permanent fails alert+degrade. | 1.1, 2.1-2.3 |
| G10b | **No manual "run now."** Can only test at 07:00. | IMPORTANT | `POST /api/admin/run-now?date=` (token-protected) firing `build_brief()`; doubles as the e2e harness and Allie's "regenerate" button. | 4.3 |
| G10c | **No run-history surface.** Silent partial failure needs log-grep to diagnose. | IMPORTANT | `GET /api/admin/runs?days=7` over `brief_snapshots` (ingest_ok, durations, actual cost, errors). | 4 |
| G10d | **VIX has no fallback/staleness.** | NICE | CBOE CSV then yfinance fallback; `MAX_VIX_STALENESS_DAYS=3` -> "VIX unavailable", badge treats VIX as neutral. | 2 |

---

## 3. Engineering scaffolding (parent-repo conventions not inherited)

| ID | Gap | Tier | Closure | Phase |
|---|---|---|---|---|
| G11s | **Config management unspecified.** Scattered `os.getenv` with no validation. | IMPORTANT | `src/settings.py` via `pydantic-settings` (validates keys, port, positive cost ceiling on startup); `config.py` stays for analytical constants. | 0.3 |
| G11m | **Schema migrations named but not specified** (DuckDB has no Alembic). | IMPORTANT | `schema_version` table + ordered `migrations/V{n}__*.sql` run at startup. | 1.3 |
| G11j | **No local dev workflow / CI.** Parent uses `just` + GitHub Actions; neither specified here. | IMPORTANT | `justfile` (`dev/test/lint/backfill/ingest/brief/check-cost`) and `.github/workflows/ci.yml` (ruff, pytest+cov, docker build, offline on synthetic fixtures). | 0.2 |
| G11t | **Templates can fail silently** (Jinja2 default `Undefined` renders empty). | IMPORTANT | `StrictUndefined`; `test_template_contract.py` renders every template incl. a holiday `BriefSnapshot`, asserts no `UndefinedError` and correct fallbacks. | 0 + 4.2 |

---

## 4. Security and operational resilience (home-LAN threat model)

| ID | Gap | Tier | Closure | Phase |
|---|---|---|---|---|
| G11 | **Dashboard is unauthenticated on the LAN** and exposes Allie's private risk/sizing config and behavioral state. Contradicts the design's own "P&L data is sensitive, separate surface" rule. | CRITICAL | NPM **Access List (Basic Auth)** on `premarket.home`, plus HTTPS at NPM (internal CA) so the credential is not plaintext on the LAN. Build gate. | 5.4, 5.6 |
| G12 | **No monthly/cumulative Databento spend cap.** Per-request guard cannot stop a crash-loop/missed-run-recovery bleed or a price change. | CRITICAL | MTD spend accumulator (`api_spend` table) + `MAX_DATABENTO_MONTHLY_USD` (alert+refuse); set Databento account-side budget too; cap missed-run recovery to ONE brief, never auto-backfill. | 1.2, 5.1, 5.5 |
| G13 | **Secrets hygiene not inherited and arrives too late (Phase 5).** No pre-commit secret scan; no log redaction (Databento client/key can serialize into a structlog exception). | CRITICAL | Phase 0: `.gitignore` + `gitleaks` pre-commit + offline CI (mirror parent 1.7). `structlog` redaction processor masking `*_API_KEY/*_PASSWORD/authorization`; never log client/request objects; tested. | 0.2 |
| G14 | **Container runs as root, writable FS, no resource limits** on a shared host that also runs HA (privileged). | IMPORTANT | Non-root `USER`, `read_only: true` + `tmpfs:/tmp`, `cap_drop:[ALL]`, `no-new-privileges`, `mem/pids/cpus` limits (also bounds the G12 crash-loop). | 5.3 |
| G14b | **No dependency/image vuln scanning;** `uv lock --upgrade` is manual and rots. | IMPORTANT | `pip-audit`/`osv-scanner` + Trivy image scan in CI; Dependabot/Renovate on the repo. | 0.2 |
| G14c | **MQTT broker is `allow_anonymous`** on the LAN: alert spoofing / info leak. | IMPORTANT | Terse alert payloads (date, failed bool, stage; NO raw error/config); failure-only, idempotent HA automation so a spoofed "success" cannot mask a real failure. | 5.5 |
| G14d | **SMTP credential blast radius + deliverability undesigned.** | IMPORTANT | Resolve SMTP toward a relay (Resend/Postmark free) or a DEDICATED Gmail app password; reject self-hosted SMTP (SPF/DKIM rabbit hole). Add inbox-placement (not just render) to the Phase 4 gate. | 2 (decide), 4.4 |
| G15s | **App hardening hygiene.** `/docs` exposed, no security headers. | NICE | `FastAPI(docs_url=None)`, `X-Content-Type-Options: nosniff`, restrictive CSP (self-rendered, no CDN). | 4.3 |

---

## 5. Delivery, validation, and ownership (the human loop)

| ID | Gap | Tier | Closure | Phase |
|---|---|---|---|---|
| G16 | **No user (Allie) design review BEFORE build.** ADHD-friendliness is not code-reviewable; the dense one-screen layout and badge tone must be validated on a prototype first. | CRITICAL | New **Phase 3.5** gate: ui-designer shows wireframes/prototype; Allie confirms 30-second readability, ladder legibility, badge tone is calm not alarmist. Sign-off before Phase 4 code. | 3.5 |
| G15 | **No structured UAT.** Phase 6 "content acceptance" is agent-subjective. | IMPORTANT | A UAT checklist Allie validates on a real brief (e.g., "identifies Risk Mode and the correct no-trade window in <30s, unaided"). Precedes the quant/risk sign-off. | 6.3 |
| G17 | **No rollout/cutover.** "Allie can use it" does not say how it enters her morning. | CRITICAL | Phase 6 rollout runbook: confirm email lands in inbox not spam; phone bookmark; one-page "first morning checklist"; Day-1 handoff ping. | 6 |
| G18 | **No feedback cadence post-launch.** | IMPORTANT | Weekly check-ins month 1, biweekly month 2, monthly after; `CHANGELOG.md`; if zero email opens for 3 days, same-day check-in. | 6+ |
| G19 | **DR is "the tarball exists."** No rebuild/re-backfill runbook; no restore test; backfill could re-spend. | IMPORTANT | `docs/disaster_recovery.md` (restore DuckDB BEFORE any backfill; rebuild sequence); `backfill.py` guard "only run if DB absent/corrupt"; quarterly restore test in `~/maintenance/`. | 6 |
| G20 | **No vendor-outage runbook.** Databento down at 06:30 on a CPI day. | IMPORTANT | Outage playbook: detect, notify Allie ("feed down, brief cancelled, consider stand-aside"), static "unavailable" page; track outages; evaluate backup feed if recurring. | 5 |
| G21 | **Ownership/cost not assigned.** Who owns the Databento bill, key rotation, monthly spend. | IMPORTANT | Ops note: account owner Bryce, est. $3-8/mo, annual key rotation (May 28), cost field on `/health`. | 5.6 |
| G22 | **Databento data-licensing not checked** for LAN display + email to a second person (Allie). | IMPORTANT | Confirm single-user/derived-display terms permit intra-household email + private LAN display; one line in `docs/licensing.md`. Pre-build. | 0 |

---

## 6. Presentation correctness (UX claims need tests)

| ID | Gap | Tier | Closure | Phase |
|---|---|---|---|---|
| G23 | **No snapshot/visual regression** for the dashboard fragment, email, or the SVG ladder coordinate math. | IMPORTANT | HTML string-snapshot of fragment + email; assert current-price `y` within 2px of expected; assert email badge bg matches `DESIGN_TOKENS` per mode. | 4.2, 4.4 |
| G24 | **WCAG/contrast + non-color-encoding claims untested.** | IMPORTANT | `test_accessibility.py`: programmatic contrast check on every token pair; assert direction glyph + Risk icon + `aria-label`/`role` present per state. | 4.5 |
| G25 | **Email render fidelity across clients untested** (Outlook drops bg fills; badge text must stay legible). | IMPORTANT | Assert premailer inlined styles present; assert badge TEXT present regardless of bg; one-time manual 4-client check documented. | 4.4 |
| G26 | **Snapshot-consistency unspecified.** Email at 07:00 vs dashboard reloaded later must agree. | IMPORTANT | Brief is a 07:00 point-in-time snapshot; poll stops 07:30; timestamp frozen "Snapshot: 07:00 ET"; mock-clock stale tests (DST-safe). | 4.2, 5.2 |
| G27 | **Mobile validation vague.** | IMPORTANT | Explicit checks at 390/768/1280px (no horizontal scroll, ladder fits), Playwright visual reg if in scope; assign ui-ux-tester. | 4.5 |

---

## 7. Plan deltas (what changes in the build plan)

- **Phase 0 grows**: add `settings.py` (pydantic-settings), `justfile`, GitHub Actions CI, `gitleaks` pre-commit + offline CI, log-redaction processor, `StrictUndefined`, vuln scanning, and a Databento-licensing + SMTP-path decision. Secrets hygiene moves here from Phase 5.
- **New Phase 3.5**: Allie design-review gate before any Phase 4 code.
- **Phase 3 grows**: DQ/freshness gate, sanity asserts, golden-file oracle, edge-case matrix, idempotency, categorical thresholds in config, and the Risk Mode decision table (risk-manager) before impl.
- **Phase 4 grows**: admin run-now + runs endpoints, snapshot/contrast/email-fidelity/template-contract tests, mobile validation, snapshot-consistency, app hardening headers.
- **Phase 5 grows**: NPM auth + HTTPS, monthly spend cap + recovery cap, container hardening, MQTT payload hygiene, vendor-outage runbook, ownership/cost note.
- **Phase 6 grows**: UAT checklist, rollout/cutover runbook, DR runbook + restore test, feedback cadence, and the service Definition of Done.
- **Realistic effort**: these add roughly 4-6 days to the ~22-day estimate, concentrated in Phase 0 (hardening/scaffold) and Phase 3 (correctness). They are the difference between a demo and something Allie can trust with a real trading decision.

**The five to block the build on**: G3 (DQ gate), G1 (correctness oracle), G11 (dashboard auth), G12 (monthly spend cap), G16 (Allie design review). Everything else can land within its phase.
