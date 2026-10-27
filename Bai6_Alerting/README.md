# Bài 6: Cảnh báo thông minh với Prometheus và Alertmanager

## Mục đích

Bộ thực hành này thêm một stack độc lập cho Bài 6, tái sử dụng Spring Boot/Maven Backend trong thư mục cha mà không sửa cấu hình của Bài 1–5. Compose tạo bốn service (`ecommerce-api`, `node-exporter`, `prometheus`, `alertmanager`) trên cùng Docker network mặc định.

Backend hiện là Spring Boot sample có Actuator và Micrometer Prometheus; đây chưa phải ứng dụng E-commerce có nghiệp vụ thực tế. Prometheus scrape `/actuator/prometheus` và Node Exporter `/metrics`.

## Cổng và file được sử dụng

- Prometheus trong container: `9090`; host: `http://localhost:9091`.
- Alertmanager trong container và host loopback: `http://localhost:9093`.
- Backend và Node Exporter chỉ expose trong Docker network, không publish cổng host.
- `docker-compose.yml` trong thư mục này là Compose duy nhất cho Bài 6. Các file Compose và Prometheus ở thư mục gốc không bị thay đổi.
- Prometheus host port `9091` được chọn vì stack Bài 4 đang dùng `127.0.0.1:9090`. Có thể chuyển thành `127.0.0.1:9090:9090` nếu đã dừng stack gốc.

## Khởi chạy bằng PowerShell

Mở PowerShell tại thư mục workspace:

```powershell
cd C:\Users\Laptop\Music\Prometheus
docker compose -p bai6-alerting -f .\Bai6_Alerting\docker-compose.yml config
docker compose -p bai6-alerting -f .\Bai6_Alerting\docker-compose.yml up --build -d
docker compose -p bai6-alerting -f .\Bai6_Alerting\docker-compose.yml ps
```

Đọc log theo service:

```powershell
docker compose -p bai6-alerting -f .\Bai6_Alerting\docker-compose.yml logs --tail=100 prometheus
docker compose -p bai6-alerting -f .\Bai6_Alerting\docker-compose.yml logs --tail=100 alertmanager
docker compose -p bai6-alerting -f .\Bai6_Alerting\docker-compose.yml logs --tail=100 node-exporter
docker compose -p bai6-alerting -f .\Bai6_Alerting\docker-compose.yml logs --tail=100 ecommerce-api
```

Hướng dẫn kiểm thử trạng thái target, rules, Alertmanager, test alert không tạo tải và reload cấu hình nằm trong [TESTING_GUIDE.md](./TESTING_GUIDE.md).

## Luật cảnh báo hệ thống

`alert_rules.yml` định nghĩa:

- `HighCPUUsage`: tỷ lệ CPU không-idle trên 90%, dựa trên `rate(...[5m])`, phải duy trì ít nhất `for: 1m`.
- `HighMemoryUsage`: tỷ lệ RAM đã dùng trên 90%, tính bằng `MemAvailable` so với `MemTotal`, duy trì ít nhất `for: 1m`.

Prometheus đánh giá rules mỗi 15 giây. Khi biểu thức đúng, cảnh báo ở trạng thái `PENDING`; chỉ sau đủ thời gian `for` theo các lần đánh giá mới chuyển `FIRING`. Cửa sổ `rate` 5 phút làm mượt CPU, còn `for` là thời gian xác nhận điều kiện. Không cố ý gây tải CPU/RAM trên máy thật để kiểm thử.

## Alertmanager và giới hạn gửi thông báo

`alertmanager.yml` định tuyến alert vào receiver `practice-ui`, gom nhóm theo alert name/severity. Receiver không cấu hình email, Slack hoặc webhook: mục tiêu là kiểm chứng alert đến Alertmanager và xem được trong UI. Điều này **không** chứng minh đã gửi thông báo ra bên ngoài. Không cần credential hoặc secret cho cấu hình demo.

## Windows Docker Desktop

Máy phát triển Windows đang dùng Docker Desktop Linux containers. Node Exporter là Linux exporter; dữ liệu phản ánh môi trường Linux mà Docker Desktop cung cấp cho container, không mặc nhiên đại diện chính xác CPU/RAM vật lý của Windows. Với Windows host, dùng Windows exporter; với VPS Linux, triển khai exporter trên VPS và giới hạn truy cập qua firewall/VPN.

## Bảo mật và dọn stack

Prometheus reload API được bật để thực hành, nhưng chỉ publish Prometheus trên loopback. Alertmanager UI cũng bind loopback. Không publish cổng Node Exporter/Backend và không mở các UI quản trị ra Internet.

Dừng stack Bài 6 (không xóa named volume):

```powershell
docker compose -p bai6-alerting -f .\Bai6_Alerting\docker-compose.yml down
```
