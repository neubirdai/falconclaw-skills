---
name: deploy-gate
description: "Pre-deploy safety check — assess whether it is safe to deploy one or more services right now."
tags: [deploy, safety, release]
homepage: https://neubird.ai/skills/deploy-gate
metadata: {"neubird":{"emoji":"🚦","requires":{"bins":["neubird"]}}}
---

# Deploy Gate

Pre-deploy safety gate for one or more services. Runs a structured 6-phase assessment and returns a clear **GO / CAUTION / NO-GO** verdict before you push to production.

## When to use

Use this skill before any production deploy — single service or a coordinated multi-service release. It is especially important when:

- Deploying during or after an active incident
- Stacking a second deploy on top of a recent one
- A service's SLO error budget is low
- You want a fast, automated pre-flight check without manually querying dashboards

## How it works

### 1. Resolve the deploy target

The skill accepts a group name (e.g. `neubird`) or an explicit list of services. Group names are expanded to their member services before assessment begins.

| Group | Services |
|-------|----------|
| neubird | ace, falcon, hawkeye-ui, raven, raven-crawler, raven-fdw-bridge, raven-store |

### 2. Per-service assessment (6 phases)

Each service is evaluated independently across six phases:

| Phase | What is checked | Hard stop condition |
|-------|----------------|---------------------|
| 1 — Active incidents | Open SEV1/SEV2 on target or direct dependencies (last 24 h) | Any open SEV1/SEV2 → NO-GO |
| 2 — Service health | Error rate, p99 latency, request volume vs. prior 1 h baseline | Error rate > 1% or latency > 1.5× baseline → flag |
| 3 — Downstream health | Error rate and latency of direct dependencies (last 30 min) | Dependency > 1% error rate or open incident → blocker |
| 4 — Recent changes | Deploys, migrations, config changes in last 24 h | Unvalidated deploy < 2 h old → NO-GO |
| 5 — Error budget | 30-day rolling SLO budget remaining | < 5% remaining → NO-GO (unless this deploy is the fix) |
| 6 — Capacity | CPU/memory headroom, DB connection pool utilization | < 30% headroom on any resource → CAUTION |

### 3. Verdict rules

- **🟢 GO** — all phases clear
- **🟡 CAUTION** — risky but not blocked; specific condition to watch is stated
- **🔴 NO-GO** — deploy blocked; specific blocker is stated
- If **any** service is NO-GO, the **combined verdict is NO-GO**
- If any service is CAUTION and none are NO-GO, the combined verdict is CAUTION

### 4. Output

A per-service verdict block followed by a combined verdict:

```
SERVICE: <name>
VERDICT: 🟢 GO | 🟡 CAUTION | 🔴 NO-GO
REASON: <one sentence>
BLOCKERS:
- <specific issue>
```

```
COMBINED DEPLOY VERDICT: 🟢 GO | 🟡 CAUTION | 🔴 NO-GO

REASON: <one sentence>

BLOCKERS:
- <service>: <specific issue>

CONDITIONS TO WATCH:
- <1-3 critical signals to monitor in the first 15 minutes post-deploy>

ROLLBACK TRIGGER:
- Roll back if <specific metric> on <service> exceeds <specific threshold> within <specific time window>
```

## Example usage

```
/deploy-gate neubird
/deploy-gate falcon, raven
/deploy-gate ace
```

## Notes

- Rollback triggers are always specific (metric, threshold, time window) — vague triggers like "if error rate increases" are not produced
- If data is missing for a phase (e.g. no SLO table), that gap is stated explicitly — it is not treated as healthy
- FDW constraints apply: no `UNION`, no subqueries, no `BETWEEN`, no `NOW()` — literal timestamps are used throughout
