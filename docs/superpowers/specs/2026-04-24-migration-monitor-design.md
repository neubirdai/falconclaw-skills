# Migration Monitor Skill — Design

**Status:** Approved, pending review
**Date:** 2026-04-24
**Skill name:** `migration-monitor`
**Repo:** `neubirdai/falconclaw-skills`
**Supersedes:** `docs/superpowers/specs/2026-04-23-migration-assistant-design.md` (scope pivoted from drive-the-migration to assure-the-migration)

## Overview

The `migration-monitor` skill provides post-cutover parity assurance for VMware → AWS migrations inside the Falcon TUI. Given a set of workloads that have been migrated, it captures baseline state on both sides, diffs infrastructure shape, endpoint behavior, and metrics, and emits a consolidated divergence report. It also generates a cron-ready wrapper script for continuous parity monitoring.

**Falcon is read-only.** Every discovery, probe, and check in this skill uses read-only verbs (`describe-*`, `govc *.info`, `GET` requests, metric query APIs). Mutations are never issued — continuous mode emits a standalone script the user installs into their own scheduler.

## Goals

1. Tell the user, with evidence, whether migrated workloads look and behave like their VMware sources.
2. Three comparison surfaces covered in v1: infrastructure shape, endpoint behavior, metrics/SLOs.
3. Auto-discovery wherever possible — metric sources, endpoint pairings, tag-inferred workload mappings — so the user isn't burdened with telemetry trivia.
4. Extensibility: future destinations (OpenShift, GCP) or additional surfaces (data parity, cost parity) drop into the flat `references/` layout without restructure.
5. Continuous monitoring mode is a deliverable the user can install and run without Falcon in the loop.

## Non-goals (v1)

- **Driving a migration** (discovery, 6Rs decisions, template conversion, cutover runbooks, decommissioning). Fully out of scope — this is the explicit pivot. Users wanting migration execution use external tooling (AWS MGN, DMS, CloudEndure).
- **Data parity** — DB row counts, schema diffs, file hash counts. Deliberately excluded: data parity is domain-specific (SQL vs NoSQL vs files vs queues) and usually belongs to application owners with their own tooling.
- **Mutations of any kind.** No tag writes, no remediation commands executed, no alert configuration. Continuous-mode scripts are handed to the user as code blocks — Falcon never installs them.
- **Cost parity.** Pre/post AWS bill comparisons are tracked by AWS Cost Explorer; not this skill's job.
- **Non-VMware sources** (Hyper-V, bare metal) and non-AWS destinations. Deferred.
- **Synthetic workload generation** (load-test the new side to confirm it handles throughput). Read-only by design means no synthetic traffic.

## Scope decisions

| # | Decision | Rationale |
|---|----------|-----------|
| 1 | Skill name: `migration-monitor` | Matches user's intent language; precise to what the skill does. |
| 2 | Form factor: single skill with orchestrator `SKILL.md` + flat `references/` + `scripts/` | Same documented FalconClaw pattern; no multi-skill split. |
| 3 | Delivery modes: one-shot (primary) + continuous wrapper (secondary) | User invokes interactively for one-shot; skill emits a script for continuous. Falcon is interactive-only — it doesn't schedule. |
| 4 | Parity surfaces v1: infrastructure, endpoint, metrics/SLO | Data parity excluded as domain-specific; cost parity excluded as tool-covered elsewhere. |
| 5 | Endpoint pairing: auto-discovery via tags + user confirmation | Auto-discover first (from `migration-source-dns` / `migration-source-vm-id` tags on AWS resources); user confirms and fills gaps. |
| 6 | Metrics discovery: fully automatic, zero user prompting | Scan env for telemetry credentials, enumerate available metrics via each source's catalog API, map to CloudWatch equivalents via a shipped table. User does not specify metrics. |
| 7 | Comparison semantics: "equivalent within tolerance," not exact equality | Right-sizing may deliberately change vCPU/RAM; tags get rewritten; security rules move to new SG IDs. Tolerance rules are encoded in each parity reference. |
| 8 | Boundaries: standalone | Skill owns the monitor workflow end-to-end. Description narrows domain to VMware → AWS post-migration parity. |
| 9 | Three user-approval gates (was six in the predecessor skill) | Fewer because there are no mutation boundaries; only need approval on operating mode, workload mapping, and divergence disposition. |
| 10 | Flat `references/` layout (Approach B from predecessor) | Same rationale; 6 references in v1, easily extended. |

## Architecture

### Directory layout

```
skills/migration-monitor/
├── SKILL.md                          (~250 lines, hard cap 500)
├── references/
│   ├── probe-and-map.md              (~250 lines) — Phases 0–2
│   ├── infra-parity.md               (~300 lines) — Phase 3
│   ├── endpoint-parity.md            (~250 lines) — Phase 4
│   ├── metrics-parity.md             (~300 lines) — Phase 5 + metric name mapping table
│   ├── divergence-report.md          (~200 lines) — Phase 6
│   └── continuous-mode.md            (~200 lines) — scheduled wrapper generation
└── scripts/
    ├── schema-vmware.json            — VMware state shape
    ├── schema-aws.json               — AWS state shape
    ├── probe-vcenter.sh              (~200 lines) — read-only govc sweep
    ├── parse-rvtools.py              (~150 lines) — RVTools fallback parser
    ├── probe-aws.py                  (~250 lines) — read-only aws describe-* sweep
    └── endpoint-diff.py              (~200 lines) — HTTP/TCP/TLS parity probe
```

Line counts are ceilings, not targets.

### Frontmatter

```yaml
---
name: migration-monitor
description: "Post-cutover parity assurance for VMware → AWS migrations: captures live state on both sides via read-only probes, diffs infrastructure shape (compute/storage/network/tags/security), endpoint behavior (HTTP/TCP/TLS response parity), and metrics (CPU/memory/latency/error-rate equivalence within tolerance), then emits a divergence report and an optional cron-ready continuous-monitoring script. Metric sources are auto-discovered; endpoint pairings are auto-inferred from AWS resource tags and user-confirmed. Falcon is read-only — all output is copy-ready. Invoke after a VMware → AWS cutover to verify the target workloads behave like the originals."
tags: [migration, monitor, parity, vmware, aws, cloud]
homepage: https://github.com/neubirdai/falconclaw-skills/tree/main/skills/migration-monitor
metadata: {"neubird":{"emoji":"🔎","requires":{"anyBins":["aws","govc"]}}}
---
```

**Design choices:**

- Description explicitly claims the "post-cutover" niche and names the three parity surfaces — keeps Falcon from invoking it on pre-migration questions.
- `🔎` emoji distinct from predecessor's 🚚 (and from sibling skills).
- `requires.anyBins: ["aws", "govc"]` — at least one side must have CLI tooling. RVTools fallback covers the no-govc case (`parse-rvtools.py` uses python).

## SKILL.md orchestrator

### Structure

```markdown
---
<frontmatter>
---

# Migration Monitor (VMware → AWS)

<5-line preamble: scope, read-only, all commands user-executed, no mutations ever>

## Workflow Overview
<ASCII flow of 7 phases with 3 gates>

## Phase 0: Capability Probe (REQUIRED FIRST STEP)
<probe commands: aws / govc / RVTools / metric sources / curl>
<mode declaration matrix>

## Phase 1: Workload Mapping
<auto-discovery via tags, user confirm + fill gaps>

## Phase 2: Baseline Capture
<run probe-vcenter.sh OR parse-rvtools.py; run probe-aws.py>

## Phases 3–6 (brief)
<one paragraph each, pointer to reference>

## Phase 7: Continuous Mode (optional)
<generate cron-ready wrapper, present as copy-ready bundle>

## Reference Guide
<table>

## Scripts
<pointers to schema files + probe scripts + endpoint-diff>

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

Declared modes:

| Mode | Triggered when | Behavior |
|------|----------------|----------|
| `full` | live vCenter + AWS + ≥1 metric source + curl | All three parity surfaces active |
| `infra-endpoint` | live or RVTools + AWS + curl, no metric source | Skip Phase 5 (metrics); infra + endpoint only |
| `infra-only` | live or RVTools + AWS, no curl | Skip Phases 4–5; infra parity only |
| `advisory` | AWS unreachable OR VMware side unavailable | Explain what's missing, ask user to remediate env, stop |

### Phase 1 — Workload mapping

Auto-discovery: enumerate AWS resources filtered by expected migration tags (per-resource query). For each tagged AWS resource, extract `migration-source-vm-id` (links to a vCenter MoRef) or `migration-source-dns` (links to a VMware-side hostname). Build the `{vmware-workload → aws-resource}` pair set.

Present discovered pairs + list of VMware-side workloads that have no corresponding AWS target + list of AWS resources with migration tags but no resolvable source. User confirms, adds pairs for untagged workloads, or drops workloads from scope.

**Gate 2 fires here** — confirm the mapping before running probes. Wrong mapping → all downstream parity checks are meaningless.

### Phase 2 — Baseline capture

For each side, run the appropriate probe:
- **VMware side:** `probe-vcenter.sh` (if live), else `parse-rvtools.py <path>` (if RVTools export in cwd).
- **AWS side:** `probe-aws.py --resources <json>` consuming the mapping from Phase 1.

Both emit JSON to stdout conforming to `schema-vmware.json` / `schema-aws.json`. User redirects to file; Falcon reads back for Phases 3–5.

### Phases 3–6

Each loads a specific reference:
- **Phase 3 — Infrastructure parity.** Load `references/infra-parity.md`. Compare compute shape, disk, network, tags, security rules, public/private exposure. Produce an infra divergence list.
- **Phase 4 — Endpoint parity.** Load `references/endpoint-parity.md`. Run `endpoint-diff.py` against the endpoint-pair list derived from Phase 1 tags. Produce an endpoint divergence list.
- **Phase 5 — Metrics parity.** Load `references/metrics-parity.md`. Auto-discover metric sources (env-var-detected), enumerate metrics per resource via the source's catalog API, map source metric names to CloudWatch equivalents via the shipped table, compare statistical summaries (p50/p95/p99/mean over a user-specified comparison window, default 7 days). Produce a metrics divergence list.
- **Phase 6 — Divergence report.** Load `references/divergence-report.md`. Consolidate outputs of Phases 3–5 into a single report. For each divergence, classify by severity (critical/major/minor) and tolerance-rule fired. Present remediation hints per divergence type.

### Phase 7 — Continuous mode (optional)

Load `references/continuous-mode.md`. Generate a self-contained bash or python wrapper that:
- Re-runs Phases 2–5 against the same mapping (stored as a JSON file alongside the script)
- Emits divergence report to stdout
- Supports a `--fail-on-severity` flag so cron/CI can alert on non-zero exit
- Optionally sends the report to a webhook (Slack, generic HTTP) if user provides `--webhook URL`

Present as a copy-ready bundle with install instructions (cron snippet, systemd timer template, GitHub Actions workflow template — user picks).

### Decision gates (three)

| # | Gate | What Falcon presents | What user confirms | Why |
|---|------|---------------------|---------------------|-----|
| 1 | After capability probe | Probe summary + declared mode | Mode is right; env is sufficient | Wrong mode = wrong parity surfaces |
| 2 | After workload mapping | Auto-discovered pairs + unpaired workloads + unresolved AWS resources | Mapping correct + scope-in/out | Wrong mapping poisons all downstream comparisons |
| 3 | After divergence report | Classified divergences | Per-divergence disposition (accepted / investigate / blocker) | Determines what gets flagged in continuous mode + drives user's next steps |

No mutation gates — there are no mutations.

### Output conventions

- **Copy-ready bundles** for any generated script (continuous-mode wrapper) and any IaC suggestion for remediation: fenced code block with `# file: <destination-path>` header.
- **Commands** numbered inline with expected output blocks where verification matters.
- **Divergence report format:** structured JSON for programmatic consumption + human summary table. Shape defined in `divergence-report.md`.

### Constraints

**MUST DO:**
- Phase 0 capability probe first every session.
- Present results and wait for user approval at each of the three gates before advancing.
- **Auto-discover metric sources from env; never ask the user to name metrics.** Common standard metric names are in `metrics-parity.md`; the skill maps source-side → CloudWatch-side mechanically.
- Reconcile probe outputs against the v1 schemas before running comparisons; if either side's probe emits non-conforming JSON, halt with a schema-validation error.
- Web-search current AWS CloudWatch metric namespaces before finalizing any metric mapping — namespaces/metric names change.
- Classify every divergence by severity using the tolerance rules in the relevant parity reference; never emit an unclassified divergence.

**MUST NOT DO:**
- Execute any mutation. Falcon is read-only, always.
- Generate synthetic traffic against either side.
- Prompt the user to enumerate metrics, endpoints, or metric queries — all three are auto-discovered.
- Emit alerts directly (continuous mode generates a script; the user installs it).
- Guess at metric-name equivalence without consulting `metrics-parity.md`; if a metric has no mapping in that table, emit it as an "unmapped" divergence rather than fabricate a pairing.
- Ship a continuous-mode script that contains hardcoded credentials — always use env-var placeholders.

## Reference file contents

### `references/probe-and-map.md` (~250 lines)

- Phase 0 capability probe recipes per telemetry source.
- **Tag-based auto-discovery for Phase 1 mapping:** the AWS `describe-*` queries filter on `tag:migration-source-vm-id` and `tag:migration-source-dns`. Examples given for EC2, RDS, EFS, ALB, NLB, FSx, ElastiCache, MSK.
- Mapping data structure (JSON example the skill builds and uses across later phases).
- User-interaction recipe for Gate 2: how to present auto-discovered pairs, handle overrides, flag orphans.
- Phase 2 baseline-capture recipes: when to suggest `probe-vcenter.sh` vs `parse-rvtools.py`, how to invoke `probe-aws.py` with the mapping JSON.

### `references/infra-parity.md` (~300 lines)

Per-attribute comparison rules and tolerance thresholds. Tables cover:

- **Compute shape:** vCPU ratio target ≥ 0.8× source (right-sizing floor), memoryMB ratio target ≥ 0.8× source telemetry p95. OS family must match (Linux→Linux, Windows→Windows). Architecture must match (x86_64 vs arm64).
- **Disk:** total attached capacity within 10% of source (excluding OS disk growth on target). Disk count may differ (one large source disk → multiple AWS EBS volumes is acceptable).
- **Network:** outbound security rules must permit at least the same destination/port set as the source. Inbound rules must permit at least the same source CIDRs for the same ports. Subnet count may differ. Public-exposure parity: if source was publicly reachable, target must be (via EIP/ALB/NLB); if private, target must be in private subnet.
- **Tags:** all tags from `required-tags` set must exist on target (configurable per workload); specifically the migration-tracking tags (`migration-wave`, `migration-source-vm-id`, `migration-date`).
- **Encryption:** if source used VMware disk encryption, target EBS volumes must have `encrypted=true`.

Divergence classification per rule (critical/major/minor).

### `references/endpoint-parity.md` (~250 lines)

Recipes for `endpoint-diff.py`. Three probe types:

- **HTTP:** GET both URLs (with optional auth header passed via env), compare status, selected structural headers (`Content-Type`, `Cache-Control`, security headers), and a semantic body diff. Semantic diffing: JSON responses normalized (keys sorted, known volatile fields like `timestamp`/`requestId`/`traceId` stripped), HTML responses stripped of doctype/comments/attribute ordering. Divergence: status mismatch = critical; header mismatch on security-relevant keys = major; body structural mismatch = major/minor per rule.
- **TCP:** `nc -z -w5 <host> <port>` on both sides. Reachability must match.
- **TLS:** `openssl s_client -connect <host>:443 -servername <host> < /dev/null | openssl x509 -noout -dates -subject -issuer`. Compare cert validity (both must be not-expired), subject CN match the endpoint, issuer family (LetsEncrypt, ACM, corporate CA) is acceptable as long as both verify.

Tolerance rules for each probe type; what counts as "same enough."

### `references/metrics-parity.md` (~300 lines)

The centerpiece. Contains:

- **Metric-source detection:** env-var → source-type mapping (`PROMETHEUS_URL` → Prometheus, `DATADOG_API_KEY` → Datadog, `NEW_RELIC_API_KEY` → New Relic, `VROPS_URL` → vROps).
- **Catalog enumeration:** per source, the API endpoint that lists metric names (`GET /api/v1/label/__name__/values` for Prometheus, etc.).
- **Metric-name mapping table** (the authoritative reference). Examples:

  | Source-side metric | CloudWatch equivalent | CloudWatch namespace |
  |--------------------|----------------------|----------------------|
  | vROps `cpu\|usage_average` | `CPUUtilization` | `AWS/EC2` |
  | vROps `mem\|usage_average` | `mem_used_percent` | `CWAgent` |
  | vROps `disk\|usage_average` | `disk_used_percent` | `CWAgent` |
  | vROps `net\|throughput_usage_average` | `NetworkIn` + `NetworkOut` (summed) | `AWS/EC2` |
  | Prom `node_cpu_seconds_total{mode!="idle"}` (rate) | `CPUUtilization` | `AWS/EC2` |
  | Prom `node_memory_MemAvailable_bytes` | `mem_used_percent` (inverted) | `CWAgent` |
  | Prom `http_request_duration_seconds{quantile="0.95"}` | ALB `TargetResponseTime` (p95) | `AWS/ApplicationELB` |
  | Prom `http_requests_total{status=~"5.."}` (rate) | ALB `HTTPCode_Target_5XX_Count` | `AWS/ApplicationELB` |
  | Datadog `system.cpu.user` + `system.cpu.system` | `CPUUtilization` | `AWS/EC2` |
  | New Relic `system.cpuPercent` | `CPUUtilization` | `AWS/EC2` |
  | RDS-side: Prom `pg_stat_database_tup_fetched` (rate) | RDS `ReadIOPS` | `AWS/RDS` |

  (Full table in the reference; above is representative.)

- **Comparison window:** default 7 days pre-cutover baseline vs 7 days post-cutover. User can override at session start.
- **Statistical comparison:** for each mapped metric, compute p50, p95, p99, mean over the window on both sides. Divergence classifications: within ±10% = OK, 10–30% = minor, 30–50% = major, >50% = critical.
- **Unmapped metric handling:** if a source-side metric has no entry in the mapping table, emit as "unmapped — manual review required" — never fabricate a pairing.

### `references/divergence-report.md` (~200 lines)

- **Report JSON schema** (committed under `scripts/schema-divergence.json`).
- **Human summary table format** — sample rendering.
- **Severity classification rubric:** critical (functional break suspected), major (observable regression, not functional), minor (cosmetic or within tolerance band).
- **Remediation hints per divergence type:**
  - `compute.undersized` → suggest instance-type change + web-search current sizing recommendations
  - `tags.missing:migration-wave` → provide `aws ec2 create-tags` command
  - `endpoint.status-mismatch` → suggest checking ALB target health + app-side config
  - `metrics.cpu.regression` → suggest scaling up or investigating noisy-neighbor
  - (full table)

### `references/continuous-mode.md` (~200 lines)

Wrapper-script generation logic. The script:

- Reads a sidecar JSON (mapping + config) the skill produces.
- Re-runs Phases 2–5 autonomously (no prompts).
- Emits divergence report to stdout and, if `--webhook` is set, POSTs to webhook.
- Exit code: 0 = all within tolerance, 1 = any `minor`, 2 = any `major`, 3 = any `critical`. Use with `--fail-on-severity` to make exit code meaningful for cron/CI.
- Install recipes (copy-ready bundles):
  - `crontab -e` entry template
  - `systemd` timer unit + service unit templates
  - GitHub Actions workflow yaml template
  - GitLab CI schedule snippet

Security notes: secrets come from env-vars the user references in their scheduler, not baked into the script.

## Scripts

### `scripts/schema-vmware.json`

Mirrors the shape established in the predecessor skill (but without the migration-planning fields since those aren't consumed here). Top-level: `schemaVersion`, `source` (enum `[govc, rvtools]`), `capturedAt`, `vcenter`, `clusters`, `hosts`, `vms`, `networks`, `datastores`.

### `scripts/schema-aws.json`

Top-level: `schemaVersion`, `source` (const `aws`), `capturedAt`, `account` (id + region), `resources`. `resources` is an array with discriminated sub-schemas per resource type (`ec2_instance`, `rds_instance`, `rds_cluster`, `efs_filesystem`, `fsx_filesystem`, `elb`, `alb`, `nlb`, `security_group`, `subnet`, `vpc`, `elasticache_cluster`, `msk_cluster`). Each resource carries the migration-tracking tags as first-class fields for mapping.

### `scripts/probe-vcenter.sh`

Read-only govc sweep. Same design as predecessor: `govc ls`, `govc *.info -json`. Emits `schema-vmware.json`-conforming JSON. Requires `GOVC_URL`, `GOVC_USERNAME`, `GOVC_PASSWORD`. Supports `--dry-run` for tests.

### `scripts/parse-rvtools.py`

RVTools `.xlsx` / `.csv` → `schema-vmware.json` normalization. Same design as predecessor. Requires `openpyxl` for xlsx.

### `scripts/probe-aws.py`

Read-only `aws describe-*` sweep. Accepts a mapping JSON (from Phase 1) as `--mapping path/to/map.json`. For each AWS resource in the mapping, runs the appropriate `describe-*` call. Supports `--dry-run` (emits canned fixture for tests). Emits `schema-aws.json`-conforming JSON.

### `scripts/endpoint-diff.py`

HTTP/TCP/TLS parity probe. Accepts a pair list as `--pairs path/to/pairs.json` (or stdin). For each pair, runs the three probe types where applicable. Emits structured divergence JSON conforming to `schema-divergence.json` (subset scope — endpoint divergences only). Supports `--dry-run`.

## Common script behaviors

All scripts:
- Print JSON to stdout on success.
- Non-zero exit + structured JSON error to stderr on failure (schema: `{error: string, hint: string, exitCode: int}`).
- Use only read-only verbs.
- Ship a `--self-test` or `--dry-run` flag that emits a canned fixture for CI.

## Success criteria

**Automated (verifiable in CI / local test):**
- `.github/workflows/validate.yml` passes for the skill.
- `scripts/*-test.py` suite passes: each script's `--dry-run` output validates against its declared schema.
- Example code in each `references/*.md` (any HCL in `continuous-mode.md`, any JSON in `*-parity.md`) is syntactically valid.

**Manual (verified by walking the skill in Falcon):**
- Capability probe correctly classifies the declared mode given synthetic env vars.
- Given a canned mapping, the skill walks Phases 2–6 to produce a divergence report, stopping at each of the three gates.
- Continuous-mode wrapper script is copy-pasteable and exits with meaningful codes against canned fixtures.
- Metrics auto-discovery produces no user prompts in `full` mode.

## Extension plan

Adding a new metric source (e.g., Splunk): add an entry to the metric-source detection logic + a section to `metrics-parity.md` + rows to the mapping table. No script changes.

Adding a new parity surface (e.g., cost parity): new reference file `cost-parity.md`, new script `probe-cost.py`, Phase 5.5 inserted. Architectural fit is clean since references are flat.

Adding a new destination (e.g., OpenShift): `probe-openshift.py`, `schema-openshift.json`, OpenShift-equivalent sections in each parity reference. No structural change needed.

Promotion to nested `references/` (predecessor's Approach C) triggers at 3+ destinations — mechanical file move, not a rewrite.

## Open items / deferred

- **Metric comparison window default.** Starting at 7 days; may need per-workload overrides if some workloads have weekly seasonality that a 7-day window doesn't capture. Handle in a v1.x if it becomes painful.
- **TLS cert identity comparison.** v1 compares validity + subject CN. Checking cert chain trust equivalence is harder (different CA trust stores) and deferred.
- **Endpoint body diff's volatile-field strip list.** v1 ships a small default list (timestamp, requestId, traceId); extensibility via a user-provided regex list is deferred to v1.1.
- **Continuous mode's webhook authentication.** v1 supports bearer-token via env var; signed-request variants (AWS SigV4, GCP OIDC) deferred.
- **Script dependency management.** `parse-rvtools.py` requires `openpyxl`; `probe-aws.py` uses boto3 only if direct awscli calls aren't enough. Install hints in script headers.
