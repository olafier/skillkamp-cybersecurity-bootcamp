## Week 03 — SOC Analyst ด้วย FortiAnalyzer

> English version: [README.md](README.md)

**วันที่:** 14–19 พฤศจิกายน 2026  
**แพลตฟอร์ม:** FortiAnalyzer 7.6.2 Analyst Virtual Lab (CIPAT SOC Analyst Lab)  
**คอร์ส:** FortiAnalyzer 7.6 Analyst (FTAO)

### ภาพรวม

สัปดาห์ที่สามเปลี่ยนจากการ**ตั้งค่า** firewall มาเป็นการ**เฝ้าดู** firewall แทนที่จะสร้าง policy บน FortiGate งานหลักย้ายไปอยู่ฝั่ง SOC (Security Operations Center) ได้แก่ รวบรวม log จาก FortiGate เข้า FortiAnalyzer, ค้นหาและสืบสวน log, เปลี่ยน log เป็น event และ incident และล่าภัยคุกคามที่ยังไม่มี rule ไหนจับได้

สภาพแวดล้อมแล็บใช้เครือข่ายองค์กรชุดเดียวกับ Week 02 (HQ-NGFW-1, HQ-ISFW, FortiGate สาขา และ ISP router) โดยมี FortiAnalyzer เป็นศูนย์กลางเก็บ log และทำงาน SOC

### หัวข้อที่เรียน

#### 1. พื้นฐาน FortiAnalyzer

- Administrative Domain (ADOM): ประเภท Fabric กับ FortiGate และปิดเป็นค่าเริ่มต้น
- โหมด Analyzer กับ Collector
- การเก็บ log ใน Security Fabric (FortiGate ตัวแรกที่เจอ session เป็นคนเขียน traffic log)
- Backup และ restore config

#### 2. เส้นทางของ log และการจัดเก็บ

- Import log จากไฟล์ backup (.log / .zst)
- ขั้นตอนของ log: รับเข้า → insert ลง SQL (Analytics) → บีบอัดเก็บ (.gz Archive)
- Data policy ของแต่ละ ADOM: ระยะเก็บ Analytics กับ Archive, การแบ่ง quota และหลัก "quota ชนะ retention"
- ตรวจอัตรา log

```bash
diagnose fortilogd lograte-device
diagnose fortilogd lograte-type
diagnose fortilogd logvol-adom all
```

#### 3. การสืบสวน log

- Real-time กับ Historical, Raw กับ Formatted
- ค้นหาแบบ Text Mode (`field=value`, wildcard, case sensitive)
- Custom View สำหรับการสืบสวนที่ทำซ้ำบ่อย
- เลือกประเภท log ให้ตรงกับ field (`catdesc` = Web Filter, `appcat` = Traffic / Application Control)

```text
appcat=P2P
appcat=*Social*
app=*Dropbox*
```

#### 4. ตรวจจับภัยด้วย FortiView

- IPS event จากการจำลองสแกนช่องโหว่เว็บ (Nikto)
- Drill-down: Top Threats → Source → Log View → log ดิบ
- เห็น Destination NAT ใน log (VIP 100.65.0.200 → 10.0.11.50, policy 5)
- จัดลำดับตาม Threat Score ไม่ใช่จำนวน session

#### 5. Event Handler

- สถานะ event: Mitigated (บล็อกแล้ว) กับ Unhandled (แค่ตรวจพบ)
- Handler สำเร็จรูป, การ Clone และการกัน event ซ้ำ
- Notification profile (แจ้งเตือนทางอีเมล)
- Handler ที่สร้างเองเพื่อจับ brute-force (login ผิด ≥ 3 ครั้งใน 5 นาที)

#### 6. การจัดการ Incident

- ยก event ขึ้นเป็น incident
- Category, ผู้รับผิดชอบ, comment และลำดับสถานะ
- Audit History เป็นบันทึกการสืบสวน

#### 7. Threat Hunting

- ล่าภัยโดยเริ่มจากสมมติฐาน: *"มีการใช้ DNS tunneling ขโมยข้อมูลหรือไม่"*
- SIEM Analytics table และการ pivot ข้าม Application Service, Destination Domain และ log
- สัญญาณ: DNS ปริมาณสูง, โดเมนสุ่ม, ใช้ DNS ภายนอก (8.8.8.8)

#### 8. Outbreak Detection Service

- FortiGuard Outbreak Alert (CVE, CVSS, known exploited, EPSS, IOC)
- Chart และ report สำเร็จรูปสำหรับภัยแต่ละเรื่อง (Apache Log4j2)

### ผลการสืบสวน

| สถานการณ์ | สิ่งที่พบ |
| --- | --- |
| ใช้ P2P | user2 ใช้ BitTorrent ผ่าน SSL VPN, policy 1 ปล่อยผ่าน |
| ใช้โซเชียลมีเดีย | Facebook และ Pinterest จาก 10.0.11.50, ปล่อยผ่าน |
| สแกนช่องโหว่เว็บ | NetworkActiv.Web.Server.XSS จาก 100.65.0.254, DNAT ไป 10.0.11.50, policy 5 |
| Botnet beaconing | 168.10.199.186 → C&C 224.141.85.77 ทุกประมาณ 1 นาที, ส่งอีเมลแจ้งเตือนแล้ว |
| Brute-force login | handler ที่สร้างเองจับ login ล้มเหลว 3 ครั้งได้ |
| DNS tunneling | 10.0.11.50 ยิง DNS query โดเมนสุ่มไปที่ 8.8.8.8 จำนวนมาก |

---

### แล็บที่ทำ

ขั้นตอนเต็มพร้อมรูปอยู่ใน [`labs/`](labs/)

**[Lab 01 — SOC Concepts and Security Fabric (Initial Configuration)](labs/LAB-01.md)** ([English](labs/LAB-01-EN.md))
Restore FortiAnalyzer จาก backup, ตรวจ ADOM1 (Fabric) และ ADOM2 (FortiGate) พร้อมอุปกรณ์ และดูสถานะเครื่องจาก CLI
ทักษะ: ตั้งค่า FortiAnalyzer, แนวคิด ADOM

**[Lab 02 — Log Data Flow and Navigating Logs](labs/LAB-02.md)** ([English](labs/LAB-02-EN.md))
Import log ย้อนหลัง, สร้าง traffic, สืบสวนการใช้ P2P และโซเชียลมีเดียใน Log View, จับการสแกนของ Nikto ด้วย IPS แล้วไล่จาก FortiView ลงไปถึง log ดิบ และตรวจอัตรา log กับพื้นที่ของ ADOM
ทักษะ: ค้นหาและสืบสวน log, FortiView, วางแผนการเก็บ log

**[Lab 03 — Incident and Event Management](labs/LAB-03.md)** ([English](labs/LAB-03-EN.md))
จัดการ event ครบวงจร: IPS event, handler ที่ clone และสร้างเองพร้อมแจ้งเตือนอีเมล, สร้างและปิด incident, ล่า DNS tunneling ด้วย Threat Hunting และ Outbreak Alert ของ Log4j2
ทักษะ: event handling, ขั้นตอน incident response, threat hunting, threat intelligence
