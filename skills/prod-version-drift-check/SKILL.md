---
name: prod-version-drift-check
description: "Prod version audit — list deployed image versions across all Kubernetes namespaces in the production cluster and send per-namespace results to the appropriate Slack channel."
tags: [kubernetes, deploy, slack, audit]
homepage: https://neubird.ai/skills/prod-version-drift-check
metadata: {"neubird":{"emoji":"🔍","requires":{"bins":["neubird","kubectl"]}}}
---

# Prod Version Drift Check

Audit deployed image versions across all Kubernetes namespaces in the production cluster, then send per-namespace results to the appropriate Slack channel.

## Prerequisites

- `kubectl` installed and configured with access to the prod cluster
- Slack Claude Code plugin installed (`claude plugin install slack`) and authenticated — messages are posted via this local MCP plugin

## When to Use

- After a rollout to verify versions propagated correctly
- During an incident to confirm which version is running where
- Routine release audits to catch stale or diverged namespaces
- Before a coordinated multi-namespace deploy to establish a baseline

## Core Workflow

### Step 1 — List namespaces

Run `kubectl get namespaces` against the `prod_cluster` context. Return the full list of namespace names.

```
kubectl get namespaces --context=prod_cluster -o jsonpath='{.items[*].metadata.name}'
```

### Step 2 — Get deployed version per namespace

For each namespace, inspect the image tag of the relevant deployment.

```
kubectl get deployment -n <namespace> --context=prod_cluster -o jsonpath='{.items[0].spec.template.spec.containers[0].image}'
```

Extract only the version tag — the segment after `:`. Examples:
- `myrepo/service:v1.4.2` → `v1.4.2`
- `myrepo/service:sha-abc1234` → `sha-abc1234`

If a namespace has no deployment or the version cannot be determined, record it as `N/A`.

### Step 3 — Resolve Slack channel per namespace

Look up each namespace in the channel mapping below. 

**If a namespace is not in the mapping, ask the user:**
> "I don't have a Slack channel for namespace `<name>`. Should I post it to `#eng` or somewhere else?"

Wait for a response before proceeding for that namespace. Do not assume a channel — always ask when in doubt.

#### Channel Mapping

| Namespace | Slack Channel |
|-----------|---------------|
| *(fill in as namespaces are discovered)* | |

Namespaces with no dedicated channel and no user-provided override default to `#eng`.

### Step 4 — Version drift detection

Before sending, compare all resolved versions across namespaces:
- Identify the **most common version** (treat it as the expected version)
- Flag any namespace running a **different version** as potentially behind or ahead

### Step 5 — Send to Slack

Use the Slack Claude Code plugin (installed via `claude plugin install slack`) to post to each resolved channel. No bot token or API key is needed — the plugin uses the OAuth access established during `claude plugin install slack`.

For each namespace, post to its resolved channel:

```
🔍 Prod Version Report — <namespace> — <date>

Version: <version tag or N/A>
Status: ✅ Current | ⚠️ Drifted (<version> vs expected <expected-version>) | ❓ Unknown
```

If version is `N/A`, include:
```
⚠️ No deployment found in this namespace. Manual verification required.
```

## Constraints

### MUST DO
- Always use `--context=prod_cluster` on every `kubectl` command — never assume the current kubeconfig context is prod
- Ask the user about unmapped namespaces before sending — never silently default without informing
- Report `N/A` honestly rather than skipping namespaces with no deployment
- Include the date in every Slack message header

### MUST NOT DO
- Post to a channel without confirming it maps to the correct namespace owner
- Declare a version "current" without comparing against other namespaces
- Skip system namespaces silently — note them as out-of-scope rather than omitting

## Output (local summary)

After all Slack messages are sent, output a summary table:

```
NAMESPACE          VERSION          CHANNEL       STATUS
---------          -------          -------       ------
namespace-a        v1.4.2           #team-alpha   ✅ Current
namespace-b        v1.3.9           #team-beta    ⚠️ Drifted
namespace-c        N/A              #eng          ❓ Unknown
```

Final line: `Sent to <N> channels. <M> namespaces flagged for drift or unknown version.`

## Example Usage

```
/prod-version-drift-check
```
