# Bài 5 — Pull vs Push trong Monitoring

Thư mục này chứa bộ sản phẩm hoàn chỉnh cho bài phân tích kiến trúc Prometheus Pull và các mô hình Agent/Push.

## Sản phẩm

- [BAO_CAO_BAI_5.md](./BAO_CAO_BAI_5.md): báo cáo đầy đủ, bảng so sánh 14 tiêu chí, phân tích Auto-scaling, Pushgateway/Cronjob, sự cố, kiến trúc đề xuất, Q&A và nguồn chính thức.
- [SO_DO_KIEN_TRUC.md](./SO_DO_KIEN_TRUC.md): các sơ đồ Mermaid riêng cho Pull, Agent Push, Service Discovery và Pushgateway.
- [SLIDE_NOI_DUNG_BAI_5.md](./SLIDE_NOI_DUNG_BAI_5.md): nội dung thuyết trình 10 slide cùng ghi chú trình bày.
- [CHECKLIST_BAI_5.md](./CHECKLIST_BAI_5.md): bảng đối chiếu đề bài, vị trí nội dung và trạng thái kiểm chứng.

## Cách đọc và xem sơ đồ

Mở file trong VS Code, chọn **Markdown: Open Preview** (thường là `Ctrl+Shift+V`) để xem Markdown. Mermaid được hiển thị nếu phiên bản VS Code/Markdown Preview hỗ trợ Mermaid; nếu không, xem code block trong [SO_DO_KIEN_TRUC.md](./SO_DO_KIEN_TRUC.md) hoặc dùng renderer Mermaid được tổ chức cho phép.

## Phạm vi kiểm chứng

Bài 5 là phân tích kiến trúc, không thay đổi hoặc triển khai stack Docker trong workspace. Các mô tả về Prometheus, Pushgateway, Datadog, Telegraf, AWS CloudWatch Agent, Kubernetes/Consul/Eureka được tổng hợp từ tài liệu tham khảo chính thức trong báo cáo. Không có benchmark hoặc kết quả runtime mới được tuyên bố.
