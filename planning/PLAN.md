# FinAlly — AI Trading Workstation

## Project Specification

## 1. Vision

FinAlly (Finance Ally) is a visually stunning AI-powered trading workstation that streams live market data, lets users trade a simulated portfolio, and integrates an LLM chat assistant that can analyze positions and execute trades on the user's behalf. It looks and feels like a modern Bloomberg terminal with an AI copilot.

This is the capstone project for an agentic AI coding course. It is built entirely by Coding Agents demonstrating how orchestrated AI agents can produce a production-quality full-stack application. Agents interact through files in `planning/`.

## 2. User Experience

### First Launch

The user runs a single Docker command (or a provided start script). A browser opens to `http://localhost:8000`. No login, no signup. They immediately see:

- A watchlist of 10 default tickers with live-updating prices in a grid
- $10,000 in virtual cash
- A dark, data-rich trading terminal aesthetic
- An AI chat panel ready to assist

### What the User Can Do

- **Watch prices stream** — prices flash green (uptick) or red (downtick) with subtle CSS animations that fade
- **View sparkline mini-charts** — price action beside each ticker in the watchlist, accumulated on the frontend from the SSE stream since page load (sparklines fill in progressively)
- **Click a ticker** to see a larger detailed chart in the main chart area, pre-filled from the backend's recent price history so it is never empty
- **Buy and sell shares** — market orders only, instant fill at current price, no fees, no confirmation dialog
- **Monitor their portfolio** — a heatmap (treemap) showing positions sized by weight and colored by P&L, plus a P&L chart tracking total portfolio value over time
- **View a positions table** — ticker, quantity, average cost, current price, unrealized P&L, % change
- **Track realized P&L** — lifetime realized gains/losses from closed or reduced positions, shown in the header
- **Chat with the AI assistant** — ask about their portfolio, get analysis, and have the AI execute trades and manage the watchlist through natural language
- **Manage the watchlist** — add/remove tickers manually or via the AI chat

### Visual Design

- **Dark theme**: backgrounds around `#0d1117` or `#1a1a2e`, muted gray borders, no pure black
- **Price flash animations**: brief green/red background highlight on price change, fading over ~500ms via CSS transitions
- **Connection status indicator**: a small colored dot (green = connected, yellow = reconnecting, red = disconnected) visible in the header
- **Professional, data-dense layout**: inspired by Bloomberg/trading terminals — every pixel earns its place
- **Responsive but desktop-first**: optimized for wide screens, functional on tablet

### Color Scheme
- Accent Yellow: `#ecad0a`
- Blue Primary: `#209dd7`
- Purple Secondary: `#753991` (submit buttons)

## 3. Architecture Overview

### Single Container, Single Port

```
┌─────────────────────────────────────────────────┐
│  Docker Container (port 8000)                   │
│                                                 │
│  FastAPI (Python/uv)                            │
│  ├── /api/*          REST endpoints             │
│  ├── /api/stream/*   SSE streaming              │
│  └── /*              Static file serving         │
│                      (Next.js export)            │
│                                                 │
│  SQLite database (volume-mounted)               │
│  Background task: market data polling/sim        │
│  Background task: portfolio snapshots (30 s)     │
└─────────────────────────────────────────────────┘
```

- **Frontend**: Next.js with TypeScript, built as a static export (`output: 'export'`), served by FastAPI as static files
- **Backend**: FastAPI (Python), managed as a `uv` project
- **Database**: SQLite, single file at `<repo>/db/finally.db` (`/app/db/finally.db` in the container), volume-mounted for persistence. Path is overridable with `FINALLY_DB_PATH`.
- **Real-time data**: Server-Sent Events (SSE) — simpler than WebSockets, one-way server→client push, works everywhere
- **AI integration**: LiteLLM → OpenRouter (Cerebras for fast inference), with structured outputs for trade execution
- **Market data**: Environment-variable driven — simulator by default, real data via Massive API if key provided

### Why These Choices

| Decision | Rationale |
|---|---|
| SSE over WebSockets | One-way push is all we need; simpler, no bidirectional complexity, universal browser support |
| Static Next.js export | Single origin, no CORS issues, one port, one container, simple deployment |
| SQLite over Postgres | No auth = no multi-user = no need for a database server; self-contained, zero config |
| Single Docker container | Students run one command; no docker-compose for production, no service orchestration |
| uv for Python | Fast, modern Python project management; reproducible lockfile; what students should learn |
| Market orders only | Eliminates order book, limit order logic, partial fills — dramatically simpler portfolio math |
| Frontend derives live P&L | The header total must update on every tick, so the frontend must do the math anyway; backend returns raw positions and the frontend is the single place where valuation happens |

---

## 4. Directory Structure

```
finally/
├── frontend/                 # Next.js TypeScript project (static export)
├── backend/                  # FastAPI uv project (Python)
│   ├── app/
│   │   ├── main.py           # FastAPI app, lifespan, router + static mounting
│   │   ├── market/           # Market data subsystem (COMPLETE — see MARKET_DATA_SUMMARY.md)
│   │   ├── db/               # Schema SQL, connection helper, seed + lazy init
│   │   ├── portfolio/        # Trade execution, valuation, snapshots
│   │   ├── watchlist/        # Watchlist CRUD
│   │   └── chat/             # LLM prompt construction, structured output, mock mode
│   └── tests/
├── planning/                 # Project-wide documentation for agents
│   ├── PLAN.md               # This document
│   ├── MARKET_DATA_SUMMARY.md
│   └── archive/
├── scripts/
│   ├── start_mac.sh          # Launch Docker container (macOS/Linux)
│   ├── stop_mac.sh           # Stop Docker container (macOS/Linux)
│   ├── start_windows.ps1     # Launch Docker container (Windows PowerShell)
│   └── stop_windows.ps1      # Stop Docker container (Windows PowerShell)
├── test/                     # Playwright E2E tests + docker-compose.test.yml
├── db/                       # Volume mount target (SQLite file lives here at runtime)
│   └── .gitkeep              # Directory exists in repo; finally.db is gitignored
├── Dockerfile                # Multi-stage build (Node → Python)
├── .env                      # Environment variables (gitignored, .env.example committed)
└── .gitignore
```

The backend package layout above is a suggestion; the Backend Engineer may restructure within `backend/app/` as long as `app/market/` is left intact and the public API in §8 is honoured.

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about Python. It talks to the backend via `/api/*` endpoints and `/api/stream/*` SSE endpoints. Internal structure is up to the Frontend Engineer.
- **`backend/`** is a self-contained uv project with its own `pyproject.toml`. It owns all server logic including database initialization, schema, seed data, API routes, SSE streaming, market data, and LLM integration.
- **`backend/app/db/`** contains schema SQL definitions and seed logic. The backend lazily initializes the database on startup — creating tables and seeding default data if the SQLite file doesn't exist or is empty.
- **`db/`** at the top level is the runtime volume mount point. The SQLite file (`db/finally.db`) is created here by the backend and persists across container restarts via Docker volume.
- **`planning/`** contains project-wide documentation, including this plan. All agents reference files here as the shared contract.
- **`test/`** contains Playwright E2E tests and supporting infrastructure (e.g., `docker-compose.test.yml`). Unit tests live within `frontend/` and `backend/` respectively, following each framework's conventions.
- **`scripts/`** contains start/stop scripts that wrap Docker commands. There is no root-level `docker-compose.yml`; the scripts are the single supported way to launch.

---

## 5. Environment Variables

```bash
# Required: OpenRouter API key for LLM chat functionality
OPENROUTER_API_KEY=your-openrouter-api-key-here

# Optional: Massive (Polygon.io) API key for real market data
# If not set, the built-in market simulator is used (recommended for most users)
MASSIVE_API_KEY=

# Optional: Set to "true" for deterministic mock LLM responses (testing)
LLM_MOCK=false

# Optional: override the SQLite file location
# Default: <backend>/../db/finally.db  (resolves to /app/db/finally.db in Docker)
FINALLY_DB_PATH=
```

### Behavior

- If `MASSIVE_API_KEY` is set and non-empty → backend uses Massive REST API for market data
- If `MASSIVE_API_KEY` is absent or empty → backend uses the built-in market simulator
- If `LLM_MOCK=true` → backend returns deterministic mock LLM responses (for E2E tests)
- The backend reads configuration from **environment variables only**. In Docker they are supplied with `--env-file .env`; for local development the backend loads `<repo>/.env` via `python-dotenv` if the file exists. The `.env` file is never copied into the image.

---

## 6. Market Data

**Status: complete.** The subsystem lives in `backend/app/market/` and is documented in `planning/MARKET_DATA_SUMMARY.md` (design, test coverage) and `backend/CLAUDE.md` (API for downstream code). Both the GBM simulator and the Massive REST poller implement the same `MarketDataSource` interface and write to a shared, thread-safe `PriceCache`; all downstream code is agnostic to the source. `create_market_data_source(cache)` selects the implementation from `MASSIVE_API_KEY`.

### Two small additions required for the rest of the platform

These are the only changes the Backend Engineer should make inside `app/market/`:

1. **Session open price.** `PriceCache` records the first price seen for each ticker as `open_price`. `PriceUpdate.to_dict()` gains `open_price` and `day_change_percent` (`(price − open_price) / open_price × 100`). This is what the watchlist displays as "change %" — since process start in simulator mode, since the poller first saw the ticker in Massive mode.
2. **Recent price history.** `PriceCache` keeps a ring buffer of the last 600 `(timestamp, price)` points per ticker (~5 minutes at 500 ms). Exposed via `GET /api/prices/{ticker}/history` so the main chart is populated immediately on selection rather than starting empty.

### Ticker Tracking Rule

The data source tracks the union of the **watchlist and all open positions**. Adding a ticker to the watchlist calls `source.add_ticker()`. Removing a ticker from the watchlist calls `source.remove_ticker()` **only if there is no open position** in it; a held ticker keeps streaming so the portfolio can be valued. When a position is closed and the ticker is not on the watchlist, it is removed from the source.

### Unknown Tickers

- Malformed symbol (not matching `^[A-Z]{1,5}$` after upper-casing) → `422`
- Simulator mode: any well-formed symbol is accepted; it gets a random seed price in the $20–$500 range and default GBM parameters
- Massive mode: the symbol is validated against the API on add; unknown symbols → `404` with a clear message

### SSE Streaming

- Endpoint: `GET /api/stream/prices`
- Long-lived SSE connection; client uses native `EventSource` API. The server sends `retry: 1000` so the browser reconnects automatically after a drop.
- **One event per tick containing all tracked tickers** (not one event per ticker). An event is sent only when the cache version has changed, at most every ~500 ms:

```
data: {"AAPL": {"ticker": "AAPL", "price": 190.52, "previous_price": 190.48,
                "open_price": 189.90, "day_change_percent": 0.33,
                "change": 0.04, "change_percent": 0.02, "direction": "up",
                "timestamp": "2026-09-11T14:03:22.512Z"},
       "GOOGL": {...}, ...}
```

- The frontend treats each event as an atomic snapshot: update every ticker present, flash the ones whose `direction` is `up`/`down`, append each price to the ticker's sparkline buffer, and recompute portfolio value.

---

## 7. Database

### SQLite with Lazy Initialization

The backend checks for the SQLite database on startup. If the file doesn't exist or tables are missing, it creates the schema and seeds default data. This means:

- No separate migration step
- No manual database setup
- Fresh Docker volumes start with a clean, seeded database automatically

### Conventions

- All timestamps are ISO 8601 in **UTC with a `Z` suffix** (e.g. `2026-09-11T14:03:22.512Z`), produced and parsed that way by backend, frontend, and tests.
- Money is rounded to 2 decimal places; share quantities to 4 decimal places.
- All tables include a `user_id` column defaulting to `"default"`. It is always `"default"` in this project; do not build multi-user abstractions around it.

### Schema

**users_profile** — User state (cash balance)
- `id` TEXT PRIMARY KEY (default: `"default"`)
- `cash_balance` REAL (default: `10000.0`)
- `created_at` TEXT (ISO timestamp)

**watchlist** — Tickers the user is watching
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `added_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**positions** — Current holdings (one row per ticker per user). A row is **deleted** when its quantity reaches 0.
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `quantity` REAL (fractional shares supported)
- `avg_cost` REAL
- `updated_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**trades** — Trade history (append-only log)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `side` TEXT (`"buy"` or `"sell"`)
- `quantity` REAL (fractional shares supported)
- `price` REAL
- `realized_pnl` REAL (NULL for buys; for sells `(price − avg_cost_at_time_of_sale) × quantity`)
- `executed_at` TEXT (ISO timestamp)

**portfolio_snapshots** — Portfolio value over time (for P&L chart). Recorded every 30 seconds by a background task, and immediately after each trade execution.
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `total_value` REAL
- `recorded_at` TEXT (ISO timestamp)

**chat_messages** — Conversation history with LLM
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `role` TEXT (`"user"` or `"assistant"`)
- `content` TEXT
- `actions` TEXT (JSON — trades executed, watchlist changes made, and any errors; null for user messages)
- `created_at` TEXT (ISO timestamp)

### Default Seed Data

- One user profile: `id="default"`, `cash_balance=10000.0`
- Ten watchlist entries: AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX

---

## 8. API Endpoints

All responses are JSON. Validation errors return `422` with FastAPI's standard `{"detail": ...}` body; business-rule failures (insufficient cash, unknown ticker, etc.) return `4xx` with `{"detail": "<human-readable message>"}`.

### Market Data
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE stream of live price updates (see §6 for event shape) |
| GET | `/api/prices/{ticker}/history` | Last ≤600 `{timestamp, price}` points for one ticker, oldest first. `404` if not tracked. |

### Portfolio
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/portfolio` | `{cash_balance, realized_pnl, positions: [{ticker, quantity, avg_cost, updated_at}]}` — raw data only; the frontend computes market value, unrealized P&L and weights from live prices |
| POST | `/api/portfolio/trade` | Execute a trade: `{ticker, quantity, side}`. Returns `{trade, cash_balance, position}` (`position` is `null` if the position was closed) |
| GET | `/api/portfolio/history` | Portfolio value snapshots `[{recorded_at, total_value}]`, oldest first. `?limit=N` returns the most recent N (default 500, max 5000) |

**Trade validation rules** (identical for manual and LLM-initiated trades):
- `ticker` is upper-cased and must match `^[A-Z]{1,5}$` → else `422`
- `quantity` must be `> 0` → else `422`
- Ticker must have a cached price → else `409 "No price available for X"`
- Buy: `quantity × price` (rounded to 2 dp) must be `≤ cash_balance` → else `400 "Insufficient cash"`
- Sell: `quantity` must be `≤` held quantity → else `400 "Insufficient shares"`
- Buys update `avg_cost` as a weighted average; sells leave `avg_cost` unchanged and record `realized_pnl`
- A position whose quantity reaches 0 is deleted; if the ticker is not on the watchlist it is also removed from the data source
- Every trade writes a `portfolio_snapshots` row immediately after execution

### Watchlist
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/watchlist` | `[{ticker, added_at}]` — tickers only; prices arrive via SSE within 500 ms and the frontend shows `—` until then |
| POST | `/api/watchlist` | Add a ticker: `{ticker}`. `409` if already present; `422`/`404` per §6 unknown-ticker rules |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker. `404` if not on the watchlist. Allowed even with an open position (streaming continues, see §6) |

### Chat
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/chat` | Send `{message}`, receive `{message, trades: [...], watchlist_changes: [...], errors: [...]}` — the LLM's reply plus the actions actually executed and any that failed validation |
| GET | `/api/chat/history` | Last 50 messages `[{id, role, content, actions, created_at}]`, oldest first, for restoring the panel on page load |

### System
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | `{"status": "ok", "data_source": "simulator" \| "massive", "db": "ok", "tickers": N}`. Used by Docker healthcheck and E2E readiness polling. |

---

## 9. LLM Integration

When writing code to make calls to LLMs, use cerebras-inference skill to use LiteLLM via OpenRouter to the `openrouter/openai/gpt-oss-120b` model with Cerebras as the inference provider. Structured Outputs should be used to interpret the results.

There is an OPENROUTER_API_KEY in the .env file in the project root.

### How It Works

When the user sends a chat message, the backend:

1. Persists the user message to `chat_messages`
2. Loads the user's current portfolio context (cash, positions with current price and unrealized P&L computed server-side for the prompt only, realized P&L, watchlist with live prices, total portfolio value)
3. Loads the **last 20 messages** from `chat_messages` (the portfolio context is rebuilt on every turn since prices change)
4. Constructs a prompt with a system message, portfolio context, conversation history, and the user's new message
5. Calls the LLM via LiteLLM → OpenRouter, requesting structured output, with a **30-second timeout**
6. Parses the structured JSON response
7. Auto-executes any trades or watchlist changes specified in the response, in order, collecting per-action errors
8. Stores the assistant message with `actions` = `{trades, watchlist_changes, errors}` in `chat_messages`
9. Returns the complete JSON response to the frontend (no token-by-token streaming — Cerebras inference is fast enough that a loading indicator is sufficient)

### Failure Handling

- LLM unreachable, timeout, or response not parseable as the schema → `502 {"detail": "The assistant is unavailable: <reason>"}`. The user message stays persisted; no assistant row is written. The frontend shows the error inline in the chat panel and leaves the input populated so the user can retry.
- A trade or watchlist action that fails validation does **not** fail the request; it is recorded in `errors` (e.g. `{"action": {"ticker": "AAPL", "side": "buy", "quantity": 1000}, "error": "Insufficient cash"}`) and shown inline. The message text from the LLM is returned unchanged.

### Structured Output Schema

The LLM is instructed to respond with JSON matching this schema:

```json
{
  "message": "Your conversational response to the user",
  "trades": [
    {"ticker": "AAPL", "side": "buy", "quantity": 10}
  ],
  "watchlist_changes": [
    {"ticker": "PYPL", "action": "add"},
    {"ticker": "NFLX", "action": "remove"}
  ]
}
```

- `message` (required): The conversational text shown to the user
- `trades` (optional): Array of trades to auto-execute. Each trade goes through the same validation as manual trades (§8)
- `watchlist_changes` (optional): Array of watchlist modifications; `action` is `"add"` or `"remove"`

### Auto-Execution

Trades specified by the LLM execute automatically — no confirmation dialog. This is a deliberate design choice:
- It's a simulated environment with fake money, so the stakes are zero
- It creates an impressive, fluid demo experience
- It demonstrates agentic AI capabilities — the core theme of the course

### System Prompt Guidance

The LLM should be prompted as "FinAlly, an AI trading assistant" with instructions to:
- Analyze portfolio composition, risk concentration, and P&L
- Suggest trades with reasoning
- Execute trades when the user asks or agrees
- Manage the watchlist proactively
- Be concise and data-driven in responses
- Always respond with valid structured JSON

### LLM Mock Mode

When `LLM_MOCK=true`, the backend returns deterministic responses instead of calling OpenRouter. The mock is a small rule set over the user message (case-insensitive), evaluated in order:

| User message matches | Mock response |
|---|---|
| `buy <N> <TICKER>` | `message: "Buying N TICKER."`, `trades: [{ticker, side: "buy", quantity: N}]` |
| `sell <N> <TICKER>` | `message: "Selling N TICKER."`, `trades: [{ticker, side: "sell", quantity: N}]` |
| `add <TICKER>` | `message: "Added TICKER to your watchlist."`, `watchlist_changes: [{ticker, action: "add"}]` |
| `remove <TICKER>` | `message: "Removed TICKER from your watchlist."`, `watchlist_changes: [{ticker, action: "remove"}]` |
| anything else | `message: "Mock response: you have $<cash> cash and <n> positions."`, no actions |

Actions from the mock go through the normal execution and validation path, so E2E tests exercise the real trade code. This enables fast, free, reproducible E2E tests, development without an API key, and CI pipelines.

---

## 10. Frontend Design

### Layout

The frontend is a single-page application with a dense, terminal-inspired layout. The specific component architecture and layout system is up to the Frontend Engineer, but the UI should include these elements:

- **Watchlist panel** — grid/table of watched tickers with: ticker symbol, current price (flashing green/red on change), `day_change_percent` from the SSE payload, and a sparkline mini-chart (accumulated from SSE since page load). Remove-ticker control per row and an add-ticker input.
- **Main chart area** — larger chart for the currently selected ticker. On selection, fetch `GET /api/prices/{ticker}/history` to seed the chart, then append live SSE points. Clicking a ticker in the watchlist or positions table selects it here.
- **Portfolio heatmap** — treemap visualization where each rectangle is a position, sized by portfolio weight, colored by unrealized P&L (green = profit, red = loss)
- **P&L chart** — line chart showing total portfolio value over time, seeded from `GET /api/portfolio/history` and extended live: append the frontend-computed total value on each SSE tick
- **Positions table** — tabular view of all positions: ticker, quantity, avg cost, current price, market value, unrealized P&L, % change. All derived columns are computed on the frontend from live prices.
- **Trade bar** — simple input area: ticker field, quantity field, buy button, sell button. Market orders, instant fill. Validation errors from the API shown inline.
- **AI chat panel** — docked/collapsible sidebar. Message input, scrolling conversation history (restored from `GET /api/chat/history` on load), loading indicator while waiting for LLM response. Executed trades, watchlist changes, and action errors shown inline as confirmations/warnings.
- **Header** — portfolio total value (`cash + Σ quantity × live price`, updating on every tick), cash balance, realized P&L, connection status indicator

### Valuation Rule

The frontend is the single place where live valuation happens. It holds the raw portfolio (`cash_balance`, `realized_pnl`, positions with `quantity`/`avg_cost`) and the latest price map from SSE, and derives market value, unrealized P&L, P&L %, weight, and total value in one memoized selector shared by the header, positions table, heatmap, and P&L chart. After any trade (manual or via chat) the frontend re-fetches `GET /api/portfolio`.

### Charting

- **Lightweight Charts** (canvas) for the main price chart and the P&L line chart
- **Sparklines**: a hand-rolled inline `<svg>` polyline component — no extra library
- **Treemap**: `d3-hierarchy` for layout, rendered as absolutely-positioned `<div>`s styled with Tailwind — no Recharts

### Technical Notes

- Use `EventSource` for SSE connection to `/api/stream/prices`; map `onopen` → green, `onerror` while `readyState === CONNECTING` → yellow, `CLOSED` → red for the status dot
- Price flash effect: on receiving a new price, briefly apply a CSS class with background color transition, then remove it
- All API calls go to the same origin (`/api/*`) — no CORS configuration needed
- Tailwind CSS for styling with a custom dark theme

### Local Development (without Docker)

Run the backend with `cd backend && uv run uvicorn app.main:app --reload --port 8000` and the frontend with `cd frontend && npm run dev` (port 3000). `next.config.ts` includes a `rewrites()` entry proxying `/api/:path*` to `http://localhost:8000/api/:path*` in development only (rewrites are ignored by static export), so the frontend code always uses relative `/api/*` URLs and there is no CORS configuration. Next.js dev-server rewrites proxy SSE correctly.

---

## 11. Docker & Deployment

### Multi-Stage Dockerfile

```
Stage 1: Node 20 slim
  - Copy frontend/
  - npm ci && npm run build (produces static export in frontend/out)

Stage 2: Python 3.12 slim
  - Install uv
  - Copy backend/ to /app/backend
  - uv sync --frozen (install Python dependencies from lockfile)
  - Copy frontend/out into /app/backend/static/
  - WORKDIR /app/backend   (so the default DB path ../db/finally.db resolves to /app/db/finally.db)
  - Expose port 8000
  - HEALTHCHECK: curl -f http://localhost:8000/api/health
  - CMD: uv run uvicorn app.main:app --host 0.0.0.0 --port 8000
```

FastAPI serves the static frontend files and all API routes on port 8000. **Register all `/api` routers before mounting the static directory at `/`**, otherwise the static mount shadows the API. Serve `static/index.html` for `/`, and files under `static/` (including `_next/*`) for everything else; the export's `404.html` serves unknown paths.

### Docker Volume

The SQLite database persists via a named Docker volume:

```bash
docker run -d --name finally -v finally-data:/app/db -p 8000:8000 --env-file .env finally
```

The `/app/db` directory in the container is the volume mount point; the backend writes `finally.db` there. The `.env` file is passed with `--env-file` and is not baked into the image.

### Start/Stop Scripts

**`scripts/start_mac.sh`** (macOS/Linux):
- Exits with a clear message if Docker is not running
- Builds the Docker image if not already built (or if `--build` flag passed)
- If a container named `finally` already exists, stops and removes it first (idempotent)
- Runs the container with the volume mount, port mapping, and `--env-file .env`
- Prints the URL to access the app and opens the browser (`open`/`xdg-open`)

**`scripts/stop_mac.sh`** (macOS/Linux):
- Stops and removes the running container; no error if it is not running
- Does NOT remove the volume (data persists)

**`scripts/start_windows.ps1`** / **`scripts/stop_windows.ps1`**: PowerShell equivalents for Windows. Check Docker Desktop with `docker info`, inspect existing containers with `docker ps -a --filter name=finally`, open the browser with `Start-Process`.

All scripts should be idempotent — safe to run multiple times.

### Optional Cloud Deployment

The container is designed to deploy to AWS App Runner, Render, or any container platform. A Terraform configuration for App Runner may be provided in a `deploy/` directory as a stretch goal, but is not part of the core build.

---

## 12. Testing Strategy

### Unit Tests (within `frontend/` and `backend/`)

**Backend (pytest)** — the primary home for logic tests:
- Market data: already complete (73 tests); add coverage for `open_price`/`day_change_percent` and the history ring buffer
- Portfolio: trade execution, weighted average cost, realized P&L on sells, position deletion at zero, every validation rule in §8, snapshot written after trade
- Watchlist: add/remove, `409` on duplicate, data-source sync rule (held ticker keeps streaming)
- LLM: structured output parsing, malformed response → `502`, action errors collected without failing the request, mock mode rules
- API routes: status codes and response shapes for every endpoint in §8, using a temporary SQLite file and the simulator

**Frontend (Vitest + React Testing Library)** — pure logic only; UI behaviour is covered by E2E:
- Valuation selector: market value, unrealized P&L, weights, total value from positions + price map
- Sparkline buffer: append, cap length, handles first point
- SSE event reducer: applies a batched event to the price map and flags direction

### E2E Tests (in `test/`)

**Infrastructure**: A separate `docker-compose.test.yml` in `test/` that spins up the app container plus a Playwright container. This keeps browser dependencies out of the production image. Tests wait on `GET /api/health` before starting.

**Environment**: Tests run with `LLM_MOCK=true` and a fresh (non-persistent) database volume for determinism.

**Key Scenarios**:
- Fresh start: default watchlist appears, $10k balance shown, prices are streaming, status dot is green
- Add and remove a ticker from the watchlist
- Buy shares: cash decreases, position appears, header total updates, heatmap shows the position
- Sell shares: cash increases, realized P&L updates, position updates or disappears
- Portfolio visualization: heatmap renders with correct colors, P&L chart has data points
- AI chat (mocked): send `buy 1 AAPL`, receive the response, trade confirmation appears inline, position table updates
- AI chat error path (mocked): `buy 100000 AAPL` shows an inline "Insufficient cash" error
- SSE resilience: block the stream, verify the dot turns yellow, unblock, verify it returns to green and prices resume

---

## 13. Build Order

Given the current state (market data complete; no app, DB, frontend, or scripts), work proceeds in this order. Each step yields a contract the next depends on.

1. **Backend skeleton** — `app/main.py` with lifespan (create `PriceCache`, start data source with watchlist ∪ positions, start 30 s snapshot task), DB lazy init + seed, `/api/health`, mount the stream router. Add `open_price` and the history ring buffer to `app/market/`.
2. **Portfolio + watchlist endpoints** with pytest coverage — pure logic, no LLM.
3. **Chat endpoint** with `LLM_MOCK=true` first, then the real LiteLLM/OpenRouter path via the cerebras-inference skill.
4. **Frontend** against the running backend — steps 1–3 provide the complete API contract.
5. **Dockerfile + scripts**, then Playwright E2E.

---

## Appendix: Design Decisions Log

Resolved during the 2026-09-11 plan review; recorded so later agents understand why the spec reads as it does.

| # | Decision | Rationale |
|---|---|---|
| 1 | SSE sends one batched event per tick | Matches the implemented `stream.py`; fewer events, atomic snapshot |
| 2 | "Change %" = change since session open (first price seen) | Simulator has no prior close; cheapest option that is meaningful |
| 3 | Data source tracks watchlist ∪ positions | Held tickers must keep streaming for valuation |
| 4 | Simulator accepts any `^[A-Z]{1,5}$`; Massive validates via API | Simulator can invent prices, Massive cannot |
| 5 | Realized P&L is in scope | Stored per sell on `trades.realized_pnl`; total exposed on `/api/portfolio`; shown in header |
| 6 | Frontend computes all live valuation; backend returns raw positions | Header must update per tick anyway; one source of truth for the math |
| 7 | Backend keeps 600-tick price history per ticker | Main chart is populated immediately on selection |
| 8 | Snapshot history `?limit=` default 500, max 5000; separate asyncio task | Bounded payload; keeps `app/market/` untouched |
| 9 | Chat: last 20 messages in prompt, 30 s timeout, `502` on LLM failure, action errors returned not raised | Predictable failure surface for the UI |
| 10 | Mock LLM is a regex rule set | Deterministic E2E that still exercises real trade code |
| 11 | UTC `Z` timestamps; 2 dp money, 4 dp shares | Cross-language agreement |
| 12 | DB path defaults to `<backend>/../db/finally.db`, `WORKDIR /app/backend` in Docker, `FINALLY_DB_PATH` override | Same relative path works locally and in the container |
| 13 | `.env` via `--env-file` only; `python-dotenv` for local dev | Never bake secrets into the image |
| 14 | No root `docker-compose.yml` | One way to launch; scripts wrap `docker run` |
| 15 | `user_id` column kept, multi-user rationale dropped | Harmless column; avoids speculative abstractions |
| 16 | Lightweight Charts + SVG sparklines + d3-hierarchy treemap | One chart library; treemap layout without pulling in Recharts |
| 17 | Frontend unit tests limited to pure logic | E2E covers UI behaviour; avoids duplicate test surface |
| 18 | `GET /api/chat/history` added | Chat panel must survive a page reload |
