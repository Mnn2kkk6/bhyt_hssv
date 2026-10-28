# Thiết kế chi tiết — API xem chi tiết một người

| Mục | Nội dung |
|---|---|
| Tên dự án | Hệ thống xử lý dữ liệu BHYT HSSV quy mô 200M |
| Hệ thống | Hệ thống tra cứu kết quả BHYT HSSV |
| Tài liệu | Thiết kế chi tiết (Detail Design) |
| Mã tài liệu | DD-API-HSSV-002 |
| Phiên bản / trạng thái | 1.0 / Draft – chờ review |
| Ngày lập | 2026-10-28 |
| Người lập / review / phê duyệt | Chưa điền |
| Tài liệu cơ bản | `HSSV-BD-001` |
| Chức năng | F-HSSV-003 / API-HSSV-002 |
| Nguồn | Repo `https://github.com/Mnn2kkk6/bhyt_hssv`; quy tắc tuần 1–2; template Detail Design đã cung cấp |

## 1. Tổng quan module

| Mục | Nội dung |
|---|---|
| ID module | DD-API-HSSV-002 |
| Hiện thực cho | F-HSSV-003 / API-HSSV-002; Basic Design `HSSV-BD-001` |
| Mức chi tiết | B – API nghiệp vụ / A – batch quy mô lớn |
| Tóm tắt xử lý | Validate định danh và quyền → resolve snapshot → tìm 1..2 kết quả của người → lấy thống kê gap → mask dữ liệu cá nhân → response. |
| Lớp/package đề xuất | `HssvController` → `HssvQueryService` / `HssvExportService` → `HssvReadRepository` / `ExportJobRepository` |
| Chữ ký phương thức | `PersonDetailResponse getPersonDetail(String soSoBhxh, String snapshotId, Criteria criteria)` |
| Được gọi bởi | API Gateway / batch orchestrator |
| Gọi tới | PostgreSQL read model, Redis cache nếu phù hợp, Object Storage nếu cần |
| Giao dịch | Đọc dùng read-only transaction; thao tác tạo job dùng transaction ngắn; không giữ transaction trong lúc tạo file |
| Xử lý đồng thời | Snapshot chỉ đọc ở trạng thái `PUBLISHED`; keyset/sắp xếp ổn định; export job có trạng thái và khóa idempotency |

## 2. Định nghĩa vào/ra

| Hướng | No | Tên tham số / thuộc tính | Kiểu | Bắt buộc | Mô tả / miền giá trị | Giá trị khi không có | Tham chiếu BD |
| --- | --- | --- | --- | --- | --- | --- | --- |
| IN | 1 | soSoBhxh | String path | Có | Mã số BHXH; phải khớp pattern/length do hệ nguồn quy định, không nhận rỗng. | — | API-HSSV-002 |
| IN | 2 | snapshotId | String query | Không | Snapshot PUBLISHED. | Latest PUBLISHED | BD §6 |
| IN | 3 | criteria | Enum query | Không | STOPPED hoặc GAP_90_PLUS để thu hẹp tiêu chí. | Không lọc | BD §8 |
| OUT | 1 | person | Object | Có | Thông tin cá nhân tối thiểu và đã mask theo role. | — | BD §8 |
| OUT | 2 | results[] | Array | Có | Một người có 1..2 kết quả tiêu chí trong snapshot. | [] không cho phép nếu API trả 200 | BD §8 |
| OUT | 3 | snapshotId/generatedAt | String | Có | Phiên bản dữ liệu và thời điểm publish. | — | BD §8 |

## 3. Xử lý chi tiết

| Bước | Nội dung xử lý | Điều kiện | Khi đúng | Khi sai | SQL / Check / MSG-ID | Ghi chú |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Xác thực caller và quyền `HSSV_RESULT_READ_DETAIL`. | Token/role hợp lệ? | Sang 2. | 401/403; ghi audit. | CHK-01 |  |
| 2 | Validate `soSoBhxh`, `snapshotId`, `criteria`; chuẩn hóa giá trị đầu vào không làm biến đổi định danh hợp lệ. | Input đúng format? | Sang 3. | 400 MSG-E001/E002. | CHK-02 | Không ghi đầy đủ mã BHXH trong log. |
| 3 | Resolve snapshot PUBLISHED. | Snapshot hợp lệ? | Sang 4. | 404 nếu không tồn tại; 409 nếu không publish. | SQL-01 |  |
| 4 | Lấy các dòng kết quả người theo snapshot và tiêu chí (SQL-02). | 0, 1..2 hay >2 dòng? | 1..2 → sang 5. | 0 → 404; >2 → lỗi nhất quán 500/MSG-E900. | SQL-02 | Khóa duy nhất snapshot + criteria + SO_SO_BHXH. |
| 5 | Lấy thông tin tổng hợp thẻ/gián đoạn và map criteria result DTO. | Có trường nhạy cảm? | Mask trường theo role, trả HTTP 200. | Không có quyền đọc PII → trả dữ liệu đã mask hoặc 403 theo policy. | SQL-03, CHK-03 | Không trả địa chỉ/CMND đầy đủ mặc định. |
| 6 | Ghi audit truy vấn chi tiết và trả JSON. | Audit service lỗi? | Trả 200; log WARN; audit retry theo chính sách. | Lỗi audit không làm rớt query đọc trừ khi quy định an ninh yêu cầu fail-closed. | — |  |

## 4. Định nghĩa SQL / truy vấn dữ liệu

Mọi truy vấn dùng bind parameter, không ghép trực tiếp giá trị do client cung cấp vào SQL. Thời gian chờ truy vấn và statement timeout phải cấu hình; khi timeout, hủy truy vấn và trả lỗi có `traceId`.

### SQL-01 — Resolve snapshot

```sql
SELECT snapshot_id, status, published_at
FROM hssv_batch_run
WHERE (:snapshotId IS NULL AND status='PUBLISHED') OR snapshot_id=:snapshotId
ORDER BY published_at DESC LIMIT 2;
```

| Thuộc tính | Giá trị |
|---|---|
| Tham số bind | snapshotId có thể null. |
| Index / partition | PK snapshot_id; published status index |
| Số bản ghi kỳ vọng | 0..1 |
| Có dữ liệu | 1 PUBLISHED → tiếp tục. |
| Không có (NOT FOUND) | 0 → 404; non-PUBLISHED → 409. |
| Nhiều hơn kỳ vọng | >1 → MSG-E900. |
| Lỗi DB / Timeout | DB error/timeout → MSG-E503/E504. |
| Khóa / Timeout | Read-only; timeout 2s. |

### SQL-02 — Đọc kết quả theo định danh

```sql
SELECT snapshot_id, criteria, so_so_bhxh, full_name, birth_date,
       ma_khoi, ma_tinh, ten_tinh, representative_card_id,
       representative_start, representative_end, total_gap_days, gap_count, generated_at
FROM hssv_person_result
WHERE snapshot_id=:snapshotId AND so_so_bhxh=:soSoBhxh
  AND (:criteria IS NULL OR criteria=:criteria)
ORDER BY criteria
LIMIT 3;
```

| Thuộc tính | Giá trị |
|---|---|
| Tham số bind | `:snapshotId`, `:soSoBhxh`, optional criteria. |
| Index / partition | Unique index `(snapshot_id, so_so_bhxh)`; PK `(snapshot,criteria,person)` |
| Số bản ghi kỳ vọng | 0..2 |
| Có dữ liệu | 1..2 → map results. |
| Không có (NOT FOUND) | 0 → 404; not in published result set. |
| Nhiều hơn kỳ vọng | 3 rows means duplicate/business defect → MSG-E900, log ERROR. |
| Lỗi DB / Timeout | DB error/timeout → MSG-E503/E504. |
| Khóa / Timeout | Read-only; timeout 2s. |

### SQL-03 — Đọc tóm tắt gap/timeline metadata

```sql
SELECT MIN(gap_start) AS first_gap_start,
       MAX(gap_end) AS last_gap_end,
       SUM(gap_days) AS total_gap_days,
       COUNT(*) AS gap_count
FROM hssv_gap_period
WHERE snapshot_id=:snapshotId AND so_so_bhxh=:soSoBhxh;
```

| Thuộc tính | Giá trị |
|---|---|
| Tham số bind | snapshotId + soSoBhxh. |
| Index / partition | Index `(snapshot_id, so_so_bhxh, gap_start)` |
| Số bản ghi kỳ vọng | 1 aggregate row |
| Có dữ liệu | Aggregate trả số liệu → kiểm tra với result row. |
| Không có (NOT FOUND) | No gap periods hợp lệ với STOPPED result → trường gap rỗng; với GAP_90_PLUS mà total<90 → MSG-E900. |
| Nhiều hơn kỳ vọng | Aggregate luôn một dòng; không thể >1 row; inconsistency là lỗi DB/SQL. |
| Lỗi DB / Timeout | DB error/timeout → MSG-E503/E504. |
| Khóa / Timeout | Read-only; timeout 2s. |

## 5. Đặc tả kiểm tra đầu vào

| Check-ID | Hạng mục | Loại kiểm tra | Điều kiện lỗi | Mã thông báo | Thứ tự | Dừng ngay? | Tham chiếu BD |
| --- | --- | --- | --- | --- | --- | --- | --- |
| CHK-01 | Authorization | RBAC | Không có quyền detail | MSG-E401/E403 | 1 | Có | BD §6 |
| CHK-02 | soSoBhxh | Format | Rỗng, quá dài hoặc ký tự ngoài miền cho phép | MSG-E001/E002 | 2 | Có | BD §8 |
| CHK-03 | PII fields | Data access | Role không được xem đầy đủ định danh | Mask hoặc MSG-E403 | 3 | Có | BD §10 |

## 6. Xử lý ngoại lệ và log

| Tình huống | Exception/logical error | HTTP | Mã | Mức log | Xử lý |
|---|---|---:|---|---|---|
| Request thiếu/sai định dạng | `ValidationError` | 400 | `MSG-E001/E002/E003/E004` | INFO | Dừng trước truy vấn DB. |
| Không tìm thấy kết quả | `NotFoundError` | 404 | `MSG-I404` | INFO | Không trả thông tin định danh khác. |
| Không có quyền | `AuthorizationError` | 403 | `MSG-E403` | WARN | Ghi audit, không log PII thô. |
| DB timeout | `QueryTimeout` | 503/504 | `MSG-E503/E504` | ERROR | Hủy query, gắn traceId, không trả stacktrace. |
| Lỗi DB khác | `DataAccessError` | 500 | `MSG-E500` | ERROR | Rollback transaction hiện tại; không lộ SQL/PII. |
| Snapshot/report không nhất quán | `DataInconsistencyError` | 500 | `MSG-E900` | ERROR | Không trả dữ liệu một phần; cảnh báo vận hành. |
| Request bị rate limit | `RateLimitError` | 429 | `MSG-E429` | WARN | Trả retry-after nếu khả dụng. |

Log bắt buộc gồm `traceId`, `snapshotId`, endpoint, caller/role, `durationMs`, kết quả và mã lỗi. Không ghi đầy đủ `SO_CMND`, địa chỉ hoặc token. Với tham số `soSoBhxh`, log ở dạng che/hash.

## 7. Đặc tả Unit Test

| UT-ID | Phân loại | Bước DD | Tiền đề/dữ liệu | Đầu vào | Thao tác | Kết quả kỳ vọng | Thực tế | OK/NG | Người test | Ngày test | Bug ID |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| UT-001 | Bình thường | 4-6 | Người thuộc kết quả, một criteria | soSoBhxh hợp lệ | GET person detail | 200, results có một item và PII mask đúng role |  |  |  |  |  |
| UT-002 | Bình thường | 4-6 | Người thuộc cả hai criteria nếu dữ liệu hợp lệ | criteria bỏ trống | GET person detail | 200, tối đa hai result items |  |  |  |  |  |
| UT-003 | Bất thường | 2 | — | soSoBhxh rỗng/invalid | GET person detail | 400 MSG-E001/E002 |  |  |  |  |  |
| UT-004 | Bất thường | 4 | Người tồn tại trong nguồn nhưng không thuộc read model | soSoBhxh không có kết quả | GET person detail | 404 MSG-I404; không lộ metadata nguồn |  |  |  |  |  |
| UT-005 | Bất thường | 4 | Mô phỏng dữ liệu trùng | cùng người có >2 record | GET person detail | 500 MSG-E900, cảnh báo vận hành |  |  |  |  |  |
| UT-006 | Bảo mật | 5 | Caller không có quyền PII | role đọc thông thường | GET person detail | CMND/số thẻ/địa chỉ được che hoặc 403 theo policy |  |  |  |  |  |

## 8. QA / câu hỏi cần xử lý

| No | Ngày hỏi | Mục | Câu hỏi | Người trả lời | Trả lời/giả định đang dùng | Ngày trả lời | Trạng thái | Đã phản ánh? |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 2026-10-28 | 03/05 | Kiểm tra ngưỡng/ngày danh mục | Đã kế thừa quy tắc tuần 1–2; câu hỏi nghiệp vụ chưa phê duyệt được ghi ở Basic Design QA. | BA/PO | Giữ `>=90 ngày`; dùng danh mục gắn snapshot. | 2026-10-28 | Open | Chưa |
