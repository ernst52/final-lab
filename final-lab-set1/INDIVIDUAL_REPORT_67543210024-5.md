# Individual report

## ข้อมูลผู้จัดทำ
- ชื่อ-นามสกุล: นายสิริ รัตนรินทร์
- รหัสนักศึกษา: 67543210024-5
- สาขา: วิศวกรรมศาสตร์ ซอฟต์แวร์ ปี2 Sec2

## ส่วนที่รับผิดชอบ
- ทดสอบ Test cases ทั้งหมด (T1-T11)
- Screenshots ภาพผลลัพธ์ทั้งหมดตามแต่ละ testcase
- จัดทำ README.md และช่วยทำ TEAM_SPLIT.md

## สิ่งที่ได้ลงมือพัฒนาด้วยตนเอง
- ใช้ Postman ในการทดสอบแต่ละ testcases 
- สร้าง hash ของบัญชีทดสอบเอง สำหรับการนำโค้ดไปทดสอบต่อไป

## ปัญหาที่พบและวิธีการแก้ไข
- ปัญหา: เรื่องการสร้าง SSL certificates (cert.pem/key.pem) สำหรับการทำ testcases เนื่องจากรูปแบบของคำสั่ง

```bash
chmod +x scripts/gen-certs.sh
./scripts/gen-certs.sh
```
- วิธีแก้: เปลี่ยนจากการทำงานบน Window มาเป็นบน Ubuntu แทน

## สิ่งที่ได้เรียนรู้จากงานนี้
- Self-Signed Certificate และ HTTPS: ตอนเปิด https://localhost ครั้งแรกใน Chrome พบหน้า
"Your connection is not private" ทำให้เข้าใจว่า self-signed certificate คือ certificate ที่สร้างเองโดยไม่ได้รับรองจาก Certificate Authority จริง เหมาะสำหรับ development เท่านั้น

- JWT Authentication: ทดสอบ GET /api/tasks/ ใน Postman โดยไม่ใส่ Authorization header
ได้รับ 401 Unauthorized พร้อม response body: {"error":"Unauthorized: No token provided"}
ทำให้เข้าใจว่า JWT middleware ทำงานเป็น "ประตู" ที่ block request ก่อนถึง business logic จริง ถ้าไม่มี token หรือ token ผิดจะไม่ผ่านเลย

- Rate Limiting: ทดสอบส่ง login ผิดติดกันเร็วๆ Postman จะตอบกลับ {"error":"Too Many Requests","retryAfter":60} ทำให้เข้าใจว่า rate limiting ป้องกัน brute-force attack โดยจำกัดจำนวนครั้งที่ IP หนึ่งสามารถ login ผิดได้ต่อนาที

- Role-Based Access Control: ทดสอบ login เป็น alice (member) แล้วเรียก GET /api/logs/ ใน Postman ได้รับ 403 Forbidden พร้อม {"error":"Forbidden: admin only"} แต่เมื่อ login เป็น admin แล้วเรียก endpoint เดิม ได้รับ 200 พร้อม log entries ทำให้เข้าใจความแตกต่างมากขึ้นระหว่าง Authentication (พิสูจน์ตัวตน) และ Authorization (ตรวจสิทธิ์) ซึ่งเป็นคนละขั้นตอนกัน

## แนวทางที่ต้องการพัฒนาต่อใน Set 2
มีส่วนร่วมในการทำส่วนโค้ดมากขึ้น 