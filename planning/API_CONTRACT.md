# API_CONTRACT.md — the frontend ↔ backend wire seam

The single source of truth for every `/api/*` shape. The **Backend Engineer owns this
file**; the **Frontend Engineer mirrors it** (AGENTS.md §1–2). The frontend and
backend share no code — this document *is* their integration. A shape changes here
first, with a `decisions.md` note, then both sides follow.

**Status legend:** 🔒 FROZEN (implemented in code — match it exactly) · 📝 PROPOSED
(ratify before building the consumer).

## Conventions

- Base path `/api`; all bodies are JSON; same-origin (no CORS).
- **Money & shares:** USD floats, fractional shares allowed. (Known caveat: float
  money drifts over many trades — acceptable for a sim; if it bites, move to integer
  cents or `Decimal` and update this note.)
- **Timestamps — two formats, on purpose:**
  - *Live price* timestamps are **Unix epoch seconds (float)** — from the market code.
  - *Persisted* timestamps (`recorded_at`, `executed_at`) are **ISO-8601 strings**
    (PLAN.md §7 schema). The frontend must handle both.
- **Errors:** FastAPI default — non-2xx with `{"detail": "<message>"}`. Validation
  failures (insufficient cash/shares, unknown ticker) are **400**; not-found is 404.
- **Mutations return fresh state.** Trade and watchlist endpoints return the updated
  portfolio/watchlist so the UI refreshes without a second round-trip. SSE carries
  *prices only* — it does not carry portfolio/watchlist changes.

---

## 🔒 SSE — `GET /api/stream/prices`   (implemented: `app/market/stream.py`)

`Content-Type: text/event-stream`. First line is a retry directive, then **unnamed
`data:` events emitted only when prices change** (~500 ms cadence). Each event's data
is a JSON **object keyed by ticker**:

```
retry: 1000

data: {"AAPL": {"ticker":"AAPL","price":190.50,"previous_price":190.10,"timestamp":1718450000.12,"change":0.40,"change_percent":0.2104,"direction":"up"}, "GOOGL": { ... }}
```

Per-ticker object (`PriceUpdate.to_dict()` — exact):

| field | type | notes |
|---|---|---|
| `ticker` | string | |
| `price` | float | |
| `previous_price` | float | |
| `timestamp` | float | Unix epoch seconds |
| `change` | float | `price - previous_price`, 4dp |
| `change_percent` | float | percent, 4dp; `0.0` if previous was 0 |
| `direction` | string | `"up"` \| `"down"` \| `"flat"` |

Client: native `EventSource` (auto-reconnects). **Tracked-set rule (📝 backend must
honor):** the price cache tracks **watchlist ∪ open positions** — a ticker you hold
but removed from the watchlist must keep streaming, or portfolio value silently
freezes.

---

## 📝 Portfolio

### `GET /api/portfolio`
```json
{
  "cash_balance": 8420.50,
  "positions": [
    {
      "ticker": "AAPL",
      "quantity": 10.0,
      "avg_cost": 188.20,
      "current_price": 190.50,
      "market_value": 1905.00,
      "unrealized_pnl": 23.00,
      "unrealized_pnl_percent": 1.22
    }
  ],
  "total_value": 10325.50,
  "total_unrealized_pnl": 23.00
}
```
`current_price` comes from the price cache; if a held ticker has no price yet,
`current_price`/`market_value`/`unrealized_pnl` are `null` (frontend renders "—").

### `POST /api/portfolio/trade`
Request: `{ "ticker": "AAPL", "side": "buy", "quantity": 10 }` (`side` ∈ `buy|sell`).
Success **200** — echoes the fill *and* the refreshed portfolio:
```json
{
  "trade": { "id": "uuid", "ticker": "AAPL", "side": "buy", "quantity": 10.0, "price": 190.50, "executed_at": "2026-06-15T12:00:00Z" },
  "portfolio": { "...": "same shape as GET /api/portfolio" }
}
```
Failure **400**: `{"detail": "Insufficient cash: need 1905.00, have 1200.00"}` /
`"Insufficient shares: hold 3, tried to sell 10"` / `"Unknown ticker: ZZZZ"`.
*Buy avg_cost = weighted average; sell reduces quantity (position row deleted when it
hits 0). Trade is atomic — wrap cash+position writes in one transaction.*

### `GET /api/portfolio/history?since=<iso8601?>`
```json
{ "snapshots": [ { "total_value": 10000.00, "recorded_at": "2026-06-15T11:30:00Z" } ] }
```
Source: `portfolio_snapshots` (every ~30 s + after each trade, PLAN.md §7).

---

## 📝 Watchlist

### `GET /api/watchlist` — tickers with latest price merged in
```json
{ "watchlist": [ { "ticker": "AAPL", "price": 190.50, "previous_price": 190.10, "change_percent": 0.21, "direction": "up" } ] }
```
`price` fields `null` until the first stream tick for a freshly added ticker.

### `POST /api/watchlist` — `{ "ticker": "PYPL" }`
**200** returns the updated list (same shape as GET). Duplicate → **409**
`{"detail": "PYPL already in watchlist"}`. (Optional: validate the symbol is known
to the data source; an unknown sim ticker simply never gets a price.)

### `DELETE /api/watchlist/{ticker}`
**200** returns the updated list. Removing a ticker you still **hold** is allowed —
it stays in the price cache via the tracked-set rule above.

---

## 📝 Chat — `POST /api/chat`

Request: `{ "message": "Buy 10 Apple and add PayPal to my watchlist" }`

Response **200** — conversational text, the actions actually executed, and refreshed
state so the UI updates inline:
```json
{
  "message": "Bought 10 AAPL at $190.50 and added PYPL to your watchlist.",
  "actions": {
    "trades": [ { "ticker": "AAPL", "side": "buy", "quantity": 10, "status": "filled", "price": 190.50 } ],
    "watchlist_changes": [ { "ticker": "PYPL", "action": "add", "status": "ok" } ]
  },
  "portfolio": { "...": "refreshed GET /api/portfolio shape" },
  "watchlist": [ { "...": "refreshed GET /api/watchlist entries" } ]
}
```
A rejected action carries `"status": "rejected"` + `"error"` (e.g. insufficient cash)
so the assistant can explain it — the request as a whole still returns 200.

**LLM structured-output schema** (model → backend internal; the backend validates it,
re-checks every action server-side, then maps to the response above — AGENTS.md §5):
```json
{
  "message": "string (required)",
  "trades": [ { "ticker": "AAPL", "side": "buy", "quantity": 10 } ],
  "watchlist_changes": [ { "ticker": "PYPL", "action": "add" } ]
}
```

---

## 📝 System — `GET /api/health`
**200** `{ "status": "ok" }`

---

> Origin: drafted 2026-06-15 (Johanna). SSE section transcribed from
> `app/market/stream.py` + `app/market/models.py` (🔒). REST sections are PROPOSED
> from PLAN.md §7–9 — Backend Engineer ratifies, then freeze.
