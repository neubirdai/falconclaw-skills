---
name: threat-intel-monitor
description: "Cross-reference IPs from customer telemetry against public threat intelligence using web search. Use when: (1) checking if IPs in your environment are malicious, (2) investigating suspicious network activity, (3) proactive security posture monitoring."
tags: [security, proactive, threat-intel]
homepage: https://neubird.ai/skills/threat-intel-monitor
metadata: {"neubird":{"emoji":"🛡️","requires":{"features":["web_search","tool_exec"]}}}
---

# Threat Intelligence Monitor

Extract IPs from your telemetry sources, then use `web_search` to check them against public threat intelligence databases.

## When to Use

- Checking whether IPs seen in your environment appear in known threat feeds
- Investigating suspicious source/destination IPs from firewall or VPC flow logs
- Periodic security sweeps of your telemetry for indicators of compromise
- Enriching incident investigations with external threat context

## How It Works

1. **Query telemetry** — Use `tool_exec` to run SQL against FDW tables (firewall logs, VPC flow logs, access logs, etc.) to extract unique IPs
2. **Check each IP** — Use `web_search` to look up extracted IPs against public threat intel sources
3. **Report findings** — Flag any IPs that appear in blocklists or have abuse reports

## Step 1: Extract IPs from Telemetry

Query FDW tables that contain network traffic data. Look for tables with columns like `source_ip`, `destination_ip`, `client_ip`, `remote_addr`, or similar.

Example queries:
```sql
-- Unique external source IPs from firewall logs (last 6h)
SELECT DISTINCT source_ip FROM "firewall_logs"."entries"
WHERE timestamp > NOW() - INTERVAL '6 hours'
LIMIT 50

-- Unique destination IPs from VPC flow logs
SELECT DISTINCT destination_ip FROM "vpc_flow"."logs"
WHERE timestamp > NOW() - INTERVAL '6 hours'
AND action = 'ACCEPT'
LIMIT 50
```

Collect the unique IPs, then move to Step 2.

## Step 2: Check IPs via Web Search

For each IP (or batch of IPs), search these sources in priority order:

1. **AbuseIPDB** — abuse confidence scores and report history
   - Search: `site:abuseipdb.com "<IP>"`
2. **VirusTotal** — aggregated detection results
   - Search: `site:virustotal.com "<IP>"`
3. **GreyNoise** — is this IP a known scanner/attacker?
   - Search: `site:viz.greynoise.io "<IP>"`
4. **Threat Fox (abuse.ch)** — IOC linked to malware campaigns
   - Search: `site:threatfox.abuse.ch "<IP>"`
5. **Shodan** — exposed services and vulnerabilities
   - Search: `site:shodan.io "<IP>"`

To check multiple IPs efficiently, batch them:
- `"<IP1>" OR "<IP2>" OR "<IP3>" malicious threat report`

## Step 3: Report Findings

Present findings as a structured table:

| IP | Source | Found In | Threat Type | Confidence | Details |
|----|--------|----------|-------------|------------|---------|
| 1.2.3.4 | firewall_logs | AbuseIPDB | Brute Force | 95% | 47 reports in last 30 days |

Above the table, include a one-line summary: how many telemetry IPs were checked, how many flagged, highest severity.

## Rules

- Query telemetry FIRST to get real IPs from the customer's environment — do not just pull generic blocklists
- Use at most 3-4 web searches per check to stay within rate limits
- Batch IPs in web searches where possible
- Always include the source URL so findings can be verified
- If an IP appears in multiple threat sources, note the corroboration — higher confidence
- Do NOT fabricate IOC data — only report what web search actually returns
- If no IPs are flagged, say "No threats detected in your telemetry" — false positives destroy trust
- When used with `/watch`, the plan should be: re-query telemetry for new IPs, then web search those
