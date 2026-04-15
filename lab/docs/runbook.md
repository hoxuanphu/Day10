# Runbook — Lab Day 10 (incident tối giản)

---

## Symptom

- Agent trả lời sai: "14 ngày làm việc" thay vì 7 ngày.
- HR phép năm trả lời 10 ngày (bản cũ) thay vì 12 ngày (2026).

---

## Detection

- Expectation suite: `refund_no_stale_14d_window` (halt) FAIL nếu còn 14d.
- Eval retrieval: CSV `hits_forbidden=yes` cho `q_refund_window` khi inject.
- Freshness: `freshness_check=FAIL` khi `latest_exported_at` > SLA 24h.

---

## Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Kiểm tra `artifacts/manifests/*.json` | Đối chiếu `raw_records/cleaned/quarantine`, `no_refund_fix` |
| 2 | Mở `artifacts/quarantine/*.csv` | Xem cột `reason` (BOM, legacy, stale HR, duplicate, …) |
| 3 | Chạy `python eval_retrieval.py --out artifacts/eval/current_eval.csv` | Xem `hits_forbidden` của `q_refund_window` |

---

## Mitigation

- Rerun pipeline chuẩn (bật fix refund) để xoá vector xấu và upsert snapshot mới:
```powershell
python etl_pipeline.py run --run-id hotfix-refund
```
- Nếu nguồn HR cũ: quarantine bằng rule hiện có; hoặc cập nhật nguồn rồi rerun.
- Banner tạm: thông báo người dùng về dữ liệu đang cập nhật nếu SLA vượt ngưỡng.

---

## Prevention

- Duy trì các expectation halt: 14d refund, marker legacy, HR 10 ngày, ISO date.
- Alert định kỳ theo SLA freshness; theo dõi `quarantine_records` và nguyên nhân.
- Peer review 3 câu hỏi (Phần E): `q_refund_window`, `q_p1_sla`, `q_leave_version` — theo dõi trước/sau trong CSV eval.

---

## Freshness command & tiêu chí PASS/WARN/FAIL

Chạy:
```powershell
python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_<run-id>.json
```
- PASS: `age_hours <= 12` (ví dụ đội chọn WARN/FAIL như dưới)
- WARN: `12 < age_hours <= 24`
- FAIL: `age_hours > 24`
