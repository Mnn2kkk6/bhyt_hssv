# Thiết kế chi tiết — API tạo job xuất báo cáo Excel/CSV

| Mục | Nội dung |
|---|---|
| Tên dự án | Hệ thống xử lý dữ liệu BHYT HSSV quy mô 200M |
| Hệ thống | Hệ thống tra cứu kết quả BHYT HSSV |
| Tài liệu | Thiết kế chi tiết (Detail Design) |
| Mã tài liệu | DD-API-HSSV-006 |
| Phiên bản / trạng thái | 1.0 / Draft – chờ review |
| Ngày lập | 2026-10-28 |
| Người lập / review / phê duyệt | Chưa điền |
| Tài liệu cơ bản | `HSSV-BD-001` |
| Chức năng | F-HSSV-007 / API-HSSV-006 |
| Nguồn | Repo `https://github.com/Mnn2kkk6/bhyt_hssv`; quy tắc tuần 1–2; template Detail Design đã cung cấp |

## 1. Tổng quan module

| Mục | Nội dung |
|---|---|
| ID module | DD-API-HSSV-006 |
| Hiện thực cho | F-HSSV-007 / API-HSSV-006; Basic Design `HSSV-BD-001` |
| Mức chi tiết | B – API nghiệp vụ / A – batch quy mô lớn |
| Tóm tắt xử lý | Validate authorization, filters and file format → resolve published snapshot → deduplicate by idempotency key → create QUEUED job → worker streams result to CSV/XLSX and object storage. |
| Lớp/package đề xuất | `HssvController` → `HssvQueryService` / `HssvExportService` → `HssvReadRepository` / `ExportJobRepository` |
| Chữ ký phương thức | `ExportJobAcceptedResponse createExportJob(ExportRequest request, AuthenticatedCaller caller)` |
| Được gọi bởi | API Gateway / batch orchestrator |
| Gọi tới | PostgreSQL read model, Redis cache nếu phù hợp, Object Storage nếu cần |
| Giao dịch | Đọc dùng read-only transaction; thao tác tạo job dùng transaction ngắn; không giữ transaction trong lúc tạo file |
| Xử lý đồng thời | Snapshot chỉ đọc ở trạng thái `PUBLISHED`; keyset/sắp xếp ổn định; export job có trạng thái và khóa idempotency |

## 2. Định nghĩa vào/ra

| Hướng | No | Tên tham số / thuộc tính | Kiểu | Bắt buộc | Mô tả / miền giá trị | Giá trị khi không có | Tham chiếu BD |
| --- | --- | --- | --- | --- | --- | --- | --- |
| IN | 1 | format | Enum | Có | CSV hoặc XLSX (không cho format khác). | — | API-HSSV-006 |
| IN | 2 | criteria/provinceCode/block | Enum/String | Không | Filter giống API-HSSV-001. | Không lọc | API-HSSV-006 |
| IN | 3 | snapshotId | String | Không | Snapshot PUBLISHED. | Latest PUBLISHED | API-HSSV-006 |
| IN | 4 | columns[] | Array<String> | Không | Allowlist các field được export; trường PII chỉ được export theo quyền. | Default safe columns | API-HSSV-006 |
| IN | 5 | maxRows | Integer | Không | Tối đa cấu hình hệ thống; user không được nâng trên system cap. | System default | API-HSSV-006 |
| OUT | 1 | jobId/status/statusUrl | String/Enum | Có | Job xử lý bất đồng bộ. | — | BD §8 |
| OUT | 2 | createdAt/estimatedRows? | Timestamp/Long | Có/Không | Thời điểm tạo; estimate nếu có sẵn. | — | BD §8 |

## 3. Xử lý chi tiết

| Bước | Nội dung xử lý | Điều kiện | Khi đúng | Khi sai | SQL / Check / MSG-ID | Ghi chú |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Authenticate/authorize token và quyền `HSSV_REPORT_EXPORT`; áp dụng rate limit theo caller. | Có quyền? | Sang bước 2. | 401/403/429. | CHK-01 |  |
| 2 | Validate body, format, filters, snapshotId, columns allowlist và maxRows. | Body hợp lệ? | Sang bước 3. | 400 MSG-E001/E002/E004 hoặc 413 MSG-E413. | CHK-02, CHK-03 | Không cho phép client tự đặt object key hay path file. |
| 3 | Resolve snapshot PUBLISHED; tính idempotency key từ caller + snapshot + canonical request nếu client gửi key. | Snapshot hợp lệ? | Sang 4. | 404/409. | SQL-01, MSG-I404/MSG-E409 |  |
| 4 | Kiểm tra job tương đương đang QUEUED/RUNNING hoặc key đã thành công. | Đã tồn tại job hợp lệ? | Trả jobId hiện có nếu idempotency key trùng. | Nếu không có thì tiếp tục insert. | SQL-01 |  |
| 5 | Insert `hssv_export_job` trạng thái QUEUED trong transaction ngắn; enqueue job sau commit. | Insert thành công? | Trả 202 Accepted và statusUrl. | Lỗi insert/enqueue: nếu insert rồi queue fail, set RETRY/FAILED có kiểm soát; không tạo job mồ côi. | SQL-02, MSG-E503 | Outbox pattern khuyến nghị để tránh DB commit nhưng queue mất. |
| 6 | Worker đọc job, chuyển RUNNING, stream các hàng theo filter và thứ tự ổn định; không nạp toàn bộ export vào RAM. | Số dòng <= maxRows? | Ghi CSV/XLSX theo streaming/chunk, cập nhật progress. | Vượt giới hạn → FAILED MSG-E413 hoặc chia file theo policy; không tự xuất vượt giới hạn. | SQL-03, CHK-04 | XLSX có giới hạn hàng của định dạng; nếu vượt phải chia nhiều file hoặc yêu cầu CSV. |
| 7 | Upload artifact lên Object Storage, tạo signed URL có TTL, cập nhật SUCCEEDED/rowCount/expiry. | Upload cập nhật thành công? | Status API trả link tải. | Lỗi storage → retry; quá số lần FAILED/MSG-E503. | SQL-03 | Link không public, không ghi PII trong object name. |
| 8 | Trả HTTP 202 ngay sau khi job được ghi nhận; không đợi file hoàn thành. | Job được tạo? | Body gồm jobId, QUEUED, statusUrl. | Nếu không thể ghi job → trả 503/500, không giả báo đã tạo. | — |  |

## 4. Định nghĩa SQL / truy vấn dữ liệu

Mọi truy vấn dùng bind parameter, không ghép trực tiếp giá trị do client cung cấp vào SQL. Thời gian chờ truy vấn và statement timeout phải cấu hình; khi timeout, hủy truy vấn và trả lỗi có `traceId`.

### SQL-01 — Tìm job trùng idempotency key

```sql
SELECT job_id, status, created_at, expires_at
FROM hssv_export_job
WHERE caller_id=:callerId AND idempotency_key=:idempotencyKey
ORDER BY created_at DESC LIMIT 2;
```

| Thuộc tính | Giá trị |
|---|---|
| Tham số bind | callerId từ identity đã xác thực, idempotencyKey chuẩn hóa. |
| Index / partition | Unique `(caller_id,idempotency_key)` |
| Số bản ghi kỳ vọng | 0..1 |
| Có dữ liệu | 1 job → trả job đã có nếu còn hợp lệ. |
| Không có (NOT FOUND) | 0 → tạo job mới. |
| Nhiều hơn kỳ vọng | >1 → MSG-E900. |
| Lỗi DB / Timeout | DB error/timeout → MSG-E503/E504. |
| Khóa / Timeout | Read-only 2s. |

### SQL-02 — Tạo job xuất

```sql
INSERT INTO hssv_export_job
(job_id, caller_id, snapshot_id, status, format, filter_json, column_set,
 max_rows, idempotency_key, created_at, expires_at)
VALUES (:jobId, :callerId, :snapshotId, 'QUEUED', :format, :filterJson,
        :columnSet, :maxRows, :idempotencyKey, now(), :expiresAt)
ON CONFLICT (caller_id, idempotency_key) DO NOTHING
RETURNING job_id, status, created_at;
```

| Thuộc tính | Giá trị |
|---|---|
| Tham số bind | callerId, jobId server-generated, validated filter/columns, expiry configuration. |
| Index / partition | Unique idempotency index |
| Số bản ghi kỳ vọng | 0..1 RETURNING row |
| Có dữ liệu | 1 row → commit and enqueue. |
| Không có (NOT FOUND) | 0 rows due conflict → re-read and return original job. |
| Nhiều hơn kỳ vọng | Unexpected >1 → impossible; fail MSG-E900. |
| Lỗi DB / Timeout | DB error/timeout → MSG-E503/E504; queue failure via outbox/retry. |
| Khóa / Timeout | Transaction ≤3s; don't keep open for export. |

### SQL-03 — Worker transition/update status

```sql
UPDATE hssv_export_job
SET status=:newStatus, progress_percent=:progress,
    row_count=:rowCount, object_key=:objectKey,
    error_code=:errorCode, updated_at=now()
WHERE job_id=:jobId AND status=:expectedStatus
RETURNING job_id, status;
```

| Thuộc tính | Giá trị |
|---|---|
| Tham số bind | jobId and expected state ensure legal state transition. |
| Index / partition | PK job_id; optional status/created_at index |
| Số bản ghi kỳ vọng | Exactly 1 row |
| Có dữ liệu | 1 row → success. |
| Không có (NOT FOUND) | 0 rows means stale worker/state conflict; reread job, do not overwrite terminal status. |
| Nhiều hơn kỳ vọng | >1 impossible; MSG-E900. |
| Lỗi DB / Timeout | DB/storage error retry bounded; timeout → MSG-E503/E504. |
| Khóa / Timeout | Atomic compare-and-set transition; timeout 3s. |

## 5. Đặc tả kiểm tra đầu vào

| Check-ID | Hạng mục | Loại kiểm tra | Điều kiện lỗi | Mã thông báo | Thứ tự | Dừng ngay? | Tham chiếu BD |
| --- | --- | --- | --- | --- | --- | --- | --- |
| CHK-01 | Authorization/rate limit | RBAC | Thiếu quyền export hoặc quá tần suất cho phép | MSG-E401/E403/MSG-E429 | 1 | Có | BD §6 |
| CHK-02 | format | Enum | Khác CSV/XLSX | MSG-E002 | 2 | Có | BD §8 |
| CHK-03 | filter/columns | Allowlist | Cột không được phép, filter sai, PII không đủ quyền | MSG-E004/MSG-E403 | 3 | Có | BD §10 |
| CHK-04 | maxRows/file capability | Limit | maxRows lớn hơn system cap hoặc XLSX vượt giới hạn định dạng | MSG-E413 | 4 | Có | BD §10 |

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
| UT-001 | Bình thường | 2-5,8 | Valid request; small result | format=CSV, maxRows within cap | POST export | 202, jobId/statusUrl, status=QUEUED |  |  |  |  |  |
| UT-002 | Idempotency | 4-5 | Same caller/request and same idempotency key | POST repeated | POST export | Same jobId returned; no duplicate export job |  |  |  |  |  |
| UT-003 | Bất thường | 2 | Unsupported format | format=PDF | POST export | 400 MSG-E002 |  |  |  |  |  |
| UT-004 | Bảo mật | 2 | columns include blocked PII without permission | columns=[soCmnd] | POST export | 403 MSG-E403; no job created |  |  |  |  |  |
| UT-005 | Biên | 6 | Row count reaches maxRows exactly | maxRows=allowedCap | Worker | Export succeeds at cap; streaming; row count logged |  |  |  |  |  |
| UT-006 | Bất thường | 6-7 | Object storage temporary failure | valid job | Worker execution | Bounded retry; then FAILED with safe errorCode; no public URL |  |  |  |  |  |
| UT-007 | Bất thường | 6 | XLSX > format row limit | large result | POST + worker | Rejected/partitioned as configured; no truncated workbook |  |  |  |  |  |
| UT-008 | Hiệu năng | 8 | Large eligible result | format=CSV | POST export | HTTP responds <=1s target; file generated asynchronously |  |  |  |  |  |

## 8. QA / câu hỏi cần xử lý

| No | Ngày hỏi | Mục | Câu hỏi | Người trả lời | Trả lời/giả định đang dùng | Ngày trả lời | Trạng thái | Đã phản ánh? |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 2026-10-28 | 03/04 | Giới hạn maxRows và TTL file là bao nhiêu? | PO/Ops | Tạm để cấu hình; cần chốt trước hiện thực. | 2026-10-28 | Open | Có trong BD QA-HSSV-005 |
| 2 | 2026-10-28 | 03/04 | Cột PII nào được export cho từng role? | Security/Data Owner | Default safe columns; allowlist và RBAC bắt buộc. | 2026-10-28 | Open | Có trong BD §10 |
