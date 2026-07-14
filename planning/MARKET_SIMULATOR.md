# Market Simulator

Approach and code structure for the built-in stock price simulator. Used by default when `MASSIVE_API_KEY` is not set.

The implementation lives in `backend/app/market/simulator.py` and `backend/app/market/seed_prices.py`.

---

## Overview

The simulator uses **Geometric Brownian Motion (GBM)** to generate realistic, continuously evolving stock price paths. GBM is the mathematical foundation of the Black-Scholes option pricing model — it's the standard way to model stock prices because:

- Prices are always positive (the exponential function never goes negative)
- Returns are lognormally distributed (matching real market data)
- Volatility and drift are independently configurable per ticker
- The model is computationally trivial, running in microseconds per step

The `GBMSimulator` generates correlated price moves for all watched tickers simultaneously, updating every 500ms via the `SimulatorDataSource` background task.

---

## GBM Mathematics

At each time step, a stock price evolves as:

```
S(t+dt) = S(t) * exp((mu - sigma²/2) * dt + sigma * sqrt(dt) * Z)
```

Where:
- `S(t)` = current price
- `mu` = annualized drift (expected return), e.g. `0.05` = 5%/year
- `sigma` = annualized volatility, e.g. `0.20` = 20%/year
- `dt` = time step as a fraction of a trading year
- `Z` = standard normal random variable drawn from N(0,1)

The term `(mu - sigma²/2) * dt` is the **drift** component (deterministic).
The term `sigma * sqrt(dt) * Z` is the **diffusion** component (random).

### Time Step Calculation

Updates fire every 500ms. A trading year has approximately 252 days × 6.5 hours × 3600 seconds = 5,896,800 seconds.

```python
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # 5,896,800
DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR   # ~8.48e-8
```

This tiny `dt` produces sub-cent moves per tick, which accumulate naturally into realistic intraday ranges over time. A ticker with `sigma=0.20` (20% annual volatility) produces about 0.06% daily volatility, which is correct for a low-vol stock like MSFT.

---

## Correlated Moves

Real stocks don't move independently — tech stocks tend to move together, banks tend to move together, etc. The simulator models this using **Cholesky decomposition** of a correlation matrix.

### The Math

Given a correlation matrix `C` (symmetric, positive semi-definite, ones on diagonal), compute its lower Cholesky factor `L` such that `C = L @ L.T`.

For a vector of independent standard normals `Z_independent`:
```
Z_correlated = L @ Z_independent
```

The resulting `Z_correlated` has the correlation structure specified by `C`.

### Correlation Groups

```python
# backend/app/market/seed_prices.py
CORRELATION_GROUPS = {
    "tech":    {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}

INTRA_TECH_CORR    = 0.6   # Tech stocks move together (moderate-high)
INTRA_FINANCE_CORR = 0.5   # Finance stocks move together (moderate)
CROSS_GROUP_CORR   = 0.3   # Between sectors, or unknown tickers
TSLA_CORR          = 0.3   # TSLA is in tech but does its own thing
```

The `_rebuild_cholesky()` method constructs the full `n×n` correlation matrix and factorizes it each time a ticker is added or removed. With `n < 50`, this is O(n²) and effectively instantaneous.

```python
def _pairwise_correlation(t1: str, t2: str) -> float:
    tech = CORRELATION_GROUPS["tech"]
    finance = CORRELATION_GROUPS["finance"]

    if t1 == "TSLA" or t2 == "TSLA":
        return TSLA_CORR
    if t1 in tech and t2 in tech:
        return INTRA_TECH_CORR
    if t1 in finance and t2 in finance:
        return INTRA_FINANCE_CORR
    return CROSS_GROUP_CORR   # Cross-sector or unknown
```

---

## Random Shock Events

Every tick, each ticker has a `0.1%` chance of a sudden 2–5% move in either direction. This adds drama to the dashboard and makes price charts visually interesting.

```python
EVENT_PROBABILITY = 0.001   # 0.1% per tick per ticker

if random.random() < self._event_prob:
    shock_magnitude = random.uniform(0.02, 0.05)   # 2-5%
    shock_sign = random.choice([-1, 1])
    self._prices[ticker] *= 1 + shock_magnitude * shock_sign
```

With 10 tickers updating at 2 ticks/second, expect a shock event somewhere in the watchlist approximately every 50 seconds — enough to keep the dashboard alive without being overwhelming.

---

## Seed Prices and Per-Ticker Parameters

```python
# backend/app/market/seed_prices.py

SEED_PRICES: dict[str, float] = {
    "AAPL": 190.00,
    "GOOGL": 175.00,
    "MSFT": 420.00,
    "AMZN": 185.00,
    "TSLA": 250.00,
    "NVDA": 800.00,
    "META": 500.00,
    "JPM":  195.00,
    "V":    280.00,
    "NFLX": 600.00,
}

TICKER_PARAMS: dict[str, dict[str, float]] = {
    "AAPL":  {"sigma": 0.22, "mu": 0.05},   # Mid vol, steady
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT":  {"sigma": 0.20, "mu": 0.05},   # Lowest vol in tech
    "AMZN":  {"sigma": 0.28, "mu": 0.05},
    "TSLA":  {"sigma": 0.50, "mu": 0.03},   # Highest vol, lowest drift
    "NVDA":  {"sigma": 0.40, "mu": 0.08},   # High vol, strong positive drift
    "META":  {"sigma": 0.30, "mu": 0.05},
    "JPM":   {"sigma": 0.18, "mu": 0.04},   # Low vol (bank)
    "V":     {"sigma": 0.17, "mu": 0.04},   # Lowest vol overall (payments)
    "NFLX":  {"sigma": 0.35, "mu": 0.05},
}

# Fallback for dynamically added tickers
DEFAULT_PARAMS: dict[str, float] = {"sigma": 0.25, "mu": 0.05}
```

Tickers added dynamically (not in `SEED_PRICES`) start at a random price between $50 and $300.

---

## GBMSimulator Class

```python
# backend/app/market/simulator.py
import math, random, logging
import numpy as np
from .seed_prices import (
    SEED_PRICES, TICKER_PARAMS, DEFAULT_PARAMS,
    CORRELATION_GROUPS, CROSS_GROUP_CORR,
    INTRA_TECH_CORR, INTRA_FINANCE_CORR, TSLA_CORR,
)

class GBMSimulator:
    """Geometric Brownian Motion simulator for correlated stock prices."""

    TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600   # 5,896,800
    DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR    # ~8.48e-8

    def __init__(self, tickers: list[str], dt: float = DEFAULT_DT,
                 event_probability: float = 0.001) -> None:
        self._dt = dt
        self._event_prob = event_probability
        self._tickers: list[str] = []
        self._prices: dict[str, float] = {}
        self._params: dict[str, dict[str, float]] = {}
        self._cholesky: np.ndarray | None = None

        # Add all initial tickers, then build Cholesky once
        for ticker in tickers:
            self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    def step(self) -> dict[str, float]:
        """Advance all tickers by one time step. Returns {ticker: new_price}.

        This is the hot path — called every 500ms. ~microseconds to execute.
        """
        n = len(self._tickers)
        if n == 0:
            return {}

        # Draw n independent standard normals
        z_independent = np.random.standard_normal(n)

        # Apply Cholesky to get correlated draws
        if self._cholesky is not None:
            z = self._cholesky @ z_independent
        else:
            z = z_independent

        result: dict[str, float] = {}
        for i, ticker in enumerate(self._tickers):
            mu = self._params[ticker]["mu"]
            sigma = self._params[ticker]["sigma"]

            # GBM step
            drift = (mu - 0.5 * sigma**2) * self._dt
            diffusion = sigma * math.sqrt(self._dt) * z[i]
            self._prices[ticker] *= math.exp(drift + diffusion)

            # Random event
            if random.random() < self._event_prob:
                shock_magnitude = random.uniform(0.02, 0.05)
                shock_sign = random.choice([-1, 1])
                self._prices[ticker] *= 1 + shock_magnitude * shock_sign

            result[ticker] = round(self._prices[ticker], 2)

        return result

    def add_ticker(self, ticker: str) -> None:
        """Add a ticker. Rebuilds the correlation matrix (O(n²), fast for n<50)."""
        if ticker in self._prices:
            return
        self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker. Rebuilds the correlation matrix."""
        if ticker not in self._prices:
            return
        self._tickers.remove(ticker)
        del self._prices[ticker]
        del self._params[ticker]
        self._rebuild_cholesky()

    def get_price(self, ticker: str) -> float | None:
        return self._prices.get(ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    def _add_ticker_internal(self, ticker: str) -> None:
        """Add without rebuilding Cholesky — for batch initialization."""
        self._tickers.append(ticker)
        self._prices[ticker] = SEED_PRICES.get(ticker, random.uniform(50.0, 300.0))
        self._params[ticker] = TICKER_PARAMS.get(ticker, dict(DEFAULT_PARAMS))

    def _rebuild_cholesky(self) -> None:
        n = len(self._tickers)
        if n <= 1:
            self._cholesky = None
            return
        corr = np.eye(n)
        for i in range(n):
            for j in range(i + 1, n):
                rho = self._pairwise_correlation(self._tickers[i], self._tickers[j])
                corr[i, j] = rho
                corr[j, i] = rho
        self._cholesky = np.linalg.cholesky(corr)

    @staticmethod
    def _pairwise_correlation(t1: str, t2: str) -> float:
        tech = CORRELATION_GROUPS["tech"]
        finance = CORRELATION_GROUPS["finance"]
        if t1 == "TSLA" or t2 == "TSLA":
            return TSLA_CORR
        if t1 in tech and t2 in tech:
            return INTRA_TECH_CORR
        if t1 in finance and t2 in finance:
            return INTRA_FINANCE_CORR
        return CROSS_GROUP_CORR
```

---

## SimulatorDataSource (Async Wrapper)

`SimulatorDataSource` implements `MarketDataSource` by wrapping `GBMSimulator` in an asyncio task loop. It bridges the synchronous GBM math with the async FastAPI application.

```python
class SimulatorDataSource(MarketDataSource):
    def __init__(self, price_cache: PriceCache, update_interval: float = 0.5,
                 event_probability: float = 0.001) -> None:
        self._cache = price_cache
        self._interval = update_interval
        self._event_prob = event_probability
        self._sim: GBMSimulator | None = None
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers=tickers, event_probability=self._event_prob)
        # Seed cache with initial prices immediately (before first step)
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
                    prices = self._sim.step()
                    for ticker, price in prices.items():
                        self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed")
            await asyncio.sleep(self._interval)
```

---

## File Structure

```
backend/
  app/
    market/
      seed_prices.py    # SEED_PRICES, TICKER_PARAMS, DEFAULT_PARAMS, CORRELATION_GROUPS + constants
      simulator.py      # GBMSimulator + SimulatorDataSource
```

`seed_prices.py` contains only data constants. `simulator.py` contains all logic.

---

## Behavior Notes

### Price Floors
GBM is multiplicative — `price * exp(...)` is always positive. Prices can never go negative or zero, no matter how many negative shocks occur in a row.

### Realistic Intraday Ranges
With `sigma=0.20` and `dt=8.48e-8`, each 500ms tick produces a standard deviation of approximately:
```
sigma * sqrt(dt) = 0.20 * sqrt(8.48e-8) ≈ 0.0058% per tick
```
Over a 6.5-hour trading day (47,000 ticks), this accumulates to ~1.26% standard deviation, which is a realistic intraday range for a low-volatility stock. TSLA at `sigma=0.50` produces ~3.1% intraday range — also realistic.

### Cholesky Rebuild Cost
When a ticker is added or removed, the `n×n` correlation matrix is factorized again. For `n=20` tickers this is negligible (microseconds). The rebuild is O(n³) for Cholesky, but n is bounded well below 50 in practice.

### Correlation Structure
The correlation groups are deliberately simple — sector-based with fixed constants. This is good enough to produce visually realistic correlated moves without overfitting to any particular market regime. Unknown tickers (those not in `TICKER_PARAMS`) get the default correlation `CROSS_GROUP_CORR = 0.3`.

### Event Frequency
With `event_probability=0.001` per tick per ticker:
- 10 tickers × 2 ticks/s = 20 tick-events per second
- Expected time between any event: `1 / (20 × 0.001)` = 50 seconds
- Events are visually dramatic (2–5% move) and will trigger the price flash animation prominently

### No Market Hours
The simulator runs continuously, 24/7. There are no market open/close transitions, gaps, or overnight effects. Prices simply evolve from their seed values and drift according to the GBM parameters.
