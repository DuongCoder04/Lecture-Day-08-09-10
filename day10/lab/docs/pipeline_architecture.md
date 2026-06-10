# Kiến trúc pipeline — Lab Day 10

## 1. Sơ đồ luồng

```
                        ┌─────────────────────────────────────────────────────────────┐
                        │                   policy_export_dirty.csv                   │
                        │                (raw/CSV — 247 records)                      │
                        └─────────────────────┬───────────────────────────────────────┘
                                              │
                                              ▼
                        ┌─────────────────────────────────────────────────────────────┐
                        │                   INGEST (load_raw_csv)                     │
                        │  • Đọc CSV, strip whitespace                                │
                        │  • Output: list[Dict[str, str]]                             │
                        └─────────────────────┬───────────────────────────────────────┘
                                              │
                                              ▼
            ┌─────────────────────────────────────────────────────────────────────────┐
            │                   TRANSFORM (clean_rows)                                │
            │                                                                         │
            │  1) unknown_doc_id ───── quarantine (109)                               │
            │  2) normalize effective_date                                            │
            │  3) stale HR effective_date < 2026 ── quarantine (22)                   │
            │  4) outdated_policy < 2026 ────────── quarantine (57)                   │
            │  5) missing/whitespace text ───────── quarantine (4)                    │
            │  6) unclear_content ──────────────── quarantine (3)                     │
            │  7) duplicate chunk_text ──────────── quarantine (15)                   │
            │  8) stale_hr_policy_content ──────── quarantine (2)                     │
            │  9) conflict_hr_policy_version ──── quarantine (1)                     │
            │ 10) repetitive_chunk_text ────────── quarantine (0)                    │
            │ 11) fix refund window 14→7 ngày                                        │
            │ 12) normalize exported_at '/' → '-'                                    │
            │                                                                         │
            │  Output: cleaned (28) + quarantine (219)                                │
            └──────────┬──────────────────────────────────┬───────────────────────────┘
                       │                                  │
                       ▼                                  ▼
            ┌──────────────────┐              ┌───────────────────────┐
            │   cleaned.csv    │              │   quarantine.csv      │
            │  (artifacts/)    │              │   (artifacts/)        │
            └────────┬─────────┘              └───────────────────────┘
                     │
                     ▼
            ┌─────────────────────────────────────────────────────────┐
            │              VALIDATE (run_expectations)                │
            │  E1: min_one_row (halt)                                 │
            │  E2: no_empty_doc_id (halt)                              │
            │  E3: refund_no_stale_14d_window (halt)                   │
            │  E4: chunk_min_length_8 (warn)                           │
            │  E5: effective_date_iso_yyyy_mm_dd (halt)                │
            │  E6: hr_leave_no_stale_10d_annual (halt)                 │
            │  E7: hr_leave_no_conflict_12d_annual (halt)              │
            │  E8: exported_at_iso_format (warn)                       │
            └──────────┬──────────────────────────────────────────────┘
                       │
                       ▼
            ┌─────────────────────────────────────────────────────────┐
            │                 EMBED (ChromaDB)                        │
            │  • Upsert theo chunk_id                                 │
            │  • Prune vector cũ không còn trong cleaned              │
            │  • Collection: day10_kb                                 │
            └──────────┬──────────────────────────────────────────────┘
                       │
                       ▼
            ┌─────────────────────────────────────────────────────────┐
            │              MONITOR (freshness_check)                  │
            │  • Đọc manifest, so sánh latest_exported_at vs now      │
            │  • SLA mặc định: 24 giờ                                 │
            └─────────────────────────────────────────────────────────┘
```

## 2. Ranh giới trách nhiệm

| Thành phần | Input | Output | Chức năng |
|------------|-------|--------|-----------|
| Ingest | `data/raw/policy_export_dirty.csv` | `list[Dict[str,str]]` | Đọc CSV, chuẩn hoá whitespace, parse date |
| Transform | Raw rows (list[Dict]) | `(cleaned, quarantine)` | 10 cleaning rules: quarantine dữ liệu lỗi, fix nội dung, chuẩn hoá |
| Quality | Cleaned rows | `(ExpectationResult[], halt)` | 8 expectations: kiểm tra số lượng, format, nội dung |
| Embed | `cleaned_*.csv` | ChromaDB upsert | Vector hoá chunk_text, prune vector cũ |
| Monitor | `manifest_*.json` | PASS/WARN/FAIL | Freshness SLA: so sánh exported_at hiện tại với thời gian chạy pipeline |

## 3. Idempotency & rerun

- Clean & quarantine: mỗi run tạo file mới theo `run_id`, không ghi đè.
- Embed: upsert theo `chunk_id` (SHA256 hash). Rerun 2 lần không phình tài nguyên.
- Prune: xoá vector id không còn trong cleaned run hiện tại (tránh vector lạc hậu).
- Manifest: mỗi run ghi 1 file JSON riêng, phục vụ freshness check.

## 4. Liên hệ Day 09

- Pipeline Day 10 tạo cleaned corpus từ export CSV, khác với Day 09 dùng `data/docs/` gốc.
- Output vector store (`chroma_db`) sẵn sàng cho retrieval ở Day 08/09.

## 5. Rủi ro đã biết

- `freshness_check=FAIL` trên data mẫu do `exported_at` cũ (April 2026) so với thời gian chạy thực (June 2026). Giảm `FRESHNESS_SLA_HOURS` hoặc cập nhật timestamp để PASS.
- ChromaDB không bắt buộc (thiếu thư viện → log WARNING, pipeline không HALT).
