# Reflection — Top 5 Lakehouse Anti-Patterns

**Họ và tên:** Phạm Văn Thắng  
**Mã học viên:** 2A202601359  

---

Trong 5 anti-patterns của Data Lakehouse, hệ thống dữ liệu của team tôi dễ gặp rủi ro nhất với: **Small Files Problem kết hợp Runaway Uncommitted Orphans** trong các luồng Streaming/CDC Ingestion.

### Lý do & Bài học áp dụng:
1. **Bối cảnh thực tế:** Team thường xuyên thu thập log gọi LLM và telemetry từ microservices theo thời gian thực với các micro-batch nhỏ (vài giây/lần) đẩy thẳng vào Bronze layer.
2. **Rủi ro tiềm ẩn:** Việc ghi liên tục tạo ra hàng trăm nghìn file vài chục KB. Đáng chú ý, khi job ghi bị crash hoặc gián đoạn mạng, các file chưa commit trở thành *orphan files*. Như đo đạc tại NB6, lệnh `VACUUM` của Delta Lake chỉ thu hồi file đã tombstone trong log chứ **không thể dọn file chưa từng commit**, gây phình dung lượng lưu trữ ngầm và tăng vọt chi phí S3 GET/LIST.
3. **Giải pháp khắc phục:** 
   - Tách biệt luồng ghi và luồng đọc, định kỳ chạy **Job 1 (Compaction)** gom file về ngưỡng 128MB–512MB kết hợp **Job 2 (Clustering/Z-ORDER)** theo `user_id`/`tenant_id` để tối ưu stats skipping.
   - Bổ sung định kỳ **Job 4 (Orphan Cleanup)** bằng phép hiệu tập hợp (quét danh sách file thực tế trên đĩa trừ đi log active) để triệt để dọn rác uncommitted.
