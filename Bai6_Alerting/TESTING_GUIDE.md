# Hướng dẫn kiểm thử Bài 6 (Windows PowerShell)

Chạy các lệnh từ thư mục workspace `C:\Users\Laptop\Music\Prometheus`. Compose file của Bài 6 là `.\Bai6_Alerting\docker-compose.yml`; project name cố định `bai6-alerting`.

## 1. Kiểm tra cấu hình và cổng

```powershell
docker compose -p bai6-alerting -f .\Bai6_Alerting\docker-compose.yml config
docker compose -p bai6-alerting -f .\Bai6_Alerting\docker-compose.yml ps
Test-NetConnection localhost -Port 9090
Test-NetConnection localhost -Port 9091
Test-NetConnection localhost -Port 9093
```

`docker compose config` kiểm tra cấu trúc Compose, nhưng không thay thế kiểm tra cú pháp Prometheus/Alertmanager. Stack Bài 4 hiện có thể đang dùng cổng `9090`; Bài 6 dùng `9091` để không dừng hoặc ghi đè stack cũ.

Kiểm tra rules bằng đúng image Prometheus và đúng file mount (lệnh chạy từ workspace):

```powershell
docker compose -p bai6-alerting -f .\Bai6_Alerting\docker-compose.yml run --rm --no-deps --entrypoint=/bin/promtool prometheus check config /etc/prometheus/prometheus.yml
docker compose -p bai6-alerting -f .\Bai6_Alerting\docker-compose.yml run --rm --no-deps --entrypoint=/bin/promtool prometheus check rules /etc/prometheus/alert_rules.yml
```

Kiểm tra Alertmanager config:

```powershell
docker compose -p bai6-alerting -f .\Bai6_Alerting\docker-compose.yml run --rm --no-deps --entrypoint=/bin/amtool alertmanager check-config /etc/alertmanager/alertmanager.yml
```

Nếu một image/tag không có công cụ hoặc Docker không thể bind mount đường dẫn Windows, xem thông báo lỗi. Sau khi stack chạy, log khởi động cũng cho biết config có được nạp hay không; không suy luận rằng cấu hình hợp lệ chỉ từ `docker compose config`.

## 2. Build và khởi chạy

```powershell
docker compose -p bai6-alerting -f .\Bai6_Alerting\docker-compose.yml up --build -d
docker compose -p bai6-alerting -f .\Bai6_Alerting\docker-compose.yml ps
```

Đọc log khi cần:

```powershell
docker compose -p bai6-alerting -f .\Bai6_Alerting\docker-compose.yml logs --tail=100 prometheus
docker compose -p bai6-alerting -f .\Bai6_Alerting\docker-compose.yml logs --tail=100 alertmanager
docker compose -p bai6-alerting -f .\Bai6_Alerting\docker-compose.yml logs --tail=100 node-exporter
docker compose -p bai6-alerting -f .\Bai6_Alerting\docker-compose.yml logs --tail=100 ecommerce-api
```

## 3. Targets, rules và giao diện

Mở:

- Prometheus: `http://localhost:9091`
- Targets: `http://localhost:9091/targets`
- Rules: `http://localhost:9091/rules`
- Alertmanager: `http://localhost:9093`

Hoặc kiểm tra endpoint HTTP bằng PowerShell:

```powershell
Invoke-WebRequest -UseBasicParsing http://localhost:9091/-/healthy
Invoke-WebRequest -UseBasicParsing http://localhost:9093/-/healthy
Invoke-RestMethod http://localhost:9091/api/v1/targets
Invoke-RestMethod http://localhost:9091/api/v1/rules
```

Mong đợi các target `prometheus`, `ecommerce-api`, `node-exporter` ở trạng thái `UP`, và nhóm `system_alerts` có hai rule đã nạp. Target `node-exporter` chỉ chứng minh endpoint exporter trả lời; trên Windows Docker Desktop, không diễn giải nó thành số đo chính xác của phần cứng Windows.

## 4. Kiểm tra đường truyền Alert mà không tạo tải

Không cần tăng tải CPU/RAM. Tạo test alert tạm thời bằng cách thêm rule sau vào cuối danh sách `rules:` trong `Bai6_Alerting\alert_rules.yml` (cùng cấp thụt lề với hai rule hiện có):

```yaml
      - alert: AlertPipelineTest
        expr: vector(1)
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: Alert pipeline test
          description: Temporary test alert; remove after validation.
```

`vector(1)` luôn cho một mẫu giá trị 1; đây chỉ là test rule, không phải monitor CPU/RAM. Reload cấu hình (endpoint lifecycle chỉ được bật trong stack thực hành):

```powershell
Invoke-WebRequest -UseBasicParsing -Method POST -Uri http://localhost:9091/-/reload
```

Trong `/rules` quan sát `AlertPipelineTest` ở `PENDING`; sau khi điều kiện được đánh giá liên tục đủ một phút, kiểm tra `/alerts` và giao diện Alertmanager ở `http://localhost:9093`. Thời điểm chuyển trạng thái có thể trễ theo `evaluation_interval` 15 giây và thời điểm scrape. Việc thấy alert trong Alertmanager chứng minh luồng Prometheus → Alertmanager và xử lý/hiển thị tại Alertmanager; không chứng minh email/Slack/webhook đã được gửi.

Sau kiểm tra, xóa rule tạm khỏi file, reload lại, rồi xác nhận rule không còn trong `/rules` và alert test đã được resolve:

```powershell
Invoke-WebRequest -UseBasicParsing -Method POST -Uri http://localhost:9091/-/reload
```

Alertmanager có thể còn hiển thị alert đã nhận cho tới khi hết `endsAt`/resolve timeout; chờ thêm vài phút rồi kiểm tra lại trước khi kết luận reload thất bại.

## 5. Hiểu `for`, scrape/evaluation và gom nhóm

- `scrape_interval: 15s`: chu kỳ lấy mẫu target.
- `evaluation_interval: 15s`: chu kỳ đánh giá PromQL/rules.
- `for: 1m`: điều kiện phải liên tục đúng qua các lần đánh giá để chuyển `PENDING` sang `FIRING`; đây không phải thời gian gom hoặc gửi thông báo.
- CPU sử dụng `rate(...[5m])`, làm mượt mẫu CPU trên cửa sổ 5 phút. Một spike 2 giây thường giảm ảnh hưởng trong rate-window; `for: 1m` thêm điều kiện duy trì một phút sau khi biểu thức vượt ngưỡng.
- `group_wait: 10s`: chờ gom nhóm trước lần thông báo đầu tiên do Alertmanager gửi.
- `group_interval: 1m`: chờ trước khi thông báo thay đổi tiếp theo trong cùng group.
- `repeat_interval: 4h`: khoảng nhắc lại group alert vẫn còn firing.

CPU/RAM alerts có thể không firing trên máy thực hành nếu ngưỡng chưa vượt 90%. Đừng gây quá tải hoặc làm đầy RAM để buộc điều kiện xảy ra.

## 6. Reload và bảo mật

Thay đổi file trên Host không tự bảo đảm Prometheus đã đọc lại. Kiểm tra cú pháp trước, gọi `POST /-/reload`, rồi xem response và log. Lifecycle endpoint có quyền quản trị reload cấu hình; chỉ bật trong môi trường thực hành có kiểm soát và không expose ra mạng công cộng. Cấu hình hiện bind Prometheus và Alertmanager vào `127.0.0.1`; không publish cổng `9100` hoặc `8080`.

## 7. Chẩn đoán nhanh

1. `docker compose ... ps` và logs xác nhận container có chạy hay crash loop.
2. HTTP `/-/healthy` xác nhận Prometheus/Alertmanager đã khởi động.
3. Ở `/targets`, kiểm tra `lastError`: DNS/service name sai, connection refused thường là sai port/service hoặc ứng dụng chưa lắng nghe, timeout thường liên quan mạng/đường truy cập.
4. `/rules` và log Prometheus giúp phát hiện rule chưa mount hoặc YAML/PromQL sai.
5. Nếu Prometheus nhận alert nhưng Alertmanager không thấy, kiểm tra `alerting.alertmanagers` trong config và log Prometheus/Alertmanager.
6. Nếu host không truy cập UI, kiểm tra port mapping loopback và xung đột cổng trước; không mở firewall công khai để khắc phục.

Dừng riêng stack Bài 6, giữ dữ liệu volume:

```powershell
docker compose -p bai6-alerting -f .\Bai6_Alerting\docker-compose.yml down
```
