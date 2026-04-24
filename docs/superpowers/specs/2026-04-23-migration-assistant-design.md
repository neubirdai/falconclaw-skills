# Migration Assistant Skill — Design

**Status:** Approved, pending review
**Date:** 2026-04-23
**Skill name:** `migration-assistant`
**Repo:** `neubirdai/falconclaw-skills`
**Audience:** Skill contributors and reviewers

## Overview

The `migration-assistant` skill executes end-to-end VMware → AWS migrations inside the Falcon (NeuBird SkillsHub) interactive terminal. It discovers the source environment (live vCenter, GitHub IaC, telemetry), reconciles drift, recommends 6Rs decisions per workload, converts VMware deployment templates (VCF Automation, Aria blueprints, Terraform vSphere, OVF) into AWS-native IaC, drives wave-based cutover runbooks, validates post-cutover health, and establishes baseline monitoring.

Falcon is a **read-only** system. The skill recommends commands and copy-ready IaC bundles; the user executes them. All of the skill's "status checks" and "validations" use read-only verbs (`describe-*`, `vm.info`, etc.).

## Goals

1. Give Falcon a precise, disciplined workflow for VMware → AWS migrations — not a generic migration primer.
2. Cover the full journey standalone: discovery → strategy → conversion → execution → validation → monitoring.
3. Minimize chat-turn overhead for common work via shipped read-only helper scripts.
4. Be extensible: adding VMware → OpenShift, VMware → GCP, or other source/destination pairs later is additive, not a restructure.
5. Enforce safety at every mutation point through explicit per-wave user approval gates.

## Non-goals (v1)

- **Refactor-to-containers** (containerizing VMware workloads onto ECS/EKS). Out of scope. Users wanting this should use `kubernetes-specialist` directly; this skill does not hand off.
- **Non-VMware sources** (Hyper-V, bare metal, other hypervisors).
- **Non-AWS destinations** (Azure, GCP, OpenShift) — deferred to future versions.
- **Packer / Ansible / PowerCLI conversion.** These are imperative and drift-prone; they deserve their own focused treatment later, not a shallow v1 subsection.
- **SaaS repurchase** or **retain/retire** strategies beyond flagging them.
- **Direct execution of mutations.** Falcon is read-only by platform constraint; every mutation is user-executed.

## Scope decisions (recorded during brainstorming)

| # | Decision | Rationale |
|---|----------|-----------|
| 1 | Form factor: single skill with orchestrator `SKILL.md` + `references/` (not multi-skill family, not extending `cloud-architect`) | Aligns with documented FalconClaw skill format; orchestrator + references pattern documented in `neubirdai/falconclaw` DESIGN.md. |
| 2 | Execution model: capability-probed adaptive | Real engagements have varying tool availability; skill detects mode from live probe rather than asking the user. |
| 3 | v1 migration strategies: Rehost + Replatform | Sweet spot for VMware → AWS engagements; Refactor / Repurchase / Retain / Retire are out of scope. |
| 4 | Discovery source authority: live vCenter (resource presence) > IaC (declared intent, conversion target) > telemetry (sizing, drift signal); static exports are fallback for (1) when vCenter unreachable | Different sources are authoritative for different questions. Precedence rules are codified in `drift-detection.md`. |
| 5 | Drift detection as first-class concern | Reconciles all three sources before any mutation recommendation. |
| 6 | Template coverage v1: VCF Automation, Aria Automation blueprints, Terraform vSphere, OVF/OVA | Covers the declarative IaC surface; leaves imperative formats (Packer/Ansible/PowerCLI/PowerShell) for later focused treatment. |
| 7 | Output model: inline code blocks + copy-ready bundles (file path + content in one block) | Falcon is read-only — no file writes, no GitHub PRs. |
| 8 | Boundaries: standalone | Skill owns the full VMware → AWS workflow end-to-end; no hand-offs to other skills. Description narrows domain to VMware → AWS to avoid collision with `cloud-architect`. |
| 9 | Session resumption: structured prompt + mandatory read-only reconciliation | Without a state file, conversation history is unreliable. Reconciliation against live state catches divergence. |
| 10 | Layout: flat `references/` (Approach B) | Matches `cloud-architect`'s reference-table convention; clean promotion path to nested `references/` when 3+ destinations exist. |

## Architecture

### Directory layout

```
skills/migration-assistant/
├── SKILL.md                                (~280 lines, hard cap 500)
├── references/
│   ├── sources-vcf-automation.md           (~250 lines)
│   ├── sources-aria-blueprint.md           (~200 lines)
│   ├── sources-terraform-vsphere.md        (~300 lines)
│   ├── sources-ovf.md                      (~150 lines)
│   ├── destinations-aws.md                 (~350 lines)
│   ├── discover-and-plan.md                (~400 lines)
│   ├── execute-and-verify.md               (~350 lines)
│   └── drift-detection.md                  (~200 lines)
└── scripts/
    ├── probe-vcenter.sh                    (~200 lines)
    └── parse-rvtools.py                    (~150 lines)
```

Line counts are ceilings, not targets. FalconClaw's validator enforces the 500-line cap on `SKILL.md` only; references and scripts are unbounded.

### Frontmatter (`SKILL.md`)

```yaml
---
name: migration-assistant
description: "Executes VMware → AWS migrations end-to-end: capability-probed discovery (live vCenter, GitHub IaC, telemetry), drift detection, 6Rs rehost/replatform decisions, template conversion (VCF Automation, Aria blueprints, Terraform vSphere, OVF) to AWS equivalents, wave-based cutover runbooks, post-cutover validation, and monitoring setup. Falcon is read-only: the skill presents commands and copy-ready IaC bundles; the user executes them. Invoke when workloads are in VMware (vSphere/VCF) and the target is AWS."
tags: [migration, vmware, aws, cloud]
homepage: https://github.com/neubirdai/falconclaw-skills/tree/main/skills/migration-assistant
metadata: {"neubird":{"emoji":"🚚","requires":{"anyBins":["aws","govc"]}}}
---
```

**Design choices:**

- **Narrow description.** The description is the main signal Falcon's model uses for invocation routing. Precise wording ("VMware → AWS", specific template formats, read-only framing) is the defense against overlap with `cloud-architect` on generic cloud-migration questions.
- **`requires.anyBins: ["aws", "govc"]`.** Gates the skill to environments that have at least one side of the migration tooled up. Realistic in practice. If this proves too strict after user feedback, loosen to include `gh` or drop `requires` entirely.
- **`🚚` emoji** distinct from adjacent skills.

## SKILL.md orchestrator

### Structure

```markdown
---
<frontmatter above>
---

# Migration Assistant (VMware → AWS)

<3-5 line preamble: scope, read-only, all commands user-executed>

## Workflow Overview
<ASCII flowchart of the 7 phases with gates>

## Phase 0: Capability Probe (REQUIRED FIRST STEP)
<probe commands + mode-interpretation rules>

## Phase 0.5: Resume vs. Fresh Start
<structured resume prompt + mandatory reconciliation>

## Phases 1–7 (brief)
<one paragraph per phase, each ending "Load `references/<file>.md`">

## Reference Guide
<table mapping topic → reference file → trigger condition>

## Scripts
<when to suggest probe-vcenter.sh and parse-rvtools.py; output schema pointer>

## Output Conventions
<copy-ready bundle format; wave/step numbering; expected-output pattern>

## Constraints (MUST DO / MUST NOT DO)
<hard rules>
```

### Phase 0 — Capability probe

Runs the first time the skill activates in a session, **before answering any user question**. Fixed set of read-only probes:

| Probe | Approach | Signals |
|-------|----------|---------|
| AWS reachable | `aws sts get-caller-identity` | account ID, active region |
| vCenter reachable | `govc about` (only if `GOVC_URL` + `GOVC_USERNAME` are set) | live-discovery mode available |
| IaC in cwd | `find . -maxdepth 3` for `*.tf` and `*.yaml`/`*.yml`, content-match for `vmware\|vcf\|aria\|blueprint` keywords | repo-based conversion source and its template format |
| Static exports | look for `RVTools*.xlsx` / `RVTools*.csv` in cwd | RVTools fallback available |
| Telemetry hints | environment-variable scan for `VROPS_`, `ARIA_`, `PROMETHEUS_`, `DATADOG_` prefixes | telemetry sources for right-sizing |

Skill emits a one-line summary and declares operating mode (e.g. "Mode: live vCenter + AWS + repo IaC, no telemetry — will right-size from live config only"). Downstream phases branch on mode.

### Phase 0.5 — Resume protocol

Structured prompt at session start:

1. New migration or resuming?
2. If resuming: current phase · waves completed · AWS account ID · source vCenter · last validated checkpoint.
3. **Mandatory** — re-run capability probe AND reconcile user-reported state vs. live reality (live VM count, live EC2 count in target account/region). Flag any divergence before suggesting the next step.

### Decision gates (six)

| # | Gate | What Falcon presents | What user confirms | Why |
|---|------|---------------------|---------------------|-----|
| 1 | After capability probe | Probe summary + declared operating mode | Mode is correct / adjust env | Wrong mode cascades through entire migration |
| 2 | After discovery + drift | Workload inventory + drift report | Inventory correct, scope-in/out, drift blocking vs informational | Wrong inventory poisons everything downstream |
| 3 | After 6Rs decision per workload | Per-workload recommendation with rationale | Approve all or override specific workloads | 6Rs determines conversion target; reversing late is expensive |
| 4 | After wave grouping | Waves, sequence, windows, rollback windows | Composition, dependencies, timing | Last chance to reshape plan before execution |
| 5 | Before every wave cutover (N times) | Cutover runbook + paired rollback runbook + validation checklist | "Execute Wave N now" (per wave) | Mutation point; enforces atomic waves = atomic rollback |
| 6 | Before decommissioning source VMs | Validation evidence + per-VM decommission list + cooling-period confirmation | Explicit approval per wave | Decommission is irreversible; default 72-hour cooling period |

Gates 5 and 6 are non-negotiable. Gates 1–4 could in principle be collapsed; the design keeps them separate to enable tight feedback loops (reject 6Rs → re-run wave grouping) and reduce single-approval cognitive load.

### Output conventions

- Converted IaC as copy-ready bundle: fenced code block with a `# file: <path>` header as the first line inside the block.
- Commands numbered `Wave N, Step M` within the wave's runbook.
- Every command paired with an **Expected output** block showing the success signature (for user-side verification and Falcon readback on resume).

### Constraints

**MUST DO:**
- Run Phase 0 capability probe before any other phase, every session.
- Present phase results and wait for explicit user approval at every decision gate before advancing.
- Web-search AWS best practices for the specific service + region before finalizing a replatform recommendation. Service behavior and quotas change; training-data-only recommendations are a known failure mode.
- Tag every recommended AWS resource with `migration-wave`, `migration-source-vm-id`, `migration-date` for traceability.
- Emit a rollback runbook for every wave, delivered before the cutover runbook.
- Reconcile drift (live vs IaC vs telemetry) before recommending any sizing.

**MUST NOT DO:**
- Execute mutations directly. Falcon is read-only, always.
- Skip the capability probe or resume reconciliation.
- Recommend deleting source VMs until validation passes AND user explicitly approves decommission (Gate 6).
- Bundle unrelated workloads in one wave.
- Emit IaC with hardcoded secrets, IAM wildcards, or public `0.0.0.0/0` ingress without explicit user override.
- Assume current AWS service behavior from training data — web-search for current quotas/limits/deprecations before committing to a target.

## Reference file contents

### Phase drivers

**`references/discover-and-plan.md`** (~400 lines) — Drives Phases 1–3. Contains:
- Mode-branched discovery recipes (one flow per capability-probe outcome).
- Telemetry ingestion for right-sizing (p95 CPU/mem, IOPS, network).
- Workload classification rules.
- **6Rs decision matrix for VMware → AWS**: decision tree per workload, prefers managed services (MSSQL → RDS, NFS → EFS, AD → Managed AD, Exchange → WorkMail) unless license/appliance/hardware constraint overrides. Explicit criteria for when to rehost (custom appliances, license-bound, specialized hardware).
- Phase 3 wave-grouping rules: dependency discovery (NSX flow data when available, else network-flow inference from telemetry + live connections), risk-tier sequencing (pilot → stateless → stateful → critical), cutover window sizing.
- Embedded web-search anchors at decision points ("before committing to RDS engine version, web-search current support matrix").

**`references/execute-and-verify.md`** (~350 lines) — Drives Phases 5–7. Contains:
- Pre-cutover checklist (DNS TTL reduced, LB pre-configured, rollback tested).
- Per-wave cutover runbook templates branched by strategy:
  - Rehost via AWS MGN: replication status → test launch → cutover launch.
  - Replatform: service-specific (RDS via DMS, EFS via DataSync, etc.).
- Validation runbook: ALB target-group health, application smoke tests, before/after metrics comparison.
- Post-cutover monitoring setup: CloudWatch alarm baselines (CPU/memory via CW agent, 5xx rates, RDS connection counts), Systems Manager Inventory + Patch Compliance, AWS Config rules for drift detection on the new AWS-native stack.
- Paired rollback runbook template.
- Gate 6 decommissioning runbook: snapshot-before-delete, 72-hour default cooling period, explicit per-VM approval.

### Cross-cutting

**`references/drift-detection.md`** (~200 lines) — Contains:
- Three-way reconciliation algorithm (live vs IaC vs telemetry).
- Classification taxonomy: stale-IaC, over-provisioned, under-provisioned, untracked-in-IaC, present-in-IaC-absent-live.
- **Trust-precedence rules:** sizing → telemetry p95 > live > IaC; resource presence → live > IaC; naming/tagging → IaC.
- Drift-report output format (structured JSON + human summary).
- User-decision rubric: which drift is blocking vs informational.

### Source-format conversions

Each contains: format anatomy, resource-type mapping matrix to AWS, 2–3 concrete before/after conversion examples, "gotchas" listing constructs with no AWS equivalent that need human review.

**`references/sources-vcf-automation.md`** (~250 lines) — VCF Automation YAML → Terraform AWS. Key mappings: `Cloud.vSphere.Machine` → `aws_instance`, VCF networks → VPC/subnets, storage profiles → EBS volume types. Gotchas: VCF-specific inputs, custom resource types, Cloud Assembly-isms.

**`references/sources-aria-blueprint.md`** (~200 lines) — Aria Automation (vRA 8.x) blueprint YAML → Terraform AWS. Key mappings: property groups, input parameters → SSM Parameters / Launch Template userdata. Gotchas: day-2 actions, approval policies (no direct AWS equivalent — flag for IAM / Step Functions consideration).

**`references/sources-terraform-vsphere.md`** (~300 lines) — `hashicorp/vsphere` provider → `hashicorp/aws` provider. Key mappings: `vsphere_virtual_machine` → `aws_instance` + `aws_ebs_volume`; `vsphere_distributed_port_group` → `aws_subnet` + `aws_security_group`; `vsphere_tag` → resource tags; `vsphere_content_library_*` → AMI / S3. State-migration guidance: strongly prefer new TF state for AWS side over in-place provider swap (explains why). Provider config replacement snippets.

**`references/sources-ovf.md`** (~150 lines) — OVF/OVA descriptors → AMI via AWS VM Import/Export. Precise command sequence: upload OVA to S3 → `aws ec2 import-image` → poll `describe-import-image-tasks`. Supported-OS matrix with VM Import limitations; workarounds for unsupported OSes (use MGN instead).

### Destination

**`references/destinations-aws.md`** (~350 lines) — AWS service-selection matrix. Contains:
- Rehost targets: EC2 instance-family picker (compute / memory / storage / GPU) with right-sizing formula from telemetry.
- Replatform targets: Database (RDS engines, Aurora, DocumentDB, OpenSearch), Storage (EFS, FSx Windows/NetApp/OpenZFS, S3), Identity (Managed AD, IAM Identity Center), Messaging (MQ, MSK, SQS/SNS).
- Networking patterns: VPC design, multi-AZ subnets, Transit Gateway for multi-VPC, Site-to-Site VPN or Direct Connect for cutover connectivity.
- Monitoring baseline: CloudWatch + CW Agent + Config.
- IaC preference: Terraform `hashicorp/aws` provider, modular structure.
- **Web-search anchors embedded at decision points** ("before finalizing RDS engine version / instance type, search current AWS service limits, deprecations, region availability").

## Scripts

Both scripts are read-only helpers. User executes; Falcon reads the output.

### Common JSON schema (contract)

Both scripts emit the **same** JSON shape so downstream skill logic has one code path:

```json
{
  "schemaVersion": "1",
  "source": "govc | rvtools",
  "capturedAt": "<ISO-8601 UTC>",
  "vcenter": {"url": "...", "version": "..."},
  "clusters": [...],
  "hosts": [...],
  "vms": [{
    "id": "vm-1234",
    "name": "web-01",
    "vcpus": 4,
    "memoryMB": 8192,
    "disks": [{"sizeGB": 80, "datastore": "ds-ssd", "thin": true}],
    "networks": [{"name": "web-net", "mac": "..."}],
    "guestOs": "ubuntu20_64Guest",
    "tags": [{"category": "app", "value": "web"}],
    "powerState": "on"
  }],
  "networks": [...],
  "datastores": [...]
}
```

Schema is documented in a comment at the top of each script and reiterated in `discover-and-plan.md`.

### `scripts/probe-vcenter.sh` (~200 lines)

- Requires `GOVC_URL`, `GOVC_USERNAME`, `GOVC_PASSWORD`.
- Runs fixed read-only sweep: `govc ls`, `govc vm.info -json`, `govc host.info`, `govc datastore.info`, `govc network.info`.
- Assembles into schema, emits to stdout.
- Non-zero exit with structured JSON error on auth/connectivity failure.
- No mutating verbs anywhere.

### `scripts/parse-rvtools.py` (~150 lines)

- Takes RVTools `.xlsx` or CSV path as arg.
- Reads `vInfo`, `vCPU`, `vMemory`, `vDisk`, `vNetwork`, `vHost`, `vDatastore` tabs.
- Normalizes into the same schema.
- Stdlib `csv` for CSVs; `openpyxl` for xlsx (listed in a header comment — user installs if needed).

## Extension plan

Adding a new destination (e.g., VMware → OpenShift) requires:
1. New `references/destinations-openshift.md` with the analogous service-selection matrix.
2. New `references/execute-and-verify-openshift.md` with OpenShift-flavored runbooks (or a branched addition to `execute-and-verify.md` if similarities dominate).
3. Updates to `discover-and-plan.md` 6Rs matrix with an OpenShift column.
4. Description update to broaden the scope clause.

Adding a new source (e.g., Hyper-V → AWS) requires:
1. New `references/sources-hyperv.md`.
2. Adjust capability probe to detect Hyper-V tooling (`SCVMM`, PowerShell modules).
3. A new probe script analogous to `probe-vcenter.sh`.

Hitting 3+ destinations is the trigger to promote from flat `references/` (Approach B) to nested (`references/destinations/*`, `references/sources/*`) — Approach C per the design exploration. This is a mechanical file-move, not a content rewrite.

## Open items / deferred

- **Install spec in metadata.** We have `requires.anyBins: ["aws","govc"]` but no install spec for `govc`. Implementation plan decides whether to add an `install` array (brew + go paths) or leave install as user responsibility.
- **`user-invocable` flag.** Default is true — the skill can be invoked as a slash command. Keep the default unless a reason surfaces to restrict.
- **Telemetry source enumeration.** v1 targets Aria Operations, Prometheus, Datadog. New Relic and generic CloudWatch-on-vSphere are not explicitly enumerated; add if a user engagement surfaces them.
- **Script dependency: `openpyxl`.** If FalconClaw's execution environment does not include Python xlsx libraries by default, the `parse-rvtools.py` script's xlsx path needs either a graceful fallback to "export as CSV instead" or an install hint.

## Success criteria

**Automated (verifiable in CI or local test):**
- `validate.yml` passes: frontmatter valid, directory name matches `name`, `SKILL.md` ≤ 500 lines.
- `probe-vcenter.sh` and `parse-rvtools.py` emit JSON that conforms to the v1 schema (validated against a JSON schema file committed alongside the scripts).
- Example conversion snippets in each `sources-*.md` are syntactically valid: HCL examples pass `terraform fmt -check`; YAML examples parse cleanly.

**Manual (verified by walking through the skill in Falcon):**
- From a simulated vCenter probe and a sample IaC repo, the skill can walk a reviewer through all seven phases and stop at each of the six decision gates.
- Emitted copy-ready IaC bundle from a sample VCF Automation template produces Terraform that passes `terraform validate` when dropped into an empty module.
- Resume protocol correctly flags divergence when a reviewer lies about the current phase during a second session.
