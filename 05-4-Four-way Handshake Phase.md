# เฟสย่อยที่ 4: Four-way Handshake Phase (การตกลงคีย์ความปลอดภัยและรับ IP Address)

หลังจากผ่านขั้นตอนการผูกสัมพันธ์ (Association) แล้ว ESP32 จะเข้าสู่เฟสที่สำคัญที่สุดในด้านความปลอดภัย คือ **Four-way Handshake Phase** ซึ่งใช้โปรโตคอล **EAPOL (Extensible Authentication Protocol over LAN)** ตามมาตรฐาน WPA2/WPA3 Personal (PSK) เพื่อพิสูจน์ทราบว่าทั้ง ESP32 และ AP รู้รหัสผ่าน (Pre-Shared Key - PSK) เดียวกัน และทำการสร้างคีย์สำหรับเข้ารหัสข้อมูลในอากาศโดยที่**ไม่มีการส่งรหัสผ่านจริงผ่านคลื่นวิทยุ**

---

## 1. แนวคิดเรื่อง Encryption Keys ใน 4-Way Handshake

* **PMK (Pairwise Master Key)**: คีย์หลักที่คำนวณจาก `Passphrase` (รหัสผ่าน Wi-Fi) และ `SSID` ผ่านอัลกอริทึม PBKDF2
* **ANonce & SNonce**: ค่าสุ่มตัวเลขโดด (Random Number) ที่สร้างโดย AP (ANonce) และ Station ESP32 (SNonce)
* **PTK (Pairwise Transient Key)**: คีย์เข้ารหัสแบบ unicast ที่ใช้จริงในการเข้ารหัสข้อมูลระหว่าง ESP32 กับ AP (สร้างจากการผสม PMK + ANonce + SNonce + MAC Addresses)
* **GTK (Group Temporal Key)**: คีย์เข้ารหัสแบบ broadcast/multicast สำหรับรับข้อมูลที่ AP กระจายหาทุก Client

---

## 2. ลำดับการแลกเปลี่ยนแพ็กเกจ 4-Way Handshake (Sequence Diagram)

```mermaid
sequenceDiagram
    autonumber
    participant STA as ESP32 (Station)
    participant AP as Access Point (Router)

    note over STA, AP: ทั้งคู่คำนวณ PMK จาก รหัสผ่าน + SSID

    rect rgb(240, 255, 240)
        note over AP: สร้าง ANonce
        AP->>STA: 1/4 EAPOL-Key Frame (ส่ง ANonce มาให้ ESP32)
        note over STA: รับ 1/4 EAPOL -> สร้าง SNonce -> คำนวณสร้าง PTK
    end

    rect rgb(255, 250, 240)
        note over STA: สร้าง MIC (Message Integrity Code) จาก PTK
        STA->>AP: 2/4 EAPOL-Key Frame (ส่ง SNonce + MIC กลับไป)
        note over AP: รับ 2/4 EAPOL -> คำนวณ PTK -> ตรวจสอบ MIC<br/>*(ถ้ารหัสผ่านผิด MIC จะไม่ตรงกันที่จุดนี้)*
    end

    rect rgb(240, 248, 255)
        AP->>STA: 3/4 EAPOL-Key Frame (ส่ง GTK + Confirm MIC)
        note over STA: ตรวจสอบ MIC และติดตั้ง PTK / GTK เข้าสู่ Wi-Fi Hardware
    end

    rect rgb(255, 240, 245)
        STA->>AP: 4/4 EAPOL-Key Frame (ส่ง Acknowledgement ยืนยันติดตั้ง Key)
    end

    note over STA: เกิด Event: WIFI_EVENT_STA_CONNECTED<br/>(Link Layer Secure Connection Complete!)
    
    rect rgb(230, 230, 250)
        note over STA, AP: เข้าสู่ Phase 5: IP Assignment (DHCP Request/ACK)
        STA->>AP: DHCP Discover / Request
        AP-->>STA: DHCP Offer / ACK (มอบหมาย IP Address, Netmask, Gateway)
        note over STA: เกิด Event: IP_EVENT_STA_GOT_IP<br/>(พร้อมรับส่งข้อมูล Internet!)
    end
```

---

## 3. รายละเอียดขั้นตอนทางเทคนิค (Step Breakdown)

* **s4.1 (1/4 EAPOL)**: เริ่มกระบวนการ Handshake AP จะส่งแพ็กเกจ **1/4 EAPOL** ซึ่งบรรจุค่า `ANonce` มายัง ESP32 หาก Handshake Timer เกิด Timeout และไม่ได้รับแพ็กเกจ 1/4 EAPOL:
  * เกิด Event: `WIFI_EVENT_STA_DISCONNECTED`
  * Result Code: **`WIFI_REASON_HANDSHAKE_TIMEOUT`** (15)
* **s4.2**: ESP32 ได้รับแพ็กเกจ **1/4 EAPOL** สำเร็จ นำ ANonce มาคำนวณสร้าง PTK ร่วมกับ SNonce ของตนเอง
* **s4.3 (2/4 EAPOL)**: ESP32 ตอบกลับแพ็กเกจ **2/4 EAPOL** โดยแนบค่า `SNonce` และ `MIC` (รหัสยืนยันความถูกต้อง) กลับไปให้ AP
  * **จุดสังเกต**: หากป้อนรหัสผ่าน Wi-Fi ผิด ค่า PMK ของ ESP32 จะไม่ตรงกับ AP ทำให้ MIC ที่สร้างในขั้นตอนนี้ผิดพลาด AP จะปฏิเสธ และเกิด Disconnect
* **s4.4 (3/4 EAPOL)**: ESP32 รอรับแพ็กเกจ **3/4 EAPOL** จาก AP หากไม่ได้รับตามเวลาที่กำหนด:
  * เกิด Event: `WIFI_EVENT_STA_DISCONNECTED`
  * Result Code: **`WIFI_REASON_HANDSHAKE_TIMEOUT`** (15) หรือ **`WIFI_REASON_4WAY_HANDSHAKE_TIMEOUT`** (204)
* **s4.5**: ESP32 ได้รับแพ็กเกจ **3/4 EAPOL** ซึ่งบรรจุคีย์ `GTK` และคำยืนยันจาก AP
* **s4.6 (4/4 EAPOL)**: ESP32 ส่งแพ็กเกจ **4/4 EAPOL** เพื่อยืนยันว่าได้ติดตั้งคีย์เข้ารหัสในระดับฮาร์ดแวร์เรียบร้อยแล้ว
* **s4.7 (Link Connected)**: เกิด Wi-Fi Event **`WIFI_EVENT_STA_CONNECTED`** แสดงว่าระดับ Link Layer ปลอดภัยและเชื่อมต่อสำเร็จ

> **หลังจากสเต็ป s4.7**: ESP32 จะเริ่มขอหมายเลข IP Address จาก DHCP Server บน Router เมื่อได้รับ IP แล้ว จะเกิด Event **`IP_EVENT_STA_GOT_IP`** ซึ่งเป็นจุดสิ้นสุดอย่างสมบูรณ์ และทำให้ ESP32 สามารถใช้งาน Wi-Fi Station รับส่งข้อมูลเครือข่ายได้!

---

## 4. ตารางรวมสรุป Disconnect Reason Codes ทั้งหมด (Wi-Fi Diagnostic Table)

ตารางนี้เป็นคู่มืออ้างอิงสำคัญสำหรับนักศึกษาในการ Debug เมื่อ Wi-Fi เกิดปัญหากลางทาง:

| Reason Constant | รหัส | เฟสที่เกิด | สาเหตุหลักและการวิเคราะห์ปัญหา |
| :--- | :--- | :--- | :--- |
| `WIFI_REASON_UNSPECIFIED` | 1 | ทุกเฟส | ข้อผิดพลาดที่ไม่ระบุสาเหตุชัดเจน |
| `WIFI_REASON_AUTH_EXPIRE` | 2 | Phase 2 (Auth) | สัญญาณอ่อนมาก ร้องขอ Auth แล้วไม่มีการตอบรับ |
| `WIFI_REASON_AUTH_FAIL` | 1 / 202 | Phase 2 / Phase 4 | AP ปฏิเสธการ Auth หรือใส่รหัสผ่านผิดอย่างรุนแรง |
| `WIFI_REASON_ASSOC_EXPIRE` | 4 | Phase 3 (Assoc) | หมดเวลาการร้องขอ Association |
| `WIFI_REASON_ASSOC_FAIL` | 3 | Phase 3 (Assoc) | AP ปฏิเสธ Association |
| `WIFI_REASON_NOT_AUTHED` | 6 | Phase 3 | พยายามส่ง Assoc ทั้งที่ยัง Auth ไม่ผ่าน |
| `WIFI_REASON_HANDSHAKE_TIMEOUT` | 15 / 204 | Phase 4 (Handshake) | **พิมพ์รหัสผ่าน Wi-Fi (Password) ผิด** หรือสัญญาณขาดหายขณะแลกเปลี่ยน EAPOL |
| `WIFI_REASON_NO_AP_FOUND` | 201 | Phase 1 (Scan) | **พิมพ์ชื่อ Wi-Fi (SSID) ผิด** หรือ AP ปิดอยู่ / เป็น 5GHz |
| `WIFI_REASON_CONNECTION_FAIL` | 208 | Phase 4/DHCP | การสถาปนาการเชื่อมต่อในระดับ Driver ล้มเหลว |

---

## 5. คำถามทบทวนความเข้าใจ (Checkpoints)

1. ในกระบวนการ 4-Way Handshake เหตุใดจึงไม่มีการส่งรหัสผ่าน (Password) ลอยไปในอากาศเลยแม้แต่แพ็กเกจเดียว?
```
เพราะ ESP32 และ Router จะใช้รหัสผ่านคำนวณ PMK และคีย์ต่าง ๆ เพื่อยืนยันว่ารู้รหัสเดียวกัน โดยส่งแค่ค่าที่ใช้ในการคำนวณและตรวจสอบ ไม่ได้ส่ง Password จริง ๆ
```
2. เหตุใดการพิมพ์รหัสผ่าน Wi-Fi ผิดใน ESP32 จึงมักหลุดที่ขั้นตอน **2/4 EAPOL** หรือ **3/4 EAPOL** ไม่ใช่ตั้งแต่เฟส 1 หรือ 2?
```
เพราะตอนแรกยังไม่ได้ตรวจรหัสผ่านจริง ๆ แต่เมื่อถึง 4-Way Handshake ESP32 จะสร้าง PMK/MIC จากรหัสผ่าน ถ้ารหัสผิด ค่าที่ได้จะไม่ตรงกับ Router ทำให้การตรวจสอบไม่ผ่านและถูกตัดการเชื่อมต่อ
```
3. ความแตกต่างสำคัญระหว่าง Event **`WIFI_EVENT_STA_CONNECTED`** กับ **`IP_EVENT_STA_GOT_IP`** คืออะไร?
```
WIFI_EVENT_STA_CONNECTED = ESP32 เชื่อมต่อกับ Router ในระดับ Wi-Fi สำเร็จแล้ว ส่วน IP_EVENT_STA_GOT_IP = ESP32 ได้รับ IP Address จาก DHCP แล้ว จึงสามารถสื่อสารบนเครือข่ายได้
```
