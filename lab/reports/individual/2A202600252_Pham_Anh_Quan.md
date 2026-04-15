# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Phạm Anh Quân  
**Vai trò:** Embed Owner  
**Ngày nộp:** 15-04-2026  
**Độ dài yêu cầu:** **400–650 từ** (ngắn hơn Day 09 vì rubric slide cá nhân ~10% — vẫn phải đủ bằng chứng)

---

> Viết **"tôi"**, đính kèm **run_id**, **tên file**, **đoạn log** hoặc **dòng CSV** thật.  
> Nếu làm phần clean/expectation: nêu **một số liệu thay đổi** (vd `quarantine_records`, `hits_forbidden`, `top1_doc_expected`) khớp bảng `metric_impact` của nhóm.  
> Lưu: `reports/individual/[ten_ban].md`

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

- Chịu trách nhiệm trực tiếp hàm `cmd_embed_internal()` trong file `etl_pipeline.py`.
- Tích hợp và vận hành ChromaDB (Collection `day10_kb`) nhằm tiến hành nhúng dữ liệu (Embedding) qua model `all-MiniLM-L6-v2`.
- Xác minh tính hiệu quả của cơ sở dữ liệu Vector bằng script test `eval_retrieval.py` đảm bảo context truy xuất không bị loãng.

**Kết nối với thành viên khác:**

- Phối hợp với Cleaning/Quality Owner: Trích xuất đầu ra của họ là file `cleaned_csv` để dùng làm Source of Truth (không lưu rác) đưa vào khâu Vectorization.
- Hỗ trợ Ingestion & Monitoring Owner lưu cấu hình db, collection path và tình trạng upsert vào `manifest` metadata phục vụ cho tác vụ Freshness check ở phần cuối.

**Bằng chứng (commit / comment trong code):**

Đóng góp trực tiếp quy trình upsert nguyên khối trong `etl_pipeline.py` (từ dòng 131-177), đặc biệt là block code đồng bộ dữ liệu:
`col.upsert(ids=ids, documents=documents, metadatas=metadatas)`
Kèm theo khai báo log tracking minh bạch: `log(f"embed_upsert count={len(ids)} collection={collection_name}")`.

---

## 2. Một quyết định kỹ thuật (100–150 từ)

**Thiết kế thuật toán "Idempotent Pruning (Cắt tỉa Vector mồi cũ)" thay vì Insert Append.**

**Bối cảnh:** Ở các lần chạy ETL liên tục, nếu một rule Cleaning cập nhật sửa lại dữ liệu (ví dụ: đổi policy thời gian refund từ 14 ngày làm việc thành 7 ngày), chunk vector cũ vẫn có khả năng cao lưu dính lặp lại trong không gian ChromaDB. Bộ Retrieval Agent sẽ quét ra cả hai, và lấy nhầm mồi cũ do text có tỉ lệ similarity cao, dẫn tới trigger lỗi vi phạm SLA.

**Quyết định xử lý:** Thay vì chỉ dùng lệnh add() truyền thống, tôi lập trình luồng *Stateful Idempotency*. Hàm Embed sẽ lấy tập DB ID cũ trừ đi tập ID vừa mới được pass qua hàm Cleaning (`drop = sorted(prev_ids - set(ids))`). Nếu Vector ID nào rác sẽ lập tức bị quét dọn qua lệnh `col.delete(ids=drop)`. 

**Hiệu quả:** Cách tiệm cận này đồng bộ được hoàn toàn file `cleaned.csv` sạch lên Database, bất kể tần suất chạy lại ETL là bao nhiêu lần đi chăng nữa. Hạn chế phình kích thước DB vô cớ theo thời gian và tăng độ chính xác 100% cho Retrieval Test.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

**Triệu chứng:** Pipeline báo Expectation Suite đã Passed, file CSV Cleaned đã lọc bỏ chính sách 14 ngày không hợp lệ. Tuy nhiên, script test Evaluator `eval_retrieval.py` cho ra thông báo Semantic Search vẫn rút nhầm thông tin cũ. Metric bị ảnh hưởng nặng ở tiêu chí `hits_forbidden = yes`.
**Nguyên nhân (Root Cause):** Khâu Embed thiếu chặt chẽ khi không nhận diện được ID Vector rác. ChromaDB mặc định cho phép nằm song song nhiều `chunk_id` độc lập nhau sinh ra từ đợt chạy trước đó của bộ dataset bị lặp dòng policy (duplicate records).
**Thực hiện fix:** Bật Rule Clean Prune List tại line 156 của `etl_pipeline.py`. Tôi cho try-catch vào khối `col.get(include=[])`, nhặt ra những ids "dư thừa" của file data clean đợt này. Lỗi triệt để bị loại bỏ. Tôi lưu lại bằng chứng vào file eval sau đó, hệ thống in trace `embed_prune_removed=X` ra console log.

---

## 4. Bằng chứng trước / sau (80–120 từ)

Dưới đây là một phần kết quả CSV từ file `before_after_eval.csv` với bằng chứng Vector Sạch, dựa trên lần đánh giá với rule fix triệt để.

```csv
question_id,question,top1_doc_id,contains_expected,hits_forbidden,top_k_used
q_refund_window,Khách hàng có bao nhiêu ngày để hoàn tiền...,policy_refund_v4,yes,no,3
q_leave_version,"Theo chính sách nghỉ phép (2026)...",hr_leave_policy,yes,no,3
```
Chỉ số `hits_forbidden` đã trở  về `no`, thay đổi này minh chứng cho tính đúng đắn khi Clean/Expectation Owner phối hợp cùng Embed Owner đã lọc xóa "dữ liệu độc" trong pipeline. Vector space cuối cùng chỉ lưu trữ đúng 1 bản duy nhất (Bản hoàn tiền 7 ngày).

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có dư 2 giờ rảnh rỗi, thiết lập chức năng "MD5 Content-Hashing" cho cột chunk id thay vì lệ thuộc tĩnh vào id do file thô cung cấp. Như vậy, mỗi khoảnh khắc text bị chỉnh sửa content dù chỉ là 1 ký tự, vector db sẽ phân định chính xác ID thay thế trước khi ném lên server Call Embedding API nhằm tiết kiệm chi phí chạy RAG ở mức Scale lớn.
