---
name: diagnose
description: "Systematic debugging and root cause analysis. Use when something is broken, an error is thrown, behavior is unexpected, or a test fails. Trigger on: 'this is broken', 'why is X happening', 'getting an error', 'it's not working', 'this fails', 'unexpected behavior', 'debug this'. The instinct to immediately fix is wrong — diagnose first. A fix without a root cause is a guess that creates new bugs."
user-invocable: true
argument-hint: "[error message or description of broken behavior]"
---

# Diagnose — Systematic Root Cause Analysis

## INPUT

Consumes: An incident or bug report. Optionally: `DEPLOY.md` from `/ops-ready` (the runbook for the system that's failing).

---

## THE LOOP

**TRIGGER:** Something is broken. An alert fired, a test failed, a user reported unexpected behavior.

**CYCLE (iterates until root cause is found):**
1. REPRODUCE — Confirm the bug is real and repeatable
2. ISOLATE — Find the smallest code path that shows it
3. HYPOTHESIZE — Form a testable explanation
4. TEST — Confirm or falsify the hypothesis
5. If falsified: update hypothesis and return to step 3
6. ROOT CAUSE — State the actual cause in one sentence
7. FIX — Apply the minimal correct fix
8. VERIFY — Confirm the fix works and nothing else broke

**EXIT CONDITION:** Root cause is stated in one sentence. Fix is applied. Regression test is written. RCA.md is complete.

**The rule:** Never jump from REPRODUCE to FIX. The gap between them is everything.

Fixing without understanding the root cause creates one of three outcomes: the bug comes back, a new bug appears, or both.

---

## PHASE 1: REPRODUCE

A bug you can't reproduce consistently is a bug you can't fix reliably.

Answer:
- What exact inputs trigger this?
- What environment? (OS, runtime version, dependencies, env vars)
- Consistent or intermittent? (Intermittent = timing, concurrency, or state)
- What changed recently? (Last deployment, config change, dependency update)

Read the full error. Don't skim. Stack traces tell you exactly where the system failed. Read bottom-up: root cause is usually at the bottom, symptom at the top.

Capture the exact state:
```
Error message: [exact text]
Stack trace: [full trace]
Environment: [runtime, version, OS]
Input that triggers it: [exact value]
Expected: [what should happen]
Actual: [what does happen]
```

If you can't reproduce it: say so. Don't guess at unfalsifiable bugs.

---

## PHASE 2: ISOLATE

Binary search the problem space. Don't read the whole codebase — find the smallest reproduction.

1. **Follow the stack trace.** Start where the error is thrown, not where it's caught.
2. **Bisect with log points.** Place output at the midpoint of the suspected path. Does it fire? You've halved the search space.
3. **Remove components.** What's the smallest reproduction? Remove everything unrelated until the bug still appears.
4. **State your assumptions explicitly.** Then verify them. Don't assume.

What to look for:

| Pattern | What it signals |
|---|---|
| Silent error swallowing (`catch (e) {}`) | The real error is hidden |
| Shared mutable state | Race conditions, non-deterministic behavior |
| Missing null check | `cannot read property X of null` |
| Off-by-one in loop | Boundary condition failure |
| Async without await | Data arrives after it's consumed |
| Cached stale data | System is correct for data it saw last time |
| Environment-specific config | Works locally, fails in prod |
| Version mismatch | Library A expects Library B at version X |

---

## PHASE 3: HYPOTHESIZE

Before touching anything: state your hypothesis.

**Form:** "I believe the bug is caused by [X] because [evidence Y]. If true, [test Z] should confirm or deny it."

Good:
> "The cache TTL is 0 in the test environment, causing every read to hit the database. The database connection pool exhausts under concurrent load. Evidence: error only appears when 5+ tests run in parallel. Test: set pool size to 20 and rerun."

Bad:
> "Something's wrong with the database."

A hypothesis that can't be falsified is not a hypothesis. Be specific enough that you'd know if you were wrong.

---

## PHASE 4: TEST

Test one hypothesis at a time. Testing two simultaneously means you don't know which one mattered.

If the hypothesis is wrong: update it with new evidence. Record what you've ruled out. "Confirmed it's not the network layer because X" is progress.

If the bug disappears when you add logging: you're looking at a Heisenbug. Look for race conditions, async issues, or performance-sensitive code paths.

---

## PHASE 5: ROOT CAUSE

State the root cause in one clear sentence before writing any fix.

The root cause is not the symptom:

| Symptom | Root Cause |
|---|---|
| "API returns 500" | "users table has no index on email; query times out under load" |
| "Test flakes" | "Test uses Date.now() without mocking time; fails when CI is slow" |
| "Users logged out randomly" | "Session tokens stored in-memory; pods restart on deploy" |
| "File upload fails for some users" | "nginx has 2MB body limit; app limit is 5MB" |

If you can't state the root cause cleanly: you haven't found it yet. Keep going.

---

## PHASE 6: FIX

Once you have the root cause, the fix is usually obvious. Apply the minimum correct fix:

- Fix the root cause, not the symptom
- Don't add defensive code around the symptom if the root cause is fixable
- Don't add workarounds that hide the problem from future engineers
- Don't refactor while fixing — that's a separate task

If the fix is risky: state the tradeoffs. Present 2-3 options. Recommend one with reasoning.

---

## PHASE 7: VERIFY

After the fix:
1. Reproduce the original bug — it should not happen.
2. Run the full test suite — nothing else should break.
3. Check for related bugs — if the root cause affected one place, did it affect others?
4. Write a regression test — a test that fails without your fix. Bugs without tests come back.

---

## COMMON BUG CLASSES

### Performance bugs

Symptom: slow. Root cause: almost never the database. Check in order:
1. N+1 queries (fetching inside a loop?)
2. Missing index (EXPLAIN ANALYZE on the slow query)
3. Unbounded fetch (loading 100K rows to return 10?)
4. No caching on hot read path
5. Then consider the database

### Intermittent bugs

Cause: timing, state, or concurrency. Look for:
- Shared mutable state between tests or requests
- `setTimeout`/`setInterval` without cleanup
- Race conditions between async operations
- Tests that depend on execution order

### "Works on my machine"

Cause: environment divergence. Check:
- Runtime version (Node, Python)
- Environment variables and their defaults
- File path separators (Windows vs Unix)
- Timezone assumptions
- Docker/OS differences in case sensitivity

### Data bugs

Symptom: wrong output. Root cause: almost never the business logic. Check:
- What data actually entered the system (log the raw input)
- Where the data transformation happens (log intermediate state)
- What constraints are assumed but not enforced (nulls, empty strings, encoding)

---

## OUTPUT: RCA.md

```markdown
# RCA.md

## Incident
[Date, what failed, who was affected]

## Reproduction
[Exact inputs, steps, environment to trigger the bug]

## Isolation
[Where in the code the failure originates — file:line if known]

## Root Cause
[One clear sentence — the actual cause, not the symptom]

## Evidence
[What confirms this is the root cause, not a guess]

## Fix
[The minimal code change that resolves the root cause]

## What Was Ruled Out
[Alternative hypotheses that were tested and eliminated]

## Regression Test
[Test that proves the bug is fixed and will catch recurrence]

## Prevention
[What would have caught this earlier — missing test, missing alert, missing constraint]
```

---

## FEEDS INTO

`/debt-audit` — patterns of bugs often reveal structural debt. After an incident, run `/debt-audit` on the affected module.
