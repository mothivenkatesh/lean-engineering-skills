# lean-engineering-skills

Claude Code skills for building stable systems. Block by block. No bloat.

These are not productivity shortcuts. They are guardrails. Each skill stops a category of mistake before it becomes a production incident.

Built from the engineering philosophy of Kailash Nadh (Zerodha), Sriram Velamur (Beetle/techiev2), Dan Luu, and Charity Majors. Indie and bootstrapped lens throughout. No Kubernetes before you need it. No Kafka before Postgres can't handle it.

---

## What this is

11 skills that form a complete engineering pipeline. Each skill is a composable reasoning loop — it consumes a named artifact from the previous skill and produces a named artifact for the next.

Every skill has:
- **INPUT** — what artifact it consumes from the previous skill
- **THE LOOP** — explicit trigger, cycle, and exit condition
- **OUTPUT** — a named artifact the next skill uses as structured input

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

## The artifact pipeline

Each skill produces a named artifact consumed by the next.

```
/challenge  →  DECISION.md
/spec       →  SPEC.md          (consumes DECISION.md)
/stack      →  STACK.md         (consumes DECISION.md)
/premortem  →  RISK.md          (consumes SPEC.md)
/db-review  →  SCHEMA.md        (consumes SPEC.md + RISK.md)
/modular    →  MODULE_MAP.md    (consumes SPEC.md + SCHEMA.md)
/tdd        →  TEST_REPORT.md   (consumes SPEC.md + MODULE_MAP.md)
/hot-path   →  PERF.md          (consumes TEST_REPORT.md)
/ops-ready  →  DEPLOY.md        (consumes RISK.md + PERF.md)

Incident path:
/diagnose   →  RCA.md           (consumes DEPLOY.md runbook)
/debt-audit →  DEBT.md          (consumes RCA.md patterns or runs standalone)
```

Each artifact is a Markdown file. You can check them into git. You can pass them between skills. You can build a paper trail from decision to deploy.

---

## The 11 skills

### Before you build

**`/challenge`**
Eliminate before implementing. Works down an elimination ladder: does this need to exist, does it already exist, can it be done without code, can it be done with less code. You reach "build it" only after everything above is exhausted.

Produces: `DECISION.md` — the build/don't-build decision with constraints.

**`/spec`**
Turns vague requirements into exact specifications. Every behavior is written as GIVEN/WHEN/THEN — the same format as a test. Every edge case has an explicit expected outcome. Every field has a type and a reason.

Consumes: `DECISION.md`. Produces: `SPEC.md`.

**`/premortem`**
Imagines the system has already failed. Works backward to find what killed it. Seven failure mode categories: config and environment, error handling, scale and load, external dependencies, data integrity, observability, rollback. Includes Chesterton's Fence (Rule Zero: read git history before changing anything).

Consumes: `SPEC.md`. Produces: `RISK.md`.

### While you build

**`/stack`**
Locks the technology stack. Strapi v5 for backend and CMS. frappe-ui design tokens for UI. Vercel for frontend. Contains setup commands, Strapi v5 API patterns (documentId not id, flat response shape), frappe-ui token reference, and deployment steps. Deviation requires a written reason.

Consumes: `DECISION.md`. Produces: `STACK.md`.

**`/db-review`**
Reviews database schema, queries, and access patterns. Covers missing indexes, unsafe migrations, N+1, unbounded queries, wrong data types (money as float is the most expensive mistake), and when to reach for another tool versus fixing the Postgres query. The canonical rule: exhaust all Postgres-level optimizations before recommending a new database.

Consumes: `SPEC.md + RISK.md`. Produces: `SCHEMA.md`.

**`/modular`**
Enforces SOLID principles at the code level. Single responsibility, open/closed, Liskov substitution, interface segregation, dependency inversion. Practical TypeScript examples for each, applied to the Strapi + Next.js stack. The test: can you change this module without touching any other module?

Consumes: `SPEC.md + SCHEMA.md`. Produces: `MODULE_MAP.md`.

**`/tdd`**
Red, green, refactor. Failing test first. Minimum code to pass. Refactor only when green. No code without a failing test. Module interfaces from MODULE_MAP.md define mock boundaries. Bug fix protocol: write a test that reproduces the bug before fixing it.

Consumes: `SPEC.md + MODULE_MAP.md`. Produces: `TEST_REPORT.md`.

**`/hot-path`**
Reviews code that runs on every request. Checks for N+1 queries, queries inside loops, sequential calls that could be parallel, missing timeouts on external calls, disk I/O on hot paths, and allocation in tight loops. Measure before and after — not "faster," but "p99 dropped from 450ms to 80ms."

Consumes: `TEST_REPORT.md`. Produces: `PERF.md`.

### Before you ship

**`/ops-ready`**
Production readiness check before any deploy. Five sections: observability (four golden signals from PERF.md baseline), rollback (procedure written before deploying), failure mode documentation (from RISK.md), dependency audit (every external call has a timeout), deploy procedure (one-command deploy, env vars validated at startup, smoke tests). Includes the $0/month indie ops stack.

Consumes: `RISK.md + PERF.md`. Produces: `DEPLOY.md`.

### After it runs

**`/diagnose`**
Systematic root cause analysis. Reproduce, isolate, hypothesize, test, iterate until root cause is found. The discipline: no code changes before the root cause is stated in one sentence. DEPLOY.md runbook is the starting point for incident response.

Consumes: `DEPLOY.md` (runbook). Produces: `RCA.md`.

**`/debt-audit`**
Technical debt inventory and payoff plan. Six debt categories (design, code, test, infra, dependency, schema). Scoring: compounding + strategic blockage + failure risk = priority. Sequenced payoff plan. Done criteria for each item (measurable, not "code is cleaner"). Uses patterns from RCA.md to find structural debt.

Consumes: `RCA.md` patterns or runs standalone. Produces: `DEBT.md`.

---

## How to use these together

The sequence for any new feature:

```
/challenge   → should this be built at all?      → DECISION.md
/spec        → what exactly are we building?     → SPEC.md
/stack       → what tools are locked in?         → STACK.md
/premortem   → what kills this in production?    → RISK.md
/db-review   → is the schema safe?               → SCHEMA.md
/modular     → are the module boundaries clean?  → MODULE_MAP.md
/tdd         → build it with tests               → TEST_REPORT.md
/hot-path    → is the hot path clean?            → PERF.md
/ops-ready   → can this be operated in prod?     → DEPLOY.md
```

When something breaks:

```
/diagnose    → root cause before any fix         → RCA.md
```

Every few sprints:

```
/debt-audit  → what is compounding and blocking? → DEBT.md
```

---

## The philosophy

No VC engineering blog recommendations. No Kafka until Postgres can't handle it. No Kubernetes until one server won't. No microservices until the monolith has proven module boundaries.

The engineers this is built from:

- **Kailash Nadh** (Zerodha CTO): $1B+ revenue, $500M+ profit, 35 engineers, 16M users, $0 marketing, $0 advertising, Postgres + Redis + Go, no Kubernetes. 40ms mean latency at 40,000+ QPS.
- **Sriram Velamur** (techiev2, Beetle): "Platforms are grown, not built." "Debt is a tool. The question is what does this prevent next quarter."
- **Dan Luu**: A decade of postmortem analysis. ~50% of global outages are caused by config changes, not code. Error handling is where the real bugs live.
- **Charity Majors**: "You only get intuition by spending time in production." No senior engineer title without production ownership.
- **37signals / DHH**: "The only way to get more done is to have less to do. It's not time management, it's obligation elimination."

One shared principle: boring is better than clever. A system you understand fully is safer than one you admire.

---

## What this is not

Not a framework. Not a methodology. Not a certification.

Each skill is a Markdown file with a structured reasoning loop and checklists. You run it. You follow it. Each skill produces a named artifact. That artifact feeds the next skill.

The skills work because they encode judgment that comes from failure. Not hypothetical failure. Real production incidents, postmortems, and 3am pages, distilled into questions you ask before the incident happens.

---

## Adding your own

The skill format:

```
your-skill/
└── SKILL.md
    ├── YAML frontmatter (name, description, user-invocable)
    ├── INPUT section (what artifact this consumes)
    ├── THE LOOP section (trigger, cycle, exit condition)
    ├── Body (reasoning, checklists, examples)
    └── OUTPUT section (named artifact this produces, and what feeds into next)
```

Keep SKILL.md under 500 lines. One skill, one concern. Name describes the action. Every skill produces a named artifact.

---

## License

MIT. Use, modify, share.
