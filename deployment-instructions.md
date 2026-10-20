# Deployment instructions

Three services and a database. Variable names and defaults are in each repository's `.env.example`; never commit
real values.

| Service | Repository | Default port | Health check |
|---------|------------|--------------|--------------|
| frontend | [Finpass/frontend](https://github.com/Finpass/frontend) | 3000 | `/` |
| backend | [Finpass/backend](https://github.com/Finpass/backend) | 8080 | `/healthz` (liveness), `/readyz` (database) |
| ai | [Finpass/ai](https://github.com/Finpass/ai) | 8001 | `/health` |
| database | PostgreSQL with `pgvector` and `pg_trgm` | 5432 | |

## Prerequisites

Go 1.27, Python 3.13 with [uv](https://docs.astral.sh/uv/), Node.js (any version Next 16 supports), PostgreSQL 17
with the `vector` and `pg_trgm` extensions (the `pgvector/pgvector:pg17` image has both), Docker (for image builds).
External accounts: an [OpenRouter](https://openrouter.ai) API key; for the RAG index, a Cloudflare R2 bucket that holds it.
Optional: a Google OAuth client id, an SMTP account, an Arize Phoenix instance.

## A. Run locally

Clone the three repositories side by side.

**1. Database and backend**

```
docker run -d --name finpass-pg -e POSTGRES_USER=finpass -e POSTGRES_PASSWORD=finpass \
  -e POSTGRES_DB=shenzen -p 5433:5432 pgvector/pgvector:pg17

cd backend
cp .env.example .env            # set DATABASE_URL=postgres://finpass:finpass@localhost:5433/shenzen?sslmode=disable
make migrate-up
make run                        # :8080
```

`APP_ENV` is required (`development` locally). Without `SMTP_HOST` the verification code is printed in the backend log.
Optional market data: `make import-market` then `make classify-assets` (needs the data set; see the backend README).

**2. AI service**

```
cd ai
cp .env.example .env            # set OPENROUTER_API_KEY and R2_* (RAG index); INTERNAL_API_KEY = backend AI_INTERNAL_API_KEY
uv sync --extra rag
make run                        # :8001
```

Set `RAG_ENABLED=false` to start without the index and the models (chat then answers without sources). On the first
start with RAG on, the service downloads the index and the reranker model, which takes a while.

**3. Frontend**

```
cd frontend
cp .env.example .env.local      # NEXT_PUBLIC_API_BASE_URL=http://localhost:8080, NEXT_PUBLIC_GOOGLE_CLIENT_ID
npm install
npm run dev                     # :3000
```

Open http://localhost:3000, register, verify with the code from the backend log, and fill in the questionnaire.

## B. Deploy (Railway, or any Docker host)

Every repository has a `Dockerfile`; the backend and ai also have a `railway.json` with the health check.
Deploy in this order, because each step needs a value from the one before.

1. **PostgreSQL.** Create the database. Use `sslmode=require` (or stronger) in the connection string.
2. **Migrations.** Run them once per release:
   `docker build --target migrate -t finpass-migrate backend && docker run --rm finpass-migrate "$DATABASE_URL" up`
3. **AI service.** Create a service from `Finpass/ai`. Set `APP_ENV=production`, `INTERNAL_API_KEY` (a long random
   value), `OPENROUTER_API_KEY`, `OPENROUTER_MODEL`, `RAG_MODEL`, and the `R2_*` variables (use a read-only key).
   Run **one worker only** (ChromaDB's SQLite files do not support several processes). Mount a volume at `/models`
   so the reranker model is not downloaded on every deploy. Plan memory for a ~570M-parameter model plus the index.
4. **Backend.** Create a service from `Finpass/backend`. Set `APP_ENV=production`, `DATABASE_URL`, `JWT_SECRET`
   (32+ random characters), `AI_SERVICE_URL` (the AI service's address, including `http://` or `https://`),
   `AI_INTERNAL_API_KEY` (same value as the AI service's `INTERNAL_API_KEY`), `INTERNAL_API_KEY`, `GOOGLE_CLIENT_ID`,
   `SMTP_*` and `MAIL_FROM`, `CORS_ORIGINS` and `PUBLIC_APP_URL` (the frontend's public URL).
   In production the API refuses to start with development secrets, `sslmode=disable`, or no SMTP.
   If proxies sit in front of the API, set `TRUSTED_PROXY_HOPS` to their real number (default `0`).
5. **Frontend.** Create a service from `Finpass/frontend` (or deploy to Vercel). Set, at build time,
   `NEXT_PUBLIC_API_BASE_URL` (the backend's public URL) and `NEXT_PUBLIC_GOOGLE_CLIENT_ID`. These are public values.
6. **WhatsApp brief (optional).** Off by default (`WA_DAILY_ENABLED`). Its variables are listed in the backend `.env.example` and README.
7. **Tracing (optional).** Set `PHOENIX_COLLECTOR_ENDPOINT` (and `PHOENIX_API_KEY`) on the AI service. Prompts and
   answers are sent only when `TRACING_CAPTURE_CONTENT=true`; keep it off in production. To show cost in Phoenix, run
   `uv run python scripts/phoenix_sync_prices.py` once (see `ai/docs/TRACING.md`).

## C. Check that it works

```
curl -fsS https://<backend>/readyz                 # database reachable
curl -fsS https://<ai>/health                      # AI service up
curl -s -o /dev/null -w "%{http_code}\n" https://<ai>/ml/profile/explain   # 401: refuses calls without the internal key
```

Then in the browser: register and verify the email, complete the questionnaire (a profile result appears), open the
dashboard, and send one chat message (an answer with citations appears).

## Troubleshooting

| Symptom | Likely cause |
|---------|--------------|
| Backend exits at start with a message about `APP_ENV`, secrets, `sslmode` or SMTP | production safety checks; set the variable it names |
| Browser shows a CORS error | the frontend URL is missing from `CORS_ORIGINS` |
| Chat returns an upstream error (502/504) | `AI_SERVICE_URL` wrong, or `INTERNAL_API_KEY` differs between the two services |
| AI service slow to start or out of memory | it downloads the index and loads the reranker at start; give it more RAM, mount `/models` |
| Phoenix cost shows 0 | run `scripts/phoenix_sync_prices.py` for your Phoenix instance |
| Rate limiting blocks everyone | `TRUSTED_PROXY_HOPS` is 0 behind a proxy, so all clients share the proxy address; set the real hop count |
