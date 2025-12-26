# System Design Interview Guide

Tài liệu học tập và chuẩn bị cho vòng phỏng vấn System Design (Thiết kế Hệ thống).

## 📚 Giới thiệu

Repository này chứa các tài liệu, bài học và hướng dẫn về System Design Interview - một vòng phỏng vấn kỹ thuật quan trọng tại các công ty công nghệ lớn.

## 📖 Nội dung

- [01. Giới thiệu](./01_Introduction.md) - Tổng quan về System Design Interview, các kỹ năng cần thiết và quy trình 4 bước "vàng"
- [02. Các khái niệm nền tảng trong Thiết kế hệ thống](./02_Foundational_Concepts_in_System_Design.md) - Latency & Throughput, Scalability, Availability, và các thành phần kiến trúc cơ bản
- [03. Tính nhất quán và khả dụng trong System Design](./03_Consistency_and_Availability_in_System_Design.md) - Định lý CAP, PACELC, các mô hình nhất quán và Quorum
- [04. Cơ sở dữ liệu và Chiến lược Phân mảnh](./04_Database_Selection_Sharding_and_Storage_Optimization.md) - B-Tree vs LSM Tree, Sharding strategies, Consistent Hashing, và Distributed KV Store
- [05. Mở rộng hệ thống & Tối ưu hiệu suất](./05_Scalability_and_Performance_Optimization.md) - Load Balancer, Caching, Rate Limiting, Message Queue, và thiết kế News Feed
- [06. Idempotency & Cơ chế Khóa](./06_Idempotency_and_Locking_Mechanisms.md) - Idempotency Key, Unique ID Generator, Pessimistic/Optimistic Locking, Distributed Lock, và URL Shortener
- [07. Các phương thức giao tiếp & Thiết kế API](./07_Communication_Methods_and_API_Design.md) - TCP vs UDP, REST vs gRPC, WebSocket, Long Polling, và thiết kế hệ thống Chat
- [08. Khả năng chịu lỗi & Phục hồi hệ thống](./08_Fault_Tolerance_and_System_Recovery.md) - Redundancy, Replication, Failover, Circuit Breaker, Bulkhead Pattern, và Retry strategies
- [09. Bảo mật trong hệ thống phân tán](./09_Security_in_Modern_Distributed_Systems.md) - Authentication & Authorization, JWT, OAuth2, Password Storage, HTTPS & TLS

## 🎯 Mục tiêu

Giúp bạn:

- Hiểu rõ về System Design Interview và tầm quan trọng của nó
- Nắm vững các khái niệm nền tảng về thiết kế hệ thống
- Rèn luyện tư duy kiến trúc và khả năng đánh giá trade-off
- Tự tin bước vào phòng phỏng vấn với sự chuẩn bị bài bản

## 🚀 Bắt đầu

Bắt đầu với [chương Giới thiệu](./01_Introduction.md) để hiểu rõ về System Design Interview và cách tiếp cận hiệu quả.

---

*Tài liệu này được biên soạn dựa trên kinh nghiệm thực tế và các best practices từ các chuyên gia tại Google, Meta và các công ty công nghệ hàng đầu.*
