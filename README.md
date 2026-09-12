# FinAlly — AI Trading Workstation

An AI-powered trading terminal: live streaming prices, a $10,000 simulated portfolio, and an LLM copilot that analyzes your positions and executes trades from natural language. Dark, data-dense, Bloomberg-inspired.

Built entirely by coding agents as the capstone of an agentic AI coding course. The shared contract between agents is [`planning/PLAN.md`](planning/PLAN.md).

## Features

- **Live prices** over SSE with green/red flash animations and sparklines
- **Simulated trading** — market orders, instant fills, no fees, fractional shares
- **Portfolio views** — treemap heatmap, P&L chart, positions table, realized P&L
- **AI assistant** — chat about your portfolio; trades and watchlist changes auto-execute
- **Watchlist** — add/remove tickers manually or via chat

## Architecture

One Docker container, one port (8000):

| Layer | Tech |
|---|---|
| Frontend | Next.js + TypeScript + Tailwind, static export served by FastAPI |
| Backend | FastAPI (Python, `uv`), SSE at `/api/stream/prices` |
| Database | SQLite at `db/finally.db`, lazily created and seeded |
| Market data | Built-in GBM simulator (default) or Massive/Polygon.io REST |
| AI | LiteLLM → OpenRouter (`openai/gpt-oss-120b` on Cerebras), structured outputs |

## Quick Start

```bash
cp .env.example .env          # add your OPENROUTER_API_KEY
./scripts/start_mac.sh        # macOS/Linux
.\scripts\start_windows.ps1   # Windows
```

Opens `http://localhost:8000`. Stop with `scripts/stop_mac.sh` / `scripts/stop_windows.ps1`; portfolio data persists in the `finally-data` Docker volume.

Equivalent manual command:

```bash
docker build -t finally .
docker run -d --name finally -v finally-data:/app/db -p 8000:8000 --env-file .env finally
```

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `OPENROUTER_API_KEY` | Yes | LLM chat via OpenRouter |
| `MASSIVE_API_KEY` | No | Real market data; omit to use the simulator |
| `LLM_MOCK` | No | `true` → deterministic mock LLM responses (tests, no API key needed) |
| `FINALLY_DB_PATH` | No | Override SQLite location (default `db/finally.db`) |

## Local Development

```bash
cd backend && uv run uvicorn app.main:app --reload --port 8000
cd frontend && npm run dev     # http://localhost:3000, proxies /api/* to the backend
```

## Tests

```bash
cd backend && uv run pytest                     # backend unit tests
cd frontend && npm test                         # frontend logic tests (Vitest)
docker compose -f test/docker-compose.test.yml up --abort-on-container-exit   # Playwright E2E
```

## Project Structure

```
finally/
├── frontend/   # Next.js static export
├── backend/    # FastAPI uv project (app/market/ = market data subsystem)
├── planning/   # PLAN.md and agent-facing docs
├── scripts/    # start/stop helpers (macOS/Linux, Windows)
├── test/       # Playwright E2E + docker-compose.test.yml
└── db/         # SQLite volume mount (runtime)
```

## Status

The market data subsystem (`backend/app/market/`) is complete and tested — see [`planning/MARKET_DATA_SUMMARY.md`](planning/MARKET_DATA_SUMMARY.md). The API, frontend, Docker packaging, and E2E tests are in progress, following the build order in `planning/PLAN.md` §13.

## License

See [LICENSE](LICENSE).
