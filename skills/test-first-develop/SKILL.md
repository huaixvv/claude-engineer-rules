---
name: test-first-develop
description: |
  Drives test-first development for any BACKEND code change — API
  endpoints, services, business logic, data access, message handlers,
  background jobs.

  Trigger on any intent to build, extend, or fix backend behavior, in
  English or Chinese, formal or casual. Examples:
  - "implement a signup endpoint" / "实现注册接口"
  - "add order refund feature" / "加一个订单退款功能"
  - "fix this 500 error" / "修一下这个 bug"
  - "把订单这块写一下" / "搞一下登录"

  Enforces: TDD (red phase first, must run tests for real), integration
  tests over mocks, contract snapshots on API boundaries, BDD scenarios
  for business rules, negative tests for security boundaries.

  Skip for: throwaway scripts, exploratory prototypes, pure UI/frontend,
  docs/config/rename refactors, or when user explicitly opts out.
---
# Test-First Development (Backend)

**Mission**: produce healthy AI-generated backend code by enforcing strict
test-first discipline.

---

## When to trigger

Match by **intent** (not strict keyword): any backend build / fix /
extend request triggers this workflow. When in doubt, trigger.

**Skip conditions** (do NOT trigger, or back off cleanly if discovered
mid-workflow — see Escape hatch below):

- Throwaway / one-off scripts (`scripts/*`, ad-hoc data fixes)
- Exploratory prototypes
- Pure UI / frontend-only tasks
- Documentation / config / pure rename refactor
- User explicitly says "skip tests" / "no tests" / "quick fix" / "先不写测试"

---

## Mandatory workflow (STRICT order — never skip a step)

### Step 1 — Establish the spec

Source: user-provided spec, OR draft a markdown spec yourself.

**Storage location**: save the spec as `docs/specs/{{feature-slug}}.spec.md`
(use a short, descriptive slug for `{{feature-slug}}`, e.g. `order-refund`,
`user-signup`). If the project has no `docs/` directory, create it. The
spec file is a persistent artifact — it stays in the repo alongside the code.

Spec format (BDD-style, mandatory):

```
# Feature: <name>

## Scenario: <happy path>
Given <preconditions>
When  <action>
Then  <expected outcome>

## Scenario: <edge case>
...
```

**Edge case discovery — must ask yourself before proceeding:**

- What inputs break it? (null / `''` / negative / huge / unicode / SQL-ish)
- What states forbid it? (already exists / soft-deleted / expired / locked)
- Who must NOT be allowed to call it? (unauthenticated / wrong role / wrong owner)
- What external failures must it survive? (DB timeout / 3rd-party 500 / network split)
- What concurrent calls cause trouble? (double-submit / race condition / idempotency)

**HANDOFF**: present the spec to user, await "OK" or revision before Step 2.

---

### Step 2 — Translate spec to tests (DO NOT WRITE IMPLEMENTATION)

Pick test types per the priority matrix below.

For each Scenario, write one `test()` / `it()` whose name **matches the
Scenario title verbatim**.

Test code itself must follow the no-fallback rules from `core_rules.md`
(Rule 4.1 / 4.2 / 4.3):

core_rules.md: @/Users/zhixuan/Desktop/PROJECTS/claude-engineer-rules/core_rules.md

Specifically:

- No `||` / `??` default fallback in test data
- No silent error swallow in setup / teardown
- No fake fields added to "make tests easier"

---

### Step 2.5 — Red phase (MANDATORY, never skip)

Run the test suite. Confirm:

- All new tests **fail** (red)
- Failure reason is "not implemented" / "expected X got undefined" — NOT a
  syntax error or setup crash

**Attach the actual command output** in this format:

```
$ <test command>
<paste real output here>
```

**If any new test is GREEN at this stage**: the test is broken (testing
nothing). Fix the test first — do NOT proceed.

**HANDOFF**: present to user the **full file paths** — both the test
files (e.g. `tests/order/refund.test.ts`) and the spec file
(e.g. `docs/specs/refund.spec.md`) — together with the red-phase command
output from this step, so the user can locate and review everything.
Await "OK" before Step 3.

---

### Step 3 — Implement

Write the **minimum** code to make tests pass.

Strictly follow project rules (already loaded in Step 2 via `core_rules.md`,
which transitively includes the API response envelope spec via Rule 12):

- Rule 4.1: no swallowing exceptions / no error-as-default
- Rule 4.2 / 4.3: no default value fallback anywhere
- Rule 12: API response envelope spec for any API endpoint output

---

### Step 4 — Green phase (MANDATORY)

Run the full test suite. **Attach actual output**.

Verify:

- All new tests pass
- No previously-green tests went red (regression check)
- Integration tests against real DB ran (if applicable)
- Linter + type checker clean

**Forbidden**: claiming "tests should pass" / "this should work" without
running them. Every green claim **requires actual run output**.

---

## Test Immutability

After tests are written and verified red in Step 2.5, you MUST wait for
explicit user approval before proceeding to implementation. Once approved,
tests become the binding contract — implementation conforms to tests,
never the reverse.

Forbidden:

- Skipping user review and going straight to implementation
- Changing assertions to match broken code
- Removing, `.skip`-ing, or commenting out hard-to-pass tests
- Renaming tests to disguise intent
- Wrapping test code in `try/catch` or `?.` to hide failures

If a test fails, the IMPLEMENTATION is wrong.

**Exception**: if the spec itself is wrong (business misunderstanding /
requirement misalignment), STOP, tell the user what's wrong, ask
permission to revise spec + tests, wait for explicit OK. Never update
tests on your own judgment.

---

## Database Strategy (Testcontainers)

New projects: any DB-touching test MUST use Testcontainers per the
disciplines below. Existing projects with their own established
DB-testing convention follow that (`core_rules.md` Rule 1), but these
disciplines still apply regardless of tool. Bypass only when the user
explicitly says "skip tests" / "先不写测试".

### Mandatory disciplines

1. **Engine + major version match production.** `postgres:15` prod →
   `postgres:15-alpine` tests. No sqlite / cross-engine substitute.

2. **Use the Testcontainers SDK.** Never manual `docker run` or
   standalone Docker Compose.

3. **Local: `.withReuse()` enabled. CI: no reuse** (automatic).

4. **DB connection ONLY via env var** (e.g. `process.env.DATABASE_URL`).
   No `||` / `??` / `DEFAULT` fallback (`core_rules.md` Rule 4.3).

5. **Auto-run migrations on test startup**, migration files committed.

6. **Test isolation = transaction rollback** (`beforeEach: BEGIN` /
   `afterEach: ROLLBACK`).

7. **Image tag: pin major version + prefer alpine.**
   ✅ `postgres:15-alpine`. ❌ `:latest`, ❌ no tag.

8. **`src/` is production-only.** Never imports from `tests/`, never
   references the testcontainers SDK — keeps prod artifact
   Docker-independent.

### No Docker on the machine

STOP and ask the user (install Docker / OrbStack / Colima, or skip
integration tests for this task). Silent fallback (sqlite, shared dev DB,
skipping) is forbidden.

---

## Backend test type priority

Apply **per file or feature** (a single PR may mix types):

| Code being tested                          | Required type                                                        | Why                                   |
| ------------------------------------------ | -------------------------------------------------------------------- | ------------------------------------- |
| API endpoint                               | **Integration** (real DB + HTTP) + **Contract snapshot** | catch schema / contract drift         |
| Service layer with DB                      | **Integration** (real DB)                                      | mocked DB lies                        |
| Pure function / algorithm                  | **Unit** + **Property-based** if non-trivial             | fast feedback, randomized inputs      |
| Business rule (multi-branch)               | **BDD-style integration** (Given-When-Then)                    | covers branches systematically        |
| Bug fix                                    | **Reproduction test first** + scan nearby code for siblings    | prevents regression + finds neighbors |
| Auth / validation / money / external state | **Negative tests mandatory**                                   | AI never thinks of these naturally    |

**Default bias**: prefer **integration over unit-with-mocks**. Only mock
the truly external (3rd-party APIs, payment gateways, email senders).

---

## Contract test (API boundary)

For every API endpoint touched, snapshot the contract shape. Conceptual
example (use your project's actual schema extraction tool — `extractApiSchema`
below is illustrative, not a real API):

```
// PSEUDO-CODE — illustration only
expect(extractApiSchema(app, '/users')).toMatchSnapshot()
```

A passing snapshot on a modified endpoint means **field names, types,
status codes, and response envelope did NOT change**. If the change was
intentional, update the snapshot in a **separate commit with explicit
review**.

Cross-reference: API response envelope spec is loaded via core_rules.md Rule 12.

---

## Negative / security test checklist

For any code touching auth, input, money, or external state, the test
suite MUST include:

- [ ] Unauthorized caller → 401
- [ ] Authenticated but wrong role → 403
- [ ] Invalid input (missing required / wrong type / malicious payload) → 422
- [ ] Resource not found → 404
- [ ] Business rule violation (insufficient balance / conflicting state) → 409 / 422
- [ ] Idempotency / double-submit handled
- [ ] Boundary numbers (0, negative, INT_MAX, decimal precision)

---

## Forbidden patterns (Hard Stops)

| Pattern                                                     | Why forbidden                            |
| ----------------------------------------------------------- | ---------------------------------------- |
| Writing implementation before tests                         | confirmation bias → tests sanctify bugs |
| Claiming "tests pass" without running                       | #1 AI deception                          |
| Mocking the database in service / integration tests         | mocked-green ≠ real-green               |
| Mocking the very thing under test                           | tests verify the mock, not the code      |
| Happy-path-only tests                                       | misses real-world failures               |
| Hardcoded conditional to pass tests (`if a===1 return 3`) | AI cheating                              |
| Empty assertions (`expect(fn()).not.toThrow()`)           | tests nothing                            |
| Massive auto-generated snapshots no one reviews             | rubber stamp                             |
| Async without await / race-prone tests                      | flaky → ignored                         |
| Test data missing edge values (only `"abc"`, `1`)       | misses real bugs                         |
| Skipping negative tests for security boundaries             | vulnerabilities                          |
| Test data with hidden default value fallbacks               | violates 4.x in test layer too           |

---

## Handoff protocol (mandatory pause points)

| After step | Action | Required output to user                                                    |
| ---------- | ------ | -------------------------------------------------------------------------- |
| Step 1     | Pause  | "Spec ready — confirm before tests?"                                      |
| Step 2.5   | Pause  | test code + file paths + red-phase output — "All N tests red — proceed?" |
| Step 4     | Final  | Actual test output + green/red count + regression status                   |

**Never** chain Step 1 → Step 4 without the two explicit pauses.

---

## Escape hatch

If any **Skip condition** above becomes true mid-workflow (e.g. user says
"just write it, I'll test later"), back off immediately and acknowledge:

> "Test-first workflow skipped per your instruction. Proceeding without tests."
