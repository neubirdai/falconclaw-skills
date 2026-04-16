---
name: threat-intel-monitor
description: "Monitor public threat intelligence feeds for malicious IPs, domains, and IOCs using web search. Use when: (1) periodic threat feed checks needed, (2) investigating suspicious network activity, (3) proactive security posture monitoring."
tags: [security, proactive, threat-intel]
homepage: https://neubird.ai/skills/threat-intel-monitor
metadata: {"neubird":{"emoji":"🛡️","requires":{"features":["web_search"]}}}
---

# Threat Intelligence Monitor

Continuously monitor public threat intelligence sources for malicious IPs, domains, and indicators of compromise (IOCs) using web search.

## When to Use

- Periodic checks of threat feeds for new malicious IPs targeting your industry/region
- Investigating whether specific IPs or domains appear in known blocklists
- Proactive security posture monitoring during oncall shifts
- Enriching incident investigations with external threat context

## How It Works

Use `web_search` to query public threat intelligence sources. Do NOT attempt SQL queries or database lookups for this — all data comes from the open web.

## Sources to Query

Search these public threat intelligence sources in priority order:

1. **AbuseIPDB** — recently reported malicious IPs, check abuse confidence scores
   - Search: `site:abuseipdb.com reported IPs today` or specific IP lookups
2. **Threat Fox (abuse.ch)** — IOCs including IPs, domains, URLs linked to malware
   - Search: `site:threatfox.abuse.ch recent IOCs`
3. **SANS Internet Storm Center** — current threat activity, top attacking IPs
   - Search: `site:isc.sans.edu infocon threat activity today`
4. **GreyNoise** — internet-wide scan and attack activity
   - Search: `site:viz.greynoise.io malicious IP activity`
5. **AlienVault OTX** — open threat exchange, community-sourced IOCs
   - Search: `site:otx.alienvault.com recent pulses malicious IP`
6. **Shodan** — exposed services and known vulnerable hosts
   - Search: `site:shodan.io vulnerability recent`
7. **URLhaus (abuse.ch)** — malicious URLs distributing malware
   - Search: `site:urlhaus.abuse.ch recent malware URLs`
8. **Feodo Tracker (abuse.ch)** — botnet C2 server tracking
   - Search: `site:feodotracker.abuse.ch botnet C2`

When the user asks about a **specific IP or domain**, search for it directly:
- `"<IP>" malicious threat report`
- `site:abuseipdb.com "<IP>"`
- `site:virustotal.com "<IP or domain>"`

## Output Format

Present findings as a structured table:

| IP/Domain | Source | Threat Type | Confidence | Last Seen | Details |
|-----------|--------|-------------|------------|-----------|---------|

Above the table, include a one-line threat summary: how many new IOCs found, highest severity, any trending attack patterns.

## Rules

- Use at most 3-4 web searches per check to stay within rate limits
- Focus on **new** or **recently reported** threats (last 24-48 hours)
- Always include the source URL so findings can be verified
- If an IP or domain appears in multiple sources, note the corroboration — higher confidence
- Do NOT fabricate IOC data — only report what web search actually returns
- If no new threats are found, say "No new threats detected" — false positives destroy trust
- When used with `/watch`, the plan should be the specific web search queries to repeat
