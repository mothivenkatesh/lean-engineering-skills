---
name: debt-audit
description: "Technical debt audit and payoff planning. Use when code is getting harder to change, velocity is slowing, engineers avoid touching certain areas, or before planning a major refactor. Trigger on: 'refactor this', 'this codebase is a mess', 'it's getting harder to ship', 'before we add X we need to clean up', 'tech debt', 'code health', 'we keep breaking things'. Debt compounds exponentially — the right time to audit is before it becomes emergency surgery."
user-invocable: true
argument-hint: "[codebase, module, or area to audit]"
---

# Debt Audit — Technical Debt Inventory and Payoff Plan

Kailash Nadh: "Technical debt is exponential. Debt creates conditions for more debt. If you never pause to clear technical debt, it grows exponentially and the system becomes a burden. We have rewritten core systems multiple times. Velocity increased after each rewrite."

Sriram Velamur: "Debt is a tool. The strategic question is: what does this debt prevent us from doing next quarter?"

The discipline: **Debt isn't inherently bad. Unexamined debt is.**

Taking debt to ship faster is a valid decision. Not knowing what debt you've taken, or what it costs to carry, is negligence. This audit makes debt visible so it can be managed.

---

## THE AUDIT FRAMEWORK

```
1. INVENTORY   → Find and classify all debt in the area
2. COST        → Quantify what each debt costs to carry
3. PRIORITY    → Rank by compounding risk and strategic impact
4. PLAN        → Build the payoff sequence in dependency order
5. BOUNDARIES  → Define what "done" looks like for each item
```

---

## PHASE 1: DEBT INVENTORY

Read the code before classifying it. Don't inventory from memory or assumptions.

### Debt Categories

**Design Debt** — Wrong abstraction or structure for current requirements
- Classes/modules that are doing too many things (violated SRP)
- Abstractions that made sense 12 months ago but now fight every new feature
- Modules that are tightly coupled when they should be independent
- Patterns copied from external sources that don't fit this domain
- Missing abstractions — duplicated logic spread across 5 files

**Code Debt** — Implementation quality issues
- Functions longer than 50 lines (usually doing more than one thing)
- Unclear naming (what does `handleData()` do?)
- Deep nesting (> 3 levels signals missing extraction)
- Commented-out code (is this needed? why was it removed?)
- Magic numbers and strings (what does `status === 3` mean?)
- TODO/FIXME/HACK comments without resolution dates

**Test Debt** — Missing or misleading test coverage
- No tests for critical paths
- Tests that test implementation, not behavior (break on refactor)
- Tests with no assertions (only check that code runs, not what it returns)
- Test setup so complex the test is hard to understand
- Flaky tests (intermittent failures erode trust in the entire suite)

**Infrastructure Debt** — Operational risk
- Undocumented environment variables and configuration
- Manual deployment steps not in CI/CD
- No health checks or readiness probes
- Logs that don't include enough context to debug production issues
- Missing or stale runbooks

**Dependency Debt** — External risk
- Dependencies with known CVEs (check with `npm audit`, `pip-audit`, `bundle audit`)
- Dependencies on abandoned or unmaintained libraries
- Dependency version ranges so wide they include breaking changes
- Transitive dependencies with conflicts

**Schema / Data Debt** — Structural risk
- Columns whose names no longer match their use
- Nullable columns that should never be null (no constraint enforced)
- Missing indexes on filtered/sorted columns
- Tables that have grown too large without partitioning
- Data that requires code to interpret (enum values stored as magic integers)

---

## PHASE 2: COST ANALYSIS

For each debt item, answer:

**Carrying cost** — What does this debt cost every sprint?
- "Every time we touch this module, we spend 2 hours understanding it before we can change it"
- "We've had 3 production incidents caused by this in the last quarter"
- "New engineers take 2 weeks to become productive in this area instead of 3 days"

**Strategic cost** — What does this debt block?
Sriram's question: *What does this debt prevent us from doing next quarter?*
- "We can't add multi-currency support because amounts are stored as strings"
- "We can't extract this into a microservice because it's tightly coupled to the monolith's DB"
- "We can't increase test coverage because the code has no seams for testing"

**Failure cost** — What's the blast radius if this fails?
- "This has no error handling — when it fails, the user gets a 500 with no recovery path"
- "This module has no tests — every change is a gamble"
- "This dependency is 4 major versions behind — the upgrade is now a multi-day project"

**Compounding cost** — Is this debt growing?
- "Every new feature in this area adds more coupling"
- "The test suite is getting slower as we add more flaky tests"
- "The schema has 3 more columns that are now misnamed after last quarter's pivot"

---

## PHASE 3: PRIORITY SCORING

Score each debt item on three axes (1-5 each, 15 = highest priority):

| Axis | 1 | 3 | 5 |
|------|---|---|---|
| **Compounding speed** | Stable, not growing | Growing slowly | Growing fast — gets worse every sprint |
| **Strategic blockage** | No features blocked | Slows some features | Blocks a key initiative |
| **Failure risk** | Low blast radius, graceful degradation | Medium impact, partial failure | High blast radius, silent failures |

**Priority = Compounding + Strategic + Failure**

Score 13-15: Repay immediately. This is costing you more each week.
Score 9-12: Schedule in next 1-2 sprints. It's slowing you down.
Score 5-8: Track and repay opportunistically (when touching the area anyway).
Score 3-4: Document and monitor. Not worth addressing unless it rises.

---

## PHASE 4: PAYOFF PLAN

Sequence the debt repayment. Not all debt should be paid off at once — that's a rewrite, which fails.

**Sequencing rules:**

1. **Infrastructure and test debt first.** You can't safely refactor code with no tests. Build the safety net before restructuring.

2. **Highest compounding debt next.** Address debt that's growing fastest before it requires emergency surgery.

3. **Dependency debt opportunistically.** Upgrade dependencies when you're already touching the relevant module.

4. **Design debt last.** Structural refactors are the riskiest. Do them with full test coverage and small steps.

**The strangler fig pattern for large refactors:**
Don't rewrite all at once. Build the new structure alongside the old. Migrate traffic to the new path. Delete the old code only when the new path is fully verified. This is how Zerodha rewrites core systems without downtime.

**Time budgeting:**
Nadh's approach: when debt is mounting, pause feature development and address it. The cadence that works:
- Identify debt in every retrospective
- Allocate 20% of sprint capacity to debt repayment when debt is moderate
- Pause new features and dedicate a full sprint when debt is critical
- Never accumulate more than one quarter of unaddressed priority debt

---

## PHASE 5: DEFINITION OF DONE

For each debt item, define what "paid" looks like. Otherwise debt repayment becomes endless.

```
Debt item: [name]
Done when:
  [ ] [Measurable criterion 1]
  [ ] [Measurable criterion 2]
  [ ] [Test coverage requirement]
  [ ] [No regression in existing behavior]
```

Examples of good done criteria:
- "All 3 duplicated payment validation functions merged into 1 shared function with unit tests"
- "N+1 query on order loading eliminated; EXPLAIN ANALYZE shows index scan; p99 latency under 80ms"
- "All 7 TODO comments resolved or converted to tracked issues with owners"

Bad done criteria:
- "Code is cleaner" (subjective, unmeasurable)
- "Refactored" (what does that mean?)

---

## DEBT AUDIT REPORT FORMAT

```
# Technical Debt Audit — [Module/Area]
Date: [today]
Auditor: [name]

## Executive Summary
- Critical debt (score 13-15): [count]
- High debt (score 9-12): [count]
- Medium debt (score 5-8): [count]
- Total estimated sprint cost: [hours/week currently lost to this debt]

## Inventory

### [DEBT-001] [Name]
Category: Design / Code / Test / Infra / Dependency / Schema
Score: [N]/15  (Compounding [X] + Strategic [Y] + Failure [Z])
Location: [file:line or module]
Description: [What the debt is]
Carrying cost: [What it costs per sprint]
Strategic blockage: [What it prevents]
Failure risk: [What happens if it breaks]
Recommendation: [Repay now / Next sprint / Opportunistic / Monitor]
Done criteria:
  [ ] [criterion 1]
  [ ] [criterion 2]

### [DEBT-002] [Name]
...

## Payoff Sequence

Sprint 1: [DEBT-XXX, DEBT-YYY] — [rationale]
Sprint 2: [DEBT-ZZZ] — [rationale]
Opportunistic: [DEBT-AAA, DEBT-BBB]

## What NOT to fix
[Items scored low and why — to prevent over-engineering]

## Risks in the Payoff Plan
[What could go wrong and how to mitigate]
```

---

## ANTI-PATTERNS IN DEBT MANAGEMENT

| Anti-Pattern | What Goes Wrong |
|---|---|
| "We'll clean it up later" | Later compounds. Schedule it now or accept the cost permanently. |
| Rewriting everything at once | High risk, long timeline, scope always expands, often fails |
| Paying debt without tests first | You introduce new bugs while fixing old structure |
| Debt repayment without done criteria | Refactors never finish; engineers keep expanding scope |
| Adding features during refactor | Mixed purpose = mixed outcome; do one or the other |
| Ignoring dependency debt | CVEs become incidents; major version gaps become multi-week projects |
| No debt tracking | Invisible debt is the most dangerous; it grows silently |
