# Thiết kế chi tiết — API tra cứu danh sách người

| Mục | Nội dung |
|---|---|
| Tên dự án | Hệ thống xử lý dữ liệu BHYT HSSV quy mô 200M |
| Hệ thống | Hệ thống tra cứu kết quả BHYT HSSV |
| Tài liệu | Thiết kế chi tiết (Detail Design) |
| Mã tài liệu | DD-API-HSSV-001 |
| Phiên bản / trạng thái | 1.0 / Draft – chờ review |
| Ngày lập | 2026-10-28 |
| Người lập / review / phê duyệt | Chưa điền |
| Tài liệu cơ bản | `HSSV-BD-001` |
| Chức năng | F-HSSV-002 / API-HSSV-001 |
| Nguồn | Repo `https://github.com/Mnn2kkk6/bhyt_hssv`; quy tắc tuần 1–2; template Detail Design đã cung cấp |

## 1. Tổng quan module

| Mục | Nội dung |
|---|---|
| ID module | DD-API-HSSV-001 |
| Hiện thực cho | F-HSSV-002 / API-HSSV-001; Basic Design `HSSV-BD-001` |
| Mức chi tiết | B – API nghiệp vụ / A – batch quy mô lớn |
| Tóm tắt xử lý | Validate filter và quyền → resolve published snapshot → count → query một trang ổn định → mask trường nhạy cảm → JSON response. |
| Lớp/package đề xuất | `HssvController` → `HssvQueryService` / `HssvExportService` → `HssvReadRepository` / `ExportJobRepository` |
| Chữ ký phương thức | `PersonPageResponse listPersons(PersonQuery query, PageRequest pageRequest)` |
| Được gọi bởi | API Gateway / batch orchestrator |
| Gọi tới | PostgreSQL read model, Redis cache nếu phù hợp, Object Storage nếu cần |
| Giao dịch | Đọc dùng read-only transaction; thao tác tạo job dùng transaction ngắn; không giữ transaction trong lúc tạo file |
| Xử lý đồng thời | Snapshot chỉ đọc ở trạng thái `PUBLISHED`; keyset/sắp xếp ổn định; export job có trạng thái và khóa idempotency |

## 2. Định nghĩa vào/ra

| Hướng | No | Tên tham số / thuộc tính | Kiểu | Bắt buộc | Mô tả / miền giá trị | Giá trị khi không có | Tham chiếu BD |
| --- | --- | --- | --- | --- | --- | --- | --- |
| IN | 1 | provinceCode | String | Không | Mã tỉnh trong danh mục được gắn với snapshot. | Không lọc | API-HSSV-001 Request |
| IN | 2 | block | Enum | Không | HS hoặc SV. | Không lọc | API-HSSV-001 Request |
| IN | 3 | criteria | Enum | Không | STOPPED hoặc GAP_90_PLUS. | Cả hai tiêu chí | API-HSSV-001 Request |
| IN | 4 | snapshotId | String | Không | Chỉ snapshot PUBLISHED. | Latest PUBLISHED | BD §6 |
| IN | 5 | page/pageSize | Integer | Không | page >= 1; pageSize 1..100. | 1 / 20 | BD §6 |
| OUT | 1 | snapshotId/criteria | String | Có | Phiên bản và tiêu chí thực tế áp dụng. | — | BD §8 |
| OUT | 2 | items[] | Array | Có | Danh sách tóm tắt người, luôn [] khi không có dữ liệu. | [] | BD §8 |
| OUT | 3 | page/pageSize/totalItems/totalPages | Integer | Có | Thông tin phân trang. | — | BD §8 |
| OUT | 4 | traceId | String | Có | Mã truy vết yêu cầu. | — | BD §8 |

## 3. Xử lý chi tiết

| Bước | Nội dung xử lý | Điều kiện | Khi đúng | Khi sai | SQL / Check / MSG-ID | Ghi chú |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Sinh hoặc nhận traceId; xác thực token và quyền `HSSV_RESULT_READ`. | Caller có token và role hợp lệ? | Tiếp tục bước 2. | 401 `MSG-E401` hoặc 403 `MSG-E403`; ghi audit. | CHK-01 | Không log token. |
| 2 | Chuẩn hóa query; xác thực `provinceCode`, `block`, `criteria`, `snapshotId`, `page`, `pageSize`. | Tất cả tham số hợp lệ? | Tiếp tục bước 3. | Trả 400 theo mã MSG tương ứng, không truy vấn kết quả. | CHK-02..CHK-04 | pageSize tối đa 100. |
| 3 | Resolve snapshot mặc định hoặc snapshotId được yêu cầu. | Snapshot tồn tại và PUBLISHED? | Gắn `snapshotId` vào query context. | Không tồn tại → 404; chưa publish → 409 `MSG-E409`. | SQL-01, MSG-I404/MSG-E409 | Không đọc staging. |
| 4 | Trong read-only transaction, chạy SQL-02 lấy `totalItems` với filter áp dụng. | Count trả về aggregate? | Lấy totalItems; totalPages = ceil(totalItems/pageSize). | DB timeout → 503/504; không trả kết quả một phần. | SQL-02 | Cache query key nếu bộ lọc phổ biến. |
| 5 | Chạy SQL-03 lấy page; sort ổn định theo khóa nghiệp vụ và `SO_SO_BHXH`. | Items < pageSize? | Trả danh sách, rỗng là hợp lệ. | Nếu DB lỗi/timeout → hủy transaction và lỗi chuẩn. | SQL-03 | Không dùng SELECT *; chỉ lấy field cần response. |
| 6 | Map DTO, mask PII theo role và trả HTTP 200. | Response có dữ liệu? | Items có thể []; trả metadata phân trang. | Không ghi danh tính đầy đủ vào log. | — | Ghi metric latency, returnedCount. |

## 4. Định nghĩa SQL / truy vấn dữ liệu

Mọi truy vấn dùng bind parameter, không ghép trực tiếp giá trị do client cung cấp vào SQL. Thời gian chờ truy vấn và statement timeout phải cấu hình; khi timeout, hủy truy vấn và trả lỗi có `traceId`.

### SQL-01 — Lấy snapshot PUBLISHED mới nhất hoặc theo ID

```sql
SELECT snapshot_id, status, published_at
FROM hssv_batch_run
WHERE (:snapshotId IS NULL AND status = 'PUBLISHED')
   OR snapshot_id = :snapshotId
ORDER BY published_at DESC
LIMIT 2;
```

| Thuộc tính | Giá trị |
|---|---|
| Tham số bind | `:snapshotId` null để lấy latest, nếu không null tìm đúng ID. |
| Index / partition | IDX_BATCH_STATUS_PUBLISHED, PK snapshot_id |
| Số bản ghi kỳ vọng | 0..1; hơn 1 là inconsistency |
| Có dữ liệu | 1 dòng PUBLISHED → dùng snapshotId. |
| Không có (NOT FOUND) | 0 dòng → `MSG-I404`; status khác PUBLISHED → `MSG-E409`. |
| Nhiều hơn kỳ vọng | >1 dòng → `MSG-E900`. |
| Lỗi DB / Timeout | DB exception → MSG-E503; timeout → MSG-E504. |
| Khóa / Timeout | Read-only; timeout 2s. |

### SQL-02 — Đếm tổng theo filter

```sql
SELECT COUNT(*) AS total_items
FROM hssv_person_result
WHERE snapshot_id = :snapshotId
  AND (:criteria IS NULL OR criteria = :criteria)
  AND (:provinceCode IS NULL OR ma_tinh = :provinceCode)
  AND (:block IS NULL OR ma_khoi = :block);
```

| Thuộc tính | Giá trị |
|---|---|
| Tham số bind | snapshotId đã resolve; filter được bind. |
| Index / partition | Index `(snapshot_id, criteria, ma_tinh, ma_khoi, so_so_bhxh)` |
| Số bản ghi kỳ vọng | Đúng 1 hàng aggregate |
| Có dữ liệu | COUNT có thể >0; lấy số tổng. |
| Không có (NOT FOUND) | COUNT=0 là hợp lệ; trả items=[] và totalPages=0. |
| Nhiều hơn kỳ vọng | Aggregate luôn 1 hàng; nếu không trả 1 hàng → lỗi truy vấn. |
| Lỗi DB / Timeout | DB error/timeout → MSG-E503/E504. |
| Khóa / Timeout | Read-only tx; target 3s; cache miss may differ. |

### SQL-03 — Lấy dữ liệu của trang

```sql
SELECT so_so_bhxh, full_name, ma_khoi, ma_tinh, ten_tinh,
       criteria, total_gap_days, gap_count
FROM hssv_person_result
WHERE snapshot_id = :snapshotId
  AND (:criteria IS NULL OR criteria = :criteria)
  AND (:provinceCode IS NULL OR ma_tinh = :provinceCode)
  AND (:block IS NULL OR ma_khoi = :block)
ORDER BY so_so_bhxh ASC
LIMIT :pageSize OFFSET :offset;
```

| Thuộc tính | Giá trị |
|---|---|
| Tham số bind | `:offset=(page-1)*pageSize`; pageSize <=100. |
| Index / partition | Serving index; sort via composite index where possible |
| Số bản ghi kỳ vọng | 0..pageSize |
| Có dữ liệu | 1..pageSize → map `items[]`. |
| Không có (NOT FOUND) | 0 rows → empty items; page may be beyond last page while total>0. |
| Nhiều hơn kỳ vọng | More than pageSize indicates query/driver contract error; discard response and MSG-E900. |
| Lỗi DB / Timeout | DB failure/timeout → MSG-E503/E504. |
| Khóa / Timeout | Read-only; timeout 3s; deep offsets require load test. |

## 5. Đặc tả kiểm tra đầu vào

| Check-ID | Hạng mục | Loại kiểm tra | Điều kiện lỗi | Mã thông báo | Thứ tự | Dừng ngay? | Tham chiếu BD |
| --- | --- | --- | --- | --- | --- | --- | --- |
| CHK-01 | Authorization | RBAC | Thiếu token hoặc role không có quyền đọc kết quả | MSG-E401/E403 | 1 | Có | BD §6 |
| CHK-02 | provinceCode/block/criteria | Enum/catalog | Mã không thuộc danh mục hoặc enum không hợp lệ | MSG-E002 | 2 | Có | BD §6 |
| CHK-03 | snapshotId | Existence/status | Snapshot không tồn tại hoặc không PUBLISHED | MSG-I404/MSG-E409 | 3 | Có | BD §6 |
| CHK-04 | page/pageSize | Range | page < 1 hoặc pageSize ngoài 1..100 | MSG-E003 | 4 | Có | BD §6 |

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
| UT-001 | Bình thường | 2-6 | Có 25 kết quả, snapshot PUBLISHED | provinceCode=01, block=SV, criteria=GAP_90_PLUS, page=1,pageSize=20 | GET persons | 200, 20 items, totalItems=25, totalPages=2 |  |  |  |  |  |
| UT-002 | Rỗng | 4-6 | Không kết quả phù hợp filter | provinceCode=99 nếu code hợp lệ nhưng không có result | GET persons | 200, items=[], totalItems=0 |  |  |  |  |  |
| UT-003 | Biên | 2 | Catalog hợp lệ | page=1,pageSize=100 | GET persons | 200; pageSize được chấp nhận |  |  |  |  |  |
| UT-004 | Bất thường | 2 | — | page=0 hoặc pageSize=101 | GET persons | 400 MSG-E003 |  |  |  |  |  |
| UT-005 | Bất thường | 3 | Snapshot đang RUNNING | snapshotId=staging-run | GET persons | 409 MSG-E409; không trả dữ liệu staging |  |  |  |  |  |
| UT-006 | Bảo mật | 1 | Caller thiếu quyền đọc | token không hợp lệ/role không đủ | GET persons | 401/403, audit ghi caller và traceId |  |  |  |  |  |
| UT-007 | Hiệu năng | 4-5 | Dữ liệu quy mô gần 200M source, serving dataset lớn | Filter tỉnh/khối/tiêu chí phổ biến | GET persons | p95 mục tiêu <=1s phải được load test xác nhận |  |  |  |  |  |

## 8. QA / câu hỏi cần xử lý

| No | Ngày hỏi | Mục | Câu hỏi | Người trả lời | Trả lời/giả định đang dùng | Ngày trả lời | Trạng thái | Đã phản ánh? |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 2026-10-28 | 03/05 | Kiểm tra ngưỡng/ngày danh mục | Đã kế thừa quy tắc tuần 1–2; câu hỏi nghiệp vụ chưa phê duyệt được ghi ở Basic Design QA. | BA/PO | Giữ `>=90 ngày`; dùng danh mục gắn snapshot. | 2026-10-28 | Open | Chưa |
