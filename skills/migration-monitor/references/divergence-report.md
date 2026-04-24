# Divergence Report Reference

## Purpose

Phase 7 consolidates all divergences collected by Phases 3–6 (infra, endpoint, metrics, and data
parity checks), classifies each by severity, attaches remediation hints, and presents a
consolidated report at Gate 3 for user disposition. The report is emitted in two forms: a
machine-parseable JSON block and a human-readable summary. No phase output is discarded — every
divergence from every phase appears in the report, even if the phase recorded zero divergences.

---

## Report JSON schema

Phase 7 emits the following JSON structure. Field descriptions follow the example block.

```json
{
  "reportVersion": "1",
  "generatedAt": "2026-04-24T15:00:00Z",
  "generatedBy": "falcon migration-monitor",
  "summary": {
    "critical": 2,
    "major": 5,
    "minor": 12,
    "skippedPhases": ["data"],
    "totalPairs": 14,
    "pairsWithCriticalDivergence": 1
  },
  "skipReasons": {
    "data": "SOURCE_DB_URI and TARGET_DB_URI not set in env"
  },
  "comparisonWindow": {
    "source_start": "2026-04-10T00:00:00Z",
    "source_end": "2026-04-17T00:00:00Z",
    "target_start": "2026-04-17T00:00:00Z",
    "target_end": "2026-04-24T00:00:00Z"
  },
  "phases": {
    "infra": [
      {
        "id": "infra-001",
        "phase": "infra",
        "type": "compute.vcpu-deficit",
        "severity": "major",
        "resourcePair": {"vmware_id": "vm-1001", "aws_id": "i-0abc"},
        "description": "Target EC2 has 2 vCPU vs source VM 4 vCPU (ratio 0.5, below 0.8 floor)",
        "evidence": {"source_vcpu": 4, "target_vcpu": 2, "ratio": 0.5, "floor": 0.8},
        "remediationHint": "Resize instance to at least t3.xlarge (4 vCPU): aws ec2 modify-instance-attribute --instance-id i-0abc --instance-type Value=t3.xlarge. Web-search current AWS EC2 instance types in the target region before applying."
      }
    ],
    "endpoint": [
      {
        "id": "endpoint-001",
        "phase": "endpoint",
        "type": "endpoint.status-mismatch",
        "severity": "critical",
        "resourcePair": {"vmware_id": "vm-1001", "aws_id": "i-0abc"},
        "pairId": "pair-web-01",
        "description": "HTTP status differs: source 200, target 500",
        "evidence": {"probe": "http", "source_url": "https://web-01.corp.example/", "target_url": "https://web-01-alb-xxx.elb.amazonaws.com/", "source_http_code": 200, "target_http_code": 500},
        "remediationHint": "Check ALB target-group health: aws elbv2 describe-target-health --target-group-arn <arn>. Inspect app-side config, reverse proxy rules, 5xx root cause in CloudWatch Logs."
      }
    ],
    "metrics": [
      {
        "id": "metrics-001",
        "phase": "metrics",
        "type": "metrics.regression",
        "severity": "major",
        "resourcePair": {"vmware_id": "vm-1001", "aws_id": "i-0abc"},
        "description": "Target CPU p95 is 42% higher than source (source p95 60%, target p95 85%)",
        "evidence": {"metric": "CPUUtilization", "source_p95": 60, "target_p95": 85, "regression_pct": 0.42, "window_days": 7},
        "remediationHint": "Scale up instance type or enable CloudWatch target-tracking scaling. Investigate noisy-neighbor on shared tenancy; consider dedicated-host deployment if consistent."
      }
    ],
    "data": []
  }
}
```

### Field semantics

- `reportVersion` — schema version; always `"1"` (string, not integer) for v1 reports.
- `generatedAt` — ISO-8601 UTC timestamp at which Falcon renders the report.
- `generatedBy` — always `"falcon migration-monitor"`; identifies the producing skill.
- `summary.critical` / `summary.major` / `summary.minor` — total divergence counts by severity
  across all phases combined.
- `summary.skippedPhases` — list of phase names that did not run because their surface was
  unavailable at Phase 0 surface-availability detection (e.g., no DB credentials → data skipped).
- `summary.totalPairs` — count of workload pairs produced by Phase 1 mapping and locked at Gate 2.
- `summary.pairsWithCriticalDivergence` — count of distinct resource pairs that have at least one
  critical-severity divergence across any phase. Multiple critical divergences within the same pair
  count as 1 (not additive).
- `skipReasons` — map of phase name → human-readable reason string, one entry per phase listed in
  `summary.skippedPhases`. Populated from Phase 0 surface-availability log.
- `comparisonWindow` — Optional — present only if Phase 5 ran. Absent if metrics parity was skipped. The source and target time windows used for metrics parity (Phase 5). Captured from Phase 5 output.
- `phases.infra` / `phases.endpoint` / `phases.metrics` / `phases.data` — arrays of divergence
  objects per phase. An empty array (`[]`) means the phase ran and found no divergences. A missing
  key means the phase was skipped (reconstruct from `summary.skippedPhases`).

Each divergence object inside a phase array carries these required fields:

- `id` — unique identifier within the run (e.g., `infra-001`, `endpoint-042`).
- `phase` — phase name string matching the containing key (`"infra"`, `"endpoint"`, `"metrics"`,
  `"data"`).
- `type` — divergence type string (see severity rubric below). The stable contract for downstream
  consumers — do not parse `description` for logic.
- `severity` — one of `"critical"`, `"major"`, `"minor"`.
- `resourcePair` — the VM-to-EC2 pair from Phase 1 mapping: `{"vmware_id": "...", "aws_id": "..."}`.
- `description` — human-readable one-line summary of what differs.
- `evidence` — structured object with probe-type-specific or check-type-specific fields. Schema
  varies by divergence type; consumers must tolerate absent fields.
- `remediationHint` — template string with `<placeholder>` markers Falcon substitutes at emit time.

Endpoint divergences additionally carry `pairId` (the endpoint pair ID from Phase 4). Metrics
divergences additionally carry `metric` (the source and target metric names).

---

## Severity rubric

Three severity levels are used across all phases. Phase 7 inherits the classification from the
emitting phase — no re-classification occurs at aggregation time.

### Critical — functional break suspected

Migration integrity is compromised. The workload is likely broken or unusable in its current state
on the target. Critical divergences must be resolved or explicitly accepted before a migration can
be considered complete.

Examples:
- Missing DB table on target (`data.table.missing-on-target`)
- HTTP status mismatch (e.g., source 200, target 500) (`endpoint.status-mismatch`)
- EBS encryption missing when source had VMware disk encryption (`storage.encryption-missing`)
- Metrics regression >50% above source baseline (`metrics.regression` critical tier)
- OS family mismatch between source and target (`compute.os-mismatch`)
- TLS certificate expired on target (`endpoint.tls-expired`)
- DB connection failure to source or target (`data.connection-failure`)
- Target publicly unreachable when source was public (`endpoint.reachability-regression`)
- Public/private exposure posture inverted (`exposure.public-regression`, `exposure.private-regression`)

### Major — observable regression, not an outright functional break

The workload may be running but exhibits user-visible performance degradation, security gaps, or
completeness problems. Major divergences are not blocking by default but require operator attention
before sign-off.

Examples:
- 30–50% metrics regression (CPU, memory, latency, error rate)
- 1–10% DB row count deficit on a paired table (`data.rowcount.deficit`). Deficit >10% escalates to critical.
- vCPU or memory below 80% of source provisioned level (`compute.vcpu-deficit`, `compute.memory-deficit`)
- Missing security group inbound or outbound rule (`security.inbound.missing`, `security.outbound.missing`)
- Missing required security header on target endpoint (`endpoint.security-header-missing`)
- Required tag key absent from target resource (`tags.required-missing`)

### Minor — cosmetic, within tolerance, or informational

Not blocking. Operator awareness is recommended but no action is required before sign-off unless
the user escalates a specific divergence.

Examples:
- Source tag key not propagated to target for non-required tags (`tags.propagation-missing`)
- Extra column on target that was not present on source (`data.column.extra-on-target`)
- 10–30% metrics regression within the lower band (`metrics.regression` minor tier)
- HTML structural drift between source and target endpoint bodies (`endpoint.body-html-drift`)
- Extra inbound rule on target security group not present on source (`security.inbound.extra-open`)
- Source metric has no CloudWatch equivalent (`metrics.unmapped`)

### Divergence type to severity mapping

Quick-reference mapping for the most common divergence types:

| Divergence type | Severity |
|---|---|
| `compute.os-mismatch` | critical |
| `compute.architecture-mismatch` | critical |
| `storage.encryption-missing` | critical |
| `data.table.missing-on-target` | critical |
| `data.column.missing-on-target` | critical |
| `endpoint.status-mismatch` | critical |
| `endpoint.tls-expired` | critical |
| `endpoint.reachability-regression` | critical |
| `exposure.public-regression` | critical |
| `exposure.private-regression` | critical |
| `metrics.regression` | minor / major / critical (scales with % deviation) |
| `compute.vcpu-deficit` | major (critical at ratio < 0.6) |
| `compute.memory-deficit` | major (critical at ratio < 0.6) |
| `security.inbound.missing` | major |
| `security.outbound.missing` | major |
| `data.rowcount.deficit` | major (critical at deficit > 10%) |
| `endpoint.security-header-missing` | major |
| `tags.required-missing` | major |
| `tags.propagation-missing` | minor |
| `tags.migration-tracking-absent` | minor |
| `endpoint.body-html-drift` | minor |
| `endpoint.both-unreachable` | minor |
| `security.inbound.extra-open` | minor |
| `data.column.extra-on-target` | minor |
| `metrics.unmapped` | minor |

---

## Remediation hints

Remediation hint strings are template-ready: `<placeholder>` markers are substituted by Falcon at
emit time using per-divergence evidence values. Downstream consumers should treat hint strings as
opaque — do not parse placeholders programmatically.

| Divergence type | Remediation hint template |
|---|---|
| `network.cross-vpc-fragmentation` | AWS resources for a cluster span multiple VPCs. Consolidate to one VPC or add Transit Gateway peering. `aws ec2 create-transit-gateway-peering-attachment --transit-gateway-id <tgw-id> --peer-transit-gateway-id <peer>`. |
| `compute.vcpu-deficit` | Resize instance to at least `<suggested_type>`: `aws ec2 modify-instance-attribute --instance-id <aws_id> --instance-type Value=<suggested_type>`. Web-search current AWS EC2 instance types in `<region>` before applying. |
| `compute.memory-deficit` | Resize to memory-optimized family (r7i/r7a) or larger compute family: `aws ec2 modify-instance-attribute --instance-id <aws_id> --instance-type Value=<suggested_type>`. |
| `compute.os-mismatch` | OS family mismatch typically requires re-provisioning. Verify AMI OS family matches source. Cannot be fixed in-place. |
| `compute.architecture-mismatch` | Architecture mismatch (x86_64 vs arm64) requires re-provisioning. Check AMI architecture against source binary compatibility. |
| `storage.capacity-deficit` | Extend EBS volume: `aws ec2 modify-volume --volume-id <vol_id> --size <new_size_gb>`. Then grow filesystem on-instance: `growpart /dev/<dev> <part>; resize2fs /dev/<dev><part>` (Linux) or `Resize-Partition` (Windows). |
| `storage.encryption-missing` | Cannot add encryption in-place. Create snapshot → create encrypted copy → detach original → attach encrypted copy → remount. Plan downtime. |
| `storage.iops-underprovisioned` | Provision more IOPS: `aws ec2 modify-volume --volume-id <vol_id> --iops <source_p95_iops * 1.2> --volume-type gp3` (or io2 for >16k IOPS). |
| `security.inbound.missing` | Allow the source-side inbound rule on target SG: `aws ec2 authorize-security-group-ingress --group-id <sg_id> --protocol <proto> --port <from>-<to> --cidr <cidr>`. |
| `security.outbound.missing` | Add outbound rule: `aws ec2 authorize-security-group-egress --group-id <sg_id> --protocol <proto> --port <from>-<to> --cidr <cidr>`. |
| `security.inbound.extra-open` | Target allows traffic source didn't — review whether intentional. To restrict: `aws ec2 revoke-security-group-ingress --group-id <sg_id> ...`. |
| `exposure.public-regression` | Source was publicly reachable; target is private. Add public-facing ALB/NLB or attach EIP. Check DNS: `aws route53 list-resource-record-sets` for stale pointers. |
| `exposure.private-regression` | Target is publicly exposed; source was private. Remove public IP / public-facing LB. Move to private subnet. |
| `tags.required-missing` | Add required tags: `aws ec2 create-tags --resources <aws_id> --tags Key=<k>,Value=<v>`. |
| `tags.propagation-missing` | Source tag not propagated to target. Optional: replicate via `aws ec2 create-tags`. |
| `tags.migration-tracking-absent` | Informational — migration-tracking tags (`migration-wave`, `migration-source-vm-id`, etc.) not present. v1 doesn't require them; add for traceability via `aws ec2 create-tags --resources <aws_id> --tags Key=migration-source-vm-id,Value=<vmware_id>`. |
| `endpoint.status-mismatch` | Check ALB target-group health: `aws elbv2 describe-target-health --target-group-arn <arn>`. Inspect app-side config, reverse proxy rules. Check 5xx root cause in CloudWatch Logs for the relevant log group. |
| `endpoint.reachability-regression` | Verify security group / NACL / route table / subnet. Confirm LB listener matches source port. `aws ec2 describe-network-acls --filters Name=vpc-id,Values=<vpc_id>` to inspect NACLs. |
| `endpoint.tls-expired` | Renew certificate. If ACM-managed: `aws acm list-certificates --certificate-statuses EXPIRED` to find. If self-managed, coordinate with ops for renewal. |
| `endpoint.tls-cn-mismatch` | Cert subject CN doesn't match endpoint. Issue new cert matching hostname (ACM or internal CA). |
| `endpoint.body-structural-mismatch` | Application-layer divergence — not infra. Inspect app config, version pin, migration-time code changes. Compare app versions source vs target. |
| `endpoint.security-header-missing` | Add missing security header to the target's web server, reverse proxy, or ALB response rules. For ALB: use a response-header modification rule in the listener. |
| `endpoint.tls-handshake-failure` | TLS handshake failed on `<side>`. Check SG allows port 443 from Falcon's IP. Verify certificate is installed and the TLS listener is active. `aws elbv2 describe-listeners --load-balancer-arn <arn>`. |
| `endpoint.content-type-mismatch` | Target returns different Content-Type than source. Inspect application config, caching layer, or reverse-proxy rewrite rules. Check ALB listener rules for content-type manipulation. |
| `endpoint.size-deviation` | Target response body size deviates from source by >50%. Inspect application-layer changes, compression (gzip/br) enabled/disabled differently, or payload version drift. |
| `endpoint.body-html-drift` | Target HTML differs structurally from source after normalization. Often benign (framework upgrade, template regeneration). Compare rendered output manually; escalate to major only if user-facing impact confirmed. |
| `endpoint.body-bytes-mismatch` | Non-JSON/HTML body bytes differ. Target may serve a different binary asset version. Compare asset hashes and deployment manifests. |
| `endpoint.unreachable` | Target endpoint unreachable at curl level (network failure / connection refused). Check target host is running, security groups allow Falcon's IP, DNS resolves. `aws ec2 describe-instances --instance-ids <aws_id> --query 'Reservations[].Instances[].State.Name'`. |
| `endpoint.tls-pre-valid` | Target TLS cert is not-yet-valid (notBefore in future). Cert was issued with incorrect date or system clock skew exists. Verify cert issuance time; renew if needed. |
| `endpoint.both-unreachable` | Neither source nor target is reachable. Often indicates source was decommissioned during migration. Informational — verify this was intentional. |
| `metrics.regression` (any severity) | Investigate root cause by metric: CPU/memory regression → scale up instance type, enable CloudWatch target-tracking scaling, investigate noisy-neighbor on shared tenancy; latency regression → inspect app-side config for cold-start, N+1 queries, cache hit rates; error-rate regression → `aws logs filter-log-events` on ALB access logs and app logs. |
| `metrics.unmapped` | Source metric has no CloudWatch equivalent in v1 mapping table. Manually correlate or request mapping-table extension in future skill release. |
| `metrics.source-unavailable` | Catalog enumeration on a metric source (Prometheus/Datadog/etc.) failed. Check credentials (`PROMETHEUS_URL`, `DATADOG_API_KEY`, etc.), network reachability, service health. |
| `metrics.insufficient-samples` | Fewer than required data points for statistical comparison. Extend comparison window or verify metric emission frequency. Skip this metric's divergence this run. |
| `data.table.missing-on-target` | Migration incomplete — source table absent on target. Check DMS replication task status: `aws dms describe-replication-tasks --filters Name=replication-instance-arn,Values=<arn>`. Restart or extend task. |
| `data.rowcount.deficit` | If DMS CDC still running, wait for lag to drain; check `aws dms describe-replication-tasks` for `PendingChanges`. Otherwise investigate replication errors in CloudWatch Logs for the task. |
| `data.column.missing-on-target` | Schema regression. Compare schemas: `\d+ <table>` (psql) or `SHOW CREATE TABLE <table>` (mysql). Reapply missing schema changes via migration tool (Flyway / Liquibase). |
| `data.connection-failure` | Cannot reach target DB. Check security group allows Falcon's IP on DB port. Check RDS status: `aws rds describe-db-instances --db-instance-identifier <id>`. |
| `data.table.extra-on-target` | Target has tables source doesn't. Often expected schema evolution post-migration. Informational — verify tables are intentional additions, not replication errors. |
| `data.rowcount.surplus` | Target has more rows than source. May indicate duplicate inserts from replication restart or data-imports run twice. Investigate DMS task history for restart events. |
| `data.column.extra-on-target` | Target table has columns source doesn't. Often expected schema evolution. Verify columns are intentional via schema-migration tool history. |
| `data.column.position-mismatch` | Same-named columns in different ordinal positions. Applications using `SELECT *` or positional column binding may break. Consider re-ordering via migration or explicit column lists in SQL. |
| `data.engine-unavailable` | Required DB CLI (`mysql`/`psql`/`sqlcmd`) not on Falcon's PATH. Install the engine client in Falcon's execution environment, or skip DB parity for this engine. |
| `data.query-timeout` | SQL query exceeded timeout (default 5 minutes). Table is very large, or engine is under heavy load. Try during off-peak, or exclude the table with `--exclude-table <pattern>` for this run. |

---

## Human-summary table format

In addition to the JSON output, Phase 7 renders a human-readable summary at Gate 3. This is what
Falcon prints for the operator to review before disposition.

```text
Migration Monitor Report — generated 2026-04-24T15:00:00Z
Workloads paired: 14 | skipped phases: [data]
Summary: 2 critical, 5 major, 12 minor

── CRITICAL ──
[infra-003] vm-1001 → i-0abc: encryption-missing
            Target EBS volumes unencrypted; source had VMware disk encryption.
            Remediation: re-provision with encrypted snapshot (plan downtime).

[endpoint-001] vm-1001 → i-0abc (pair-web-01): status-mismatch
            HTTP status differs: source 200, target 500.
            Remediation: check ALB target health + app-side 5xx root cause in
            CloudWatch Logs.

── MAJOR ──
[infra-001] vm-1001 → i-0abc: vcpu-deficit (2 vCPU vs 4, ratio 0.5)
            Remediation: aws ec2 modify-instance-attribute ...

[metrics-002] vm-1003 → i-0def: regression (CPU p95 +42%)
            Remediation: scale up or enable auto-scaling.

... (12 minor divergences collapsed — expand for details)
```

### Format rules

- Severity groups appear in order: critical first, then major, then minor.
- Within each severity group, entries are grouped by resource pair
  (`vmware_id → aws_id`).
- Each entry shows: ID in brackets, pair identifiers, divergence type, a one-line description, and
  the first sentence of the remediation hint.
- Endpoint divergences additionally show the `pairId` in parentheses after the pair identifiers.
- Minor divergences are collapsed when there are more than 5 in the minor group. A single note
  counts them and offers expansion at Gate 3 (e.g., "12 minor divergences collapsed — expand for
  details").
- The header line always shows: generation timestamp, paired workload count, skipped phases list,
  and the three-number severity summary.

---

## Report generation notes

How Falcon builds the report at Phase 7:

1. **Aggregate divergences.** Collect the divergence arrays emitted by each of Phases 3–6. Each
   phase emits its own array independently; Phase 7 concatenates them into the `phases` map.

2. **Count by severity.** Iterate all collected divergences and increment `summary.critical`,
   `summary.major`, or `summary.minor` based on each divergence's `severity` field.

3. **Populate skipped phases.** At Phase 0, Falcon records surface-availability results. Phase 7
   reads these to populate `summary.skippedPhases` and `skipReasons`. A phase appears in
   `skippedPhases` if and only if it did not run; its key is omitted from the `phases` map entirely
   (rather than being set to an empty array).

4. **Compute pairsWithCriticalDivergence.** Group critical divergences by `resourcePair.vmware_id`.
   Count distinct vmware_id values — multiple critical divergences on the same pair count as 1.

5. **Attach remediation hints.** For each divergence, Falcon looks up the type in the remediation
   table above and substitutes evidence-field values into `<placeholder>` markers to produce the
   final `remediationHint` string.

6. **Emit JSON first, then human summary.** The full JSON block is rendered before the human
   summary so that any downstream tooling parsing stdout can extract the JSON without seeing prose.

7. **Gate 3 disposition.** At Gate 3, the user reviews the report and dispositions each divergence
   as one of: **accepted** (acknowledged, no action needed), **investigate** (follow up later,
   does not block), or **blocker** (must resolve before re-running the monitor). Dispositions are
   stored only in conversation context — no file is written.

8. **Unknown divergence types.** If Phase 7 encounters a divergence `type` it does not recognize
   in the remediation table (e.g., from a future phase reference file), it MUST still include the
   divergence in the report. The severity is inherited as emitted by the phase. Phase 7 annotates
   `summary` with an `unknownTypeCount` field counting these occurrences.

### Re-run behavior

> **Re-run behavior.** Each Phase 7 run produces a fresh report from the current Phase 3–6 outputs. Prior-session dispositions (accepted / investigate / blocker) are NOT automatically re-applied. If a user accepted a divergence in one session, it will re-appear in the next session's report unless the underlying condition has been remediated. This is intentional — the report reflects current state, not historical human judgment. Users who need persistent dispositions must track them externally.

---

## Consumer guidance

Pointers for downstream tools and operators that consume the divergence report:

> **Placeholder warning.** Remediation hint strings contain `<placeholder>` tokens that Falcon substitutes at emit time. **Before executing any hint in a shell, verify that all `<…>` tokens have been substituted.** Unresolved placeholders will be interpreted by the shell as input redirection and fail. Consumers parsing hints programmatically must also treat unresolved placeholders as a substitution failure.

- **Match on `type`, not `description`.** The `type` field (e.g., `compute.vcpu-deficit`) is the
  stable contract across skill versions. The `description` field is human-prose and may change
  wording between releases. Downstream tools that branch on divergence type must use `type`.

- **Tolerate absent `evidence` fields.** The `evidence` object schema is probe-type-specific and
  phase-specific. Consumers MUST handle absent or null fields gracefully and fall back to generic
  key/value enumeration when rendering unknown evidence shapes.

- **Treat `remediationHint` as opaque.** Hint strings may contain `<placeholder>` markers that
  Falcon substitutes at emit time. Consumers that display hints should render them as plain strings.
  Do not attempt to parse or extract placeholder values from hint text.

- **Unknown divergence types must not be dropped.** If a consumer encounters a `type` it does not
  recognize — for example, because a future skill release adds new divergence types — it MUST still
  include the divergence in any downstream rendering or aggregation. The `severity` field is always
  populated and sufficient for classification. Check `summary.unknownTypeCount` (if present) to
  detect whether any unknown types were encountered during report generation.

- **`phases` key absence vs empty array.** A missing key in `phases` means the phase was skipped.
  An empty array (`[]`) means the phase ran and found no divergences. Consumers must distinguish
  these two states — a skipped phase is a data gap, an empty phase is a clean result.

- **`pairsWithCriticalDivergence` counts pairs, not criticals.** This field is a count of distinct
  workload pairs with at least one critical finding, not a count of total critical divergences.
  Use `summary.critical` for the total divergence count.

- **Disposition values are conversational only.** Gate 3 dispositions (accepted / investigate /
  blocker) live in conversation context only. No file is written. Consumers that need persistence
  must implement their own storage layer.
