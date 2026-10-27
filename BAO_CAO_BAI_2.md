# BÁO CÁO BÀI 2: BẬT RADAR CHO SPRING BOOT ACTUATOR

## 1. Tên bài thực hành

**Bài 2: Vận dụng cơ bản – Bật Radar cho Spring Boot (Actuator)**

## 2. Mục tiêu

- Dùng Spring Boot Actuator để cung cấp endpoint theo dõi ứng dụng.
- Kết hợp Micrometer Prometheus Registry để xuất metrics dạng Prometheus.
- Chỉ expose các endpoint cần thiết, không công khai các endpoint nhạy cảm.
- Kiểm tra cấu hình và endpoint bằng Maven, Docker Compose, MockMvc và HTTP thực tế.

## 3. Mô tả project và tình huống ban đầu

Workspace đã có project thực hành của Bài 1. Project dùng Java 21, Spring Boot 3.5.7 và Maven (`pom.xml`); không có `build.gradle`, `build.gradle.kts` hay Gradle Wrapper. Vì vậy không tạo project Gradle mới và không thêm một hệ thống build thứ hai. Maven 3.9.11 đang có trên máy.

Trước khi sửa, `src/main/resources/application.properties` đã giữ tên ứng dụng `prometheus-scrape-demo`, cổng `8080`, chỉ expose `health,prometheus`, và đặt `management.endpoint.health.show-details=never`. Project đã có Actuator và Micrometer Prometheus Registry. Không tìm thấy Spring Security hay cấu hình management port riêng trong project.

## 4. Phân tích Actuator và Micrometer Prometheus Registry

- `spring-boot-starter-actuator` cung cấp các endpoint vận hành như health và thông tin ứng dụng. Dependency này không tự cung cấp định dạng registry Prometheus.
- `micrometer-registry-prometheus` đăng ký Prometheus meter registry để Micrometer xuất metrics theo định dạng Prometheus và hỗ trợ endpoint `/actuator/prometheus`.
- Nếu chỉ có Actuator, health vẫn có thể trả JSON, nhưng endpoint Prometheus không có registry tương ứng và thường không được cung cấp.
- `/actuator/health` trả JSON trạng thái, ví dụ `{"status":"UP"}`. `/actuator/prometheus` trả văn bản metrics theo định dạng Prometheus, thường gồm `# HELP`, `# TYPE` và metric samples. JSON ở health không phải lỗi; cần đánh giá nội dung theo endpoint đang gọi.

Dependency còn thiếu trong bài tập mẫu là:

```groovy
runtimeOnly 'io.micrometer:micrometer-registry-prometheus'
```

Tuy nhiên dependency này **đã có** trong project Maven với scope `runtime`. Không thêm lần nữa và không đổi phiên bản; Spring Boot dependency management quản lý phiên bản tương thích.

## 5. Build file trước và sau khi sửa

`build.gradle` không tồn tại trước khi thực hiện và không được tạo. Project dùng `pom.xml`; nội dung dependency liên quan trước khi sửa:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
    <scope>runtime</scope>
</dependency>
```

Sau khi sửa, các dependency này không thay đổi vì đã đáp ứng yêu cầu. `pom.xml` giữ Spring Boot 3.5.7 và Java 21; không thêm dependency hoặc phiên bản trùng lặp.

## 6. Nội dung `application.yml` sau khi sửa

File `src/main/resources/application.properties` được thay bằng `src/main/resources/application.yml`, giữ nguyên tên ứng dụng, cổng và cấu hình health trước đó; bổ sung endpoint `info` vào allowlist:

```yaml
spring:
  application:
    name: prometheus-scrape-demo

server:
  port: 8080

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    health:
      show-details: never
```

Endpoint không có trong danh sách `include` không được expose qua HTTP. Không dùng `include: "*"`, không expose `env`, không bật `shutdown`. `info` được expose nhưng không thêm thông tin build hoặc dữ liệu nhạy cảm vào phản hồi.

## 7. Quy trình chạy ứng dụng

Trong môi trường hiện tại, chạy project bằng Maven:

```powershell
Set-Location 'C:\Users\Laptop\Music\Prometheus'
mvn -B clean test
docker compose config
docker compose up --build -d
docker compose ps
```

Project không có Gradle Wrapper nên `.\gradlew.bat bootRun` không áp dụng; cũng không dùng `gradle bootRun`. Cổng host `8080` đã được một container khác sử dụng, nên Compose publish backend tại `127.0.0.1:8081`; cổng nội bộ của backend vẫn là `8080`. Prometheus dùng target `backend:8080` trong network Compose.

Kiểm tra HTTP bằng PowerShell:

```powershell
Invoke-RestMethod http://localhost:8081/actuator/health
$metrics = Invoke-WebRequest -UseBasicParsing http://localhost:8081/actuator/prometheus
$metrics.StatusCode
$metrics.Headers['Content-Type']
$metrics.Content -split "`n" | Select-String '^# (HELP|TYPE)|^jvm_memory_used_bytes|^process_cpu_usage' | Select-Object -First 10
```

Kiểm tra cấu hình Prometheus và target:

```powershell
docker compose exec -T prometheus promtool check config /etc/prometheus/prometheus.yml
Invoke-RestMethod 'http://localhost:9090/api/v1/query?query=up%7Bjob%3D%22backend%22%7D'
```

Giao diện Prometheus chỉ được publish trên loopback tại `http://localhost:9090`. Xem `/targets` để kiểm tra job `backend`, hoặc `/graph` với truy vấn `up{job="backend"}`.

## 8. Kết quả kiểm thử

Các kết quả sau đã được kiểm tra thực tế sau khi cập nhật cấu hình:

- `mvn -B test`: thành công, gồm test context khởi động và các test endpoint.
- `docker compose config`: cấu hình hợp lệ.
- `docker compose up --build -d`: hai service `backend` và `prometheus` đã khởi động.
- `promtool check config /etc/prometheus/prometheus.yml`: báo `SUCCESS`.
- `GET http://localhost:8081/actuator/health`: HTTP `200`, trạng thái `UP`; không có health details.
- `GET http://localhost:8081/actuator/prometheus`: HTTP `200`, content type `text/plain;version=0.0.4;charset=utf-8`; nội dung có `# HELP`, `# TYPE` và `jvm_memory_used_bytes`.
- Kiểm tra Prometheus API: target `backend` ở trạng thái `up`, scrape URL là `http://backend:8080/actuator/prometheus`; `up{job="backend"}` có giá trị `1`.
- Kiểm tra HTTP `GET /actuator/env` và `/actuator/shutdown` bằng test MockMvc: đều trả `404 Not Found`.

Các metric cụ thể thay đổi theo runtime và môi trường; không yêu cầu mọi metric ví dụ phải xuất hiện.

## 9. Biện pháp bảo mật endpoint quản trị

- Chỉ expose `health`, `info`, `prometheus`; `/actuator/env` và `/actuator/shutdown` không được expose. Shutdown không được bật bằng cấu hình.
- Health details được đặt `never`; test xác nhận response không chứa `components`.
- Docker Compose publish backend và Prometheus trên `127.0.0.1`, không bind ra toàn bộ interface host. Container Prometheus vẫn gọi backend trên network nội bộ Compose.
- Không có Spring Security trong project hiện tại. Nếu triển khai trên môi trường dùng chung hoặc production, cần xác thực/phân quyền phù hợp cho Prometheus và giới hạn bằng firewall, mạng riêng hoặc reverse proxy; không đưa endpoint metrics ra Internet công khai. Expose allowlist không thay thế kiểm soát truy cập mạng.

## 10. Kết luận và checklist

Actuator cung cấp endpoint vận hành, trong khi Prometheus Registry cung cấp định dạng metrics cho Prometheus. Hai phần đã hiện diện trong Maven project; thay đổi chính của bài này là chuyển cấu hình sang YAML, thêm `info` vào danh sách expose, kiểm thử an toàn endpoint và giới hạn cổng host ở loopback.

- [x] Kiểm tra build system, phiên bản Java/Spring Boot và dependency trước khi sửa.
- [x] Không tạo project Gradle trùng lặp hoặc thêm dependency Prometheus lần nữa.
- [x] Expose `health`, `info`, `prometheus`; không expose `env` hoặc `shutdown`.
- [x] Health details không được trả ra endpoint.
- [x] Maven tests, Compose config, Prometheus config và endpoint đã được kiểm tra.
- [x] Prometheus scrape backend với `up{job="backend"} = 1`.
- [ ] Chụp ảnh màn hình thật của `/targets`, query `up{job="backend"}` và hai endpoint backend để làm minh chứng nộp bài.
