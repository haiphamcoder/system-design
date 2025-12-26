# Idempotency & Cơ chế Khóa (Locking)

Xử lý đồng thời (Concurrency) là cơn ác mộng của mọi hệ thống phân tán. Chương này cung cấp giải pháp kỹ thuật sâu sắc cho vấn đề Race Condition và Duplicate Request.

---

## 1. Idempotency (Tính bất biến)

### Cơ chế Token (Idempotency Key)

Để đảm bảo một hành động (như thanh toán) chỉ diễn ra đúng 1 lần dù Client có retry bao nhiêu lần, ta cần Idempotency Key.

- **Flow chuẩn**:
    1. Client sinh UUID (Key) cho request: `POST /pay {amount: 100, key: "abc-123"}`.
    2. Server kiểm tra Key trong DB (bảng `idempotency_store`).
    3. Nếu Key chưa tồn tại: Thực hiện giao dịch -> Lưu Key và Kết quả vào DB -> Trả về Client.
    4. Nếu Key đã tồn tại: Lấy kết quả cũ từ DB trả về ngay lập tức.
- **Atomic Check-and-Set**: Bước kiểm tra sự tồn tại và lưu Key phải là nguyên tử (Atomic). Thường dùng Unique Constraint của Database (`INSERT IGNORE`) hoặc `SETNX` của Redis để đảm bảo không có 2 thread nào cùng thấy "Key chưa tồn tại" cùng lúc.

### Tạo ID duy nhất (Unique ID Generator)

Để Idempotency Key thực sự hiệu quả, nó phải là duy nhất trên toàn hệ thống. Với quy mô lớn, ta không thể dùng `auto_increment` của Database vì nó là nút thắt cổ chai. Một giải pháp kinh điển là **Twitter Snowflake**:

- **Cấu trúc 64-bit ID**:
  - 1 bit dấu (luôn là 0).
  - 41 bit Timestamp: Đảm bảo ID tăng dần theo thời gian (giúp index DB hiệu quả).
  - 10 bit Machine/Datacenter ID: Phân biệt các máy chủ tạo ID khác nhau.
  - 12 bit Sequence number: Cho phép tạo tới 4096 ID mỗi mili giây trên mỗi máy.
- **Ưu điểm**: Tạo ID cực nhanh (không cần network coordination), có thể sắp xếp theo thời gian, và độ dài chỉ 64-bit (tiết kiệm hơn UUID 128-bit).

---

## 2. Locking Strategies: Pessimistic vs Optimistic

### Pessimistic Locking (Khóa Bi quan)

Sử dụng khóa của Database (Row Lock) để chặn người khác.

- **SQL**: `SELECT * FROM seats WHERE id=1 FOR UPDATE;`
  - Lệnh `FOR UPDATE` sẽ báo Database engine khóa dòng dữ liệu này lại. Mọi transaction khác muốn đọc/ghi dòng này đều phải chờ (Wait) cho đến khi transaction giữ khóa Commit.
- **Deadlock**: Rủi ro lớn nhất. Transaction A khóa row 1 chờ row 2. Transaction B khóa row 2 chờ row 1. Cả hai treo vĩnh viễn. Cần timeout và cơ chế phát hiện Deadlock của DB.

### Optimistic Locking (Khóa Lạc quan) & CAS

Không dùng khóa DB, mà dùng phiên bản (Versioning) hoặc Timestamp.

- **Compare-And-Swap (CAS)** pattern:
  - Lệnh SQL: `UPDATE products SET stock = stock - 1, version = version + 1 WHERE id = 1 AND version = 5;`
  - Nếu dòng dữ liệu vẫn ở `version = 5` (chưa ai sửa), lệnh Update thành công (Rows affected = 1).
  - Nếu dòng dữ liệu đã bị người khác sửa thành `version = 6`, điều kiện `WHERE` sai, lệnh Update không tác động dòng nào (Rows affected = 0). App nhận biết thất bại và Retry.
- **Ưu điểm**: Không gây tắc nghẽn (Blocking) DB. Rất nhanh khi ít tranh chấp.
- **Nhược điểm**: Nếu tranh chấp cao (Flash sale), tỷ lệ retry quá lớn gây lãng phí CPU.

---

## 3. Distributed Lock (Khóa phân tán)

Khi hệ thống có nhiều Server truy cập tài nguyên chung (không phải DB, ví dụ file trên S3), ta cần khóa phân tán (Redis Redlock, ZooKeeper).

### Redis Redlock Algorithm

Để khóa an toàn trên Cluster Redis (tránh trường hợp Redis node giữ khóa bị chết):

1. Client xin khóa trên N node Redis độc lập (ví dụ 5 node).
2. Nếu xin được thành công trên quá bán (N/2 + 1 = 3 node) và tổng thời gian xin < thời gian hết hạn khóa -> Thành công.
3. **Lease Time**: Mọi khóa phân tán đều phải có thời gian tự hủy (TTL) để phòng trường hợp Client crash mà quên trả khóa, gây tắc nghẽn vĩnh viễn hệ thống.

---

## 4. Ứng dụng thực tế: Thiết kế Hệ thống rút gọn Link (URL Shortener)

Để rút gọn một URL dài thành một đường dẫn ngắn (ví dụ: `tiny.com/ab123`), ta áp dụng các kiến thức về ID Generator và Hash.

### 4.1. Cơ chế sinh mã rút gọn

Sử dụng **Twitter Snowflake** để sinh ra một ID số 64-bit duy nhất. Sau đó, ta chuyển đổi số này từ cơ số 10 sang **Cơ số 62 (Base62)**.

- **Tại sao dùng Base62?**: Nó bao gồm `[0-9, a-z, A-Z]`, là các ký tự an toàn cho URL và giúp mã rút gọn cực kỳ ngắn (ví dụ số 1,000,000,000,000 chỉ tốn khoảng 7 ký tự).

### 4.2. Chênh lệch giữa mã lỗi 301 và 302

Khi người dùng click vào link ngắn, server sẽ chuyển hướng (Redirect) họ tới link dài.

- **301 Redirect (Moved Permanently)**: Trình duyệt sẽ cache link này. Lần sau user click, nó không cần hỏi server nữa. Tiết kiệm tài nguyên cho hệ thống nhưng **không thể theo dõi (Analytics)** được số lượng click.
- **302 Redirect (Found/Temporary)**: Trình duyệt luôn luôn phải hỏi server để lấy link dài. Giúp hệ thống ghi lại được mọi lượt click và vị trí của user.

> **Lời khuyên**: Dùng **302** nếu bạn cần làm Dashboard thống kê lượt click (như Bitly).

---

## Tổng kết chương

1. **Idempotency** cần được cài đặt ở tầng ứng dụng với sự hỗ trợ của Unique Constraint (DB/Redis).
2. Dùng **Pessimistic Lock (`FOR UPDATE`)** cho giao dịch tài chính quan trọng, chấp nhận chậm.
3. Dùng **Optimistic Lock (CAS)** cho hầu hết các trường hợp khác để có hiệu năng cao.
4. Với tài nguyên ngoài DB, sử dụng **Redis Redlock** hoặc **ZooKeeper** với cơ chế TTL cẩn thận.
