---
name: create-github-issue
description: "Create a GitHub issue from investigation findings — RCA summary, affected services, reproduction steps, suggested labels. Requires gh CLI."
tags: [github, workflow]
homepage: https://neubird.ai/skills/create-github-issue
metadata: {"neubird":{"emoji":"🐛","requires":{"bins":["neubird","gh"]}}}
---

# Create GitHub Issue

Create a GitHub issue directly from investigation findings.

## When to Use

- After investigation identifies a code-level bug
- When a fix needs to be tracked in a GitHub project
- To hand off investigation context to a development team

## What It Creates

- Issue title from the root cause summary
- Body with RCA details, affected services, and reproduction steps
- Suggested labels based on the investigation
- Links back to the investigation session
