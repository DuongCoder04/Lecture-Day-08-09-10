# Runbook — Lab Day 10 (incident tối giản)

---

## Symptom

| Symptom | Ví dụ |
|---------|-------|
| Pipeline exit code ≠ 0 | `etl_pipeline.py run` HALT với expectation FAIL |
| Freshness FAIL | Log: `freshness_check=FAIL reason=freshness_sla_exceeded` |
| Response sai (chatbot) | Trả lời "14 ngày" thay vì "7 ngày" hoặc "10 ngày" thay vì "12 ngày" |
| Quarantine bất thường | `quarantine_records` tăng đột biến (thường > 200) |

---

## Detection

| Metric | Nguồn | Threshold |
|--------|-------|-----------|
| expectation[xxx] FAIL | Pipeline log | Bất kỳ expectation halt nào FAIL |
| freshness_check | Manifest / log | `age_hours > sla_hours` (default 24h) |
| quarantine_records | Pipeline log | > 220 bất thường |
| eval `hits_forbidden` | `eval_retrieval.py` / grading | `hits_forbidden=true` |

---

## Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Kiểm tra `artifacts/manifests/*.json` | Xác nhận run_id, raw_records, cleaned_records |
| 2 | Mở `artifacts/quarantine/*.csv` | Xem lý do quarantine theo cột reason |
| 3 | `grep expectation\[...\] FAIL` trong log | Xác định expectation nào fail và số violations |
| 4 | So sánh `before_after_eval.csv` | Xem delta giữa run clean và inject |
| 5 | Chạy `python etl_pipeline.py run` với `--skip-validate` | Debug expectation mà không HALT pipeline |

---

## Mitigation

| Tình huống | Cách xử lý |
|------------|-------------|
| Expectation HALT do stale content | Thêm/sửa cleaning rule tương ứng, chạy lại pipeline |
| Freshness FAIL | Giảm `FRESHNESS_SLA_HOURS` hoặc cập nhật exported_at trong raw CSV |
| Quarantine quá nhiều | Kiểm tra allowlist, kiểm tra điều kiện date trong rule |
| ChromaDB lỗi | `pip install -r requirements.txt` (cần sentence-transformers) |

---

## Prevention

| Biện pháp | Mô tả |
|-----------|-------|
| Thêm expectation | Phát hiện sớm stale content, missing field, format sai |
| Freshness alert | Cấu hình `FRESHNESS_SLA_HOURS` phù hợp với batch schedule |
| Cleaning rules versioning | Không hard-code năm, đọc từ contract/env |
| Quality report mỗi run | Ghi lại cleaned_records, quarantine_records, expectations |
| Inject test định kỳ | Chạy Sprint 3 injection để kiểm tra rule có hoạt động không |
