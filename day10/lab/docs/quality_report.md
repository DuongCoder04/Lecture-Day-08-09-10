# Quality report — Lab Day 10

**run_id:** final-full  
**Ngày:** 2026-06-10

---

## 1. Tóm tắt số liệu

| Chỉ số | Baseline (before fix) | Sau fix (final) | Ghi chú |
|--------|----------------------|-----------------|---------|
| raw_records | 247 | 247 | Không đổi |
| cleaned_records | ~37 (ước lượng) | 28 | Giảm do rules 7, 11, 12, 13 |
| quarantine_records | ~210 (ước lượng) | 219 | Tăng do bắt thêm HR stale/conflict + outdated |
| Expectation halt? | CÓ (E6: hr_leave_no_stale_10d_annual) | KHÔNG | Tất cả expectation PASS |

## 2. Before / after retrieval

> Xem `artifacts/eval/before_after_eval.csv`

**Câu hỏi then chốt:** HR version conflict (`q_leave_version`)

**Before (baseline code, thiếu rules 7, 11):**
- 9 hr_leave_policy records lọt vào cleaned
- 2 records chứa "10 ngày phép năm" (stale content) → E6 FAIL, pipeline HALT

**After (sau khi thêm rules 7, 11, 12, 13):**
- 5 hr_leave_policy records trong cleaned (chỉ giữ các policy hợp lệ: 15 ngày, 18 ngày, nghỉ ốm, remote work)
- E6 (hr_leave_no_stale_10d_annual) PASS
- E7 (hr_leave_no_conflict_12d_annual) PASS

## 3. Freshness & monitor

- `freshness_check=FAIL` — nguyên nhân: `exported_at` trong sample data là April 2026, trong khi pipeline chạy June 2026 (age ≈ 1448 giờ, vượt SLA 24h).
- **Giải thích:** Data mẫu cố tình dùng timestamp cũ. Trong production, SLA sẽ được tính từ thời điểm export thực tế.
- Có thể set `FRESHNESS_SLA_HOURS=2000` để PASS nếu muốn.

## 4. Corruption inject (Sprint 3)

Đã inject 6 record vào `policy_export_injected.csv`:

| # | Loại corruption | Target rule | Kết quả |
|---|----------------|-------------|---------|
| 1 | "10 ngày phép năm" với nội dung mới (không trùng) | Rule 7 (stale_hr_policy_content) | Quarantine |
| 2 | "12 ngày phép năm" với nội dung conflict | Rule 11 (conflict_hr_policy_version) | Quarantine |
| 3 | policy_refund_v4 với effective_date 2025 | Rule 12 (outdated_policy) | Quarantine |
| 4 | "Nội dung không rõ ràng:" prefix trong SLA | Rule 9 (unclear_content) | Quarantine |
| 5 | Câu lặp lại 4 lần trong it_helpdesk_faq | Rule 13 (repetitive_chunk_text) | Quarantine |
| 6 | Record hợp lệ (access_control_sop mới) | Không rule nào | Quarantine: 5/5 dirty → quarantined |

**Delta:** `cleaned_records: 28 → 29` (1 record valid mới), `quarantine_records: 219 → 224` (+5 dirty bị quarantine)

## 5. Hạn chế & việc chưa làm

- Chưa tích hợp Great Expectations (dùng custom ExpectationResult đơn giản).
- Chưa có LLM-judge cho eval (dùng keyword match).
- `freshness_check` luôn FAIL trên data mẫu do timestamp cũ — cần cập nhật SLA phù hợp.
- Allowlist chưa mở rộng (data_privacy_guideline, security_policy có content hợp lệ nhưng chưa được thêm).
