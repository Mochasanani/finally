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
- **Click a ticker** to see a larger detailed chart in the main chart area
- **Buy and sell shares** — market orders only, instant fill at current price, no fees, no confirmation dialog
- **Monitor their portfolio** — a heatmap (treemap) showing positions sized by weight and colored by P&L, plus a P&L chart tracking total portfolio value over time
- **View a positions table** — ticker, quantity, average cost, current price, unrealized P&L, % change
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
└─────────────────────────────────────────────────┘
```

- **Frontend**: Next.js with TypeScript, built as a static export (`output: 'export'`), served by FastAPI as static files
- **Backend**: FastAPI (Python), managed as a `uv` project
- **Database**: SQLite, single file at `db/finally.db`, volume-mounted for persistence
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

---

## 4. Directory Structure

```
finally/
├── frontend/                 # Next.js TypeScript project (static export)
├── backend/                  # FastAPI uv project (Python)
├── planning/                 # Project-wide documentation for agents
│   ├── PLAN.md               # This document
│   └── ...                   # Additional agent reference docs
├── scripts/
│   ├── start_mac.sh          # Launch Docker container (macOS/Linux)
│   └── stop_mac.sh           # Stop Docker container (macOS/Linux)
├── test/                     # Playwright E2E tests + docker-compose.test.yml
├── Dockerfile                # Multi-stage build (Node → Python)
├── .env                      # Environment variables (gitignored, .env.example committed)
└── .gitignore
```

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about Python. It talks to the backend via `/api/*` endpoints and `/api/stream/*` SSE endpoints. Internal structure is up to the Frontend Engineer agent.
- **`backend/`** is a self-contained uv project with its own `pyproject.toml`. It owns all server logic including database initialization (a single `CREATE TABLE IF NOT EXISTS` bootstrap module — no migrations), schema, seed data, API routes, SSE streaming, market data, and LLM integration. Internal structure is up to the Backend/Market Data agents.
- **SQLite file location.** The DB lives inside the container at `/app/db/finally.db` and is persisted via a named Docker volume (`finally-data`). There is no `db/` directory checked into the repo — the named volume handles persistence. For local development outside Docker, the backend creates `finally.db` at a path configured by an env var (defaulting to the backend project root); this local file is gitignored.
- **`planning/`** contains project-wide documentation, including this plan. All agents reference files here as the shared contract.
- **`test/`** contains Playwright E2E tests and supporting infrastructure (e.g., `docker-compose.test.yml`). Backend unit tests live within `backend/tests/`. Frontend unit tests are out of scope for v1 — Playwright E2E is the only frontend test layer.
- **`scripts/`** contains start/stop scripts that wrap Docker commands. macOS/Linux only for v1; Windows PowerShell scripts are a follow-up.

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
```

### Behavior

- If `MASSIVE_API_KEY` is set and non-empty → backend uses Massive REST API for market data
- If `MASSIVE_API_KEY` is absent or empty → backend uses the built-in market simulator
- If `LLM_MOCK=true` → backend returns deterministic mock LLM responses (for E2E tests); `OPENROUTER_API_KEY` is **not** required in this mode
- If `LLM_MOCK` is unset or `false` → `OPENROUTER_API_KEY` is required at startup and the backend fails fast with a clear error if missing
- The backend reads `.env` from the project root (mounted into the container or read via docker `--env-file`)

---

## 6. Market Data

### Two Implementations, One Interface

Both the simulator and the Massive client implement the same abstract interface. The backend selects which to use based on the environment variable. All downstream code (SSE streaming, price cache, frontend) is agnostic to the source.

### Simulator (Default)

- Generates prices using geometric Brownian motion (GBM) with configurable drift and volatility per ticker
- Updates at ~500ms intervals
- Correlated moves across tickers (e.g., tech stocks move together)
- Occasional random "events" — sudden 2-5% moves on a ticker for drama
- Starts from realistic seed prices (e.g., AAPL ~$190, GOOGL ~$175, etc.)
- Runs as an in-process background task — no external dependencies

### Massive API (Optional)

- REST API polling (not WebSocket) — simpler, works on all tiers
- Polls for the union of all watched tickers on a configurable interval
- Free tier (5 calls/min): poll every 15 seconds
- Paid tiers: poll every 2-15 seconds depending on tier
- Parses REST response into the same format as the simulator
- **Market-closed behavior** is intentional: the simulator always ticks, while Massive returns stale prices until the market reopens. Both are valid — the frontend does not need to distinguish.

### Shared Price Cache

- A single background task (simulator or Massive poller) writes to an in-memory price cache
- The cache holds the latest price, previous price, a `session_open` price (the first price observed since the backend process started, used as the reference for daily change %), and a timestamp for each ticker
- SSE streams read from this cache and push updates to connected clients
- This architecture supports future multi-user scenarios without changes to the data layer

### SSE Streaming

- Endpoint: `GET /api/stream/prices`
- Long-lived SSE connection; client uses native `EventSource` API
- Server pushes price updates for all tickers known to the system at a regular cadence (~500ms) — in the single-user model this is equivalent to the user's watchlist
- Uses **version-based change detection**: events fire only when the cached price actually changes. With Massive on the free tier that means roughly one burst every 15s; with the simulator it is roughly every 500ms. The frontend should not assume a fixed cadence.
- Each SSE event contains ticker, price, previous price, session_open price, timestamp, and change direction
- Client handles reconnection automatically (EventSource has built-in retry)

---

## 7. Database

### SQLite with Startup Initialization

At application startup the backend opens the SQLite database. If tables are missing, it creates the schema and seeds default data in a single bootstrap step. This means:

- No separate migration step and no `CREATE`-on-first-request branch
- No manual database setup
- Fresh Docker volumes start with a clean, seeded database automatically

### Schema

Single-user for v1 — no `user_id` columns. Multi-user is explicitly out of scope; adding a column later is a one-line `ALTER TABLE` if we ever need it.

The starting cash balance is a constant (`STARTING_CASH = 10_000.0` in backend config), not a stored column. `cash_balance` in `users_profile` reflects the current balance; the baseline for total-P&L calculations is the constant.

**users_profile** — User state (singleton row)
- `id` TEXT PRIMARY KEY (always `"default"`)
- `cash_balance` REAL (default: `10000.0`)
- `created_at` TEXT (ISO timestamp)

**watchlist** — Tickers the user is watching
- `id` TEXT PRIMARY KEY (UUID)
- `ticker` TEXT UNIQUE
- `added_at` TEXT (ISO timestamp)

**positions** — Current holdings (one row per ticker)
- `id` TEXT PRIMARY KEY (UUID)
- `ticker` TEXT UNIQUE
- `quantity` REAL (fractional shares supported)
- `avg_cost` REAL
- `updated_at` TEXT (ISO timestamp)

**trades** — Trade history (append-only log)
- `id` TEXT PRIMARY KEY (UUID)
- `ticker` TEXT
- `side` TEXT (`"buy"` or `"sell"`)
- `quantity` REAL (fractional shares supported)
- `price` REAL
- `executed_at` TEXT (ISO timestamp)

**portfolio_snapshots** — Portfolio value over time (for P&L chart). Recorded every 30 seconds by a background task and immediately after each trade execution. A retention job trims the table to the most recent 10,000 rows on each insert (plenty of resolution for the P&L chart; keeps the DB bounded).
- `id` TEXT PRIMARY KEY (UUID)
- `total_value` REAL
- `recorded_at` TEXT (ISO timestamp)

No `chat_messages` table in v1. Conversation history is held in frontend state and passed back with each `POST /api/chat` request (see §9). This removes history-pagination, context-window truncation, and a separate `GET /api/chat/history` endpoint from scope.

### Default Seed Data

- One user profile: `id="default"`, `cash_balance=10000.0`
- Ten watchlist entries: AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX

---

## 8. API Endpoints

### Market Data
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE stream of live price updates |

### Portfolio
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/portfolio` | Positions (each enriched with the current cached price, unrealized P&L, and % change from avg cost), cash balance, total portfolio value |
| POST | `/api/portfolio/trade` | Execute a trade: `{ticker, quantity, side}` |
| GET | `/api/portfolio/history` | Portfolio value snapshots over time (for P&L chart) |

### Watchlist
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/watchlist` | Current watchlist tickers, each enriched with the latest cached price, session_open price, and computed daily change % |
| POST | `/api/watchlist` | Add a ticker: `{ticker}` |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker |

### Chat
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/chat` | Send a message plus recent history, receive complete JSON response (message + executed actions). Request body: `{message: string, history: [{role, content}]}` — frontend supplies the last ~20 turns; backend does not persist chat. |

### System
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | Health check (for Docker/deployment) |

### Error responses

All error responses follow the FastAPI convention: HTTP 4xx/5xx with a JSON body `{"detail": "<human-readable message>"}`. Trade validation failures return `400`; not-found returns `404`; unexpected errors return `500`. The frontend reads `detail` and surfaces it to the user.

### Input validation

- Trade `quantity` must be a positive float with a minimum of `0.0001` and a sanity maximum of `1_000_000` shares.
- `ticker` is uppercased and must match `^[A-Z]{1,10}$`.
- `side` must be exactly `"buy"` or `"sell"`.

### Rate limiting

None in v1. The app is designed for localhost use. If deployed publicly, add a reverse proxy or CDN in front — the application layer does not enforce rate limits.

---

## 9. LLM Integration

When writing code to make calls to LLMs, use cerebras-inference skill to use LiteLLM via OpenRouter to the `openrouter/openai/gpt-oss-120b` model with Cerebras as the inference provider. Structured Outputs should be used to interpret the results.

There is an OPENROUTER_API_KEY in the .env file in the project root.

### How It Works

When the user sends a chat message, the backend:

1. Loads the user's current portfolio context (cash, positions with P&L, watchlist with live prices, total portfolio value)
2. Takes the `history` array supplied in the request (frontend-managed, capped at the last 20 turns)
3. Constructs a prompt with a system message, portfolio context, the supplied history, and the new user message
4. Calls the LLM via LiteLLM → OpenRouter, requesting structured output, using the cerebras-inference skill
5. Parses the complete structured JSON response
6. Auto-executes any trades or watchlist changes specified in the response, collecting a per-action status (success or error detail)
7. Returns the complete JSON response plus the executed-action results to the frontend (no token-by-token streaming — Cerebras inference is fast enough that a loading indicator is sufficient)

A single in-process `asyncio.Lock` serializes chat execution so a double-submit cannot cause two concurrent trade flows to race on the cash balance.

### Structured Output Schema

The LLM is instructed to respond with JSON matching this schema:

```json
{
  "message": "Your conversational response to the user",
  "trades": [
    {"ticker": "AAPL", "side": "buy", "quantity": 10}
  ],
  "watchlist_changes": [
    {"ticker": "PYPL", "action": "add"}
  ]
}
```

- `message` (required): The conversational text shown to the user
- `trades` (optional): Array of trades to auto-execute. Each trade goes through the same validation as manual trades (sufficient cash for buys, sufficient shares for sells, quantity bounds from §8).
- `watchlist_changes` (optional): Array of watchlist modifications. `action` is an enum and must be exactly `"add"` or `"remove"`.

### API Response Shape

`POST /api/chat` returns:

```json
{
  "message": "Bought 10 AAPL for you.",
  "executed_actions": [
    {"type": "trade", "status": "ok", "detail": "Bought 10 AAPL @ $190.24"},
    {"type": "watchlist", "status": "error", "detail": "PYPL already on watchlist"}
  ]
}
```

`executed_actions` is the authoritative record the frontend renders inline — the LLM's `trades`/`watchlist_changes` are its *request*, and this array is the server's actual outcome. The frontend appends the assistant message + executed actions to its local history for the next turn.

### Auto-Execution

Trades specified by the LLM execute automatically — no confirmation dialog. This is a deliberate design choice:
- It's a simulated environment with fake money, so the stakes are zero
- It creates an impressive, fluid demo experience
- It demonstrates agentic AI capabilities — the core theme of the course

If a trade fails validation (e.g., insufficient cash), the error is captured in the `executed_actions` array and the LLM's message can reference it.

### System Prompt Guidance

The LLM should be prompted as "FinAlly, an AI trading assistant" with instructions to:
- Analyze portfolio composition, risk concentration, and P&L
- Suggest trades with reasoning
- Execute trades when the user asks or agrees
- Manage the watchlist proactively
- Be concise and data-driven in responses
- Always respond with valid structured JSON

### LLM Mock Mode

When `LLM_MOCK=true`, the backend returns deterministic mock responses instead of calling OpenRouter. This enables:
- Fast, free, reproducible E2E tests
- Development without an API key
- CI/CD pipelines

---

## 10. Frontend Design

### Layout

The frontend is a single-page application with a dense, terminal-inspired layout. The specific component architecture and layout system is up to the Frontend Engineer, but the UI should include these elements:

- **Watchlist panel** — grid/table of watched tickers with: ticker symbol, current price (flashing green/red on change), daily change % (current price vs. the backend's `session_open`), and a sparkline mini-chart. The sparkline is accumulated from the SSE stream since the current page load and is reset on refresh — this is expected behavior, not a bug.
- **Main chart area** — larger chart for the currently selected ticker, with at minimum price over time. Clicking a ticker in the watchlist selects it; the selected ticker is persisted in the URL as `?symbol=AAPL` so deep links and refreshes land on the same chart.
- **Portfolio heatmap** — treemap visualization where each rectangle is a position, sized by portfolio weight, colored by P&L (green = profit, red = loss)
- **P&L chart** — line chart showing total portfolio value over time, using data from `portfolio_snapshots`
- **Positions table** — tabular view of all positions: ticker, quantity, avg cost, current price, unrealized P&L, % change
- **Trade bar** — simple input area: ticker field, quantity field, buy button, sell button. Market orders, instant fill.
- **AI chat panel** — docked/collapsible sidebar. Message input, scrolling conversation history held in frontend state, loading indicator while waiting for LLM response. The frontend sends the last ~20 turns of history with each request. Conversation is not persisted across page reloads. Executed trades and watchlist changes from the `executed_actions` array are rendered inline as confirmation rows.
- **Header** — portfolio total value (updating live), connection status indicator, cash balance

### Technical Notes

- Use `EventSource` for SSE connection to `/api/stream/prices`
- **Connection status indicator** is derived from `EventSource.readyState`: `OPEN` → green, `CONNECTING` → yellow (covers both initial connect and automatic reconnection after a drop), `CLOSED` → red. No custom reconnection logic is needed — `EventSource` reconnects automatically.
- **Charting**: use `lightweight-charts` for all chart surfaces (sparklines, main chart, P&L line). One dependency, canvas-backed, matches the terminal aesthetic.
- Price flash effect: on receiving a new price, briefly apply a CSS class with background color transition, then remove it
- All API calls go to the same origin (`/api/*`) — no CORS configuration needed
- Tailwind CSS for styling with a custom dark theme

---

## 11. Docker & Deployment

### Multi-Stage Dockerfile

```
Stage 1: Node 20 slim
  - Copy frontend/
  - npm install && npm run build (produces static export)

Stage 2: Python 3.12 slim
  - Install uv
  - Copy backend/ into /app
  - uv sync (install Python dependencies from lockfile)
  - Copy frontend build output into /app/static
  - Expose port 8000
  - CMD: uvicorn serving FastAPI app
```

FastAPI mounts `/app/static` as the static file root and serves all `/api/*` routes on the same port 8000.

### Docker Volume

The SQLite database persists via a **named Docker volume** (`finally-data`) mounted at `/app/db` in the container. The backend writes `/app/db/finally.db`:

```bash
docker run -v finally-data:/app/db -p 8000:8000 --env-file .env finally
```

No bind-mount to a host `db/` directory — the named volume is the single source of truth for persisted state. Named volumes are easier to manage across platforms and avoid host-filesystem permission issues.

### Start/Stop Scripts

**`scripts/start_mac.sh`** (macOS/Linux):
- Builds the Docker image if not already built (or if `--build` flag passed)
- Runs the container with the named volume, port mapping, and `.env` file
- Prints the URL to access the app
- Optionally opens the browser

**`scripts/stop_mac.sh`** (macOS/Linux):
- Stops and removes the running container
- Does NOT remove the volume (data persists)

All scripts should be idempotent — safe to run multiple times. Windows PowerShell scripts are a follow-up and not part of v1.

### Public deployment notes

If the container is deployed to a publicly reachable URL, add a `robots.txt` with `User-agent: *` / `Disallow: /` to the static assets to avoid accidental indexing of the demo. This is a one-line addition in the Dockerfile.

### Optional Cloud Deployment

The container is designed to deploy to AWS App Runner, Render, or any container platform. A Terraform configuration for App Runner may be provided in a `deploy/` directory as a stretch goal, but is not part of the core build.

---

## 12. Testing Strategy

### Backend Unit Tests (`backend/tests/`, pytest)

- Market data: simulator generates valid prices, GBM math is correct, Massive API response parsing works, both implementations conform to the abstract interface
- Portfolio: trade execution logic, P&L calculations, edge cases (selling more than owned, buying with insufficient cash, selling at a loss, quantity bounds from §8)
- LLM: structured output parsing handles all valid schemas, graceful handling of malformed responses, trade validation within chat flow, per-action status reporting in `executed_actions`
- API routes: correct status codes, `{"detail": "..."}` error shape, response schemas

Frontend unit tests are out of scope for v1 — Playwright E2E is the only frontend test layer.

### E2E Tests (in `test/`)

**Infrastructure**: A separate `docker-compose.test.yml` in `test/` that spins up the app container plus a Playwright container. This keeps browser dependencies out of the production image. The app container in the test compose uses a `tmpfs` mount at `/app/db` (no named volume) so every test run starts from a fresh, seeded database — state never leaks between runs.

**Environment**: Tests run with `LLM_MOCK=true` by default for speed and determinism. No `OPENROUTER_API_KEY` is required in CI.

**Key Scenarios**:
- Fresh start: default watchlist appears, $10k balance shown, prices are streaming
- Add and remove a ticker from the watchlist
- Buy shares: cash decreases, position appears, portfolio updates
- Sell shares: cash increases, position updates or disappears
- Portfolio visualization: heatmap renders with correct colors, P&L chart has data points
- AI chat (mocked): send a message, receive a response, `executed_actions` rendered inline as confirmation rows
- SSE resilience: disconnect and verify reconnection via `EventSource.readyState` transitions
