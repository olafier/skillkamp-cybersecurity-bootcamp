## สัปดาห์ที่ 04 — Digital Forensics & Blue Team CTF

> English version: [README.md](README.md)

**วันที่:** 22 พฤศจิกายน 2026
**แพลตฟอร์ม:** Cyber Lab 2026 (CTFd) · Kali Linux · เครื่องมือ forensic บน Windows 11
**หลักสูตร:** Workshop 1 — Digital Forensics (RPCA Cyber Club, คณะนิติวิทยาศาสตร์ โรงเรียนนายร้อยตำรวจ)

### ภาพรวม

สัปดาห์ที่สี่ขยับจากการเฝ้าดูเครือข่าย มาเป็นการตรวจสอบหลักฐานที่ถูกทิ้งไว้ในระบบ workshop สอนพื้นฐาน Digital Forensics ตั้งแต่การค้นหา เก็บรักษา วิเคราะห์ และนำเสนอพยานหลักฐานดิจิทัลให้ใช้ได้ตามกฎหมาย แล้วนำแนวคิดเหล่านั้นไปลองแก้โจทย์ CTF จริง

ภาคปฏิบัติแบ่งตาม 4 สาขาหลัก (File, Disk, Memory, Network) บวก Steganography โดยใช้เครื่องมือฟรีและ open-source บน Kali Linux ส่วน FTK Imager, Autopsy และ Wireshark รันบน Windows 11 ARM64 VM บนเครื่อง Apple Silicon

### หัวข้อที่เรียน

#### 1. พื้นฐาน Digital Forensics

- กระบวนการ 5 ขั้นพร้อม Chain of Custody: Identification → Collection → Preservation → Analysis → Presentation
- ระดับการดึงข้อมูล (Acquisition): Manual, Logical, Physical (bit-by-bit)
- หลักการเก็บหลักฐาน: อุปกรณ์ที่ตกน้ำให้เก็บมาทั้งน้ำ, assume ว่าอุปกรณ์พังแล้วเพื่อรักษาสภาพ, ห้ามอัปโหลดหลักฐานจริงขึ้นเว็บออนไลน์

#### 2. File Forensics

- File signature (magic number): JPG `FF D8 … FF D9`, PNG `89 50 4E 47 … IEND`, PDF `%PDF … %%EOF`, ZIP/Office `50 4B 03 04 … 50 4B 05 06`, EXE `MZ`, ELF `7F 45 4C 46`
- ไฟล์ Office (docx/xlsx/pptx) คือไฟล์ ZIP — แตกออกมาดู XML ข้างในได้
- การอ่าน metadata, การ carve ไฟล์, และการหาข้อมูลที่ซ่อนต่อท้าย trailer

```bash
file suspect.jpg            # เช็กชนิดจริงจาก magic number ไม่ใช่นามสกุล
binwalk suspect.jpg         # หาไฟล์ที่ฝังอยู่
foremost -i suspect.jpg -o out/   # carve จาก header/trailer
exiftool suspect.jpg        # metadata, GPS, thumbnail ที่ฝังอยู่
strings suspect.jpg | grep -i flag
```

#### 3. Disk Forensics

- สื่อเก็บข้อมูล non-volatile vs RAM ที่ volatile; order of volatility
- Forensic image เป็นแบบ bit-by-bit → ไฟล์ image ใหญ่เท่าดิสก์ต้นฉบับ
- Artifacts (browser history, search terms, recent files) อยู่ในดิสก์ — VPN ลบไม่ได้

เครื่องมือ: FTK Imager, Autopsy (GUI ของ The Sleuth Kit), `dd`, Guymager

#### 4. Memory Forensics

- วิเคราะห์ RAM หา process ที่รันอยู่, malware และรหัสผ่านที่ค้างในหน่วยความจำ

```bash
# Volatility 2 — ต้องหา profile ก่อน
volatility -f dump.raw imageinfo
volatility -f dump.raw --profile=Win10x64 pslist
volatility -f dump.raw --profile=Win10x64 memdump -p <PID> -D ./

# Volatility 3 — ไม่ต้องใช้ profile
vol -f dump.raw windows.pslist
vol -f dump.raw windows.cmdline
```

ตัวอย่างคลาสสิก (Google CTF For1): `pslist` เจอ `mspaint.exe` → `memdump` ดึง process ออกมา → เปลี่ยน `.dmp` เป็น `.data` → เปิดใน GIMP เป็นภาพดิบ → เจอ flag ที่ถูกวาดค้างไว้ใน Paint

#### 5. Network Forensics

- วิเคราะห์ไฟล์ `.pcap` / `.pcapng`
- Flow ใน Wireshark: Protocol Hierarchy → Display Filter → Follow Stream → Export Objects
- HTTP / FTP / Telnet อ่านได้ (plain text), TLS เข้ารหัสอ่านไม่ได้

```text
http.request.method == "POST"
ip.addr == 10.0.11.50
frame contains "password"
```

เครื่องมือ: Wireshark, TShark, tcpdump, NetworkMiner

#### 6. Steganography

- ซ่อน "การมีอยู่" ของข้อมูลในไฟล์ปกติ ไม่ใช่แค่ซ่อนเนื้อหา
- ภาพ: Steghide, Stegseek, zsteg (LSB ของ PNG), Stegsolve
- เสียง: เปิดใน Audacity / Sonic Visualiser แล้วเปลี่ยนเป็น **Spectrogram** จะเห็นข้อความซ่อนในคลื่นความถี่

### หมายเหตุเรื่องเครื่องมือ (Apple Silicon)

| สาขา | เครื่องมือ | รันบน |
| --- | --- | --- |
| File | Hex Fiend, binwalk, foremost, exiftool, strings | macOS / Kali |
| Disk | FTK Imager, Autopsy 4.23.1 | Windows 11 ARM64 VM (x64 ผ่าน emulation) |
| Memory | Volatility | Kali |
| Network | Wireshark 4.6.10 (arm64 native), tshark | Windows VM / Kali |
| Stego | Audacity | Windows VM / Kali |

> บทเรียน: งาน CTF Forensics ทำบน Kali Linux เสถียรที่สุด เพราะเครื่องมือ CLI เป็น ARM native ไม่ต้อง emulate ส่วน HxD เป็นของ Windows เท่านั้น — macOS ใช้ Hex Fiend แทน

### โจทย์ CTF ที่แก้ได้

| โจทย์ | หมวด | เทคนิค | เครื่องมือ |
| --- | --- | --- | --- |
| Hidden Text | File | ข้อความฝังแบบ plain byte | `strings \| grep` |
| Comment | File | docx คือ ZIP — ดู XML ข้างใน | `binwalk -e` / `unzip` + `grep` |
| The Sound of Flower | Stego | ภาพซ่อนใน spectrogram ของเสียง | Audacity (Spectrogram) |

---

### แล็บภาคปฏิบัติ

write-up แบบละเอียดทีละขั้นอยู่ใน [`labs/`](labs/)

**[Lab 01 — File Forensics & CTF Practice](labs/LAB-01.md)** ([English](labs/LAB-01-EN.md))
ติดตั้งชุดเครื่องมือข้าม macOS, Kali และ Windows 11 ARM VM แล้วลองแก้โจทย์ File, Memory, Network และ Stego บนแพลตฟอร์ม Cyber Lab 2026 — ตั้งแต่ระบุชนิดไฟล์, carve ภาพที่ฝัง, ดู XML ของ docx ไปจนถึงอ่านข้อความซ่อนจาก spectrogram ของเสียง
ทักษะ: file signature analysis, carving, metadata extraction, steganography, วิธีคิดแบบ CTF
