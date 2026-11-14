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
- **Buy and sell shares** — market orders only, instant fill at current price, no fees, no confirmation dialog. Fractional quantities are supported; there are no market hours, no short selling and no margin
- **Monitor their portfolio** — a heatmap (treemap) showing positions sized by weight and colored by P&L, plus a P&L chart tracking total portfolio value over time
- **View a positions table** — ticker, quantity, average cost, current price, unrealized P&L, % change. Fractional holdings are made obvious (quantity shown with its decimals plus a small "fractional" badge) so the user can manage them, e.g. sell the remainder
- **Chat with the AI assistant** — ask about their portfolio, get analysis, and have the AI execute trades and manage the watchlist through natural language
- **Manage the watchlist** — add/remove tickers manually or via the AI chat. A ticker with an open position cannot be removed until the position is sold

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
│   └── db/                   # Schema definitions, seed data, migration logic
├── planning/                 # Project-wide documentation for agents
│   ├── PLAN.md               # This document
│   └── ...                   # Additional agent reference docs
├── scripts/
│   ├── start.py              # Launch Docker container (cross-platform; run with `uv run scripts/start.py`)
│   └── stop.py               # Stop Docker container (cross-platform)
├── test/                     # Playwright E2E tests + docker-compose.test.yml
├── db/                       # Volume mount target (SQLite file lives here at runtime)
│   └── .gitkeep              # Directory exists in repo; finally.db is gitignored
├── Dockerfile                # Multi-stage build (Node → Python)
├── .env                      # Environment variables (gitignored, .env.example committed)
└── .gitignore
```

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about Python. It talks to the backend via `/api/*` endpoints and `/api/stream/*` SSE endpoints. Internal structure is up to the Frontend Engineer agent.
- **`backend/`** is a self-contained uv project with its own `pyproject.toml`. It owns all server logic including database initialization, schema, seed data, API routes, SSE streaming, market data, and LLM integration. Internal structure is up to the Backend/Market Data agents.
- **`backend/db/`** contains schema SQL definitions and seed logic. The backend initializes the database once at startup (FastAPI lifespan) — creating tables and seeding default data if the SQLite file doesn't exist or is empty.
- **`db/`** at the top level is the runtime volume mount point. The SQLite file (`db/finally.db`) is created here by the backend and persists across container restarts via Docker volume.
- **`planning/`** contains project-wide documentation, including this plan. All agents reference files here as the shared contract.
- **`test/`** contains Playwright E2E tests and supporting infrastructure (e.g., `docker-compose.test.yml`). Unit tests live within `frontend/` and `backend/` respectively, following each framework's conventions.
- **`scripts/`** contains two cross-platform start/stop scripts that wrap Docker commands.

---

## 5. Environment Variables

```bash
# Required: OpenRouter API key for LLM chat functionality
# (copy the committed `.env.example` to `.env`, which is gitignored)
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
- If `LLM_MOCK=true` → backend returns deterministic mock LLM responses (for E2E tests)
- In Docker, variables are supplied with `--env-file .env`; the backend reads only process environment variables and `.env` is never copied into the image. For local (non-Docker) development, the backend loads the project-root `.env` if present
- `.env.example` is committed with the variables above and placeholder values

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
- Supports a **known universe** of tickers defined in `seed_prices.py`: the 10 defaults plus a curated list of ~50 common US tickers (including PYPL, used by tests). Tickers outside the universe are rejected, never given random prices
- Runs as an in-process background task — no external dependencies

### Massive API (Optional)

- REST API polling (not WebSocket) — simpler, works on all tiers
- Polls for all watchlist tickers on a configurable interval
- Free tier (5 calls/min): poll every 15 seconds
- Paid tiers: poll every 2-15 seconds depending on tier
- Parses REST response into the same format as the simulator
- Ticker validation: a ticker is valid if the API returns a price for it; otherwise it is rejected

### Shared Price Cache

- A single background task (simulator or Massive poller) writes to an in-memory price cache
- The cache holds the latest price, previous price, and timestamp for each ticker
- The tracked tickers are exactly the watchlist. Because a held ticker cannot be removed from the watchlist, every position always has a live price
- SSE streams read from this cache and push updates to connected clients
- This architecture supports future multi-user scenarios without changes to the data layer

### SSE Streaming

- Endpoint: `GET /api/stream/prices`
- Long-lived SSE connection; client uses native `EventSource` API
- Server pushes price updates for all tickers known to the system at a regular cadence (~500ms) — in the single-user model this is equivalent to the user's watchlist
- One SSE event per ticker update (not batched). Each event's `data` is JSON: `{"ticker": "AAPL", "price": 190.12, "previous_price": 190.05, "ref_price": 188.40, "timestamp": 1731600000.5, "direction": "up"}` where `direction` is `up`, `down` or `flat`
- `ref_price` is the prior trading day's close for the ticker's market, and is the baseline for "daily change %", computed by the client as `(price - ref_price) / ref_price`. For Massive it is the previous close returned by the API. The simulator has no real trading days, so it treats each ticker's seed price as the prior close
- Client handles reconnection automatically (EventSource has built-in retry)

---

## 7. Database

### SQLite with Startup Initialization

The backend checks for the SQLite database once at startup. If the file doesn't exist or tables are missing, it creates the schema and seeds default data. This means:

- No separate migration step
- No manual database setup
- Fresh Docker volumes start with a clean, seeded database automatically

### Schema

**Money is stored as integer cents** (`*_cents` INTEGER columns) to avoid floating-point drift. Share quantities remain REAL (fractional shares). A trade's total is `round(quantity * price_cents)`. The in-memory price cache and the API use dollar values, converted to/from cents at the database boundary.

All tables include a `user_id` column defaulting to `"default"`. This is hardcoded for now (single-user) but enables future multi-user support without schema migration.

**users_profile** — User state (cash balance)
- `id` TEXT PRIMARY KEY (default: `"default"`)
- `cash_balance_cents` INTEGER (default: `1000000`)
- `created_at` TEXT (ISO timestamp)

**watchlist** — Tickers the user is watching
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `added_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**positions** — Current holdings (one row per ticker per user). A row is deleted only when its quantity reaches exactly zero
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `quantity` REAL (fractional shares supported)
- `avg_cost_cents` INTEGER (unchanged by partial sells)
- `updated_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**trades** — Trade history (append-only log)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `side` TEXT (`"buy"` or `"sell"`)
- `quantity` REAL (fractional shares supported)
- `price_cents` INTEGER
- `executed_at` TEXT (ISO timestamp)

**portfolio_snapshots** — Portfolio value over time (for P&L chart). Recorded by a background task every 30 seconds (only while at least one position exists), and immediately after each trade execution.
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `total_value_cents` INTEGER
- `recorded_at` TEXT (ISO timestamp)

**chat_messages** — Conversation history with LLM
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `role` TEXT (`"user"` or `"assistant"`)
- `content` TEXT
- `actions` TEXT (JSON — trades executed, watchlist changes made; null for user messages)
- `created_at` TEXT (ISO timestamp)

### Default Seed Data

- One user profile: `id="default"`, `cash_balance_cents=1000000` ($10,000.00)
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
| GET | `/api/portfolio` | Current positions, cash balance, total value, unrealized P&L |
| POST | `/api/portfolio/trade` | Execute a trade: `{ticker, quantity, side}` |
| GET | `/api/portfolio/history` | Portfolio value snapshots over time (for P&L chart) |

### Watchlist
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/watchlist` | Current watchlist tickers with latest prices |
| POST | `/api/watchlist` | Add a ticker: `{ticker}` |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker. Returns 409 if the user holds a position in it |

### Chat
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/chat` | Send a message, receive complete JSON response (message + executed actions) |

### System
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | Health check (for Docker/deployment) |

### Conventions and Response Shapes

- All money values in requests and responses are dollars (numbers, 2dp); quantities are numbers (fractional allowed)
- Errors use the appropriate HTTP status (400 validation, 404 not found, 409 conflict) with body `{"error": "<human-readable message>"}`. Unknown tickers return 400 `Unknown ticker: XYZ`
- `GET /api/portfolio` → `{"cash": 10000.00, "total_value": 10250.50, "unrealized_pnl": 250.50, "positions": [{"ticker", "quantity", "avg_cost", "current_price", "market_value", "unrealized_pnl", "pnl_pct"}]}`
- `POST /api/portfolio/trade` `{"ticker", "quantity", "side"}` → the updated portfolio (same shape as `GET /api/portfolio`) on success. `quantity` must be > 0
- `GET /api/portfolio/history` → `{"snapshots": [{"recorded_at", "total_value"}]}` covering the last 24 hours, downsampled server-side to at most 1,000 points
- `GET /api/watchlist` → `{"watchlist": [{"ticker", "price", "previous_price", "ref_price"}]}`. `POST /api/watchlist` and `DELETE /api/watchlist/{ticker}` return the updated watchlist
- `POST /api/chat` `{"message"}` → `{"message", "executed_trades": [...], "watchlist_changes": [...], "errors": [...], "remaining_trades": [...]}`
- The Next.js export has no dynamic routes. FastAPI registers all `/api/*` routes first, then mounts the static files last, with `/` served as `index.html`

---

## 9. LLM Integration

When writing code to make calls to LLMs, use cerebras-inference skill to use LiteLLM via OpenRouter to the `openrouter/openai/gpt-oss-120b` model with Cerebras as the inference provider. Structured Outputs should be used to interpret the results.

There is an OPENROUTER_API_KEY in the .env file in the project root. Provider pinning to Cerebras and the structured-output (`response_format`) parameters follow the cerebras-inference skill; verify structured outputs work on this route before building on it.

### How It Works

When the user sends a chat message, the backend:

1. Loads the user's current portfolio context (cash, positions with P&L, watchlist with live prices, total portfolio value)
2. Loads recent conversation history from the `chat_messages` table (the most recent 20 messages)
3. Constructs a prompt with a system message, portfolio context, conversation history, and the user's new message
4. Calls the LLM via LiteLLM → OpenRouter, requesting structured output, using the cerebras-inference skill
5. Parses the complete structured JSON response
6. Auto-executes any trades or watchlist changes specified in the response, trades sequentially in the order given (see Trade Failures below)
7. Stores the message and executed actions in `chat_messages`
8. Returns the complete JSON response to the frontend (no token-by-token streaming — Cerebras inference is fast enough that a loading indicator is sufficient)

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
- `trades` (optional): Array of trades to auto-execute. Each trade goes through the same validation as manual trades (sufficient cash for buys, sufficient shares for sells)
- `watchlist_changes` (optional): Array of watchlist modifications
- At most 5 trades and 5 watchlist changes are processed per response; any extras are ignored and reported in the chat message

### Auto-Execution

Trades specified by the LLM execute automatically — no confirmation dialog. This is a deliberate design choice:
- It's a simulated environment with fake money, so the stakes are zero
- It creates an impressive, fluid demo experience
- It demonstrates agentic AI capabilities — the core theme of the course

### Trade Failures

Auto-execution applies to successful trades only. If a trade fails validation (e.g., insufficient cash or shares), execution **stops at the failed trade** and the user decides whether to continue:

- The chat response includes the executed actions, the error for the failed trade, and `remaining_trades` (trades not yet attempted)
- The chat panel shows the error inline with "Execute remaining trades" / "Skip" buttons. Skip dismisses them; Execute submits each remaining trade through `POST /api/portfolio/trade`, showing any further failure inline
- Watchlist changes are applied independently of trade failures. Removing a held ticker fails validation like any other invalid action
- The error is stored in `chat_messages.actions`, so the LLM sees it in the history on the next turn

### Malformed LLM Output

If the response is not valid JSON matching the schema, the backend retries the call once. If the retry also fails, it returns a plain assistant message ("Sorry, I couldn't process that — please try again"), executes nothing, and stores no actions.

### System Prompt Guidance

The LLM should be prompted as "FinAlly, an AI trading assistant" with instructions to:
- Analyze portfolio composition, risk concentration, and P&L
- Suggest trades with reasoning
- Execute trades when the user asks or agrees
- Manage the watchlist proactively
- Be concise and data-driven in responses
- Always respond with valid structured JSON

### LLM Mock Mode

When `LLM_MOCK=true`, the backend returns deterministic mock responses instead of calling OpenRouter. Rules (first match wins, case-insensitive, on the user message):

| Message contains | Mock response |
|---|---|
| `fail` | Two trades: buy 1000000 AAPL (fails on cash), then buy 1 MSFT (left in `remaining_trades`) |
| `malformed` | Invalid JSON (exercises the retry/fallback path) |
| `buy` | Message plus trade: buy 1 AAPL |
| `sell` | Message plus trade: sell 1 AAPL |
| `add` | Watchlist change: add PYPL |
| `remove` | Watchlist change: remove PYPL |
| anything else | Message only ("Mock response"), no actions |

This enables:
- Fast, free, reproducible E2E tests
- Development without an API key
- CI/CD pipelines

---

## 10. Frontend Design

### Layout

The frontend is a single-page application with a dense, terminal-inspired layout. The specific component architecture and layout system is up to the Frontend Engineer, but the UI should include these elements:

- **Watchlist panel** — grid/table of watched tickers with: ticker symbol, current price (flashing green/red on change), daily change %, and a sparkline mini-chart (accumulated from SSE since page load). Daily change % uses the `ref_price` from the SSE event (see §6)
- **Main chart area** — larger chart for the currently selected ticker, with at minimum price over time. Clicking a ticker in the watchlist selects it here.
- **Portfolio heatmap** — treemap visualization where each rectangle is a position, sized by portfolio weight, colored by P&L (green = profit, red = loss)
- **P&L chart** — line chart showing total portfolio value over time, using data from `portfolio_snapshots`
- **Positions table** — tabular view of all positions: ticker, quantity (decimals shown; fractional holdings badged), avg cost, current price, unrealized P&L, % change
- **Trade bar** — simple input area: ticker field, quantity field, buy button, sell button. Market orders, instant fill.
- **AI chat panel** — docked/collapsible sidebar. Message input, scrolling conversation history, loading indicator while waiting for LLM response. Trade executions and watchlist changes shown inline as confirmations.
- **Header** — portfolio total value (updating live), connection status indicator, cash balance

### Technical Notes

- Use `EventSource` for SSE connection to `/api/stream/prices`
- A single frontend price-history store holds a capped array (500 points) per ticker, filled from the SSE stream. Both the sparklines and the main chart read from it, so they are not built twice. The main chart starts empty on page load and fills progressively; there is no history endpoint for prices
- Charting: Lightweight Charts (canvas) for the main chart, sparklines and P&L chart; Recharts `Treemap` for the portfolio heatmap
- Empty states: with no positions, the heatmap and positions table show a "No positions yet — place a trade" message, and the P&L chart shows the starting cash value as a flat line
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
  - Copy backend/
  - uv sync (install Python dependencies from lockfile)
  - Copy frontend build output into a static/ directory
  - Expose port 8000
  - CMD: uvicorn serving FastAPI app
```

FastAPI serves the static frontend files and all API routes on port 8000.

### Docker Volume

The SQLite database persists via a named Docker volume:

```bash
docker run -v finally-data:/app/db -p 8000:8000 --env-file .env finally
```

The `db/` directory in the project root maps to `/app/db` in the container. The backend writes `finally.db` to this path.

### Start/Stop Scripts

Two cross-platform Python scripts (standard library only), run with `uv run scripts/start.py` / `uv run scripts/stop.py` so no system Python is needed. They work on Windows, macOS and Linux.

**`scripts/start.py`**:
- Builds the Docker image if not already built (or if `--build` flag passed)
- Runs the container with the volume mount, port mapping, and `.env` file
- Prints the URL to access the app
- Optionally opens the browser

**`scripts/stop.py`**:
- Stops and removes the running container
- Does NOT remove the volume (data persists)

Both scripts are idempotent — safe to run multiple times.

### Out of Scope for the Initial Build

There is no root `docker-compose.yml` (the `docker run` command and scripts cover it; `test/docker-compose.test.yml` is the only compose file). Cloud deployment (AWS App Runner, Render, Terraform) is not part of the initial build, though the single container is designed to deploy to any container platform.

---

## 12. Testing Strategy

### Unit Tests (within `frontend/` and `backend/`)

**Backend (pytest)**:
- Market data: simulator generates valid prices, GBM math is correct, Massive API response parsing works, both implementations conform to the abstract interface
- Portfolio: trade execution logic, P&L calculations, edge cases (selling more than owned, buying with insufficient cash, selling at a loss)
- Money: integer-cent arithmetic and rounding for buys/sells, P&L and snapshots
- LLM: structured output parsing handles all valid schemas, malformed-response retry and fallback, trade validation and stop-on-failure within chat flow, mock-mode rules
- API routes: correct status codes, response shapes, error handling

**Frontend (React Testing Library or similar)**:
- Component rendering with mock data
- Price flash animation triggers correctly on price changes
- Watchlist CRUD operations
- Portfolio display calculations
- Chat message rendering and loading state

### E2E Tests (in `test/`)

**Infrastructure**: A `docker-compose.test.yml` in `test/` with two separate services: the app and a dedicated Playwright service. This keeps browser dependencies out of the production image.
- The Playwright image version is pinned
- Playwright waits for the app's health check (`/api/health`) to pass before tests start (`depends_on` with `condition: service_healthy`)
- Test reports and screenshots are written to a mounted `test/artifacts/` directory so they persist after the run

**Environment**: Tests run with `LLM_MOCK=true` by default for speed and determinism.

**Key Scenarios**:
- Fresh start: default watchlist appears, $10k balance shown, prices are streaming
- Add and remove a ticker from the watchlist
- Buy shares: cash decreases, position appears, portfolio updates
- Sell shares: cash increases, position updates or disappears
- Portfolio visualization: heatmap renders with correct colors, P&L chart has data points
- AI chat (mocked): send a message, receive a response, trade execution appears inline
- AI chat trade failure (mocked, message containing "fail"): error shown inline; remaining trades wait for the user's Execute/Skip choice
- Remove a held ticker from the watchlist: rejected with an error until the position is sold
- Fractional share purchase: position shows its decimal quantity and badge
- SSE resilience: disconnect and verify reconnection

---

## 13. Review Log: Decisions and Where They Landed

This section records how each review question and simplification suggestion was resolved. The sections referenced are the source of truth; this log is for traceability.

### Questions

| # | Topic | Resolution | Where |
|---|---|---|---|
| 1 | Daily change % baseline | Prior trading day's close of the relevant market (Massive: API previous close; simulator: seed price treated as prior close). Delivered as `ref_price` | §6, §10 |
| 2 | Ticker validation | Validate against a known universe (simulator: ~50 curated tickers incl. PYPL; Massive: valid if API returns a price). Unknown → 400 | §6, §8 |
| 3 | Fractional shares | Supported; fractional holdings badged and shown with decimals | §2, §10 |
| 4 | Selling to zero | A position row is deleted only when quantity reaches zero; partial sells keep `avg_cost_cents` | §7 |
| 5 | Held ticker removal | Feed = watchlist; removal of a held ticker is blocked (409) | §2, §6, §8 |
| 6 | Chat trade failures | Stop at first failed trade, user chooses Execute remaining / Skip | §9 |
| 7 | Chat history window | Most recent 20 messages | §9 |
| 8 | Malformed LLM output | Retry once, then apologetic message, nothing executed | §9 |
| 9 | `LLM_MOCK` rules | Documented keyword table | §9 |
| 10 | Snapshot growth | History = last 24h, downsampled to ≤1,000 points; snapshots only while positions exist | §7, §8 |
| 11 | Empty states | "No positions yet" message; P&L chart flat at starting cash | §10 |
| 12 | Response shapes | Conventions and shapes added (**`/api/chat` field names are proposed, not yet confirmed**) | §8 |
| 13 | SSE event format | One event per ticker, JSON shape specified | §6 |
| 14 | Static serving | API routes first, static mount last, no dynamic routes | §8 |
| 15 | `.env` handling | Docker: `--env-file`; local dev: load root `.env` | §5 |
| 16 | Chart library | Lightweight Charts + Recharts Treemap | §10 |
| 17 | LLM route | Follow cerebras-inference skill; verify structured outputs on the route | §9 |
| 18 | Script platforms | Two cross-platform scripts cover all OSes | §11 |
| 19 | `.env.example` | `.env.example` created at the project root; `.env` is already gitignored | §5 |

### Simplification Suggestions

| # | Suggestion | Decision |
|---|---|---|
| 1 | Drop `user_id` columns | **Rejected** — intentional, kept for future multi-user |
| 2 | Snapshot cadence | Accepted (§7) |
| 3 | Money as integer cents | Accepted (§7) |
| 4 | Playwright setup | Dedicated Playwright service, pinned image, health-check wait, saved artifacts (§12) |
| 5 | Drop root compose / Terraform | Accepted; only `test/docker-compose.test.yml` remains (§4, §11) |
| 6 | Collapse scripts | Accepted: `scripts/start.py` and `scripts/stop.py`, run via `uv run` (§4, §11) |
| 7 | Simplify Massive path | **Rejected** — Massive stays a supported path |
| 8 | Shared frontend price-history store | Accepted (§10) |
| 9 | Startup-only DB init | Accepted (§4, §7) |
| 10 | Cap chat actions per response | Accepted: 5 trades, 5 watchlist changes (§9) |
| 11 | State out-of-scope trading features | Accepted (§2) |
| 12 | Single batched SSE event | **Rejected** — one event per ticker (§6) |

### Open Items

- Confirm the proposed `/api/chat` response field names (§8)
- Confirm the `uv run` Python scripts are acceptable in place of shell/PowerShell scripts (§11)
