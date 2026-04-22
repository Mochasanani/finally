# FinAlly — Finance Ally

An AI-powered trading workstation that streams live market data, runs a simulated portfolio, and integrates an LLM assistant that can analyze positions and execute trades on your behalf.

Built entirely by orchestrated coding agents as the capstone for an agentic AI coding course.

## Features

- Live-updating watchlist with sparkline charts and price flash animations
- Simulated portfolio starting with $10,000 — market orders, instant fill
- Portfolio heatmap, P&L chart, and positions table
- AI chat assistant that can trade and manage the watchlist via natural language
- Dark, Bloomberg-inspired terminal aesthetic

## Stack

- **Frontend** — Next.js (TypeScript, static export), Tailwind, `lightweight-charts`
- **Backend** — FastAPI (Python, `uv`), SQLite, SSE streaming
- **Market data** — built-in GBM simulator by default; Massive (Polygon.io) API optional
- **LLM** — LiteLLM → OpenRouter → `openrouter/openai/gpt-oss-120b` via Cerebras, with structured outputs

## Quick Start

```bash
cp .env.example .env        # add OPENROUTER_API_KEY
./scripts/start_mac.sh      # build + run the container
```

Open http://localhost:8000.

Stop with `./scripts/stop_mac.sh`. Data persists in the `finally-data` Docker volume.

## Environment

| Variable | Required | Purpose |
|---|---|---|
| `OPENROUTER_API_KEY` | Yes (unless `LLM_MOCK=true`) | LLM chat |
| `MASSIVE_API_KEY` | No | Real market data; omit to use the simulator |
| `LLM_MOCK` | No | `true` returns deterministic responses for tests |

## Project Layout

```
frontend/   Next.js app (static export)
backend/    FastAPI + SQLite + market data + LLM
planning/   PLAN.md and agent-facing docs
scripts/    start/stop helpers
test/       Playwright E2E + docker-compose.test.yml
```

See `planning/PLAN.md` for the full specification.

## License

See `LICENSE`.
