# Thiết kế chi tiết — API xem trạng thái job xuất

| Mục | Nội dung |
|---|---|
| Tên dự án | Hệ thống xử lý dữ liệu BHYT HSSV quy mô 200M |
| Hệ thống | Hệ thống tra cứu kết quả BHYT HSSV |
| Tài liệu | Thiết kế chi tiết (Detail Design) |
| Mã tài liệu | DD-API-HSSV-007 |
| Phiên bản / trạng thái | 1.0 / Draft – chờ review |
| Ngày lập | 2026-10-28 |
| Người lập / review / phê duyệt | Chưa điền |
| Tài liệu cơ bản | `HSSV-BD-001` |
| Chức năng | F-HSSV-008 / API-HSSV-007 |
| Nguồn | Repo `https://github.com/Mnn2kkk6/bhyt_hssv`; quy tắc tuần 1–2; template Detail Design đã cung cấp |

## 1. Tổng quan module

| Mục | Nội dung |
|---|---|
| ID module | DD-API-HSSV-007 |
| Hiện thực cho | F-HSSV-008 / API-HSSV-007; Basic Design `HSSV-BD-001` |
| Mức chi tiết | B – API nghiệp vụ / A – batch quy mô lớn |
| Tóm tắt xử lý | Validate job owner/role → load status by jobId → return progress for active jobs or short-lived signed download URL for successful jobs → hide internal storage details. |
| Lớp/package đề xuất | `HssvController` → `HssvQueryService` / `HssvExportService` → `HssvReadRepository` / `ExportJobRepository` |
| Chữ ký phương thức | `ExportJobStatusResponse getExportJobStatus(String jobId, AuthenticatedCaller caller)` |
| Được gọi bởi | API Gateway / batch orchestrator |
| Gọi tới | PostgreSQL read model, Redis cache nếu phù hợp, Object Storage nếu cần |
| Giao dịch | Đọc dùng read-only transaction; thao tác tạo job dùng transaction ngắn; không giữ transaction trong lúc tạo file |
| Xử lý đồng thời | Snapshot chỉ đọc ở trạng thái `PUBLISHED`; keyset/sắp xếp ổn định; export job có trạng thái và khóa idempotency |

## 2. Định nghĩa vào/ra

| Hướng | No | Tên tham số / thuộc tính | Kiểu | Bắt buộc | Mô tả / miền giá trị | Giá trị khi không có | Tham chiếu BD |
| --- | --- | --- | --- | --- | --- | --- | --- |
| IN | 1 | jobId | String path | Có | ID job do server tạo; không suy diễn object key từ ID. | — | API-HSSV-007 |
| OUT | 1 | jobId/status | String/Enum | Có | QUEUED, RUNNING, SUCCEEDED, FAILED, EXPIRED. | — | BD §8 |
| OUT | 2 | progressPercent/rowCount | Integer/Long | Không | Có khi worker cập nhật. | null nếu chưa có | BD §8 |
| OUT | 3 | downloadUrl/expiresAt | String/Timestamp | Có điều kiện | Chỉ có khi SUCCEEDED và file chưa hết hạn. | Không có | BD §8 |
| OUT | 4 | errorCode | String | Có điều kiện | Mã lỗi an toàn khi job FAILED. | Không có | BD §8 |

## 3. Xử lý chi tiết

| Bước | Nội dung xử lý | Điều kiện | Khi đúng | Khi sai | SQL / Check / MSG-ID | Ghi chú |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Xác thực token và quyền owner hoặc role `HSSV_REPORT_EXPORT_ADMIN`. | Caller có quyền xem job? | Sang bước 2. | 401/403; không tiết lộ job người khác. | CHK-01 |  |
| 2 | Validate jobId và load job metadata. | ID hợp lệ? | Sang bước 3. | 400 MSG-E002 nếu malformed; 404 MSG-I404 nếu không có quyền truy cập/không tồn tại theo policy. | CHK-02, SQL-01 |  |
| 3 | Xem trạng thái job. | QUEUED/RUNNING? | Trả status/progress, không có downloadUrl. | SUCCEEDED/FAILED/EXPIRED → xử lý theo bước 4. | — |  |
| 4 | Nếu SUCCEEDED, kiểm tra expiry và xác thực object tồn tại trước khi cấp signed URL ngắn hạn. | File còn hiệu lực? | Trả downloadUrl và expiresAt. | Hết hạn → trạng thái EXPIRED/404 link; file không tồn tại → alert, MSG-E900. | SQL-01 | Không trả object key nội bộ. |
| 5 | Nếu FAILED, trả errorCode/message an toàn; không trả stacktrace hoặc đường dẫn storage. | errorCode có trong allowlist? | Trả lỗi đã chuẩn hóa. | Unknown code → MSG-E500 chung. | — |  |
| 6 | Ghi access audit cho status/download link và trả 200. | Audit ghi lỗi? | Trả response; log WARN. | Không lưu signed URL đầy đủ trong log. | — |  |

## 4. Định nghĩa SQL / truy vấn dữ liệu

Mọi truy vấn dùng bind parameter, không ghép trực tiếp giá trị do client cung cấp vào SQL. Thời gian chờ truy vấn và statement timeout phải cấu hình; khi timeout, hủy truy vấn và trả lỗi có `traceId`.

### SQL-01 — Đọc trạng thái job

```sql
SELECT job_id, caller_id, snapshot_id, status, progress_percent,
       row_count, error_code, object_key, created_at, expires_at, updated_at
FROM hssv_export_job
WHERE job_id=:jobId
LIMIT 2;
```

| Thuộc tính | Giá trị |
|---|---|
| Tham số bind | jobId server-generated. |
| Index / partition | PK job_id |
| Số bản ghi kỳ vọng | 0..1 |
| Có dữ liệu | 1 row and caller permitted → return safe status DTO. |
| Không có (NOT FOUND) | 0 row → MSG-I404. |
| Nhiều hơn kỳ vọng | >1 row impossible → MSG-E900. |
| Lỗi DB / Timeout | DB error/timeout → MSG-E503/E504. |
| Khóa / Timeout | Read-only; timeout 1s. |

## 5. Đặc tả kiểm tra đầu vào

| Check-ID | Hạng mục | Loại kiểm tra | Điều kiện lỗi | Mã thông báo | Thứ tự | Dừng ngay? | Tham chiếu BD |
| --- | --- | --- | --- | --- | --- | --- | --- |
| CHK-01 | Authorization/ownership | RBAC | Caller không phải owner hoặc admin | MSG-E401/E403/MSG-I404 | 1 | Có | BD §6 |
| CHK-02 | jobId | Format/existence | jobId sai định dạng hoặc không có job | MSG-E002/MSG-I404 | 2 | Có | BD §8 |
| CHK-03 | download expiry | TTL/object store | Job SUCCEEDED nhưng file hết TTL hoặc object mất | MSG-I404/MSG-E900 | 3 | Có | BD §8 |

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
| UT-001 | Bình thường | 3-6 | Job RUNNING do caller tạo | jobId hợp lệ | GET export status | 200, RUNNING/progress, không có URL |  |  |  |  |  |
| UT-002 | Bình thường | 4 | Job SUCCEEDED, file còn hạn | jobId owner | GET export status | 200, signed URL có TTL và expiresAt |  |  |  |  |  |
| UT-003 | Biên | 4 | Job SUCCEEDED nhưng đã hết TTL | jobId expired | GET export status | EXPIRED hoặc 404 link; không cấp URL mới |  |  |  |  |  |
| UT-004 | Bất thường | 2 | Job không tồn tại | jobId unknown | GET export status | 404 MSG-I404 |  |  |  |  |  |
| UT-005 | Bảo mật | 1 | Job thuộc user khác | caller non-owner | GET export status | 403 hoặc 404 theo policy; không lộ trạng thái job |  |  |  |  |  |
| UT-006 | Bất thường | 5 | Worker failure contains stack trace | job status FAILED | GET export status | Chỉ errorCode/message an toàn; không stacktrace |  |  |  |  |  |

## 8. QA / câu hỏi cần xử lý

| No | Ngày hỏi | Mục | Câu hỏi | Người trả lời | Trả lời/giả định đang dùng | Ngày trả lời | Trạng thái | Đã phản ánh? |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 2026-10-28 | 03/05 | Kiểm tra ngưỡng/ngày danh mục | Đã kế thừa quy tắc tuần 1–2; câu hỏi nghiệp vụ chưa phê duyệt được ghi ở Basic Design QA. | BA/PO | Giữ `>=90 ngày`; dùng danh mục gắn snapshot. | 2026-10-28 | Open | Chưa |
