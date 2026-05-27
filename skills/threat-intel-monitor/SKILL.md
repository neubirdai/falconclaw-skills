---
name: threat-intel-monitor
description: "Check if your services are being attacked using threat intelligence websites. Merlin automatically runs this in the background and saves findings. Use when: user asks about malicious IPs, threats, or security posture."
tags: [security, proactive, threat-intel]
homepage: https://neubird.ai/skills/threat-intel-monitor
metadata: {"neubird":{"emoji":"🛡️","requires":{"features":["tool_exec","web_search"]}}}
---

# Threat Intelligence Monitor

Check if your services are being attacked by looking up IPs and network indicators against threat intelligence websites.

## Threat Intelligence Websites

Use these sites to check IPs and indicators:

**IP Reputation & Abuse**
- https://www.abuseipdb.com/ — crowdsourced IP abuse reports, confidence scores
- https://www.virustotal.com/ — multi-engine malware/IP scanning
- https://www.shodan.io/ — exposed ports, services, vulnerabilities

**Blocklists & Known Bad IPs**
- https://isc.sans.edu/ — SANS Internet Storm Center threat feeds
- https://feodotracker.abuse.ch/ — Emotet/TrickBot/Dridex C2 servers
- https://www.spamhaus.org/ — spam and botnet blocklists
- https://lists.blocklist.de/ — fail2ban aggregated attack reports

**Threat Intelligence Platforms**
- https://otx.alienvault.com/ — AlienVault Open Threat Exchange
- https://threatfox.abuse.ch/ — malware IOC sharing
- https://urlhaus.abuse.ch/ — malicious URL tracking
- https://bazaar.abuse.ch/ — malware sample exchange

**Network Intelligence**
- https://bgpview.io/ — ASN/IP ownership
- https://ipinfo.io/ — geolocation, hosting provider
- https://www.whois.com/ — domain/IP registration

**Government / CERT**
- https://www.cisa.gov/known-exploited-vulnerabilities-catalog — CISA KEV
- https://www.circl.lu/ — Luxembourg CERT, passive DNS/SSL

## How It Works

Merlin's sentinel runs this automatically in the background. Each sweep:
1. Collects your telemetry data (logs, CloudTrail, VPC flows, etc.)
2. Sends it to Claude with web_search: "Using these threat intel sites, check if my services are being attacked"
3. Claude finds IPs in the data, searches them, and reports back
4. Any threats get saved as `minority_report` incidents with `lens=threat-intel`

## Steps

### View existing findings

```sql
SELECT id, title, summary, risk, created_at, enriched_raw_data
FROM merlin.incidents
WHERE category = 'minority_report'
AND enriched_raw_data LIKE '%"lens":"threat-intel"%'
ORDER BY created_at DESC
LIMIT 25
```

### Investigate a specific IP (if user asks)

Use `web_search`:

- `"<IP>" site:abuseipdb.com`
- `"<IP>" site:virustotal.com`
- `"<IP>" site:shodan.io`
- `"<IP>" site:otx.alienvault.com`

## Output Format

**🛡 Threat Intel Report: X finding(s)**

| IP | Threat Type | Source | Seen In | Confidence | When |
|----|-------------|--------|---------|------------|------|

If no findings: "No malicious IPs detected. Your services look clean."
