# Checklist đối chiếu Bài 5

| Yêu cầu | Vị trí trong sản phẩm | Trạng thái |
|---|---|---|
| Giữ nguyên các bài cũ, tạo thư mục riêng | `Bai5_Pull_vs_Push/` | Đã tạo riêng; không sửa file bài cũ |
| Giới thiệu E-commerce, monitoring, metrics, thuật ngữ | Báo cáo Phần 1 | Đã viết; E-commerce là bối cảnh giả định |
| Pull flow, scrape interval, endpoint, TSDB, `up` | Báo cáo Phần 2; Sơ đồ 1 | Đã viết và đối chiếu tài liệu Prometheus |
| Ưu nhược Pull, tách nghiệp vụ và giới hạn mất dữ liệu | Báo cáo Phần 2, 7, 9 | Đã viết; không tuyên bố Pull không mất dữ liệu |
| Push, Agent/SDK, batch, buffer/retry | Báo cáo Phần 3; Sơ đồ 2 | Đã viết; semantics tùy sản phẩm/cấu hình |
| Datadog Agent | Báo cáo Phần 3.2 | Đối chiếu tài liệu Datadog chính thức |
| Telegraf input/process/output, không đồng nhất Push | Báo cáo Phần 3.3 | Đã phân biệt; tài liệu chính thức được liệt kê |
| AWS CloudWatch Agent và destinations | Báo cáo Phần 3.4 | Đối chiếu tài liệu AWS chính thức |
| Bảng so sánh tối thiểu 12 tiêu chí | Báo cáo Phần 4 | Đã có 14 tiêu chí |
| Auto-scaling và Service Discovery flow | Báo cáo Phần 5; Sơ đồ 3 | Đã viết; không giả định SD tự hoạt động |
| Kubernetes SD: RBAC, labels/ports/path | Báo cáo Phần 5.2 | Đã nêu điều kiện |
| Consul SD: registration, ACL, relabel | Báo cáo Phần 5.3 | Đã nêu điều kiện |
| Eureka Service Discovery | Báo cáo Phần 5.4; Slide 6 | Xác nhận Prometheus có native `eureka_sd_configs`; nêu URL, reachability, metadata/port/path, relabeling và auth |
| Capacity/cardinality/sharding | Báo cáo Phần 5.5 | Có công thức minh họa theo giả định; không phải benchmark |
| Cronjob 3–5s, scrape 15s | Báo cáo Phần 6; Sơ đồ 4 | Đã phân tích |
| Pushgateway phù hợp batch service-level, không thay TSDB | Báo cáo Phần 6.2–6.3 | Đối chiếu khuyến nghị Prometheus |
| Stale metrics, cleanup, `up` của gateway không phải job outcome | Báo cáo Phần 6.3 | Đã nêu |
| Node Exporter textfile collector | Báo cáo Phần 6.4 | Đã nêu use case host-bound |
| Tối thiểu 5 sự cố: nguyên nhân, hệ quả, logs/metrics, fix, giới hạn | Báo cáo Phần 7 | Có 6 tình huống |
| Kiến trúc đề xuất và kết luận bảo vệ | Báo cáo Phần 8–9 | Đã viết cân bằng |
| Tối thiểu 10 câu hỏi bảo vệ | Báo cáo Phần 10 | Có 15 câu hỏi/đáp án |
| Tài liệu tham khảo chính thức | Báo cáo Phần 11 | Các URL đã được truy cập/đối chiếu về nội dung |
| README hướng dẫn mở/preview | `README.md` trong thư mục | Đã tạo |
| Sơ đồ Pull/Push/SD/Pushgateway | `SO_DO_KIEN_TRUC.md` | Đã tạo 4 sơ đồ Mermaid |
| Slide 8–10 trang | `SLIDE_NOI_DUNG_BAI_5.md` | Đã tạo 10 slide kèm ghi chú |
| Phân biệt viết vs kiểm chứng | Bảng này | Bài 5 là báo cáo phân tích; không chạy benchmark hoặc thay stack monitoring |

## Kiểm chứng nội dung và giới hạn

- Đã kiểm tra cấu trúc workspace và xác nhận báo cáo Bài 2–4 hiện có trước khi tạo file.
- Đã tham khảo tài liệu chính thức Prometheus (jobs/instances, pushing, configuration gồm `eureka_sd_configs`, storage), Datadog Agent, AWS CloudWatch Agent, Spring Cloud Netflix Eureka và Node Exporter textfile collector.
- Trang Telegraf docs chính thức được kiểm tra để xác nhận tài liệu hiện hành; cách mô tả plugin input/processor/aggregator/output dựa trên mô hình plugin của Telegraf và cần tham khảo cấu hình plugin cụ thể khi triển khai.
- Không sửa cấu hình Docker/project cho Bài 5. Không chạy load test, benchmark hoặc tạo ảnh chụp màn hình.
- Sơ đồ Mermaid được viết theo cú pháp sequenceDiagram/flowchart thông dụng; chưa chạy công cụ renderer Mermaid chuyên biệt.
