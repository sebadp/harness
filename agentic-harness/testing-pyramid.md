# Testing Pyramid (Layer 4)

**Layer:** Mixed — CI-gated (L1–L3) · on-demand (L4) · deferred (L5–L7)
**Related ADRs:** [ADR-002](../adr/002-testing-pyramid.md) · [ADR-003](../adr/003-testsprite-e2e-regression.md)

## The seven layers

```
Layer 7 ── Mutation testing         [DEFERRED — unlock when coverage ≥ 80%]
Layer 6 ── Property-based testing   [DEFERRED — unlock after mutation shows gaps]
Layer 5 ── Contract testing         [DEFERRED — unlock when API has external consumers]
Layer 4 ── TestSprite E2E/API       [ACTIVE — on-demand]
Layer 3 ── pytest e2e (TestClient)  [ACTIVE — 106 tests, CI required]
Layer 2 ── pytest unit              [ACTIVE — 154 tests, CI required]
Layer 1 ── Types + style            [ACTIVE — pre-commit + agent Stop hook]
```

Each layer covers a risk the one below cannot. Layers 5–7 are explicitly deferred
with documented unlock criteria — not "never", but "when X happens".

## Layer 1 — Types + style

- **Tools:** `ruff` (lint + format), `pyright` (types, strict mode), habit-hooks (design)
- **When:** every commit (pre-commit) and every agent turn (Stop hook)
- **Covers:** type errors, imports, formatting, structural design smells (C901, PLR09xx)
- **Doesn't cover:** runtime behavior, business logic
- **Gate:** blocks commits; blocks agent turns via exit 2

## Layer 2 — pytest unit (154 tests, ~7s)

- **Covers:** pure logic — schemas, validators, prompt compiler, eval metrics, OTel baggage
- **Rule:** a test is "unit" if it does NOT use fixtures matching `*client`, `*session`, `*engine`
- **Doesn't cover:** integration with FastAPI, DB, network
- **Gate:** CI `unit` job, required on every PR
- **Auto-marking:** `tests/conftest.py` inspects fixture names and applies `pytest.mark.unit` or `pytest.mark.e2e` automatically

## Layer 3 — pytest e2e (106 tests, ~19s)

- **Covers:** full FastAPI endpoints, auth flows, workspace CRUD, prompt publish/rollback, Gmail sync
- **Runs against:** TestClient + SQLite in-memory
- **Known limitation:** SQLite differences from Postgres (RETURNING, CTEs, transaction isolation) not caught here
- **Gate:** CI `e2e` job, required on every PR

## Layer 4 — TestSprite E2E/API regression

**Product:** [testsprite.com](https://testsprite.com) — AI-generated E2E/API tests
**Details:** [ADR-003](../adr/003-testsprite-e2e-regression.md)

- **Covers:** real HTTP flows against the live server (localhost:8000); regressions from AI agent changes
- **Not covered by pytest:** validates against actual response contract (headers, status codes, optional fields)
- **When to add tests:** after an exec plan closes with new endpoints or UI flows
- **Not CI-gated (yet):** requires running server + credits + Groq key; used on-demand via `make check-testsprite` (future)

### First-run results (2026-08-27)

10 tests generated, 5/10 passed. The 5 failures were all infrastructure issues, not
bugs in the app:

| Test | Result | Reason |
|---|---|---|
| TC001 GET /health | ✓ PASS | — |
| TC002 POST /auth/signup | ✓ PASS | — |
| TC003 POST /auth/login (valid) | ✓ PASS | — |
| TC004 POST /auth/login (invalid) | ✓ PASS | — |
| TC005 GET /auth/me | ✓ PASS | — |
| TC006 POST /auth/rotate-key | ✗ FAIL | Rate limit — signup 5/min exhausted |
| TC007 POST /triage | ✗ FAIL | Rate limit — signup 5/min exhausted |
| TC008 POST /triage/stream | ✗ FAIL | `sseclient` module not in sandbox |
| TC009 POST /workspaces | ✗ FAIL | Rate limit |
| TC010 GET /workspaces/:id | ✗ FAIL | Rate limit |

### Known fixes needed

1. **Rate limiting:** tests create a fresh user per test — the 5/min signup cap
   exhausts fast. Fix: seed a test account in `.env.test` and reuse it across
   related test cases.
2. **SSE:** TC008 imports `sseclient` which isn't in the TestSprite sandbox. Fix:
   use `requests.get(stream=True)` and parse the `data:` lines manually.

### MCP tools

| Tool | When to call |
|---|---|
| `testsprite_check_account_info` | Before any large run — verify credits |
| `testsprite_generate_code_summary` | First time or after major refactor |
| `testsprite_generate_standardized_prd` | After code summary |
| `testsprite_generate_backend_test_plan` | After a new exec plan closes |
| `testsprite_generate_frontend_test_plan` | After new UI flows |
| `testsprite_generate_code_and_execute` | Server running + plan generated |
| `testsprite_open_test_result_dashboard` | Review failures, edit steps |

## Layer 5 — Contract testing (DEFERRED)

- **Would cover:** schema compatibility between frontend and backend
- **Deferred because:** monorepo with pyright + Zod already catches most drift
- **Unlock when:** the API has external consumers or splits into microservices
- **Candidate tool:** Pact (consumer-driven) or OpenAPI schema validation

## Layer 6 — Property-based testing (DEFERRED)

- **Would cover:** invariants — "confidence in [0,1]", "slug is alphanumeric",
  "compiled prompt includes all active categories"
- **Deferred because:** pyright + Pydantic already enforce most invariants
- **Unlock when:** mutation testing reveals gaps that generative testing could catch
- **Candidate tool:** `hypothesis` + `pytest-hypothesis`

## Layer 7 — Mutation testing (DEFERRED)

- **Would cover:** measure how good the existing tests are (would they catch a
  real bug?)
- **Deferred because:** requires running the test suite N times per mutant — expensive
- **Unlock when:** coverage gate is active and reaches ~80%
- **Candidate tool:** `mutmut` on `email_triage/services/` and `email_triage/auth/` first

## Coverage gate (PENDING — high priority)

- **Current state:** no `--cov-fail-under`; unknown percentage of code covered
- **Decision pending:** activate `pytest-cov` with 80% threshold before enabling
  mutation testing (Layer 7)
- **Not in this ADR:** requires its own exec plan
