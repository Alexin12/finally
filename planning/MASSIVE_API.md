# Massive API (formerly Polygon.io) — Reference for FinAlly

Research notes and code examples for retrieving real-time and end-of-day stock
prices for multiple tickers. Massive is the rebrand of Polygon.io (October 2025);
the REST API paths and response shapes are unchanged from Polygon.

This document is the source of truth for the Massive client in
[MARKET_INTERFACE.md](MARKET_INTERFACE.md). The simulator alternative is in
[MARKET_SIMULATOR.md](MARKET_SIMULATOR.md).

---

## 1. Basics

- **Base URL:** `https://api.massive.com`
- **Format:** JSON over HTTPS, plain REST (we use REST polling, not WebSocket —
  simpler and works on every tier, per PLAN.md).
- **Auth:** pass the API key as a Bearer token header (preferred), or as the
  legacy `apiKey` query parameter.

```
Authorization: Bearer <MASSIVE_API_KEY>
```

- **Rate limits:** free tier is **5 requests/minute** → poll every ~15s. Paid
  tiers raise or remove the limit → poll every 2–15s. FinAlly chooses the poll
  interval from the tier (see MARKET_INTERFACE.md).

---

## 2. Endpoints we use

FinAlly only needs two things: a live price for every watched ticker, and a
previous-close baseline for the daily change %.

| Need | Endpoint | Notes |
|------|----------|-------|
| Real-time price for a set of tickers | `GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,TSLA,GOOG` | One call returns all watched tickers — ideal for our poll loop. |
| Single ticker real-time snapshot | `GET /v2/snapshot/locale/us/markets/stocks/tickers/{ticker}` | Only if we ever need one symbol. |
| Previous day close (per ticker) | `GET /v2/aggs/ticker/{ticker}/prev` | Baseline for change %. One call per ticker. |
| All tickers' end-of-day OHLC (one date) | `GET /v2/aggs/grouped/locale/us/market/stocks/{date}` | Optional: one call seeds prev-close for the whole market. |

### 2.1 Multi-ticker snapshot (the main one)

This single call covers the whole watchlist. The `tickers` parameter is a
**case-sensitive, comma-separated** list.

```
GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,TSLA,GOOG
```

Response (trimmed):

```json
{
  "status": "OK",
  "count": 3,
  "tickers": [
    {
      "ticker": "AAPL",
      "todaysChange": 1.23,
      "todaysChangePerc": 0.65,
      "updated": 1699999999000000000,
      "day":     { "o": 189.1, "h": 191.0, "l": 188.5, "c": 190.4, "v": 5123456, "vw": 190.1 },
      "prevDay": { "o": 187.0, "h": 189.5, "l": 186.2, "c": 189.2, "v": 6231456, "vw": 188.4 },
      "lastTrade": { "p": 190.42, "s": 100, "t": 1699999999000000000, "x": 11, "c": [] },
      "lastQuote": { "P": 190.45, "S": 2, "p": 190.40, "s": 3, "t": 1699999999000000000 },
      "min": { "o": 190.2, "h": 190.5, "l": 190.1, "c": 190.42, "v": 12345, "vw": 190.3 }
    }
  ]
}
```

**Field meaning** (compact OHLCV keys are shared across Massive endpoints):

| Key | Meaning |
|-----|---------|
| `lastTrade.p` | last traded price → **our "current price"** |
| `day.c` | latest day close-so-far (fallback if `lastTrade` is stale/missing) |
| `prevDay.c` | previous session close → baseline for change % |
| `todaysChange` / `todaysChangePerc` | absolute / percent change vs prev close |
| `updated` | last update time, **Unix nanoseconds** (divide by 1e9 for seconds) |
| `o h l c v vw` | open, high, low, close, volume, volume-weighted avg price |

**Price selection rule for FinAlly:** use `lastTrade.p`; if it is missing or
zero (e.g. pre-market with no trades), fall back to `day.c`, then `prevDay.c`.

### 2.2 Previous close (baseline)

```
GET /v2/aggs/ticker/AAPL/prev
```

```json
{
  "status": "OK",
  "ticker": "AAPL",
  "resultsCount": 1,
  "results": [
    { "T": "AAPL", "o": 187.0, "h": 189.5, "l": 186.2, "c": 189.2, "v": 6231456, "vw": 188.4, "t": 1699900000000 }
  ]
}
```

`results[0].c` is the previous close. Note the multi-ticker snapshot already
includes `prevDay.c`, so a separate call here is usually unnecessary.

### 2.3 Grouped daily (optional, whole-market EOD)

```
GET /v2/aggs/grouped/locale/us/market/stocks/2026-06-04?adjusted=true
```

Returns one `results[]` row per ticker with `T, o, h, l, c, v, vw, n, t`. Useful
to seed prev-close for many tickers in a single call when markets are closed.

---

## 3. Minimal Python client (httpx)

The project uses `uv` + `httpx`. This is the raw shape the Massive adapter wraps;
the unified interface in MARKET_INTERFACE.md normalizes it to a `Quote`.

```python
import os
import httpx

BASE_URL = "https://api.massive.com"


def _client() -> httpx.Client:
    key = os.environ["MASSIVE_API_KEY"]
    return httpx.Client(
        base_url=BASE_URL,
        headers={"Authorization": f"Bearer {key}"},
        timeout=10.0,
    )


def fetch_snapshots(tickers: list[str]) -> dict[str, dict]:
    """Return {ticker: {"price": float, "prev_close": float, "ts": float}}."""
    joined = ",".join(tickers)  # case-sensitive
    with _client() as client:
        resp = client.get(
            "/v2/snapshot/locale/us/markets/stocks/tickers",
            params={"tickers": joined},
        )
        resp.raise_for_status()
        payload = resp.json()

    result: dict[str, dict] = {}
    for t in payload.get("tickers", []):
        price = t.get("lastTrade", {}).get("p") or t.get("day", {}).get("c")
        prev = t.get("prevDay", {}).get("c")
        if not price:
            continue
        result[t["ticker"]] = {
            "price": price,
            "prev_close": prev,
            "ts": t.get("updated", 0) / 1e9,  # ns -> s
        }
    return result
```

**Async note:** the price-cache poller (PLAN.md §6) runs as a background task, so
the real adapter uses `httpx.AsyncClient` with the same paths. See
MARKET_INTERFACE.md for the async version that writes into the shared cache.

---

## 4. Errors & gotchas

- **401 Unauthorized** → bad/missing key. Surface clearly so the user knows to
  fix `.env`; do not silently fall back to the simulator (the simulator is only
  chosen when the key is *absent*, per PLAN.md §5).
- **429 Too Many Requests** → exceeded the tier rate limit. Back off and keep the
  last cached prices; never crash the poll loop.
- **Tickers are case-sensitive** — always upper-case symbols before the call.
- **Timestamps are nanoseconds** in snapshots but **milliseconds** in `/aggs`.
  Normalize to seconds at the adapter boundary.
- **Closed market** → `lastTrade` may be stale; rely on `day.c` / `prevDay.c`.
- The official `massive-com/client-python` SDK exists, but FinAlly uses plain
  `httpx` to keep dependencies minimal and the request flow explicit.

---

## Sources

- [Massive — Stocks REST API Overview](https://massive.com/docs/rest/stocks/overview)
- [Massive — Full Market Snapshot](https://massive.com/docs/rest/stocks/snapshots/full-market-snapshot)
- [Massive — Previous Day Bar](https://massive.com/docs/rest/stocks/aggregates/previous-day-bar)
- [Massive — Daily Market Summary (grouped daily)](https://massive.com/docs/rest/stocks/aggregates/daily-market-summary)
- [Massive — REST API Quickstart](https://massive.com/docs/rest/quickstart)
- [Polygon.io is Now Massive](https://massive.com/blog/polygon-is-now-massive)
- [massive-com/client-python (official SDK)](https://github.com/massive-com/client-python)
