# Ma trận truy vết — HSSV 200M

| Requirement ID | Yêu cầu | Basic Design | API/Batch ID | Detail Design | Kiểm thử / nghiệm thu |
|---|---|---|---|---|---|
| REQ-HSSV-001 | Xử lý 200M dòng thẻ BHYT, không nạp toàn bộ vào RAM | §3–4, §7, §10 | `F-HSSV-001`, `BAT-HSSV-001` | `DD-BAT-HSSV-001` | `UT-B001..UT-B008`; benchmark 25M/100M/200M; kiểm tra RAM/spill/throughput |
| REQ-HSSV-002 | Xác định HSSV dừng tham gia | §2, §4 | `F-HSSV-001`, criteria `STOPPED` | `DD-BAT-HSSV-001` | Có HS/SV ở 10/2024; không có thẻ hợp lệ bất kỳ khối giao 08/2026 |
| REQ-HSSV-003 | Xác định gián đoạn cộng dồn | §2, §4 | `F-HSSV-001`, criteria `GAP_90_PLUS` | `DD-BAT-HSSV-001` | T01: 90 ngày đạt; T02: 89 ngày không đạt; overlap/adjacent/nested/canceled tests |
| REQ-HSSV-004 | Tra cứu danh sách lọc tỉnh/khối/tiêu chí có phân trang | §5–8 | `API-HSSV-001` | `DD-API-HSSV-001` | `UT-001..UT-007`; page/pageSize invalid, no result, ACL, load test |
| REQ-HSSV-005 | Xem chi tiết một người | §6, §8 | `API-HSSV-002` | `DD-API-HSSV-002` | `UT-001..UT-006`; not found, >2 result anomaly, PII masking |
| REQ-HSSV-006 | Xem timeline thẻ và các khoảng gián đoạn | §6, §8 | `API-HSSV-003` | `DD-API-HSSV-003` | `UT-001..UT-006`; chronology, exact day gap, total reconciliation |
| REQ-HSSV-007 | Báo cáo theo 63 tỉnh/TP | §6–8 | `API-HSSV-004` | `DD-API-HSSV-004` | 63 unique rows; zero-count provinces; catalog mismatch returns `MSG-E900` |
| REQ-HSSV-008 | Phân bố thời gian gián đoạn | §6, §8 | `API-HSSV-005` | `DD-API-HSSV-005` | 4 buckets × HS/SV; zero-filled buckets; denominator=0; summary consistency |
| REQ-HSSV-009 | Xuất báo cáo CSV/XLSX | §6, §8, §10 | `API-HSSV-006` | `DD-API-HSSV-006` | Async 202, allowlist, maxRows, idempotency, storage failure, XLSX row limit |
| REQ-HSSV-010 | Theo dõi export job và nhận URL tải | §6, §8 | `API-HSSV-007` (phụ trợ) | `DD-API-HSSV-007` | Job states, owner/RBAC, signed URL TTL, expired file |
| REQ-HSSV-011 | Quyền truy cập, audit, che PII | §6, §10 | Tất cả API | Tất cả API DD | 401/403, PII masking, audit logs, signed URL safety |
| REQ-HSSV-012 | Snapshot publish nguyên tử, retry không nhân đôi | §4, §7, §10 | `BAT-HSSV-001` | `DD-BAT-HSSV-001` | Fail giữa batch; active pointer cũ vẫn dùng; replay idempotent; reconciliation |

## Các quy tắc cần nhất quán

- `TRANG_THAI=1` là điều kiện thẻ hợp lệ; thẻ bị hủy không bao phủ ngày.
- Để xét giao tháng: `TU_NGAY <= month_end` và `DEN_NGAY >= month_start`; ngày đầu và cuối đều inclusive.
- Dừng tham gia = có HS/SV giao tháng 10/2024 trừ người có bất kỳ thẻ hợp lệ giao tháng 08/2026.
- Gián đoạn = có HS/SV giao tháng 10/2024 giao với người có bất kỳ thẻ hợp lệ giao tháng 07/2026; dùng cửa sổ cá nhân; gộp khoảng chồng lấn/nối tiếp; `gapDays = nextStart - previousEnd - 1`; tổng `>=90` theo rule tuần 2.
- Thẻ gốc và thẻ đại diện phải có tie-break ổn định giống `solve.py`/`solve_y2.py`.
- Chỉ snapshot `PUBLISHED` được phục vụ qua API; lỗi batch không làm mất bản đang active.
- Mọi SQL truy vấn dùng bind parameter; API không scan raw 200M dòng.
