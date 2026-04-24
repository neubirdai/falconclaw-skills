# Migration Monitor Skill — Design

**Status:** Approved, pending review
**Date:** 2026-04-24
**Skill name:** `migration-monitor`
**Repo:** `neubirdai/falconclaw-skills`
**Supersedes:** `docs/superpowers/specs/2026-04-23-migration-assistant-design.md` (scope pivoted from drive-the-migration to assure-the-migration)

## Overview

The `migration-monitor` skill provides post-cutover parity assurance for VMware → AWS migrations inside the Falcon TUI. Given a set of workloads that have been migrated, it captures baseline state on both sides, diffs infrastructure shape, endpoint behavior, metrics, and database data parity, then emits a consolidated divergence report. The skill runs as a one-shot interactive analysis; users re-invoke it as needed.

**Falcon is read-only.** Every discovery, probe, and check in this skill uses read-only verbs (`describe-*`, `govc *.info`, `GET` requests, metric query APIs, `SELECT` queries). Mutations are never issued.

## Goals

1. Tell the user, with evidence, whether migrated workloads look and behave like their VMware sources.
2. Four comparison surfaces in v1: infrastructure shape, endpoint behavior, metrics/SLOs, database data parity.
3. **Best-effort automatic workload mapping** from VMware side to AWS side using multiple signals (name, IP, hostname, DNS, OS family, sizing plausibility); ask the user only for ambiguous or unresolved pairings.
4. **Zero-prompt metric source discovery.** The skill scans env vars for telemetry credentials, enumerates available metrics per source, and maps them to CloudWatch equivalents via a shipped table. No metric queries or names are solicited from the user.
5. Extensibility: future destinations (OpenShift, GCP) or additional surfaces drop into the flat `references/` layout without restructure.

## Non-goals (v1)

- **Driving a migration** (discovery, 6Rs decisions, template conversion, cutover runbooks, decommissioning). Fully out of scope — this is the explicit pivot. Users wanting migration execution use external tooling (AWS MGN, DMS, CloudEndure).
- **Continuous monitoring mode.** v1 is one-shot only. Users re-invoke the skill as needed. Continuous mode (cron-ready wrapper generation) is deferred to v1.1.
- **NoSQL data parity** (Mongo, Cassandra, DynamoDB). Different consistency models, different validation semantics — out of scope. v1 covers SQL DBs only (MySQL, PostgreSQL, SQL Server).
- **File / object-store parity** (NFS → EFS file hash comparisons, S3 inventory diffs). Out of scope — application-specific and easy to do incorrectly at scale.
- **Mutations of any kind.** No tag writes, no remediation commands executed.
- **Cost parity** — AWS Cost Explorer covers this.
- **Non-VMware sources** (Hyper-V, bare metal) and non-AWS destinations. Deferred.
- **Synthetic workload generation** — read-only by design means no synthetic traffic or load testing.
- **Relying on migration-tracking tags** (`migration-wave`, `migration-source-vm-id`, etc.) as the primary mapping mechanism. Real migrations don't reliably apply these. Tags are consumed as one signal among many, not required.

## Scope decisions

| # | Decision | Rationale |
|---|----------|-----------|
| 1 | Skill name: `migration-monitor` | Matches user's intent; precise to what the skill does. |
| 2 | Form factor: single skill with orchestrator `SKILL.md` + flat `references/` + `scripts/` | Documented FalconClaw pattern. |
| 3 | Delivery mode: **one-shot only** in v1 | Continuous wrapper deferred to v1.1. Skill is interactive; user re-runs as needed. |
| 4 | Parity surfaces v1: infrastructure, endpoint, metrics/SLO, DB data (SQL-only) | NoSQL, file-system, cost parity explicitly out. |
| 5 | Workload mapping: **multi-signal heuristic with confidence scoring**; ask user only for ambiguities | Real migrations don't reliably tag. Signals: exact/fuzzy name match, private/public IP match, DNS/hostname match, OS family, sizing plausibility. Auto-pair ≥80 confidence; present candidates 40–79; solicit <40. |
| 6 | Metrics discovery: fully automatic, zero user prompting | Scan env for telemetry creds, enumerate metrics via each source's catalog API, map to CloudWatch via shipped table. |
| 7 | Comparison semantics: "equivalent within tolerance," not exact equality | Right-sizing may deliberately change vCPU/RAM; tags get rewritten; security rules move to new SG IDs. Tolerance rules encoded in each parity reference. |
| 8 | Boundaries: standalone | Skill owns the monitor workflow end-to-end. |
| 9 | Three user-approval gates | No mutation boundaries; only need approval on operating mode, workload mapping, and divergence disposition. |
| 10 | Flat `references/` layout | Consistent with sibling skills; 6 references in v1. |

## Architecture

### Directory layout

```
skills/migration-monitor/
├── SKILL.md                          (~280 lines, hard cap 500)
├── references/
│   ├── probe-and-map.md              (~350 lines) — Phases 0–2 + mapping heuristics
│   ├── infra-parity.md               (~300 lines) — Phase 3
│   ├── endpoint-parity.md            (~250 lines) — Phase 4
│   ├── metrics-parity.md             (~300 lines) — Phase 5 + metric name mapping table
│   ├── data-parity.md                (~200 lines) — Phase 6 (DB)
│   └── divergence-report.md          (~200 lines) — Phase 7
└── scripts/
    ├── schema-vmware.json            — VMware state shape
    ├── schema-aws.json               — AWS state shape
    ├── schema-divergence.json        — Divergence report shape
    ├── probe-vcenter.sh              (~200 lines) — read-only govc sweep
    ├── parse-rvtools.py              (~150 lines) — RVTools fallback parser
    ├── probe-aws.py                  (~300 lines) — read-only aws describe-* sweep (enumerate full account/region, not filtered by tags)
    ├── map-workloads.py              (~250 lines) — NEW: consumes vmware + aws inventories, emits mapping with confidence scores
    ├── endpoint-diff.py              (~200 lines) — HTTP/TCP/TLS parity probe
    └── db-parity.py                  (~250 lines) — SQL DB parity (MySQL/PostgreSQL/SQL Server)
```

Line counts are ceilings, not targets.

### Frontmatter

```yaml
---
name: migration-monitor
description: "Post-cutover parity assurance for VMware → AWS migrations: captures live state on both sides via read-only probes, diffs infrastructure shape (compute/storage/network/tags/security), endpoint behavior (HTTP/TCP/TLS response parity), metrics (CPU/memory/latency/error-rate equivalence within tolerance), and SQL database data parity (table presence, row counts, column schema), then emits a consolidated divergence report. Workload pairings are auto-discovered via multi-signal heuristics (name, IP, DNS, hostname, OS, sizing) with user confirmation only for ambiguous matches; metric sources are auto-discovered from env vars. Falcon is read-only — all output is copy-ready. One-shot analysis; re-invoke as needed. Use after a VMware → AWS cutover to verify the target workloads behave like the originals."
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

<5-line preamble: scope, read-only, one-shot, no mutations ever>

## Workflow Overview
<ASCII flow of 7 phases with 3 gates>

## Phase 0: Capability Probe (REQUIRED FIRST STEP)
<probe commands; available-surfaces declaration>

## Phase 1: Workload Mapping (Multi-signal Heuristic)
<auto-match algorithm with confidence scoring; present pairs at Gate 2>

## Phase 2: Baseline Capture
<run probes to capture both sides' full state>

## Phases 3-7 (brief)
<one paragraph each, pointer to reference>

## Reference Guide
<table>

## Scripts
<pointers to schema + probe + map + endpoint-diff + db-parity>

## Output Conventions
<copy-ready bundle format, divergence report format>

## Constraints (MUST DO / MUST NOT DO)
```

### Phase 0 — Capability probe

Runs first every session. Read-only probes:

| Probe | Approach | Signals |
|-------|----------|---------|
| AWS reachable | `aws sts get-caller-identity` | account ID, active region |
| vCenter reachable | `govc about` (only if `GOVC_URL` + `GOVC_USERNAME` set) | live VMware discovery available |
| Static exports | scan cwd for `RVTools*.xlsx` / `RVTools*.csv` | RVTools fallback available |
| Metric sources | env-var scan for `PROMETHEUS_URL`, `DATADOG_API_KEY` + `DATADOG_APP_KEY`, `NEW_RELIC_API_KEY` + `NEW_RELIC_ACCOUNT`, `VROPS_URL` + `VROPS_USERNAME` | available metric sources |
| Endpoint probing | `curl --version` | ability to diff HTTP endpoints |
| TLS probing | `openssl version` | ability to validate TLS certs |
| DB parity | env-var scan for `SOURCE_DB_URI` + `SOURCE_DB_PASSWORD` and `TARGET_DB_URI` + `TARGET_DB_PASSWORD` | DB parity available |

Rather than declaring a fixed "mode" from an enumerated list, the skill enumerates **available surfaces** based on probe outcomes:

- **Required:** AWS + (live vCenter OR RVTools export) — without both sides we can't compare anything. If either is missing, stop and ask user to remediate.
- **Optional surfaces** (skipped if unavailable, logged with reason):
  - Endpoint parity (requires `curl`)
  - TLS checks (requires `openssl`)
  - Metrics parity (requires ≥1 metric source)
  - DB parity (requires both source and target DB URIs + passwords)

Skill prints the surface checklist at Gate 1, user confirms mode is acceptable (or adds missing env vars and re-runs Phase 0).

### Phase 1 — Workload mapping (multi-signal heuristic)

**The mapping problem:** given a VMware inventory (M workloads) and an AWS inventory (N resources), produce a `{vmware-workload → aws-resource}` pairing with best-effort confidence scoring. Do not rely on any single signal — especially not migration-tracking tags, which may not exist on real migrations.

**Signals** (each contributes to pair score 0–100):

| Signal | Weight | Extraction |
|--------|--------|------------|
| Exact name match | +40 | VM name equals AWS resource `Name` tag (case-insensitive) |
| Fuzzy name match | +20 | Token similarity ≥0.7 OR Levenshtein distance ≤3 between VM name and AWS Name tag |
| Private IP match | +30 | Any VMware VM IPv4 address equals any AWS ENI private IP (for EC2) or endpoint address (for RDS) |
| Public IP match | +30 | VMware VM public IP equals AWS EIP attached to target |
| Hostname match | +25 | VMware guest hostname (via VMware Tools) equals AWS Route 53 record target, or ALB/NLB DNS name |
| OS family match | +10 | Source Linux → target Linux, Windows → Windows; −20 on explicit mismatch |
| Sizing plausibility | +10 | Target vCPU within factor of 2 of source vCPU AND target memory within factor of 2 |
| Migration-tracking tag present | +15 | AWS resource has `migration-source-vm-id` tag matching VM MoRef, or `migration-source-name` matching VM name. **One signal among many — not required.** |
| Annotation/notes reference | +5 | VMware VM annotation contains the AWS resource name/ID, or vice versa |

**Classification thresholds:**

- **≥80** — high confidence: auto-pair, do not prompt.
- **40–79** — medium confidence: present top 3 candidates to user, user picks.
- **<40 or no candidates** — unresolved: present to user as "VMware workload `web-01` has no confident match; pick from AWS inventory or mark as not-migrated."

**Orphans:**
- VMware VMs with no candidate ≥40 → "unresolved source" list.
- AWS resources no VMware VM mapped to → "unresolved target" list (could be native AWS workloads, not migrations).

**User interaction at Gate 2:** present three lists:
1. Auto-paired (≥80), collapsed — user can expand to override.
2. Ambiguous (40–79) with top 3 candidates per VMware workload — user picks.
3. Unresolved — user either provides a pairing or declares the VM out-of-scope / not-yet-migrated.

The Gate 2 mapping is the contract for all downstream phases.

### Phase 2 — Baseline capture

For each side, capture current state:

- **VMware side:** `probe-vcenter.sh` (if live), else `parse-rvtools.py <path>` (RVTools export in cwd).
- **AWS side:** `probe-aws.py` (enumerate the full account+region, NOT filtered by tags — we don't trust tags as the mapping source).

Both emit JSON to stdout conforming to `schema-vmware.json` / `schema-aws.json`. User redirects to files; Falcon reads back.

### Phases 3–7

Each phase loads a specific reference file. Phases are skipped if the corresponding surface was unavailable at Phase 0.

- **Phase 3 — Infrastructure parity.** Load `references/infra-parity.md`. Compare compute shape, disk, network, tags, security rules, public/private exposure.
- **Phase 4 — Endpoint parity.** Load `references/endpoint-parity.md`. Run `endpoint-diff.py` against endpoint pairs derived from Phase 1 mapping (ALB DNS, NLB DNS, Route 53 records, EIPs).
- **Phase 5 — Metrics parity.** Load `references/metrics-parity.md`. Auto-discover metric sources (env-detected), enumerate metrics per resource via source catalog APIs, map source metric names to CloudWatch equivalents via the shipped table, compare statistical summaries (p50/p95/p99/mean over a 7-day comparison window, user-overridable).
- **Phase 6 — Data parity (SQL DBs).** Load `references/data-parity.md`. Run `db-parity.py` against paired DB instances. Compare table inventory, row counts per table, column schema.
- **Phase 7 — Divergence report.** Load `references/divergence-report.md`. Consolidate Phases 3–6 into a single classified report. Present at Gate 3.

### Decision gates (three)

| # | Gate | What Falcon presents | What user confirms | Why |
|---|------|---------------------|---------------------|-----|
| 1 | After capability probe (end of Phase 0) | Available-surfaces checklist | Mode is right; env is sufficient | Wrong mode = wrong parity surfaces. |
| 2 | After workload mapping (end of Phase 1) | Auto-paired + ambiguous + unresolved lists | Mapping correct + scope-in/out | Wrong mapping poisons all downstream comparisons. |
| 3 | After divergence report (end of Phase 7) | Classified divergences | Per-divergence disposition (accepted / investigate / blocker) | Drives user's next steps and next-run scope. |

No mutation gates. Three gates instead of six from the predecessor.

### Output conventions

- **Copy-ready bundles** for any remediation script the user might run: fenced code block with `# file: <destination-path>` header as first line.
- **Commands** numbered with **Expected output** blocks where verification matters.
- **Divergence report** is both a structured JSON (schema: `scripts/schema-divergence.json`) and a human summary table. Both rendered at Gate 3.

### Constraints

**MUST DO:**
- Phase 0 capability probe first every session.
- Present results and wait for user approval at each of the three gates before advancing.
- **Use multi-signal heuristic matching for Phase 1; never rely on tags as the sole signal.** Use the weighted signal table in `probe-and-map.md`.
- **Auto-discover metric sources from env; never ask the user to name metrics.** Common metric names are mapped mechanically in `metrics-parity.md`.
- Reconcile probe outputs against the v1 schemas before running comparisons; if either side's probe emits non-conforming JSON, halt with a schema-validation error.
- Web-search current AWS CloudWatch metric namespaces before finalizing any metric mapping — namespaces and metric names evolve.
- Classify every divergence by severity using tolerance rules in the relevant parity reference; never emit an unclassified divergence.

**MUST NOT DO:**
- Execute any mutation. Falcon is read-only, always.
- Generate synthetic traffic against either side.
- Rely on migration-tracking tags alone for workload mapping.
- Prompt the user to enumerate metrics, metric queries, or metric names.
- Guess at metric-name equivalence without consulting `metrics-parity.md`; if a metric has no mapping in that table, emit it as an "unmapped" divergence rather than fabricate a pairing.
- Run DB queries that aren't `SELECT` (including `SELECT INTO`, `UPDATE`, `DELETE`, DDL).
- Skip phases silently — if a surface is unavailable, explicitly state "Phase N skipped — reason X" in the report.

## Reference file contents

### `references/probe-and-map.md` (~350 lines)

The centerpiece for Phases 0–2. Contains:

- **Phase 0 recipes:** capability-probe commands per telemetry source, with expected output patterns.
- **Phase 1 multi-signal matching algorithm** — the weighted table above, plus implementation detail:
  - How each signal is computed.
  - Handling of multi-IP VMs (one VM has several NICs/IPs) and multi-ENI AWS instances.
  - Scoring tie-breakers.
  - Pseudo-code for the matching loop (input: vmware inventory, aws inventory; output: pairs + scores + unresolved).
- **Phase 2 baseline-capture recipes:** when to suggest `probe-vcenter.sh` vs `parse-rvtools.py`, how to invoke `probe-aws.py`, how to store outputs so later phases find them.
- **User-interaction recipe for Gate 2:** how to present auto-paired / ambiguous / unresolved lists; what overrides look like; the final "mapping JSON" structure passed downstream.

### `references/infra-parity.md` (~300 lines)

Per-attribute comparison rules and tolerance thresholds.

- **Compute shape:** vCPU ratio target ≥ 0.8× source, memoryMB ratio target ≥ 0.8× source telemetry p95. OS family must match. Architecture must match.
- **Disk:** total attached capacity within 10% of source (excluding OS disk growth on target). Disk count may differ (one large source disk → multiple AWS EBS volumes acceptable).
- **Network:** outbound security rules must permit at least the same destination/port set; inbound rules must permit at least the same source CIDRs for same ports; subnet count may differ; public-exposure parity preserved.
- **Tags:** tag parity is informational — if source had a tag `app=web`, target should too. **Divergences on migration-tracking tags (`migration-wave`, etc.) are minor/informational** since v1 doesn't require them.
- **Encryption:** if source used VMware disk encryption, target EBS volumes must have `encrypted=true`.

Divergence classification per rule (critical/major/minor).

### `references/endpoint-parity.md` (~250 lines)

Recipes for `endpoint-diff.py`. Three probe types:

- **HTTP:** GET both URLs (optional auth header via env), compare status, selected structural headers (`Content-Type`, `Cache-Control`, security headers), and a semantic body diff. Semantic diffing normalizes JSON (sorted keys, strip known volatile fields: `timestamp`, `requestId`, `traceId`, `date`), HTML (strip doctype/comments/attribute ordering). Divergences: status mismatch = critical; security-header mismatch = major; body structural mismatch = major/minor per rule.
- **TCP:** `nc -z -w5 <host> <port>` on both sides. Reachability must match.
- **TLS:** `openssl s_client -connect <host>:443 -servername <host> < /dev/null | openssl x509 -noout -dates -subject -issuer`. Compare cert validity (both must be not-expired), subject CN matches endpoint, issuer family acceptable.

Per-probe tolerance rules and classification.

### `references/metrics-parity.md` (~300 lines)

- **Metric-source detection** — env-var → source-type mapping.
- **Catalog enumeration** — per source, the API endpoint that lists metric names (`GET /api/v1/label/__name__/values` for Prometheus, etc.).
- **Metric-name mapping table** (authoritative). Representative rows:

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
- **Unmapped metric handling:** if a source metric has no entry in the mapping table, emit as "unmapped — manual review required" — never fabricate a pairing.

### `references/data-parity.md` (~200 lines)

SQL database parity for DBs that migrated to RDS or Aurora.

- **Scope:** table presence, row counts, column schema.
- **Out of scope:** data content checksums (too brittle on large tables), NoSQL, file-system parity.
- **Checks:**
  - **Table inventory parity:** both sides have same set of user-defined tables (excludes `information_schema`, `pg_catalog`, `sys` / `mysql` system tables). Missing on target = critical; extra on target = minor.
  - **Row count parity per shared table:** `SELECT COUNT(*)` on both sides. Tolerance: within 1% (accounts for in-flight data during probe). Strict mode (0%) available via flag.
  - **Column schema parity:** column count + column names (not types, because platform-specific type name drift is expected — MySQL `INT` vs PostgreSQL `INTEGER`). Missing column on target = critical; extra column = minor.
- **Cutoff-timestamp option:** if CDC replication is still running during the check, user can specify a `--cutoff-field` and `--cutoff-value` so counts only include rows older than the cutoff.
- **Engines supported v1:** MySQL 5.7+, PostgreSQL 12+, SQL Server 2017+. Oracle deferred (requires proprietary client).
- **Script delivery:** `scripts/db-parity.py` takes source and target URIs, emits structured divergence JSON.

### `references/divergence-report.md` (~200 lines)

- **Report JSON schema** — committed as `scripts/schema-divergence.json`. Structure: `{phases: {infra: [...], endpoint: [...], metrics: [...], data: [...]}, summary: {critical: N, major: N, minor: N, skipped_phases: [...]}, generated_at: ISO8601}`.
- **Human summary table format** — sample rendering with critical-first ordering.
- **Severity classification rubric:** critical (functional break suspected), major (observable regression, not functional), minor (cosmetic or within tolerance band).
- **Remediation hints per divergence type** — table of common divergences with suggested investigation steps:
  - `compute.undersized` → suggest instance-type change; web-search current sizing recommendations
  - `security.inbound.missing-port` → provide `aws ec2 authorize-security-group-ingress` command for user to run
  - `endpoint.status-mismatch` → suggest checking ALB target health + app-side config
  - `metrics.cpu.regression` → suggest scaling up or investigating noisy-neighbor
  - `data.row-count.deficit` → suggest checking DMS CDC lag or replication errors
  - (full table)

## Scripts

### `scripts/schema-vmware.json`

Top-level: `schemaVersion`, `source` (enum `[govc, rvtools]`), `capturedAt`, `vcenter`, `clusters`, `hosts`, `vms`, `networks`, `datastores`.

### `scripts/schema-aws.json`

Top-level: `schemaVersion`, `source` (const `aws`), `capturedAt`, `account` (id + region), `resources`. `resources` is an array with discriminated sub-schemas per resource type: `ec2_instance`, `rds_instance`, `rds_cluster`, `efs_filesystem`, `fsx_filesystem`, `elb` (classic), `alb`, `nlb`, `security_group`, `subnet`, `vpc`, `elasticache_cluster`, `msk_cluster`, `route53_record`. Every resource carries its full tag map (for Phase 3 tag parity) and its IPs/DNS names (for Phase 1 matching).

### `scripts/schema-divergence.json`

Top-level: `reportVersion`, `generatedAt`, `phases` (object with `infra`, `endpoint`, `metrics`, `data` arrays — any may be absent if the phase was skipped), `summary`, `skippedPhases`. Each divergence: `{id, severity, type, phase, resourcePair, description, remediationHint}`.

### `scripts/probe-vcenter.sh`

Read-only govc sweep. Requires `GOVC_URL`, `GOVC_USERNAME`, `GOVC_PASSWORD`. Commands: `govc ls`, `govc vm.info -json`, `govc host.info`, `govc datastore.info`, `govc network.info`. Captures guest IP addresses and hostnames (for Phase 1 matching). Emits `schema-vmware.json`-conforming JSON. Supports `--dry-run`.

### `scripts/parse-rvtools.py`

RVTools `.xlsx` / `.csv` → `schema-vmware.json` normalization. Handles `vInfo` / `vCPU` / `vMemory` / `vDisk` / `vNetwork` / `vHost` / `vDatastore` / `vRPool` tabs in xlsx. For CSV, expects `vInfo` header. Requires `openpyxl` for xlsx.

### `scripts/probe-aws.py`

Read-only `aws describe-*` sweep. Enumerates the full account+region without tag filters. Calls: `aws ec2 describe-instances`, `aws ec2 describe-security-groups`, `aws ec2 describe-subnets`, `aws ec2 describe-vpcs`, `aws rds describe-db-instances`, `aws rds describe-db-clusters`, `aws elbv2 describe-load-balancers` + `describe-target-groups` + `describe-target-health`, `aws elb describe-load-balancers` (classic), `aws efs describe-file-systems`, `aws fsx describe-file-systems`, `aws elasticache describe-cache-clusters`, `aws kafka list-clusters-v2`, `aws route53 list-hosted-zones` + `list-resource-record-sets`. Emits `schema-aws.json`-conforming JSON. Supports `--dry-run`.

### `scripts/map-workloads.py`

NEW in v1. Takes `--vmware <path>` and `--aws <path>` (both JSON files from the probes). Applies the weighted-signal matching algorithm and emits:

```json
{
  "pairs": [ { "vmware_id": "vm-1001", "aws_id": "i-abc", "confidence": 92, "signals": ["exact-name", "private-ip", "os-family"] } ],
  "ambiguous": [ { "vmware_id": "vm-1002", "candidates": [{"aws_id": "i-def", "confidence": 65, "signals": [...]}, ...] } ],
  "unresolved_vmware": ["vm-1003"],
  "unresolved_aws": ["i-ghi"]
}
```

Supports `--dry-run`. Falcon reads this output, presents at Gate 2, user overrides interactively, final mapping is the contract for Phases 3–7.

### `scripts/endpoint-diff.py`

HTTP / TCP / TLS parity probe. Takes `--pairs <path>` (JSON list of `{source_url, target_url, probe_type}` from Phase 1 enrichment). For each pair, runs the applicable probes. Emits divergence JSON conforming to `schema-divergence.json` (endpoint subset). Supports `--dry-run`.

### `scripts/db-parity.py`

SQL DB parity. Takes `--source <uri>` and `--target <uri>`:
- `mysql://user@host:3306/db`
- `postgresql://user@host:5432/db`
- `mssql://user@host:1433/db`

Password via env vars: `SOURCE_DB_PASSWORD`, `TARGET_DB_PASSWORD`. Runs only `SELECT` queries — row counts, information_schema/pg_catalog/sys queries. Supports `--cutoff-field` and `--cutoff-value` for CDC-in-flight handling. Supports `--exclude-table <regex>` for skipping volatile tables. Emits divergence JSON conforming to `schema-divergence.json` (data subset). Supports `--dry-run`.

Dependencies (optional per engine): `pymysql`, `psycopg2-binary`, `pymssql`. Script handles `ImportError` gracefully — emits `data.engine-unavailable` divergence and continues to next engine.

## Common script behaviors

All scripts:
- Print JSON to stdout on success.
- Non-zero exit + structured JSON error to stderr on failure (schema: `{error, hint, exitCode}`).
- Use only read-only verbs.
- Ship a `--dry-run` (or `--self-test`) flag emitting a canned fixture for CI.

## Success criteria

**Automated (verifiable in CI / local test):**
- `.github/workflows/validate.yml` passes for the skill.
- Script test suite: each script's `--dry-run` output validates against its declared schema.
- `map-workloads.py` against canned source+target fixtures produces the expected pairing with expected confidences.
- Example code in each `references/*.md` is syntactically valid (HCL via `terraform fmt`, JSON via `python -m json.tool`).

**Manual (verified by walking the skill in Falcon):**
- Capability probe correctly classifies available surfaces given synthetic env vars.
- Given canned inventories, the skill walks Phases 1–7 to produce a divergence report, stopping at each of the three gates.
- Phase 1 mapping auto-pairs high-confidence matches without prompting, presents top-3 candidates for medium-confidence matches, and lists unresolved items clearly.
- Metrics auto-discovery produces no user prompts.

## Extension plan

Adding a new metric source (e.g., Splunk): entry in metric-source detection + section in `metrics-parity.md` + rows in mapping table. No script changes.

Adding a new parity surface (e.g., cost parity in v1.1): new reference `cost-parity.md`, new script `probe-cost.py`, Phase inserted between metrics and report.

Adding a new destination (e.g., OpenShift): `probe-openshift.py`, `schema-openshift.json`, OpenShift-equivalent sections in each parity reference.

**v1.1 roadmap:**
- Continuous-mode wrapper generation (dropped from v1).
- NoSQL data parity (Mongo, DynamoDB).
- Oracle support in `db-parity.py`.
- File-system parity (EFS vs source NFS) via hash sampling.

## Open items / deferred

- **Metric comparison window default.** Starting at 7 days; per-workload overrides deferred.
- **TLS cert chain-trust comparison.** v1 compares validity + subject CN only.
- **Endpoint body diff's volatile-field strip list.** v1 ships default list (`timestamp`, `requestId`, `traceId`, `date`); user-provided regex extensibility deferred.
- **DB engines.** v1 ships MySQL, PostgreSQL, SQL Server. Oracle + NoSQL deferred.
- **Script dependency management.** `parse-rvtools.py` needs `openpyxl`; `db-parity.py` needs engine-specific drivers as optional imports. Install hints documented in each script header.
- **Performance for large inventories.** v1 probes full account/region. For accounts with 10k+ resources, this could be slow. Pagination is respected but not parallelized. Parallelization deferred.
