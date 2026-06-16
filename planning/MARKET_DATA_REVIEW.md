# Market Data Backend — Code Review (Round 2)

**Date:** 2026-06-15
**Reviewer:** Coding Agent (code review pass)
**Scope:** `backend/app/market/` (9 source modules) + `backend/tests/market/` (6 test modules)
**Baseline:** Supersedes `planning/archive/MARKET_DATA_REVIEW.md` (2026-02-10). This pass
verifies the previously-claimed fixes, re-checks every issue, and adds new findings.

---

## 1. Headline

The market data subsystem is **well-designed, correct, and ready for downstream
integration.** The strategy pattern (ABC + simulator/Massive), the shared thread-safe
`PriceCache`, the GBM math, the Cholesky-correlated moves, and the SSE endpoint all
work as specified and match the 🔒 FROZEN contract in `API_CONTRACT.md` exactly.

The remaining issues are **low-severity polish and test-coverage gaps**, not
correctness blockers. The one item I would not ship without addressing is the
**absent test coverage on the SSE endpoint (`stream.py`, 33%)** — it is the primary
consumer-facing surface and is currently unverified by any automated test.

---

## 2. How I Verified

Ran the full suite, coverage, and lint as the project's `CLAUDE.md` prescribes
(`uv` had to be installed first — it was not present on this machine):

```
uv run --extra dev pytest -q --cov=app --cov-report=term-missing
uv run --extra dev ruff check app/ tests/
```

**Result: 73 passed, 0 failed. Ruff: all checks passed.**

Coverage (this run):

| Module | Stmts | Miss | Cover | Uncovered |
|---|---|---|---|---|
| models.py | 26 | 0 | 100% | |
| cache.py | 39 | 0 | 100% | |
| interface.py | 13 | 0 | 100% | |
| seed_prices.py | 8 | 0 | 100% | |
| factory.py | 15 | 0 | 100% | |
| simulator.py | 139 | 3 | 98% | L149 dup-guard, L268-269 run-loop except |
| massive_client.py | 67 | 4 | 94% | L85-87 poll-loop, L125 real fetch |
| **stream.py** | **36** | **24** | **33%** | **L26-48 route, L62-87 generator** |
| **TOTAL** | **349** | **31** | **91%** | |

Note: overall coverage is now **91%** (up from the 84% recorded in the prior review);
`massive_client.py` rose to 94% and `simulator.py` to 98%. `stream.py` is essentially
unchanged and remains the single notable gap.

I also ran targeted out-of-band checks (not part of the suite) to confirm specific
behaviors — see §4 and §5.

---

## 3. Status of the Prior Review's Issues

All seven enumerated issues from the 2026-02-10 review are **confirmed fixed**:

| # | Prior issue | Status |
|---|---|---|
| 3.1 | `pyproject.toml` hatchling wheel packages | ✅ Fixed — `[tool.hatch.build.targets.wheel] packages = ["app"]` present; `uv sync` succeeds |
| 3.2 | Massive test fragility w/ lazy imports | ✅ Fixed — `RESTClient`/`SnapshotMarketType` imported at module top; tests mock correctly and pass |
| 3.3 | `_generate_events` return type | ✅ Fixed — annotated `AsyncGenerator[str, None]` |
| 3.5 | `get_tickers` reaching into private state | ✅ Fixed — `GBMSimulator.get_tickers()` is public; source delegates to it |
| 3.7 | Unused test imports | ✅ Fixed — ruff clean |
| 4.3 | `DEFAULT_CORR` vs `CROSS_GROUP_CORR` naming | ✅ Fixed — `DEFAULT_CORR` removed, consolidated to `CROSS_GROUP_CORR` |

**Still open** (carried from the prior review's "nice to have" / "missing tests"):

- The **module-level router footgun** (prior §3.6) — still present, and I confirmed it
  is a *live* defect, not just theoretical (see §4.1).
- The **`version` property read outside the lock** (prior §3.4) — still present (§4.4).
- **No SSE test, no concurrency test, no full-10-ticker Cholesky test** (prior §4.2) —
  still missing (§6).

The `MARKET_DATA_SUMMARY.md` line "all issues resolved" is accurate for the seven
enumerated fixes but overstates the test-gap items, which were observations rather than
enumerated fixes and remain open.

---

## 4. Findings

### 4.1 `create_stream_router()` registers routes on a module-level singleton (Severity: Low → Medium)

`stream.py:17` defines a module-global `router = APIRouter(...)`, and
`create_stream_router(price_cache)` attaches `@router.get("/prices")` to that *same*
global object on every call, returning it. This was flagged as a "latent footgun" last
round; I confirmed it is an actual bug under repeated calls:

```python
r1 = create_stream_router(cache1)
r2 = create_stream_router(cache2)
# r1 is r2  -> True   (same global object)
# r1 routes -> ['/api/stream/prices', '/api/stream/prices']  (duplicated)
```

Consequences:
- The route is registered **twice**; Starlette matches the **first** registration, so
  the second cache (`cache2`) is silently ignored — a caller holding `r2` and expecting
  it to serve `cache2` gets `cache1`.
- It defeats test isolation, which is part of *why* `stream.py` has no tests today —
  each test would pollute the shared router.

The app only calls this once at startup, so it works in production *today*. But the
docstring claims the factory exists "to inject the PriceCache without globals," which is
the opposite of what it does. **Fix:** create the `APIRouter` *inside* the factory so
each call returns a fresh router bound to its own closure. This also unblocks SSE
testing (§6).

### 4.2 Massive ticker-casing is inconsistent across the lifecycle (Severity: Low)

`add_ticker`/`remove_ticker` normalize with `.upper().strip()`, but `start(tickers)`
stores them verbatim (`self._tickers = list(tickers)`), and `_fetch_snapshots` sends
whatever is in `self._tickers`. Confirmed effect:

```python
source._tickers = ['aapl']          # as if start(['aapl'])
await source.remove_ticker('aapl')  # uppercases to 'AAPL'
source.get_tickers()  ->  ['aapl']  # stale entry never removed
```

In practice the app seeds uppercase tickers (`AAPL`, …) from the watchlist, so this
won't bite the default flow, but mixed-case input from the chat/watchlist path could
leak a ticker that never polls or never gets removed. **Fix:** normalize once in
`start()` (and ideally in a single private `_norm()` helper used everywhere).
Note the simulator does *no* normalization at all — pick one convention and document it
in `API_CONTRACT.md` so the Backend Engineer can normalize at the boundary instead.

### 4.3 No price sanity check on Massive snapshots (Severity: Low)

`_poll_once` accepts whatever `snap.last_trade.price` returns. `None`/missing fields are
correctly caught (`AttributeError, TypeError`) and skipped, but a `0.0` or negative
price would be written to the cache, then propagate into `change_percent` (which guards
`previous_price == 0` but not a *current* price of 0) and portfolio valuation.
Real-world snapshots occasionally carry stale/zero last-trade values pre-market.
**Fix:** skip updates where `price is None or price <= 0`.

### 4.4 `PriceCache.version` read outside the lock (Severity: Low — unchanged)

Still reads `self._version` without `self._lock`. Safe under CPython's GIL (single-int
read is atomic) and the SSE loop only needs a monotonic hint, so this is benign today.
It would become a real race only on a free-threaded (no-GIL) build. Leaving as-is is
defensible; if touched, wrap the read in the lock for consistency.

### 4.5 SSE has no heartbeat / keep-alive (Severity: Low)

`_generate_events` only yields when `price_cache.version` changes. With the **simulator**
(updates every 500 ms) this is a non-issue. With the **Massive** source the cadence is
15 s (free tier); between polls the generator loops silently with no comment/heartbeat
frame. Some proxies and load balancers idle-timeout a stream with no bytes for >10–30 s.
`X-Accel-Buffering: no` and the `retry: 1000` directive mitigate, and EventSource will
reconnect, but a periodic comment line (`: keepalive\n\n` every ~10–15 s) would make the
stream robust behind proxies for the real-data path. Optional for the simulator-default
demo.

### 4.6 Tracked-set rule is an unmet integration obligation (Severity: Note, not a defect here)

`API_CONTRACT.md` states the cache must track **watchlist ∪ open positions** so a held
ticker removed from the watchlist keeps streaming (else portfolio value silently
freezes). The market subsystem correctly provides the *primitives* —
`remove_ticker` unconditionally evicts from the cache, which is the right low-level
behavior — but the **don't-evict-a-held-ticker** policy lives one layer up and is owned
by the Backend Engineer. Flagging so it is not lost in the seam: the orchestration code
must not call `remove_ticker` for a symbol with an open position. A contract test should
assert this once the portfolio layer exists.

---

## 5. What's Done Well (verified, not just asserted)

- **GBM is mathematically correct.** Log-normal path
  `S·exp((μ − ½σ²)·dt + σ·√dt·Z)`; prices provably stay positive — I ran 10,000 steps
  and confirmed strict positivity (matches `test_prices_are_positive`).
- **Cholesky correlation is sound and stable.** I built the full default 10-ticker
  matrix and the 7-stock all-tech (ρ=0.6) block; both decompose without `LinAlgError`.
  The `n <= 1 → None` short-circuit is correct. (One robustness note: there is no
  `try/except LinAlgError` fallback — if a future correlation scheme produced a
  non-positive-definite matrix, `_rebuild_cholesky` would raise. The current constants
  are safe, so this is a latent-only concern.)
- **SSE output exactly matches the frozen contract.** I drove `_generate_events` with a
  mocked `Request`: it emits `retry: 1000\n\n` first, then
  `data: {"AAPL": {...}}\n\n` with the precise `PriceUpdate.to_dict()` shape
  (`direction:"up"`, `change:0.4`, all seven fields). Version-based change detection
  avoids redundant frames. The fact that this was *easy* to exercise with a mock is
  exactly why the missing test (§6) is a cheap fix.
- **Immutable `PriceUpdate`** (`frozen=True, slots=True`) — correct and memory-efficient;
  immutability is tested.
- **Resilient background loops.** Both `_run_loop` (simulator) and
  `_poll_once`/`_poll_loop` (Massive) catch-and-continue, and `stop()` is cancel-clean
  and idempotent — verified by tests and by inspection.
- **Seed-on-start.** `start()` writes seed prices into the cache before the first loop
  tick, so SSE has data immediately — no blank first frame.
- **Factory + env-var selection** is clean and fully covered (100%), including the
  empty/whitespace key cases.

---

## 6. Test Assessment & Recommended Additions

The suite is genuinely good: 73 focused tests, fast (~3.4 s), deterministic, clean
naming, and it exercises the GBM math, cache semantics, both data sources, factory
selection, and Massive error/malformed-snapshot handling. Coverage is 91%.

The gaps, in priority order:

1. **SSE endpoint tests (highest value).** `stream.py` is at 33% and is the
   consumer-facing surface. After the §4.1 fix, add:
   - an ASGI/`httpx.AsyncClient` (or direct-generator) test asserting the first frame is
     `retry: 1000`, a data frame is valid JSON keyed by ticker, and each entry matches
     `to_dict()`;
   - a disconnect test asserting the loop exits when `request.is_disconnected()` is true;
   - a "no change → no frame" test (version unchanged ⇒ nothing emitted).
   These are low-effort given the generator is a plain async function (demonstrated in §5).
2. **PriceCache concurrency test.** The class advertises thread-safety but no test
   exercises it. Spawn N threads hammering `update`/`get_all` and assert no exception and
   a consistent final count.
3. **Full-10-ticker Cholesky test.** All current sim tests use 1–2 tickers. Add one that
   constructs `GBMSimulator` with all 10 defaults and asserts `_cholesky is not None` and
   `step()` returns 10 finite positive prices — guards the correlation matrix against
   future PD regressions.
4. **Regression tests for the new findings:** Massive casing round-trip (§4.2) and
   zero/negative price rejection (§4.3).

---

## 7. Recommendations (ordered)

**Should fix before building dependent layers:**
1. Make `create_stream_router` build its own `APIRouter` (§4.1) — removes the silent
   wrong-cache bug and unblocks SSE testing.
2. Add SSE endpoint tests (§6.1).

**Should fix soon:**
3. Normalize Massive tickers consistently in `start()` / via one helper (§4.2).
4. Reject `None`/`<= 0` prices in `_poll_once` (§4.3).
5. Add concurrency + full-10-ticker tests (§6.2–6.3).

**Nice to have:**
6. SSE keep-alive comment for the Massive cadence (§4.5).
7. `version` read under lock for consistency (§4.4).
8. Document the tracked-set obligation as a Backend-owned contract test (§4.6).
9. Optional `try/except LinAlgError` fallback in `_rebuild_cholesky` (§5).

---

## 8. Verdict

**Approved for integration.** The subsystem is correct, well-architected, and matches
the frozen wire contract; the test suite is green and the prior round's issues are all
resolved. The open items are polish and coverage, with one of them — the module-level
router (§4.1) — being a confirmed (if currently dormant) bug that should be fixed
together with adding the missing SSE tests, since the two are linked. None of the
findings block downstream work on the portfolio, chat, or frontend layers.
