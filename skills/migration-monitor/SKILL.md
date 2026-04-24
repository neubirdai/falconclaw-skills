---
name: migration-monitor
description: "Post-cutover parity assurance for VMware → AWS migrations: captures live state on both sides via Falcon's native read-only discovery, diffs infrastructure shape (compute/storage/network/tags/security), endpoint behavior (HTTP/TCP/TLS response parity), metrics (CPU/memory/latency/error-rate equivalence within tolerance), and SQL database data parity (table presence, row counts, column schema), then emits a consolidated divergence report. Workload pairings are auto-discovered via multi-signal heuristic scoring (name, IP, DNS, hostname, OS, sizing) with user confirmation only for ambiguous matches; metric sources and available metrics are auto-discovered. Falcon is read-only and runs all probes natively; this skill provides the workflow, scoring algorithms, and parity rules. One-shot analysis; re-invoke as needed. Use after a VMware → AWS cutover to verify the target workloads behave like the originals."
tags: [migration, monitor, parity, vmware, aws, cloud]
homepage: https://neubird.ai/skills/migration-monitor
metadata: {"neubird":{"emoji":"🔎","requires":{"anyBins":["aws","govc"]}}}
---

# Migration Monitor (VMware → AWS)

_Body written in Task 2._
