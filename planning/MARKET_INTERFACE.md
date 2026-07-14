# Market Data Interface

Unified Python interface for market data in FinAlly. Two implementations — `SimulatorDataSource` and `MassiveDataSource` — sit behind one abstract interface. All downstream code (SSE streaming, trade execution, portfolio valuation) is source-agnostic.

The implementation lives in `backend/app/market/`.

---

## Architecture

```
MarketDataSource (ABC)          interface.py
├── SimulatorDataSource    →    simulator.py   (default, no external dependency)
└── MassiveDataSource      →    massive_client.py  (when MASSIVE_API_KEY is set)
         │
         ▼ writes to
    PriceCache (thread-safe)    cache.py
         │
         ├──→ SSE stream endpoint (/api/stream/prices)    stream.py
         ├──→ Trade execution (read current price)
         └──→ Portfolio valuation (GET /api/portfolio)
```

Both data sources write to the same `PriceCache` on their own schedule. Downstream code never calls a data source directly for prices — it only reads from the cache.

---

## Core Data Model

```python
# backend/app/market/models.py
from dataclasses import dataclass

@dataclass(frozen=True)
class PriceUpdate:
    ticker: str
    price: float            # Current price (rounded to 2 decimal places)
    previous_price: float   # Price from the previous update
    timestamp: float        # Unix seconds

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
        """'up', 'down', or 'flat'"""
        if self.price > self.previous_price:
            return "up"
        elif self.price < self.previous_price:
            return "down"
        return "flat"

    def to_dict(self) -> dict:
        return {
            "ticker": self.ticker,
            "price": self.price,
            "previous_price": self.previous_price,
            "timestamp": self.timestamp,
            "change": self.change,
            "change_percent": self.change_percent,
            "direction": self.direction,
        }
```

`PriceUpdate` is frozen (immutable). On the first update for a ticker, `previous_price == price` and `direction == "flat"`.

---

## Abstract Interface

```python
# backend/app/market/interface.py
from abc import ABC, abstractmethod

class MarketDataSource(ABC):
    """Contract for market data providers.

    Implementations push price updates into a PriceCache on their own schedule.
    Downstream code reads from the cache — never from the source directly.

    Lifecycle:
        source = create_market_data_source(cache)
        await source.start(["AAPL", "GOOGL", ...])  # Starts background task
        await source.add_ticker("TSLA")              # Hot-add during operation
        await source.remove_ticker("GOOGL")          # Hot-remove during operation
        await source.stop()                          # Clean shutdown
    """

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing price updates. Must be called exactly once."""

    @abstractmethod
    async def stop(self) -> None:
        """Stop the background task. Safe to call multiple times."""

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Add a ticker. No-op if already present. Takes effect next update cycle."""

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker. Also removes it from the PriceCache."""

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Return the current list of actively tracked tickers."""
```

---

## Price Cache

Thread-safe in-memory store. One writer (the active data source), many readers (SSE, trades, portfolio).

```python
# backend/app/market/cache.py
from threading import Lock

class PriceCache:
    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = Lock()
        self._version: int = 0  # Increments on every update (for SSE change detection)

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
        """Record a new price. Returns the PriceUpdate. Thread-safe."""
        with self._lock:
            ts = timestamp or time.time()
            prev = self._prices.get(ticker)
            previous_price = prev.price if prev else price
            update = PriceUpdate(
                ticker=ticker,
                price=round(price, 2),
                previous_price=round(previous_price, 2),
                timestamp=ts,
            )
            self._prices[ticker] = update
            self._version += 1
            return update

    def get(self, ticker: str) -> PriceUpdate | None:
        """Latest PriceUpdate for a ticker, or None."""
        with self._lock:
            return self._prices.get(ticker)

    def get_price(self, ticker: str) -> float | None:
        """Convenience: just the price float, or None."""
        update = self.get(ticker)
        return update.price if update else None

    def get_all(self) -> dict[str, PriceUpdate]:
        """Snapshot of all current prices. Returns a shallow copy."""
        with self._lock:
            return dict(self._prices)

    def remove(self, ticker: str) -> None:
        with self._lock:
            self._prices.pop(ticker, None)

    @property
    def version(self) -> int:
        """Monotonic counter. Useful for SSE long-poll change detection."""
        return self._version
```

The `version` property enables efficient SSE: the stream endpoint records the version it last sent, and only pushes an update when `cache.version` has changed.

---

## Factory Function

Selects the data source at startup based on the environment:

```python
# backend/app/market/factory.py
import os

def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()

    if api_key:
        from .massive_client import MassiveDataSource
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    else:
        from .simulator import SimulatorDataSource
        return SimulatorDataSource(price_cache=price_cache)
```

---

## MassiveDataSource Implementation

```python
# backend/app/market/massive_client.py
import asyncio
import logging
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

class MassiveDataSource(MarketDataSource):
    def __init__(self, api_key: str, price_cache: PriceCache, poll_interval: float = 15.0):
        self._api_key = api_key
        self._cache = price_cache
        self._interval = poll_interval      # 15s for free tier, 2-5s for paid
        self._tickers: list[str] = []
        self._task: asyncio.Task | None = None
        self._client: RESTClient | None = None

    async def start(self, tickers: list[str]) -> None:
        self._client = RESTClient(api_key=self._api_key)
        self._tickers = list(tickers)
        await self._poll_once()             # Populate cache immediately on start
        self._task = asyncio.create_task(self._poll_loop(), name="massive-poller")

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        self._client = None

    async def add_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        if ticker not in self._tickers:
            self._tickers.append(ticker)
            # Will appear in cache on next poll cycle

    async def remove_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        self._tickers = [t for t in self._tickers if t != ticker]
        self._cache.remove(ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    async def _poll_loop(self) -> None:
        while True:
            await asyncio.sleep(self._interval)
            await self._poll_once()

    async def _poll_once(self) -> None:
        if not self._tickers or not self._client:
            return
        try:
            # RESTClient is synchronous — run in thread pool
            snapshots = await asyncio.to_thread(self._fetch_snapshots)
            for snap in snapshots:
                try:
                    self._cache.update(
                        ticker=snap.ticker,
                        price=snap.last_trade.price,
                        timestamp=snap.last_trade.timestamp / 1000.0,  # ms -> seconds
                    )
                except (AttributeError, TypeError) as e:
                    logger.warning("Skipping snapshot for %s: %s", snap.ticker, e)
        except Exception as e:
            logger.error("Massive poll failed: %s", e)
            # Don't re-raise — loop retries on next interval

    def _fetch_snapshots(self) -> list:
        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS,
            tickers=self._tickers,
        )
```

**Key behaviors**:
- An immediate poll in `start()` ensures the SSE stream has data before any client connects
- All API errors are caught and logged; the loop always continues
- The synchronous Massive client runs in `asyncio.to_thread()` to avoid blocking FastAPI's event loop
- New tickers appear in the cache after the next poll cycle (no restart required)

---

## SimulatorDataSource Implementation

```python
# backend/app/market/simulator.py (SimulatorDataSource portion)
import asyncio
import logging

class SimulatorDataSource(MarketDataSource):
    def __init__(self, price_cache: PriceCache, update_interval: float = 0.5,
                 event_probability: float = 0.001):
        self._cache = price_cache
        self._interval = update_interval    # 500ms = 2 updates per second
        self._event_prob = event_probability
        self._sim: GBMSimulator | None = None
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers=tickers, event_probability=self._event_prob)
        # Seed cache immediately so SSE has data before the first step
        for ticker in tickers:
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None

    async def add_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.add_ticker(ticker)
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)  # Seed immediately

    async def remove_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.remove_ticker(ticker)
        self._cache.remove(ticker)

    def get_tickers(self) -> list[str]:
        return self._sim.get_tickers() if self._sim else []

    async def _run_loop(self) -> None:
        while True:
            try:
                if self._sim:
                    prices = self._sim.step()   # Returns {ticker: new_price}
                    for ticker, price in prices.items():
                        self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed")
            await asyncio.sleep(self._interval)
```

**Key behaviors**:
- Seeds the cache with initial prices immediately in `start()` and `add_ticker()`
- Calls `GBMSimulator.step()` every 500ms, writing all updated prices to the cache in one batch
- Exceptions in `step()` are logged but never propagate — the loop always continues

---

## Integration with SSE

The SSE endpoint reads the entire `PriceCache` on a timer and pushes updates to connected clients. The `version` counter enables change detection — the endpoint only yields when the cache has new data:

```python
# Simplified from backend/app/market/stream.py
import asyncio, json
from fastapi import APIRouter
from fastapi.responses import StreamingResponse

def create_stream_router(price_cache: PriceCache) -> APIRouter:
    router = APIRouter()

    @router.get("/api/stream/prices")
    async def stream_prices():
        async def generate():
            last_version = -1
            while True:
                if price_cache.version != last_version:
                    last_version = price_cache.version
                    data = {
                        ticker: update.to_dict()
                        for ticker, update in price_cache.get_all().items()
                    }
                    yield f"data: {json.dumps(data)}\n\n"
                await asyncio.sleep(0.5)

        return StreamingResponse(generate(), media_type="text/event-stream")

    return router
```

---

## Module File Structure

```
backend/
  app/
    market/
      __init__.py         # Exports: PriceCache, PriceUpdate, MarketDataSource, create_market_data_source, create_stream_router
      models.py           # PriceUpdate dataclass
      interface.py        # MarketDataSource ABC
      cache.py            # PriceCache
      factory.py          # create_market_data_source()
      massive_client.py   # MassiveDataSource
      simulator.py        # GBMSimulator + SimulatorDataSource
      seed_prices.py      # SEED_PRICES, TICKER_PARAMS, DEFAULT_PARAMS, CORRELATION_GROUPS
      stream.py           # create_stream_router() — SSE endpoint
```

---

## Application Startup

```python
from app.market import PriceCache, create_market_data_source, create_stream_router

# Create shared cache
cache = PriceCache()

# Create data source (reads MASSIVE_API_KEY from env)
source = create_market_data_source(cache)

# Register SSE router
app.include_router(create_stream_router(cache))

# Start producing prices (initial tickers from DB watchlist)
@app.on_event("startup")
async def startup():
    initial_tickers = await db.get_watchlist_tickers()
    await source.start(initial_tickers)

@app.on_event("shutdown")
async def shutdown():
    await source.stop()
```

## Downstream Usage (Other API Routes)

```python
# Reading prices in trade/portfolio routes
update = cache.get("AAPL")           # PriceUpdate or None
price = cache.get_price("AAPL")      # float or None
all_prices = cache.get_all()         # dict[str, PriceUpdate]

# Watchlist route — hot-add/remove
await source.add_ticker("PYPL")
await source.remove_ticker("NFLX")
```

The route handlers for `POST /api/watchlist` and `DELETE /api/watchlist/{ticker}` must call both the database layer (to persist the change) and the source (to update the live price feed) in the same request handler.
