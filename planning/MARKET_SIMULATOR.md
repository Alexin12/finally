# Market Simulator — Approach & Code Structure

How FinAlly generates realistic-looking stock prices when no `MASSIVE_API_KEY` is
set. This is the default data source (PLAN.md §6) and runs entirely in-process
with no external dependencies.

- Where it plugs in: [MARKET_INTERFACE.md](MARKET_INTERFACE.md) (`SimulatorSource`)
- The real-data alternative: [MASSIVE_API.md](MASSIVE_API.md)

---

## 1. Goals

From PLAN.md §6, the simulator should:

- Generate prices via **geometric Brownian motion (GBM)** with per-ticker drift
  and volatility.
- Update at **~500ms** intervals.
- Produce **correlated moves** (e.g. tech stocks drift together).
- Fire occasional **events** — sudden 2–5% jumps on a ticker for drama.
- Start from **realistic seed prices** (AAPL ~$190, GOOGL ~$175, …).
- Be deterministic when seeded, for reproducible tests.

It only needs to *look* convincing on a chart, not be financially accurate.

---

## 2. The math: GBM, one step at a time

GBM is the standard model for stock-price simulation. The discrete update for one
time step `dt` is:

```
S_next = S * exp((mu - 0.5 * sigma^2) * dt + sigma * sqrt(dt) * Z)
```

- `S` = current price
- `mu` = annual drift (small, e.g. 0.05)
- `sigma` = annual volatility (e.g. 0.2 for stable, 0.6 for TSLA-like)
- `dt` = time step as a fraction of a year (500ms ≈ `0.5 / (252*6.5*3600)` of a
  trading year; in practice we use a small constant tuned for visible motion)
- `Z` = a standard-normal random draw (this is where correlation enters, §3)

We advance one step per tick — no need to simulate a full path in advance.

---

## 3. Correlation between tickers

Independent random draws make every ticker wiggle on its own, which looks fake.
Real markets move together. We get this cheaply with a **shared market factor**:

```
Z_i = beta_i * Z_market + sqrt(1 - beta_i^2) * Z_i_idiosyncratic
```

- `Z_market` — one draw per tick, shared by all tickers (the "market mood").
- `Z_i_idiosyncratic` — a per-ticker independent draw.
- `beta_i` — how strongly ticker `i` follows the market (0 = independent,
  1 = fully market-driven). Tech names get a high beta; defensives lower.

This single-factor model gives believable co-movement without a full covariance
matrix — three lines instead of a premature abstraction.

---

## 4. Random events

Each tick, with small probability, pick a random ticker and apply a one-off shock
of ±2–5% on top of the GBM step. This produces the occasional dramatic spike the
demo wants.

```python
if random.random() < EVENT_PROB_PER_TICK:      # e.g. 0.01
    victim = random.choice(tickers)
    shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
    prices[victim] *= (1 + shock)
```

---

## 5. Seed data

Per-ticker starting price, drift, volatility, and market beta live in one table.
Prices are anchored to realistic 2026 levels; tweak freely.

```python
SEEDS = {
    #          price    mu     sigma  beta
    "AAPL":  (190.0,  0.06,  0.22,  0.9),
    "GOOGL": (175.0,  0.07,  0.25,  0.9),
    "MSFT":  (420.0,  0.06,  0.21,  0.9),
    "AMZN":  (185.0,  0.08,  0.28,  0.85),
    "TSLA":  (250.0,  0.05,  0.60,  0.8),
    "NVDA":  (120.0,  0.12,  0.45,  0.85),
    "META":  (500.0,  0.08,  0.30,  0.8),
    "JPM":   (200.0,  0.04,  0.18,  0.5),
    "V":     (280.0,  0.05,  0.16,  0.5),
    "NFLX":  (650.0,  0.07,  0.35,  0.7),
}
```

`prev_close` for each ticker is fixed at the seed price for the session, so the
daily change % grows naturally from the starting point.

---

## 6. Code structure

One engine class holds state (current prices) and exposes a `step()` that the
`SimulatorSource` (MARKET_INTERFACE.md §4.2) calls each tick.

```python
import math
import random


class SimulatorEngine:
    """In-process GBM price generator with market correlation and events."""

    DT = 0.0005             # tuned for visible but not wild motion per tick
    EVENT_PROB = 0.01       # chance of a shock event per tick

    def __init__(self, seeds: dict | None = None, rng_seed: int | None = None):
        seeds = seeds or SEEDS
        self._rng = random.Random(rng_seed)   # seedable for tests
        self._params = seeds
        self._prices = {tk: cfg[0] for tk, cfg in seeds.items()}
        self._prev_close = dict(self._prices)  # baseline fixed at session start

    def prev_close(self, ticker: str) -> float:
        return self._prev_close.get(ticker, self._prices.get(ticker, 0.0))

    def step(self, tickers: list[str]) -> dict[str, float]:
        """Advance one tick and return current prices for the requested tickers."""
        z_market = self._rng.gauss(0, 1)            # shared market factor
        known = [t for t in tickers if t in self._params]

        for tk in known:
            price, mu, sigma, beta = self._prices[tk], *self._params[tk][1:]
            z_idio = self._rng.gauss(0, 1)
            z = beta * z_market + math.sqrt(max(0.0, 1 - beta * beta)) * z_idio
            drift = (mu - 0.5 * sigma * sigma) * self.DT
            shock = sigma * math.sqrt(self.DT) * z
            self._prices[tk] = price * math.exp(drift + shock)

        # occasional dramatic event
        if known and self._rng.random() < self.EVENT_PROB:
            victim = self._rng.choice(known)
            jump = self._rng.uniform(0.02, 0.05) * self._rng.choice([-1, 1])
            self._prices[victim] *= (1 + jump)

        return {tk: round(self._prices[tk], 2) for tk in known}
```

Unknown tickers (added to the watchlist but not in `SEEDS`) are simply ignored
here; the source can lazily seed a new ticker at a default price + volatility
when first requested.

---

## 7. How it runs

The `SimulatorSource.fetch()` (MARKET_INTERFACE.md §4.2) calls `engine.step()`
each poll and wraps the result in `Quote` objects. The poller writes them to the
`PriceCache` at the simulator's `poll_interval` (~0.5s), and SSE streams them to
the frontend. No background threads beyond the single shared poller, no external
calls.

---

## 8. Testing

- **Deterministic:** construct `SimulatorEngine(rng_seed=42)` and assert the
  first N steps match expected values — reproducible because the RNG is seeded.
- **Sanity:** prices stay positive (GBM is multiplicative), never NaN/inf.
- **Correlation:** with high betas, per-tick returns across tickers are positively
  correlated; verify the sign agreement over many ticks.
- **Interface conformance:** `SimulatorSource` satisfies the same `fetch` /
  `poll_interval` contract as `MassiveSource` (MARKET_INTERFACE.md §3).

---

## 9. Why this shape

- **GBM** is the simplest model that yields realistic price paths — positive,
  trending, noisy.
- **Single market factor** buys believable co-movement in three lines; a full
  covariance matrix would be over-engineering for a demo.
- **Seedable RNG** makes the simulator usable as the default `LLM_MOCK`-style
  deterministic source in tests and CI.
- **Same `Quote` contract** as Massive means zero downstream changes when
  switching sources.
