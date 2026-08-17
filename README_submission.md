# Báo Cáo Thu Hoạch: Multi-Memory Agent với Zep (Lab 17)

## 1. Phân Tích Benchmark
- **Layer rủi ro nhất (thiết kế):** Episodic — ngân sách chỉ 3% (240 token) dễ cắt mất episode chứa marker nếu không dùng `episode_char_cap`; run cuối đạt 100% hit rate (11/11) sau khi tune tham số này.
- **Query retrieve nhiều token nhất:** E02/E03 (Long-term, 1340 token) — Context Block trả toàn bộ user context, chưa bị budget 4% trim mạnh.
- **Case E07 (Mixed, 485 token):** Kết hợp `Long-term` (sở thích `Python`) và `Semantic` (quy tắc `Idempotency-Key`); cả hai marker bắt buộc xuất hiện dù ngân sách nhỏ hơn nhiều so với E02/E03.
- **Token Reduction vs Hit Rate:** No-memory baseline đạt mức giảm token tuyệt đối nhưng hit rate chỉ đạt 2/11 (~18%) vì thiếu dữ liệu lịch sử. Retrieval rỗng rất rẻ nhưng vô giá trị; kiểm soát token chỉ có ý nghĩa khi đi kèm với hit rate cao (>= 80%).

## 2. Trả Lời Câu Hỏi Phản Biện (Reflection)
- **Layer quan trọng nhất trong bộ test:** `Long-term Memory` (chiếm các case E02, E03, E08, E09). Nó đảm bảo khả năng duy trì sở thích cross-session, phát hiện xung đột/công nghệ mới (Recency) và cô lập dữ liệu giữa người dùng (User Isolation).
- **Trade-off Zep Context Block vs Tự dựng Redis + Qdrant:**
  - *Zep Context Block:* Tự động hóa trích xuất entity, đồ thị quan hệ và tính toán relevance động theo thread, nhưng phụ thuộc vào cloud API và latency mạng.
  - *Redis + Qdrant:* Toàn quyền kiểm soát hạ tầng on-premise và độ trễ thấp, nhưng đòi hỏi tự viết pipeline đồng bộ vector, xử lý recency và schema phức tạp.
- **Guardrail chống Memory Poisoning:** Bắt buộc áp dụng consent opt-in, kiểm duyệt/sanitization làm sạch PII trước khi lưu durable memory, và chỉ cho phép ghi nhận fact từ các nguồn/hành động đã được xác thực an toàn.

## 3. Phân Tích Case Đặc Thù
- **E08 (Recency):** Khi user đổi từ Python sang TypeScript/NestJS ở session mới, Zep trích xuất fact mới nhất (`BLUEBIRD-42`, `TypeScript`, `NestJS`) và ưu tiên fact mới hơn để giải quyết xung đột.
- **E10 (Compaction):** Khi số lượng turn vượt ngưỡng, short-term memory nén hội thoại cũ nhưng vẫn bảo toàn ràng buộc quan trọng (`REVIEW-DEADLINE-1600`, `Friday`, `16:00`) qua cơ chế trích xuất durable notes.
