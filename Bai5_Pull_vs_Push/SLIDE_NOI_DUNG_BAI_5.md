# Nội dung thuyết trình Bài 5 — 10 slide

## Slide 1 — Câu hỏi kiến trúc

**Ý chính**
- E-commerce có service/replica thay đổi theo Auto-scaling.
- Pull phải biết “địa chỉ cửa” ở đâu; Push phải biết “server nhận” ở đâu.
- Mục tiêu: chọn luồng monitoring hợp topology và yêu cầu vận hành.

**Ghi chú thuyết trình**  
Không tranh luận Pull hay Push như một lựa chọn tuyệt đối. Phân biệt cách lấy metrics từ ứng dụng và cách gửi dữ liệu về hệ thống trung tâm.

## Slide 2 — Pull của Prometheus

**Ý chính**
- Service/exporter expose `/metrics` hoặc `/actuator/prometheus`.
- Prometheus gửi HTTP GET theo `scrape_interval`.
- Gắn labels `job`, `instance`; lưu samples vào TSDB.
- `up=1` scrape thành công; `up=0` scrape thất bại.

**Ghi chú thuyết trình**  
`up` đo trạng thái scrape endpoint, không phải health check toàn diện cho business flow.

## Slide 3 — Push và Agent

**Ý chính**
- Agent/SDK thu thập, xử lý, batch và gửi dữ liệu.
- Datadog Agent, Telegraf, CloudWatch Agent có kiến trúc/config khác nhau.
- Một Agent có thể Pull từ app cục bộ rồi Push outbound.

**Ghi chú thuyết trình**  
Agent Push không đồng nghĩa mọi service tự viết code gửi từng metric. Buffer/retry phụ thuộc agent, output và cấu hình.

## Slide 4 — Trade-off Pull vs Push

**Ý chính**
- Pull: target health rõ, scrape config tập trung; cần route từ Prometheus tới target.
- Push: outbound dễ đi qua NAT/firewall hơn; cần credential, buffer và ingest backend.
- Cả hai cần quản lý cardinality, bảo mật, scale và chi phí.

**Ghi chú thuyết trình**  
Nêu ví dụ: private Kubernetes network thường thuận Pull; môi trường outbound-only/SaaS agent có thể thuận Push.

## Slide 5 — Auto-scaling 1.000 service

**Ý chính**
- Service mới → Kubernetes/Consul registry cập nhật → Prometheus SD tìm target.
- Relabel lọc target và chọn port/path → Prometheus scrape.
- Scale down → SD loại target.
- Cần RBAC/ACL, metadata, endpoint, network policy.

**Ghi chú thuyết trình**  
Prometheus không tự phát hiện target mặc định. Service Discovery giải quyết inventory/address, không giải quyết capacity.

## Slide 6 — Kubernetes, Consul và Eureka

**Ý chính**
- Kubernetes: native `kubernetes_sd_config`, cần API access, RBAC, relabel.
- Consul: `consul_sd_config`, registration/health, ACL và metadata.
- Eureka: Prometheus có native `eureka_sd_configs`; cần server URL, reachable address, metadata metrics port/path và relabeling.

**Ghi chú thuyết trình**  
Địa chỉ instance chưa đủ: cần metrics port, path, scheme, readiness và quyền scrape. Eureka discovery là native trong Prometheus; bridge chỉ là phương án khi phiên bản/topology không dùng được trực tiếp.

## Slide 7 — Cronjob và Pushgateway

**Ý chính**
- Job đối soát chạy 3–5s, scrape mỗi 15s: có thể kết thúc trước lần scrape.
- Pushgateway giữ kết quả job để Prometheus scrape sau.
- Hợp với batch service-level, không thay scrape mọi service.
- Dọn stale series; push `last_success`/outcome.

**Ghi chú thuyết trình**  
Job gắn với một host có thể dùng Node Exporter textfile collector. Gateway scrape `up` không chứng minh cron vừa thành công.

## Slide 8 — Sự cố và độ bền dữ liệu

**Ý chính**
- Prometheus down: app có thể tiếp tục; snapshot chưa scrape có thể mất.
- Agent downlink: buffer/retry có thể giúp nhưng hữu hạn.
- Discovery sai: target không xuất hiện, có thể không có `up`.
- Metrics endpoint lỗi không nhất thiết đồng nghĩa business API chết.

**Ghi chú thuyết trình**  
Kiểm tra log, `/targets`, agent queue, registry và timestamps. Phân biệt scrape health với application SLI.

## Slide 9 — Kiến trúc đề xuất

**Ý chính**
- Prometheus Pull cho services/exporters trong cluster/private network.
- Kubernetes SD hoặc Consul SD theo môi trường.
- Grafana/Alertmanager cho dashboard/cảnh báo.
- Pushgateway cho batch phù hợp; agent SaaS khi có yêu cầu outbound/managed.
- Remote Write là Prometheus gửi sample đã scrape tới storage.

**Ghi chú thuyết trình**  
Định nghĩa owner và labels để tránh thu thập trùng; capacity plan theo target × series × frequency.

## Slide 10 — Kết luận và câu hỏi

**Ý chính**
- Chọn Prometheus khi Pull, SD, PromQL và tự vận hành phù hợp.
- 1.000 service cần SD + phân vùng/sharding theo capacity.
- Chọn Pushgateway cho batch ngắn; Agent Push cho topology/managed ingest phù hợp.
- Pull và Push có thể kết hợp.

**Ghi chú thuyết trình**  
Kết thúc bằng trade-off: topology, độ bền, bảo mật, chi phí, khả năng vận hành. Mời hội đồng hỏi về `up`, discovery, retry, cardinality và Pushgateway.
