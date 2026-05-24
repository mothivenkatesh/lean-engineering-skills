# lean-engineering-skills

Claude Code skills for building stable systems. Block by block. No bloat.

These are not productivity shortcuts. They are guardrails. Each skill stops a category of mistake before it becomes a production incident.

Built from the engineering philosophy of Kailash Nadh (Zerodha), Sriram Velamur (Beetle/techiev2), Matt Pocock, Dan Luu, and Charity Majors. Indie and bootstrapped lens throughout. No Kubernetes before you need it. No Kafka before Postgres can't handle it.

---

## What this is

11 skills that form a complete engineering system. Each one covers a phase of building: before you write code, while you write code, before you ship, and after it runs.

The skills enforce the discipline that comes from being paged at 3am. AI models complete tasks. These skills question tasks.

---

## Install

Copy any skill folder into your Claude Code skills directory:

```bash
cp -r skills/tdd ~/.claude/skills/tdd
```

Or copy all at once:

```bash
cp -r skills/* ~/.claude/skills/
```

Restart Claude Code. The skill is available immediately as `/skill-name`.

---

## The 11 skills

### Before you build

**`/challenge`**
Run this before accepting any build task. Works down an elimination ladder: does this need to exist, does it already exist, can it be done without code, can it be done with less code. You reach "build it" only after everything above is exhausted.

The instinct to build is a bias. This skill corrects it.

**`/spec`**
Turns vague requirements into exact specifications before code. Inputs, outputs, edge cases, data model, build order. Every ambiguity that surfaces here would have surfaced as a bug in production.

Code written without a spec is a bet. This forces the bet to be explicit.

**`/premortem`**
Imagines the system has already failed in production. Works backward to find what killed it. Covers config bugs (the most common cause of global outages), error handling gaps, N+1 queries, cascade failures, thundering herd, idempotency failures, and reversibility.

AI models think forward. This skill thinks backward.

### While you build

**`/stack`**
Locks the technology stack. Strapi v5 for backend and CMS. frappe-ui design tokens for UI. Vercel for frontend. Railway or Render for Strapi.

Contains setup commands, API patterns, frappe-ui token reference, and deployment steps. Deviation requires a written reason, not a feeling.

**`/modular`**
Enforces SOLID principles at the code level. Single responsibility, open/closed, Liskov substitution, interface segregation, dependency inversion. Practical examples for each, applied to the Strapi + Next.js stack.

The test: can you change this module without touching any other module?

**`/tdd`**
Red, green, refactor. Failing test first. Minimum code to pass. Refactor only when green. No code without a failing test.

Covers parameterized tests, mocking discipline, bug fix protocol (write a test that reproduces the bug before fixing it), and the anti-patterns that make test suites useless.

**`/db-review`**
Reviews database schema, queries, and access patterns. Covers missing indexes, unsafe migrations, N+1, unbounded queries, wrong data types (money as float is the most expensive mistake), and when to reach for Redis, Elasticsearch, or another tool versus fixing the Postgres query.

The canonical rule: exhausts all Postgres-level optimizations before recommending a new database.

**`/hot-path`**
Reviews code that runs on every request. Checks for network calls inside loops, missing timeouts, disk IO on hot paths, sequential calls that could be parallel, and allocation in tight loops.

Kailash Nadh's discipline: everything on the hot path serves from memory. Zerodha hits 40ms mean latency at 40,000+ QPS on Postgres + Redis + Go with 35 engineers. No Kubernetes.

### Before you ship

**`/ops-ready`**
Production readiness check before any deploy. Confirms: structured logging on every request, four golden signals instrumented (latency, errors, traffic, saturation), rollback procedure written and tested, database migrations are backward-compatible, all external calls have timeouts, startup validates required env vars.

The test: can someone who didn't write this code fix it at 3am using only a runbook?

### After it runs

**`/diagnose`**
Systematic root cause analysis. Reproduce, isolate, hypothesize, test, name the root cause, fix, write a regression test. The discipline: no code changes before the root cause is stated in one sentence.

Fixing the symptom brings the bug back. This forces the root cause.

**`/debt-audit`**
Technical debt inventory and payoff plan. Classifies debt by type (design, code, test, infra, dependency, schema). Scores by compounding speed, strategic blockage, and failure risk. Produces a prioritized payoff sequence.

Kailash Nadh's principle: debt creates conditions for more debt. Velocity increases after each systematic repayment.

---

## How to use these together

The sequence for any new feature:

```
/challenge   → should this be built at all?
/spec        → what exactly are we building?
/premortem   → what kills this in production?
/db-review   → is the schema safe before data exists?
/modular     → are the module boundaries clean?
/tdd         → build it with tests
/hot-path    → is the hot path clean?
/ops-ready   → can this be operated in production?
```

When something breaks:

```
/diagnose    → root cause before any fix
```

Every few sprints:

```
/debt-audit  → what is compounding and blocking?
```

---

## The philosophy

No VC engineering blog recommendations. No Kafka until Postgres can't handle it. No Kubernetes until one server won't. No microservices until the monolith has proven module boundaries.

The engineers this is built from:

- **Kailash Nadh** (Zerodha CTO): $1B+ revenue, $500M+ profit, 35 engineers, 16M users, $0 marketing, $0 advertising, Postgres + Redis + Go, no Kubernetes.
- **Sriram Velamur** (techiev2, Beetle): "Platforms are grown, not built." "Debt is a tool. The question is what does this prevent next quarter."
- **Dan Luu**: A decade of postmortem analysis. Config bugs cause more global outages than code bugs. Error handling is where the real bugs live.
- **Charity Majors**: "You only get intuition by spending time in production." No senior engineer title without production ownership.
- **37signals / DHH**: "The only way to get more done is to have less to do. It's not time management, it's obligation elimination."
- **Matt Pocock**: Engineering skills as composable, reusable reasoning loops.

One shared principle across all of them: boring is better than clever. A system you understand fully is safer than one you admire.

---

## What this is not

Not a framework. Not a methodology. Not a certification.

Each skill is a Markdown file with structured reasoning and checklists. You run it. You follow it. It stops specific categories of mistakes.

The skills work because they encode judgment that comes from failure. Not hypothetical failure. Real production incidents, postmortems, and 3am pages, distilled into questions you ask before the incident happens.

---

## Adding your own

The skill format is simple:

```
your-skill/
└── SKILL.md
    ├── YAML frontmatter (name, description, triggers)
    └── Markdown instructions
```

Keep SKILL.md under 500 lines. One skill, one concern. Name describes the action, not the category.

---

## License

MIT. Use, modify, share.
