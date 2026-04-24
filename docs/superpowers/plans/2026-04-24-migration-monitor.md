# Migration Monitor Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship the `migration-monitor` skill to `neubirdai/falconclaw-skills` — a post-cutover parity-assurance skill for VMware → AWS migrations, delivered as pure knowledge (workflow + decision logic + reference content) with no shipped scripts or binaries.

**Architecture:** Single skill at `skills/migration-monitor/` with an orchestrator `SKILL.md` (≤500 lines) and six reference markdown files loaded on demand. Falcon handles all discovery, probing, and command execution natively — the skill provides the knowledge layer only. No `scripts/` subdirectory.

**Tech Stack:** Markdown (only). Indirect dependencies (what Falcon needs to execute the recipes the references specify): `aws` CLI, `govc`, `curl`, `nc`, `openssl`, `mysql`/`psql`/`sqlcmd` CLIs, `jq`, `awk`/Python for percentile math. These are user-environment dependencies, not skill dependencies.

**Spec:** `docs/superpowers/specs/2026-04-24-migration-monitor-design.md` — keep open while executing.

**Branching:** Create a feature branch (`feat/migration-monitor`) in a worktree before Task 1. Open a PR after Task 10.

---

## Conventions used in this plan

- **No TDD.** This skill is markdown-only — no executable code to test. Validation is structural: CI validator passes, required sections exist, embedded code blocks parse (HCL via `terraform fmt`, SQL via `sqlparse`, JSON via `python -m json.tool`).
- **Reference files are written from detailed outlines.** Each task specifies required sections, required tables, and required example commands. Narrative prose is the implementer's job; the plan tells them *what to cover*, not *exact wording of every sentence*.
- **SKILL.md is embedded verbatim in Task 2.** It's the only file Falcon's model sees at invocation — precision matters.
- **Every task ends with a commit.** Message convention: `<type>(migration-monitor): <subject>`.
- **Spec is the authoritative reference.** When ambiguity arises, the spec wins. If the spec conflicts with the plan, flag it and ask — don't guess.

---

## Prerequisites

- [ ] On `main` with clean working tree: `git status --short` outputs nothing.
- [ ] Node + npm available: `node --version`.
- [ ] Python 3.9+ available: `python3 --version`.
- [ ] (Optional) `terraform` CLI for HCL fmt: `terraform version`. If missing, fmt checks skip.
- [ ] (Optional) `sqlparse` via `pip install --user sqlparse` for SQL validation in Task 9.

---

## Task 1: Scaffold skill directory + stub SKILL.md

**Purpose:** Get CI green with minimal content before filling in the orchestrator.

**Files:**
- Create: `skills/migration-monitor/SKILL.md`
- Create: `skills/migration-monitor/references/` (directory)

### Steps

- [ ] **Step 1: Verify worktree is on `feat/migration-monitor`**

```bash
git branch --show-current
```

Expected: `feat/migration-monitor`.

- [ ] **Step 2: Create directory structure**

```bash
mkdir -p skills/migration-monitor/references
```

(No `scripts/` subdirectory — spec scope decision #11.)

- [ ] **Step 3: Write stub `SKILL.md` with frontmatter + placeholder body**

Write to `skills/migration-monitor/SKILL.md`:

```markdown
---
name: migration-monitor
description: "Post-cutover parity assurance for VMware → AWS migrations: captures live state on both sides via Falcon's native read-only discovery, diffs infrastructure shape (compute/storage/network/tags/security), endpoint behavior (HTTP/TCP/TLS response parity), metrics (CPU/memory/latency/error-rate equivalence within tolerance), and SQL database data parity (table presence, row counts, column schema), then emits a consolidated divergence report. Workload pairings are auto-discovered via multi-signal heuristic scoring (name, IP, DNS, hostname, OS, sizing) with user confirmation only for ambiguous matches; metric sources and available metrics are auto-discovered. Falcon is read-only and runs all probes natively; this skill provides the workflow, scoring algorithms, and parity rules. One-shot analysis; re-invoke as needed. Use after a VMware → AWS cutover to verify the target workloads behave like the originals."
tags: [migration, monitor, parity, vmware, aws, cloud]
homepage: https://github.com/neubirdai/falconclaw-skills/tree/main/skills/migration-monitor
metadata: {"neubird":{"emoji":"🔎","requires":{"anyBins":["aws","govc"]}}}
---

# Migration Monitor (VMware → AWS)

_Body written in Task 2._
```

Preserve the `→` (U+2192) characters exactly — do not replace with `->`.

- [ ] **Step 4: Install CI validator dep if missing**

```bash
node --input-type=module -e "import { parse } from 'yaml'; console.log('yaml OK');" 2>/dev/null || npm install --no-save yaml
```

- [ ] **Step 5: Run CI validator locally**

```bash
node --input-type=module -e '
import { readFileSync } from "fs";
import { parse as parseYaml } from "yaml";
const path = "skills/migration-monitor/SKILL.md";
const content = readFileSync(path, "utf-8");
const lines = content.split("\n").length;
if (lines > 500) { console.error(`FAIL: ${lines} > 500`); process.exit(1); }
const m = content.match(/^---\n([\s\S]*?)\n---/);
if (!m) { console.error("FAIL: no frontmatter"); process.exit(1); }
const fm = parseYaml(m[1]);
if (fm.name !== "migration-monitor") { console.error(`FAIL: name "${fm.name}"`); process.exit(1); }
if (!fm.description) { console.error("FAIL: missing description"); process.exit(1); }
console.log(`OK (${lines} lines)`);
'
```

Expected: `OK (<N> lines)` where N ≈ 12.

- [ ] **Step 6: Commit**

```bash
git add skills/migration-monitor/
git commit -m "feat(migration-monitor): scaffold skill directory with stub SKILL.md"
```

---

## Task 2: SKILL.md orchestrator — full content

**Purpose:** The orchestrator is the only file Falcon's model loads at invocation time. Precision matters most here.

**Files:**
- Modify: `skills/migration-monitor/SKILL.md`

### Step 1: Replace the entire `SKILL.md` body

Write to `skills/migration-monitor/SKILL.md`:

````markdown
---
name: migration-monitor
description: "Post-cutover parity assurance for VMware → AWS migrations: captures live state on both sides via Falcon's native read-only discovery, diffs infrastructure shape (compute/storage/network/tags/security), endpoint behavior (HTTP/TCP/TLS response parity), metrics (CPU/memory/latency/error-rate equivalence within tolerance), and SQL database data parity (table presence, row counts, column schema), then emits a consolidated divergence report. Workload pairings are auto-discovered via multi-signal heuristic scoring (name, IP, DNS, hostname, OS, sizing) with user confirmation only for ambiguous matches; metric sources and available metrics are auto-discovered. Falcon is read-only and runs all probes natively; this skill provides the workflow, scoring algorithms, and parity rules. One-shot analysis; re-invoke as needed. Use after a VMware → AWS cutover to verify the target workloads behave like the originals."
tags: [migration, monitor, parity, vmware, aws, cloud]
homepage: https://github.com/neubirdai/falconclaw-skills/tree/main/skills/migration-monitor
metadata: {"neubird":{"emoji":"🔎","requires":{"anyBins":["aws","govc"]}}}
---

# Migration Monitor (VMware → AWS)

Captures live state on both sides of a VMware → AWS migration, diffs infrastructure, endpoint behavior, metrics, and database data, and emits a divergence report. One-shot analysis — re-invoke as needed.

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
  ```yaml
  # file: manifests/networkpolicy-web.yaml
  apiVersion: networking.k8s.io/v1
  ...
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
- Classify every divergence by severity using the tolerance rules in the relevant parity reference.
- Skip unavailable phases explicitly — the divergence report must include a `skippedPhases` list with reasons. Never silently skip.

### MUST NOT DO

- Execute any mutation. Falcon is read-only.
- Ship or suggest running probe scripts, helper binaries, or executable code. Falcon natively runs `aws`, `govc`, `curl`, `mysql`, etc.
- Generate synthetic traffic.
- Rely on migration-tracking tags as the sole mapping signal.
- Prompt the user to enumerate metrics, metric queries, or metric names.
- Guess metric-name equivalence without consulting `metrics-parity.md`. If a metric has no mapping, emit as "unmapped."
- Run SQL that isn't `SELECT` (including `SELECT INTO`, DDL, DML).
````

### Step 2: Validate structure and line count

```bash
node --input-type=module -e '
import { readFileSync } from "fs";
import { parse as parseYaml } from "yaml";
const path = "skills/migration-monitor/SKILL.md";
const content = readFileSync(path, "utf-8");
const lines = content.split("\n").length;
console.log(`Lines: ${lines} (cap 500)`);
if (lines > 500) { console.error(`FAIL: ${lines} > 500`); process.exit(1); }
const m = content.match(/^---\n([\s\S]*?)\n---/);
const fm = parseYaml(m[1]);
if (fm.name !== "migration-monitor") { console.error("FAIL: name mismatch"); process.exit(1); }
if (!fm.description) { console.error("FAIL: no description"); process.exit(1); }
console.log("OK");
'
```

Expected: `Lines: <N> (cap 500)` then `OK`. N should be in the ~170–210 range.

### Step 3: Verify all required sections are present

```bash
for section in \
  "# Migration Monitor (VMware → AWS)" \
  "## Workflow Overview" \
  "## Phase 0: Capability Probe (REQUIRED FIRST STEP)" \
  "## Phase 1: Workload Mapping" \
  "## Phase 2: Baseline Capture" \
  "## Phase 3: Infrastructure Parity" \
  "## Phase 4: Endpoint Parity" \
  "## Phase 5: Metrics Parity" \
  "## Phase 6: Data Parity (SQL DBs)" \
  "## Phase 7: Divergence Report" \
  "## Reference Guide" \
  "## Output Conventions" \
  "## Constraints"
do
  grep -qF "$section" skills/migration-monitor/SKILL.md || { echo "MISSING: $section"; exit 1; }
done
echo "All sections present."
```

### Step 4: Commit

```bash
git add skills/migration-monitor/SKILL.md
git commit -m "feat(migration-monitor): add SKILL.md orchestrator with 8-phase workflow and 3 gates"
```

---

## Task 3: references/probe-and-map.md

**Purpose:** The centerpiece. Drives Phases 0–2. Contains Phase 0 capability-probe recipes, Phase 1 multi-signal mapping algorithm, Phase 2 discovery command lists, and the canonical inventory shape Falcon produces.

**File:** `skills/migration-monitor/references/probe-and-map.md`

### Required sections (in this order)

1. **Purpose** (~5 lines) — summarize what phases load this reference.

2. **Phase 0: Capability Probe Recipes** (~80 lines)
   - Exact CLI command per probe from the SKILL.md Phase 0 table.
   - Expected output patterns (snippets) for success vs failure per probe.
   - Interpretation rules: what "AWS reachable" means (authenticated + region resolvable); what "vCenter reachable" means (govc.about returns a version).
   - Env-var scanning recipe: `env | grep -iE '^(PROMETHEUS_URL|DATADOG_API_KEY|DATADOG_APP_KEY|NEW_RELIC_API_KEY|NEW_RELIC_ACCOUNT|VROPS_URL|VROPS_USERNAME|SOURCE_DB_URI|TARGET_DB_URI)='`
   - Available-surfaces assembly logic (required + optional lists).

3. **Phase 1: Multi-signal Workload Mapping** (~150 lines)

   a. **Signal table** (embed verbatim):

   | Signal | Weight | Extraction |
   |--------|--------|------------|
   | Exact name match | +40 | VM name equals AWS resource natural identifier (EC2 `Name` tag; RDS `DBInstanceIdentifier`; ALB/NLB name; EFS `CreationToken`) — case-insensitive |
   | Fuzzy name match (mutex with exact) | +20 | Token similarity ≥0.7 OR Levenshtein ≤3; only if exact did not hit |
   | Private IP match | +30 | Any VMware VM IPv4 equals any AWS ENI private IP (EC2) or endpoint address (RDS) |
   | Public IP match | +30 | VMware public IP equals AWS EIP attached to target |
   | Hostname match | +25 | VMware guest hostname equals AWS Route 53 record target or ALB/NLB DNS name |
   | OS family match | +10 | Source Linux → target Linux, Windows → Windows; −20 on explicit mismatch |
   | Sizing plausibility | +10 | Target vCPU within 2× source AND target memory within 2× source |
   | Migration-tracking tag present | +15 | AWS resource has `migration-source-vm-id`/`migration-source-name` matching the VMware VM |
   | Annotation/notes cross-reference | +5 | VMware annotation contains target name/ID, or AWS `Description` contains source name |

   b. **Signal extraction details** — how each signal is computed in practice:
   - Name normalization: strip domain suffixes, lowercase, replace `_`/`-` with nothing for comparison.
   - IP matching: iterate all source NICs × all target ENIs.
   - Hostname vs DNS name: strip trailing dot, lowercase.
   - OS family extraction: from `guestOs` field on VMware side (e.g. `ubuntu20_64Guest` → Linux), from `Platform` on AWS side (`windows` or empty → Linux).
   - Sizing ratio math: compute (target/source) for both vCPU and memoryMB; both must be within [0.5, 2.0].
   - Tag signal: only hits if the AWS resource actually has one of the named tags — present this as +15 bonus, not a required signal.

   c. **Scoring thresholds and classification:**
   - Score is uncapped additive (spec scope decision #12).
   - ≥80 → auto-pair.
   - 40–79 → ambiguous, present top 3 candidates.
   - <40 → unresolved.

   d. **Pseudo-code for matching loop** (embed):

   ```text
   Input:  vmware_inventory (list of VMs with properties)
           aws_inventory    (list of AWS resources with properties)

   pairs = []           # high-confidence auto-pairs
   ambiguous = []       # medium-confidence, need user choice
   unresolved_vmware = [] # VMs with no candidate ≥40
   aws_claimed = set()  # AWS resources already in a high-confidence pair

   for vm in vmware_inventory:
     candidates = []
     for aws in aws_inventory:
       if aws.type not in applicable_types(vm):
         continue                 # don't pair a VM with an S3 bucket
       score = sum(signal_weight for signal in applicable_signals(vm, aws))
       if score > 0:
         candidates.append((aws, score, signals_hit))
     candidates.sort(by score desc)
     if candidates and candidates[0].score >= 80:
       pairs.append((vm, candidates[0]))
       aws_claimed.add(candidates[0].aws.id)
     elif candidates and candidates[0].score >= 40:
       top3 = [c for c in candidates[:3] if c.aws.id not in aws_claimed]
       ambiguous.append((vm, top3))
     else:
       unresolved_vmware.append(vm)

   unresolved_aws = [a for a in aws_inventory if a.id not in aws_claimed
                    and not in any ambiguous candidate set]

   return pairs, ambiguous, unresolved_vmware, unresolved_aws
   ```

   e. **Tie-breakers:** if two candidates have the same score, prefer (1) same region, (2) same OS, (3) earlier creation timestamp.

   f. **Gate 2 interaction recipe** — how to present results to the user:
   - Auto-paired list, collapsed with count ("Auto-paired: 14 workloads").
   - Ambiguous list, expanded with top 3 candidates per VM.
   - Unresolved VMware list with instruction: "pick an AWS resource ID, or declare out-of-scope."
   - Unresolved AWS list with instruction: "pick a VMware VM, or declare 'native AWS resource not migrated from VMware'."
   - Final mapping output structure (JSON example).

4. **Phase 2: Discovery Command Lists** (~150 lines)

   a. **VMware live discovery** (when `govc` reachable):
   - Exact commands:
     - `govc find / -type m` → list of VM inventory paths
     - `govc vm.info -json -vm.ipath=<path>` for each VM
     - `govc ls -json /... /cluster/...` for clusters/hosts/datastores
     - `govc host.info -json`
     - `govc datastore.info -json`
     - `govc network.info -json`
   - Field extraction per VM: id (MoRef), name, guestOs (Config.GuestId), vcpus (Config.Hardware.NumCPU), memoryMB (Config.Hardware.MemoryMB), disks (iterate Config.Hardware.Device for Hard disk entries), networks (iterate for MacAddress entries), tags (via `govc tags.attached.ls -r VirtualMachine:<id>`), guest hostname (Guest.HostName), guest IPs (Guest.Net[].IpAddress), annotation (Config.Annotation), powerState (Runtime.PowerState).

   b. **VMware RVTools fallback** (when RVTools export in cwd):
   - Expected tabs: `vInfo`, `vCPU`, `vMemory`, `vDisk`, `vNetwork`, `vHost`, `vDatastore`.
   - Column extraction per tab (for xlsx and CSV). Falcon reads via `python3 -c "import openpyxl; ..."` or equivalent.

   c. **AWS discovery** (full account + active region):
   - Exact commands (all read-only):
     - `aws sts get-caller-identity` → account, user/role context
     - `aws ec2 describe-instances --region <region>`
     - `aws ec2 describe-security-groups --region <region>`
     - `aws ec2 describe-subnets --region <region>`
     - `aws ec2 describe-vpcs --region <region>`
     - `aws ec2 describe-addresses --region <region>` (EIPs)
     - `aws rds describe-db-instances --region <region>`
     - `aws rds describe-db-clusters --region <region>`
     - `aws elbv2 describe-load-balancers --region <region>`
     - `aws elbv2 describe-target-groups --region <region>`
     - `aws elbv2 describe-target-health --target-group-arn <arn>` (per TG)
     - `aws elb describe-load-balancers --region <region>` (classic ELB)
     - `aws efs describe-file-systems --region <region>`
     - `aws fsx describe-file-systems --region <region>`
     - `aws elasticache describe-cache-clusters --region <region>`
     - `aws kafka list-clusters-v2 --region <region>`
     - `aws route53 list-hosted-zones` (global) + `aws route53 list-resource-record-sets --hosted-zone-id <id>` per zone
   - Field extraction per resource type (summary rule: for each resource, extract identifier + Name tag + all private/public IPs + all tags + `Description`/annotation-equivalent fields).
   - Note for large accounts: pagination is handled by `--no-cli-pager` and `--max-items` defaults. If the account is very large, user can set `MIGRATION_MONITOR_VPC_ID` to scope (documented in this file).

   d. **Canonical inventory JSON shape** (embed inline — this is guidance for Falcon's output, not a runtime validator):

   ```json
   {
     "capturedAt": "<ISO8601 UTC>",
     "vmware": {
       "source": "govc | rvtools",
       "vcenter": { "url": "...", "version": "..." },
       "vms": [
         {
           "id": "vm-1001",
           "name": "web-01",
           "guestOs": "ubuntu20_64Guest",
           "hostname": "web-01.corp.example",
           "vcpus": 4,
           "memoryMB": 8192,
           "disks": [{"sizeGB": 80, "datastore": "ds-01", "thin": true}],
           "networks": [{"mac": "00:50:56:aa:bb:cc", "portGroup": "web-net"}],
           "ipAddresses": ["10.0.1.10"],
           "publicIp": null,
           "tags": [{"category": "app", "value": "web"}],
           "annotation": "",
           "powerState": "on"
         }
       ]
     },
     "aws": {
       "account": "123456789012",
       "region": "us-east-1",
       "resources": [
         {
           "type": "ec2_instance",
           "id": "i-0abc123",
           "name": "web-01",
           "instanceType": "t3.large",
           "platform": "Linux",
           "privateIps": ["10.0.1.10"],
           "publicIp": null,
           "subnetId": "subnet-xxx",
           "securityGroupIds": ["sg-xxx"],
           "tags": {"Name": "web-01", "Environment": "prod"}
         },
         {
           "type": "rds_instance",
           "id": "db-prod-01",
           "engine": "postgres",
           "endpoint": "db-prod-01.xxx.us-east-1.rds.amazonaws.com",
           "privateIps": ["10.0.2.20"]
         }
       ]
     }
   }
   ```

5. **Mapping output shape** (~30 lines) — final structure passed to Phases 3–7:

   ```json
   {
     "pairs": [
       {"vmware_id": "vm-1001", "aws_id": "i-0abc123", "confidence": 95, "signals_hit": ["exact-name", "private-ip", "os-family", "sizing"]}
     ],
     "unresolved_vmware": ["vm-1999"],
     "unresolved_aws": ["i-deadbeef"],
     "skipped_vmware": ["vm-archive-01"]
   }
   ```

### Steps

- [ ] **Step 1: Write the file per the outline.** Embed the signal table, pseudo-code, command lists, and JSON examples verbatim.

- [ ] **Step 2: Structural validation**

```bash
for section in \
  "## Phase 0: Capability Probe Recipes" \
  "## Phase 1: Multi-signal Workload Mapping" \
  "## Phase 2: Discovery Command Lists" \
  "## Mapping output shape"
do
  grep -qF "$section" skills/migration-monitor/references/probe-and-map.md \
    || { echo "MISSING: $section"; exit 1; }
done
echo "OK"
```

- [ ] **Step 3: Validate JSON examples parse**

```bash
python3 <<'PY'
import re, json, sys
content = open("skills/migration-monitor/references/probe-and-map.md").read()
blocks = re.findall(r"```json\n(.*?)```", content, re.DOTALL)
failed = 0
for i, b in enumerate(blocks):
    try: json.loads(b)
    except Exception as e:
        print(f"JSON block {i} FAILED: {e}"); failed += 1
print(f"{len(blocks)} JSON blocks, {failed} failures")
sys.exit(1 if failed else 0)
PY
```

- [ ] **Step 4: Commit**

```bash
git add skills/migration-monitor/references/probe-and-map.md
git commit -m "docs(migration-monitor): add probe-and-map reference with mapping algorithm and discovery commands"
```

---

## Task 4: references/infra-parity.md

**Purpose:** Phase 3 rules — per-attribute comparison with tolerance thresholds.

**File:** `skills/migration-monitor/references/infra-parity.md`

### Required sections

1. **Purpose** (~5 lines).

2. **Compute shape** (~40 lines)
   - Rule: `target.vcpu / source.vcpu ≥ 0.8` (right-sizing floor)
   - Rule: `target.memoryMB / source.memoryMB ≥ 0.8` when no telemetry p95 is available; otherwise `target.memoryMB ≥ 1.2 × source.memoryP95MB`
   - OS family must match (Linux→Linux, Windows→Windows). Explicit mismatch = critical divergence.
   - Architecture must match (x86_64 vs arm64). Mismatch = critical unless user acknowledges.
   - Severity: below floor = major; below 60% of floor = critical.

3. **Disk / storage** (~40 lines)
   - Rule: total attached capacity on target within 10% of source total. Excludes OS-disk growth (acceptable because AWS AMIs may repartition).
   - Disk count may differ — one large source VMDK → multiple AWS EBS volumes is fine.
   - Encryption parity: if source used VMware disk encryption (property on VMFS datastore or VM-level), all target EBS volumes must have `Encrypted=true`. Non-compliance = critical.
   - Severity: critical if any volume required-encrypted but actual is unencrypted; major if capacity deficit > 20%.

4. **Network** (~60 lines)
   - Outbound rule parity: target security groups' outbound rules must permit at least the same destinations/ports as the source allowed. Represent as set-membership check.
   - Inbound rule parity: target inbound rules must permit at least the same source CIDRs on the same ports.
   - Public exposure: if source had a public IP / EIP, target must be publicly reachable (EIP OR fronted by public ALB/NLB). If source was private, target must be in private subnet (no public IP on primary ENI, no public-facing LB).
   - Subnet count may differ (consolidation is fine).
   - NACLs: not checked in v1 (complex enough to warrant its own treatment; out of scope).
   - Severity: missing required-allowed port = critical; extra open port on target = major (potential security regression); different CIDR granularity = minor.

5. **Tags** (~30 lines)
   - Rule: tag keys that exist on source should exist on target (values may differ — e.g., environment `prod` on source might map to `production` on target).
   - Migration-tracking tags (`migration-wave`, `migration-source-vm-id`, `migration-date`): **informational only** — their absence is a minor divergence, not major. v1 doesn't require them.
   - User can specify a `required-tags` set in session state; missing required tags on target = major.
   - Severity: missing required tag = major; minor otherwise.

6. **Public/private exposure summary table** (~20 lines) — quick reference for the common cases (public web → ALB; private DB → subnet-only; mixed → flag).

7. **Divergence classification matrix** (~20 lines) — summary table of all divergence types in this reference with their severity floors.

### Steps

- [ ] **Step 1: Write the file per the outline.** Every rule must cite the severity it produces.

- [ ] **Step 2: Structural validation**

```bash
for section in \
  "## Compute shape" \
  "## Disk / storage" \
  "## Network" \
  "## Tags"
do
  grep -qF "$section" skills/migration-monitor/references/infra-parity.md \
    || { echo "MISSING: $section"; exit 1; }
done
echo "OK"
```

- [ ] **Step 3: Commit**

```bash
git add skills/migration-monitor/references/infra-parity.md
git commit -m "docs(migration-monitor): add infra-parity reference with tolerance rules and severity classifications"
```

---

## Task 5: references/endpoint-parity.md

**Purpose:** Phase 4 — exact `curl`/`nc`/`openssl` recipes and diff rules that Falcon emits and executes.

**File:** `skills/migration-monitor/references/endpoint-parity.md`

### Required sections

1. **Purpose** (~5 lines).

2. **Endpoint pair derivation** (~30 lines) — how endpoint pairs come out of the Phase 1 mapping:
   - VMware VM → AWS EC2 with public IP: source URL is the VM's hostname (from guest tools) or public IP; target URL is EIP DNS or Route 53 record.
   - VMware VM behind a load balancer → AWS ALB/NLB: source URL is original FQDN; target URL is ALB DNS.
   - VMware DB port → AWS RDS endpoint: TCP + TLS probes only (no HTTP).

3. **HTTP probe recipe** (~80 lines)

   a. **Exact commands** (embed):

   ```bash
   # Source side
   curl -sS -o /tmp/mmon_source_body_${PAIR_ID}.out \
        -D /tmp/mmon_source_headers_${PAIR_ID}.out \
        -w '%{http_code}\n%{time_total}\n%{content_type}\n%{size_download}\n' \
        --max-time 30 \
        "$SOURCE_URL"

   # Target side (same flags)
   curl -sS -o /tmp/mmon_target_body_${PAIR_ID}.out \
        -D /tmp/mmon_target_headers_${PAIR_ID}.out \
        -w '%{http_code}\n%{time_total}\n%{content_type}\n%{size_download}\n' \
        --max-time 30 \
        "$TARGET_URL"
   ```

   Supports `AUTH_HEADER` env var: `--header "$AUTH_HEADER"` if set.

   b. **Comparison rules:**
   - `http_code`: mismatch = critical.
   - `content_type`: structural header mismatch (e.g., `application/json` vs `text/html`) = major; charset-only mismatch (`utf-8` vs `UTF-8`) = informational.
   - `time_total`: informational only at this layer (metrics parity handles performance).
   - `size_download`: ±20% = informational; >50% deviation = minor (possible content drift).
   - Structural headers: compare `Content-Type`, `Cache-Control`, security headers (`Strict-Transport-Security`, `X-Content-Type-Options`, `X-Frame-Options`, `Content-Security-Policy`). Missing security header on target that source had = major.

   c. **Body semantic diff:**
   - JSON: parse both, sort keys recursively, strip fields in the volatile-strip list (`timestamp`, `requestId`, `traceId`, `date`, `server_time`, `now`, case-insensitive match), then `diff` the canonical JSON. Any remaining diff = major (structural change).
   - HTML: use a normalization pass (strip doctype, comments, lowercase attribute names, sort attributes within each tag, collapse whitespace), then `diff`. Any remaining diff = minor by default (HTML drift is common and benign).
   - Other content-types: byte-exact compare, diff = minor (not enough info to judge semantic equivalence).

4. **TCP probe recipe** (~30 lines)

   ```bash
   nc -z -w5 $SOURCE_HOST $SOURCE_PORT; SRC_REACH=$?
   nc -z -w5 $TARGET_HOST $TARGET_PORT; TGT_REACH=$?
   ```

   Divergence: `SRC_REACH != TGT_REACH` = critical (reachability parity broken).

5. **TLS probe recipe** (~40 lines)

   ```bash
   echo | openssl s_client -connect "$HOST:443" -servername "$HOST" 2>/dev/null \
     | openssl x509 -noout -dates -subject -issuer > /tmp/mmon_tls_${PAIR_ID}_${SIDE}.txt
   ```

   Extract from both sides:
   - `notBefore`, `notAfter` dates → validate both are currently valid (not-expired, within-validity-window).
   - `subject CN` → should match the endpoint hostname (wildcards allowed if applicable).
   - `issuer CN` and `issuer O` → record. Issuer family (LetsEncrypt / ACM / corporate CA) acceptable as long as both verify.

   Divergences:
   - Target cert expired or not-yet-valid = critical.
   - Subject CN mismatch = major.
   - Different issuer family is informational unless policy requires same family.

6. **Divergence emission format** (~30 lines) — each divergence `{id, phase: "endpoint", severity, type, pair, description, evidence}` where `evidence` includes `{source_response_code, target_response_code, sample_diff_lines}`.

### Steps

- [ ] **Step 1: Write the file per the outline.** All shell commands must be syntactically valid (testable via shellcheck extraction in Task 9).

- [ ] **Step 2: Structural validation**

```bash
for section in \
  "## Endpoint pair derivation" \
  "## HTTP probe recipe" \
  "## TCP probe recipe" \
  "## TLS probe recipe"
do
  grep -qF "$section" skills/migration-monitor/references/endpoint-parity.md \
    || { echo "MISSING: $section"; exit 1; }
done
echo "OK"
```

- [ ] **Step 3: Commit**

```bash
git add skills/migration-monitor/references/endpoint-parity.md
git commit -m "docs(migration-monitor): add endpoint-parity reference with HTTP/TCP/TLS recipes"
```

---

## Task 6: references/metrics-parity.md

**Purpose:** Phase 5 — metric source detection, metric catalog enumeration, the metric-name mapping table (CloudWatch equivalents), statistical comparison, and tolerance thresholds. This is the longest reference.

**File:** `skills/migration-monitor/references/metrics-parity.md`

### Required sections

1. **Purpose** (~5 lines).

2. **Metric-source detection** (~30 lines)

   Env-var → source-type table:

   | Env var(s) present | Source type |
   |---|---|
   | `PROMETHEUS_URL` | Prometheus |
   | `DATADOG_API_KEY` + `DATADOG_APP_KEY` | Datadog |
   | `NEW_RELIC_API_KEY` + `NEW_RELIC_ACCOUNT` | New Relic |
   | `VROPS_URL` + `VROPS_USERNAME` + `VROPS_PASSWORD` | vROps (VMware Aria Operations) |

   CloudWatch is always the target side (implicit from AWS credentials available at Phase 0).

3. **Per-source catalog enumeration** (~50 lines)

   Exact commands Falcon runs to list available metrics:

   - **Prometheus:**
     ```bash
     curl -s "$PROMETHEUS_URL/api/v1/label/__name__/values" | jq -r '.data[]'
     ```

   - **Datadog:**
     ```bash
     curl -s -H "DD-API-KEY: $DATADOG_API_KEY" -H "DD-APPLICATION-KEY: $DATADOG_APP_KEY" \
          "https://api.datadoghq.com/api/v1/metrics?from=$(date -u -d '1 day ago' +%s)" \
          | jq -r '.metrics[]'
     ```

   - **New Relic** (NRQL via GraphQL):
     ```bash
     curl -s -H "Api-Key: $NEW_RELIC_API_KEY" -H "Content-Type: application/json" \
          "https://api.newrelic.com/graphql" \
          -d '{"query":"{ actor { account(id: $NEW_RELIC_ACCOUNT) { nrql(query: \"SHOW METRICS\") { results } } } }"}'
     ```

   - **vROps** (per resource):
     ```bash
     curl -s -u "$VROPS_USERNAME:$VROPS_PASSWORD" \
          "$VROPS_URL/suite-api/api/resources/<resource-id>/statkeys"
     ```

   - **CloudWatch** (per EC2/RDS/ALB/etc.):
     ```bash
     aws cloudwatch list-metrics --namespace AWS/EC2 \
          --dimensions Name=InstanceId,Value=<instance-id>
     ```

4. **Metric-name mapping table** (~150 lines) — the authoritative mapping. Embed at least 20 rows covering EC2, RDS, ALB/NLB, EFS, and general system metrics. Minimum rows:

   | Source metric | CloudWatch equivalent | CloudWatch namespace |
   |---------------|----------------------|----------------------|
   | vROps `cpu\|usage_average` | `CPUUtilization` | `AWS/EC2` |
   | vROps `mem\|usage_average` | `mem_used_percent` | `CWAgent` |
   | vROps `mem\|active_average` | `mem_used_percent` | `CWAgent` |
   | vROps `disk\|usage_average` | `disk_used_percent` | `CWAgent` |
   | vROps `disk\|commandsAveraged_average` | `DiskReadOps` + `DiskWriteOps` (sum) | `AWS/EC2` |
   | vROps `net\|throughput_usage_average` | `NetworkIn` + `NetworkOut` (sum) | `AWS/EC2` |
   | vROps `net\|packetsRx_summation` | `NetworkPacketsIn` | `AWS/EC2` |
   | vROps `net\|packetsTx_summation` | `NetworkPacketsOut` | `AWS/EC2` |
   | Prom `node_cpu_seconds_total{mode!="idle"}` (rate) | `CPUUtilization` | `AWS/EC2` |
   | Prom `node_memory_MemAvailable_bytes` | `mem_used_percent` (inverted) | `CWAgent` |
   | Prom `node_filesystem_avail_bytes` | `disk_used_percent` (inverted) | `CWAgent` |
   | Prom `node_network_receive_bytes_total` (rate) | `NetworkIn` | `AWS/EC2` |
   | Prom `node_network_transmit_bytes_total` (rate) | `NetworkOut` | `AWS/EC2` |
   | Prom `http_request_duration_seconds{quantile="0.95"}` | `TargetResponseTime` (p95) | `AWS/ApplicationELB` |
   | Prom `http_requests_total{status=~"5.."}` (rate) | `HTTPCode_Target_5XX_Count` | `AWS/ApplicationELB` |
   | Prom `http_requests_total` (rate) | `RequestCount` | `AWS/ApplicationELB` |
   | Datadog `system.cpu.user` + `system.cpu.system` | `CPUUtilization` | `AWS/EC2` |
   | Datadog `system.mem.used` / `system.mem.total` | `mem_used_percent` | `CWAgent` |
   | Datadog `system.disk.in_use` | `disk_used_percent` | `CWAgent` |
   | Datadog `system.net.bytes_rcvd` | `NetworkIn` | `AWS/EC2` |
   | New Relic `system.cpuPercent` | `CPUUtilization` | `AWS/EC2` |
   | New Relic `system.memoryUsedPercent` | `mem_used_percent` | `CWAgent` |
   | RDS Prom `pg_stat_database_tup_fetched` (rate) | `ReadIOPS` | `AWS/RDS` |
   | RDS Prom `pg_stat_database_tup_inserted+updated+deleted` (rate) | `WriteIOPS` | `AWS/RDS` |
   | vROps `summary\|runtime.connectionState` | `StatusCheckFailed` (inverted) | `AWS/EC2` |

   For metrics with no direct equivalent, the table must have a "(no mapping — manual review)" note so the implementer's algorithm can emit "unmapped" rather than fabricate.

5. **Statistical comparison** (~50 lines)

   For each mapped metric pair:
   - Fetch source-side values over the `source_window` (default 7 days pre-cutover).
   - Fetch target-side values over the `target_window` (default 7 days post-cutover).
   - Compute p50, p95, p99, mean on both sides.
   - For each statistic, compute divergence ratio: `abs(target - source) / source`.
   - Classification:
     - Within ±10% → OK (within tolerance)
     - 10–30% → minor
     - 30–50% → major
     - >50% → critical

6. **Percentile computation recipe** (~30 lines) — reference ships both an `awk` and a Python one-liner. Falcon picks whichever fits the environment.

   `awk` one-liner for a newline-delimited numeric stream:
   ```bash
   sort -n | awk 'BEGIN{c=0} {a[c++]=$1} END{
     p50=a[int(c*0.5)]; p95=a[int(c*0.95)]; p99=a[int(c*0.99)];
     sum=0; for(i=0;i<c;i++) sum+=a[i]; mean=sum/c;
     print p50, p95, p99, mean
   }'
   ```

   Python one-liner:
   ```bash
   python3 -c "import sys, statistics as s; v=sorted(float(x) for x in sys.stdin); print(v[int(len(v)*0.5)], v[int(len(v)*0.95)], v[int(len(v)*0.99)], s.mean(v))"
   ```

7. **Unmapped metric handling** (~15 lines) — if a metric is present on source side but has no entry in the mapping table, emit as:

   ```json
   {"type": "metrics.unmapped", "severity": "minor", "metric": "<name>", "hint": "No CloudWatch equivalent in v1 mapping table — manual review required."}
   ```

### Steps

- [ ] **Step 1: Write the file per the outline.** The mapping table is the centerpiece — at minimum 20 rows covering the representative cases listed.

- [ ] **Step 2: Structural validation**

```bash
for section in \
  "## Metric-source detection" \
  "## Per-source catalog enumeration" \
  "## Metric-name mapping table" \
  "## Statistical comparison" \
  "## Percentile computation recipe" \
  "## Unmapped metric handling"
do
  grep -qF "$section" skills/migration-monitor/references/metrics-parity.md \
    || { echo "MISSING: $section"; exit 1; }
done
echo "OK"
```

- [ ] **Step 3: Commit**

```bash
git add skills/migration-monitor/references/metrics-parity.md
git commit -m "docs(migration-monitor): add metrics-parity reference with source-to-CloudWatch mapping table"
```

---

## Task 7: references/data-parity.md

**Purpose:** Phase 6 — SQL queries per engine, tolerance rules, cutoff-timestamp handling.

**File:** `skills/migration-monitor/references/data-parity.md`

### Required sections

1. **Purpose** (~5 lines).

2. **Supported engines** (~15 lines) — v1 scope: MySQL 5.7+, PostgreSQL 12+, SQL Server 2017+. Oracle deferred. NoSQL deferred. Each engine's CLI tool: `mysql`, `psql`, `sqlcmd`.

3. **URI format and credential handling** (~25 lines)

   URIs: `mysql://user@host:3306/db`, `postgresql://user@host:5432/db`, `mssql://user@host:1433/db`.
   Passwords from env: `SOURCE_DB_PASSWORD`, `TARGET_DB_PASSWORD`. Never from command-line args (visible in `ps`).

   Concrete invocation patterns (embed):
   ```bash
   # MySQL
   mysql -h <host> -P <port> -u <user> -p"$DB_PASSWORD" -D <db> -N -B -e "<query>"

   # PostgreSQL
   PGPASSWORD="$DB_PASSWORD" psql -h <host> -p <port> -U <user> -d <db> -t -A -c "<query>"

   # SQL Server
   sqlcmd -S <host>,<port> -U <user> -P "$DB_PASSWORD" -d <db> -h -1 -W -Q "<query>"
   ```

4. **Table inventory queries** (~40 lines) — per engine, exact SQL:

   **MySQL:**
   ```sql
   SELECT table_name
     FROM information_schema.tables
    WHERE table_schema = DATABASE()
      AND table_type = 'BASE TABLE'
    ORDER BY table_name;
   ```

   **PostgreSQL:**
   ```sql
   SELECT tablename
     FROM pg_catalog.pg_tables
    WHERE schemaname NOT IN ('pg_catalog','information_schema')
    ORDER BY tablename;
   ```

   **SQL Server:**
   ```sql
   SELECT TABLE_NAME
     FROM INFORMATION_SCHEMA.TABLES
    WHERE TABLE_TYPE = 'BASE TABLE'
      AND TABLE_SCHEMA NOT IN ('sys','INFORMATION_SCHEMA')
    ORDER BY TABLE_NAME;
   ```

5. **Row count queries** (~30 lines)

   Base form: `SELECT COUNT(*) FROM <schema>.<table>;` (schema qualifier added where engine requires).

   With cutoff (if `cutoff_field` and `cutoff_value` are provided for CDC-in-flight): `SELECT COUNT(*) FROM <schema>.<table> WHERE <cutoff_field> < '<cutoff_value>';` — date values must be passed as strings with engine-specific formatting (MySQL `YYYY-MM-DD HH:MM:SS`, PostgreSQL ISO 8601, SQL Server per engine config).

   Tolerance: default within 1% (accounts for in-flight). Strict mode (0%) via user flag.

6. **Column schema queries** (~40 lines)

   **MySQL / PostgreSQL:**
   ```sql
   SELECT column_name, ordinal_position
     FROM information_schema.columns
    WHERE table_schema = <schema>
      AND table_name = <table>
    ORDER BY ordinal_position;
   ```

   **SQL Server:**
   ```sql
   SELECT COLUMN_NAME, ORDINAL_POSITION
     FROM INFORMATION_SCHEMA.COLUMNS
    WHERE TABLE_SCHEMA = <schema>
      AND TABLE_NAME = <table>
    ORDER BY ORDINAL_POSITION;
   ```

   Comparison: column count + column names. **Types are NOT compared** (platform type name drift is expected, e.g., MySQL `INT` vs PostgreSQL `INTEGER`).

7. **Divergence classification** (~25 lines)

   | Divergence type | Severity | Notes |
   |-----------------|----------|-------|
   | `data.table.missing-on-target` | critical | Migration incomplete. |
   | `data.table.extra-on-target` | minor | May be expected schema evolution. |
   | `data.rowcount.deficit` (>1%) | major | Possible replication lag or data loss. |
   | `data.rowcount.deficit` (>10%) | critical | Significant data loss risk. |
   | `data.rowcount.surplus` (>1%) | minor | Target has more rows — possibly duplicate inserts. |
   | `data.column.missing-on-target` | critical | Schema regression. |
   | `data.column.extra-on-target` | minor | Schema evolution. |
   | `data.engine-unavailable` | blocked | CLI tool missing on Falcon's host. |

8. **Exclusion and safety** (~20 lines)
   - `--exclude-table <regex>` for skipping volatile tables.
   - **Read-only enforcement:** only `SELECT` queries. Any command that would modify data (INSERT/UPDATE/DELETE/DDL/SELECT INTO) must be rejected before emission.
   - Connection timeouts: set `--connect-timeout 10` where the CLI supports it.

### Steps

- [ ] **Step 1: Write the file per the outline.** Every SQL example must parse. Test with `sqlparse` if available.

- [ ] **Step 2: Structural validation + SQL syntax check**

```bash
for section in \
  "## Supported engines" \
  "## URI format and credential handling" \
  "## Table inventory queries" \
  "## Row count queries" \
  "## Column schema queries" \
  "## Divergence classification"
do
  grep -qF "$section" skills/migration-monitor/references/data-parity.md \
    || { echo "MISSING: $section"; exit 1; }
done
echo "OK"

# Extract SQL blocks and parse via sqlparse if available
python3 <<'PY'
import re, sys
content = open("skills/migration-monitor/references/data-parity.md").read()
blocks = re.findall(r"```sql\n(.*?)```", content, re.DOTALL)
print(f"Found {len(blocks)} SQL blocks")
try:
    import sqlparse
    for i, b in enumerate(blocks):
        parsed = sqlparse.parse(b)
        if not parsed:
            print(f"Block {i} failed to parse"); sys.exit(1)
    print("All SQL blocks parse via sqlparse")
except ImportError:
    print("sqlparse not installed — skipping syntax check. Install with: pip install sqlparse")
PY
```

- [ ] **Step 3: Commit**

```bash
git add skills/migration-monitor/references/data-parity.md
git commit -m "docs(migration-monitor): add data-parity reference with SQL queries and row-count rules"
```

---

## Task 8: references/divergence-report.md

**Purpose:** Phase 7 — report JSON schema (as inline example), severity rubric, remediation hints, human-summary format.

**File:** `skills/migration-monitor/references/divergence-report.md`

### Required sections

1. **Purpose** (~5 lines).

2. **Report JSON schema** (~60 lines) — inline JSON example showing the full structure:

   ```json
   {
     "reportVersion": "1",
     "generatedAt": "2026-04-24T15:00:00Z",
     "generatedBy": "falcon migration-monitor",
     "summary": {
       "critical": 2,
       "major": 5,
       "minor": 12,
       "skippedPhases": ["data"]
     },
     "skipReasons": {
       "data": "SOURCE_DB_URI and TARGET_DB_URI not set in env"
     },
     "phases": {
       "infra": [
         {
           "id": "infra-001",
           "phase": "infra",
           "type": "compute.undersized",
           "severity": "major",
           "resourcePair": {"vmware_id": "vm-1001", "aws_id": "i-0abc"},
           "description": "Target EC2 has 2 vCPU vs source VM 4 vCPU — 0.5× ratio, below 0.8 floor",
           "evidence": {"source_vcpu": 4, "target_vcpu": 2, "ratio": 0.5, "floor": 0.8},
           "remediationHint": "Resize instance to at least t3.large (2→4 vCPU): `aws ec2 modify-instance-attribute --instance-id i-0abc --instance-type Value=t3.large`"
         }
       ],
       "endpoint": [ ... ],
       "metrics": [ ... ],
       "data": []
     }
   }
   ```

3. **Severity rubric** (~30 lines)
   - **Critical:** functional break suspected. Examples: missing DB table on target, HTTP status mismatch, encryption disabled where required, metrics >50% regression.
   - **Major:** observable regression, not functional. Examples: 10–30% performance regression, missing security header, security-rule deficit, body structural diff.
   - **Minor:** cosmetic or within tolerance band. Examples: tag diff, extra column on target, 5% row count drift, content-charset diff.

4. **Remediation hints table** (~80 lines) — embed at least 15 rows covering common divergence types. Representative minimum:

   | Divergence type | Remediation hint template |
   |-----------------|---------------------------|
   | `compute.undersized` | Resize to instance type `<suggested>`: `aws ec2 modify-instance-attribute --instance-id <id> --instance-type Value=<type>`. Web-search current `AWS EC2 instance types <region>` before applying. |
   | `compute.wrong-architecture` | Architecture mismatch — usually requires re-provisioning. Verify AMI architecture. |
   | `storage.capacity-deficit` | Extend EBS volume: `aws ec2 modify-volume --volume-id <vol> --size <gb>`. Re-provision filesystem with `growpart` + `resize2fs`/`xfs_growfs`. |
   | `storage.encryption-missing` | Must re-provision — EBS encryption cannot be added in-place. Create snapshot, create encrypted copy, re-attach. |
   | `security.inbound.missing-port` | `aws ec2 authorize-security-group-ingress --group-id <sg> --protocol tcp --port <port> --cidr <cidr>` |
   | `security.outbound.missing-destination` | `aws ec2 authorize-security-group-egress --group-id <sg> ...` |
   | `tags.missing-required` | `aws ec2 create-tags --resources <id> --tags Key=<k>,Value=<v>` |
   | `endpoint.status-mismatch` | Check ALB target-group health: `aws elbv2 describe-target-health --target-group-arn <arn>`. Inspect app-side config, reverse proxy rules. |
   | `endpoint.reachability-regression` | Verify security group / NACL / route table / subnet; confirm LB listener matches source port. |
   | `endpoint.tls-cert-expired` | Renew cert. If using ACM, check `aws acm list-certificates --certificate-statuses EXPIRED`. |
   | `metrics.cpu.regression` | Scaling candidate — `aws ec2 modify-instance-attribute` to next tier, or enable CloudWatch alarm-based auto-scaling. Investigate noisy-neighbor on shared tenancy. |
   | `metrics.latency.regression` | Check ALB/target-group latency panels; inspect app-side config for cold-start, N+1 queries, DB round-trip count. |
   | `metrics.error-rate.regression` | `aws logs filter-log-events` on ALB access logs + app logs. |
   | `data.table.missing-on-target` | Migration incomplete — restart replication task or manual schema recreate. `aws dms describe-replication-tasks` to check DMS state. |
   | `data.rowcount.deficit` | If DMS CDC is running, wait for lag to drain. Otherwise investigate replication errors via `aws dms describe-replication-tasks --query '...'`. |
   | `data.column.missing-on-target` | Schema regression — compare schemas via `\d+ <table>` (PG) or `SHOW CREATE TABLE` (MySQL); reapply schema changes. |
   | `metrics.unmapped` | Add to mapping table in future skill release, or manually correlate. No automated action. |

5. **Human-summary table format** (~30 lines) — sample rendering showing severity-first grouping:

   ```
   Migration Monitor Report — generated 2026-04-24T15:00:00Z

   Summary: 2 critical, 5 major, 12 minor  |  skipped: [data]

   ── CRITICAL ──
   [infra-003] vm-1001 → i-0abc: encryption disabled on target EBS
              Remediation: re-provision with encrypted snapshot
   [endpoint-001] web-01.corp → web-01-alb: HTTP 500 on target, 200 on source
              Remediation: check ALB target health + app logs

   ── MAJOR ──
   [infra-001] vm-1001 → i-0abc: undersized (2 vCPU vs 4)
   ...
   ```

6. **Generation notes** (~15 lines) — how Falcon builds the report:
   - Aggregate divergences from each phase's output.
   - Count by severity for `summary`.
   - Populate `skippedPhases` and `skipReasons` from Phase 0 surface-availability results.
   - Render JSON first (machine-parseable), then the human summary table at Gate 3.

### Steps

- [ ] **Step 1: Write the file per the outline.** The remediation table is the centerpiece — must have ≥15 rows.

- [ ] **Step 2: Validate JSON examples parse**

```bash
python3 <<'PY'
import re, json, sys
content = open("skills/migration-monitor/references/divergence-report.md").read()
blocks = re.findall(r"```json\n(.*?)```", content, re.DOTALL)
failed = 0
for i, b in enumerate(blocks):
    try: json.loads(b)
    except Exception as e: print(f"JSON block {i} FAILED: {e}"); failed += 1
print(f"{len(blocks)} JSON blocks, {failed} failures")
sys.exit(1 if failed else 0)
PY
```

- [ ] **Step 3: Structural validation**

```bash
for section in \
  "## Report JSON schema" \
  "## Severity rubric" \
  "## Remediation hints" \
  "## Human-summary table format"
do
  grep -qF "$section" skills/migration-monitor/references/divergence-report.md \
    || { echo "MISSING: $section"; exit 1; }
done
echo "OK"
```

- [ ] **Step 4: Commit**

```bash
git add skills/migration-monitor/references/divergence-report.md
git commit -m "docs(migration-monitor): add divergence-report reference with schema, rubric, and remediation hints"
```

---

## Task 9: Final validation

**Purpose:** Pre-PR verification. Run the full CI validator locally + structural checks + embedded-code sanity across all references.

**Files:** None to create.

### Steps

- [ ] **Step 1: Run full CI validator locally** (mirrors `.github/workflows/validate.yml`)

```bash
node --input-type=module -e '
import { readdirSync, readFileSync } from "fs";
import { join } from "path";
import { parse as parseYaml } from "yaml";

const SKILLS_DIR = "skills";
const NAME_RE = /^[a-z0-9][a-z0-9-]{0,63}$/;
const MAX_LINES = 500;
let errors = 0;
const fail = (s, m) => { console.error(`❌ ${s}: ${m}`); errors++; };

const dirs = readdirSync(SKILLS_DIR, { withFileTypes: true })
  .filter(d => d.isDirectory()).map(d => d.name);

for (const dir of dirs) {
  if (!NAME_RE.test(dir)) fail(dir, "bad name");
  const p = join(SKILLS_DIR, dir, "SKILL.md");
  let content;
  try { content = readFileSync(p, "utf-8"); }
  catch { fail(dir, "missing SKILL.md"); continue; }
  const lines = content.split("\n").length;
  if (lines > MAX_LINES) fail(dir, `${lines} > ${MAX_LINES}`);
  const m = content.match(/^---\n([\s\S]*?)\n---/);
  if (!m) { fail(dir, "no frontmatter"); continue; }
  let fm; try { fm = parseYaml(m[1]); } catch(e) { fail(dir, "bad yaml"); continue; }
  if (!fm.name) fail(dir, "no name");
  if (!fm.description) fail(dir, "no description");
  if (fm.name && fm.name !== dir) fail(dir, `name/dir mismatch`);
}
if (errors) { console.error(`\n${errors} error(s)`); process.exit(1); }
console.log(`✅ ${dirs.length} skills validated`);
'
```

Expected: `✅ 17 skills validated` (16 existing + migration-monitor).

- [ ] **Step 2: Verify all required sections across every reference**

```bash
set -e
SKILL=skills/migration-monitor
declare -A req=(
  [references/probe-and-map.md]="## Phase 0: Capability Probe Recipes|## Phase 1: Multi-signal Workload Mapping|## Phase 2: Discovery Command Lists|## Mapping output shape"
  [references/infra-parity.md]="## Compute shape|## Disk / storage|## Network|## Tags"
  [references/endpoint-parity.md]="## Endpoint pair derivation|## HTTP probe recipe|## TCP probe recipe|## TLS probe recipe"
  [references/metrics-parity.md]="## Metric-source detection|## Per-source catalog enumeration|## Metric-name mapping table|## Statistical comparison|## Percentile computation recipe|## Unmapped metric handling"
  [references/data-parity.md]="## Supported engines|## URI format and credential handling|## Table inventory queries|## Row count queries|## Column schema queries|## Divergence classification"
  [references/divergence-report.md]="## Report JSON schema|## Severity rubric|## Remediation hints|## Human-summary table format"
)
for f in "${!req[@]}"; do
  IFS='|' read -ra sections <<< "${req[$f]}"
  for s in "${sections[@]}"; do
    grep -qF "$s" "$SKILL/$f" || { echo "❌ $f missing: $s"; exit 1; }
  done
  echo "✅ $f"
done
echo "All reference sections present."
```

- [ ] **Step 3: Validate JSON in every reference**

```bash
python3 <<'PY'
import re, json, glob, sys
failed = 0
for f in glob.glob("skills/migration-monitor/references/*.md") + ["skills/migration-monitor/SKILL.md"]:
    content = open(f).read()
    blocks = re.findall(r"```json\n(.*?)```", content, re.DOTALL)
    for i, b in enumerate(blocks):
        try: json.loads(b)
        except Exception as e:
            print(f"❌ {f} JSON block {i}: {e}")
            failed += 1
print(f"JSON check: {failed} failures" if failed else "✅ All JSON blocks parse")
sys.exit(1 if failed else 0)
PY
```

- [ ] **Step 4: Validate SQL in data-parity.md** (if sqlparse available)

```bash
python3 <<'PY'
try:
    import sqlparse
except ImportError:
    print("sqlparse not installed — skipping SQL validation"); import sys; sys.exit(0)
import re, sys
content = open("skills/migration-monitor/references/data-parity.md").read()
blocks = re.findall(r"```sql\n(.*?)```", content, re.DOTALL)
print(f"{len(blocks)} SQL blocks found")
for i, b in enumerate(blocks):
    parsed = sqlparse.parse(b)
    if not parsed or not parsed[0].tokens:
        print(f"Block {i} failed"); sys.exit(1)
print("✅ All SQL blocks parse")
PY
```

- [ ] **Step 5: Shell-syntax check for embedded bash blocks** (optional — skip if `shellcheck` not installed)

```bash
if command -v shellcheck >/dev/null 2>&1; then
  python3 <<'PY'
import re, subprocess, glob, tempfile, os, sys
failed = 0
for f in glob.glob("skills/migration-monitor/references/*.md"):
    content = open(f).read()
    blocks = re.findall(r"```bash\n(.*?)```", content, re.DOTALL)
    for i, b in enumerate(blocks):
        # Strip template placeholders like <region>, $VAR — shellcheck handles $VAR
        b_sanitized = re.sub(r"<([a-z_][a-z_0-9-]*)>", "_PLACEHOLDER", b)
        with tempfile.NamedTemporaryFile("w", suffix=".sh", delete=False) as tf:
            tf.write("#!/usr/bin/env bash\n" + b_sanitized)
            tp = tf.name
        try:
            r = subprocess.run(["shellcheck", "-S", "error", tp], capture_output=True, text=True)
            if r.returncode != 0:
                print(f"{f} block {i}: {r.stdout.strip()}"); failed += 1
        finally:
            os.unlink(tp)
print(f"shellcheck: {failed} errors" if failed else "✅ All bash blocks shellcheck-clean (error level)")
sys.exit(1 if failed else 0)
PY
else
  echo "shellcheck not installed — skipping"
fi
```

- [ ] **Step 6: Line-budget summary**

```bash
wc -l skills/migration-monitor/SKILL.md skills/migration-monitor/references/*.md
```

Expected: `SKILL.md` ≤500; total ≥ 1800 lines of content across references.

- [ ] **Step 7: No commit.** Validation only. If any step fails, return to the offending task to fix and re-run Task 9.

---

## Task 10: Open pull request

**Purpose:** Push the branch and open the PR.

### Steps

- [ ] **Step 1: Push the branch**

```bash
git push -u origin feat/migration-monitor
```

- [ ] **Step 2: Open PR**

```bash
gh pr create --title "feat: migration-monitor skill (VMware → AWS parity)" --body "$(cat <<'EOF'
## Summary

- Adds `skills/migration-monitor/` — a post-cutover parity-assurance skill for VMware → AWS migrations.
- **Pure knowledge skill**: no shipped scripts or binaries. Falcon handles discovery natively; this skill provides workflow, scoring algorithms, tolerance rules, metric-equivalence tables, SQL queries, and report schema.
- 7-phase workflow with 3 user-approval gates (mode / mapping / divergence disposition).
- Four parity surfaces: infrastructure, endpoint (HTTP/TCP/TLS), metrics (with auto-discovery), SQL database data.
- Multi-signal heuristic workload mapping — tags are one signal among many, never required.

**Spec:** `docs/superpowers/specs/2026-04-24-migration-monitor-design.md`
**Plan:** `docs/superpowers/plans/2026-04-24-migration-monitor.md`

## Test plan

- [ ] CI `validate.yml` passes on this branch.
- [ ] `SKILL.md` ≤500 lines.
- [ ] Manual walkthrough in Falcon: capability probe runs, Gate 1 fires with correct surface checklist.
- [ ] Given synthetic inventories, mapping algorithm produces expected pairings (high-confidence auto-paired, medium presented as candidates).
- [ ] Metrics auto-discovery does not prompt for metric names.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

- [ ] **Step 3: Record PR URL**

The PR URL prints from `gh pr create`. Paste it below:

**PR URL:** _(paste after creating)_

---

## Appendix: Spec coverage audit

| Spec section | Implemented by |
|--------------|----------------|
| Architecture (no `scripts/`, 6 references) | Task 1 (scaffold) + Tasks 3–8 (references) |
| Frontmatter | Task 1 (stub) → Task 2 (final, embedded) |
| SKILL.md orchestrator (Phases 0–7, 3 gates, constraints) | Task 2 |
| Phase 0 capability probe recipes | Task 3 |
| Phase 1 multi-signal mapping algorithm | Task 3 |
| Phase 2 discovery command lists | Task 3 |
| Canonical inventory JSON shape | Task 3 |
| Phase 3 infra tolerance rules | Task 4 |
| Phase 4 HTTP/TCP/TLS recipes | Task 5 |
| Phase 5 metric source detection + catalog enum + mapping table + stats | Task 6 |
| Phase 6 SQL queries per engine + row-count tolerance + schema | Task 7 |
| Phase 7 report schema + severity rubric + remediation hints | Task 8 |
| CI validation + structural checks + embedded-code sanity | Task 9 |
| PR | Task 10 |

**Deferred per spec (v1.1+):**
- Continuous mode wrapper.
- NoSQL data parity.
- Oracle DB support.
- File-system parity.
- OpenShift destination.
