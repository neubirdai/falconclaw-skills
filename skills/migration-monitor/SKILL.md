---
name: migration-monitor
description: "Post-cutover parity assurance for VMware → AWS migrations: captures live state on both sides via Falcon's native read-only discovery, diffs infrastructure shape (compute/storage/network/tags/security), endpoint behavior (HTTP/TCP/TLS response parity), metrics (CPU/memory/latency/error-rate equivalence within tolerance), and SQL database data parity (table presence, row counts, column schema), then emits a consolidated divergence report. Workload pairings are auto-discovered via multi-signal heuristic scoring (name, IP, DNS, hostname, OS, sizing) with user confirmation only for ambiguous matches; metric sources and available metrics are auto-discovered. Falcon is read-only and runs all probes natively; this skill provides the workflow, scoring algorithms, and parity rules. One-shot analysis; re-invoke as needed. Use after a VMware → AWS cutover to verify the target workloads behave like the originals."
tags: [migration, monitor, parity, vmware, aws, cloud]
homepage: https://neubird.ai/skills/migration-monitor
metadata: {"neubird":{"emoji":"🔎","requires":{"anyBins":["aws","govc"]}}}
---

# Migration Monitor (VMware → AWS)

Captures live state on both sides of a VMware → AWS migration, diffs infrastructure, endpoint behavior, metrics, and database data, and emits a divergence report. One-shot analysis for **post-cutover verification** — re-invoke as needed.

**Falcon handles all discovery natively.** This skill is pure knowledge: the workflow, signal weights, tolerance rules, metric equivalence tables, SQL queries, and report format. Every probe, command, and analysis step below is either Falcon's built-in capability or a concrete CLI invocation specified in the reference files. This skill ships no scripts or binaries.

**Read-only always.** No mutations are issued. Remediation suggestions are emitted as copy-ready commands for the user to execute.

## Workflow Overview

```
Phase 0    Capability probe              → Gate 1 (available surfaces)
Phase 1    Workload mapping              → Gate 2 (paired workloads)
Phase 2    Baseline capture              (Falcon enumerates both sides)
Phase 3    Infrastructure parity
Phase 4    Endpoint parity
Phase 5    Metrics parity
Phase 6    Data parity (SQL DBs)
Phase 7    Divergence report             → Gate 3 (disposition)
```

Phases 3–6 are skipped if their surface was unavailable at Phase 0; skipped phases are explicitly logged in the report.

## Phase 0: Capability Probe (REQUIRED FIRST STEP)

Run these read-only probes first every session. Determine which parity surfaces are available.

| Probe | Command | What it signals |
|-------|---------|-----------------|
| AWS reachable | `aws sts get-caller-identity` | account ID + active region |
| vCenter reachable | `govc about` (only if `GOVC_URL` + `GOVC_USERNAME` set) | live VMware discovery |
| Static VMware exports | look for `RVTools*.xlsx` / `RVTools*.csv` in cwd | RVTools fallback if no live vCenter |
| Metric sources | env-var scan: `PROMETHEUS_URL`, `DATADOG_API_KEY` + `DATADOG_APP_KEY`, `NEW_RELIC_API_KEY` + `NEW_RELIC_ACCOUNT`, `VROPS_URL` + `VROPS_USERNAME` | queryable metric sources |
| Endpoint probing | `curl --version` | HTTP/TCP diff available |
| TLS checks | `openssl version` | TLS cert validation available |
| DB parity | env-var scan: `SOURCE_DB_URI` + `SOURCE_DB_PASSWORD` AND `TARGET_DB_URI` + `TARGET_DB_PASSWORD` | DB parity available |

**Available-surfaces rules:**

- **Required** (block if missing): AWS + (live vCenter OR RVTools export).
- **Optional** (skip phase with logged reason if absent):
  - Endpoint parity requires `curl`.
  - TLS checks require `openssl`.
  - Metrics parity requires ≥1 metric source.
  - DB parity requires both source and target DB URIs + passwords.

Present the surface checklist as a summary and **stop at Gate 1** for user confirmation. User may adjust env and re-run Phase 0 rather than proceed.

## Phase 1: Workload Mapping

Produce `{vmware-workload → aws-resource}` pairings from the two inventories captured in Phase 2. Use multi-signal heuristic scoring; never rely on any single signal.

Signal extraction and scoring rules are in `references/probe-and-map.md`. Key summary: additive uncapped score; ≥80 auto-pair; 40–79 present top 3 candidates to user; <40 mark unresolved.

Present pairing results grouped as auto-paired / ambiguous / unresolved, and **stop at Gate 2** for user confirmation and overrides. The confirmed mapping is the contract for Phases 3–7.

## Phase 2: Baseline Capture

Enumerate state on both sides using Falcon's native CLI capabilities. Do not filter by tags — the discovery must be comprehensive because no single signal (tags included) is trustworthy for scoping.

Exact discovery command lists (AWS `describe-*` verbs, `govc *.info` commands, RVTools tab mapping) are in `references/probe-and-map.md`. Falcon executes them, parses outputs, and assembles the canonical in-memory inventory shape documented there.

## Phase 3: Infrastructure Parity

Load `references/infra-parity.md`. Compare compute shape, disk, network, tags, security rules, public/private exposure per the tolerance rules. Emit structured divergences per the classification rubric in that file.

## Phase 4: Endpoint Parity

Load `references/endpoint-parity.md`. For each endpoint pair derived from the Phase 1 mapping (ALB DNS, NLB DNS, Route 53 records, EIPs paired with their source counterparts), emit and execute the `curl` / `nc` / `openssl` recipes. Compare status, structural headers, semantic body, reachability, and TLS validity.

## Phase 5: Metrics Parity

Load `references/metrics-parity.md`. Before running, confirm the metric comparison window with the user (default 7 days pre-cutover vs 7 days post-cutover; user may override). Enumerate available metrics on both sides via the detected metric sources' catalog APIs. Map source metric names to CloudWatch equivalents via the shipped table. Compute p50/p95/p99/mean for each mapped metric over the comparison window on both sides. Emit divergences per the tolerance rules.

**Never prompt the user to name metrics or provide queries.** If a source metric has no mapping entry, emit it as "unmapped — manual review required."

## Phase 6: Data Parity (SQL DBs)

Load `references/data-parity.md`. Run the engine-specific read-only SQL queries (MySQL, PostgreSQL, SQL Server) on both source and target DBs. Compare table inventory, row counts per shared table, and column schema. Respect the optional `--cutoff-field` / `--cutoff-value` for CDC-in-flight workloads.

**Only `SELECT` queries are permitted.** No DDL, no `SELECT INTO`, no `UPDATE`/`DELETE`/`INSERT`.

## Phase 7: Divergence Report

Load `references/divergence-report.md`. Consolidate outputs of Phases 3–6 into a single classified report: structured JSON (format defined inline in that reference) + human summary table. Present at **Gate 3**. User dispositions each divergence (accepted / investigate / blocker).

## Reference Guide

| Topic | Reference | Load when |
|-------|-----------|-----------|
| Discovery commands + mapping algorithm | `references/probe-and-map.md` | Phases 0–2 |
| Infra tolerance rules | `references/infra-parity.md` | Phase 3 |
| Endpoint curl/nc/openssl recipes | `references/endpoint-parity.md` | Phase 4 |
| Metric source detection + name mapping | `references/metrics-parity.md` | Phase 5 |
| SQL queries per engine + comparison rules | `references/data-parity.md` | Phase 6 |
| Report schema + severity rubric + remediation | `references/divergence-report.md` | Phase 7 |

## Output Conventions

- **Copy-ready remediation commands** use the fenced-bundle format with file-path header as the first line inside the block where the output is a file:

  ````
  ```hcl
  # file: terraform/security-groups.tf
  resource "aws_security_group_rule" "web_ingress_443" {
    type              = "ingress"
    from_port         = 443
    to_port           = 443
    protocol          = "tcp"
    cidr_blocks       = ["10.0.0.0/8"]
    security_group_id = aws_security_group.web.id
  }
  ```
  ````

- **Runbook-style commands** are numbered and paired with **Expected output** blocks:

  ````
  **Step 1:** Discover AWS inventory.

  ```bash
  aws ec2 describe-instances --region <region>
  ```

  **Expected output:** JSON blob under `Reservations[].Instances[]`.
  ````

- **Divergence report** is a structured JSON object (schema defined in `references/divergence-report.md`) plus a human-readable summary table. Both are rendered at Gate 3.

## Constraints

### MUST DO

- Run Phase 0 capability probe first every session.
- Present results and wait for explicit user approval at each of the three gates before advancing.
- **Use multi-signal heuristic matching for Phase 1.** Use the weighted signal table in `probe-and-map.md`. Never rely on a single signal — especially not migration-tracking tags.
- **Auto-discover metric sources and available metrics from env.** Never ask the user to name metrics or provide metric queries.
- **Emit precise CLI commands for every probe, discovery, and comparison step.** No narrative handwaving. The references specify exact invocations; follow them.
- **Web-search current AWS CloudWatch metric namespaces** before finalizing any metric mapping — namespaces and metric names evolve.
- **Confirm the metric comparison window with the user before Phase 5 runs.** Default is 7 days pre-cutover vs 7 days post-cutover; user may override.
- Classify every divergence by severity using the tolerance rules in the relevant parity reference.
- Skip unavailable phases explicitly — the divergence report must include a `skippedPhases` list with reasons. Never silently skip.

### MUST NOT DO

- Execute any mutation. Falcon is read-only.
- Ship or suggest running probe scripts, helper binaries, or executable code. Falcon natively runs `aws`, `govc`, `curl`, `mysql`, etc.
- Generate synthetic traffic.
- Rely on migration-tracking tags as the sole mapping signal.
- Filter discovery by tags during Phase 2 baseline capture. Discovery must be comprehensive regardless of tag state.
- Prompt the user to enumerate metrics, metric queries, or metric names.
- Guess metric-name equivalence without consulting `metrics-parity.md`. If a metric has no mapping, emit as "unmapped."
- Run SQL that isn't `SELECT` (including `SELECT INTO`, DDL, DML).
