---
name: weekend-check
description: "Pre-oncall handoff risk assessment — recent changes, capacity trends, fragile spots, expiring certificates, error budget status, and upcoming maintenance. Produces a prioritized risk summary so oncall knows what to watch."
tags: [oncall, handoff]
homepage: https://neubird.ai/skills/weekend-check
metadata: {"neubird":{"emoji":"🌍","requires":{"bins":["neubird"]}}}
---

# Weekend Check

Risk assessment for oncall handoffs — what could break and what to watch.

## When to Use

- Before starting an oncall shift
- Friday afternoon pre-weekend review
- Any handoff between oncall rotations
- Before a holiday or reduced-staffing period

## Core Workflow

1. **Recent changes** — What was deployed or modified in the last 48 hours
2. **Active issues** — Any ongoing incidents or elevated error rates
3. **Capacity** — Services approaching resource thresholds
4. **Expiring resources** — Certificates, API keys, or licenses expiring soon
5. **Error budgets** — Services with depleted or rapidly burning budgets
6. **Upcoming events** — Scheduled maintenance, planned deploys, traffic events
7. **Fragile spots** — Services with known flakiness or recent near-misses

## What to Check

### Recent Changes (last 48h)
Query deployment logs, git history, and infrastructure change records:
- Which services were deployed and by whom
- Any config changes (feature flags, environment variables, scaling policies)
- Infrastructure modifications (Terraform applies, security group changes)
- Database migrations or schema changes
- **Red flag**: Any change that was rolled back (indicates instability)

### Active Issues
- Alerts currently firing or recently resolved
- Error rates above baseline for any service
- PagerDuty/Opsgenie incidents from the outgoing shift
- Known issues that are being monitored but not yet resolved

### Capacity Trends
For each production service, check:
- CPU utilization trend (is it approaching 80%?)
- Memory utilization (any steady upward trend = potential leak)
- Disk usage on databases and persistent volumes
- Connection pool utilization
- **Red flag**: Any metric that will cross a threshold within 72 hours at current growth rate

### Expiring Resources
- TLS certificates expiring within 14 days
- API keys or tokens with known expiry dates
- Reserved instance or savings plan expirations
- License renewals due

### Error Budget Status
For SLO-tracked services:
- Current error budget remaining (percentage)
- Burn rate over the last 24 hours
- **Red flag**: Any service below 20% remaining budget

### Upcoming Events
- Scheduled maintenance windows during the shift
- Planned production deployments
- Marketing campaigns or product launches that increase traffic
- Vendor maintenance notifications (AWS, GCP, Datadog, etc.)

### Fragile Spots
Services that have been problematic recently:
- Services with >3 incidents in the last 30 days
- Services with recent scaling events
- Services with known technical debt or deferred fixes
- Dependencies with known reliability issues

## Risk Scoring

For each finding, assess:
- **Likelihood**: How likely is this to cause an incident during the shift? (High/Medium/Low)
- **Impact**: If it happens, how severe? (SEV1/SEV2/SEV3)
- **Preparation**: Is there a runbook? Is oncall familiar with this service?

Priority = Likelihood × Impact ÷ Preparation

## Output

A prioritized briefing document:
1. **Critical watch items** (high likelihood + high impact) — top of document
2. **Recent changes** that could have residual effects
3. **Capacity warnings** with projected time-to-threshold
4. **Expiring resources** requiring action
5. **Upcoming events** requiring awareness
6. **Runbook links** for each watch item
7. **Escalation contacts** for the shift
