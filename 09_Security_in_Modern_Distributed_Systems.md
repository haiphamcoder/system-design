# Bảo mật trong hệ thống phân tán

Bảo mật không phải là tính năng "làm cho có". Nó là cánh cổng sắt bảo vệ ngôi nhà của bạn khỏi trộm cướp. Chương này sẽ đi từ nền tảng đến các kỹ thuật triển khai chuyên sâu cho: **Authentication, Authorization, HTTPS, Data Protection**.

---

## 1. Authentication (Xác thực) & Authorization (Phân quyền)

Hãy phân biệt rõ hai khái niệm cốt lõi:

- **Authentication (AuthN)**: Xác định *Bạn là ai?* (Giống như Hộ chiếu).
- **Authorization (AuthZ)**: Xác định *Bạn được làm gì?* (Giống như Vé vào cửa).

### Cơ chế lưu trữ & Token: Session vs JWT

Việc lựa chọn cách lưu trữ trạng thái đăng nhập ảnh hưởng trực tiếp đến khả năng mở rộng (Scalability) của hệ thống.

#### Session-based (Stateful)

Trong mô hình truyền thống, Server tạo `SessionID` và lưu trong bộ nhớ (RAM hoặc Database), sau đó gửi `SessionID` về Client qua Cookie.

- **Thách thức mở rộng**: Trong môi trường Cluster nhiều server, nếu request thứ hai của user rơi vào Server B (trong khi Session nằm ở Server A), user sẽ bị "đá" ra ngoài.
- **Giải pháp**: Cần sử dụng **Sticky Session** (Load Balancer luôn trỏ user về server cũ) hoặc **Centralized Session Store** (Dùng Redis cluster riêng để lưu session tập trung).

#### JWT (JSON Web Token - Stateless)

JWT cho phép xác thực mà không cần lưu trạng thái trên Server. Server tạo chuỗi JSON chứa thông tin, **KÝ** (Sign) nó bằng Secret Key, rồi gửi cho Client. Khi Client gửi lại, Server chỉ cần verify chữ ký.

- **Cấu trúc JWT**: Gồm 3 phần `Header.Payload.Signature`.
- **Vấn đề bảo mật & Token Revocation**: Vì Server không lưu token, nên rất khó để thu hồi (revoke) token khi muốn chặn user ngay lập tức.
- **Chiến lược Refresh Token**: Để giải quyết, ta sử dụng cặp **Access Token** (ngắn hạn - ví dụ 5 phút) và **Refresh Token** (dài hạn - ví dụ 7 ngày). Khi Access Token hết hạn, Client dùng Refresh Token để đổi lấy cái mới. Nếu cần block user, hệ thống chỉ cần xóa hoặc cấm Refresh Token trong Database.

---

## 2. OAuth2 & OpenID Connect

OAuth2 là tiêu chuẩn công nghiệp để cấp quyền truy cập (Authorization), giúp user đăng nhập bằng Facebook/Google mà không cần tiết lộ mật khẩu. Tuy nhiên, việc triển khai OAuth2 không đơn giản là "dùng thư viện", bạn cần chọn đúng **Grant Type** (Luồng cấp quyền).

### Các luồng OAuth2 (Grant Types)

1. **Authorization Code Flow**: Đây là luồng chuẩn và bảo mật nhất, dành cho Server-side Web App. Quá trình trao đổi token diễn ra backend-to-backend thông qua một "Code" trung gian, đảm bảo Client Secret không bao giờ bị lộ ra trình duyệt.
2. **PKCE (Proof Key for Code Exchange)**: Dành cho Mobile App hoặc Single Page App (SPA) như React/Vue - những nơi không thể giữ bí mật Client Secret an toàn. Thay vì dùng Secret cố định, App sinh ra một mã bí mật mã hóa động (Code Verifier) cho mỗi lần request. Đây là chuẩn bắt buộc cho ứng dụng Mobile hiện đại.
3. **Client Credentials Flow**: Dành cho giao tiếp giữa các Server (Machine-to-Machine) trong nội bộ, không có sự tham gia của User.

---

## 3. Bảo vệ mật khẩu (Password Storage)

Lưu mật khẩu người dùng dưới dạng Plain Text là sai lầm chết người. Kể cả MD5 hay SHA-256 (băm đơn thuần) cũng đã lỗi thời vì tốc độ tính toán của GPU hiện nay có thể dò ngược (Rainbow Table) rất nhanh.

### Kỹ thuật Băm mật khẩu an toàn

1. **Salt (Muối)**: Mỗi user được cấp một chuỗi ngẫu nhiên riêng (Salt) cộng vào mật khẩu trước khi băm. Điều này đảm bảo hai user có cùng mật khẩu `123456` sẽ có chuỗi hash hoàn toàn khác nhau, vô hiệu hóa Rainbow Table.
2. **Pepper (Tiêu)**: Một chuỗi bí mật được lưu cứng trong Application Config (không lưu cùng Database). Nếu hacker trộm được Database (chứa Hash + Salt), họ vẫn thiếu "Pepper" để có thể dò được mật khẩu.
3. **Work Factor (Độ khó)**: Sử dụng các thuật toán được thiết kế để *chạy chậm* và *tốn bộ nhớ* như **bcrypt** hoặc **Argon2** (chuẩn mới nhất). Các thuật toán này buộc quá trình băm tốn nhiều RAM/CPU, khiến việc hacker dùng "trâu cày" GPU để dò mật khẩu số lượng lớn trở nên bất khả thi về mặt chi phí.

---

## 4. HTTPS & TLS Handshake

HTTPS không chỉ là "ổ khóa màu xanh" trên trình duyệt. Nó là lớp bảo vệ đảm bảo tính toàn vẹn và bí mật của dữ liệu trên đường truyền.

### Tối ưu hiệu năng TLS (TLS 1.2 vs 1.3)

Quá trình bắt tay (Handshake) để thiết lập kết nối bảo mật thường gây ra độ trễ (Latency) đáng kể.

- **TLS 1.2**: Cần **2-RTT** (2 vòng gửi đi gửi về) để hoàn tất bắt tay.
- **TLS 1.3**: Tối ưu hóa quy trình, chỉ cần **1-RTT** để bắt đầu truyền dữ liệu. Thậm chí hỗ trợ **0-RTT** (Zero Round Trip) nếu client đã từng kết nối trước đó (Session Resumption). Việc nâng cấp lên TLS 1.3 giúp cải thiện tốc độ tải trang rõ rệt.

### Forward Secrecy

HTTPS hiện đại yêu cầu tính năng **Perfect Forward Secrecy**. Điều này đảm bảo rằng ngay cả khi hacker lấy trộm được Private Key của Server ngày hôm nay, hắn cũng **KHÔNG THỂ** giải mã lại các tin nhắn đã bắt được từ quá khứ. Mỗi phiên làm việc sử dụng một khóa tạm thời (Ephemeral Key) riêng biệt và bị hủy ngay sau khi kết thúc phiên.

---

## Tổng kết chương

1. Trong Microservices, ưu tiên **JWT** kết hợp chiến lược **Refresh Token Rotation** để cân bằng giữa mở rộng và bảo mật.
2. Với Mobile App/SPA, bắt buộc dùng **OAuth2 với PKCE**.
3. Lưu mật khẩu bằng **Argon2** hoặc **bcrypt** với Salt & Pepper.
4. Cấu hình Server sử dụng **TLS 1.3** để giảm độ trễ kết nối mà vẫn đảm bảo an toàn tối đa.
