# Migration Monitor Skill — Design

**Status:** Approved, pending review
**Date:** 2026-04-24
**Skill name:** `migration-monitor`
**Repo:** `neubirdai/falconclaw-skills`
**Supersedes:** `docs/superpowers/specs/2026-04-23-migration-assistant-design.md` (scope pivoted from drive-the-migration to assure-the-migration)

## Overview

The `migration-monitor` skill provides post-cutover parity assurance for VMware → AWS migrations inside the Falcon TUI. Given a set of workloads that have been migrated, it captures baseline state on both sides, diffs infrastructure shape, endpoint behavior, metrics, and database data parity, then emits a consolidated divergence report. The skill runs as a one-shot interactive analysis; users re-invoke it as needed.

**Falcon is read-only and handles all discovery natively.** The skill ships as pure knowledge: workflow structure, decision logic, scoring algorithms, metric-equivalence tables, tolerance rules, and reporting formats. Falcon's built-in capabilities (AWS CLI, `govc`, file readers, web search, shell, API calls) perform the actual probing, parsing, and command execution. This skill does **not** ship probe scripts, binaries, or helper executables — it works within the boundaries of what Falcon already discovers.

## Goals

1. Tell the user, with evidence, whether migrated workloads look and behave like their VMware sources.
2. Four comparison surfaces in v1: infrastructure shape, endpoint behavior, metrics/SLOs, SQL database data parity.
3. **Best-effort automatic workload mapping** from VMware side to AWS side using multiple signals (name, IP, hostname, DNS, OS family, sizing plausibility); ask the user only for ambiguous or unresolved pairings.
4. **Zero-prompt metric source discovery.** Falcon scans env vars for telemetry credentials, enumerates available metrics per source, and maps them to CloudWatch equivalents via a shipped table. No metric queries or names are solicited from the user.
5. **Work within Falcon's native discovery surface** — no shipped scripts. Every probe, command, and analysis step is either a direct Falcon capability or a concrete recipe encoded in a reference file that Falcon follows.
6. Extensibility: future destinations (OpenShift, GCP) or additional surfaces drop into the flat `references/` layout without restructure.

## Non-goals (v1)

- **Driving a migration** (discovery, 6Rs decisions, template conversion, cutover runbooks, decommissioning). Fully out of scope — this is the explicit pivot.
- **Continuous monitoring mode.** v1 is one-shot only. Users re-invoke the skill as needed. Continuous mode (cron-ready wrapper generation) is deferred to v1.1.
- **NoSQL data parity** (Mongo, Cassandra, DynamoDB). Different consistency models, different validation semantics. v1 covers SQL DBs only (MySQL, PostgreSQL, SQL Server).
- **File / object-store parity** (NFS → EFS file hash comparisons, S3 inventory diffs). Out of scope — application-specific and easy to do incorrectly at scale.
- **Mutations of any kind.** No tag writes, no remediation commands executed.
- **Cost parity** — AWS Cost Explorer covers this.
- **Non-VMware sources** (Hyper-V, bare metal) and non-AWS destinations. Deferred.
- **Synthetic workload generation** — read-only by design means no synthetic traffic or load testing.
- **Shipping discovery scripts or binaries.** Falcon natively discovers; this skill provides the knowledge layer, not the execution layer.
- **Relying on migration-tracking tags as primary mapping** — real migrations don't reliably apply these.

## Scope decisions

| # | Decision | Rationale |
|---|----------|-----------|
| 1 | Skill name: `migration-monitor` | Matches user intent; precise. |
| 2 | Form factor: single skill with orchestrator `SKILL.md` + flat `references/` | Documented FalconClaw pattern; no `scripts/` because Falcon handles execution. |
| 3 | Delivery mode: one-shot only in v1 | Continuous wrapper deferred to v1.1. |
| 4 | Parity surfaces v1: infrastructure, endpoint, metrics/SLO, DB data (SQL-only). Cost flags as a parallel observation in Phase 7 (see decision #14). | NoSQL parity, file-system parity explicitly out. Cost *parity* (source-vs-target billing comparison) is out — but target-side cost *analysis* is in (decision #14). |
| 5 | Workload mapping: multi-signal heuristic with confidence scoring; ask user only for ambiguities | Real migrations don't reliably tag. Signals: exact/fuzzy name, private/public IP, hostname, DNS, OS family, sizing plausibility. Auto-pair ≥80; present candidates 40–79; solicit <40. |
| 6 | Metrics discovery: fully automatic, zero user prompting | Falcon scans env vars, enumerates metrics via source APIs, maps to CloudWatch via shipped table. |
| 7 | Comparison semantics: equivalent within tolerance, not exact equality | Right-sizing changes shapes; rules encoded per parity reference. |
| 8 | Boundaries: standalone | Skill owns monitor workflow end-to-end. |
| 9 | Three user-approval gates | No mutation boundaries; only mode / mapping / divergence-disposition need approval. |
| 10 | Flat `references/` layout | Consistent with sibling skills; 6 references in v1. |
| 11 | No shipped scripts / schemas-as-files | Falcon discovers natively; schemas live as inline examples in references. Avoids duplicating what Falcon already does. |
| 12 | Mapping score is uncapped additive | Signals sum to an unbounded score. Thresholds (≥80 / 40–79 / <40) operate on raw sum; higher scores indicate stronger match even above 100. |
| 13 | GitHub repo scan as a Phase 2 source, ranked second after live vCenter | Real teams keep RVTools snapshots and IaC in the same repo as their migration plan. When `gh auth status` succeeds AND `MIGRATION_MONITOR_VMWARE_REPO` is set (or user names the repo at Gate 1), Falcon scans the repo for RVTools exports + raw IaC (Terraform vSphere, VCF/Aria, OVF) before falling back to local cwd. The repo is the team's source of truth. Sensitive files (`.tfvars`, `.env`, `*.pem`, `*.key`, `id_rsa*`, `*.secret*`) are skipped — must never round-trip into reports. |
| 14 | Cost analysis as a parallel deliverable in Phase 7 (alongside divergence report) | High-impact AWS cost gotchas (EKS extended-support fees, EC2 previous-generation, unattached EIPs/EBS, gp2-vs-gp3, CW Logs without retention, NAT Gateway throughput, RDS Multi-AZ on non-prod) commonly cost teams thousands of dollars per month and are missed without explicit attention. ~10 well-known patterns with concrete detection rules ship in `references/cost-analysis.md`. Cost flags use the same severity scale as divergences but are calibrated to monthly impact ($), not functional severity. Cost flags are NEVER blockers — surfaced as heads-ups; user dispositions at Gate 3. Falcon MUST web-search current AWS pricing before emitting specific dollar figures and MUST emit the standard disclaimer (estimates use retail rates; RIs/Savings Plans/EDP/regional differences alter actuals). v1 also extends Phase 2 AWS discovery with cost-relevant API calls (`aws eks describe-cluster`, `aws ec2 describe-snapshots`, `aws logs describe-log-groups`, NAT throughput metrics). |

## Architecture

### Directory layout

```
skills/migration-monitor/
├── SKILL.md                          (~280 lines, hard cap 500)
└── references/
    ├── probe-and-map.md              (~450 lines) — Phases 0–2: capability probe, discovery command lists, mapping scoring algorithm, Gate 2 interaction
    ├── infra-parity.md               (~300 lines) — Phase 3 rules + tolerance thresholds
    ├── endpoint-parity.md            (~300 lines) — Phase 4 curl/nc/openssl recipes + diff rules
    ├── metrics-parity.md             (~350 lines) — Phase 5 metric-source detection + catalog enumeration + mapping table + statistical comparison
    ├── data-parity.md                (~300 lines) — Phase 6 SQL queries per engine + tolerance rules
    └── divergence-report.md          (~250 lines) — Phase 7 report schema (as inline JSON example) + severity rubric + remediation hints
```

No `scripts/` subdirectory.

Line counts are ceilings, not targets.

### Frontmatter

```yaml
---
name: migration-monitor
description: "Post-cutover parity assurance for VMware → AWS migrations: captures live state on both sides via Falcon's native read-only discovery, diffs infrastructure shape (compute/storage/network/tags/security), endpoint behavior (HTTP/TCP/TLS response parity), metrics (CPU/memory/latency/error-rate equivalence within tolerance), and SQL database data parity (table presence, row counts, column schema), then emits a consolidated divergence report. Workload pairings are auto-discovered via multi-signal heuristic scoring (name, IP, DNS, hostname, OS, sizing) with user confirmation only for ambiguous matches; metric sources and available metrics are auto-discovered. Falcon is read-only and runs all probes natively; this skill provides the workflow, scoring algorithms, and parity rules. One-shot analysis; re-invoke as needed. Use after a VMware → AWS cutover to verify the target workloads behave like the originals."
tags: [migration, monitor, parity, vmware, aws, cloud]
homepage: https://github.com/neubirdai/falconclaw-skills/tree/main/skills/migration-monitor
metadata: {"neubird":{"emoji":"🔎","requires":{"anyBins":["aws","govc"]}}}
---
```

## SKILL.md orchestrator

### Structure

```markdown
---
<frontmatter>
---

# Migration Monitor (VMware → AWS)

<5-line preamble: scope, Falcon handles discovery natively, read-only, one-shot>

## Workflow Overview
<ASCII flow of 7 phases with 3 gates>

## Phase 0: Capability Probe (REQUIRED FIRST STEP)
<probe commands Falcon runs; available-surfaces checklist>

## Phase 1: Workload Mapping (Multi-signal Heuristic)
<mapping algorithm; Gate 2 interaction>

## Phase 2: Baseline Capture
<Falcon enumerates source + target inventories using its native discovery>

## Phases 3–7 (brief)
<one paragraph each, pointer to reference>

## Reference Guide
<table>

## Output Conventions
<copy-ready bundle format, divergence report format>

## Constraints (MUST DO / MUST NOT DO)
```

### Phase 0 — Capability probe

Runs first every session. Falcon executes the probes (it has the CLI tools and env access natively):

| Probe | Approach | Signals available |
|-------|----------|-------------------|
| AWS reachable | `aws sts get-caller-identity` | account ID, active region |
| vCenter reachable | `govc about` (only if `GOVC_URL` + `GOVC_USERNAME` set) | live VMware discovery |
| Static VMware exports | scan cwd for `RVTools*.xlsx` / `RVTools*.csv` | fallback if no live vCenter |
| GitHub VMware configs | `gh auth status` succeeds AND (`MIGRATION_MONITOR_VMWARE_REPO=<owner>/<repo>` set OR user names the repo at Gate 1) | scan repo for RVTools exports + raw IaC (Terraform vSphere, VCF/Aria, OVF) |
| Metric sources | env-var scan: `PROMETHEUS_URL`, `DATADOG_API_KEY` + `DATADOG_APP_KEY`, `NEW_RELIC_API_KEY` + `NEW_RELIC_ACCOUNT`, `VROPS_URL` + `VROPS_USERNAME` | which metric sources are queryable |
| Endpoint probing | `curl --version` | HTTP/TCP diffing available |
| TLS probing | `openssl version` | TLS cert checks available |
| DB parity | env-var scan: `SOURCE_DB_URI` + `SOURCE_DB_PASSWORD` and `TARGET_DB_URI` + `TARGET_DB_PASSWORD` | DB parity available |

The skill enumerates **available surfaces** (not a fixed "mode" enum) from probe outcomes:

- **Required:** AWS + (live vCenter OR GitHub repo with VMware configs OR RVTools export in cwd OR raw VMware IaC in cwd). At least one VMware-side source must produce an inventory; the four are tried in priority order (live > GitHub > local-cwd-export > local-cwd-IaC).
- **Optional surfaces** (skipped if unavailable, logged with reason):
  - Endpoint parity (requires `curl`)
  - TLS checks (requires `openssl`)
  - Metrics parity (requires ≥1 metric source)
  - DB parity (requires both DB URIs + passwords)

Skill prints the surface checklist at **Gate 1**. User confirms or adjusts env and re-runs Phase 0.

### Phase 1 — Workload mapping (multi-signal heuristic)

**The mapping problem:** given a VMware inventory (M workloads) and an AWS inventory (N resources), produce `{vmware-workload → aws-resource}` pairings with best-effort confidence scoring. Do not rely on any single signal.

**Signals** (each contributes an additive score — not capped at 100):

| Signal | Weight | Extraction |
|--------|--------|------------|
| Exact name match | +40 | VM name equals AWS resource natural identifier (EC2 `Name` tag; RDS `DBInstanceIdentifier`; ALB/NLB name; EFS `CreationToken`) — case-insensitive |
| Fuzzy name match (mutex with exact) | +20 | Token similarity ≥0.7 OR Levenshtein ≤3, only if exact did not hit |
| Private IP match | +30 | Any VMware VM IPv4 equals any AWS ENI private IP (EC2) or endpoint address (RDS) |
| Public IP match | +30 | VMware public IP equals AWS EIP attached to target |
| Hostname match | +25 | VMware guest hostname equals AWS Route 53 record target or ALB/NLB DNS name |
| OS family match | +10 | Source Linux → target Linux, Windows → Windows; −20 on explicit mismatch |
| Sizing plausibility | +10 | Target vCPU within 2× source AND target memory within 2× source |
| Migration-tracking tag present | +15 | AWS resource has `migration-source-vm-id`/`migration-source-name` matching the VMware VM. One signal among many. |
| Annotation/notes reference | +5 | VMware annotation contains target name/ID, or AWS `Description` contains source name |

**Score is uncapped additive.** Realistic matches land in 50–120 range. Thresholds:

- **≥80** — high confidence: auto-pair, do not prompt.
- **40–79** — medium confidence: present top 3 candidates to user at Gate 2, user picks.
- **<40 or no candidates** — unresolved: present to user, user either provides pairing or declares out-of-scope.

**Orphans:**
- VMware VMs with no candidate ≥40 → "unresolved source" list.
- AWS resources no VMware VM mapped to → "unresolved target" list (could be native AWS workloads).

**User interaction at Gate 2:** Falcon presents three grouped lists (auto-paired collapsed, ambiguous with top 3 per item, unresolved), and the user overrides any pairing before proceeding.

### Phase 2 — Baseline capture

Falcon enumerates state on both sides using its native discovery. Discovery is comprehensive (full account / region for AWS, full vCenter for VMware), not filtered — because we don't trust any single signal (including tags) to scope correctly.

**VMware side discovery — source priority** (try in order; stop at first that yields an inventory):

1. **Live vCenter** via `govc *.info` — highest fidelity, current state. VM inventory: name, MoRef, guest OS, guest hostname (via VMware Tools), primary + secondary IPs (all NICs), MAC addresses, vCPUs, memoryMB, disk capacities, annotations/notes, tags. Host + cluster + datastore + network context.
2. **GitHub repo scan** — when `gh auth status` ok and `MIGRATION_MONITOR_VMWARE_REPO` is set (or user names a repo at Gate 1). Falcon uses `gh api` to fetch `RVTools_*.xlsx` / `RVTools_*.csv` from the repo; treats the repo as the team's source of truth. Falls through to raw-IaC scan in the same repo if no exports found.
3. **RVTools export in cwd** — `vInfo` / `vCPU` / `vMemory` / `vDisk` / `vNetwork` / `vHost` / `vDatastore` tabs merged into the same normalized view.
4. **Raw VMware IaC** (cwd or GitHub) — Terraform `vsphere_virtual_machine` resources, VCF Automation `Cloud.vSphere.Machine` YAML, Aria Automation blueprints, OVF descriptors. Lower fidelity (declared intent, not live state) — Phase 1 mapping confidence is reduced for these sources because IPs and runtime context are typically absent. Raw-IaC parsers are documented in `references/probe-and-map.md`.

**Sensitive-content guard.** When scanning a GitHub repo, Falcon must skip `.tfvars`, `.env`, `*.secret*`, `*.pem`, `*.key`, and `id_rsa*` files entirely. These typically contain credentials and must not round-trip into divergence reports or remediation hints.

**AWS side discovery** (Falcon runs these):
- EC2 instances (`aws ec2 describe-instances`) — Name tag, ENI private/public IPs, tags, instance type, platform (Windows/Linux).
- RDS instances + clusters (`aws rds describe-db-instances`, `describe-db-clusters`) — identifier, engine, endpoints, tags.
- Load balancers — ALB (`aws elbv2 describe-load-balancers`, `describe-target-groups`, `describe-target-health`), NLB (same), classic ELB (`aws elb describe-load-balancers`).
- EFS (`aws efs describe-file-systems`), FSx (`aws fsx describe-file-systems`).
- ElastiCache (`aws elasticache describe-cache-clusters`), MSK (`aws kafka list-clusters-v2`).
- Security groups (`aws ec2 describe-security-groups`), subnets, VPCs.
- Route 53 records (`aws route53 list-hosted-zones`, `list-resource-record-sets`).

Both sides' inventories are structured in memory in a canonical shape documented as an inline JSON example in `references/probe-and-map.md` — Falcon produces this shape from raw discovery output; the shape is a reference for consistency, not a runtime validator.

### Phases 3–7

Each phase loads a specific reference file. Phases are skipped if the corresponding surface was unavailable.

- **Phase 3 — Infrastructure parity.** Load `references/infra-parity.md`. Compare compute shape, disk, network, tags, security rules, public/private exposure. Falcon applies the per-attribute tolerance rules against both inventories.
- **Phase 4 — Endpoint parity.** Load `references/endpoint-parity.md`. For each endpoint pair derived from Phase 1 mapping (ALB DNS / NLB DNS / Route 53 record / EIP), Falcon emits + executes `curl`, `nc`, and `openssl` probes per the recipes in the reference.
- **Phase 5 — Metrics parity.** Load `references/metrics-parity.md`. Falcon uses the env-detected metric sources to enumerate metrics per resource, maps source metric names to CloudWatch equivalents via the shipped table, and computes statistical summaries (p50/p95/p99/mean) over a 7-day comparison window (user-overridable at session start).
- **Phase 6 — Data parity (SQL DBs).** Load `references/data-parity.md`. Falcon emits + executes engine-specific SQL queries (via `mysql -e`, `psql -c`, `sqlcmd -Q`) on both sides. Compares table inventory, row counts per table, column schema.
- **Phase 7 — Divergence report.** Load `references/divergence-report.md`. Consolidate outputs of Phases 3–6 into a classified report. Present at Gate 3.

### Decision gates (three)

| # | Gate | What Falcon presents | What user confirms |
|---|------|---------------------|---------------------|
| 1 | After capability probe (Phase 0) | Available-surfaces checklist | Mode is right; env is sufficient |
| 2 | After workload mapping (Phase 1) | Auto-paired + ambiguous + unresolved lists | Mapping correct + scope-in/out |
| 3 | After divergence report (Phase 7) | Classified divergences | Per-divergence disposition (accepted / investigate / blocker) |

No mutation gates — no mutations in this skill.

### Output conventions

- **Copy-ready bundles** for any remediation script the user might run: fenced code block with `# file: <destination-path>` header.
- **Commands** numbered with **Expected output** blocks where verification matters.
- **Divergence report** emitted as structured JSON (schema defined as an inline example in `references/divergence-report.md`) and rendered as a human-readable summary table at Gate 3.

### Constraints

**MUST DO:**
- Phase 0 capability probe first every session.
- Present results and wait for user approval at each of three gates before advancing.
- **Use multi-signal heuristic matching for Phase 1; never rely on tags as the sole signal.**
- **Auto-discover metric sources and available metrics from env; never ask the user to name metrics or provide metric queries.**
- **Emit precise commands for Falcon to execute** — every discovery, probe, and comparison step is a concrete CLI invocation the reference specifies. No narrative "Falcon figures it out" handwaving.
- Web-search current AWS CloudWatch metric namespaces before finalizing any metric mapping — namespaces/metric names evolve.
- Classify every divergence by severity using tolerance rules in the relevant parity reference.
- Skip phases explicitly when their surface is unavailable (emit `Phase N skipped — reason X` in the report; never silently).

**MUST NOT DO:**
- Execute any mutation.
- Ship probe scripts, helper binaries, or executable code under a `scripts/` directory. Falcon discovers natively.
- Generate synthetic traffic.
- Rely on migration-tracking tags alone for workload mapping.
- Prompt the user to enumerate metrics, metric queries, or metric names.
- Guess metric-name equivalence without consulting `metrics-parity.md`; if a source metric has no mapping, emit it as "unmapped — manual review."
- Execute SQL that isn't `SELECT` (including `SELECT INTO`, `UPDATE`, `DELETE`, DDL).

## Reference file contents

### `references/probe-and-map.md` (~450 lines)

The centerpiece for Phases 0–2.

- **Phase 0 capability-probe recipes** — exact CLI commands Falcon runs for each probe (listed above), with expected-output patterns and failure interpretations.
- **Phase 1 multi-signal matching algorithm** — the weighted signal table (above) plus:
  - How each signal is computed (with concrete examples).
  - Handling of multi-IP VMs (one VM has multiple NICs/IPs) and multi-ENI AWS instances.
  - Scoring tie-breakers (if two candidates tie, prefer same-region + same-OS).
  - Pseudo-code for the matching loop: input = (vmware_inventory, aws_inventory); output = (pairs, scores, ambiguous, unresolved).
  - Signal extraction per resource type (how to get "natural name" for EC2 vs RDS vs ALB).
- **Phase 2 discovery command lists** — the exact AWS CLI verbs and `govc` commands Falcon invokes. Includes canonical JSON shape for the in-memory inventory both sides produce (as an inline example, not a separate file).
- **Gate 2 interaction recipe** — how to present auto-paired / ambiguous / unresolved lists; what overrides look like; the final mapping structure passed to Phases 3–7.

### `references/infra-parity.md` (~300 lines)

Per-attribute comparison rules and tolerance thresholds.

- **Compute shape:** vCPU ratio target ≥ 0.8× source; memoryMB ratio target ≥ 0.8× source telemetry p95. OS family and architecture must match.
- **Disk:** total attached capacity within 10% of source (excluding OS disk growth on target). Disk count may differ.
- **Network:** outbound SG rules must permit at least the same destination/port set; inbound rules must permit the same source CIDRs on same ports; subnet count may differ; public-exposure parity preserved.
- **Tags:** parity is informational. Migration-tracking tag divergences classified minor/informational since v1 doesn't require them.
- **Encryption:** if source used disk encryption, target EBS volumes must have `encrypted=true`.

Each rule has a severity classification (critical/major/minor).

### `references/endpoint-parity.md` (~300 lines)

Precise probe recipes Falcon emits + executes, per pair.

- **HTTP:** exact curl invocations — `curl -sS -o body.out -D headers.out -w "%{http_code}" <url>` on both sides with optional auth via env. Compare: status, structural headers (`Content-Type`, `Cache-Control`, security headers), semantic body diff. Normalization rules: JSON keys sorted + volatile-field strip list (`timestamp`, `requestId`, `traceId`, `date`); HTML attribute-ordering-agnostic diff.
- **TCP:** `nc -z -w5 <host> <port>` on both sides. Reachability must match.
- **TLS:** `openssl s_client -connect <host>:443 -servername <host> < /dev/null | openssl x509 -noout -dates -subject -issuer`. Compare validity + subject CN + issuer family acceptability.

Per-probe tolerance rules and severity classification. Divergence JSON shape for Phase 7 consumption.

### `references/metrics-parity.md` (~350 lines)

- **Metric-source detection:** env-var → source-type mapping, per-source catalog API.
- **Per-source enumeration commands** — exact API calls Falcon makes:
  - Prometheus: `curl -s "$PROMETHEUS_URL/api/v1/label/__name__/values"`.
  - Datadog: `curl -H "DD-API-KEY: $DATADOG_API_KEY" -H "DD-APPLICATION-KEY: $DATADOG_APP_KEY" "https://api.datadoghq.com/api/v1/metrics?from=<epoch>"`.
  - New Relic: `curl -H "Api-Key: $NEW_RELIC_API_KEY" "https://api.newrelic.com/graphql" -d '{"query":"...NRQL..."}'`.
  - vROps: `curl -u $VROPS_USERNAME:$VROPS_PASSWORD "$VROPS_URL/suite-api/api/resources/{id}/stats"`.
  - CloudWatch (target side): `aws cloudwatch list-metrics --namespace AWS/EC2 --dimensions Name=InstanceId,Value=<id>`.
- **Metric-name mapping table** (authoritative, representative rows):

  | Source metric | CloudWatch equivalent | CloudWatch namespace |
  |---------------|----------------------|----------------------|
  | vROps `cpu\|usage_average` | `CPUUtilization` | `AWS/EC2` |
  | vROps `mem\|usage_average` | `mem_used_percent` | `CWAgent` |
  | vROps `disk\|usage_average` | `disk_used_percent` | `CWAgent` |
  | vROps `net\|throughput_usage_average` | `NetworkIn` + `NetworkOut` (sum) | `AWS/EC2` |
  | Prom `node_cpu_seconds_total{mode!="idle"}` (rate) | `CPUUtilization` | `AWS/EC2` |
  | Prom `node_memory_MemAvailable_bytes` | `mem_used_percent` (inverted) | `CWAgent` |
  | Prom `http_request_duration_seconds{quantile="0.95"}` | `TargetResponseTime` (p95) | `AWS/ApplicationELB` |
  | Prom `http_requests_total{status=~"5.."}` (rate) | `HTTPCode_Target_5XX_Count` | `AWS/ApplicationELB` |
  | Datadog `system.cpu.user` + `system.cpu.system` | `CPUUtilization` | `AWS/EC2` |
  | New Relic `system.cpuPercent` | `CPUUtilization` | `AWS/EC2` |
  | RDS Prom `pg_stat_database_tup_fetched` (rate) | `ReadIOPS` | `AWS/RDS` |

  (Full table in the reference.)

- **Comparison window:** default 7 days pre-cutover vs 7 days post-cutover. User overrides at session start.
- **Statistical comparison:** for each mapped metric, compute p50, p95, p99, mean over the window on both sides. Divergences: within ±10% = OK, 10–30% = minor, 30–50% = major, >50% = critical.
- **Unmapped metric handling:** if a source metric has no entry in the table, emit as "unmapped — manual review required."
- **Falcon computation note:** percentile math uses shell one-liners (`awk`/`jq`) or a small Python snippet emitted inline. No shipped script — the reference provides the exact snippet.

### `references/data-parity.md` (~300 lines)

SQL DB parity for DBs migrated to RDS/Aurora.

- **Scope:** table inventory, row counts, column schema for user-defined tables. System tables excluded.
- **Engines supported:** MySQL 5.7+, PostgreSQL 12+, SQL Server 2017+. Oracle deferred.
- **Required env:** `SOURCE_DB_URI`, `SOURCE_DB_PASSWORD`, `TARGET_DB_URI`, `TARGET_DB_PASSWORD`. URI format: `mysql://user@host:3306/db`, `postgresql://user@host:5432/db`, `mssql://user@host:1433/db`.
- **Query recipes** — exact SQL per engine Falcon emits + executes on both sides (via `mysql -e`, `psql -c`, `sqlcmd -Q` — each is a read-only CLI):

  **Table inventory (MySQL):**
  ```sql
  SELECT table_name FROM information_schema.tables
  WHERE table_schema = DATABASE() AND table_type = 'BASE TABLE'
  ORDER BY table_name;
  ```

  **Table inventory (PostgreSQL):**
  ```sql
  SELECT tablename FROM pg_catalog.pg_tables
  WHERE schemaname NOT IN ('pg_catalog','information_schema')
  ORDER BY tablename;
  ```

  **Table inventory (SQL Server):**
  ```sql
  SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES
  WHERE TABLE_TYPE = 'BASE TABLE' AND TABLE_SCHEMA NOT IN ('sys','INFORMATION_SCHEMA')
  ORDER BY TABLE_NAME;
  ```

  **Row count (all engines):** `SELECT COUNT(*) FROM <table>;`

  **Column schema (MySQL / PostgreSQL):**
  ```sql
  SELECT column_name, ordinal_position FROM information_schema.columns
  WHERE table_schema = <schema> AND table_name = <table>
  ORDER BY ordinal_position;
  ```

  **Column schema (SQL Server):** same as above, `table_schema` replaced by `TABLE_SCHEMA` and the schema predicate adjusted.

- **Checks:**
  - **Table inventory parity:** missing on target = critical; extra on target = minor (may reflect expected schema evolution).
  - **Row count parity:** within 1% (accounts for in-flight during probe). Strict mode (0%) available via user flag.
  - **Column schema parity:** column count + column names (not types — platform type-name drift expected). Missing column = critical; extra = minor.
- **Cutoff-timestamp option:** user specifies `cutoff_field` and `cutoff_value` to restrict row-count queries to `WHERE <cutoff_field> < <cutoff_value>` — handles CDC-in-flight scenarios.
- **Security:** Falcon never reads password from terminal input; only from the env vars. Reference states this as a MUST DO.

### `references/divergence-report.md` (~250 lines)

- **Report JSON schema** — defined as an inline JSON example. Top-level: `reportVersion`, `generatedAt`, `phases` (object with `infra`, `endpoint`, `metrics`, `data` — any may be absent if the phase was skipped), `summary` (critical/major/minor counts + `skippedPhases` list), `generatedBy`. Each divergence: `{id, severity, type, phase, resourcePair, description, remediationHint, evidence}`.
- **Human summary table format** — sample rendering with critical-first ordering.
- **Severity classification rubric:**
  - Critical: functional break suspected (missing table, endpoint status mismatch, encryption disabled where required).
  - Major: observable regression, not functional (latency ≥30% worse, body structural diff, missing security header).
  - Minor: cosmetic / within tolerance band (tag diff, extra column, latency 10–30% off).
- **Remediation hints per divergence type** (representative):
  - `compute.undersized` → suggest instance-type bump; web-search current AWS sizing guidance.
  - `security.inbound.missing-port` → provide `aws ec2 authorize-security-group-ingress` command for user to run.
  - `endpoint.status-mismatch` → check ALB target-group health, app-side config.
  - `metrics.cpu.regression` → scale-up candidate or noisy-neighbor investigation.
  - `data.row-count.deficit` → check DMS CDC lag or replication task errors.
  - `data.table.missing-on-target` → block — source table absent on target, migration incomplete.

## Success criteria

**Automated (verifiable in CI / local test):**
- `.github/workflows/validate.yml` passes for the skill.
- Every `SELECT` query in `data-parity.md` parses as valid SQL for its engine (via `sqlparse` or engine-specific dry-run syntax check, if CI can provide them).
- Every `curl` / `nc` / `openssl` invocation in `endpoint-parity.md` parses (shellcheck over extracted commands).
- Every JSON example (schema, fixture, report) in any reference parses cleanly via `python -m json.tool`.

**Manual (verified by walking the skill in Falcon):**
- Capability probe correctly classifies available surfaces given synthetic env vars.
- Given canned inventories, Falcon walks Phases 1–7 to produce a divergence report, stopping at each of the three gates.
- Phase 1 mapping auto-pairs high-confidence matches without prompting, presents top-3 candidates for medium-confidence, lists unresolved clearly.
- Metrics auto-discovery produces no user prompts.
- DB parity queries run cleanly against a test MySQL + PostgreSQL instance and produce expected divergences when tables/rows/columns differ.

## Extension plan

Adding a new metric source (e.g., Splunk): entry in metric-source detection + section in `metrics-parity.md` + rows in mapping table. No other changes.

Adding a new parity surface (e.g., cost parity in v1.1): new reference `cost-parity.md`, Phase inserted between metrics and data. No scripts needed.

Adding a new destination (e.g., OpenShift): destination-specific sections in each parity reference, a new `probe-and-map.md` section for OpenShift-side discovery commands. No scripts.

**v1.1 roadmap:**
- Continuous-mode wrapper generation (dropped from v1).
- NoSQL data parity (Mongo, DynamoDB).
- Oracle support in data-parity.
- File-system parity (EFS vs source NFS) via hash sampling.

## Open items / deferred

- **Metric comparison window default.** Starting at 7 days; per-workload overrides deferred.
- **TLS cert chain-trust comparison.** v1 compares validity + subject CN only.
- **Endpoint body diff's volatile-field strip list.** v1 ships default list; user-provided regex extensibility deferred.
- **DB engines.** v1 ships MySQL, PostgreSQL, SQL Server. Oracle + NoSQL deferred.
- **Large-inventory performance.** For accounts with 10k+ resources, full-account probing is slow. Pagination is respected; parallelization deferred. User can set an env override (e.g., `MIGRATION_MONITOR_VPC_ID`) documented in `probe-and-map.md` to scope probing.
- **Shell-vs-Python for percentile computation.** Reference ships an `awk` one-liner and a Python snippet; Falcon picks whichever fits the available environment.
