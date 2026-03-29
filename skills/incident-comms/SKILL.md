---
name: incident-comms
description: "Generate stakeholder communications — status page updates, exec summaries, postmortem drafts. Adapts tone and detail level to the audience: technical for engineering, business impact for executives, user-facing for status pages."
tags: [comms, postmortem]
homepage: https://neubird.ai/skills/incident-comms
metadata: {"neubird":{"emoji":"📣","requires":{"bins":["neubird"]}}}
---

# Incident Communications

Generate stakeholder communications from investigation findings, adapted to audience.

## When to Use

- During active incidents — status page updates for customers
- After resolution — exec summary for leadership
- Post-incident — postmortem draft for engineering
- Escalation — communicating severity change to stakeholders
- Scheduled maintenance — advance notice to affected users

## Communication Types

### Status Page Update (Customer-Facing)

**Tone**: Factual, calm, no jargon. Tell users what's affected and when to expect resolution.

**Template**:
```
Title: [Service] experiencing [degraded performance / partial outage / full outage]

Status: Investigating / Identified / Monitoring / Resolved

[Timestamp] — We are aware of an issue affecting [specific functionality].
[What users experience: e.g., "Some users may see slower load times on the dashboard."]
Our team is actively investigating and we will provide updates every 30 minutes.

Impact: [Which users/regions/features are affected]
Workaround: [If any — e.g., "Refreshing the page may resolve the issue temporarily."]
```

**Rules**:
- Never blame a vendor, team, or individual
- Never include technical details (no stack traces, error codes, or service names)
- State what users experience, not what's broken internally
- Include a workaround if one exists
- Commit to an update cadence (every 30 or 60 minutes)

### Exec Summary (Leadership)

**Tone**: Business impact focused. Duration, user impact, revenue impact, what's being done.

**Template**:
```
Subject: [SEV level] — [Service] incident summary

Duration: [Start time] to [End time] ([X] minutes / ongoing)
Impact: [N users affected, N% error rate, $X estimated revenue impact]
Root Cause: [One sentence, non-technical]
Status: [Resolved / Monitoring / Ongoing]
Next Steps: [What's being done to prevent recurrence]
Owner: [Name/team responsible for follow-up]
```

**Rules**:
- Lead with business impact (users, revenue, SLA)
- One sentence for root cause — no technical deep-dive
- Always include next steps and owner
- Keep under 10 sentences

### Postmortem Draft (Engineering)

**Tone**: Technical, blameless, actionable. Focus on systemic improvements.

**Template**:
```
# Postmortem: [Title]

## Summary
[2-3 sentences: what happened, duration, impact]

## Timeline
| Time (UTC) | Event |
|---|---|
| HH:MM | First alert fired |
| HH:MM | Oncall acknowledged |
| HH:MM | Root cause identified |
| HH:MM | Mitigation applied |
| HH:MM | Full resolution confirmed |

## Root Cause
[Technical explanation of what went wrong and why]

## Impact
- Users affected: [count or percentage]
- Error rate peak: [X%]
- Duration: [X minutes]
- SLO impact: [X% of error budget consumed]

## What Went Well
- [Detection was fast, mitigation was effective, etc.]

## What Went Wrong
- [Detection was slow, runbook was outdated, etc.]

## Action Items
| Action | Owner | Priority | Due Date |
|---|---|---|---|
| [Specific action] | [Name] | P0/P1/P2 | [Date] |

## Lessons Learned
[Systemic insights — not "person X should have done Y"]
```

**Rules**:
- Blameless — describe systems and processes, not individuals
- Every "what went wrong" must have a corresponding action item
- Action items must have an owner and due date
- Include "what went well" — reinforce good practices

## Constraints

### MUST DO
- Adapt language to the audience (no jargon for execs/customers)
- Include timestamps in UTC
- Quantify impact (users, error rate, duration)
- Blameless language in all communications
- Commit to follow-up actions with owners

### MUST NOT DO
- Include speculative root causes in customer communications
- Share internal service names or infrastructure details publicly
- Blame individuals, teams, or vendors in any communication
- Skip the timeline (stakeholders need to understand the sequence)
- Promise specific resolution times unless confident

## Output

The appropriate communication document for the requested audience, ready to send.
