---
name: sev-hunt
description: "Hunt for active SEVs and brewing incidents across all connected data sources. Use when: (1) proactive incident sweep needed, (2) pre-oncall handoff, (3) something feels off but no alert has fired."
tags: [incident, proactive]
homepage: https://neubird.ai/skills/sev-hunt
metadata: {"neubird":{"emoji":"🚨","requires":{"bins":["neubird"]}}}
---

# SEV Hunt

Proactive sweep for active and brewing incidents across your infrastructure.

## When to Use

- Before oncall handoff — get a full picture of what's happening
- When something feels off but no alert has fired yet
- Periodic proactive sweeps during quiet periods

## What It Does

1. Queries all connected monitoring data sources (Datadog, PagerDuty, CloudWatch, etc.)
2. Correlates alerts across services to identify related incidents
3. Checks for anomalous patterns that haven't triggered alerts yet
4. Produces a prioritized list of findings with severity estimates

## Output

- List of active/suspected incidents ranked by severity
- Affected services and blast radius for each
- Recommended next steps (investigate, monitor, escalate)
