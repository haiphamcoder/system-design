# Cơ sở dữ liệu và Chiến lược Phân mảnh (Sharding)

Dữ liệu là tài sản quý giá nhất. Chương này đi sâu vào kiến trúc lưu trữ, so sánh bản chất SQL/NoSQL và kỹ thuật Sharding cho dữ liệu quy mô lớn.

---

## 1. Kiến trúc Lưu trữ: B-Tree vs LSM Tree

Khi nói chọn SQL hay NoSQL, thực chất ta đang chọn cấu trúc dữ liệu bên dưới (Storage Engine).

### B-Tree (MySQL, PostgreSQL)

- **Cơ chế**: Dữ liệu lưu trong cấu trúc cây cân bằng (Balanced Tree). Dữ liệu được đọc/ghi theo Page (thường 4KB-16KB).
- **Ưu điểm**: Tối ưu cho thao tác **Đọc** (Read-heavy). Hỗ trợ Range Query rất tốt.
- **Nhược điểm**: Ghi chậm vì phải cập nhật và cân bằng lại cây, ghi ngẫu nhiên (Random Write) làm ổ cứng quay nhiều.

### LSM Tree - Log Structured Merge Tree (Cassandra, RocksDB, Kafka)

- **Cơ chế**: Dữ liệu mới luôn được **Ghi nối đuôi** (Append-only) vào MemTable trên RAM, sau đó xả xuống đĩa thành file SSTable. Định kỳ hệ thống sẽ chạy tiến trình Compact để gộp các file và xóa dữ liệu cũ.
- **Ưu điểm**: Tối ưu tuyệt đối cho thao tác **Ghi** (Write-heavy) vì biến mọi thao tác ghi thành Sequential Write (Ghi tuần tự), rất nhanh trên cả ổ HDD và SSD.
- **Nhược điểm**: Đọc có thể chậm hơn vì phải tìm trên nhiều file SSTable. Để tối ưu lệnh đọc, các hệ thống này sử dụng **Bloom Filter** – một cấu trúc dữ liệu xác suất giúp trả lời cực nhanh câu hỏi: "Key này *chắc chắn không* có trong file này", giúp bỏ qua việc đọc đĩa vô ích.

---

## 2. Sharding: Chia để trị

Khi một máy chủ không thể chứa nổi dữ liệu (Storage limit) hoặc không chịu nổi tải (Throughput limit), ta phải dùng Sharding (Phân mảnh).

### Các chiến lược Sharding

1. **Range Based Sharding**: Chia theo dải (Ví dụ: User A-M vào Shard 1, N-Z vào Shard 2).
    - *Vấn đề*: **Data Skew** (Lệch tải). Nếu user tên vần A dùng nhiều hơn vần Z, Shard 1 sẽ quá tải trong khi Shard 2 rảnh rỗi.
2. **Hash Sharding**: Dùng hàm băm `Hash(key) % N` để chia đều.
    - *Vấn đề*: **Resharding**. Khi thêm server mới, công thức `% N` thay đổi, khiến gần như toàn bộ dữ liệu bị xáo trộn và phải di chuyển lại.

### Consistent Hashing (Băm nhất quán)

Để giải quyết bài toán Resharding kinh điển, ta dùng Consistent Hashing (thường dùng trong Cassandra, DynamoDB, Discord).

- **Cơ chế**: Coi không gian Hash là một **Vòng tròn** (Ring). Các server và key đều được băm lên vòng tròn đó. Mỗi key sẽ thuộc về server gần nhất theo chiều kim đồng hồ.
- **Ưu điểm**: Khi thêm/bớt 1 server, chỉ một phần nhỏ dữ liệu (các key nằm giữa server mới và server cũ) bị ảnh hưởng và cần di chuyển. Giảm thiểu tối đa lưu lượng mạng khi scale hệ thống.
- **Virtual Nodes**: Để cân bằng tải tốt hơn nữa, mỗi server vật lý sẽ được đại diện bởi nhiều "node ảo" rải rác trên vòng tròn. Số lượng node ảo có thể tỉ lệ thuận với cấu hình máy (Heterogeneity): máy mạnh có nhiều node ảo hơn, giữ nhiều dữ liệu hơn.

### Phục hồi sau sự cố (Failure Recovery)

Trong một hệ thống lớn, node hỏng là chuyện thường xuyên. Ta cần các cơ chế để hệ thống tự chữa lành:

1. **Sloppy Quorum & Hinted Handoff**: Khi một node chính bị sập tạm thời, hệ thống sẽ chọn node kế tiếp trên vòng tròn để ghi dữ liệu tạm. Khi node chính sống lại, node tạm sẽ bàn giao lại dữ liệu (Hinted Handoff). Điều này đảm bảo tính khả dụng (Availability) cực cao.
2. **Merkle Trees (Cây băm)**: Để đồng bộ dữ liệu giữa hai bản sao sau một thời gian dài mất kết nối, ta không thể gửi toàn bộ database sang để so sánh (quá tốn băng thông). Thay vào đó, ta dùng Merkle Tree để so sánh mã băm của từng khối dữ liệu. Ta chỉ cần gửi đi những khối có mã băm khác nhau, giúp quá trình đồng bộ diễn ra cực nhanh và tiết kiệm.

---

## 3. SQL vs NoSQL: Góc nhìn ACID vs BASE

### SQL (ACID)

- **Atomicity**: Giao dịch 1 ăn cả ngã về không.
- **Isolation**: Các giao dịch độc lập, không nhìn thấy dữ liệu dở dang của nhau.
- Phù hợp: Tài chính, Payment, Inventory.

### NoSQL (BASE)

- **Basically Available**: Hệ thống cơ bản là chạy, có thể lỗi nhẹ.
- **Soft state**: Trạng thái có thể chưa đồng bộ ngay.
- **Eventual consistency**: Cuối cùng sẽ đúng.
- Phù hợp: Social Feed, Analytics, Logs.

---

## 4. Ứng dụng thực tế: Thiết kế Hệ thống lưu trữ Key-Value (Distributed KV Store)

Hệ thống lưu trữ như **DynamoDB** hay **Cassandra** áp dụng các kỹ thuật đỉnh cao để đảm bảo dữ liệu luôn sẵn sàng.

### 4.1. Phân mảnh & Sao lưu (Partitioning & Replication)

Dữ liệu được băm lên một **Vòng tròn (Consistent Hashing)**. Để không bị mất dữ liệu, mỗi Key sẽ được lưu tại N node kế tiếp trên vòng tròn đó theo chiều kim đồng hồ.

### 4.2. Cấu hình Quorum (Tùy chỉnh tính nhất quán)

Bạn có thể tùy chỉnh tốc độ và độ an toàn qua bộ ba (N, W, R):

- **N**: Số lượng node sao lưu.
- **W**: Số lượng node tối thiểu phải xác nhận Ghi thành công.
- **R**: Số lượng node tối thiểu phải trả về dữ liệu khi Đọc.

> **Công thức Vàng**: Nếu **W + R > N**, hệ thống đảm bảo **Strong Consistency** (Dữ liệu đọc ra luôn là mới nhất).

### 4.3. Xử lý xung đột bằng Vector Clocks

Khi hai người dùng cùng sửa một dữ liệu ở hai quốc gia khác nhau, hệ thống làm sao biết cái nào có sau?

- **Vector Clocks**: Đính kèm một "danh sách phiên bản" [Server, Counter] vào mỗi dữ liệu. Nếu phiên bản này "bao hàm" phiên bản kia, ta biết cái nào cũ hơn. Nếu không, ta biết có **xung đột (Conflict)** và yêu cầu Client xử lý bằng tay (như khi bạn Resolve Conflict trong Git).

---

## Tổng kết chương

1. Hiểu **Storage Engine** (B-Tree vs LSM Tree) quan trọng hơn chỉ biết tên DB. Nếu cần ghi cực nhanh, chọn DB dùng LSM Tree.
2. **Sharding** là giải pháp cuối cùng. Hãy thử Scale Up và Cache trước.
3. Khi Sharding, ưu tiên **Consistent Hashing** để dễ dàng thêm bớt server vận hành sau này.
