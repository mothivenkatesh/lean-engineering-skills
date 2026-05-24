---
name: ops-ready
description: "Operational readiness check before deploying anything to production. Enforces: can you observe it, can you roll it back, what breaks it, who fixes it at 3am. Trigger on: 'ready to deploy', 'about to ship', 'going to production', 'launch this', 'before I deploy', 'production checklist', 'is this ready'. AI models write code that works. This skill asks whether the code can be operated by a human who might be woken up at 3am to fix it."
user-invocable: true
argument-hint: "[feature or system being deployed]"
---

# Ops-Ready — Can This Be Operated in Production?

Charity Majors: "No one should be promoted to senior software engineer unless they are proficient in production and know how to own their software by writing it, deploying it, and debugging it in production through the lens of their instrumentation."

AI models write code that passes tests. They have never been paged at 2am. They have never spent 45 minutes finding the right log line. They have never deployed a rollback while users are actively hitting the broken endpoint.

This skill asks the questions that 3am teaches you to ask before shipping.

**The core question:** Can someone who didn't write this code operate it in production without paging you?

If the answer is no — it's not ready.

---

## THE OPS-READY CHECKLIST

Run all five sections. No shortcuts.

---

## SECTION 1: OBSERVABILITY

"If you can't observe it, you can't operate it." — Charity Majors

The four things you must be able to answer about any production system **from logs and metrics, without reading the code:**

1. Is it working right now?
2. When did it start failing?
3. How many users are affected?
4. What specifically is failing?

If you can't answer all four from your observability stack: you're flying blind.

### Logging checklist

- [ ] Every request logs: timestamp, endpoint, user identifier, duration, outcome (success/error)
- [ ] Every error logs: the error message, the full stack trace, the input that caused it
- [ ] Every external API call logs: which service, what was called, duration, response code
- [ ] Every background job logs: when it started, what it processed, how long it took, whether it succeeded
- [ ] Logs are structured (JSON), not free-text — so they can be queried
- [ ] Sensitive data (passwords, tokens, payment details) is never logged

**What bad logging looks like:**
```
ERROR: Something went wrong
INFO: Done
```

**What good logging looks like:**
```json
{"level":"error","timestamp":"2026-05-24T10:30:00Z","endpoint":"/api/payments","user_id":"u_123","error":"Payment gateway timeout","duration_ms":5001,"request_id":"req_abc"}
```

### Metrics checklist (Four Golden Signals)

These four must be instrumented before production. Nothing else is optional:

| Signal | What to measure | Alert threshold |
|---|---|---|
| **Latency** | p50 and p99 response time | Alert if p99 > 2x normal baseline |
| **Error rate** | % of requests returning 5xx | Alert if > 1% |
| **Traffic** | Requests per minute | Alert if drops to 0 (means users can't reach it) |
| **Saturation** | DB connection pool %, memory %, disk % | Alert at 70% (not 100%) |

For indie/bootstrapped scale: these don't require Datadog. Sentry (errors), Uptime Robot (availability), and Postgres `pg_stat_activity` (connection pool) cover 90% of what you need for under $30/month.

### Alerting checklist

- [ ] An alert fires before users report a problem (not after)
- [ ] The alert tells you what broke, not just "something is wrong"
- [ ] The alert links to the relevant logs / dashboard
- [ ] There is an on-call rotation (even if it's just you) — alerts don't go unacked
- [ ] Alerts are tuned to not false-positive (a noisy alert is ignored; silence is dangerous)

---

## SECTION 2: ROLLBACK

Every deployment must have a defined rollback path before it deploys. Not during the incident. Before.

The test: **Can an engineer who didn't write this code roll it back in under 10 minutes, without paging you, using only a written runbook?**

If no: the deploy is riskier than it appears.

### Rollback checklist

- [ ] The rollback procedure is written down before deploying
- [ ] Rolling back the code does not require rolling back the database
- [ ] If there is a database migration, the old code can run against the new schema (additive changes only)
- [ ] Rollback triggers are pre-agreed: "Roll back if error rate exceeds 1%, or p99 latency doubles"
- [ ] The rollback can be executed in one command or one button click

### Database migration safety

The most common cause of "we can't roll back" is a database migration that the old code doesn't understand.

**Safe migrations (old code can run against new schema):**
- Adding a nullable column ✓
- Adding a new table ✓
- Adding an index (`CREATE INDEX CONCURRENTLY`) ✓
- Adding a column with a default (Postgres 11+) ✓

**Unsafe migrations (break the old code):**
- Removing a column ✗ (remove code references first, deploy, then drop)
- Renaming a column ✗ (add new column, migrate data, remove old — 3 deploys)
- Changing a column type ✗ (table rewrite; blocks reads and writes)
- Adding NOT NULL without a default ✗ (old code may not supply the value)

**The two-deploy pattern for column renames:**
```
Deploy 1: Add new column. Write to both old and new. Read from old.
Deploy 2: Read from new. Remove writes to old.
Deploy 3: Drop old column.
```
Never rename in one deploy. It will break the running application.

---

## SECTION 3: FAILURE MODE DOCUMENTATION

Before deploying: document what breaks and what the recovery procedure is. This is not optional — this is the difference between a 20-minute incident and a 4-hour incident.

### For each external dependency, answer:

**If [service] goes down:**
- What user-facing behavior degrades?
- Does the system fail hard or degrade gracefully?
- Is there a fallback? (cached data, default behavior, error message)
- What's the recovery procedure when [service] comes back?

**If the database goes down:**
- What happens to in-flight requests?
- Does the application restart cleanly when the database returns?
- Are there any write-loss scenarios? (in-memory queues, buffered writes)

**If this deployment itself breaks:**
- What's the symptom that triggers rollback?
- What's the rollback command?
- Who is responsible for executing it?

### The 3am scenario

Write this out explicitly: it is 3 AM. You are asleep. This system starts failing. An alert fires. The person who picks it up has not worked on this feature.

What do they see in the logs?
What do they search for?
What steps do they take?
Where is the runbook?

If you cannot answer these questions, the system is not ops-ready.

---

## SECTION 4: DEPENDENCY AUDIT

Every dependency is a dependency on someone else's reliability.

- [ ] Every external HTTP call has a timeout set (not using the library default, which is often infinite)
- [ ] Every external call has a circuit breaker or at minimum a fallback behavior
- [ ] Third-party dependencies have been checked for known vulnerabilities (`npm audit`, `pip-audit`)
- [ ] No dependency is pinned to `latest` in production (frozen version = reproducible builds)
- [ ] If a third-party service fails, the core user flow still works (degraded is better than down)

**Timeout rule:** Set timeouts explicitly. Always. Library defaults are often 30 seconds or infinite. A 30-second hung request blocks a thread for 30 seconds. At 100 concurrent users, this exhausts your thread pool.

```
External API timeout: 500ms - 2000ms depending on expected latency
Database query timeout: 5000ms for complex queries, 1000ms for simple reads
HTTP connection timeout: 5000ms
```

**Cascading failure prevention:** If service A calls service B calls service C, a timeout in service C causes latency in service B, which causes latency in service A, which causes users to see slow responses everywhere. Each service boundary needs a timeout. Not the same timeout — appropriate timeouts for what each call needs.

---

## SECTION 5: DEPLOY PROCEDURE

The deploy itself is a risk surface.

- [ ] The deploy can be done with one command (not 12 manual steps)
- [ ] Environment variables in production match what the code expects (no silent mismatches)
- [ ] The production environment has been tested with a dry run or staging equivalent
- [ ] There is a smoke test: 3-5 automated checks that confirm the critical paths work after deploy
- [ ] The deploy is done during low-traffic hours (not on a Friday afternoon)

**Smoke test examples:**
- `curl https://api.example.com/health` returns 200
- The main user-facing page loads and returns data
- An authenticated endpoint returns the correct response for a test user
- The database has at least 1 record in the primary table (not empty)

**Environment variable drift:** The most common deploy failure after code is correct. Add a startup check:

```typescript
// At application startup — fail loudly, not silently
const required = ['DATABASE_URL', 'API_SECRET', 'STRAPI_URL']
for (const key of required) {
  if (!process.env[key]) {
    throw new Error(`Missing required environment variable: ${key}`)
  }
}
```

If the application starts without required variables, it fails in ways that are hard to trace. Make it fail immediately with a clear message.

---

## INDIE/BOOTSTRAPPED OPERATIONAL STACK

You don't need Datadog at $100/month. You don't need PagerDuty at $50/month per user. You need:

| Need | Tool | Cost |
|---|---|---|
| Error tracking | Sentry (free tier: 5K errors/month) | $0 |
| Uptime monitoring | Uptime Robot (free: 50 monitors, 5-min intervals) | $0 |
| Log access | Vercel/Railway built-in logs (structured, searchable) | $0 |
| DB monitoring | pg_stat_activity (built into Postgres) | $0 |
| Alerting | Uptime Robot → email/Slack webhook | $0 |

Total: $0/month for an ops stack that catches 90% of production failures before users report them.

Add Sentry's $26/month plan when error volume exceeds the free tier. Nothing else until revenue justifies it.

---

## THE OPS-READY VERDICT

After running all sections:

**READY:** All sections pass. Deploy with confidence.

**READY WITH CONDITIONS:** 1-2 items missing that are low-risk. Document them as known gaps and prioritize in the next sprint.

**NOT READY:** Critical gaps in observability, rollback, or failure mode documentation. Fix before deploying.

---

## OUTPUT FORMAT

```
## System Being Deployed
[What is being deployed — feature, version, infrastructure change]

## Section 1: Observability
✓/✗ Logging covers: requests, errors, external calls, background jobs
✓/✗ Four Golden Signals instrumented: latency, errors, traffic, saturation
✓/✗ Alert fires before user reports (not after)
Gaps: [list]

## Section 2: Rollback
✓/✗ Rollback procedure is written
✓/✗ Database migrations are backward-compatible
✓/✗ Rollback triggers are pre-defined
Rollback command: [exact command]
Rollback triggers: [exact thresholds]

## Section 3: Failure Mode Documentation
[For each external dependency: what breaks, what degrades, how to recover]
3am runbook location: [link or "MISSING"]

## Section 4: Dependency Audit
✓/✗ All external calls have timeouts set
✓/✗ All dependencies have fallback behavior
✓/✗ No known CVEs in dependencies
Gaps: [list]

## Section 5: Deploy Procedure
✓/✗ One-command deploy
✓/✗ Env vars validated at startup
✓/✗ Smoke tests defined
Smoke test commands: [list]

## Verdict
READY / READY WITH CONDITIONS / NOT READY

## Conditions or Blockers
[What must be addressed before or immediately after deploy]
```
