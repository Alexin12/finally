# Market Data Interface — Unified Price API for FinAlly

How the backend retrieves stock prices through **one interface** that uses the
Massive API when `MASSIVE_API_KEY` is set, and the built-in simulator otherwise.
All downstream code (price cache, SSE, frontend) is agnostic to the source.

- Massive details: [MASSIVE_API.md](MASSIVE_API.md)
- Simulator details: [MARKET_SIMULATOR.md](MARKET_SIMULATOR.md)

---

## 1. Goal

PLAN.md §6 requires "Two Implementations, One Interface". The backend selects the
source from an environment variable; everything above the interface stays the
same. This keeps the swap to a single factory call and makes tests trivial.

```
                       ┌────────────────────────────┐
  background poller →  │  MarketDataSource (ABC)     │
                       │   - MassiveSource           │  ← MASSIVE_API_KEY set
                       │   - SimulatorSource         │  ← otherwise (default)
                       └────────────┬───────────────┘
                                    │ writes Quote objects
                                    ▼
                          PriceCache (in-memory)
                                    │ read
                                    ▼
                       SSE  GET /api/stream/prices  → frontend
```

---

## 2. The shared data type

One small dataclass is the contract between every source and the cache. Both the
Massive adapter and the simulator produce these; nothing downstream knows which.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Quote:
    ticker: str
    price: float
    prev_close: float        # baseline for daily change %
    timestamp: float         # Unix seconds

    @property
    def change(self) -> float:
        return self.price - self.prev_close

    @property
    def change_pct(self) -> float:
        if not self.prev_close:
            return 0.0
        return (self.price - self.prev_close) / self.prev_close * 100
```

---

## 3. The abstract interface

A source knows how to fetch quotes for a set of tickers. That is the entire
surface area — deliberately minimal.

```python
from abc import ABC, abstractmethod


class MarketDataSource(ABC):
    """A source of stock prices. Implemented by MassiveSource and SimulatorSource."""

    @abstractmethod
    async def fetch(self, tickers: list[str]) -> dict[str, Quote]:
        """Return the latest Quote for each ticker (best effort; may skip some)."""

    @property
    @abstractmethod
    def poll_interval(self) -> float:
        """Seconds between polls for this source."""
```

- `fetch` is async because the Massive adapter does network I/O. The simulator
  implements it too (it just computes the next step), so the poller is identical
  for both.
- `poll_interval` lets each source pick its cadence: simulator ~0.5s, Massive
  15s on free tier (see §6).

---

## 4. The two implementations

### 4.1 MassiveSource

Wraps the REST calls from MASSIVE_API.md. One snapshot call covers all watched
tickers.

```python
import os
import httpx

BASE_URL = "https://api.massive.com"


class MassiveSource(MarketDataSource):
    def __init__(self, api_key: str, poll_interval: float = 15.0):
        self._key = api_key
        self._poll = poll_interval
        self._client = httpx.AsyncClient(
            base_url=BASE_URL,
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=10.0,
        )

    @property
    def poll_interval(self) -> float:
        return self._poll

    async def fetch(self, tickers: list[str]) -> dict[str, Quote]:
        if not tickers:
            return {}
        resp = await self._client.get(
            "/v2/snapshot/locale/us/markets/stocks/tickers",
            params={"tickers": ",".join(t.upper() for t in tickers)},
        )
        resp.raise_for_status()
        out: dict[str, Quote] = {}
        for t in resp.json().get("tickers", []):
            price = t.get("lastTrade", {}).get("p") or t.get("day", {}).get("c")
            prev = t.get("prevDay", {}).get("c")
            if not price:
                continue
            out[t["ticker"]] = Quote(
                ticker=t["ticker"],
                price=price,
                prev_close=prev or price,
                timestamp=t.get("updated", 0) / 1e9,  # ns -> s
            )
        return out
```

### 4.2 SimulatorSource

Implements the same `fetch`/`poll_interval`, generating prices with geometric
Brownian motion. Full design in MARKET_SIMULATOR.md. Sketch:

```python
import time


class SimulatorSource(MarketDataSource):
    def __init__(self, poll_interval: float = 0.5):
        self._poll = poll_interval
        self._engine = SimulatorEngine()  # see MARKET_SIMULATOR.md

    @property
    def poll_interval(self) -> float:
        return self._poll

    async def fetch(self, tickers: list[str]) -> dict[str, Quote]:
        prices = self._engine.step(tickers)  # advances GBM one tick
        now = time.time()
        return {
            tk: Quote(tk, price=p, prev_close=self._engine.prev_close(tk), timestamp=now)
            for tk, p in prices.items()
        }
```

---

## 5. Selecting the source

A single factory reads the environment (PLAN.md §5). This is the **only** place
the choice is made.

```python
import os


def make_source() -> MarketDataSource:
    key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if key:
        return MassiveSource(api_key=key, poll_interval=_poll_interval_for_tier())
    return SimulatorSource()


def _poll_interval_for_tier() -> float:
    # Free tier = 5 calls/min. Override via env for paid tiers.
    return float(os.environ.get("MASSIVE_POLL_SECONDS", "15"))
```

Rule (from PLAN.md §5): key present and non-empty → Massive; absent or empty →
simulator. A bad key should error loudly (401), not silently fall back.

---

## 6. The price cache and poller

A single background task drives the chosen source and writes the latest quotes
into an in-memory cache. SSE reads from the cache. This is the architecture in
PLAN.md §6 ("Shared Price Cache") and supports future multi-user without changes.

```python
import asyncio


class PriceCache:
    def __init__(self):
        self._quotes: dict[str, Quote] = {}

    def update(self, quotes: dict[str, Quote]) -> None:
        self._quotes.update(quotes)

    def get(self, ticker: str) -> Quote | None:
        return self._quotes.get(ticker)

    def all(self) -> dict[str, Quote]:
        return dict(self._quotes)


async def run_poller(source: MarketDataSource, cache: PriceCache, watched: set[str]):
    """Background task: poll the source forever, write into the cache."""
    while True:
        try:
            quotes = await source.fetch(sorted(watched))
            cache.update(quotes)
        except Exception:
            # never let the loop die; keep last cached prices
            pass
        await asyncio.sleep(source.poll_interval)
```

- `watched` is the union of all watched tickers (single-user = the watchlist).
- On error (e.g. 429) the loop keeps the previous cache and retries next tick.
- SSE endpoint reads `cache.all()` every ~500ms and pushes diffs to clients,
  independent of how fast the source actually updates.

---

## 7. Why this shape

- **One factory, one interface** — swapping data source is a config change, not a
  code change. Matches PLAN.md's "agnostic downstream code".
- **`Quote` is the whole contract** — small, frozen, computes change %; both
  sources and all consumers share it.
- **Cache decouples cadence** — Massive may poll every 15s while SSE pushes every
  500ms; the frontend animation stays smooth regardless of source.
- **No premature abstraction** — only two methods on the ABC; we add more only
  when a real need appears.
