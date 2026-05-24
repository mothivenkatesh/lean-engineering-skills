---
name: spec
description: "Turn vague requirements into exact specifications before code. Runs after /challenge confirms something should be built. Produces SPEC.md consumed by /premortem, /db-review, /modular, and /tdd. Trigger on: 'what exactly are we building', 'spec this out', 'before I start coding', 'define the requirements'. Every ambiguity that surfaces here would have surfaced as a bug in production."
user-invocable: true
argument-hint: "[feature or system to specify]"
---

# Spec — Define It Before You Build It

## INPUT

Consumes: `DECISION.md` from `/challenge`

Uses: the actual problem statement, the recommendation, the constraints (reversibility, minimum version).

If DECISION.md doesn't exist: run `/challenge` first. Don't spec a solution before validating the problem.

---

## THE LOOP

**TRIGGER:** DECISION.md says "build it."

**CYCLE:**
1. Restate the problem in one sentence (no solution language)
2. Name every actor who interacts with this system
3. Write every behavior as a testable GIVEN/WHEN/THEN
4. Enumerate every edge case and failure mode
5. Define every interface exactly (types, not descriptions)
6. Define the data model (schema, constraints, invariants)
7. Decompose into ordered build steps
8. Read the spec back — find the gaps

**EXIT CONDITION:** Every question about what to build has a written answer. No behavior is ambiguous. SPEC.md is complete and reviewable.

---

## HARD CONSTRAINTS
- **Required:** DECISION.md from `/challenge` with recommendation "build it." If DECISION.md doesn't exist, run `/challenge` first.
- **Refuse if:** DECISION.md recommendation is "don't build" or "get clarity first." Don't spec a rejected proposal.
- **Refuse if:** Any behavior in Step 3 cannot be written as GIVEN/WHEN/THEN. Force specificity before proceeding.
- **Scope boundary:** Spec only what DECISION.md approved. No expansion during speccing.
- **Every edge case requires an exact expected outcome.** "Throw an error" is not a spec.

---

## STEP 1: ONE-SENTENCE PROBLEM

Write the problem in one sentence. No solution language.

Bad: "We need a payment retry system."
Good: "Failed payments drop silently. Merchants don't know until they check manually."

If you can't write one sentence, you have multiple problems. Split them.

---

## STEP 2: USERS

Name every actor. For each:
- **Who:** Role, not persona. (Admin, merchant, background job, API consumer)
- **Goal:** What outcome they need
- **Starting state:** What is true before they act
- **End state:** What is true after success

No assumed users. If you don't know who uses it, you don't know what it needs to do.

---

## STEP 3: BEHAVIORS

Every behavior is a testable fact.

```
GIVEN [precondition]
WHEN [action or event]
THEN [exact expected outcome]
```

Examples:
```
GIVEN a payment with status=failed
WHEN the merchant calls GET /payments/{id}
THEN the response includes status, failure_reason, and failed_at

GIVEN a payment retried 3 times already
WHEN a retry is attempted
THEN it is rejected with error RETRY_LIMIT_EXCEEDED
```

If a behavior can't be written as a test, it is not specific enough. Rewrite it.

---

## STEP 4: EDGE CASES

For every behavior, ask:
- Required input is missing?
- Input is wrong type?
- Input is at the boundary (0, -1, max, empty string)?
- Called twice (idempotency)?
- Called simultaneously by two users (race condition)?
- Partial failure mid-operation?

State the exact expected outcome for each. "Throw an error" is not a spec. "Return HTTP 422 with `{ error: 'RETRY_LIMIT_EXCEEDED' }`" is.

Edge cases are 80% of the real spec. The happy path is the easy part.

---

## STEP 5: INTERFACE

Define every public interface precisely.

For HTTP APIs:
```
METHOD /path
Auth: [required / none]

Request: { field: Type  // [required/optional] }
Response 200: { field: Type }
Response 4xx: { error: "ERROR_CODE", message: "..." }
```

For functions:
```typescript
function name(param: Type): ReturnType
// throws: ErrorType when [condition]
// idempotent: yes / no
// side effects: [list]
```

No "object" or "any." Every field has a type and a purpose.

---

## STEP 6: DATA MODEL

For each entity:
```
Table: [name]
Fields:
  - id: uuid, primary key
  - field: type  [nullable? indexed? unique?]
  - created_at: timestamptz
  - updated_at: timestamptz

Relationships: belongs_to / has_many
Constraints: [field invariants]
```

Money: always INTEGER (paise, cents). Never FLOAT.
Timestamps: always TIMESTAMPTZ. Never TIMESTAMP.
Enums: DB-level CHECK constraint. Not application strings.

---

## STEP 7: BUILD ORDER

List steps in dependency order. Step N cannot start until step N-1 is complete.

```
Step 1: [Foundation — what everything else depends on]
Step 2: [Next layer]
Step 3: [...]
```

Each step must be independently deployable. If step 3 requires step 2 in production first: say so.

---

## STEP 8: VALIDATE

Before marking done:
- [ ] Every behavior in Step 3 maps to the problem in DECISION.md
- [ ] Every edge case in Step 4 has an exact expected outcome
- [ ] Every field in Step 6 has a type and a reason
- [ ] Build order in Step 7 can be followed by someone who didn't write the spec
- [ ] No open question is left unanswered (assign to a person if unclear)

---

## OUTPUT: SPEC.md

```markdown
# SPEC.md

## Problem
[One sentence — no solution language]

## Out of Scope
[What this explicitly does NOT do]

## Users
| User | Goal | Starting State | End State |
|------|------|----------------|-----------|

## Behaviors
GIVEN / WHEN / THEN for each behavior

## Edge Cases
| Case | Expected Outcome |
|------|-----------------|

## Interface
[Exact API contract or function signatures]

## Data Model
[Tables, fields, types, constraints]

## Build Order
Step 1: [...]
Step 2: [...]

## Open Questions
[Anything unresolved — with owner assigned]

## Feeds into
/premortem (RISK.md), /db-review (SCHEMA.md), /modular (MODULE_MAP.md), /tdd (TEST_REPORT.md)
```

---

## FEEDS INTO

`/premortem` — stress-tests every behavior against failure modes.
`/db-review` — reviews data model and schema safety.
`/modular` — draws module boundaries from the interface definitions.
`/tdd` — uses GIVEN/WHEN/THEN behaviors as test specifications.
