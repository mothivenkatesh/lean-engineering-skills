---
name: premortem
description: "Production failure thinking BEFORE building. Run this before writing any feature, schema, or integration. Ask: what kills this in production? Trigger on: 'I'm about to build X', 'before I start', 'is this design solid?', 'review my plan', 'pre-launch check', 'anything I'm missing?'. AI models complete tasks. This skill asks whether the task, as designed, will survive contact with reality."
user-invocable: true
argument-hint: "[feature, system, or design to stress-test]"
---

# Premortem — Kill Your System Before Reality Does

AI completes tasks. It does not ask whether the task, as designed, will cause a 3am incident in three months.

This skill does.

**The method** (from PayPal engineering + Dan Luu's decade of postmortem analysis): Imagine the system has already failed catastrophically. Work backward. What killed it?

Prospective hindsight — imagining failure has already occurred — increases the ability to identify real failure reasons by 30% compared to forward-looking risk assessment. You find things you never find by asking "what could go wrong?"

---

## RULE ZERO: READ BEFORE JUDGING

Before running this protocol on existing code: read the git history. Run `git log --oneline -20` on the relevant files. That "weird" timeout value, that "unnecessary" check, that "redundant" query — look for the commit that introduced it before deciding it's wrong.

Chesterton's Fence: do not remove or change anything until you understand why it was put there. The sleep() call that looks like dead code may be preventing a race condition on a specific OS. The 31-second timeout may have been tuned because 30 seconds caused a production incident.

If git history is silent and there's no comment: the code may encode knowledge that was never written down. That is the most dangerous kind.

---

## THE PREMORTEM PROTOCOL

### Step 1: State what you're building (one paragraph)

Write it out. If you can't describe it clearly, you don't understand it yet. Vague design produces vague failure analysis.

### Step 2: Imagine it's 90 days from now and the system is down

Not "might be down." It IS down. Production users are affected. You are being paged.

Now answer:
- What failed?
- When did it start failing?
- How long until you found out?
- How long until you fixed it?

Write the most plausible incident story. Don't aim for the dramatic — aim for the mundane. The most common production failures are boring: a config change, a missing index, a third-party API that went down.

### Step 3: Run the failure mode checklist

Work through each category. Don't skip.

---

## FAILURE MODE CHECKLIST

### Category 1: Config and Environment

Dan Luu's most important postmortem finding: ~50% of global outages are caused by **config changes, not code changes**. Config rarely gets the same staging/review rigor as code.

- [ ] What config values does this feature depend on? (env vars, feature flags, API keys, limits)
- [ ] What happens if a config value is wrong? (wrong URL, wrong limit, wrong flag)
- [ ] What happens if a config value is missing? (undefined, null, empty string)
- [ ] Is config validated at startup or silently used wrong at runtime?
- [ ] Does config differ between dev and prod in a way that masks problems?

**Most likely failure here:** The feature works in staging because staging has a permissive config. Prod has a stricter value. Nobody checked.

### Category 2: Error Handling

Bugs in error handling code cause disproportionately severe outages. The error path is the least-tested code in most systems.

- [ ] What happens when the database is slow? (timeout? retry? queue? fail open?)
- [ ] What happens when an external API returns 500? (crash? bad error message? silent fail?)
- [ ] What happens when an external API times out? Is there a timeout set at all?
- [ ] What happens when the error handler itself throws an error?
- [ ] Does a failure in this feature affect unrelated features? (shared DB connection pool, shared process)
- [ ] Are errors surfaced to the user in a recoverable way, or is the UX a dead end?

**Most likely failure here:** Error swallowing. `catch (e) {}` or `catch (e) { logger.error(e) }` with no fallback behavior. User sees a blank screen or stale data with no explanation.

### Category 3: Scale and Load

Not "will this scale to 10M users" — that's irrelevant for most systems. The real question: what breaks at **3x current load**?

- [ ] Is there a query that runs inside a loop? (N+1)
- [ ] Is there a list endpoint without a LIMIT? (unbounded result set)
- [ ] Is there a cache? What happens when it's cold? (thundering herd on cache expiry)
- [ ] What happens if 100 users submit this form simultaneously? (race condition? duplicate writes?)
- [ ] Does this feature hold a database lock? For how long?

**Thundering herd:** Multiple servers, all caches expire simultaneously, all hit the database at once. Database goes from 10 QPS to 1000 QPS in one second. If any retry logic exists, load amplifies further (retry storm). Solution: staggered TTLs with jitter. `TTL = base_ttl + random(0, base_ttl * 0.1)`.

### Category 4: External Dependencies

Every external HTTP call is a failure surface you don't control.

- [ ] Does this feature call an external API? (payment gateway, email service, SMS, third-party)
- [ ] Is there a timeout on every external call? (what is it? is it appropriate?)
- [ ] What happens when the external API is down? Does the feature fail hard or degrade gracefully?
- [ ] What happens when the external API is slow (100ms becomes 5000ms)?
- [ ] Is there a circuit breaker? (stop hammering a failing service)
- [ ] Can this feature be used without the external dependency? (degraded mode)

**The rule:** Every external dependency is a dependency on someone else's uptime. Design for their failure, not their success.

### Category 5: Data Integrity and Consistency

Partial failures leave data in inconsistent states. This is invisible until a user reports "my order shows paid but no item was delivered."

- [ ] What happens if the process crashes halfway through this operation?
- [ ] Is this operation atomic? (database transaction) Or could it partially succeed?
- [ ] If this writes to multiple tables, what happens if the second write fails?
- [ ] If this calls an external API and then writes to the database, what happens if the DB write fails after the API call succeeded?
- [ ] Are there duplicate submission risks? (user clicks submit twice) Is this operation idempotent?

**Idempotency rule:** For any operation that creates money, sends a message, or changes critical state — use an idempotency key. The client generates a unique key. The server stores it. If the same key arrives twice, return the original result. Never use timestamps as idempotency keys — clock skew makes them unreliable.

### Category 6: Observability

Dan Luu: "A large fraction of worst near-disasters come from not having the right alerting set up." If you can't see it, you can't know it's failing.

- [ ] How will you know this feature is broken before a user reports it?
- [ ] What logs does this feature emit on success? On failure?
- [ ] What metric would spike or drop if this feature started failing?
- [ ] Is there an alert that fires when the failure metric crosses a threshold?
- [ ] Can you query production logs to answer "how many times did this fail in the last hour?"

**Minimum floor (Four Golden Signals applied to indie scale):**
1. **Latency** — How long does this take? What's normal? What's a red flag?
2. **Error rate** — What % of requests fail? Alert when it exceeds 1%.
3. **Traffic** — How much is this used? Sudden drop = users can't reach it.
4. **Saturation** — Is anything running out? (DB connections, memory, disk)

### Category 7: Rollback

Every deploy is reversible until you say otherwise.

- [ ] If this feature breaks in production, can you roll it back without a data migration?
- [ ] If this adds a database column, can the previous version of the code run against the new schema?
- [ ] If this changes an API response shape, will old clients break?
- [ ] What's the rollback procedure? Can someone else execute it cold without paging you?

**The test:** Write the rollback steps in plain English. If they take more than 5 steps or require specialized knowledge, the deploy is riskier than it appears.

---

## REVERSIBILITY RATING

Before shipping, classify this feature's key decisions:

| Decision | Type 1 (irreversible) | Type 2 (reversible) |
|---|---|---|
| Schema change that removes a column | ✓ | |
| New endpoint added | | ✓ |
| Renamed API field that clients depend on | ✓ | |
| Feature flag behind a toggle | | ✓ |
| External API integration (no fallback) | ✓ | |
| New optional feature (can be disabled) | | ✓ |

**Type 1 decisions require deliberate acceptance.** If a decision is irreversible and wrong, you're living with it. Name it explicitly before committing.

---

## SPECIAL CASE: BORING VS CLEVER

Indie/bootstrapped lens: clever systems are expensive to operate. Boring systems run at 3am without anyone awake.

Before choosing any novel approach, ask:
- "Is there a bash script + cron job that solves this?"
- "Is there a SQL query that replaces this service?"
- "Is there a Postgres feature that replaces this infrastructure?"

Zerodha runs 15-20% of India's daily stock trades on Postgres + Redis + Go with 35 engineers. No Kafka. No Kubernetes. No microservices. Boring scales when well-applied.

A system you don't understand is a liability at 3am. A system you understand fully is an asset.

---

## OUTPUT FORMAT

```
## System Under Review
[What is being built — one paragraph]

## The Failure Story
[The most plausible 90-day incident. Narrative form. What failed, when, how long to detect, how long to fix.]

## Failure Modes Found

### Critical (fix before shipping)
| Mode | Likelihood | Impact | Fix |
|------|------------|--------|-----|
| N+1 query on hot path | High | DB overload at 3x load | Batch load with JOIN |
| ...

### Significant (fix before scaling)
...

### Acceptable (document and monitor)
...

## Reversibility Assessment
- Type 1 decisions (irreversible): [list]
- Type 2 decisions (reversible): [list]

## Observability Gap
- What's visible today: [list]
- What must be added before deploy: [list]

## Go / No-Go
[Verdict: ship as-is / ship with fixes / redesign needed]
[What the minimum-safe version is if redesign is too costly]
```

---

## WHAT THIS SKILL CANNOT REPLACE

This skill encodes patterns from decades of production postmortems. It does not replace:
- Your knowledge of your specific system's constraints
- Reading your own git history and incident reports
- The intuition that comes from having been paged at 2am for your own code

Charity Majors: "You only get intuition by spending time in production." This skill accelerates the pattern recognition. It does not substitute for production ownership.
