# Quality report — Lab Day 10 (nhóm)

**run_id:** sprint3-before → sprint3-after-inject  
**Ngày:** 2026-04-15

---

## 1. Tóm tắt số liệu

| Chỉ số | Trước (sprint3-before) | Sau (sprint3-after-inject) | Ghi chú |
|--------|-------------------------|----------------------------|---------|
| raw_records | 10 | 13 | Theo manifest_*.json |
| cleaned_records | 5 | 7 | Inject thêm bản ghi và bỏ fix refund |
| quarantine_records | 5 | 6 | Quarantine tăng do BOM + lockout>10 |
| Expectation halt? | Không | Có (refund 14d) | Dùng --skip-validate để tiếp tục embed |

Nguồn: [manifest_sprint3-before.json](../artifacts/manifests/manifest_sprint3-before.json), [manifest_sprint3-after-inject.json](../artifacts/manifests/manifest_sprint3-after-inject.json)

---

## 2. Before / after retrieval (bắt buộc)

Kết quả: [artifacts/eval/before_eval.csv](../artifacts/eval/before_eval.csv), [artifacts/eval/after_inject_bad.csv](../artifacts/eval/after_inject_bad.csv)

- Câu then chốt `q_refund_window` (forbidden = "14 ngày làm việc"):
  - Trước: contains_expected=yes, hits_forbidden=no, top1_doc_id=policy_refund_v4
  - Sau (inject + no-fix): contains_expected=yes, hits_forbidden=yes, top1_doc_id=policy_refund_v4

- Merit `q_leave_version` (HR versioning):
  - Trước: contains_expected=yes, hits_forbidden=no, top1_doc_expected=yes
  - Sau: contains_expected=yes, hits_forbidden=no, top1_doc_expected=yes (baseline quarantine bản HR cũ)

---

## 3. Freshness & monitor

- Freshness check: FAIL do `latest_exported_at=2026-04-10T08:00:00` > SLA 24h (age ~121h). Xem trường `freshness_check` trong log run.
- SLA chọn: 24 giờ cho dữ liệu chính sách nội bộ; WARN khi 12–24h, FAIL khi >24h (áp dụng trong `monitoring/freshness_check.py`).

---

## 4. Corruption inject (Sprint 3)

- Kiểu hỏng dữ liệu:
  - Thêm chunk BOM/control-char → quarantine `control_characters_or_bom` (R1)
  - IT lockout 12 lần → quarantine `it_lockout_threshold_suspect` (R3)
  - Refund chứa "14 ngày làm việc" không gắn marker legacy → expectation `refund_no_stale_14d_window` FAIL khi tắt fix
- Chứng minh: [quarantine_sprint3-after-inject.csv](../artifacts/quarantine/quarantine_sprint3-after-inject.csv), [run_sprint3-after-inject.log](../artifacts/logs/run_sprint3-after-inject.log)

---

## 5. Hạn chế & việc chưa làm

- Chưa thêm cột `scenario` gộp 2 pha vào cùng CSV (đang dùng 2 file riêng).
- Freshness hiện hard-coded SLA=24h qua env; chưa có cảnh báo tự động.
