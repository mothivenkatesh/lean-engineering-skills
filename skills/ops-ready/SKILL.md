---
name: ops-ready
description: "Production readiness check before any deploy. Confirms: can you observe it, can you roll it back, what breaks it, who fixes it at 3am. Trigger on: 'ready to deploy', 'about to ship', 'going to production', 'launch this', 'before I deploy', 'production checklist'. AI models write code that passes tests. This skill asks whether the code can be operated by a human who might be woken up at 3am to fix it."
user-invocable: true
argument-hint: "[feature or system being deployed]"
---

# Ops-Ready — Can This Be Operated in Production?

## INPUT

Consumes: `RISK.md` from `/premortem` and `PERF.md` from `/hot-path`

Uses: failure modes and observability gaps from RISK.md; latency baseline from PERF.md for golden signal thresholds.

---

## THE LOOP

**TRIGGER:** Code is tested and reviewed. You're about to deploy.

**CYCLE:** Run all five sections. Mark every gap. Don't deploy until critical gaps are closed.

**EXIT CONDITION:** All five sections pass or gaps are explicitly accepted with a plan. DEPLOY.md is written.

**The core question:** Can someone who didn't write this code fix it at 3am using only a runbook?

If no: it's not ready.

---

## HARD CONSTRAINTS
- **Required:** Code is tested (TEST_REPORT.md) and hot-path reviewed (PERF.md). Or explicit written acknowledgment these were skipped.
- **Refuse if:** No rollback procedure is written. A rollback procedure is not optional — write it before the deploy checklist proceeds.
- **Refuse if:** Any database migration is not classified against the safe/unsafe migration table. "I think it's safe" is not a classification.
- **All 5 sections are mandatory.** A partial ops-ready check is not ops-ready.
- **Never mark NOT READY as READY.** If critical gaps exist, the verdict is NOT READY regardless of timeline pressure.

Charity Majors: "No one should be promoted to senior software engineer unless they are proficient in production and know how to own their software by writing it, deploying it, and debugging it in production through the lens of their instrumentation."

---

## SECTION 1: OBSERVABILITY

"If you can't observe it, you can't operate it."

Four questions you must be able to answer from logs and metrics, without reading code:

1. Is it working right now?
2. When did it start failing?
3. How many users are affected?
4. What specifically is failing?

If you can't answer all four from your observability stack: you're flying blind.

### Logging checklist

- [ ] Every request logs: timestamp, endpoint, user identifier, duration, outcome
- [ ] Every error logs: error message, full stack trace, input that caused it
- [ ] Every external API call logs: which service, what was called, duration, response code
- [ ] Every background job logs: when it started, what it processed, how long, whether it succeeded
- [ ] Logs are structured (JSON), not free-text
- [ ] No sensitive data logged (passwords, tokens, payment details)

Bad logging:
```
ERROR: Something went wrong
INFO: Done
```

Good logging:
```json
{"level":"error","ts":"2026-05-24T10:30:00Z","endpoint":"/api/payments","user_id":"u_123","error":"Payment gateway timeout","duration_ms":5001,"request_id":"req_abc"}
```

### Four Golden Signals (from PERF.md baseline)

| Signal | What to measure | Alert threshold |
|--------|-----------------|-----------------|
| Latency | p50 and p99 response time | Alert if p99 > 2x normal baseline |
| Error rate | % of requests returning 5xx | Alert if > 1% |
| Traffic | Requests per minute | Alert if drops to 0 |
| Saturation | DB connection pool %, memory %, disk % | Alert at 70%, not 100% |

### Alerting checklist

- [ ] An alert fires before users report a problem
- [ ] The alert tells you what broke, not just "something is wrong"
- [ ] The alert links to relevant logs or dashboard
- [ ] Alerts are tuned to not false-positive (noisy alerts get ignored)

---

## SECTION 2: ROLLBACK

Every deployment must have a defined rollback path before it deploys. Not during the incident. Before.

**The test:** Can an engineer who didn't write this code roll it back in under 10 minutes without paging you?

### Rollback checklist

- [ ] The rollback procedure is written down before deploying
- [ ] Rolling back the code does not require rolling back the database
- [ ] If there is a database migration, the old code can run against the new schema (additive changes only)
- [ ] Rollback triggers are pre-agreed: "Roll back if error rate exceeds 1%, or p99 latency doubles"
- [ ] The rollback is one command or one button click

### Database migration safety

The most common reason "we can't roll back" is a migration the old code doesn't understand.

Safe (old code runs against new schema):
- Adding a nullable column
- Adding a new table
- `CREATE INDEX CONCURRENTLY`
- Adding a column with a default (Postgres 11+)

Unsafe (old code breaks):
- Removing a column (remove code references first, deploy, then drop — 2 deploys)
- Renaming a column (add new column, migrate data, remove old — 3 deploys)
- Changing a column type (table rewrite; blocks reads and writes)
- Adding NOT NULL without a default (old code may not supply the value)

Never rename a column in one deploy. It will break the running application.

---

## SECTION 3: FAILURE MODE DOCUMENTATION

From RISK.md critical findings — document recovery procedures before deploying.

For each external dependency:
- What user-facing behavior degrades if it goes down?
- Does the system fail hard or degrade gracefully?
- Is there a fallback?
- What's the recovery procedure when it comes back?

For the database going down:
- What happens to in-flight requests?
- Does the application restart cleanly when the database returns?
- Are there write-loss scenarios?

### The 3am scenario

Write this out explicitly: it is 3am. You are asleep. This system starts failing. An alert fires. The person who picks it up has not worked on this feature.

What do they see in the logs?
What do they search for?
What steps do they take?
Where is the runbook?

If you cannot answer these questions, the system is not ops-ready.

---

## SECTION 4: DEPENDENCY AUDIT

Every dependency is a dependency on someone else's reliability.

- [ ] Every external HTTP call has a timeout set explicitly (not the library default)
- [ ] Every external call has fallback behavior when it fails
- [ ] Third-party dependencies checked for CVEs (`npm audit`, `pip-audit`)
- [ ] No dependency pinned to `latest` in production (frozen version = reproducible builds)
- [ ] If a third-party service fails, the core user flow still works (degraded > down)

**Timeout rule:** Set explicitly. Always. Library defaults are often 30 seconds or infinite. A 30-second hung request blocks a thread. At 100 concurrent users, this exhausts the thread pool.

```
External API:     500ms–2000ms depending on expected latency
Database queries: 5000ms complex, 1000ms simple reads
HTTP connection:  5000ms
```

**Cascading failure:** If service A calls B calls C, a timeout in C causes latency in B which causes latency in A which causes users to see slow responses everywhere. Each service boundary needs its own timeout.

---

## SECTION 5: DEPLOY PROCEDURE

The deploy is itself a risk surface.

- [ ] Deploy is one command, not 12 manual steps
- [ ] Environment variables in production match what the code expects
- [ ] Required env vars validated at startup — fail loudly, not silently
- [ ] Smoke tests defined: 3–5 checks confirming critical paths work after deploy
- [ ] Deploy during low-traffic hours (not Friday afternoon)

### Startup env var validation
```typescript
const required = ['DATABASE_URL', 'API_SECRET', 'STRAPI_URL']
for (const key of required) {
  if (!process.env[key]) throw new Error(`Missing required env var: ${key}`)
}
```

Fail immediately with a clear message. Silent startup with missing vars fails in ways that are hard to trace.

### Smoke tests
```bash
curl https://api.example.com/health           # returns 200
curl https://api.example.com/api/articles     # returns data, not empty
# + authenticated endpoint check for a test user
```

---

## INDIE/BOOTSTRAPPED OPS STACK

You don't need Datadog or PagerDuty. You need:

| Need | Tool | Cost |
|------|------|------|
| Error tracking | Sentry (free: 5K errors/month) | $0 |
| Uptime monitoring | Uptime Robot (free: 50 monitors) | $0 |
| Logs | Vercel/Railway built-in logs | $0 |
| DB monitoring | `pg_stat_activity` (built into Postgres) | $0 |
| Alerting | Uptime Robot → email/Slack webhook | $0 |

$0/month catches 90% of production failures before users report them. Add Sentry's $26/month plan when error volume exceeds the free tier.

---

## OUTPUT: DEPLOY.md

```markdown
# DEPLOY.md

## System Being Deployed
[Feature, version, or infrastructure change]

## Section 1: Observability
✓/✗ Logging: requests, errors, external calls, background jobs
✓/✗ Four Golden Signals: latency, errors, traffic, saturation
✓/✗ Alert fires before user reports
Gaps: [list]

## Section 2: Rollback
✓/✗ Rollback procedure is written
✓/✗ Database migrations are backward-compatible
✓/✗ Rollback triggers are pre-defined
Rollback command: [exact command]
Rollback trigger: [e.g., "error rate > 1% or p99 > 2x baseline for 5 minutes"]

## Section 3: Failure Mode Documentation
[Each external dependency: what breaks, what degrades, recovery]
3am runbook: [location or MISSING]

## Section 4: Dependency Audit
✓/✗ All external calls have explicit timeouts
✓/✗ Fallback behavior defined for all dependencies
✓/✗ No known CVEs
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

---

## FEEDS INTO

`/diagnose` — DEPLOY.md runbook is the starting point for incident response. When something breaks, the diagnose skill uses the runbook to orient.
