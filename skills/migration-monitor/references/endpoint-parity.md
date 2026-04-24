# Endpoint Parity Reference

## Purpose

This reference drives Phase 4 endpoint parity checks. It is consumed after Phase 1 mapping is locked — only derived endpoint pairs from confirmed VM-to-AWS pairings are evaluated. Falcon runs read-only probes against each endpoint pair, compares responses, and emits structured divergences. All probes are read-only: curl uses GET by default, nc tests connectivity only, and openssl reads certificate metadata only. No POST, PUT, PATCH, or DELETE verbs are ever issued.

---

## Endpoint pair derivation

Endpoint pairs are constructed from the Phase 1 mapping. Each pairing maps a source VM endpoint to its AWS counterpart based on the VM's exposure type.

### VM with public IP → AWS EC2 with EIP or public DNS

Source URL is built from the VM's hostname or public IP (`source.publicIp` or `source.hostname`). Target URL is the Elastic IP's public DNS record, or the paired Route 53 record if DNS-managed. Pair supports `probe_types: ["http", "tcp", "tls"]`.

### VM behind a load balancer → AWS ALB/NLB

Source URL is the original FQDN the VM registered under (from user input or DNS lookup at discovery time). Target URL is the ALB/NLB DNS name retrieved from:

```bash
aws elbv2 describe-load-balancers --query 'LoadBalancers[].DNSName'
```

Pair supports `probe_types: ["http", "tcp", "tls"]`.

### VM DB port (3306 / 5432 / 1433) → AWS RDS endpoint

Source is the VM's IP and database port. Target is the RDS endpoint hostname and the same port. Because database ports do not serve HTTP, only TCP and TLS probes are run. Pair supports `probe_types: ["tcp", "tls"]`.

### Internal-only VM (no public exposure)

VMs with no public IP, no load balancer registration, and no user-provided probe endpoint are skipped for endpoint parity. Falcon records a skip notice in the Phase 4 log but does not emit a divergence. Users may supply an internal probe endpoint in config to opt internal VMs into parity checks.

### Endpoint pair list structure

Falcon builds the following JSON structure from the Phase 1 mapping before Phase 4 begins:

```json
{
  "pairs": [
    {
      "id": "pair-web-01",
      "source": { "scheme": "https", "host": "web-01.corp.example", "port": 443, "path": "/" },
      "target": { "scheme": "https", "host": "web-01-alb-xxx.us-east-1.elb.amazonaws.com", "port": 443, "path": "/" },
      "probe_types": ["http", "tcp", "tls"]
    },
    {
      "id": "pair-db-01",
      "source": { "scheme": "tcp", "host": "10.0.2.10", "port": 5432 },
      "target": { "scheme": "tcp", "host": "db-prod-01.xxx.us-east-1.rds.amazonaws.com", "port": 5432 },
      "probe_types": ["tcp", "tls"]
    }
  ]
}
```

---

## HTTP probe recipe

### a. Exact curl commands Falcon emits

For each pair with `"http"` in `probe_types`, Falcon emits these exact commands (substituting `$PAIR_ID`, `$SOURCE_URL`, `$TARGET_URL` at runtime):

```bash
# Source side
curl -sS \
     -o /tmp/mmon_source_body_${PAIR_ID}.out \
     -D /tmp/mmon_source_headers_${PAIR_ID}.out \
     -w '%{http_code}\n%{time_total}\n%{content_type}\n%{size_download}\n' \
     --max-time 30 \
     "$SOURCE_URL"

# Target side (same flags, different output paths)
curl -sS \
     -o /tmp/mmon_target_body_${PAIR_ID}.out \
     -D /tmp/mmon_target_headers_${PAIR_ID}.out \
     -w '%{http_code}\n%{time_total}\n%{content_type}\n%{size_download}\n' \
     --max-time 30 \
     "$TARGET_URL"
```

When the `$AUTH_HEADER` environment variable is set (e.g. `Authorization: Bearer xxx`), append `--header "$AUTH_HEADER"` to both commands. The `--max-time 30` flag is mandatory and must not be removed; it prevents hung probes from blocking Phase 4 indefinitely.

### b. Status and performance comparison rules

**HTTP status code (`http_code`):**
Mismatch between source and target = **critical** divergence `endpoint.status-mismatch`. This check fires regardless of what the status codes are; any difference is a critical signal.

**Content-Type (`content_type`):**
Structural mismatch (e.g., `application/json` on source vs `text/html` on target) = **major** `endpoint.content-type-mismatch`. Charset-only differences (`utf-8` vs `UTF-8`) are normalized to lowercase before comparison; charset-only mismatches are informational only (no divergence emitted).

**Response time (`time_total`):**
Informational only at this layer. Performance latency deltas belong to Phase 5 metrics parity and are not assessed here.

**Download size (`size_download`):**
Within ±20% of the source value = informational. Greater than 50% deviation = **minor** `endpoint.size-deviation` (possible content drift between source and target). The ±20% band accounts for minor dynamic content differences; the >50% threshold flags material divergence.

### c. Structural header comparison

Headers are compared case-insensitively (keys normalized to lowercase). Headers compared:

- **`Content-Type`** — covered by rule b above.
- **`Cache-Control`** — divergence if directive sets differ (e.g., source `no-store` but target `max-age=3600`). Directive comparison ignores order.
- **Security headers:** `Strict-Transport-Security`, `X-Content-Type-Options`, `X-Frame-Options`, `Content-Security-Policy`.

Security header rule: a security header present on the source but absent from the target = **major** `endpoint.security-header-missing`. An extra security header present on the target but not on the source = informational (target has more protection — not a regression).

### d. Semantic body diff

Body comparison is content-type-aware.

**JSON responses (`Content-Type: application/json`):**

1. Parse both body files as JSON.
2. Canonicalize each parsed object: recursively sort all object keys alphabetically; strip any field whose key matches the volatile-field strip list (case-insensitive regex match): `timestamp`, `requestId`, `traceId`, `date`, `server_time`, `now`. These fields change between requests and must be excluded before diffing.
3. Serialize both canonicalized objects with consistent formatting.
4. Diff the serialized forms. Any remaining structural difference = **major** `endpoint.body-structural-mismatch`.

**HTML responses (`Content-Type: text/html`):**

1. Normalize both bodies: strip doctype declarations and HTML comments; lowercase all attribute names; sort attributes within each tag alphabetically; collapse all whitespace (multiple spaces, newlines) to a single space.
2. Diff the normalized forms. Any remaining difference = **minor** `endpoint.body-html-drift`. HTML drift is common and often benign (inline timestamps, session tokens in markup, minor template changes). It is minor by default but users may escalate the severity floor at Gate 3.

**All other content-types:**

Perform a byte-exact comparison of both body files. Any difference = **minor** `endpoint.body-bytes-mismatch`. Without content-type-specific parsing there is insufficient information to judge semantic equivalence, so this is kept at minor severity.

### e. Failure modes

If either curl invocation exits with a non-zero exit code — meaning the host was entirely unreachable, connection was refused, or the request timed out — Falcon emits **critical** `endpoint.unreachable`. The evidence identifies which side failed and includes the curl exit code. Note: a non-200 HTTP status is not a curl error; `endpoint.unreachable` fires only on curl process-level failure (exit code ≠ 0), not on HTTP 4xx/5xx.

---

## TCP probe recipe

For each pair with `"tcp"` in `probe_types`, Falcon runs the following nc commands:

```bash
nc -z -w5 "$SOURCE_HOST" "$SOURCE_PORT"; SRC_REACH=$?
nc -z -w5 "$TARGET_HOST" "$TARGET_PORT"; TGT_REACH=$?
```

`-z` scans without sending data; `-w5` limits the connection wait to 5 seconds. Both commands capture exit code only (0 = reachable, non-zero = unreachable).

**Divergences:**

- `SRC_REACH=0, TGT_REACH≠0` (source reachable, target not) = **critical** `endpoint.reachability-regression`. The migration has broken connectivity — the target port is not accepting connections where the source was. Example:

```json
{
  "id": "endpoint-010",
  "phase": "endpoint",
  "type": "endpoint.reachability-regression",
  "severity": "critical",
  "resourcePair": {"vmware_id": "vm-2002", "aws_id": "i-0def456"},
  "pairId": "pair-db-01",
  "description": "Source TCP port reachable but target TCP port unreachable — migration connectivity broken",
  "evidence": {
    "probe": "tcp",
    "source_host": "10.0.2.10",
    "source_port": 5432,
    "target_host": "db-prod-01.xxx.us-east-1.rds.amazonaws.com",
    "target_port": 5432,
    "source_exit_code": 0,
    "target_exit_code": 1
  }
}
```

- `SRC_REACH≠0, TGT_REACH=0` (target reachable, source was not) = **critical** `endpoint.reachability-regression`. A previously unreachable port is now publicly open on AWS — a potential security regression that requires immediate review.

- `SRC_REACH=0, TGT_REACH=0` (both reachable) = no divergence.

- `SRC_REACH≠0, TGT_REACH≠0` (both unreachable) = **minor** `endpoint.both-unreachable`. This may be expected if the source has been decommissioned or if the port was never publicly reachable during the probe window. Example:

```json
{
  "id": "endpoint-011",
  "phase": "endpoint",
  "type": "endpoint.both-unreachable",
  "severity": "minor",
  "resourcePair": {"vmware_id": "vm-2002", "aws_id": "i-0def456"},
  "pairId": "pair-db-01",
  "description": "Both source and target TCP ports unreachable — source may have been decommissioned",
  "evidence": {
    "probe": "tcp",
    "source_host": "10.0.2.10",
    "source_port": 5432,
    "target_host": "db-prod-01.xxx.us-east-1.rds.amazonaws.com",
    "target_port": 5432,
    "source_exit_code": 1,
    "target_exit_code": 1
  }
}
```

---

## TLS probe recipe

For each pair with `"tls"` in `probe_types`, Falcon fetches certificate details from both sides using openssl. The `PAIR_ID` and `SIDE` variables are substituted at runtime (`SIDE` = `source` or `target`):

```bash
# For each side, fetch cert details
echo | openssl s_client -connect "$HOST:$PORT" -servername "$HOST" 2>/dev/null \
  | openssl x509 -noout -dates -subject -issuer \
  > /tmp/mmon_tls_${PAIR_ID}_${SIDE}.txt
```

This command is read-only: it connects, reads the certificate, then closes immediately. No data is written to the server.

Falcon extracts the following fields from each output file:
- `notBefore` — certificate valid-from date.
- `notAfter` — certificate expiry date.
- `subject` CN — the Common Name from the subject DN.
- `issuer` — full issuer distinguished name.

### Checks

**Target certificate validity (not expired):**
Parse `notAfter` from the target side. If `notAfter` is in the past = **critical** `endpoint.tls-expired`. An expired certificate will cause browser and client rejections immediately after cutover.

**Target certificate not-yet-valid:**
Parse `notBefore` from the target side. If `notBefore` is in the future = **critical** `endpoint.tls-pre-valid`. A pre-valid certificate will be rejected by TLS clients until its validity window opens.

**Subject CN match:**
The target certificate's subject CN must match the endpoint hostname being probed. Wildcard certificates are acceptable: `*.example.com` matches `foo.example.com` (single label substitution only — `*.example.com` does not match `a.b.example.com`). If the CN does not match the hostname (and no Subject Alternative Name covers it) = **major** `endpoint.tls-cn-mismatch`.

**Issuer family:**
Record both the source and target issuers. If the issuer families differ (e.g., LetsEncrypt on source, AWS ACM on target, or corporate CA on source), this is **informational** — a different CA is acceptable as long as the certificate is valid and the CN check passes. Record issuer details in evidence for audit purposes but do not emit a divergence.

**Note:** v1 does not attempt chain-trust equivalence between source and target. Verifying full trust chains requires matching trust stores, which differ across operating systems and environments. Chain-trust equivalence is deferred to v1.1.

Example divergence (expired target certificate):

```json
{
  "id": "endpoint-020",
  "phase": "endpoint",
  "type": "endpoint.tls-expired",
  "severity": "critical",
  "resourcePair": {"vmware_id": "vm-3003", "aws_id": "i-0ghi789"},
  "pairId": "pair-web-01",
  "description": "Target TLS certificate expired: notAfter=2024-01-15T00:00:00Z is in the past",
  "evidence": {
    "probe": "tls",
    "side": "target",
    "host": "web-01-alb-xxx.us-east-1.elb.amazonaws.com",
    "port": 443,
    "not_before": "2023-01-15T00:00:00Z",
    "not_after": "2024-01-15T00:00:00Z",
    "subject_cn": "web-01-alb-xxx.us-east-1.elb.amazonaws.com",
    "issuer": "CN=Amazon RSA 2048 M01, O=Amazon, C=US"
  }
}
```

---

## Divergence emission format

Every Phase 4 divergence follows the shape the report schema formalizes in Task 8:

```json
{
  "id": "endpoint-001",
  "phase": "endpoint",
  "type": "endpoint.status-mismatch",
  "severity": "critical",
  "resourcePair": {"vmware_id": "vm-1001", "aws_id": "i-0abc123"},
  "pairId": "pair-web-01",
  "description": "HTTP status differs: source 200, target 500",
  "evidence": {
    "probe": "http",
    "source_url": "https://web-01.corp.example/",
    "target_url": "https://web-01-alb-xxx.elb.amazonaws.com/",
    "source_http_code": 200,
    "target_http_code": 500,
    "source_body_sample": "<first 200 bytes>",
    "target_body_sample": "<first 200 bytes>"
  }
}
```

Required fields on every divergence:

- `id` — unique string within the Phase 4 run (e.g., `endpoint-NNN`).
- `phase` — always `"endpoint"` for Phase 4 divergences.
- `type` — one of the 13 divergence type strings defined in the classification matrix below.
- `severity` — one of `"critical"`, `"major"`, `"minor"`, `"informational"`.
- `resourcePair` — the VM-to-EC2 pair from Phase 1 mapping.
- `pairId` — the endpoint pair `id` from the pair list structure.
- `description` — human-readable summary of what differs.
- `evidence` — structured object with probe-type-specific fields.

All 13 divergence types produced by Phase 4:

| Divergence type | Severity |
|---|---|
| `endpoint.status-mismatch` | critical |
| `endpoint.content-type-mismatch` | major |
| `endpoint.size-deviation` | minor |
| `endpoint.security-header-missing` | major |
| `endpoint.body-structural-mismatch` | major |
| `endpoint.body-html-drift` | minor |
| `endpoint.body-bytes-mismatch` | minor |
| `endpoint.unreachable` | critical |
| `endpoint.reachability-regression` | critical |
| `endpoint.both-unreachable` | minor |
| `endpoint.tls-expired` | critical |
| `endpoint.tls-pre-valid` | critical |
| `endpoint.tls-cn-mismatch` | major |

---

## Divergence classification matrix

Summary of all divergence types produced by Phase 4, their severity, and the probe that generates them.

| Divergence type | Severity | Probe | Notes |
|---|---|---|---|
| `endpoint.status-mismatch` | critical | http | Any HTTP status code mismatch |
| `endpoint.content-type-mismatch` | major | http | Structural type mismatch; charset-only differences are informational |
| `endpoint.size-deviation` | minor | http | >50% response body size deviation |
| `endpoint.security-header-missing` | major | http | Security header present on source but absent on target |
| `endpoint.body-structural-mismatch` | major | http | JSON body differs after canonicalization and volatile-field stripping |
| `endpoint.body-html-drift` | minor | http | HTML body differs after normalization; often benign |
| `endpoint.body-bytes-mismatch` | minor | http | Non-JSON/HTML body differs byte-for-byte |
| `endpoint.unreachable` | critical | http | curl process-level failure (non-zero exit code) on either side |
| `endpoint.reachability-regression` | critical | tcp | Source reachable but target not, or target newly open where source was closed |
| `endpoint.both-unreachable` | minor | tcp | Both sides unreachable; may be expected post-decommission |
| `endpoint.tls-expired` | critical | tls | Target certificate notAfter is in the past |
| `endpoint.tls-pre-valid` | critical | tls | Target certificate notBefore is in the future |
| `endpoint.tls-cn-mismatch` | major | tls | Target certificate CN does not match probed hostname |
