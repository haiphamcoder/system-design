# Các phương thức giao tiếp & Thiết kế API

Lựa chọn giao thức sai có thể khiến hệ thống "ì ạch" dù thuật toán tối ưu đến đâu. Chương này mổ xẻ các giao thức từ tầng vận chuyển đến tầng ứng dụng.

---

## 1. Tầng Giao vận: TCP vs UDP Internals

### TCP: Congestion Control & Reliability

Tại sao TCP tin cậy? Bên cạnh "3-way handshake", TCP có cơ chế:

- **Sliding Window & Flow Control**: Bên nhận báo cho bên gửi biết mình còn bao nhiêu bộ nhớ đệm (Receive Window). Bên gửi sẽ không bao giờ gửi quá số lượng đó, tránh làm ngập bên nhận.
- **Congestion Control (Slow Start)**: TCP bắt đầu gửi chậm, rồi tăng dần tốc độ (Exponential) cho đến khi gặp packet loss thì giảm xuống. Cơ chế này giúp dò tìm băng thông tối đa của mạng mà không gây nghẽn.
- **Head-of-Line (HOL) Blocking**: Nếu gói tin số 1 bị mất, TCP bắt toàn bộ hàng đợi dừng lại chờ gói 1 được gửi lại, dù gói 2, 3 đã tới nơi. Đây là nhược điểm chí tử khiến HTTP/1.1 và HTTP/2 bị chậm trên đường truyền kém.

### UDP: Raw Speed

UDP bỏ qua tất cả các cơ chế trên. Không Handshake, không Flow Control, không Order.

- Dùng cho: Real-time (VoIP, Game). Thà mất hình (glitch) còn hơn bị dừng hình (lag) để chờ.
- **QUIC (HTTP/3)**: Google xây dựng giao thức mới dựa trên UDP để thay thế TCP, khắc phục lỗi HOL Blocking, giúp web tải nhanh hơn nhiều.

---

## 2. API Paradigms: REST vs RPC

### REST & HTTP/1.1 vs HTTP/2

- **HTTP/1.1**: Mỗi request thường tốn 1 kết nối TCP (hoặc dùng Keep-Alive tuần tự). Text-based (Header to đùng).
- **HTTP/2 (Multiplexing)**: Cho phép bắn song song hàng trăm request trên **Duy nhất 1 kết nối TCP**. Header được nén (HPACK). Đây là lý do gRPC chọn xây trên HTTP/2.

### gRPC & Protocol Buffers

Tại sao gRPC nhanh hơn REST (JSON)?

1. **Binary Serialization (Protobuf)**:
    - JSON: `{"id": 123, "name": "John"}` (Tốn nhiều byte ký tự).
    - Protobuf: `08 7B 12 04 4A 6F 68 6E` (Mã hóa nhị phân cực gọn, có schema định nghĩa trước). Giảm kích thước payload tới 30-50%.
2. **Strong Typing**: Interface được định nghĩa chặt chẽ bằng file `.proto`. Code client/server được gen tự động, tránh lỗi sai kiểu dữ liệu thường gặp ở JSON.

---

## 3. WebHoojs & Long Polling

Khi Client muốn nhận tin tức thì từ Server (ví dụ: tin nhắn mới):

1. **Polling (Hỏi liên tục)**: Client chọc Server mỗi giây 1 lần. Rất tốn tài nguyên.
2. **Long Polling**: Client chọc Server, Server **treo** kết nối đó (không trả lời ngay) cho đến khi thực sự có tin nhắn mới thì mới trả lời. Đỡ tốn hơn Polling nhưng vẫn giữ kết nối.
3. **WebSocket**: Kết nối 2 chiều (Bi-directional) liên tục. Tối ưu nhất cho Chat/Game.
4. **Webhook**: Dành cho Server-to-Server. Khi sự kiện xảy ra, Server A tự động gọi API của Server B để báo. (Lưu ý: Server B cần verify chữ ký HMAC để chắc chắn request đến từ A chứ không phải hacker giả mạo).

---

## 4. Ứng dụng thực tế: Thiết kế Hệ thống Chat trực tuyến

Hệ thống Chat đặt ra yêu cầu cực cao về độ trễ và khả năng truyền tin 2 chiều.

### 4.1. Tại sao chọn WebSocket thay vì HTTP?

- **Vấn đề của HTTP**: HTTP là giao thức "Client hỏi - Server trả lời". Trong Chat, Server cần đẩy tin nhắn chủ động cho người nhận ngay khi có tin mới.
- **Giải pháp**: **WebSocket** thiết lập một kết nối lâu dài, 2 chiều sau khi thực hiện "Handshake" qua HTTP. Nó rất nhẹ (Header chỉ vài byte) và cho phép Server gởi dữ liệu cho Client bất cứ lúc nào.

### 4.2. Quản lý trạng thái Online (Presence Service)

Làm sao biết bạn bè đang Online hay Offline?

- **Heartbeat (Nhịp đập)**: Client định kỳ gửi một gói tin nhỏ (Heartbeat) cho Server qua WebSocket.
- **Cơ chế**: Nếu trong 30 giây Server không nhận được Heartbeat từ Client, nó sẽ coi user đó đã mất kết nối và cập nhật trạng thái sang **Offline**.
- **Lưu trữ**: Trạng thái Online thường được lưu trong **Redis** để các server khác nhau có thể cùng truy cập nhanh chóng.

---

## Tổng kết chương

1. Hiểu **TCP HOL Blocking** để biết tại sao thế giới đang chuyển sang **HTTP/3 (QUIC/UDP)**.
2. Dùng **gRPC (Protobuf)** cho giao tiếp nội bộ (Internal) để tiết kiệm băng thông và CPU.
3. Dùng **REST (JSON)** cho Public API vì tính tương thích.
4. Sử dụng **WebSocket** cho real-time user interaction và **Webhook** cho tích hợp hệ thống bên thứ 3.
