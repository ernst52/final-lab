# รายงานรายบุคคล — Final Lab

## 1. ข้อมูลผู้จัดทำ

- ชื่อ-นามสกุล: นายปรานต์ มีเดช
- รหัสนักศึกษา: 67543210003-9
- Section: 1
- GitHub Username: ernst52

---

## 2. ส่วนที่รับผิดชอบ

- Auth Service — ระบบ Login ด้วย Seed Users + JWT
- Task Service — REST API สำหรับ CRUD Tasks พร้อม Role-based Access
- Log Service — บันทึก Log ลง PostgreSQL ผ่าน REST API
- Nginx — ตั้งค่า HTTPS ด้วย Self-Signed Certificate และ Reverse Proxy
- Docker Compose — กำหนด Service, Network และ Environment ทั้งระบบ
- Frontend — หน้า Task Board (index.html) และ Log Dashboard (logs.html)

---

## 3. สิ่งที่ได้ลงมือพัฒนาด้วยตนเอง

- เขียน `auth-service/src/routes/auth.js` โดยใช้ฐานจาก Week 12 แล้วลบ `/register` ออก และเพิ่ม `logEvent()` helper สำหรับส่ง log ไปยัง Log Service
- เขียน `task-service/src/routes/tasks.js` พร้อม `authMiddleware.js` และ `jwtUtils.js` สำหรับตรวจสอบ JWT
- เขียน `log-service` สำหรับรับและบันทึก log ลง PostgreSQL
- ตั้งค่า `nginx/nginx.conf` ให้รองรับ HTTPS port 443 และ redirect HTTP → HTTPS
- เขียน `docker-compose.yml` เชื่อม Service ทั้งหมดเข้าด้วยกัน
- เขียน `db/init.sql` พร้อม Schema และ Seed Users ที่มี bcrypt hash จริง
- เขียน `frontend/index.html` (Task Board UI + JWT Inspector) และ `frontend/logs.html` (Log Dashboard) โดยการนำฐานจาก Week 12 มารวมร่าง

---

## 4. ปัญหาที่พบและวิธีการแก้ไข

#### `apiFetch is not defined` หน้า Task แหลวไม่มีไรโหลดเลย
- เนื่องจาก function ขาดหายไปใน index.html เลย เพิ่ม `apiFetch()`, `authHeaders()`, และ `toast()` ใน script block ซึ่งไปเอามาจาก Week12 ลืมใส่ 555
#### `loadProfile()` เรียก `/api/users/me` บ่ได้หน้า profile แหลวไม่โหลด  
- นึกได้ว่ายังไม่มี user-service เลยแก้ให้ใช้ข้อมูลจาก `currentUser` โดยตรงแทนการเรียก API 
#### bcrypt hash ใน `db/init.sql` เป็น placeholder ทำให้ login ไม่ได้ 
- รัน `bcrypt.hashSync()` ใน auth-service แล้วแทนค่าใน init.sql |
#### `node -e` รัน bcrypt บ่ได้เพราะไม่มี module 
- ต้อง `cd auth-service && npm install` ก่อนแล้วค่อยรัน เพราะ auth-service มี module อยู่แล้ว
#### Frontend พังกระจุย กดปุ่มไรไม่ได้เลย
- Format code ไม่ดีเลยกดใช้ Prettier กลับมาใช้ได้ปกติ
#### nginx ไม่โหลด
- เผลอเอา Dockerfile กับ nginx.conf ไปยัดใน certs เลยต้องเอาออกมาไว้ข้างนอก

---

## 5. สิ่งที่ได้เรียนรู้จากงานนี้

- เข้าใจการทำงานของ JWT ตั้งแต่การ Sign ใน Auth Service จนถึงการ Verify ใน Task Service โดยไม่ต้องเรียกกลับ
- เข้าใจ Timing Attack Prevention ด้วย DUMMY_BCRYPT_HASH
- เรียนรู้การใช้ Docker Compose จัดการ Multi-Service Application

---

## 6. แนวทางที่ต้องการพัฒนาต่อใน Set 2

- อย่าลืมกลับไปแก้ `async function loadProfile()` เดี้ยวจะแหลวเอา