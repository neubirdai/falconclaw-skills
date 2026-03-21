---
name: deploy-review
description: "Post-deployment health validation — compare key metrics before and after a deploy window."
tags: [deploy, validation]
homepage: https://neubird.ai/skills/deploy-review
metadata: {"neubird":{"emoji":"🚀","requires":{"bins":["neubird"]}}}
---

# Deploy Review

Post-deployment health validation.

## When to Use

- After a production deployment completes
- To validate that a deploy didn't introduce regressions
- When comparing metrics across a deploy window

## What It Compares

- Error rates (before vs after)
- Latency percentiles (p50, p95, p99)
- Resource utilization
- Alert volume
