# Review of `planning/PLAN.md`

## Overall assessment

The plan is unusually complete for a capstone: it defines the product, architecture, public routes, persistence model, failure behavior, build order, and test strategy in one place. The division between the price cache, portfolio state, and frontend-derived valuation is sensible, and mock LLM mode is a strong choice for deterministic end-to-end tests.

I would approve the direction, but not start parallel frontend/backend implementation until the blocking contracts below are resolved. Several claims in the plan do not match the market subsystem that already exists, and the trade path needs a stronger consistency contract.

## Blocking issues

### 1. The promised market-data contract requires more than the two listed changes

Section 6 says only `open_price` and history need to be added under `app/market/`, but the existing implementation differs from the plan in three additional ways:

- `PriceUpdate.to_dict()` currently emits `timestamp` as Unix seconds, while the plan requires ISO-8601 UTC strings everywhere and illustrates that shape in the SSE payload.
- Producers call `PriceCache.update()` once per ticker. The cache version therefore changes several times during one simulator/poller cycle, so the SSE task can observe a partially updated set. That does not guarantee the stated "one atomic snapshot per tick" behavior.
- `MassiveDataSource.add_ticker()` only appends a symbol. It neither validates the symbol nor fetches it immediately. A new ticker can remain without a price until the next 15-second poll, contradicting both the promised `404` behavior and "prices arrive via SSE within 500 ms."

Recommended resolution: either expand the allowed market work or weaken the API promises. The cleaner implementation is to add a cache-level `update_many()` that installs a complete cycle and increments the version once; serialize all public timestamps at the API boundary; and give the Massive source an explicit `validate/add-and-prime` operation. Define what happens on provider timeout, rate limiting, and a valid but temporarily unpriced symbol. Tests should cover all three differences.

Also define reset semantics: removing a ticker should clear its open price and history, and re-adding it should begin a new session. If that is not intended, say that history survives removal.

### 2. Trade execution needs an atomicity and numeric-precision contract

A trade is currently specified as a series of reads and writes: read cached price, validate cash/shares, update cash and position, append a trade, and append a snapshot. Concurrent manual/chat requests can both pass validation and then overspend or oversell unless this is one serialized database transaction. A crash midway can also leave cash, positions, trades, and snapshots inconsistent.

The rounding rules are not sufficient by themselves. SQLite `REAL` plus fractional quantities makes exact equality at position close fragile, and an input smaller than four decimal places after quantization could be accepted as `> 0` but become zero.

Recommended resolution:

- Put the entire operation in one transaction (`BEGIN IMMEDIATE` is appropriate for this single-user SQLite design), and use the same service function for manual and LLM trades.
- State the rounding mode, whether quantities with more than four decimals are rejected or quantized, and reject values that become zero.
- Prefer integer cents plus fixed-point share units, or explicitly use `Decimal` at the service boundary and convert only at storage/API boundaries.
- Define the snapshot valuation point and ensure the trade price used for cash, cost basis, trade log, and immediate snapshot is the same captured quote.

Add a concurrency test proving that two simultaneous buys cannot create a negative balance and two simultaneous sells cannot create a negative position.

### 3. Massive/provider failure behavior is not represented in health or startup semantics

The existing Massive poller logs and swallows authentication, network, and rate-limit failures. A bad `MASSIVE_API_KEY` can therefore yield an application reporting `{"status":"ok"}` with no useful prices. The plan also does not say whether startup should fail if initial pricing fails, whether stale prices may still be traded, or how old a quote may be.

Recommended resolution: add provider readiness and last-success metadata, expose it in `/api/health`, and define a maximum tradeable quote age. At minimum, health should distinguish healthy, degraded/stale, and unavailable states. Decide whether an invalid configured provider should fail startup or fall back to the simulator; do not silently present an empty terminal as healthy.

### 4. The Docker health check depends on a binary absent from the stated image

`python:3.12-slim` does not guarantee `curl`, yet the Docker stage specifies a curl-based `HEALTHCHECK`. Install curl explicitly (and clean apt metadata), or use a small Python/stdlib health-check command. This should be decided in the plan so the final image and test readiness do not fail unexpectedly.

## Important contract clarifications

### 5. The current chat message is likely included twice

The flow persists the user message, loads the last 20 messages, and then constructs a prompt from that history plus the user's new message. Unless the history query excludes the newly inserted row, the current message appears twice. Specify one approach: persist then load a history that already includes it, or load prior history before persisting and append it once. Clarify whether "20 messages" includes the current message and whether trimming preserves user/assistant turn pairs.

### 6. Static export and development rewrites need an explicit conditional configuration

The statement that rewrites are simply ignored by static export is unsafe as an implementation contract: unsupported Next.js routing features can make an export build fail. Require `rewrites()` to be returned only during the development-server phase (or use a separate development proxy setup), and add `npm run build` to CI. The plan should also pin a supported Node/Next/Tailwind combination rather than leaving agents to independently choose potentially incompatible current majors.

### 7. The API response contracts are incomplete for independent frontend work

Several shapes needed by the frontend remain implicit:

- the fields in the returned `trade` object;
- the successful POST/DELETE watchlist response and status codes;
- the exact successful action objects in chat (`trades` and `watchlist_changes` appear to mean actions actually executed, but their response shapes are unspecified);
- history timestamp/price types and empty-history behavior;
- normalization rules for ticker and quantity in both requests and responses.

Define request/response models (including status codes) for every endpoint and make the generated OpenAPI schema the executable contract. If DELETE returns `204`, qualify the statement that all responses are JSON. This will prevent the frontend and E2E agents from inventing different shapes.

### 8. Configuration requirements conflict with mock/local behavior

The plan calls `OPENROUTER_API_KEY` required, says mock mode enables development without an API key, and defaults `LLM_MOCK=false`. It also states that a root `.env` already contains a key, while `.env` is intentionally untracked and is not present in a fresh clone. A start script using `--env-file .env` will fail before the app starts when that file is absent.

Recommended resolution: make the key conditionally required only when `LLM_MOCK` is false, commit `.env.example`, and have start scripts either create/copy instructions for `.env` or fail with a precise message. Define whether the rest of the application starts when real chat is unconfigured and what `/api/health` reports in that state.

### 9. Portfolio chart initialization and client-side buffer limits are underspecified

A fresh database has no portfolio snapshots until 30 seconds elapse, but the UI and E2E plan expect chart data. Record an initial startup snapshot (after prices are available), or explicitly define an empty state. The browser then appends portfolio values every SSE tick indefinitely, while backend history is capped only by response count. Main-chart and P&L client buffers need explicit caps/downsampling so a long-running tab does not grow without bound. Define how duplicate timestamps are merged when historical data is followed by live data.

### 10. SSE needs heartbeat and ordering/reconnection rules

When the cache version does not change, the endpoint sends nothing. Provider outages, closed markets, or an empty tracked set can leave intermediaries timing out the connection. Emit periodic SSE comments as heartbeats. It would also help to include a monotonically increasing event ID/version so reconnecting clients can reject stale/out-of-order events and tests can assert recovery. Clarify whether reconnect sends a full current snapshot (recommended) before incremental updates.

### 11. Deterministic E2E database isolation needs a concrete mechanism

A Compose volume is persistent by default, so "fresh (non-persistent) database volume" is ambiguous. Specify a `tmpfs`, an anonymous volume recreated for each run, or teardown with `docker compose down -v` under a unique project name. Without that, cash, positions, and chat history can leak between test runs. Also specify that Playwright reaches the app by its Compose service name rather than relying on a host port.

## Additional recommendations

- Define database indexes for time-ordered history queries (`portfolio_snapshots(user_id, recorded_at)` and `chat_messages(user_id, created_at)`) and enable/check SQLite foreign-key and busy-timeout settings per connection.
- Bound persisted history or define retention. The API response is bounded, but the snapshot and chat tables otherwise grow forever.
- Specify UI prevention of accidental duplicate submissions while requests are in flight. For stronger retry safety, accept a client-generated idempotency key for trades.
- Clarify whether the `30-second timeout` covers only the LLM call or the entire chat endpoint, and ensure cancellation cannot occur after only part of an action list executes without a recorded assistant/action result.
- Add accessibility acceptance criteria for status/color cues: price direction and P&L must not be communicated by color alone, and controls need keyboard/focus labels.
- Add backend tests for restart persistence, rollback on mid-trade failure, stale quote rejection, background-task cancellation, and simultaneous SSE clients.
- Add a CI acceptance gate covering backend tests/lint, frontend test/build, Docker image health, and E2E. The test list is good, but the plan does not state the command that constitutes completion.

## Strengths worth preserving

- The single-origin, single-container deployment is appropriate for the course and avoids unnecessary operational complexity.
- Tracking the union of watchlist symbols and held positions is the correct invariant for live valuation.
- Keeping live valuation in one frontend selector prevents inconsistent totals across the header, table, and heatmap.
- The mock LLM still routes actions through real domain validation, giving the E2E suite meaningful coverage.
- Persisting raw positions/trades while treating market prices as ephemeral is a clean separation of concerns.
- The build order is dependency-aware and should work well once the contracts above are tightened.

## Suggested approval criteria

The plan is ready for implementation when sections 6-9 and 11-12 explicitly settle: atomic market batches and timestamps, Massive validation/readiness, transactional fixed-precision trades, exact API schemas, conditional LLM configuration, initial/bounded chart history, a working image health check, and disposable E2E storage. Those changes are small relative to the plan, but resolving them before parallel implementation will eliminate most likely integration rework.
