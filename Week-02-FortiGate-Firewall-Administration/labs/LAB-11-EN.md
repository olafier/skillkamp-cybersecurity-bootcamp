# 🔬 LAB 11 Summary: IPsec VPN — Dial-up vs Static (FortiGate)

> 🇹🇭 ภาษาไทย: [LAB-11.md](LAB-11.md)
> Note: screenshots live in `images/lab11/`

## LAB 11 overview

**Goal:** connect the head office (HQ) and a branch (BR1) across the internet with an encrypted IPsec tunnel, in two styles — **Dial-up** (Exercise 1) and **Static** (Exercise 2). Result: both Exercises passed, ping goes through the tunnel with ttl=62, 0% packet loss.

The header picture is: HQ-PC-1 (10.0.11.50) → HQ-NGFW-1 (100.65.0.101) → **IPsec tunnel** → BR1-FGT (100.65.1.111) → BR1-PC-1 (172.20.1.60)

![LAB 11 overview (infographic)](images/lab11/LAB11_infographic.png)

Simple analogy: a VPN is a "secret pipe" running over a public road (the internet), letting two offices talk as if on one LAN; outsiders see only the pipe, not what's inside.

### Abbreviations used in this lab

| Abbr. | Full name | Short meaning |
| --- | --- | --- |
| VPN | Virtual Private Network | a private network running over a public one |
| IPsec | Internet Protocol Security | a set of protocols that encrypt IP packets |
| IKE | Internet Key Exchange | the protocol that negotiates keys and authenticates (Phase 1) |
| SA | Security Association | the "contract" on how to encrypt (Phase 2 creates the IPsec SA) |
| PSK | Pre-Shared Key | a password set identically on both sides (here: fortinet) |
| NAT | Network Address Translation | source-IP translation — must be **off** on a VPN policy |
| HQ | Headquarters | head office |
| BR1 | Branch 1 | branch site 1 |
| TTL | Time To Live | a hop counter decremented by 1 at each router |

### Three pieces you always need

1. **Phase 1 (IKE):** agree on how to talk and authenticate — PSK, IKE version, Mode (Main/Aggressive)
2. **Phase 2 (IPsec SA):** agree which subnets go through the tunnel (local/remote selectors)
3. **Route + Firewall Policy:** the road that sends traffic into the tunnel, plus two policies (out/in) with NAT off

### How Dial-up and Static differ

|  | Exercise 1: Dial-up | Exercise 2: Static |
| --- | --- | --- |
| Does HQ know the branch IP? | No (Remote gateway = Dialup User) | Yes (100.65.1.111) |
| Matched by | Peer ID **BR1-FGT** (requires Aggressive mode) | the gateway's IP |
| HQ Phase 2 | Remote 0.0.0.0/0 | Remote 172.20.1.0/24 |
| Route on HQ | FortiGate adds it automatically | must be created manually |
| Who brings the tunnel up | the branch only | either side |

Easy mnemonic: **Dial-up** = HQ opens a door and waits for the branch to dial in · **Static** = both sides know each other's number

## Exercise 1: IPsec Dial-up (step by step)

Task: HQ **doesn't know** the branch's IP, so set it as a Dialup User and match using a **Peer ID** instead.

### Step 1 — Create Phase 1 on HQ-NGFW-1

- VPN > VPN Tunnels > Create New > **Custom**
- Name **ToBranch** · Remote gateway **Dialup User** · Interface **port2**
- Pre-shared key **fortinet** · IKE Version **1** · Mode **Aggressive**
- Accepted peer ID: **Specific peer ID = BR1-FGT**

Why Aggressive? Because the Peer ID is sent in the first step; it can only be sent before encryption in Aggressive mode. In Main mode, HQ wouldn't know who dialed in.

![Fig 1: ToBranch Phase 1 — Dialup User, Aggressive, Peer ID BR1-FGT](images/lab11/01_hq_phase1_aggressive.png)

### Step 2 — Phase 2 on HQ

- Local address **10.0.11.0/24**
- Remote address **0.0.0.0/0** (leave the default)

0.0.0.0/0 means it accepts any branch subnet; adding a new branch later can reuse the same tunnel.

![Fig 2: Phase 2 selector on HQ](images/lab11/02_hq_phase2_selector.png)

### Step 3 — The ToBranch tunnel is created

Its status is still **Inactive**, which is normal, because HQ is waiting for the branch to dial in.

![Fig 3: ToBranch status Inactive](images/lab11/03_hq_tobranch_inactive.png)

### Step 4 — Two firewall policies on HQ

- **Branch_out:** port4 (LAN) → ToBranch
- **Branch_in:** ToBranch → port4
- Service ALL · **NAT off** on both

Why NAT off? Both sides must see each other's real IPs; translating them makes traffic mismatch the Phase 2 selector.

![Fig 4: Branch_out policy](images/lab11/04_hq_branch_out.png)

![Fig 5: Branch_in policy](images/lab11/05_hq_branch_in.png)

### Step 5 — Phase 1 on BR1-FGT (branch side)

- Name **ToHQ** · Remote gateway **Static IP 100.65.0.101** · Interface port2
- PSK **fortinet** · IKE Version 1 · Mode **Aggressive**
- **Local ID = BR1-FGT** (must match the Peer ID HQ is waiting for)

![Fig 6: ToHQ Network section](images/lab11/06_br1_tohq_network.png)

![Fig 7: Authentication section and Local ID](images/lab11/07_br1_tohq_auth.png)

### Step 6 — Phase 2 on BR1-FGT

- Local **172.20.1.0/24** · Remote **10.0.11.0/24**
- The branch knows its destination for sure, so it specifies the subnet directly

![Fig 8: Phase 2 on the branch side](images/lab11/08_br1_phase2.png)

### Step 7 — Static route on the branch

- Destination **10.0.11.0/24** → Interface **ToHQ**
- The dialing side must have its own road into the tunnel (unlike HQ, which gets its route automatically)

![Fig 9: static route creation form](images/lab11/09_br1_route_form.png)

![Fig 10: route list after creation](images/lab11/10_br1_routes.png)

### Step 8 — Two policies on the branch

- **HQ_out:** port4 → ToHQ · **HQ_in:** ToHQ → port4 · NAT Disabled on both

![Fig 11: policies on BR1-FGT](images/lab11/11_br1_policies.png)

### Step 9 — Bring Up the tunnel (start from the branch)

- Dashboard > Network > IPsec
- Right-click ToHQ > **Bring Up > All Phase 2 Selectors**
- Phase 1 and Phase 2 turn into green arrows = the tunnel is up

![Fig 12: Bring Up menu](images/lab11/12_bringup_menu.png)

![Fig 13: tunnel up, green arrows](images/lab11/13_tunnel_up.png)

### Step 10 — Test with ping

- Open **HQ-PC-1's Terminal** (not FortiGate's CLI) and type `ping 172.20.1.60`
- Replies in ~1 ms, **ttl=62** → 64 minus 2 = passed 2 hops (HQ-NGFW-1 and BR1-FGT)
- The IPsec page shows Incoming/Outgoing equal (6.72 kB)

![Fig 14: ping from HQ-PC-1 to BR1-PC-1 succeeds](images/lab11/14_ping_hq_to_br1.png)

![Fig 15: data volume through the tunnel](images/lab11/15_ipsec_data.png)

### Step 11 — Verify Exercise 1

- Take quiz > Verify > Execute = **PASSED**
- Must be done **before** the Revert in Exercise 2, because Revert deletes Exercise 1's config

![Fig 16: Verify Exercise 1 passed](images/lab11/16_verify_ex1.png)

## Exercise 2: IPsec Static (step by step)

Task: now HQ **knows** the branch IP (100.65.1.111), so build a Static tunnel; the branch side changes nothing.

### Step 1 — Revert HQ to the starting point

- Top-right admin > Configuration > **Revisions**
- Pick **15 · HQ-static-ipsec** > Revert > OK (careful not to pick 14, which is dial-up)
- The box reboots for ~1–2 minutes, then log in again

![Fig 17: Revisions page, select revision 15](images/lab11/17_revert_static.png)

### Step 2 — Static Phase 1 on HQ

- Name ToBranch · Remote gateway **Static IP Address = 100.65.1.111** · port2
- PSK fortinet · Version 1 · Aggressive (to match the branch already set)
- Accepted peer ID: **Any**, because HQ knows the IP and needn't check Peer ID

![Fig 18: Static-IP Phase 1](images/lab11/18_static_tunnel.png)

### Step 3 — Static Phase 2

- Local **10.0.11.0/24** · Remote **172.20.1.0/24** (don't use 0.0.0.0/0)
- The values must be "swapped" with the branch exactly: our local = their remote

![Fig 19: Phase 2 specifying the branch subnet directly](images/lab11/19_static_phase2.png)

### Step 4 — Static route on HQ

- Destination **172.20.1.0/24** → Interface **ToBranch**
- Static has no automatic route like Dial-up; create it yourself

![Fig 20: static route to the branch](images/lab11/20_static_route.png)

### Step 5 — Two policies on HQ

- Branch_out / Branch_in as in Exercise 1 · NAT off
- Revert deleted the old policies, so recreate them

![Fig 21: two policies on HQ](images/lab11/21_static_policies.png)

### Step 6 — Bring Up from the HQ side

- Right-click ToBranch > Bring Up
- Static can start from either side, because both know each other's IP

![Fig 22: Bring Up from HQ](images/lab11/22_static_bringup.png)

![Fig 23: Static tunnel is up](images/lab11/23_static_up.png)

### Step 7 — Ping, then Verify

- HQ-PC-1's Terminal: `ping 172.20.1.60` replies fully, ttl=62
- Verify Exercise 2 = **PASSED**

![Fig 24: ping through the Static tunnel](images/lab11/24_static_ping.png)

![Fig 25: Verify Exercise 2 passed](images/lab11/25_verify_ex2.png)

## Problems encountered

| Symptom | Cause | Fix |
| --- | --- | --- |
| Tunnel won't come up | Mode set to Main | Change to **Aggressive** on both sides |
| BR1-PC-1 won't open | that lab machine isn't enabled | ping from HQ-PC-1 toward it instead |
| `ping` in CLI shows Unknown action 0 | typed ping in FortiGate's CLI | use the PC Terminal (the `$` prompt) or `execute ping` |
| Verify Ex2 = FAILED | pressed before finishing | complete all steps, then press again |

![Fig 26: example of a wrong Main-mode setting that keeps the tunnel down](images/lab11/26_mistake_main_mode.png)

## Quiz review

- **Does only Bring Up make a tunnel come up?** → **False** — traffic matching a Phase 2 selector can open the tunnel by itself
- **What must match on both sides?** → **Pre-shared key** and **Phase 2 local/remote** (swapped)

## Cloud comparison (the field we're aiming at)

| FortiGate | AWS | Azure |
| --- | --- | --- |
| IPsec site-to-site VPN | Site-to-Site VPN + Virtual Private Gateway (VGW) or Transit Gateway (TGW) | VPN Gateway (S2S) |
| Phase 1/2, PSK | Tunnel options (IKEv1/v2, PSK) | Connection + IPsec policy |
| the office-side device | Customer Gateway (CGW) | Local Network Gateway |
| static route into the tunnel | Static routes or BGP (Border Gateway Protocol) | Address space in the Local Network Gateway or BGP |

In real work the office FortiGate often acts as a Customer Gateway, building a tunnel into AWS/Azure with the same Phase 1/2 values as this lab.

## Short summary

- IPsec = Phase 1 (authenticate) + Phase 2 (select subnets) + Route + a Policy with NAT off
- Dial-up is used when HQ doesn't know the branch IP; match by Peer ID + Aggressive mode; the branch initiates
- Static is used when both IPs are known; create the route yourself, and either side can initiate
- Verify results with a ping from a PC (look for ttl=62) and the Incoming/Outgoing on the IPsec page
