# Kiến trúc pipeline — Lab Day 10

**Nhóm:** AI-Pipeline-Team  
**Cập nhật:** 2026-04-15

---

## 1. Sơ đồ luồng (bắt buộc có 1 diagram: Mermaid / ASCII)

```mermaid
flowchart LR
	A[Raw export CSV<br/>data/raw/policy_export_*.csv] --> B[Clean rules<br/>transform/cleaning_rules.py]
	B --> C[Validate expectations<br/>quality/expectations.py]
	C -->|halt=pass| D[Embed (ChromaDB)<br/>upsert + prune]
	C -->|halt=fail & --skip-validate| D
	D --> E[Manifest + Logs<br/>artifacts/manifests + artifacts/logs]
	E --> F[Serving for Day 08/09]

	subgraph Monitoring
	M[Freshness check<br/>monitoring/freshness_check.py]
	end
	E --> M
```

> Vẽ thêm: điểm đo **freshness**, chỗ ghi **run_id**, và file **quarantine**.

---

## 2. Ranh giới trách nhiệm

| Thành phần | Input | Output | Owner nhóm |
|------------|-------|--------|--------------|
| Ingest (`etl_pipeline.py`) | `data/raw/*.csv` | dict rows (memory) | Ingestion Owner |
| Transform (`transform/cleaning_rules.py`) | rows | `artifacts/cleaned/*.csv`, `artifacts/quarantine/*.csv` | Cleaning Owner |
| Quality (`quality/expectations.py`) | cleaned rows | suite results (halt/warn) | Quality Owner |
| Embed (ChromaDB) | cleaned csv | collection `day10_kb` | Embed Owner |
| Monitor (`monitoring/freshness_check.py`) | manifest | PASS/WARN/FAIL | Monitoring Owner |

---

## 3. Idempotency & rerun

Idempotent embed: `etl_pipeline.py` dùng `chunk_id` ổn định (SHA-256 từ `doc_id|chunk_text|seq`) và `col.upsert(...)`. Trước khi upsert, đọc các `ids` hiện có và `delete` những id không còn trong lần clean hiện tại (prune) để tránh vector cũ lẫn vào top-k. Rerun cùng input không tạo duplicate.

---

## 4. Liên hệ Day 09

Collection `day10_kb` lưu embedding cho 5 tài liệu trong `data/docs/`. Day 09 có thể trỏ đọc cùng Chroma DB path/collection để dùng snapshot mới nhất sau mỗi lần publish.

---

## 5. Rủi ro đã biết

- Export time không tươi (freshness FAIL) có thể gây trả lời lỗi thời.
- Input có BOM/control-character → lỗi embedding/eval nếu không quarantine.
- Ngưỡng lockout cấu hình sai (IT) gây trả lời không chuẩn; đã thêm guardrail.
