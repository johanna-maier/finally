# Massive API Reference (formerly Polygon.io)

Reference for the **Massive** market-data REST API as used by FinAlly to retrieve
real-time and end-of-day stock prices for multiple tickers.

> **Status:** The market-data subsystem is already implemented in
> `backend/app/market/`. This document describes the API surface that the shipped
> `MassiveDataSource` (`backend/app/market/massive_client.py`) depends on, plus the
> endpoints worth knowing for future work (detail charts, historical data).

## Background: Polygon.io is now Massive

Polygon.io rebranded to **Massive** on **October 30, 2025**. For our purposes the
change is cosmetic:

- New API base URL: `https://api.massive.com`
- Legacy base URL `https://api.polygon.io` remains supported for an extended period
- Existing API keys, accounts, and integrations continue to work unchanged
- The official SDKs were renamed (Python: `polygon-api-client` → `massive`) and now
  default to `api.massive.com`

The data model, endpoint paths (`/v2/...`, `/v3/...`), and auth scheme are identical
to the Polygon.io API, so older Polygon.io documentation and examples still apply.

## Quick Facts

| | |
|---|---|
| **Base URL** | `https://api.massive.com` (legacy `https://api.polygon.io` still works) |
| **Python package** | `massive` (`uv add massive` / `pip install -U massive`) — pinned as `massive>=1.0.0` in `backend/pyproject.toml` |
| **Min Python** | 3.9+ (FinAlly targets 3.12) |
| **Auth** | API key via `MASSIVE_API_KEY` env var, or `RESTClient(api_key=...)` |
| **Auth transport** | `Authorization: Bearer <API_KEY>` header (the client adds this automatically) |
| **Coverage** | All 19 US exchanges + dark pools, OTC, FINRA TRFs; also options, crypto, forex, indices; history back to 2003 |
| **Access methods** | REST (what we use), WebSocket streams, flat files |

## Rate Limits & Polling Strategy

| Tier | Request limit | FinAlly poll interval |
|------|---------------|-----------------------|
| Free / Basic | 5 requests/minute | **15 s** (default in code) |
| Paid (Starter and up) | Unlimited (be reasonable; stay well under 100 req/s) | 2–5 s |

The key efficiency lever: the **snapshot-all endpoint returns every requested ticker
in a single HTTP call**. Polling 10 (or 50) tickers costs **one** request per cycle,
so the free tier's 5 req/min budget is plenty at a 15 s cadence (≈4 req/min).

FinAlly never polls per-ticker in a loop. One snapshot call per cycle, full stop.

## Authentication / Client Initialization

```python
from massive import RESTClient

# Reads MASSIVE_API_KEY from the environment automatically
client = RESTClient()

# Or pass the key explicitly (this is what FinAlly does — see massive_client.py)
client = RESTClient(api_key="your_key_here")
```

The `RESTClient` is **synchronous** (blocking). FinAlly runs on FastAPI/asyncio, so
every call is dispatched with `asyncio.to_thread(...)` to avoid blocking the event
loop (see [How FinAlly Uses the API](#how-finally-uses-the-api)).

## Endpoints

### 1. Snapshot — Multiple Tickers (primary endpoint for live polling)

Returns the current full-day snapshot for many tickers in **one** call. This is the
endpoint FinAlly's poller hits every cycle.

**REST**

```
GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,GOOGL,MSFT
```

**Python client** (as used in `massive_client.py`)

```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient(api_key=API_KEY)

snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA"],
)

for snap in snapshots:
    print(f"{snap.ticker}: ${snap.last_trade.price}")
    print(f"  Day change:  {snap.todays_change_percent:.2f}%")
    print(f"  Day OHLC:    O={snap.day.open} H={snap.day.high} "
          f"L={snap.day.low} C={snap.day.close}")
    print(f"  Volume:      {snap.day.volume}")
    print(f"  Trade @ (ms):{snap.last_trade.timestamp}")
```

> **SDK note:** Older Polygon SDK builds also expose `list_snapshot_tickers(tickers)`
> and `list_snapshot_all_tickers()`. FinAlly standardizes on
> `get_snapshot_all(market_type=..., tickers=[...])` because that is the signature
> available in the pinned `massive>=1.0.0` package and it takes the explicit ticker
> list we want. If you upgrade the SDK and the method disappears, the
> `list_snapshot_tickers` variant is the drop-in replacement.

**Snapshot object — fields that matter to us**

| Attribute | Meaning | FinAlly use |
|-----------|---------|-------------|
| `snap.ticker` | Symbol | Cache key |
| `snap.last_trade.price` | Most recent trade price | **The price we cache & trade on** |
| `snap.last_trade.timestamp` | Trade time, **Unix milliseconds** | Converted to seconds before caching |
| `snap.last_trade.size` | Trade size | (unused) |
| `snap.day.open/high/low/close` | Today's OHLC | Detail view (future) |
| `snap.day.volume` | Today's volume | Detail view (future) |
| `snap.prev_day.close` | Previous session close | Day-change baseline (future) |
| `snap.todays_change` | Abs change vs prev close | Day-change display (future) |
| `snap.todays_change_percent` | % change vs prev close | Day-change display (future) |
| `snap.last_quote.bid_price/ask_price` | NBBO | Detail view (future) |

**Representative JSON (per ticker)**

```json
{
  "ticker": "AAPL",
  "todays_change": -4.54,
  "todays_change_percent": -3.50,
  "updated": 1675190399000000000,
  "day": {
    "o": 129.61, "h": 130.15, "l": 125.07, "c": 125.07,
    "v": 111237700, "vw": 127.35
  },
  "last_trade": {
    "p": 125.07, "s": 100, "x": 11, "t": 1675190399000
  },
  "last_quote": {
    "P": 125.08, "S": 10, "p": 125.06, "s": 5, "t": 1675190399500
  },
  "prevDay": {
    "o": 130.46, "h": 133.41, "l": 129.89, "c": 129.61, "v": 70790800
  }
}
```

The Python SDK maps these short keys onto readable attribute names
(`p` → `price`, `t` → `timestamp`, `o/h/l/c/v` → `open/high/low/close/volume`, etc.),
so application code reads `snap.last_trade.price`, not `snap["last_trade"]["p"]`.

### 2. Snapshot — Single Ticker

For the detail view when a user clicks one ticker.

**REST**

```
GET /v2/snapshot/locale/us/markets/stocks/tickers/{ticker}
```

**Python client**

```python
snap = client.get_snapshot_ticker(
    market_type=SnapshotMarketType.STOCKS,
    ticker="AAPL",
)
print(f"Price:  ${snap.last_trade.price}")
print(f"Bid/Ask:${snap.last_quote.bid_price} / {snap.last_quote.ask_price}")
print(f"Range:  ${snap.day.low} – ${snap.day.high}")
```

### 3. Previous Close (end-of-day OHLC)

Previous trading day's OHLCV for a ticker. Useful for **seed prices** (initializing
the cache before the first live snapshot lands) and as a day-change baseline.

**REST**

```
GET /v2/aggs/ticker/{ticker}/prev?adjusted=true
```

**Python client**

```python
prev = client.get_previous_close_agg(ticker="AAPL")
# Returns a list with a single aggregate bar
for agg in prev:
    print(f"Prev close: ${agg.close}")
    print(f"OHLC: O={agg.open} H={agg.high} L={agg.low} C={agg.close} V={agg.volume}")
```

> **SDK note:** Depending on SDK version this method is named
> `get_previous_close_agg(ticker=...)` (current) or `get_previous_close(ticker=...)`
> (older). Both hit `/v2/aggs/ticker/{ticker}/prev`.

**Response**

```json
{
  "ticker": "AAPL",
  "queryCount": 1,
  "resultsCount": 1,
  "adjusted": true,
  "results": [
    { "T": "AAPL", "o": 150.0, "h": 155.0, "l": 149.0, "c": 154.5, "v": 1000000, "t": 1672531200000 }
  ]
}
```

### 4. Aggregates / Bars (historical OHLCV)

Historical candles over a date range — for the main price chart's history (the live
sparkline is accumulated client-side from SSE, but a detail chart wants real history).

**REST**

```
GET /v2/aggs/ticker/{ticker}/range/{multiplier}/{timespan}/{from}/{to}?adjusted=true&sort=asc&limit=50000
```

`timespan` ∈ `minute | hour | day | week | month | quarter | year`.

**Python client** (`list_aggs` auto-paginates and yields bars)

```python
bars = []
for a in client.list_aggs(
    ticker="AAPL",
    multiplier=1,
    timespan="day",
    from_="2024-01-01",   # note the trailing underscore: from_ (from is reserved)
    to="2024-01-31",
    adjusted=True,
    sort="asc",
    limit=50000,
):
    bars.append(a)

for a in bars:
    print(f"t={a.timestamp} O={a.open} H={a.high} L={a.low} C={a.close} V={a.volume}")
```

### 5. Last Trade / Last Quote (single value)

Lightweight single-value reads. FinAlly does **not** use these in the hot path
(the multi-ticker snapshot is far more rate-limit-efficient), but they are handy
for diagnostics.

```python
trade = client.get_last_trade(ticker="AAPL")
print(f"Last trade: ${trade.price} x {trade.size}")

quote = client.get_last_quote(ticker="AAPL")
print(f"Bid {quote.bid_price} x {quote.bid_size} / Ask {quote.ask_price} x {quote.ask_size}")
```

## How FinAlly Uses the API

The poller is `MassiveDataSource` in `backend/app/market/massive_client.py`. Its loop:

1. Hold the active ticker set (seeded from the watchlist; mutated by
   `add_ticker` / `remove_ticker`).
2. Once per cycle, call `get_snapshot_all(market_type=STOCKS, tickers=...)` —
   **one** HTTP request for all tickers — inside `asyncio.to_thread` so the blocking
   SDK call doesn't stall the event loop.
3. For each snapshot, extract `last_trade.price` and `last_trade.timestamp`, convert
   the timestamp from **milliseconds → seconds**, and write to the shared `PriceCache`.
4. Sleep `poll_interval` (default 15 s), then repeat. An immediate first poll runs in
   `start()` so the cache has data before the first sleep.

Condensed from the shipped implementation:

```python
import asyncio
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

class MassiveDataSource(MarketDataSource):
    def __init__(self, api_key, price_cache, poll_interval=15.0):
        self._api_key = api_key
        self._cache = price_cache
        self._interval = poll_interval
        self._tickers = []
        self._task = None
        self._client = None

    async def start(self, tickers):
        self._client = RESTClient(api_key=self._api_key)
        self._tickers = list(tickers)
        await self._poll_once()                       # immediate first fill
        self._task = asyncio.create_task(self._poll_loop(), name="massive-poller")

    async def _poll_loop(self):
        while True:
            await asyncio.sleep(self._interval)
            await self._poll_once()

    async def _poll_once(self):
        if not self._tickers or not self._client:
            return
        try:
            snapshots = await asyncio.to_thread(self._fetch_snapshots)
            for snap in snapshots:
                try:
                    self._cache.update(
                        ticker=snap.ticker,
                        price=snap.last_trade.price,
                        timestamp=snap.last_trade.timestamp / 1000.0,  # ms -> s
                    )
                except (AttributeError, TypeError) as e:
                    logger.warning("Skipping %s: %s", getattr(snap, "ticker", "???"), e)
        except Exception as e:
            logger.error("Massive poll failed: %s", e)   # 401/429/network — retry next cycle

    def _fetch_snapshots(self):
        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS,
            tickers=self._tickers,
        )
```

See `MARKET_INTERFACE.md` for how `MassiveDataSource` and the simulator share one
abstract interface and a common `PriceCache`.

## Error Handling

The client raises on HTTP errors. The poller catches **all** exceptions per cycle and
logs them rather than crashing the background task — the next cycle simply retries.

| Status | Cause | FinAlly behavior |
|--------|-------|------------------|
| **401** | Invalid / missing API key | Log error; keep retrying (cache goes stale). Operator should fix `MASSIVE_API_KEY` |
| **403** | Plan lacks the endpoint/entitlement | Log error; retry |
| **429** | Rate limit exceeded | Log error; retry next cycle. Increase `poll_interval` if persistent (free tier = 5 req/min) |
| **5xx** | Massive server error | SDK retries internally (≈3×); then logged and retried next cycle |
| Network / timeout | Connectivity | Logged; retried next cycle |

Per-snapshot parsing is also guarded: a malformed snapshot (missing `last_trade`)
is skipped with a warning, so one bad ticker can't poison the whole cycle.

## Gotchas & Notes

- **Timestamps are Unix milliseconds.** Always divide by 1000 before storing as the
  Unix-seconds `timestamp` FinAlly uses elsewhere.
- **Market hours.** Outside regular hours, `last_trade.price` reflects the last
  traded price (which can include extended-hours trades). The `day.*` fields reset at
  the open; during pre-market they may still show the previous session.
- **Single call for many tickers.** Never loop per-ticker for live prices — it
  multiplies request count and will blow the free-tier limit.
- **Snapshot vs. trade price.** FinAlly trades and values the portfolio off
  `last_trade.price` (the most recent print), not `day.close`.
- **Adjusted data.** Aggregate/previous-close endpoints accept `adjusted=true`
  (split/dividend-adjusted) — the sensible default for charts.
- **Free-tier data delay.** The Basic (free) plan serves **end-of-day / 15-minute
  delayed** data; real-time prints require a paid real-time plan. The code path is
  identical either way — only the freshness differs.

## Sources

- [Polygon.io is Now Massive — announcement](https://massive.com/blog/polygon-is-now-massive)
- [Massive — Stocks REST API overview](https://massive.com/docs/rest/stocks/overview)
- [Massive — REST API Quickstart](https://massive.com/docs/rest/quickstart)
- [massive-com/client-python (official Python SDK)](https://github.com/massive-com/client-python)
- [client-python RESTClient Guide (DeepWiki)](https://deepwiki.com/massive-com/client-python/3-restclient-guide)
