# Crypto Intelligence Platform

Read-only crypto wallet monitoring Telegram Mini App. Tracks public wallet balances, tokens, and transactions, with configurable alerts and plain-text activity summaries.

The service never requests or stores private keys, seed phrases, or passwords. All data is read from public blockchain APIs.

## Architecture

- Frontend: React 18, TypeScript, Vite, TanStack Query, Tailwind CSS.
- Backend: Python 3.12, FastAPI, Pydantic v2, SQLAlchemy 2 (async), Alembic, asyncpg, Redis, httpx.
- Bot & Worker: aiogram 3.x Telegram bot, background task worker with distributed locks.
- Storage: PostgreSQL 16, Redis 7.

```
Telegram client -> aiogram bot (onboarding)
                -> React Mini App -> FastAPI -> PostgreSQL
                                             -> Redis (rate limits, locks, replay cache)
                                             -> Etherscan API
                                             -> LLM provider (OpenAI-compatible)
                <- worker sends alerts via bot
```

Layout:

```
apps/
  api/     FastAPI backend, Alembic migrations, test suite
  bot/     aiogram 3 bot
  web/     React Mini App
  worker/  Background task runner
packages/
  shared/  Shared schemas and domain models
infra/     Dockerfiles, nginx configs
```

## Security

- Identity: Verified via Telegram HMAC `initData`. The backend does not trust raw client-supplied IDs.
- Replay prevention: Redis nonce cache drops reused `initData` payloads.
- Outbound requests: Custom transport resolves DNS before connecting, blocking loopback, RFC1918, link-local, and cloud metadata IPs.
- Access control: Every query filters by user ownership; unauthorized IDs return 404.
- Rate limiting: Redis Lua scripts enforce per-user authenticated limits and per-IP login limits.
- Sanitized outputs: Summary output strips raw HTML, markdown scripts, and URLs before presentation.

## Quick Start (Docker)

```bash
cp .env.example .env
docker compose up -d db redis
docker compose --profile migrate run --rm migrate
docker compose up -d api worker web
```

- Web UI: `http://localhost:5173`
- API health: `http://localhost:8000/api/v1/health/live`
- API docs: `http://localhost:8000/docs` (debug mode only)

## Local Development

Backend:

```bash
python -m venv .venv && source .venv/bin/activate
pip install -e packages/shared -e .[dev]
alembic upgrade head
uvicorn apps.api.app.main:app --reload --port 8000
```

Worker and bot:

```bash
python -m apps.worker.worker.run_worker
python -m apps.bot.bot.main
```

Frontend:

```bash
cd apps/web
npm ci
npm run dev
```

## Testing

Run unit and integration checks:

```bash
pytest apps/api/tests -ra
ruff check apps packages
mypy apps/api/app apps/worker/worker apps/bot/bot packages/shared/shared
```

## License

MIT (see [LICENSE](LICENSE)).
