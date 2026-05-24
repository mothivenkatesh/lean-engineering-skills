---
name: tdd
description: "Test-driven development. Use when writing new features, fixing bugs, or building any non-trivial logic. Forces the red-green-refactor loop: failing test first, minimal code to pass, then refactor. Trigger on: 'write tests', 'TDD', 'add feature', 'fix bug', 'write a function that', 'implement X'. The goal is code that is provably correct, not code that seems to work."
user-invocable: true
argument-hint: "[feature or bug description]"
---

# TDD — Test-Driven Development

## INPUT

Consumes: `SPEC.md` from `/spec` and `MODULE_MAP.md` from `/modular`

Uses: GIVEN/WHEN/THEN behaviors from SPEC.md as test specifications. Module interfaces from MODULE_MAP.md to define mock boundaries.

---

## THE LOOP

**TRIGGER:** Behavior is defined in SPEC.md. Module boundaries are drawn in MODULE_MAP.md.

**CYCLE (repeats for each behavior):**
1. RED — Write the smallest failing test that describes one behavior
2. GREEN — Write the minimum code to make it pass (nothing more)
3. REFACTOR — Clean up while tests stay green
4. REPEAT — Pick the next behavior from SPEC.md

**EXIT CONDITION:** Every GIVEN/WHEN/THEN from SPEC.md has a passing test. Every module passes in isolation with mocked dependencies. TEST_REPORT.md is written.

```
RED → GREEN → REFACTOR → REPEAT
```

Never skip RED. Never go to REFACTOR while RED. Never write more code than GREEN demands.

---

## HARD CONSTRAINTS
- **Required:** At least one behavior written as GIVEN/WHEN/THEN, or a bug with exact reproduction steps. "Write tests for the user module" with no specific behavior is not enough — ask for the behavior.
- **Refuse if:** User wants to write tests after code is already written. That is verification, not TDD. Acknowledge the code exists, but write tests against the interface, not the implementation.
- **The RED phase is non-negotiable.** If the test passes before any implementation is written: stop. The test is wrong or the behavior already exists. Do not proceed until the test fails first.
- **One behavior per test.** Never combine behaviors into one test. If a test name requires "and," split it.
- **Bug fix protocol is mandatory:** Write a reproducing test before touching the fix. No exceptions.

---

## PHASE 1: UNDERSTAND BEFORE YOU WRITE

Before any test, answer:
1. What is the behavior? (What should the system DO, not how)
2. What are the inputs? (Types, ranges, edge cases)
3. What are the expected outputs? (Exact values, shapes, side effects)
4. What are the failure modes? (What happens when input is wrong)
5. What already exists? (Read existing tests and code first)

State your understanding:
> "You want a function that takes X, returns Y, and throws Z when W. Starting with the happy path."

---

## PHASE 2: RED — The Failing Test

### Rules for a good test
- **One behavior per test.** "Creates a user and sends an email" = two tests.
- **Name describes the contract.** `should_return_null_when_user_not_found` > `test_user_lookup`
- **Arrange / Act / Assert.** Set up → call → verify. Never mix these.
- **Test the interface, not the implementation.** If it breaks when you rename a private method, the test is wrong.
- **Assert the exact output.** Not "truthy" — the actual value, type, shape.

### Naming pattern
```
[unit]_[action]_[expected outcome]
login_with_invalid_password_returns_401
cart_total_with_empty_cart_returns_zero
parse_date_with_future_date_throws_error
```

### The test must FAIL before you write implementation

If it passes before implementation code exists:
- The test is wrong (testing the wrong thing), or
- The behavior already exists (stop and re-scope)

A test that can't fail can't protect you.

---

## PHASE 3: GREEN — Minimum Code to Pass

Write the least code that makes the test pass.

Not the cleanest. Not the most general. Not the future-proof version. The minimum.

Why: if you over-build at GREEN, you've added code with no test. That code is a liability.

Mistakes at GREEN:
- Adding error handling that no test covers yet → don't. Write the error test first.
- Generalizing for future inputs → don't. Write for the current test only.
- Refactoring while implementing → don't. That's what REFACTOR is for.

Run all tests after every change. Not just the new one. If any existing test breaks: stop. You've broken a contract.

---

## PHASE 4: REFACTOR — Clean While Green

Only refactor when all tests are green. The tests are your safety net.

What to look for:
- Duplication between new code and existing code
- Unclear variable names or function names
- Functions doing more than one thing
- Three similar blocks of code (extract a function)
- Business logic leaking into infrastructure layer

Rules:
- Run full test suite after every refactor step. Not at the end — after every change.
- If any test breaks during refactor: revert immediately. The refactor was wrong.
- Don't add new behavior during REFACTOR. Write a new test first.

---

## PHASE 5: BEHAVIOR ORDER

Work outward from the happy path:

```
1. Happy path (the common case works correctly)
2. Edge cases (empty, zero, max, boundary values)
3. Error cases (invalid input, missing data, permission denied)
4. Integration (how this module connects to others)
```

Don't jump to error cases before the happy path is solid. Don't write integration tests before unit tests pass.

---

## BUG FIX PROTOCOL

When fixing a bug:
1. Write a test that reproduces the bug. The test fails. This proves the bug exists.
2. Fix the bug. The test passes.
3. Full test suite is green.

Never fix a bug without a test. A bug without a test comes back.

---

## TEST DESIGN PATTERNS

### Parameterized tests
```python
@pytest.mark.parametrize("input,expected", [
    (0, "zero"),
    (1, "positive"),
    (-1, "negative"),
])
def test_classify_number(input, expected):
    assert classify(input) == expected
```

### Test isolation

Each test must be independent. If test B relies on test A having run first: extract shared setup into a fixture.

Tests that depend on order are a time bomb.

### Mocking — use sparingly

Mock external dependencies (APIs, databases, file system). Not your own code.

- Don't mock internals. If you're mocking internals, the design is wrong.
- Prefer real implementations for fast dependencies (in-memory DB, real objects).
- Mocks that don't match reality are worse than no tests.

From MODULE_MAP.md: module interfaces define exactly what to mock at each boundary.

---

## WHAT GOOD LOOKS LIKE

When done with a feature:
- Every behavior from SPEC.md has a named test
- Every test is green
- No code exists without a corresponding test
- Tests read like documentation of the system's contract
- A new person can understand what the system does by reading the tests alone

---

## ANTI-PATTERNS

| Anti-Pattern | Why It Fails |
|---|---|
| Writing all tests after the code | You write tests that fit the code, not tests that verify correctness |
| Testing implementation details | Refactors break tests; you fix tests instead of writing features |
| One massive test per function | When it fails, you can't tell which behavior broke |
| Mocking everything | Tests pass but the system is wrong; mocks drifted from reality |
| Skipping REFACTOR | Code quality degrades; next test is harder to write |
| "I'll add tests later" | Later never comes. Debt compounds. |

---

## OUTPUT: TEST_REPORT.md

After all behaviors are implemented:

```markdown
# TEST_REPORT.md

## Module: [name]

## Coverage by Behavior
| Behavior (from SPEC.md) | Test | Status |
|-------------------------|------|--------|
| GIVEN / WHEN / THEN | test_name | PASSING |

## Edge Cases Covered
[List from SPEC.md edge cases — all should have tests]

## Error Cases Covered
[List]

## Not Covered (and why)
[What was intentionally deferred]

## How to run
[exact command]

## Feeds into
/hot-path (PERF.md) — tests confirm behavior; hot-path reviews performance
```

---

## FEEDS INTO

`/hot-path` — once all behaviors are tested, review the hot path for performance. Tests prove correctness; hot-path review proves speed.
