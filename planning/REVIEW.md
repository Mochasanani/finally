# PLAN.md Review

## Summary of Changes

The diff is a net simplification of the spec (+90/-76 lines). The dominant themes are:

1. **Persistence simplified** — removed `chat_messages` table, removed `user_id` columns, moved to startup (not lazy) DB init, moved to named Docker volume only.
2. **Chat is now stateless on the server** — history lives in the frontend and is passed back on each `POST /api/chat`.
3. **Chat response formalized** — introduces `executed_actions` as the authoritative per-action result array.
4. **New `session_open` concept** added to the price cache and SSE event for computing daily change %.
5. **Tightened operational detail** — explicit `OPENROUTER_API_KEY` startup requirement, `asyncio.Lock` for chat serialization, input validation (§8), error shape, `lightweight-charts` chosen, URL `?symbol=` for chart selection, Windows/frontend-unit-tests deferred, `tmpfs` for E2E.

Most changes are improvements. There are, however, several concrete conflicts and gaps below.

---

## MUST-FIX (spec is in conflict with itself or with the existing build)

### 1. `session_open` is specified but conflicts with the already-built `PriceCache` / `PriceUpdate`

- PLAN §6 "Shared Price Cache": *"The cache holds the latest price, previous price, a `session_open` price ... and a timestamp for each ticker"*.
- PLAN §6 SSE: *"Each SSE event contains ticker, price, previous price, session_open price, timestamp, and change direction"*.
- PLAN §8: `GET /api/watchlist` enriches each ticker with `session_open`.
- PLAN §10: *"daily change % (current price vs. the backend's `session_open`)"*.

The existing implementation at `backend/app/market/cache.py` (the `PriceCache.update` method) only stores `previous_price` and does not track any first-observed price. `PriceUpdate` in `backend/app/market/models.py` has fields `ticker, price, previous_price, timestamp` — no `session_open`. Its `change` / `change_percent` / `direction` are all computed tick-over-tick, not against a session baseline. `to_dict()` emits exactly those tick-over-tick fields.

Consequence: implementing the spec will require modifying `PriceCache.update` (capture first price for each ticker under lock), extending `PriceUpdate` (or adding a parallel struct), and updating `to_dict()` — which in turn changes the SSE payload consumed by downstream code and tests. The MARKET_DATA_SUMMARY claims "Status: Complete." The spec silently invalidates that. This should be called out explicitly: either amend the spec to match the built cache, or flag `PriceCache`/`PriceUpdate` as "requires extension for session_open" so the next implementing agent knows the market-data module is not frozen.

Additionally: "the first price observed since the backend process started" creates a testability/operational issue — restart the container mid-session and `session_open` resets, so "daily change %" on the UI will jump. Two realistic options the spec should pick between:

- (a) **Process-scoped** (what the spec says) — acceptable for a demo; state the reset-on-restart behavior explicitly so frontend and testers don't treat the jump as a bug.
- (b) **Persisted session anchor** — store a per-day open in SQLite (or a simple JSON) keyed by date. More honest for "daily change", more work.

As written, (a) is implied but unacknowledged. Name it.

### 2. `EventSource.readyState` connection-status mapping is subtly wrong for the "disconnected" case

PLAN §10 Technical Notes: *"`OPEN` → green, `CONNECTING` → yellow (covers both initial connect and automatic reconnection after a drop), `CLOSED` → red. No custom reconnection logic is needed — `EventSource` reconnects automatically."*

`EventSource.CLOSED` is only reached after `.close()` is called or after a fatal (HTTP 204 / 4xx) response — transient network drops produce `CONNECTING`, not `CLOSED`. So under the mapping as written, the "red = disconnected" indicator described in §2 UX (*"red = disconnected"*) will essentially never light up in normal operation; the user sees yellow forever during an outage. Either:

- accept that and drop "red = disconnected" from §2, **or**
- add a heuristic (e.g., "no event received for N seconds while `readyState === CONNECTING`" → red).

Pick one; current §2 and §10 contradict each other.

### 3. Default seed data under-specifies the `users_profile` row

§7 says `users_profile` has `id`, `cash_balance`, `created_at`. "Default Seed Data" lists *"One user profile: `id="default"`, `cash_balance=10000.0`"* — but `created_at` is not specified. The schema says ISO timestamp with no default. Clarify: seed sets `created_at` to `now()` at bootstrap. Minor but an implementing agent will ask.

### 4. Trade endpoint request schema is underspecified for fractional shares vs. integer

§8: trade `{ticker, quantity, side}`. §8 Input validation: *"Trade `quantity` must be a positive float with a minimum of `0.0001`"*. §7: `quantity REAL (fractional shares supported)`. Consistent so far. But §2 UX says *"Buy and sell shares — market orders only, instant fill at current price, no fees, no confirmation dialog"* and the Trade bar described in §10 is *"ticker field, quantity field, buy button, sell button"* with no hint that fractional input is allowed. Low risk of misinterpretation, but worth either (a) making the trade bar examples show fractional input, or (b) restricting UI entry to integers and only letting the LLM place fractional trades. Flag so the Frontend Engineer doesn't default to `type="number" step="1"`.

---

## SHOULD-FIX (gaps or ambiguities an agent will guess wrong on)

### 5. `POST /api/chat` history shape is under-specified

§8 body: `{message: string, history: [{role, content}]}`. §9 flow: *"Takes the `history` array supplied in the request (frontend-managed, capped at the last 20 turns)"*.

Questions the spec doesn't answer:
- Is "20 turns" 20 messages or 20 user+assistant pairs (40 messages)?
- Does the `history` include the prior `executed_actions`, or is it stripped down to `{role, content}` only? §9 "Authoritative record" says *"The frontend appends the assistant message + executed actions to its local history for the next turn."* — but the API signature is `{role, content}`, which can't carry `executed_actions` unless they're serialized into `content`. Decide: either (a) widen the schema to `{role, content, executed_actions?}`, or (b) stipulate that the frontend flattens executed_actions into the `content` string.
- What does the backend do if the frontend sends more than 20 turns? Truncate? 400? Silently accept?

### 6. `/api/portfolio/history` has no pagination / time-window parameters

§7 retention trims `portfolio_snapshots` to 10,000 rows. At one snapshot every 30s that's ~3.5 days of continuous uptime. §8 specifies `GET /api/portfolio/history` with no query params. Is the frontend expected to fetch all 10k rows every page load? For a P&L line chart that's fine but should be stated explicitly: either "returns all snapshots" or "accepts `?since=<iso>`/`?limit=<n>`". Otherwise the Backend agent will invent one and the Frontend agent will assume a different one.

### 7. Quantity bound "1,000,000 shares" is lower than a valid BRK-A buy with $10k cash — but that's fine; the inverse edge is missing

§8 validation says min `0.0001`, max `1_000_000`. With `STARTING_CASH = 10_000` and seeded AAPL ~$190, max practical buy is ~52 shares. The max bound is trivia; no problem. The missing piece is: what happens when a **sell** quantity is greater than the position? §12 already calls out "selling more than owned" as an edge case, but the spec never states the HTTP response. Presumably `400` with `{"detail": "insufficient shares"}` — say so explicitly under §8 so all implementers agree.

### 8. SSE "cadence" language is now contradictory between §6 bullets

- Bullet 3: *"Server pushes price updates ... at a regular cadence (~500ms) — in the single-user model this is equivalent to the user's watchlist"*.
- Bullet 4 (new): *"Uses **version-based change detection**: events fire only when the cached price actually changes. With Massive on the free tier that means roughly one burst every 15s; with the simulator it is roughly every 500ms."*

The existing `stream.py` implementation (`backend/app/market/stream.py`) is already version-based — it sleeps `interval=0.5s` and only yields if `version` changed. So bullet 4 is correct. Bullet 3's *"at a regular cadence (~500ms)"* is misleading if kept — Massive free-tier users will see one burst per 15s, not 500ms. Reword bullet 3 to *"polls the cache at ~500ms"* or drop it.

### 9. Watchlist add/remove — what about the price cache lifecycle?

§8 `POST /api/watchlist` adds a ticker and `DELETE /api/watchlist/{ticker}` removes it. The existing `MarketDataSource` interface has `add_ticker()` / `remove_ticker()`. The spec doesn't say that the watchlist router must call these. This is "obviously" what should happen, but the Backend Engineer should not have to derive it. One explicit sentence: *"Watchlist mutations call `source.add_ticker` / `source.remove_ticker` so the price cache tracks only watched tickers."*

Related: what if the user holds a **position** in a ticker that's not on the watchlist? §8 says `/api/portfolio` returns positions enriched with current price. If the cache doesn't track that ticker, there's no price. Options: (a) positions implicitly subscribe; (b) removing from the watchlist is blocked while a position exists; (c) the enrichment tolerates `null` price. Pick one. Ambiguity here *will* produce divergent implementations.

### 10. "Session_open" vs. Massive first-poll timing

If Massive is used, the *first* price observed is whenever the poller returns — potentially mid-session for the real market. Labelling that as `session_open` and showing it as "daily change %" will be flat-out wrong (a price observed at 2pm is not the open). The spec owes the implementing agent one of:

- Rename the concept to `reference_price` / `baseline_price` and make the UI say "change since start" rather than "daily change %", **or**
- For Massive, source the actual previous close / open from the API and populate `session_open` accordingly, **or**
- Keep "daily change %" and simply accept the inaccuracy in the Massive path (state it).

Right now §10 says *"daily change %"* and §6 says *"first price observed since the backend process started"*. These are not the same thing.

### 11. LLM mock-mode behavior is under-specified

§9: *"When `LLM_MOCK=true`, the backend returns deterministic mock responses"*. §12 E2E: *"AI chat (mocked): send a message, receive a response, `executed_actions` rendered inline as confirmation rows"*.

What shape? Does the mock execute trades, or just return a message? If tests assert executed_actions rendering, the mock must produce at least one `trades` or `watchlist_changes` entry on some input. Worth a one-paragraph appendix: "mock responds to certain keywords (e.g., 'buy 5 AAPL') by returning a canned trade; otherwise returns a no-action message." Otherwise the Backend agent writes one mock and the E2E agent writes test assertions that don't match.

---

## NICE-TO-HAVE (polish)

### 12. Minor: `fractional shares supported` + trade price formatting

§7 `quantity REAL`, §9 example `{"ticker": "AAPL", "side": "buy", "quantity": 10}` — shows an int. Purely cosmetic, but if the structured-output schema is generated from JSON Schema with `"type": "number"`, an LLM may produce `"quantity": "10"` strings. Consider stating: *"`quantity` is JSON number, not string; the backend rejects non-numeric values with 400."*

### 13. Seed consistency vs. simulator description

§2 says "10 default tickers" — §7 lists 10. Good. But §6 simulator description says "Correlated moves across tickers (e.g., tech stocks move together)" — the implementation uses sector-based Cholesky with 0.6/0.5/0.3 per `MARKET_DATA_SUMMARY.md`. The spec could either drop the detail or mention the sector-grouping so the simulator description and summary don't drift further apart.

### 14. "A retention job trims the table to the most recent 10,000 rows on each insert"

Per-insert trim on SQLite is a `DELETE ... WHERE id NOT IN (SELECT id ... ORDER BY recorded_at DESC LIMIT 10000)` — fine at this scale, but worth stating that the trim runs inside the same transaction as the insert, so the "30 seconds between inserts" cadence doesn't create a gap where another reader sees empty state.

### 15. `GET /api/health` contract

Standard, but not specified: what's the 200 body? `{"status":"ok"}`? Docker healthcheck needs to know. One line.

### 16. Robots.txt note sits under "Public deployment"

§11 says *"add a `robots.txt` with `User-agent: *` / `Disallow: /` to the static assets"* and *"This is a one-line addition in the Dockerfile."* The frontend static export already has a public directory; `robots.txt` belongs there, not in the Dockerfile. Minor; mention the location or drop the Dockerfile phrasing.

### 17. Frontend unit tests removed — explicit, good. But §2 still says "responsive but desktop-first, functional on tablet"

No testing strategy for "functional on tablet" now that frontend unit tests are out. Fine, but Playwright scenarios in §12 only reference behavior, not viewport. One sentence: "Playwright scenarios run at a single desktop viewport for v1."

---

## What Got Better (worth acknowledging)

- **Single-origin static-export + FastAPI static mount** is now unambiguous (`/app/static`, `/app/db`) — this was vague before.
- **`executed_actions` as authoritative** cleanly solves the old problem of "what if the LLM requests a trade that fails?" — server-side truth, LLM-side intent. Good.
- **Frontend-owned chat history** drops an entire table, an endpoint, and a source of persistence bugs. Worth the trade.
- **`asyncio.Lock` around chat execution** — explicitly calling out the cash-balance race on double-submit is the kind of detail that saves a post-launch bug.
- **`tmpfs` at `/app/db` for E2E** — neat way to eliminate test-run state leak without a cleanup script.
- **Input validation in §8** is a real improvement; regex for tickers, bounds for quantity, enum for side are exactly the gaps an LLM-driven trade flow creates.

---

## Bottom Line

The diff is directionally excellent — a lot of premature complexity was correctly removed. But it slips in one architecture-level change (`session_open`) that silently contradicts the "complete" market-data module, and leaves 5–6 decision points underspecified that two independent agents will resolve differently (chat history shape, portfolio history pagination, watchlist↔cache linkage, readyState→red mapping, mock LLM contract). Fixing those before agents start implementing will save a round of rework.
