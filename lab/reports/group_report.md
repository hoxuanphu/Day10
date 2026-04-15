# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** AI-Pipeline-Team  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Hồ Xuân Phú | Ingestion Owner; Cleaning & Quality Owner | (team member) |
| Đào Danh Đăng Phụng | Monitoring / Docs Owner | phung352100@gmail.com |

**Ngày nộp:** 2026-04-15  
**Repo:** Day10 lab  
**Độ dài khuyến nghị:** 600–1000 từ

---

## 1. Pipeline tổng quan

Pipeline ETL đọc export CSV từ `data/raw/policy_export_dirty.csv`, áp dụng rule làm sạch trong `transform/cleaning_rules.py` (allowlist `doc_id`, chuẩn hóa `effective_date`, quarantine HR cũ, dedupe, fix refund 14→7 + 3 rule mới: BOM/control, legacy marker, IT lockout>10). Sau đó chạy expectation suite `quality/expectations.py` (8 expectation, gồm 2 mới: E7 không marker legacy; E8 refund phải chứa 7d). Nếu tất cả halt-pass, pipeline embed vào ChromaDB collection `day10_kb` với idempotency: upsert theo `chunk_id` ổn định (SHA-256 từ `doc_id|chunk_text|seq`) và prune id thừa. Mỗi run ghi `manifest_*.json` và log vào `artifacts/` để phục vụ freshness check và audit.

**Kết quả run cuối (run_id=`final-clean`):**
- `raw_records=10`, `cleaned_records=5`, `quarantine_records=5`
- Tất cả 8 expectation OK (không halt)
- `embed_prune_removed=2` (dọn vector cũ từ inject trước đó)
- Freshness: FAIL (dữ liệu mẫu exported 2026-04-10, age ~123h > SLA 24h — hợp lý, xem mục 4)

**Lệnh chạy một dòng (PowerShell):**

```powershell
python -m venv .venv; .\.venv\Scripts\Activate; pip install -r requirements.txt; python etl_pipeline.py run; python eval_retrieval.py --out artifacts/eval/before_eval.csv
```

---

## 2. Cleaning & expectation

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới (tên ngắn) | Trước (final-clean) | Sau / khi inject (inject-bad) | Chứng cứ (log / CSV) |
|-----------------------------------|----------------------|-------------------------------|----------------------|
| R1: Quarantine BOM/Control (`control_characters_or_bom`) | quarantine=0 | quarantine tăng do BOM trong inject | [quarantine_inject-bad.csv](../artifacts/quarantine/quarantine_inject-bad.csv) |
| R2: Quarantine Legacy Marker (`stale_or_legacy_marker`) | quarantine=1 (dòng 3 CSV dirty chứa "bản sync cũ policy-v3") | quarantine=1 | [quarantine_final-clean.csv](../artifacts/quarantine/quarantine_final-clean.csv) |
| R3: IT Lockout >10 (`it_lockout_threshold_suspect`) | quarantine=0 | quarantine=1 (dòng 12 inject: 12 lần) | [quarantine_inject-bad.csv](../artifacts/quarantine/quarantine_inject-bad.csv) |
| E7: `no_legacy_or_migration_markers` (halt) | violations=0 | violations=0 (đã quarantine trước khi vào cleaned) | [run_final-clean.log](../artifacts/logs/run_final-clean.log) |
| E8: `refund_contains_7d_window` (halt) | refund_rows=1, OK | refund_rows=2, OK | [run_inject-bad.log](../artifacts/logs/run_inject-bad.log) |
| E3 (baseline): `refund_no_stale_14d_window` (halt) | pass (violations=0) | FAIL (violations=1) khi tắt fix | [run_inject-bad.log](../artifacts/logs/run_inject-bad.log) |

**Rule chính:**
- Baseline: allowlist `doc_id`, chuẩn hoá `effective_date`, quarantine HR cũ (< 2026-01-01), dedupe nội dung, fix refund 14→7.
- R1 (mới): quarantine chunk có ký tự control/BOM — chặn lỗi encoding.
- R2 (mới): quarantine nếu chứa marker legacy/migration ("bản sync cũ", "deprecated", "policy-v3"…).
- R3 (mới): guardrail IT — quarantine nếu lockout threshold >10 lần đăng nhập sai.

**Ví dụ expectation fail:**
- Inject: tắt fix refund bằng `--no-refund-fix` → expectation `refund_no_stale_14d_window` FAIL (halt, violations=1). Phải dùng `--skip-validate` để tiếp tục embed cho demo. Khắc phục: bật lại fix (mặc định).

---

## 3. Before / after ảnh hưởng retrieval

**Kịch bản inject:** dùng `data/raw/policy_export_inject_sprint2.csv` chứa 1 dòng refund "14 ngày làm việc" không gắn marker, 1 dòng lockout 12 lần (IT), 1 dòng BOM. Chạy `python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate` để ép embed snapshot xấu.

**Kết quả (top-k=3):**

| Câu hỏi | Before (final-clean) | After inject (inject-bad) |
|---------|---------------------|--------------------------|
| `q_refund_window` | contains_expected=yes, **hits_forbidden=no** | contains_expected=yes, **hits_forbidden=yes** (14 ngày lọt top-k) |
| `q_p1_sla` | contains_expected=yes, hits_forbidden=no | contains_expected=yes, hits_forbidden=no |
| `q_lockout` | contains_expected=yes, hits_forbidden=no | contains_expected=yes, hits_forbidden=no |
| `q_leave_version` | contains_expected=yes, hits_forbidden=no, top1_doc_expected=yes | contains_expected=yes, hits_forbidden=no, top1_doc_expected=yes |

CSV: [before_eval.csv](../artifacts/eval/before_eval.csv), [after_inject_bad.csv](../artifacts/eval/after_inject_bad.csv)

Rõ ràng inject gây degradation ở `q_refund_window`: chunk "14 ngày làm việc" lọt vào top-k khiến `hits_forbidden=yes`. Sau khi chạy lại pipeline chuẩn (`final-clean`), prune xóa 2 vector inject, eval trở lại `hits_forbidden=no`.

---

## 4. Freshness & monitoring

Sử dụng `python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_final-clean.json` để tính tuổi dữ liệu theo `latest_exported_at`. Tiêu chí: PASS ≤ SLA 24h, FAIL > 24h.

Kết quả: **FAIL** — `latest_exported_at=2026-04-10T08:00:00`, `age_hours=123.2h` > SLA 24h. Đây là hợp lý vì CSV mẫu có timestamp cũ. Kịch bản vận hành: nếu FAIL, bật banner "data stale" cho agent và lập lịch cập nhật nguồn / rerun pipeline; đồng thời theo dõi `quarantine_records` tăng bất thường.

Chi tiết SLA và xử lý ghi tại: [docs/runbook.md](../docs/runbook.md)

---

## 5. Liên hệ Day 09

Snapshot Chroma `day10_kb` phục vụ trực tiếp retrieval của Day 09. Khi pipeline publish, cơ chế upsert+prune đảm bảo index là ảnh chụp "đúng version" — tránh vector cũ len vào top-k. Điều này hỗ trợ agent Day 09 trả lời theo chính sách 2026 (7 ngày refund, 12 ngày phép) thay vì nội dung cũ.

---

## 6. Grading JSONL

File: [artifacts/eval/grading_run.jsonl](../artifacts/eval/grading_run.jsonl)

| Câu | contains_expected | hits_forbidden | top1_doc_matches | Kết quả |
|-----|------------------|----------------|-----------------|---------|
| `gq_d10_01` (refund 7d) | true | false | — | PASS |
| `gq_d10_02` (P1 resolution 4h) | true | false | — | PASS |
| `gq_d10_03` (HR 12 ngày phép) | true | false | true | PASS |

Instructor quick check: tất cả MERIT_CHECK OK.

---

## 7. Rủi ro còn lại & việc chưa làm

- Chưa tự động hoá cảnh báo freshness (mới có CLI).
- Chưa hợp nhất before/after vào một CSV có cột `scenario`.
- Freshness hiện hard-coded SLA=24h qua env; có thể đọc từ contract.

---

### Peer review (Phần E)

- Q1 `q_refund_window`: Before OK → After_bad degrade (hits_forbidden=yes) → Sau fix: OK.
- Q2 `q_p1_sla`: Before/After_bad đều OK.
- Q3 `q_leave_version`: Before/After_bad đều OK; top1 từ `hr_leave_policy` đúng kỳ vọng.
