# BÁO CÁO FINAL ASSIGNMENT — INTRO TO DEVOPS

## Dự án: Lá Shop — CI/CD và Continuous Monitoring

**Sinh viên:** Hà Văn Hoàng Phú -124010124087  
**Lớp:** K24WD03  
**Giảng viên:** Phạm Trần Đức Trọng  
**Ngày nộp:** 5/11/2026

> Báo cáo này được soạn từ quá trình triển khai thực tế và các ảnh/log đã trao đổi.

## 1. Mục tiêu

Xây dựng một quy trình DevOps cho website Lá Shop: quản lý mã nguồn bằng GitHub, tự động kiểm thử và build bằng Jenkins, đóng gói qua Docker, triển khai lên VPS, giám sát bằng Prometheus/Grafana và nhận thông báo kết quả CI/CD qua Telegram.

## 2. Giới thiệu hệ thống

- **Website:** https://lashop.hoangphu.click
- **API health:** https://lashop.hoangphu.click/api/health
- **Frontend:** React, Vite.
- **Backend:** Go.
- **Database:** MySQL 8.4.
- **Frontend source:** https://github.com/hoangphu11099/Lashop-Fontend
- **Backend source:** https://github.com/hoangphu11099/Lashop-Backend
- **Máy build:** Windows với Jenkins Agent `lashop-windows`, Docker Desktop và Buildx.
- **Máy chủ:** Ubuntu VPS.

### 2.1. Kiến trúc DevOps

```mermaid
flowchart TD
  A[Developer] -->|git push| B[GitHub - Frontend / Backend]
  B -->|Poll SCM 5 phút| C[Jenkins Controller]
  C --> D[CI: ESLint, React Build, Go Unit Test]
  C --> E[Windows Agent - Docker Buildx]
  E -->|SCP/SSH| F[VPS Ubuntu - Docker Compose]
  F --> G[Frontend Nginx]
  F --> H[Backend Go API]
  H --> I[(MySQL 8.4)]
  G --> J[HTTPS Lá Shop]
  F --> K[Node Exporter]
  J --> L[Blackbox Exporter]
  K --> M[Prometheus]
  L --> M
  M --> N[Grafana]
  C --> O[Telegram Pipeline Notification]
```

**[ẢNH 01: Sơ đồ kiến trúc tổng quan]**
![alt text](mermaid-diagram-2.png)

## 3. Docker hóa ứng dụng

Ứng dụng được chia thành 3 service trong Docker Compose: `frontend`, `backend`, `mysql`. Frontend được phục vụ qua Nginx, backend sử dụng API Go và database MySQL 8.4. MySQL sử dụng named volume để lưu dữ liệu bền vững.

Trên VPS, cấu hình production nằm tại `/var/www/lashop`, sử dụng `compose.yaml` kết hợp `compose.deploy.yaml` nhằm chọn những images `lashop-frontend:deploy` và `lashop-backend:deploy`.

Lệnh kiểm tra:

```bash
cd /var/www/lashop
docker compose -f compose.yaml -f compose.deploy.yaml config --quiet
docker compose -f compose.yaml -f compose.deploy.yaml ps
curl -fsS https://lashop.hoangphu.click/api/health
```

Kết quả đã quan sát: các container frontend, backend và MySQL chạy; MySQL healthy; health endpoint trả `{"status":"ok"}`; website HTTPS trả HTTP 200.

**[ẢNH 02: Dockerfile frontend/backend]**
![alt text](image.png)
![alt text](image-1.png)
**[ẢNH 03: Docker Compose / `docker compose ps`]**
![alt text](image-2.png)
**[ẢNH 04: Website và health API]**
![alt text](image-3.png)

## 4. Jenkins CI/CD

![alt text](image-4.png)

### 4.1. Môi trường Jenkins

- Jenkins Controller chạy trong Docker trên Windows, truy cập qua `http://localhost:8081`.
- Windows Jenkins Agent: `lashop-windows`, kết nối qua Jenkins inbound agent.
- CI/CD Job: `LaShop-CI-CD`.
- SSH credential: quản lý trong Jenkins Credentials, không ghi khóa bí mật trong report.
- Trigger: Poll SCM với lịch `H/5 * * * *`, giúp Jenkins tự phát hiện commit mới sau `git push`.

### 4.2. Các giai đoạn Pipeline

1. Checkout mã nguồn 2 nhánh `main` từ frontend/backend repositories.
2. Frontend: `npm ci`, `npm run lint`, `npm run build`.
3. Backend: `go test ./...`, `go build ./...`.
4. Docker Buildx chạy trên Windows Agent, tạo images cho `linux/amd64`.
5. Chuyển archive images qua SSH/SCP đến VPS, cổng 24700.
6. VPS load và gán tag deploy, sau đó script `/usr/local/sbin/lashop-deploy` áp dụng Docker Compose. Script lưu image cũ để rollback khi gặp lỗi.
7. Kiểm tra kết quả deploy, gửi Telegram thông báo `SUCCESS` hoặc `FAILURE`.

Trong lần kiểm thử, Jenkins đã tự tạo build mới từ thay đổi GitHub. Kết quả build thành công và Telegram đã gửi thông báo. Cần chèn ảnh Console Output mới nhất có bước lint để chứng minh lint thật sự chạy trong Pipeline.

**[ẢNH 05: Jenkins Job + Poll SCM]**
![alt text](image-7.png)
**[ẢNH 06: Console Output có npm run lint / go test]**
![alt text](image-6.png)
**[ẢNH 07: Docker Build + SSH deploy + Finished: SUCCESS]**  
**[ẢNH 08: Telegram thông báo]**
![alt text](image-8.png)

### 4.3. Phục hồi khi deploy lỗi

Script deploy trên VPS lưu image của frontend/backend đang chạy thành tag rollback trước khi cập nhật. Khi lệnh triển khai thất bại, script có logic khôi phục các tags và recreate dịch vụ. Đây là cơ chế giảm rủi ro, không thay thế backup dữ liệu database. **[CẦN BỔ SUNG: ảnh hoặc test minh chứng rollback nếu đề yêu cầu]**.

## 5. Monitoring liên tục

### 5.1. Thành phần

Monitoring stack được tách tại `/opt/lashop-monitoring`:

| Service           | Vai trò                              |
| ----------------- | ------------------------------------ |
| Prometheus        | Thu thập metrics theo chu kỳ 30 giây |
| Node Exporter     | Đo CPU, RAM, disk của VPS            |
| Blackbox Exporter | HTTP probe website và API health     |
| Grafana           | Dashboard trực quan hóa metrics      |

Prometheus sử dụng retention 7 ngày / giới hạn block 512MB. Grafana và Prometheus được bind loopback trên VPS, truy cập qua SSH Tunnel để tránh phơi cổng quản trị công khai.

### 5.2. Dashboard và queries

Tên dashboard: **LaShop DevOps Monitoring**.

**CPU**

```promql
100 * (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])))
```

**RAM**

```promql
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
```

**Disk**

```promql
100 * (1 - node_filesystem_avail_bytes{mountpoint="/",fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{mountpoint="/",fstype!~"tmpfs|overlay"})
```

**Website / API status**

```promql
probe_success{job="lashop-website"}
```

Ảnh Dashboard cho thấy đầy đủ 4 panel: CPU, RAM, Disk và Website/API status. Trong lần kiểm tra, 2 HTTP probes có giá trị `1`, nghĩa là probe HTTP website/API thành công tại thời điểm đó.

**[ẢNH : Dashboard 4 panel]**
![alt text](image-5.png)

### 5.3. Cảnh báo sự cố

## 6. Kết quả kiểm thử

| Hạng mục        | Kết quả đã quan sát                        | Minh chứng cần chèn         |
| --------------- | ------------------------------------------ | --------------------------- |
| Frontend ESLint | `npm run lint` chạy không lỗi trên Windows | Log npm                     |
| Frontend build  | `npm run build` thành công                 | Log Vite                    |
| Go unit tests   | `go test ./...` thành công                 | Jenkins Console Output      |
| Docker / VPS    | frontend/backend/mysql chạy, MySQL healthy | Compose ps                  |
| HTTPS/API       | HTTP 200, API `status: ok`                 | curl / website              |
| Auto trigger    | Poll SCM phát hiện push, tự bắt đầu build  | Build History / Polling Log |
| Telegram        | Nhận tin pipeline hoàn tất                 | Telegram screenshot         |
| Monitoring      | CPU/RAM/Disk metrics; 2 probe_success = 1  | Grafana screenshot          |
| Grafana alerts  | Chưa xác nhận                              | Bổ sung nếu cấu hình        |

## 7. Các bước tái tạo hệ thống (tóm tắt)

1. Clone cả hai repositories vào cùng workspace; chuẩn bị file `.env` dựa trên `.env.example`.
2. Kiểm tra và chạy các Docker Compose services (`docker compose config`, `docker compose up -d --build`).
3. Cài Jenkins Controller, SSH Agent plugin và Windows Agent; cấu hình GitHub URLs, Jenkins Credentials và Poll SCM.
4. Xuất và lưu Pipeline chạy thành công vào `Jenkinsfile` trong repo nộp bài. **[CẦN BỔ SUNG: bản Jenkinsfile đã kiểm tra không chứa bí mật]**.
5. Chuẩn bị VPS Ubuntu, Docker Compose và deployment script `/usr/local/sbin/lashop-deploy`; cấu hình deploy user/SSH và chỉ các quyền cần thiết.
6. Build Linux images, chuyển đến VPS, deploy bằng Compose, kiểm tra HTTPS/API.
7. Chạy monitoring stack riêng; kết nối Grafana với Prometheus, tạo Dashboard và test HTTP probes.
8. Quay video toàn bộ luồng thay đổi code → Git push → Jenkins auto build → VPS update → Telegram → Grafana.

> Không đăng mật khẩu, token, private key lên Git hoặc quay rõ bí mật trong video. Các file `.env` chỉ dùng tại môi trường tương ứng.

## 8. Kết luận

Dự án Lá Shop đã minh họa quy trình CI/CD từ GitHub qua Jenkins đến triển khai Docker Compose trên VPS Ubuntu, kết hợp giám sát tài nguyên và HTTP website/API bằng Prometheus/Grafana. Việc kết hợp lint/test/build/deploy và notification hỗ trợ tự động hóa, tăng khả năng phát hiện lỗi và giảm thao tác thủ công.
