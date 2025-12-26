# Monitoring và Observability

Hệ thống phân tán ngày nay quá phức tạp để vận hành nếu chỉ nhìn vào CPU và RAM. Chương này sẽ giúp bạn xây dựng hệ thống Observability toàn diện để không chỉ biết *khi nào* hệ thống hỏng, mà còn hiểu *tại sao* nó hỏng.

---

## 1. Metrics: Kiến trúc Push vs Pull

Metrics là các con số định lượng (CPU load, Request count, Latency). Khi xây dựng hệ thống Monitoring, quyết định kiến trúc quan trọng nhất là chọn cách thu thập dữ liệu: **Pull** hay **Push**.

### Mô hình Pull (Prometheus)

Trong mô hình này, hệ thống Monitoring (như Prometheus) định kỳ kết nối đến từng Server để "kéo" (scrape) số liệu về.

- **Ưu điểm Flow Control**: Server Monitoring chủ động quyết định tốc độ lấy dữ liệu. Nếu Server ứng dụng đang quá tải, nó sẽ không bị chết thêm do áp lực phải đẩy metrics đi.
- **Dễ phát hiện lỗi**: Nếu Monitoring không kết nối được tới Server -> Biết ngay Server đó có vấn đề (Down).
- **Nhược điểm**: Khó triển khai qua môi trường NAT/Firewall hoặc các job ngắn hạn (batch job), thường cần thêm component trung gian như **Pushgateway**.

### Mô hình Push (Datadog, Graphite)

Server ứng dụng chủ động gửi (đẩy) số liệu về hệ thống Monitoring trung tâm.

- **Ưu điểm**: Phù hợp với các job chạy ngắn hoặc môi trường mạng phức tạp. Dữ liệu realtime hơn.
- **Nhược điểm**: Nếu gừi quá nhiều metrics (ví dụ debug), có thể làm nghẽn mạng hoặc quá tải Server Monitoring trung tâm.

---

## 2. Distributed Tracing (Truy vết phân tán)

Trong kiến trúc Microservices, một request đơn lẻ có thể đi qua hàng chục service khác nhau. Tracing là công cụ giúp vẽ lại bản đồ đường đi đó để tìm điểm nghẽn.

### Context Propagation (Lan truyền ngữ cảnh)

Để nối các mảnh log rời rạc từ các service khác nhau thành một bức tranh liền mạch, ta cần cơ chế Context Propagation:

1. **Trace ID**: Ngay tại điểm vào (API Gateway), một `TraceID` duy nhất được sinh ra.
2. **Header Forwarding**: ID này phải được truyền đi kèm (inject) trong Header của mọi request nội bộ (HTTP/gRPC) giữa các service (ví dụ header `X-B3-TraceId` hoặc `traceparent` của chuẩn W3C).
3. **Rủi ro**: Chỉ cần một service trung gian quên forward header này, toàn bộ chuỗi trace sẽ bị đứt đoạn (Broken Trace).

### Chiến lược Sampling (Lấy mẫu)

Ghi lại Trace cho 100% request tiêu tốn rất nhiều tài nguyên lưu trữ và băng thông. Ta cần chiến lược lấy mẫu (Sampling) hợp lý:

- **Head-based Sampling**: Quyết định ngay từ đầu vào (ví dụ: chỉ lấy ngẫu nhiên 5% request). Cách này nhanh, nhẹ nhưng dễ bỏ lỡ các lỗi xảy ra ở những request hiếm gặp (không nằm trong 5% được chọn).
- **Tail-based Sampling**: Thu thập trace của toàn bộ request, nhưng chỉ lưu lại những trace *có lỗi* hoặc *có độ trễ cao*. Cách này đảm bảo bắt được mọi sự cố nhưng đòi hỏi hệ thống xử lý trung gian cực mạnh để buffer dữ liệu trước khi quyết định lưu hay bỏ.

---

## 3. Logging Architecture

Log không chỉ là text file. Trong hệ thống lớn, Log cần được chuẩn hóa để máy có thể đọc và phân tích tự động.

### Structured Logging & Correlation

Thay vì ghi log dạng text tự do (`Order failed user 123`), hãy chuyển sang **Structured Log (JSON)**:
`{"time": "2023-10-01...", "event": "order_fail", "user_id": 123, "error_code": "DB_LOCK", "trace_id": "abc-xyz"}`
Lợi ích lớn nhất là khả năng **Log Correlation**: Bằng cách tự động inject `trace_id` vào mọi dòng log, kỹ sư có thể tìm kiếm `trace_id=abc-xyz` trên hệ thống log tập trung (ELK/Loki) và thấy ngay toàn bộ lịch sử log của request đó nằm rải rác trên 10 server khác nhau. Đây là sự kết hợp sức mạnh giữa Logging và Tracing.

---

## 4. Báo động (Alerting) thông minh

Mục tiêu của Alert là báo đúng người, đúng việc, tránh "báo động giả" gây nhờn (Alert Fatigue).

### Chiến lược chống nhiễu

1. **Hysteresis (Ngưỡng trễ)**:
    Thay vì báo động ngay khi CPU > 90% (dễ bị spam nếu CPU dao động 89% -> 91% -> 89%), hãy dùng cơ chế Hysteresis: Báo động khi CPU > 90%, nhưng chỉ tắt báo động khi CPU đã giảm xuống dưới 80%.
2. **Anomaly Detection (Dò bất thường)**:
    Sử dụng thuật toán học máy đơn giản dựa trên lịch sử thay vì ngưỡng cứng. Ví dụ: CPU 50% vào ban ngày là bình thường, nhưng 50% vào lúc 3 giờ sáng (khi không có user) là bất thường (dấu hiệu bị tấn công hoặc đào coin trộm).

---

## Tổng kết chương

Observability thực chiến đòi hỏi thiết kế kiến trúc bài bản từ đầu:

1. Hiểu rõ **Pull (Prometheus)** vs **Push** để chọn mô hình phù hợp quy mô.
2. Triển khai **Distributed Tracing** bắt buộc phải tuân thủ chuẩn **Context Propagation**.
3. Chuyển đổi sang **Structured Log (JSON)** có gắn **TraceID** để liên kết dữ liệu.
4. Thiết kế **Alert** thông minh với **Hysteresis** và **Anomaly Detection** để giảm nhiễu.
