# Khả năng chịu lỗi (Fault Tolerance)

"Hệ thống chắc chắn sẽ chết, vấn đề chỉ là khi nào."
Tư duy của một kỹ sư xịn không phải là cầu khấn cho server không bao giờ hỏng, mà là thiết kế sao cho **Server hỏng mà User không hề hay biết**.

---

## 1. Redundancy (Dự phòng) & Replication

Nguyên tắc vàng trong thiết kế hệ thống là **Không bỏ trứng vào một giỏ (No Single Point of Failure)**. Nếu một thành phần chết, phải có thành phần khác thay thế ngay.

### Chiến lược Triển khai

- **Active-Passive**: Server B ở chế độ chờ (standby), chỉ hoạt động khi A gặp sự cố. Phương án này tiết kiệm chi phí nhưng lãng phí tài nguyên chờ.
- **Active-Active**: Cả Server A và B cùng xử lý traffic. Nếu A chết, B sẽ gánh toàn bộ tải. Phương án này tận dụng tối đa tài nguyên nhưng đòi hỏi cấu hình phức tạp hơn.

### Cơ chế đồng bộ dữ liệu (Replication)

Vấn đề lớn nhất của Redundancy là tính nhất quán dữ liệu. Khi A và B cùng chạy, làm sao biết dữ liệu ở B giống hệt A?

1. **Synchronous Replication (Đồng bộ)**:
    - Dữ liệu được ghi vào A -> A gửi sang B -> B báo OK -> A mới báo thành công cho User.
    - **Đặc điểm**: An toàn tuyệt đối (RPO = 0), nhưng hiệu năng phụ thuộc vào node chậm nhất. Nếu B chết, A cũng không thể trả về kết quả thành công.
2. **Asynchronous Replication (Bất đồng bộ)**:
    - Dữ liệu ghi vào A -> A báo thành công ngay lặp tức. Việc copy sang B diễn ra ngầm sau đó.
    - **Đặc điểm**: Hiệu năng cực cao nhưng có rủi ro mất dữ liệu (Data Loss) nếu A chết trước khi kịp đồng bộ sang B.
3. **Semi-Sync**:
    - Cân bằng giữa hai phương án trên: Ghi vào A, chờ *ít nhất 1* Replica báo OK mới trả về.

> **Lưu ý triển khai**: Trong mô hình Active-Active Multi-Region (ví dụ chạy cả ở US và Asia), nên sử dụng **Asynchronous** để tránh độ trễ đường truyền quá lớn, chấp nhận mô hình Eventual Consistency.

---

## 2. Failover (Chuyển đổi dự phòng)

Failover là quá trình tự động tách server hỏng ra khỏi hệ thống và điều hướng traffic sang server dự phòng. Quá trình này dựa vào cơ chế **Health Check** (Khám sức khỏe) liên tục.

### Vấn đề "Split-Brain" (Não đôi)

Health Check đơn giản thường dùng lệnh Ping, nhưng thực tế có thể xảy ra tình huống nguy hiểm gọi là **Split-Brain**.
Điều này xảy ra khi kết nối mạng giữa hai server bị đứt, nhưng cả hai đều vẫn đang chạy tốt. Server B không thấy A trả lời nên tưởng A chết -> B tự phong làm Master. Server A vẫn thấy mình kết nối được với user -> A vẫn đinh ninh mình là Master.
Hậu quả: Cả hai server cùng nhận lệnh ghi (Write), dẫn đến xung đột dữ liệu nghiêm trọng khó hồi phục.

### Giải pháp Quorum và Fencing

Để ngăn chặn Split-Brain, các hệ thống phân tán sử dụng:

1. **Quorum (Đa số)**: Một node chỉ được làm Master khi nhận được sự đồng ý (vote) của quá nửa số node trong cụm (N/2 + 1). Ví dụ cụm 3 node cần 2 phiếu.
2. **Fencing Token**: Khi một node lên làm Master, nó được cấp một token (ID tăng dần). Nếu Master cũ "sống lại" và gửi lệnh với token ID cũ, hệ thống lưu trữ sẽ từ chối.

---

## 3. Circuit Breaker (Cầu chì)

Để tránh hiệu ứng Domino (Service A chết kéo theo Service B, C, D treo vì chờ đợi), ta sử dụng **Circuit Breaker** để ngắt mạch tạm thời.

Circuit Breaker hoạt động như một máy trạng thái (Finite State Machine) với 3 trạng thái chính:

1. **CLOSED (Đóng - Bình thường)**:
    - Request được đi qua bình thường.
    - Hệ thống đếm số lỗi. Nếu tỷ lệ lỗi vượt quá ngưỡng (Threshold, ví dụ 50%), Circuit Breaker chuyển sang trạng thái OPEN.
2. **OPEN (Mở - Ngắt mạch)**:
    - Chặn ngay lập tức mọi request (Fail Fast) mà không gọi đến Service bị lỗi. Trả về lỗi mặc định hoặc dữ liệu cached.
    - Giúp giảm tải cho Service lỗi để nó có thời gian phục hồi.
    - Sau một khoảng thời gian chờ (Sleep Window, ví dụ 5s), chuyển sang HALF-OPEN.
3. **HALF-OPEN (Nửa mở - Thăm dò)**:
    - Cho phép *một vài* request đi qua thử ("Chuột bạch").
    - Nếu thành công -> Hệ thống đã hồi phục -> Chuyển về CLOSED.
    - Nếu vẫn lỗi -> Chuyển ngược lại OPEN và chờ tiếp.

Các thư viện phổ biến để triển khai: **Resilience4j** (Java), **Polly** (.NET), **Hystrix** (Cũ).

---

## 4. Bulkhead Pattern (Vách ngăn)

Lấy cảm hứng từ thiết kế tàu thủy, Bulkhead chia hệ thống thành các khoang riêng biệt. Nếu Service A hỏng, nó chỉ ảnh hưởng tài nguyên của khoang A, các service khác vẫn hoạt động bình thường.

Trong kỹ thuật phần mềm, Bulkhead thường được triển khai bằng cách **cô lập Thread Pool**:
Thay vì dùng chung một Thread Pool lớn (ví dụ 200 threads) cho mọi API, ta chia nhỏ:

- Pool riêng cho API Reporting (tác vụ nặng): 50 threads.
- Pool riêng cho API Login (tác vụ nhẹ): 20 threads.
Nếu API Reporting bị quá tải và chiếm hết 50 threads, các request tiếp theo vào Reporting sẽ bị từ chối, nhưng API Login vẫn hoạt động trơn tru trong pool 20 threads của nó.

---

## 5. Retry & Exponential Backoff

Thử lại (Retry) là chiến lược cơ bản để xử lý các lỗi tạm thời (Transient errors) như rớt mạng gói tin. Tuy nhiên, Retry không đúng cách có thể gây sập toàn bộ hệ thống (Retry Storm).

### Chiến lược Retry thông minh

1. **Exponential Backoff (Lùi theo hàm mũ)**:
    Thay vì thử lại ngay lập tức, hãy tăng thời gian chờ sau mỗi lần thất bại: Chờ 1s, 2s, 4s, 8s... Điều này giúp giảm tải cho server đang gặp sự cố.
2. **Jitter (Độ trễ ngẫu nhiên)**:
    Nếu tất cả client đều cùng Retry sau đúng 2 giây (ví dụ vào lúc 10:00:02), server sẽ lại bị quá tải bởi một cơn sóng request đồng loạt (Thundering Herd Problem).
    Jitter thêm một khoảng thời gian ngẫu nhiên vào mỗi lần chờ (ví dụ: `wait_time = 2s + random(0-500ms)`), giúp dàn trải request ra theo thời gian.

---

## Tổng kết chương

Khi thiết kế Hệ thống chịu lỗi, hãy nhớ:

1. **Redundancy** phải đi kèm chiến lược **Replication** (Async cho hiệu năng hoặc Sync cho an toàn).
2. Thiết lập **Failover** cần tính đến giải pháp cho **Split-Brain** (sử dụng Quorum).
3. **Circuit Breaker** cần được cấu hình đúng máy trạng thái để vừa bảo vệ vừa tự phục hồi.
4. Cách ly tài nguyên bằng **Bulkhead** để lỗi không lan truyền.
5. Sử dụng **Retry** kết hợp **Exponential Backoff** và **Jitter** để tránh làm sập hệ thống khi thử lại.
