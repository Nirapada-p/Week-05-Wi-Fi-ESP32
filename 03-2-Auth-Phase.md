# เฟสย่อยที่ 2: Auth Phase (Authentication Phase)

หลังจากที่ ESP32 สแกนพบ Access Point (AP) เป้าหมายเรียบร้อยแล้ว ESP32 จะเข้าสู่ **Auth Phase** ซึ่งเป็นกระบวนการยืนยันตัวตนระดับ Link Layer (802.11 Authentication) ก่อนที่จะเริ่มขอสิทธิ์การผูกสัมพันธ์ในระดับเครือข่าย

> [!NOTE]
> ในเครือข่าย Wi-Fi ที่ใช้ความปลอดภัยแบบ WPA2/WPA3 ขั้นตอนนี้จะเป็นแบบ **Open System Authentication** (ยินยอมให้ยืนยันตัวตนผ่านในระดับ 802.11 Link Layer ไปก่อน) แล้วค่อยไปตรวจสอบรหัสผ่านจริงใน **Four-way Handshake Phase (เฟส 4)**

---

## 1. ลำดับการแลกเปลี่ยนแพ็กเกจ Authentication (Sequence Diagram)

```mermaid
sequenceDiagram
    autonumber
    participant STA as ESP32 (Station)
    participant AP as Access Point (Router)

    note over STA: เริ่มต้น Auth Timer (นับเวลาถอยหลัง)
    STA->>AP: 802.11 Authentication Request (Algorithm: Open System)
    
    alt AP ตอบรับทันเวลา (Auth Success)
        AP-->>STA: 802.11 Authentication Response (Status: Success)
        note over STA: หยุด Auth Timer -> ผ่านเข้าสู่ Association Phase
    else AP ไม่ตอบกลับในเวลาที่กำหนด (Auth Timeout)
        note over STA: Auth Timer หมดเวลา (Expired)
        note over STA: ส่ง Event WIFI_EVENT_STA_DISCONNECTED<br/>(Reason: WIFI_REASON_AUTH_EXPIRE)
    else AP ปฏิเสธการ Auth (Auth Rejected)
        AP-->>STA: 802.11 Authentication Response (Status: Failure)
        note over STA: ส่ง Event WIFI_EVENT_STA_DISCONNECTED<br/>(Reason: WIFI_REASON_AUTH_FAIL)
    end
```

---

## 2. รายละเอียดขั้นตอนทางเทคนิค (Step Breakdown)

* **s2.1**: ESP32 เริ่มต้นขั้นตอนการร้องขอการยืนยันตัวตน (Authentication Request) ไปยัง AP พร้อมกันนั้น **Auth Timer** ของระบบปฏิบัติการจะเริ่มนับเวลา เพื่อควบคุมเวลาที่ใช้ในการรอคอยแพ็กเกจตอบรับ
* **s2.2 (กรณีหมดเวลา)**: หากเวลาผ่านไปจนเกินค่า Timeout ที่กำหนดไว้แล้ว แต่ยังไม่ได้รับการตอบรับจาก AP:
  * ไดรเวอร์จะปล่อย Event ชื่อ **`WIFI_EVENT_STA_DISCONNECTED`**
  * ไดรเวอร์จะแจ้ง Result Code / Disconnect Reason ออกมาเป็น: **`WIFI_REASON_AUTH_EXPIRE`** (Value 2)
* **s2.3 (กรณีสำเร็จ)**: หากได้รับแพ็กเกจตอบรับจาก AP เป็นสถานะ Success ภายในเวลาที่กำหนด **Auth Timer จะหยุดลงทันที** และสิ้นสุดกระบวนการ Authentication ในระดับ Link Layer เพื่อเข้าสู่เฟสถัดไป
* **s2.4 (กรณีถูกปฏิเสธ)**: หาก AP ส่งแพ็กเกจ Authentication Response ตอบกลับมา แต่ระบุสถานะ**ปฏิเสธการเข้าถึง (Rejection)**:
  * ไดรเวอร์จะปล่อย Event ชื่อ **`WIFI_EVENT_STA_DISCONNECTED`**
  * ไดรเวอร์จะแจ้ง Result Code / Disconnect Reason ออกมาเป็น: **`WIFI_REASON_AUTH_FAIL`** (Value 1)

---

## 3. ตารางสรุป Result Code และสาเหตุใน Auth Phase

| Reason Code Constant | รหัสตัวเลข | สื่อความหมายถึง | สาเหตุที่อาจเกิดขึ้นในทางปฏิบัติ |
| :--- | :--- | :--- | :--- |
| `WIFI_REASON_AUTH_FAIL` | 1 | AP ปฏิเสธคำขอ Authentication | MAC Address Filtering บน AP บล็อก ESP32 ไว้ หรือ AP อยู่ในสถานะ Overload |
| `WIFI_REASON_AUTH_EXPIRE` | 2 | หมดเวลาการรอตอบรับ Auth | สัญญาณ Wi-Fi อ่อนมาก แพ็กเกจหล่นหายในอากาศ (Packet Loss) |

---

## 4. คำถามทบทวนความเข้าใจ (Checkpoints)

1. ทำไมในเครือข่าย WPA2-PSK ขั้นตอน Auth Phase นี้ถึงผ่านได้แม้เราจะพิมพ์รหัสผ่านผิด?
```
เพราะช่วง Auth Phase ยังไม่ได้ตรวจสอบรหัสผ่านจริง ๆ แต่เป็นการยืนยันเบื้องต้นว่าอุปกรณ์สามารถเข้ามาขอเชื่อมต่อกับ Router ได้
```
2. หาก Router มีการเปิดใช้งาน **MAC Address Filtering** (อนุญาตเฉพาะอุปกรณ์ที่ลงทะเบียน MAC ไว้) ESP32 จะล้มเหลวที่ขั้นตอนใด และได้ Reason Code อะไร?
```
ESP32 จะล้มเหลวตั้งแต่ Auth Phase เพราะ Router ตรวจสอบ MAC Address แล้วพบว่า ESP32 ไม่ได้รับอนุญาตให้เชื่อมต่อ เลยปฏิเสธการเชื่อมต่อกลับมา โดยจะได้ Reason Code = WIFI_REASON_AUTH_FAIL (รหัส 1)
```
