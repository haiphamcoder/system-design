# Mở rộng hệ thống & Tối ưu hiệu suất

Hệ thống ngày nay không thể scale chỉ bằng cách "mua máy chủ to hơn" (Vertical Scaling). Ta cần các kỹ thuật để chia sẻ gánh nặng cho hàng ngàn máy chủ nhỏ (Horizontal Scaling).

---

## 1. Tầng Web và Load Balancer

### 1.1. Stateless Web Tier (Tầng Web phi trạng thái)

Để scale ngang hoàn hảo, tầng Web nên là **Stateless**.

- **Vấn đề**: Nếu lưu Session (thông tin đăng nhập) trong RAM của Server A, khi User bị Load Balancer đẩy sang Server B, họ sẽ bị bắt đăng nhập lại.
- **Giải pháp**: Chuyển toàn bộ dữ liệu trạng thái (Session, Profile) sang một **Shared Data Store** dùng chung (như Redis hoặc Database). Lúc này, mọi server web đều giống hệt nhau, ta có thể thêm/bớt server tùy ý mà không làm gián đoạn người dùng.

### 1.2. L4 vs L7 Load Balancing

Sự khác biệt nằm ở tầng hoạt động trong mô hình OSI:

1. **Layer 4 (Transport Layer - TCP/UDP)**:
    - LB chỉ nhìn vào IP và Port. Nó forward gói tin (packet) mà không cần giải mã nội dung bên trong.
    - *Ưu điểm*: Tốc độ cực nhanh, chịu tải cực lớn (Hàng triệu request/s).
    - *Nhược điểm*: "Mù" nội dung (không đọc được URL hay Cookie).
2. **Layer 7 (Application Layer - HTTP)**:
    - LB đóng vai trò Reverse Proxy, đọc nội dung HTTP rồi mới điều hướng.
    - *Ưu điểm*: Thông minh. Có thể định tuyến `video.com/api` sang Server A, `video.com/static` sang Server B. Hỗ trợ SSL Termination.
    - *Nhược điểm*: Chậm hơn L4 do tốn CPU giải mã.

---

## 2. Caching: Từ CDN đến Database

### 2.1. CDN (Mạng phân phối nội dung)

CDN là hệ thống các server rải rác toàn cầu để cache dữ liệu gần User nhất.

- **Static Cache**: Ảnh, Video, JS/CSS.
- **Dynamic Cache**: Cache kết quả HTML dựa trên query parameters hoặc headers.
- **Lợi ích**: Giảm tải cho Server gốc (Origin) và tăng tốc độ tải trang đáng kể cho User ở xa.

### 2.2. Chiến lược Cache Invalidation

1. **Cache Aside (Lazy Loading)**: Client tự quản lý. Nếu cache miss -> Đọc DB -> Ghi Cache. Đây là phương án phổ biến nhất, đảm bảo hệ thống vẫn chạy nếu Cache sập.
2. **Read-Through / Write-Through**: App coi Cache là nơi lưu trữ chính. Cache tự lo việc đồng bộ với DB. Giúp code gọn hơn.
3. **Write-Back (Write-Behind)**: Ghi vào Cache trước rồi báo OK ngay. Cache sẽ âm thầm ghi xuống DB sau. Tốc độ ghi cực nhanh nhưng rủi ro mất dữ liệu nếu Cache sập trước khi kịp sync.

---

## 3. Rate Limiting (Kiểm soát lưu lượng)

Để hệ thống không bị "ngộp" bởi quá nhiều request, ta cần thiết lập bộ kiểm soát:

1. **Token Bucket**: Server cấp token vào thùng theo tốc độ cố định. Request đi qua phải lấy 1 token. Phù hợp cho các đợt bùng nổ traffic (Burst traffic) ngắn hạn.
2. **Leaky Bucket**: Request chảy vào thùng, nhưng server xử lý (chảy ra) với tốc độ không đổi. Nếu thùng đầy, request mới bị loại bỏ. Đảm bảo tốc độ xử lý luôn ổn định.
3. **Fixed Window Counter**: Chia thời gian thành các cửa sổ (ví dụ 1 phút). Mỗi cửa sổ cho phép N request. Đơn giản nhất nhưng dễ bị quá tải ở điểm giao giữa 2 cửa sổ.

---

## 4. Message Queue: Xử lý bất đồng bộ

### Push vs Pull Model

1. **Push (RabbitMQ)**: Queue chủ động đẩy message cho Consumer ngay khi có tin. Tốc độ real-time nhưng dễ làm ngập Consumer nếu họ xử lý chậm.
2. **Pull (Kafka)**: Consumer chủ động hỏi "Có tin mới không?". Consumer tự làm chủ tốc độ và có thể lấy theo lô (Batching), phù hợp với hệ thống throughput lớn.

---

## 5. Ứng dụng thực tế: Thiết kế các hệ thống lớn

### 5.1. Thiết kế Bộ kiểm soát lưu lượng (Rate Limiter)

Trong một hệ thống microservices, Rate Limiter thường được đặt ở tầng **API Gateway**.

- **Thuật toán**: Linh hoạt nhất là **Token Bucket**.
- **Lưu trữ**: Dùng **Redis** để lưu các bộ đếm (counters). Redis cực nhanh và hỗ trợ lệnh `INCR` và `EXPIRE` giúp tự động reset giới hạn theo thời gian.
- **Xử lý lỗi**: Khi user vượt quá giới hạn, hệ thống trả về mã lỗi **HTTP 429** (Too Many Requests) kèm header `X-Ratelimit-Remaining` để báo cho phía Client biết khi nào họ có thể thử lại.

### 5.2. Thiết kế Hệ thống Bảng tin (News Feed)

Thách thức lớn nhất của News Feed là bài toán **Fan-out** (Phân phối tin nhắn).

1. **Fan-out on Write (Push model)**: Khi bạn đăng bài, bài viết được đẩy ngay vào "News Feed Cache" của tất cả bạn bè.
    - *Ưu điểm*: Xem feed cực nhanh vì nó đã được chuẩn bị sẵn.
    - *Nhược điểm*: Nếu là người nổi tiếng (Celeb) có hàng triệu fan, việc ghi hàng triệu bản ghi cùng lúc sẽ gây nghẽn hệ thống (Hot key problem).
2. **Fan-out on Read (Pull model)**: Feed chỉ được tạo ra khi người dùng mở app.
    - *Ưu điểm*: Không lo lắng về tài khoản Celeb.
    - *Nhược điểm*: Phản hồi chậm nếu người dùng có quá nhiều bạn bè đang đăng bài.
3. **Mô hình Hybrid (Lai)**: Dùng **Push** cho người dùng bình thường và dùng **Pull** cho các tài khoản Celeb. Đây là cách Twitter và Facebook xử lý để vừa đảm bảo tốc độ vừa tránh quá tải.

---

## Tổng kết chương

1. Thiết kế **Stateless Web** để scale ngang dễ dàng.
2. Sử dụng **CDN** cho nội dung tĩnh để giảm Latency toàn cầu.
3. Kết hợp **L4 LB** phía trước và **L7 LB** phía sau để tối ưu hiệu năng và tính thông minh.
4. Dùng **Message Queue** (Kafka/RabbitMQ) để tách biệt các tác vụ nặng, giúp web phản hồi user nhanh hơn.
