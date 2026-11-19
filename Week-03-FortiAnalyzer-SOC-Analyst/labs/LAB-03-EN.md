# 🔬 LAB 03 Summary: Incident and Event Management

> Thai version: [LAB-03.md](LAB-03.md)
> Screenshots are in `images/lab03/`

## Lab Overview

**Goal:** manage events end to end: let IPS raise events from a Nikto scan, clone a predefined handler to send email alerts, build a custom handler for failed logins, escalate an event into an incident, hunt for DNS tunneling, and review the Log4j2 Outbreak Alert.

| Device | IP | Role in the lab |
| --- | --- | --- |
| HQ-PC-1 | 10.0.11.50 | Terminal, PuTTY, Thunderbird |
| FortiAnalyzer | 10.0.13.125 | ADOM1 (Ex 1, 2, 4–6) · root (Ex 3) |
| HQ-NGFW-1 | FGVM02TM24013423 | IPS profile WEBSERVER_Protection (policy 5) |
| LINUX-ISP | 100.65.0.254 | Attacker (Nikto) |

## Exercise 1: Examining and Managing Events

1. Set the **WEBSERVER_Protection** filter to **TGT Server · SEV Medium/High/Critical · APP Apache**. Signatures drop from 5,655 to **315** (different categories = AND, same category = OR).
2. Run `nikto.pl -host 100.65.0.200` from LINUX-ISP.
3. **Incidents & Events > Event Monitor > All Events** → right-click Event Type → `eventtype="ips"`.

| Event | Status | Why |
| --- | --- | --- |
| HTTP.URI.SQL.Injection | **Mitigated** | IPS blocked it |
| Apache.Expect.Header.XSS | **Unhandled** | IPS only detected it; an analyst must follow up |

Rows with **+** are groups (by endpoint / by threat), not events, so one event can appear in several groups. Status comes from the **action in the log**, not the event name (test_botnet says "blocked" in its name, but the log action is detected).

![IPS filter with 5 entries, 315 signatures](images/lab03/01_ex1_ips_filter_5_entries.png)

![Nikto scanning 100.65.0.200](images/lab03/02_ex1_nikto_running.png)

![Events grouped by endpoint and threat](images/lab03/03_ex1_events_grouped.png)

![View Logs: action = detected](images/lab03/04_ex1_view_logs_detected.png)

## Exercise 2: Customizing Predefined Event Handlers

1. Open the handler from the link in the event row (**Default-Botnet-Communication-Detection-By-Threat**). The matching rule is highlighted in yellow: *Traffic to Botnet CnC blocked in ips log 2* (`logid==0422016400`).
2. **Search in Log View** → Raw Log shows 168.10.199.186 → 224.141.85.77 repeating about every minute (beaconing).
3. Create Notification Profile `Email_Server` (admin@training.lab, mail server 100.65.0.254).
4. **Clone** the handler as `Simulated Botnet Attack Handler` and attach Email_Server.
5. **Disable rule ips log 2 in the original handler**, otherwise two handlers raise duplicate events.
6. Thunderbird receives *FAZ Event Notification* (Trigger = Simulated Botnet Attack Handler).

![Matching rule highlighted in yellow](images/lab03/05_ex2_handler_rule_highlight.png)

![View Rule on the predefined handler](images/lab03/06_ex2_view_rule.png)

![Raw Log of test_botnet](images/lab03/07_ex2_raw_log_botnet.png)

![Notification Profile Email_Server](images/lab03/08_ex2_notification_profile.png)

![Duplicate events from two handlers](images/lab03/09_ex2_duplicate_events.png)

![Rule ips log 2 disabled in the original](images/lab03/10_ex2_original_rule_disabled.png)

![Alert email](images/lab03/11_ex2_alert_email.png)

## Exercise 3: Creating a Custom Event Handler (root ADOM)

1. Log in with `fake` / `fake` three times until *Too many login failures* appears (brute-force protection).
2. FortiAnalyzer's own log: *User 'fake' login failed from GUI(10.0.11.50)*, level alert.
3. Create handler `Detect failed login attempts` → rule `Failed_Login`:
   - Log Device Type **FortiAnalyzer** · Log Type **Event Log**
   - Log Field **Source End User (enduser)**
   - Log Filters Level = **Alert** (the default Emergency never matches)
   - Log Filter by Text `msg~"login failed"`
   - Event Conditions **≥ 3 within 5 minutes**
4. Three more failed logins → an event with Count 3.

![Login page locked out](images/lab03/12_ex3_login_lockout.png)

![Failed login log detail](images/lab03/13_ex3_failed_login_detail.png)

![Event Conditions ≥ 3 in 5 minutes](images/lab03/14_ex3_event_conditions.png)

![Event from the custom handler](images/lab03/15_ex3_custom_handler_event.png)

## Exercise 4: Managing Incidents

1. Right-click the Simulated Botnet Attack Handler event → **Create New Incident** → Category **Unauthorized Access**, Assigned To **admin** → **IN00000001**.
2. **Analysis** → Status **Analysis** → **Update**, then POST a comment.
3. Change Status to **Closed: False Positive**.
4. **Audit History** records *status from draft to analysis* (New is stored internally as draft) and the attached comment.

![Creating an incident from an event](images/lab03/16_ex4_create_incident.png)

![Audit History](images/lab03/17_ex4_audit_history.png)

## Exercise 5: Exploring Threat Hunting

1. Run `hunting` in the HQ-PC-1 terminal and leave it for 3–5 minutes.
2. **Log View > Threat Hunting** → **Last 5 Minutes**.
3. **Application Service:** DNS ranks first.
4. **Destination Domain:** rows with **no domain name** have the highest count and Threat Score (random domains).
5. Double-click DNS → logs **10.0.11.50 → 8.8.8.8:53**, alternating notice / *Connection Failed*.
6. Log detail: Event Action **dns**, Policy ID **1**, Data Source **HQ-NGFW-1**.
7. Press Ctrl+C to stop the script.

**Conclusion:** 10.0.11.50 is compromised · a high volume of DNS requests to random domains = **DNS tunneling** · HQ-NGFW-1 forwards some of it · queries go to 8.8.8.8, not the corporate DNS.

![DNS ranked first in Application Service](images/lab03/18_ex5_dns_top_service.png)

![Destination Domain: rows without a domain](images/lab03/19_ex5_destination_domain.png)

![DNS logs to 8.8.8.8](images/lab03/20_ex5_dns_logs.png)

![Log detail: Policy 1, HQ-NGFW-1](images/lab03/21_ex5_log_detail.png)

## Exercise 6: Exploring the Outbreak Detection Service

1. **Incidents & Events > Outbreak Alerts** (141 from FortiGuard) → search `Log4j` → **Apache Log4j2 Vulnerability**.
2. Overview: CVE-2021-44228 plus 4 more CVEs · CVSS **10.0** · Known Exploited **Yes** · EPSS 100%.
3. Threat Intelligence: **3,237** IOCs (domains, Active).
4. **Reports > Report Definitions > Chart Library**, search `Log4j` → 3 charts (FGT IPS, FGT Vuln IPS, FWB WAF).
5. **All Reports** → *Outbreak Alert - Log4j2 Vulnerability Report* → **Editor** tab contains all 3 charts.

![Log4j2: Threat Radar](images/lab03/22_ex6_log4j_threat_radar.png)

![IOC list](images/lab03/23_ex6_ioc_list.png)

![Chart Library: Log4j](images/lab03/24_ex6_chart_library.png)

![Report Editor: 3 charts](images/lab03/25_ex6_report_editor.png)

## Issues and Fixes

- Typed into the signature-list search instead of the **Select Entries** pane → click **+** inside the Filter box first.
- Searching `TGT` found nothing → search by value, such as `Server`.
- Cloned a handler but forgot to disable the rule in the original → duplicate events.
- Posted a comment before clicking Update → Status reverted.
- Skipped running `hunting` in Ex 5 → nothing to hunt; found botnet and policy-violation traffic that was not the answer.

## Key Takeaways

- **Unhandled** events need attention before **Mitigated** ones.
- Predefined handlers cannot be renamed or have rules edited; **Clone** them first. One email is sent per event.
- FortiAnalyzer's own logs live in the **root ADOM**.
- Incidents: Category, Assign, Comment, Status and Audit History.
- Threat Hunting = SIEM Analytics table + pivoting (Add to Hunting / View Logs): start from a hypothesis and follow the evidence.
- The Outbreak Detection Service ships alerts, handlers, charts and reports; clone before customizing.
