# Finpass

Finpass is a wealth advisory agent for FinTechathon Shenzhen, Topic 3. It profiles a person's finances with a
deterministic rule engine, shows a portfolio and market dashboard, answers questions in a chat grounded in
Indonesian financial regulation and news (with citations), and can send a daily brief over WhatsApp.
The app is available in Indonesian and English.

Rules decide, the model only words the result: risk profile, scores and recommendations come from code that can be
tested; the language model writes the explanation and the chat answer, and a guardrail rejects answers that
contain numbers the system did not provide.

## Source code

| Repository | Stack | Role |
|------------|-------|------|
| [Finpass/frontend](https://github.com/Finpass/frontend) | Next.js 16, React 19, TypeScript, Tailwind 4 | Web app: sign-in, questionnaire, dashboard, advisor chat |
| [Finpass/backend](https://github.com/Finpass/backend) | Go, chi, PostgreSQL (pgx, goose, sqlc) | API: auth, profile rule engine, portfolio, market data, news, matching, WhatsApp brief |
| [Finpass/ai](https://github.com/Finpass/ai) | Python 3.13, FastAPI, ChromaDB, OpenRouter | RAG advisor chat and profile wording; only the backend calls it |
| [Finpass/documentation](https://github.com/Finpass/documentation) | Markdown | This repository |

Each repository has its own README with architecture notes, commands and environment variables.

## Documents in this repository

| File | Content |
|------|---------|
| [README.md](README.md) | This overview |
| [deployment-instructions.md](deployment-instructions.md) | Run it locally and deploy it, step by step |
| [testcases.md](testcases.md) | How to run every test suite and what each one checks |

## How it fits together

```
Browser ── frontend (Next.js, :3000)
              │  HTTPS, JSON envelope, Bearer token + refresh cookie
              ▼
           backend (Go, :8080) ──── PostgreSQL
              │  X-Internal-Api-Key
              ▼
           ai (FastAPI, :8001) ──── ChromaDB index (downloaded from Cloudflare R2)
                     │
                     └── OpenRouter (LLM and embeddings)      optional: Arize Phoenix (traces)
```

- The browser only talks to the backend. The AI service is reachable only with the shared internal key.
- All three services use one response and error format (see "Response and error contract" in the backend README).
- Personal data (chat messages, phone numbers, questionnaire answers) is encrypted in the database; the audit trail is append-only.

## Modules

1. **Financial profiling engine**: questionnaire v4.0 with autosave, consent, a deterministic rule engine, immutable assessments, and an AI-written explanation with a template fallback.
2. **Dashboard and advisory chat**: portfolio valuation, asset classification, discover and news pages, and a RAG chat with citations and a number guardrail.
3. **Personalized daily brief**: matching engine for daily picks and a WhatsApp message scheduler (opt-in, per-user time; optional, off by default).

## Diagrams

Standalone HTML files in [`diagrams/`](diagrams/); open them in a browser (clone the repo, or use an HTML preview).

| File | Shows |
|------|-------|
| `system-architecture.html`, `system-architecture-layered.html` | services, data stores and external providers |
| `module-breakdown.html` | the three modules: profiling engine, dashboard and advisory chat, daily brief |
| `core-algorithms.html` | the algorithm behind each module (14 diagrams) |
| `chat-sequence.html` | one advisory chat turn, end to end |
| `matching-engine.html` | the five rule layers of product matching |
| `trust-layers.html` | rules decide, the language model only words the result |
| `user-journey.html` | a user from first visit to the daily brief |
| `data-model.html` | the core tables |
| `security-defence-in-depth.html`, `security-data-protection.html` | security layers and how personal data is protected |

## Quick start

See [deployment-instructions.md](deployment-instructions.md). Short version: PostgreSQL, then `make migrate-up` and
`make run` in the backend, `make run` in the AI service, `npm run dev` in the frontend; open http://localhost:3000.

## Limits you should know

- The advisor gives general information, not licensed investment advice; every answer carries a disclaimer.
- The AI service loads a reranker model in memory, so it needs far more RAM than the other two services.
