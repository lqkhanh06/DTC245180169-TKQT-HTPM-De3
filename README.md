# ĐỀ TÀI 03: WEBSITE QUẢNG BÁ SẢN PHẨM (WORDPRESS)

> **Môn học:** Triển khai và Quản trị Hệ thống Phần mềm  
> **Hệ thống triển khai:** WordPress + MySQL 8.0 + phpMyAdmin + Nginx Reverse Proxy + Prometheus + Grafana + Loki + Promtail  
> **Phương pháp triển khai:** Docker Compose  

---

## 📌 1. Giới thiệu tổng quan hệ thống

Dự án triển khai một trang web quảng bá sản phẩm hoàn chỉnh dựa trên nền tảng **WordPress**, cơ sở dữ liệu **MySQL**, công cụ quản trị **phpMyAdmin**, được bảo vệ bởi **Nginx Reverse Proxy** (hỗ trợ **HTTPS SSL tự ký**, **Security Headers**, ẩn thông tin phiên bản), tích hợp hệ thống **Giám sát (Prometheus + Grafana)** và hệ thống **Ghi log tập trung (Loki + Promtail + LogQL)**.

### Sơ đồ kiến trúc toàn diện (Full Architecture):
```text
                            [ Client / Browser ]
                                     │
                     ┌───────────────┴───────────────┐
                     │ HTTP (80/8080) -> Redirect    │ HTTPS (443 / 8443)
                     ▼                               ▼
       ┌──────────────────────────────────────────────────────────┐
       │                   NGINX REVERSE PROXY                    │
       │   (SSL Termination, Security Headers, server_tokens off) │
       └──────────────┬────────────────────────────┬──────────────┘
                      │ (frontend_net)             │ (/var/log/nginx)
         ┌────────────┴────────────┐               ▼
         ▼                         ▼         ┌───────────┐
  ┌──────────────┐          ┌──────────────┐ │ PROMTAIL  │
  │  WORDPRESS   │          │  PHPMYADMIN  │ └─────┬─────┘
  │  (Port 80)   │          │  (Port 80)   │       │ (Push Logs)
  └──────┬───────┘          └──────┬───────┘       ▼
         │                         │         ┌───────────┐
         └────────────┬────────────┘         │   LOKI    │ (Port 3100)
                      │ (backend_net)        └─────┬─────┘
                      ▼                            │
               ┌──────────────┐                    │ (LogQL)
               │ MYSQL DB 8.0 │                    │
               │ (Port 3306)  │                    ▼
               └──────┬───────┘              ┌───────────┐
                      │                      │  GRAFANA  │ (Port 3000)
                      ▼                      │ (UI Dash) │
               ┌──────────────┐              └─────▲─────┘
               │ MYSQL EXPORT │                    │
               │ (Port 9104)  │                    │ (Metrics)
               └──────┬───────┘                    │
                      │ (monitoring_net)           │
                      ▼                            │
               ┌──────────────┐                    │
               │  PROMETHEUS  │────────────────────┘
               │ (Port 9090)  │
               └──────▲───────┘
                      │
        ┌─────────────┴─────────────┐
        ▼                           ▼
 ┌──────────────┐            ┌──────────────┐
 │ NGINX EXPORT │            │ NODE EXPORT  │
 │ (Port 9113)  │            │ (Port 9100)  │
 └──────────────┘            └──────────────┘
```

---

## 🔒 2. Các biện pháp bảo vệ & Hardening (Đạt yêu cầu Mục 6)

1. **Network Isolation (Phân tách mạng nội bộ Docker):**
   - `frontend_net`: Nginx <-> WordPress, phpMyAdmin.
   - `backend_net`: WordPress, phpMyAdmin <-> MySQL.
   - `monitoring_net`: Prometheus, Grafana, Loki, Promtail, Exporters.
   - **MySQL hoàn toàn không mở cổng ra máy host**, ngăn chặn triệt để tấn công dò quét cổng từ Internet.
2. **Nginx Security Headers & SSL/TLS:**
   - Hỗ trợ giao thức `TLSv1.2` và `TLSv1.3`.
   - `server_tokens off;` (Ẩn hoàn toàn phiên bản Nginx trong header phản hồi).
   - `X-Frame-Options: SAMEORIGIN` (Chống Clickjacking).
   - `X-Content-Type-Options: nosniff` (Chống MIME sniffing).
   - `X-XSS-Protection: 1; mode=block` (Chống tấn công Cross-Site Scripting).
   - `Strict-Transport-Security: max-age=31536000; includeSubDomains` (Bắt buộc dùng HTTPS).
3. **Quản lý mật khẩu & An toàn mã nguồn:**
   - Mật khẩu lưu trong file `.env` và `mysql-exporter.cnf`.
   - Cả hai file chứa mật khẩu đều được đưa vào `.gitignore`, cung cấp các file mẫu `.env.example` và `mysql-exporter.cnf.example`.
4. **Hạn chế quyền Database User:**
   - Ứng dụng WordPress kết nối bằng tài khoản giới hạn `wordpress_user` chỉ có quyền trên `wordpress_db`, không sử dụng quyền `root`.

---

## 🌐 3. Danh sách dịch vụ và Cổng truy cập

| Dịch vụ | Địa chỉ truy cập | Giao thức | Chức năng |
| :--- | :--- | :--- | :--- |
| **WordPress (Website)** | `https://localhost` | HTTPS (SSL) | Website giới thiệu sản phẩm chính |
| **WordPress Redirect** | `http://localhost` | HTTP | Tự động chuyển hướng sang 443 HTTPS |
| **phpMyAdmin** | `https://localhost:8443` | HTTPS (SSL) | Giao diện quản trị cơ sở dữ liệu MySQL |
| **phpMyAdmin Redirect** | `http://localhost:8080` | HTTP | Tự động chuyển hướng sang 8443 HTTPS |
| **Grafana** | `http://localhost:3000` | HTTP | Giao diện Dashboard giám sát & LogQL |
| **Prometheus** | `http://localhost:9090` | HTTP | Máy chủ thu thập Metrics thời gian thực |
| **Loki** | `http://localhost:3100` | HTTP | Máy chủ lưu trữ và lập chỉ mục Logs |

*Tài khoản mặc định Grafana:* `admin` / `admin`  
*Tài khoản mặc định phpMyAdmin:* `wordpress_user` / xem trong `.env`

---

## 📊 4. Hệ thống Giám sát & Metrics (Commit 2)

* **Prometheus Targets:** Quản lý và thu thập 4 nguồn số liệu:
  - `nginx`: Thu thập từ `nginx-prometheus-exporter` qua endpoint `/stub_status`.
  - `mysql`: Thu thập từ `mysqld-exporter` kết nối MySQL nội bộ.
  - `node`: Thu thập thông số tải CPU, RAM, Disk của máy chủ/container từ `node-exporter`.
  - `prometheus`: Tự giám sát hoạt động của chính máy chủ Prometheus.
* **Grafana Dashboard:** Được cấu hình tự động (Provisioning) qua file `system-overview.json`:
  - **Nginx:** Active Connections, Requests per Second.
  - **MySQL:** Queries Per Second (QPS), Threads Connected.
  - **System:** CPU Usage (%), RAM Usage (%).

---

## 📝 5. Hệ thống Log tập trung & Truy vấn LogQL (Commit 3)

Promtail đọc liên tục log truy cập và log lỗi của Nginx (`/var/log/nginx/app_*.log`) và gửi về Loki.

### Các câu truy vấn LogQL mẫu đã thử nghiệm thành công:
1. **Xem toàn bộ log của Web Server:**
   ```logql
   {job="nginx"}
   ```
2. **Lọc log các yêu cầu tới trang quản trị WordPress:**
   ```logql
   {job="nginx"} |= "wp-admin"
   ```
3. **Lọc theo phương thức HTTP GET:**
   ```logql
   {job="nginx"} |= "GET"
   ```
4. **Vẽ biểu đồ tần suất request theo thời gian (Log rate):**
   ```logql
   rate({job="nginx"}[1m])
   ```

---

## 🚀 6. Hướng dẫn chạy dự án

1. **Chuẩn bị file cấu hình:**
   ```bash
   cp .env.example .env
   cp mysql-exporter.cnf.example mysql-exporter.cnf
   # Điều chỉnh mật khẩu trong .env và mysql-exporter.cnf nếu cần
   ```
2. **Khởi động toàn bộ 11 container:**
   ```bash
   docker compose up -d
   ```
3. **Kiểm tra trạng thái:**
   ```bash
   docker compose ps
   ```

---

## 📦 7. Tiến trình Commit Git (Đúng theo yêu cầu Đề tài)

- [x] **Commit 1:** `feat: Khoi tao du an, trien khai WordPress, MySQL, phpMyAdmin va Nginx HTTPS Reverse Proxy`
- [x] **Commit 2:** `feat: Tich hop Prometheus va Grafana giam sat container, web server va database`
- [x] **Commit 3:** `feat: Trien khai he thong log tap trung Loki va Promtail, ho tro truy van LogQL`
