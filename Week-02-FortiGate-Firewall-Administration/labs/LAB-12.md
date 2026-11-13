# 🔬 LAB 12 สรุป: SD-WAN

> 🇬🇧 English version: [LAB-12-EN.md](LAB-12-EN.md)
> รูปภาพประกอบอยู่ใน `images/lab12/`

ให้ HQ-NGFW-1 ส่ง Bloomberg และ CNN ออก port2 ก่อน ส่วนโซเชียลมีเดียกับเกมออก port3 อย่างเดียว

![ภาพรวม LAB 12 (infographic)](images/lab12/LAB12_infographic.png)

## ไอเดียหลัก

SD-WAN (Software-Defined Wide Area Network) เลือกสายอินเทอร์เน็ตตาม**แอป** ไม่ใช่แค่ IP ปลายทางแบบ ECMP ใน LAB 04 และคอยเช็กสุขภาพของสายตลอดเวลา ถ้าสายหลักล่มจะย้ายทราฟฟิกให้เอง

| Rule | แอป | สายที่ใช้ |
| --- | --- | --- |
| Critical-DIA | Bloomberg, CNN | port2 ก่อน ถ้าล่มใช้ port3 |
| Non-Critical-DIA | Addicting.Games, Social.Media (Facebook, LinkedIn) | port3 เท่านั้น |
| Implicit (ไม่ตรง rule ไหน) | อื่นๆ เช่น DNS, เว็บทั่วไป | แบ่งตาม Source-Destination IP |

![ภาพ 1: Topology — port2 (100.65.0.254) และ port3 (100.66.0.254) เป็นสายอินเทอร์เน็ต 2 สาย ส่วน HQ-PC-1 (10.0.11.50) อยู่หลัง port4](images/lab12/00_topology.png)

**คำย่อที่ใช้ใน lab**

| คำย่อ | ชื่อเต็ม | ความหมายใน lab นี้ |
| --- | --- | --- |
| SD-WAN | Software-Defined Wide Area Network | ใช้หลายสายพร้อมกันและเลือกสายตามแอป |
| DIA | Direct Internet Access | ออกอินเทอร์เน็ตตรงจาก HQ ไม่ต้องอ้อม |
| SLA | Service Level Agreement | เกณฑ์สุขภาพสาย (packet loss, latency, jitter) |
| ECMP | Equal-Cost Multi-Path | แบ่งทราฟฟิกหลายสายตาม IP (LAB 04) |
| ISDB | Internet Service Database | ฐานข้อมูล IP ของบริการ เช่น CNN-Web |
| ADVPN | Auto-Discovery VPN | อุโมงค์ overlay ที่สร้างเองระหว่างสาขา |
| FAZ | FortiAnalyzer | เก็บและค้น log |

## Exercise 1: ตั้งค่า SD-WAN ทีละขั้น

### ขั้น 1: ลบ policy Internet

Policy & Objects → Firewall Policy → ขยาย port4 → port2 → เลือก `Internet (1)` → Delete ต้อง**ลบ** ไม่ใช่ Disable เพราะ policy ที่ปิดไว้ยังนับว่าอ้างอิง port2 และ port ที่ถูกอ้างอิงเข้า SD-WAN ไม่ได้ อย่าแตะ `To-FAZ` (port4 → port6) เพราะใช้ส่ง log ไป FortiAnalyzer

![ภาพ: policy list ก่อนลบ](images/lab12/01_policy_list_before.png)

![ภาพ 2: policy Internet (1) ในกลุ่ม port4 → port2 ก่อนลบ](images/lab12/02_internet_policy.png)

![ภาพ 3: ลบแล้วเหลือแค่ To-FAZ กับ Implicit](images/lab12/03_internet_deleted.png)

### ขั้น 2: สร้าง zone Underlay

Network → SD-WAN → SD-WAN Zones → Create new → SD-WAN Zone → Name `Underlay` → OK ช่อง Interface members ปล่อยว่าง เพราะจะเพิ่มผ่าน SD-WAN Member ต่อไป

![ภาพ: หน้า SD-WAN Zones ยังว่าง](images/lab12/04_sdwan_zones_empty.png)

![ภาพ 4: ตั้งชื่อ zone (รอบแรกพิมพ์ผิดเป็น Underly แก้ก่อนกด OK)](images/lab12/05_zone_typo_underly.png)

![ภาพ: แก้ชื่อเป็น Underlay ถูกต้อง](images/lab12/06_zone_underlay.png)

### ขั้น 3: เพิ่มสมาชิก port2 และ port3

Create new → SD-WAN Member ทีละตัว ผลคือ zone Underlay มี 2 SD-WAN Members และไอคอนเปลี่ยนเป็นสีเขียว

| Interface | Zone | Gateway | Cost | Priority |
| --- | --- | --- | --- | --- |
| port2 | Underlay | 100.65.0.254 | 0 | 1 |
| port3 | Underlay | 100.66.0.254 | 0 | 1 |

![ภาพ 5: สมาชิก port2 gateway 100.65.0.254](images/lab12/07_member_port2.png)

![ภาพ 6: สมาชิก port3 gateway 100.66.0.254](images/lab12/08_member_port3.png)

![ภาพ 7: หน้า SD-WAN Zones ขึ้น 2 SD-WAN Members](images/lab12/09_zones_2_members.png)

### ขั้น 4: Performance SLA `Level3_DNS`

แท็บ Performance SLAs → Create new → Probe mode **Active**, Protocol **Ping**, Servers `4.2.2.1` + `4.2.2.2`, Participants **Specify** port2 + port3 ค่าอื่นเป็นค่าเริ่มต้น (Check interval 500 ms, Failures before inactive 5, Restore link after 5) ตอนสร้างเสร็จจะขึ้น ? สีเทา รอ 10–20 วินาทีจนได้ผลตรวจครั้งแรก

![ภาพ 8: ฟอร์ม Performance SLA](images/lab12/10_sla_form.png)

![ภาพ 9: Level3_DNS ในรายการ (ยังเป็น ? เพราะเพิ่งสร้าง)](images/lab12/11_sla_list.png)

### ขั้น 5: SD-WAN Rule `Critical-DIA`

แท็บ SD-WAN Rules → Create new → Source address `HQ_SUBNET`, Destination → Internet service `CNN-Web` + Application `Bloomberg`, Interface selection strategy **Manual**, Interface preference **port2 แล้ว port3** (ลำดับสำคัญ: ตัวแรกคือสายหลัก)

![ภาพ 10: Critical-DIA ส่วน Source/Destination](images/lab12/12_rule1_top.png)

![ภาพ 11: Critical-DIA ส่วน Outgoing Interfaces เลือก Manual + port2, port3](images/lab12/13_rule1_outgoing.png)

### ขั้น 6: SD-WAN Rule `Non-Critical-DIA`

Source `HQ_SUBNET`, Application เลือก 2 อย่าง: แท็บ Apps → `Addicting.Games` และแท็บ Category → `Social.Media` (ไอคอนโฟลเดอร์ = ทั้งหมวด) Interface preference **port3 เท่านั้น**

![ภาพ 12: Non-Critical-DIA ส่วน Source/Destination](images/lab12/14_rule2_top.png)

![ภาพ 13: Non-Critical-DIA ส่วน Outgoing Interfaces เลือก port3](images/lab12/15_rule2_outgoing.png)

![ภาพ: รายการ SD-WAN Rules หลังสร้าง 2 rule](images/lab12/16_rules_list.png)

### ขั้น 7: แก้ Implicit rule เป็น Source-Destination IP

ดับเบิลคลิก `sd-wan` ในกลุ่ม Implicit → เลือก Source-Destination IP → OK ทราฟฟิกที่ไม่ตรง rule ไหนจะแบ่งตามคู่ source + destination IP คู่เดิมออกสายเดิมเสมอ หลังกด OK ตารางยังโชว์ Source IP อยู่พักหนึ่ง ต้อง refresh จึงเห็นค่าใหม่

![ภาพ 14: เลือก Source-Destination IP](images/lab12/17_implicit_srcdst.png)

![ภาพ 15: ตาราง SD-WAN Rules สุดท้าย: port2 มีติ๊กดำ = สายที่ถูกเลือก และ Implicit เป็น Source-Destination IP](images/lab12/18_rules_final.png)

### ขั้น 8: Static route ไป Underlay

Network → Static Routes → Create New → Destination `0.0.0.0/0.0.0.0`, Interface **Underlay** ไม่ต้องใส่ gateway เพราะ FortiGate ใช้ gateway ที่ตั้งในสมาชิกแต่ละตัว และสร้าง default route ให้ port2 กับ port3 ด้วย distance เท่ากัน

![ภาพ 16: ฟอร์ม static route 0.0.0.0/0 → Underlay](images/lab12/19_route_form.png)

![ภาพ 17: รายการ static route (172.20.1.0/24 เดิมปล่อยไว้)](images/lab12/20_route_list.png)

### ขั้น 9: Firewall policy `DIA`

| ช่อง | ค่า |
| --- | --- |
| Incoming → Outgoing | port4 → Underlay |
| Source → Destination | HQ_SUBNET → all |
| Service / Action | ALL / ACCEPT |
| NAT | เปิด (ออกอินเทอร์เน็ต) |
| Application control | เปิด → default |
| SSL inspection | certificate-inspection |
| Log allowed traffic | All sessions |

**ทำไมต้องเปิด Application control:** เป็นตัวที่บอก FortiGate ว่าทราฟฟิกไหนเป็น Bloomberg หรือ Facebook ถ้าไม่เปิด rule แบบแอปจะไม่ match

![ภาพ 18: ฟอร์ม policy DIA ส่วนบน](images/lab12/21_dia_top.png)

![ภาพ 19: ส่วน Security Profiles และ Logging](images/lab12/22_dia_profiles.png)

![ภาพ 20: กลุ่ม port4 → Underlay มี DIA (3)](images/lab12/23_dia_list.png)

### ขั้น 10: Quiz + Verify

Quiz: SD-WAN member แบบ underlay คืออะไร? → **Physical** (underlay = สายจริง, overlay = อุโมงค์ IPsec/ADVPN) จากนั้น Verify → Execute ผล **PASSED**

![ภาพ: quiz Exercise 1](images/lab12/24_quiz1.png)

![ภาพ 21: Verify Exercise 1 PASSED](images/lab12/25_verify_ex1.png)

## Exercise 2: Monitoring

บน HQ-PC-1 รัน `cd Desktop` และ `sh traffic-generation.sh` แล้วเปิด linkedin.com, facebook.com, cnn.com จากนั้นดูกราฟ Bandwidth / Volume / Sessions ที่ Network → SD-WAN → SD-WAN Zones และ Hit Count ในแท็บ SD-WAN Rules Exercise นี้ไม่มี Verify แยก ตอบ quiz ถูกก็ผ่าน

ตาม Guide ใน FortiAnalyzer จะเห็น CNN ตรง Critical-DIA ออก port2 ส่วน Facebook และ LinkedIn ตรง Non-Critical-DIA ออก port3 ทราฟฟิกอื่น เช่น DNS ไม่มีชื่อ rule เพราะตกไปที่ implicit rule

Quiz: ดูสถานะของแต่ละสายที่ไหน? → **SD-WAN widget บน Dashboard** (Forward Traffic log เป็นราย session, หน้า Rules บอกแค่สายที่ถูกเลือก)

![ภาพ: quiz Exercise 2](images/lab12/26_quiz2.png)

![ภาพ 22: เปิด LinkedIn บน HQ-PC-1 เพื่อสร้างทราฟฟิก Social.Media](images/lab12/27_linkedin.png)

![ภาพ 23: quiz Exercise 2 Correct! และ Outline ติ๊กเขียวครบ](images/lab12/28_ex2_done.png)

## ปัญหาที่เจอ

| อาการ | สาเหตุ | วิธีแก้ |
| --- | --- | --- |
| ชื่อ zone เป็น Underly | พิมพ์ตกตัว a | แก้เป็น Underlay ก่อนกด OK (Verify ดูชื่อ) |
| Implicit ยังขึ้น Source IP | หน้าตารางยังไม่ refresh | กดแท็บ SD-WAN Rules ใหม่ |
| Performance SLA ขึ้น ? | ยังไม่มีผลตรวจครั้งแรก | รอ 10–20 วินาที |
| port3 ของ rule 2 ยังไม่ติ๊ก | ยังไม่มี route ไป Underlay | สร้าง static route แล้วขึ้นเอง |

## เทียบกับ Cloud

| FortiGate SD-WAN | AWS | Azure |
| --- | --- | --- |
| SD-WAN zone + members (หลายสาย) | Cloud WAN segments / หลาย Site-to-Site VPN | Virtual WAN hub |
| Performance SLA (ping เช็กสาย) | Route 53 health check, VPN tunnel monitoring | Connection Monitor (Network Watcher) |
| SD-WAN rule ตามแอป | Global Accelerator / Transit Gateway route table ตามปลายทาง | Virtual WAN routing intent |
| Implicit rule (Source-Destination IP) | ECMP บน Transit Gateway | ECMP บน VPN Gateway active-active |
| DIA (ออกเน็ตตรง) | NAT Gateway / Internet Gateway ต่อ VPC | NAT Gateway ต่อ VNet |

บริษัทหลายแห่งวาง FortiGate-VM ไว้ใน VPC หรือ VNet และใช้ SD-WAN ต่อสาขาเข้า cloud โดยตรง
