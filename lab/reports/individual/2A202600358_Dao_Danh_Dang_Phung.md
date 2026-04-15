# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Đào Danh Đăng Phụng  
**Mã học viên:** 2A202600358  
**Email:** phung352100@gmail.com  
**Vai trò:** Monitoring / Docs Owner  
**Ngày nộp:** 2026-04-15  
**Độ dài:** ~550 từ

---

## 1. Tôi phụ trách phần nào?

**File / module:**

- `monitoring/freshness_check.py` — module kiểm tra freshness từ manifest pipeline, so sánh `latest_exported_at` với SLA (mặc định 24h), trả về PASS/WARN/FAIL kèm detail dict (age_hours, sla_hours, reason).
- `docs/pipeline_architecture.md` — sơ đồ Mermaid luồng pipeline (Raw → Clean → Validate → Embed → Manifest), bảng ranh giới trách nhiệm, idempotency, liên hệ Day 09, rủi ro đã biết.
- `docs/data_contract.md` — source map 5 nguồn với failure mode và metric, schema cleaned (5 cột), quy tắc quarantine vs drop.
- `docs/runbook.md` — 5 mục Symptom → Detection → Diagnosis → Mitigation → Prevention; kèm lệnh freshness và tiêu chí PASS/WARN/FAIL.
- `contracts/data_contract.yaml` — điền owner_team, alert_channel, đồng bộ thông tin freshness SLA.
- `reports/group_report.md` — tổng hợp kết quả pipeline, metric_impact, before/after, freshness, grading.

**Kết nối với thành viên khác:**

Hồ Xuân Phú (Ingestion/Cleaning/Quality Owner) hoàn thành code pipeline và rule cleaning. Tôi nhận output từ các run (manifest, log, quarantine CSV, eval CSV) để viết documentation, chạy freshness check, và tổng hợp báo cáo nhóm.

**Bằng chứng:** commit trên các file `docs/*.md`, `contracts/data_contract.yaml`, `reports/group_report.md`, `monitoring/freshness_check.py`.

---

## 2. Một quyết định kỹ thuật

Khi thiết kế freshness check trong `monitoring/freshness_check.py` và runbook, tôi quyết định đo freshness tại boundary **publish** (dựa trên `latest_exported_at` trong manifest) thay vì boundary ingest. Lý do: `latest_exported_at` phản ánh thời điểm nguồn export, là thước đo trực tiếp "dữ liệu mới đến đâu" mà người dùng cuối quan tâm. SLA nhóm chọn 24h: nếu dữ liệu cũ hơn 24h → FAIL, cần cập nhật nguồn và rerun pipeline. Với dữ liệu mẫu (exported 2026-04-10), freshness luôn FAIL (age ~123h) — đây là kết quả hợp lý, được giải thích trong runbook: "FAIL là dự kiến với CSV mẫu; trong production cần lập lịch cập nhật định kỳ hoặc bật banner cảnh báo."

Tôi cũng quyết định phân tầng PASS/WARN/FAIL (thay vì chỉ binary) để vận hành linh hoạt hơn, dù module hiện tại chỉ hỗ trợ 2 mức (PASS vs FAIL theo SLA).

---

## 3. Một lỗi hoặc anomaly đã xử lý

Khi chạy pipeline inject trên Windows PowerShell, gặp lỗi `UnicodeEncodeError: 'charmap' codec can't encode character '\u2192'` do log message chứa ký tự Unicode "→" không tương thích với encoding cp1252 mặc định của Windows console. Lỗi này khiến pipeline crash giữa chừng, không ghi được manifest.

**Phát hiện:** traceback trực tiếp khi chạy `python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate`.

**Fix:** thay ký tự Unicode "→" bằng ASCII "->" trong log message của `etl_pipeline.py` dòng 91. Sau fix, inject run hoàn tất, ghi manifest và embed thành công. Đây là anomaly liên quan đến cross-platform compatibility — trên Linux/macOS không gặp vấn đề này.

---

## 4. Bằng chứng trước / sau

**Manifest (run_id=`final-clean` vs `inject-bad`):**
- Trước: `raw_records=10`, `cleaned_records=5`, `quarantine_records=5`
- Sau inject: `raw_records=13`, `cleaned_records=7`, `quarantine_records=6` (thêm 1 quarantine do lockout>10)

**Eval retrieval (top-k=3):**
- Trước (`before_eval.csv`): `q_refund_window` → `contains_expected=yes`, `hits_forbidden=no`
- Sau inject (`after_inject_bad.csv`): `q_refund_window` → `contains_expected=yes`, **`hits_forbidden=yes`** (chunk "14 ngày" lọt top-k)
- Sau fix lại (`final-clean` + prune): `hits_forbidden=no`, `embed_prune_removed=2`

**Freshness:**
- `freshness_check=FAIL`, `age_hours=123.2`, `sla_hours=24.0` — ghi rõ trong runbook.

**Grading (run_id=`final-clean`):**
- `gq_d10_01`: contains_expected=true, hits_forbidden=false
- `gq_d10_02`: contains_expected=true
- `gq_d10_03`: contains_expected=true, hits_forbidden=false, top1_doc_matches=true

---

## 5. Cải tiến tiếp theo (2 giờ)

Nếu có thêm 2 giờ, tôi sẽ mở rộng `monitoring/freshness_check.py` để đo freshness ở **2 boundary** (ingest timestamp + publish timestamp) và ghi log cả hai vào manifest — đạt điều kiện Distinction (b) và bonus +1. Cụ thể: thêm trường `ingest_timestamp` (thời điểm pipeline chạy) bên cạnh `latest_exported_at` (thời điểm nguồn export), so sánh cả hai với SLA riêng (ingest SLA 1h, publish SLA 24h), và ghi kết quả freshness chi tiết vào manifest thay vì chỉ log ra console.
