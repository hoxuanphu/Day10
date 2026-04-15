# Data contract — Lab Day 10

Đồng bộ với `contracts/data_contract.yaml` (owner, SLA, schema). Bản tóm tắt vận hành ở dưới.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| policy_refund_v4 | CSV ingest | Cửa sổ 14d (stale), missing date | E3 fail count, missing date count |
| sla_p1_2026 | CSV ingest | BOM/control char | Quarantine `control_characters_or_bom` |
| it_helpdesk_faq | CSV ingest | Lockout threshold bất thường | Quarantine `it_lockout_threshold_suspect` |
| hr_leave_policy | CSV ingest | Stale version 2025 (10 ngày) | Quarantine `stale_hr_policy_effective_date` |
| legacy_catalog_* | CSV ingest | Doc_id ngoài allowlist | Quarantine `unknown_doc_id` |

---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| chunk_id | string | Có | Stable hash từ `doc_id|chunk_text|seq` |
| doc_id | string | Có | Allowlist: refund, sla, it_faq, hr_leave |
| chunk_text | string | Có | Dedupe theo text chuẩn hoá, loại BOM/control |
| effective_date | date (YYYY-MM-DD) | Có | Chuẩn hoá từ ISO hoặc dd/mm/yyyy |
| exported_at | datetime | Không | Truy vết nguồn; không dùng cho filter kỳ vọng |

---

## 3. Quy tắc quarantine vs drop

Record vi phạm đi vào `artifacts/quarantine/*.csv` với cột `reason`. Chủ nguồn (owner) xem xét và hợp thức hoá/thay đổi rule nếu cần, sau đó mới được merge lại nguồn.

---

## 4. Phiên bản & canonical

Canonical: `data/docs/policy_refund_v4.txt` (nội dung văn bản) và `doc_id=policy_refund_v4` trong export; cửa sổ hợp lệ 7 ngày làm việc kể từ xác nhận đơn.
