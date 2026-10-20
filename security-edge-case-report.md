# Security edge case report

Hostile-input testing of the Finpass backend (Go) and AI service (Python). 76 edge cases (46 for the AI service, 30 for the backend), implemented as automated tests that run on every build.

## 1. Summary

| | |
|---|---|
| What was tested | The HTTP surface of the two services that take untrusted input: authentication, input validation, injection, prompt injection, data leakage, output safety, rate limiting, and availability |
| Edge cases | **76**: AI-01 to AI-45 plus AI-07b, and BE-01 to BE-29 plus BE-18b |
| Automated tests | AI: 199 tests in `tests/test_security_edge.py` (198 pass, 1 expected failure). Backend: 48 tests and subtests in `internal/app/security_edge_test.go` plus 3 unit tests in `internal/platform/httpx/edge_test.go` |
| Method | Write the test for the *secure* behaviour first, run it against the code as it was, then fix what failed. The "Before" column below is the result of that first run |
| Real defects found | **17** (F-01 to F-17). 16 fixed and covered by a regression test; 1 accepted with a reason (F-17) |
| Controls that held | 50 of the 76 cases passed on the first run without any change (24 exposed a defect, 1 is a known limit, 1 is an accepted risk) |
| Not covered | See section 6. Most important: model-level jailbreaks cannot be fully prevented by any filter, and the browser side (how the frontend renders text) was not tested |

**Result in one line:** the authentication, authorisation, rate limiting, CORS and error handling were solid. The weak spots were in the AI service: several protections that had been written were never connected to the live endpoint, and the input and output filters had gaps.

### How the first run went (honest numbers)

The first run of the AI suite had 86 failures. 37 of them were mistakes in the tests themselves, which were corrected: the prompt-tag test compared against the wrong baseline (36 cases) and one test imported a function from the wrong module. A recount of the tag test against the *old* code showed that 5 of the 9 breakout variants genuinely got through, so that finding is real. The remaining 49 failures were real defects.

## 2. How to reproduce

```
# AI service
cd ai && uv sync --extra rag --group dev && uv run pytest tests/test_security_edge.py -q

# Backend (needs a throwaway PostgreSQL with the migrations applied; the tests create and delete sec-*@edge.test users)
cd backend
SEC_TEST_DATABASE_URL=postgres://localhost/finpass_test?sslmode=disable go test -p 1 -count=1 -run TestBE ./internal/app/
```

Use `-p 1` when running the whole backend suite against a database: several packages write to the same tables and race each other otherwise.

No network, API key or model is needed: the language model and the retriever are replaced by fakes that record what they were sent.

## 3. Edge case catalogue

Result column: **Held** = the code was already safe. **Found** = a defect, fixed (finding id given). **Accepted** = known and documented. **Limit** = a known limit that is tested as an expected failure.

### 3.1 AI service

| ID | Edge case | Expected | Before | Finding / control |
|----|-----------|----------|--------|-------------------|
| AI-01 | 8 bad internal keys (empty, blank, wrong, 1 char longer, 1 char shorter, 50 KB, non-ASCII, SQL-like) on all 3 protected routes | 401, no 500 | Held | Constant-time compare against every accepted key |
| AI-02 | Key sent in the query string or the body instead of the header | 401 | Held | Only the header is read |
| AI-03 | Production: is the API schema hidden? | `/docs`, `/redoc`, `/openapi.json` all off | Found | F-03 |
| AI-04 | `/health` discloses nothing | body has only the envelope | Held | |
| AI-05 | 14 invalid chat bodies (empty, blank, 1001 chars, wrong types, bad language, `system` role in history, 13 turns, 4001-char turn, run id with `../`, space, newline, 129 chars) | 422 | 13 held, blank message **found** | F-07 |
| AI-06 | 9 odd but valid messages (NUL bytes, right-to-left override, zero-width, 1000 emoji, SQL, template and log4shell strings, HTML, control characters) | 200, no crash | Held | |
| AI-07 | Payload bombs: 100,000 holdings, 100,000 held kinds, 5,000 recommended assets, 100,000 priorities | rejected fast | Found (timed out at 20 s) | F-05 |
| AI-07b | A 3 MB body, declared by length and streamed in chunks | rejected before parsing | Found (no limit existed, by code review) | F-04 |
| AI-08 | `NaN`, `Infinity`, `-Infinity` in a numeric field | 422 | Found | F-06 |
| AI-09 | A 400-digit integer | no crash | Held | |
| AI-10 | JSON nested 100,000 levels deep | 4xx | Held | |
| AI-11 | Form body, truncated JSON, invalid UTF-8, empty body, wrong content type | 400 or 422 | Held | |
| AI-12 | GET/PUT/DELETE/PATCH on the chat route | 405 | Held | |
| AI-13 | Path traversal and odd paths (`../`, `%2e%2e%2f`, `%00`, `/.env`, `//`) | 404, no 500 | Held | |
| AI-14 | Turn-trail query abuse (`since=-1`, `abc`, 20-digit number, `1e9`, empty) | no 500 | Held | |
| AI-15 | Does run A's trail leak into run B? | no | Held | |
| AI-16 | Inbound request id with 10,000 chars, spaces, quotes and HTML, non-ASCII | echoed id is a plain token of at most 64 chars | Found | F-13 |
| AI-17 | 7 prompt-injection phrases (English, Indonesian, zero-width space inside the keyword, fullwidth letter) | blocked before the model | Found (7 of 7 reached the model) | F-01 |
| AI-18 | A normal regulation question that contains the word "instruksi" | not blocked | Held | No false positive |
| AI-19 | Leetspeak injection (`1gn0re`) | blocked | **Limit** | Expected failure; see section 6 |
| AI-20 | The message reaches the model only as the last user turn; a fake `SYSTEM:` line in history stays an assistant turn | yes | Held | |
| AI-21 | Only the last 6 history turns are forwarded | yes | Held | |
| AI-22 | 9 ways to close or reopen the `<fakta>` and `<sumber>` data blocks, in 4 fields (news title, news summary, goal, holding name) | no tag survives | Found (5 of 9 variants survived: inner spaces, attributes, zero-width, fullwidth, soft hyphen) | F-08 |
| AI-23 | 8 personal identifiers in the question and history (national id plain, spaced, dashed; email; phone local, international, spaced, dashed) | never sent to the model or the retriever | Found (8 of 8 sent) | F-01 |
| AI-24 | A phone number inside the stated goal | masked | Found | F-01 |
| AI-25 | 5 answers that leak prompt structure or secrets (`<fakta>` tag, `KUTIPAN SUMBER:`, `FAKTA USER:`, an `sk-or-` key, a bearer token) | never returned | Found (5 of 5 returned) | F-09 |
| AI-26 | A leaky first answer followed by a clean retry | clean one accepted | Found | F-09 |
| AI-27 | The API key in logs, errors and `repr` after a failed call | absent | Held | |
| AI-28 | An internal exception with a password in its message | 500 with a generic message | Held | |
| AI-29 | Model output with a markdown image to an attacker URL, `<script>`, `<img onerror>`, a `javascript:` link | stripped | Found (4 of 4 passed through) | F-10 |
| AI-30 | A 2 MB model answer | cut | Found | F-11 |
| AI-31 | An empty or blank model answer | falls back | Found | F-11 |
| AI-32 | Forged citation markers (`[100]`, `[1][2][3]` with 2 sources, `[4]`) | never become citations | Held | |
| AI-33 | 10 ways to tell the user to buy or sell, or to promise a profit (imperative, "must", "recommend", "guaranteed", "risk-free", zero-width space inside the verb) | refused | Found (8 of 10 passed) | F-12 |
| AI-34 | 5 plain education sentences ("sebelum membeli", "jual beli saham", ...) | not refused | Held | No false positive |
| AI-35 | A number the sources do not contain | detected | Held | The function works; see section 6 on enforcement |
| AI-36 | 40 simultaneous chats | at most `RAG_MAX_CONCURRENT` model calls at once | Found | F-02 |
| AI-37 | A model that always breaks the rules | at most 2 model calls per question | Held | Bounded cost |
| AI-38 | Model timeouts | fallback excerpts, no error | Held | |
| AI-39 | 7 malformed provider responses (HTML page, empty choices, null content, list content, truncated JSON, 429, 500) | `LLMError` only | Held | |
| AI-40 | 5 pathological answers of 50 KB (unclosed images, brackets, digit dots, repeated phrases) | each filter under 1 s | Held | No ReDoS in the answer filters |
| AI-41 | Masking is fast on the longest allowed inputs | under 1 s | n/a (no masking ran in the live path) | F-01; no ReDoS in the new patterns |
| AI-42 | Free text in the profile request (name, answers, flag code) reaches the model | no | Found (a flag code went into the prompt as free text) | F-14 |
| AI-43 | 8 invalid or oversized profile requests (extra field, bad enums, negative income, 5,000 answers, 5,000 flags) | 422 | 6 held, 2 found | F-14 |
| AI-44 | 5 hostile model replies for the profile text (script tag, 5,000 chars, missing key, wrong type, invented number) | template fallback | 4 held, script tag **found** | F-15 |
| AI-45 | Non-JSON model reply for the profile | template fallback | Held | |

### 3.2 Backend

| ID | Edge case | Expected | Before | Finding / control |
|----|-----------|----------|--------|-------------------|
| BE-01 | 4 SQL payloads in login fields (`' OR '1'='1`, `DROP TABLE`, `UNION SELECT`) | 401 or 422, table intact | Held | Parameterised queries (sqlc) |
| BE-02 | SQL and control bytes in the search parameters (7 payloads incl. `%00`, 5,000 chars) | no 500 | Found (`%00` gave 500) | F-16 |
| BE-03 | A name like `Robert'); DROP TABLE users;--` | stored as plain text | Held | |
| BE-04 | Forged tokens: `alg:none` (library, hand built, no signature segment), tampered payload, garbage, empty segments, 100 KB | 401 | Held | Only HS256 is accepted |
| BE-05 | A token signed with another secret, or with HS512 | 401 | Held | |
| BE-06 | Expired token, token with no expiry | 401 | Held | Expiry is required |
| BE-07 | Wrong issuer | 401 | Held | |
| BE-08 | Empty subject, unknown user, SQL in the subject | 401 | Held | |
| BE-09 | 9 `Authorization` variants, token in the query string, token in a cookie | 401 | Held | |
| BE-10 | User B patches, deletes and lists user A's holding; tries to reassign it with `userId` | 404, nothing changes | Held | Ownership is part of every query |
| BE-11 | 9 failed logins with a different `X-Forwarded-For` each time | 429 after 5; the header changes nothing | Held | Proxy headers ignored unless a proxy count is configured |
| BE-16 | Passwords: 7, 72, 73 chars, 144-byte multi-byte, emoji, spaces, NUL, 10 KB, SQL, combining marks | never a 500 | Held | Rune count and bcrypt byte limit both handled |
| BE-17 | E-mail case and spacing, 250-char and non-ASCII addresses | one account per address | Held | |
| BE-18 | Do login, resend and forgot-password reveal whether an address has an account? | identical responses | Held | |
| BE-18b | Does registering an existing address reveal it? | identical | Accepted | F-17 |
| BE-19 | 8 wrong verification codes | locked after 5, account not verified | Held | |
| BE-12 | A 2 MB body | 4xx, not read in full | Held | 1 MB cap |
| BE-13 | Mass assignment (`email_verified`, `id`, `role` in the body) | refused, or the flag ignored | Held | Unknown fields are rejected |
| BE-14 | JSON nested 200,000 levels | 4xx | Held | |
| BE-15 | 12 malformed bodies (empty, array, null, NaN, two objects, wrong types, form body, escaped NUL ...) | 4xx | 11 held, escaped NUL (`\u0000`) **found** | F-16 |
| BE-20 | 7 hostile origins (`null`, look-alike hosts, other ports, https) on GET and preflight | no CORS headers | Held | The configured origin is allowed; never `*` |
| BE-21 | Security headers on 404, 405, 401, 422, health, preflight and bad requests | all present | Held | `Cache-Control: no-store` on personal data |
| BE-22 | The same hostile request ids as AI-16 | plain token | Found | F-13 |
| BE-23 | 8 odd paths and hostile ids (traversal, double encoding, 3,000 segments, `%00`) | no 500 | Found (`%00` gave 500) | F-16 |
| BE-24 | Method override header, TRACE, CONNECT, PROPFIND, PATCH, DELETE on a read route | no 500, no state change | Held | |
| BE-25 | Old access token after the account was deleted | 401 | Held | |
| BE-26 | Forged refresh cookies; cookie flags | 401; HttpOnly and SameSite set; refresh token not in the body | Held | |
| BE-27 | 11 deliberately broken requests across profile, dashboard, news, discover, advisory | no database error, stack trace, path or secret in any response | Held | Checked on **every** response of every backend test |
| BE-28 | Sign-in before the e-mail is verified | refused | Held | |
| BE-29 | Data export called 5 times | limited | Held | 3 per hour |

## 4. Findings

Severity is the author's judgement of impact for this product, not a CVSS score.

| ID | Severity | Service | Finding | Fix | Regression test |
|----|----------|---------|---------|-----|-----------------|
| F-01 | **High** | AI | National id, e-mail and phone number masking, and the prompt-injection block, existed only in a module that no route used. The live chat sent identifiers straight to the model provider | Moved into the domain guardrail, wired into the live service, and extended to spaced and dashed forms; also applied to the stated goal | AI-17, AI-23, AI-24 |
| F-02 | Medium | AI | The limit on simultaneous model calls (`RAG_MAX_CONCURRENT`) was in the same unused module, so a burst of requests was unbounded in cost and load | A per-event-loop semaphore in the live service | AI-36 |
| F-03 | Low | AI | In production `/docs` was hidden but `/openapi.json` and `/redoc` were open, handing out the full API surface | All three disabled in production | AI-03 |
| F-04 | Medium | AI | No request body size limit | 256 KB limit, enforced for declared and streamed bodies | AI-07b |
| F-05 | Medium | AI | Lists and strings in the chat request were unbounded (100,000 holdings made a request take more than 20 s) | Length limits on every list and string | AI-07 |
| F-06 | Low | AI | `NaN` and `Infinity` were accepted and printed into the prompt | Refused at validation | AI-08 |
| F-07 | Low | AI | A blank (whitespace-only) question was accepted | Refused | AI-05 |
| F-08 | Medium | AI | Untrusted text (news, goal, holding names) could close or reopen the `<fakta>` and `<sumber>` data blocks: 5 of 9 variants got through (inner spaces, attributes, zero-width and soft-hyphen characters, fullwidth brackets) | Text is Unicode-normalised first, then any tag-like token, with attributes or spaces, is removed | AI-22 |
| F-09 | Medium | AI | An answer that repeated the prompt structure or contained a key or token was returned to the user | Such answers are rejected like any other rule violation (retry, then fallback) | AI-25, AI-26 |
| F-10 | Medium | AI | Model output could carry markdown images (a data exfiltration channel), raw HTML and `javascript:` links | Images, HTML and script links are removed from every answer, including source excerpts | AI-29 |
| F-11 | Low | AI | Empty answers and answers of any size were returned | Empty is a rule violation; answers are cut at 8,000 characters | AI-30, AI-31 |
| F-12 | Medium | AI | The "no buy/sell advice" filter caught 2 of 10 realistic phrasings, and nothing stopped profit guarantees | Broader patterns for commands, "you must", recommendations and guarantees, checked on normalised text; education sentences still pass | AI-33, AI-34 |
| F-13 | Low | AI and backend | An inbound `X-Request-Id` of up to 64 arbitrary characters was echoed in the response and written to the logs (log and header injection) | Only plain tokens (letters, digits, `_`, `-`) are accepted; anything else gets a fresh id | AI-16, BE-22 |
| F-14 | Low | AI | A profile flag code was free text that went into the prompt; answers and flags were unbounded | Flag codes must be lowercase snake case; at most 50 flags and 200 answers | AI-42, AI-43 |
| F-15 | Low | AI | HTML in the model's profile wording was accepted | Markup is rejected, the template is used | AI-44 |
| F-16 | Low | Backend | A NUL character in a query value, a path value, or escaped inside a JSON string reached PostgreSQL and produced a 500 | Rejected with 400 before the handler; JSON bodies are checked for the escape | BE-02, BE-15, BE-23 |
| F-17 | Info | Backend | Registering an address that already has an account answers 409, while login, resend and forgot-password do not reveal accounts | **Accepted**: the message tells a person who forgot they registered to sign in. Registration is limited to 30 per minute per client address | BE-18b (fails if the behaviour changes, so the decision is revisited) |

### Where the defects came from

F-01 and F-02 deserve a note. A teammate had written the masking, the injection block and the concurrency limit in a service module, but the router calls a different, older service. The unit tests for those functions passed, so everything looked covered. Only a test that drives the real endpoint and looks at what the model receives could see that none of it ran. This is the main argument for testing through the HTTP layer.

## 5. Results

| Suite | Before the fixes | After the fixes |
|-------|------------------|-----------------|
| AI `test_security_edge.py` | 86 failed (37 test mistakes, 49 real), 110 passed, 1 expected failure | **198 passed, 1 expected failure** |
| AI full suite | 135 passed | **333 passed, 1 expected failure** |
| Backend `TestBE*` | 4 test functions failed: 2 real defects (fixed), 1 accepted risk, 1 mistake in the test | **51 tests and subtests pass, 0 fail, 0 skipped** (48 in `internal/app`, 3 unit tests in `httpx`) |
| Backend full suite (`-p 1`, with database, race detector) | all passed | **all 40 packages pass** (run before the 3 `httpx` unit tests were added; those pass in the line above) |

## 6. What this does not prove

- **Jailbreaks.** A filter cannot stop every attempt to steer a language model. The design limits the damage instead: the model never decides anything (rules do), untrusted text is fenced as data, answers are checked before they are shown, personal identifiers are masked, and a question costs at most two model calls. AI-19 (leetspeak such as `1gn0re`) is kept as an expected failure so this limit stays visible.
- **Invented numbers.** `untraceable_numbers` detects a figure that is not in the sources (AI-35) but the setting `RAG_ENFORCE_NUMBERS` is not read by the live chat, and the existing tests state that model-computed numbers are allowed. The profile explanation does enforce its number check.
- **Rate limiting in the AI service.** There is none by design: only the backend can call it (shared secret), and the backend limits chats per user. The AI service limits concurrency (F-02) and body size (F-04), not request rate.
- **Frontend.** How the browser renders model text, and whether it could execute anything, was not tested. The AI service now strips images, HTML and script links (F-10), which removes the main risk at the source.
- **Timing attacks** on the key comparison were reviewed in code (constant time) but not measured.
- **WhatsApp webhook** signature handling is covered by the existing tests in the WhatsApp package, not repeated here.
- **Load and capacity.** Concurrency was tested for correctness, not for throughput.
- **Real providers.** OpenRouter, Cloudflare R2, SMTP and the WhatsApp gateway were faked.

## 7. Recommendations

1. Keep the edge case tests in CI (they need no network). The backend ones need a database; run them with `-p 1`.
2. Decide whether `RAG_ENFORCE_NUMBERS` should be wired into the live chat. It would reduce invented figures at the cost of more fallbacks.
3. Revisit F-17 if account discovery becomes a concern; the test will fail when the behaviour changes.
4. Add a browser-level check that model text is rendered as plain text.
5. When teammates add a protection, add a test that drives the real route. A unit test of an unused function gives false comfort (see F-01).
