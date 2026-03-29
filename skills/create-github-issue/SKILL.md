---
name: create-github-issue
description: "Create a GitHub issue from investigation findings — RCA summary, affected services, reproduction steps, suggested labels. Requires gh CLI."
tags: [github, workflow]
homepage: https://neubird.ai/skills/create-github-issue
metadata: {"neubird":{"emoji":"🐛","requires":{"bins":["neubird","gh"]}}}
---

# Create GitHub Issue

Create a well-structured GitHub issue directly from investigation findings.

## When to Use

- After investigation identifies a code-level bug that needs a fix
- When a fix needs to be tracked in a GitHub project
- To hand off investigation context to a development team
- When the issue isn't urgent enough for a coding-agent handoff but needs tracking

## Core Workflow

1. **Extract findings** — Pull root cause, evidence, and affected services from the investigation
2. **Structure the issue** — Format into a clear bug report with reproduction steps
3. **Apply metadata** — Labels, assignees, project, and milestone
4. **Create** — Submit via `gh` CLI or API
5. **Link back** — Reference the investigation session in the issue body

## Issue Template

### Title
`[Service] Brief description of the defect` — e.g., `[auth-service] Nil pointer on upstream 502 in validateToken()`

### Body Structure

```markdown
## Summary
One-sentence description of the bug and its user-facing impact.

## Root Cause
What the investigation found — the specific code path, configuration, or data condition causing the failure.

## Evidence
- Error message or log line (with timestamp)
- Metric showing the impact (error rate, latency spike)
- Link to investigation session or RCA document

## Reproduction
1. Step-by-step to trigger the bug (or conditions under which it occurs)
2. Expected behavior
3. Actual behavior

## Affected Services
- Service name, namespace, version
- Blast radius: which downstream services/users are impacted

## Suggested Fix
Brief description of the recommended approach (if known from investigation).

## Acceptance Criteria
- [ ] Bug no longer reproduces under the described conditions
- [ ] Existing tests pass
- [ ] New test covers the failure mode
- [ ] Monitoring confirms the fix (error rate returns to baseline)
```

### Labels
Apply based on investigation findings:
- **Severity**: `sev:critical`, `sev:high`, `sev:medium`, `sev:low`
- **Type**: `bug`, `reliability`, `performance`, `security`
- **Area**: `backend`, `infrastructure`, `database`, `networking`
- **Source**: `investigation`, `oncall`, `postmortem`

## Example gh CLI Command

```bash
gh issue create \
  --repo neubirdai/raven \
  --title "[auth-service] Nil pointer on upstream 502 in validateToken()" \
  --body-file /tmp/issue-body.md \
  --label "bug,sev:high,backend,investigation" \
  --assignee "@me" \
  --project "Q1 Reliability"
```

## Constraints

### MUST DO
- Include the root cause from the investigation (not just symptoms)
- Include reproduction steps or triggering conditions
- Link back to the investigation session
- Set severity label based on actual impact
- Include acceptance criteria so the fixer knows when they're done

### MUST NOT DO
- Create issues without root cause (that's a symptom report, not a bug)
- Use vague titles ("Fix auth issue")
- Skip the evidence (logs, metrics) — the fixer needs context
- Create duplicate issues (search first)

## Output

A GitHub issue URL with the complete structured bug report.
