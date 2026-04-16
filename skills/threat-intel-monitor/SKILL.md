---
name: threat-intel-monitor
description: "Show threat intelligence findings — blocklist IPs that were found in your telemetry. Use when: (1) user asks about malicious IPs or threat intel, (2) user wants a threat report, (3) periodic security posture review."
tags: [security, proactive, threat-intel]
homepage: https://neubird.ai/skills/threat-intel-monitor
metadata: {"neubird":{"emoji":"🛡️","requires":{"features":["tool_exec"]}}}
---

# Threat Intelligence Monitor

Retrieve and present threat intelligence findings from Merlin's automated sentinel scans.

## How It Works

Merlin's sentinel runs in the background at a configured interval. Each run:
1. Fetches known-malicious IPs from public feeds (SANS ISC, Feodo Tracker)
2. Cross-references those IPs against your telemetry (FDW log tables)
3. Persists only the **matches** — blocklist IPs that were actually seen in your network

This skill retrieves those findings. It does NOT redo the scan.

## Steps

Query the incidents table for recent threat intel findings:

```sql
SELECT id, title, summary, risk, created_at, enriched_raw_data
FROM merlin.incidents
WHERE category = 'minority_report'
AND enriched_raw_data LIKE '%"lens":"threat-intel"%'
ORDER BY created_at DESC
LIMIT 25
```

Parse `enriched_raw_data` (JSON string) to extract: `ip`, `feed`, `threat_type`, `confidence`, `telemetry_source`.

## Output Format

One-line summary first, then the table:

**Threat Intel Report: X finding(s) — Y blocklist IPs detected in your telemetry**

| IP | Threat Type | Feed | Seen In | Confidence | When |
|----|-------------|------|---------|------------|------|

- **Seen In** = the telemetry table where the IP was found (e.g. `log_vpc_flow.entries`)
- If no findings exist, say: "No blocklist IPs detected in your telemetry. Your network is clean."

## Rules

- Do NOT run web_search or fetch external feeds — Merlin already did that
- Just query the database and present what's there
- When used with `/watch`, re-query the DB each interval to pick up new findings from the latest Merlin run
- If the user asks about a specific IP, filter: `AND enriched_raw_data LIKE '%"ip":"<IP>"%'`
