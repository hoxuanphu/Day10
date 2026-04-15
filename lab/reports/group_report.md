# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** ___________  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| ___ | Ingestion / Raw Owner | ___ |
| ___ | Cleaning & Quality Owner | ___ |
| Phạm Anh Quân| Embed & Idempotency Owner | hquan123cp04@gmail.com |
| Đào Danh Đăng Phụng | Monitoring / Docs Owner | phung352100@gmail.com |

**Ngày nộp:** ___________  
**Repo:** ___________  
**Độ dài khuyến nghị:** 600–1000 từ

---

> **Nộp tại:** `reports/group_report.md`  
> **Deadline commit:** xem `SCORING.md` (code/trace sớm; report có thể muộn hơn nếu được phép).  
> Phải có **run_id**, **đường dẫn artifact**, và **bằng chứng before/after** (CSV eval hoặc screenshot).

---

## 1. Pipeline tổng quan (150–200 từ)

> Nguồn raw là gì (CSV mẫu / export thật)? Chuỗi lệnh chạy end-to-end? `run_id` lấy ở đâu trong log?

**Tóm tắt luồng:**

Pipeline ETL đọc export CSV từ `data/raw/`, áp dụng rule làm sạch trong `transform/cleaning_rules.py` (allowlist `doc_id`, chuẩn hóa `effective_date`, quarantine HR cũ, dedupe, fix refund 14→7 + 3 rule mới: BOM/control, legacy marker, IT lockout>10). Sau đó chạy expectation suite `quality/expectations.py` (2 expectation mới: không marker legacy; refund phải chứa 7d). Nếu tất cả halt-pass, pipeline embed vào ChromaDB với idempotency: upsert theo `chunk_id` và prune id thừa. Mỗi run ghi `manifest_*.json` và log vào `artifacts/` để phục vụ freshness check và audit. Sprint 3 inject cố ý (tắt fix refund, thêm BOM/threshold) cho thấy retrieval degrade ở `q_refund_window`.

**Lệnh chạy một dòng (copy từ README thực tế của nhóm):**

Windows PowerShell (one-liner):

```powershell
python -m venv .venv; .\.venv\Scripts\Activate; pip install -r requirements.txt; python etl_pipeline.py run; python eval_retrieval.py --out artifacts/eval/before_eval.csv
```

---

## 2. Cleaning & expectation (150–200 từ)

> Baseline đã có nhiều rule (allowlist, ngày ISO, HR stale, refund, dedupe…). Nhóm thêm **≥3 rule mới** + **≥2 expectation mới**. Khai báo expectation nào **halt**.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới (tên ngắn) | Trước (sprint2-metrics) | Sau / khi inject (sprint2-inject) | Chứng cứ (log / CSV) |
|-----------------------------------|--------------------------|-----------------------------------|----------------------|
| R1: Quarantine BOM/Control (`control_characters_or_bom`) | 0 | 1 | [run_sprint2-metrics.log](artifacts/logs/run_sprint2-metrics.log), [quarantine_sprint2-inject.csv](artifacts/quarantine/quarantine_sprint2-inject.csv) |
| R2: Quarantine Legacy Marker (`stale_or_legacy_marker`) | 1 | 1 | [quarantine_sprint2-metrics.csv](artifacts/quarantine/quarantine_sprint2-metrics.csv) |
| R3: IT Lockout Threshold >10 (`it_lockout_threshold_suspect`) | 0 | 1 | [quarantine_sprint2-inject.csv](artifacts/quarantine/quarantine_sprint2-inject.csv) |
| E7: Expectation `no_legacy_or_migration_markers` (halt) | violations=0 | violations=0 | [run_sprint2-metrics.log](artifacts/logs/run_sprint2-metrics.log) |
| E8: Expectation `refund_contains_7d_window` (halt) | refund_rows=1 | refund_rows=2 | [run_sprint2-inject.log](artifacts/logs/run_sprint2-inject.log) |
| E3 (baseline): `refund_no_stale_14d_window` — fail khi tắt fix | pass | FAIL (violations=1) | [run_sprint2-inject-nofix.log](artifacts/logs/run_sprint2-inject-nofix.log) |

**Rule chính (baseline + mở rộng):**

- Baseline: allowlist `doc_id`, chuẩn hoá `effective_date` (YYYY-MM-DD), quarantine HR cũ (`< 2026-01-01`), dedupe nội dung, fix refund 14→7.
- R1 (mới): quarantine chunk có ký tự control/BOM → chặn lỗi encoding khi ingest (thấy 1 bản ghi khi inject BOM).
- R2 (mới): quarantine nếu chứa marker legacy/migration note (`deprecated`, `legacy`, `policy-v3`, …) → loại bỏ bản sync cũ.
- R3 (mới): guardrail IT — quarantine nếu ngưỡng khoá tài khoản >10 lần đăng nhập sai (phát hiện cấu hình bất thường trên inject).

**Ví dụ 1 lần expectation fail (nếu có) và cách xử lý:**

- Tắt rule fix refund bằng `--no-refund-fix` với bộ inject có 1 dòng `14 ngày làm việc` không mang marker legacy ⇒ expectation `refund_no_stale_14d_window` FAIL (halt) với `violations=1` (xem [run_sprint2-inject-nofix.log](artifacts/logs/run_sprint2-inject-nofix.log)). Khắc phục: bật lại fix (mặc định) hoặc sửa nguồn; expectation pass ở run thường.

---

## 3. Before / after ảnh hưởng retrieval hoặc agent (200–250 từ)

> Bắt buộc: inject corruption (Sprint 3) — mô tả + dẫn `artifacts/eval/…` hoặc log.

**Kịch bản inject:**

Kịch bản inject: dùng `data/raw/policy_export_inject_sprint2.csv` chứa 1 dòng refund "14 ngày làm việc" không gắn marker, 1 dòng lockout 12 lần (IT), và 1 dòng BOM/control. Chạy `python etl_pipeline.py run --run-id sprint3-after-inject --no-refund-fix --skip-validate` để ép embed snapshot xấu.

**Kết quả định lượng (từ CSV / bảng):**

Tóm tắt định lượng (top-k=3):
- `q_refund_window`: Before `hits_forbidden=no` → After_bad `hits_forbidden=yes` (14 ngày lọt vào top-k).
- `q_leave_version`: Before/After_bad đều OK (bản HR 2025 bị quarantine theo rule baseline).
- `q_p1_sla`, `q_lockout`: không suy giảm trong mẫu này.

CSV: [artifacts/eval/before_eval.csv](../artifacts/eval/before_eval.csv), [artifacts/eval/after_inject_bad.csv](../artifacts/eval/after_inject_bad.csv)

---

## 4. Freshness & monitoring (100–150 từ)

> SLA bạn chọn, ý nghĩa PASS/WARN/FAIL trên manifest mẫu.

Sử dụng `python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_<run-id>.json` để tính tuổi dữ liệu theo `latest_exported_at`. Tiêu chí: PASS ≤ 12h, WARN 12–24h, FAIL > 24h. Với dữ liệu mẫu, cả hai manifest `sprint3-before` và `sprint3-after-inject` cho FAIL do xuất khẩu ngày 2026-04-10 (vượt SLA 24h). Kịch bản vận hành: nếu FAIL, bật banner “data stale” cho agent và lập lịch cập nhật nguồn / rerun pipeline; đồng thời theo dõi `quarantine_records` tăng bất thường (BOM/threshold/legacy) để phát hiện nhiễu nguồn.

---

## 5. Liên hệ Day 09 (50–100 từ)

> Dữ liệu sau embed có phục vụ lại multi-agent Day 09 không? Nếu có, mô tả tích hợp; nếu không, giải thích vì sao tách collection.

Snapshot Chroma `day10_kb` phục vụ trực tiếp retrieval của Day 09. Khi pipeline publish, cơ chế upsert+prune đảm bảo index là ảnh chụp “đúng version” — tránh vector cũ len vào top-k. Điều này hỗ trợ agent Day 09 trả lời theo chính sách 2026 (7 ngày refund, 12 ngày phép) thay vì nội dung cũ.

---

## 6. Rủi ro còn lại & việc chưa làm

- Chưa tự động hoá cảnh báo freshness (mới có CLI).
- Chưa hợp nhất before/after vào một CSV có cột `scenario` (đang dùng 2 file).
- Chưa có check leakage nội dung "draft" (có thể thêm marker riêng nếu mở rộng).

---

### Peer review (Phần E)

- Q1 `q_refund_window`: Before OK, After_bad degrade (forbidden hit) — pass sau khi bật fix.
- Q2 `q_p1_sla`: Before/After_bad OK.
- Q3 `q_leave_version`: Before/After_bad OK; top1 từ `hr_leave_policy` đúng kỳ vọng.
