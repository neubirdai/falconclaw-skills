---
name: rca-template
description: "Structured root cause analysis with timeline, blast radius, and corrective actions. Produces investigation reports that identify the specific change, when it took effect, and the mechanism by which it caused the observed failure."
tags: [incident, postmortem]
homepage: https://neubird.ai/skills/rca-template
metadata: {"neubird":{"emoji":"🔬","requires":{"bins":["neubird"]}}}
---

# RCA Template

Structured root cause analysis generation with evidence-backed findings.

## When to Use

- After an incident is resolved or mitigated
- When preparing a postmortem document
- To generate a structured timeline of events
- When leadership needs a formal incident analysis

## Root Cause Standard

A valid root cause must answer three questions:
1. **What changed?** — The specific code deploy, config change, data condition, or external event
2. **When did it take effect?** — The exact timestamp the change started impacting users
3. **How did it cause the failure?** — The mechanical chain from change → system behavior → user impact

If you can't answer all three, the investigation isn't complete.

## RCA Structure

### Title
`[Service] [failure type] due to [root cause] — [date] [time range]`

Example: `Payment API 5xx errors due to connection pool exhaustion from leaked DB connections — 2026-03-28 14:00-14:45 UTC`

### Summary
2-3 sentences covering: what failed, why, how long, who was affected, and whether it's fully resolved.

### Lookback Window
The time range investigated, with start and end timestamps.

### Timeline of Events
Chronological table of key events:
| Time (UTC) | Event |
|---|---|
| 14:00 | Deploy v2.3.1 rolled out to production |
| 14:05 | Connection pool usage begins climbing |
| 14:15 | First PodNotReady alert fires |
| 14:20 | Oncall acknowledges, begins investigation |
| 14:35 | Root cause identified: leaked connections in new DB client |
| 14:40 | Rollback to v2.3.0 initiated |
| 14:45 | Connection pool drains, error rate returns to baseline |

### Incident Description
- Affected services, namespaces, clusters
- Affected users/regions
- Key metrics (peak error rate, latency, duration)

### Root Cause
Detailed explanation addressing the three questions above. Include:
- The specific change (commit hash, config diff, or external event)
- The mechanism (step-by-step chain of causation)
- Why existing safeguards didn't catch it (tests, canary, monitoring)

### Evidence
Supporting data for the root cause claim:
- Log entries with timestamps
- Metric graphs showing the correlation
- Config diffs or commit references
- Alert firing sequence

### Impact Assessment
- **Ugly**: Complete failures, data loss, extended outage
- **Bad**: Degraded service, elevated errors, slower response
- **Monitoring gaps**: What should have caught this earlier

### Connected Findings
Other issues discovered during investigation that aren't the root cause but need attention.

### Corrective Actions
Tiered by urgency:
- **Immediate**: What was done to resolve this specific incident
- **Short-term** (1-2 weeks): Prevent this exact failure from recurring
- **Long-term** (1-3 months): Systemic improvements to prevent this class of failure

Each action must have an **owner** and **due date**.

## Constraints

### MUST DO
- Include evidence for every claim (log lines, metrics, timestamps)
- Answer the three root cause questions explicitly
- Include corrective actions with owners
- Maintain blameless language throughout
- Distinguish between root cause and contributing factors

### MUST NOT DO
- State root cause without evidence
- List symptoms as the root cause ("the service crashed" is not a root cause)
- Skip the mechanism (HOW the change caused the failure)
- Omit corrective actions
- Blame individuals

## Output

A complete RCA document with all sections above, formatted in markdown with:
1. Structured JSON for machine consumption (timeline, sections)
2. Readable markdown for human consumption
3. Corrective actions table with owners and dates
