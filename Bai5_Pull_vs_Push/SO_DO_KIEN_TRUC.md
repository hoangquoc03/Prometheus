# Sơ đồ kiến trúc — Bài 5 Pull vs Push

## 1. Prometheus Pull

```mermaid
sequenceDiagram
    autonumber
    participant SD as Service Discovery
    participant P as Prometheus
    participant API as Backend metrics endpoint
    participant TSDB as Prometheus TSDB
    P->>SD: Đồng bộ target và metadata
    SD-->>P: Danh sách instance cần scrape
    loop Mỗi scrape_interval
        P->>API: HTTP GET /actuator/prometheus
        API-->>P: Metrics exposition format
        P->>TSDB: Gắn job/instance labels và lưu samples
    end
```

## 2. Agent thu thập rồi Push

```mermaid
flowchart LR
    Sources[OS, app endpoint, logs] --> Input[Agent inputs / SDK]
    Input --> Process[Processors / aggregation]
    Process --> Buffer[Batch, buffer, retry theo cấu hình]
    Buffer -->|Outbound HTTPS / protocol| Platform[Monitoring platform]
    Platform --> Store[(Central storage and query)]
```

Agent có thể poll endpoint tại chỗ (Pull local) nhưng gửi batch ra nền tảng (Push outbound). Chiều thu thập ở nguồn và chiều vận chuyển tới backend là hai khái niệm khác nhau.

## 3. Service Discovery và Auto-scaling

```mermaid
sequenceDiagram
    autonumber
    participant K as Kubernetes / Consul / Registry
    participant D as Prometheus SD
    participant R as Relabel/filter rules
    participant P as Prometheus scraper
    participant I as New service instance
    K->>K: Tạo instance và cập nhật metadata
    K-->>D: Cung cấp danh sách/changes
    D->>R: Target labels, address, annotations
    R-->>P: Chỉ giữ targets phù hợp, đặt port/path
    P->>I: Scrape endpoint metrics
    I-->>P: Metrics samples
    K->>K: Scale down / xóa instance
    K-->>D: Registry/API cập nhật
    D-->>P: Loại target đã kết thúc
```

Discovery cần cấu hình và quyền truy cập. Prometheus có discovery config cho Kubernetes, Consul và Eureka (`kubernetes_sd_configs`, `consul_sd_configs`, `eureka_sd_configs`). Eureka vẫn cần server URL reachable, metadata metrics port/path và relabeling phù hợp.

## 4. Cronjob → Pushgateway → Prometheus

```mermaid
sequenceDiagram
    autonumber
    participant J as Batch job (3–5 giây)
    participant G as Pushgateway
    participant P as Prometheus (scrape mỗi 15 giây)
    participant DB as Prometheus TSDB
    J->>G: Push kết quả, last-success, counts
    Note over J: Job kết thúc trước lần scrape
    P->>G: HTTP GET /metrics
    G-->>P: Metrics đã giữ sau khi job kết thúc
    P->>DB: Lưu sample và labels
```

Pushgateway không thay thế TSDB và không tự xóa series khi job kết thúc; cần metric biểu đạt outcome và quy trình quản lý stale groups. Với job gắn riêng một host, có thể xem Node Exporter textfile collector.
