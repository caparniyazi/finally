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
│   └── db/                   # Schema definitions, seed data, migration logic
├── planning/                 # Project-wide documentation for agents
│   ├── PLAN.md               # This document
│   └── ...                   # Additional agent reference docs
├── scripts/
│   ├── start_mac.sh          # Launch Docker container (macOS/Linux)
│   ├── stop_mac.sh           # Stop Docker container (macOS/Linux)
│   ├── start_windows.ps1     # Launch Docker container (Windows PowerShell)
│   └── stop_windows.ps1      # Stop Docker container (Windows PowerShell)
├── test/                     # Playwright E2E tests + docker-compose.test.yml
├── db/                       # Volume mount target (SQLite file lives here at runtime)
│   └── .gitkeep              # Directory exists in repo; finally.db is gitignored
├── Dockerfile                # Multi-stage build (Node → Python)
├── docker-compose.yml        # Optional convenience wrapper
├── .env                      # Environment variables (gitignored, .env.example committed)
└── .gitignore
```

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about Python. It talks to the backend via `/api/*` endpoints and `/api/stream/*` SSE endpoints. Internal structure is up to the Frontend Engineer agent.
- **`backend/`** is a self-contained uv project with its own `pyproject.toml`. It owns all server logic including database initialization, schema, seed data, API routes, SSE streaming, market data, and LLM integration. Internal structure is up to the Backend/Market Data agents.
- **`backend/db/`** contains schema SQL definitions and seed logic. The backend lazily initializes the database on first request — creating tables and seeding default data if the SQLite file doesn't exist or is empty.
- **`db/`** at the top level is the runtime volume mount point. The SQLite file (`db/finally.db`) is created here by the backend and persists across container restarts via Docker volume.
- **`planning/`** contains project-wide documentation, including this plan. All agents reference files here as the shared contract.
- **`test/`** contains Playwright E2E tests and supporting infrastructure (e.g., `docker-compose.test.yml`). Unit tests live within `frontend/` and `backend/` respectively, following each framework's conventions.
- **`scripts/`** contains start/stop scripts that wrap Docker commands.

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
- If `LLM_MOCK=true` → backend returns deterministic mock LLM responses (for E2E tests)
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

### Shared Price Cache

- A single background task (simulator or Massive poller) writes to an in-memory price cache
- The cache holds the latest price, previous price, and timestamp for each ticker
- SSE streams read from this cache and push updates to connected clients
- This architecture supports future multi-user scenarios without changes to the data layer

### SSE Streaming

- Endpoint: `GET /api/stream/prices`
- Long-lived SSE connection; client uses native `EventSource` API
- Server pushes price updates for all tickers known to the system at a regular cadence (~500ms) — in the single-user model this is equivalent to the user's watchlist
- Each SSE event contains ticker, price, previous price, timestamp, and change direction
- Client handles reconnection automatically (EventSource has built-in retry)

---

## 7. Database

### SQLite with Lazy Initialization

The backend checks for the SQLite database on startup (or first request). If the file doesn't exist or tables are missing, it creates the schema and seeds default data. This means:

- No separate migration step
- No manual database setup
- Fresh Docker volumes start with a clean, seeded database automatically

### Schema

All tables include a `user_id` column defaulting to `"default"`. This is hardcoded for now (single-user) but enables future multi-user support without schema migration.

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

**positions** — Current holdings (one row per ticker per user)
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
- `actions` TEXT (JSON — trades executed, watchlist changes made; null for user messages)
- `created_at` TEXT (ISO timestamp)

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
| GET | `/api/portfolio` | Current positions, cash balance, total value, unrealized P&L |
| POST | `/api/portfolio/trade` | Execute a trade: `{ticker, quantity, side}` |
| GET | `/api/portfolio/history` | Portfolio value snapshots over time (for P&L chart) |

### Watchlist
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/watchlist` | Current watchlist tickers with latest prices |
| POST | `/api/watchlist` | Add a ticker: `{ticker}` |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker |

### Chat
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/chat` | Send a message, receive complete JSON response (message + executed actions) |

### System
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | Health check (for Docker/deployment) |

---

## 9. LLM Integration

When writing code to make calls to LLMs, use cerebras-inference skill to use LiteLLM via OpenRouter to the `openrouter/openai/gpt-oss-120b` model with Cerebras as the inference provider. Structured Outputs should be used to interpret the results.

There is an OPENROUTER_API_KEY in the .env file in the project root.

### How It Works

When the user sends a chat message, the backend:

1. Loads the user's current portfolio context (cash, positions with P&L, watchlist with live prices, total portfolio value)
2. Loads recent conversation history from the `chat_messages` table
3. Constructs a prompt with a system message, portfolio context, conversation history, and the user's new message
4. Calls the LLM via LiteLLM → OpenRouter, requesting structured output, using the cerebras-inference skill
5. Parses the complete structured JSON response
6. Auto-executes any trades or watchlist changes specified in the response
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

### Auto-Execution

Trades specified by the LLM execute automatically — no confirmation dialog. This is a deliberate design choice:
- It's a simulated environment with fake money, so the stakes are zero
- It creates an impressive, fluid demo experience
- It demonstrates agentic AI capabilities — the core theme of the course

If a trade fails validation (e.g., insufficient cash), the error is included in the chat response so the LLM can inform the user.

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

- **Watchlist panel** — grid/table of watched tickers with: ticker symbol, current price (flashing green/red on change), daily change %, and a sparkline mini-chart (accumulated from SSE since page load)
- **Main chart area** — larger chart for the currently selected ticker, with at minimum price over time. Clicking a ticker in the watchlist selects it here.
- **Portfolio heatmap** — treemap visualization where each rectangle is a position, sized by portfolio weight, colored by P&L (green = profit, red = loss)
- **P&L chart** — line chart showing total portfolio value over time, using data from `portfolio_snapshots`
- **Positions table** — tabular view of all positions: ticker, quantity, avg cost, current price, unrealized P&L, % change
- **Trade bar** — simple input area: ticker field, quantity field, buy button, sell button. Market orders, instant fill.
- **AI chat panel** — docked/collapsible sidebar. Message input, scrolling conversation history, loading indicator while waiting for LLM response. Trade executions and watchlist changes shown inline as confirmations.
- **Header** — portfolio total value (updating live), connection status indicator, cash balance

### Technical Notes

- Use `EventSource` for SSE connection to `/api/stream/prices`
- Canvas-based charting library preferred (Lightweight Charts or Recharts) for performance
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

**`scripts/start_mac.sh`** (macOS/Linux):
- Builds the Docker image if not already built (or if `--build` flag passed)
- Runs the container with the volume mount, port mapping, and `.env` file
- Prints the URL to access the app
- Optionally opens the browser

**`scripts/stop_mac.sh`** (macOS/Linux):
- Stops and removes the running container
- Does NOT remove the volume (data persists)

**`scripts/start_windows.ps1`** / **`scripts/stop_windows.ps1`**: PowerShell equivalents for Windows.

All scripts should be idempotent — safe to run multiple times.

### Optional Cloud Deployment

The container is designed to deploy to AWS App Runner, Render, or any container platform. A Terraform configuration for App Runner may be provided in a `deploy/` directory as a stretch goal, but is not part of the core build.

---

## 12. Testing Strategy

### Unit Tests (within `frontend/` and `backend/`)

**Backend (pytest)**:
- Market data: simulator generates valid prices, GBM math is correct, Massive API response parsing works, both implementations conform to the abstract interface
- Portfolio: trade execution logic, P&L calculations, edge cases (selling more than owned, buying with insufficient cash, selling at a loss)
- LLM: structured output parsing handles all valid schemas, graceful handling of malformed responses, trade validation within chat flow
- API routes: correct status codes, response shapes, error handling

**Frontend (React Testing Library or similar)**:
- Component rendering with mock data
- Price flash animation triggers correctly on price changes
- Watchlist CRUD operations
- Portfolio display calculations
- Chat message rendering and loading state

### E2E Tests (in `test/`)

**Infrastructure**: A separate `docker-compose.test.yml` in `test/` that spins up the app container plus a Playwright container. This keeps browser dependencies out of the production image.

**Environment**: Tests run with `LLM_MOCK=true` by default for speed and determinism.

**Key Scenarios**:
- Fresh start: default watchlist appears, $10k balance shown, prices are streaming
- Add and remove a ticker from the watchlist
- Buy shares: cash decreases, position appears, portfolio updates
- Sell shares: cash increases, position updates or disappears
- Portfolio visualization: heatmap renders with correct colors, P&L chart has data points
- AI chat (mocked): send a message, receive a response, trade execution appears inline
- SSE resilience: disconnect and verify reconnection

---

## 13. Review: Questions, Clarifications & Simplification Opportunities

*Added by doc review (2026-11-10). Items are grouped by priority. Each one either needs a decision from the project owner or is a suggested wording/scope change. Resolve and fold answers back into the relevant section, then delete the item.*

### A. Gaps that block implementation (need a decision)

1. **"Daily change %" has no data source.** The watchlist (§10) shows "daily change %", but nothing stores a daily open or previous close. `PriceUpdate.change` in the finished market data module is tick-to-tick, not daily. Options: (a) treat the first price seen since backend start as the reference ("session change %"), (b) add a `session_open` field to the price cache, (c) for Massive, use the previous close from the snapshot API. **Suggestion:** use (a) or (b) for both sources and rename the column "Change %" so the label matches what is measured.

2. **API response shapes are not defined.** §8 lists endpoints but gives no request/response bodies, status codes, or error format. The frontend and backend agents work in parallel, so this is the riskiest gap for integration. **Suggestion:** add a short JSON example for each endpoint, plus one standard error body (e.g. `{"error": "insufficient_cash", "detail": "..."}` with HTTP 400). At minimum, define:
   - `GET /api/portfolio`: the fields for each position (`ticker, quantity, avg_cost, current_price, market_value, unrealized_pnl, unrealized_pnl_pct`) and the totals.
   - `POST /api/portfolio/trade`: what comes back (the executed trade, the updated portfolio, or both).
   - `POST /api/chat`: the exact response, including how each action's success or failure is reported (e.g. `actions: [{type, ticker, side, quantity, status: "executed"|"failed", error?}]`).
   - SSE event format: one event per ticker or one batched event per tick? Event name? Field names? These should match `PriceUpdate` in `backend/app/market/models.py`.

3. **Ticker validation is undefined.** When a user (or the LLM) adds an unknown ticker such as `XYZQ`, the simulator currently invents a random price between $50 and $300 for it. Is that acceptable? With Massive, an invalid ticker returns no data. **Questions:** Should adding a ticker be validated (format regex, known-ticker list, or a Massive lookup)? What does the watchlist show for a ticker that has no price yet?

4. **Trading tickers that are not on the watchlist.** The trade bar accepts any ticker. If the ticker is not on the watchlist, there is no price in the cache. **Options:** (a) reject the trade with an error, (b) auto-add the ticker to the watchlist (and the data source) and then fill. **Suggestion:** (b) for the simulator, because it is simpler for users and for the LLM. State this explicitly either way.

5. **Removing a watched ticker that is still held.** If the user removes a ticker they own from the watchlist, its price stops updating and the position cannot be valued. **Suggestion:** the set of tickers the data source tracks should be watchlist ∪ open positions. That may be what "all tickers known to the system" in §6 means. If so, say it directly.

6. **Selected-ticker chart history.** The main chart (§10) needs history, but sparklines are built only from SSE data since page load, and there is no history endpoint. Is the main chart also SSE-accumulated since page load (empty on refresh)? **Suggestion:** yes, keep it client-side and state it explicitly. A `/api/prices/{ticker}/history` endpoint can be a stretch goal.

### B. Ambiguities and inconsistencies

7. **Lazy init "on startup (or first request)".** §4 says "first request", §7 says "startup (or first request)". **Suggestion:** pick startup (FastAPI lifespan). The market data source must be seeded with the watchlist at startup anyway, so the DB has to be ready before the first request.

8. **`users_profile` lacks `user_id`.** §7 says "All tables include a `user_id` column", but `users_profile` uses `id` as the user key. This is fine, but the sentence should say "all tables except `users_profile`, whose `id` *is* the user id".

9. **`.env` location in the container.** §5 says the backend "reads `.env` from the project root (mounted into the container or read via docker `--env-file`)". The run command uses `--env-file`, so no file mount is needed. **Suggestion:** say env vars come from the process environment, with `.env` loaded via `python-dotenv` only for local non-Docker development.

10. **`db/` volume wording.** §11 says "The `db/` directory in the project root maps to `/app/db`", but the command uses a *named* volume (`finally-data`), not a bind mount of `./db`. These are different behaviors. Pick one. A named volume is simpler and works the same on Windows, so `db/.gitkeep` would only matter for local non-Docker runs.

11. **Fractional shares.** The schema supports fractional quantities, but nothing says whether the trade bar or the LLM may submit them. Confirm that fractional quantities are allowed, and set a minimum such as `> 0`.

12. **Selling a whole position.** When quantity reaches 0, is the `positions` row deleted, or kept with quantity 0? The E2E test says the position "disappears", which implies the row is deleted. State this, and also whether floating-point dust (e.g. `1e-12`) is treated as zero.

13. **P&L chart vs. "unrealized P&L".** The P&L chart plots *total portfolio value*, not P&L. Consider renaming it "Portfolio Value chart" or plotting value minus $10,000. Also: is realized P&L tracked or shown anywhere? Currently it can only be derived from `trades`.

14. **Snapshot retention.** A snapshot every 30 seconds is about 2,880 rows per day, forever. That is fine for SQLite, but `/api/portfolio/history` should probably have a limit or downsampling rule. Also, should a snapshot be recorded at seed time, so the chart has a starting point of $10,000?

15. **Chat history depth.** "Recent conversation history" needs a number (e.g. last 20 messages). Also, does the frontend load prior chat history on page load? If so, it needs a `GET /api/chat` (or history) endpoint, which is not listed in §8.

16. **Charting library.** "Canvas-based ... (Lightweight Charts or Recharts)": Recharts is SVG-based, not canvas. Also, neither library provides a treemap with the needed control except Recharts' `Treemap`. **Suggestion:** Lightweight Charts for the price and value charts, and a Recharts `Treemap` (or a small custom div/CSS treemap) for the heatmap. Say this explicitly.

17. **Heatmap color scale.** Colored by P&L in absolute dollars or %? Is there a clamp range (e.g. ±5% maps to full saturation)? Also, does cash appear as a tile?

18. **Massive poll interval.** "Configurable interval" is mentioned, but no env var is defined. Add `MASSIVE_POLL_INTERVAL` (default 15s) to §5, or state that it is hard-coded.

19. **`OPENROUTER_API_KEY` marked "Required".** §9 says mock mode allows "development without an API key", so the key is only required when `LLM_MOCK=false`. Also define what `/api/chat` returns when the key is missing and mock mode is off (a friendly error message rather than a 500).

### C. LLM integration details worth pinning down

20. **Order of execution and failures.** Are trades executed in array order, with partial success allowed (e.g. the first buy succeeds, the second fails for insufficient cash)? Are watchlist changes applied before trades? (Applying them first helps when the LLM adds and buys a new ticker in the same response.)

21. **"Error included so the LLM can inform the user".** The flow is single-pass: the LLM writes `message` *before* the trades execute, so it cannot react to a failure in that same reply. **Options:** (a) the backend appends or annotates failures in the returned actions and the frontend shows them inline (simple), or (b) a second LLM call to rephrase. **Suggestion:** (a). Update the wording, and store failure results in `chat_messages.actions` so the LLM sees them on the next turn.

22. **Guardrails.** Should there be a max quantity or max notional per LLM trade, or a cap on trades per message? Probably not needed with fake money, but say so explicitly.

23. **Mock responses.** Define the deterministic mock behavior so E2E tests can rely on it. Example: a message containing "buy" returns a trade of 1 share of AAPL; any other message returns a fixed text reply with no actions.

### D. Simplification opportunities

24. **Drop the `id` UUID primary key where a natural key exists.** `positions` and `watchlist` already have `UNIQUE (user_id, ticker)`. Using that as the primary key removes UUID generation and simplifies upserts (`INSERT ... ON CONFLICT`). This is optional, since the current design is not wrong.

25. **Merge the four start/stop scripts conceptually.** The scripts are thin wrappers around `docker build` / `docker run` / `docker rm -f`. Consider listing the exact commands once in §11 and saying the scripts wrap them. That removes the need to describe each script and keeps behavior identical across platforms.

26. **`docker-compose.yml` at the root.** The table in §3 says "no docker-compose for production", but §4 lists an "optional convenience" `docker-compose.yml`. If the start scripts are the supported path, consider dropping it to remove one more artifact to maintain.

    27. **Trim duplicated rationale.** "SSE over WebSockets" is justified three times (§3 bullet, §3 table, §6), and the market data section duplicates `MARKET_DATA_SUMMARY.md`. Since market data is done, §6 could shrink to a short contract summary (cache API + SSE event shape) with a link to the summary doc.

28. **Remove the time-sensitive "§9 There is an OPENROUTER_API_KEY in the .env file" line.** It is environment-specific and already covered by §5.

29. **Make the testing list a minimum, not a checklist.** The §12 unit test lists are good guidance. Consider saying "at minimum" so agents do not feel bound to write a test for every listed bullet before shipping.

### E. Minor wording fixes

- §3 diagram: the right-hand border is misaligned on the "Static file serving", "(Next.js export)" and "Background task" lines.
- §2 says "a grid" for the watchlist, while §10 says "grid/table". Pick one term.
- §10 Header: "portfolio total value (updating live)" implies the frontend recomputes the value from SSE prices × positions. State that this computation happens client-side, so the backend does not need to push portfolio updates.
