# Market Data Interface Design

The unified Python interface FinAlly uses to retrieve stock prices. **One abstract
interface, two interchangeable implementations:** the Massive API client when
`MASSIVE_API_KEY` is set, otherwise the built-in GBM simulator. Everything downstream
— SSE streaming, portfolio valuation, trade execution — is source-agnostic and never
knows which one is running.

> **Status:** Implemented and shipped in `backend/app/market/`. This document
> describes the design as built; file/line references point at the real code.
> See `MASSIVE_API.md` (the real-data client) and `MARKET_SIMULATOR.md` (the simulator).

## Design Goals

1. **Source-agnostic downstream.** No code outside `app/market/` should branch on
   "real vs. simulated". It reads prices from one place: the `PriceCache`.
2. **Push, don't pull.** Data sources run their own background task and *push* updates
   into the cache on their own cadence (simulator: 500 ms; Massive: 15 s). Readers
   never call the source directly.
3. **Selection by environment, decided once at startup.** The factory inspects
   `MASSIVE_API_KEY` and returns the right implementation. No runtime switching.
4. **Future multi-user ready.** The cache + interface layering means adding per-user
   watchlists later doesn't touch the data layer.

## Architecture

```
                       ┌──────────────────────────────────────┐
                       │            create_market_data_source  │
   MASSIVE_API_KEY set │  factory.py — picks ONE at startup    │
   ────────────────────┤                                        │
                       └───────────────┬───────────────────────┘
                                       │
            ┌──────────────────────────┴───────────────────────┐
            ▼                                                   ▼
   ┌───────────────────┐                            ┌────────────────────────┐
   │ MassiveDataSource │  poll 15s (REST snapshot)  │  SimulatorDataSource   │  step 500ms (GBM)
   │ massive_client.py │                            │  simulator.py          │
   └─────────┬─────────┘                            └───────────┬────────────┘
             │  writes PriceUpdate                              │  writes PriceUpdate
             └───────────────────────┬──────────────────────────┘
                                     ▼
                          ┌────────────────────┐
                          │     PriceCache      │  thread-safe, in-memory, versioned
                          │     cache.py        │
                          └─────────┬──────────┘
                       reads        │        reads
            ┌──────────────────────┼───────────────────────┐
            ▼                      ▼                        ▼
   SSE stream (stream.py)   Portfolio valuation       Trade execution
   GET /api/stream/prices   (current prices)          (fill at last price)
```

## Module Layout (as built)

```
backend/app/market/
├── __init__.py          # Public exports
├── models.py            # PriceUpdate dataclass
├── interface.py         # MarketDataSource ABC
├── cache.py             # PriceCache (thread-safe, versioned)
├── factory.py           # create_market_data_source()
├── massive_client.py    # MassiveDataSource (real data)
├── simulator.py         # GBMSimulator + SimulatorDataSource
├── seed_prices.py       # SEED_PRICES, TICKER_PARAMS, correlation constants
└── stream.py            # SSE router (create_stream_router)
```

Public imports:

```python
from app.market import (
    PriceUpdate, PriceCache,
    MarketDataSource, create_market_data_source,
    create_stream_router,
)
```

## Core Data Model — `PriceUpdate`

The only type that crosses the market-data boundary. Frozen, slotted, immutable.
Derived fields are computed properties, not stored, so they can't drift.

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)   # Unix seconds

    @property
    def change(self) -> float:           # price - previous_price (rounded 4dp)
    @property
    def change_percent(self) -> float:   # % change (0.0 if previous_price == 0)
    @property
    def direction(self) -> str:          # "up" | "down" | "flat"

    def to_dict(self) -> dict:           # JSON/SSE serialization (includes derived fields)
```

`change`, `change_percent`, and `direction` are computed against the **previous price
in the cache** — i.e. tick-over-tick movement, which is what drives the green/red flash
animation on the frontend.

## Abstract Interface — `MarketDataSource`

```python
class MarketDataSource(ABC):
    @abstractmethod
    async def start(self, tickers: list[str]) -> None: ...
    @abstractmethod
    async def stop(self) -> None: ...
    @abstractmethod
    async def add_ticker(self, ticker: str) -> None: ...
    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None: ...
    @abstractmethod
    def get_tickers(self) -> list[str]: ...
```

Contract:

| Method | Guarantee |
|--------|-----------|
| `start(tickers)` | Begin the background task; write initial prices to the cache immediately so SSE has data on first connect. Call **exactly once**. |
| `stop()` | Cancel the background task; release resources. **Idempotent** — safe to call repeatedly. |
| `add_ticker(t)` | Add `t` to the active set. No-op if present. Appears on the next update cycle. |
| `remove_ticker(t)` | Remove `t` from the active set **and** from the `PriceCache`. No-op if absent. |
| `get_tickers()` | Current active ticker list (sync, cheap). |

The interface deliberately **returns no prices**. Prices only ever flow out through
the `PriceCache`. This is what keeps the two implementations interchangeable.

## Shared `PriceCache`

Thread-safe because the simulator/poller may write from a worker thread
(`asyncio.to_thread`) while the SSE generator reads from the event loop.

```python
class PriceCache:
    def update(self, ticker, price, timestamp=None) -> PriceUpdate
    def get(self, ticker) -> PriceUpdate | None
    def get_all(self) -> dict[str, PriceUpdate]      # shallow copy
    def get_price(self, ticker) -> float | None
    def remove(self, ticker) -> None
    @property
    def version(self) -> int                          # monotonic; ++ on every update
```

Key behaviors:

- **Auto-derives `previous_price`.** On `update()`, the prior cached price becomes
  `previous_price`. First update for a ticker → `previous_price == price`
  (`direction == "flat"`).
- **Rounds to 2 dp** on store, so cache values match what users see.
- **`version` counter** powers efficient SSE change-detection: the stream only
  re-serializes and pushes when `version` advanced since the last emit.
- A single `threading.Lock` guards all reads/writes.

## Factory — selection by environment

```python
def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)  # real data
    return SimulatorDataSource(price_cache=price_cache)                     # simulation
```

- Returns an **unstarted** source. The caller owns `await source.start(tickers)`.
- Decision is made **once at startup**. `MASSIVE_API_KEY` empty/unset/whitespace → simulator.
- This is the single branch point between real and simulated data in the whole app.

## The Two Implementations (summary)

| | `SimulatorDataSource` | `MassiveDataSource` |
|---|---|---|
| Source | In-process GBM model | Massive REST snapshot API |
| Cadence | `step()` every **500 ms** | poll every **15 s** (free tier) |
| Mechanism | `GBMSimulator.step()` → cache | `get_snapshot_all()` → cache |
| Blocking? | Pure-Python/NumPy in the loop | Sync SDK call wrapped in `asyncio.to_thread` |
| Failure mode | Logs & continues (`logger.exception`) | Catches all errors per cycle, retries next interval |
| Detail | `MARKET_SIMULATOR.md` | `MASSIVE_API.md` |

Both write `PriceUpdate`s into the same `PriceCache`. Nothing downstream can tell them
apart.

## SSE Integration

The stream endpoint (`stream.py`, `GET /api/stream/prices`) is a thin reader over the
cache — it has no knowledge of the data source:

```python
last_version = -1
while True:
    if await request.is_disconnected():
        break
    if price_cache.version != last_version:          # only push on change
        last_version = price_cache.version
        prices = price_cache.get_all()
        data = {t: u.to_dict() for t, u in prices.items()}
        yield f"data: {json.dumps(data)}\n\n"
    await asyncio.sleep(0.5)
```

Emits `retry: 1000` on connect so the browser's `EventSource` auto-reconnects after
1 s if the connection drops. Sends all tracked tickers as one JSON object per event.

## Lifecycle

```python
# --- App startup ---
cache  = PriceCache()
source = create_market_data_source(cache)          # factory picks real vs. sim
await source.start(initial_watchlist_tickers)       # background task begins
app.include_router(create_stream_router(cache))     # SSE reads the same cache

# --- Watchlist changes (REST or AI chat) ---
await source.add_ticker("PYPL")
await source.remove_ticker("GOOGL")                 # also evicts from cache

# --- Trade execution / portfolio valuation ---
price = cache.get_price("AAPL")                      # fill / mark-to-market

# --- App shutdown ---
await source.stop()                                  # idempotent
```

## Why This Design

| Choice | Rationale |
|--------|-----------|
| Push into a cache (not pull from source) | Decouples update cadence from read cadence; one poll feeds many SSE clients; trade/valuation reads are O(1) and never hit the network |
| Abstract `MarketDataSource` | Real and simulated sources are swapped at one factory line; tests run against the simulator with zero API dependency |
| `PriceUpdate` with computed properties | Derived fields (`change`, `direction`) can't desync from raw values; immutable + slotted = cheap and safe to share across threads |
| `version` counter on the cache | SSE pushes only on actual change — no redundant serialization or bandwidth when prices are static (e.g. market closed) |
| Thread-safe cache | Lets the blocking Massive SDK run in a worker thread without a separate async cache layer |
| Decide source once at startup | No per-request branching; behavior is predictable and matches the env the container booted with |
