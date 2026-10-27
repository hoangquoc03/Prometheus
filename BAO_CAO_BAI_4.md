# BÁO CÁO BÀI 4: LIÊN MINH DOCKER COMPOSE – PROMETHEUS + E-COMMERCE APP

## 1. Tên bài thực hành

**Bài 4: Vận dụng chuyên sâu – Liên minh Docker Compose (Prometheus + App)**

## 2. Mục tiêu

- Dùng Docker Compose để build và chạy backend cùng Prometheus.
- Giao tiếp nội bộ qua DNS service trên Docker network mặc định.
- Scrape Actuator endpoint bằng `ecommerce-api:8080`.
- Mount file Prometheus từ Host ở chế độ chỉ đọc, lưu trữ dữ liệu Prometheus và reload cấu hình qua lifecycle API.
- Giữ Node Exporter của Bài 3 mà không public cổng `9100`.

## 3. Kiểm tra hiện trạng và phạm vi ứng dụng

Workspace đã có Spring Boot sample, `Dockerfile`, Docker Compose, Prometheus config, Actuator/Micrometer và Node Exporter. Ứng dụng dùng Java 21, Spring Boot 3.5.7, Maven; Docker Engine đang chạy Linux qua Docker Desktop trên Windows 11. Trước khi sửa, Prometheus scrape backend và Node Exporter đều `UP`, backend health `UP`.

**Đây không phải ứng dụng E-commerce đầy đủ:** mã nguồn không có nghiệp vụ sản phẩm, giỏ hàng hay đơn hàng. Để không tạo project trùng lặp hoặc giả vờ có image e-commerce, bài thực hành tái sử dụng Spring Boot sample hiện có và đặt tên Compose service `ecommerce-api` để hoàn thành phần tích hợp networking/scrape. Docker image được build từ Dockerfile trong workspace, không dùng image giả định có sẵn.

Backend lắng nghe `0.0.0.0:8080` trong container. Compose chỉ expose cổng nội bộ `8080`; không publish backend ra host. Prometheus được publish riêng trên `127.0.0.1:9090`. Node Exporter tiếp tục chạy trong network nội bộ, chỉ expose `9100`.

## 4. Kiến trúc Docker Compose

```text
Host (Windows + Docker Desktop Linux Engine)
  └─ Docker Compose default network
       ├─ ecommerce-api:8080
       ├─ prometheus:9090 ── bind host 127.0.0.1:9090
       └─ node-exporter:9100 (nội bộ, không publish)

Prometheus ──> ecommerce-api:8080/actuator/prometheus
Prometheus ──> prometheus:9090/metrics
Prometheus ──> node-exporter:9100/metrics
```

Docker Compose tạo network mặc định và DNS cho từng service. Vì vậy Prometheus truy cập backend qua `ecommerce-api:8080`. `localhost` bên trong container Prometheus là chính container Prometheus; `localhost:8080` không trỏ đến backend.

## 5. Nội dung `docker-compose.yml` hoàn chỉnh

```yaml
services:
  ecommerce-api:
    build:
      context: .
      dockerfile: Dockerfile
    image: ecommerce-api:local
    restart: unless-stopped
    expose:
      - "8080"

  prometheus:
    image: prom/prometheus:v3.5.0
    restart: unless-stopped
    ports:
      - "127.0.0.1:9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--web.enable-lifecycle"
    depends_on:
      - ecommerce-api
      - node-exporter

  node-exporter:
    image: prom/node-exporter:v1.9.1
    restart: unless-stopped
    command:
      - "--path.procfs=/host/proc"
      - "--path.sysfs=/host/sys"
      - "--path.rootfs=/host/rootfs"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/host/rootfs:ro
    expose:
      - "9100"

volumes:
  prometheus_data:
```

`expose` dùng cho giao tiếp nội bộ; không mở backend hay Node Exporter trên interface host. `prometheus_data` giữ dữ liệu TSDB qua vòng đời container.

## 6. Nội dung `prometheus.yml` hoàn chỉnh

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["prometheus:9090"]

  - job_name: "ecommerce-api"
    metrics_path: "/actuator/prometheus"
    static_configs:
      - targets: ["ecommerce-api:8080"]

  - job_name: "node"
    static_configs:
      - targets: ["node-exporter:9100"]
```

Job `ecommerce-api` đặt đúng metrics path của Actuator. Job `node` được giữ từ Bài 3 và dùng endpoint mặc định `/metrics`. Không khai báo job trùng lặp.

## 7. Bind mount file cấu hình

```yaml
- ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
```

- `./prometheus.yml` là file trên Host, tương đối theo thư mục chứa file Compose.
- `/etc/prometheus/prometheus.yml` là đường dẫn bên trong container Prometheus.
- `:ro` nghĩa là container chỉ đọc file nguồn.
- Bind mount tách cấu hình khỏi image, thuận tiện chỉnh file trên Host.

File nguồn phải tồn tại và YAML phải hợp lệ. Sửa file trên Host không tự làm Prometheus nạp lại cấu hình; mount cập nhật nội dung nhìn thấy từ container trong điều kiện bind mount hỗ trợ, nhưng Prometheus cần được reload. Không sửa file `:ro` bên trong container.

## 8. Build, khởi động và kiểm tra

Chạy tại đúng thư mục project:

```powershell
Set-Location 'C:\Users\Laptop\Music\Prometheus'
docker compose config
docker compose up --build -d
docker compose ps
docker compose logs --tail=100 ecommerce-api
docker compose logs --tail=100 prometheus
docker compose logs --tail=100 node-exporter
docker compose exec -T prometheus promtool check config /etc/prometheus/prometheus.yml
```

Dashboard truy cập tại `http://localhost:9090`; targets tại `http://localhost:9090/targets`. Vì backend không publish port host, `http://localhost:8080/...` không dùng để kiểm tra. Có thể kiểm tra từ chính network bằng container Prometheus:

```powershell
docker compose exec -T prometheus wget -qO- http://ecommerce-api:8080/actuator/health
docker compose exec -T prometheus wget -qO- http://ecommerce-api:8080/actuator/prometheus
```

Hoặc dùng API Prometheus:

```powershell
Invoke-RestMethod 'http://localhost:9090/api/v1/targets'
Invoke-RestMethod 'http://localhost:9090/api/v1/query?query=up%7Bjob%3D%22ecommerce-api%22%7D'
```

`up{job="ecommerce-api"}` trả `1` nghĩa là scrape thành công. Nếu target `DOWN`, xem lỗi trên `/targets`, kiểm tra log hai service, DNS service name, network mặc định, cổng nội bộ `8080` và endpoint Actuator.

## 9. Kết quả kiểm thử thực tế

Trạng thái trước khi cập nhật Compose: service `backend` và `node-exporter` đã chạy; backend health là `UP`; Docker Engine Linux của Docker Desktop đang hoạt động.

Sau khi chỉnh sửa và chạy `docker compose up --build -d --remove-orphans`:

- `docker compose config`: hợp lệ; service là `ecommerce-api`, `prometheus`, `node-exporter`.
- Image `ecommerce-api:local` build thành công từ Dockerfile hiện có. Compose dọn container orphan `prometheus-backend-1` sau khi service được đổi tên.
- `docker compose ps`: cả ba service chạy; `ecommerce-api` chỉ có `8080/tcp`, Node Exporter chỉ có `9100/tcp`; duy nhất Prometheus bind host `127.0.0.1:9090`.
- Named volume `prometheus_prometheus_data` được tạo và gắn vào `/prometheus`.
- `docker compose exec -T prometheus promtool check config /etc/prometheus/prometheus.yml`: `SUCCESS`.
- Qua network Compose, `/actuator/health` trả `{"status":"UP"}`; `/actuator/prometheus` trả text có `# HELP` và metric `jvm_memory_used_bytes`.
- Prometheus API báo `prometheus`, `ecommerce-api`, `node` đều `up`, lần lượt scrape từ `prometheus:9090/metrics`, `ecommerce-api:8080/actuator/prometheus`, `node-exporter:9100/metrics`.
- `up{job="ecommerce-api"}` trả `1`.
- Trong lần rà soát sau đổi tên service đã phát hiện job `backend` cũ còn trong YAML; xóa job lỗi thời, chạy lại `promtool` và POST reload. API reload trả HTTP `200`, log ghi nhận cấu hình được nạp lại, target `backend` cũ không còn; cả ba target hiện tại vẫn `up`.

Lưu ý: đây là xác nhận tích hợp monitoring trên Spring Boot sample được đặt tên Compose service `ecommerce-api`, không phải xác nhận tính năng nghiệp vụ E-commerce.

## 10. Reload cấu hình Prometheus

Compose truyền cờ `--web.enable-lifecycle`, bật API reload. Sau khi sửa `prometheus.yml` trên Host, trước tiên kiểm tra cú pháp:

```powershell
docker compose exec -T prometheus promtool check config /etc/prometheus/prometheus.yml
```

Nếu hợp lệ, gọi API reload:

```powershell
Invoke-WebRequest -UseBasicParsing `
  -Method POST `
  -Uri "http://localhost:9090/-/reload"
```

Hoặc:

```powershell
curl.exe -X POST http://localhost:9090/-/reload
```

API thường trả HTTP `200` khi reload được xử lý thành công. Kiểm tra log:

```powershell
docker compose logs --tail=50 prometheus
```

Nếu cấu hình mới không hợp lệ, Prometheus có thể từ chối cấu hình đó và tiếp tục dùng cấu hình cũ. Nếu bind mount không phản ánh thay đổi trong môi trường Docker hiện tại, xác minh mount và cân nhắc recreate service:

```powershell
docker compose up -d --force-recreate prometheus
```

## 11. Bảo mật

- Prometheus chỉ bind `127.0.0.1:9090`, phù hợp truy cập từ máy Host hoặc thông qua SSH tunnel; không tự mở firewall.
- Backend không publish cổng host chỉ để Prometheus scrape.
- Node Exporter `9100` không publish cổng host.
- Lifecycle reload là API quản trị không xác thực mặc định. Chỉ bind Prometheus trên loopback và bảo vệ máy host; không phơi cổng 9090 ra Internet khi lifecycle được bật.
- Truy cập từ xa nên qua VPN/SSH tunnel. Nếu cần nhiều người dùng, dùng reverse proxy có xác thực và giới hạn mạng.

## 12. Kết luận và checklist

- [x] Kiểm tra và tái sử dụng project Spring Boot/Prometheus/Node Exporter hiện có.
- [x] Docker Compose dùng service `ecommerce-api` và build image từ Dockerfile hiện có.
- [x] Backend và Prometheus giao tiếp qua network mặc định, không dùng `localhost` cho target backend.
- [x] Prometheus mount config read-only, lưu TSDB bằng named volume và bật lifecycle reload.
- [x] Prometheus UI chỉ bind loopback; backend và Node Exporter không publish host ports.
- [x] `docker compose config`, build/start, promtool và các target đã được xác nhận bằng kết quả runtime.
- [x] Reload API POST trả HTTP `200`.
- [x] Ghi rõ project hiện tại là Spring Boot sample, không có nghiệp vụ E-commerce.
