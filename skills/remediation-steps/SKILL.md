---
name: remediation-steps
description: "Generate tiered remediation recommendations — verify, mitigate, high-risk — each with rollback plans, approval guidance, and expected outcome. Use after root cause analysis identifies the problem, before executing any corrective action."
tags: [remediation, safety]
homepage: https://neubird.ai/skills/remediation-steps
metadata: {"neubird":{"emoji":"🛡️","requires":{"bins":["neubird"]}}}
---

# Remediation Steps

Tiered remediation recommendations with rollback plans and safety controls.

## When to Use

- After root cause analysis identifies the problem
- When you need structured remediation options with risk levels
- Before executing any corrective action in production
- When oncall needs guidance on what to try first vs. escalate

## Tier Framework

### Tier 1: Verify (Safe — No Approval Needed)
Low-risk diagnostic actions to confirm the root cause and gather more context. These commands are read-only and cannot cause additional impact.

**Characteristics**:
- Read-only operations (queries, logs, describe, get)
- No state changes
- Safe to run at any time
- Confirms or refutes the root cause hypothesis

**Examples**:
- Check pod status and events
- Query metrics for the affected time window
- Read application logs for error patterns
- Describe the resource configuration
- Check recent deploys and config changes

**Template**:
```
## Tier 1: Verify

### Step 1.1: [What to check]
Command: `kubectl get pods -n production -l app=my-service`
Expected: All pods Running and Ready
If abnormal: [What it means and which Tier 2 action to take]

### Step 1.2: [What to check]
Command: `kubectl logs -n production -l app=my-service --since=15m | grep ERROR`
Expected: [Normal error count]
If abnormal: [Escalation path]
```

### Tier 2: Mitigate (Low-Medium Risk — Team Lead Approval)
Actions that reduce user impact without fixing the root cause. These are reversible and have well-understood blast radius.

**Characteristics**:
- Reversible within minutes
- Well-understood blast radius
- May require team lead approval
- Reduces impact while root cause fix is prepared

**Examples**:
- Scale up replicas to absorb load
- Restart a stuck deployment
- Disable a feature flag
- Redirect traffic away from a bad backend
- Extend a resource limit temporarily

**Template**:
```
## Tier 2: Mitigate

### Step 2.1: [Action]
Command: `kubectl scale deployment/my-service -n production --replicas=6`
Risk: Low — adds capacity, does not change code or config
Rollback: `kubectl scale deployment/my-service -n production --replicas=3`
Expected outcome: Error rate drops within 2 minutes
Approval: Team lead
```

### Tier 3: High-Risk (Requires Explicit Approval + Rollback Ready)
Actions that could cause additional impact if incorrect, or that involve irreversible state changes. Only execute after Tier 1 verification confirms the root cause.

**Characteristics**:
- May cause brief additional impact during execution
- May involve data changes or infrastructure modifications
- Rollback plan must be tested before execution
- Requires explicit approval from incident commander or engineering lead

**Examples**:
- Rollback to a previous deployment version
- Apply a database migration or schema change
- Modify security group or network policy rules
- Failover to a secondary database
- Terminate and replace a node

**Template**:
```
## Tier 3: High-Risk

### Step 3.1: [Action]
Command: `kubectl rollout undo deployment/my-service -n production`
Risk: Medium — brief disruption during rollout, previous version may have its own issues
Pre-check: Verify previous version (v2.3.0) was stable: [query/command]
Rollback: `kubectl rollout undo deployment/my-service -n production` (rolls forward again)
Expected outcome: Error rate returns to pre-deploy baseline within 5 minutes
Approval: Incident commander
Verify after: `kubectl rollout status deployment/my-service -n production`
```

## Constraints

### MUST DO
- Start with Tier 1 (verify) before jumping to fixes
- Include a rollback command for every Tier 2 and Tier 3 action
- Include expected outcome so the oncall knows if the action worked
- Include approval level for each action
- Include a verification step after each action

### MUST NOT DO
- Skip verification and go straight to Tier 3
- Execute Tier 3 actions without confirming root cause via Tier 1
- Provide actions without rollback plans
- Assume the first remediation will work (always have a fallback)
- Execute multiple changes simultaneously (one at a time, verify each)

## Output

A structured remediation plan with:
1. Tier 1 verification steps (2-4 commands with expected output)
2. Tier 2 mitigation options (1-3 reversible actions)
3. Tier 3 high-risk fixes (1-2 actions with full rollback procedures)
4. Decision tree: "If Tier 1 shows X → do Tier 2.1, if Y → do Tier 3.1"
