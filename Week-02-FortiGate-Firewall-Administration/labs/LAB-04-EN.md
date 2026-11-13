# 🔬 LAB 04 Summary: Routing (Failover and ECMP)

> 🇹🇭 ภาษาไทย: [LAB-04.md](LAB-04.md)
> Note: screenshots live in `images/lab04/`

![LAB 04 overview (infographic)](images/lab04/LAB04_infographic.png)

## Overview

Passed both 2/2 Exercises in about 50 minutes. HQ-NGFW-1 has two internet links, and the lab lets you choose standby (Failover) or simultaneous use (ECMP).

| Link | Interface | Gateway |
| --- | --- | --- |
| Primary | port2 (100.65.0.101) | 100.65.0.254 |
| Backup | port3 (100.66.0.101) | 100.66.0.254 |
| Internal | port4 (10.0.11.254) | HQ-PC-1 = 10.0.11.50 |

**Route selection rules (remember in order)**

1. Most specific wins (longest match)
2. Lower **Distance** wins; the losing route isn't in the table (standby)
3. Equal Distance → lower **Priority** is used, but both stay in the table
4. Everything equal → **ECMP** load-balances across all routes

Starting point: a single default route via port2 with Distance 10

![Starting point: routing monitor has a default route via port2](images/lab04/01_routing_monitor_start.png)

## Exercise 1: Route Failover, step by step

Goal: use port2 as primary and have port3 work only when port2 is down.

### Step 1: Enable the Distance and Priority columns

Network > Static Routes, right-click the table header, tick Distance and Priority, then Apply (a static route's defaults = Distance 10, Priority 1).

![Step 1: select the Distance/Priority columns](images/lab04/02_select_columns.png)

### Step 2: Create a backup default route

Create New: Gateway `100.66.0.254`, Interface **port3**, Distance **20**, Advanced Options > Priority **5**

![Step 2: backup route (port3) creation form](images/lab04/03_new_route_port3_form.png)

![Step 2: static routes table (10 and 20)](images/lab04/04_static_routes_10_20.png)

Mistake made: typed the gateway as `10.66.0.254` (missing a 0) and put 5 in the Comments field; the system warned "gateway unreachable" and it had to be edited.

![Step 2: gateway unreachable error (typo)](images/lab04/23_mistake_gateway.png)

### Step 3: Prepare the Firewall Policy

Change the Internet policy's Log to All sessions, then create `Backup_Internet` (port4 → port3, HQ_SUBNET → all, ALL, NAT, Log All sessions), because routing picks the egress first and then finds a policy; without a policy for port3, failover traffic hits Implicit Deny.

![Step 3: Internet policy = All sessions](images/lab04/05_internet_policy_allsessions.png)

![Step 3: Backup_Internet form (port4 → port3)](images/lab04/06_backup_internet_form.png)

![Step 3: policy list with Backup_Internet](images/lab04/07_policy_list_backup.png)

### Step 4: View the routing table in CLI

`get router info routing-table all` shows the default route only via port2; `routing-table database` shows port3 `[20/0]` but without `*>` = standby.

![Step 4: routing-table all (port2 only)](images/lab04/08_rt_all_port2_only.png)

![Step 4: routing-table database (port3 standby)](images/lab04/09_rt_database_standby.png)

### Step 5: Confirm port2 is used

FortiAnalyzer > Fortinet Logs > FortiGate > Traffic, add the Destination Interface column, filter Application = HTTP.

![Step 5: HTTP log leaving via port2](images/lab04/10_faz_port2.png)

### Step 6: Simulate the primary link failing

Network > Interfaces > port2 > Status = Disabled (don't touch port4, which manages the GUI).

![Step 6: port2 Disabled](images/lab04/11_port2_disabled.png)

![Step 6: Interfaces page, port2 greyed out](images/lab04/12_interfaces_port2_grey.png)

### Step 7: Failover works

port2's route becomes inactive and disappears; port3 (Distance 20) takes over. Browse a site in HQ-PC-1 (not Chrome on the Mac) and the log shows port3.

![Step 7: routing switches to port3 (failover)](images/lab04/13_routing_port3_failover.png)

![Step 7: log leaving via port3 in FortiAnalyzer](images/lab04/14_faz_port3.png)

### Step 8: Restore and Verify

Re-enable port2 (Enabled), the route returns to port2, then do the quiz + Verify immediately before Exercise 2.

![Step 8: routing returns to port2](images/lab04/15_routing_back_port2.png)

![Step 8: Verify Ex1 PASSED](images/lab04/16_verify_ex1.png)

## Exercise 2: ECMP, step by step

Goal: use port2 and port3 simultaneously by making every value equal.

### Step 1: Set port3's Distance = 10

Edit the port3 route to Distance 10, but leave Priority 5 for now.

![Step 1: set port3 Distance = 10](images/lab04/17_port3_distance10.png)

### Step 2: Change the ECMP mode and view the table

```txt
config system settings
set v4-ecmp-mode source-dest-ip-based
end
get router info routing-table all
```

The default source-ip-based sends one client out the same link forever, so change it to source+destination. Result: both routes are **in the table**, but `[1/0]` vs `[5/0]` differ, so only port2 is used.

![Step 2: ECMP mode set, but priority still differs](images/lab04/18_ecmp_mode_priority_diff.png)

### Step 3: Set port3's Priority = 1

![Step 3: static routes equal on both links](images/lab04/19_static_routes_equal.png)

![Step 3: routing table has both routes (ECMP)](images/lab04/20_rt_ecmp_both.png)

### Step 4: Sniff packets

```txt
diagnose sniff packet any 'tcp[tcpflags] & tcp-syn !=0 and tcp[tcpflags] & tcp-ack == 0 and port 80' 4
```

Every line leaves via port2 because the neverssl page self-refreshes to the same destination (151.101.193.91); ECMP hashes on source+destination IP, so the same destination always takes the same link. On a Mac, stop the sniffer with Control+C (not Command).

![Step 4: sniffer shows SYN leaving via port2](images/lab04/21_sniffer.png)

### Step 5: Verify

The script checks the config, so it passes even though the sniffer doesn't see port3.

![Step 5: Verify Ex2 PASSED](images/lab04/22_verify_ex2.png)

## Quiz, cloud comparison and abbreviations

| Exercise | Question | Answer |
| --- | --- | --- |
| 1 | Two routes to the same destination — which is in the table? | The one with **lower distance** |
| 2 | To split traffic equally, what two things do you set? | **equal distance + priority** and **firewall policy for both paths** |

| FortiGate | AWS / Azure |
| --- | --- |
| Failover (different distance) | Route 53 failover routing / Traffic Manager priority |
| ECMP | Transit Gateway ECMP over multiple VPNs / Load Balancer |
| routing table (longest match) | VPC / VNet route table (longest prefix match) |

| Abbr. | Full name |
| --- | --- |
| ECMP | Equal-Cost Multi-Path |
| RIB / FIB | Routing Information Base / Forwarding Information Base |
| SYN / ACK | Synchronize / Acknowledge (TCP flags) |
| AD | Administrative Distance |
| FAZ | FortiAnalyzer |
| ISDB | Internet Service Database |
| RPF | Reverse Path Forwarding |
