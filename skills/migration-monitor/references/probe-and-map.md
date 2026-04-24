# Probe and Map Reference

## Purpose

This reference drives Phases 0–2 of the migration-monitor workflow. Phase 0 uses the capability probe recipes to determine which parity surfaces are available. Phase 1 uses the multi-signal scoring algorithm to produce `{vmware-workload → aws-resource}` pairings. Phase 2 uses the discovery command lists to enumerate comprehensive inventory on both the VMware and AWS sides. Falcon executes all commands natively; this file is pure knowledge.

---

## Phase 0: Capability Probe Recipes

Run every probe before presenting Gate 1. Each probe is independent; run all of them regardless of earlier results.

### Probe 1: AWS Reachability

```bash
aws sts get-caller-identity
```

**Success pattern:** JSON response containing `Account`, `Arn`, and `UserId` fields.

```json
{
    "UserId": "AIDAEXAMPLEID",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/deploy-svc"
}
```

**Failure interpretation:** `AuthFailure` or `InvalidClientTokenId` → credentials absent or expired; rotate keys or re-assume role. Network error (connection timeout, no route) → VPN or proxy not configured.

---

### Probe 2: vCenter Reachability

**Pre-condition:** Only run if both `GOVC_URL` AND `GOVC_USERNAME` are set in the environment. If either is missing, skip silently and log "vCenter probe skipped — GOVC_URL or GOVC_USERNAME not set."

```bash
govc about
```

**Success pattern:** JSON with `Version` and `FullName` populated.

```json
{
  "FullName": "VMware vCenter Server 8.0.2 build-12345678",
  "Version": "8.0.2"
}
```

**Failure interpretation:** Authentication error → `GOVC_PASSWORD` missing or wrong. `x509: certificate signed by unknown authority` → set `GOVC_INSECURE=1` or install the vCenter CA. Connection refused / timeout → `GOVC_URL` points to wrong host or port 443 is blocked.

---

### Probe 3: Static VMware Exports (RVTools Fallback)

```bash
find . -maxdepth 2 -type f \( -iname "RVTools*.xlsx" -o -iname "RVTools*.csv" \)
```

**Success pattern:** At least one file path printed to stdout.

**Failure interpretation:** No output → no RVTools export in the working directory or immediate subdirectories. Ask the user to place RVTools exports in the current directory or a direct subdirectory.

---

### Probe 4: Metric Source Environment Scan

Check the following environment variables. For pair-based sources, **both** vars of the pair must be present for the source to be considered available.

| Source | Required env vars |
|--------|-------------------|
| Prometheus | `PROMETHEUS_URL` |
| Datadog | `DATADOG_API_KEY` AND `DATADOG_APP_KEY` |
| New Relic | `NEW_RELIC_API_KEY` AND `NEW_RELIC_ACCOUNT` |
| vRealize Operations | `VROPS_URL` AND `VROPS_USERNAME` |

**Success pattern:** At least one source has all its required vars set.

**Failure interpretation:** No sources found → metrics parity (Phase 5) will be skipped. Log "Metrics probe: no metric source env vars detected — Phase 5 skipped."

---

### Probe 5: Endpoint Probing (curl)

```bash
curl --version
```

**Success pattern:** First line of output is `curl <version> ...`.

**Failure interpretation:** `command not found` → curl not installed; endpoint parity (Phase 4) will be skipped. Log "Endpoint probe: curl not found — Phase 4 skipped."

---

### Probe 6: TLS Certificate Checks (openssl)

```bash
openssl version
```

**Success pattern:** First line of output is `OpenSSL <version> ...`.

**Failure interpretation:** `command not found` → openssl not installed; TLS validation within Phase 4 will be skipped. Endpoint reachability checks (curl) may still run.

---

### Probe 7: Database Parity

Check for both source and target pairs simultaneously. The probe passes only if **all four** vars are set.

Required: `SOURCE_DB_URI` AND `SOURCE_DB_PASSWORD` AND `TARGET_DB_URI` AND `TARGET_DB_PASSWORD`.

**Success pattern:** All four variables are non-empty in the environment.

**Failure interpretation:** Any var missing → DB parity (Phase 6) will be skipped. Log which specific vars are absent so the user knows exactly what to supply.

---

### Available-Surfaces Assembly Logic

After all seven probes complete, Falcon assembles the available-surfaces set using these rules.

**Required surfaces (Gate 1 blocks if either is missing):**

1. AWS must be reachable (Probe 1 succeeded).
2. At least one VMware source must be available: either live vCenter (Probe 2 succeeded) OR at least one RVTools file found (Probe 3 succeeded).

If either required surface is missing, stop at Gate 1 and present the remediation steps for the failing probe. Do not proceed to Phase 1.

**Optional surfaces (skipped with logged reason if dependency missing):**

- Endpoint parity (Phase 4): requires Probe 5 (curl) to succeed.
- TLS validation within Phase 4: requires Probe 6 (openssl) to succeed.
- Metrics parity (Phase 5): requires Probe 4 to find ≥1 metric source.
- DB parity (Phase 6): requires Probe 7 to find all four DB vars.

**Gate 1 checklist format Falcon presents to the user:**

```
Surface Availability Checklist
═══════════════════════════════════════════════════════════════════
  [✓] AWS reachability       Account: 123456789012 (us-east-1)
  [✓] VMware source          Live vCenter 8.0.2 at vcenter.corp.example
  [✗] RVTools export         Not found (live vCenter available — OK)
  [✓] Metric sources         Datadog
  [✓] curl / endpoints       curl 8.6.0
  [✓] openssl / TLS          OpenSSL 3.3.0
  [✗] DB parity              TARGET_DB_URI not set — Phase 6 will be skipped

Phases enabled:   0, 1, 2, 3, 4, 5, 7
Phases skipped:   6 (missing TARGET_DB_URI / TARGET_DB_PASSWORD)
═══════════════════════════════════════════════════════════════════
Proceed to Phase 1? (yes / re-run after fixing env / abort)
```

---

## Phase 1: Multi-signal Workload Mapping

Produce `{vmware-workload → aws-resource}` pairings via additive, uncapped scoring across nine signals. Never rely on a single signal.

### Signal Table

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

### Signal Extraction Details

**Name normalization.** Before any name comparison, apply the following transformations to both sides: strip trailing domain components (remove everything after the first `.`), convert to lowercase, remove all `-`, `_`, and `.` characters. For example, `Web-01.corp.example` normalizes to `web01`. Both exact and fuzzy matching operate on normalized forms.

**IP matching.** Iterate every NIC in the VMware VM's network device list (Guest.Net[].IpAddress, flattened) against every ENI private IP in the AWS resource's network interfaces. A single overlapping address is sufficient for the signal to fire. For RDS, compare against the endpoint address resolved to IP; for ALB/NLB, compare against ENI private IPs in the load balancer's network interfaces.

**Hostname vs DNS matching.** Strip a trailing `.` from both sides and lowercase before comparing. The VMware guest hostname (Guest.HostName) is compared against Route 53 record targets (Value field in resource record sets) and ALB/NLB DNS names. Partial subdomain suffix matches (hostname is a prefix of the DNS name) do not count — the comparison is exact after normalization.

**OS family extraction.** From VMware: inspect `Config.GuestId` (e.g., `ubuntu20_64Guest`, `windows2019srv_64Guest`). If the string contains `"windows"` (case-insensitive), classify as Windows; otherwise classify as Linux/Other. From AWS EC2: inspect the `Platform` field — if it equals `"windows"` (case-insensitive), classify as Windows; if absent or any other value, classify as Linux. OS family match fires +10 if both sides agree; fires −20 if the sides are an explicit mismatch (one Windows, one Linux). Unknown on either side → signal does not fire (neither bonus nor penalty).

**Sizing ratio.** Compute `target_vcpu / source_vcpu` and `target_memory_mb / source_memory_mb`. Both ratios must fall within [0.5, 2.0] for the signal to fire. If either ratio is outside that range, the signal does not contribute. AWS instance type vCPU and memory can be resolved via `aws ec2 describe-instance-types` if not already in the inventory.

**Migration-tracking tag signal.** Check the AWS resource's tag map for keys `migration-source-vm-id` (value must match VMware VM MoRef ID) or `migration-source-name` (value must match normalized VM name). This is a bonus signal — its absence is not penalized. It fires +15 only when explicitly present and matching.

**Annotation/notes signal.** Perform substring match only. Check whether the VMware `Config.Annotation` string contains the AWS resource's ID or name (normalized). Also check whether the AWS resource's `Description` field (where applicable — e.g., EC2 instance description, security group description) contains the VMware VM's name (normalized). Either direction firing is sufficient for +5.

### Applicable-Resource-Type Rules

Not every VMware VM is a candidate for every AWS resource type. Filtering out inapplicable pairings before scoring prevents noise and false positives.

`applicable_types(vm)` returns the set of AWS resource types to consider for a given VM based on the VM's role, inferred from:
- Guest OS (Windows VMs are rarely RDS or EFS targets).
- Running services known from telemetry if available (e.g., a VM named `mysql-*` or with annotation "MySQL 8.0" may match an RDS instance).
- Sizing (a VM with <2 vCPU is unlikely to be an MSK broker).

Primary type mappings:
- General-purpose VM → EC2 instance (always in the candidate set).
- VM identified as a database host → RDS instance or ElastiCache cluster added to candidate set.
- VM identified as a load balancer or proxy → ALB/NLB added to candidate set.
- VM identified as a file server → EFS or FSx added to candidate set.
- MSK is considered only for VMs explicitly identified as Kafka brokers.

Do not pair a VM with resources of incompatible type. In particular, EC2 VM → S3 bucket yields 0 applicable signals and must be excluded from candidates before scoring begins.

### Scoring Thresholds

- **≥80:** Auto-pair. High confidence; no user intervention required for the pairing decision (though users may override at Gate 2).
- **40–79:** Ambiguous. Present top 3 candidates to the user with signal breakdowns for manual selection.
- **<40:** Unresolved. No plausible match found; present to user for manual pairing or declaration of out-of-scope.

Score is uncapped additive. Theoretical maximum depends on which signals fire (e.g., exact name + private IP + hostname + OS + sizing + tag = 40+30+25+10+10+15 = 130).

### Pseudo-code for Matching Loop

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
      continue                      # don't pair a VM with an S3 bucket
    if aws.id in aws_claimed:
      continue                      # already assigned to a high-confidence pair
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

ambiguous_claimed = {c.aws.id for (vm, top3) in ambiguous for c in top3}
unresolved_aws = [a for a in aws_inventory
                  if a.id not in aws_claimed
                  and a.id not in ambiguous_claimed]

return pairs, ambiguous, unresolved_vmware, unresolved_aws
```

### Tie-Breakers

When two or more candidates share the same top score, apply these tie-breakers in order until the tie is resolved:

1. **Same AWS region as the vCenter datacenter's primary region** (prefer the resource in the region that matches the migration target region set in `AWS_DEFAULT_REGION` or the region of the first discovered EC2 resource).
2. **Same OS family** (prefer the resource whose OS family matches the VMware guest, if not already captured by the OS-family signal).
3. **Earlier creation timestamp** (prefer the older AWS resource, on the theory that earlier-created resources are more likely to be deliberate migration targets rather than incidentally named resources).

If all three tie-breakers are exhausted and the candidates are still tied, demote to ambiguous and present to the user even if the score is ≥80.

### Gate 2 Interaction Recipe

After the matching loop completes, Falcon presents results in three groups. The goal is minimal user friction: auto-paired results require only a scan/override, while ambiguous and unresolved require explicit choices.

**Auto-paired group:** Render collapsed by default with a count and a toggle to expand for review. Example:

```
Auto-paired: 14 workloads — expand to review
  web-01          → i-0abc123 (EC2 t3.large)    [score: 95, signals: exact-name, private-ip, os-family, sizing]
  db-prod-01      → db-prod-01 (RDS postgres)   [score: 110, signals: exact-name, private-ip, hostname]
  ...
```

**Ambiguous group:** Render expanded with top 3 candidates per VM. The user must pick one or mark the VM as out-of-scope.

```
Ambiguous — pick a match or mark out-of-scope:

  app-server-02 (4 vCPU, 16 GB, Ubuntu, 10.0.1.22)
    [A] i-0def456  EC2 t3.xlarge   score=65   signals: fuzzy-name, private-ip
    [B] i-0ghi789  EC2 t3.xlarge   score=60   signals: fuzzy-name, os-family, sizing
    [C] i-0jkl012  EC2 m5.large    score=45   signals: private-ip
    [D] out-of-scope
  Choice for app-server-02: ___
```

**Unresolved VMware group:** Each VM with no candidate ≥40 is presented with a free-text prompt. Options: enter an AWS resource ID to pair manually, or declare out-of-scope.

```
Unresolved VMware VMs — provide an AWS resource ID or mark out-of-scope:

  vm-1999  (2 vCPU, 4 GB, Windows Server 2019, no IP match)
    Pair with AWS resource ID: ___ (or type "out-of-scope")
```

**Out-of-scope declarations.** During Gate 2 review, the user may declare any VMware
workload out-of-scope for monitoring (for example: archive VMs, decommissioned workloads,
VMs intentionally not migrated). Such VMs move from `unresolved_vmware` or `ambiguous`
into `skipped_vmware` in the final mapping output. They are excluded from Phases 3–7.

**Unresolved AWS group:** AWS resources not claimed by any pair or ambiguous candidate are presented for disposition. Options: pair with a VMware VM ID, or declare as "native AWS, not a migration target."

```
Unresolved AWS resources — pick a VMware VM or declare native:

  i-deadbeef  EC2 t2.micro  Name: "legacy-jump-host"
    Pair with VMware VM ID: ___ (or type "native-aws")
```

After the user completes all choices, Falcon emits the final mapping output (see Section 5) and advances to Phase 2.

---

## Phase 2: Discovery Command Lists

All discovery is read-only and comprehensive — no tag filters, no scope narrowing. The full account/region inventory is captured so the matching algorithm has complete signal data.

### VMware Live Discovery (govc)

Run these commands when live vCenter is available (Probe 2 succeeded). Execute in the order listed; later commands reference MoRef IDs collected in earlier steps.

**List all VM inventory paths:**

```bash
govc find / -type m
```

Produces one inventory path per VM (e.g., `/datacenter/vm/web-01`). Use these paths for per-VM detail queries.

**Per-VM detail (for each path from the listing above):**

```bash
govc vm.info -json -vm.ipath='<path>'
```

Extract these fields from the JSON response:

| Field path | Inventory key |
|------------|---------------|
| `VirtualMachines[0].Self.Value` | `id` (MoRef) |
| `VirtualMachines[0].Config.Uuid` | `uuid` (alternate ID) |
| `VirtualMachines[0].Config.Name` | `name` |
| `VirtualMachines[0].Config.GuestId` | `guestOs` |
| `VirtualMachines[0].Config.Hardware.NumCPU` | `vcpus` |
| `VirtualMachines[0].Config.Hardware.MemoryMB` | `memoryMB` |
| `VirtualMachines[0].Config.Hardware.Device[]` filtered for `VirtualDisk` type | `disks` (sizeGB, datastore, thin) |
| `VirtualMachines[0].Config.Hardware.Device[]` filtered for NIC types (MAC + backing) | `networks` (mac, portGroup) |
| `VirtualMachines[0].Config.Annotation` | `annotation` |
| `VirtualMachines[0].Guest.HostName` | `hostname` |
| `VirtualMachines[0].Guest.Net[].IpAddress` flattened | `ipAddresses` |
| `VirtualMachines[0].Runtime.PowerState` | `powerState` |

**Tags per VM (separate call using MoRef):**

```bash
govc tags.attached.ls -r VirtualMachine:<MoRef>
```

Produces a list of `category/value` tag strings. Add to the VM's `tags` array.

**Clusters:**

```bash
govc find / -type c
```

Then for each cluster path:

```bash
govc cluster.usage -json '<cluster-path>'
```

Extract cluster name, host count, total CPU, total memory.

**Hosts:**

```bash
govc find / -type h
```

Then for each host path:

```bash
govc host.info -host='<path>' -json
```

Extract host name, CPU model, total CPU, total memory, ESXi version.

**Datastores:**

```bash
govc datastore.info -json
```

Extract datastore name, type (VMFS, NFS, vSAN), capacity, free space.

**Networks:**

```bash
govc network.info
```

Extract portgroup names and VLAN IDs. Used to enrich VM network context.

---

### VMware RVTools Fallback

Use when Probe 3 found RVTools exports and live vCenter is unavailable. RVTools exports are Excel workbooks (`.xlsx`) or CSV files (`.csv`).

**Expected tabs in `.xlsx` exports:** `vInfo`, `vCPU`, `vMemory`, `vDisk`, `vNetwork`, `vHost`, `vDatastore`.

For CSV fallback (single-sheet export), treat the single sheet as `vInfo` and extract all available columns from it.

**Column extraction rules per tab:**

| Tab | Key columns to extract |
|-----|------------------------|
| `vInfo` | `VM` (name), `Powerstate`, `DNS Name` (hostname), `IP Address`, `OS according to the VMware Tools`, `OS according to the configuration file`, `Annotation`, `VM ID`, `Cluster`, `Host`, `Datacenter` |
| `vCPU` | `VM`, `CPUs`, `Cores per Socket` |
| `vMemory` | `VM`, `Size MiB` |
| `vDisk` | `VM`, `Disk`, `Capacity MiB`, `Thin provisioned`, `Datastore` |
| `vNetwork` | `VM`, `Network`, `MAC Address`, `IP Address` |
| `vHost` | `Host`, `Cluster`, `CPU Model`, `# CPU`, `# Cores`, `# Memory` |
| `vDatastore` | `Datastore`, `Type`, `Capacity MiB`, `Free MiB` |

Falcon reads `.xlsx` files via:

```bash
python3 -c "import openpyxl; wb = openpyxl.load_workbook('RVTools.xlsx', data_only=True); ..."
```

CSV files are read natively via Falcon's file-reading capabilities. After reading, assemble VMs into the canonical inventory shape (see Section 4d) using `vInfo` as the primary record and joining `vCPU`, `vMemory`, `vDisk`, `vNetwork` on the `VM` name column.

---

### AWS Discovery

Run all commands below. All are read-only. Run per-region for each region that contains migrated workloads. If `AWS_DEFAULT_REGION` is set, start there; supplement with additional regions if the user indicates a multi-region migration.

Append `--no-cli-pager` to all commands to prevent pagination blocking. For large accounts, optionally scope EC2 and subnet queries via `--filters Name=vpc-id,Values=<id>` if the env var `MIGRATION_MONITOR_VPC_ID` is set.

**Account identity:**

```bash
aws sts get-caller-identity
```

**EC2 compute:**

```bash
aws ec2 describe-instances --region <region>
```

Extract from `Reservations[].Instances[]`: `InstanceId`, `InstanceType`, `Platform`, `PrivateIpAddress`, `PublicIpAddress`, `NetworkInterfaces[].PrivateIpAddresses[].PrivateIpAddress`, `Tags`, `State.Name`, `Placement.AvailabilityZone`, `SubnetId`, `VpcId`.

**Security groups:**

```bash
aws ec2 describe-security-groups --region <region>
```

Extract: `GroupId`, `GroupName`, `Description`, `IpPermissions`, `IpPermissionsEgress`, `Tags`.

**Subnets:**

```bash
aws ec2 describe-subnets --region <region>
```

Extract: `SubnetId`, `VpcId`, `CidrBlock`, `AvailabilityZone`, `Tags`.

**VPCs:**

```bash
aws ec2 describe-vpcs --region <region>
```

Extract: `VpcId`, `CidrBlock`, `IsDefault`, `Tags`.

**Elastic IPs:**

```bash
aws ec2 describe-addresses --region <region>
```

Extract: `AllocationId`, `PublicIp`, `PrivateIpAddress`, `InstanceId`, `NetworkInterfaceId`, `Tags`.

**RDS instances:**

```bash
aws rds describe-db-instances --region <region>
```

Extract: `DBInstanceIdentifier`, `Engine`, `EngineVersion`, `DBInstanceClass`, `Endpoint.Address`, `Endpoint.Port`, `MasterUsername`, `MultiAZ`, `Tags`.

**RDS clusters (Aurora):**

```bash
aws rds describe-db-clusters --region <region>
```

Extract: `DBClusterIdentifier`, `Engine`, `Endpoint`, `ReaderEndpoint`, `DatabaseName`, `TagList`.

**Application / Network Load Balancers:**

```bash
aws elbv2 describe-load-balancers --region <region>
```

Extract: `LoadBalancerArn`, `LoadBalancerName`, `DNSName`, `Type`, `Scheme`, `VpcId`, `AvailabilityZones[].LoadBalancerAddresses[].IpAddress`, `Tags`.

**Target groups:**

```bash
aws elbv2 describe-target-groups --region <region>
```

Extract: `TargetGroupArn`, `TargetGroupName`, `Protocol`, `Port`, `VpcId`, `LoadBalancerArns`.

**Target health (per target group ARN):**

```bash
aws elbv2 describe-target-health --target-group-arn <arn>
```

Extract: `TargetHealthDescriptions[].Target.Id` (instance IDs), `TargetHealthDescriptions[].TargetHealth.State`.

**Classic ELBs:**

```bash
aws elb describe-load-balancers --region <region>
```

Extract: `LoadBalancerName`, `DNSName`, `Scheme`, `Instances[].InstanceId`, `ListenerDescriptions`.

**EFS:**

```bash
aws efs describe-file-systems --region <region>
```

Extract: `FileSystemId`, `CreationToken`, `Name` (from Tags), `SizeInBytes`, `Tags`.

**FSx:**

```bash
aws fsx describe-file-systems --region <region>
```

Extract: `FileSystemId`, `FileSystemType`, `DNSName`, `StorageCapacity`, `Tags`.

**ElastiCache:**

```bash
aws elasticache describe-cache-clusters --region <region>
```

Extract: `CacheClusterId`, `Engine`, `CacheNodeType`, `CacheNodes[].Endpoint.Address`, `Tags`.

**MSK (Kafka):**

```bash
aws kafka list-clusters-v2 --region <region>
```

Extract: `ClusterInfoList[].ClusterName`, `ClusterInfoList[].ClusterArn`, `ClusterInfoList[].State`.

**Route 53 hosted zones:**

```bash
aws route53 list-hosted-zones
```

Then for each zone ID:

```bash
aws route53 list-resource-record-sets --hosted-zone-id <zone-id>
```

Extract: `Name`, `Type`, `TTL`, `ResourceRecords[].Value`, `AliasTarget.DNSName`. DNS records feed the hostname-match signal and endpoint-parity Phase 4.

**Pagination note:** All list/describe commands return paginated results. The `--no-cli-pager` flag prevents interactive blocking. For very large accounts (>500 instances), consider adding `--max-items 1000` and iterating with `--starting-token`; Falcon handles this natively via its CLI output parsing.

---

### Canonical Inventory JSON Shape

After executing all discovery commands, Falcon assembles results into this in-memory structure. This is the input to the Phase 1 matching loop and the baseline stored for Phase 3 comparison.

```json
{
  "capturedAt": "2026-04-24T15:00:00Z",
  "vmware": {
    "source": "govc",
    "vcenter": { "url": "https://vcenter.corp.example", "version": "8.0.2" },
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

When the VMware source is RVTools (not live govc), set `"source": "rvtools"` and omit the `"vcenter"` key. Fields that RVTools does not provide (e.g., `publicIp` for VMs) are set to `null`.

---

## Mapping output shape

This is the structure Falcon produces at the end of Phase 1 (after Gate 2 user confirmation). It is the contract for all subsequent phases (3–7).

```json
{
  "pairs": [
    {
      "vmware_id": "vm-1001",
      "aws_id": "i-0abc123",
      "confidence": 95,
      "signals_hit": ["exact-name", "private-ip", "os-family", "sizing"]
    }
  ],
  "unresolved_vmware": ["vm-1999"],
  "unresolved_aws": ["i-deadbeef"],
  "skipped_vmware": ["vm-archive-01"]
}
```

**Semantics of each list:**

- **`pairs`:** High-confidence auto-pairs plus all user-confirmed ambiguous matches from Gate 2. Each entry records the VMware VM ID, the AWS resource ID, the final confidence score, and the list of signals that fired. This list drives all Phase 3–7 comparisons — only paired workloads are analyzed.

- **`unresolved_vmware`:** VMware VMs for which the user could not identify a matching AWS resource at Gate 2, or explicitly deferred. These are flagged in the divergence report (Phase 7) as "no AWS counterpart found — investigate." They are not analyzed in Phases 3–6.

- **`unresolved_aws`:** AWS resources that were not claimed by any pair and that the user declared as "native AWS, not a migration target" at Gate 2. These are excluded from parity analysis. They are listed in the divergence report for completeness but generate no divergence entries.

- **`skipped_vmware`:** VMware VMs the user explicitly declared out-of-scope for monitoring (e.g., archived VMs, template VMs, infrastructure VMs not subject to migration). These are excluded from all analysis and appear in the report's `skippedPhases` context for auditability.
