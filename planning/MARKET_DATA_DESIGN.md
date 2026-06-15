# Market Data Backend — Detailed Design

Implementation-ready design for the FinAlly market data subsystem. It documents the
unified data-source interface, the in-memory price cache, the GBM simulator, the
Massive (Polygon.io) REST client, the SSE streaming endpoint, and how all of it plugs
into the FastAPI application lifecycle.

This document reflects the **shipped** implementation in `backend/app/market/`
(status: complete, 73 tests passing — see `MARKET_DATA_SUMMARY.md`). It supersedes the
earlier pre-review draft kept at `planning/archive/MARKET_DATA_DESIGN.md`; where the two
differ, this one matches the code (top-level `massive` imports, public
`GBMSimulator.get_tickers()`, typed SSE generator, no `DEFAULT_CORR`).

> **Wire contract.** The SSE shape below is the frozen seam in `planning/API_CONTRACT.md`
> (🔒). If you change `PriceUpdate.to_dict()`, change `API_CONTRACT.md` + `decisions.md`
> first — the frontend mirrors that file (AGENTS.md §2).

---

## Table of Contents

1. [Architecture at a Glance](#1-architecture-at-a-glance)
2. [File Structure](#2-file-structure)
3. [Data Model — `models.py`](#3-data-model--modelspy)
4. [Price Cache — `cache.py`](#4-price-cache--cachepy)
5. [Unified Interface — `interface.py`](#5-unified-interface--interfacepy)
6. [Seed Prices & Parameters — `seed_prices.py`](#6-seed-prices--parameters--seed_pricespy)
7. [GBM Simulator — `simulator.py`](#7-gbm-simulator--simulatorpy)
8. [Massive API Client — `massive_client.py`](#8-massive-api-client--massive_clientpy)
9. [Factory — `factory.py`](#9-factory--factorypy)
10. [SSE Streaming Endpoint — `stream.py`](#10-sse-streaming-endpoint--streampy)
11. [FastAPI Lifecycle Integration](#11-fastapi-lifecycle-integration)
12. [Watchlist Coordination & the Tracked-Set Rule](#12-watchlist-coordination--the-tracked-set-rule)
13. [Testing Strategy](#13-testing-strategy)
14. [Error Handling & Edge Cases](#14-error-handling--edge-cases)
15. [Configuration Summary](#15-configuration-summary)

---

## 1. Architecture at a Glance

One background producer writes prices into a shared cache; many consumers read from it.
The producer is chosen at startup and is the only part of the system that knows whether
data is simulated or real.

```
                      ┌──────────────────────────────┐
   MASSIVE_API_KEY?   │   create_market_data_source  │
   ───────────────────▶          (factory)           │
                      └───────────────┬──────────────┘
                  set & non-empty │    │ empty / unset
                                  ▼    ▼
                    MassiveDataSource   SimulatorDataSource
                    (REST poll ~15s)    (GBM step ~500ms)
                                  │    │
                                  ▼    ▼
                          ┌──────────────────┐   writers
                          │    PriceCache    │◀──────────
                          │ (thread-safe,    │
                          │  version counter)│──────────▶ readers
                          └──────────────────┘
                            │        │        │
                            ▼        ▼        ▼
                       SSE stream  Portfolio  Trade
                       /api/stream  valuation execution
```

Three properties make this work:

- **Strategy pattern** — both sources implement `MarketDataSource`; nothing downstream
  branches on the source type.
- **Cache as single source of truth** — producers `update()`, consumers `get()`. The
  two sides are decoupled in both timing and identity.
- **Push, not pull** — the source writes on *its* schedule (sim: 500 ms, Massive: 15 s);
  the SSE layer reads on *its own* 500 ms cadence and uses a version counter to skip
  redundant sends.

---

## 2. File Structure

Everything lives under `backend/app/market/`. Each module has a single responsibility;
`__init__.py` re-exports the public surface so the rest of the backend imports from
`app.market` and never reaches into submodules.

```
backend/app/market/
  __init__.py        # Public API: PriceUpdate, PriceCache, MarketDataSource,
                     #             create_market_data_source, create_stream_router
  models.py          # PriceUpdate (immutable dataclass)
  cache.py           # PriceCache (thread-safe in-memory store + version counter)
  interface.py       # MarketDataSource (ABC)
  seed_prices.py     # SEED_PRICES, TICKER_PARAMS, DEFAULT_PARAMS, correlation consts
  simulator.py       # GBMSimulator (math) + SimulatorDataSource (async wrapper)
  massive_client.py  # MassiveDataSource (Polygon.io REST poller)
  factory.py         # create_market_data_source()
  stream.py          # create_stream_router() — FastAPI SSE endpoint
```

```python
# backend/app/market/__init__.py
"""Market data subsystem for FinAlly.

Public API:
    PriceUpdate         - Immutable price snapshot dataclass
    PriceCache          - Thread-safe in-memory price store
    MarketDataSource    - Abstract interface for data providers
    create_market_data_source - Factory that selects simulator or Massive
    create_stream_router - FastAPI router factory for SSE endpoint
"""

from .cache import PriceCache
from .factory import create_market_data_source
from .interface import MarketDataSource
from .models import PriceUpdate
from .stream import create_stream_router

__all__ = [
    "PriceUpdate",
    "PriceCache",
    "MarketDataSource",
    "create_market_data_source",
    "create_stream_router",
]
```

Dependencies (`backend/pyproject.toml`): `numpy` (Cholesky + normal draws) and `massive`
(the Polygon.io client) are **core** dependencies — both are always installed, so imports
are top-level, not lazy. `rich` powers the standalone demo.

---

## 3. Data Model — `models.py`

`PriceUpdate` is the only type that leaves the market layer. SSE, portfolio valuation,
and trade execution all speak it.

```python
from __future__ import annotations

import time
from dataclasses import dataclass, field


@dataclass(frozen=True, slots=True)
class PriceUpdate:
    """Immutable snapshot of a single ticker's price at a point in time."""

    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)  # Unix seconds

    @property
    def change(self) -> float:
        """Absolute price change from previous update."""
        return round(self.price - self.previous_price, 4)

    @property
    def change_percent(self) -> float:
        """Percentage change from previous update."""
        if self.previous_price == 0:
            return 0.0
        return round((self.price - self.previous_price) / self.previous_price * 100, 4)

    @property
    def direction(self) -> str:
        """'up', 'down', or 'flat'."""
        if self.price > self.previous_price:
            return "up"
        elif self.price < self.previous_price:
            return "down"
        return "flat"

    def to_dict(self) -> dict:
        """Serialize for JSON / SSE transmission."""
        return {
            "ticker": self.ticker,
            "price": self.price,
            "previous_price": self.previous_price,
            "timestamp": self.timestamp,
            "change": self.change,
            "change_percent": self.change_percent,
            "direction": self.direction,
        }
```

**Why these choices**

- `frozen=True` — value object; safe to share across async tasks without copying.
- `slots=True` — small memory win; we create many per second.
- **Computed properties** (`change`, `change_percent`, `direction`) — derived from
  `price`/`previous_price`, so they can never go stale or disagree with each other.
- `to_dict()` — the single serialization point. The keys here *are* the per-ticker wire
  schema in `API_CONTRACT.md`. Note `timestamp` is **Unix epoch seconds (float)** for
  live prices (persisted timestamps elsewhere are ISO-8601 — two formats on purpose).

Example:

```python
>>> u = PriceUpdate("AAPL", price=190.50, previous_price=190.10, timestamp=1718450000.12)
>>> u.change, u.change_percent, u.direction
(0.4, 0.2104, 'up')
>>> u.to_dict()
{'ticker': 'AAPL', 'price': 190.5, 'previous_price': 190.1, 'timestamp': 1718450000.12,
 'change': 0.4, 'change_percent': 0.2104, 'direction': 'up'}
```

---

## 4. Price Cache — `cache.py`

The hub. Sources write; SSE, valuation, and trade execution read. It is **thread-safe**
because the Massive client's synchronous fetch runs in `asyncio.to_thread()` (a real OS
thread) while SSE reads happen on the event loop.

```python
from __future__ import annotations

import time
from threading import Lock

from .models import PriceUpdate


class PriceCache:
    """Thread-safe in-memory cache of the latest price for each ticker.

    Writers: SimulatorDataSource or MassiveDataSource (one at a time).
    Readers: SSE streaming endpoint, portfolio valuation, trade execution.
    """

    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = Lock()
        self._version: int = 0  # Monotonically increasing; bumped on every update

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
        """Record a new price for a ticker. Returns the created PriceUpdate.

        Automatically computes direction/change from the previous price. On the
        first update for a ticker, previous_price == price (direction='flat').
        """
        with self._lock:
            ts = timestamp or time.time()
            prev = self._prices.get(ticker)
            previous_price = prev.price if prev else price

            update = PriceUpdate(
                ticker=ticker,
                price=round(price, 2),
                previous_price=round(previous_price, 2),
                timestamp=ts,
            )
            self._prices[ticker] = update
            self._version += 1
            return update

    def get(self, ticker: str) -> PriceUpdate | None:
        with self._lock:
            return self._prices.get(ticker)

    def get_all(self) -> dict[str, PriceUpdate]:
        """Snapshot of all current prices (shallow copy)."""
        with self._lock:
            return dict(self._prices)

    def get_price(self, ticker: str) -> float | None:
        update = self.get(ticker)
        return update.price if update else None

    def remove(self, ticker: str) -> None:
        with self._lock:
            self._prices.pop(ticker, None)

    @property
    def version(self) -> int:
        """Monotonic counter, bumped on every update — for SSE change detection."""
        return self._version

    def __len__(self) -> int:
        with self._lock:
            return len(self._prices)

    def __contains__(self, ticker: str) -> bool:
        with self._lock:
            return ticker in self._prices
```

**Why a version counter?** The SSE loop wakes every ~500 ms, but Massive only refreshes
every 15 s. Without the counter, SSE would re-serialize and re-send identical prices
~30× between Massive polls. With it, the loop sends *only when something changed*:

```python
last_version = -1
while True:
    if price_cache.version != last_version:
        last_version = price_cache.version
        yield format_sse(price_cache.get_all())
    await asyncio.sleep(0.5)
```

**Why `threading.Lock` and not `asyncio.Lock`?** The Massive fetch runs in a worker
thread; an `asyncio.Lock` only guards coroutines on the loop and would not protect
against a concurrent write from that thread. `threading.Lock` is correct from both
contexts and the critical section (a dict get/set) is tiny, so contention is negligible.

---

## 5. Unified Interface — `interface.py`

Both data sources implement this ABC. It is the seam that makes simulator vs. real data
a one-line factory decision.

```python
from __future__ import annotations

from abc import ABC, abstractmethod


class MarketDataSource(ABC):
    """Contract for market data providers.

    Implementations push price updates into a shared PriceCache on their own
    schedule. Downstream code never calls the data source directly for prices —
    it reads from the cache.

    Lifecycle:
        source = create_market_data_source(cache)
        await source.start(["AAPL", "GOOGL", ...])
        await source.add_ticker("TSLA")
        await source.remove_ticker("GOOGL")
        await source.stop()
    """

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing updates for the given tickers (starts a background task).
        Must be called exactly once."""

    @abstractmethod
    async def stop(self) -> None:
        """Stop the background task and release resources. Safe to call repeatedly."""

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the active set. No-op if already present."""

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker from the active set and from the cache. No-op if absent."""

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Return the currently tracked tickers."""
```

**Push, not pull.** The source writes into the cache instead of returning prices on
demand. This decouples timing entirely: the SSE layer never needs to know the source's
update interval, and swapping simulator ↔ Massive changes nothing downstream.

---

## 6. Seed Prices & Parameters — `seed_prices.py`

Constants only — no logic, no third-party imports. Shared by the simulator (initial
prices, GBM params, correlation structure).

```python
"""Seed prices and per-ticker parameters for the market simulator."""

# Realistic starting prices for the default watchlist
SEED_PRICES: dict[str, float] = {
    "AAPL": 190.00, "GOOGL": 175.00, "MSFT": 420.00, "AMZN": 185.00, "TSLA": 250.00,
    "NVDA": 800.00, "META": 500.00, "JPM": 195.00, "V": 280.00, "NFLX": 600.00,
}

# Per-ticker GBM parameters. sigma = annualized volatility, mu = annualized drift.
TICKER_PARAMS: dict[str, dict[str, float]] = {
    "AAPL":  {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT":  {"sigma": 0.20, "mu": 0.05},
    "AMZN":  {"sigma": 0.28, "mu": 0.05},
    "TSLA":  {"sigma": 0.50, "mu": 0.03},   # High volatility
    "NVDA":  {"sigma": 0.40, "mu": 0.08},   # High volatility, strong drift
    "META":  {"sigma": 0.30, "mu": 0.05},
    "JPM":   {"sigma": 0.18, "mu": 0.04},   # Low volatility (bank)
    "V":     {"sigma": 0.17, "mu": 0.04},   # Low volatility (payments)
    "NFLX":  {"sigma": 0.35, "mu": 0.05},
}

# Fallback for tickers added at runtime that aren't in TICKER_PARAMS
DEFAULT_PARAMS: dict[str, float] = {"sigma": 0.25, "mu": 0.05}

# Correlation groups for the simulator's Cholesky decomposition
CORRELATION_GROUPS: dict[str, set[str]] = {
    "tech": {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}

# Correlation coefficients
INTRA_TECH_CORR = 0.6      # Tech stocks move together
INTRA_FINANCE_CORR = 0.5   # Finance stocks move together
CROSS_GROUP_CORR = 0.3     # Between sectors / unknown tickers
TSLA_CORR = 0.3            # TSLA does its own thing
```

A runtime-added ticker with no seed gets a random starting price in `[50, 300]` and
`DEFAULT_PARAMS` (see the simulator). `CROSS_GROUP_CORR` doubles as the "unknown ticker"
correlation, so there is no separate `DEFAULT_CORR` constant.

---

## 7. GBM Simulator — `simulator.py`

Two classes: `GBMSimulator` (pure, synchronous math) and `SimulatorDataSource` (the
async `MarketDataSource` wrapper that drives it and writes to the cache).

### 7.1 The math: geometric Brownian motion

```
S(t+dt) = S(t) · exp( (mu − ½σ²)·dt  +  σ·√dt·Z )
```

`dt` is one tick expressed as a fraction of a trading year:

```
TRADING_SECONDS_PER_YEAR = 252 · 6.5 · 3600 = 5,896,800
DEFAULT_DT = 0.5 / 5,896,800 ≈ 8.48e-8     # a 500 ms tick
```

The tiny `dt` yields sub-cent moves per tick that accumulate naturally. `exp(...)` keeps
prices strictly positive, so they can never go negative.

**Correlated moves.** Each tick draws `n` independent standard normals `Z`, then
multiplies by the lower-triangular Cholesky factor `L` of the correlation matrix `C`
(`C = L·Lᵀ`) to produce correlated draws — tech names rise and fall together, finance
names move as a pair, TSLA stays loosely coupled.

```python
from __future__ import annotations

import asyncio
import logging
import math
import random

import numpy as np

from .cache import PriceCache
from .interface import MarketDataSource
from .seed_prices import (
    CORRELATION_GROUPS, CROSS_GROUP_CORR, DEFAULT_PARAMS,
    INTRA_FINANCE_CORR, INTRA_TECH_CORR, SEED_PRICES, TICKER_PARAMS, TSLA_CORR,
)

logger = logging.getLogger(__name__)


class GBMSimulator:
    """Geometric Brownian Motion simulator for correlated stock prices."""

    TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # 5,896,800
    DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR   # ~8.48e-8

    def __init__(self, tickers: list[str], dt: float = DEFAULT_DT,
                 event_probability: float = 0.001) -> None:
        self._dt = dt
        self._event_prob = event_probability
        self._tickers: list[str] = []
        self._prices: dict[str, float] = {}
        self._params: dict[str, dict[str, float]] = {}
        self._cholesky: np.ndarray | None = None

        for ticker in tickers:
            self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    def step(self) -> dict[str, float]:
        """Advance all tickers one tick. Returns {ticker: new_price}. Hot path."""
        n = len(self._tickers)
        if n == 0:
            return {}

        z_independent = np.random.standard_normal(n)
        if self._cholesky is not None:
            z_correlated = self._cholesky @ z_independent
        else:
            z_correlated = z_independent

        result: dict[str, float] = {}
        for i, ticker in enumerate(self._tickers):
            p = self._params[ticker]
            mu, sigma = p["mu"], p["sigma"]

            drift = (mu - 0.5 * sigma**2) * self._dt
            diffusion = sigma * math.sqrt(self._dt) * z_correlated[i]
            self._prices[ticker] *= math.exp(drift + diffusion)

            # Random "drama" event: ~0.1% per tick per ticker → ~1 every 50s at 10×2/s
            if random.random() < self._event_prob:
                shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
                self._prices[ticker] *= 1 + shock
                logger.debug("Random event on %s: %.1f%%", ticker, shock * 100)

            result[ticker] = round(self._prices[ticker], 2)
        return result

    def add_ticker(self, ticker: str) -> None:
        if ticker in self._prices:
            return
        self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    def remove_ticker(self, ticker: str) -> None:
        if ticker not in self._prices:
            return
        self._tickers.remove(ticker)
        del self._prices[ticker]
        del self._params[ticker]
        self._rebuild_cholesky()

    def get_price(self, ticker: str) -> float | None:
        return self._prices.get(ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    # --- internals ---

    def _add_ticker_internal(self, ticker: str) -> None:
        if ticker in self._prices:
            return
        self._tickers.append(ticker)
        self._prices[ticker] = SEED_PRICES.get(ticker, random.uniform(50.0, 300.0))
        self._params[ticker] = TICKER_PARAMS.get(ticker, dict(DEFAULT_PARAMS))

    def _rebuild_cholesky(self) -> None:
        """Rebuild Cholesky factor of the correlation matrix. O(n^2), n < 50."""
        n = len(self._tickers)
        if n <= 1:
            self._cholesky = None
            return
        corr = np.eye(n)
        for i in range(n):
            for j in range(i + 1, n):
                rho = self._pairwise_correlation(self._tickers[i], self._tickers[j])
                corr[i, j] = corr[j, i] = rho
        self._cholesky = np.linalg.cholesky(corr)

    @staticmethod
    def _pairwise_correlation(t1: str, t2: str) -> float:
        tech, finance = CORRELATION_GROUPS["tech"], CORRELATION_GROUPS["finance"]
        if t1 == "TSLA" or t2 == "TSLA":
            return TSLA_CORR
        if t1 in tech and t2 in tech:
            return INTRA_TECH_CORR
        if t1 in finance and t2 in finance:
            return INTRA_FINANCE_CORR
        return CROSS_GROUP_CORR
```

> **Cholesky validity.** The factor exists only if the correlation matrix is positive
> definite. With the block structure here (one tech cluster at 0.6, one finance pair at
> 0.5, everything else 0.3, unit diagonal) the matrix stays PD for the default and
> realistically-sized watchlists. If a future correlation scheme pushes it non-PD,
> `np.linalg.cholesky` raises `LinAlgError` — clamp off-diagonals or shrink toward the
> identity (`αC + (1−α)I`) rather than loosening the diagonal.

### 7.2 The async wrapper

```python
class SimulatorDataSource(MarketDataSource):
    """MarketDataSource backed by the GBM simulator.

    Runs a background asyncio task that calls GBMSimulator.step() every
    `update_interval` seconds and writes results to the PriceCache.
    """

    def __init__(self, price_cache: PriceCache, update_interval: float = 0.5,
                 event_probability: float = 0.001) -> None:
        self._cache = price_cache
        self._interval = update_interval
        self._event_prob = event_probability
        self._sim: GBMSimulator | None = None
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers=tickers, event_probability=self._event_prob)
        # Seed the cache up-front so SSE has data on its very first tick.
        for ticker in tickers:
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")
        logger.info("Simulator started with %d tickers", len(tickers))

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        logger.info("Simulator stopped")

    async def add_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.add_ticker(ticker)
            price = self._sim.get_price(ticker)   # seed immediately, no blank cell
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
            logger.info("Simulator: added ticker %s", ticker)

    async def remove_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.remove_ticker(ticker)
        self._cache.remove(ticker)
        logger.info("Simulator: removed ticker %s", ticker)

    def get_tickers(self) -> list[str]:
        return self._sim.get_tickers() if self._sim else []

    async def _run_loop(self) -> None:
        while True:
            try:
                if self._sim:
                    for ticker, price in self._sim.step().items():
                        self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed")  # one bad tick ≠ dead feed
            await asyncio.sleep(self._interval)
```

**Key behaviors**

- **Immediate seeding** — `start()` and `add_ticker()` write seed prices *before* the
  loop runs, so there is never a blank watchlist cell.
- **Graceful shutdown** — `stop()` cancels and awaits the task, swallowing
  `CancelledError`; idempotent (safe to call twice).
- **Resilient loop** — exceptions are caught per tick, logged, and the feed continues.

---

## 8. Massive API Client — `massive_client.py`

Polls the Massive (Polygon.io) snapshot endpoint for the union of tracked tickers in one
call, then writes results to the same cache. The Massive `RESTClient` is synchronous, so
the network call runs in `asyncio.to_thread()` to keep the event loop free.

```python
from __future__ import annotations

import asyncio
import logging

from massive import RESTClient
from massive.rest.models import SnapshotMarketType

from .cache import PriceCache
from .interface import MarketDataSource

logger = logging.getLogger(__name__)


class MassiveDataSource(MarketDataSource):
    """MarketDataSource backed by the Massive (Polygon.io) REST API.

    Polls the stocks snapshot for all watched tickers in one call, then writes
    results to the PriceCache.

    Rate limits:
      - Free tier: 5 req/min → poll every 15s (default)
      - Paid tiers: higher limits → poll every 2-5s
    """

    def __init__(self, api_key: str, price_cache: PriceCache,
                 poll_interval: float = 15.0) -> None:
        self._api_key = api_key
        self._cache = price_cache
        self._interval = poll_interval
        self._tickers: list[str] = []
        self._task: asyncio.Task | None = None
        self._client: RESTClient | None = None

    async def start(self, tickers: list[str]) -> None:
        self._client = RESTClient(api_key=self._api_key)
        self._tickers = list(tickers)
        await self._poll_once()  # immediate first poll → cache has data right away
        self._task = asyncio.create_task(self._poll_loop(), name="massive-poller")
        logger.info("Massive poller started: %d tickers, %.1fs interval",
                    len(tickers), self._interval)

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        self._client = None
        logger.info("Massive poller stopped")

    async def add_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        if ticker not in self._tickers:
            self._tickers.append(ticker)
            logger.info("Massive: added ticker %s (appears on next poll)", ticker)

    async def remove_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        self._tickers = [t for t in self._tickers if t != ticker]
        self._cache.remove(ticker)
        logger.info("Massive: removed ticker %s", ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    # --- internal ---

    async def _poll_loop(self) -> None:
        while True:
            await asyncio.sleep(self._interval)
            await self._poll_once()

    async def _poll_once(self) -> None:
        if not self._tickers or not self._client:
            return
        try:
            snapshots = await asyncio.to_thread(self._fetch_snapshots)
            processed = 0
            for snap in snapshots:
                try:
                    price = snap.last_trade.price
                    timestamp = snap.last_trade.timestamp / 1000.0  # ms → seconds
                    self._cache.update(ticker=snap.ticker, price=price, timestamp=timestamp)
                    processed += 1
                except (AttributeError, TypeError) as e:
                    logger.warning("Skipping snapshot for %s: %s",
                                   getattr(snap, "ticker", "???"), e)
            logger.debug("Massive poll: updated %d/%d tickers", processed, len(self._tickers))
        except Exception as e:
            logger.error("Massive poll failed: %s", e)
            # Don't re-raise — retry next interval. Common: 401, 429, network errors.

    def _fetch_snapshots(self) -> list:
        """Synchronous Massive REST call. Runs in a worker thread."""
        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS,
            tickers=self._tickers,
        )
```

**Notes**

- **Same `PriceUpdate` shape.** The poller feeds the cache through the identical
  `update()` path; the SSE wire format is byte-for-byte the same whether prices are real
  or simulated.
- **Timestamps.** Massive returns Unix **milliseconds**; we divide by 1000 to match the
  cache's seconds convention.
- **Defensive parsing.** A malformed snapshot (missing `last_trade`) is skipped per
  ticker with a warning — one bad row never drops the rest of the batch.
- **`add_ticker` lag.** A newly added ticker has no price until the next poll (up to
  `poll_interval` later). Unlike the simulator there's nothing to seed; downstream
  renders "—" until the first tick (`API_CONTRACT.md`: `price` is `null` until then).

### Failure behavior

| Failure | Behavior |
|---|---|
| 401 Unauthorized (bad key) | Logged as error; loop keeps running. Fix `.env`, restart. |
| 429 Rate limited | Logged; next poll retries after `poll_interval`. Raise the interval if persistent. |
| Network timeout | Logged; auto-retry next cycle. |
| Malformed snapshot | That ticker skipped with a warning; others still processed. |
| Whole poll throws | Cache keeps last-known prices; SSE streams stale-but-present data. |

---

## 9. Factory — `factory.py`

One environment variable selects the source. `MASSIVE_API_KEY` set and non-empty →
real data; otherwise the simulator. Both `massive` and `numpy` are core deps, so imports
are top-level (no lazy-import dance).

```python
from __future__ import annotations

import logging
import os

from .cache import PriceCache
from .interface import MarketDataSource
from .massive_client import MassiveDataSource
from .simulator import SimulatorDataSource

logger = logging.getLogger(__name__)


def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    """Select the data source from the environment.

    - MASSIVE_API_KEY set and non-empty → MassiveDataSource (real data)
    - otherwise → SimulatorDataSource (GBM simulation)

    Returns an *unstarted* source; the caller must await source.start(tickers).
    """
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        logger.info("Market data source: Massive API (real data)")
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    logger.info("Market data source: GBM Simulator")
    return SimulatorDataSource(price_cache=price_cache)
```

`.strip()` means a key that is whitespace-only counts as unset — a common `.env`
copy-paste mistake falls back to the simulator instead of failing with a 401 storm.

---

## 10. SSE Streaming Endpoint — `stream.py`

A FastAPI route that holds a long-lived `text/event-stream` connection and pushes the
full price map whenever the cache version changes.

```python
from __future__ import annotations

import asyncio
import json
import logging
from collections.abc import AsyncGenerator

from fastapi import APIRouter, Request
from fastapi.responses import StreamingResponse

from .cache import PriceCache

logger = logging.getLogger(__name__)

router = APIRouter(prefix="/api/stream", tags=["streaming"])


def create_stream_router(price_cache: PriceCache) -> APIRouter:
    """Create the SSE router with a reference to the price cache (no globals)."""

    @router.get("/prices")
    async def stream_prices(request: Request) -> StreamingResponse:
        return StreamingResponse(
            _generate_events(price_cache, request),
            media_type="text/event-stream",
            headers={
                "Cache-Control": "no-cache",
                "Connection": "keep-alive",
                "X-Accel-Buffering": "no",  # disable nginx buffering if proxied
            },
        )

    return router


async def _generate_events(
    price_cache: PriceCache,
    request: Request,
    interval: float = 0.5,
) -> AsyncGenerator[str, None]:
    """Yield SSE-formatted price events; stop on client disconnect."""
    yield "retry: 1000\n\n"  # browser EventSource reconnects after 1s if dropped

    last_version = -1
    client_ip = request.client.host if request.client else "unknown"
    logger.info("SSE client connected: %s", client_ip)

    try:
        while True:
            if await request.is_disconnected():
                logger.info("SSE client disconnected: %s", client_ip)
                break

            current_version = price_cache.version
            if current_version != last_version:
                last_version = current_version
                prices = price_cache.get_all()
                if prices:
                    data = {t: u.to_dict() for t, u in prices.items()}
                    yield f"data: {json.dumps(data)}\n\n"

            await asyncio.sleep(interval)
    except asyncio.CancelledError:
        logger.info("SSE stream cancelled for: %s", client_ip)
```

**Wire format** (one event, abbreviated — matches `API_CONTRACT.md` 🔒):

```
retry: 1000

data: {"AAPL":{"ticker":"AAPL","price":190.50,"previous_price":190.42,"timestamp":1707580800.5,"change":0.08,"change_percent":0.042,"direction":"up"},"GOOGL":{...}}
```

**Client** (native `EventSource`, auto-reconnects):

```javascript
const es = new EventSource('/api/stream/prices');
es.onmessage = (e) => {
  const prices = JSON.parse(e.data);   // { AAPL: {ticker, price, ...}, ... }
  for (const [ticker, u] of Object.entries(prices)) applyTick(ticker, u);
};
```

**Design points**

- **Poll-and-push with a version gate.** Even though the loop wakes every 500 ms, it
  emits only when `version` advanced — quiet between Massive polls, lively under the
  simulator. The frontend gets evenly-spaced, change-only updates, ideal for sparklines.
- **Disconnect detection.** `request.is_disconnected()` ends the generator when the tab
  closes, so dropped clients don't leak server-side loops.
- **Object-keyed payload, not an array.** One JSON object keyed by ticker per event lets
  the client upsert without scanning a list.

---

## 11. FastAPI Lifecycle Integration

The cache and source are created at app startup and torn down at shutdown via the
`lifespan` context manager. Backend Engineer owns `main.py`; this is the integration
the market layer expects.

```python
from contextlib import asynccontextmanager

from fastapi import FastAPI

from app.market import PriceCache, create_market_data_source, create_stream_router
from app.market.interface import MarketDataSource


@asynccontextmanager
async def lifespan(app: FastAPI):
    # --- startup ---
    price_cache = PriceCache()
    app.state.price_cache = price_cache

    source = create_market_data_source(price_cache)   # reads MASSIVE_API_KEY
    app.state.market_source = source

    # Track watchlist ∪ open positions (see §12), not just the watchlist.
    initial_tickers = await load_tracked_tickers()     # from SQLite
    await source.start(initial_tickers)

    app.include_router(create_stream_router(price_cache))

    yield

    # --- shutdown ---
    await source.stop()


app = FastAPI(title="FinAlly", lifespan=lifespan)


# Dependency providers for other routers
def get_price_cache(request: Request) -> PriceCache:
    return request.app.state.price_cache


def get_market_source(request: Request) -> MarketDataSource:
    return request.app.state.market_source
```

Consumers reach the cache/source through DI:

```python
@router.post("/portfolio/trade")
async def execute_trade(trade: TradeRequest,
                        cache: PriceCache = Depends(get_price_cache)):
    price = cache.get_price(trade.ticker)
    if price is None:
        raise HTTPException(400, f"Price not yet available for {trade.ticker}")
    # ... fill at `price`, persist atomically ...
```

> The SSE router is created with `create_stream_router(price_cache)` and captures the
> cache by closure, so it needs no `Depends`. REST routers that need live prices use the
> `get_price_cache` / `get_market_source` providers above.

---

## 12. Watchlist Coordination & the Tracked-Set Rule

The source must track **watchlist ∪ open positions**, not just the watchlist
(`API_CONTRACT.md` tracked-set rule). Otherwise a stock you hold but removed from the
watchlist stops getting prices and your portfolio value silently freezes.

**Add** (`POST /api/watchlist`):

```
insert into watchlist (SQLite)
await source.add_ticker(ticker)
    simulator: add to GBM, rebuild Cholesky, seed cache immediately
    massive:   append to poll set, price arrives on next poll
return refreshed watchlist
```

**Remove** (`DELETE /api/watchlist/{ticker}`) — only stop tracking if not held:

```python
@router.delete("/watchlist/{ticker}")
async def remove_from_watchlist(
    ticker: str,
    source: MarketDataSource = Depends(get_market_source),
):
    await db.delete_watchlist_entry(ticker)

    position = await db.get_position(ticker)
    if position is None or position.quantity == 0:
        await source.remove_ticker(ticker)   # safe to drop from cache
    # else: keep streaming so portfolio valuation stays live

    return await build_watchlist_response()
```

The mirror case: a **buy** of a ticker that isn't tracked (e.g. the LLM buys something
off-watchlist) must `await source.add_ticker(ticker)` so its price keeps flowing for
valuation.

---

## 13. Testing Strategy

73 tests across `backend/tests/market/`, all passing (84% coverage; Massive lower by
design — its network methods are mocked). Run:

```bash
cd backend
uv run --extra dev pytest -v                 # all
uv run --extra dev pytest --cov=app          # with coverage
uv run --extra dev ruff check app/ tests/    # lint
```

| Module | Focus |
|---|---|
| `test_models.py` | `change` / `change_percent` / `direction`, zero-prev guard, `to_dict()` keys |
| `test_cache.py` | update/get/remove, first-update-flat, version increments, direction transitions |
| `test_simulator.py` | positivity, seed prices, add/remove, Cholesky rebuild, empty step, drift |
| `test_simulator_source.py` | cache seeded on start, prices move, clean (double) stop, add/remove |
| `test_factory.py` | env-var selection (set/empty/whitespace) → correct class |
| `test_massive.py` | poll updates cache, malformed snapshot skipped, API error doesn't crash |

**GBM properties** (`test_simulator.py`):

```python
def test_prices_are_positive():
    sim = GBMSimulator(tickers=["AAPL"])
    for _ in range(10_000):
        assert sim.step()["AAPL"] > 0          # exp() ⇒ never negative

def test_cholesky_rebuilds_on_add():
    sim = GBMSimulator(tickers=["AAPL"])
    assert sim._cholesky is None               # 1 ticker → no matrix
    sim.add_ticker("GOOGL")
    assert sim._cholesky is not None           # 2 tickers → matrix exists
```

**Massive without the network** — patch `_fetch_snapshots`, call `_poll_once()` directly
with a long `poll_interval` so the loop never auto-fires:

```python
def _make_snapshot(ticker, price, ts_ms):
    snap = MagicMock()
    snap.ticker = ticker
    snap.last_trade.price = price
    snap.last_trade.timestamp = ts_ms
    return snap

@pytest.mark.asyncio
async def test_poll_updates_cache():
    cache = PriceCache()
    source = MassiveDataSource("test-key", cache, poll_interval=60.0)
    snaps = [_make_snapshot("AAPL", 190.50, 1707580800000)]
    with patch.object(source, "_fetch_snapshots", return_value=snaps):
        await source._poll_once()
    assert cache.get_price("AAPL") == 190.50

@pytest.mark.asyncio
async def test_api_error_does_not_crash():
    cache = PriceCache()
    source = MassiveDataSource("test-key", cache, poll_interval=60.0)
    source._tickers = ["AAPL"]
    with patch.object(source, "_fetch_snapshots", side_effect=Exception("network")):
        await source._poll_once()              # must not raise
    assert cache.get_price("AAPL") is None
```

**Contract test (recommended at the seam).** Assert `set(PriceUpdate.to_dict().keys())`
equals the field list documented in `API_CONTRACT.md`, so a serialization change turns
drift into a red test instead of a frontend surprise (AGENTS.md §2).

---

## 14. Error Handling & Edge Cases

- **Empty watchlist at startup.** `start([])` is fine: the simulator produces nothing,
  the poller skips its call, SSE sends nothing until a ticker is added.
- **Trade before first price.** A freshly added ticker may have no cached price (Massive
  hasn't polled yet). Trade execution returns **400** with a clear "price not yet
  available" message. The simulator avoids this by seeding on `add_ticker`.
- **Invalid Massive key.** First poll 401s; logged; loop keeps running. SSE stays
  "connected" but carries no data until the key is fixed and the app restarted.
- **Thread safety under load.** `threading.Lock` mutex; critical section is a dict
  get/set. At 10 tickers × 2 ticks/s contention is negligible. A read-write lock would
  only matter at hundreds of tickers with many SSE readers — out of scope here.
- **Numerical stability.** Prices are `round(_, 2)` on output; `exp(drift + diffusion)`
  is stable and strictly positive; tiny `dt` never overflows.
- **Cholesky / PD.** Covered in §7.1 — current correlation block is positive definite;
  shrink toward identity if a future scheme breaks it.

---

## 15. Configuration Summary

| Parameter | Location | Default | Meaning |
|---|---|---|---|
| `MASSIVE_API_KEY` | env var | `""` | Set → Massive; empty/whitespace → simulator |
| `update_interval` | `SimulatorDataSource` | `0.5 s` | Simulator tick cadence |
| `event_probability` | `GBMSimulator` / source | `0.001` | Per-tick per-ticker shock chance |
| `dt` | `GBMSimulator` | `~8.48e-8` | GBM step (fraction of a trading year) |
| `poll_interval` | `MassiveDataSource` | `15.0 s` | Massive REST poll cadence (free tier) |
| SSE push interval | `_generate_events` | `0.5 s` | Cache poll cadence for SSE |
| SSE retry | `_generate_events` | `1000 ms` | EventSource reconnect hint |

### Quick start for downstream code

```python
from app.market import PriceCache, create_market_data_source

cache = PriceCache()
source = create_market_data_source(cache)          # reads MASSIVE_API_KEY
await source.start(["AAPL", "GOOGL", "MSFT"])      # tracked = watchlist ∪ positions

cache.get("AAPL")        # PriceUpdate | None
cache.get_price("AAPL")  # float | None
cache.get_all()          # dict[str, PriceUpdate]

await source.add_ticker("TSLA")
await source.remove_ticker("GOOGL")
await source.stop()
```

### Demo

```bash
cd backend
uv run market_data_demo.py   # live Rich dashboard: sparklines, direction arrows, event log
```
