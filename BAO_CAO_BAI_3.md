# BÁO CÁO BÀI 3: KHÁM SỨC KHỎE MÁY CHỦ VỚI NODE EXPORTER

> Lưu ý cập nhật sau Bài 4: báo cáo này ghi lại cấu hình và service names tại thời điểm thực hiện Bài 3. Compose hiện tại đổi tên service backend thành `ecommerce-api` và Prometheus có các job `prometheus`, `ecommerce-api`, `node`; Node Exporter vẫn được giữ nguyên. Xem [BAO_CAO_BAI_4.md](./BAO_CAO_BAI_4.md) để biết cấu hình đang dùng.

## 1. Tên bài thực hành

**Bài 3: Vận dụng chuyên sâu – Khám sức khỏe máy chủ với Node Exporter**

## 2. Mục tiêu

- Triển khai Node Exporter bằng Docker Compose cùng Prometheus hiện có.
- Cấu hình Node Exporter đọc các mount hệ thống `/proc`, `/sys` và root filesystem ở chế độ chỉ đọc.
- Thêm job `node` mà không làm mất job `backend`.
- Xác nhận Prometheus scrape được Node Exporter.
- Nhận biết giới hạn đo đạc khi Docker Desktop chạy Linux container trên Windows và rủi ro khi mở cổng metrics.

## 3. Tình huống và môi trường đã kiểm tra

Project đã có [docker-compose.yml](./docker-compose.yml) và [prometheus.yml](./prometheus.yml) của các bài trước. Prometheus service đang dùng image `prom/prometheus:v3.5.0`; backend service tên `backend`, lắng nghe `8080` nội bộ và được publish host tại `127.0.0.1:8081`. Trước khi thêm Node Exporter, backend health là `UP` và target `backend` của Prometheus ở trạng thái `up`.

Máy làm việc là Windows 11, Docker context `desktop-linux`, Docker Engine báo `OSType=linux` và hệ điều hành Docker Desktop. Đây không phải Docker Engine Linux cài trực tiếp trên VPS/host Linux. Node Exporter chạy trong Linux container có thể đọc môi trường Linux VM mà Docker Desktop cung cấp; số liệu đó không phải đại diện đầy đủ/chính xác cho toàn bộ phần cứng Windows. Nếu cần theo dõi Windows, chọn Windows exporter. Nếu cần metrics của VPS Linux, triển khai exporter trên chính VPS Linux và giới hạn kết nối bằng VPN/firewall hoặc mạng riêng.

## 4. Kiến trúc Prometheus – Node Exporter

```text
Windows host
  └─ Docker Desktop Linux VM / Compose network
       ├─ prometheus :9090 (host publish loopback)
       ├─ backend :8080 (host publish loopback :8081)
       └─ node-exporter :9100 (chỉ expose nội bộ Compose)
              └─ /metrics

Prometheus job node ── HTTP nội bộ ──> node-exporter:9100/metrics
Prometheus job backend ── HTTP nội bộ ──> backend:8080/actuator/prometheus
```

Ba container được kết nối với network mặc định do Docker Compose tạo. `expose: 9100` ghi nhận cổng nội bộ cho giao tiếp container; không publish cổng 9100 lên host. Không tạo Prometheus service thứ hai.

## 5. `docker-compose.yml` hoàn chỉnh sau khi sửa

```yaml
services:
  backend:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "127.0.0.1:8081:8080"

  prometheus:
    image: prom/prometheus:v3.5.0
    ports:
      - "127.0.0.1:9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    depends_on:
      - backend

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
```

Không có khai báo `ports` cho Node Exporter. Backend host port `8081` được giữ nguyên vì cổng host `8080` đã được tiến trình khác sử dụng. Backend vẫn là `backend:8080` trên network Compose.

## 6. `prometheus.yml` hoàn chỉnh sau khi sửa

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "backend"
    metrics_path: "/actuator/prometheus"
    static_configs:
      - targets: ["backend:8080"]

  - job_name: "node"
    static_configs:
      - targets: ["node-exporter:9100"]
```

Job `backend` được giữ nguyên. Node Exporter mặc định cung cấp metrics tại `/metrics`, vì vậy job `node` không cần `metrics_path` riêng. Target service DNS `node-exporter:9100` hoạt động vì hai container cùng Compose network.

## 7. Giải thích volume và tham số

- `/proc:/host/proc:ro`: mount procfs từ môi trường mà Docker daemon cung cấp vào `/host/proc` bên trong container. `/proc` chứa thông tin tiến trình và một số thống kê hệ thống. `--path.procfs=/host/proc` báo cho exporter đọc ở vị trí mount này.
- `/sys:/host/sys:ro`: mount sysfs vào `/host/sys`; sysfs cung cấp thông tin kernel và thiết bị. `--path.sysfs=/host/sys` chỉ vị trí đó.
- `/:/host/rootfs:ro`: mount root filesystem mà daemon cung cấp vào `/host/rootfs`, chỉ đọc. `--path.rootfs=/host/rootfs` giúp collector filesystem phân giải filesystem từ root được mount.
- `:ro` giữ mount read-only để exporter không thể ghi qua các mount đó. Ban đầu đã thử `rslave` theo mẫu Linux, nhưng Docker Desktop Engine báo `/` không phải shared/slave mount; vì vậy bỏ propagation option để cấu hình chạy được trong môi trường thực tế này. Trên Linux host thật, chỉ thêm propagation nếu yêu cầu mount propagation và host mount được cấu hình phù hợp.

Các mount không đồng nghĩa mọi thiết bị hay loại phần cứng đều được giám sát. Metrics phụ thuộc collector được bật, quyền của container, filesystem và môi trường Linux mà Docker daemon thực sự cung cấp. Trên máy này, đó là Linux environment của Docker Desktop, không phải Windows kernel.

## 8. Cổng 9100, endpoint và an toàn

Node Exporter mặc định lắng nghe HTTP trên cổng `9100` và xuất metrics tại `/metrics`. Metrics có thể tiết lộ thông tin hệ thống như CPU, RAM, filesystem, thiết bị và hoạt động máy. Node Exporter mặc định không cung cấp xác thực bằng mật khẩu; mở cổng `9100` ra Internet có thể cho phép người ngoài thu thập thông tin vận hành.

Trong cấu hình bài lab, cổng `9100` chỉ được expose cho mạng Compose, không bind vào host và không public ra Internet. Nếu cần truy cập an toàn từ máy quản trị, ưu tiên mạng nội bộ/VPN hoặc SSH tunnel; nếu buộc phải publish, chỉ bind loopback trên chính máy chủ và kiểm soát firewall. Với VPS, tránh dùng `0.0.0.0:9100` công khai.

## 9. Chạy và kiểm thử

Chạy các lệnh tại thư mục chứa `docker-compose.yml`:

```powershell
Set-Location 'C:\Users\Laptop\Music\Prometheus'
docker compose config
docker compose up -d
docker compose ps
docker compose logs --tail=100 prometheus
docker compose logs --tail=100 node-exporter
docker compose exec -T prometheus promtool check config /etc/prometheus/prometheus.yml
```

Đợi ít nhất một chu kỳ scrape (15 giây), sau đó kiểm tra API Prometheus:

```powershell
Invoke-RestMethod 'http://localhost:9090/api/v1/targets'
Invoke-RestMethod 'http://localhost:9090/api/v1/query?query=up%7Bjob%3D%22node%22%7D'
```

Hoặc mở `http://localhost:9090/targets` và xác nhận job `node` là `UP`; tại `http://localhost:9090/graph`, chạy:

```promql
up{job="node"}
```

Giá trị `1` nghĩa là scrape thành công. Có thể thử các metric nếu collector/môi trường cung cấp:

```promql
node_cpu_seconds_total
node_memory_MemAvailable_bytes
node_filesystem_avail_bytes
```

`node_cpu_seconds_total` là thời gian CPU tích lũy theo chế độ; `node_memory_MemAvailable_bytes` là lượng RAM ước tính còn dùng được; `node_filesystem_avail_bytes` là dung lượng filesystem còn khả dụng. Giá trị và filesystem xuất hiện phụ thuộc môi trường.

## 10. Kết quả kiểm tra

Kết quả trước khi thêm Node Exporter: Docker context là `desktop-linux`; Docker Engine là Linux của Docker Desktop; Prometheus và backend Compose đang chạy, backend health `UP`, và Prometheus target `backend` `up`. Lần khởi động đầu tiên bị Docker từ chối vì tùy chọn `rslave`; đã bỏ tùy chọn đó, khởi động lại và xác minh thành công.

Sau khi bỏ `rslave` theo yêu cầu của Docker Desktop, Node Exporter v1.9.1 chạy ổn định; `docker compose ps` hiển thị service `node-exporter` Up và chỉ có cổng nội bộ `9100/tcp`, không publish ra host. `docker compose config` hợp lệ và `promtool check config` báo `SUCCESS`. Prometheus API báo:

- Job `backend`: `up`, scrape URL `http://backend:8080/actuator/prometheus`.
- Job `node`: `up`, scrape URL `http://node-exporter:9100/metrics`.
- `up{job="node"}`: `1`.
- Query `node_cpu_seconds_total`: 96 series; `node_memory_MemAvailable_bytes`: 1 series; `node_filesystem_avail_bytes`: 52 series.

Log Node Exporter có thông báo mức ERROR từ collector `diskstats` vì không đọc được `/run/udev/data`; phần udev device properties bị tắt, nhưng exporter vẫn khởi động, lắng nghe `:9100` và scrape thành công. Các kết quả này là từ Linux environment Docker Desktop và không đại diện cho toàn bộ phần cứng Windows.

## 11. Xử lý sự cố target `DOWN`

1. Kiểm tra `docker compose ps` xem `node-exporter` có chạy không.
2. Đọc `docker compose logs --tail=100 node-exporter`; kiểm tra lỗi permission hoặc mount.
3. Xác nhận Prometheus và exporter có cùng Compose network.
4. Đảm bảo target là `node-exporter:9100` và tên service chính xác.
5. Đảm bảo exporter lắng nghe cổng mặc định `9100`.
6. Chạy `docker compose exec -T prometheus promtool check config /etc/prometheus/prometheus.yml`.
7. Xem lỗi chi tiết trên `/targets`: `connection refused` thường là cổng không có listener; timeout có thể do routing/firewall; lỗi DNS thường do khác network hoặc sai service name; lỗi cấu hình thường phát hiện bằng promtool/log Prometheus.

## 12. Checklist hoàn thành

- [x] Kiểm tra OS, Docker Engine, Compose services, port backend và Prometheus config trước khi sửa.
- [x] Giữ nguyên backend, Prometheus hiện có và job `backend`.
- [x] Thêm Node Exporter image pin `v1.9.1`.
- [x] Mount `/proc`, `/sys` và rootfs chỉ đọc; không publish cổng `9100`.
- [x] Thêm job `node` trỏ service trong cùng Compose network.
- [x] `docker compose config` và `promtool check config` thành công sau khi thêm service/job.
- [x] Container Node Exporter chạy ổn định.
- [x] `/targets` báo job `node` là `UP`.
- [x] `up{job="node"}` trả `1`.
- [x] Ghi rõ Docker Desktop Linux không đại diện đầy đủ cho hệ điều hành/phần cứng Windows.
