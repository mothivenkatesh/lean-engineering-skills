---
name: tdd
description: "Test-driven development skill. Use when writing new features, fixing bugs, or building any non-trivial logic. Forces the red-green-refactor loop: failing test first, minimal code to pass, then refactor. Trigger on: 'write tests', 'TDD', 'add feature', 'fix bug', 'write a function that', 'implement X'. The goal is code that is provably correct, not code that seems to work."
user-invocable: true
argument-hint: "[feature or bug description]"
---

# TDD — Test-Driven Development

You are running the red-green-refactor loop. Code without a failing test first is a guess. Tests are not verification — they are specification. Writing tests after the fact proves the code exists; writing tests first proves the code is correct.

The discipline: **No code without a failing test. No refactor without green.**

---

## THE LOOP

```
1. RED    → Write the smallest failing test that describes the behavior
2. GREEN  → Write the minimum code to make it pass (nothing more)
3. REFACTOR → Clean up while tests stay green
4. REPEAT → Next behavior, next test
```

Never skip RED. Never go to REFACTOR while RED. Never write more code than GREEN demands.

---

## PHASE 1: UNDERSTAND BEFORE YOU WRITE

Before any test, answer these questions. Don't assume — ask if unclear.

1. **What is the behavior?** (Not "what is the code" — what should the system DO?)
2. **What are the inputs?** (Types, ranges, edge cases, invalid values)
3. **What are the expected outputs?** (Exact values, shapes, side effects)
4. **What are the failure modes?** (What should happen when input is wrong?)
5. **What already exists?** (Read existing tests and code before writing anything)

State your understanding back before writing a single line:
> "You want a function that takes X, returns Y, and throws Z when W. I'll start with the happy path."

---

## PHASE 2: RED — The Failing Test

### Rules for a good failing test

- **One behavior per test.** Not "creates a user and sends an email." Two behaviors = two tests.
- **Name describes the contract.** `should_return_null_when_user_not_found` > `test_user_lookup`.
- **Arrange / Act / Assert structure.** Set up → call → verify. Never mix these.
- **Test the interface, not the implementation.** If the test breaks when you rename a private method, the test is wrong.
- **Assert the exact output.** Not "truthy" — the actual value, type, shape.

### Test naming pattern

```
[unit]_[action]_[expected outcome]
login_with_invalid_password_returns_401
cart_total_with_empty_cart_returns_zero
parse_date_with_future_date_throws_validation_error
```

### Run the test. It must FAIL.

If it passes before you've written implementation code:
- The test is wrong (testing the wrong thing)
- Or the behavior already exists (which means you should stop and re-scope)

A test that can't fail can't protect you.

---

## PHASE 3: GREEN — Minimum Code to Pass

The rule: **Write the least code that makes the test pass.**

Not the cleanest. Not the most general. Not the version that handles all future cases. The minimum.

Why? Because if you over-build at GREEN, you've added code that has no test. That code is a liability.

Common mistakes at GREEN:
- Adding error handling that no test covers yet → DON'T. Add a test for the error case first.
- Generalizing for future inputs → DON'T. Write for the current test only.
- Refactoring while implementing → DON'T. That's what REFACTOR is for.

**Run all tests after every change.** Not just the new one. If any existing test breaks, stop immediately — you've broken a contract.

---

## PHASE 4: REFACTOR — Clean While Green

Only refactor when all tests are green. The tests are your safety net.

What to look for:
- Duplication between the new code and existing code
- Unclear variable names or function names
- Functions doing more than one thing
- Missing abstractions (three similar blocks of code = extract a function)
- Wrong abstraction level (business logic leaking into infrastructure layer)

Rules:
- Run the full test suite after every refactor step. Not at the end — after every change.
- If any test breaks during refactor, REVERT immediately. The refactor was wrong.
- Don't add new behavior during REFACTOR. Add a new test first.

---

## PHASE 5: NEXT BEHAVIOR

After RED → GREEN → REFACTOR, pick the next behavior. Work outward from the happy path:

```
Order:
1. Happy path (the common case)
2. Edge cases (empty, zero, max, boundary values)
3. Error cases (invalid input, missing data, permission denied)
4. Integration (how this connects to other components)
```

Don't jump to error cases before the happy path is solid. Don't write integration tests before unit tests pass.

---

## BUG FIX PROTOCOL

When fixing a bug, TDD means:

1. **Write a test that reproduces the bug.** The test fails. This proves the bug exists.
2. **Fix the bug.** The test passes.
3. **Verify no regression.** Full test suite is green.

Never fix a bug without a test. A bug without a test will come back.

---

## TEST DESIGN PATTERNS

### Parameterized tests for multiple inputs
When the same behavior should hold for multiple inputs, don't copy-paste tests:

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
Each test must be independent. If test B relies on test A having run first:
- Extract the shared setup into a fixture/beforeEach
- Or inline the setup in each test

Tests that depend on order are a time bomb.

### Mocking — use sparingly
Mock external dependencies (APIs, databases, file system) to isolate the unit under test. But:
- Don't mock your own code. If you're mocking internals, the design is wrong.
- Don't mock what you could use directly. Mocks that don't match reality are worse than no tests.
- Prefer real implementations for fast dependencies (in-memory DB, real objects).

---

## WHAT GOOD LOOKS LIKE

When you're done with a feature:
- Every behavior has a named test
- Every test is green
- No code exists without a corresponding test
- The tests read like documentation of the system's contract
- A new person can understand what the system does by reading the tests alone

---

## ANTI-PATTERNS

| Anti-Pattern | Why It Fails |
|---|---|
| Writing all tests after the code | You'll unconsciously write tests that fit the code, not tests that verify correctness |
| Testing implementation details | Refactors break tests; you fix tests instead of writing features |
| One massive test per function | When it fails, you can't tell which behavior broke |
| Mocking everything | Tests pass but the system is wrong; mocks drifted from reality |
| Skipping refactor phase | Code quality degrades; next test is harder to write |
| "I'll add tests later" | Later never comes. Debt compounds. |

---

## OUTPUT FORMAT

For each TDD cycle, present:

```
## Test: [test name]
[Test code]

Status: FAILING ✗ / PASSING ✓

## Implementation
[Minimum code to pass]

## Refactor (if applicable)
[What was cleaned up and why]

## Next behavior
[What we're testing next]
```

After all behaviors are implemented:

```
## Coverage Summary
- Behaviors tested: [list]
- Edge cases covered: [list]
- Error cases covered: [list]
- What's NOT covered and why: [list]
```
