# Market Simulator Design

Approach and code structure for simulating realistic stock prices when no Massive API
key is configured. This is FinAlly's **default** data source — most users (and all E2E
tests) run on the simulator, so it has to look and feel like a live market.

> **Status:** Implemented in `backend/app/market/simulator.py` (model +
> `MarketDataSource` wrapper) and `backend/app/market/seed_prices.py` (constants).
> It conforms to the `MarketDataSource` interface described in `MARKET_INTERFACE.md`.

## Overview

The simulator uses **Geometric Brownian Motion (GBM)** — the same process underlying
Black-Scholes — to generate price paths that:

- evolve continuously with random noise,
- **never go negative** (GBM is multiplicative; `exp()` is always positive),
- exhibit the lognormal return distribution seen in real markets,
- **move in correlated sectors** (tech stocks rise/fall together), via a Cholesky
  decomposition of a correlation matrix,
- occasionally **spike** 2–5% for visual drama.

A background asyncio task advances every ticker every **500 ms** and writes the new
prices into the shared `PriceCache`. From the frontend's perspective this is
indistinguishable from a live feed.

## GBM Math

Each tick, a price evolves as:

```
S(t+dt) = S(t) · exp( (μ − σ²/2)·dt  +  σ·√dt·Z )
```

| Symbol | Meaning |
|--------|---------|
| `S(t)` | Current price |
| `μ` (mu) | Annualized drift / expected return (e.g. 0.05 = 5%) |
| `σ` (sigma) | Annualized volatility (e.g. 0.20 = 20%) |
| `dt` | Time step as a fraction of a trading year |
| `Z` | Standard normal draw — **correlated** across tickers (see below) |

### Choosing `dt`

A trading year is modeled as 252 days × 6.5 hours × 3600 s = **5,896,800 trading
seconds**. A 500 ms tick is therefore:

```
dt = 0.5 / 5_896_800 ≈ 8.48e-8
```

In code (`simulator.py`):

```python
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600    # 5,896,800
DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR    # ~8.48e-8
```

This tiny `dt` yields sub-cent moves per tick that accumulate naturally — with
`σ = 0.50` (TSLA), a full simulated session produces roughly the right intraday range.

## Correlated Moves (Cholesky)

Real stocks don't move independently. We build a correlation matrix `C` for the active
tickers and compute its Cholesky factor `L = cholesky(C)`. Then, given a vector of
independent standard normals `Z_ind`:

```
Z_correlated = L @ Z_ind
```

`Z_correlated` has the desired covariance structure, so sector peers move together.

**Correlation structure** (`seed_prices.py`):

| Pair | ρ | Constant |
|------|---|----------|
| Two tech names | 0.6 | `INTRA_TECH_CORR` |
| Two finance names | 0.5 | `INTRA_FINANCE_CORR` |
| Anything involving **TSLA** | 0.3 | `TSLA_CORR` (it does its own thing) |
| Cross-sector / unknown | 0.3 | `CROSS_GROUP_CORR` |

Groups:

```python
CORRELATION_GROUPS = {
    "tech":    {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}
```

The matrix is **rebuilt whenever a ticker is added or removed** — O(n²), but `n` is
small (< 50), so it's negligible. Cholesky requires a positive-semidefinite matrix;
the chosen ρ values keep it valid.

## Random Events

Each ticker, each tick, has a small probability (`event_probability = 0.001`) of a
sudden 2–5% shock — adds drama and keeps the dashboard visually alive:

```python
if random.random() < self._event_prob:
    shock_magnitude = random.uniform(0.02, 0.05)
    shock_sign = random.choice([-1, 1])
    self._prices[ticker] *= 1 + shock_magnitude * shock_sign
```

At ~0.1% per tick per ticker, with 10 tickers at 2 ticks/s, expect an event roughly
**every ~50 seconds** somewhere on the board.

## Seed Prices & Per-Ticker Parameters

Realistic starting prices (as of project creation) and per-ticker volatility/drift
live in `seed_prices.py`:

```python
SEED_PRICES = {
    "AAPL": 190.0, "GOOGL": 175.0, "MSFT": 420.0, "AMZN": 185.0, "TSLA": 250.0,
    "NVDA": 800.0, "META": 500.0, "JPM": 195.0,  "V": 280.0,   "NFLX": 600.0,
}

TICKER_PARAMS = {
    "AAPL":  {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT":  {"sigma": 0.20, "mu": 0.05},
    "AMZN":  {"sigma": 0.28, "mu": 0.05},
    "TSLA":  {"sigma": 0.50, "mu": 0.03},   # high vol, modest drift
    "NVDA":  {"sigma": 0.40, "mu": 0.08},   # high vol, strong drift
    "META":  {"sigma": 0.30, "mu": 0.05},
    "JPM":   {"sigma": 0.18, "mu": 0.04},   # low vol (bank)
    "V":     {"sigma": 0.17, "mu": 0.04},   # low vol (payments)
    "NFLX":  {"sigma": 0.35, "mu": 0.05},
}

DEFAULT_PARAMS = {"sigma": 0.25, "mu": 0.05}   # used for unknown tickers
```

Tickers added mid-session that aren't in `SEED_PRICES` start at a random price between
**$50 and $300** and use `DEFAULT_PARAMS`.

## Code Structure

Two classes in `simulator.py`, split by responsibility:

- **`GBMSimulator`** — the pure math model. Holds prices/params/Cholesky, exposes
  `step()`, `add_ticker()`, `remove_ticker()`, `get_price()`, `get_tickers()`. No I/O,
  no asyncio — trivially unit-testable.
- **`SimulatorDataSource`** — the `MarketDataSource` implementation. Wraps a
  `GBMSimulator` in a 500 ms asyncio loop that writes to the `PriceCache`.

```
backend/app/market/
├── seed_prices.py    # SEED_PRICES, TICKER_PARAMS, DEFAULT_PARAMS, correlation constants
└── simulator.py      # GBMSimulator (model) + SimulatorDataSource (async wrapper)
```

### `GBMSimulator.step()` — the hot path (called every 500 ms)

```python
def step(self) -> dict[str, float]:
    n = len(self._tickers)
    if n == 0:
        return {}

    z_ind = np.random.standard_normal(n)
    z = self._cholesky @ z_ind if self._cholesky is not None else z_ind

    result = {}
    for i, ticker in enumerate(self._tickers):
        mu, sigma = self._params[ticker]["mu"], self._params[ticker]["sigma"]

        drift = (mu - 0.5 * sigma**2) * self._dt
        diffusion = sigma * math.sqrt(self._dt) * z[i]
        self._prices[ticker] *= math.exp(drift + diffusion)

        if random.random() < self._event_prob:          # occasional shock
            shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
            self._prices[ticker] *= 1 + shock

        result[ticker] = round(self._prices[ticker], 2)
    return result
```

### `SimulatorDataSource` — the async wrapper

```python
class SimulatorDataSource(MarketDataSource):
    def __init__(self, price_cache, update_interval=0.5, event_probability=0.001):
        self._cache = price_cache
        self._interval = update_interval
        self._event_prob = event_probability
        self._sim = None
        self._task = None

    async def start(self, tickers):
        self._sim = GBMSimulator(tickers=tickers, event_probability=self._event_prob)
        for t in tickers:                                # seed cache immediately
            price = self._sim.get_price(t)
            if price is not None:
                self._cache.update(ticker=t, price=price)
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")

    async def stop(self):                                # idempotent
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None

    async def add_ticker(self, ticker):
        if self._sim:
            self._sim.add_ticker(ticker)
            price = self._sim.get_price(ticker)          # seed cache right away
            if price is not None:
                self._cache.update(ticker=ticker, price=price)

    async def remove_ticker(self, ticker):
        if self._sim:
            self._sim.remove_ticker(ticker)
        self._cache.remove(ticker)

    def get_tickers(self):
        return self._sim.get_tickers() if self._sim else []

    async def _run_loop(self):
        while True:
            try:
                if self._sim:
                    for ticker, price in self._sim.step().items():
                        self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed")   # never let the loop die
            await asyncio.sleep(self._interval)
```

## Behavior Notes

- **No negative prices** — multiplicative GBM with `exp()` guarantees positivity.
- **Sub-cent per-tick moves** accumulate into realistic intraday ranges over minutes.
- **Cache seeded on `start()` and on `add_ticker()`** so SSE/UI never sees an empty or
  price-less ticker, even before the first `step()`.
- **Loop is crash-proof** — a per-iteration `try/except` logs and continues; one bad
  step can't kill the background task.
- **Cholesky rebuilt on ticker add/remove** — O(n²), negligible for n < 50.
- **Determinism for tests** — seed NumPy/`random` (`np.random.seed`, `random.seed`)
  before stepping to get reproducible paths; the model has no other hidden state.

## Why GBM (vs. simpler random walks)

| Property | Why it matters here |
|----------|---------------------|
| Lognormal returns | Matches how real equity prices are distributed; percentage moves, not additive |
| Can't go negative | A naive additive random walk can cross zero — nonsensical for a stock |
| Tunable μ / σ per ticker | Lets TSLA jump around while JPM stays calm — visually believable |
| Correlation via Cholesky | Sector co-movement makes the board feel like a real market, not 10 independent noises |
| Cheap to compute | One `exp()` + one matrix-vector product per tick; trivial for 10–50 tickers at 2 Hz |
