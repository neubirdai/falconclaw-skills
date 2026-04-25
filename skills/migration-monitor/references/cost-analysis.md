# Cost Analysis (AWS-side)

## Purpose

This reference drives the cost-analysis pass that runs as part of Phase 7 (Divergence Report). It surfaces high-impact cost patterns on the deployed AWS counterparts — patterns that frequently cost teams thousands of dollars per month and are easy to miss without explicit attention.

The pass is not a full FinOps audit. It targets ~10 well-known gotchas where the cost signal is detectable from the same read-only AWS API surface used elsewhere in the skill, and where the remediation is concrete enough to recommend.

**Falcon is read-only.** Cost flags are surfaced as recommendations; Falcon never modifies a resource to "fix" a cost issue.

## When this runs

After Phase 2 baseline capture has populated the AWS inventory, and before Phase 7 renders the report. Cost analysis consumes the same AWS data Phase 3–6 use, plus a small set of additional API calls listed below in **Additional Discovery Required**.

## Output

A `cost_flags` array attached to the Phase 7 report (see `references/divergence-report.md` schema). Each flag has the same envelope as a divergence (id, severity, type, resource, description, evidence, remediationHint) plus an `estimated_monthly_cost` field with a rough USD figure for the magnitude of the issue.

**Cost figures are illustrative.** Falcon MUST web-search current AWS pricing for the user's region before emitting any specific dollar amount — pricing changes, regional differences, and committed-use discounts (Savings Plans, Reserved Instances, EDP) all distort the headline rate. The estimates in this reference are typical retail prices and should be treated as upper bounds.

---

## Additional Discovery Required

The following AWS API calls are NOT in the standard Phase 2 baseline capture (which targets parity, not cost). Cost analysis adds these:

```bash
# EKS clusters and their versions
aws eks list-clusters --region "$AWS_REGION" --no-cli-pager
aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --no-cli-pager

# EBS snapshots owned by this account
aws ec2 describe-snapshots --owner-ids self --region "$AWS_REGION" --no-cli-pager

# Elastic IPs (already in Phase 2 if `aws ec2 describe-addresses` was called — confirm)
aws ec2 describe-addresses --region "$AWS_REGION" --no-cli-pager

# CloudWatch log groups + retention policies
aws logs describe-log-groups --region "$AWS_REGION" --no-cli-pager

# NAT Gateways (and their VPC associations for traffic context)
aws ec2 describe-nat-gateways --region "$AWS_REGION" --no-cli-pager

# RDS multi-AZ status (already in Phase 2 if `describe-db-instances` was called)
aws rds describe-db-instances --region "$AWS_REGION" --no-cli-pager

# CloudWatch metric data for NAT throughput estimation (BytesOutToDestination)
aws cloudwatch get-metric-statistics --namespace AWS/NATGateway \
  --metric-name BytesOutToDestination --dimensions Name=NatGatewayId,Value="$NAT_ID" \
  --start-time "$(date -u -d '30 days ago' +%FT%TZ 2>/dev/null || date -u -v-30d +%FT%TZ)" \
  --end-time "$(date -u +%FT%TZ)" --period 86400 --statistics Sum --region "$AWS_REGION"
```

All commands are read-only. The NAT Gateway metric query uses the same portable date arithmetic as `references/metrics-parity.md` to handle macOS/GNU `date` differences.

---

## Cost Flag Rules

Each flag has: detection criteria, severity, monthly-cost estimation formula, and remediation hint.

### CF-1: `cost.eks.extended-support-active` — CRITICAL

**Detection:** EKS cluster version is past the standard support end date (typically 14 months after the version's GA release). AWS publishes the current standard-support window at https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html.

**Web-search anchor:** Before flagging, Falcon MUST search for "AWS EKS Kubernetes version standard support end date <version>" — the support timeline shifts as AWS announces new releases.

**Cost estimation:**

```
extended_support_fee = $0.10 / vCPU / hour  (typical 2026 rate; verify by web search)
total_cluster_vcpus  = sum(node.vcpus for node in cluster.nodegroups)
monthly_cost_added   = total_cluster_vcpus × $0.10 × 24 × 30
```

For the retail-store demo example (60 vCPUs across all nodes): 60 × 0.10 × 720 = **$4,320/month** in extended support fees alone, on top of the standard $73/month cluster fee.

**Remediation hint:**

> EKS cluster `<cluster_name>` is on version `<version>`, which entered extended support on `<date>`. Extended support fee adds $0.10/vCPU/hour on top of the cluster fee. At your current cluster size (`<total_vcpus>` vCPUs), this is approximately `$<monthly_cost>`/month. Upgrade to a currently-supported version: `aws eks update-cluster-version --name <cluster_name> --version <target_version>`. Plan a blue/green rollout for stateful workloads. Web-search "AWS EKS Kubernetes version standard support" for the current support timeline.

### CF-2: `cost.eks.extended-support-imminent` — MAJOR

**Detection:** EKS cluster version exits standard support within 6 months. Same web-search anchor as CF-1.

**Cost estimation:** No active fee yet, but post-deadline rate is the same as CF-1. Reported as "preventable monthly cost if not upgraded by `<date>`."

**Remediation hint:**

> EKS cluster `<cluster_name>` is on version `<version>`. Standard support ends `<date>` (in `<N>` days). After that, extended support fees of `$<monthly_cost>`/month at current cluster size will apply automatically. Plan upgrade now: aim to be on `<recommended_version>` before the cutoff.

### CF-3: `cost.ec2.previous-generation-instance-types` — MAJOR

**Detection:** EC2 instances using previous-generation families. As of 2026, families to flag: `t1`, `t2`, `m1`, `m2`, `m3`, `m4`, `m5` (still common), `c1`, `c3`, `c4`, `c5`, `r3`, `r4`, `r5`, `i2`, `i3`. Newer alternatives: `t3`/`t4g`/`t3a`, `m6i`/`m6a`/`m7i`/`m7a`, `c6i`/`c7i`/`c7a`, `r6i`/`r7i`, `i4i`/`im4gn`.

**Cost estimation:**

```
old_rate_per_hour = (per-region rate for current instance type)
new_rate_per_hour = (per-region rate for recommended same-shape instance type)
savings_per_month = (old_rate - new_rate) × 720
```

Web-search is required for current pricing. As of 2026, m5.large in us-east-1 is ~$0.096/hr; m7i.large is ~$0.0856/hr (~11% cheaper for typically better performance).

**Remediation hint:**

> Instance `<instance_id>` is using `<old_type>` (previous-generation). Equivalent newer family `<recommended_type>` is approximately `<savings_pct>%` cheaper at similar or better performance. Resize: `aws ec2 modify-instance-attribute --instance-id <instance_id> --instance-type Value=<recommended_type>`. Requires brief stop/start. Web-search current pricing in `<region>` to confirm savings before applying.

### CF-4: `cost.ebs.gp2-volumes` — MINOR

**Detection:** EBS volumes with `VolumeType: gp2`. Modern alternative is `gp3`, which is roughly 20% cheaper per GB at the same baseline IOPS, and lets you tune IOPS/throughput independently.

**Cost estimation:**

```
gp2_rate = $0.10 / GB-month  (typical us-east-1)
gp3_rate = $0.08 / GB-month
savings  = sum(volume.size_gb) × ($0.10 - $0.08)
```

A 500 GB gp2 → gp3 change saves ~$10/month per volume. Across many volumes this adds up.

**Remediation hint:**

> Volume `<volume_id>` is `gp2` (`<size_gb>` GB). Migrate to `gp3` for ~20% storage cost reduction at equivalent baseline performance: `aws ec2 modify-volume --volume-id <volume_id> --volume-type gp3`. The change is online (no detach required); IOPS/throughput stay at default unless you tune them. Estimated savings: `$<monthly_savings>`/month for this volume.

### CF-5: `cost.eip.unattached` — MAJOR

**Detection:** Elastic IPs where `AssociationId` is null AND `InstanceId` is null.

**Cost estimation:**

```
unattached_eip_rate = $0.005 / hour (typical 2026)
monthly_cost_per    = $0.005 × 720 = ~$3.60
total_monthly_cost  = count(unattached_eips) × $3.60
```

Cheap individually, expensive at scale. A team with 50 forgotten EIPs is paying $180/month for nothing.

**Remediation hint:**

> Elastic IP `<allocation_id>` (`<public_ip>`) is not attached to any resource — AWS bills for unattached EIPs at $0.005/hour ($3.60/month). If unused, release: `aws ec2 release-address --allocation-id <allocation_id>`. If reserved for a future workload, attach to a running instance or NLB to halt the charge.

### CF-6: `cost.ebs.unattached-volumes` — MAJOR

**Detection:** EBS volumes where `State: available` (not attached to any instance).

**Cost estimation:**

```
gp3_rate = $0.08 / GB-month
total_orphan_monthly = sum(volume.size_gb for volume in available) × $0.08
```

100 GB of orphaned volumes = $8/month. A few terabytes of orphaned snapshots from prior auto-scaling events can quietly cost hundreds.

**Remediation hint:**

> Volume `<volume_id>` (`<size_gb>` GB, type `<volume_type>`) is in `available` state — billable but unattached. If safe to delete: snapshot then `aws ec2 delete-volume --volume-id <volume_id>`. Verify no application owner needs the data first.

### CF-7: `cost.ebs.old-snapshots` — MINOR

**Detection:** EBS snapshots with `StartTime` older than 90 days AND not referenced by any active AMI (`aws ec2 describe-images --owners self --filters Name=block-device-mapping.snapshot-id,Values=<id>` returns empty).

**Cost estimation:**

```
snapshot_rate     = $0.05 / GB-month  (Standard tier)
total_old_storage = sum(snapshot.size_gb for snapshot in stale)
monthly_cost      = total_old_storage × $0.05
```

Snapshot storage is the surprise: years of automated daily snapshots that nobody cleans up. A 200 GB volume snapshotted nightly for 2 years = 73 TB cumulatively — though incremental dedup brings effective billed storage down significantly. Falcon should report the API-reported `VolumeSize` (not actual stored bytes) for transparency, and note that incremental savings make actuals lower.

**Remediation hint:**

> `<count>` snapshots older than 90 days totaling `<volume_gb>` GB of provisioned size. If part of a backup policy with explicit retention, ignore. Otherwise consider lifecycle automation: AWS Backup or Data Lifecycle Manager (DLM) with a retention policy. Manual cleanup: `aws ec2 delete-snapshot --snapshot-id <snapshot_id>`. Note: actual billed bytes are typically much less than `VolumeSize × count` due to incremental dedup; verify via Cost Explorer before bulk-delete.

### CF-8: `cost.cloudwatch-logs.no-retention` — MAJOR

**Detection:** CloudWatch Log Groups where `retentionInDays` is null (interpreted as "never expire").

**Cost estimation:**

```
ingestion_rate = $0.50 / GB ingested  (us-east-1 standard)
storage_rate   = $0.03 / GB-month
```

Log groups ingest at $0.50/GB and store at $0.03/GB-month with no upper bound when retention is unlimited. A noisy app logging 100 GB/month with no retention adds $3/month forever — across many log groups, hundreds of dollars.

**Remediation hint:**

> Log group `<log_group_name>` has no retention policy (logs kept forever). Set retention appropriate for the use case (often 30 or 90 days for application logs, longer for audit/compliance): `aws logs put-retention-policy --log-group-name <log_group_name> --retention-in-days 30`. Coordinate with audit/compliance owners before reducing retention on logs subject to data retention requirements.

### CF-9: `cost.nat-gateway.high-throughput` — INFORMATIONAL

**Detection:** NAT Gateway `BytesOutToDestination` summed over the last 30 days exceeds 1 TB.

**Cost estimation:**

```
nat_data_rate     = $0.045 / GB processed
monthly_throughput_gb = (BytesOutToDestination / 1024^3) extrapolated to 30 days
monthly_processing_fee = monthly_throughput_gb × $0.045
```

A workload pushing 5 TB/month through NAT pays $225/month in data processing alone, on top of the $0.045/hour gateway fee (~$32.40/month per NAT).

**Remediation hint:**

> NAT Gateway `<nat_id>` processed `<gb>` GB in the last 30 days (~$<estimated_processing_fee>/month at $0.045/GB processed). Investigate top destinations (VPC Flow Logs); consider VPC Endpoints for AWS service traffic (S3, DynamoDB, ECR, SSM) to bypass NAT entirely: `aws ec2 create-vpc-endpoint --vpc-id <vpc_id> --service-name com.amazonaws.<region>.s3 --vpc-endpoint-type Gateway`. Egress to S3 in-region via Gateway Endpoint is free.

### CF-10: `cost.rds.multi-az-on-non-prod` — MAJOR

**Detection:** RDS instances with `MultiAZ: true` AND tags suggesting non-production (case-insensitive match on `Environment`/`env`/`Tier` tag values containing `dev`, `staging`, `qa`, `test`, `non-prod`, `nonprod`).

**Cost estimation:**

```
multi_az_premium = ~2× single-AZ rate (typical doubling for the standby)
monthly_savings = (current_multi_az_monthly_cost) / 2
```

A db.r6i.large at $0.252/hr Multi-AZ vs $0.126/hr single-AZ saves $90/month per non-prod instance.

**Remediation hint:**

> RDS instance `<db_id>` is Multi-AZ with environment tag `<env_tag>` suggesting non-production. Multi-AZ doubles the cost; for non-prod, single-AZ is typically sufficient. Convert: `aws rds modify-db-instance --db-instance-identifier <db_id> --no-multi-az --apply-immediately`. Confirm with the application owner before applying — disabling Multi-AZ briefly interrupts the standby promotion option.

---

## Severity calibration

The severity of each cost flag is calibrated against typical monthly impact, NOT functional severity (cost flags are not break-anything events):

| Severity | Typical impact | Examples |
|----------|---------------|----------|
| Critical | Active fees > $1,000/month per occurrence OR rapid growth pattern | EKS extended support active |
| Major | Active fees $100–$1,000/month OR straightforward remediation with significant savings | EKS extended-support imminent, EC2 previous-generation, unattached EIPs/volumes, RDS Multi-AZ on non-prod, CW Logs no-retention |
| Minor | Active fees < $100/month OR optimization opportunity | EBS gp2 → gp3, old snapshots |
| Informational | Surfaces a context for the user to investigate; not actionable without more data | NAT Gateway high throughput |

Note that critical here is much more permissive than for divergences. A cost flag is never a *blocker* — at Gate 3, the user disposes cost flags the same way as divergences (accepted / investigate / remediate-later). Cost flags don't gate migration sign-off.

---

## Output JSON shape

Each cost flag follows this shape (added to the Phase 7 report's top-level `cost_flags` array):

```json
{
  "id": "cost-001",
  "type": "cost.eks.extended-support-active",
  "severity": "critical",
  "resource": {
    "type": "eks_cluster",
    "id": "retail-store-prod",
    "region": "us-east-1"
  },
  "description": "EKS cluster `retail-store-prod` is on version 1.28, which entered extended support on 2024-03-23. Extended support fee adds $0.10/vCPU/hour.",
  "estimated_monthly_cost": "$4320",
  "evidence": {
    "version": "1.28",
    "vcpus_in_cluster": 60,
    "extended_support_start": "2024-03-23",
    "rate_per_vcpu_hr": 0.10,
    "monthly_hours": 720
  },
  "remediationHint": "Upgrade to a supported version: `aws eks update-cluster-version --name retail-store-prod --version 1.31`. Plan blue/green rollout for stateful workloads."
}
```

The `estimated_monthly_cost` field is a string with a `$` prefix and no decimals — always treated as approximate. Falcon MUST emit a top-level disclaimer with the report:

> Cost estimates use typical 2026 retail rates and reflect approximate magnitude only. Reserved Instances, Savings Plans, EDP discounts, Spot pricing, and regional rate differences will all alter the actual figures. Validate against AWS Cost Explorer before acting on any specific number.

---

## User-supplied cost concerns (extension point)

Beyond the 10 built-in flags, users can specify additional cost concerns at session start via `--cost-concerns <path>` (a JSON file describing custom rules), or by stating them at Gate 1. Falcon evaluates these alongside the built-in rules and emits divergences with `type: cost.user-supplied` and the user's classification.

This handles cases not covered in v1: SageMaker endpoints left running, Redshift clusters paused/unused, S3 storage class mismatches, etc. v1.1 may grow the built-in list based on which user-supplied rules surface most often.

---

## Cost flag classification matrix

| Flag type | Severity | Typical impact (USD/month) | Detection complexity |
|-----------|----------|----------------------------|---------------------|
| `cost.eks.extended-support-active` | critical | $1,000+ | Single API call + version comparison |
| `cost.eks.extended-support-imminent` | major | $0 (preventable) | Single API call + date math |
| `cost.ec2.previous-generation-instance-types` | major | $10–$100 per instance | Type enumeration + price lookup |
| `cost.ebs.gp2-volumes` | minor | $5–$20 per volume | Volume type filter |
| `cost.eip.unattached` | major | $3.60 per EIP | Association check |
| `cost.ebs.unattached-volumes` | major | $5–$50 per volume | State filter |
| `cost.ebs.old-snapshots` | minor | Highly variable | Snapshot age + AMI cross-reference |
| `cost.cloudwatch-logs.no-retention` | major | Highly variable, grows | Retention policy null check |
| `cost.nat-gateway.high-throughput` | informational | $50–$1,000+ | CloudWatch metric query |
| `cost.rds.multi-az-on-non-prod` | major | $50–$200 per instance | MultiAZ flag + tag heuristic |
| `cost.user-supplied` | (user-defined) | (user-defined) | (user-defined) |
