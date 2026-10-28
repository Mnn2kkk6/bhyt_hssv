# Thiết kế chi tiết — API thống kê theo tỉnh/TP

| Mục | Nội dung |
|---|---|
| Tên dự án | Hệ thống xử lý dữ liệu BHYT HSSV quy mô 200M |
| Hệ thống | Hệ thống tra cứu kết quả BHYT HSSV |
| Tài liệu | Thiết kế chi tiết (Detail Design) |
| Mã tài liệu | DD-API-HSSV-004 |
| Phiên bản / trạng thái | 1.0 / Draft – chờ review |
| Ngày lập | 2026-10-28 |
| Người lập / review / phê duyệt | Chưa điền |
| Tài liệu cơ bản | `HSSV-BD-001` |
| Chức năng | F-HSSV-005 / API-HSSV-004 |
| Nguồn | Repo `https://github.com/Mnn2kkk6/bhyt_hssv`; quy tắc tuần 1–2; template Detail Design đã cung cấp |

## 1. Tổng quan module

| Mục | Nội dung |
|---|---|
| ID module | DD-API-HSSV-004 |
| Hiện thực cho | F-HSSV-005 / API-HSSV-004; Basic Design `HSSV-BD-001` |
| Mức chi tiết | B – API nghiệp vụ / A – batch quy mô lớn |
| Tóm tắt xử lý | Resolve snapshot and its province catalog version → use cache or precomputed summary → require one row per catalog unit → validate totals → respond. |
| Lớp/package đề xuất | `HssvController` → `HssvQueryService` / `HssvExportService` → `HssvReadRepository` / `ExportJobRepository` |
| Chữ ký phương thức | `ProvinceReportResponse getProvinceReport(String snapshotId, Criteria criteria, Block block)` |
| Được gọi bởi | API Gateway / batch orchestrator |
| Gọi tới | PostgreSQL read model, Redis cache nếu phù hợp, Object Storage nếu cần |
| Giao dịch | Đọc dùng read-only transaction; thao tác tạo job dùng transaction ngắn; không giữ transaction trong lúc tạo file |
| Xử lý đồng thời | Snapshot chỉ đọc ở trạng thái `PUBLISHED`; keyset/sắp xếp ổn định; export job có trạng thái và khóa idempotency |

## 2. Định nghĩa vào/ra

| Hướng | No | Tên tham số / thuộc tính | Kiểu | Bắt buộc | Mô tả / miền giá trị | Giá trị khi không có | Tham chiếu BD |
| --- | --- | --- | --- | --- | --- | --- | --- |
| IN | 1 | snapshotId | String query | Không | Snapshot PUBLISHED. | Latest PUBLISHED | API-HSSV-004 |
| IN | 2 | criteria | Enum query | Không | STOPPED hoặc GAP_90_PLUS; null trả hai cột chỉ tiêu. | Hai tiêu chí | API-HSSV-004 |
| IN | 3 | block | Enum query | Không | HS hoặc SV. | Cả HS và SV | API-HSSV-004 |
| OUT | 1 | items[] | Array | Có | Mỗi tỉnh/TP trong danh mục version snapshot xuất hiện đúng một lần. | 63 items nếu danh mục có 63 đơn vị | BD §8 |
| OUT | 2 | counts | Integer | Có | Tách học sinh, sinh viên, tổng theo tiêu chí. | 0 | BD §8 |
| OUT | 3 | snapshotId/generatedAt | String | Có | Nguồn số liệu và thời điểm tổng hợp. | — | BD §8 |

## 3. Xử lý chi tiết

| Bước | Nội dung xử lý | Điều kiện | Khi đúng | Khi sai | SQL / Check / MSG-ID | Ghi chú |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Authenticate/authorize caller với `HSSV_REPORT_READ`. | Có token/role hợp lệ? | Sang bước 2. | 401/403. | CHK-01 |  |
| 2 | Validate snapshotId, criteria và block; resolve snapshot PUBLISHED. | Tham số và snapshot hợp lệ? | Sang bước 3. | 400/404/409 phù hợp. | CHK-02, SQL-01 |  |
| 3 | Kiểm tra cache key theo snapshot + filter; nếu cache hit, kiểm tra TTL/version. | Có cache hit chưa hết hạn? | Trả từ cache; bỏ qua DB read. | Cache miss → bước 4. | — | Invalidate khi snapshot mới publish. |
| 4 | Đọc hssv_province_report và join danh mục versioned để giữ tỉnh/TP có count 0. | Số dòng là đúng cardinality catalog? | Map cả hai nhóm metrics (STOPPED và GAP_90_PLUS) trong cùng dòng tỉnh. | Không đủ tỉnh hoặc trùng tỉnh → MSG-E900, không trả báo cáo một phần. | SQL-02 | criteria/block chỉ điều khiển metrics trả trong DTO; không nhân đôi tỉnh. |
| 5 | Tính `totalCount = studentCount + universityStudentCount` phù hợp tiêu chí; không cộng trùng các tiêu chí khi criteria null. | Các số tổng khớp? | Sang bước 6. | Mismatch summary → MSG-E900. | CHK-03 |  |
| 6 | Cache response và trả HTTP 200; ghi duration/rowCount. | Có lỗi cache? | Response DB vẫn trả được; cache lỗi log WARN. | Không để lỗi cache làm hỏng response đã có. | — |  |

## 4. Định nghĩa SQL / truy vấn dữ liệu

Mọi truy vấn dùng bind parameter, không ghép trực tiếp giá trị do client cung cấp vào SQL. Thời gian chờ truy vấn và statement timeout phải cấu hình; khi timeout, hủy truy vấn và trả lỗi có `traceId`.

### SQL-01 — Resolve snapshot and catalog version

```sql
SELECT snapshot_id, status, province_catalog_version
FROM hssv_batch_run
WHERE (:snapshotId IS NULL AND status='PUBLISHED') OR snapshot_id=:snapshotId
ORDER BY published_at DESC LIMIT 2;
```

| Thuộc tính | Giá trị |
|---|---|
| Tham số bind | snapshotId optional. |
| Index / partition | PK snapshot; published index |
| Số bản ghi kỳ vọng | 0..1 |
| Có dữ liệu | PUBLISHED snapshot → continue. |
| Không có (NOT FOUND) | 0 → MSG-I404; not PUBLISHED → MSG-E409. |
| Nhiều hơn kỳ vọng | >1 → MSG-E900. |
| Lỗi DB / Timeout | DB error/timeout → MSG-E503/E504. |
| Khóa / Timeout | Read-only 2s. |

### SQL-02 — Đọc summary tỉnh/TP, giữ cả tỉnh có số lượng 0

```sql
SELECT d.ma_dvi AS province_code, d.ten_dvi AS province_name,
       COALESCE(r.stopped_hs,0) AS stopped_student_count,
       COALESCE(r.stopped_sv,0) AS stopped_university_student_count,
       COALESCE(r.gap90_hs,0) AS gap90_plus_student_count,
       COALESCE(r.gap90_sv,0) AS gap90_plus_university_student_count
FROM dm_bhxh_versioned d
LEFT JOIN hssv_province_report r
  ON r.snapshot_id=:snapshotId AND r.ma_tinh=d.ma_dvi
WHERE d.ma_cha='00' AND d.catalog_version=:catalogVersion
ORDER BY d.ma_dvi ASC;
```

| Thuộc tính | Giá trị |
|---|---|
| Tham số bind | snapshotId, catalogVersion; criteria/block lọc/shape response ở service, không làm nhân đôi province rows. |
| Index / partition | Index catalog `(catalog_version, ma_cha, ma_dvi)`; report unique `(snapshot_id, ma_tinh)` |
| Số bản ghi kỳ vọng | Đúng cardinality catalog (63 với snapshot danh mục bài tập) |
| Có dữ liệu | 63 dòng → map; zero counters giữ là 0. |
| Không có (NOT FOUND) | 0 dòng → catalog version thiếu, MSG-E900 thay vì trả report rỗng. |
| Nhiều hơn kỳ vọng | > expected cardinality hoặc duplicate ma_tinh → MSG-E900. |
| Lỗi DB / Timeout | DB error/timeout → MSG-E503/E504. |
| Khóa / Timeout | Read-only; timeout 2s; summary table, không scan raw. |

## 5. Đặc tả kiểm tra đầu vào

| Check-ID | Hạng mục | Loại kiểm tra | Điều kiện lỗi | Mã thông báo | Thứ tự | Dừng ngay? | Tham chiếu BD |
| --- | --- | --- | --- | --- | --- | --- | --- |
| CHK-01 | Authorization | RBAC | Caller không có quyền báo cáo | MSG-E401/E403 | 1 | Có | BD §6 |
| CHK-02 | snapshot/filters | Enum/status | criteria, block sai; snapshot không tồn tại/chưa publish | MSG-E002/MSG-I404/MSG-E409 | 2 | Có | BD §6 |
| CHK-03 | Completeness | Data integrity | Báo cáo thiếu tỉnh theo catalog hoặc count âm/lệch | MSG-E900 | 3 | Có | BD §11 |

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
| UT-001 | Bình thường | 4-6 | 63 tỉnh/TP trong catalog | snapshot latest, no filters | GET provinces | 200, 63 unique province rows; zero rows preserved |  |  |  |  |  |
| UT-002 | Lọc | 2-5 | Summary có HS và SV counts | criteria=GAP_90_PLUS, block=SV | GET provinces | 200, SV counts only per documented contract |  |  |  |  |  |
| UT-003 | Bất thường | 2 | Snapshot RUNNING | snapshotId=running | GET provinces | 409 MSG-E409 |  |  |  |  |  |
| UT-004 | Bất thường | 4-5 | Mock catalog có thiếu tỉnh hoặc duplicate code | snapshot latest | GET provinces | 500 MSG-E900; alert data completeness |  |  |  |  |  |
| UT-005 | Hiệu năng | 3-6 | Summary prepared; cold and warm cache | no filters | GET provinces | p95 <=1s target; cache invalidated on publish |  |  |  |  |  |

## 8. QA / câu hỏi cần xử lý

| No | Ngày hỏi | Mục | Câu hỏi | Người trả lời | Trả lời/giả định đang dùng | Ngày trả lời | Trạng thái | Đã phản ánh? |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 2026-10-28 | 03/05 | Kiểm tra ngưỡng/ngày danh mục | Đã kế thừa quy tắc tuần 1–2; câu hỏi nghiệp vụ chưa phê duyệt được ghi ở Basic Design QA. | BA/PO | Giữ `>=90 ngày`; dùng danh mục gắn snapshot. | 2026-10-28 | Open | Chưa |
