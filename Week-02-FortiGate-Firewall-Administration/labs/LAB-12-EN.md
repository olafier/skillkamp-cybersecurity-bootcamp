# 🔬 LAB 12 Summary: SD-WAN

> 🇹🇭 ภาษาไทย: [LAB-12.md](LAB-12.md)
> Note: screenshots live in `images/lab12/`

Make HQ-NGFW-1 send Bloomberg and CNN out port2 first, while social media and games go out port3 only.

![LAB 12 overview (infographic)](images/lab12/LAB12_infographic.png)

## Core idea

SD-WAN (Software-Defined Wide Area Network) picks an internet link by **application**, not just destination IP like ECMP in LAB 04, and continuously checks each link's health; if the primary link fails it moves traffic for you.

| Rule | Apps | Link used |
| --- | --- | --- |
| Critical-DIA | Bloomberg, CNN | port2 first, port3 if it fails |
| Non-Critical-DIA | Addicting.Games, Social.Media (Facebook, LinkedIn) | port3 only |
| Implicit (matches no rule) | others, e.g. DNS, general web | split by Source-Destination IP |

![Fig 1: Topology — port2 (100.65.0.254) and port3 (100.66.0.254) are the two internet links; HQ-PC-1 (10.0.11.50) sits behind port4](images/lab12/00_topology.png)

**Abbreviations used in the lab**

| Abbr. | Full name | Meaning in this lab |
| --- | --- | --- |
| SD-WAN | Software-Defined Wide Area Network | use several links at once and pick by application |
| DIA | Direct Internet Access | go to the internet directly from HQ, no detour |
| SLA | Service Level Agreement | link-health criteria (packet loss, latency, jitter) |
| ECMP | Equal-Cost Multi-Path | split traffic across links by IP (LAB 04) |
| ISDB | Internet Service Database | a database of service IPs, e.g. CNN-Web |
| ADVPN | Auto-Discovery VPN | overlay tunnels auto-built between branches |
| FAZ | FortiAnalyzer | collect and search logs |

## Exercise 1: Configure SD-WAN, step by step

### Step 1: Delete the Internet policy

Policy & Objects → Firewall Policy → expand port4 → port2 → select `Internet (1)` → Delete. You must **delete**, not disable, because a disabled policy still counts as referencing port2, and a referenced port can't be added to SD-WAN. Don't touch `To-FAZ` (port4 → port6); it sends logs to FortiAnalyzer.

![Fig: policy list before deleting](images/lab12/01_policy_list_before.png)

![Fig 2: Internet (1) policy in the port4 → port2 group before deletion](images/lab12/02_internet_policy.png)

![Fig 3: after deletion only To-FAZ and Implicit remain](images/lab12/03_internet_deleted.png)

### Step 2: Create the Underlay zone

Network → SD-WAN → SD-WAN Zones → Create new → SD-WAN Zone → Name `Underlay` → OK. Leave Interface members empty; they'll be added via SD-WAN Members next.

![Fig: SD-WAN Zones page, still empty](images/lab12/04_sdwan_zones_empty.png)

![Fig 4: naming the zone (first typed wrong as "Underly"; fixed before OK)](images/lab12/05_zone_typo_underly.png)

![Fig: corrected name to Underlay](images/lab12/06_zone_underlay.png)

### Step 3: Add members port2 and port3

Create new → SD-WAN Member, one at a time. Result: the Underlay zone has 2 SD-WAN Members and the icon turns green.

| Interface | Zone | Gateway | Cost | Priority |
| --- | --- | --- | --- | --- |
| port2 | Underlay | 100.65.0.254 | 0 | 1 |
| port3 | Underlay | 100.66.0.254 | 0 | 1 |

![Fig 5: member port2, gateway 100.65.0.254](images/lab12/07_member_port2.png)

![Fig 6: member port3, gateway 100.66.0.254](images/lab12/08_member_port3.png)

![Fig 7: SD-WAN Zones page shows 2 SD-WAN Members](images/lab12/09_zones_2_members.png)

### Step 4: Performance SLA `Level3_DNS`

Performance SLAs tab → Create new → Probe mode **Active**, Protocol **Ping**, Servers `4.2.2.1` + `4.2.2.2`, Participants **Specify** port2 + port3; leave other defaults (Check interval 500 ms, Failures before inactive 5, Restore link after 5). After creating it shows a grey ?; wait 10–20 seconds for the first probe result.

![Fig 8: Performance SLA form](images/lab12/10_sla_form.png)

![Fig 9: Level3_DNS in the list (still ? because just created)](images/lab12/11_sla_list.png)

### Step 5: SD-WAN Rule `Critical-DIA`

SD-WAN Rules tab → Create new → Source address `HQ_SUBNET`, Destination → Internet service `CNN-Web` + Application `Bloomberg`, Interface selection strategy **Manual**, Interface preference **port2 then port3** (order matters: the first is the primary link).

![Fig 10: Critical-DIA Source/Destination section](images/lab12/12_rule1_top.png)

![Fig 11: Critical-DIA Outgoing Interfaces — Manual + port2, port3](images/lab12/13_rule1_outgoing.png)

### Step 6: SD-WAN Rule `Non-Critical-DIA`

Source `HQ_SUBNET`, Application picks two things: the Apps tab → `Addicting.Games`, and the Category tab → `Social.Media` (the folder icon = the whole category). Interface preference **port3 only**.

![Fig 12: Non-Critical-DIA Source/Destination section](images/lab12/14_rule2_top.png)

![Fig 13: Non-Critical-DIA Outgoing Interfaces — port3](images/lab12/15_rule2_outgoing.png)

![Fig: SD-WAN Rules list after creating the two rules](images/lab12/16_rules_list.png)

### Step 7: Change the Implicit rule to Source-Destination IP

Double-click `sd-wan` in the Implicit group → choose Source-Destination IP → OK. Traffic matching no rule is split by source+destination IP pair — the same pair always leaves the same link. After OK the table still shows Source IP for a while; refresh to see the new value.

![Fig 14: choose Source-Destination IP](images/lab12/17_implicit_srcdst.png)

![Fig 15: final SD-WAN Rules table — port2 has a black tick = the chosen link, and Implicit is Source-Destination IP](images/lab12/18_rules_final.png)

### Step 8: Static route to Underlay

Network → Static Routes → Create New → Destination `0.0.0.0/0.0.0.0`, Interface **Underlay**. No gateway needed, because FortiGate uses each member's own gateway, and it creates default routes for port2 and port3 at equal distance.

![Fig 16: static route form 0.0.0.0/0 → Underlay](images/lab12/19_route_form.png)

![Fig 17: static route list (the old 172.20.1.0/24 left in place)](images/lab12/20_route_list.png)

### Step 9: Firewall policy `DIA`

| Field | Value |
| --- | --- |
| Incoming → Outgoing | port4 → Underlay |
| Source → Destination | HQ_SUBNET → all |
| Service / Action | ALL / ACCEPT |
| NAT | on (going to the internet) |
| Application control | on → default |
| SSL inspection | certificate-inspection |
| Log allowed traffic | All sessions |

**Why enable Application control:** it's what tells FortiGate which traffic is Bloomberg or Facebook; without it, app-based rules won't match.

![Fig 18: DIA policy form, upper part](images/lab12/21_dia_top.png)

![Fig 19: Security Profiles and Logging section](images/lab12/22_dia_profiles.png)

![Fig 20: port4 → Underlay group has DIA (3)](images/lab12/23_dia_list.png)

### Step 10: Quiz + Verify

Quiz: what is an underlay SD-WAN member? → **Physical** (underlay = real links, overlay = IPsec/ADVPN tunnels). Then Verify → Execute = **PASSED**.

![Fig: Exercise 1 quiz](images/lab12/24_quiz1.png)

![Fig 21: Verify Exercise 1 PASSED](images/lab12/25_verify_ex1.png)

## Exercise 2: Monitoring

On HQ-PC-1 run `cd Desktop` and `sh traffic-generation.sh`, then open linkedin.com, facebook.com, cnn.com; watch the Bandwidth / Volume / Sessions graphs at Network → SD-WAN → SD-WAN Zones and the Hit Count on the SD-WAN Rules tab. This Exercise has no separate Verify; answering the quiz correctly passes it.

Per the Guide, in FortiAnalyzer you'll see CNN matching Critical-DIA out port2, while Facebook and LinkedIn match Non-Critical-DIA out port3; other traffic like DNS has no rule name because it falls to the implicit rule.

Quiz: where do you see each link's status? → **the SD-WAN widget on the Dashboard** (the Forward Traffic log is per-session; the Rules page only shows the chosen link).

![Fig: Exercise 2 quiz](images/lab12/26_quiz2.png)

![Fig 22: open LinkedIn on HQ-PC-1 to generate Social.Media traffic](images/lab12/27_linkedin.png)

![Fig 23: Exercise 2 quiz Correct! and the Outline all green](images/lab12/28_ex2_done.png)

## Problems encountered

| Symptom | Cause | Fix |
| --- | --- | --- |
| Zone name is "Underly" | dropped the letter a | fix to Underlay before OK (Verify checks the name) |
| Implicit still shows Source IP | the table hasn't refreshed | click the SD-WAN Rules tab again |
| Performance SLA shows ? | no first probe result yet | wait 10–20 seconds |
| rule 2's port3 not ticked yet | no route to Underlay yet | create the static route and it appears |

## Cloud comparison

| FortiGate SD-WAN | AWS | Azure |
| --- | --- | --- |
| SD-WAN zone + members (several links) | Cloud WAN segments / multiple Site-to-Site VPNs | Virtual WAN hub |
| Performance SLA (ping link checks) | Route 53 health check, VPN tunnel monitoring | Connection Monitor (Network Watcher) |
| SD-WAN rule by application | Global Accelerator / Transit Gateway route table by destination | Virtual WAN routing intent |
| Implicit rule (Source-Destination IP) | ECMP on Transit Gateway | ECMP on active-active VPN Gateway |
| DIA (direct internet) | NAT Gateway / Internet Gateway on a VPC | NAT Gateway on a VNet |

Many companies place a FortiGate-VM in a VPC or VNet and use SD-WAN to connect branches directly into the cloud.
