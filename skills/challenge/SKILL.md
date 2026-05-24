---
name: challenge
description: "Eliminate before implementing. Run before building anything. Forces the question: should this be built at all? Is there a simpler path? Trigger on: 'I need to build X', 'we need a feature for Y', 'the user wants Z', 'can you help me implement', 'we should add'. AI models build what they're asked. This skill asks whether what you're asking for is actually what you need."
user-invocable: true
argument-hint: "[feature or system being proposed]"
---

# Challenge — Eliminate Before You Implement

Jason Fried (37signals): "The only way to get more done is to have less to do. It's not time management, it's obligation elimination."

Doug McIlroy (Unix philosophy): "Write programs that do one thing and do it well. To do a new job, build afresh rather than complicate old programs by adding new features. Early Unix developers regularly asked: What can we throw out?"

Zerodha has spent $0 on advertising. They have never added a feature to increase engagement metrics. Their product philosophy is user disengagement — only do what is genuinely useful. $1B+ revenue. $500M+ profit. 35 engineers. 16M customers. The constraint produced the moat.

**This skill does one thing:** pushes back before you build.

---

## THE CHALLENGE PROTOCOL

Run through this before accepting any feature request as a build task.

### Step 1: Restate the problem, not the solution

The request is almost always stated as a solution: "Build a notification system." "Add a dashboard." "Create an API for X."

Restate it as a problem: "Users don't know when their payment fails." "Operators can't see transaction volume." "System Y needs data from system X."

If you can't restate it as a problem, you don't understand the request well enough to build it.

**The trap:** building the solution as stated without validating that it solves the actual problem. This produces correct implementations of the wrong thing.

---

### Step 2: The Elimination Ladder

Work down this ladder. Stop at the first level that solves the problem.

```
Level 0: Does this problem need to be solved at all?
  → Is it a real problem (happens often, causes real pain) or a hypothetical?
  → Do you have evidence it's a problem? (user complaints, support tickets, data)
  → If you don't solve it, what actually happens?

Level 1: Does it already exist?
  → Is there existing functionality that handles this case?
  → Is there an existing third-party service that solves exactly this?
  → Is there a library, package, or tool that does this?

Level 2: Can you solve it without code?
  → A documented process, a runbook, a manual step
  → A configuration change
  → A database query someone runs on demand
  → An email, a Slack message, a cron job that runs a script

Level 3: Can you solve it with less code than proposed?
  → One endpoint instead of a service
  → A SQL view instead of a new table
  → A cron job instead of an event-driven system
  → A feature flag instead of a new UI flow
  → Postgres instead of Redis, Redis instead of Kafka

Level 4: Can you build a smaller version that tests the hypothesis?
  → What's the minimum version that tells you if this is worth building fully?
  → What can you hardcode, fake, or manually handle until you know it's real?

Level 5: Build it as proposed.
  → Only reach here if all levels above are genuinely exhausted
```

**You don't get to skip levels.** The pressure to jump to Level 5 is always there. Resist it. The cost of Levels 0–4 is hours. The cost of building Level 5 wrong is weeks.

---

### Step 3: The Three Questions

For anything that survives the elimination ladder:

**Question 1: What does this enable that isn't possible without it?**
Not "it's useful" or "users would like it." Specifically: what can users or operators DO after this that they cannot do now? If the answer is vague, the feature is vague.

**Question 2: What does this add to the operational surface?**
Every feature is a surface you maintain forever. Every API endpoint is a contract. Every config option is a combinatorial state. Every integration is a dependency. What does this add to the list of things that can break on a Saturday?

**Question 3: Can you take this back?**
If you build it and it's wrong — wrong design, wrong assumption, wrong priority — can you remove it? API contracts, database schemas, and public-facing features are hard to remove once users depend on them. The harder it is to take back, the higher the bar to build it.

---

### Step 4: The Simpler Alternative Test

For any proposed solution, generate a simpler alternative and compare them honestly.

| Dimension | Proposed | Simpler Alternative |
|---|---|---|
| Lines of code | ? | ? |
| New dependencies | ? | ? |
| New infrastructure | ? | ? |
| Time to build | ? | ? |
| Time to operate | ? | ? |
| Can be rolled back | ? | ? |
| Solves the actual problem | ? | ? |

If the simpler alternative solves the actual problem at lower cost in all dimensions: build the simpler alternative. The engineering instinct to build the sophisticated version is a bias, not a judgment.

---

## SPECIFIC CHALLENGES TO RUN

### "We need a cache"

Before adding Redis or any caching layer:
- Have you run EXPLAIN ANALYZE on the slow query?
- Have you added the missing index?
- Are you fetching more data than you need (SELECT *)?
- Are you running queries inside a loop (N+1)?
- Have you tried vertical scaling (more RAM, faster disk)?

If all of these are exhausted and the query is still slow: add a cache. Not before.

### "We need a microservice"

Before splitting:
- Does the monolith actually have a performance problem that decomposition solves?
- Can you add a module boundary inside the monolith first?
- Does the team need independent deployment cycles? (Usually not until 10+ engineers)
- Are you adding a distributed systems problem to solve a code organization problem?

Shopify handles 30TB/minute on Black Friday with a 2.8M-line Ruby monolith. The answer to "should we split this service" is almost always "add a module boundary first."

### "We need Kafka / an event queue"

Before adding:
- Is your database table not a valid queue? (Row locking + bounded jobs = valid queue at moderate scale)
- Do you actually have asymmetric ingestion and processing rates that require buffering?
- Is this event system for decoupling or for actual throughput?

A Postgres table with a `status` column and a cron job processes thousands of jobs per minute without a message broker. Add the broker when you have evidence Postgres can't keep up, not before.

### "We need a dashboard / analytics page"

Before building:
- Can the information be queried from the database on demand?
- Can it be a scheduled report (email, Slack, CSV) instead of a live UI?
- Is this used daily or quarterly? (Quarterly use cases don't need real-time UIs)
- Who specifically is using this, how often, and what decision does it enable?

Build the SQL query first. If someone runs it manually 5 times a day and finds it useful: build the dashboard. Don't build the dashboard on the hypothesis that someone will find it useful.

### "We need a notification system"

Before building:
- How many notification types will actually exist? (If it's 1-3, hardcoding is fine)
- Can this be a cron job that sends an email directly?
- Does this need real-time delivery or is a 15-minute delay acceptable?
- Who receives the notifications? (If it's admins, Slack webhook is sufficient)

A notification "system" is a platform. Platforms take months to build and are hard to change. A cron job that checks a condition and sends an email takes a day and can be deleted without consequences.

---

## THE INDIE/BOOTSTRAPPED LENS

Corporate engineering teams build platforms because they optimize for team autonomy and org chart boundaries. You don't have that problem.

The questions that matter for bootstrapped/indie scale:

**Can one person understand this entire system?**
If not, it's too complex for your team size. Complexity has an operational tax paid every time something breaks.

**Can this run without a DevOps engineer?**
Managed services, not self-managed infrastructure. Railway, Render, Vercel — not Kubernetes. The engineering time saved on infrastructure is engineering time spent on product.

**Would Pieter Levels build this?**
Levels.io built Nomad List, Remote OK, and multiple products with a single Node.js server, SQLite (later Postgres), and minimal dependencies. Each product generates hundreds of thousands of dollars. The constraint is the discipline.

**Does this generate complexity that outlasts the feature?**
Some features are easy to add and hard to remove. The database schema, the API contract, the user-facing behavior — these create expectations. Build features that are easy to reason about and easy to remove if they're wrong.

---

## ACCEPTANCE CRITERIA FOR BUILDING

Only proceed to build if:
1. You've confirmed the problem is real (evidence, not hypothesis)
2. You've exhausted simpler alternatives
3. You can state clearly what this enables that isn't possible without it
4. You've accepted the operational surface it adds
5. You know whether it's reversible or not

If any of these are unclear: the answer is not "build it anyway." The answer is "get clarity first."

---

## OUTPUT FORMAT

```
## Request
[What was asked for]

## Actual Problem
[The underlying problem, restated without the solution]

## Elimination Ladder Result
Level 0: [Does it need solving? Evidence?]
Level 1: [Already exists?]
Level 2: [Solve without code?]
Level 3: [Solve with less code?]
Level 4: [Build smaller to test first?]
Level 5: [Build as proposed — only if above are exhausted]

## Simpler Alternative
[Concrete alternative with comparison table]

## Recommendation
[Build as proposed / Build simpler version / Don't build / Get more information first]

## If Building: Acceptance Criteria
- The problem it solves: [specific]
- The operational surface it adds: [specific]
- Whether it's reversible: [yes/no, and why]
- The minimum version: [what can be deferred]
```
