# Market Data Backend — Detailed Design

**Status of this document:** current design of record for `backend/app/market/`.
It supersedes `planning/archive/MARKET_DATA_DESIGN.md`, which described the first
implementation round. The subsystem described here is the as-built code plus the two
additions PLAN §6 requires (session open price, recent price history) and the
serialization/validation rules that the rest of the platform depends on.

**Audience:** the Backend Engineer implementing the remaining platform, and any agent
touching `app/market/`.

---

## Table of Contents

1. [Scope and Contracts](#1-scope-and-contracts)
2. [File Structure](#2-file-structure)
3. [Time and Number Conventions](#3-time-and-number-conventions)
4. [Data Model — `PriceUpdate`](#4-data-model--priceupdate)
5. [Price Cache](#5-price-cache)
6. [Ticker Validation](#6-ticker-validation)
7. [Abstract Interface — `MarketDataSource`](#7-abstract-interface--marketdatasource)
8. [Seed Prices and Ticker Parameters](#8-seed-prices-and-ticker-parameters)
9. [Simulator](#9-simulator)
10. [Massive API Client](#10-massive-api-client)
11. [Factory](#11-factory)
12. [HTTP Surface — SSE Stream and History](#12-http-surface--sse-stream-and-history)
13. [App Lifecycle Integration](#13-app-lifecycle-integration)
14. [Watchlist and Position Coordination](#14-watchlist-and-position-coordination)
15. [Error Handling and Edge Cases](#15-error-handling-and-edge-cases)
16. [Testing Strategy](#16-testing-strategy)
17. [Configuration Summary](#17-configuration-summary)
18. [Delta From the As-Built Code](#18-delta-from-the-as-built-code)

---

## 1. Scope and Contracts

The market data subsystem is the only part of the platform that knows where prices come
from. It owns exactly three things:

1. **Production** — a background task that generates (simulator) or fetches (Massive)
   prices for a set of tickers.
2. **Storage** — a thread-safe in-memory cache of the latest price, the session open
   price, and a bounded recent history per ticker. Nothing is persisted to SQLite.
3. **Publication** — an SSE endpoint that pushes batched snapshots to the browser, and a
   REST endpoint that returns recent history for one ticker.

Everything else — portfolio valuation, trade execution, the LLM prompt context — is a
*reader* of `PriceCache` and must never import `SimulatorDataSource` or
`MassiveDataSource` directly.

### The three contracts other components rely on

| Contract | Consumers | Guarantee |
|---|---|---|
| `PriceCache.get_price(ticker) -> float \| None` | trade execution, portfolio, chat context | Latest price rounded to 2 dp, or `None` if the ticker is not tracked |
| `GET /api/stream/prices` | frontend | One batched SSE event per changed tick, at most every ~500 ms, containing every tracked ticker |
| `GET /api/prices/{ticker}/history` | frontend main chart | Up to 600 `{timestamp, price}` points, oldest first |

A trade against a ticker with no cached price is a `409`, never a guess. That rule lives
in the portfolio layer but exists because of this contract — the cache returns `None`
rather than a stale or invented price.

---

## 2. File Structure

```
backend/app/market/
├── __init__.py          # Public API re-exports — the only import surface
├── models.py            # PriceUpdate dataclass + ISO-8601 helpers
├── cache.py             # PriceCache: latest price, open price, history ring buffer
├── interface.py         # MarketDataSource ABC
├── validation.py        # normalize_ticker() + InvalidTickerError / UnknownTickerError
├── seed_prices.py       # SEED_PRICES, TICKER_PARAMS, correlation constants
├── simulator.py         # GBMSimulator + SimulatorDataSource
├── massive_client.py    # MassiveDataSource (Polygon.io REST poller)
├── factory.py           # create_market_data_source()
└── stream.py            # create_stream_router(), create_prices_router()
```

Dependency direction is strictly downward — no module imports one listed below it:

```
stream.py  factory.py
     │         │
     ├─────────┼──────────────┬──────────────────┐
     │         │              │                  │
     ▼         ▼              ▼                  ▼
  cache.py  simulator.py  massive_client.py  validation.py
     │         │              │
     └────┬────┴──────────────┘
          ▼
     models.py ── interface.py ── seed_prices.py
```

`models.py`, `interface.py`, `seed_prices.py` and `validation.py` have no intra-package
imports and no FastAPI dependency. Only `stream.py` imports FastAPI.

---

## 3. Time and Number Conventions

PLAN §7 requires ISO 8601 UTC with a `Z` suffix on every timestamp crossing a boundary.
PLAN §6 requires 500 ms tick arithmetic and a 600-point ring buffer, where float seconds
are far cheaper to store and compare.

**Resolution: floats internally, ISO-Z at the boundary.** `PriceUpdate.timestamp` is a
Unix float in seconds. Every serializer (`to_dict()`, the history endpoint) converts to
ISO-Z. Nothing outside `app/market/` ever sees a float timestamp.

```python
# app/market/models.py
from datetime import datetime, timezone


def to_iso_z(ts: float) -> str:
    """Unix seconds -> '2026-09-11T14:03:22.512Z' (millisecond precision, UTC)."""
    return (
        datetime.fromtimestamp(ts, tz=timezone.utc)
        .isoformat(timespec="milliseconds")
        .replace("+00:00", "Z")
    )


def from_iso_z(value: str) -> float:
    """Inverse of to_iso_z(). Used by tests and by any caller parsing our own output."""
    return datetime.fromisoformat(value.replace("Z", "+00:00")).timestamp()
```

Rounding rules, applied at the point of storage so every reader sees the same number:

| Quantity | Rule | Where applied |
|---|---|---|
| Price | 2 dp | `PriceCache.update()` |
| `change` | 4 dp | `PriceUpdate.change` property |
| `change_percent`, `day_change_percent` | 4 dp | `PriceUpdate` properties |
| Share quantity | 4 dp | portfolio layer (not this subsystem) |

Prices are rounded **once**, in `PriceCache.update()`. The simulator keeps unrounded
internal state so rounding error does not accumulate across ticks — see §9.

---

## 4. Data Model — `PriceUpdate`

The only structure that leaves the market data layer. Frozen and slotted: it is created
twice per second per ticker and must never be mutated by a reader.

```python
# app/market/models.py
from __future__ import annotations

import time
from dataclasses import dataclass, field


@dataclass(frozen=True, slots=True)
class PriceUpdate:
    """Immutable snapshot of a single ticker's price at a point in time."""

    ticker: str
    price: float
    previous_price: float
    open_price: float                                     # First price seen this session
    timestamp: float = field(default_factory=time.time)   # Unix seconds

    # --- Tick-to-tick deltas (drive the green/red flash) ---

    @property
    def change(self) -> float:
        return round(self.price - self.previous_price, 4)

    @property
    def change_percent(self) -> float:
        if self.previous_price == 0:
            return 0.0
        return round((self.price - self.previous_price) / self.previous_price * 100, 4)

    @property
    def direction(self) -> str:
        """'up', 'down', or 'flat' — the frontend flashes on up/down."""
        if self.price > self.previous_price:
            return "up"
        if self.price < self.previous_price:
            return "down"
        return "flat"

    # --- Session delta (drives the watchlist "change %" column) ---

    @property
    def day_change_percent(self) -> float:
        """(price - open_price) / open_price * 100."""
        if self.open_price == 0:
            return 0.0
        return round((self.price - self.open_price) / self.open_price * 100, 4)

    def to_dict(self) -> dict:
        """Serialize for JSON / SSE. Timestamp becomes ISO-8601 UTC with Z."""
        return {
            "ticker": self.ticker,
            "price": self.price,
            "previous_price": self.previous_price,
            "open_price": self.open_price,
            "change": self.change,
            "change_percent": self.change_percent,
            "day_change_percent": self.day_change_percent,
            "direction": self.direction,
            "timestamp": to_iso_z(self.timestamp),
        }
```

### Two different "change" numbers, deliberately

- `change` / `change_percent` / `direction` compare against the **previous tick**. They
  are tiny (sub-cent) and exist only so the frontend knows which way to flash a cell.
- `day_change_percent` compares against the **session open** — the first price the
  process ever saw for that ticker. This is the number the watchlist displays as
  "change %". In simulator mode "session" means since process start; in Massive mode it
  means since the poller first saw the ticker (PLAN §6, Decision #2).

A ticker added to the watchlist mid-session therefore starts at `0.00%` and drifts from
there. That is correct and intended — there is no prior close to compare against in
simulator mode.

### First update for a ticker

`previous_price == price` and `open_price == price`, so `direction == "flat"`,
`change == 0.0`, `day_change_percent == 0.0`. No special-casing needed downstream.

---

## 5. Price Cache

`PriceCache` is the single point of truth. Producers write; SSE, valuation, trades, and
the chat prompt builder read. It holds three parallel structures per ticker.

```python
# app/market/cache.py
from __future__ import annotations

import time
from collections import deque
from threading import Lock

from .models import PriceUpdate, to_iso_z

HISTORY_MAXLEN = 600  # ~5 minutes at 500 ms ticks (PLAN §6)


class PriceCache:
    """Thread-safe store of latest price, session open, and recent history.

    Writers: exactly one data source task (simulator loop or Massive poller).
    Readers: SSE generator, portfolio valuation, trade execution, chat context.

    The Massive client executes its HTTP call via asyncio.to_thread, so writes can
    arrive from a worker thread while the event loop reads. Hence a real Lock rather
    than relying on the GIL.
    """

    def __init__(self, history_maxlen: int = HISTORY_MAXLEN) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._open: dict[str, float] = {}
        self._history: dict[str, deque[tuple[float, float]]] = {}
        self._history_maxlen = history_maxlen
        self._lock = Lock()
        self._version: int = 0  # Bumped on every update; SSE change detection

    # --- Write path ---

    def update(
        self,
        ticker: str,
        price: float,
        timestamp: float | None = None,
        open_price: float | None = None,
    ) -> PriceUpdate:
        """Record a new price. Returns the created PriceUpdate.

        The first update for a ticker establishes its session open price, unless
        `open_price` is supplied explicitly (see the sidebar below).
        """
        with self._lock:
            ts = timestamp if timestamp is not None else time.time()
            rounded = round(price, 2)

            prev = self._prices.get(ticker)
            previous_price = prev.price if prev else rounded

            if ticker not in self._open:
                self._open[ticker] = round(open_price, 2) if open_price is not None else rounded

            update = PriceUpdate(
                ticker=ticker,
                price=rounded,
                previous_price=previous_price,
                open_price=self._open[ticker],
                timestamp=ts,
            )
            self._prices[ticker] = update

            history = self._history.get(ticker)
            if history is None:
                history = deque(maxlen=self._history_maxlen)
                self._history[ticker] = history
            history.append((ts, rounded))

            self._version += 1
            return update

    # --- Read path ---

    def get(self, ticker: str) -> PriceUpdate | None:
        with self._lock:
            return self._prices.get(ticker)

    def get_price(self, ticker: str) -> float | None:
        """Just the price float, or None if the ticker is not tracked."""
        with self._lock:
            update = self._prices.get(ticker)
            return update.price if update else None

    def get_all(self) -> dict[str, PriceUpdate]:
        """Shallow copy of every tracked ticker's latest update."""
        with self._lock:
            return dict(self._prices)

    def get_open_price(self, ticker: str) -> float | None:
        with self._lock:
            return self._open.get(ticker)

    def get_history(self, ticker: str) -> list[dict] | None:
        """Recent points, oldest first, ISO-Z timestamps. None if not tracked.

        `None` (not tracked) and `[]` (tracked, no points yet) are different answers:
        the route turns the former into a 404 and the latter into an empty array.
        """
        with self._lock:
            history = self._history.get(ticker)
            if history is None:
                return None
            return [{"timestamp": to_iso_z(ts), "price": price} for ts, price in history]

    # --- Maintenance ---

    def remove(self, ticker: str) -> None:
        """Drop a ticker entirely — latest price, open price, and history."""
        with self._lock:
            self._prices.pop(ticker, None)
            self._open.pop(ticker, None)
            self._history.pop(ticker, None)

    @property
    def version(self) -> int:
        """Monotonic counter, incremented on every update. SSE change detection."""
        with self._lock:
            return self._version

    def __len__(self) -> int:
        with self._lock:
            return len(self._prices)

    def __contains__(self, ticker: str) -> bool:
        with self._lock:
            return ticker in self._prices
```

### Why a version counter

The SSE generator wakes every 500 ms but must not re-send an unchanged payload (the
simulator can be idle if the watchlist is empty; the Massive poller only writes every
15 s while the generator ticks 30 times in that window). Comparing an integer is O(1);
comparing payloads would mean serializing on every wake.

```python
current = cache.version
if current != last_version:       # Something changed since the last send
    last_version = current
    yield f"data: {json.dumps(...)}\n\n"
```

The counter is read under the lock. On CPython an `int` read is atomic today, but the
lock costs nothing measurable at this frequency and keeps the class correct on a
free-threaded build.

### Memory bound

Per ticker: one `PriceUpdate` (~5 slots) plus 600 two-tuples of floats ≈ 40 KB. At 50
tickers that is ~2 MB — bounded, no eviction policy needed. `deque(maxlen=...)` drops
the oldest point in O(1) on append; there is no scan or compaction.

> **Sidebar — the `open_price` parameter.** The pollers do **not** pass it; PLAN §6
> defines session open as the first price seen. It exists so that a future Massive
> refinement can pass `snap.day.open` and get a true exchange-day change percent
> without touching the cache's shape. Leave it unused unless the spec changes.

---

## 6. Ticker Validation

Both HTTP routes and both data sources need the same normalization, so it lives in one
framework-free module and the route layer maps the exceptions to status codes.

```python
# app/market/validation.py
from __future__ import annotations

import re

TICKER_PATTERN = re.compile(r"^[A-Z]{1,5}$")


class InvalidTickerError(ValueError):
    """Symbol is malformed — does not match ^[A-Z]{1,5}$ after upper-casing. -> 422"""


class UnknownTickerError(LookupError):
    """Symbol is well-formed but the upstream provider does not recognise it. -> 404"""


def normalize_ticker(raw: str) -> str:
    """Strip, upper-case, and validate. Raises InvalidTickerError on malformed input.

    >>> normalize_ticker("  aapl ")
    'AAPL'
    >>> normalize_ticker("BRK.B")
    Traceback (most recent call last):
    InvalidTickerError: Invalid ticker symbol: 'BRK.B'
    """
    ticker = (raw or "").strip().upper()
    if not TICKER_PATTERN.match(ticker):
        raise InvalidTickerError(f"Invalid ticker symbol: {raw!r}")
    return ticker
```

Mapping at the route layer (used by the watchlist router, the trade endpoint, and the
history endpoint alike):

```python
from fastapi import HTTPException

from app.market.validation import InvalidTickerError, UnknownTickerError, normalize_ticker


def require_ticker(raw: str) -> str:
    try:
        return normalize_ticker(raw)
    except InvalidTickerError as exc:
        raise HTTPException(status_code=422, detail=str(exc)) from exc
```

| Input | Result |
|---|---|
| `"aapl"`, `"  AAPL  "` | `"AAPL"` |
| `"NVDA"` | `"NVDA"` |
| `""`, `"TOOLONG"`, `"BRK.B"`, `"12"` | `InvalidTickerError` → `422` |
| `"ZZZZZ"` in simulator mode | Accepted; random seed price in $20–$500 |
| `"ZZZZZ"` in Massive mode | `UnknownTickerError` → `404` |

Note the seed range for unknown tickers is **$20–$500** per PLAN §6, wider than the
$50–$300 in the archived simulator note. PLAN wins.

---

## 7. Abstract Interface — `MarketDataSource`

```python
# app/market/interface.py
from __future__ import annotations

from abc import ABC, abstractmethod


class MarketDataSource(ABC):
    """Contract for market data providers.

    Implementations push updates into a shared PriceCache on their own schedule.
    Downstream code never calls a data source for prices — it reads the cache.

    Lifecycle:
        source = create_market_data_source(cache)
        await source.start(["AAPL", "GOOGL"])   # exactly once
        await source.add_ticker("TSLA")          # any number of times
        await source.remove_ticker("GOOGL")
        await source.stop()                      # idempotent
    """

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Seed the cache and start the background task. Call exactly once."""

    @abstractmethod
    async def stop(self) -> None:
        """Cancel the background task and release resources. Safe to call twice."""

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Add to the active set. No-op if present.

        Raises UnknownTickerError if the provider rejects the symbol (Massive only).
        Callers must normalize the symbol first.
        """

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Remove from the active set and from the cache. No-op if absent."""

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Currently tracked symbols. Synchronous — callers poll this freely."""
```

### Why push-to-cache instead of returning prices

A `get_price()` on the interface would mean either a blocking network call on the
request path (Massive) or a source-specific cache anyway. Pushing into a shared cache
gives one read path with identical latency characteristics for both sources, and lets
the SSE generator poll a local dict instead of coordinating with a producer.

`get_tickers()` is deliberately synchronous — it reads a local list, and making it async
would force every caller (including the health endpoint) into an await.

---

## 8. Seed Prices and Ticker Parameters

Pure constants, no logic. Unchanged from the as-built code except for the unknown-ticker
range note above.

```python
# app/market/seed_prices.py

SEED_PRICES: dict[str, float] = {
    "AAPL": 190.00, "GOOGL": 175.00, "MSFT": 420.00, "AMZN": 185.00, "TSLA": 250.00,
    "NVDA": 800.00, "META": 500.00, "JPM": 195.00,  "V": 280.00,    "NFLX": 600.00,
}

# sigma: annualized volatility; mu: annualized drift
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

DEFAULT_PARAMS: dict[str, float] = {"sigma": 0.25, "mu": 0.05}

# Seed range for tickers with no entry in SEED_PRICES (PLAN §6)
UNKNOWN_PRICE_RANGE: tuple[float, float] = (20.0, 500.0)

CORRELATION_GROUPS: dict[str, set[str]] = {
    "tech": {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}

INTRA_TECH_CORR = 0.6     # Tech names move together
INTRA_FINANCE_CORR = 0.5  # Financials move together
CROSS_GROUP_CORR = 0.3    # Between sectors, and the fallback for unknown tickers
TSLA_CORR = 0.3           # TSLA does its own thing
```

The parameter spread is what makes the demo legible: side by side, TSLA visibly jitters
while V barely moves, because `sigma` differs by 3×.

---

## 9. Simulator

### 9.1 The math

Each tick advances every price by one Geometric Brownian Motion step:

```
S(t+dt) = S(t) · exp( (mu − sigma²/2)·dt  +  sigma·√dt·Z )
```

- `mu` — annualized drift. `sigma` — annualized volatility.
- `dt` — the tick as a fraction of a trading year:
  `0.5 s / (252 days × 6.5 h × 3600 s) = 0.5 / 5,896,800 ≈ 8.48e-8`
- `Z` — a standard normal draw, correlated across tickers (§9.2).

GBM is multiplicative and `exp()` is strictly positive, so a simulated price can never
go zero or negative — no clamping needed. The `−sigma²/2` term is the Itô correction; it
makes `mu` the expected *log* return, which is why a high-sigma ticker like TSLA does not
drift upward faster than its `mu` implies.

At this `dt`, a single tick moves AAPL by roughly `190 × 0.22 × √8.48e-8 ≈ $0.012` — about
a cent, which is exactly the texture a live tape should have.

### 9.2 Correlated moves via Cholesky

Independent draws look wrong: real sectors move together, and a watchlist where every
row flashes a different direction every tick reads as noise. Given correlation matrix
`C`, its Cholesky factor `L` (where `L·Lᵀ = C`) turns independent normals into
correlated ones:

```
Z_correlated = L @ Z_independent
```

The matrix is rebuilt on every add/remove — O(n²) to build, O(n³) to factor, with n < 50
and changes only on watchlist edits.

### 9.3 `GBMSimulator`

```python
# app/market/simulator.py
from __future__ import annotations

import logging
import math
import random

import numpy as np

from .seed_prices import (
    CORRELATION_GROUPS, CROSS_GROUP_CORR, DEFAULT_PARAMS, INTRA_FINANCE_CORR,
    INTRA_TECH_CORR, SEED_PRICES, TICKER_PARAMS, TSLA_CORR, UNKNOWN_PRICE_RANGE,
)

logger = logging.getLogger(__name__)


class GBMSimulator:
    """Correlated Geometric Brownian Motion price generator.

    Holds unrounded internal prices; step() returns them rounded to 2 dp. Rounding
    the internal state instead would let error accumulate over thousands of ticks.
    """

    TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # 5,896,800
    DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR  # ~8.48e-8

    def __init__(
        self,
        tickers: list[str],
        dt: float = DEFAULT_DT,
        event_probability: float = 0.001,
    ) -> None:
        self._dt = dt
        self._event_prob = event_probability
        self._tickers: list[str] = []
        self._prices: dict[str, float] = {}          # Unrounded
        self._params: dict[str, dict[str, float]] = {}
        self._cholesky: np.ndarray | None = None

        for ticker in tickers:
            self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    # --- Public API ---

    def step(self) -> dict[str, float]:
        """Advance every ticker one tick. Returns {ticker: price rounded to 2 dp}.

        Hot path — runs twice a second. Everything here is O(n) plus one n×n matvec.
        """
        n = len(self._tickers)
        if n == 0:
            return {}

        z = np.random.standard_normal(n)
        if self._cholesky is not None:
            z = self._cholesky @ z

        result: dict[str, float] = {}
        for i, ticker in enumerate(self._tickers):
            params = self._params[ticker]
            mu, sigma = params["mu"], params["sigma"]

            drift = (mu - 0.5 * sigma**2) * self._dt
            diffusion = sigma * math.sqrt(self._dt) * z[i]
            self._prices[ticker] *= math.exp(drift + diffusion)

            # Random shock: ~0.1% per ticker per tick. With 10 tickers at 2 ticks/s,
            # something notable happens roughly every 50 seconds.
            if random.random() < self._event_prob:
                magnitude = random.uniform(0.02, 0.05)
                sign = random.choice([-1, 1])
                self._prices[ticker] *= 1 + magnitude * sign
                logger.debug(
                    "Shock on %s: %.1f%% %s",
                    ticker, magnitude * 100, "up" if sign > 0 else "down",
                )

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
        price = self._prices.get(ticker)
        return round(price, 2) if price is not None else None

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    # --- Internals ---

    def _add_ticker_internal(self, ticker: str) -> None:
        """Add without rebuilding Cholesky — used for batch init."""
        if ticker in self._prices:
            return
        self._tickers.append(ticker)
        self._prices[ticker] = SEED_PRICES.get(ticker) or random.uniform(*UNKNOWN_PRICE_RANGE)
        self._params[ticker] = dict(TICKER_PARAMS.get(ticker, DEFAULT_PARAMS))

    def _rebuild_cholesky(self) -> None:
        n = len(self._tickers)
        if n <= 1:
            self._cholesky = None
            return

        corr = np.eye(n)
        for i in range(n):
            for j in range(i + 1, n):
                rho = self._pairwise_correlation(self._tickers[i], self._tickers[j])
                corr[i, j] = corr[j, i] = rho

        try:
            self._cholesky = np.linalg.cholesky(corr)
        except np.linalg.LinAlgError:
            # Defensive: a correlation structure that is not positive semi-definite
            # would break the factorization. Fall back to independent draws rather
            # than killing the loop.
            logger.warning("Correlation matrix not PSD for %d tickers; using independent draws", n)
            self._cholesky = None

    @staticmethod
    def _pairwise_correlation(t1: str, t2: str) -> float:
        """Sector-based pairwise correlation.

        TSLA first: it is in the tech set but is checked before the tech rule so it
        stays at 0.3 with everything, including other tech names.
        """
        if t1 == "TSLA" or t2 == "TSLA":
            return TSLA_CORR

        tech = CORRELATION_GROUPS["tech"]
        finance = CORRELATION_GROUPS["finance"]

        if t1 in tech and t2 in tech:
            return INTRA_TECH_CORR
        if t1 in finance and t2 in finance:
            return INTRA_FINANCE_CORR
        return CROSS_GROUP_CORR
```

The `try/except LinAlgError` is the one addition to the as-built math: with the current
constants the matrix is always positive semi-definite, but a future correlation edit
should degrade to independent draws rather than crash the loop on every rebuild.

### 9.4 `SimulatorDataSource`

The async wrapper. Its only jobs are seeding the cache immediately (so the first SSE
event is never empty), running the loop, and staying alive through errors.

```python
# app/market/simulator.py (continued)
import asyncio

from .cache import PriceCache
from .interface import MarketDataSource


class SimulatorDataSource(MarketDataSource):
    """MarketDataSource backed by GBMSimulator. Ticks every `update_interval` seconds."""

    def __init__(
        self,
        price_cache: PriceCache,
        update_interval: float = 0.5,
        event_probability: float = 0.001,
    ) -> None:
        self._cache = price_cache
        self._interval = update_interval
        self._event_prob = event_probability
        self._sim: GBMSimulator | None = None
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers=tickers, event_probability=self._event_prob)

        # Seed the cache before the first tick: this establishes each ticker's session
        # open price and guarantees the first SSE event carries real data.
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
        """Any well-formed symbol is accepted — the simulator invents a price for it."""
        if not self._sim:
            return
        self._sim.add_ticker(ticker)
        price = self._sim.get_price(ticker)
        if price is not None:
            self._cache.update(ticker=ticker, price=price)   # Sets session open
        logger.info("Simulator: added %s", ticker)

    async def remove_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.remove_ticker(ticker)
        self._cache.remove(ticker)
        logger.info("Simulator: removed %s", ticker)

    def get_tickers(self) -> list[str]:
        return self._sim.get_tickers() if self._sim else []

    async def _run_loop(self) -> None:
        while True:
            try:
                if self._sim:
                    for ticker, price in self._sim.step().items():
                        self._cache.update(ticker=ticker, price=price)
            except Exception:
                # Never let one bad tick kill the stream for the rest of the session.
                logger.exception("Simulator step failed")
            await asyncio.sleep(self._interval)
```

Behaviors worth stating explicitly:

- **Re-adding a removed ticker resets its session open.** `remove_ticker` drops the
  cache entry including `_open`; the next `add_ticker` seeds a fresh open price at a new
  random or seed value. Day change restarts at 0% — acceptable, and the alternative
  (retaining opens for untracked tickers) leaks memory over a long session.
- **The loop sleeps after work**, so the interval is "at least 500 ms", not exactly. Tick
  timing is not a contract; the SSE generator rate-limits independently.
- **Cancellation is awaited** in `stop()` so shutdown does not race with a half-finished
  `update()` write.

---

## 10. Massive API Client

### 10.1 Why one snapshot call

`GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=...` returns every requested
ticker in a single response. On the free tier (5 requests/minute) per-ticker calls would
exhaust the budget with two tickers; one batched call per 15 s uses 4 requests/minute for
the whole watchlist.

| Tier | Limit | Poll interval |
|---|---|---|
| Free | 5 req/min | 15 s (default) |
| Paid | effectively unlimited | 2–5 s via `MASSIVE_POLL_INTERVAL` |

Fields consumed per snapshot: `ticker`, `last_trade.price`, `last_trade.timestamp`
(Unix **milliseconds** — divide by 1000).

### 10.2 `MassiveDataSource`

```python
# app/market/massive_client.py
from __future__ import annotations

import asyncio
import logging

from massive import RESTClient
from massive.rest.models import SnapshotMarketType

from .cache import PriceCache
from .interface import MarketDataSource
from .validation import UnknownTickerError

logger = logging.getLogger(__name__)


class MassiveDataSource(MarketDataSource):
    """MarketDataSource backed by the Massive (Polygon.io) REST API.

    Polls the batched stocks snapshot endpoint for every tracked ticker in one call,
    then writes results into the PriceCache. The RESTClient is synchronous, so every
    call runs through asyncio.to_thread to keep the event loop free.
    """

    def __init__(
        self,
        api_key: str,
        price_cache: PriceCache,
        poll_interval: float = 15.0,
    ) -> None:
        self._api_key = api_key
        self._cache = price_cache
        self._interval = poll_interval
        self._tickers: list[str] = []
        self._task: asyncio.Task | None = None
        self._client: RESTClient | None = None

    async def start(self, tickers: list[str]) -> None:
        self._client = RESTClient(api_key=self._api_key)
        self._tickers = list(tickers)

        await self._poll_once()   # Immediate first poll: cache is warm before serving
        self._task = asyncio.create_task(self._poll_loop(), name="massive-poller")
        logger.info(
            "Massive poller started: %d tickers, %.1fs interval", len(tickers), self._interval
        )

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
        """Validate against the API, then track.

        Raises UnknownTickerError (-> 404) if Massive does not know the symbol. The
        simulator accepts anything well-formed; Massive cannot invent a price.
        """
        if ticker in self._tickers:
            return

        if not await self._validate_ticker(ticker):
            raise UnknownTickerError(f"Unknown ticker symbol: {ticker}")

        self._tickers.append(ticker)
        await self._poll_once()   # Don't make the user wait up to 15s for a first price
        logger.info("Massive: added %s", ticker)

    async def remove_ticker(self, ticker: str) -> None:
        self._tickers = [t for t in self._tickers if t != ticker]
        self._cache.remove(ticker)
        logger.info("Massive: removed %s", ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    # --- Internals ---

    async def _poll_loop(self) -> None:
        """Poll on interval. The first poll already happened in start()."""
        while True:
            await asyncio.sleep(self._interval)
            await self._poll_once()

    async def _poll_once(self) -> None:
        """One poll cycle: fetch snapshots, write the cache. Never raises."""
        if not self._tickers or not self._client:
            return

        try:
            snapshots = await asyncio.to_thread(self._fetch_snapshots)
        except Exception as exc:
            # 401 bad key, 429 rate limit, network error, provider 5xx. Log and retry
            # on the next interval — a transient upstream failure must not stop the app.
            logger.error("Massive poll failed: %s", exc)
            return

        processed = 0
        for snap in snapshots:
            try:
                price = snap.last_trade.price
                timestamp = snap.last_trade.timestamp / 1000.0   # ms -> seconds
            except (AttributeError, TypeError) as exc:
                # A halted or newly listed ticker can come back without a last trade.
                logger.warning("Skipping snapshot for %s: %s", getattr(snap, "ticker", "???"), exc)
                continue
            self._cache.update(ticker=snap.ticker, price=price, timestamp=timestamp)
            processed += 1

        logger.debug("Massive poll: updated %d/%d tickers", processed, len(self._tickers))

    def _fetch_snapshots(self) -> list:
        """Synchronous REST call. Runs in a worker thread."""
        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS,
            tickers=self._tickers,
        )

    async def _validate_ticker(self, ticker: str) -> bool:
        """True if Massive recognises the symbol. Used by add_ticker() only."""
        if not self._client:
            return False
        try:
            snapshot = await asyncio.to_thread(
                self._client.get_snapshot_ticker,
                market_type=SnapshotMarketType.STOCKS,
                ticker=ticker,
            )
        except Exception as exc:
            logger.warning("Ticker validation failed for %s: %s", ticker, exc)
            return False
        return snapshot is not None and getattr(snapshot, "ticker", None) is not None
```

### 10.3 Two different error philosophies, on purpose

| Path | On failure | Why |
|---|---|---|
| `_poll_once()` (background) | Log, keep the previous cached prices, retry next interval | Nobody is waiting on it. Stale prices beat a dead app; a 429 self-heals in a minute. |
| `add_ticker()` (request path) | Raise `UnknownTickerError` → `404` | A user is waiting on an answer and needs to know the symbol was rejected. |

A validation call that fails for a *transport* reason (429, network) returns `False` and
therefore surfaces as `404 Unknown ticker`, which is technically imprecise. That is the
deliberate trade: on the free tier a rate-limited validation is far more likely to be a
genuinely unknown symbol typed by a user than a real outage, and a `404` with a clear
message is more actionable than a `502` the user cannot act on. The log line records the
real cause.

### 10.4 Timestamps and market hours

- Massive timestamps are Unix **milliseconds**; the cache stores seconds. The `/1000.0`
  is the single conversion point.
- Outside market hours `last_trade.price` is the last traded price, so it repeats between
  polls. The cache still bumps `version` on each write, so the SSE generator re-sends —
  harmless, and `direction` correctly reports `"flat"`.
- Session open in Massive mode is the first price the poller saw, per PLAN §6 — not the
  exchange's opening print. See the sidebar in §5 for the refinement path.

---

## 11. Factory

Source selection happens in exactly one place, read from the environment at startup.

```python
# app/market/factory.py
from __future__ import annotations

import logging
import os

from .cache import PriceCache
from .interface import MarketDataSource
from .massive_client import MassiveDataSource
from .simulator import SimulatorDataSource

logger = logging.getLogger(__name__)

DEFAULT_MASSIVE_POLL_INTERVAL = 15.0


def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    """Select a data source from the environment. Returns an *unstarted* source.

    MASSIVE_API_KEY set and non-empty  -> MassiveDataSource (real data)
    otherwise                          -> SimulatorDataSource (GBM)

    Optional: MASSIVE_POLL_INTERVAL (seconds, default 15.0).
    """
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()

    if not api_key:
        logger.info("Market data source: GBM simulator")
        return SimulatorDataSource(price_cache=price_cache)

    try:
        interval = float(os.environ.get("MASSIVE_POLL_INTERVAL", DEFAULT_MASSIVE_POLL_INTERVAL))
    except ValueError:
        logger.warning("Invalid MASSIVE_POLL_INTERVAL; falling back to %.1fs",
                       DEFAULT_MASSIVE_POLL_INTERVAL)
        interval = DEFAULT_MASSIVE_POLL_INTERVAL

    logger.info("Market data source: Massive API (%.1fs poll)", interval)
    return MassiveDataSource(api_key=api_key, price_cache=price_cache, poll_interval=interval)


def active_source_name() -> str:
    """'massive' or 'simulator' — for GET /api/health."""
    return "massive" if os.environ.get("MASSIVE_API_KEY", "").strip() else "simulator"
```

Whitespace-only keys count as absent: a `.env` containing `MASSIVE_API_KEY=` must fall
back to the simulator rather than starting a poller that 401s forever.

Note `massive` is a **core** dependency in `pyproject.toml`, so the import is top-level.
The earlier lazy-import approach made the module's names unpatchable in tests (see
`planning/archive/MARKET_DATA_REVIEW.md` §3.2) and bought nothing.

---

## 12. HTTP Surface — SSE Stream and History

Both routers are built by factories that create their own `APIRouter` instance, so
calling a factory twice (as tests do) cannot double-register a route.

### 12.1 SSE stream

```python
# app/market/stream.py
from __future__ import annotations

import asyncio
import json
import logging
from collections.abc import AsyncGenerator

from fastapi import APIRouter, HTTPException, Request
from fastapi.responses import StreamingResponse

from .cache import PriceCache
from .validation import InvalidTickerError, normalize_ticker

logger = logging.getLogger(__name__)

STREAM_INTERVAL = 0.5   # Seconds between wakes
HEARTBEAT_AFTER = 15.0  # Seconds of no change before sending a keep-alive comment


def create_stream_router(price_cache: PriceCache) -> APIRouter:
    """Build the SSE router. A fresh APIRouter per call — safe to call in tests."""
    router = APIRouter(prefix="/api/stream", tags=["streaming"])

    @router.get("/prices")
    async def stream_prices(request: Request) -> StreamingResponse:
        return StreamingResponse(
            _generate_events(price_cache, request),
            media_type="text/event-stream",
            headers={
                "Cache-Control": "no-cache",
                "Connection": "keep-alive",
                "X-Accel-Buffering": "no",   # Defeat nginx response buffering if proxied
            },
        )

    return router


async def _generate_events(
    price_cache: PriceCache,
    request: Request,
    interval: float = STREAM_INTERVAL,
) -> AsyncGenerator[str, None]:
    """Yield one batched SSE event per cache change, at most every `interval` seconds."""
    yield "retry: 1000\n\n"   # Browser reconnects 1s after a drop (PLAN §6)

    last_version = -1
    silent_for = 0.0
    client = request.client.host if request.client else "unknown"
    logger.info("SSE client connected: %s", client)

    try:
        while True:
            if await request.is_disconnected():
                logger.info("SSE client disconnected: %s", client)
                break

            version = price_cache.version
            if version != last_version:
                last_version = version
                prices = price_cache.get_all()
                if prices:
                    payload = json.dumps({t: u.to_dict() for t, u in prices.items()})
                    yield f"data: {payload}\n\n"
                    silent_for = 0.0
            else:
                silent_for += interval
                if silent_for >= HEARTBEAT_AFTER:
                    # A comment line keeps intermediaries from timing the connection out
                    # during a quiet Massive interval. EventSource ignores comments.
                    yield ": keep-alive\n\n"
                    silent_for = 0.0

            await asyncio.sleep(interval)
    except asyncio.CancelledError:
        logger.info("SSE stream cancelled: %s", client)
        raise
```

Wire format — one event carrying **every** tracked ticker, not one event per ticker
(PLAN §6, Decision #1):

```
retry: 1000

data: {"AAPL":{"ticker":"AAPL","price":190.52,"previous_price":190.48,"open_price":189.90,"change":0.04,"change_percent":0.021,"day_change_percent":0.3265,"direction":"up","timestamp":"2026-09-11T14:03:22.512Z"},"GOOGL":{...}}

data: {"AAPL":{...},"GOOGL":{...}}

: keep-alive

```

The frontend treats each event as an atomic snapshot: update every ticker present, flash
the ones whose `direction` is `up`/`down`, append each price to that ticker's sparkline
buffer, and recompute portfolio value.

**Why poll-and-push rather than event-driven.** A pub/sub fan-out from the cache to N
generators would need per-client queues and backpressure handling. Polling a local dict
every 500 ms costs a dict copy per client per tick, the version check skips the
serialization entirely when nothing changed, and a slow client simply misses intermediate
states instead of growing an unbounded queue — the right failure mode for a live tape,
where only the latest price matters.

### 12.2 History endpoint

```python
# app/market/stream.py (continued)

def create_prices_router(price_cache: PriceCache) -> APIRouter:
    """REST router for price history. GET /api/prices/{ticker}/history."""
    router = APIRouter(prefix="/api/prices", tags=["prices"])

    @router.get("/{ticker}/history")
    async def get_price_history(ticker: str) -> dict:
        """Up to 600 recent points, oldest first.

        422 if the symbol is malformed; 404 if it is not currently tracked.
        """
        try:
            symbol = normalize_ticker(ticker)
        except InvalidTickerError as exc:
            raise HTTPException(status_code=422, detail=str(exc)) from exc

        history = price_cache.get_history(symbol)
        if history is None:
            raise HTTPException(status_code=404, detail=f"Ticker not tracked: {symbol}")

        return {"ticker": symbol, "points": history}

    return router
```

Response:

```json
{
  "ticker": "AAPL",
  "points": [
    {"timestamp": "2026-09-11T13:58:22.011Z", "price": 189.90},
    {"timestamp": "2026-09-11T13:58:22.512Z", "price": 189.94}
  ]
}
```

The frontend fetches this once when a ticker is selected, seeds the Lightweight Charts
series with `points`, and appends live SSE ticks from there. That is why the main chart
is never empty on selection (PLAN §10).

---

## 13. App Lifecycle Integration

```python
# app/main.py (market-data portions only)
from __future__ import annotations

import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI

from app.db import get_open_position_tickers, get_watchlist_tickers, init_db
from app.market import (
    PriceCache, active_source_name, create_market_data_source,
    create_prices_router, create_stream_router,
)

logger = logging.getLogger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    init_db()   # Lazy schema creation + seed (PLAN §7)

    cache = PriceCache()
    source = create_market_data_source(cache)

    # Track the union of the watchlist and every open position (PLAN §6).
    # A held ticker must keep streaming even if it was removed from the watchlist,
    # otherwise the portfolio cannot be valued.
    tickers = sorted(set(get_watchlist_tickers()) | set(get_open_position_tickers()))
    await source.start(tickers)

    app.state.price_cache = cache
    app.state.market_source = source
    logger.info("Market data ready: %s, %d tickers", active_source_name(), len(tickers))

    try:
        yield
    finally:
        await source.stop()


app = FastAPI(title="FinAlly", lifespan=lifespan)

# Register API routers BEFORE mounting static files at "/" (PLAN §11), or the
# static mount shadows every API route.
_cache = PriceCache()          # Replaced by the lifespan instance below; see note
app.include_router(create_stream_router(_cache))
app.include_router(create_prices_router(_cache))
```

> **Note on router wiring.** Routers are built at import time but the cache is created in
> `lifespan`. Build the `PriceCache` at module scope and hand the *same* instance to both
> the routers and the lifespan (`cache = PriceCache()` as a module-level singleton), or
> have the route handlers resolve `request.app.state.price_cache` instead of closing over
> the instance. The first is simpler; pick one and be consistent. Do **not** create two
> caches — half the app would read an empty one.

Accessing prices from any other route:

```python
from fastapi import Depends, Request

from app.market import PriceCache


def get_price_cache(request: Request) -> PriceCache:
    return request.app.state.price_cache


@router.post("/api/portfolio/trade")
async def execute_trade(body: TradeRequest, cache: PriceCache = Depends(get_price_cache)):
    price = cache.get_price(body.ticker)
    if price is None:
        raise HTTPException(status_code=409, detail=f"No price available for {body.ticker}")
    ...
```

Health endpoint:

```python
@app.get("/api/health")
async def health(request: Request) -> dict:
    cache: PriceCache = request.app.state.price_cache
    return {
        "status": "ok",
        "data_source": active_source_name(),
        "db": "ok" if db_ok() else "error",
        "tickers": len(cache),
    }
```

---

## 14. Watchlist and Position Coordination

The data source tracks the **union of the watchlist and all open positions**. The
watchlist and portfolio layers own these transitions; the market layer only exposes
`add_ticker` / `remove_ticker`.

### Adding to the watchlist — `POST /api/watchlist`

```python
symbol = require_ticker(body.ticker)          # 422 if malformed

if watchlist_contains(symbol):
    raise HTTPException(409, detail=f"{symbol} is already on your watchlist")

try:
    await source.add_ticker(symbol)           # 404 if Massive rejects it
except UnknownTickerError as exc:
    raise HTTPException(404, detail=str(exc)) from exc

insert_watchlist_row(symbol)                  # Only after the source accepts it
return {"ticker": symbol, "added_at": now_iso_z()}
```

Order matters: validate with the source **before** the DB write, so a symbol Massive
rejects never lands in the watchlist table.

### Removing from the watchlist — `DELETE /api/watchlist/{ticker}`

```python
symbol = require_ticker(ticker)

if not watchlist_contains(symbol):
    raise HTTPException(404, detail=f"{symbol} is not on your watchlist")

delete_watchlist_row(symbol)

# Keep streaming if the ticker is still held — the portfolio must stay valuable.
if not has_open_position(symbol):
    await source.remove_ticker(symbol)
```

### Closing a position

When a sell takes quantity to zero, the position row is deleted. If the ticker is not on
the watchlist, stop tracking it:

```python
if position_closed and not watchlist_contains(symbol):
    await source.remove_ticker(symbol)
```

### The resulting matrix

| On watchlist | Open position | Tracked? | Removal trigger |
|---|---|---|---|
| Yes | Yes | Yes | Neither removal alone stops it |
| Yes | No | Yes | Watchlist delete |
| No | Yes | Yes | Position close |
| No | No | No | — |

**Why removal drops the cache entry.** `remove_ticker` calls `cache.remove()`, clearing
the latest price, session open, and history. The ticker then vanishes from the next SSE
snapshot, and a trade against it returns `409 No price available` — which is the correct
answer for a symbol the platform no longer tracks. It also means re-adding resets day
change to 0%, as noted in §9.4.

---

## 15. Error Handling and Edge Cases

| Case | Behavior | Rationale |
|---|---|---|
| Empty watchlist at startup (all removed, no positions) | `source.start([])` — loop runs, `step()` returns `{}`, no SSE payloads sent | Nothing to stream; adding a ticker later brings it back to life with no restart |
| Trade against an untracked ticker | `get_price()` → `None` → `409 "No price available for X"` | Never invent or reuse a stale price for money math |
| Massive returns a snapshot with no `last_trade` | Warn, skip that ticker, keep the rest of the batch | A halted or newly listed symbol must not void the whole poll |
| Invalid `MASSIVE_API_KEY` | Every poll logs a 401, cache keeps the last values (empty at startup) | App stays up; the frontend shows `—` and `/api/health` still answers |
| Rate limit (429) | Same as any poll failure — log and retry next interval | Self-heals within a minute on the free tier |
| Client disconnects mid-SSE | `request.is_disconnected()` breaks the generator within 500 ms | No leaked tasks per reconnect |
| Two SSE clients | Each gets its own generator and its own `last_version`; both read the same cache | Cache reads are cheap and lock-guarded |
| Simulator raises mid-tick | Caught and logged inside `_run_loop`; the loop continues | One bad tick must not end the session's stream |
| Price rounds to the same value two ticks running | `direction == "flat"`, no flash, cache version still bumps | A sub-cent move on a $800 ticker genuinely is no change at display precision |
| `remove_ticker` for an untracked symbol | No-op in both sources | Idempotent; callers do not pre-check |
| `stop()` called twice | Second call is a no-op | Lifespan teardown can run after an error path |

---

## 16. Testing Strategy

Existing coverage: 73 tests across 6 modules in `backend/tests/market/`. The additions in
this design need the following new tests.

### 16.1 Session open price and day change

```python
# tests/market/test_cache.py
def test_first_update_sets_open_price():
    cache = PriceCache()
    update = cache.update("AAPL", 190.00)

    assert update.open_price == 190.00
    assert update.day_change_percent == 0.0
    assert update.direction == "flat"


def test_day_change_percent_tracks_session_open():
    cache = PriceCache()
    cache.update("AAPL", 200.00)
    update = cache.update("AAPL", 210.00)

    assert update.open_price == 200.00           # Open is sticky
    assert update.day_change_percent == 5.0      # (210-200)/200*100
    assert update.change_percent == 5.0          # Same here: only two ticks
    assert update.direction == "up"


def test_open_price_survives_many_updates():
    cache = PriceCache()
    cache.update("AAPL", 100.00)
    for price in (101.0, 99.0, 105.0, 98.0):
        cache.update("AAPL", price)

    assert cache.get("AAPL").open_price == 100.00
    assert cache.get("AAPL").day_change_percent == -2.0


def test_remove_then_readd_resets_open_price():
    cache = PriceCache()
    cache.update("AAPL", 100.00)
    cache.remove("AAPL")
    update = cache.update("AAPL", 250.00)

    assert update.open_price == 250.00
    assert update.day_change_percent == 0.0
```

### 16.2 History ring buffer

```python
def test_history_records_points_oldest_first():
    cache = PriceCache()
    cache.update("AAPL", 100.00, timestamp=1_000.0)
    cache.update("AAPL", 101.00, timestamp=1_000.5)

    history = cache.get_history("AAPL")
    assert [p["price"] for p in history] == [100.00, 101.00]
    assert history[0]["timestamp"].endswith("Z")


def test_history_caps_at_maxlen():
    cache = PriceCache(history_maxlen=600)
    for i in range(700):
        cache.update("AAPL", 100.00 + i * 0.01)

    history = cache.get_history("AAPL")
    assert len(history) == 600
    assert history[0]["price"] == round(100.00 + 100 * 0.01, 2)   # Oldest 100 dropped


def test_history_none_for_untracked_ticker():
    assert PriceCache().get_history("ZZZZZ") is None     # -> 404, not an empty chart


def test_remove_clears_history():
    cache = PriceCache()
    cache.update("AAPL", 100.00)
    cache.remove("AAPL")
    assert cache.get_history("AAPL") is None
```

### 16.3 Serialization

```python
# tests/market/test_models.py
def test_to_dict_emits_iso_z_timestamp():
    update = PriceUpdate("AAPL", 190.52, 190.48, open_price=189.90, timestamp=1_789_135_402.512)
    payload = update.to_dict()

    assert payload["timestamp"] == "2026-09-11T14:03:22.512Z"
    assert payload["timestamp"].endswith("Z")
    assert "+00:00" not in payload["timestamp"]


def test_to_dict_contains_every_field_the_frontend_reads():
    payload = PriceUpdate("AAPL", 190.52, 190.48, 189.90).to_dict()
    assert set(payload) == {
        "ticker", "price", "previous_price", "open_price", "change",
        "change_percent", "day_change_percent", "direction", "timestamp",
    }


def test_iso_round_trip():
    ts = 1_789_135_402.512
    assert from_iso_z(to_iso_z(ts)) == pytest.approx(ts, abs=0.001)
```

### 16.4 Validation

```python
# tests/market/test_validation.py
import pytest

from app.market.validation import InvalidTickerError, normalize_ticker


@pytest.mark.parametrize("raw,expected", [("aapl", "AAPL"), ("  msft  ", "MSFT"), ("V", "V")])
def test_normalize_accepts_well_formed(raw, expected):
    assert normalize_ticker(raw) == expected


@pytest.mark.parametrize("raw", ["", "   ", "TOOLONG", "BRK.B", "12", "AA PL", None])
def test_normalize_rejects_malformed(raw):
    with pytest.raises(InvalidTickerError):
        normalize_ticker(raw)
```

### 16.5 Simulator

```python
# tests/market/test_simulator.py
def test_prices_stay_positive_over_many_steps():
    sim = GBMSimulator(["AAPL"], event_probability=0.5)   # Shock-heavy
    for _ in range(5_000):
        assert all(p > 0 for p in sim.step().values())


def test_cholesky_builds_for_full_default_watchlist():
    sim = GBMSimulator(list(SEED_PRICES))
    prices = sim.step()
    assert len(prices) == 10
    assert sim._cholesky.shape == (10, 10)


def test_unknown_ticker_gets_seed_in_plan_range():
    sim = GBMSimulator(["ZZZZZ"])
    assert 20.0 <= sim.get_price("ZZZZZ") <= 500.0


def test_internal_state_is_unrounded():
    """Rounding internal state would accumulate drift across thousands of ticks."""
    sim = GBMSimulator(["AAPL"])
    sim.step()
    assert sim._prices["AAPL"] != round(sim._prices["AAPL"], 2)  # Almost surely
```

### 16.6 SSE integration

```python
# tests/market/test_stream.py
import json

import httpx
from fastapi import FastAPI

from app.market import PriceCache, create_stream_router


async def test_sse_emits_batched_snapshot():
    cache = PriceCache()
    cache.update("AAPL", 190.00)
    cache.update("GOOGL", 175.00)

    app = FastAPI()
    app.include_router(create_stream_router(cache))

    transport = httpx.ASGITransport(app=app)
    async with httpx.AsyncClient(transport=transport, base_url="http://test") as client:
        async with client.stream("GET", "/api/stream/prices") as response:
            assert response.headers["content-type"].startswith("text/event-stream")

            chunks = []
            async for line in response.aiter_lines():
                chunks.append(line)
                if line.startswith("data: "):
                    break

    assert "retry: 1000" in chunks[0]
    payload = json.loads(chunks[-1].removeprefix("data: "))
    assert set(payload) == {"AAPL", "GOOGL"}          # One event, all tickers
    assert payload["AAPL"]["open_price"] == 190.00
```

### 16.7 Massive, mocked

The `massive` package is a core dependency, so `RESTClient` exists at module level and
patches resolve without `create=True`.

```python
# tests/market/test_massive.py
from types import SimpleNamespace
from unittest.mock import MagicMock, patch


def _snapshot(ticker, price, ts_ms):
    return SimpleNamespace(ticker=ticker, last_trade=SimpleNamespace(price=price, timestamp=ts_ms))


async def test_poll_converts_ms_timestamps_to_seconds():
    cache = PriceCache()
    source = MassiveDataSource(api_key="k", price_cache=cache)
    source._client = MagicMock()
    source._tickers = ["AAPL"]
    source._fetch_snapshots = MagicMock(return_value=[_snapshot("AAPL", 190.52, 1_789_135_402_512)])

    await source._poll_once()

    assert cache.get_price("AAPL") == 190.52
    assert cache.get("AAPL").timestamp == 1_789_135_402.512


async def test_snapshot_without_last_trade_is_skipped_not_fatal():
    cache = PriceCache()
    source = MassiveDataSource(api_key="k", price_cache=cache)
    source._client = MagicMock()
    source._tickers = ["AAPL", "GOOGL"]
    source._fetch_snapshots = MagicMock(return_value=[
        SimpleNamespace(ticker="AAPL", last_trade=None),
        _snapshot("GOOGL", 175.00, 1_789_135_402_512),
    ])

    await source._poll_once()

    assert cache.get_price("AAPL") is None
    assert cache.get_price("GOOGL") == 175.00     # Batch survives one bad entry


async def test_poll_failure_keeps_previous_prices():
    cache = PriceCache()
    cache.update("AAPL", 190.00)
    source = MassiveDataSource(api_key="k", price_cache=cache)
    source._client = MagicMock()
    source._tickers = ["AAPL"]
    source._fetch_snapshots = MagicMock(side_effect=RuntimeError("429 rate limit"))

    await source._poll_once()      # Must not raise

    assert cache.get_price("AAPL") == 190.00


async def test_add_unknown_ticker_raises():
    source = MassiveDataSource(api_key="k", price_cache=PriceCache())
    source._client = MagicMock()
    with patch.object(source, "_validate_ticker", return_value=False):
        with pytest.raises(UnknownTickerError):
            await source.add_ticker("ZZZZZ")
```

### 16.8 Thread safety

```python
def test_concurrent_writes_do_not_lose_updates():
    cache = PriceCache()
    tickers = [f"T{i}" for i in range(10)]

    def writer(ticker):
        for i in range(500):
            cache.update(ticker, 100.0 + i * 0.01)

    with ThreadPoolExecutor(max_workers=10) as pool:
        list(pool.map(writer, tickers))

    assert cache.version == 5_000
    assert len(cache) == 10
    assert all(len(cache.get_history(t)) == 500 for t in tickers)
```

This is the test that would catch a regression if someone dropped the lock from
`update()` — the Massive path writes from a `to_thread` worker while the SSE generator
reads from the event loop.

---

## 17. Configuration Summary

### Environment variables

| Variable | Default | Effect |
|---|---|---|
| `MASSIVE_API_KEY` | unset | Non-empty → Massive REST poller; empty/unset → GBM simulator |
| `MASSIVE_POLL_INTERVAL` | `15.0` | Seconds between snapshot polls. Free tier: keep ≥ 12 s |

### Tunable constants

| Constant | Module | Value | Meaning |
|---|---|---|---|
| `HISTORY_MAXLEN` | `cache.py` | 600 | Ring buffer depth ≈ 5 min at 500 ms |
| `STREAM_INTERVAL` | `stream.py` | 0.5 | SSE wake interval |
| `HEARTBEAT_AFTER` | `stream.py` | 15.0 | Quiet seconds before a keep-alive comment |
| `DEFAULT_DT` | `simulator.py` | ~8.48e-8 | 500 ms as a fraction of a trading year |
| `event_probability` | `simulator.py` | 0.001 | Shock chance per ticker per tick |
| `update_interval` | `simulator.py` | 0.5 | Simulator tick interval |
| `UNKNOWN_PRICE_RANGE` | `seed_prices.py` | (20, 500) | Seed range for unseeded tickers |

### Public API — `app/market/__init__.py`

```python
"""Market data subsystem for FinAlly.

Everything downstream imports from here; no module outside this package should
import app.market.simulator or app.market.massive_client directly.
"""

from .cache import HISTORY_MAXLEN, PriceCache
from .factory import active_source_name, create_market_data_source
from .interface import MarketDataSource
from .models import PriceUpdate, from_iso_z, to_iso_z
from .stream import create_prices_router, create_stream_router
from .validation import InvalidTickerError, UnknownTickerError, normalize_ticker

__all__ = [
    "PriceUpdate", "PriceCache", "HISTORY_MAXLEN",
    "MarketDataSource", "create_market_data_source", "active_source_name",
    "create_stream_router", "create_prices_router",
    "normalize_ticker", "InvalidTickerError", "UnknownTickerError",
    "to_iso_z", "from_iso_z",
]
```

---

## 18. Delta From the As-Built Code

`backend/app/market/` currently implements everything in §7–§11 except the items below.
This is the implementation checklist.

| # | Change | Files | Why |
|---|---|---|---|
| 1 | Add `open_price` field and `day_change_percent` property to `PriceUpdate` | `models.py` | PLAN §6 addition 1 — the watchlist "change %" column |
| 2 | Serialize `timestamp` as ISO-8601 Z in `to_dict()`; add `to_iso_z` / `from_iso_z` | `models.py` | PLAN §7 convention; currently emits a float |
| 3 | Track `_open` per ticker; set on first update; clear on `remove()` | `cache.py` | Backs change 1 |
| 4 | Add the 600-point history ring buffer, `get_history()`, `HISTORY_MAXLEN` | `cache.py` | PLAN §6 addition 2 — seeds the main chart |
| 5 | Read `version` under the lock | `cache.py` | Review §3.4 |
| 6 | New `validation.py` with `normalize_ticker` and the two error types | new file | PLAN §6/§8 — `422` / `404` rules, shared by routes and sources |
| 7 | Widen the unseeded price range to $20–$500 | `seed_prices.py` | PLAN §6 (archived note said $50–$300) |
| 8 | Guard `np.linalg.cholesky` with `LinAlgError` fallback | `simulator.py` | Degrade to independent draws instead of crashing rebuilds |
| 9 | `GBMSimulator.get_price()` returns a rounded value | `simulator.py` | Callers should never see unrounded internal state |
| 10 | `MassiveDataSource.add_ticker()` validates and raises `UnknownTickerError`; polls once on success | `massive_client.py` | PLAN §6 unknown-ticker rule; avoids a 15 s blank row |
| 11 | Build the `APIRouter` **inside** `create_stream_router()` | `stream.py` | Review §3.6 — module-level router double-registers under repeated calls |
| 12 | Add `create_prices_router()` for `GET /api/prices/{ticker}/history` | `stream.py` | PLAN §8 |
| 13 | Add the SSE keep-alive comment after 15 quiet seconds | `stream.py` | Massive mode leaves the stream silent for 15 s at a time |
| 14 | Add `active_source_name()` and `MASSIVE_POLL_INTERVAL` | `factory.py` | `/api/health.data_source`; paid-tier tuning |
| 15 | Re-export the new names | `__init__.py` | Keep one import surface |
| 16 | New tests per §16 | `tests/market/` | open price, history, validation, ISO-Z, SSE, thread safety |

None of these change the shape of the existing public API — `PriceCache.get_price()`,
`create_market_data_source()`, and the SSE endpoint path all keep working for code
written against the current modules. `to_dict()`'s `timestamp` type change (float → ISO
string) is the one breaking change, and it has no consumers yet.
