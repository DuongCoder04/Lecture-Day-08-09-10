# Data contract — Lab Day 10

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| Policy Export CSV (`policy_export_dirty.csv`) | `load_raw_csv()` → CSV DictReader + strip | unknown_doc_id, date parse error, missing field | quarantine_records tăng, expectation FAIL |
| Sprint 3 Inject | Thêm dòng vào CSV thủ công / script | Stale content, conflict version, repetitive text | quarantine_records delta, eval before/after |
| ChromaDB (vector store) | `col.upsert(ids=chunk_ids)` | Embedding model fail, collection missing | `embed_prune_removed` trong log |

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| chunk_id | string (prefix `{doc_id}_{seq}_{hash[:16]}`) | Có | Unique — dùng cho upsert embed |
| doc_id | string | Có | Allowlist: policy_refund_v4, sla_p1_2026, it_helpdesk_faq, hr_leave_policy, access_control_sop |
| chunk_text | string (UTF-8) | Có | ≥8 ký tự, không chứa marker stale/content cũ |
| effective_date | date (ISO: YYYY-MM-DD) | Có | ≥ 2026-01-01 (outdated policy bị quarantine) |
| exported_at | datetime (ISO: YYYY-MM-DDTHH:MM:SS) | Có | Đã chuẩn hoá: '/' → '-' |

### Schema quarantine

| Cột | Kiểu | Ghi chú |
|-----|------|---------|
| Tất cả cột raw | string | Giữ nguyên dữ liệu gốc |
| reason | string | Lý do quarantine (xem rule list) |
| effective_date_normalized | string (ISO) | Chỉ có ở rule 3, 12 |

## 3. Quy tắc quarantine vs drop

- **Quarantine**: ghi vào `artifacts/quarantine/quarantine_*.csv` với lý do cụ thể.
- **Không tự động merge lại**: quarantine cần xem xét thủ công, sửa nguồn rồi chạy lại pipeline.
- **Drop**: dữ liệu không hợp lệ nhưng không rõ lý do → ghi log warning.

## 4. Phiên bản & canonical

- Source of truth cho document content và effective_date: CSV export từ hệ thống policy.
- Doc_id là key để phân loại policy (HR, refund, SLA, FAQ, access control).
- Version conflict (10 vs 12 ngày phép năm) được xử lý bằng cleaning rules (quarantine cả hai).
