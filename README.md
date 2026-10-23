# Rạp Sao – Hệ thống đặt vé xem phim (Đề 11)

Môn: Triển khai và quản trị hệ thống phần mềm.
Website đặt vé (suất chiếu, ghế, thanh toán giả lập) chạy bằng **Express + MySQL + phpMyAdmin**, đặt sau **Nginx reverse proxy**, giám sát bằng **Prometheus + Grafana**, log tập trung bằng **Loki + Promtail (LogQL)**, có **hardening**.

## 1. Chạy hệ thống (Ubuntu hoặc máy có Docker)

```bash
git clone https://github.com/<username>/cinema-booking.git
cd cinema-booking
cp .env.example .env          # sửa mật khẩu trong .env
sh nginx/gen-cert.sh          # tạo chứng chỉ TLS tự ký
docker compose up -d --build
docker compose ps             # tất cả phải Up / healthy
```

Lần đầu MySQL cần khoảng 30–60 giây để khởi tạo dữ liệu (`db/init.sql`).

## 2. Khi thi thực hành cần mở những gì

Chuẩn bị sẵn các tab/cửa sổ này (nếu máy thi là máy ảo, thay `localhost` bằng IP máy ảo):

| # | Mở cái gì | Địa chỉ / lệnh | Cho thầy thấy |
|---|---|---|---|
| 1 | **GitHub repo** | https://github.com/`<username>`/cinema-booking | Mã nguồn, README, lịch sử commit, `.github/workflows/ci.yml` (tab Actions chạy xanh), `.env` KHÔNG bị commit |
| 2 | **Website qua Nginx** | https://localhost/ | Đăng nhập → chọn phim → suất chiếu → chọn ghế → thanh toán giả lập → "Vé của tôi" |
| 3 | **phpMyAdmin** | https://localhost/pma/ (đăng nhập `cinema_app` hoặc `root`) | Bảng `bookings`, `booking_seats`, `payments` thay đổi theo thao tác vừa làm |
| 4 | **Prometheus** | http://127.0.0.1:9090 → Status → Targets, Alerts | 4 target `UP` (cinema-app, node, mysql, prometheus); các alert rule |
| 5 | **Grafana dashboard** | https://localhost/grafana/ (admin / `GRAFANA_PASSWORD`) → Dashboards → Cinema | Biểu đồ request/s, lỗi 5xx, P95, đơn đặt vé, CPU, kết nối MySQL |
| 6 | **Grafana Explore + Loki (LogQL)** | Grafana → Explore → chọn datasource **Loki** | Chạy các truy vấn ở mục 4 |
| 7 | **Terminal** | `bash scripts/demo-checks.sh` | `docker compose ps`, `nginx -t`, header bảo mật, `/metrics` bị chặn (403), rate limit (429), `ufw status`, MySQL không publish ra ngoài |

Tài khoản mẫu: `admin@cinema.local` (mật khẩu `ADMIN_PASSWORD`), `user@cinema.local` (mật khẩu `DEMO_PASSWORD`).

## 3. Kịch bản demo gợi ý (5–7 phút)

1. Mở GitHub, nói qua cấu trúc repo và CI.
2. Mở website, đặt một vé thành công. Mở phpMyAdmin, cho thấy bản ghi mới.
3. Dùng 2 trình duyệt (hoặc ẩn danh) chọn cùng một ghế: người thứ hai nhận lỗi "Ghế vừa có người khác đặt" (UNIQUE + transaction).
4. Bấm "Giả lập thanh toán lỗi": ghế được trả lại, Grafana hiện đơn `failed`, Loki hiện log `payment_failed`.
5. Chạy 20–30 lần đăng nhập sai (hoặc `demo-checks.sh`): Nginx trả 429, Loki thấy `login_failed` từ cùng IP.
6. Tắt app: `docker compose stop app` → Prometheus Alerts: `ServiceDown` chuyển Pending → Firing sau 1 phút. Bật lại: `docker compose start app`.
7. Chạy `demo-checks.sh` để chứng minh hardening.

## 4. Truy vấn mẫu

**PromQL**

```
sum(rate(http_request_duration_seconds_count[1m]))
sum(rate(http_request_duration_seconds_count{status=~"5.."}[5m]))
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
sum by (status) (increase(bookings_total[5m]))
mysql_global_status_threads_connected
```

**LogQL** (nhãn `service` lấy từ tên service trong docker-compose; nhãn `job="nginx"` cho log Nginx)

```
{service="app"}
{service="app"} |= "error"
{service="app"} | json | level="error"
{service="app"} | json | msg="booking_created"
{service="app"} | json | msg="login_failed"
{service="app"} |= "SEAT_TAKEN"
{job="nginx"} |~ " 5[0-9]{2} "
sum(count_over_time({job="nginx"} |~ " 5[0-9]{2} " [5m]))
sum(rate({job="nginx"}[1m]))
sum by (ip) (count_over_time({service="app"} | json | msg="login_failed" [5m]))
```

## 5. Hardening đã áp dụng

| Lớp | Biện pháp | File |
|---|---|---|
| Máy chủ | UFW chỉ mở 22/80/443, SSH chỉ dùng key, fail2ban, unattended-upgrades, sysctl | `scripts/hardening.sh` |
| Nginx | Ẩn phiên bản, TLS 1.2/1.3, header bảo mật + CSP, rate limit API và đăng nhập, chặn `/metrics`, phpMyAdmin chỉ cho mạng nội bộ | `nginx/nginx.conf` |
| Docker | Hai mạng tách biệt, chỉ Nginx publish cổng, app chạy non-root + `read_only` + `cap_drop: ALL` + `no-new-privileges`, image ghim phiên bản | `docker-compose.yml`, `app/Dockerfile` |
| MySQL | User ứng dụng chỉ có SELECT/INSERT/UPDATE/DELETE; user `exporter` riêng; không publish cổng 3306 | `db/init.sql` |
| Ứng dụng | Prepared statement, bcrypt, JWT hết hạn 2 giờ, `helmet`, validate đầu vào, giới hạn kích thước body | `app/src/server.js` |
| Bí mật | `.env` nằm trong `.gitignore`, chỉ commit `.env.example` | `.gitignore` |

Chạy hardening máy chủ (chỉ trên máy Ubuntu thật/VM, đã có SSH key): `sudo bash scripts/hardening.sh`.

## 6. Cấu trúc

```
app/                 Express API + giao diện web (app/src/public)
db/init.sql          Schema + dữ liệu mẫu + phân quyền
nginx/               nginx.conf, gen-cert.sh
monitoring/          prometheus.yml, alert.rules.yml, loki-config.yml, promtail-config.yml, mysqld.cnf
grafana/             Datasource + dashboard tự động nạp
scripts/             hardening.sh, demo-checks.sh
.github/workflows/   CI
```

## 7. Xử lý sự cố

| Triệu chứng | Cách xử lý |
|---|---|
| `nginx` báo thiếu `server.crt` | Chạy `sh nginx/gen-cert.sh` rồi `docker compose up -d` |
| App không kết nối được DB | `docker compose logs db` (đợi init xong); nếu đổi mật khẩu trong `.env` sau lần chạy đầu: `docker compose down -v` rồi chạy lại |
| Grafana không có dữ liệu log | `docker compose logs promtail`; kiểm tra `/var/run/docker.sock` được mount |
| Trình duyệt cảnh báo chứng chỉ | Bình thường với chứng chỉ tự ký, chọn Advanced → Proceed |
| Prometheus target `mysql` DOWN | `docker compose logs mysqld-exporter`; kiểm tra user `exporter` đã được tạo trong `db/init.sql` |
