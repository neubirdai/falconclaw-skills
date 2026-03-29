---
name: coding-agent
description: "Hand off a diagnosed issue to a coding agent (Claude Code, Codex, etc.) for fix implementation — passes investigation context, affected files/services, and suggested fix approach."
tags: [automation, coding]
homepage: https://neubird.ai/skills/coding-agent
metadata: {"neubird":{"emoji":"🤖","requires":{"bins":["neubird"]}}}
---

# Coding Agent Handoff

Hand off a diagnosed incident to a coding agent for automated fix implementation.

## When to Use

- After investigation has identified the root cause at code level
- When the fix is well-scoped (specific file, specific function)
- To accelerate remediation by delegating implementation
- When the fix follows a known pattern (null check, timeout, retry logic, config update)

## Core Workflow

1. **Validate readiness** — Confirm the root cause is code-level and the fix scope is clear
2. **Build context package** — Assemble everything the coding agent needs
3. **Define acceptance criteria** — What tests must pass for the fix to be valid
4. **Hand off** — Pass the structured context to the coding agent
5. **Verify** — Review the generated fix against the original investigation

## Context Package Structure

The handoff must include all of these — missing any one leads to bad fixes:

### Investigation Summary
- Root cause statement (one sentence)
- Evidence: the specific log line, error message, or metric that proves the cause
- Timeline: when it started, what changed

### Affected Code
- Repository name and branch
- File path(s) and line number(s)
- The specific function or method that needs fixing
- Related tests that cover this code path

### Suggested Fix Approach
- What the fix should do (e.g., "add nil check before dereferencing response.Body")
- What the fix must NOT do (e.g., "do not change the function signature")
- Edge cases to handle
- Performance constraints (if any)

### Acceptance Criteria
- Existing tests that must still pass
- New test case(s) the fix should satisfy
- Expected behavior after the fix
- How to verify in staging

## Example Handoff

```
## Root Cause
Nil pointer dereference in auth middleware when upstream returns
HTTP 502 with empty body — response.Body is nil but code calls
response.Body.Close() unconditionally.

## Affected Code
- Repo: neubirdai/raven
- File: src/middleware/auth.go:142
- Function: validateToken()

## Fix
Add nil check: `if response != nil && response.Body != nil`
before the Close() call. Return a structured error (not panic)
when upstream returns non-200.

## Must NOT Change
- Function signature of validateToken()
- The retry logic in the caller

## Tests
- Existing: auth_test.go TestValidateToken_Success must still pass
- New: add TestValidateToken_UpstreamBadGateway that sends 502
  with nil body and expects a clean error return, not a panic
```

## Constraints

### MUST DO
- Include the exact file path and line number
- Include the specific error or failure mode
- Define what the fix must NOT change (blast radius control)
- Include at least one new test case
- Review the generated fix before merging

### MUST NOT DO
- Hand off vague problems ("something is wrong with auth")
- Skip the acceptance criteria (the agent needs to know when it's done)
- Hand off architectural changes (coding agents are best for targeted fixes)
- Trust the fix without human review
- Hand off if the root cause isn't confirmed (you'll get a wrong fix)

## Output

A structured handoff document containing:
1. Root cause summary with evidence
2. Affected files with line numbers
3. Suggested fix approach with constraints
4. Acceptance criteria and test cases
5. Verification steps for staging
