# Giới thiệu

## System Design Interview là gì?

**System Design Interview** (Phỏng vấn thiết kế hệ thống) là một vòng phỏng vấn kỹ thuật đặc thù, nơi bạn được yêu cầu thiết kế kiến trúc cho một hệ thống phần mềm giả định thay vì viết code chi tiết.

Thông thường, người phỏng vấn sẽ đưa ra một bài toán mở, ví dụ:

> *"Hãy thiết kế một mạng xã hội chia sẻ ảnh như Instagram."*
>
> *"Hãy thiết kế một hệ thống chat như Messenger."*

Nhiệm vụ của bạn là thảo luận và đề xuất một **giải pháp kiến trúc tổng thể**, bao gồm:

1. **Các thành phần chính**: Server, Database, Cache, Message Queue...
2. **Cách tương tác**: Dữ liệu đi từ đâu đến đâu?
3. **Đáp ứng yêu cầu**: Làm sao để hệ thống chạy nhanh, chịu tải tốt, và không bị sập?

Trong vòng này, bạn **không cần viết code** (hoặc viết rất ít). Thay vào đó, bạn tập trung vào **tư duy kiến trúc**, **vẽ sơ đồ** (trên bảng hoặc công cụ), và **biện luận** cho các quyết định của mình.

### Tại sao vòng phỏng vấn này quan trọng?

Tại các công ty công nghệ lớn (Big Tech) và các vị trí Senior trở lên, đây là vòng phỏng vấn **mang tính quyết định**.

- **Đánh giá tư duy**: Kiểm tra khả năng nhìn bức tranh toàn cảnh (Big Picture) thay vì chỉ chi tiết code.
- **Dự đoán hiệu suất làm việc**: Người thiết kế tốt sẽ giúp công ty tiết kiệm hàng triệu đô la chi phí vận hành và bảo trì sau này.
- **Ảnh hưởng đến Level/Lương**: Kết quả vòng này thường định định cấp bậc (Senior, Staff, Principal) và mức lương engineer.

---

## Những khó khăn người mới thường gặp

Nếu bạn là Junior hoặc chưa từng làm hệ thống lớn, bạn sẽ dễ gặp các "cú sốc" sau:

1. **Câu hỏi quá mở (Open-ended)**:
    - Không có đáp án Đúng/Sai tuyệt đối.
    - *Ví dụ:* "Dùng SQL hay NoSQL?" -> Câu trả lời đúng là "Tùy vào trường hợp", và bạn phải giải thích được cái "Tùy" đó.

2. **Thiếu kinh nghiệm thực tế**:
    - Bạn có thể chưa bao giờ chạm vào một hệ thống có 1 triệu người dùng.
    - Bạn nghe nói về *Load Balancer*, *Redis*, *Kafka*... nhưng chưa biết ghép chúng lại với nhau thế nào cho hợp lý.

3. **Áp lực thời gian**:
    - Bạn chỉ có 45-60 phút để thiết kế một hệ thống mà thực tế các kỹ sư phải mất nhiều tháng để xây dựng.
    - Kỹ năng quản lý thời gian và đi vào trọng tâm là cực kỳ quan trọng.

4. **Ám ảnh về "Sự hoàn hảo"**:
    - Nhiều bạn sợ thiết kế của mình có lổ hổng.
    - *Lời khuyên:* Không ai mong đợi một thiết kế hoàn hảo trong 45 phút. Người phỏng vấn tìm kiếm **quy trình suy nghĩ (thought process)** và khả năng **cân nhắc đánh đổi (trade-off)** của bạn.

---

## Các kỹ năng nhà tuyển dụng tìm kiếm

Để vượt qua vòng này, bạn cần trang bị "balo kỹ năng" gồm:

### 1. Kiến thức nền tảng vững vàng

Bạn không thể thiết kế nếu không hiểu nguyên liệu. Hãy nắm chắc:

- **Cơ sở dữ liệu**: SQL vs NoSQL, Replication, Sharding.
- **Hệ thống phân tán**: CAP Theorem, Consistency (Tính nhất quán).
- **Thành phần hạ tầng**: Load Balancer, Caching, CDN, Message Queue.

### 2. Tư duy "Chia để trị"

Bài toán lớn (như thiết kế Facebook) rất đáng sợ. Kỹ năng quan trọng là biết chia nhỏ nó ra:

- Xác định yêu cầu (Scope).
- Thiết kế API.
- Thiết kế Database.
- Giải quyết các vấn đề về quy mô (Scaling).

### 3. Khả năng cân nhắc (Trade-off)

Đây là kỹ năng "ăn tiền" nhất. Mọi quyết định kỹ thuật đều có hai mặt.

- *Ví dụ:* Chọn **Consistency** (Nhất quán) thì có thể phải hy sinh **Availability** (Sẵn sàng).
- Người giỏi là người biết chọn cái nào quan trọng hơn trong ngữ cảnh bài toán cụ thể.

### 4. Kỹ năng giao tiếp

Phỏng vấn System Design là một cuộc **đối thoại**, không phải bài kiểm tra một chiều. Hãy chủ động đặt câu hỏi, lắng nghe gợi ý và thảo luận như hai đồng nghiệp.

---

## 🔥 Quy trình 4 bước "vàng" khi phỏng vấn

Để không bị lạc lối trong 45-60 phút ngắn ngủi, hãy áp dụng khung sườn sau (được đúc kết từ các chuyên gia tại Google/Meta):

1. **Bước 1: Hiểu bài toán & Xác định phạm vi (3-10 phút)**:
    - **Đừng như "Jimmy"**: Đừng nhảy vào giải quyết ngay. Hãy hỏi để làm rõ: "Hệ thống cho bao nhiêu người dùng?", "Có hỗ trợ Video không?", "Ưu tiên độ trễ hay tính nhất quán?".
    - Viết ra các giả định (Assumptions) về lượng traffic (DAU), dung lượng lưu trữ.
2. **Bước 2: Đề xuất thiết kế tổng thể (10-15 phút)**:
    - Vẽ sơ đồ khối (Boxes & Arrows) các thành phần chính: Web Server, API Gateway, Load Balancer, DB, Cache...
    - Yêu cầu phản hồi (Feedback) từ người phỏng vấn để chắc chắn bạn đang đi đúng hướng.
3. **Bước 3: Đi sâu vào chi tiết (Deep Dive) (10-25 phút)**:
    - Đây là lúc bạn tỏa sáng kỹ năng chuyên môn. Tùy vào bài toán, hãy đi sâu vào: Thuật toán Hash, Thiết kế Database schema, Cơ chế Replication, hay cách xử lý lỗi.
4. **Bước 4: Tổng kết & Mở rộng (3-5 phút)**:
    - Tóm tắt lại thiết kế.
    - Chỉ ra các điểm nghẽn (Bottlenecks) tiềm ẩn.
    - Đề xuất cách cải tiến nếu có thêm thời gian hoặc khi số lượng người dùng tăng gấp 10 lần.

---

## Tóm tắt

Vòng phỏng vấn System Design không đáng sợ nếu bạn có sự chuẩn bị bài bản. Nó là cơ hội để bạn chứng minh mình không chỉ là một thợ code (), mà là một kỹ sư có tư duy kiến trúc ().

Trong các chương tiếp theo, tôi sẽ trang bị cho bạn từng vũ khí cụ thể để tự tin bước vào phòng phỏng vấn. Hãy bắt đầu với những khái niệm nền tảng nhé!
