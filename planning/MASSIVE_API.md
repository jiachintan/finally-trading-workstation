# Massive API Reference (formerly Polygon.io)

Reference documentation for the Massive (formerly Polygon.io) REST API as used in FinAlly.

Polygon.io rebranded as Massive on October 30, 2025. Existing API keys, accounts, and integrations continue to work as before. The `massive` Python package (successor to `polygon-api-client`) defaults to `api.massive.com` while `api.polygon.io` remains supported.

## Overview

- **Base URL**: `https://api.massive.com` (legacy `https://api.polygon.io` still supported)
- **Python package**: `massive` (`uv add massive` / `pip install -U massive`)
- **Min Python version**: 3.9+
- **Auth**: API key passed to `RESTClient(api_key=...)` or read from `MASSIVE_API_KEY` env var automatically
- **Auth header**: `Authorization: Bearer <API_KEY>` (the client sets this for you)

## Rate Limits

| Tier | Limit |
|------|-------|
| Free | 5 requests/minute |
| Paid (all tiers) | Unlimited (recommended: stay under 100 req/s) |

FinAlly polls on a timer. Free tier: poll every 15s. Paid tiers: poll every 2–5s.

## Client Initialization

```python
from massive import RESTClient

# Reads MASSIVE_API_KEY from environment automatically
client = RESTClient()

# Or pass explicitly
client = RESTClient(api_key="your_key_here")
```

The client is **synchronous**. When using it inside an async FastAPI app, wrap calls with `asyncio.to_thread()` to avoid blocking the event loop:

```python
import asyncio
from massive import RESTClient

client = RESTClient(api_key="...")
snapshots = await asyncio.to_thread(
    client.get_snapshot_all,
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "GOOGL"],
)
```

---

## Endpoints Used in FinAlly

### 1. Snapshot — All Tickers (Primary Endpoint)

Gets the current price for multiple tickers in **a single API call**. This is the main endpoint FinAlly uses for polling the watchlist.

**REST path**: `GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,GOOGL,MSFT`

**Python**:
```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient()

snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA"],
)

for snap in snapshots:
    print(f"{snap.ticker}: ${snap.last_trade.price}")
    print(f"  Day change: {snap.day.change_percent:.2f}%")
    print(f"  Day OHLC: O={snap.day.open} H={snap.day.high} L={snap.day.low} C={snap.day.close}")
    print(f"  Volume: {snap.day.volume:,}")
    print(f"  Prev close: {snap.day.previous_close}")
```

**Response structure per ticker**:
```json
{
  "ticker": "AAPL",
  "day": {
    "open": 129.61,
    "high": 130.15,
    "low": 125.07,
    "close": 125.07,
    "volume": 111237700,
    "volume_weighted_average_price": 127.35,
    "previous_close": 129.61,
    "change": -4.54,
    "change_percent": -3.50
  },
  "last_trade": {
    "price": 125.07,
    "size": 100,
    "exchange": "XNYS",
    "timestamp": 1675190399000
  },
  "last_quote": {
    "bid_price": 125.06,
    "ask_price": 125.08,
    "bid_size": 500,
    "ask_size": 1000,
    "spread": 0.02,
    "timestamp": 1675190399500
  },
  "prev_daily_bar": { "...": "previous day OHLCV" },
  "min": { "...": "most recent minute bar" }
}
```

**Key fields extracted by FinAlly**:
| Field | Used for |
|-------|---------|
| `last_trade.price` | Current price for display, trades, portfolio valuation |
| `last_trade.timestamp` | Price timestamp (Unix **milliseconds** — divide by 1000 for seconds) |
| `day.previous_close` | Day-change calculation (not currently stored in PriceCache, available if needed) |
| `day.change_percent` | Day change % (available for display) |

### 2. Single Ticker Snapshot

For detailed data on one ticker (e.g., user clicks a ticker for the detail view).

**REST path**: `GET /v2/snapshot/locale/us/markets/stocks/tickers/{ticker}`

**Python**:
```python
snapshot = client.get_snapshot_ticker(
    market_type=SnapshotMarketType.STOCKS,
    ticker="AAPL",
)

print(f"Price: ${snapshot.last_trade.price}")
print(f"Bid/Ask: ${snapshot.last_quote.bid_price} / ${snapshot.last_quote.ask_price}")
print(f"Day range: ${snapshot.day.low} – ${snapshot.day.high}")
print(f"Volume: {snapshot.day.volume:,}")
```

### 3. Previous Close

Returns the previous trading day's OHLCV bar. Useful for seeding initial prices or showing daily context.

**REST path**: `GET /v2/aggs/ticker/{ticker}/prev`

**Python**:
```python
for bar in client.get_previous_close_agg(ticker="AAPL"):
    print(f"Prev close: ${bar.close}")
    print(f"OHLC: O={bar.open} H={bar.high} L={bar.low} C={bar.close}")
    print(f"Volume: {bar.volume:,}")
    print(f"Timestamp (ms): {bar.timestamp}")
```

**Response** (each result bar):
```json
{
  "o": 150.0,
  "h": 155.0,
  "l": 149.0,
  "c": 154.5,
  "v": 1000000,
  "vw": 152.3,
  "t": 1672531200000
}
```

### 4. Aggregates (Historical Bars)

Historical OHLCV bars over a date range. Not used for live polling but available for historical charts.

**REST path**: `GET /v2/aggs/ticker/{ticker}/range/{multiplier}/{timespan}/{from}/{to}`

**Python**:
```python
aggs = list(client.list_aggs(
    ticker="AAPL",
    multiplier=1,
    timespan="day",      # "minute", "hour", "day", "week", "month"
    from_="2024-01-01",
    to="2024-01-31",
    limit=50000,         # Max per request; auto-paginates
))

for a in aggs:
    print(f"t={a.timestamp}, O={a.open} H={a.high} L={a.low} C={a.close} V={a.volume}")
```

### 5. Last Trade / Last Quote

Individual endpoints for the single most recent trade or NBBO quote.

```python
# Most recent trade for a ticker
trade = client.get_last_trade(ticker="AAPL")
print(f"Last trade: ${trade.price} × {trade.size} @ exchange {trade.exchange}")
print(f"Timestamp (ms): {trade.timestamp}")

# Most recent NBBO quote
quote = client.get_last_quote(ticker="AAPL")
print(f"Bid: ${quote.bid} × {quote.bid_size}")
print(f"Ask: ${quote.ask} × {quote.ask_size}")
```

---

## How FinAlly Uses the API

The `MassiveDataSource` runs as a background asyncio task:

1. Collects all tickers from the in-memory ticker list
2. Calls `get_snapshot_all()` with those tickers (**one API call** for all tickers)
3. Extracts `last_trade.price` and `last_trade.timestamp` from each snapshot
4. Writes to the shared `PriceCache`
5. Sleeps for the poll interval, then repeats

An immediate first poll happens in `start()` so the cache is populated before any SSE clients connect.

```python
# Simplified from backend/app/market/massive_client.py
import asyncio
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

class MassiveDataSource:
    async def start(self, tickers: list[str]) -> None:
        self._client = RESTClient(api_key=self._api_key)
        self._tickers = list(tickers)
        await self._poll_once()  # Populate cache immediately
        self._task = asyncio.create_task(self._poll_loop())

    async def _poll_loop(self) -> None:
        while True:
            await asyncio.sleep(self._interval)
            await self._poll_once()

    async def _poll_once(self) -> None:
        if not self._tickers or not self._client:
            return
        # RESTClient is synchronous — run in thread pool
        snapshots = await asyncio.to_thread(self._fetch_snapshots)
        for snap in snapshots:
            price = snap.last_trade.price
            timestamp = snap.last_trade.timestamp / 1000.0  # ms -> seconds
            self._cache.update(ticker=snap.ticker, price=price, timestamp=timestamp)

    def _fetch_snapshots(self) -> list:
        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS,
            tickers=self._tickers,
        )
```

---

## Error Handling

The client raises exceptions for HTTP errors. The `MassiveDataSource` catches all exceptions in `_poll_once()` and logs them — the poll loop continues running regardless, retrying on the next interval.

| Code | Cause | Action |
|------|-------|--------|
| 401 | Invalid API key | Check `MASSIVE_API_KEY` env var |
| 403 | Plan doesn't include the endpoint | Upgrade plan or use different endpoint |
| 429 | Rate limit exceeded | Increase poll interval; free tier: ≥15s |
| 5xx | Server error | Automatic retry on next poll cycle |

---

## Market Hours & Data Freshness

- **During market hours (9:30 AM – 4:00 PM ET)**: `last_trade.price` is the most recent trade price, typically seconds old
- **Pre/post-market**: `last_trade.price` reflects the last extended-hours trade (may be stale)
- **Market closed**: `last_trade.price` is the last price from the previous session
- **Snapshot data** resets daily at 3:30 AM ET; repopulates from ~4:00 AM ET as exchanges open
- **Timestamps** are Unix **milliseconds** — always divide by 1000 before storing or comparing

## Notes

- The snapshot endpoint returns all requested tickers in **one API call** — critical for staying within the free tier's 5 req/min limit
- The `massive` Python package is synchronous; wrap all calls with `asyncio.to_thread()` inside async code
- A new ticker added via `add_ticker()` appears in the SSE stream on the next poll cycle (no restart required)
- The Python package works with both `api.massive.com` and `api.polygon.io` base URLs
