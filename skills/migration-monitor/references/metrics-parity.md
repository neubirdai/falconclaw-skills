# Metrics Parity Reference

## Purpose

This reference drives Phase 5 metrics parity checks. Falcon uses it to:
1. Detect available metric sources from environment variables — zero user prompts for metric names.
2. Enumerate available metrics from each source via its catalog API.
3. Map source-side metric names to CloudWatch equivalents using the shipped table below.
4. Fetch time-series data on both sides and compute p50, p95, p99, and mean.
5. Compare statistics and emit structured `metrics.*` divergences where deviations exceed thresholds.

No user interaction is required for metric discovery. Falcon operates fully automatically: it reads env vars, calls catalog APIs, applies the mapping table, queries both sides, and classifies divergences without prompting.

---

## Metric-source detection

Falcon inspects environment variables at Phase 0 to determine which metric sources are available. Sources with incomplete credentials are logged as unavailable and skipped.

| Env vars required | Source type |
|---|---|
| `PROMETHEUS_URL` | Prometheus |
| `DATADOG_API_KEY` AND `DATADOG_APP_KEY` | Datadog |
| `NEW_RELIC_API_KEY` AND `NEW_RELIC_ACCOUNT` | New Relic |
| `VROPS_URL` AND `VROPS_USERNAME` AND `VROPS_PASSWORD` | vROps (VMware Aria Operations) |

**Partial credential rules:**

- Datadog: both `DATADOG_API_KEY` and `DATADOG_APP_KEY` must be set. If either is missing, Datadog is unavailable.
- New Relic: both `NEW_RELIC_API_KEY` and `NEW_RELIC_ACCOUNT` must be set. If either is missing, New Relic is unavailable.
- vROps: all three of `VROPS_URL`, `VROPS_USERNAME`, and `VROPS_PASSWORD` must be set. If any is missing, vROps is unavailable.

When a source is unavailable due to partial credentials, Falcon logs the reason at Phase 0 as `metrics.source-unavailable` (minor severity) and continues.

**CloudWatch is always the target side.** It is implicit from the AWS credentials already required by earlier phases — no additional env var check is needed.

---

## Per-source catalog enumeration

Falcon emits the following commands to enumerate available metrics from each source. All curl commands use `--max-time 30` to prevent hangs. No command performs a mutation — catalog enumeration is strictly read-only.

### Prometheus

**Authentication:** No authentication header required when `PROMETHEUS_URL` is an internal endpoint. If basic auth is needed, add `-u "${PROMETHEUS_USER}:${PROMETHEUS_PASS}"`.

**Command:**
```bash
curl -sS -G "${PROMETHEUS_URL}/api/v1/label/__name__/values" \
  --max-time 30 \
  | jq -r '.data[]'
```

**Expected output:** Newline-separated metric names, e.g.:
```
node_cpu_seconds_total
node_memory_MemAvailable_bytes
node_network_receive_bytes_total
http_request_duration_seconds
```

**Pagination:** None. The label values endpoint returns the full list in a single response.

---

### Datadog

**Authentication:** DD-API-KEY and DD-APPLICATION-KEY headers.

**Command:**
```bash
curl -sS \
  -H "DD-API-KEY: ${DATADOG_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DATADOG_APP_KEY}" \
  --max-time 30 \
  "https://api.datadoghq.com/api/v1/metrics?from=$(date -u -d '1 day ago' +%s)" \
  | jq -r '.metrics[]'
```

**Expected output:** Newline-separated metric names, e.g.:
```
system.cpu.user
system.mem.pct_usable
system.disk.in_use
system.net.bytes_rcvd
```

**Regional variant:** If `DATADOG_SITE` is set (e.g., `eu`, `us3`, `us5`, `ap1`), use the corresponding regional base URL. Examples:
- `DATADOG_SITE=eu` → `api.datadoghq.eu`
- `DATADOG_SITE=us3` → `us3.datadoghq.com`
- Default (unset or `us1`) → `api.datadoghq.com`

Falcon substitutes the base URL based on `DATADOG_SITE` before issuing the catalog request.

**Pagination:** The `/api/v1/metrics` endpoint returns all active metrics from the last 24 hours in a single response. No cursor-based pagination.

---

### New Relic

**Authentication:** `Api-Key` header using `NEW_RELIC_API_KEY`.

**Command:**
```bash
curl -sS -X POST \
  -H "Api-Key: ${NEW_RELIC_API_KEY}" \
  -H "Content-Type: application/json" \
  --max-time 30 \
  "https://api.newrelic.com/graphql" \
  -d "{\"query\":\"{ actor { account(id: ${NEW_RELIC_ACCOUNT}) { nrql(query: \\\"SHOW METRICS SINCE 1 DAY AGO\\\") { results } } } }\"}" \
  | jq -r '.data.actor.account.nrql.results[].metric'
```

**Important note:** NRQL `SHOW METRICS` is read-only metadata despite using the POST HTTP verb. The New Relic GraphQL API always uses POST for queries. This is a metadata query, not a mutation — it is acceptable for a read-only skill. No data is written or modified.

**Expected output:** Newline-separated metric names, e.g.:
```
system.cpuPercent
system.memoryUsedPercent
system.diskUsedPercent
```

**Pagination:** The NRQL response returns all metrics matching the time window in a single payload. For accounts with extremely large metric catalogs (>10,000 metrics), consider adding `LIMIT MAX` to the NRQL query. Falcon appends `LIMIT MAX` automatically.

---

### vROps (VMware Aria Operations)

**Authentication:** HTTP Basic auth using `VROPS_USERNAME` and `VROPS_PASSWORD`. The `-k` flag is required to accept self-signed TLS certificates, which are standard in on-premises vROps deployments. This is intentional for internal vROps instances and is expected behavior.

**Step 1 — enumerate resource IDs.** vROps identifies VMs by its own UUID, not the vCenter MoRef. Falcon must first collect resource identifiers:
```bash
curl -sS -k -u "${VROPS_USERNAME}:${VROPS_PASSWORD}" \
  --max-time 30 \
  -H "Accept: application/json" \
  "${VROPS_URL}/suite-api/api/resources?resourceKind=VirtualMachine&pageSize=1000" \
  | jq -r '.resourceList[].identifier'
```

**Step 2 — per-resource stat keys.** For each resource ID returned above, enumerate available stat keys:
```bash
curl -sS -k -u "${VROPS_USERNAME}:${VROPS_PASSWORD}" \
  --max-time 30 \
  -H "Accept: application/json" \
  "${VROPS_URL}/suite-api/api/resources/${RESOURCE_ID}/statkeys" \
  | jq -r '.stat[].key'
```

**Expected output (stat keys):**
```
cpu|usage_average
cpu|ready_summation
mem|usage_average
mem|active_average
disk|usage_average
net|throughput_usage_average
```

**Pagination:** The resources endpoint supports `pageSize` and `page` query params. Default page size is 100; Falcon uses 1000. For environments with more than 1000 VMs, Falcon iterates pages until `resourceList` is empty.

---

### CloudWatch (target side)

**Authentication:** Implicit from the AWS credentials and role already configured for earlier phases.

**Command:**
```bash
aws cloudwatch list-metrics \
  --namespace "AWS/EC2" \
  --dimensions "Name=InstanceId,Value=${INSTANCE_ID}" \
  --region "${AWS_REGION}" \
  --no-cli-pager
```

Adjust `--namespace` and `--dimensions` per target resource type:

| Resource type | Namespace | Key dimension |
|---|---|---|
| EC2 instance | `AWS/EC2` | `InstanceId` |
| EBS volume | `AWS/EBS` | `VolumeId` |
| RDS instance | `AWS/RDS` | `DBInstanceIdentifier` |
| Application Load Balancer | `AWS/ApplicationELB` | `LoadBalancer` |
| CloudWatch Agent (host-level) | `CWAgent` | `InstanceId` |

---

## Metric-name mapping table

This table is the centerpiece of Phase 5. Falcon matches source-side metric names from catalog enumeration against the "Source metric" column and uses the "CloudWatch equivalent" and "CloudWatch namespace" columns to query the target side. Matching is exact for structured names (e.g., vROps, Datadog, New Relic) and regex/substring for Prometheus expression patterns.

| Source metric | CloudWatch equivalent | CloudWatch namespace |
|---|---|---|
| vROps `cpu\|usage_average` | `CPUUtilization` | `AWS/EC2` |
| vROps `cpu\|ready_summation` | `CPUCreditBalance` (burstable only) | `AWS/EC2` |
| vROps `mem\|usage_average` | `mem_used_percent` | `CWAgent` |
| vROps `mem\|active_average` | `mem_used_percent` | `CWAgent` |
| vROps `mem\|swapped_average` | `swap_used_percent` | `CWAgent` |
| vROps `disk\|usage_average` | `disk_used_percent` | `CWAgent` |
| vROps `disk\|commandsAveraged_average` | `DiskReadOps` + `DiskWriteOps` (sum) | `AWS/EC2` |
| vROps `disk\|totalLatency_average` | `VolumeTotalReadTime` + `VolumeTotalWriteTime` (avg) | `AWS/EBS` |
| vROps `net\|throughput_usage_average` | `NetworkIn` + `NetworkOut` (sum, bps) | `AWS/EC2` |
| vROps `net\|packetsRx_summation` | `NetworkPacketsIn` | `AWS/EC2` |
| vROps `net\|packetsTx_summation` | `NetworkPacketsOut` | `AWS/EC2` |
| vROps `summary\|runtime.connectionState` | `StatusCheckFailed` (inverted) | `AWS/EC2` |
| Prom `rate(node_cpu_seconds_total{mode!="idle"}[5m])` | `CPUUtilization` | `AWS/EC2` |
| Prom `100 - node_memory_MemAvailable_bytes/node_memory_MemTotal_bytes * 100` | `mem_used_percent` | `CWAgent` |
| Prom `node_filesystem_avail_bytes / node_filesystem_size_bytes * 100` | `disk_used_percent` (inverted) | `CWAgent` |
| Prom `rate(node_network_receive_bytes_total[5m])` | `NetworkIn` | `AWS/EC2` |
| Prom `rate(node_network_transmit_bytes_total[5m])` | `NetworkOut` | `AWS/EC2` |
| Prom `http_request_duration_seconds{quantile="0.95"}` | `TargetResponseTime` (p95) | `AWS/ApplicationELB` |
| Prom `rate(http_requests_total{status=~"5.."}[5m])` | `HTTPCode_Target_5XX_Count` | `AWS/ApplicationELB` |
| Prom `rate(http_requests_total[5m])` | `RequestCount` | `AWS/ApplicationELB` |
| Datadog `system.cpu.user + system.cpu.system` | `CPUUtilization` | `AWS/EC2` |
| Datadog `system.mem.pct_usable` | `mem_used_percent` (inverted) | `CWAgent` |
| Datadog `system.disk.in_use` | `disk_used_percent` | `CWAgent` |
| Datadog `system.net.bytes_rcvd` | `NetworkIn` | `AWS/EC2` |
| Datadog `system.net.bytes_sent` | `NetworkOut` | `AWS/EC2` |
| New Relic `system.cpuPercent` | `CPUUtilization` | `AWS/EC2` |
| New Relic `system.memoryUsedPercent` | `mem_used_percent` | `CWAgent` |
| New Relic `system.diskUsedPercent` | `disk_used_percent` | `CWAgent` |
| RDS-level Prom `rate(pg_stat_database_tup_fetched[5m])` | `ReadIOPS` | `AWS/RDS` |
| RDS-level Prom `rate(pg_stat_database_tup_inserted + pg_stat_database_tup_updated + pg_stat_database_tup_deleted)[5m]` | `WriteIOPS` | `AWS/RDS` |
| RDS-level Prom `pg_stat_activity_count` | `DatabaseConnections` | `AWS/RDS` |

### Dimensions note

The mapping table specifies the metric name and namespace. Falcon must also supply the appropriate CloudWatch dimension to scope the query to a specific resource. The dimension to use depends on the namespace:

- `AWS/EC2` → `Name=InstanceId,Value=<aws_id>`
- `AWS/EBS` → `Name=VolumeId,Value=<volume_id>` (resolved from the EC2 instance's attached volumes)
- `AWS/RDS` → `Name=DBInstanceIdentifier,Value=<db_instance_id>`
- `AWS/ApplicationELB` → `Name=LoadBalancer,Value=<load_balancer_arn_suffix>`
- `CWAgent` → `Name=InstanceId,Value=<aws_id>` (plus any additional dimensions configured in the agent, e.g., `fstype`, `path` for disk metrics)

The mapping context (resource pair from Gate 2) provides `aws_id`; Falcon resolves related resource IDs (volumes, RDS instances) from the EC2 describe APIs as needed.

### Aggregation notes

Where the CloudWatch equivalent shows a compound expression, Falcon applies the specified aggregation:

- **Sum:** `DiskReadOps + DiskWriteOps`, `NetworkIn + NetworkOut` — fetch both metrics, sum the time-series values point-by-point before computing percentiles.
- **Average:** `VolumeTotalReadTime + VolumeTotalWriteTime (avg)` — fetch both metrics, average the time-series values point-by-point before computing percentiles.
- **Inverted:** `disk_used_percent (inverted)` for the Prometheus `node_filesystem_avail_bytes` ratio — Falcon computes `100 - source_value` before comparison since Prometheus returns available fraction while CloudWatch reports used percent. Similarly, `system.mem.pct_usable (inverted)` from Datadog is `100 - source_value`.

---

## Statistical comparison

For each mapped metric pair on a paired resource, Falcon executes the following comparison procedure.

### Data collection

1. **Source side:** Query the source metric over `source_window` (default: 7 days immediately preceding the recorded cutover timestamp). Collect the raw time-series as a list of numeric values (one per sample interval).
2. **Target side:** Query the CloudWatch metric over `target_window` (default: 7 days immediately following the recorded cutover timestamp). Collect the raw time-series as a list of numeric values.

Both windows default to 7 days but can be overridden via `METRICS_SOURCE_WINDOW_DAYS` and `METRICS_TARGET_WINDOW_DAYS` environment variables.

### Statistical summary

Compute for each side's time series:
- `p50` — median (50th percentile)
- `p95` — 95th percentile
- `p99` — 99th percentile
- `mean` — arithmetic mean

### Ratio computation

For each statistic `s` in {p50, p95, p99, mean}:

```
ratio_s = target_s / source_s
```

If `source_s` is zero, set `ratio_s = null` and skip that statistic to avoid division by zero.

### Divergence classification

Determine the maximum deviation across the four statistics, taking into account the regression direction for the metric class (see table below):

| Metric class | Regression direction |
|---|---|
| CPU utilization % | up |
| Memory usage % | up |
| Disk usage % | up |
| Network throughput | Neutral — for steady workloads, DOWN is regression; for bursty workloads, context-dependent. Default: flag as informational if ≥30% delta in either direction. |
| Response time / latency | up |
| 5xx error rate | up |
| Request count | Neutral — informational only, regardless of direction. |

**Severity thresholds** (applied to the worst-case statistic across p50, p95, p99, mean):

| Max divergence | Severity |
|---|---|
| Within ±10% of source (ratio 0.90–1.10) | OK — no divergence emitted |
| 10–30% regression | minor `metrics.regression` |
| 30–50% regression | major `metrics.regression` |
| >50% regression | critical `metrics.regression` |

"Regression" means movement in the direction indicated bad by the metric class's regression direction column. For example, CPU going from 40% (source) to 65% (target) is a regression (up direction); CPU going from 40% to 20% is an improvement.

**Improvements:** If the target is more than 10% better than the source (in the beneficial direction), Falcon emits `metrics.improvement` — informational, not a divergence. Improvements are logged for the operator's awareness but do not affect the overall parity status.

### Example divergence (regression)

```json
{
  "type": "metrics.regression",
  "severity": "major",
  "phase": "metrics",
  "resourcePair": {"vmware_id": "vm-1001", "aws_id": "i-0abc123"},
  "metric": {
    "source_system": "prometheus",
    "source_metric": "rate(node_cpu_seconds_total{mode!=\"idle\"}[5m])",
    "cloudwatch_metric": "CPUUtilization",
    "cloudwatch_namespace": "AWS/EC2"
  },
  "evidence": {
    "source_window": "2024-03-01T00:00:00Z/2024-03-08T00:00:00Z",
    "target_window": "2024-03-08T00:00:00Z/2024-03-15T00:00:00Z",
    "source_stats": {"p50": 28.5, "p95": 61.2, "p99": 74.1, "mean": 31.0},
    "target_stats": {"p50": 48.3, "p95": 88.7, "p99": 95.0, "mean": 52.4},
    "ratios": {"p50": 1.70, "p95": 1.45, "p99": 1.28, "mean": 1.69},
    "max_regression_ratio": 1.70,
    "regression_direction": "up"
  }
}
```

---

## Percentile computation recipe

Falcon computes percentiles from raw time-series data piped as one value per line. Two recipes are provided; Falcon selects based on environment availability.

### awk one-liner (any POSIX system)

```bash
sort -n | awk 'BEGIN{c=0} {a[c++]=$1} END{
  p50=a[int(c*0.5)]; p95=a[int(c*0.95)]; p99=a[int(c*0.99)];
  sum=0; for(i=0;i<c;i++) sum+=a[i]; mean=sum/c;
  printf "p50=%g p95=%g p99=%g mean=%g\n", p50, p95, p99, mean
}'
```

Usage: pipe raw values (one per line) into this command.

### Python one-liner (Python 3.9+ required)

```bash
python3 -c "import sys, statistics as s
v = sorted(float(x) for x in sys.stdin if x.strip())
print(f'p50={v[int(len(v)*0.5)]} p95={v[int(len(v)*0.95)]} p99={v[int(len(v)*0.99)]} mean={s.mean(v)}')"
```

Usage: pipe raw values (one per line) into this command.

### Selection logic

Falcon checks `python3 --version` at Phase 0 init. If Python 3.9 or later is available, use the Python recipe (more readable output, handles edge cases cleanly). Otherwise fall back to the awk recipe.

### Precision notes

Both recipes use the **nearest-rank** percentile method: the percentile index is computed as `floor(n * percentile_fraction)`. This is a common, standard approach sufficient for operational comparison purposes.

**Small-sample warning:** If the time series contains fewer than 100 data points, Falcon emits `metrics.insufficient-samples` (minor severity) alongside any divergence, flagging the results as low-confidence. Statistical conclusions from fewer than 100 points should not be treated as definitive.

### Example pipeline

```bash
# Fetch CloudWatch data points and pipe into percentile recipe
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions "Name=InstanceId,Value=${INSTANCE_ID}" \
  --start-time "${TARGET_START}" --end-time "${TARGET_END}" \
  --period 300 --statistics Average \
  --region "${AWS_REGION}" --no-cli-pager \
  | jq -r '.Datapoints[].Average' \
  | sort -n | awk 'BEGIN{c=0} {a[c++]=$1} END{
      p50=a[int(c*0.5)]; p95=a[int(c*0.95)]; p99=a[int(c*0.99)];
      sum=0; for(i=0;i<c;i++) sum+=a[i]; mean=sum/c;
      printf "p50=%g p95=%g p99=%g mean=%g\n", p50, p95, p99, mean
    }'
```

---

## Unmapped metric handling

If a source-side metric enumerated from the catalog has no matching entry in the mapping table above, Falcon must NOT attempt to fabricate or guess a CloudWatch equivalent. Guessing introduces false confidence into parity assessments.

**Protocol for unmapped metrics:**

1. Do not query CloudWatch for this metric.
2. Do not skip silently — always record the finding.
3. Emit a `metrics.unmapped` divergence at minor severity.
4. Include in evidence: the source metric name, the source system, and a note explaining that manual review is required.

**Example divergence:**

```json
{
  "type": "metrics.unmapped",
  "severity": "minor",
  "phase": "metrics",
  "resourcePair": {"vmware_id": "vm-1001", "aws_id": "i-0abc"},
  "evidence": {
    "source_system": "prometheus",
    "source_metric": "custom_app_worker_queue_depth",
    "note": "No CloudWatch equivalent in v1 mapping table — manual review required."
  }
}
```

Unmapped metrics accumulate in the Phase 7 report under a dedicated "Unmapped Metrics" section so operators can decide whether to add custom CloudWatch metrics or accept the gap.

---

## Divergence classification matrix

Summary of all `metrics.*` divergence types Falcon may emit in Phase 5:

| Type | Severity | Description |
|---|---|---|
| `metrics.regression` | minor (10–30%) / major (30–50%) / critical (>50%) | Target metric is statistically worse than source beyond the tolerance threshold. |
| `metrics.improvement` | informational (not a divergence, logged only) | Target metric is statistically better than source by more than 10%. Does not affect parity status. |
| `metrics.unmapped` | minor | Source metric has no CloudWatch equivalent in the v1 mapping table. Manual review required. |
| `metrics.source-unavailable` | minor | A configured metric source's catalog enumeration failed (bad credentials, network error, partial env vars). Logged per source. |
| `metrics.insufficient-samples` | minor | Fewer than 100 data points were available for statistical comparison. Results are low-confidence. |

**Parity gate logic:** Phase 5 does not block the workflow. All divergences are recorded and forwarded to Phase 7's consolidated report. Critical-severity `metrics.regression` findings must be explicitly acknowledged in the Phase 7 human-review checklist before sign-off.
