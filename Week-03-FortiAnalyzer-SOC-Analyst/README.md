## Week 03 — SOC Analyst using FortiAnalyzer

**Date:** November 14–19, 2026  
**Platform:** FortiAnalyzer 7.6.2 Analyst Virtual Lab (CIPAT SOC Analyst Lab)  
**Course:** FortiAnalyzer 7.6 Analyst (FTAO)

### Overview

The third week moved from configuring a firewall to watching one. Instead of building policies on FortiGate, the focus was on the Security Operations Center (SOC) side: collecting logs from FortiGate devices into FortiAnalyzer, searching and investigating them, turning them into events and incidents, and hunting for threats that no rule had flagged yet.

The lab environment reused the enterprise network from Week 02 (HQ-NGFW-1, HQ-ISFW, a branch FortiGate and an ISP router), now with FortiAnalyzer as the central log and SOC platform.

### Topics Covered

#### 1. FortiAnalyzer Fundamentals

- Administrative Domains (ADOMs): Fabric vs FortiGate type, disabled by default
- Analyzer vs collector mode
- Security Fabric logging (the first FortiGate that sees a session writes the traffic log)
- Configuration backup and restore

#### 2. Log Data Flow and Storage

- Log import from backup files (.log / .zst)
- Log insert pipeline: received → inserted into SQL (Analytics) → compressed archive (.gz)
- Data policy per ADOM: Analytics vs Archive retention, disk quota split, "quota beats retention"
- Log rate monitoring

```bash
diagnose fortilogd lograte-device
diagnose fortilogd lograte-type
diagnose fortilogd logvol-adom all
```

#### 3. Log Investigation

- Real-time vs historical logs, raw vs formatted view
- Text-mode searching (`field=value`, wildcards, case sensitivity)
- Custom Views for repeatable investigations
- Matching the field to the log type (`catdesc` = Web Filter, `appcat` = Traffic / Application Control)

```text
appcat=P2P
appcat=*Social*
app=*Dropbox*
```

#### 4. Threat Detection with FortiView

- IPS events from a simulated web vulnerability scan (Nikto)
- Drill-down: Top Threats → Source → Log View → raw log
- Destination NAT visible in the log (VIP 100.65.0.200 → 10.0.11.50, policy 5)
- Prioritizing by Threat Score rather than session count

#### 5. Event Handlers

- Event status: Mitigated (blocked) vs Unhandled (detected only)
- Predefined handlers, cloning, and avoiding duplicate events
- Notification profiles (email alerts)
- Custom handler for brute-force detection (≥ 3 failed logins in 5 minutes)

#### 6. Incident Management

- Escalating an event into an incident
- Category, assignment, comments, status workflow
- Audit History as the investigation record

#### 7. Threat Hunting

- Hypothesis-driven hunting: *"Has DNS tunneling been used to exfiltrate data?"*
- SIEM Analytics table and pivoting across Application Service, Destination Domain and logs
- Indicators: high DNS volume, random domains, external resolver (8.8.8.8)

#### 8. Outbreak Detection Service

- FortiGuard Outbreak Alerts (CVE, CVSS, known exploited, EPSS, IOCs)
- Bundled charts and reports for a specific outbreak (Apache Log4j2)

### Investigation Results

| Scenario | Finding |
| --- | --- |
| P2P usage | BitTorrent by user2 over SSL VPN, allowed by policy 1 |
| Social media | Facebook and Pinterest from 10.0.11.50, allowed |
| Web vulnerability scan | NetworkActiv.Web.Server.XSS from 100.65.0.254, DNAT to 10.0.11.50, policy 5 |
| Botnet beaconing | 168.10.199.186 → C&C 224.141.85.77 every ~1 minute; email alert sent |
| Brute-force login | 3 failed logins detected by a custom handler |
| DNS tunneling | 10.0.11.50 flooding 8.8.8.8 with random-domain queries |

---

### Hands-on Labs

Full step-by-step write-ups with screenshots are in [`labs/`](labs/).

**[Lab 01 — SOC Concepts and Security Fabric (Initial Configuration)](labs/LAB-01.md)** ([English](labs/LAB-01-EN.md))
Restored FortiAnalyzer from a backup, verified ADOM1 (Fabric) and ADOM2 (FortiGate) with their devices, and checked system status from the CLI.
Skills: FortiAnalyzer setup, ADOM concepts.

**[Lab 02 — Log Data Flow and Navigating Logs](labs/LAB-02.md)** ([English](labs/LAB-02-EN.md))
Imported historical logs, generated traffic, investigated P2P and social-media usage in Log View, detected a Nikto scan with IPS and drilled down from FortiView to the raw log, then reviewed log rates and ADOM storage.
Skills: log search and investigation, FortiView, log pipeline and storage planning.

**[Lab 03 — Incident and Event Management](labs/LAB-03.md)** ([English](labs/LAB-03-EN.md))
Worked through the full event lifecycle: IPS events, cloned and custom event handlers with email alerts, incident creation and closure, hypothesis-driven threat hunting for DNS tunneling, and the Log4j2 Outbreak Alert.
Skills: event handling, incident response workflow, threat hunting, threat intelligence.

*Lab 04 (Indicators, enrichment with FortiGuard/VirusTotal, and an automation stitch that bans an IP on FortiGate) was reviewed from the lab guide only, because the lab access period ended before it could be run.*
