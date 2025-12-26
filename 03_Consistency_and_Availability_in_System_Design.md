# Tính nhất quán và khả dụng trong System Design

Trong thiết kế hệ thống phân tán, có một chân lý: **Bạn không thể có tất cả mọi thứ**. Chương này sẽ giúp bạn hiểu sâu về sự đánh đổi giữa Tính nhất quán (Consistency) và Tính khả dụng (Availability) thông qua Định lý CAP và các mô hình triển khai thực tế.

---

## 1. Định lý CAP & Mở rộng PACELC

Định lý CAP phát biểu rằng một hệ thống phân tán không thể đảm bảo đồng thời cả 3 yếu tố: **C**onsistency (Nhất quán), **A**vailability (Sẵn sàng), và **P**artition Tolerance (Chịu lỗi mạng).
Vì trong môi trường mạng, việc đứt kết nối (Network Partition) là điều không thể tránh khỏi, nên thực tế ta chỉ được chọn:

- **CP (Consistency + Partition Tolerance)**: Ưu tiên dữ liệu đúng. Nếu mạng lỗi, hệ thống thà ngừng phục vụ (trả lỗi) còn hơn trả về dữ liệu sai. Ví dụ: Ngân hàng.
- **AP (Availability + Partition Tolerance)**: Ưu tiên luôn trả lời. Nếu mạng lỗi, cứ trả về dữ liệu cũ, miễn là user không bị chặn. Ví dụ: Mạng xã hội.

### Mở rộng: Định lý PACELC

CAP chỉ nói về lúc mạng bị lỗi (Partition). Vậy lúc mạng bình thường (Else) thì sao? PACELC bổ sung thêm:

- Nếu có Partition (**P**): Chọn **A** hoặc **C**.
- Nếu không có Partition (**E** - Else): Chọn **L** (Latency - Tốc độ) hoặc **C** (Consistency - Nhất quán).
Ví dụ: DynamoDB hay Cassandra cho phép bạn tùy chỉnh cấu hình để chọn tốc độ (L) hay độ chính xác (C) ngay cả khi mạng bình thường.

---

## 2. Các mức độ Nhất quán (Consistency Models)

### Strong Consistency (Nhất quán mạnh)

Mọi thao tác đọc *sau* thao tác ghi thành công đều phải trả về kết quả mới nhất.

- **Cơ chế**: Hệ thống thường dùng các giao thức đồng thuận (Consensus Protocol) như **Paxos** hoặc **Raft**. Khi một node nhận lệnh ghi, nó phải đồng bộ với các node khác trước khi báo thành công.
- **Đánh đổi**: Độ trễ (Latency) cao và Hiệu năng ghi thấp.

### Eventual Consistency & Xung đột dữ liệu

Hệ thống đảm bảo rằng nếu không có lệnh ghi mới, đến một lúc nào đó, tất cả bản sao sẽ đồng nhất dữ liệu. Tuy nhiên, vấn đề lớn nhất là **xung đột dữ liệu** khi nhiều node cùng nhận lệnh ghi cho cùng một key.

1. **Vector Clocks**: Để phát hiện xung đột, mỗi dữ liệu được gắn một chuỗi cặp `[node, version]`.
    - Nếu phiên bản mới có vector clock "lớn hơn" (ancestor) phiên bản cũ, nó sẽ ghi đè.
    - Nếu hai vector clock không so sánh được (conflict), hệ thống sẽ lưu cả hai và để ứng dụng tự giải quyết (Conflict Resolution) khi đọc. Đây là cách DynamoDB xử lý giỏ hàng khi có sự cố mạng.
2. **Anti-entropy (Chống nhiễu loạn)**: Để các node luôn tiến tới sự đồng nhất, chúng sử dụng **Gossip Protocol**. Mỗi node định kỳ chọn ngẫu nhiên một node khác để trao đổi thông tin, giúp dữ liệu lan truyền như "tin đồn" trong toàn bộ cluster mà không cần một server trung tâm.

---

## 3. Quorum: Công thức cân bằng

Làm sao để vừa muốn nhất quán (gần như Strong) mà vừa muốn nhanh? Chúng ta dùng kỹ thuật **Quorum (Bỏ phiếu)**.

Công thức: `R + W > N`

- **N**: Tổng số bản sao (Replicas).
- **W**: Số node cần ghi thành công để coi là Write OK.
- **R**: Số node cần đọc thành công để coi là Read OK.

**Cơ chế hoạt động**:
Nếu `R + W > N`, theo nguyên lý Dirichlet (chuồng bồ câu), tập hợp các node vừa ghi và tập hợp các node vừa đọc chắc chắn sẽ giao nhau ít nhất 1 node. Node giao nhau này chứa dữ liệu mới nhất.

- Ví dụ `N=3`.
  - Chọn `W=2`, `R=2` -> `2+2 > 3` -> Đảm bảo Strong Consistency.
  - Chọn `W=1` (Ghi cực nhanh), `R=3` (Đọc chậm, phải đọc hết) -> Vẫn Strong Consistency.
  - Chọn `W=1`, `R=1` -> `1+1 < 3` -> Write/Read cực nhanh nhưng chỉ đạt Eventual Consistency (có thể đọc phải dữ liệu cũ).

Đây là cách các NoSQL DB như Cassandra hay DynamoDB cho phép bạn điều chỉnh linh hoạt giữa Tốc độ và Nhất quán trên từng câu truy vấn.

---

## Tổng kết

Hiểu về Consistency không chỉ là chọn SQL hay NoSQL:

1. **CAP** ép ta chọn giữa CP (Ngân hàng) và AP (Mạng xã hội).
2. **PACELC** nhắc ta rằng ngay cả khi mạng tốt, vẫn phải đánh đổi giữa Tốc độ (Latency) và Nhất quán.
3. **Quorum (R + W > N)** là công cụ toán học mạnh mẽ để tinh chỉnh hệ thống theo nhu cầu thực tế.
