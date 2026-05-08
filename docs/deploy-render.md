# Deploy Strapi v5 lên Render (Free Tier)

## Lưu ý trước khi bắt đầu

Render free tier có **2 hạn chế cần biết**:

| Hạn chế | Chi tiết |
| --- | --- |
| **Service sleep** | Tự sleep sau 15 phút không có request, lần đầu load lại mất ~30–60 giây |
| **Database** | Render không có MySQL/MariaDB free — chỉ có **PostgreSQL free** |

> Driver `pg` đã cài sẵn trong project, chỉ cần đổi env là xong.

---

## Yêu cầu

- Tài khoản [Render](https://render.com) (đăng ký bằng GitHub)
- Repo đã push lên GitHub
- Node.js >=20

---

## Bước 1 — Tạo PostgreSQL database

1. Vào [dashboard.render.com](https://dashboard.render.com) → **New +** → **PostgreSQL**
2. Điền thông tin:
   - **Name:** `strapi-db` (hoặc tuỳ ý)
   - **Region:** Singapore (gần VN nhất)
   - **Plan:** **Free**
3. Click **Create Database**
4. Sau khi tạo xong, vào tab **Info** → copy **Internal Database URL** (dạng `postgresql://user:pass@host/dbname`)

> Dùng **Internal URL** (không phải External) để kết nối giữa các service trong cùng Render — nhanh hơn và miễn phí bandwidth.

---

## Bước 2 — Tạo Web Service cho Strapi

1. **New +** → **Web Service**
2. Chọn GitHub repo `strapi_sass`
3. Điền thông tin:

| Trường | Giá trị |
| --- | --- |
| **Name** | `strapi-app` |
| **Region** | Singapore |
| **Branch** | `main` |
| **Runtime** | Node |
| **Build Command** | `npm install && npm run build` |
| **Start Command** | `npm run start` |
| **Plan** | **Free** |

4. Click **Create Web Service** — **chưa deploy vội**, cần set env trước

---

## Bước 3 — Cấu hình biến môi trường

Trong Web Service vừa tạo → tab **Environment** → thêm từng biến:

### Database

```env
DATABASE_CLIENT=postgres
DATABASE_URL=<Internal Database URL từ Bước 1>
DATABASE_SSL=true
```

> **Quan trọng:** Phải dùng **Internal Database URL**, không phải External URL. Copy từ tab **Info** của PostgreSQL service → mục **Internal Database URL**.
> Internal: `postgresql://user:pass@dpg-xxx-a/dbname` — External (sai): `postgresql://user:pass@dpg-xxx-a.singapore-postgres.render.com/dbname`

> Render PostgreSQL **bắt buộc SSL** — không set hoặc set `false` sẽ lỗi `SSL/TLS required` khi start.
>
> `config/database.ts` đã hỗ trợ `connectionString` — chỉ cần set `DATABASE_URL` là đủ, không cần set host/port/name riêng lẻ.

### Strapi secrets

Chạy lệnh này trên máy local để tạo giá trị ngẫu nhiên:

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"
```

Chạy 6 lần, lấy 6 giá trị khác nhau:

```env
APP_KEYS=<key1>,<key2>
API_TOKEN_SALT=<key3>
ADMIN_JWT_SECRET=<key4>
TRANSFER_TOKEN_SALT=<key5>
USERS_PERMISSIONS_JWT_SECRET=<key6>
ENCRYPTION_KEY=<key7>
```

### Server

```env
HOST=0.0.0.0
PORT=10000
NODE_ENV=production
```

> Render gán port `10000` theo mặc định cho Web Service — Strapi cần khớp với giá trị này.

### Email (nếu cần)

```env
SMTP_HOST=smtp.office365.com
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USERNAME=your-email@domain.com
SMTP_PASSWORD=your-password
SMTP_DEFAULT_FROM=your-email@domain.com
SMTP_DEFAULT_REPLY_TO=your-email@domain.com
```

---

## Bước 4 — Deploy

1. Sau khi set xong env → click **Save Changes**
2. Render tự trigger deploy đầu tiên
3. Xem log realtime trong tab **Logs**
4. Build lần đầu mất ~5–8 phút

### Log thành công

```text
info: Starting Strapi in production mode...
info: Strapi started successfully
info: Server listening on 0.0.0.0:10000
```

URL public sẽ dạng: `https://strapi-app.onrender.com`

Strapi Admin Panel: `https://strapi-app.onrender.com/admin`

---

## Bước 5 — Giải quyết vấn đề sleep (quan trọng khi demo)

Free tier sleep sau 15 phút idle. Để tránh giật lag khi demo, dùng một trong hai cách:

### Cách 1: UptimeRobot (miễn phí)

1. Vào [uptimerobot.com](https://uptimerobot.com) → tạo tài khoản free
2. **New Monitor** → **HTTP(s)**
3. URL: `https://strapi-app.onrender.com`
4. Interval: **5 minutes**
5. UptimeRobot ping mỗi 5 phút → service không bao giờ sleep

### Cách 2: Cron job đơn giản (nếu không muốn dùng service ngoài)

Thêm vào `package.json` một script ping, hoặc dùng GitHub Actions schedule để tự ping URL mỗi 10 phút.

---

## Hướng dẫn cho intern — Deploy branch riêng

### 1. Fork repo

Intern fork repo trên GitHub UI, clone về máy:

```bash
git clone https://github.com/<intern-username>/strapi_sass
cd strapi_sass
git checkout -b feature/ten-tinh-nang
```

### 2. Deploy lên Render

- Mỗi intern tạo Render account riêng
- Lặp lại Bước 1–4 với fork của mình
- Mỗi intern có 1 URL riêng: `https://strapi-<ten>.onrender.com`

### 3. Push code → tự deploy

```bash
git push origin feature/ten-tinh-nang
```

Render tự build và deploy khi có commit mới trên branch được track.

---

## Troubleshooting

### Lỗi: `role "strapi" does not exist`

Đảm bảo đang dùng đúng **Internal Database URL** từ Render, không phải tự gõ tay.

### Lỗi: `SSL/TLS required`

Render PostgreSQL **luôn bắt buộc SSL**. Set:

```env
DATABASE_SSL=true
```

### Lỗi: `Missing jwtSecret`

Thiếu biến `USERS_PERMISSIONS_JWT_SECRET`. Thêm vào env:

```env
USERS_PERMISSIONS_JWT_SECRET=<random base64 string>
```

Tạo giá trị: `node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"`

### Lỗi: `Cannot find module` khi build

Build command phải là `npm install && npm run build` — không bỏ phần `npm install`.

### Admin panel trắng sau deploy

Đảm bảo `NODE_ENV=production` đã set — Strapi cần build admin panel trước khi serve.

### Service không nhận port

Kiểm tra `PORT=10000` đã set trong env — Render expose đúng port này ra ngoài.

---

## Chi phí

| Service | Plan | Giá |
| --- | --- | --- |
| Strapi Web Service | Free | $0 |
| PostgreSQL | Free (1GB, 90 ngày) | $0 |
| **Tổng** | | **$0** |

> PostgreSQL free tier trên Render **hết hạn sau 90 ngày** — cần tạo lại DB mới và migrate data nếu muốn dùng tiếp. Phù hợp cho intern demo trong 3–4 tháng.
