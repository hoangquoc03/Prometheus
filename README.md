# Bài thực hành 1: Khắc phục lỗi scrape metrics với Prometheus và Spring Boot

## Mục tiêu

- Hiểu cấu hình scrape trong `prometheus.yml`.
- Phân biệt `localhost` bên trong container với máy host.
- Expose metrics Spring Boot Actuator/Micrometer tại `/actuator/prometheus`.
- Xác nhận Prometheus scrape backend thành công qua Docker Compose.

## Cấu trúc project

```text
.
|-- Dockerfile
|-- docker-compose.yml
|-- pom.xml
|-- prometheus.yml
|-- README.md
`-- src
    |-- main
    |   |-- java\com\example\prometheusdemo\PrometheusDemoApplication.java
    |   `-- resources\application.yml
    `-- test
        `-- java\com\example\prometheusdemo\PrometheusDemoApplicationTests.java
```

Project được tạo mới cho Java 21 (đã có trên máy) và Maven 3.9.11 (đã có trên máy). Spring Boot 3.5.7 quản lý phiên bản tương thích cho Actuator và Micrometer; không khai báo phiên bản riêng cho các dependency đó. Docker Desktop Engine cũng đã phản hồi lệnh `docker version` tại thời điểm kiểm tra môi trường. Ở Bài 1–3, backend từng được publish trên `127.0.0.1:8081` do host port `8080` đã được sử dụng; trong cấu hình hiện tại của Bài 4 backend không publish cổng host.

> **Cấu hình hiện tại sau Bài 4:** Docker Compose dùng service `ecommerce-api`, `prometheus` và `node-exporter`; backend chỉ expose `8080` trong network nội bộ, dashboard Prometheus bind tại `127.0.0.1:9090`. Những ví dụ cấu hình/lệnh của Bài 1–3 bên dưới phản ánh các bước trước đó; để chạy stack hiện tại, dùng mục **Bài 4** cuối tài liệu.

## Phân tích cấu hình và nguyên nhân lỗi

- `scrape_configs` khai báo các job mà Prometheus định kỳ lấy metrics.
- `job_name` đặt nhãn nhận diện job trong Prometheus, ví dụ `backend`.
- `static_configs` khai báo các target cố định thay vì phát hiện service động.
- `targets` là địa chỉ `host:port` mà Prometheus kết nối tới.
- `localhost` bên trong Prometheus container trỏ về chính container Prometheus, không phải backend hay máy host. Vì vậy `localhost:8080` không đúng khi backend ở container khác; trong Compose, dùng tên service `backend:8080`.
- `connection refused` nghĩa là không có dịch vụ chấp nhận kết nối tại địa chỉ/cổng đó (hoặc kết nối bị từ chối). Đó là lỗi kết nối/cổng, khác với HTTP `404` khi đã kết nối nhưng URL không tồn tại. `401/403` báo endpoint bị giới hạn quyền; `timeout` thường chỉ vấn đề định tuyến, firewall hoặc kết nối bị chặn.
- Prometheus mặc định lấy `/metrics`, nhưng Spring Boot Actuator cùng Micrometer Prometheus cung cấp `/actuator/prometheus`. Do đó file này đặt `metrics_path` đúng với endpoint backend. `metrics_path` không thể sửa lỗi kết nối mạng hoặc cổng.

## Các file cấu hình chính

### `prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "backend"
    metrics_path: "/actuator/prometheus"
    static_configs:
      - targets: ["backend:8080"]
```

Trong cấu hình lịch sử Bài 1, `backend` khớp tên service khi đó. Cấu hình hiện tại ở mục Bài 4 dùng `ecommerce-api` làm tên service và target. Hai container giao tiếp bằng DNS service trên network mặc định do Compose tạo; không dùng `localhost:8080` trong Prometheus.

### `docker-compose.yml`

Trong cấu hình hiện tại của Bài 4, Compose build app từ `Dockerfile`, chỉ expose backend trong network Compose và bind giao diện Prometheus trên `127.0.0.1:9090`; cấu hình Prometheus được mount read-only. Cổng target `ecommerce-api:8080` là cổng nội bộ; không có publish backend lên host.

### Backend và dependencies

`pom.xml` sử dụng Java 21, Spring Boot 3.5.7, `spring-boot-starter-actuator`, `spring-boot-starter-web`, và `micrometer-registry-prometheus`. Phiên bản dependency được Spring Boot parent quản lý.

`src/main/resources/application.yml` expose `health`, `info` và `prometheus` qua HTTP; các thông tin chi tiết health không được hiển thị. Không expose `env`, không bật hoặc expose `shutdown`, và không mở toàn bộ endpoint bằng `*`. Compose chỉ publish cổng host trên loopback; Prometheus trong Compose vẫn scrape backend qua network nội bộ.

### Node Exporter (Bài 3)

Compose có thêm `node-exporter` (image pin `prom/node-exporter:v1.9.1`). Node Exporter expose cổng `9100` chỉ trên network Compose; không publish `9100` trên host. Prometheus scrape job `node` qua `node-exporter:9100`, endpoint `/metrics` mặc định.

Mount `/proc` và `/sys` vào đường dẫn container chỉ đọc để exporter đọc procfs/sysfs mà Docker daemon cung cấp; mount `/` vào `/host/rootfs` chỉ đọc để collector filesystem quan sát root filesystem. Không dùng propagation `rslave` vì Docker Desktop Engine báo root mount của nó không phải shared/slave. Tùy chọn `--path.procfs`, `--path.sysfs` và `--path.rootfs` chỉ rõ vị trí các cây hệ thống bên trong exporter. Các mount này không đảm bảo mọi thiết bị/phần cứng được thu thập.

Máy phát triển là Windows chạy Docker Desktop Linux containers. Node Exporter Linux trong môi trường này quan sát Linux VM/môi trường Docker mà daemon cung cấp, không tự động đại diện chính xác toàn bộ phần cứng Windows. Muốn giám sát Windows, dùng Windows exporter; muốn giám sát VPS Linux thật, cài Node Exporter trên chính VPS và cấu hình Prometheus truy cập qua mạng riêng/VPN hoặc firewall giới hạn nguồn. Không public cổng `9100` ra Internet.

Kiểm tra job mới sau khi khởi động Compose:

```powershell
docker compose config
docker compose up -d
docker compose logs --tail=100 node-exporter
docker compose exec -T prometheus promtool check config /etc/prometheus/prometheus.yml
Start-Sleep -Seconds 20
Invoke-RestMethod 'http://localhost:9090/api/v1/targets'
Invoke-RestMethod 'http://localhost:9090/api/v1/query?query=up%7Bjob%3D%22node%22%7D'
```

Sau khi cập nhật `prometheus.yml`, hãy đảm bảo Prometheus đã nạp cấu hình mới; nếu target `node` chưa xuất hiện, chạy `docker compose up -d --force-recreate prometheus` rồi kiểm tra lại `/targets`.

## Bài 4: Liên minh Docker Compose – Prometheus + App

Project không có nghiệp vụ E-commerce (không có sản phẩm, giỏ hàng hay đơn hàng). Bài này tái sử dụng Spring Boot sample có Actuator, đặt tên Compose service là `ecommerce-api` để thực hành service discovery/scrape; không giả định có image E-commerce riêng.

Compose hiện build image `ecommerce-api:local` từ Dockerfile hiện có. Service chỉ expose `8080` trên Docker network, không publish backend ra host. Prometheus bind `127.0.0.1:9090`, mount `prometheus.yml` read-only, lưu dữ liệu trong named volume `prometheus_data` và bật `--web.enable-lifecycle`. Job `prometheus`, `ecommerce-api` và `node` dùng Docker DNS trên network mặc định.

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

Sau khi cập nhật file trên Host, kiểm tra cú pháp trước khi reload. Bind mount `./prometheus.yml:/etc/prometheus/prometheus.yml:ro` giúp container đọc file ngoài image nhưng không tự reload cấu hình. Gọi lifecycle API vì `--web.enable-lifecycle` đã bật; API reload được giới hạn host qua bind `127.0.0.1:9090`:

```powershell
curl.exe -X POST http://localhost:9090/-/reload
docker compose logs --tail=50 prometheus
```

API lifecycle là giao diện quản trị không xác thực mặc định; không đưa cổng 9090 ra Internet. Nếu cần truy cập dashboard từ xa, dùng VPN hoặc SSH tunnel; nếu cần chia sẻ cho nhiều người dùng, đặt sau reverse proxy có xác thực và giới hạn mạng.

Kiểm tra scrape:

```powershell
Invoke-RestMethod 'http://localhost:9090/api/v1/targets'
Invoke-RestMethod 'http://localhost:9090/api/v1/query?query=up%7Bjob%3D%22ecommerce-api%22%7D'
Invoke-RestMethod 'http://localhost:9090/api/v1/query?query=up%7Bjob%3D%22node%22%7D'
```

Mở `http://localhost:9090/targets` và xác nhận các job `ecommerce-api`, `prometheus`, `node` ở trạng thái `UP`. Backend không còn publish cổng host, vì vậy URL `http://localhost:8080/actuator/health` không áp dụng; kiểm tra endpoint nội bộ từ container Prometheus:

```powershell
docker compose exec -T prometheus wget -qO- http://ecommerce-api:8080/actuator/health
docker compose exec -T prometheus wget -qO- http://ecommerce-api:8080/actuator/prometheus
```

`ecommerce-api:8080` phân giải qua DNS Docker Compose. `localhost:8080` bên trong container Prometheus trỏ về chính Prometheus, không phải backend.

## Chạy bằng PowerShell

Mở PowerShell trong thư mục project:

```powershell
Set-Location 'C:\Users\Laptop\Music\Prometheus'
docker compose config
docker compose up --build -d
docker compose ps
```

`docker compose config` kiểm tra cấu hình Compose; lệnh đó không thay thế kiểm tra cấu hình Prometheus. Trong cấu hình Bài 4 backend không publish lên host, nên kiểm tra endpoint qua Docker network:

```powershell
docker compose exec -T prometheus wget -qO- http://ecommerce-api:8080/actuator/health
docker compose exec -T prometheus wget -qO- http://ecommerce-api:8080/actuator/prometheus
```

Health dự kiến trả trạng thái `UP`. Prometheus endpoint trả metrics dạng văn bản. Việc tìm tên metric phụ thuộc phiên bản/runtime.

Kiểm tra log và cấu hình Prometheus trong container:

```powershell
docker compose logs --tail 100 ecommerce-api prometheus node-exporter
docker compose exec prometheus promtool check config /etc/prometheus/prometheus.yml
```

Nếu `promtool check config` thành công, công cụ in thông báo `SUCCESS`. Tiếp theo mở:

- `http://localhost:9090/targets` — các job `ecommerce-api`, `prometheus`, `node` phải có trạng thái `UP`.
- `http://localhost:9090/graph` — nhập `up{job="ecommerce-api"}`; kết quả `1` là scrape thành công, `0` là scrape thất bại.

Có thể kiểm tra query qua API bằng PowerShell:

```powershell
Invoke-RestMethod 'http://localhost:9090/api/v1/query?query=up%7Bjob%3D%22ecommerce-api%22%7D'
```

Prometheus scrape theo chu kỳ 15 giây; đợi ít nhất một chu kỳ sau khi hai service sẵn sàng rồi kiểm tra `/targets`.

## Chẩn đoán sự cố

| Kết quả | Ý nghĩa và kiểm tra |
|---|---|
| `connection refused` | Sai service/port, backend chưa khởi động hoặc không listen. Kiểm tra `docker compose ps` và log. |
| `timeout` | Kiểm tra network hoặc kết nối bị chặn. Với cấu hình này, xác nhận cả hai service đang dùng network Compose mặc định. |
| HTTP `404` | Kết nối đã tới backend nhưng sai path; kiểm tra `metrics_path`, `server.port`, context path và exposure. |
| HTTP `401`/`403` | Endpoint bị Spring Security hoặc proxy bảo vệ; kiểm tra cấu hình security. Project mẫu không thêm Spring Security. |
| HTTP `200` nhưng target chưa `UP` | Xem lỗi scrape chi tiết trên `/targets`, kiểm tra log Prometheus và đợi chu kỳ scrape. |

## Kết quả mong đợi và ảnh chứng minh

Sau khi chạy được stack, chụp ảnh thật (không dùng ảnh minh họa thay kết quả) của:

1. `/targets` hiển thị job `backend` là `UP`.
2. Trang Graph với truy vấn `up{job="backend"}` có kết quả `1`.
3. PowerShell hoặc trình duyệt thể hiện health `UP` và endpoint Prometheus trả metrics.

Nếu chưa chạy các lệnh, đây là kết quả mong đợi chứ chưa phải kết quả đã xác minh. Docker Engine đã được phát hiện hoạt động trong lần kiểm tra môi trường ban đầu; trạng thái build và chạy project chỉ được xác nhận sau khi thực thi các lệnh bên trên.

## Dừng project

```powershell
docker compose down
```

## Checklist hoàn thành

- [ ] `docker compose config` hoàn tất không lỗi.
- [ ] `docker compose up --build -d` khởi động được cả `backend` và `prometheus`.
- [ ] `/actuator/health` trả HTTP `200` và trạng thái `UP`.
- [ ] `/actuator/prometheus` trả HTTP `200` và nội dung metrics.
- [ ] `promtool check config` xác nhận file Prometheus hợp lệ.
- [ ] `/targets` báo job `backend` là `UP`.
- [ ] `up{job="backend"}` trả giá trị `1`.
- [ ] Đã lưu ảnh chụp màn hình thật làm bằng chứng.

## Kết luận

Trong Docker, `localhost` luôn chỉ container hiện tại. Các container trong cùng Compose network nên giao tiếp bằng tên service và cổng container. Ngoài ra, cần phân biệt lỗi không kết nối được tới host/port với lỗi HTTP ở endpoint: sửa `metrics_path` chỉ giải quyết trường hợp sai đường dẫn sau khi đã kết nối được.
