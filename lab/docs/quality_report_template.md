# Quality report — Lab Day 10 (nhóm)

**run_id:** sprint3-before → sprint3-after-inject  
**Ngày:** 2026-04-15

---

## 1. Tóm tắt số liệu

| Chỉ số | Trước (sprint3-before) | Sau (sprint3-after-inject) | Ghi chú |
|--------|-------------------------|----------------------------|---------|
| raw_records | 10 | 13 | Theo manifest_*.json |
| cleaned_records | 5 | 7 | Inject thêm bản ghi và tắt fix refund |
| quarantine_records | 5 | 6 | BOM/control + lockout>10 thêm vào quarantine |
| Expectation halt? | Không | Có (refund 14d) | Dùng `--skip-validate` để tiếp tục embed |

Nguồn: [manifest_sprint3-before.json](../artifacts/manifests/manifest_sprint3-before.json), [manifest_sprint3-after-inject.json](../artifacts/manifests/manifest_sprint3-after-inject.json)

---

## 2. Before / after retrieval (bắt buộc)

Kết quả: [artifacts/eval/before_eval.csv](../artifacts/eval/before_eval.csv), [artifacts/eval/after_inject_bad.csv](../artifacts/eval/after_inject_bad.csv)

**Câu hỏi then chốt:** refund window (`q_refund_window`)

- Trước (trích dòng CSV):
	- question_id=q_refund_window, top1_doc_id=policy_refund_v4, contains_expected=yes, hits_forbidden=no
	- top1_preview: “Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng.”
- Sau (inject + no-fix, trích dòng CSV):
	- question_id=q_refund_window, top1_doc_id=policy_refund_v4, contains_expected=yes, hits_forbidden=no
	- top1_preview: “Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng.”

Ghi chú: Ở snapshot hiện tại top-k chưa chứa chunk “14 ngày” trong kết quả, nhưng expectation đã FAIL (halt) ở bước validate khi tắt fix — xem mục 3 & manifest/log.

**Merit (khuyến nghị):** versioning HR — `q_leave_version`

- Trước: contains_expected=yes, hits_forbidden=no, top1_doc_expected=yes
- Sau: contains_expected=yes, hits_forbidden=no, top1_doc_expected=yes (bản HR 2025 bị quarantine theo rule baseline)

---

## 3. Freshness & monitor

- Freshness check: FAIL do `latest_exported_at=2026-04-10T08:00:00` > SLA 24h.
- SLA nhóm: PASS ≤ 12h, WARN 12–24h, FAIL > 24h. Với dữ liệu mẫu, FAIL là hợp lý; ghi rõ trong runbook cách xử lý (banner + lên lịch cập nhật nguồn).

Chạy kiểm tra:
```
python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_sprint3-before.json
python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_sprint3-after-inject.json
```

---

## 4. Corruption inject (Sprint 3)

- Dữ liệu inject: `data/raw/policy_export_inject_sprint2.csv` gồm:
	- Refund có câu “14 ngày làm việc” (không gắn marker) → để expectation refund fail khi tắt fix.
	- IT lockout 12 lần → quarantine `it_lockout_threshold_suspect`.
	- 1 dòng BOM/control → quarantine `control_characters_or_bom`.
- Chạy: `python etl_pipeline.py run --run-id sprint3-after-inject --raw data/raw/policy_export_inject_sprint2.csv --no-refund-fix --skip-validate`
- Bằng chứng quarantine: [quarantine_sprint3-after-inject.csv](../artifacts/quarantine/quarantine_sprint3-after-inject.csv)

---

## 5. Hạn chế & việc chưa làm

- Top-k hiện tại chưa bộc lộ `hits_forbidden=yes` cho `q_refund_window` dù expectation đã FAIL; có thể tăng `top-k`, thay đổi seed, hoặc chèn thêm mẫu để thể hiện rõ degradation trong eval.
- Chưa gộp before/after vào 1 CSV có cột `scenario` (đang dùng 2 file riêng).
- Freshness cảnh báo thủ công qua CLI; chưa tích hợp cảnh báo tự động.
