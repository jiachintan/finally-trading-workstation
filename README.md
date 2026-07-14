# FinAlly — AI Trading Workstation

An AI-powered trading terminal with live market data, simulated portfolio management, and an LLM chat assistant that can analyze positions and execute trades.

## Quick Start

```bash
cp .env.example .env
# Add your ANTHROPIC_API_KEY to .env
./scripts/start_mac.sh
```

Then open [http://localhost:8000](http://localhost:8000).

> **Windows**: use `scripts/start_windows.ps1`

## Features

- **Live price streaming** — prices flash green/red on tick via SSE
- **Sparkline charts** — mini price charts per ticker, accumulated since page load
- **Simulated portfolio** — start with $10,000 cash, buy/sell at market price instantly
- **Portfolio heatmap** — treemap sized by weight, colored by P&L
- **P&L chart** — total portfolio value over time
- **AI chat assistant** — natural language trading: "buy 10 shares of AAPL" or "what's my biggest position?"

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `ANTHROPIC_API_KEY` | Yes | Claude API key for AI chat |
| `MASSIVE_API_KEY` | No | Real market data (simulator used if omitted) |
| `LLM_MOCK` | No | Set `true` for deterministic mock responses (testing) |

## Architecture

- **Frontend**: Next.js (TypeScript, static export) served by FastAPI
- **Backend**: FastAPI + Python (`uv`), SQLite database
- **Real-time**: Server-Sent Events (`/api/stream/prices`)
- **AI**: Anthropic Claude (`claude-sonnet-4-6`) with structured outputs
- **Deployment**: Single Docker container on port 8000

## Stop

```bash
./scripts/stop_mac.sh
```

The SQLite database persists in a Docker volume (`finally-data`) across restarts.
