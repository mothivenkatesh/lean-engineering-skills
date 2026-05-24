---
name: debt-audit
description: "Technical debt audit and payoff plan. Use when code is getting harder to change, velocity is slowing, engineers avoid certain areas, or before a major refactor. Trigger on: 'refactor this', 'this codebase is a mess', 'it's getting harder to ship', 'tech debt', 'code health', 'we keep breaking things'. Debt compounds exponentially. The right time to audit is before it becomes emergency surgery."
user-invocable: true
argument-hint: "[codebase, module, or area to audit]"
---

# Debt Audit — Technical Debt Inventory and Payoff Plan

## INPUT

Consumes: `RCA.md` from `/diagnose` (patterns of bugs reveal structural debt) or run standalone periodically.

Looks at: the current codebase, git history, test coverage, and incident history.

---

## THE LOOP

**TRIGGER:** Velocity is slowing. Incident patterns suggest structural problems. Periodic (every few sprints). Or before a major refactor.

---

## HARD CONSTRAINTS
- **Required:** A specific module, file, or bounded area. "Audit the codebase" is too broad — ask for a specific area before starting.
- **Refuse to classify debt from memory.** Read the code first. Every classification requires reading the actual file.
- **Mandatory:** Build test coverage before paying down design debt. Do not restructure code that has no tests.
- **Every debt item requires a score and done criteria.** Items without scores cannot be prioritized. Items without done criteria never get finished.
- **Refuse if:** User wants to rewrite everything at once. Identify the strangler pattern. Start with the highest-compounding item only.

**CYCLE:**
1. Read the code before classifying it (never from memory or assumptions)
2. Classify every debt item by type
3. Score each item on three axes (compounding, strategic blockage, failure risk)
4. Sequence the payoff in dependency order
5. Define "done" for each item

**EXIT CONDITION:** All debt is visible, scored, and sequenced. DEBT.md is written with a prioritized payoff plan.

---

## THE CORE PRINCIPLE

Kailash Nadh: "Technical debt is exponential. Debt creates conditions for more debt. If you never pause to clear technical debt, it grows exponentially and the system becomes a burden. We have rewritten core systems multiple times. Velocity increased after each rewrite."

Sriram Velamur: "Debt is a tool. The strategic question is: what does this debt prevent us from doing next quarter?"

Debt isn't inherently bad. Unexamined debt is. Taking debt to ship faster is valid. Not knowing what debt you've taken — or what it costs to carry — is negligence.

---

## PHASE 1: INVENTORY

Read the code before classifying it.

### Design Debt
Wrong abstraction or structure for current requirements.
- Modules doing too many things (SRP violated)
- Abstractions that made sense 12 months ago but now fight every new feature
- Modules tightly coupled when they should be independent
- Missing abstractions — duplicated logic across 5 files

### Code Debt
Implementation quality issues.
- Functions longer than 50 lines (usually doing more than one thing)
- Unclear naming (`handleData()`, `processStuff()`)
- Deep nesting (> 3 levels signals missing extraction)
- Commented-out code without explanation
- Magic numbers and strings (`status === 3` — what does 3 mean?)
- TODO/FIXME/HACK comments without resolution dates

### Test Debt
Missing or misleading coverage.
- No tests for critical paths
- Tests that test implementation, not behavior (break on refactor)
- Tests with no assertions (only check that code runs)
- Test setup so complex the test is hard to understand
- Flaky tests (intermittent failures erode trust in the entire suite)

### Infrastructure Debt
Operational risk.
- Undocumented environment variables
- Manual deployment steps not in CI/CD
- No health checks
- Logs without enough context to debug production issues
- Missing or stale runbooks

### Dependency Debt
External risk.
- Dependencies with known CVEs (`npm audit`, `pip-audit`, `bundle audit`)
- Abandoned or unmaintained libraries
- Version ranges so wide they include breaking changes
- Transitive dependency conflicts

### Schema / Data Debt
Structural risk.
- Columns whose names no longer match their use
- Nullable columns that should never be null
- Missing indexes on filtered/sorted columns
- Data requiring code to interpret (magic integers as status codes)

---

## PHASE 2: COST ANALYSIS

For each debt item:

**Carrying cost:** What does this cost every sprint?
- "Every time we touch this module, 2 hours of understanding before any change"
- "3 production incidents in the last quarter caused by this"

**Strategic cost** (Sriram's question: what does this prevent next quarter?):
- "Can't add multi-currency because amounts are stored as strings"
- "Can't extract this into a separate module because it's coupled to the DB directly"

**Failure cost:** Blast radius if this breaks.
- "No error handling — when it fails, the user gets a 500 with no recovery path"
- "No tests — every change is a gamble"

**Compounding cost:** Is this debt growing?
- "Every new feature adds more coupling to this module"
- "The schema has 3 more misnamed columns after last quarter's pivot"

---

## PHASE 3: PRIORITY SCORING

Score each item on three axes (1–5 each, max 15):

| Axis | 1 | 3 | 5 |
|------|---|---|---|
| Compounding | Stable, not growing | Growing slowly | Gets worse every sprint |
| Strategic blockage | No features blocked | Slows some features | Blocks a key initiative |
| Failure risk | Low blast radius | Medium impact, partial failure | High blast radius, silent failures |

**Score = Compounding + Strategic + Failure**

- 13–15: Repay immediately. Costing more each week.
- 9–12: Schedule in next 1–2 sprints. Slowing you down.
- 5–8: Repay opportunistically (when touching the area anyway).
- 3–4: Document and monitor. Not worth addressing unless it rises.

---

## PHASE 4: PAYOFF SEQUENCE

**Sequencing rules:**

1. **Test and infrastructure debt first.** You can't safely refactor code with no tests. Build the safety net before restructuring.
2. **Highest compounding debt next.** Address what's growing fastest before it requires emergency surgery.
3. **Dependency debt opportunistically.** Upgrade when you're already touching the module.
4. **Design debt last.** Structural refactors are the riskiest. Do them with full test coverage and small steps.

**The strangler fig pattern for large refactors:**
Don't rewrite all at once. Build new structure alongside old. Migrate traffic to the new path. Delete old code only when new path is verified. This is how Zerodha rewrites core systems without downtime.

**Time budgeting:**
- Allocate 20% of sprint capacity to debt repayment when debt is moderate
- Pause new features and dedicate a full sprint when debt is critical
- Never accumulate more than one quarter of unaddressed priority debt

---

## PHASE 5: DEFINITION OF DONE

For each debt item, define what "paid" means. Otherwise repayment becomes endless.

Good done criteria:
- "3 duplicated payment validation functions merged into 1 with unit tests"
- "N+1 query on order loading eliminated; EXPLAIN ANALYZE shows index scan; p99 under 80ms"
- "All 7 TODO comments resolved or converted to tracked issues with owners"

Bad done criteria:
- "Code is cleaner" (subjective)
- "Refactored" (means nothing)

---

## OUTPUT: DEBT.md

```markdown
# DEBT.md

## Module / Area: [name]
Date: [today]

## Executive Summary
- Critical debt (13–15): [count]
- High debt (9–12): [count]
- Medium debt (5–8): [count]
- Estimated sprint cost: [hours/week lost to this debt]

## Inventory

### DEBT-001: [Name]
Category: Design / Code / Test / Infra / Dependency / Schema
Score: N/15 (Compounding X + Strategic Y + Failure Z)
Location: [file:line or module]
Description: [What the debt is]
Carrying cost: [What it costs per sprint]
Strategic blockage: [What it prevents next quarter]
Failure risk: [What happens if it breaks]
Recommendation: Repay now / Next sprint / Opportunistic / Monitor
Done when:
  [ ] [measurable criterion]
  [ ] [test coverage requirement]

### DEBT-002: [Name]
...

## Payoff Sequence

Sprint 1: [DEBT-001, DEBT-003] — reason
Sprint 2: [DEBT-002] — reason
Opportunistic: [DEBT-004, DEBT-005]

## What NOT to fix
[Items scored low — to prevent over-engineering]

## Risks in the Payoff Plan
[What could go wrong and how to mitigate]
```

---

## ANTI-PATTERNS

| Anti-Pattern | What Goes Wrong |
|---|---|
| "We'll clean it up later" | Later compounds. Schedule it now or accept the cost permanently. |
| Rewriting everything at once | High risk, long timeline, scope always expands, often fails |
| Paying debt without tests first | Introducing new bugs while fixing old structure |
| No done criteria | Refactors never finish; engineers keep expanding scope |
| Adding features during refactor | Mixed purpose = mixed outcome |
| Ignoring dependency debt | CVEs become incidents; major version gaps become multi-week projects |

---

## FEEDS INTO

`/modular` — after clearing debt, redraw module boundaries. Debt often reveals coupling that should be broken.
