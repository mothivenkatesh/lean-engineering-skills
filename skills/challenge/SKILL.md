---
name: challenge
description: "Eliminate before implementing. Run before building anything. Works down an elimination ladder: does this need to exist, does it already exist, can it be done without code, can it be done with less code. Trigger on: 'I need to build X', 'we need a feature for Y', 'can you implement', 'we should add', 'build this'. You reach build-it only after everything above is exhausted."
user-invocable: true
argument-hint: "[feature or system being proposed]"
---

# Challenge — Eliminate Before You Implement

## INPUT

None. This is the entry point of the build pipeline.

Takes: a request to build something.

---

## THE LOOP

**TRIGGER:** Any request to build, implement, or add something.

**CYCLE:**
1. Restate the problem behind the request (not the solution)
2. Work down the elimination ladder — stop at the first level that solves it
3. Answer the three questions for anything that survives
4. If "build it": produce DECISION.md and exit

**EXIT CONDITION:** You have a decision. Build / don't build / build less. Exit with DECISION.md.

---

## STEP 1: RESTATE THE PROBLEM

Every request comes as a solution. "Build a notification system." "Add a dashboard." "Create an API."

Restate it as a problem. "Users don't know when their payment fails." "Operators can't see volume."

If you cannot restate it as a problem, you do not understand the request. Do not proceed.

**The trap:** Building the solution as stated without validating it solves the actual problem. This produces correct implementations of the wrong thing.

---

## STEP 2: THE ELIMINATION LADDER

Work top to bottom. Stop at the first level that solves the problem.

```
LEVEL 0: Does this need to be solved at all?
  → Is it a real problem (evidence: user complaints, support tickets, data) or hypothetical?
  → If you don't solve it, what actually happens?
  → If nothing bad happens: stop here.

LEVEL 1: Does it already exist?
  → Existing functionality that handles this case?
  → Existing third-party that solves exactly this?
  → Library or tool that does this?

LEVEL 2: Can you solve it without code?
  → A documented process or runbook
  → A configuration change
  → A database query run on demand
  → A cron job that sends a report

LEVEL 3: Can you solve it with less code?
  → One endpoint instead of a service
  → A SQL view instead of a new table
  → A cron job instead of event-driven
  → Postgres instead of Redis, Redis instead of Kafka

LEVEL 4: Can you build a smaller version to test the assumption?
  → What can be hardcoded or faked until you know it's real?
  → What is the minimum that tells you whether to build fully?

LEVEL 5: Build as proposed.
  → Only here if all above are genuinely exhausted.
```

You do not skip levels. The pressure to jump to Level 5 is constant. Resist it. Levels 0–4 cost hours. Building Level 5 wrong costs weeks.

---

## STEP 3: THE THREE QUESTIONS

For anything that reaches Level 5:

**Q1: What does this enable that is not possible without it?**
Not "it's useful." Specifically: what can users or operators DO after this that they cannot do now? Vague answer = vague feature.

**Q2: What does this add to the operational surface?**
Every feature is maintained forever. Every API endpoint is a contract. Every integration is a dependency. Name what goes on the list of things that can break on a Saturday.

**Q3: Is this reversible?**
API contracts, database schemas, and user-visible behavior are hard to remove. The harder it is to take back, the higher the bar to build it.

---

## SPECIFIC CHALLENGES

### "We need a cache"
Before adding Redis:
- Run EXPLAIN ANALYZE on the slow query
- Add the missing index
- Fix the SELECT * or N+1
- Try vertical scaling

If all exhausted: add a cache. Not before.

### "We need a microservice"
Before splitting:
- Does the monolith have a measurable problem decomposition solves?
- Can you add a module boundary inside the monolith first?
- Are you adding distributed systems complexity to solve a code organization problem?

Shopify handles 30TB/minute on Black Friday with a 2.8M-line Ruby monolith.

### "We need Kafka"
Before adding:
- Is your database table not a valid queue? (Row locking + bounded jobs = valid queue)
- Do you have asymmetric ingestion/processing rates that require buffering?

A Postgres table with a `status` column and a cron job processes thousands of jobs per minute without a message broker.

### "We need a dashboard"
Before building:
- Can the information be queried on demand?
- Can it be a scheduled report (email, CSV) instead of a live UI?
- Who uses this, how often, and what decision does it enable?

Build the SQL query first. If someone runs it manually 5 times a day: build the dashboard. Not before.

### "We need a notification system"
Before building:
- How many notification types actually exist? (1–3: hardcode them)
- Can a cron job check a condition and send an email?
- Is real-time delivery required or is 15-minute delay acceptable?

A notification "system" is a platform. A cron job is a file. Platform = months. File = a day.

---

## THE INDIE FILTER

Three questions before committing to any build:

**Can one person understand this entire system?**
If no: too complex for your team size. Complexity has an operational tax paid every time something breaks.

**Can this run without a DevOps engineer?**
Managed services, not self-managed infrastructure. Railway, Render, Vercel — not Kubernetes.

**Would Pieter Levels build this?**
Levels.io built Nomad List, Remote OK, and multiple revenue-generating products on a single server with minimal dependencies. The constraint is the discipline.

---

## OUTPUT: DECISION.md

Produce this file when exiting the loop:

```markdown
# DECISION.md

## Request
[What was asked]

## Actual Problem
[The problem, restated without the solution]

## Elimination Ladder
Level 0: [Real problem? Evidence?]
Level 1: [Already exists?]
Level 2: [Without code?]
Level 3: [Less code?]
Level 4: [Smaller version?]
Level 5: [Build as proposed — reached only if above exhausted]

## Stopped At
Level [N]: [Why this level is the answer]

## Recommendation
[Build as proposed / Build simpler version / Don't build / Get clarity first]

## If Building: Constraints
- Problem it solves: [specific]
- Operational surface added: [specific]
- Reversible: [yes / no / partial — and why]
- Minimum version: [what can be deferred]
- Feeds into: /spec
```

---

## FEEDS INTO

`/spec` — consumes DECISION.md to define exactly what is being built.
`/stack` — consumes DECISION.md to lock the technology decisions.
