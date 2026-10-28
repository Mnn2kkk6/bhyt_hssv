# Thiết kế chi tiết — API thống kê phân bố thời gian gián đoạn

| Mục | Nội dung |
|---|---|
| Tên dự án | Hệ thống xử lý dữ liệu BHYT HSSV quy mô 200M |
| Hệ thống | Hệ thống tra cứu kết quả BHYT HSSV |
| Tài liệu | Thiết kế chi tiết (Detail Design) |
| Mã tài liệu | DD-API-HSSV-005 |
| Phiên bản / trạng thái | 1.0 / Draft – chờ review |
| Ngày lập | 2026-10-28 |
| Người lập / review / phê duyệt | Chưa điền |
| Tài liệu cơ bản | `HSSV-BD-001` |
| Chức năng | F-HSSV-006 / API-HSSV-005 |
| Nguồn | Repo `https://github.com/Mnn2kkk6/bhyt_hssv`; quy tắc tuần 1–2; template Detail Design đã cung cấp |

## 1. Tổng quan module

| Mục | Nội dung |
|---|---|
| ID module | DD-API-HSSV-005 |
| Hiện thực cho | F-HSSV-006 / API-HSSV-005; Basic Design `HSSV-BD-001` |
| Mức chi tiết | B – API nghiệp vụ / A – batch quy mô lớn |
| Tóm tắt xử lý | Resolve published snapshot → validate dimensions → read precomputed distribution summary → fill missing bucket combinations with zero → calculate shares safely → respond. |
| Lớp/package đề xuất | `HssvController` → `HssvQueryService` / `HssvExportService` → `HssvReadRepository` / `ExportJobRepository` |
| Chữ ký phương thức | `GapDistributionResponse getGapDistribution(String snapshotId, String provinceCode, Block block)` |
| Được gọi bởi | API Gateway / batch orchestrator |
| Gọi tới | PostgreSQL read model, Redis cache nếu phù hợp, Object Storage nếu cần |
| Giao dịch | Đọc dùng read-only transaction; thao tác tạo job dùng transaction ngắn; không giữ transaction trong lúc tạo file |
| Xử lý đồng thời | Snapshot chỉ đọc ở trạng thái `PUBLISHED`; keyset/sắp xếp ổn định; export job có trạng thái và khóa idempotency |

## 2. Định nghĩa vào/ra

| Hướng | No | Tên tham số / thuộc tính | Kiểu | Bắt buộc | Mô tả / miền giá trị | Giá trị khi không có | Tham chiếu BD |
| --- | --- | --- | --- | --- | --- | --- | --- |
| IN | 1 | snapshotId | String query | Không | Snapshot PUBLISHED. | Latest PUBLISHED | API-HSSV-005 |
| IN | 2 | provinceCode | String query | Không | Mã tỉnh theo catalog version. | Không lọc | API-HSSV-005 |
| IN | 3 | block | Enum query | Không | HS/SV. | Cả hai | API-HSSV-005 |
| OUT | 1 | buckets[] | Array | Có | Buckets cố định `090-119`,`120-179`,`180-269`,`270_PLUS`. | Đủ 4 bucket / khối; count=0 nếu không có | BD §8 |
| OUT | 2 | peopleCount/share | Number | Có | Số người và tỷ trọng trên tập đủ điều kiện sau filter. | 0 | BD §8 |
| OUT | 3 | snapshotId/thresholdDays | String/Integer | Có | Snapshot và ngưỡng rule áp dụng. | 90 | BD §8 |

## 3. Xử lý chi tiết

| Bước | Nội dung xử lý | Điều kiện | Khi đúng | Khi sai | SQL / Check / MSG-ID | Ghi chú |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Xác thực caller có `HSSV_REPORT_READ`. | Caller hợp lệ? | Sang 2. | 401/403. | CHK-01 |  |
| 2 | Validate snapshot, provinceCode và block; đọc catalog version để validate mã tỉnh. | Tham số hợp lệ? | Sang 3. | 400/404/409. | CHK-02, SQL-01 |  |
| 3 | Resolve snapshot, áp dụng dimension filter và kiểm tra cache. | Cache hit? | Trả cache sau khi bảo đảm snapshot match. | Cache miss → bước 4. | — | Cache key bao gồm snapshot, tỉnh, khối, ruleVersion. |
| 4 | Đọc summary distribution từ bảng tổng hợp; dùng bucket definitions cố định và sinh hàng 0 cho bucket thiếu. | Có dữ liệu summary? | Map theo thứ tự bucket chuẩn. | Summary thiếu toàn bộ trong snapshot đã publish → MSG-E900. | SQL-02 | Phân bố mặc định chỉ trên result GAP_90_PLUS đã đạt ngưỡng. |
| 5 | Tính tỷ trọng bằng `bucketCount / filteredTotal`; nếu tổng=0, share=0 và không chia cho 0. | FilteredTotal = 0? | Mọi bucket count=0, share=0. | Nếu tổng âm hoặc tổng bucket lệch tổng chi tiết → MSG-E900. | SQL-02, CHK-03 |  |
| 6 | Trả 200 và cache response. | Cache write lỗi? | Trả response; cache lỗi chỉ WARN. | Không trả SQL/stacktrace. | — |  |

## 4. Định nghĩa SQL / truy vấn dữ liệu

Mọi truy vấn dùng bind parameter, không ghép trực tiếp giá trị do client cung cấp vào SQL. Thời gian chờ truy vấn và statement timeout phải cấu hình; khi timeout, hủy truy vấn và trả lỗi có `traceId`.

### SQL-01 — Resolve snapshot/catalog

```sql
SELECT snapshot_id, status, province_catalog_version, gap_threshold_days
FROM hssv_batch_run
WHERE (:snapshotId IS NULL AND status='PUBLISHED') OR snapshot_id=:snapshotId
ORDER BY published_at DESC LIMIT 2;
```

| Thuộc tính | Giá trị |
|---|---|
| Tham số bind | snapshotId optional. |
| Index / partition | PK snapshot; index status/published |
| Số bản ghi kỳ vọng | 0..1 |
| Có dữ liệu | 1 PUBLISHED row → continue. |
| Không có (NOT FOUND) | 0 → 404; non-published → 409. |
| Nhiều hơn kỳ vọng | >1 → MSG-E900. |
| Lỗi DB / Timeout | DB error/timeout → MSG-E503/E504. |
| Khóa / Timeout | Read-only 2s. |

### SQL-02 — Read precomputed distribution

```sql
SELECT bucket_code, ma_khoi, SUM(people_count) AS people_count
FROM hssv_gap_distribution
WHERE snapshot_id=:snapshotId
  AND (:provinceCode IS NULL OR ma_tinh=:provinceCode)
  AND (:block IS NULL OR ma_khoi=:block)
GROUP BY bucket_code, ma_khoi
ORDER BY CASE bucket_code WHEN '090-119' THEN 1 WHEN '120-179' THEN 2
                          WHEN '180-269' THEN 3 WHEN '270_PLUS' THEN 4 ELSE 99 END,
         ma_khoi;
```

| Thuộc tính | Giá trị |
|---|---|
| Tham số bind | SnapshotId; optional provinceCode/block. |
| Index / partition | Index `(snapshot_id, ma_tinh, ma_khoi, bucket_code)` |
| Số bản ghi kỳ vọng | 0..8 rows before service fills bucket zeros; max 4 when block filter set |
| Có dữ liệu | Rows map; service fills absent bucket combinations with zero. |
| Không có (NOT FOUND) | No rows may mean no eligible person; return all buckets as 0 if the snapshot's total confirms zero. |
| Nhiều hơn kỳ vọng | Unknown bucket or >8 combinations → MSG-E900. |
| Lỗi DB / Timeout | DB failure/timeout → MSG-E503/E504. |
| Khóa / Timeout | Read summary only; 2s timeout. |

## 5. Đặc tả kiểm tra đầu vào

| Check-ID | Hạng mục | Loại kiểm tra | Điều kiện lỗi | Mã thông báo | Thứ tự | Dừng ngay? | Tham chiếu BD |
| --- | --- | --- | --- | --- | --- | --- | --- |
| CHK-01 | Authorization | RBAC | Không có quyền đọc báo cáo | MSG-E401/E403 | 1 | Có | BD §6 |
| CHK-02 | Filters | Enum/catalog | Mã tỉnh không có trong catalog hoặc block sai | MSG-E002 | 2 | Có | BD §6 |
| CHK-03 | Summary consistency | Arithmetic | Tổng bucket không khớp tổng người đủ điều kiện hoặc share NaN/Infinity | MSG-E900 | 3 | Có | BD §11 |

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
| UT-001 | Bình thường | 4-6 | Output has all four buckets across HS/SV | no filters | GET distribution | 200, 8 combinations ordered correctly; counts and share valid |  |  |  |  |  |
| UT-002 | Lọc | 2-5 | One province has only one block represented | provinceCode=01, block=HS | GET distribution | 200, four HS buckets; absent buckets are zero |  |  |  |  |  |
| UT-003 | Rỗng | 4-5 | No eligible persons in filter | province with zero | GET distribution | 200; four bucket rows, counts=0, shares=0 |  |  |  |  |  |
| UT-004 | Bất thường | 4-5 | Unknown bucket or inconsistent total | snapshot summary corrupted | GET distribution | 500 MSG-E900; alert and no partial report |  |  |  |  |  |
| UT-005 | Biên | 5 | filteredTotal=0 | share calculation | GET distribution | No divide-by-zero; all shares 0 |  |  |  |  |  |

## 8. QA / câu hỏi cần xử lý

| No | Ngày hỏi | Mục | Câu hỏi | Người trả lời | Trả lời/giả định đang dùng | Ngày trả lời | Trạng thái | Đã phản ánh? |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 2026-10-28 | 03/05 | Kiểm tra ngưỡng/ngày danh mục | Đã kế thừa quy tắc tuần 1–2; câu hỏi nghiệp vụ chưa phê duyệt được ghi ở Basic Design QA. | BA/PO | Giữ `>=90 ngày`; dùng danh mục gắn snapshot. | 2026-10-28 | Open | Chưa |
