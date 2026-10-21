# Web App QA & Debug Report — RealWorld (Conduit)

**Tester:** Sasi Kumar Yadav · **Project:** Personal QA audit of the RealWorld (Conduit) reference app · **Date:** 11 October 2026

## 1. Scope and method
The test plan targeted the hosted demo at demo.realworld.io. During the test window the public demo API
(api.realworld.io) was unreachable — every request returned HTTP 530 (origin error) — so the
official RealWorld backend (gothinkster/node-express-realworld-example-app, Express + Prisma) was run locally and
exercised directly over its REST API. All planned flows were covered: sign-up, login, article create / edit /
delete, favorites, follows and comments, plus edge cases: invalid input, duplicate data, cross-user authorization,
malformed payloads, pagination abuse and repeated login failures. Every issue below was reproduced live; status
codes and response bodies are quoted from the actual runs.

## 2. Issues found

| # | Title / Summary | Steps to Reproduce | Expected vs Actual | Severity | Suspected Cause |
|---|---|---|---|---|---|
| 1 | Invalid email addresses are accepted at registration | POST /api/users with email "not-an-email" | Expected 422; actual **201** — full account + token created | High | No format validation on email; only presence/uniqueness checked |
| 2 | No rate limiting or lockout on login | POST /api/users/login with wrong password 8× rapidly | Expected throttling/lockout; actual all 8 processed instantly (403 each) | High | No throttling middleware or failed-attempt tracking |
| 3 | Negative pagination crashes the list endpoint and leaks server internals | GET /api/articles?limit=-5&offset=-1 | Expected 400/clamped; actual **500** with the Prisma query-construction error, internal file path and line (article.service.ts:77) | High | limit/offset passed to the ORM unvalidated; no global error handler |
| 4 | Script/HTML payloads stored verbatim (stored-XSS risk) | Create article titled `<script>alert(1)</script>`, onerror img in description | Expected escaped/rejected; actual **201**, fields stored and returned byte-for-byte | High | No sanitisation at write/read; safety left to whichever frontend renders the data |
| 5 | Malformed JSON returns 500 with a raw parser error | POST /api/users/login with body `{not json` | Expected 400; actual **500** with the parser's raw message | Medium | Body-parser syntax error not mapped by error middleware |
| 6 | A user can follow themselves | POST /api/profiles/{own-username}/follow with own token | Expected 422/400; actual **200**, following: true | Medium | Follow handler never checks target != requester |
| 7 | No maximum length on article titles | Create article with a 600-character title | Expected a sane cap + 422; actual **201**, ~600-char slug generated | Low | No length constraint in validation or schema |
| 8 | Inconsistent authorization errors | Delete another user's comment vs edit another user's article | Comment delete returns **404**; article edit returns **403** with a message — two conventions for the same error class | Low | Services written to different patterns (scoped query vs explicit ownership check) |

## 3. Root-cause analysis — Issue 3 (negative pagination)
**What happens:** GET /api/articles?limit=-5&offset=-1 returns HTTP 500 whose body is the ORM's internal
query-construction error, including the server-side source file path and exact line number.

**Why:** the list endpoint reads limit/offset straight from the query string into Prisma's findMany take/skip.
Negative values are meaningless to the query planner, so Prisma throws during query-graph construction. With no
input-validation layer and no central error-handling middleware, the exception reaches Express's default handler,
which serialises the error — internals included — into the response. Two missing layers, not one bug: validation
would have prevented the throw; a proper error handler would have contained it.

**Fix:** (1) validate/clamp pagination at the route boundary (integers, limit in [1,100], offset ≥ 0 — a small zod
schema on the query object); (2) add a global error middleware mapping client errors to 400/422 and everything else
to a generic 500 with a correlation ID, stacks logged server-side only; (3) regression tests for limit=-1, offset=-1,
limit=abc, limit=100000.

## 4. What worked correctly (verified)
Duplicate email → 422 "has already been taken"; blank title/password → 422; duplicate article title → 422
"must be unique"; article create without token → 401; editing another user's article → 403 and article unchanged;
delete own article → 204, later GET → 404; favoriting twice is idempotent (count stays 1).

## 5. Notes
The hosted demo's API outage (HTTP 530 throughout the test window) is itself worth flagging to the demo's
maintainers. All other findings were reproduced against the project's own reference backend, so they reflect the
codebase, not the hosting.
