# ENGSE207 Software Architecture — Final Lab Set 1
## Microservices Task Board + HTTPS + Lightweight Logging

| | |
|---|---|
| **วิชา** | ENGSE207 Software Architecture |
| **งาน** | Final Lab — ชุดที่ 1 |
| **รูปแบบ** | งานกลุ่ม 2 คน |
| **สมาชิก 1** | นาย สิริ รัตนรินทร์ รหัส 67543210024-5 |
| **สมาชิก 2** | นาย ปรานต์ มีเดช รหัส 67543210003-9 |

---

## สารบัญ

1. [ภาพรวมของระบบและวัตถุประสงค์](#1-ภาพรวมของระบบและวัตถุประสงค์)
2. [Architecture Diagram](#2-architecture-diagram)
3. [โครงสร้าง Repository](#3-โครงสร้าง-repository)
4. [วิธีสร้าง Certificate และรันระบบ](#4-วิธีสร้าง-certificate-และรันระบบ)
5. [Seed Users และการสร้าง bcrypt hash](#5-seed-users-และการสร้าง-bcrypt-hash)
6. [วิธีทดสอบ API และ Frontend](#6-วิธีทดสอบ-api-และ-frontend)
7. [การทำงานของ HTTPS, JWT และ Logging](#7-การทำงานของ-https-jwt-และ-logging)
8. [Known Limitations](#8-known-limitations)

---

## 1. ภาพรวมของระบบและวัตถุประสงค์

ระบบ **Task Board Microservices** คือแอปพลิเคชันสำหรับจัดการ task ออกแบบตามสถาปัตยกรรม microservices ที่เน้นความปลอดภัย ประกอบด้วย 6 services ทำงานร่วมกันผ่าน Docker Compose

### วัตถุประสงค์ของงาน

- ฝึกตั้งค่า **HTTPS** บน Nginx ด้วย Self-Signed Certificate
- ออกแบบ **Auth Service** แบบไม่มี Register ใช้ Seed Users + JWT
- สร้าง **Lightweight Logging** เก็บลง PostgreSQL ผ่าน REST API
- เชื่อมต่อ **Frontend** กับ Backend ผ่าน HTTPS
- จัดโครงสร้าง **Microservices Repository** ให้ถูกต้อง

### คุณสมบัติหลัก

- **ไม่มีหน้า Register** — ใช้เฉพาะ Seed Users ที่กำหนดไว้ล่วงหน้า
- **HTTPS** — Nginx ใช้ Self-Signed Certificate (port 443) พร้อม redirect HTTP → HTTPS
- **JWT Authentication** — ทุก request ที่ protected ต้องแนบ Bearer token
- **Role-Based Access** — `admin` เห็น task ทุกคนและดู logs ได้, `member` เห็นเฉพาะของตัวเอง
- **Lightweight Logging** — บันทึก security events ลง PostgreSQL ผ่าน Log Service
- **Rate Limiting** — Nginx จำกัด login 5 ครั้ง/นาที ป้องกัน brute-force attack

---

## 2. Architecture Diagram

```
Browser / Postman / curl
         │
         │ HTTPS :443
         │ (HTTP :80 → 301 redirect → HTTPS)
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│   Nginx  (API Gateway + TLS Termination + Rate Limiter)             │
│                                                                     │
│   Rate Limit:  login = 5r/m  |  api = 30r/m  (per IP)              │
│                                                                     │
│   /api/auth/*         → auth-service:3001   (public)               │
│   /api/tasks/*        → task-service:3002   [JWT required]         │
│   /api/logs/internal  → 403 BLOCKED from outside                   │
│   /api/logs/*         → log-service:3003    [Admin JWT only]       │
│   /                   → frontend:80         (static HTML)          │
└───────┬──────────────────┬──────────────────┬───────────────────────┘
        │                  │                  │
        ▼                  ▼                  ▼
┌─────────────┐   ┌───────────────┐   ┌─────────────────┐
│  Auth Svc   │   │  Task Svc     │   │  Log Service    │
│   :3001     │   │   :3002       │   │   :3003         │
│             │   │               │   │                 │
│ POST /login │   │ GET    /      │   │ POST /internal  │
│ GET  /verify│   │ POST   /      │   │ GET  /          │
│ GET  /me    │   │ PUT    /:id   │   │ GET  /stats     │
│ GET  /health│   │ DELETE /:id   │   │ GET  /health    │
│             │   │ GET    /health│   │                 │
│ + seed users│   │ + JWT Guard   │   │ no JWT on POST  │
│   on startup│   │ + logEvent()  │   │ admin only GET  │
└──────┬──────┘   └──────┬────────┘   └────────────────┘
       │                 │
       └────────┬─────────┘
                ▼
     ┌─────────────────────────┐
     │   PostgreSQL :5432      │
     │   (1 shared database)   │
     │                         │
     │   users       table     │
     │   tasks       table     │
     │   logs        table     │
     │   seed_status table     │
     └─────────────────────────┘
```

### Request Flow — Login → Create Task → View Log

```
1. Browser  --HTTPS POST /api/auth/login-->  Nginx (TLS termination)
2. Nginx    --HTTP-->  auth-service:3001
3. auth-service  --SELECT WHERE email = $1-->  PostgreSQL
4. auth-service  --bcrypt.compare(password, hash)-->  valid
5. auth-service  --jwt.sign({ sub, email, role })-->  return JWT
6. auth-service  --POST /api/logs/internal-->  log-service (LOGIN_SUCCESS)

7. Browser  --HTTPS POST /api/tasks/ + Bearer token-->  Nginx
8. task-service  --jwt.verify(token, SECRET)-->  decoded payload
9. task-service  --INSERT INTO tasks-->  PostgreSQL
10. task-service  --POST /api/logs/internal-->  log-service (TASK_CREATED)

11. Admin  --HTTPS GET /api/logs/ + Admin JWT-->  log-service
12. log-service  --verifyAdminJWT (role=admin)-->  pass
13. log-service  --SELECT * FROM logs-->  return log entries
```

---

## 3. โครงสร้าง Repository

```
final-lab/
└── set1/
    ├── README.md
    ├── TEAM_SPLIT.md
    ├── INDIVIDUAL_REPORT_[studentid].md
    ├── docker-compose.yml
    ├── .env.example
    ├── .gitignore
    │
    ├── nginx/
    │   ├── nginx.conf          ← HTTPS + reverse proxy + rate limit
    │   ├── Dockerfile
    │   └── certs/              ← สร้างโดย gen-certs.sh (ไม่ commit)
    │       ├── cert.pem
    │       └── key.pem
    │
    ├── frontend/
    │   ├── Dockerfile
    │   ├── index.html          ← Task Board UI
    │   └── logs.html           ← Log Dashboard (admin only)
    │
    ├── auth-service/
    │   ├── Dockerfile
    │   ├── package.json
    │   └── src/
    │       ├── index.js
    │       ├── routes/auth.js
    │       ├── middleware/jwtUtils.js
    │       └── db/db.js
    │
    ├── task-service/
    │   ├── Dockerfile
    │   ├── package.json
    │   └── src/
    │       ├── index.js
    │       ├── routes/tasks.js
    │       ├── middleware/
    │       │   ├── authMiddleware.js
    │       │   └── jwtUtils.js
    │       └── db/db.js
    │
    ├── log-service/
    │   ├── Dockerfile
    │   ├── package.json
    │   └── src/
    │       └── index.js
    │
    ├── db/
    │   ├── init.sql            ← Schema: users, tasks, logs, seed_status
    │   └── seed.js             ← bcrypt hash generation + insert users
    │
    ├── scripts/
    │   └── gen-certs.sh        ← สร้าง self-signed certificate
    │
    └── screenshots/
        ├── 01_docker_running.png
        ├── 02_https_browser.png
        └── ...
```

---

## 4. วิธีสร้าง Certificate และรันระบบ

### ขั้นตอนที่ 1 — Clone และเข้าโฟลเดอร์

```bash
git clone <your-repo-url>
cd final-lab/set1
```

### ขั้นตอนที่ 2 — สร้างไฟล์ .env

```bash
cp .env.example .env
```

เนื้อหาใน `.env`:

```env
POSTGRES_DB=taskboard
POSTGRES_USER=admin
POSTGRES_PASSWORD=secret123
JWT_SECRET=engse207-super-secret-change-in-production-abc123
JWT_EXPIRES=1h
```

### ขั้นตอนที่ 3 — สร้าง Self-Signed Certificate

```bash
chmod +x scripts/gen-certs.sh
./scripts/gen-certs.sh
```

Script รันคำสั่ง openssl นี้:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/certs/key.pem \
  -out    nginx/certs/cert.pem \
  -subj "/C=TH/ST=Bangkok/L=Bangkok/O=RMUTL/OU=ENGSE207/CN=localhost"
```

ผลลัพธ์: `nginx/certs/cert.pem` และ `nginx/certs/key.pem`

### ขั้นตอนที่ 4 — Build และ Start

```bash
docker compose up --build
```

รอจนเห็น log:

```
auth-service  | [seed] Seeded user: alice (member)
auth-service  | [seed] Seeded user: bob (member)
auth-service  | [seed] Seeded user: admin (admin)
auth-service  | [seed] Seeding complete.
auth-service  | [auth-service] Running on :3001
task-service  | [task-service] Running on :3002
log-service   | [log-service] Running on :3003
```

### ขั้นตอนที่ 5 — เปิด Browser

เปิด **https://localhost**

> Browser จะแสดง certificate warning เพราะเป็น self-signed  
> กด **Advanced → Proceed to localhost** เพื่อเข้าใช้งาน

### Reset ฐานข้อมูล

```bash
docker compose down -v
docker compose up --build
```

---

## 5. Seed Users และการสร้าง bcrypt hash

### รายชื่อ Seed Users

| Username | Email | Password | Role |
|---|---|---|---|
| alice | alice@lab.local | `alice123` | member |
| bob | bob@lab.local | `bob456` | member |
| admin | admin@lab.local | `adminpass` | admin |

### วิธีที่เราสร้าง bcrypt hash

โปรเจกต์นี้**ไม่ได้ใช้ placeholder hash ใน SQL** เพราะ hash ที่ generate ล่วงหน้าอาจไม่ compatible กับ library version ที่รันจริง

เราสร้างไฟล์ `db/seed.js` ที่รันภายใน Node container ซึ่งมี `bcryptjs` ติดตั้งอยู่แล้ว ทำให้สร้าง hash จริงได้ตอน startup โดยอัตโนมัติ:

```javascript
// db/seed.js (ส่วนสำคัญ)
const users = [
  { username: 'alice', password: 'alice123',  role: 'member' },
  { username: 'bob',   password: 'bob456',    role: 'member' },
  { username: 'admin', password: 'adminpass', role: 'admin'  },
];

for (const u of users) {
  const hash = bcrypt.hashSync(u.password, 10);  // saltRounds = 10
  await pool.query(
    `INSERT INTO users (username, email, password_hash, role)
     VALUES ($1, $2, $3, $4)
     ON CONFLICT (username) DO UPDATE SET password_hash = EXCLUDED.password_hash`,
    [u.username, u.email, hash, u.role]
  );
}
```

`auth-service` จะเรียก seed script นี้ทุกครั้งที่ start โดยตรวจสอบ `seed_status` table ก่อน ถ้า seed แล้วจะข้ามไปไม่รันซ้ำ

### ทดสอบสร้าง hash ด้วยตนเอง

```bash
docker run --rm node:20-alpine sh -c \
  "npm install bcryptjs --quiet && node -e \"
const b = require('bcryptjs');
console.log('alice123 :', b.hashSync('alice123', 10));
console.log('bob456   :', b.hashSync('bob456',   10));
console.log('adminpass:', b.hashSync('adminpass', 10));
\""
```

---

## 6. วิธีทดสอบ API และ Frontend

### ตั้งค่า Postman ก่อนใช้งาน

ไปที่ **Settings → General → SSL certificate verification → OFF**
เพราะใช้ self-signed cert ซึ่ง Postman reject โดย default

### ทดสอบด้วย curl

```bash
BASE="https://localhost"

# T3: Login สำเร็จ
TOKEN=$(curl -sk -X POST $BASE/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@lab.local","password":"alice123"}' | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

# T4: Login ผิด password -> 401
curl -sk -X POST $BASE/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@lab.local","password":"wrongpass"}'

# T5: Create Task -> 201
curl -sk -X POST $BASE/api/tasks/ \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"Test Task","priority":"high"}'

# T6: Get Tasks -> 200
curl -sk $BASE/api/tasks/ -H "Authorization: Bearer $TOKEN"

# T7: Update Task -> 200
curl -sk -X PUT $BASE/api/tasks/1 \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status":"DONE"}'

# T8: Delete Task -> 200
curl -sk -X DELETE $BASE/api/tasks/1 -H "Authorization: Bearer $TOKEN"

# T9: ไม่มี JWT -> 401
curl -sk $BASE/api/tasks/

# T10A: Member เรียก /api/logs/ -> 403
curl -sk -i $BASE/api/logs/ -H "Authorization: Bearer $TOKEN"

# T10B: Admin เรียก /api/logs/ -> 200
ADMIN_TOKEN=$(curl -sk -X POST $BASE/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@lab.local","password":"adminpass"}' | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")
curl -sk $BASE/api/logs/ -H "Authorization: Bearer $ADMIN_TOKEN"

# T11: Rate Limit -> 429 หลัง attempt ที่ 6
for i in {1..8}; do
  echo -n "Attempt $i: "
  curl -sk -o /dev/null -w "%{http_code}\n" \
    -X POST $BASE/api/auth/login \
    -H "Content-Type: application/json" \
    -d '{"email":"alice@lab.local","password":"wrong"}'
  sleep 0.1
done
```

### ทดสอบ Frontend

1. เปิด **https://localhost** → Login ด้วย `alice@lab.local` / `alice123`
2. ทดลองสร้าง แก้ไข ลบ task
3. กด **Profile & JWT** เพื่อดู decoded JWT payload
4. กด link **Log Dashboard** (ต้อง login ใหม่เป็น admin)
5. ตรวจว่า login และ task events ถูกบันทึกครบ

---

## 7. การทำงานของ HTTPS, JWT และ Logging

### HTTPS

Nginx ทำหน้าที่ TLS Termination — รับ HTTPS จาก client แล้วส่งต่อเป็น plain HTTP ภายใน Docker network

```
Client --HTTPS:443--> Nginx --HTTP--> Services (auth:3001, task:3002, log:3003)
Client --HTTP:80---> Nginx --301 redirect--> https://...
```

Certificate ถูกสร้างด้วย openssl (RSA 2048-bit, อายุ 365 วัน) และโหลดเข้า Nginx ผ่าน Dockerfile ในขั้นตอน build

Nginx เพิ่ม security headers ทุก response:

```
Strict-Transport-Security: max-age=31536000
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
```

### JWT

auth-service ออก JWT เมื่อ login สำเร็จ โดย sign ด้วย `JWT_SECRET` จาก environment variable

Payload ที่ฝังใน token:

```json
{
  "sub": 1,
  "email": "alice@lab.local",
  "role": "member",
  "username": "alice",
  "iat": 1700000000,
  "exp": 1700003600
}
```

task-service และ log-service ตรวจ token ด้วย `jwt.verify(token, JWT_SECRET)` ใน middleware โดยไม่ต้องเรียก auth-service ซ้ำ เพราะใช้ secret เดียวกัน

การป้องกัน Timing Attack — ถ้าไม่พบ user ในระบบ auth-service ยังคง `bcrypt.compare()` กับ dummy hash อยู่ดี ทำให้ response time เท่ากันไม่ว่า email จะมีอยู่หรือไม่

### Logging

Services ส่ง log เข้า log-service ผ่าน internal endpoint ที่ Nginx บล็อกจากภายนอก:

```
POST http://log-service:3003/api/logs/internal
```

log-service รับข้อมูลแล้ว INSERT ลงตาราง `logs` ใน PostgreSQL โดยไม่ต้องการ JWT (เพราะเรียกภายใน Docker network เท่านั้น)

Admin ดู log ได้ผ่าน `GET /api/logs/` ซึ่งต้องใช้ JWT ที่มี `role = admin` เท่านั้น Member จะได้ 403 Forbidden

---

## 8. Known Limitations

**Self-Signed Certificate** — Browser แสดง warning ทุกครั้ง ต้องกด proceed เอง ใช้งาน production จริงต้องใช้ Let's Encrypt

**Shared Database** — ทุก service ใช้ PostgreSQL instance เดียวกัน สะดวกสำหรับ lab แต่ไม่ตรง microservices pattern จริงที่ควรแยก DB ต่อ service

**No Refresh Token** — JWT หมดอายุใน 1 ชั่วโมง user ต้อง login ใหม่ ไม่มี silent refresh

**Fire-and-Forget Logging** — ถ้า log-service down ชั่วคราว log จะหายไปเพราะไม่มี retry หรือ message queue

**Rate Limit Reset on Restart** — Nginx เก็บ rate limit ใน memory ถ้า restart counter จะ reset ทันที

**No HTTPS Between Services** — ภายใน Docker network ยังใช้ plain HTTP เพราะถือว่า internal network trusted
