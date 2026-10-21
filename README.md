# ĐỀ TÀI 03: WEBSITE QUẢNG BÁ SẢN PHẨM (WORDPRESS)

> **Môn học:** Triển khai và Quản trị Hệ thống Phần mềm  
> **Hệ thống triển khai:** WordPress + MySQL 8.0 + phpMyAdmin + Nginx Reverse Proxy + Prometheus + Grafana + Loki + Promtail  
> **Phương pháp triển khai:** Docker Compose  

---

## 📌 1. Giới thiệu tổng quan hệ thống

Dự án triển khai một trang web quảng bá sản phẩm hoàn chỉnh dựa trên nền tảng **WordPress**, cơ sở dữ liệu **MySQL**, công cụ quản trị **phpMyAdmin**, được đặt sau lớp bảo vệ của **Nginx Reverse Proxy** với chứng chỉ **HTTPS (SSL tự ký)** và các biện pháp bảo mật (**Hardening**).

### Mô hình kiến trúc mạng (Network Architecture):
```text
[ Người dùng / Trình duyệt ]
          │
          │ HTTP (80) -> Redirect 301 HTTPS
          │ HTTPS (443 - WordPress) | HTTPS (8443 - phpMyAdmin)
          ▼
   ┌────────────────────────────────────────────────────────┐
   │                  NGINX REVERSE PROXY                   │
   │  (SSL Termination, Security Headers, server_tokens off)│
   └──────────────────────────┬─────────────────────────────┘
                              │ (Mạng frontend_net)
         ┌────────────────────┴────────────────────┐
         ▼                                         ▼
┌──────────────────┐                     ┌──────────────────┐
│  WORDPRESS APP   │                     │  PHPMYADMIN TOOL │
│   (Port 80)      │                     │    (Port 80)     │
└────────┬─────────┘                     └────────┬─────────┘
         │                                        │
         └────────────────────┬───────────────────┘
                              │ (Mạng backend_net)
                              ▼
                   ┌──────────────────────┐
                   │    MYSQL DATABASE    │
                   │ (Port 3306 - Nội bộ) │
                   └──────────────────────┘
```

---

## 🔒 2. Các biện pháp bảo vệ & Hardening (Phase 1)

1. **Network Isolation (Cách ly mạng Docker):**
   - `frontend_net`: Chỉ cho phép Nginx giao tiếp với WordPress và phpMyAdmin.
   - `backend_net`: Chỉ cho phép WordPress và phpMyAdmin kết nối tới MySQL.
   - **MySQL không mở cổng (no port mapping) ra máy host**, ngăn chặn hoàn toàn nguy cơ quét cổng và tấn công từ bên ngoài.
2. **Nginx Security Headers & SSL:**
   - Hỗ trợ giao thức an toàn `TLSv1.2` và `TLSv1.3`.
   - `server_tokens off;` (Ẩn hoàn toàn thông tin phiên bản Nginx trong HTTP Header).
   - `X-Frame-Options: SAMEORIGIN` (Chống tấn công Clickjacking).
   - `X-Content-Type-Options: nosniff` (Chống MIME-sniffing).
   - `X-XSS-Protection: 1; mode=block` (Bảo vệ chống tấn công Cross-Site Scripting).
   - `Strict-Transport-Security` (Buộc giao tiếp qua HTTPS).
3. **Quản lý biến môi trường an toàn:**
   - Toàn bộ thông tin mật khẩu, tài khoản quản trị lưu trữ trong file `.env`.
   - File `.env` được đưa vào `.gitignore`, cung cấp file mẫu `.env.example` cho việc triển khai.

---

## 🚀 3. Hướng dẫn cài đặt và khởi chạy

### Yêu cầu tiên quyết:
- Đã cài đặt **Docker** và **Docker Compose**.
- Đã cài đặt **Git**.

### Các bước thực hiện:

#### Bước 1: Chuẩn bị cấu hình môi trường
Nếu chưa có file `.env`, tạo file `.env` từ file mẫu `.env.example`:
```bash
cp .env.example .env
```

#### Bước 2: Khởi động toàn bộ dịch vụ
Chạy lệnh sau tại thư mục gốc của dự án:
```bash
docker compose up -d
```

Kiểm tra trạng thái các container đang chạy:
```bash
docker compose ps
```

---

## 🌐 4. Địa chỉ truy cập các dịch vụ

| Dịch vụ | Địa chỉ truy cập | Giao thức | Mô tả |
| :--- | :--- | :--- | :--- |
| **WordPress (Website)** | `https://localhost` | HTTPS (SSL) | Trang web giới thiệu sản phẩm chính |
| **WordPress (Redirect)** | `http://localhost` | HTTP | Tự động chuyển hướng sang `https://localhost` |
| **phpMyAdmin** | `https://localhost:8443` | HTTPS (SSL) | Giao diện quản trị cơ sở dữ liệu MySQL |
| **phpMyAdmin (Redirect)**| `http://localhost:8080` | HTTP | Tự động chuyển hướng sang `https://localhost:8443` |

> *Lưu ý: Do sử dụng chứng chỉ SSL tự ký (self-signed certificate) cho môi trường nội bộ, trình duyệt (Chrome/Firefox/Edge) sẽ hiển thị cảnh báo bảo mật. Bạn chỉ cần chọn **"Nâng cao" (Advanced)** và nhấn **"Tiếp tục truy cập localhost (không an toàn)"**.*

### Thông tin đăng nhập phpMyAdmin mặc định:
- **Server:** `mysql`
- **Username:** `wordpress_user` (hoặc `root`)
- **Password:** Xem trong file `.env` (Mặc định: `StrongWpPasswordSecure2026!` hoặc `SuperRootPasswordSecure2026!`)

---

## 📦 5. Tiến trình Commit (Theo yêu cầu đề bài)

- [x] **Commit 1:** Triển khai WordPress + MySQL + phpMyAdmin + Nginx Reverse Proxy (HTTPS, Security Headers, Network Isolation).
- [ ] **Commit 2:** Tích hợp Prometheus + Grafana giám sát container, web server và database.
- [ ] **Commit 3:** Triển khai hệ thống log tập trung Loki + Promtail, truy vấn log bằng LogQL.
