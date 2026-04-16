---
name: threat-intel-monitor
description: "Show threat intelligence findings — malicious IPs detected in your telemetry. Merlin's sentinel automatically extracts IPs from telemetry and checks them via web search against threat databases. Use when: (1) user asks about malicious IPs or threat intel, (2) user wants a threat report, (3) periodic security posture review."
tags: [security, proactive, threat-intel]
homepage: https://neubird.ai/skills/threat-intel-monitor
metadata: {"neubird":{"emoji":"🛡️","requires":{"features":["tool_exec","web_search"]}}}
---

# Threat Intelligence Monitor

Surface threat intelligence findings from Merlin's automated background scans, and optionally investigate specific IPs deeper via web search.

## How It Works

Merlin's sentinel runs in the background at a configured interval. Each run:
1. Extracts distinct external IPs from customer telemetry (FDW log tables)
2. Checks those IPs via Anthropic's web_search against threat intelligence databases (AbuseIPDB, VirusTotal, Shodan, Spamhaus, etc.)
3. Persists any flagged IPs as `minority_report` incidents with `lens=threat-intel`

This skill retrieves those findings. It can also do deeper investigation on specific IPs.

## Steps

### View existing findings

Query the incidents table for recent threat intel findings:

```sql
SELECT id, title, summary, risk, created_at, enriched_raw_data
FROM merlin.incidents
WHERE category = 'minority_report'
AND enriched_raw_data LIKE '%"lens":"threat-intel"%'
ORDER BY created_at DESC
LIMIT 25
```

Parse `enriched_raw_data` (JSON string) to extract: `ip`, `feed`, `threat_type`, `confidence`, `telemetry_source`, `port`, `first_seen`.

### Investigate a specific IP (optional, if user asks)

If the user asks about a specific IP or wants deeper analysis, use `web_search`:

- Search: `"<IP>" AbuseIPDB threat intelligence`
- Search: `"<IP>" VirusTotal malicious`
- Search: `"<IP>" Shodan open ports`

Present findings with source links.

## Output Format

One-line summary first, then the table:

**🛡 Threat Intel Report: X finding(s) — Y malicious IPs detected in your telemetry**

| IP | Threat Type | Source | Seen In | Confidence | When |
|----|-------------|--------|---------|------------|------|

- **Seen In** = the telemetry table where the IP was found (e.g. `log_vpc_flow.entries`)
- **Source** = which threat intel database flagged it (e.g. `AbuseIPDB`)
- If no findings exist, say: "No malicious IPs detected in your telemetry. Your network is clean."

## Rules

- Always check the database FIRST for existing Merlin findings
- Only use web_search for deeper investigation on specific IPs the user asks about
- When used with `/watch`, re-query the DB each interval to pick up new findings
- If the user asks about a specific IP, filter: `AND enriched_raw_data LIKE '%"ip":"<IP>"%'`
