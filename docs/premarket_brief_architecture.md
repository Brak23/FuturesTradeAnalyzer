# Pre-Market Brief: Architecture Blueprint

**Service name**: `premarket`
**Target domain**: `premarket.home`
**Host**: homehub (`~/docker/premarket/`)
**Stack anchor**: Python 3.11+, FastAPI, HTMX + Jinja2, DuckDB, APScheduler (DST-aware), UTC-internal / ET-at-display discipline throughout.
**Companion docs**: `premarket_brief_design.md` (product + content), `premarket_brief_build_plan.md` (step-by-step + agent assignments).

---

## 0. The blocker, first

**Feed A (session-tagged overnight/Globex bars + official daily settlements) is the single architectural blocker.** Everything else can be built and tested against synthetic data; the brief cannot fire correctly without Feed A. Resolve it before writing the brief engine.

Ranked options:

| Source | Session tags | Settlement | Cost | Effort |
|---|---|---|---|---|
| **Databento** (recommended) | Yes, first-class (`GLBX.MDP3`) | Yes | ~$30-40/mo for NQ | 1-2 days |
| Tradovate / AMP broker API | Yes (ETH/RTH) | Yes | Free if Allie trades there | Broker-dependent |
| NinjaTrader 8 local export | Yes if ETH enabled | Yes | Free | Manual/scripted export |
| Polygon.io paid | RTH flagged; Globex needs aggregation | Yes | $79/mo | Moderate |
| yfinance / Yahoo | No | No | Free | Disqualified |

**Recommendation**: Databento `GLBX.MDP3`, schema `ohlcv-1m`, with the settlement record at EOD. Cleaner than fighting broker APIs and the cost is below the engineering time saved. If Allie's NinjaTrader already has an ETH-capable feed, NT8 local export is free and equally clean (different ingestion module, identical rest-of-stack).

**Verification test (run before trusting any source)**: pull a Tuesday's bars. You must see a continuous series from 18:00 ET Monday through 17:00 ET Tuesday with a gap only for the 17:00-18:00 ET CME maintenance halt. If you only see 09:30-16:00 ET, the source is RTH-only and disqualified, regardless of its documentation.

---

## 1. Architecture

```
                  External sources: Databento (or NT8) | CBOE/yfinance VIX | FMP calendar+earnings
                                            |
                                            v
+------------------------------------------------------------------------------+
|                              premarket-app container                         |
|                                                                              |
|  Feed workers          Scheduler (APScheduler)        Brief engine           |
|  feed_bars.py          06:30 ET ingest                levels.py              |
|  feed_vix.py     --->  07:00 ET brief + email   --->  overnight.py           |
|  feed_econ.py          17:30 ET settlement refresh    volatility.py          |
|  feed_earn.py          DST-aware (America/New_York)   catalysts.py           |
|       |                                                    |                 |
|       v                                                    v                 |
|  DuckDB file  <----------------------------------->  FastAPI + HTMX/Jinja2   |
|  (named vol)                                         GET /  GET /api/*       |
|                                                            |                 |
|  Email sender (SMTP) <-------------------------------------+                 |
+------------------------------------------------------------------------------+
        |  :8090 (loopback)                |  MQTT publish on failure
        v                                  v
  nginx-proxy-manager                 mosquitto 192.168.50.200
  premarket.home                      (HA phone alert)
```

### Container boundaries: one app container + DuckDB file

Single user, single instrument, runs once per day, total runtime seconds-to-minutes. Splitting scheduler / worker / API into separate containers would force a network DB or multi-container volume mounts for zero benefit. **Two containers at most**: `premarket-app` plus an optional Postgres sidecar later if DuckDB durability is ever insufficient (it will not be at this scale). One container = one log stream, one healthcheck, one restart policy.

---

## 2. Tech choices and rationale

- **Backend: FastAPI.** Matches the parent repo (Python 3.11+, pydantic). Tiny API surface (3-5 routes); async lets the dashboard render from DuckDB without blocking the 07:00 job.
- **Frontend: HTMX + Jinja2, server-rendered.** Read-only, single-user, one daily view. A SPA's Node build pipeline and state management are pure overhead. HTMX adds the one interactive piece (a poll-to-refresh during the pre-market window) with two HTML attributes and no build step. Reuses the parent repo's Jinja2 ADHD-friendly template conventions. Alpine.js can be added later for a sparkline with zero build tooling.
- **Storage: DuckDB (single file, named volume).** First-class time-series aggregation and native parquet IO, no second container, no connection pool. Better than SQLite for OHLCV; simpler than Postgres at this scale; better than parquet-scan for request-time level queries. DAO layer keeps a Postgres swap to one file if ever needed.
- **Scheduler: APScheduler in-process, `AsyncIOScheduler`, `CronTrigger(timezone='America/New_York')`.** DST-correct without offset math (the failure mode a UTC container cron hits). Startup check fires a missed brief if the container restarted during the 07:00-08:30 window.
- **Market data**: Feed A Databento or NT8 (see section 0); VIX from CBOE daily CSV or `yfinance ^VIX` (prior close only); econ calendar + earnings from **FMP** (free tier, one API key, has an `impact` field and `bmo/amc` timing). Maintain a static `NDX_COMPONENTS` list in `config.py` (refreshed quarterly) to filter earnings without paying for a constituents API. **One paid API total (Databento), or zero if NT8.**

---

## 3. docker-compose.yml

```yaml
# ~/docker/premarket/docker-compose.yml
services:
  app:
    build:
      context: ./app
      dockerfile: Dockerfile
    container_name: premarket-app
    restart: unless-stopped
    env_file: .env
    volumes:
      - ./data:/data            # bind mount so ~/backup.sh tarball covers the DuckDB file
      - ./logs:/logs
      # - ./nt-exports:/nt-exports:ro   # uncomment for the NT8 local-export path
    ports:
      - "127.0.0.1:8090:8080"   # loopback only; NPM proxies externally
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 60s
      timeout: 10s
      retries: 3
      start_period: 30s
    logging:
      driver: "json-file"
      options: { max-size: "10m", max-file: "5" }
    networks:
      - npm_network

networks:
  npm_network:
    external: true
    name: nginx-proxy-manager_default   # confirm: docker network ls
```

`.env` (gitignored; ship `.env.example`):

```bash
DATABENTO_API_KEY=db-xxxx           # or NT8_EXPORT_PATH=/nt-exports
FMP_API_KEY=xxxx
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=bryce.remington@gmail.com
SMTP_PASSWORD=app-specific-password
BRIEF_TO_EMAIL=allie@example.com
BRIEF_BCC_EMAIL=bryce.remington@gmail.com
MQTT_HOST=192.168.50.200
MQTT_PORT=1883
MQTT_ALERT_TOPIC=premarket/alerts
TZ=UTC                              # container UTC; app converts to ET at display
DB_PATH=/data/premarket.duckdb
BRIEF_HISTORY_DAYS=90
```

**NPM proxy host** (set in the NPM UI): domain `premarket.home`, scheme `http`, forward host `premarket-app`, port `8090` (container name resolves on the shared `npm_network`), block common exploits on, SSL per your existing pattern. AdGuard Home: CNAME `premarket.home` -> homehub.

---

## 4. Repo / file structure

```
~/docker/premarket/
  docker-compose.yml
  .env  /  .env.example
  app/
    Dockerfile
    pyproject.toml  uv.lock  .python-version
    src/
      config.py            # session boundaries, ATR/ONR lookbacks, NDX_COMPONENTS, DST_ZONE
      contracts/bar_source.py     # BarSource protocol (reused from FuturesTradeAnalyzer)
      feeds/
        bars_databento.py  bars_nt8.py        # one BarSource impl; nothing else imports vendor SDKs
        vix.py  econ_calendar.py  earnings.py
      ingest/
        run_ingest.py      # orchestrates all feeds (06:30 ET)
        normalizer.py      # UTC normalize, session tag, OHLC DQ
      store/
        db.py  schema.sql  queries.py          # all SQL named; no inline SQL elsewhere
      brief/
        levels.py  overnight.py  volatility.py  catalysts.py
        engine.py          # build_brief() -> BriefSnapshot
        session.py         # is_rth()/is_eth(), CME holiday + halt logic, ZoneInfo("America/New_York")
      email_sender/send.py
      scheduler/jobs.py    # CronTrigger ET; missed-run recovery
      api/
        main.py  routes_dashboard.py  routes_api.py  deps.py
      templates/
        base.html  dashboard.html  brief_fragment.html  email.html
    tests/
      conftest.py  test_session.py  test_levels.py  test_overnight.py
      test_volatility.py  test_catalysts.py  test_feeds.py  test_ingest.py  test_scheduler.py
```

`Dockerfile`:

```dockerfile
FROM python:3.11-slim
RUN pip install uv
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev
COPY src/ ./src/
COPY tests/ ./tests/
ENV PYTHONPATH=/app
CMD ["uv", "run", "uvicorn", "src.api.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

---

## 5. Data model (DuckDB)

- **`bars`** (PK ts, symbol): session-tagged 1-min OHLCV for NQ (reference series, labeled for MNQ/NQ in output). `session` is `RTH` | `ETH`. Keep all history (~50 MB / 5yr). Index on (ts, symbol).
- **`settlements`** (PK settle_date, symbol): official CME daily settlement. Indefinite retention.
- **`computed_levels`** (PK level_date, symbol): pre-computed daily set written by the 06:30 job so the brief reads, not recomputes: pdh/pdl/pdc, prior_settle, onh/onl, on_open, on_vwap, weekly/monthly H/L, weekly/monthly open, atr_14, avg_onr_20, current_price, gap_pts, gap_pct, gap_as_atr_frac. Retain 90 days.
- **`calendar_events`** (PK event_id = hash(date+name+et)): event_date, event_name, scheduled_et, importance, actual/forecast/previous. Retain 30 days.
- **`earnings_events`** (PK event_id): symbol, report_date, timing (`bmo`/`amc`), eps_est/eps_actual. Retain 30 days.
- **`brief_snapshots`** (PK brief_date): brief_json (full BriefSnapshot dump), brief_html, email_html, email_sent_at, email_error, ingest_ok, ingest_errors[]. Audit trail of "what the brief said." Retain 90 days.

Schema managed in `schema.sql` with `CREATE TABLE IF NOT EXISTS`; schema changes get a migration script under `src/store/migrations/`.

---

## 6. Operations

- **Secrets**: all in `~/docker/premarket/.env` via compose `env_file` (not inline `environment:`, so they do not show in `docker inspect`). Covered by the host `~/backup.sh` tarball rotation. Rotate Databento + FMP keys at setup.
- **Backups**: bind-mount `./data` (above) so the DuckDB file lives under `~/docker/premarket/` and is tarballed by `backup.sh` automatically.
- **Failure alerting (triple layer, reuses existing infra so a silent 07:00 failure is near-impossible)**:
  1. On any `job_brief` exception, publish to MQTT `premarket/alerts`; a Home Assistant automation on the existing `192.168.50.200` broker sends Bryce a critical phone notification.
  2. If the MQTT publish itself fails, fall through to a plain-text SMTP email to `BRIEF_BCC_EMAIL` ("[premarket] FAILED <date>").
  3. `GET /health` returns `{status, last_brief, last_ingest}`; the existing **Uptime Kuma** monitors it and alerts on a down container or failed check.
- **Updates**: `docker compose build --pull && docker compose up -d`. Run `pytest` before rebuild. Dependency bumps via `uv lock --upgrade`.
- **Logging**: structured JSON to stdout (`structlog`, matching the parent repo), `json-file` driver capped at 10m x 5. Each run emits a one-line summary (`brief_complete` with durations and feed status; `brief_failed` with stage + error).

---

## 7. Alignment with FuturesTradeAnalyzer

- `BarSource` protocol reused directly; the Databento (or NT8) adapter built here is a drop-in for the analytics project once its bar-source decision (PROJECT_KICKOFF 4.3) is made. Building this service first is the forcing function to resolve that decision.
- Identical UTC/DST discipline (`zoneinfo`, `pytz` banned), pydantic v2 contracts, ruff (line length 100), green/red/gray color convention, no em-dashes in generated docs/reports.
- Read-only consumer: no trade-history dependency, so it ships standalone and stays useful even if the rubric is shelved at M0.

---

## 8. Decisions requiring Bryce's input

1. **Feed A vendor**: Databento (recommended, unblocks immediately) vs NT8 local export (free if Allie has ETH data enabled). Determines `bars_databento.py` vs `bars_nt8.py`.
2. **SMTP path**: Gmail app-specific password (fastest), a self-hosted SMTP on homehub, or a transactional relay (Resend/Postmark free tier).
3. **FMP tier**: free tier (250 calls/day) is ample at ~3 calls/day; just register for a key.
