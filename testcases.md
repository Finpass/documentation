# Test cases

All suites run without internet access, API keys or model downloads: the language model, embeddings and vector store
are replaced by fakes, and HTTP peers by `httptest` servers. Only a few backend tests need a PostgreSQL database
and skip themselves without one. CI runs the suites on every pull request in each repository.

Last full run (all passing): ai 132 tests, frontend 139 tests in 17 files, backend all packages (6 database-backed tests skip without a database).

## Run them

| Repository | Command | Also runs |
|------------|---------|-----------|
| [Finpass/ai](https://github.com/Finpass/ai) | `uv sync --extra rag --group dev && uv run pytest -q` | `make check` adds ruff and mypy |
| [Finpass/backend](https://github.com/Finpass/backend) | `go test -race -count=1 ./...` | `make check` adds gofmt and the linter |
| [Finpass/frontend](https://github.com/Finpass/frontend) | `npm install && npm run test` | `npm run check` adds eslint and `tsc --noEmit` |

Backend tests that need PostgreSQL (use a throwaway database, they insert and delete rows):

```
createdb finpass_test
DATABASE_URL=postgres://localhost/finpass_test?sslmode=disable make migrate-up
SEC_TEST_DATABASE_URL=postgres://localhost/finpass_test?sslmode=disable make test
```

## AI service (pytest)

| File | What it checks |
|------|----------------|
| `tests/test_api.py` | success and error envelope, status codes, internal API key required, request id returned |
| `tests/test_rag.py` | chat flow: retrieval, citations, guardrail rejects ungrounded numbers, retry, refusal and source-excerpt fallback, no-document answer |
| `tests/test_profile.py` | profile facts, number guardrail on the explanation, template fallback when the model fails or is rejected, Indonesian and English |
| `tests/test_chroma_retrieval.py` | vector search filters, reranking order, year balance and score cut-off on a small index |
| `tests/test_hardening.py` | prompt fencing of untrusted text, internal key rotation, provider settings |
| `tests/test_tracing.py` | span tree, token and cost totals, hidden content when capture is off, setup never breaks the service |
| `tests/test_config.py` | production refuses the development API key |

## Backend (go test)

| Area (`internal/...`) | What it checks |
|-----------------------|----------------|
| `features/auth` | registration, email code, login, refresh rotation and replay revocation, password reset, Google link rules |
| `features/profile` | rule-engine components and boundaries (40/70/3/6/12/25), 36 scenarios, monotonicity properties, 10,000 simulated answer sets, validation, retake cooldown |
| `features/market` | asset classifier thresholds and 20,000 random inputs, news category rules; integration tests skip on an empty database |
| `features/portfolio`, `dashboard`, `news` | valuation, FX, history, weights, ownership isolation, limits, HTTP status and envelope |
| `features/matching`, `whatsapp` | matching engine, scheduler, opt-in and STOP handling, webhook signature, message client against a fake server |
| `features/advisory` | chat sessions, message limits, user-context building (no name, email or phone is sent) |
| `platform/...` | error envelope, rate limiting and trusted-proxy handling, field encryption, audit chain, retention purge, config checks (production refuses unsafe settings) |

## Frontend (vitest)

| Area | What it checks |
|------|----------------|
| `lib/api-error` | every failure becomes one `ApiError`; server errors hide detail and show the request id |
| `features/auth/validation`, `features/profile/validation` | form rules and error messages |
| `features/profile/*` | numeric input, radar chart data, Indonesian/English text |
| `features/dashboard/lib/*` | matching, comparison, goals, projection, holdings, charts, formatting, discover, advisory helpers |
| `features/dashboard/i18n`, `design-system/colors` | translation keys complete in both languages; design tokens |

## Manual end-to-end checklist

Run once after a deploy ([deployment-instructions.md](deployment-instructions.md), part C).

| # | Step | Expected |
|---|------|----------|
| 1 | Register with an email | a 6-digit code arrives by email (backend log in development) |
| 2 | Enter a wrong code, then the right one | wrong code rejected; right code signs you in |
| 3 | Open the questionnaire without consenting | answers are refused until consent is given |
| 4 | Complete all sections and submit | a profile result with scores and an explanation appears |
| 5 | Retake immediately | refused, with the date the next retake is allowed |
| 6 | Open the dashboard, add a holding | portfolio value and weights update |
| 7 | Ask the chat a regulation question | an answer with numbered citations; a disclaimer is shown |
| 8 | Ask the chat for a number it cannot know | it refuses or answers from sources only, never inventing a figure |
| 9 | Call the AI service without the internal key | 401 |
| 10 | Switch language | text changes in both the app and the explanation |

## Not covered by automated tests

- Real calls to OpenRouter, Cloudflare R2 and WhatsApp (faked in tests).
- Browser-level (end-to-end UI) tests; the manual checklist above stands in for them.
- Load and capacity tests, including the AI service's memory use.
