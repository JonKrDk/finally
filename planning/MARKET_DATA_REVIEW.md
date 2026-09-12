# Market Data Backend — Code Review (2026-09-12)

**Scope:** `backend/app/market/` (9 modules) and `backend/tests/market/` (6 modules), reviewed
against `planning/PLAN.md` §6–§8 and `planning/MARKET_DATA_DESIGN.md` (the design of record).
Every finding below was verified by running code, not just by reading it.

**Verdict:** The round-1 subsystem is well-structured, lint-clean, and all 73 tests pass. The
simulator path works end-to-end (verified against a live uvicorn server). However:

1. **The Massive (real data) path is broken and the tests cannot see it** — the poller reads a
   field that does not exist on the SDK's `LastTrade` model, so every snapshot is skipped and no
   real price is ever cached. `MagicMock` snapshots in the tests auto-create the missing
   attribute, which is why the suite is green. *(Critical)*
2. **None of the PLAN §6 / DESIGN §18 additions are implemented** — no `open_price`,
   `day_change_percent`, history ring buffer, `/api/prices/{ticker}/history`, ISO-Z timestamps,
   `validation.py`, or Massive-side ticker validation. The SSE payload shape the frontend is
   specified against does not yet exist. This is expected (the design doc was committed after
   the code, as a checklist), but `MARKET_DATA_SUMMARY.md` calling the subsystem "complete" is
   misleading for downstream agents.
3. Two issues from the previous review (`archive/MARKET_DATA_REVIEW.md` §3.4 and §3.6) were
   reported as resolved but are still present in the code.

Recommendation: **do not build the portfolio/watchlist layers on the current `to_dict()` shape.**
Land the §18 delta first (it is the contract everything else consumes), fix the Massive field
bug alongside it, and replace the `MagicMock` snapshots with real SDK model instances so the
regression cannot recur.

---

## 1. Test Results

```
cd backend && uv run --extra dev pytest -q --cov=app --cov-report=term-missing
73 passed, 73 warnings in 3.06s          (Python 3.14.6, pytest 9.0.2, pytest-asyncio 1.3.0)

uv run --extra dev ruff check app/ tests/
All checks passed!
```

| Module | Stmts | Cover | Uncovered lines | Note |
|---|---|---|---|---|
| `models.py` | 26 | 100% | — | |
| `cache.py` | 39 | 100% | — | |
| `interface.py` | 13 | 100% | — | |
| `seed_prices.py` | 8 | 100% | — | |
| `factory.py` | 15 | 100% | — | |
| `simulator.py` | 139 | 98% | 149, 268–269 | `except Exception` branch in `_run_loop` never exercised |
| `massive_client.py` | 67 | 94% | 85–87, 125 | `_poll_loop` body and the real `_fetch_snapshots` call |
| `stream.py` | 36 | **33%** | 26–48, 62–87 | **No test touches the SSE endpoint or generator** |
| **Total** | 349 | 91% | | |

The 73 warnings are all one line: `conftest.py:11: DeprecationWarning: 'asyncio.DefaultEventLoopPolicy'
is deprecated and slated for removal in Python 3.16`. The `event_loop_policy` fixture in
`tests/conftest.py` is unnecessary with `asyncio_mode = "auto"` and should be deleted.

Coverage numbers in `MARKET_DATA_SUMMARY.md` (84% overall, `massive_client.py` 56%,
`test_simulator.py` 17 tests) are stale — measured today: 91%, 94%, and 19 tests respectively.

### Manual verification performed

| Check | Result |
|---|---|
| Live SSE against uvicorn, 3 tickers, 4 events read | `200 text/event-stream`, `Cache-Control: no-cache`, first line `retry: 1000`, one batched event per ~500 ms containing all tickers, `direction` alternates correctly |
| `massive.rest.models.trades.LastTrade` field list | `ticker, trf_timestamp, sequence_number, sip_timestamp, participant_timestamp, conditions, correction, id, price, trf_id, size, exchange, tape` — **no `timestamp`** |
| `TickerSnapshot.from_dict({"lastTrade": {"p": 190.52, "t": 1707580800000123456}})` then `.last_trade.timestamp` | `AttributeError: 'LastTrade' object has no attribute 'timestamp'` |
| `RESTClient.get_snapshot_all` / `get_snapshot_ticker` signatures | Match the kwargs used in `massive_client.py` (`market_type=`, `tickers=` / `ticker=`) |
| `create_stream_router()` called twice | Same `APIRouter` object returned; `/api/stream/prices` registered twice |
| `PriceCache.update("X", 1.0, timestamp=0.0)` | Stored timestamp is `time.time()`, not `0.0` |
| `GBMSimulator(["AAPL"])._params["AAPL"] is TICKER_PARAMS["AAPL"]` | `True` — shared mutable module constant |
| `GBMSimulator.get_price()` after one step | Returns unrounded internal value (`190.0035941280501`) |

---

## 2. Issues

Severity: **Critical** = feature does not work; **High** = spec violation a downstream layer will
hit; **Medium** = latent bug / footgun; **Low** = hygiene.

### 2.1 Massive poller reads a non-existent field — real data never reaches the cache (Critical)

`app/market/massive_client.py:101-103`

```python
price = snap.last_trade.price
timestamp = snap.last_trade.timestamp / 1000.0   # ms -> seconds
```

The `massive` SDK (2.2.0, the version in `uv.lock`) models the snapshot's `lastTrade.t` field as
`LastTrade.sip_timestamp`. There is no `timestamp` attribute. The `AttributeError` is caught by the
`except (AttributeError, TypeError)` on line 110, logged at WARNING as "Skipping snapshot", and
the loop moves on. **Every ticker on every poll is skipped.** With a valid API key the app
starts, `/api/health` will report `tickers: 0`, and the frontend shows `—` forever.

There is a second, independent error on the same line: the SIP timestamp is in **nanoseconds**,
not milliseconds. Dividing by `1000.0` would produce a timestamp ~55,000 years in the future.

Root cause: `planning/archive/MASSIVE_API.md:77-81` shows a hand-written response structure with
`"last_trade": {"timestamp": 1675190399000}` that does not match the SDK. The implementer coded
against the document instead of the library.

Why the tests miss it: `tests/market/test_massive.py:11-18` builds snapshots with `MagicMock()`,
which fabricates any attribute on access, including `.timestamp`.

**Fix:**

```python
price = snap.last_trade.price
ts_ns = snap.last_trade.sip_timestamp
if price is None or ts_ns is None:
    raise TypeError("snapshot has no last trade")
timestamp = ts_ns / 1e9
```

and rebuild the test fixtures from the real model so the shape is enforced:

```python
from massive.rest.models.snapshot import TickerSnapshot

def _make_snapshot(ticker, price, ts_ns):
    return TickerSnapshot.from_dict({"ticker": ticker, "lastTrade": {"p": price, "t": ts_ns}})
```

### 2.2 PLAN §6 additions and DESIGN §18 delta not implemented (High)

The design document's §18 lists 16 items as "the implementation checklist". **Zero of them are
in the code.** The ones that break the contract other layers are specified against:

| # | Missing | Consumer that will break | File |
|---|---|---|---|
| 1, 3 | `PriceUpdate.open_price`, `day_change_percent` | Watchlist "change %" column (PLAN §10) | `models.py`, `cache.py` |
| 2 | `to_dict()["timestamp"]` is a float, not ISO-8601 `Z` | PLAN §7 convention; frontend date parsing | `models.py:45` |
| 4, 12 | History ring buffer + `GET /api/prices/{ticker}/history` | Main chart "never empty on selection" (PLAN §8, §10) | `cache.py`, `stream.py` |
| 6 | `validation.py` (`normalize_ticker`, `InvalidTickerError`, `UnknownTickerError`) | 422/404 rules for watchlist, trade, history routes (PLAN §6, §8) | new file |
| 10 | Massive `add_ticker()` validates the symbol and polls once | PLAN §6 "unknown symbols → 404"; currently a bad symbol is silently added and the user waits up to 15 s for nothing | `massive_client.py:66-70` |
| 14 | `active_source_name()` | `/api/health.data_source` (PLAN §8) | `factory.py` |
| 7 | Unknown-ticker seed range is `(50, 300)`, PLAN says `$20–$500` | Cosmetic, but `test_simulator.py:60` asserts the wrong range | `simulator.py:151` |

The current SSE event shape is:

```json
{"ticker":"AAPL","price":190.01,"previous_price":190.0,"timestamp":1789229557.287,
 "change":0.01,"change_percent":0.0053,"direction":"up"}
```

PLAN §6 specifies `open_price`, `day_change_percent`, and an ISO string timestamp in addition.
The frontend must not be written until this is landed.

### 2.3 Module-level `APIRouter` — double registration on repeated factory calls (Medium)

`app/market/stream.py:17`

```python
router = APIRouter(prefix="/api/stream", tags=["streaming"])

def create_stream_router(price_cache: PriceCache) -> APIRouter:
    @router.get("/prices")
    ...
    return router
```

Verified: two calls return the same object with `/api/stream/prices` registered twice, each
closing over a different cache. FastAPI dispatches to the first match, so the second cache is
silently ignored — a test that builds a fresh app per test case would read stale prices from the
first test's cache. This was §3.6 of the previous review; `MARKET_DATA_SUMMARY.md` says "all
issues resolved" but it was not in the list of seven fixes and is still present. Move the
`APIRouter(...)` construction inside the function (DESIGN §12.1 already shows this).

### 2.4 `PriceCache.version` read outside the lock (Medium-Low)

`app/market/cache.py:64-67`. Same story as 2.3 — previous review §3.4, reported resolved, still
present. Every other accessor takes the lock; this one does not. Harmless on CPython today, but
the Massive path writes from an `asyncio.to_thread` worker while the SSE generator reads on the
event loop, so this is exactly the field a free-threaded build would race on. One-line fix.

### 2.5 Removed Massive ticker can be resurrected by an in-flight poll (Medium)

`app/market/massive_client.py:72-76` and `:97-108`. `remove_ticker()` runs on the event loop
while `_fetch_snapshots` may be mid-flight in a worker thread with the *old* ticker list. When the
thread returns, `_poll_once` writes every returned snapshot to the cache, including the ticker
just removed. Nothing ever removes it again (subsequent polls do not include it, and only
`remove_ticker` calls `cache.remove`), so it stays in every SSE event until restart. The
15 s poll window makes this easy to hit from the UI.

Fix: skip snapshots whose ticker is not in `self._tickers` at write time.

### 2.6 `timestamp or time.time()` treats `0.0` as absent (Low)

`app/market/cache.py:30`. `ts = timestamp or time.time()` — a caller passing `0.0` (or any
falsy float) gets `time.time()` instead. Use `timestamp if timestamp is not None else time.time()`
(DESIGN §5 has it right).

### 2.7 `GBMSimulator.get_price()` returns unrounded internal state (Low)

`app/market/simulator.py:136-138`. Returns e.g. `190.0035941280501`. The cache rounds on
`update()` so the SSE output is fine, but any direct caller (the demo, a future test) sees
unrounded numbers. DESIGN §18 item 9. `test_initial_prices_match_seeds` only passes because seed
prices are already 2 dp.

### 2.8 Known-ticker params share the module-level dict (Low)

`app/market/simulator.py:152`: `TICKER_PARAMS.get(ticker, dict(DEFAULT_PARAMS))` copies the
*default* but hands out the *shared* dict for known tickers. Any future per-instance mutation
(e.g. a "volatility regime" feature) would leak across simulator instances and across tests.
Wrap in `dict(...)` unconditionally. Also on line 151, `random.uniform(50.0, 300.0)` is evaluated
as the `.get()` default on every call even for seeded tickers — harmless but wasteful; use `or`.

### 2.9 `_generate_events` swallows `CancelledError` (Low)

`app/market/stream.py:86-87`. Catching `asyncio.CancelledError` without re-raising converts a
cancellation into a normal return. Starlette tolerates this, but it is the pattern that causes
"task never finishes cancelling" bugs elsewhere. Add `raise` after the log line (DESIGN §12.1
does).

### 2.10 No Cholesky failure guard (Low)

`app/market/simulator.py:172`. `np.linalg.cholesky(corr)` will raise `LinAlgError` if the
correlation matrix is ever not positive-definite. With the current constants it always is, but
`_rebuild_cholesky` runs inside `add_ticker`, which is on the request path for
`POST /api/watchlist` — a bad constant edit would turn every watchlist add into a 500 rather
than degrading to independent draws. DESIGN §18 item 8.

---

## 3. Test Suite Assessment

**Strengths:** good unit coverage of the math and cache semantics; async lifecycle tests are
clean (start/stop/idempotent stop); factory env-var handling covers empty and whitespace keys;
`test_prices_are_positive` runs 10 000 steps which is a meaningful invariant check.

**Weaknesses, in priority order:**

1. **`MagicMock` snapshots hide the real SDK shape** (`test_massive.py:11-18`). This is the direct
   cause of 2.1 shipping green. Every Massive test that touches `last_trade` should construct a
   real `TickerSnapshot` (via `from_dict`) or at minimum a `SimpleNamespace` with the real field
   names. A `MagicMock` cannot fail an attribute lookup, so it cannot test attribute lookups.
2. **`stream.py` is untested** (33%). No test opens the SSE endpoint, checks the `retry:` line,
   verifies one-event-per-tick batching, or verifies the disconnect path. DESIGN §16.6 shows the
   shape of the test, but it needs `httpx`, which is **not in the dev extras** — add
   `"httpx>=0.27"` to `[project.optional-dependencies].dev`. Note that `httpx.ASGITransport`
   buffers the full body and cannot consume an infinite stream; the test must either run the
   generator directly with a fake `Request`, or use a real server thread as this review did.
3. **`test_exception_resilience` does not inject an exception** (`test_simulator_source.py:96`).
   It starts a source, sleeps, and asserts the task is alive — which would pass without the
   `try/except` in `_run_loop`. Coverage confirms lines 268–269 are never executed. Patch
   `source._sim.step` with `side_effect=RuntimeError` and assert the loop is still running and
   the next tick recovers.
4. **`test_custom_event_probability` asserts nothing** about shocks. With `event_probability=1.0`
   every tick should move the price by 2–5%; assert that.
5. **`test_unknown_ticker_gets_random_seed_price`** locks in the `50–300` range that contradicts
   PLAN §6 (`$20–$500`). Update the constant and the assertion together.
6. **Timing-sensitive assertions** — `test_custom_update_interval` sleeps 50 ms and requires
   `> 2` updates at a 10 ms interval. On a loaded CI runner this will flake. Prefer polling
   `cache.version` with a deadline, or use a larger margin.
7. `test_prices_rounded_to_two_decimals` checks `str(price).split('.')`, which passes for any
   float that happens to print short. Use `price == round(price, 2)`.
8. `tests/conftest.py` — delete the `event_loop_policy` fixture (source of all 73 warnings; not
   needed with `asyncio_mode = "auto"`).

Tests still to be written per DESIGN §16: open price / day change (16.1), history buffer (16.2),
ISO-Z serialization (16.3), validation (16.4), SSE (16.6), Massive with real models (16.7),
thread safety (16.8).

---

## 4. Documentation Drift

| Document | Says | Actual |
|---|---|---|
| `MARKET_DATA_SUMMARY.md` | "Complete, tested, reviewed, all issues resolved" | Round-1 complete; §18 delta not started; two prior-review items open; Massive path non-functional |
| `MARKET_DATA_SUMMARY.md` | 84% coverage, `massive_client.py` 56%, `test_simulator.py` 17 tests | 91%, 94%, 19 |
| `backend/CLAUDE.md` | `PriceUpdate` fields: `ticker, price, previous_price, timestamp` | Correct for as-built, but omits that this is pre-§18; downstream agents will code against it |
| `backend/README.md` | `uv sync --dev` | Dev tools are in `[project.optional-dependencies]`, so the command is `uv sync --extra dev` (as `CLAUDE.md` says). `--dev` selects dependency *groups*, which this project does not define. |
| `archive/MASSIVE_API.md:77-81` | `last_trade.timestamp` in ms | SDK exposes `last_trade.sip_timestamp` in ns. Add a correction note so the next implementer is not misled again. |

Environment note: there is no `.python-version` in the repo; `uv` picked Python **3.14.6** locally
while the Dockerfile in PLAN §11 specifies 3.12-slim. Pinning `3.12` avoids "works locally, differs
in the container" surprises (the deprecation warning above is 3.14-specific, for example).

---

## 5. What Is Done Well

- Clean strategy pattern: `MarketDataSource` ABC, two implementations, one `PriceCache`, one
  factory. Dependency direction is strictly downward; only `stream.py` imports FastAPI.
- The GBM implementation is correct: Itô drift correction, `dt` derived from trading seconds,
  unrounded internal state with rounding at the boundary, Cholesky-correlated draws with a
  sensible sector structure. The sub-cent per-tick move is the right texture for a live tape.
- Seeding the cache in `start()` before the first tick means the first SSE event is never empty.
- Version-counter change detection in the SSE generator is the right call — O(1) and skips
  serialization when nothing changed.
- Error philosophy is consistent: background loops log and continue; nothing in the hot path can
  kill the stream.
- `asyncio.to_thread` for the synchronous `RESTClient` keeps the event loop free.
- Lint-clean under `ruff` with `E,F,I,N,W`.

---

## 6. Action Checklist

Ordered so that each step leaves the suite green.

1. **Fix 2.1** — `sip_timestamp / 1e9`, guard `None`; rewrite `_make_snapshot` with
   `TickerSnapshot.from_dict`. *(blocks any real-data demo)*
2. Fix 2.3, 2.4, 2.6, 2.9 — four one-to-three-line changes.
3. Fix 2.5 — filter stale snapshots on write.
4. Add `httpx` to dev extras; delete the `conftest.py` fixture; strengthen the four weak tests in
   §3 (items 3, 4, 6, 7).
5. Implement DESIGN §18 items 1–16 with the §16 tests. This is the contract the portfolio,
   watchlist, chat, and frontend work all depend on — it should be the very next task, before
   PLAN §13 step 1 ("Backend skeleton").
6. Update `MARKET_DATA_SUMMARY.md`, `backend/CLAUDE.md`, and `backend/README.md`; add a
   correction note to `archive/MASSIVE_API.md`; add `.python-version` = `3.12`.
