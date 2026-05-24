---
name: diagnose
description: "Systematic debugging and root cause analysis. Use when something is broken, an error is thrown, behavior is unexpected, or a test fails unexpectedly. Trigger on: 'this is broken', 'why is X happening', 'getting an error', 'it's not working', 'this fails', 'unexpected behavior', 'debug this'. The instinct to immediately fix is wrong — diagnose first. A fix without a root cause is a guess that creates new bugs."
user-invocable: true
argument-hint: "[error message or description of broken behavior]"
---

# Diagnose — Systematic Root Cause Analysis

The first instinct when something breaks is to fix it. That instinct is wrong.

Fixing without understanding the root cause creates one of three outcomes: the bug comes back, a new bug appears, or both. The discipline is: **understand the system before touching it**.

Kailash Nadh's rule: "My database is slow" is almost always bad indexing, bad queries, or bad schema. The symptom is never the cause. Diagnose the cause.

---

## THE DIAGNOSTIC PROTOCOL

```
1. REPRODUCE   → Confirm the bug exists and is repeatable
2. ISOLATE     → Narrow down where in the system it lives
3. HYPOTHESIZE → Form a testable explanation
4. TEST        → Verify or falsify the hypothesis
5. ROOT CAUSE  → State the actual cause (not the symptom)
6. FIX         → Apply the minimal correct fix
7. VERIFY      → Confirm the fix works and nothing regressed
```

Never jump from REPRODUCE to FIX. The gap between them is everything.

---

## PHASE 1: REPRODUCE

A bug you can't reproduce consistently is a bug you can't fix reliably.

**Reproduce with precision. Answer:**
- What exact inputs trigger this? (Not "large data" — exact size, format, value)
- What environment? (OS, runtime version, dependencies, env vars)
- Is it consistent or intermittent? (Intermittent = timing, concurrency, or state)
- What changed recently? (Last deployment, config change, dependency update)

**Read the full error.** Don't skim. The stack trace tells you exactly where the system failed. Read it bottom-up: the root cause is usually at the bottom, the symptom at the top.

**Capture the exact state:**
```
- Error message: [exact text]
- Stack trace: [full trace]
- Environment: [runtime, version, OS]
- Input that triggers it: [exact value]
- Expected behavior: [what should happen]
- Actual behavior: [what does happen]
```

If you can't reproduce it, say so. Don't guess at unfalsifiable bugs.

---

## PHASE 2: ISOLATE

Binary search the problem space. Don't read the whole codebase — find the smallest possible code that reproduces the issue.

### The isolation process

1. **Follow the stack trace.** Start where the error is thrown, not where it's caught.
2. **Bisect by adding log points.** Place `console.log` / `print` at the midpoint of the suspected path. Does the log fire? You've halved the search space.
3. **Remove components.** What's the smallest reproduction? Remove everything unrelated until the bug still appears. What remains is the problem.
4. **Check your assumptions.** What are you assuming about the system that might be wrong? State assumptions explicitly, then verify them.

### What to look for in code

Read the code, don't skim it. Specifically look for:

| Pattern | What it signals |
|---|---|
| Silent error swallowing (`catch (e) {}`) | The real error is being hidden |
| Shared mutable state | Non-deterministic behavior; race conditions |
| Missing null/undefined check | `cannot read property X of null` class bugs |
| Off-by-one in loops | Boundary condition failures |
| Async without await | Data arrives after it's consumed |
| Cached stale data | System is behaving correctly for data it saw last time |
| Environment-specific config | Works locally, fails in prod |
| Version mismatch | Library A expects library B at version X |

---

## PHASE 3: HYPOTHESIZE

Before touching anything, state your hypothesis.

**Form:** "I believe the bug is caused by [X] because [evidence Y]. If true, [test Z] should confirm or deny it."

Good hypothesis:
> "The cache TTL is set to 0 in the test environment, causing every read to hit the database. The database connection pool is exhausted under concurrent test load, causing the timeout. Evidence: the error only appears when more than 5 tests run in parallel. Test: set pool size to 20 and rerun."

Bad hypothesis:
> "Something's wrong with the database."

A hypothesis that can't be falsified is not a hypothesis — it's a shrug. Be specific enough that you'd know if you were wrong.

---

## PHASE 4: TEST

Run the test that confirms or falsifies your hypothesis. Add log output, assertions, or a minimal reproduction script.

**Rules:**
- Test one hypothesis at a time. Testing two simultaneously means you don't know which one mattered.
- If the hypothesis is wrong, update it with new evidence. Don't chase the fix — update your understanding.
- Record what you've ruled out. "I confirmed it's not the network layer because X" is progress.

**If the bug disappears when you add logging:** You're looking at a timing or Heisenbug (observer effect). Look for race conditions, async issues, or performance-sensitive code paths.

---

## PHASE 5: ROOT CAUSE

State the root cause in one clear sentence before writing any fix.

The root cause is **not** the symptom. Examples:

| Symptom | Root Cause |
|---|---|
| "The API returns 500" | "The database query times out because the users table has no index on email" |
| "The test flakes" | "The test uses `Date.now()` without mocking time, so assertions fail when CI is slow" |
| "Users are logged out randomly" | "Session tokens are stored in-memory; pods restart on deploy and memory is cleared" |
| "File upload fails for some users" | "The file size limit is 5MB but the nginx proxy has a 2MB request body limit" |

If you can't state the root cause cleanly, you haven't found it yet.

---

## PHASE 6: FIX — THE MINIMAL CORRECT FIX

Once you have the root cause, the fix is usually obvious. Apply the minimum correct fix:

- Fix the root cause, not the symptom
- Don't add defensive code around the symptom if the root cause is fixable
- Don't add workarounds that hide the problem from future engineers
- Don't refactor while fixing — that's a separate task

**If the fix is non-obvious or risky:**
1. State the tradeoffs explicitly
2. Present 2-3 options with their implications
3. Recommend one, with reasoning

---

## PHASE 7: VERIFY

After the fix:

1. **Reproduce the original bug.** Does it still happen? (It shouldn't.)
2. **Run the full test suite.** Did the fix break anything else?
3. **Check for related bugs.** If the root cause affected one place, did it affect other places too?
4. **Write a regression test.** Add a test that fails without your fix. Bugs without regression tests come back.

---

## COMMON BUG CLASSES

### Performance bugs
Symptom: slow. Root cause: almost never the database. Check in order:
1. N+1 queries (are you fetching inside a loop?)
2. Missing index (run `EXPLAIN ANALYZE` on the slow query)
3. Unbounded data fetch (are you loading 100K rows to return 10?)
4. No caching on a hot read path
5. Then consider the database

### Intermittent bugs
Cause: timing, state, or concurrency. Look for:
- Shared mutable state between tests or requests
- `setTimeout`/`setInterval` without proper cleanup
- Race conditions between async operations
- Tests that depend on execution order

### "Works on my machine" bugs
Cause: environment divergence. Check:
- Node/Python/runtime version
- Environment variables and their defaults
- File path separators (Windows vs Unix)
- Timezone assumptions
- Docker/OS differences in case sensitivity

### Data bugs
Symptom: wrong output. Root cause: almost never the business logic. Check:
- What data actually entered the system (log the raw input)
- Where the data transformation happens (log the intermediate state)
- What constraints are assumed but not enforced (nulls, empty strings, encoding)

---

## OUTPUT FORMAT

Present your diagnosis as:

```
## Reproduction
[How to trigger the bug. Exact inputs, steps, environment.]

## Isolation
[Where in the code the failure originates. File:line if known.]

## Root Cause
[One clear sentence: the actual cause, not the symptom.]

## Evidence
[What confirms this is the root cause, not a guess.]

## Fix
[The minimal code change that resolves the root cause.]

## What Was Ruled Out
[Alternative hypotheses that were tested and eliminated.]

## Regression Test
[The test that proves the bug is fixed and will detect recurrence.]
```

---

## ANTI-PATTERNS

| Anti-Pattern | Why It Fails |
|---|---|
| Fixing the symptom | Bug returns; new bugs introduced |
| Changing multiple things at once | You don't know what fixed it |
| "Let me just try X" | Intuition-driven debugging is random walk through solution space |
| Adding logging after the fact | Log what matters before you know what matters |
| Assuming the bug is where it appears | Stack traces show where errors are caught, not where they're caused |
| Not writing a regression test | The bug will return — guaranteed |
