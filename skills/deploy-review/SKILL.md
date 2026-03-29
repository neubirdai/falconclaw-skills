---
name: deploy-review
description: "Post-deployment health validation — compare key metrics before and after a deploy window to detect regressions in latency, error rate, resource usage, and throughput."
tags: [deploy, validation]
homepage: https://neubird.ai/skills/deploy-review
metadata: {"neubird":{"emoji":"🚀","requires":{"bins":["neubird"]}}}
---

# Deploy Review

Post-deployment health validation — detect regressions by comparing metrics before and after a deploy.

## When to Use

- After a production deployment completes
- To validate that a deploy didn't introduce regressions
- When comparing metrics across a deploy window
- Before marking a canary deployment as promoted
- When oncall reports "something feels different" after a deploy

## Core Workflow

1. **Identify the deploy window** — Exact start and end time of the deployment
2. **Define comparison windows** — Typically 1h before deploy vs. 1h after deploy
3. **Query key metrics** — Error rate, latency percentiles, throughput, resource usage
4. **Compare** — Statistical comparison (not just eyeballing)
5. **Assess** — Regression detected or deployment healthy
6. **Recommend** — Continue monitoring, roll back, or investigate further

## Key Metrics to Compare

### Error Rate
- HTTP 5xx rate: before vs. after
- Application-level error rate (logged errors per minute)
- Upstream dependency error rate (did the deploy break a caller?)
- Threshold: >2x increase = regression

### Latency
- p50, p95, p99 response time per endpoint
- Compare the same endpoints, not aggregate
- Threshold: >20% increase in p99 = investigate

### Throughput
- Requests per second: should be stable (unless deploy coincided with traffic change)
- If throughput dropped, check if clients are receiving errors and retrying less

### Resource Usage
- CPU utilization: significant increase may indicate inefficient code
- Memory usage: step increase may indicate a leak introduced
- Pod restart count: any restarts after deploy = immediate concern
- GC pause time (Java/Go): increased pause = allocation pressure

### Business Metrics
- Conversion rate, checkout completion, sign-ups (if available)
- These are the ultimate "did we break anything" signal

## Comparison Methodology

For each metric:
1. Query the **baseline** window (1h before deploy, same day-of-week if possible)
2. Query the **post-deploy** window (1h after deploy stabilizes)
3. Calculate the **delta**: `(post - pre) / pre * 100%`
4. Apply **threshold**: flag if delta exceeds acceptable range
5. **Exclude noise**: filter out one-time spikes, batch jobs, and known anomalies

## Assessment Framework

| Metric | No Regression | Warning | Regression |
|---|---|---|---|
| Error rate | <1.1x baseline | 1.1-2x baseline | >2x baseline |
| p99 latency | <1.1x baseline | 1.1-1.5x baseline | >1.5x baseline |
| CPU usage | <1.2x baseline | 1.2-1.5x baseline | >1.5x baseline |
| Memory | Stable | Slowly growing | Step increase |
| Pod restarts | 0 | 1-2 | >2 |

## Constraints

### MUST DO
- Compare the same time-of-day (avoid comparing peak vs. off-peak)
- Check per-endpoint metrics, not just service-level aggregates
- Include resource metrics, not just request metrics
- Check for delayed regressions (memory leaks may take hours to show)
- Verify the deploy actually rolled out (check running image tag)

### MUST NOT DO
- Compare against a different day without accounting for traffic patterns
- Rely only on average latency (p50 hides tail latency regressions)
- Skip checking upstream callers (your deploy may break your callers)
- Declare "no regression" after only 5 minutes

## Output

1. Deploy metadata (service, version, timestamp, deployer)
2. Metric comparison table (before vs. after with delta percentage)
3. Assessment: healthy / warning / regression detected
4. If regression: affected endpoints and recommended action (monitor / rollback)
5. If healthy: recommended monitoring period before closing
