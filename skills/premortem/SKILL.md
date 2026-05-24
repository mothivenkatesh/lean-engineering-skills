---
name: premortem
description: "Production failure thinking BEFORE building. Imagines the system has already failed. Works backward to find what killed it. Trigger on: 'I'm about to build X', 'before I start', 'is this design solid?', 'review my plan', 'anything I'm missing?'. AI models complete tasks. This skill asks whether the task, as designed, will survive contact with reality."
user-invocable: true
argument-hint: "[feature, system, or design to stress-test]"
---

# Premortem — Kill Your System Before Reality Does

## INPUT

Consumes: `SPEC.md` from `/spec`

Uses: behaviors, edge cases, interface definitions, data model, build order.

If SPEC.md doesn't exist: run `/spec` first. You cannot stress-test a system that isn't defined.

---

## THE LOOP

**TRIGGER:** SPEC.md is complete. Before writing any code.

**CYCLE:**
1. State what is being built (one paragraph — forces clarity)
2. Imagine it's 90 days out and the system is down — write the failure story
3. Run every failure mode category (7 categories, no skipping)
4. Classify each finding as critical / significant / acceptable
5. Rate reversibility of every key decision

**EXIT CONDITION:** Every failure mode category has been examined. Critical findings are addressed before build starts. RISK.md is written.

**Rule Zero:** If reviewing existing code, run `git log --oneline -20` before changing anything. The "weird" timeout, the "unnecessary" check — read the git history first. Chesterton's Fence. Do not remove what you don't understand.

---

## STEP 1: STATE WHAT IS BEING BUILT

One paragraph. Concrete. If it's vague, the failure analysis will be vague.

---

## STEP 2: THE FAILURE STORY

Imagine it is 90 days from now. The system is down. Production users are affected. You are paged.

Write the most plausible incident narrative:
- What failed?
- When did it start?
- How long until the alert fired?
- How long until you found the root cause?
- How long until it was fixed?

Aim for the boring, mundane failure. The common ones are: a config change, a missing index, an API that went down with no timeout, a silent error getting swallowed.

---

## STEP 3: FAILURE MODE CHECKLIST

Work every category. No skipping.

---

### CATEGORY 1: CONFIG AND ENVIRONMENT

Dan Luu: ~50% of global outages are caused by config changes, not code. Config rarely gets the same review rigor as code.

- [ ] What env vars, feature flags, API keys, or limits does this depend on?
- [ ] What happens if a config value is wrong? (wrong URL, wrong limit)
- [ ] What happens if a config value is missing? (null, empty string, undefined)
- [ ] Is config validated at startup or silently used wrong at runtime?
- [ ] Does dev config mask a prod problem? (permissive dev, strict prod)

**Most likely failure:** Works in staging because staging has a permissive config. Prod has a stricter value. Nobody checked.

---

### CATEGORY 2: ERROR HANDLING

Bugs in error handling cause the most severe outages. The error path is the least-tested code in every system.

- [ ] What happens when the database is slow? (timeout? retry? queue? fail open?)
- [ ] What happens when an external API returns 500?
- [ ] What happens when an external API times out? Is there a timeout set at all?
- [ ] What happens when the error handler itself throws?
- [ ] Does failure here affect unrelated features? (shared DB pool, shared process)
- [ ] Does the user get a recoverable error, or a dead end?

**Most likely failure:** Silent swallowing. `catch (e) {}` with no fallback. User sees a blank screen. No log. No alert.

---

### CATEGORY 3: SCALE AND LOAD

Not "will this scale to 10M users." The real question: what breaks at **3x current load**?

- [ ] Is there a query inside a loop? (N+1)
- [ ] Is there a list endpoint without LIMIT? (unbounded result set)
- [ ] Is there a cache? What happens when it's cold? (thundering herd on expiry)
- [ ] What if 100 users submit this form simultaneously? (race condition, duplicate writes)
- [ ] Does this hold a database lock? For how long?

**Thundering herd fix:** `TTL = base_ttl + random(0, base_ttl * 0.1)` — stagger cache expiry across instances.

---

### CATEGORY 4: EXTERNAL DEPENDENCIES

Every external call is a failure surface you don't control.

- [ ] Does this call an external API?
- [ ] Is there a timeout on every external call? What is it?
- [ ] What happens when the external API is down? Hard fail or degrade gracefully?
- [ ] What happens when it's slow (100ms becomes 5000ms)?
- [ ] Is there a circuit breaker?
- [ ] Can this feature be used without the external dependency?

**The rule:** Design for their failure, not their success.

---

### CATEGORY 5: DATA INTEGRITY

Partial failures leave data in inconsistent states. Invisible until a user reports "my order shows paid but nothing was delivered."

- [ ] What happens if the process crashes halfway through?
- [ ] Is this operation atomic? Or could it partially succeed?
- [ ] If this writes to multiple tables, what if the second write fails?
- [ ] If this calls an external API then writes to DB, what if the DB write fails after the API call succeeded?
- [ ] Are there duplicate submission risks? Is this idempotent?

**Idempotency rule:** For operations that create money, send messages, or change critical state — use an idempotency key. Client generates a unique key. Server stores it. Same key arrives twice: return the original result. Never use timestamps as idempotency keys (clock skew).

---

### CATEGORY 6: OBSERVABILITY

"A large fraction of worst near-disasters come from not having the right alerting." — Dan Luu

- [ ] How will you know this is broken before a user reports it?
- [ ] What does this log on success? On failure?
- [ ] What metric spikes or drops when this fails?
- [ ] Is there an alert that fires when the failure metric crosses a threshold?
- [ ] Can you answer "how many times did this fail in the last hour?" from logs?

**Minimum floor:**
1. Latency — how long does this take? What's the p99 baseline?
2. Error rate — what % of requests fail? Alert above 1%.
3. Traffic — sudden drop = users can't reach it.
4. Saturation — DB connections, memory, disk running out?

---

### CATEGORY 7: ROLLBACK

Every deploy is reversible until you say otherwise.

- [ ] Can you roll this back without a data migration?
- [ ] If this adds a DB column, can the previous code version run against the new schema?
- [ ] If this changes an API response shape, do old clients break?
- [ ] What's the rollback procedure? Can someone else execute it cold?

**The test:** Write rollback steps in plain English. More than 5 steps or specialized knowledge required = riskier than it appears.

---

## REVERSIBILITY RATING

Classify every key decision before committing:

| Decision | Type 1 (irreversible) | Type 2 (reversible) |
|---|---|---|
| Schema change that removes a column | ✓ | |
| New endpoint added | | ✓ |
| Renamed API field that clients depend on | ✓ | |
| Behind a feature flag | | ✓ |
| External API integration, no fallback | ✓ | |

**Type 1 decisions require deliberate acceptance.** Name them before committing.

---

## OUTPUT: RISK.md

```markdown
# RISK.md

## System Under Review
[What is being built — one paragraph]

## The Failure Story
[Most plausible 90-day incident. What failed, when, how long to detect, how long to fix.]

## Critical Findings (fix before shipping)
| Mode | Likelihood | Impact | Fix |
|------|------------|--------|-----|

## Significant Findings (fix before scaling)
| Mode | Likelihood | Impact | Fix |

## Acceptable (document and monitor)
| Mode | Why acceptable | Monitoring |

## Reversibility Assessment
- Type 1 (irreversible): [list]
- Type 2 (reversible): [list]

## Observability Gap
- Visible today: [list]
- Must add before deploy: [list]

## Go / No-Go
[Ship as-is / Ship with fixes / Redesign needed]
[Minimum safe version if redesign is too costly]

## Feeds into
/ops-ready (DEPLOY.md)
```

---

## FEEDS INTO

`/ops-ready` — uses RISK.md failure modes and observability gaps to build the deploy checklist.
