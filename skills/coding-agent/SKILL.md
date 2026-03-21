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

## What It Passes

- Full investigation context and RCA
- Affected files and services
- Suggested fix approach
- Test cases to validate the fix
