# Infrastructure Parity Reference

## Purpose

This reference drives Phase 3 infrastructure parity checks. It is consumed after Gate 2 mapping is locked — only paired workloads are evaluated. Each rule below defines a formula, a severity classification, and the structured divergence shape Falcon emits when the rule is violated. Divergences feed Phase 7's consolidated report. Phase 3 does not block the workflow; it records divergences for human review.

---

## Compute shape

### vCPU parity

Formula: `target.vcpus / source.vcpus`. Must be ≥ 0.8.

- Below 0.8 → major
- Below 0.6 → critical

Resolve `target.vcpus` from `aws ec2 describe-instance-types`. `source.vcpus` from the canonical VMware inventory.

Example divergence (target 2 vCPU vs source 4 vCPU):
`{"type":"compute.vcpu-deficit","severity":"major","description":"Target EC2 has 2 vCPU vs source VM 4 vCPU (ratio 0.50, below 0.8 floor)","evidence":{"source_vcpus":4,"target_vcpus":2,"ratio":0.50,"floor":0.8}}`

---

### Memory parity (no telemetry)

Formula: `target.memoryMB / source.memoryMB`. Must be ≥ 0.8.

- Below 0.8 → major
- Below 0.6 → critical

Resolve `target.memoryMB` from `aws ec2 describe-instance-types`. `source.memoryMB` from canonical VMware inventory.

Example divergence (target 8192 MB vs source 16384 MB):
`{"type":"compute.memory-deficit","severity":"major","description":"Target EC2 has 8192 MB vs source VM 16384 MB (ratio 0.50, below 0.8 floor)","evidence":{"source_memoryMB":16384,"target_memoryMB":8192,"ratio":0.50,"floor":0.8}}`

---

### Memory parity (with telemetry)

Formula: `target.memoryMB ≥ 1.2 × source.telemetry.memoryP95MB`.

- Below threshold → major (not escalated to critical — observed usage already reflects a headroom buffer).

Applies only when `source.telemetry.memoryP95MB` is populated. Fall back to the no-telemetry rule when unavailable.

Example divergence (target 8192 MB, p95 7500 MB, required ≥ 9000 MB):
`{"type":"compute.memory-deficit","severity":"major","description":"Target EC2 memory 8192 MB below 1.2× source p95 usage 7500 MB (required ≥9000 MB)","evidence":{"source_memoryP95MB":7500,"target_memoryMB":8192,"required_memoryMB":9000}}`

---

### OS family parity

Rule: source Linux → target Linux; source Windows → target Windows. Any cross-OS pairing = critical.

Uses same extraction logic as Phase 1: `source.guestOs` containing `"windows"` → Windows; else Linux. `target.platform = "windows"` → Windows; absent or other → Linux.

Example divergence (source Ubuntu → target Windows Server):
`{"type":"compute.os-mismatch","severity":"critical","description":"Source VM guestOs 'ubuntu20_64Guest' (Linux) paired with Windows EC2 platform","evidence":{"source_os_family":"linux","target_os_family":"windows"}}`

---

### Architecture parity

Rule: `x86_64` / `arm64` must match between source and target. Mismatch = critical unless the user acknowledged it at Gate 2.

Source arch from `source.guestOs` (`"arm"`/`"aarch"` → arm64; else x86_64). Target arch from `ProcessorInfo.SupportedArchitectures` in `aws ec2 describe-instance-types`. When `pair.acknowledgedArchMismatch: true`, downgrade to informational.

Example divergence (source x86_64 → target arm64, not acknowledged):
`{"type":"compute.architecture-mismatch","severity":"critical","description":"Source VM is x86_64 but target EC2 instance type 'm7g.large' is arm64","evidence":{"source_arch":"x86_64","target_arch":"arm64","target_instanceType":"m7g.large"}}`

---

### Hyperthreading note

VMware reports vCPUs as logical CPUs (hyperthreads). AWS also reports vCPUs as hyperthreads. The vCPU parity formula uses the counts as reported on both sides with no HT adjustment. Do not halve either side before applying the formula.

---

## Disk / storage

### Total capacity parity

Formula: `deficit = 1 - (sum(target EBS volume sizeGB) / source_total_diskGB)`. Must be ≤ 0.10 (10%).

`source_total_diskGB` = sum of `source.disks[].sizeGB`. OS disk exclusion is allowed: if the user designates an OS disk, exclude it from both sides before computing the ratio. Target EBS volumes enumerated via `aws ec2 describe-volumes --filters Name=attachment.instance-id,Values=<instanceId>`.

Edge cases: if `source_total_diskGB == 0` (diskless boot-from-SAN or similar), skip this check and emit no divergence. The encryption parity rule also skips — a VM with no disks can have no encryption mismatch.

- Deficit > 10% → major
- Deficit > 30% → critical

Example divergence (source 400 GB, target 240 GB, deficit 40%):
`{"type":"storage.capacity-deficit","severity":"critical","description":"Target EBS total 240 GB vs source VMDK total 400 GB (deficit 40%, exceeds 30% threshold)","evidence":{"source_total_diskGB":400,"target_total_diskGB":240,"deficit_pct":40,"threshold_pct":30}}`

---

### Disk count

Differences in disk count between source and target are acceptable. A single source VMDK may be split across multiple EBS volumes (e.g., data migration tooling may repartition). This is not a divergence and produces no output.

---

### Encryption parity

Rule: if the source VM had VMware disk encryption enabled (detected via `source.disks[].encrypted: true` on any disk, or datastore-level encryption indicated by the datastore record), then ALL target EBS volumes must have `Encrypted: true`. Any unencrypted target EBS volume when source was encrypted = critical.

`source.disks[].encrypted` is set from Phase 2 discovery. Absent or null → assume unencrypted (no divergence).

Example divergence (one of two target volumes unencrypted):
`{"type":"storage.encryption-missing","severity":"critical","description":"Source VM disks were encrypted but target EBS volume 'vol-0abc123' has Encrypted=false","evidence":{"source_encrypted":true,"unencrypted_volumes":["vol-0abc123"]}}`

---

### IOPS class parity (telemetry required)

Rule: `source.telemetry.diskIopsP95 ≤ target_provisioned_iops`. Applies only when `source.telemetry.diskIopsP95` is populated.

Target IOPS by volume type: `gp3` → `Iops` attribute (baseline 3000); `io1`/`io2` → `Iops` attribute; `gp2` → `3 × sizeGB` (cap 16000); `st1`/`sc1` → treat as 0.

If p95 exceeds provisioned → major.

Example divergence (source p95 5000 IOPS, target gp3 3000 baseline):
`{"type":"storage.iops-underprovisioned","severity":"major","description":"Source VM disk p95 IOPS 5000 exceeds target EBS provisioned baseline 3000 (vol-0def456, gp3)","evidence":{"source_diskIopsP95":5000,"target_provisioned_iops":3000,"target_volume_id":"vol-0def456","target_volume_type":"gp3"}}`

---

## Network

### a. Security group rule parity

Rules are compared as sets of `(cidr, protocol, fromPort, toPort)` tuples. Source firewall rules come from VMware portgroup / NSX rules attached to `source.networks[].portGroup`, provided as pre-processed input. If unavailable, skip and log.

#### Inbound

Target SG `IpPermissions` must cover every `(sourceCIDR, protocol, fromPort, toPort)` tuple the source firewall allowed inbound. Missing → major per tuple.

A source rule is "covered" if there exists a target rule with the same (cidr, protocol) and a range that fully contains [fromPort, toPort]. If no target rule covers a source rule, emit `security.inbound.missing` with evidence containing the source rule's full tuple. Partial coverage (e.g., source 8000-9000 vs target 8000-8999) counts as missing — the uncovered sub-range is the `missing` tuple in evidence.

Algorithm: compute `source_inbound_tuples - target_inbound_tuples`; emit one divergence per missing tuple.

Example divergence (port 8080 from 10.0.0.0/8 missing on target):

```json
{
  "type": "security.inbound.missing",
  "severity": "major",
  "description": "Target SG has no inbound rule for (10.0.0.0/8, tcp, 8080, 8080) present on source firewall",
  "evidence": {"cidr": "10.0.0.0/8", "protocol": "tcp", "fromPort": 8080, "toPort": 8080, "target_sg_ids": ["sg-xxx"]}
}
```

#### Outbound

Target SG `IpPermissionsEgress` must cover every `(destCIDR, protocol, fromPort, toPort)` tuple the source firewall allowed outbound. Missing → major per tuple.

A source rule is "covered" if there exists a target rule with the same (cidr, protocol) and a range that fully contains [fromPort, toPort]. If no target rule covers a source rule, emit `security.outbound.missing` with evidence containing the source rule's full tuple. Partial coverage (e.g., source 8000-9000 vs target 8000-8999) counts as missing — the uncovered sub-range is the `missing` tuple in evidence.

Algorithm: compute `source_outbound_tuples - target_outbound_tuples`; emit one divergence per missing tuple.

Example divergence (outbound to 203.0.113.0/24 port 443 missing):

```json
{
  "type": "security.outbound.missing",
  "severity": "major",
  "description": "Target SG has no outbound rule for (203.0.113.0/24, tcp, 443, 443) present on source firewall",
  "evidence": {"cidr": "203.0.113.0/24", "protocol": "tcp", "fromPort": 443, "toPort": 443, "target_sg_ids": ["sg-xxx"]}
}
```

#### Extra open rules (target allows more than source)

Algorithm: compute `target_inbound_tuples - source_inbound_tuples`; emit one minor divergence per extra tuple.

Extra rules are informational only — potential security regression. They are flagged but never block migration gate advancement.

Example divergence:

```json
{
  "type": "security.inbound.extra-open",
  "severity": "minor",
  "description": "Target SG allows inbound (0.0.0.0/0, tcp, 22) not present in source firewall rules",
  "evidence": {"cidr": "0.0.0.0/0", "protocol": "tcp", "port": 22, "target_sg_ids": ["sg-xxx"]}
}
```

---

### b. Public/private exposure parity

Checks whether the target preserves the source's public or private posture. Evaluated per-pair.

**Source had public IP (source.publicIp non-null):** Target must satisfy at least one of: public IP on primary ENI, an EIP attached, or registered in an internet-facing ALB/NLB (`scheme=internet-facing`). None satisfied → `exposure.public-regression` (critical).

Example divergence:

```json
{
  "type": "exposure.public-regression",
  "severity": "critical",
  "description": "Source VM had public IP 203.0.113.10 but target EC2 i-0abc123 has no public IP, EIP, or internet-facing LB",
  "evidence": {"source_publicIp": "203.0.113.10", "target_id": "i-0abc123", "target_publicIp": null, "target_eip": null, "target_internet_facing_lb": null}
}
```

**Source was private (source.publicIp null, not fronted by public LB):** Target must have no public IP on primary ENI, no EIP, and not be registered in any internet-facing LB. Any violation → `exposure.private-regression` (critical).

Example divergence (target unexpectedly gets a public IP):

```json
{
  "type": "exposure.private-regression",
  "severity": "critical",
  "description": "Source VM was private (no public IP) but target EC2 i-0def456 has public IP 54.0.0.1",
  "evidence": {"source_publicIp": null, "target_id": "i-0def456", "target_publicIp": "54.0.0.1"}
}
```

---

### c. Subnet / VPC topology

**Subnet count:** Consolidation acceptable. No divergence.

**VPC membership:** All target resources for one VMware cluster should reside in one VPC or explicitly peered VPCs. Spread across unpeered VPCs → minor (architectural note only).

This check is cluster-scoped rather than pair-scoped. It fires at most once per VMware cluster whose AWS resource pairs span multiple VPCs that are not peered.

Example divergence:

```json
{
  "type": "network.cross-vpc-fragmentation",
  "severity": "minor",
  "description": "Workloads from VMware cluster 'prod-cluster-01' are split across unpeered VPCs vpc-aaa and vpc-bbb",
  "evidence": {"cluster": "prod-cluster-01", "vpcs": ["vpc-aaa", "vpc-bbb"]}
}
```

**NACLs:** Deferred to a future phase. Stateless, additive rules require more complex comparison than SGs.

---

## Tags

### Required-tag parity

User specifies a set of required tag keys. Each key absent from `target.tags` → major. Values may differ (e.g., `env=prod` vs `Environment=production`) — only key presence is enforced.

Example divergence (required tag "CostCenter" absent):

```json
{
  "type": "tags.required-missing",
  "severity": "major",
  "description": "Required tag 'CostCenter' is absent from target EC2 i-0abc123",
  "evidence": {"missing_key": "CostCenter", "target_id": "i-0abc123", "target_tags": {"Name": "web-01", "Environment": "prod"}}
}
```

---

### Source-tag propagation (non-required)

Tag keys in `source.tags[].category` absent from `target.tags` → minor. Advisory only; does not affect gate status. Applies only to tag keys not in the user-specified required-tags set.

Example divergence:

```json
{
  "type": "tags.propagation-missing",
  "severity": "minor",
  "description": "Source VM tag category 'app' has no corresponding key on target EC2 i-0abc123",
  "evidence": {"missing_key": "app", "source_tag_value": "web", "target_id": "i-0abc123"}
}
```

---

### Migration-tracking tags

Keys `migration-wave`, `migration-source-vm-id`, `migration-date` are informational only. Absence → minor, never major. v1 does not require them for any gate to pass.

Example divergence:

```json
{
  "type": "tags.migration-tracking-absent",
  "severity": "minor",
  "description": "Migration-tracking tag 'migration-source-vm-id' absent from target EC2 i-0abc123",
  "evidence": {"missing_key": "migration-source-vm-id", "target_id": "i-0abc123"}
}
```

---

## Public/private exposure summary table

Quick-reference for common architectural patterns and the divergence triggered when parity is violated.

| Source configuration | Expected target | Divergence if violated |
|----------------------|-----------------|------------------------|
| Public web tier (public IP + port 80/443 open) | Public ALB with HTTPS listener OR EC2 with EIP | `exposure.public-regression` — critical |
| Internal API (private IP only, internal LB) | Private subnet + internal ALB/NLB, no public IP | `exposure.private-regression` — critical |
| DB tier (private IP, port 3306 / 5432) | Private subnet RDS with SG restricting ingress to app-tier SG only | `exposure.private-regression` + `security.inbound.extra-open` if ingress is over-broad |
| Appliance VM (single NIC, managed access) | EC2 with equivalent SG set, public/private posture preserved from source | `exposure.*-regression` if posture changes |
| Bastion / jump host (public IP, port 22 restricted) | EC2 with EIP, SG allowing port 22 from restricted CIDR only | `exposure.public-regression` if no EIP; `security.inbound.extra-open` if 0.0.0.0/0 added |

---

## Divergence classification matrix

Summary of all divergence types produced by Phase 3, their default severity, and any escalation thresholds.

| Divergence type | Default severity | Escalation threshold |
|-----------------|------------------|---------------------|
| `compute.vcpu-deficit` | major | critical at ratio < 0.6 |
| `compute.memory-deficit` | major | critical at ratio < 0.6 (no-telemetry path only) |
| `compute.os-mismatch` | critical | — |
| `compute.architecture-mismatch` | critical | — (downgrade to informational if `acknowledgedArchMismatch: true`) |
| `storage.capacity-deficit` | major | critical at deficit > 30% |
| `storage.encryption-missing` | critical | — |
| `storage.iops-underprovisioned` | major | — |
| `security.inbound.missing` | major | — |
| `security.outbound.missing` | major | — |
| `security.inbound.extra-open` | minor | — |
| `exposure.public-regression` | critical | — |
| `exposure.private-regression` | critical | — |
| `tags.required-missing` | major | — |
| `tags.propagation-missing` | minor | — |
| `tags.migration-tracking-absent` | minor | — |
| `network.cross-vpc-fragmentation` | minor | — |
