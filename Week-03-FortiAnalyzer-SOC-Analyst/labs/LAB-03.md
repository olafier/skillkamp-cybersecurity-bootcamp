# 🔬 LAB 03 สรุป: Incident and Event Management

> English version: [LAB-03-EN.md](LAB-03-EN.md)
> รูปภาพประกอบอยู่ใน `images/lab03/`

## ภาพรวมแล็บ

**เป้าหมาย:** จัดการ event ตั้งแต่ต้นจนจบ ได้แก่ ให้ IPS สร้าง event จากการสแกนของ Nikto, clone handler สำเร็จรูปให้ส่งอีเมล, สร้าง handler เองเพื่อจับ login ล้มเหลว, ยก event ขึ้นเป็น incident, ล่า DNS tunneling ด้วย Threat Hunting และดู Outbreak Alert ของ Log4j2

| อุปกรณ์ | IP | หน้าที่ในแล็บ |
| --- | --- | --- |
| HQ-PC-1 | 10.0.11.50 | Terminal, PuTTY, Thunderbird |
| FortiAnalyzer | 10.0.13.125 | ADOM1 (Ex 1, 2, 4–6) · root (Ex 3) |
| HQ-NGFW-1 | FGVM02TM24013423 | IPS profile WEBSERVER_Protection (policy 5) |
| LINUX-ISP | 100.65.0.254 | ผู้โจมตี (Nikto) |

## Exercise 1: Examining and Managing Events

1. แก้ filter ของ **WEBSERVER_Protection** ให้เป็น **TGT Server · SEV Medium/High/Critical · APP Apache** จำนวน signature ลดจาก 5,655 เหลือ **315** (ต่างหมวดกัน = AND, หมวดเดียวกัน = OR)
2. รัน `nikto.pl -host 100.65.0.200` จาก LINUX-ISP
3. **Incidents & Events > Event Monitor > All Events** → คลิกขวา Event Type → `eventtype="ips"`

| Event | Status | เหตุผล |
| --- | --- | --- |
| HTTP.URI.SQL.Injection | **Mitigated** | IPS บล็อกแล้ว |
| Apache.Expect.Header.XSS | **Unhandled** | IPS แค่ตรวจพบ ต้องให้ analyst ตรวจต่อ |

แถว **+** คือกลุ่ม (by Endpoint / by Threat) ไม่ใช่ event จริง event เดียวกันจึงโผล่ได้หลายกลุ่ม และสถานะดูจาก **action ใน log** ไม่ใช่จากชื่อ event (test_botnet ชื่อว่า blocked แต่ log มี action = detected)

![filter IPS ครบ 5 รายการ เหลือ 315 signature](images/lab03/01_ex1_ips_filter_5_entries.png)

![Nikto สแกน 100.65.0.200](images/lab03/02_ex1_nikto_running.png)

![event ที่จัดกลุ่มตาม endpoint และ threat](images/lab03/03_ex1_events_grouped.png)

![View Logs: action = detected](images/lab03/04_ex1_view_logs_detected.png)

## Exercise 2: Customizing Predefined Event Handlers

1. เปิด handler จากลิงก์ในแถว event (**Default-Botnet-Communication-Detection-By-Threat**) rule ที่ match จะไฮไลต์สีเหลือง: *Traffic to Botnet CnC blocked in ips log 2* (`logid==0422016400`)
2. **Search in Log View** → Raw Log เห็น 168.10.199.186 → 224.141.85.77 ซ้ำทุกประมาณ 1 นาที (beaconing)
3. สร้าง Notification Profile `Email_Server` (admin@training.lab, mail server 100.65.0.254)
4. **Clone** handler เป็น `Simulated Botnet Attack Handler` แล้วผูก Email_Server
5. **ปิด rule ips log 2 ใน handler ต้นฉบับ** ไม่งั้น event เกิดซ้ำจาก 2 handler
6. Thunderbird ได้อีเมล *FAZ Event Notification* (Trigger = Simulated Botnet Attack Handler)

![rule ที่ match ไฮไลต์สีเหลือง](images/lab03/05_ex2_handler_rule_highlight.png)

![View Rule ของ handler สำเร็จรูป](images/lab03/06_ex2_view_rule.png)

![Raw Log ของ test_botnet](images/lab03/07_ex2_raw_log_botnet.png)

![Notification Profile Email_Server](images/lab03/08_ex2_notification_profile.png)

![event ซ้ำจาก 2 handler](images/lab03/09_ex2_duplicate_events.png)

![ปิด rule ips log 2 ในต้นฉบับ](images/lab03/10_ex2_original_rule_disabled.png)

![อีเมลแจ้งเตือน](images/lab03/11_ex2_alert_email.png)

## Exercise 3: Creating a Custom Event Handler (ADOM root)

1. login ด้วย `fake` / `fake` 3 ครั้ง จนขึ้น *Too many login failures* (ระบบกัน brute-force)
2. log ของ FortiAnalyzer เอง: *User 'fake' login failed from GUI(10.0.11.50)* ระดับ alert
3. สร้าง handler `Detect failed login attempts` → rule `Failed_Login`:
   - Log Device Type **FortiAnalyzer** · Log Type **Event Log**
   - Log Field **Source End User (enduser)**
   - Log Filters Level = **Alert** (ค่าเริ่มต้น Emergency จะไม่ match)
   - Log Filter by Text `msg~"login failed"`
   - Event Conditions **≥ 3 ครั้งภายใน 5 นาที**
4. login ผิดอีก 3 ครั้ง → เกิด event Count 3

![หน้า login ถูกล็อก](images/lab03/12_ex3_login_lockout.png)

![รายละเอียด log login failed](images/lab03/13_ex3_failed_login_detail.png)

![Event Conditions ≥ 3 ใน 5 นาที](images/lab03/14_ex3_event_conditions.png)

![event จาก handler ที่สร้างเอง](images/lab03/15_ex3_custom_handler_event.png)

## Exercise 4: Managing Incidents

1. คลิกขวา event ของ Simulated Botnet Attack Handler → **Create New Incident** → Category **Unauthorized Access**, Assigned To **admin** → ได้ **IN00000001**
2. **Analysis** → Status **Analysis** → **Update** แล้วค่อย POST comment
3. เปลี่ยน Status เป็น **Closed: False Positive**
4. **Audit History** บันทึก *status from draft to analysis* (สถานะ New เก็บภายในเป็น draft) และการแนบ comment

![สร้าง incident จาก event](images/lab03/16_ex4_create_incident.png)

![Audit History](images/lab03/17_ex4_audit_history.png)

## Exercise 5: Exploring Threat Hunting

1. รันคำสั่ง `hunting` ใน Terminal ของ HQ-PC-1 ทิ้งไว้ 3–5 นาที
2. **Log View > Threat Hunting** → **Last 5 Minutes**
3. **Application Service:** DNS ขึ้นอันดับ 1
4. **Destination Domain:** แถวที่**ไม่มีชื่อโดเมน** Count และ Threat Score สูงสุด (โดเมนสุ่ม)
5. ดับเบิลคลิก DNS → log **10.0.11.50 → 8.8.8.8:53** สลับ notice / *Connection Failed*
6. รายละเอียด log: Event Action **dns**, Policy ID **1**, Data Source **HQ-NGFW-1**
7. กด Ctrl+C หยุด script

**ข้อสรุป:** 10.0.11.50 ถูกเจาะ · DNS request จำนวนมากกับโดเมนสุ่ม = **DNS tunneling** · HQ-NGFW-1 ปล่อยผ่านบางส่วน · ใช้ 8.8.8.8 ไม่ใช่ DNS ขององค์กร

![DNS ขึ้นอันดับ 1 ใน Application Service](images/lab03/18_ex5_dns_top_service.png)

![Destination Domain: แถวไม่มีชื่อโดเมน](images/lab03/19_ex5_destination_domain.png)

![log DNS ไป 8.8.8.8](images/lab03/20_ex5_dns_logs.png)

![รายละเอียด log: Policy 1, HQ-NGFW-1](images/lab03/21_ex5_log_detail.png)

## Exercise 6: Exploring the Outbreak Detection Service

1. **Incidents & Events > Outbreak Alerts** (141 รายการจาก FortiGuard) → ค้น `Log4j` → **Apache Log4j2 Vulnerability**
2. Overview: CVE-2021-44228 และอีก 4 CVE · CVSS **10.0** · Known Exploited **Yes** · EPSS 100%
3. Threat Intelligence: IOC **3,237** รายการ (โดเมน, Active)
4. **Reports > Report Definitions > Chart Library** ค้น `Log4j` → 3 charts (FGT IPS, FGT Vuln IPS, FWB WAF)
5. **All Reports** → *Outbreak Alert - Log4j2 Vulnerability Report* → แท็บ **Editor** มีครบทั้ง 3 charts

![Log4j2: Threat Radar](images/lab03/22_ex6_log4j_threat_radar.png)

![IOC list](images/lab03/23_ex6_ioc_list.png)

![Chart Library: Log4j](images/lab03/24_ex6_chart_library.png)

![Report Editor: 3 charts](images/lab03/25_ex6_report_editor.png)

## ปัญหาที่เจอและวิธีแก้

- พิมพ์ค้นในช่องค้นรายการ signature แทน**แผง Select Entries** → ต้องกด **+** ในกรอบ Filter ก่อน
- ค้น `TGT` ไม่เจอ → ต้องค้นด้วยค่า เช่น `Server`
- Clone handler แล้วลืมปิด rule ในต้นฉบับ → event ซ้ำ
- POST comment ก่อนกด Update → Status กลับเป็นค่าเดิม
- Ex 5 ข้ามขั้นรัน `hunting` → ไม่มีข้อมูลให้ล่า ไปเจอ botnet และ policy violation ที่ไม่ใช่คำตอบแทน

## บทเรียน

- **Unhandled** ต้องได้รับความสนใจก่อน **Mitigated**
- Predefined handler แก้ชื่อหรือ rule ไม่ได้ ต้อง **Clone** ก่อน · อีเมลส่ง 1 ฉบับต่อ 1 event
- log ภายในของ FortiAnalyzer อยู่ใน **root ADOM**
- Incident: Category, Assign, Comment, Status และ Audit History
- Threat Hunting = SIEM Analytics table + pivot (Add to Hunting / View Logs) เริ่มจากสมมติฐานแล้วไล่หาหลักฐาน
- Outbreak Detection Service ส่ง alert, handler, chart และ report มาให้ ถ้าจะแก้ควร Clone ก่อน
