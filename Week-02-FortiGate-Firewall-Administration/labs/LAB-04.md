# 🔬 LAB 04 สรุป: Routing (Failover และ ECMP)

> หมายเหตุ: รูปภาพประกอบอยู่ใน `images/lab04/`

![ภาพรวม LAB 04 (infographic)](images/lab04/LAB04_infographic.png)

## ภาพรวม

ผ่านครบ 2/2 Exercise ใช้เวลาประมาณ 50 นาที HQ-NGFW-1 มีสายอินเทอร์เน็ต 2 สาย และแล็บให้เลือกว่าจะใช้แบบสำรอง (Failover) หรือใช้พร้อมกัน (ECMP)

| สาย | Interface | Gateway |
| --- | --- | --- |
| หลัก | port2 (100.65.0.101) | 100.65.0.254 |
| สำรอง | port3 (100.66.0.101) | 100.66.0.254 |
| ฝั่งภายใน | port4 (10.0.11.254) | HQ-PC-1 = 10.0.11.50 |

**หลักการเลือก route (จำตามลำดับ)**

1. เจาะจงที่สุดชนะ (longest match)
2. **Distance** ต่ำกว่าชนะ เส้นที่แพ้ไม่อยู่ใน table (standby)
3. Distance เท่ากัน → **Priority** ต่ำกว่าถูกใช้ แต่อยู่ใน table ทั้งคู่
4. เท่ากันทุกค่า → **ECMP** แบ่งโหลดทุกเส้น

จุดเริ่มต้น: มี default route เส้นเดียวผ่าน port2 Distance 10

![จุดเริ่มต้น: routing monitor มี default route ผ่าน port2](images/lab04/01_routing_monitor_start.png)

## Exercise 1: Route Failover ทีละขั้น

เป้าหมาย: ใช้ port2 เป็นหลัก และให้ port3 ทำงานเฉพาะตอน port2 ล่ม

### ขั้น 1: เปิดคอลัมน์ Distance และ Priority

Network > Static Routes คลิกขวาหัวตาราง ติ๊ก Distance และ Priority แล้ว Apply (ค่าเริ่มต้นของ static route = Distance 10, Priority 1)

![ขั้น 1: เลือกคอลัมน์ Distance/Priority](images/lab04/02_select_columns.png)

### ขั้น 2: สร้าง default route สำรอง

Create New: Gateway `100.66.0.254`, Interface **port3**, Distance **20**, Advanced Options > Priority **5**

![ขั้น 2: ฟอร์มสร้าง route สำรอง (port3)](images/lab04/03_new_route_port3_form.png)

![ขั้น 2: static route ตาราง (10 และ 20)](images/lab04/04_static_routes_10_20.png)

พลาดที่เจอ: พิมพ์ gateway เป็น `10.66.0.254` (ขาดเลข 0) และใส่เลข 5 ในช่อง Comments ระบบเตือน gateway unreachable ต้อง Edit แก้

![ขั้น 2: error gateway unreachable (พิมพ์ผิด)](images/lab04/23_mistake_gateway.png)

### ขั้น 3: เตรียม Firewall Policy

แก้ policy Internet ให้ Log = All sessions แล้วสร้าง `Backup_Internet` (port4 → port3, HQ_SUBNET → all, ALL, NAT, Log All sessions) เพราะ routing เลือกขาออกก่อนแล้วค่อยหา policy ถ้าไม่มี policy สำหรับ port3 ตอน failover จะโดน Implicit Deny

![ขั้น 3: policy Internet = All sessions](images/lab04/05_internet_policy_allsessions.png)

![ขั้น 3: ฟอร์ม Backup_Internet (port4 → port3)](images/lab04/06_backup_internet_form.png)

![ขั้น 3: policy list มี Backup_Internet](images/lab04/07_policy_list_backup.png)

### ขั้น 4: ดู routing table ใน CLI

`get router info routing-table all` เห็น default route แค่ port2 ส่วน `routing-table database` เห็น port3 `[20/0]` แต่ไม่มี `*>` = standby

![ขั้น 4: routing-table all (port2 เท่านั้น)](images/lab04/08_rt_all_port2_only.png)

![ขั้น 4: routing-table database (port3 standby)](images/lab04/09_rt_database_standby.png)

### ขั้น 5: ยืนยันว่าใช้ port2

FortiAnalyzer > Fortinet Logs > FortiGate > Traffic เพิ่มคอลัมน์ Destination Interface และ filter Application = HTTP

![ขั้น 5: log HTTP ออก port2](images/lab04/10_faz_port2.png)

### ขั้น 6: จำลองสายหลักล่ม

Network > Interfaces > port2 > Status = Disabled (ห้ามแตะ port4 เพราะใช้จัดการ GUI)

![ขั้น 6: port2 ถูก Disabled](images/lab04/11_port2_disabled.png)

![ขั้น 6: หน้า Interfaces port2 เป็นสีเทา](images/lab04/12_interfaces_port2_grey.png)

### ขั้น 7: Failover ทำงาน

route ของ port2 กลายเป็น inactive และหายไป port3 (Distance 20) ขึ้นมาแทน เข้าเว็บใน HQ-PC-1 (ไม่ใช่ Chrome บน Mac) แล้ว log เป็น port3

![ขั้น 7: routing เปลี่ยนไป port3 (failover)](images/lab04/13_routing_port3_failover.png)

![ขั้น 7: log ออก port3 ใน FortiAnalyzer](images/lab04/14_faz_port3.png)

### ขั้น 8: คืนค่าและ Verify

เปิด port2 กลับ (Enabled) route กลับเป็น port2 แล้วทำ quiz + Verify ทันทีก่อนไป Exercise 2

![ขั้น 8: routing กลับมา port2](images/lab04/15_routing_back_port2.png)

![ขั้น 8: Verify Ex1 PASSED](images/lab04/16_verify_ex1.png)

## Exercise 2: ECMP ทีละขั้น

เป้าหมาย: ใช้ port2 และ port3 พร้อมกัน โดยทำให้ทุกค่าเท่ากัน

### ขั้น 1: ตั้ง Distance ของ port3 = 10

แก้ route port3 ให้ Distance 10 แต่ปล่อย Priority 5 ไว้ก่อน

![ขั้น 1: ตั้ง Distance port3 = 10](images/lab04/17_port3_distance10.png)

### ขั้น 2: เปลี่ยน ECMP mode และดู table

```txt
config system settings
set v4-ecmp-mode source-dest-ip-based
end
get router info routing-table all
```

ค่าเริ่มต้น source-ip-based จะส่ง client เดียวออกเส้นเดิมตลอด จึงเปลี่ยนเป็น source+destination ผล: ทั้ง 2 เส้น**อยู่ใน table** แต่ `[1/0]` กับ `[5/0]` ต่างกัน จึงใช้แค่ port2

![ขั้น 2: ECMP mode แล้ว priority ยังต่างกัน](images/lab04/18_ecmp_mode_priority_diff.png)

### ขั้น 3: ตั้ง Priority ของ port3 = 1

![ขั้น 3: static routes เท่ากันทั้งสองเส้น](images/lab04/19_static_routes_equal.png)

![ขั้น 3: routing table มีทั้ง 2 เส้น (ECMP)](images/lab04/20_rt_ecmp_both.png)

### ขั้น 4: ดักดู packet

```txt
diagnose sniff packet any 'tcp[tcpflags] & tcp-syn !=0 and tcp[tcpflags] & tcp-ack == 0 and port 80' 4
```

ทุกบรรทัดออก port2 เพราะหน้า neverssl รีเฟรชตัวเองไปปลายทางเดิม (151.101.193.91) ECMP คำนวณ hash จาก IP ต้นทาง+ปลายทาง ปลายทางเดิมจึงออกเส้นเดิมเสมอ บน Mac หยุด sniffer ด้วย Control+C (ไม่ใช่ Command)

![ขั้น 4: sniffer เห็น SYN ออก port2](images/lab04/21_sniffer.png)

### ขั้น 5: Verify

สคริปต์ตรวจจาก config จึงผ่านแม้ sniffer ไม่เห็น port3

![ขั้น 5: Verify Ex2 PASSED](images/lab04/22_verify_ex2.png)

## Quiz, เทียบ Cloud และตัวย่อ

| Exercise | คำถาม | คำตอบ |
| --- | --- | --- |
| 1 | 2 route ไปปลายทางเดียวกัน เส้นไหนอยู่ใน table | เส้นที่ **distance ต่ำกว่า** |
| 2 | แบ่งทราฟฟิกเท่ากันต้องตั้งอะไร (2 ข้อ) | **distance + priority เท่ากัน** และ **firewall policy รองรับทั้ง 2 ทาง** |

| FortiGate | AWS / Azure |
| --- | --- |
| Failover (distance ต่างกัน) | Route 53 failover routing / Traffic Manager priority |
| ECMP | Transit Gateway ECMP บน VPN หลายเส้น / Load Balancer |
| routing table (longest match) | VPC / VNet route table (longest prefix match) |

| ตัวย่อ | ชื่อเต็ม |
| --- | --- |
| ECMP | Equal-Cost Multi-Path |
| RIB / FIB | Routing Information Base / Forwarding Information Base |
| SYN / ACK | Synchronize / Acknowledge (TCP flags) |
| AD | Administrative Distance |
| FAZ | FortiAnalyzer |
| ISDB | Internet Service Database |
| RPF | Reverse Path Forwarding |
