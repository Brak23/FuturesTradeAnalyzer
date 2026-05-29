# Bar Data Vendor Research and Decision

**Question**: best market-data vendor for an individual building a self-hosted (home server, Docker, Python) pre-market tool for the Nasdaq-100 futures (NQ/MNQ), needing full-session (Globex) intraday bars + official CME settlements, on a budget.

**Decision**: **Databento (`GLBX.MDP3`)**. Rationale and the field below.

---

## What actually differentiates vendors

Two requirements split the field:
1. **Official CME settlements** programmatically (distinct from last-traded price).
2. **Full Globex-session intraday bars** (not just 09:30-16:00 ET RTH).

Nuance: almost no vendor hands you an `RTH`/`ETH` flag. You derive the session from the bar timestamp against CME hours (done in `session.py`). So the real ask is "full-session 1-min bars + official settlements + clean Python."

---

## Comparison

| Vendor | Globex intraday bars | Official settlements | Python -> DataFrame | Realistic cost (1 user) | Verdict |
|---|---|---|---|---|---|
| **Databento** (`GLBX.MDP3`) | Yes, full MDP 3.0 feed (`ohlcv-1m`/`5m`); derive RTH/ETH from ts | **Yes, `statistics` schema** (settlement + open interest) | Yes, first-class (`databento` pkg) | Pay-as-you-go; $125 free credits; ~$2.17 for 5 days of one contract's *trades* (OHLCV is cheaper); one instrument = single-digit $ backfill + a few $/mo | **Chosen** |
| NinjaTrader + Kinetick | Yes, loads intraday incl. Globex locally | Weak (daily bar approximates settle, not official) | No native API; CSV bridge | Free EOD is daily-only (insufficient); intraday ~$13.65/mo CME non-pro | Cheap if already on NT, but manual + no official settle |
| Interactive Brokers API | Yes (`reqHistoricalData(useRTH=0)`) | No clean settlement endpoint | Yes, but pacing-limited | ~$1.50/mo CME non-pro (or $10 bundle, waived by commissions) | Cheapest, but painful multi-year 1-min backfills |
| Polygon.io / "Massive" | Yes, 10+ yrs, minute aggregates | Unclear / not its strength | Yes | Flat monthly, pricier than Databento PAYG for one instrument | Viable, no settlement edge |
| CQG / Rithmic / dxFeed | Yes (pro feeds) | Yes | Enterprise SDKs | Enterprise pricing + redistribution licensing | Overkill for one user |
| Barchart OnDemand | Yes, has settlements | Yes | Yes (`getHistory`) | Opaque / quote-based | Skip, pricing friction |

---

## Why Databento wins for this use case

- Only option that cleanly nails BOTH hard requirements: `ohlcv-1m` for the full Globex session AND the `statistics` schema for **official settlements + open interest** (the exact gap the architecture flagged).
- Pay-as-you-go beats a flat monthly plan for a single instrument: cents-to-dollars to backfill NQ, a few dollars/month ongoing, $125 free credits to start.
- Native Python returning DataFrames; no trading platform or CSV bridge required.
- The `BarSource` adapter built against it also resolves FuturesTradeAnalyzer's open bar-source decision (PROJECT_KICKOFF 4.3 / ADR-0002).

**Cheapest fallback (not chosen)**: if Allie already runs NinjaTrader with a Globex-capable feed, use NT's local intraday bars for free and pull only `statistics` settlements from Databento. Rejected as the primary because it is manual (CSV bridge) and lacks a clean official-settlement path; kept as a documented option if Databento cost ever becomes a concern.

---

## Sources

- Databento GLBX.MDP3 dataset: https://databento.com/datasets/GLBX.MDP3
- Databento, retrieving OI and settlement prices: https://databento.com/docs/examples/futures/retrieving-oi-and-settlement-prices
- Databento pricing: https://databento.com/pricing ; API demo (Python): https://databento.com/blog/api-demo-python
- Polygon/Massive futures API: https://polygon.io/futures ; pricing: https://polygon.io/pricing
- Polygon (Massive) vs Databento, Alphanume: https://www.alphanume.com/blog/polygon-massive-vs-databento
- Kinetick for NinjaTrader: https://kinetick.com/NinjaTrader ; CME non-pro fees: https://kinetick.com/CME ; Cheapest futures data for NT8 (2025): https://www.xabcdtrading.com/blog/cheapest-futures-data-for-ninjatrader-8/
- IBKR market data pricing: https://www.interactivebrokers.com/en/pricing/market-data-pricing.php ; TWS API historical data: https://interactivebrokers.github.io/tws-api/historical_data.html
