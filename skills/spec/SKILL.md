---
name: spec
description: "Problem specification and requirement decomposition before writing code. Use before building any feature, endpoint, component, or system. Trigger on: 'I want to build X', 'add a feature', 'help me plan this', 'how should I approach X', 'I need to implement Y', 'let's build Z'. Code written without a spec is a bet that your first interpretation was correct. That bet loses most of the time. Spec first, code second."
user-invocable: true
argument-hint: "[feature or system to specify]"
---

# Spec — Problem Specification Before Code

Kailash Nadh: "Code is cheap. Show me the talk." In the LLM era, generating code takes seconds. The value is in thinking clearly about what to build before building it.

The discipline: **The spec is not a formality before the real work. The spec IS the real work.**

A spec that takes 30 minutes to write prevents a 3-day refactor. Every ambiguity that surfaces during speccing would have surfaced as a bug during production.

---

## THE SPEC PROCESS

```
1. PROBLEM    → Define what problem you're actually solving
2. USERS      → Who uses this, and what do they need?
3. BEHAVIOR   → Exact inputs, outputs, state changes
4. EDGE CASES → What happens at the boundaries and when things go wrong
5. INTERFACE  → How does this connect to the rest of the system?
6. DATA       → What data structures are needed?
7. DECOMPOSE  → Break into buildable blocks in dependency order
8. VALIDATE   → Read the spec back and find the holes
```

---

## PHASE 1: PROBLEM DEFINITION

Before anything else — what problem is being solved?

The most common engineering failure is building the right solution to the wrong problem. State the problem in one sentence. If you can't, you don't understand it yet.

**The framing questions:**
- What user pain or business need triggers this?
- What currently happens? What should happen instead?
- How will you know this is solved? (What's the measurable outcome?)
- What are you NOT solving? (Explicit scope exclusion prevents scope creep)

**Red flags in problem statements:**
- "We need a feature that..." (describes a solution, not a problem)
- "Users should be able to..." (starts with solution — back up to the problem)
- "Make it work like X" (copies a solution without validating the problem)

**Valid problem statement:**
> "Merchants are calling support to find out why payments failed. We don't give them a reason code. We need them to self-serve this information."

**Invalid problem statement:**
> "Build a payment failure reason code page."

---

## PHASE 2: USERS AND THEIR GOALS

Every behavior in the spec serves a user goal. Unnamed users lead to features nobody uses.

For each user type, specify:
- **Who they are** (concrete, not abstract: "a Razorpay merchant with 500+ orders/month" not "a user")
- **What they're trying to accomplish** (the goal, not the feature)
- **What they know** (technical sophistication, existing mental models)
- **What they can't do today** (the gap this feature closes)

If there are multiple user types, rank them. Build for the primary user first. Secondary users get accommodated, not designed for.

---

## PHASE 3: EXACT BEHAVIOR

This is the core of the spec. For every action in the system, define:

### Input → Output mapping

For each operation:
```
Operation: [name]
Input:     [exact types, formats, valid ranges]
Output:    [exact shape, type, content]
Side effects: [what else changes — DB writes, events emitted, emails sent]
```

**Precision matters.** "Returns user data" is not a spec. "Returns `{ id: UUID, email: string, created_at: ISO8601, role: 'admin'|'user' }` with HTTP 200" is a spec.

### State machine (if applicable)

If the feature involves state transitions, draw the state machine:
```
States: [list all valid states]
Transitions: [state A --event--> state B]
Invalid transitions: [state A cannot directly become state C]
```

### Timing and latency expectations

- Is this synchronous or asynchronous?
- What's the acceptable latency? (p50, p99)
- What happens if it takes too long? (timeout? retry? queue?)

---

## PHASE 4: EDGE CASES

Edge cases are not special — they are the full specification. The happy path is the easy part.

Work through every boundary:

| Category | Questions |
|---|---|
| Empty / Zero | What if the list is empty? The number is 0? The string is blank? |
| Null / Missing | What if a required field is absent? Optional field is null? |
| Max / Large | What if there are 1M records? A 50MB file? A 10,000-character string? |
| Duplicate | What if this is submitted twice? (idempotency) |
| Concurrent | What if two users do this simultaneously? |
| Partial failure | What if step 3 of 5 fails? Is the system in a consistent state? |
| Permission | What if an unauthorized user tries this? |
| Stale data | What if the data changed between read and write? |
| Bad input | What if the input is malformed, unexpected type, injection attempt? |

For each edge case: state the expected behavior explicitly. "Throw an error" is not a spec. "Return HTTP 422 with `{ error: 'email_already_registered', message: 'An account with this email exists' }`" is a spec.

---

## PHASE 5: SYSTEM INTERFACE

How does this connect to everything else?

- **API contract:** endpoints, request/response shapes, HTTP status codes, auth requirements
- **Database changes:** new tables, columns, indexes, migrations
- **Events:** what events are emitted, what events are consumed
- **External services:** which third-party APIs are called, what happens when they fail
- **Configuration:** what can be configured, where (env vars, DB, feature flags)

**Sriram Velamur's principle:** Platforms are grown, not built. Every interface you define today is a contract you'll maintain for years. Design interfaces conservatively — it's easy to add, hard to remove.

---

## PHASE 6: DATA STRUCTURES

Define the data before the code. Bad schemas are the most expensive bugs — they require migrations, data backfills, and coordinated deploys.

For each new entity:
```
Table/Model: [name]
Fields:
  - [field]: [type] [nullable?] [default?] [constraint?]
  - ...
Indexes: [what to index and why]
Relationships: [foreign keys, ownership]
Invariants: [what must always be true about this data]
```

**The canonical rule from Zerodha:** If you're unsure what your schema should look like, start simpler. You can always add columns. You can't easily remove ones in production.

---

## PHASE 7: DECOMPOSE INTO BLOCKS

Break the spec into independently buildable blocks in dependency order. No block should depend on a block that hasn't been built yet.

```
Block 1: [name] — [what it delivers]
  Depends on: nothing
  Output: [what's verifiable when done]

Block 2: [name] — [what it delivers]
  Depends on: Block 1
  Output: [what's verifiable when done]

Block 3: [name] — [what it delivers]
  Depends on: Blocks 1, 2
  Output: [what's verifiable when done]
```

**Ordering rules:**
1. Data layer before logic layer before presentation layer
2. Core happy path before edge cases
3. Internal components before external interfaces
4. Reading before writing (prove you can read before you prove you can write)

---

## PHASE 8: VALIDATE THE SPEC

Read the spec back with fresh eyes. Look for:

- **Ambiguity:** Any sentence that could be interpreted two ways will be interpreted two ways.
- **Missing states:** Have you covered every input → output combination?
- **Unstated assumptions:** What are you assuming about the system that might be wrong?
- **Dependencies:** Have you identified everything this needs from the existing system?
- **Reversibility:** What happens if you need to roll this back? Is it possible?
- **Test coverage:** Can you write a test for every behavior you've specified?

If any spec item can't be turned into a test, the item is too vague.

---

## SPEC TEMPLATE

```markdown
# Feature: [Name]

## Problem
[One sentence: what user pain or business need does this address?]

## Out of scope
[What this explicitly does NOT do]

## Users
- **Primary:** [who, what they're trying to do]
- **Secondary:** [who, what they're trying to do]

## Behavior

### [Operation name]
- Input: [exact types and valid ranges]
- Output: [exact shape and types]
- Side effects: [DB, events, emails, etc.]
- Errors: [when it fails and how]

### [Operation name]
...

## Edge Cases
| Case | Expected Behavior |
|------|------------------|
| [case] | [exact behavior] |

## Data Model
[Tables, fields, indexes, constraints]

## API Contract
[Endpoints, request/response shapes, status codes]

## Build Order
1. [Block 1]: [deliverable]
2. [Block 2]: [deliverable]
3. [Block 3]: [deliverable]

## Open Questions
[Anything unresolved that needs a decision before building]

## Definition of Done
- [ ] [Test criterion 1]
- [ ] [Test criterion 2]
- [ ] [Test criterion 3]
```

---

## ANTI-PATTERNS

| Anti-Pattern | What Goes Wrong |
|---|---|
| Speccing the solution, not the problem | Build the wrong thing correctly |
| "Just start coding, we'll figure it out" | Schema migrations, API contract rewrites, thrown-away UI |
| Vague acceptance criteria | "It works" means different things to different people |
| Not speccing error cases | Error handling is retrofitted, inconsistent, and wrong |
| Skipping the data model | Schema changes after data exists = expensive migration |
| "Obvious" edge cases left unspecified | Every edge case that isn't specified will be wrong |
